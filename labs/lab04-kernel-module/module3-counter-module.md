# module3 · Практика: модуль-счётчик с управлением из sysfs

← [Назад](module2-sysfs-params.md) · [На главную](../INDEX.md) · [Следующий →](module4-ftrace.md)

---

## Что будет реализовано

Модуль `my_counter` с тремя независимыми функциями, каждая управляется из
sysfs. Все внутренние функции намеренно вынесены в отдельные символы и
помечены `noinline` — в следующем модуле они будут трассироваться через
ftrace по отдельности.

Единый префикс `my_counter_` во всех именах решает две задачи сразу:
исключает конфликты с именами ядра (например, `reset_store`, `value_show`
встречаются во многих драйверах) и позволяет задать точный фильтр
`my_counter_*` в ftrace без лишнего шума.

```
  /sys/kernel/my_counter/
  ├── value        — текущее значение счётчика (чтение/запись)
  ├── step         — шаг инкремента (параметр модуля)
  ├── increment    — записать 1 чтобы увеличить счётчик на step
  ├── reset        — записать 1 чтобы обнулить счётчик
  └── trigger      — триггер: булево состояние
```

---

## Код модуля

Создать каталог:

```bash
$ mkdir -p ~/labs/lab04/my_counter
$ cd ~/labs/lab04/my_counter
```

Файл `my_counter.c`:

```c
// my_counter.c — counter module with sysfs interface and ftrace-friendly design
//
// Все имена функций начинаются с my_counter_ чтобы:
//   1. Не конфликтовать с функциями ядра при фильтрации в ftrace
//   2. Позволить точный фильтр 'my_counter_*' в set_ftrace_filter
//
// Функции my_counter_do_* помечены noinline чтобы компилятор не мог их
// заинлайнить при -O2 и они появились как отдельные символы в ftrace.
//
// Provides:
//   /sys/kernel/my_counter/value      - read/write counter value
//   /sys/kernel/my_counter/increment  - write 1 to increment by step
//   /sys/kernel/my_counter/reset      - write 1 to reset counter to 0
//   /sys/kernel/my_counter/trigger    - read/write boolean trigger state
//
// Module parameter:
//   step=N  - increment step (default: 1)

#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/init.h>
#include <linux/kobject.h>
#include <linux/sysfs.h>
#include <linux/string.h>

MODULE_LICENSE("GPL");
MODULE_AUTHOR("lab04");
MODULE_DESCRIPTION("Counter module with sysfs control and ftrace instrumentation");

/* Module parameter */
static int step = 1;
module_param(step, int, 0644);
MODULE_PARM_DESC(step, "Increment step (default: 1)");

/* Internal state */
static struct kobject *my_counter_kobj;
static long  my_counter_value   = 0;
static bool  my_counter_trigger = false;

/* ------------------------------------------------------------------ */
/* my_counter_do_increment — увеличить счётчик на step                */
/*                                                                    */
/* noinline: запрещает компилятору встроить тело функции в вызывающий */
/* код при -O2. Без этого атрибута GCC может не создать отдельный     */
/* символ, и функция не появится в ftrace available_filter_functions. */
/* ------------------------------------------------------------------ */
static noinline void my_counter_do_increment(void)
{
	my_counter_value += step;
	pr_info("my_counter: incremented to %ld (step=%d)\n",
		my_counter_value, step);
}

/* ------------------------------------------------------------------ */
/* my_counter_do_reset — обнулить счётчик                             */
/* ------------------------------------------------------------------ */
static noinline void my_counter_do_reset(void)
{
	my_counter_value = 0;
	pr_info("my_counter: reset to 0\n");
}

/* ------------------------------------------------------------------ */
/* my_counter_do_trigger — установить состояние триггера              */
/* ------------------------------------------------------------------ */
static noinline void my_counter_do_trigger(bool new_state)
{
	my_counter_trigger = new_state;
	pr_info("my_counter: trigger set to %d\n", my_counter_trigger);
}

/* --- value --- */
static ssize_t my_counter_value_show(struct kobject *kobj,
				     struct kobj_attribute *attr, char *buf)
{
	return sysfs_emit(buf, "%ld\n", my_counter_value);
}

static ssize_t my_counter_value_store(struct kobject *kobj,
				      struct kobj_attribute *attr,
				      const char *buf, size_t count)
{
	int ret = kstrtol(buf, 10, &my_counter_value);
	if (ret)
		return ret;
	pr_info("my_counter: value set directly to %ld\n", my_counter_value);
	return count;
}

/* --- increment --- */
static ssize_t my_counter_increment_show(struct kobject *kobj,
					 struct kobj_attribute *attr, char *buf)
{
	return sysfs_emit(buf, "write 1 to increment by %d\n", step);
}

static ssize_t my_counter_increment_store(struct kobject *kobj,
					  struct kobj_attribute *attr,
					  const char *buf, size_t count)
{
	unsigned int val;
	int ret = kstrtouint(buf, 10, &val);
	if (ret)
		return ret;
	if (val == 1)
		my_counter_do_increment();
	return count;
}

/* --- reset --- */
static ssize_t my_counter_reset_show(struct kobject *kobj,
				     struct kobj_attribute *attr, char *buf)
{
	return sysfs_emit(buf, "write 1 to reset counter\n");
}

static ssize_t my_counter_reset_store(struct kobject *kobj,
				      struct kobj_attribute *attr,
				      const char *buf, size_t count)
{
	unsigned int val;
	int ret = kstrtouint(buf, 10, &val);
	if (ret)
		return ret;
	if (val == 1)
		my_counter_do_reset();
	return count;
}

/* --- trigger --- */
static ssize_t my_counter_trigger_show(struct kobject *kobj,
				       struct kobj_attribute *attr, char *buf)
{
	return sysfs_emit(buf, "%d\n", my_counter_trigger);
}

static ssize_t my_counter_trigger_store(struct kobject *kobj,
					struct kobj_attribute *attr,
					const char *buf, size_t count)
{
	bool val;
	int ret = kstrtobool(buf, &val);
	if (ret)
		return ret;
	my_counter_do_trigger(val);
	return count;
}

/* Объявление атрибутов */
static struct kobj_attribute my_counter_value_attr =
	__ATTR(value,     0664, my_counter_value_show,     my_counter_value_store);
static struct kobj_attribute my_counter_increment_attr =
	__ATTR(increment, 0664, my_counter_increment_show, my_counter_increment_store);
static struct kobj_attribute my_counter_reset_attr =
	__ATTR(reset,     0664, my_counter_reset_show,     my_counter_reset_store);
static struct kobj_attribute my_counter_trigger_attr =
	__ATTR(trigger,   0664, my_counter_trigger_show,   my_counter_trigger_store);

static struct attribute *my_counter_attrs[] = {
	&my_counter_value_attr.attr,
	&my_counter_increment_attr.attr,
	&my_counter_reset_attr.attr,
	&my_counter_trigger_attr.attr,
	NULL,
};

static struct attribute_group my_counter_attr_group = {
	.attrs = my_counter_attrs,
};

static int __init my_counter_init(void)
{
	int ret;

	my_counter_kobj = kobject_create_and_add("my_counter", kernel_kobj);
	if (!my_counter_kobj)
		return -ENOMEM;

	ret = sysfs_create_group(my_counter_kobj, &my_counter_attr_group);
	if (ret) {
		kobject_put(my_counter_kobj);
		return ret;
	}

	pr_info("my_counter: loaded, step=%d\n", step);
	pr_info("my_counter: sysfs interface at /sys/kernel/my_counter/\n");
	return 0;
}

static void __exit my_counter_exit(void)
{
	sysfs_remove_group(my_counter_kobj, &my_counter_attr_group);
	kobject_put(my_counter_kobj);
	pr_info("my_counter: unloaded, final value=%ld\n", my_counter_value);
}

module_init(my_counter_init);
module_exit(my_counter_exit);
```

Файл `Makefile`:

```makefile
obj-m := my_counter.o

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
$ ls -la my_counter.ko
```

Проверить что `my_counter_do_*` — отдельные символы (не заинлайнены):

```bash
$ objdump -d my_counter.ko | grep -E "^[0-9a-f]+ <my_counter_do"
0000000000000000 <my_counter_do_trigger>:
0000000000000034 <my_counter_do_reset>:
0000000000000064 <my_counter_do_increment>:
```

Если символы есть — `noinline` сработал и функции будут видны в ftrace.

---

## Полный сценарий работы

> Для удобства параллельного наблюдения запустите в другом терминале
> `dmesg -w`

```bash
# Загрузить с шагом 5
$ sudo insmod my_counter.ko step=5

# Убедиться что sysfs-интерфейс создан
$ ls /sys/kernel/my_counter/
# вывод: increment  reset  trigger  value

# Прочитать начальное значение
$ cat /sys/kernel/my_counter/value
# вывод: 0

# Увеличить счётчик
$ echo 1 | sudo tee /sys/kernel/my_counter/increment
$ cat /sys/kernel/my_counter/value
# вывод: 5

$ echo 1 | sudo tee /sys/kernel/my_counter/increment
$ echo 1 | sudo tee /sys/kernel/my_counter/increment
$ cat /sys/kernel/my_counter/value
# вывод: 15

# Записать произвольное значение напрямую
$ echo 100 | sudo tee /sys/kernel/my_counter/value
$ cat /sys/kernel/my_counter/value
# вывод: 100

# Сбросить в 0
$ echo 1 | sudo tee /sys/kernel/my_counter/reset
$ cat /sys/kernel/my_counter/value
# вывод: 0

# Работа с триггером
$ cat /sys/kernel/my_counter/trigger
# вывод: 0

$ echo 1 | sudo tee /sys/kernel/my_counter/trigger
$ cat /sys/kernel/my_counter/trigger
# вывод: 1

$ echo 0 | sudo tee /sys/kernel/my_counter/trigger
$ cat /sys/kernel/my_counter/trigger
# вывод: 0

# Посмотреть всё что написал модуль в dmesg
$ dmesg | grep "my_counter:"
[12835.550535] my_counter: loaded, step=5
[12835.550632] my_counter: sysfs interface at /sys/kernel/my_counter/
[12865.212234] my_counter: incremented to 5 (step=5)
[12874.028816] my_counter: incremented to 10 (step=5)
[12878.957255] my_counter: incremented to 15 (step=5)
[12888.535778] my_counter: value set directly to 100
[12896.219117] my_counter: reset to 0
[12908.887189] my_counter: trigger set to 1
[12917.808493] my_counter: trigger set to 0

# Изменить step в runtime через параметр модуля
$ echo 10 | sudo tee /sys/module/my_counter/parameters/step
$ echo 1 | sudo tee /sys/kernel/my_counter/increment
$ cat /sys/kernel/my_counter/value
# вывод: 10 (шаг изменился на 10)

# Выгрузить
$ sudo rmmod my_counter
$ dmesg | tail -2
[12957.877184] my_counter: incremented to 10 (step=10)
[12977.457966] my_counter: unloaded, final value=10
```

---

## Проверка через скрипт

```bash
#!/bin/bash
# test_my_counter.sh — automated test for my_counter module

set -e

echo "[*] loading module with step=3"
sudo insmod my_counter.ko step=3

echo "[*] initial value:"
cat /sys/kernel/my_counter/value

echo "[*] increment 3 times"
for i in 1 2 3; do
    echo 1 | sudo tee /sys/kernel/my_counter/increment > /dev/null
done
VAL=$(cat /sys/kernel/my_counter/value)
echo "[*] value after 3 increments: $VAL"
[ "$VAL" = "9" ] && echo "[OK] expected 9" || echo "[FAIL] expected 9, got $VAL"

echo "[*] reset"
echo 1 | sudo tee /sys/kernel/my_counter/reset > /dev/null
VAL=$(cat /sys/kernel/my_counter/value)
[ "$VAL" = "0" ] && echo "[OK] reset to 0" || echo "[FAIL] expected 0, got $VAL"

echo "[*] trigger test"
echo 1 | sudo tee /sys/kernel/my_counter/trigger > /dev/null
VAL=$(cat /sys/kernel/my_counter/trigger)
[ "$VAL" = "1" ] && echo "[OK] trigger=1" || echo "[FAIL]"

echo "[*] unloading"
sudo rmmod my_counter
echo "[*] done"
```

```bash
$ chmod +x test_my_counter.sh
$ sudo ./test_my_counter.sh
[*] loading module with step=3
[*] initial value:
0
[*] increment 3 times
[*] value after 3 increments: 9
[OK] expected 9
[*] reset
[OK] reset to 0
[*] trigger test
[OK] trigger=1
[*] unloading
[*] done
```

---

← [Назад](module2-sysfs-params.md) · [На главную](../INDEX.md) · [Следующий →](module4-ftrace.md)
