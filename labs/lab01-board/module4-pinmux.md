# module4 · Pinmux и pinctrl: управление мультиплексированием

← [Назад](module3-pinout.md) · [На главную](../INDEX.md) · [Урок 02 →](../lab02-riscv/README.md)

> **Kernel source:** `drivers/pinctrl/starfive/pinctrl-starfive-jh7110-sys.c`
> [elixir.bootlin.com/linux/v6.12/source/drivers/pinctrl/starfive](https://elixir.bootlin.com/linux/v6.12/source/drivers/pinctrl/starfive)

---

## Проблема: пинов меньше, чем функций

На GPIO40-разъёме 28 сигнальных пинов. Внутри JH7110 — 6 UART, 7 I2C,
7 SPI и 64 GPIO. Физически один пин не может одновременно выполнять
несколько функций. Решение — **мультиплексирование**: каждый физический
пин подключён к нескольким внутренним сигналам через аппаратный
коммутатор (IOMUX). В каждый момент активен только один; выбор задаётся
записью в регистры.

```
          Пример: GPIO5 (пин 8 на разъёме)

     Внутри SoC                              Снаружи
  ┌────────────────────────────────┐
  │           sys_iomux            │
  │          @0x13040000           │
  │                                │
  │  ┌─────────────┐               │
  │  │  GPIO fn    │ ───────────┐  │
  │  └─────────────┘            │  │
  │  ┌─────────────┐            ├──┼──► физический пад (пин 8)
  │  │  UART0_TX   │ ───────────┘  │
  │  └─────────────┘           ▲   │
  │  ┌─────────────┐           │   │
  │  │  (другие)   │     регистр   │
  │  └─────────────┘    GPIOMUX    │
  └────────────────────────────────┘
```

Регистры `sys_iomux` находятся по адресу `0x13040000`. Linux пишет
в них при инициализации любого периферийного устройства.

---

## Подсистема pinctrl в Linux

**Kernel docs:** [kernel.org/doc/html/latest/driver-api/pin-control.html](https://www.kernel.org/doc/html/latest/driver-api/pin-control.html)

Драйвер I2C или UART не трогает регистры IOMUX напрямую. Он запрашивает
нужную конфигурацию у подсистемы **pinctrl**, которая знает, как именно
нужно переключить регистры для данной платформы.

```
  Драйвер I2C (i2c@10030000)
       │
       │  devm_pinctrl_get()
       │  pinctrl_select_state("default")
       ▼
  pinctrl core (drivers/pinctrl/core.c)
       │
       │  pinmux_ops->set_mux()
       ▼
  JH7110 pinctrl driver
  (pinctrl-starfive-jh7110-sys.c)
       │
       │  writel() → регистры sys_iomux @0x13040000
       ▼
  GPIO57 → I2C0_SCL
  GPIO58 → I2C0_SDA
```

Такое разделение позволяет коду драйвера I2C быть платформонезависимым:
один и тот же `drivers/i2c/busses/i2c-designware-core.c` работает на
десятках SoC, потому что переключением пинов занимается отдельный слой.

---

## Формат значения `pinmux` в DTS

Каждый пин в DTS описан одним 32-битным числом. Макрос `GPIOMUX`
определён в [`jh7110-pinfunc.h`](https://elixir.bootlin.com/linux/v6.12/source/arch/riscv/boot/dts/starfive/jh7110-pinfunc.h#L21):

```c
/*
 * mux bits:
 *  | 31 - 24 | 23 - 16 | 15 - 10 |  9 - 8   |  7 - 0  |
 *  |  din    |  dout   |  doen   | function | gpio nr |
 */
#define GPIOMUX(n, dout, doen, din) ( \
    (((din)  & 0xff) << 24) | \
    (((dout) & 0xff) << 16) | \
    (((doen) & 0x3f) << 10) | \
    ((n) & 0x3f))
```

Поля:

| Биты | Поле | Описание |
|------|------|----------|
| [31:24] | `din` | Индекс входного сигнала (`GPI_SYS_*`). `0xff` = `GPI_NONE` — пин не является входом |
| [23:16] | `dout` | Индекс выходного сигнала (`GPOUT_SYS_*`). `0x00` = `GPOUT_LOW` |
| [15:10] | `doen` | Индекс сигнала разрешения выхода (`GPOEN_*`). `0` = `GPOEN_ENABLE`, `1` = `GPOEN_DISABLE` |
| [9:8]   | `function` | Селектор режима. `0` — режим GPIOMUX |
| [7:0]   | `gpio nr` | Номер GPIO-пина (0–63) |

Символьные имена для всех трёх таблиц (`GPOUT_SYS_*`, `GPOEN_*`,
`GPI_SYS_*`) перечислены в том же файле `jh7110-pinfunc.h`.

---

## Конфигурация в Device Tree

Всё что написано ниже — реальные данные из DTS этой платы
(`dtc -I fs /proc/device-tree`).

### Пример 1: I2C0 (пины 3 и 5)

```dts
/* Группа пинов для I2C0, описана в узле pinctrl@13040000 */
i2c0-0 {

    i2c-pins {
        bias-disable;
        pinmux = <0x9001439 0xa00183a>;
        input-schmitt-enable;
        input-enable;
    };
};

/* Привязка к устройству */
i2c@10030000 {
    pinctrl-names = "default";
    pinctrl-0 = <0x17>;       /* ссылка на группу i2c0-0 */
    clock-frequency = <0x186a0>; /* 100 кГц */
    status = "okay";
    ...
};
```

Два значения `pinmux` — по одному на каждый пин. Декодирование:

```
0x09001439:
  gpio nr  [ 7: 0] = 0x39 = 57  → GPIO57  (пин 5, SCL)
  doen     [15:10] = 0b000101=5 → GPOEN_SYS_I2C0_CLK
  dout     [23:16] = 0x00 = 0   → GPOUT_LOW
  din      [31:24] = 0x09 = 9   → GPI_SYS_I2C0_CLK

0x0a00183a:
  gpio nr  [ 7: 0] = 0x3a = 58  → GPIO58  (пин 3, SDA)
  doen     [15:10] = 0b000110=6 → GPOEN_SYS_I2C0_DATA
  dout     [23:16] = 0x00 = 0   → GPOUT_LOW
  din      [31:24] = 0x0a = 10  → GPI_SYS_I2C0_DATA
```

Почему `dout = GPOUT_LOW`? I2C — **открытый коллектор**: пин никогда
не тянется активно к высокому уровню, только удерживается в LOW или
отпускается (HIGH через внешнюю подтяжку). Направлением управляет
`doen` — контроллер I2C сам переключает сигналы `GPOEN_SYS_I2C0_CLK`
и `GPOEN_SYS_I2C0_DATA` между `GPOEN_ENABLE` (пин — выход LOW)
и `GPOEN_DISABLE` (высокий импеданс, шина отпущена).

Именно эти записи определяют, что при загрузке I2C0-драйвера пины 3 и 5
(GPIO58, GPIO57) переключаются на I2C0_SDA и I2C0_SCL и перестают быть
свободными GPIO.

### Пример 2: UART0 (пины 8 и 10)

```dts
uart0-0 {

    tx-pins {
        input-schmitt-disable;
        input-disable;
        drive-strength = <0x0c>;
        bias-disable;
        pinmux = <0xff140005>;
        slew-rate = <0x00>;
    };

    rx-pins {
        drive-strength = <0x02>;
        bias-disable;
        pinmux = <0xe000406>;
        input-schmitt-enable;
        slew-rate = <0x00>;
        input-enable;
    };
};
```

```
0xff140005:
  gpio nr  [ 7: 0] = 0x05 = 5   → GPIO5   (пин 8, TX)
  doen     [15:10] = 0b000000=0 → GPOEN_ENABLE  (выход всегда разрешён)
  dout     [23:16] = 0x14 = 20  → GPOUT_SYS_UART0_TX
  din      [31:24] = 0xff = 255 → GPI_NONE  (нет входа — чистый выход)

0x0e000406:
  gpio nr  [ 7: 0] = 0x06 = 6   → GPIO6   (пин 10, RX)
  doen     [15:10] = 0b000001=1 → GPOEN_DISABLE  (выход отключён — чистый вход)
  dout     [23:16] = 0x00 = 0   → GPOUT_LOW
  din      [31:24] = 0x0e = 14  → GPI_SYS_UART0_RX
```

UART0 — это тот самый debug-UART, к которому подключается USB-UART
адаптер при работе с платой через консоль (см. setup.md). Оба пина
выведены на 40-пиновый разъём.

### Пример 3: SPI0 (пины 19, 21, 23, 24)

```dts
spi0-0 {

    miso-pins {
        pinmux = <0x1c000435>;
        input-schmitt-enable;
        bias-pull-up;
        input-enable;
    };

    mosi-pins {
        input-schmitt-disable;
        input-disable;
        bias-disable;
        pinmux = <0xff200034>;
    };

    ss-pins {
        input-schmitt-disable;
        input-disable;
        bias-disable;
        pinmux = <0x1b1f0031>;
    };

    sck-pins {
        input-schmitt-disable;
        input-disable;
        bias-disable;
        pinmux = <0x1a1e0030>;
    };
};
```

```
0xff200034 (MOSI):
  gpio nr = 0x34 = 52  → GPIO52 (пин 19)
  doen    = 0          → GPOEN_ENABLE
  dout    = 0x20 = 32  → GPOUT_SYS_SPI0_TXD
  din     = 0xff       → GPI_NONE

0x1c000435 (MISO):
  gpio nr = 0x35 = 53  → GPIO53 (пин 21)
  doen    = 1          → GPOEN_DISABLE
  dout    = 0          → GPOUT_LOW
  din     = 0x1c = 28  → GPI_SYS_SPI0_RXD

0x1a1e0030 (SCK):
  gpio nr = 0x30 = 48  → GPIO48 (пин 23)
  doen    = 0          → GPOEN_ENABLE
  dout    = 0x1a = 26  → GPOUT_SYS_SPI0_CLK
  din     = 0x1e = 30  → GPI_SYS_SPI0_CLK     ← нужен для slave-режима

0x1b1f0031 (SS0):
  gpio nr = 0x31 = 49  → GPIO49 (пин 24)
  doen    = 0          → GPOEN_ENABLE
  dout    = 0x1b = 27  → GPOUT_SYS_SPI0_FSS
  din     = 0x1f = 31  → GPI_SYS_SPI0_FSS     ← нужен для slave-режима
```

Для MISO включён `input-schmitt-enable` и `bias-pull-up` — стандартная
практика для линий с возможной неопределённостью уровня в режиме
ожидания. MOSI — чистый выход: `GPI_NONE` + `GPOEN_ENABLE`, симметрично
тому, как устроен UART TX.

SCK и SS имеют заполненные `din`, хотя плата сейчас работает в режиме
мастера. Это позволяет без изменений DTS переключиться в slave-режим.

---

## Практика: инспекция через debugfs

```bash
# Список pinctrl-контроллеров
$ sudo ls /sys/kernel/debug/pinctrl/
```

Ожидаемый вывод:
```
13040000.pinctrl   ← основная периферия (sys-iomux)
17020000.pinctrl   ← домен Always-On (не обесточивается при suspend) (aon-iomux)
pinctrl-devices
pinctrl-handles
pinctrl-maps
```

```bash
# Текущее назначение каждого занятого пина
$ sudo cat /sys/kernel/debug/pinctrl/13040000.pinctrl/pinmux-pins
```

Пример строк (реальные данные для активных устройств):
```
pin 5 (GPIO5): 10000000.serial (GPIO UNCLAIMED) function uart0-0 group uart0-0.tx-pins
pin 6 (GPIO6): 10000000.serial (GPIO UNCLAIMED) function uart0-0 group uart0-0.rx-pins
pin 48 (GPIO48): 10060000.spi (GPIO UNCLAIMED) function spi0-0 group spi0-0.sck-pins
pin 52 (GPIO52): 10060000.spi (GPIO UNCLAIMED) function spi0-0 group spi0-0.mosi-pins
pin 57 (GPIO57): 10030000.i2c (GPIO UNCLAIMED) function i2c0-0 group i2c0-0.i2c-pins
pin 58 (GPIO58): 10030000.i2c (GPIO UNCLAIMED) function i2c0-0 group i2c0-0.i2c-pins
```

`GPIO UNCLAIMED` означает, что GPIO-контроллер не держит пин —
он передан драйверу периферии. Попытка управлять им через libgpiod
вернёт `EBUSY`.

```bash
# Электрические параметры: drive strength, pull-up/down, schmitt
$ sudo cat /sys/kernel/debug/pinctrl/13040000.pinctrl/pinconf-pins

# Все зарегистрированные функции мультиплексора
$ sudo cat /sys/kernel/debug/pinctrl/13040000.pinctrl/pinmux-functions
```
---

← [Назад](module3-pinout.md) · [На главную](../INDEX.md) · [Урок 02 →](../lab02-riscv/README.md)
