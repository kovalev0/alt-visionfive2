# module5 · Device Tree: синтаксис, компиляция, реальные узлы

← [Назад](module4-uboot.md) · [На главную](../INDEX.md) · [Урок 04 →](../lab04-kernel-module/README.md)

> **Kernel docs:** [kernel.org/doc/html/latest/devicetree/usage-model.html](https://www.kernel.org/doc/html/latest/devicetree/usage-model.html)

> **DTS-файлы платы** (в исходниках ядра):
> `arch/riscv/boot/dts/starfive/jh7110-starfive-visionfive-2.dtsi`
> `arch/riscv/boot/dts/starfive/jh7110.dtsi`
> [elixir.bootlin.com/linux/v6.12/source/arch/riscv/boot/dts/starfive](https://elixir.bootlin.com/linux/v6.12/source/arch/riscv/boot/dts/starfive)

---

## Зачем нужен Device Tree

На x86 ядро обнаруживает периферию через стандартные шины (PCI, ACPI).
На встраиваемых платформах (RISC-V, ARM) большинство периферии не самообнаруживается —
GPIO, I2C, SPI висят на статически адресованных шинах.

Device Tree (DT) — это структура данных описывающая аппаратуру платы.
Она компилируется в бинарный формат DTB (Device Tree Blob) и передаётся
ядру загрузчиком (U-Boot помещает адрес DTB в регистр `a1`).

```
  jh7110.dtsi               — описание SoC (все возможные блоки)
       +
  jh7110-starfive-visionfive-2.dtsi  — специфика этой платы
       +
  jh7110-starfive-visionfive-2-v1.3b.dts  — ревизия 1.3b

       │  dtc (Device Tree Compiler)
       ▼
  jh7110-starfive-visionfive-2-v1.3b.dtb  — бинарный блоб

       │  U-Boot передаёт адрес DTB в a1
       ▼
  Linux kernel парсит DTB, создаёт платформенные устройства
```

Важная деталь: в финальный DTB попадает **результат слияния** всех
трёх уровней. Значение свойства из `jh7110.dtsi` может быть
переопределено в `visionfive-2.dtsi` или в `.dts`. Именно поэтому
`status = "disabled"` из SoC-файла превращается в `"okay"` в
скомпилированном DTB для конкретной платы.

---

## Синтаксис DTS

DTS (Device Tree Source) — текстовый формат. Основные понятия:

```dts
/dts-v1/;                   /* версия формата */

/ {                          /* корневой узел */
    #address-cells = <1>;    /* сколько 32-бит ячеек в адресе */
    #size-cells = <1>;       /* сколько 32-бит ячеек в размере */

    compatible = "starfive,visionfive-2-v1.3b", "starfive,jh7110";

    cpus {
        #address-cells = <1>;
        #size-cells = <0>;

        cpu@0 {              /* узел с адресом 0 */
            compatible = "sifive,u74-mc", "riscv";
            reg = <0>;       /* hart ID */
            device_type = "cpu";
        };
    };

    soc {                    /* узел SoC — контейнер */
        #address-cells = <2>;   /* 64-бит адреса: 2 ячейки по 32 бит */
        #size-cells = <2>;
        ranges;              /* адреса дочерних узлов = адреса родителя */

        uart0: serial@10000000 {    /* имя@адрес */
            compatible = "starfive,jh7110-uart", "snps,dw-apb-uart";
            reg = <0x0 0x10000000 0x0 0x10000>;  /* адрес и размер */
            clocks = <&syscrg JH7110_SYSCLK_UART0_CORE>,
                     <&syscrg JH7110_SYSCLK_UART0_APB>;
            clock-names = "baudclk", "apb_pclk";
            interrupts = <32>;
            status = "okay";
        };
    };
};
```

### Ключевые свойства

**`compatible`** — самое важное свойство. Ядро ищет драйвер по этой строке.
Формат: `"производитель,модель"`. Может быть список от специфичного к общему —
ядро пробует каждую строку по порядку. Так `"starfive,jh7110-uart"` даёт
возможность подключить специфичный для JH7110 драйвер, а `"snps,dw-apb-uart"`
— универсальный Synopsys DesignWare, если специфичного нет.

**`reg`** — физический адрес и размер в памяти. Интерпретируется с учётом
`#address-cells` и `#size-cells` родительского узла.

**`status`** — `"okay"` означает что устройство активно и для него нужно
загружать драйвер. `"disabled"` — устройство есть в SoC но не используется
на этой плате.

**`interrupts`** — номер прерывания. Интерпретируется совместно с
`interrupt-parent` — ссылкой на контроллер прерываний (PLIC).

---

## Реальные узлы: GPIO

Из `arch/riscv/boot/dts/starfive/jh7110.dtsi` (символьные имена),
скомпилированный DTB — в `/proc/device-tree`:

```dts
sysgpio: pinctrl@13040000 {
    compatible = "starfive,jh7110-sys-pinctrl";
    reg = <0x0 0x13040000 0x0 0x10000>;
    clocks = <&syscrg JH7110_SYSCLK_IOMUX_APB>;
    resets = <&syscrg JH7110_SYSRST_IOMUX_APB>;
    interrupts = <86>;
    interrupt-controller;
    #interrupt-cells = <2>;
    gpio-controller;
    #gpio-cells = <2>;
};
```

Разбор:
- `reg = <0x0 0x13040000 0x0 0x10000>` — адрес `0x13040000`, размер `0x10000`
  (совпадает с `/proc/iomem`: `13040000-1304ffff : 13040000.pinctrl`)
- `gpio-controller` — этот узел является контроллером GPIO
- `#gpio-cells = <2>` — ссылка на GPIO требует двух ячеек: номер линии + флаги
- `interrupt-controller` — GPIO может быть источником прерываний
- `clocks`, `resets` — ссылки на Clock/Reset контроллеры (CCF)

---

## Реальные узлы: I2C

В `jh7110.dtsi` устройство описано со `status = "disabled"` — так принято
для SoC-файлов, где перечислены все возможные блоки, но не все из них
задействованы на конкретной плате. В `jh7110-starfive-visionfive-2.dtsi`
I2C0 переопределяется: добавляются pinctrl и `status = "okay"`.
В скомпилированном DTB на работающей плате узел выглядит так
(`dtc -I fs /proc/device-tree`):

```dts
i2c@10030000 {
    pinctrl-names = "default";
    pinctrl-0 = <&i2c0_0>;
    clock-names = "ref";
    i2c-sda-hold-time-ns = <0x12c>;
    i2c-scl-falling-time-ns = <0x1fe>;
    i2c-sda-falling-time-ns = <0x1fe>;
    resets = <&syscrg JH7110_SYSRST_I2C0_APB>;
    interrupts = <35>;
    clocks = <&syscrg JH7110_SYSCLK_I2C0_APB>;
    clock-frequency = <100000>;
    compatible = "snps,designware-i2c";
    status = "okay";
    reg = <0x0 0x10030000 0x0 0x10000>;
};
```

Узел `i2c0_0` — это группа pinmux-пинов из `pinctrl@13040000`, разобранная
в module4. Именно через неё драйвер получает GPIO57/GPIO58 под SCL/SDA.

Пример того, как в дополнительном DTS-файле к существующей шине I2C0
добавляется дочернее устройство — аудиокодек WM8960
(из `jh7110-starfive-visionfive-2-wm8960.dts`):

```dts
/* Дополнительный DTS для конфигурации с аудиокодеком */
&i2c0 {
    wm8960: codec@1a {
        compatible = "wlf,wm8960";
        reg = <0x1a>;
        wlf,shared-lrclk;
        #sound-dai-cells = <0>;
    };
};
```

Синтаксис `&i2c0 { ... }` — это **переопределение** (overlay-style):
берётся уже существующий узел по метке `i2c0` и в него добавляются
новые свойства или дочерние узлы. Компилятор сливает это с оригиналом.

---

## Реальные узлы: SPI

Аналогично I2C — в `jh7110.dtsi` SPI0 описан с `status = "disabled"`,
а в `jh7110-starfive-visionfive-2.dtsi` переопределяется. В итоговом
скомпилированном DTB на работающей плате:

```dts
spi@10060000 {
    arm,primecell-periphid = <0x41022>;
    pinctrl-names = "default";
    pinctrl-0 = <&spi0_0>;
    num-cs = <1>;
    clock-names = "sspclk", "apb_pclk";
    resets = <&syscrg JH7110_SYSRST_SPI0_APB>;
    interrupts = <38>;
    clocks = <&syscrg JH7110_SYSCLK_SPI0_APB>,
             <&syscrg JH7110_SYSCLK_SPI0_APB>;
    compatible = "arm,pl022", "arm,primecell";
    status = "okay";
    reg = <0x0 0x10060000 0x0 0x10000>;

    spi@0 {
        spi-max-frequency = <10000000>;  /* 0x989680 */
        compatible = "rohm,dh2228fv";
        reg = <0>;                       /* chip select 0 */
    };
};
```

Дочерний узел `spi@0` с `compatible = "rohm,dh2228fv"` — это `spidev`:
псевдосовместимость, говорящая ядру создать `/dev/spidev0.0` для
доступа из userspace. `spi-max-frequency` задаёт максимальную тактовую
частоту шины при работе с этим устройством.

---

## Практика: исследование Device Tree в работающей системе

```bash
# Device Tree экспортируется ядром через sysfs
$ ls /sys/firmware/devicetree/base/

# Имя платы (две строки — список compatible корневого узла)
$ cat /sys/firmware/devicetree/base/compatible | tr '\0' '\n'
# вывод: starfive,visionfive-2-v1.3b
#        starfive,jh7110

# Посмотреть узел GPIO
$ ls /sys/firmware/devicetree/base/soc/pinctrl@13040000/
```

### Декомпиляция DTB обратно в DTS

```bash
# DTB файл лежит на загрузочном разделе
$ sudo file /boot/BOOT/jh7110-starfive-visionfive-2-v1.3b.dtb
/boot/BOOT/jh7110-starfive-visionfive-2-v1.3b.dtb: Device Tree Blob version 17, size=58144, boot CPU=0, string block size=5096, DT structure block size=52992

# Декомпилировать в читаемый DTS
$ sudo dtc -I dtb -O dts -o /tmp/board.dts  /boot/BOOT/jh7110-starfive-visionfive-2-v1.3b.dtb

# Посмотреть узлы GPIO, I2C, SPI
$ grep -A 10 "pinctrl@13040000" /tmp/board.dts
$ grep -A 10 "i2c@" /tmp/board.dts
$ grep -A 10 "spi@" /tmp/board.dts
```

Либо — напрямую из работающего ядра, без DTB-файла:

```bash
$ dtc -I fs /proc/device-tree 2>/dev/null | grep -A 15 "i2c@10030000"
```

### Поиск устройств по compatible

```bash
# Все узлы с compatible в /proc/device-tree
$ find /sys/firmware/devicetree/base -name compatible \
    -exec sh -c 'echo "=== $1 ==="; cat "$1" | tr "\0" "\n"' _ {} \; \
    2>/dev/null | head -80
```

### Связь DTS → платформенное устройство → драйвер

```bash
# Платформенные устройства созданные из DTS
$ ls /sys/bus/platform/devices/ | grep -E "i2c|spi|gpio|pinctrl"
10030000.i2c
10050000.i2c
12050000.i2c
12060000.i2c
13010000.spi
13040000.pinctrl
17020000.pinctrl
gpio-restart

# Для каждого устройства — какой драйвер привязан
$ for d in i2c spi pinctrl; do
    echo "=== $d ==="
    ls /sys/bus/platform/devices/ | grep $d | while read dev; do
        driver=$(readlink /sys/bus/platform/devices/$dev/driver 2>/dev/null)
        echo "  $dev -> ${driver##*/}"
    done
done
=== i2c ===
  10030000.i2c -> i2c_designware
  10050000.i2c -> i2c_designware
  12050000.i2c -> i2c_designware
  12060000.i2c -> i2c_designware
=== spi ===
  13010000.spi -> cadence-qspi
=== pinctrl ===
  13040000.pinctrl -> starfive-jh7110-sys-pinctrl
  17020000.pinctrl -> starfive-jh7110-aon-pinctrl
```



---

← [Назад](module4-uboot.md) · [На главную](../INDEX.md) · [Урок 04 →](../lab04-kernel-module/README.md)
