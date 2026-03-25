# module2 · Виртуальный GPIO: gpio-mockup + debugfs

← [Назад](module1-theory.md) · [На главную](../INDEX.md) · [Следующий →](module3-sysfs-gpio.md)

> **Kernel source:** `drivers/gpio/gpio-mockup.c`
> [elixir.bootlin.com/linux/v6.12/source/drivers/gpio/gpio-mockup.c](https://elixir.bootlin.com/linux/v6.12/source/drivers/gpio/gpio-mockup.c)

---

## Зачем нужен gpio-mockup

В module1 мы работали с реальными GPIO60/61 напрямую через регистры.
Это опасно: неверная запись в занятый пин может нарушить работу другого
периферийного устройства. Кроме того, для изучения GPIO API (gpiolib, libgpiod)
не всегда нужно реальное железо.

`gpio-mockup` — модуль ядра, который создаёт **виртуальный GPIO-контроллер**:

- полностью эмулируется в памяти ядра
- поддерживает весь стандартный GPIO API
- реагирует на прерывания (программно)
- безопасен: никакого реального железа
- доступен через `/sys/kernel/debug/gpio-mockup/` для инспекции

```
  Userspace (gpioset/gpioget/libgpiod)
       │
       ▼
  gpiolib (drivers/gpio/gpiolib.c)
       │
       ▼
  gpio-mockup driver (drivers/gpio/gpio-mockup.c)
       │
       ▼
  Виртуальные пины в памяти ядра (нет реального железа)
       │
       ↕  ← debugfs: /sys/kernel/debug/gpio-mockup/
       │    можно читать и записывать состояние из userspace
```

---

## Загрузка gpio-mockup

Модуль принимает параметр `gpio_mockup_ranges` — список пар `[base, ngpio]`.

```bash
# Проверить GPIO на плате
$ sudo gpiodetect
gpiochip0 [13040000.pinctrl] (64 lines)
gpiochip1 [17020000.pinctrl] (4 lines)

# Заняты: 0..67 (68 lines)

# Проверить что модуль есть в системе
$ modinfo gpio-mockup
filename:       /lib/modules/6.12.77-6.12-alt1.forge.rv64/kernel/drivers/gpio/gpio-mockup.ko
license:        GPL v2
description:    GPIO Testing driver
author:         Bartosz Golaszewski <brgl@bgdev.pl>
author:         Bamvor Jian Zhang <bamv2005@gmail.com>
author:         Kamlakant Patel <kamlakant.patel@broadcom.com>
alias:          of:N*T*Cgpio-mockupC*
alias:          of:N*T*Cgpio-mockup
depends:        
intree:         Y
name:           gpio_mockup
vermagic:       6.12.77-6.12-alt1.forge.rv64 SMP mod_unload riscv
parm:           gpio_mockup_ranges:array of int
parm:           gpio_mockup_named_lines:bool

# Загрузить: создать 1 чип с 8 линиями (начиная с '>' 68)
$ sudo modprobe gpio-mockup gpio_mockup_ranges="100,8"

# Убедиться что чип появился
$ $ sudo gpiodetect | grep gpio-mockup
gpiochip2 [gpio-mockup-A] (8 lines)

# Сохранить имя чипа в переменную
$ MOCK_CHIP=$(sudo gpiodetect | grep mockup | awk '{print $1}')
$ echo "Mock chip: $MOCK_CHIP"
Mock chip: gpiochip2
```

```bash
# Подробная информация о всех линиях (включая виртуальные)
$ sudo gpioinfo -c $MOCK_CHIP
```

Ожидаемый вывод:

```
...
gpiochip2 - 8 lines:
        line   0:       unnamed                 input
        line   1:       unnamed                 input
        ...
        line   7:       unnamed                 input
```

---

## Управление через debugfs

`gpio-mockup` предоставляет debugfs-интерфейс для имитации входных сигналов:

```bash
# Найти debugfs-каталог для mockup
$ sudo ls /sys/kernel/debug/gpio-mockup/
# ожидаемо: gpiochip2

$ MOCK_DIR="/sys/kernel/debug/gpio-mockup/$MOCK_CHIP"
$ sudo ls $MOCK_DIR
# вывод: 0  1  2  3  4  5  6  7  (файл на каждую линию)
```

```bash
# Прочитать текущее значение линии 0 (как её видит контроллер)
$ sudo cat $MOCK_DIR/0
# вывод: 0  (default — низкий уровень)

# Установить высокий уровень на линии 0 (имитация внешнего сигнала)
$ echo 1 | sudo tee $MOCK_DIR/0
$ sudo cat $MOCK_DIR/0
# вывод: 1

# Прочитать уровень через gpioget (то что видит userspace)
$ sudo gpioget --numeric -c $MOCK_CHIP 0
# вывод: 1
# 
```

---

## Сценарий loopback на виртуальном GPIO

Воспроизводим схему из лабы на виртуальных пинах: пин 0 = OUT, пин 1 = IN.

```
  Виртуальная схема (аналог GPIO60→GPIO61):

  mockup pin 0 (OUT) ──→ mockup pin 1 (IN)
                     ↑
             симулируем через debugfs
```

Одновременно обращаться к одному пину двумя утилитами gpio запрещено:
```bash
# Терминал 1: удержание сигнала (HIGH на пине '0')
$ sudo gpioset -c $MOCK_CHIP 0=1
# Терминал 2: чтение уровня сигнала на пине '0'
$ sudo gpioget --numeric -c $MOCK_CHIP 0
gpioget: unable to request lines: Device or resource busy
```
Поэтому будем читать значение из debugfs:
```bash
# Терминал 1: удержание сигнала (HIGH на пине '0')
$ sudo gpioset -c $MOCK_CHIP 0=1
# Терминал 2: чтение уровня сигнала на пине '0' через debugfs
$ sudo cat /sys/kernel/debug/gpio | grep gpioset
 gpio-100 (                    |gpioset             ) out hi
# out hi - высокий уровень выходного сигнала
# или читать в цифровом формате
$ sudo cat /sys/kernel/debug/gpio | grep "gpioset" | awk '{print ($NF == "hi" ? 1 : 0)}'
1
```

```bash
# Терминал 1: мониторинг входа
$ sudo cat /sys/kernel/debug/gpio | grep "gpioset" | awk '{print ($NF == "hi" ? 1 : 0)}'

# Терминал 2: управление выходом и имитация связи
# Установить линию 0 в высокий уровень (выход)
$ sudo gpioset -c $MOCK_CHIP 0=1

# Имитировать что линия 1 (вход) получает тот же сигнал
$ echo 1 | sudo tee $MOCK_DIR/1
$ sudo gpioget --numeric -c $MOCK_CHIP 1
# вывод: 1

# Имитировать низкий уровень
$ echo 0 | sudo tee $MOCK_DIR/1
$ sudo gpioget --numeric -c $MOCK_CHIP 1
# вывод: 0
```

---

## Написать простую программу loopback на C

Создать файл `~/labs/lab05/mockup_test.c`:

```c
// mockup_test.c — GPIO loopback test using libgpiod v2
//
// Line 0: output (driven by us)
// Line 1: input  (level set via debugfs, read via gpiod)

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <errno.h>
#include <gpiod.h>

#define CHIP_LABEL  "gpio-mockup-A"
#define DEBUGFS_DIR "/sys/kernel/debug/gpio-mockup/gpiochip2"
#define OUT_OFFSET  0
#define IN_OFFSET   1

// Найти /dev/gpiochipN по label
static struct gpiod_chip *open_chip_by_label(const char *label)
{
    char path[32];
    for (int i = 0; i < 16; i++) {
        snprintf(path, sizeof(path), "/dev/gpiochip%d", i);
        struct gpiod_chip *chip = gpiod_chip_open(path);
        if (!chip) continue;

        struct gpiod_chip_info *info = gpiod_chip_get_info(chip);
        if (!info) { gpiod_chip_close(chip); continue; }

        int match = (strcmp(gpiod_chip_info_get_label(info), label) == 0);
        gpiod_chip_info_free(info);

        if (match) return chip;
        gpiod_chip_close(chip);
    }
    return NULL;
}

// Установить уровень на линии через debugfs
static int debugfs_set(int line, int val)
{
    char path[128];
    snprintf(path, sizeof(path), "%s/%d", DEBUGFS_DIR, line);
    FILE *f = fopen(path, "w");
    if (!f) { perror(path); return -1; }
    fprintf(f, "%d\n", val);
    fclose(f);
    return 0;
}

int main(void)
{
    // Открыть чип
    struct gpiod_chip *chip = open_chip_by_label(CHIP_LABEL);
    if (!chip) {
        fprintf(stderr, "Chip '%s' not found. Is gpio-mockup loaded?\n", CHIP_LABEL);
        return EXIT_FAILURE;
    }

    // Настроить line 0 как выход
    struct gpiod_line_settings *out_settings = gpiod_line_settings_new();
    gpiod_line_settings_set_direction(out_settings, GPIOD_LINE_DIRECTION_OUTPUT);
    gpiod_line_settings_set_output_value(out_settings, GPIOD_LINE_VALUE_INACTIVE);

    struct gpiod_line_config *out_cfg = gpiod_line_config_new();
    unsigned int out_off = OUT_OFFSET;
    gpiod_line_config_add_line_settings(out_cfg, &out_off, 1, out_settings);

    struct gpiod_request_config *out_req_cfg = gpiod_request_config_new();
    gpiod_request_config_set_consumer(out_req_cfg, "mockup_test");

    struct gpiod_line_request *out_req =
        gpiod_chip_request_lines(chip, out_req_cfg, out_cfg);
    if (!out_req) {
        perror("request output line");
        return EXIT_FAILURE;
    }

    // Настроить line 1 как вход
    struct gpiod_line_settings *in_settings = gpiod_line_settings_new();
    gpiod_line_settings_set_direction(in_settings, GPIOD_LINE_DIRECTION_INPUT);

    struct gpiod_line_config *in_cfg = gpiod_line_config_new();
    unsigned int in_off = IN_OFFSET;
    gpiod_line_config_add_line_settings(in_cfg, &in_off, 1, in_settings);

    struct gpiod_request_config *in_req_cfg = gpiod_request_config_new();
    gpiod_request_config_set_consumer(in_req_cfg, "mockup_test");

    struct gpiod_line_request *in_req =
        gpiod_chip_request_lines(chip, in_req_cfg, in_cfg);
    if (!in_req) {
        perror("request input line");
        return EXIT_FAILURE;
    }

    printf("GPIO loopback test (mockup) — libgpiod v2\n");
    printf("OUT=line%d  IN=line%d\n\n", OUT_OFFSET, IN_OFFSET);

    int pass = 0, fail = 0;

    for (int i = 0; i < 4; i++) {
        int out_val = i % 2;  // 0, 1, 0, 1
        enum gpiod_line_value gpiod_out =
            out_val ? GPIOD_LINE_VALUE_ACTIVE : GPIOD_LINE_VALUE_INACTIVE;

        // Выставить выход
        gpiod_line_request_set_value(out_req, OUT_OFFSET, gpiod_out);
        printf("OUT set to %d\n", out_val);

        // Имитировать тот же уровень на входе через debugfs
        debugfs_set(IN_OFFSET, out_val);

        usleep(100000);  // 100 мс

        // Прочитать вход
        enum gpiod_line_value raw = gpiod_line_request_get_value(in_req, IN_OFFSET);
        int in_val = (raw == GPIOD_LINE_VALUE_ACTIVE) ? 1 : 0;

        printf("IN  read %d  — %s\n\n",
               in_val, in_val == out_val ? "OK" : "MISMATCH");

        if (in_val == out_val) pass++; else fail++;
    }

    printf("Result: %d passed / %d failed\n", pass, fail);

    // Освободить ресурсы
    gpiod_line_request_release(out_req);
    gpiod_line_request_release(in_req);
    gpiod_line_settings_free(out_settings);
    gpiod_line_settings_free(in_settings);
    gpiod_line_config_free(out_cfg);
    gpiod_line_config_free(in_cfg);
    gpiod_request_config_free(out_req_cfg);
    gpiod_request_config_free(in_req_cfg);
    gpiod_chip_close(chip);

    return fail == 0 ? EXIT_SUCCESS : EXIT_FAILURE;
}
```

```bash
$ cd ~/labs/lab05
$ gcc -o mockup_test mockup_test.c -lgpiod
$ sudo ./mockup_test
```

Ожидаемый вывод:

```
GPIO loopback test (mockup) — libgpiod v2
OUT=line0  IN=line1

OUT set to 0
IN  read 0  — OK

OUT set to 1
IN  read 1  — OK

OUT set to 0
IN  read 0  — OK

OUT set to 1
IN  read 1  — OK

Result: 4 passed / 0 failed
```

---

## ftrace на виртуальном GPIO

```bash
# Посмотреть функции gpio-mockup доступные для трассировки
$ sudo cat /sys/kernel/debug/tracing/available_filter_functions | grep mockup
gpio_mockup_debugfs_cleanup [gpio_mockup]
gpio_mockup_free [gpio_mockup]
gpio_mockup_request [gpio_mockup]
gpio_mockup_get_direction [gpio_mockup]
gpio_mockup_dirin [gpio_mockup]
gpio_mockup_dirout [gpio_mockup]
gpio_mockup_set [gpio_mockup]
gpio_mockup_get [gpio_mockup]
gpio_mockup_to_irq [gpio_mockup]
gpio_mockup_apply_pull [gpio_mockup]
gpio_mockup_set_config [gpio_mockup]
gpio_mockup_set_multiple [gpio_mockup]
gpio_mockup_get_multiple [gpio_mockup]
gpio_mockup_debugfs_open [gpio_mockup]
gpio_mockup_debugfs_write [gpio_mockup]
gpio_mockup_dispose_mappings [gpio_mockup]
gpio_mockup_probe [gpio_mockup]
gpio_mockup_debugfs_read [gpio_mockup]
gpio_mockup_unregister_pdevs [gpio_mockup]

# Трассировать операции чтения и записи
$ echo | sudo tee /sys/kernel/debug/tracing/set_ftrace_filter
$ echo 'gpio*set* gpio*get* gpiod_*' | sudo tee /sys/kernel/debug/tracing/set_graph_function
$ echo function_graph | sudo tee /sys/kernel/debug/tracing/current_tracer
$ echo   | sudo tee /sys/kernel/debug/tracing/trace
$ echo 1 | sudo tee /sys/kernel/debug/tracing/tracing_on

$ sudo ./mockup_test

$ echo 0 | sudo tee /sys/kernel/debug/tracing/tracing_on
$ sudo cat /sys/kernel/debug/tracing/trace | less
```

Ожидаемый вывод (неполный):

```bash
# tracer: function_graph
#
# CPU  DURATION                  FUNCTION CALLS
# |     |   |                     |   |   |   |
 0)               |  gpio_device_get() {
 0)   1.750 us    |    get_device();
 0)   6.250 us    |  }
 0)               |  gpio_device_get() {
 0)   1.500 us    |    get_device();
 0)   5.750 us    |  }
 0)   1.500 us    |  gpio_device_get_desc();
 0)               |  gpiod_request() {
 0)   2.250 us    |    try_module_get();
 0)               |    gpiod_request_commit() {
 0)   1.750 us    |      __srcu_read_lock();
 0)   1.500 us    |      gpiochip_line_is_valid();
 0)               |      gpio_mockup_request [gpio_mockup]() {
 0)   1.500 us    |        gpiochip_get_data();
 0)   1.500 us    |        mutex_lock();
 0)   1.500 us    |        mutex_unlock();
 0) + 13.250 us   |      }
 0)               |      gpiod_get_direction() {
 0)   1.500 us    |        __srcu_read_lock();
 0)               |        gpio_mockup_get_direction [gpio_mockup]() {
 0)   1.500 us    |          gpiochip_get_data();
 0)   1.500 us    |          mutex_lock();
 0)   1.500 us    |          mutex_unlock();
 0) + 12.750 us   |        }
 0)   1.750 us    |        __srcu_read_unlock();
 0) + 24.500 us   |      }
 0)               |      desc_set_label() {
 0)   2.250 us    |        __kmalloc_noprof();
 0)   6.250 us    |      }
 0)   1.500 us    |      __srcu_read_unlock();
 0) + 64.250 us   |    }
 0)   1.500 us    |    get_device();
 0) + 76.500 us   |  }
```

Представление о графе вызываемых функций помогает в отладке кода
драйвера, например можно отследить исходный код функции
[`gpiod_request`](https://elixir.bootlin.com/linux/v6.12/source/drivers/gpio/gpiolib.c#L2346).

---

## Выгрузка gpio-mockup

```bash
# Сначала освободить все линии (программы должны завершиться)
$ sudo rmmod gpio-mockup

# Убедиться что чип исчез
$ sudo gpiodetect | grep mockup
# пустой вывод
```

---

← [Назад](module1-theory.md) · [На главную](../INDEX.md) · [Следующий →](module3-sysfs-gpio.md)
