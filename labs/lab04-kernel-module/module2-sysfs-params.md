# module2 · sysfs-атрибуты и параметры модуля

← [Назад](module1-lkm-basics.md) · [На главную](../INDEX.md) · [Следующий →](module3-counter-module.md)

> **Kernel docs:** [kernel.org/doc/html/latest/filesystems/sysfs.html](https://www.kernel.org/doc/html/latest/filesystems/sysfs.html)

> **Kernel source:** `include/linux/sysfs.h`
> [elixir.bootlin.com/linux/v6.12/source/include/linux/sysfs.h](https://elixir.bootlin.com/linux/v6.12/source/include/linux/sysfs.h)

---

## Что такое sysfs

sysfs — виртуальная файловая система ядра, смонтированная в `/sys`.
Она экспортирует внутренние объекты ядра (kobject) в виде каталогов и
файлов. Запись в файл sysfs == вызов функции в ядре. Чтение == получение
текущего состояния.

```
  userspace                    kernel
  ─────────                    ──────
  echo 1 > /sys/.../enable
       │
       │  write() syscall
       ▼
  sysfs VFS layer
       │
       │  store() callback
       ▼
  функция в модуле ядра
       │
       ▼
  изменение состояния / управление железом
```

---

## Параметры модуля: module_param

Параметры модуля позволяют передавать значения при загрузке через `insmod`
и изменять их из userspace через `/sys/module/<name>/parameters/`.

```c
// params.c — module with parameters

#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/init.h>

MODULE_LICENSE("GPL");

/* Объявление параметров */
static int   count = 0;
static char *name  = "default";
static bool  verbose = false;

module_param(count,   int,  0644);  /* 0644 = rw-r--r-- в sysfs */
module_param(name,    charp, 0444); /* 0444 = r--r--r-- (только чтение) */
module_param(verbose, bool, 0644);

MODULE_PARM_DESC(count,   "Initial counter value");
MODULE_PARM_DESC(name,    "Module instance name");
MODULE_PARM_DESC(verbose, "Enable verbose output");

static int __init params_init(void)
{
	pr_info("params: loaded, name=%s count=%d verbose=%d\n",
		 name, count, verbose);
	return 0;
}

static void __exit params_exit(void)
{
	pr_info("params: unloaded\n");
}

module_init(params_init);
module_exit(params_exit);
```

```bash
# Загрузка с параметрами
$ sudo insmod params.ko count=42 name="test" verbose=1

# Проверить значения через sysfs
$ cat /sys/module/params/parameters/count
42
$ cat /sys/module/params/parameters/verbose
Y

# Изменить значение в runtime (если разрешено режимом 0644)
$ echo 100 | sudo tee /sys/module/params/parameters/count
$ cat /sys/module/params/parameters/count
100

$ dmesg | tail -1
[ 1022.962893] params: loaded, name=test count=42 verbose=1
```

Третий аргумент `module_param` — права доступа к файлу в sysfs:
- `0` — параметр не экспортируется в sysfs (только `insmod`)
- `0444` — только чтение
- `0644` — чтение и запись

---

## sysfs-атрибуты: DEVICE_ATTR и kobject

Для создания произвольных файлов в sysfs нужен `kobject` — базовый объект
ядра. Модуль может создать свой kobject и прикрепить к нему атрибуты.

```c
// sysfs_demo.c — custom sysfs attributes via kobject

#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/init.h>
#include <linux/kobject.h>
#include <linux/sysfs.h>
#include <linux/string.h>

MODULE_LICENSE("GPL");

static struct kobject *demo_kobj;
static int demo_value = 0;

/* --- show: читается при cat /sys/kernel/demo/value --- */
static ssize_t value_show(struct kobject *kobj,
                           struct kobj_attribute *attr,
                           char *buf)
{
	return sysfs_emit(buf, "%d\n", demo_value);
}

/* --- store: вызывается при echo N > /sys/kernel/demo/value --- */
static ssize_t value_store(struct kobject *kobj,
                            struct kobj_attribute *attr,
                            const char *buf, size_t count)
{
	int ret;
	int val;

	ret = kstrtoint(buf, 10, &val);
	if (ret)
		return ret;

	demo_value = val;
	pr_info("sysfs_demo: value set to %d\n", demo_value);
	return count;
}

/* Объявление атрибута: имя, режим, show, store */
static struct kobj_attribute value_attr =
	__ATTR(value, 0664, value_show, value_store);

/* Список атрибутов (NULL-terminated) */
static struct attribute *demo_attrs[] = {
	&value_attr.attr,
	NULL,
};

/* Группа атрибутов */
static struct attribute_group demo_attr_group = {
	.attrs = demo_attrs,
};

static int __init sysfs_demo_init(void)
{
	int ret;

	/* Создать /sys/kernel/demo/ */
	demo_kobj = kobject_create_and_add("demo", kernel_kobj);
	if (!demo_kobj)
		return -ENOMEM;

	/* Создать файлы атрибутов в /sys/kernel/demo/ */
	ret = sysfs_create_group(demo_kobj, &demo_attr_group);
	if (ret) {
		kobject_put(demo_kobj);
		return ret;
	}

	pr_info("sysfs_demo: loaded, /sys/kernel/demo/value created\n");
	return 0;
}

static void __exit sysfs_demo_exit(void)
{
	sysfs_remove_group(demo_kobj, &demo_attr_group);
	kobject_put(demo_kobj);
	pr_info("sysfs_demo: unloaded\n");
}

module_init(sysfs_demo_init);
module_exit(sysfs_demo_exit);
```

```bash
$ sudo insmod sysfs_demo.ko

# Появился каталог в sysfs
$ ls /sys/kernel/demo/
value

# Прочитать значение
$ cat /sys/kernel/demo/value
0

# Записать значение
$ echo 42 | sudo tee /sys/kernel/demo/value

# Прочитать снова
$ cat /sys/kernel/demo/value
42

# Убедиться что модуль обработал запись
$ dmesg | tail -2
[ 1431.913533] sysfs_demo: loaded, /sys/kernel/demo/value created
[ 1494.124525] sysfs_demo: value set to 42

$ sudo rmmod sysfs_demo
```

---

## Важные детали

### sysfs_emit вместо sprintf

Для современных ядер в `show`-функциях следует использовать `sysfs_emit()`
вместо `sprintf()`. `sysfs_emit()` знает размер буфера sysfs (PAGE_SIZE)
и не допускает переполнения.

```c
/* Правильно: */
return sysfs_emit(buf, "%d\n", value);

/* Устаревший способ: */
return sprintf(buf, "%d\n", value);
```

### kstrtoint и родственные функции

Для парсинга входных данных в `store` нельзя использовать `atoi()` или
`sscanf()` из libc — их нет в ядре. Вместо этого:

```c
kstrtoint(buf, 10, &val);   /* строка → int */
kstrtou32(buf, 10, &val);   /* строка → u32 */
kstrtobool(buf, &val);      /* строка → bool ("1"/"0"/"y"/"n") */
```

Все возвращают 0 при успехе или отрицательный код ошибки.

### Завершающий `\n`

В `show` всегда нужно добавлять `\n` в конце строки — иначе `cat`
выведет значение без перевода строки и следующее приглашение командной
строки окажется на той же строке.

---

← [Назад](module1-lkm-basics.md) · [На главную](../INDEX.md) · [Следующий →](module3-counter-module.md)
