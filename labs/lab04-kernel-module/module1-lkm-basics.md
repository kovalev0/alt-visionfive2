# module1 · Структура модуля: init/exit, Makefile, insmod/rmmod

← [Назад (README)](README.md) · [На главную](../INDEX.md) · [Следующий →](module2-sysfs-params.md)

> **Kernel docs:** [kernel.org/doc/html/latest/kbuild/modules.html](https://www.kernel.org/doc/html/latest/kbuild/modules.html)

> **Kernel source (module API):** `include/linux/module.h`
> [elixir.bootlin.com/linux/v6.12/source/include/linux/module.h](https://elixir.bootlin.com/linux/v6.12/source/include/linux/module.h)

---

## Чем модуль ядра отличается от userspace программы

Userspace программа запускается в U-mode, имеет собственное виртуальное
адресное пространство, и при ошибке падает сама — ядро изолирует её от
остальной системы.

Модуль ядра работает в S-mode, в адресном пространстве самого ядра. Ошибка
(разыменование нулевого указателя, выход за границы массива) приводит к
kernel panic или oops и может повесить всю систему. Это нужно держать в
голове при каждом написании кода.

Ещё одно отличие: в модуле нет `main()`, нет стандартной библиотеки C
(libc), нет `printf()`. Вместо этого — `module_init()`, `module_exit()`
и `printk()`.

---

## Минимальный модуль

Создать каталог и файлы:

```bash
$ mkdir -p ~/labs/lab04/hello
$ cd ~/labs/lab04/hello
```

Файл `hello.c`:

```c
// hello.c — minimal kernel module

#include <linux/module.h>   /* MODULE_LICENSE, module_init, module_exit */
#include <linux/kernel.h>   /* printk, KERN_INFO */
#include <linux/init.h>     /* __init, __exit */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("lab04");
MODULE_DESCRIPTION("Minimal kernel module example");

static int __init hello_init(void)
{
	printk(KERN_INFO "hello: module loaded\n");
	return 0;   /* 0 = успех; любое ненулевое значение = ошибка загрузки */
}

static void __exit hello_exit(void)
{
	printk(KERN_INFO "hello: module unloaded\n");
}

module_init(hello_init);
module_exit(hello_exit);
```

Файл `Makefile`:

```makefile
# Makefile for out-of-tree kernel module

obj-m := hello.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	$(MAKE) -C $(KDIR) M=$(PWD) modules

clean:
	$(MAKE) -C $(KDIR) M=$(PWD) clean
```

> Отступы в Makefile — строго символ табуляции (`\t`), не пробелы.
> Если копировать текст — нужно проверить.

---

## Сборка и загрузка

```bash
$ make
```

Ожидаемый вывод:

```
make -C /lib/modules/6.12.77-6.12-alt1.forge.rv64/build M=/home/user/labs/lab04/hello modules
make[1]: вход в каталог «/usr/src/linux-6.12.77-6.12-alt1.forge.rv64»
  CC [M]  /home/user/labs/lab04/hello/hello.o
  MODPOST /home/user/labs/lab04/hello/Module.symvers
  CC [M]  /home/user/labs/lab04/hello/hello.mod.o
  CC [M]  /home/user/labs/lab04/hello/.module-common.o
  LD [M]  /home/user/labs/lab04/hello/hello.ko
  BTF [M] /home/user/labs/lab04/hello/hello.ko
Skipping BTF generation for /home/user/labs/lab04/hello/hello.ko due to unavailability of vmlinux
make[1]: выход из каталога «/usr/src/linux-6.12.77-6.12-alt1.forge.rv64»
```

Результат — файл `hello.ko` (Kernel Object).

```bash
# Загрузить модуль
$ sudo insmod hello.ko

# Убедиться что загружен
$ lsmod | grep hello

# Вывод модуля
$ dmesg | grep hello
# Ожидаемый вывод:
[20626.595997] hello: loading out-of-tree module taints kernel.
[20626.597685] hello: module loaded

# Выгрузить модуль
$ sudo rmmod hello
$ dmesg | tail -1
# ожидаемо: hello: module unloaded
```

> Если в dmesg лог "засорен" сообщениями от audit `[21136.930439] audit: type=1110 audit..` - отключите службу, добавив параметр ядра:

> в файле `/boot/BOOT/extlinux/extlinux.conf`: append root=/dev/mmcblk1p4 rw console=tty0 console=ttyS0,115200 earlycon rootwait stmmaceth=chain_mode:1 selinux=0 **audit=0**

---

## Что происходит при insmod

```
  insmod hello.ko
       │
       ▼
  finit_module() syscall (или init_module())
       │
       ▼
  kernel/module/main.c :: load_module()
       │
       ├── проверка ELF-заголовка .ko файла
       ├── разрешение символов (kallsyms, Module.symvers)
       ├── выделение памяти в пространстве ядра
       ├── копирование и релокация секций
       ├── проверка лицензии (MODULE_LICENSE)
       │
       ▼
  вызов hello_init()
       │
       ▼
  модуль в состоянии "Live", виден в lsmod
```

```bash
# Полная информация о загруженном модуле
$ sudo modinfo hello.ko
filename:       /home/user/labs/lab04/hello/hello.ko
description:    Minimal kernel module example
author:         lab04
license:        GPL
depends:        
name:           hello
vermagic:       6.12.77-6.12-alt1.forge.rv64 SMP mod_unload riscv

# Все загруженные модули
$ lsmod
```

---

## Атрибуты `__init` и `__exit`

```c
static int __init hello_init(void) { ... }
static void __exit hello_exit(void) { ... }
```

`__init` — макрос, помечающий функцию как относящуюся к секции `.init.text`.
После успешного завершения `module_init()` ядро **освобождает** эту секцию
из памяти. Функция инициализации вызывается ровно один раз — при загрузке.

`__exit` — секция `.exit.text`. Для встроенных в ядро модулей
(built-in, не `.ko`) эта секция тоже освобождается, потому что built-in
модуль нельзя выгрузить.

```bash
# Убедиться в наличии секций в .ko файле
$ objdump -h hello.ko | grep -E "init|exit|text"
  2 .text                        00000000  0000000000000000  0000000000000000  00000094  2**1  CONTENTS, ALLOC, LOAD, READONLY, CODE
  3 .init.text                   0000002a  0000000000000000  0000000000000000  00000094  2**1  CONTENTS, ALLOC, LOAD, RELOC, READONLY, CODE
  4 .exit.text                   00000020  0000000000000000  0000000000000000  000000be  2**1  CONTENTS, ALLOC, LOAD, RELOC, READONLY, CODE
  9 .exit.data                   00000008  0000000000000000  0000000000000000  000001b0  2**3  CONTENTS, ALLOC, LOAD, RELOC, DATA
 10 .init.data                   00000008  0000000000000000  0000000000000000  000001b8  2**3  CONTENTS, ALLOC, LOAD, RELOC, DATA
```

---

## `printk` и уровни важности

`printk()` — аналог `printf()` для ядра. Вывод идёт в кольцевой буфер
ядра (`dmesg`) и на консоль (если уровень важности достаточно высокий).

```c
printk(KERN_EMERG   "system is dead\n");   /* 0 — аварийная ситуация */
printk(KERN_ALERT   "action required\n");  /* 1 */
printk(KERN_CRIT    "critical\n");         /* 2 */
printk(KERN_ERR     "error\n");            /* 3 */
printk(KERN_WARNING "warning\n");          /* 4 */
printk(KERN_NOTICE  "notice\n");           /* 5 */
printk(KERN_INFO    "info\n");             /* 6 */
printk(KERN_DEBUG   "debug\n");            /* 7 */
```

Уровни `KERN_*` — это строковые макросы: `KERN_INFO` разворачивается в `"\001" "6"`.
printk-вызовы с уровнем выше текущего `console_loglevel` не выводятся на
консоль, но всегда попадают в `dmesg`.

В современном ядре для удобства есть обёртки:

```c
pr_info("hello: value = %d\n", val);   /* KERN_INFO */
pr_err("hello: failed: %d\n", ret);    /* KERN_ERR */
pr_warn("hello: warning\n");           /* KERN_WARNING */
pr_debug("hello: debug %s\n", str);    /* KERN_DEBUG (только при DEBUG) */
```

```bash
# Текущий уровень вывода консоли
$ cat /proc/sys/kernel/printk
# 4 format: current_loglevel default_loglevel min_loglevel boot_loglevel
1       4       1       7

# Временно увеличить уровень чтобы видеть KERN_DEBUG
$ echo 8 | sudo tee /proc/sys/kernel/printk
```

---

## Структура .ko файла

`.ko` — обычный ELF-файл с дополнительными секциями:

```bash
$ objdump -h hello.ko
```

Важные секции:

```
.text          — код модуля
.init.text     — функция __init (освобождается после загрузки)
.exit.text     — функция __exit
.rodata        — строковые константы
.modinfo       — MODULE_LICENSE, MODULE_AUTHOR и т.д.
__versions     — CRC версий используемых символов ядра
```

```bash
# Посмотреть строки из секции .modinfo
$ objdump -s -j .modinfo hello.ko

hello.ko:     формат файла elf64-littleriscv

Содержимое раздела .modinfo:
 0000 64657363 72697074 696f6e3d 4d696e69  description=Mini
 0010 6d616c20 6b65726e 656c206d 6f64756c  mal kernel modul
 0020 65206578 616d706c 65006175 74686f72  e example.author
 0030 3d6c6162 3034006c 6963656e 73653d47  =lab04.license=G
 0040 504c0064 6570656e 64733d00 6e616d65  PL.depends=.name
 0050 3d68656c 6c6f0076 65726d61 6769633d  =hello.vermagic=
 0060 362e3132 2e37342d 362e3132 2d616c74  6.12.77-6.12-alt
 0070 312e666f 7267652e 72763634 20534d50  1.forge.rv64 SMP
 0080 206d6f64 5f756e6c 6f616420 72697363   mod_unload risc
 0090 7600                                 v.
# или
$ strings hello.ko | grep -E "license|author|description|vermagic"
description=Minimal kernel module example
author=lab04
license=GPL
vermagic=6.12.77-6.12-alt1.forge.rv64 SMP mod_unload riscv
__UNIQUE_ID_description271
__UNIQUE_ID_author270
__UNIQUE_ID_license269
description
description
description
__UNIQUE_ID_vermagic269
__UNIQUE_ID_vermagic269
__UNIQUE_ID_description271
__UNIQUE_ID_author270
__UNIQUE_ID_license269
```

`vermagic` — строка совместимости: версия ядра + конфигурационные флаги.
insmod проверяет что `vermagic` модуля совпадает с текущим ядром.

---

← [Назад (README)](README.md) · [На главную](../INDEX.md) · [Следующий →](module2-sysfs-params.md)
