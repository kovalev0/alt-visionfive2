# module5 · Модуль ядра: loopback OUT→IN, gpiod API

← [Назад](module3-sysfs-gpio.md) · [На главную](../INDEX.md) · [Следующий →](module5-pwm-generator.md)

> **Kernel docs:** [kernel.org/doc/html/latest/driver-api/gpio/consumer.html](https://www.kernel.org/doc/html/latest/driver-api/gpio/consumer.html)

> **Kernel source:** `include/linux/gpio/consumer.h`
> [elixir.bootlin.com/linux/v6.12/source/include/linux/gpio/consumer.h](https://elixir.bootlin.com/linux/v6.12/source/include/linux/gpio/consumer.h)

---

## Подготовка: соединение пинов

Перед загрузкой модуля нужно физически соединить два пина [перемычкой](README.md#что-понадобится)

```
          GPIO40 разъём VisionFive2
   (вид сверху, кнопка вкл. питания снизу)

  ┌─────────────────────────────────────┐
  │  ...  33 ○           ○ 34  ...      │
  │  ...  35 ○           ○ 36  ...      │
  │       37 ● GPIO60  GPIO61 ● 38      │
  │  ...  39 ○           ○ 40  ...      │
  └─────────────────────────────────────┘

                     ████
            (кнопка вкл. питания)


        37 ● GPIO60──GPIO61 ● 38
           │  OUT──────IN   │
           │                │
           └──── dupont ────┘

  ● — используемый пин
  ○ — неиспользуемый
  ── — провод dupont
```

> Пин 39 (GND) не соединяется — он нужен только если подключается
> внешнее устройство со своим питанием. В нашем случае оба пина
> питаются от одного SoC и GND общий.

```bash
# Убедиться что обе линии свободны
$ sudo gpioinfo | grep -E "line.*6[01]"
# ожидаемо: unnamed  input/output  (без consumer)
```

---

## Код модуля

Создать каталог:

```bash
$ mkdir -p ~/labs/lab05/gpio_loopback
$ cd ~/labs/lab05/gpio_loopback
```

Файл `gpio_loopback.c`:

```c
// gpio_loopback.c — kernel module: GPIO loopback OUT→IN test
//
// GPIO60 (pin 35): configured as output
// GPIO61 (pin 37): configured as input
//
// sysfs interface: /sys/kernel/gpio_loopback/
//   out_value  — write 0/1 to drive GPIO60
//   in_value   — read current value of GPIO61
//   run_test   — write 1 to run automatic loopback test

#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/init.h>
#include <linux/kobject.h>
#include <linux/sysfs.h>
#include <linux/gpio/consumer.h>
#include <linux/platform_device.h>
#include <linux/delay.h>

MODULE_LICENSE("GPL");
MODULE_AUTHOR("lab05");
MODULE_DESCRIPTION("GPIO loopback test module: GPIO60 OUT -> GPIO61 IN");

/* GPIO descriptors */
static struct gpio_desc *gpio_out;	/* GPIO60, pin 35 */
static struct gpio_desc *gpio_in;	/* GPIO61, pin 37 */

static struct kobject *lb_kobj;

/* ------------------------------------------------------------------ */
/* gpio_loopback_set_out — установить уровень на выходном пине        */
/* ------------------------------------------------------------------ */
static noinline void gpio_loopback_set_out(int val)
{
	gpiod_set_value(gpio_out, val);
	pr_info("gpio_loopback: OUT (GPIO60) = %d\n", val);
}

/* ------------------------------------------------------------------ */
/* gpio_loopback_read_in — прочитать уровень со входного пина         */
/* ------------------------------------------------------------------ */
static noinline int gpio_loopback_read_in(void)
{
	int val = gpiod_get_value(gpio_in);

	pr_info("gpio_loopback: IN  (GPIO61) = %d\n", val);
	return val;
}

/* ------------------------------------------------------------------ */
/* gpio_loopback_run_test — автоматический тест: 0 и 1                */
/* ------------------------------------------------------------------ */
static noinline void gpio_loopback_run_test(void)
{
	int out_val, in_val;
	int pass = 0, fail = 0;

	pr_info("gpio_loopback: starting loopback test\n");

	for (out_val = 0; out_val <= 1; out_val++) {
		gpio_loopback_set_out(out_val);
		udelay(10);	/* дать сигналу стабилизироваться */

		in_val = gpio_loopback_read_in();

		if (in_val == out_val) {
			pr_info("gpio_loopback: OUT=%d IN=%d PASS\n",
				out_val, in_val);
			pass++;
		} else {
			pr_err("gpio_loopback: OUT=%d IN=%d FAIL (check wiring!)\n",
			       out_val, in_val);
			fail++;
		}
	}

	pr_info("gpio_loopback: test done — %d passed, %d failed\n",
		pass, fail);
}

/* --- sysfs: out_value --- */
static ssize_t gpio_loopback_out_value_show(struct kobject *kobj,
			       struct kobj_attribute *attr, char *buf)
{
	return sysfs_emit(buf, "%d\n", gpiod_get_value(gpio_out));
}

static ssize_t gpio_loopback_out_value_store(struct kobject *kobj,
				struct kobj_attribute *attr,
				const char *buf, size_t count)
{
	unsigned int val;
	int ret = kstrtouint(buf, 10, &val);

	if (ret)
		return ret;
	if (val > 1)
		return -EINVAL;
	gpio_loopback_set_out(val);
	return count;
}

/* --- sysfs: in_value --- */
static ssize_t gpio_loopback_in_value_show(struct kobject *kobj,
			      struct kobj_attribute *attr, char *buf)
{
	return sysfs_emit(buf, "%d\n", gpio_loopback_read_in());
}

static ssize_t gpio_loopback_in_value_store(struct kobject *kobj,
			       struct kobj_attribute *attr,
			       const char *buf, size_t count)
{
	return -EPERM;	/* вход — только чтение */
}

/* --- sysfs: run_test --- */
static ssize_t gpio_loopback_run_test_show(struct kobject *kobj,
			      struct kobj_attribute *attr, char *buf)
{
	return sysfs_emit(buf, "write 1 to run loopback test\n");
}

static ssize_t gpio_loopback_run_test_store(struct kobject *kobj,
			       struct kobj_attribute *attr,
			       const char *buf, size_t count)
{
	unsigned int val;
	int ret = kstrtouint(buf, 10, &val);

	if (ret)
		return ret;
	if (val == 1)
		gpio_loopback_run_test();
	return count;
}

static struct kobj_attribute out_value_attr =
	__ATTR(out_value, 0664, gpio_loopback_out_value_show, gpio_loopback_out_value_store);
static struct kobj_attribute in_value_attr =
	__ATTR(in_value,  0664, gpio_loopback_in_value_show,  gpio_loopback_in_value_store);
static struct kobj_attribute run_test_attr =
	__ATTR(run_test,  0664, gpio_loopback_run_test_show,  gpio_loopback_run_test_store);

static struct attribute *lb_attrs[] = {
	&out_value_attr.attr,
	&in_value_attr.attr,
	&run_test_attr.attr,
	NULL,
};

static struct attribute_group lb_attr_group = {
	.attrs = lb_attrs,
};

static int __init gpio_loopback_init(void)
{
	int ret;

	/*
	 * Получить GPIO по глобальному номеру.
	 * gpio_to_desc() преобразует глобальный номер в дескриптор.
	 * GPIO60: gpiochip0 offset=60, физический пин 35
	 * GPIO61: gpiochip0 offset=61, физический пин 37
	 */
	gpio_out = gpio_to_desc(60);
	gpio_in  = gpio_to_desc(61);

	if (!gpio_out || !gpio_in) {
		pr_err("gpio_loopback: failed to get GPIO descriptors\n");
		return -ENODEV;
	}

	/* Настроить направления */
	ret = gpiod_direction_output(gpio_out, 0);
	if (ret) {
		pr_err("gpio_loopback: direction_output failed: %d\n", ret);
		return ret;
	}

	ret = gpiod_direction_input(gpio_in);
	if (ret) {
		pr_err("gpio_loopback: direction_input failed: %d\n", ret);
		gpiod_direction_input(gpio_out);	/* вернуть в исходное */
		return ret;
	}

	/* Создать sysfs-интерфейс */
	lb_kobj = kobject_create_and_add("gpio_loopback", kernel_kobj);
	if (!lb_kobj)
		return -ENOMEM;

	ret = sysfs_create_group(lb_kobj, &lb_attr_group);
	if (ret) {
		kobject_put(lb_kobj);
		return ret;
	}

	pr_info("gpio_loopback: loaded\n");
	pr_info("gpio_loopback: GPIO60 (pin 35) = OUT, GPIO61 (pin 37) = IN\n");
	pr_info("gpio_loopback: connect pin 35 to pin 37 with a wire\n");
	pr_info("gpio_loopback: sysfs at /sys/kernel/gpio_loopback/\n");
	return 0;
}

static void __exit gpio_loopback_exit(void)
{
	/* Вернуть пины в режим входа */
	gpiod_direction_input(gpio_out);

	sysfs_remove_group(lb_kobj, &lb_attr_group);
	kobject_put(lb_kobj);
	pr_info("gpio_loopback: unloaded\n");
}

module_init(gpio_loopback_init);
module_exit(gpio_loopback_exit);
```

Файл `Makefile`:

```makefile
obj-m := gpio_loopback.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	$(MAKE) -C $(KDIR) M=$(PWD) modules

clean:
	$(MAKE) -C $(KDIR) M=$(PWD) clean
```

---

## Сборка и тестирование

```bash
$ make

# Убедиться что пины 35 и 37 соединены перемычкой
# Загрузить модуль
$ sudo insmod gpio_loopback.ko

$ dmesg | tail -5
[24291.258183] gpio_loopback: loading out-of-tree module taints kernel.
[24291.262456] gpio_loopback: loaded
[24291.262553] gpio_loopback: GPIO60 (pin 35) = OUT, GPIO61 (pin 37) = IN
[24291.262646] gpio_loopback: connect pin 35 to pin 37 with a wire
[24291.262739] gpio_loopback: sysfs at /sys/kernel/gpio_loopback/

# Интерфейс в sysfs
$ sudo ls /sys/kernel/gpio_loopback/
# вывод: in_value  out_value  run_test

# Ручное управление
$ echo 1 | sudo tee /sys/kernel/gpio_loopback/out_value
$ sudo cat /sys/kernel/gpio_loopback/in_value
# вывод: 1  (провод передаёт уровень)

$ echo 0 | sudo tee /sys/kernel/gpio_loopback/out_value
$ sudo cat /sys/kernel/gpio_loopback/in_value
# вывод: 0

# Автоматический тест
$ echo 1 | sudo tee /sys/kernel/gpio_loopback/run_test
$ dmesg | grep "gpio_loopback" | tail -8
[24633.887792] gpio_loopback: starting loopback test
[24633.887899] gpio_loopback: OUT (GPIO60) = 0
[24633.888011] gpio_loopback: IN  (GPIO61) = 0
[24633.888105] gpio_loopback: OUT=0 IN=0 PASS
[24633.888209] gpio_loopback: OUT (GPIO60) = 1
[24633.888317] gpio_loopback: IN  (GPIO61) = 1
[24633.888411] gpio_loopback: OUT=1 IN=1 PASS
[24633.888656] gpio_loopback: test done — 2 passed, 0 failed

$ sudo rmmod gpio_loopback
```

Если тест показывает FAIL — скорее всего провод не подключён или подключён
не к тем пинам.

---

## ftrace: трассировка пути записи в GPIO

```bash
$ sudo insmod gpio_loopback.ko

# Трассировать функции gpio от gpiod_set_value до записи в железо
$ echo | sudo tee /sys/kernel/debug/tracing/set_ftrace_filter

$ echo 'gpiod_set_value* gpio_loopback_* starfive*' | \
    sudo tee /sys/kernel/debug/tracing/set_graph_function

$ echo function_graph | sudo tee /sys/kernel/debug/tracing/current_tracer
$ echo 5 | sudo tee /sys/kernel/debug/tracing/max_graph_depth
$ echo | sudo tee /sys/kernel/debug/tracing/trace
$ echo 1 | sudo tee /sys/kernel/debug/tracing/tracing_on

$ echo 1 | sudo tee /sys/kernel/gpio_loopback/out_value

$ echo 0 | sudo tee /sys/kernel/debug/tracing/tracing_on
$ sudo cat /sys/kernel/debug/tracing/trace
```

В выводе будет видна цепочка: `gpio_loopback_set_out()` →
`gpiod_set_value()` → `gpiod_set_value_nocheck()` →
`gpiod_set_raw_value_commit()` → запись в регистр DOUT:

```bash
# tracer: function_graph
#
# CPU  DURATION                  FUNCTION CALLS
# |     |   |                     |   |   |   |

 2)               |  gpio_loopback_out_value_store [gpio_loopback]() {
 2)               |    gpio_loopback_set_out [gpio_loopback]() {
 2)               |      gpiod_set_value() {
 2)               |        gpiod_set_value_nocheck() {
 2)   9.250 us    |          gpiod_set_raw_value_commit();
 2) + 14.000 us   |        }
 2) + 18.750 us   |      }
 2)               |      _printk() {
 2)               |        vprintk() {
 2)   2.500 us    |          is_printk_legacy_deferred();
 2) + 93.000 us   |          vprintk_default();
 2) ! 101.500 us  |        }
 2) ! 105.500 us  |      }
 2) ! 131.250 us  |    }
 2) ! 141.500 us  |  }

```

```bash
# Очистить после работы
$ echo | sudo tee /sys/kernel/debug/tracing/set_ftrace_filter
$ echo nop | sudo tee /sys/kernel/debug/tracing/current_tracer
$ sudo rmmod gpio_loopback
```

---

← [Назад](module3-sysfs-gpio.md) · [На главную](../INDEX.md) · [Следующий →](module5-pwm-generator.md)
