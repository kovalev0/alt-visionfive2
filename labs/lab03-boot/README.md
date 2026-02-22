# Урок 03: Загрузка платы

← [Урок 02](../lab02-riscv/README.md) · [На главную](../INDEX.md)

---

## Цель урока

Понять что происходит с момента подачи питания до появления приглашения
shell. Каждый компонент цепочки загрузки — BootROM, SPL, OpenSBI, U-Boot,
ядро Linux — существует по конкретной причине и решает конкретную задачу.

Без понимания этой цепочки непонятно откуда берётся Device Tree, почему
ядро не может напрямую общаться с железом без OpenSBI, и как U-Boot
передаёт управление ядру. Эти знания потребуются при отладке загрузки
и при написании драйверов, которые зависят от корректно настроенного DTS.

---

## Модули

| Модуль | Тема |
|--------|------|
| [module1-boot-sequence.md](module1-boot-sequence.md) | От подачи питания до ядра: BootROM → SPL → OpenSBI → U-Boot → Linux |
| [module2-privilege-levels.md](module2-privilege-levels.md) | Уровни привилегий RISC-V: M/S/U и переходы между ними |
| [module3-opensbi.md](module3-opensbi.md) | OpenSBI: SBI-интерфейс, вызовы, роль прошивки |
| [module4-uboot.md](module4-uboot.md) | U-Boot: extlinux.conf, переменные окружения, FIT-образ |
| [module5-devicetree.md](module5-devicetree.md) | Device Tree: синтаксис, компиляция, реальные узлы GPIO/I2C/SPI |

---

## Что понадобится

- Плата с ALT Linux, доступ через UART-консоль (не SSH — нужен вывод U-Boot)

---

[Начнём →](module1-boot-sequence.md)
