# Урок 04: Модуль ядра + sysfs + ftrace

← [Урок 03](../lab03-boot/README.md) · [На главную](../INDEX.md) · [Урок 05 →](../lab05-gpio/README.md)

---

## Цель урока

Написать, собрать и загрузить модуль ядра Linux. Научиться управлять им
из userspace через sysfs и параметры модуля. Разобрать ftrace — главный
инструмент трассировки ядра — вплоть до уровня ассемблера: что именно
происходит с функцией когда она инструментируется на RISC-V.

Этот урок — фундамент для всех последующих лабораторных. Каждый драйвер
GPIO, I2C, SPI — это модуль ядра. Каждый раз когда нужно понять что
происходит внутри ядра — используется ftrace.

---

## Модули

| Модуль | Тема |
|--------|------|
| [module1-lkm-basics.md](module1-lkm-basics.md) | Структура модуля: init/exit, Makefile, insmod/rmmod |
| [module2-sysfs-params.md](module2-sysfs-params.md) | sysfs-атрибуты и параметры модуля (`module_param`) |
| [module3-counter-module.md](module3-counter-module.md) | Практика: модуль-счётчик с несколькими функциями и управлением из sysfs |
| [module4-ftrace.md](module4-ftrace.md) | ftrace: function/function_graph трассировщики, инструментация на уровне ассемблера RISC-V |

---

## Что понадобится

- Плата с ALT Linux, заголовки ядра установлены
- `gcc`, `make` установлены
- `trace-cmd` установлен: `apt-get install trace-cmd`
- Рабочий каталог: `~/labs/lab04/`

```bash
$ mkdir -p ~/labs/lab04
$ cd ~/labs/lab04
```

---

[Начнём →](module1-lkm-basics.md)
