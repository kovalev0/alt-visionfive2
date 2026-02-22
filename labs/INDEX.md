# Курс: Низкоуровневое программирование Linux на VisionFive2 (RISC-V)

> **Платформа**: StarFive VisionFive2 · **ОС**: ALT Linux · **Ядро**: 6.12

---

## О курсе

Этот курс — практическое погружение в низкоуровневую разработку под Linux
на примере реального RISC-V одноплатного компьютера StarFive VisionFive2.

Каждая тема по возможности раскрывается цепочкой: теория из документации → эксперимент 
в виртуальной среде → эксперимент на железе → наблюдение через инструменты ядра.

```
  Документация          Ядро Linux              Hardware
  (TRM, DTS)    --->   (sysfs, debugfs,   --->  (пины, осциллограф,
                        ftrace, dmesg)           логический анализатор)
```

При прохождении курса предполагается выполнение следующих задач:

- Чтение и анализ TRM и схемотехнической документации платы;
- Сравнительный анализ архитектуры RISC-V и её отличий от x86/ARM;
- Трассировка пути данных от user space до физического пина;
- Разработка, загрузка и отладка модулей ядра Linux;
- Работа с интерфейсами GPIO, I2C, SPI, UART, механизмами DMA и прерываниями;
- Использование ftrace, perf, sysfs, debugfs и procfs в качестве инструментов отладки;
- Анализ осциллограмм и их сопоставление с кодом драйвера;
- Написание USB-gadget и USB host-драйверов;
- Работа с ALSA/ASoC: организация воспроизведения и записи через I2S + WM8960;
- Вывод изображения на HDMI через DRM/KMS API;
- Написание сетевых драйверов и использование XDP/eBPF для обработки пакетов;
- Работа с VFS и реализация простых файловых систем;
- Управление тактированием и питанием через CCF, Regulator и PM Runtime.
---

## Необходимое оборудование и ПО

| Что | Зачем |
|-----|-------|
| StarFive VisionFive2 | Основная плата |
| MicroSD карта ≥ 16 ГБ | Загрузочный носитель |
| USB-UART адаптер (3.3V) или HDMI монитор | Консоль для первичной настройки |
| Осциллограф (любой, 2 канала) | Наблюдение сигналов |
| Соединительные провода (dupont) | Подключение к GPIO |
| ПК с Linux / WSL | SSH |
| Опционально: логический анализатор | I2C / SPI декодирование |
| Опционально: I2C датчик (например BMP280) | Урок по I2C |

---

## Структура репозитория (ожидается добавление новых уроков/тем)

```
labs/
├── INDEX.md                        ← Вы здесь
│
├── setup.md                        ← Подготовка рабочего места
│                                   (Запись образа, вход в консоль)
│
├── lab01-board/                    ← Урок 01: Знакомство с платой
│   ├── README.md                   (обзор урока, навигация)
│   ├── module1-overview.md         Компоненты платы: что физически находится на PCB
│   ├── module2-jh7110.md           SoC JH7110: ядра, шины, карта адресного пространства
│   ├── module3-pinout.md           GPIO40: назначение пинов, альтернативные функции
│   └── module4-pinmux.md           Pinmux и pinctrl: как Linux управляет мультиплексированием

├── lab02-riscv/                    ← Урок 02: Архитектура RISC-V
│   ├── README.md
│   ├── module1-isa.md              ISA: RV64GC, расширения, отличия от x86/ARM
│   ├── module2-registers.md        Регистры, ABI, соглашение о вызовах
│   ├── module3-assembly.md         Первая программа на ассемблере в userspace
│   └── module4-syscalls.md         Системные вызовы: ecall, таблица, strace
```

---

## Методология каждого урока

Каждый модуль, если это возможно, внутри урока следует принципу:

```
  1. ТЕОРИЯ          Объяснение принципа работы со ссылками на TRM
        │
        ▼
  2. ВИРТУАЛЬНАЯ     Эксперимент с эмуляцией hardware: stub/mock/debugfs
     СРЕДА           Наблюдение через dmesg, sysfs, debugfs
        │
        ▼
  3. РЕАЛЬНОЕ        То же самое, но на реальных пинах
     ОБОРУДОВАНИЕ    Наблюдение на осциллографе
        │
        ▼
  4. ТРАССИРОВКА     ftrace / perf: смотрим что происходит внутри ядра
```

---

## Ссылки на документацию

| Документ | Где найти |
|----------|-----------|
| VisionFive 2 Design Schematics | [wiki.rvspace.org](https://doc-en.rvspace.org/VisionFive2/PDF/RV002_V1.3B_20230208.PDF) |
| JH7110 TRM (Technical Reference Manual) | [doc-en.rvspace.org/JH7110/TRM](https://doc-en.rvspace.org/JH7110/TRM/) |
| VisionFive2 Software Technical Reference Manual | [doc-en.rvspace.org](https://doc-en.rvspace.org/VisionFive2/SWTRM/) |
| VisionFive2 Available PDF Documentation | [wiki.rvspace.org](https://wiki.rvspace.org/en/project/Document_Publish_Status) |
| RISC-V ISA Specification | [riscv.org/specifications](https://riscv.org/specifications/) |
| RISC-V Privileged Architecture | [riscv.github.io/riscv-isa-manual/](https://riscv.github.io/riscv-isa-manual/snapshot/privileged/) |
| Linux Kernel Documentation | [kernel.org/doc](https://www.kernel.org/doc/html/latest/) |
| Linux DMAengine API | [kernel.org/doc](https://www.kernel.org/doc/html/latest/driver-api/dmaengine/index.html) |
| Linux DRM/KMS API | [kernel.org/doc](https://www.kernel.org/doc/html/latest/gpu/index.html) |
| ALSA / ASoC Documentation | [kernel.org/doc](https://www.kernel.org/doc/html/latest/sound/index.html) |
| Kernel source (elixir) | [elixir.bootlin.com/linux/v6.12](https://elixir.bootlin.com/linux/v6.12/source) |
| Kernel source (git) | [git.altlinux.org/people/kovalev](https://git.altlinux.org/people/kovalev/packages/kernel-image.git?p=kernel-image.git;a=shortlog;h=refs/heads/alt-JH7110_VisionFive2_6.12.y_devel) |
| ALT Linux Wiki | [altlinux.org/StarFive_VisionFive_v2](https://www.altlinux.org/StarFive_VisionFive_v2) |
| Gentoo Wiki | [gentoo.org/wiki/StarFive_VisionFive_2](https://wiki.gentoo.org/wiki/StarFive_VisionFive_2) |


---

*[Подготовка рабочего места](setup.md)*

*[Урок 01: Знакомство с платой](lab01-board/README.md)*
