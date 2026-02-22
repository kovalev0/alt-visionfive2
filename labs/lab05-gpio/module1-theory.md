# module1 · Теория: GPIO-контроллер JH7110 и прямая работа с регистрами

← [Назад (README)](README.md) · [На главную](../INDEX.md) · [Следующий →](module2-virtual-gpio.md)

> **TRM ref:**
> [GPIO Overview](https://doc-en.rvspace.org/JH7110/TRM/JH7110_TRM/overview_gpio.html)

> [SYS IOMUX CFG](https://doc-en.rvspace.org/JH7110/TRM/JH7110_TRM/sys_iomux_cfg.html)

> **Драйвер в ядре:** `drivers/pinctrl/starfive/`
> [elixir.bootlin.com/linux/v6.12/source/drivers/pinctrl/starfive](https://elixir.bootlin.com/linux/v6.12/source/drivers/pinctrl/starfive)

---

## Физическая схема подключения

В этой лабе мы работаем с двумя соседними пинами разъёма GPIO40:

```
  GPIO40 разъём VisionFive2
  (фрагмент, вид сверху)

        нечётные     чётные
  35 ○  GPIO63    GPIO36  ○ 36
       ┌────────────────┐
  37 ● │ GPIO60   GPIO61│ ● 38   ← провод dupont
       └────────────────┘
  39 ○  GND       GPIO44  ○ 40

  ● — используемый пин     ○ — неиспользуемый
  └──── провод dupont соединяет пин 37 и пин 38 ────┘
  GPIO60 (пин 37) = OUT,  GPIO61 (пин 38) = IN
```

Провод dupont [соединяет](README.md#что-понадобится) пин 37 (GPIO60, выход) и
пин 38 (GPIO61, вход).
Всё что мы будем программно выставлять на GPIO60 — тут же можно читать с GPIO61.

---

## Что такое GPIO на уровне железа

GPIO-пин — это ячейка в SoC с тремя функциональными элементами:

```
  Физический пин (pad)
        │
        │  двунаправленный буфер
        │
  ┌─────┴────────────────────────────────────────────────────┐
  │                   SYS IOMUX Controlle                    │
  │                                                          │
  │  DOEN mux ──→  output enable                             │
  │    6-битное поле: выбирает источник сигнала OEN          │
  │    0x00 = GPOEN_ENABLE  → пин-выход (всегда включён)     │
  │    0x01 = GPOEN_DISABLE → пин-вход  (выход отключён)     │
  │    0x02…0x31 = OEN управляется периферией (UART, SPI…)   │
  │                                                          │
  │  DOUT mux ──→  data output                               │
  │    7-битное поле: выбирает что именно выдаётся на пин    │
  │    0x00 = GPOUT_LOW  → постоянный LOW                    │
  │    0x01 = GPOUT_HIGH → постоянный HIGH                   │
  │    0x02…0x6B = выход от периферии (UART TX, SPI MOSI…)   │
  │                                                          │
  │  GPIOIN  ←──  фактическое состояние пина                 │
  │    1 бит, читается независимо от DOEN                    │
  └──────────────────────────────────────────────────────────┘
```

**Ключевое отличие JH7110 от классических GPIO-контроллеров:**
DOEN и DOUT — это **мультиплексоры сигналов**, а не прямые регистры
направления и значения. Каждый пин хранит индекс (номер строки в таблице
мультиплексора), а не сам бит 0/1. Это позволяет подключать к физическому
пину любой внутренний сигнал SoC, не переконфигурируя сам пин.

---

## Регистровая карта GPIO-контроллера JH7110

Контроллер `sys_iomux` — на адресе `0x13040000`.

```
$ sudo cat /proc/iomem | grep 13040000
13040000-1304ffff : 13040000.pinctrl
```

Полная карта регистров (из [`drivers/pinctrl/starfive/pinctrl-starfive-jh7110-sys.c`](https://elixir.bootlin.com/linux/v6.12/source/drivers/pinctrl/starfive/pinctrl-starfive-jh7110-sys.c#L58)):

```
  Базовый адрес: 0x13040000

  Смещение   Имя в ядре               Описание
  ──────────────────────────────────────────────────────────────────
  0x000      JH7110_SYS_DOEN          DOEN мультиплексор, GPIO0–63
                                      6 бит/GPIO, 4 GPIO в регистре
                                      16 регистров: 0x000 … 0x03C

  0x040      JH7110_SYS_DOUT          DOUT мультиплексор, GPIO0–63
                                      7 бит/GPIO, 4 GPIO в регистре
                                      16 регистров: 0x040 … 0x07C

  0x080      JH7110_SYS_GPI           GPI routing (входные сигналы → периферия)
                                      регистры: 0x080 … 0x0D8

  0x0DC      JH7110_SYS_GPIOEN        Глобальное включение прерываний GPIO
  0x0E0      JH7110_SYS_GPIOIS0       Тип прерывания: edge/level, GPIO0–31
  0x0E4      JH7110_SYS_GPIOIS1       Тип прерывания: edge/level, GPIO32–63
  0x0E8      JH7110_SYS_GPIOIC0       Сброс прерывания, GPIO0–31
  0x0EC      JH7110_SYS_GPIOIC1       Сброс прерывания, GPIO32–63
  0x0F0      JH7110_SYS_GPIOIBE0      Both-edges, GPIO0–31
  0x0F4      JH7110_SYS_GPIOIBE1      Both-edges, GPIO32–63
  0x0F8      JH7110_SYS_GPIOIEV0      Полярность прерывания, GPIO0–31
  0x0FC      JH7110_SYS_GPIOIEV1      Полярность прерывания, GPIO32–63
  0x100      JH7110_SYS_GPIOIE0       Interrupt enable, GPIO0–31
  0x104      JH7110_SYS_GPIOIE1       Interrupt enable, GPIO32–63
  0x108      JH7110_SYS_GPIORIS0      Raw interrupt status, GPIO0–31
  0x10C      JH7110_SYS_GPIORIS1      Raw interrupt status, GPIO32–63
  0x110      JH7110_SYS_GPIOMIS0      Masked interrupt status, GPIO0–31
  0x114      JH7110_SYS_GPIOMIS1      Masked interrupt status, GPIO32–63
  0x118      JH7110_SYS_GPIOIN        Состояние пинов GPIO0–31  (1 бит/GPIO)
  0x11C      JH7110_SYS_GPIOIN+4      Состояние пинов GPIO32–63 (1 бит/GPIO)
  0x120      JH7110_SYS_GPO_PDA…      Pad config (pull, drive strength, schmitt)
  ──────────────────────────────────────────────────────────────────
```

---

## Как упакованы поля в регистрах DOEN и DOUT

4 GPIO помещаются в один 32-битный регистр. Каждый занимает 8 бит (хотя
реально использует 6 или 7 — остальные зарезервированы):

```
  DOEN-регистр для GPIO60-63 @ 0x1304003C:

  биты:  31…24     23…16     15…8      7…0
         GPIO63    GPIO62    GPIO61    GPIO60
         [29:24]   [21:16]   [13:8]    [5:0]    ← 6 значимых бит из 8

  DOUT-регистр для GPIO60-63 @ 0x1304007C:

  биты:  31…24     23…16     15…8      7…0
         GPIO63    GPIO62    GPIO61    GPIO60
         [30:24]   [22:16]   [14:8]    [6:0]    ← 7 значимых бит из 8
```

Формула для произвольного GPIO N:

```
  DOEN-регистр:
    адрес   = 0x13040000 + 0x000 + (N / 4) * 4
    биты    = [5:0] сдвинутые на (N % 4) * 8

  DOUT-регистр:
    адрес   = 0x13040000 + 0x040 + (N / 4) * 4
    биты    = [6:0] сдвинутые на (N % 4) * 8

  GPIOIN-регистр:
    адрес   = 0x13040000 + 0x118 + (N / 32) * 4
    бит     = N % 32
```

Для **GPIO60**: N=60, N/4=15, N%4=0 → поля в битах [5:0] / [6:0], регистр 15-й.
Для **GPIO61**: N=61, N/4=15, N%4=1 → поля в битах [13:8] / [14:8], тот же регистр.

---

## Раздел I: чтение регистров — busybox devmem

> Установите утилиту busybox командой `sudo apt-get install busybox`

`busybox devmem` читает одно 32-битное слово по физическому адресу:

```bash
$ sudo busybox devmem АДРЕС
```

### Читаем DOEN для GPIO60-63

[SYS IOMUX CFGSAIF SYSCFG FMUX 15](https://doc-en.rvspace.org/JH7110/TRM/JH7110_TRM/sys_iomux_cfg.html#sys_iomux_cfg__section_fv5_s3b_xsb)

```bash
$ DOEN=`sudo busybox devmem 0x1304003c`
$ echo $DOEN
0x01000101
```

Разбираем значение `0x01000101`:

```
0x01000101 = 0b 00000001 00000000 00000001 00000001

  биты [31:24] = 0x01 → GPIO63 DOEN = 0x01 = GPOEN_DISABLE = вход
  биты [23:16] = 0x00 → GPIO62 DOEN = 0x00 = GPOEN_ENABLE  = выход
  биты [15:8]  = 0x01 → GPIO61 DOEN = 0x01 = GPOEN_DISABLE = вход
  биты [7:0]   = 0x01 → GPIO60 DOEN = 0x01 = GPOEN_DISABLE = вход
```

Ядро при старте настроило GPIO60, GPIO61, GPIO63 как **входы** (0x01),
а GPIO62 — как **выход** (0x00). GPIO62 — это сброс MMC (`mmc0-0.rst-pins`),
это видно из `pinmux-pins`:

```bash
$ sudo cat /sys/kernel/debug/pinctrl/13040000.pinctrl/pinmux-pins | grep GPIO62
pin 62 (GPIO62): 16010000.mmc (GPIO UNCLAIMED) function mmc0-0 group mmc0-0.rst-pins
```

### Читаем DOUT для GPIO60-63

[SYS IOMUX CFGSAIF SYSCFG FMUX 31](https://doc-en.rvspace.org/JH7110/TRM/JH7110_TRM/sys_iomux_cfg.html#sys_iomux_cfg__section_bfb_t3b_xsb)

[GPIO OEN List for SYS_IOMUX](https://doc-en.rvspace.org/JH7110/TRM/JH7110_TRM/gpio_oen_list.html#gpio_oen_list__table_qnr_fsg_ltb)

```bash
$ DOUT=`sudo busybox devmem 0x1304007c`
$ echo $DOUT
0x5F135D5E
```

```
0x5F135D5E = 0b 01011111 00010011 01011101 01011110

  GPIO60 DOUT = биты[6:0]   = 0x5e = 94 → u4_ssp_spi_SSPFSSOUT
  GPIO61 DOUT = биты[14:8]  = 0x5D = 93 → u4_ssp_spi_SSPCLKOUT
  GPIO62 DOUT = биты[22:16] = 0x13 = 19 → u0_sdio_rst_n
  GPIO63 DOUT = биты[30:24] = 0x5F = 95 → u4_ssp_spi_SSPTXD
```

Пины настроены на альтернативные функции: GPIO62 — сброс интерфейса SDIO,
остальные (GPIO60, GPIO61, GPIO63) — SPI сигналы.

### Читаем состояние пинов GPIOIN

[SYS IOMUX CFGSAIF SYSCFG IOIRQ 71 Register Description](https://doc-en.rvspace.org/JH7110/TRM/JH7110_TRM/sys_iomux_cfg.html#sys_iomux_cfg__section_rlq_t3b_xsb)

```bash
$ GPIOIN=`sudo busybox devmem 0x1304011c`
$ echo $GPIOIN
0x1720BDF1
```

```
  0x1720BDF1 = 0b 00010111 00100000 10111101 11110001
                  │ ││             ...              │
             GPIO63 ││             ...           GPIO32
               GPIO61│
                GPIO60

  бит 28 = GPIO60 = 1   (HIGH)
  бит 29 = GPIO61 = 0   (LOW)
```

---

## Раздел II: запись в регистры — делаем GPIO60 HIGH

Цель: GPIO60 → выход HIGH, читаем HIGH на GPIO61.

### Шаг 1. Выставляем GPIO60 как выход (DOEN = GPOEN_ENABLE = 0x00)

Текущее значение регистра DOEN: `0x01000101`.
Нам нужно обнулить биты [5:0] (поле GPIO60), остальное не трогать:

```
  Было:   0x01000101  (GPIO60 = 0x01 = вход)
  Маска:  0xFFFFFF C0  (обнуляем биты[5:0])
  Стало:  0x01000100  (GPIO60 = 0x00 = выход)
```

```bash
$ sudo busybox devmem 0x1304003c w 0x01000100
```

### Шаг 2. Выставляем GPIO60 HIGH (DOUT = GPOUT_HIGH = 0x01)

Текущее значение DOUT: `0x5F135D5E`.
Нам нужно биты [6:0] заменить на 0x01:

```
  Было:   0x5F135D5E  (GPIO60 = 0x5e — альт. сигнал)
  Маска:  0xFFFFFF80  (обнуляем биты[6:0])
  OR:     0x00000001  (GPIO60 = 0x01 = HIGH)
  Стало:  0x5F135D01
```

```bash
$ sudo busybox devmem 0x1304007c w 0x5F135D01
```

### Шаг 3. Читаем GPIO61 (вход)

Замыкаем на разъеме 2 пина ()

```bash
$ sudo busybox devmem 0x1304011c
0x3720BDF1
```

```
  бит 28 = GPIO60 = 1   ✓  HIGH (мы выставили)
  бит 29 = GPIO61 = 1   ✓  HIGH (пришёл через провод)
```

При подключённом проводе GPIO61 должен читать то же что выдаёт GPIO60.

### Шаг 4. Выставляем LOW и проверяем

```bash
$ sudo busybox devmem 0x1304007c w 0x5F135D00
$ sudo busybox devmem 0x1304011c
0x0720BDF1
# бит 28 и 29 стали нулями
```

### Восстанавливаем исходное состояние

```bash
# Возвращаем GPIO60 в режим вход (DOEN = 0x01) (DOEN==0x01000101)
$ sudo busybox devmem 0x1304003c w $DOEN
# Возвращаем DOUT в исходное (DOUT==0x5F135D5E)
$ sudo busybox devmem 0x1304007c w $DOUT
```

---

## Раздел III: что происходит «под капотом» при этих записях

Когда мы пишем в `/dev/mem` — мы обходим **все** уровни абстракции ядра:

```
  busybox devmem
       │  /dev/mem → физическая память
       │  (никаких блокировок, никаких проверок ядра)
       ▼
  sys_iomux @ 0x13040000
       │
       ▼
  Физический пин меняет состояние
```

Когда же работает Linux GPIO API (следующие модули):

```
  gpioset / gpiod_set_value()
       │
       ▼
  gpiolib (drivers/gpio/gpiolib.c)
       │  блокировки, проверки, уведомления
       ▼
  chip->set() — коллбэк pinctrl-драйвера
       │
       ▼
  starfive jh7110 driver
  drivers/pinctrl/starfive/pinctrl-starfive-jh7110-sys.c
       │  те же самые writel() в те же самые регистры
       ▼
  sys_iomux @ 0x13040000
       │
       ▼
  Физический пин меняет состояние
```

Итог один и тот же — биты в тех же регистрах. Разница в том, что через API:
- ядро знает кто «владеет» пином и не даст двум драйверам конфликтовать
- изменения видны в `/sys/kernel/debug/pinctrl/`
- всё корректно работает с DT (device tree) и pinctrl

---

## Дополнительные функции пина

[SYS IOMUX CFG SAIF SYSCFG 288 Register Description](https://doc-en.rvspace.org/JH7110/TRM/JH7110_TRM/sys_iomux_cfg.html#sys_iomux_cfg__table_htq_t3b_xsb)

Каждый пин имеет pad-конфигурацию (смещение `0x120`, отдельный регистр на пин):

**Pull-up / Pull-down** — внутренние подтягивающие резисторы. Определяют
состояние пина когда он ни к чему не подключён (floating). Без pull-up/down
состояние floating пина непредсказуемо — именно поэтому GPIO61 мог читать
0 или 1 в разных экспериментах до подключения провода.

**Drive strength** — сила тока на выходе. JH7110 поддерживает 2, 4, 8, 12 мА.
По умолчанию — 2 мА. Для LED или длинных линий нужно больше.

**Schmitt trigger** — гистерезис на входе. Убирает дребезг на входном сигнале.

```bash
# Посмотреть pad-конфигурацию GPIO60 и GPIO61
$ sudo cat /sys/kernel/debug/pinctrl/13040000.pinctrl/pinconf-pins \
    | grep -E "pin 6[01]"
pin 60 (GPIO60): input bias pull up (1 ohms), output drive strength (2 mA), input enabled, slew rate (0x0) (0x09)
pin 61 (GPIO61): input bias disabled, output drive strength (2 mA), input enabled, slew rate (0x0) (0x01)
```

---

## Прерывания от GPIO (кратко)

GPIO-линия может генерировать прерывание при изменении уровня. JH7110
поддерживает пять режимов (конфигурируются через `GPIOIEV`, `GPIOIBE`, `GPIOIS`):

```
  EDGE_RISE    — фронт (0→1)
  EDGE_FALL    — спад  (1→0)
  EDGE_BOTH    — любое изменение
  LEVEL_HIGH   — пока HIGH
  LEVEL_LOW    — пока LOW
```

Настраивается через `request_irq()` / `gpiod_to_irq()`. Прерывания от GPIO
проходят через PLIC → Linux IRQ. Детальнее планируется в будущих уроках.

---

В следующем модуле мы изучим `gpio-mockup` — виртуальный GPIO-контроллер,
который позволяет тренироваться с GPIO API без реального железа.

---

← [Назад (README)](README.md) · [На главную](../INDEX.md) · [Следующий →](module2-virtual-gpio.md)
