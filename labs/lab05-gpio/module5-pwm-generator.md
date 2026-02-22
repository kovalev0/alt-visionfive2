# module6 · Программный ШИМ: модуль ядра генерирует меандр

← [Назад](module4-driver-gpio.md) · [На главную](../INDEX.md) · [Следующий →](module6-terminal-scope.md)

> **Kernel docs:** [kernel.org/doc/html/latest/driver-api/gpio/consumer.html](https://www.kernel.org/doc/html/latest/driver-api/gpio/consumer.html)

> **Kernel source:** `include/linux/kthread.h`, `include/linux/hrtimer.h`

---

## Что будет реализовано

Модуль `gpio_pwm` запускает поток ядра (`kthread`), который программно
генерирует прямоугольный сигнал (меандр) на **GPIO60 (пин 37)**.
Через sysfs в реальном времени можно менять:

- `frequency_hz`  — частоту сигнала (1 — 500 Гц)
- `duty_percent`  — скважность в % (1 — 99)
- `enable`        — вкл/выкл генерацию (0/1)

**GPIO61 (пин 38)** настроен как вход — читает то, что подаётся
с GPIO60 через [перемычку](README.md#что-понадобится). Это позволяет модулю проверять
собственный выходной сигнал.

```
  пин 37  GPIO60  (OUT) ──────┐  генератор меандра
                              │
  пин 38  GPIO61  (IN)  ──────┘  (loopback — тот же провод dupont)
```

Схема соединения та же, что в module4: провод dupont между пином 37
и пином 38.

```
  /sys/kernel/gpio_pwm/
  ├── enable         — write 0/1 для вкл/выкл генерации
  ├── frequency_hz   — частота в Гц (1–500)
  ├── duty_percent   — скважность в % (1–99)
  └── in_value       — чтение текущего состояния GPIO61 (вход)
```

---

## Почему kthread, а не hrtimer

`hrtimer` — прерывание от таймера высокого разрешения, работает в
контексте прерывания. Вызывать `gpiod_set_value()` из прерывания можно,
но задержки между фронтами будут зависеть от jitter самого таймера.

`kthread` + `usleep_range()` — поток ядра, работает в контексте процесса.
Позволяет без проблем вызывать любые функции GPIO. При низких частотах
(до ~500 Гц) точность достаточная для демонстрационных целей. При частотах
выше 1 кГц рекомендуется переходить на hrtimer или аппаратный ШИМ.

---

## Код модуля

```bash
$ mkdir -p ~/labs/lab05/gpio_pwm
$ cd ~/labs/lab05/gpio_pwm
```

Файл `gpio_pwm.c`:

```c
// gpio_pwm.c — software PWM generator on GPIO60 with sysfs control
//
// GPIO60 (pin 37): output — square wave generator
// GPIO61 (pin 38): input  — loopback readback
//
// Uses a kthread that toggles GPIO60 at the configured frequency/duty.
//
// sysfs interface: /sys/kernel/gpio_pwm/
//   enable        — write 0/1 to start/stop generation
//   frequency_hz  — frequency in Hz (1–500)
//   duty_percent  — duty cycle in % (1–99)
//   in_value      — read current state of GPIO61 (loopback input)

#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/init.h>
#include <linux/kobject.h>
#include <linux/sysfs.h>
#include <linux/gpio/consumer.h>
#include <linux/kthread.h>
#include <linux/delay.h>
#include <linux/atomic.h>
#include <linux/mutex.h>

MODULE_LICENSE("GPL");
MODULE_AUTHOR("lab05");
MODULE_DESCRIPTION("Software PWM on GPIO60: square wave generator with sysfs control");

/* ------------------------------------------------------------------ */
/* Параметры генератора (защищены gpio_pwm_mutex)                     */
/* ------------------------------------------------------------------ */
static DEFINE_MUTEX(gpio_pwm_mutex);

static unsigned int gpio_pwm_freq_hz    = 1;	/* частота, Гц       */
static unsigned int gpio_pwm_duty_pct   = 50;	/* скважность, %     */
static atomic_t     gpio_pwm_enabled    = ATOMIC_INIT(0);

/* GPIO дескрипторы */
static struct gpio_desc *gpio_out;	/* GPIO60, pin 37 */
static struct gpio_desc *gpio_in;	/* GPIO61, pin 38 */

/* kthread */
static struct task_struct *gpio_pwm_thread;

/* sysfs kobject */
static struct kobject *gpio_pwm_kobj;

/* -----------------------------------------------------------------------  */
/* gpio_pwm_thread_fn — основной цикл генератора                            */
/*                                                                          */
/* Рассчитывает длительности HIGH и LOW фаз в микросекундах,                */
/* переключает GPIO60 через usleep_range() и проверяет kthread_should_stop. */
/* ------------------------------------------------------------------------ */
static int gpio_pwm_thread_fn(void *data)
{
	unsigned int freq, duty;
	unsigned long period_us, high_us, low_us;

	pr_info("gpio_pwm: generator thread started\n");

	while (!kthread_should_stop()) {
		if (!atomic_read(&gpio_pwm_enabled)) {
			gpiod_set_value(gpio_out, 0);
			usleep_range(5000, 6000);	/* 5 мс ожидание */
			continue;
		}

		/* Прочитать параметры под мьютексом */
		mutex_lock(&gpio_pwm_mutex);
		freq = gpio_pwm_freq_hz;
		duty = gpio_pwm_duty_pct;
		mutex_unlock(&gpio_pwm_mutex);

		/* Вычислить длительности фаз */
		period_us = 1000000UL / freq;		/* период в мкс   */
		high_us   = period_us * duty / 100;	/* HIGH фаза      */
		low_us    = period_us - high_us;	/* LOW  фаза      */

		/* Ограничить снизу: минимум 100 мкс на фазу */
		if (high_us < 100)
			high_us = 100;
		if (low_us < 100)
			low_us = 100;

		/* HIGH */
		gpiod_set_value(gpio_out, 1);
		usleep_range(high_us, high_us + high_us / 10 + 1);

		if (kthread_should_stop())
			break;

		/* LOW */
		gpiod_set_value(gpio_out, 0);
		usleep_range(low_us, low_us + low_us / 10 + 1);
	}

	gpiod_set_value(gpio_out, 0);
	pr_info("gpio_pwm: generator thread stopped\n");
	return 0;
}

/* ------------------------------------------------------------------ */
/* sysfs: enable                                                      */
/* ------------------------------------------------------------------ */
static ssize_t gpio_pwm_enable_show(struct kobject *kobj,
			    struct kobj_attribute *attr, char *buf)
{
	return sysfs_emit(buf, "%d\n", atomic_read(&gpio_pwm_enabled));
}

static ssize_t gpio_pwm_enable_store(struct kobject *kobj,
			     struct kobj_attribute *attr,
			     const char *buf, size_t count)
{
	unsigned int val;
	int ret = kstrtouint(buf, 10, &val);

	if (ret)
		return ret;
	if (val > 1)
		return -EINVAL;

	atomic_set(&gpio_pwm_enabled, val);
	pr_info("gpio_pwm: %s\n", val ? "generation started" : "generation stopped");
	return count;
}

/* ------------------------------------------------------------------ */
/* sysfs: frequency_hz                                                */
/* ------------------------------------------------------------------ */
static ssize_t gpio_pwm_frequency_hz_show(struct kobject *kobj,
				  struct kobj_attribute *attr, char *buf)
{
	unsigned int freq;

	mutex_lock(&gpio_pwm_mutex);
	freq = gpio_pwm_freq_hz;
	mutex_unlock(&gpio_pwm_mutex);
	return sysfs_emit(buf, "%u\n", freq);
}

static ssize_t gpio_pwm_frequency_hz_store(struct kobject *kobj,
				   struct kobj_attribute *attr,
				   const char *buf, size_t count)
{
	unsigned int val;
	int ret = kstrtouint(buf, 10, &val);

	if (ret)
		return ret;
	if (val < 1 || val > 500)
		return -EINVAL;

	mutex_lock(&gpio_pwm_mutex);
	gpio_pwm_freq_hz = val;
	mutex_unlock(&gpio_pwm_mutex);

	pr_info("gpio_pwm: frequency set to %u Hz\n", val);
	return count;
}

/* ------------------------------------------------------------------ */
/* sysfs: duty_percent                                                */
/* ------------------------------------------------------------------ */
static ssize_t gpio_pwm_duty_percent_show(struct kobject *kobj,
				  struct kobj_attribute *attr, char *buf)
{
	unsigned int duty;

	mutex_lock(&gpio_pwm_mutex);
	duty = gpio_pwm_duty_pct;
	mutex_unlock(&gpio_pwm_mutex);
	return sysfs_emit(buf, "%u\n", duty);
}

static ssize_t gpio_pwm_duty_percent_store(struct kobject *kobj,
				   struct kobj_attribute *attr,
				   const char *buf, size_t count)
{
	unsigned int val;
	int ret = kstrtouint(buf, 10, &val);

	if (ret)
		return ret;
	if (val < 1 || val > 99)
		return -EINVAL;

	mutex_lock(&gpio_pwm_mutex);
	gpio_pwm_duty_pct = val;
	mutex_unlock(&gpio_pwm_mutex);

	pr_info("gpio_pwm: duty cycle set to %u%%\n", val);
	return count;
}

/* ------------------------------------------------------------------ */
/* sysfs: in_value — чтение GPIO61 (loopback вход)                    */
/* ------------------------------------------------------------------ */
static ssize_t gpio_pwm_in_value_show(struct kobject *kobj,
			      struct kobj_attribute *attr, char *buf)
{
	return sysfs_emit(buf, "%d\n", gpiod_get_value(gpio_in));
}

static ssize_t gpio_pwm_in_value_store(struct kobject *kobj,
			       struct kobj_attribute *attr,
			       const char *buf, size_t count)
{
	return -EPERM;
}

static struct kobj_attribute enable_attr =
	__ATTR(enable,       0664, gpio_pwm_enable_show,       gpio_pwm_enable_store);
static struct kobj_attribute frequency_hz_attr =
	__ATTR(frequency_hz, 0664, gpio_pwm_frequency_hz_show, gpio_pwm_frequency_hz_store);
static struct kobj_attribute duty_percent_attr =
	__ATTR(duty_percent, 0664, gpio_pwm_duty_percent_show, gpio_pwm_duty_percent_store);
static struct kobj_attribute in_value_attr =
	__ATTR(in_value,     0444, gpio_pwm_in_value_show,     gpio_pwm_in_value_store);

static struct attribute *gpio_pwm_attrs[] = {
	&enable_attr.attr,
	&frequency_hz_attr.attr,
	&duty_percent_attr.attr,
	&in_value_attr.attr,
	NULL,
};

static struct attribute_group gpio_pwm_attr_group = {
	.attrs = gpio_pwm_attrs,
};

static int __init gpio_pwm_init(void)
{
	int ret;

	/* Получить GPIO-дескрипторы по глобальным номерам */
	gpio_out = gpio_to_desc(60);	/* GPIO60, пин 37, выход */
	gpio_in  = gpio_to_desc(61);	/* GPIO61, пин 38, вход  */

	if (!gpio_out || !gpio_in) {
		pr_err("gpio_pwm: failed to get GPIO descriptors\n");
		return -ENODEV;
	}

	ret = gpiod_direction_output(gpio_out, 0);
	if (ret) {
		pr_err("gpio_pwm: direction_output GPIO60 failed: %d\n", ret);
		return ret;
	}

	ret = gpiod_direction_input(gpio_in);
	if (ret) {
		pr_err("gpio_pwm: direction_input GPIO61 failed: %d\n", ret);
		gpiod_direction_input(gpio_out);
		return ret;
	}

	/* Создать sysfs-интерфейс */
	gpio_pwm_kobj = kobject_create_and_add("gpio_pwm", kernel_kobj);
	if (!gpio_pwm_kobj)
		return -ENOMEM;

	ret = sysfs_create_group(gpio_pwm_kobj, &gpio_pwm_attr_group);
	if (ret) {
		kobject_put(gpio_pwm_kobj);
		return ret;
	}

	/* Запустить поток генератора */
	gpio_pwm_thread = kthread_run(gpio_pwm_thread_fn, NULL, "gpio_pwm");
	if (IS_ERR(gpio_pwm_thread)) {
		ret = PTR_ERR(gpio_pwm_thread);
		pr_err("gpio_pwm: kthread_run failed: %d\n", ret);
		sysfs_remove_group(gpio_pwm_kobj, &gpio_pwm_attr_group);
		kobject_put(gpio_pwm_kobj);
		return ret;
	}

	pr_info("gpio_pwm: loaded\n");
	pr_info("gpio_pwm: GPIO60 (pin 37) = OUT (generator)\n");
	pr_info("gpio_pwm: GPIO61 (pin 38) = IN  (loopback)\n");
	pr_info("gpio_pwm: sysfs at /sys/kernel/gpio_pwm/\n");
	pr_info("gpio_pwm: default: freq=1 Hz, duty=50%%, disabled\n");
	return 0;
}

static void __exit gpio_pwm_exit(void)
{
	/* Остановить генератор и поток */
	atomic_set(&gpio_pwm_enabled, 0);
	kthread_stop(gpio_pwm_thread);

	/* Опустить выходной пин */
	gpiod_set_value(gpio_out, 0);
	gpiod_direction_input(gpio_out);

	sysfs_remove_group(gpio_pwm_kobj, &gpio_pwm_attr_group);
	kobject_put(gpio_pwm_kobj);
	pr_info("gpio_pwm: unloaded\n");
}

module_init(gpio_pwm_init);
module_exit(gpio_pwm_exit);
```

Файл `Makefile`:

```makefile
obj-m := gpio_pwm.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	$(MAKE) -C $(KDIR) M=$(PWD) modules

clean:
	$(MAKE) -C $(KDIR) M=$(PWD) clean
```

---

## Сборка

```bash
$ make
$ ls -lh gpio_pwm.ko
```

---

## Тестирование из командной строки

```bash
# Убедиться что пины 37 и 38 соединены перемычкой (GPIO60 → GPIO61)
$ sudo gpioinfo | grep -E "line.*6[01]"

$ sudo insmod gpio_pwm.ko
$ dmesg | tail -6
[26824.501087] gpio_pwm: loaded
[26824.501096] gpio_pwm: generator thread started
[26824.501104] gpio_pwm: GPIO60 (pin 37) = OUT (generator)
[26824.501110] gpio_pwm: GPIO61 (pin 38) = IN  (loopback)
[26824.501115] gpio_pwm: sysfs at /sys/kernel/gpio_pwm/
[26824.501120] gpio_pwm: default: freq=1 Hz, duty=50%, disabled

$ sudo ls /sys/kernel/gpio_pwm/
# вывод: duty_percent  enable  frequency_hz  in_value

# Установить параметры
$ echo 2 | sudo tee /sys/kernel/gpio_pwm/frequency_hz
$ echo 75 | sudo tee /sys/kernel/gpio_pwm/duty_percent

# Запустить генерацию
$ echo 1 | sudo tee /sys/kernel/gpio_pwm/enable

# Убедиться что loopback видит сигнал (значение будет меняться ~2 Гц)
$ for i in $(seq 1 8); do
    sudo cat /sys/kernel/gpio_pwm/in_value
    sleep 0.2
  done
# чередование 0 и 1

# Изменить частоту на лету (без перезагрузки модуля!)
$ echo 5 | sudo tee /sys/kernel/gpio_pwm/frequency_hz
$ echo 10 | sudo tee /sys/kernel/gpio_pwm/frequency_hz

# Остановить генерацию
$ echo 0 | sudo tee /sys/kernel/gpio_pwm/enable

$ sudo rmmod gpio_pwm
```

---

## Эксперимент: изменение параметров в реальном времени

Запустить генерацию и поменять частоту в цикле — это наглядно демонстрирует
управление ядерным модулем через sysfs без перезагрузки:

```bash
$ sudo insmod gpio_pwm.ko
$ echo 1 | sudo tee /sys/kernel/gpio_pwm/enable

# Плавно поднять частоту от 1 до 20 Гц
$ for f in 1 2 3 5 8 10 15 20; do
   echo $f | sudo tee /sys/kernel/gpio_pwm/frequency_hz
   echo "freq=$f Hz, duty=$(sudo cat /sys/kernel/gpio_pwm/duty_percent)%"
   sleep 1
 done

# Изменить скважность при фиксированной частоте 10 Гц
$ echo 10 | sudo tee /sys/kernel/gpio_pwm/frequency_hz
$ for d in 10 25 50 75 90; do
   echo $d | sudo tee /sys/kernel/gpio_pwm/duty_percent
   echo "duty=$d%"
   sleep 1
 done
```

> **Следующий шаг:** в module6 написан терминальный осциллограф,
> который читает GPIO61 и рисует сигнал в терминале. Можно запустить
> оба одновременно: в одном окне этот скрипт, в другом — осциллограф.

---

## ftrace: как kthread переключает GPIO

```bash
$ sudo insmod gpio_pwm.ko

$ echo 'gpio_pwm* gpiod_set_value*' | \
    sudo tee /sys/kernel/debug/tracing/set_ftrace_filter

$ echo function | sudo tee /sys/kernel/debug/tracing/current_tracer
$ echo | sudo tee /sys/kernel/debug/tracing/trace
$ echo 1 | sudo tee /sys/kernel/debug/tracing/tracing_on

$ echo 2 | sudo tee /sys/kernel/gpio_pwm/frequency_hz
$ echo 1 | sudo tee /sys/kernel/gpio_pwm/enable
$ sleep 1

$ echo 0 | sudo tee /sys/kernel/debug/tracing/tracing_on
$ sudo cat /sys/kernel/debug/tracing/trace | head -30
```

В выводе будет видно чередование `gpiod_set_value` из потока `gpio_pwm`
с временны́ми метками — промежутки соответствуют периоду сигнала:

```bash
# tracer: function
#
# entries-in-buffer/entries-written: 234/234   #P:4
#
#                                _-----=> irqs-off/BH-disabled
#                               / _----=> need-resched
#                              | / _---=> hardirq/softirq
#                              || / _--=> preempt-depth
#                              ||| / _-=> migrate-disable
#                              |||| /     delay
#           TASK-PID     CPU#  |||||  TIMESTAMP  FUNCTION
#              | |         |   |||||     |         |
        gpio_pwm-7214    [002] ..... 27289.232729: gpiod_set_value <-gpio_pwm_thread_fn
        gpio_pwm-7214    [002] ..... 27289.232738: gpiod_set_value_nocheck <-gpiod_set_value
        gpio_pwm-7214    [002] ..... 27289.243842: gpiod_set_value <-gpio_pwm_thread_fn
        gpio_pwm-7214    [002] ..... 27289.243846: gpiod_set_value_nocheck <-gpiod_set_value
        gpio_pwm-7214    [002] ..... 27289.343006: gpiod_set_value <-gpio_pwm_thread_fn
        gpio_pwm-7214    [002] ..... 27289.343011: gpiod_set_value_nocheck <-gpiod_set_value
        gpio_pwm-7214    [002] ..... 27289.354123: gpiod_set_value <-gpio_pwm_thread_fn
        gpio_pwm-7214    [002] ..... 27289.354127: gpiod_set_value_nocheck <-gpiod_set_value
        gpio_pwm-7214    [002] ..... 27289.453284: gpiod_set_value <-gpio_pwm_thread_fn
        gpio_pwm-7214    [002] ..... 27289.453289: gpiod_set_value_nocheck <-gpiod_set_value
        gpio_pwm-7214    [002] ..... 27289.464401: gpiod_set_value <-gpio_pwm_thread_fn
        gpio_pwm-7214    [002] ..... 27289.464406: gpiod_set_value_nocheck <-gpiod_set_value
        gpio_pwm-7214    [002] ..... 27289.563495: gpiod_set_value <-gpio_pwm_thread_fn
        gpio_pwm-7214    [002] ..... 27289.563500: gpiod_set_value_nocheck <-gpiod_set_value
        gpio_pwm-7214    [002] ..... 27289.574617: gpiod_set_value <-gpio_pwm_thread_fn
        gpio_pwm-7214    [002] ..... 27289.574622: gpiod_set_value_nocheck <-gpiod_set_value
        gpio_pwm-7214    [002] ..... 27289.673778: gpiod_set_value <-gpio_pwm_thread_fn
        gpio_pwm-7214    [002] ..... 27289.673783: gpiod_set_value_nocheck <-gpiod_set_value
```

```bash
# Очистить
$ echo 0 | sudo tee /sys/kernel/gpio_pwm/enable
$ echo | sudo tee /sys/kernel/debug/tracing/set_ftrace_filter
$ echo nop | sudo tee /sys/kernel/debug/tracing/current_tracer
$ sudo rmmod gpio_pwm
```

---

← [Назад](module4-driver-gpio.md) · [На главную](../INDEX.md) · [Следующий →](module6-terminal-scope.md)
