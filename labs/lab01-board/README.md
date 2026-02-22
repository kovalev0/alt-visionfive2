# Урок 01: Знакомство с платой VisionFive2

← [Подготовка](../setup.md) · [На главную](../INDEX.md)

---

## Цель урока

Прежде чем писать драйверный код — необходимо чётко понимать, с чем именно
предстоит работать: какой процессор, как организована его память, какая
периферия существует, где каждый блок находится в адресном пространстве
и как Linux об этом узнаёт.

Урок разбит на четыре модуля: от физического вида платы до механизма
переключения функций пинов на уровне ядра.

---

## Модули

| Модуль | Тема |
|--------|------|
| [module1-overview.md](module1-overview.md) | Компоненты платы: что физически находится на PCB |
| [module2-jh7110.md](module2-jh7110.md) | SoC JH7110: ядра, шины, карта адресного пространства |
| [module3-pinout.md](module3-pinout.md) | GPIO40: назначение пинов, альтернативные функции |
| [module4-pinmux.md](module4-pinmux.md) | Pinmux и pinctrl: как Linux управляет мультиплексированием |

---

## Что понадобится

- Плата с запущенным ALT Linux, доступ через UART или SSH
- JH7110 TRM: [doc-en.rvspace.org/JH7110/TRM](https://doc-en.rvspace.org/JH7110/TRM/)
- GPIO Pinout PDF: [doc-en.rvspace.org (PDF)](https://doc-en.rvspace.org/VisionFive2/PDF/VisionFive2_40-Pin_GPIO_Header_UG.pdf)

---

[Начнём →](module1-overview.md)
