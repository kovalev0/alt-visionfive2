# module3 · Legacy-интерфейс `/sys/class/gpio`

← [Назад](module2-virtual-gpio.md) · [На главную](../INDEX.md) · [Следующий →](module4-driver-gpio.md)

> **Статус:** deprecated с ядра 4.8, в новом коде не использовать — только для понимания legacy.

---

## Как устроен интерфейс

`/sys/class/gpio` — управление GPIO через файловую систему:

```
$ echo 60  | sudo tee /sys/class/gpio/export        # сообщаем ядру: хотим управлять GPIO60
$ echo out | sudo tee /sys/class/gpio/gpio60/direction
$ echo 1   | sudo tee /sys/class/gpio/gpio60/value
$ echo 60  | sudo tee /sys/class/gpio/unexport     # освобождаем
```

Номера — глобальные номера ядра. GPIO60 → 60, GPIO61 → 61.

```
  Userspace (echo/cat, shell-скрипты)
       │  /sys/class/gpio/
       ▼
  gpiolib sysfs (drivers/gpio/gpiolib-sysfs.c)
       │
       ▼
  pinctrl-starfive-jh7110-sys.c → writel() → 0x13040000
```

---

## Проверяем доступность

```bash
$ sudo ls /sys/class/gpio/
export  gpiochip0  gpiochip64  unexport

$ sudo cat /sys/class/gpio/gpiochip0/label
13040000.pinctrl

$ sudo cat /sys/class/gpio/gpiochip0/ngpio
64
```

Проверяем что GPIO60 и GPIO61 свободны:

```bash
$ sudo cat /sys/kernel/debug/pinctrl/13040000.pinctrl/pinmux-pins | grep -E "pin 6[01]"
pin 60 (GPIO60): (MUX UNCLAIMED) (GPIO UNCLAIMED)
pin 61 (GPIO61): (MUX UNCLAIMED) (GPIO UNCLAIMED)
```

---

## Эксперимент: loopback GPIO60 → GPIO61

Провод dupont между пин 37 (GPIO60) и пин 38 (GPIO61).

### Экспортируем и настраиваем

```bash
$ echo 60 | sudo tee /sys/class/gpio/export
$ echo 61 | sudo tee /sys/class/gpio/export

$ echo out | sudo tee /sys/class/gpio/gpio60/direction
$ echo in  | sudo tee /sys/class/gpio/gpio61/direction
```

Что появилось в директории пина:

```bash
$ sudo ls /sys/class/gpio/gpio60/
active_low  device  direction  edge  power  subsystem  uevent  value
```

| Файл | Назначение |
|---|---|
| `direction` | `in` / `out` |
| `value` | `0` / `1` — читать и писать |
| `edge` | `none` / `rising` / `falling` / `both` — для poll() |
| `active_low` | `1` — инвертировать логику |

### Читаем и пишем

```bash
$ echo 1 | sudo tee /sys/class/gpio/gpio60/value
$ sudo cat /sys/class/gpio/gpio61/value
1

$ echo 0 | sudo tee /sys/class/gpio/gpio60/value
$ sudo cat /sys/class/gpio/gpio61/value
0
```

### Смотрим что произошло в регистрах

sysfs сделал те же записи что мы делали вручную в module1:

```bash
$ sudo busybox devmem 0x1304003c
0x01000100
# GPIO60 DOEN = 0x00 = GPOEN_ENABLE → выход ✓

$ sudo busybox devmem 0x1304007c
0x5F130000
# GPIO60 DOUT = 0x00 = LOW ✓
```

### Освобождаем

```bash
$ echo 60 | sudo tee /sys/class/gpio/unexport
$ echo 61 | sudo tee /sys/class/gpio/unexport
```

---

## Почему sysfs deprecated

| Проблема sysfs | Решение в libgpiod (module2) |
|---|---|
| Нет атомарного запроса N пинов | один запрос на N пинов сразу |
| Нет owner: кто держит пин — неизвестно | consumer-строка видна в `gpioinfo` |
| `poll()` — нестандартное использование файлов | события через `read()` |
| Нет timestamp событий | timestamp в наносекундах |
| Утечка если процесс умер без `unexport` | fd закрывается автоматически |
| Гонка между export и configure | атомарная настройка при запросе |

```bash
$ dmesg | grep -i "GPIO.*deprecated"
[...] gpio gpiochip0: Static allocation of GPIO base is deprecated, use dynamic allocation
```

---

← [Назад](module2-virtual-gpio.md) · [На главную](../INDEX.md) · [Следующий →](module4-driver-gpio.md)
