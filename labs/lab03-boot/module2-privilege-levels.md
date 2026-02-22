# module2 · Уровни привилегий RISC-V: M/S/U

← [Назад](module1-boot-sequence.md) · [На главную](../INDEX.md) · [Следующий →](module3-opensbi.md)

> **Спецификация:** RISC-V Privileged Architecture, Volume II
> [riscv.org/specifications](https://riscv.org/specifications/)

---

## Три уровня привилегий

RISC-V определяет три уровня привилегий (privilege levels), также называемых
режимами или кольцами защиты:

```
  ┌────────────────────────────────────────────────────────┐
  │                   M-mode (Machine)                     │
  │  Наивысший уровень привилегий.                         │
  │  Полный доступ к железу, всем CSR, физической памяти.  │
  │  Запускается первым при включении питания.             │
  │  В работающей системе: OpenSBI (резидентная прошивка)  │
  │                                                        │
  │   ┌───────────────────────────────────────────────┐    │
  │   │              S-mode (Supervisor)              │    │
  │   │  Уровень ОС. Доступ к виртуальной памяти      │    │
  │   │  через MMU (satp), управление прерываниями    │    │
  │   │  через PLIC/CLINT. Нет прямого доступа к      │    │
  │   │  аппаратным регистрам M-mode.                 │    │
  │   │  В работающей системе: Linux kernel           │    │
  │   │                                               │    │
  │   │   ┌──────────────────────────────────────┐    │    │
  │   │   │           U-mode (User)              │    │    │
  │   │   │  Наименьшие привилегии. Нет прямого  │    │    │
  │   │   │  доступа к железу. Нет доступа к     │    │    │
  │   │   │  памяти ядра. Системные вызовы через │    │    │
  │   │   │  ecall.                              │    │    │
  │   │   │  В работающей системе: userspace     │    │    │
  │   │   └──────────────────────────────────────┘    │    │
  │   └───────────────────────────────────────────────┘    │
  └────────────────────────────────────────────────────────┘
```

Каждый режим имеет свой набор CSR (Control and Status Registers) -регистров.
Обращение к CSR более привилегированного режима из менее привилегированного
вызывает исключение (Illegal Instruction trap).

---

## Переходы между режимами

```
  Включение питания
       │
       ▼  CPU стартует в M-mode
  M-mode (BootROM, SPL, OpenSBI init)
       │
       │  mret (Machine Return)
       │  Перед mret: mstatus.MPP = 01 (S-mode)
       │              mepc = адрес U-Boot
       ▼
  S-mode (U-Boot, затем Linux kernel)
       │
       │  ecall из U-mode → trap в S-mode
       │  sret (Supervisor Return) → возврат в U-mode
       ▼
  U-mode (пользовательские процессы)
```

**`mret`** — инструкция возврата из M-mode. CPU смотрит на `mstatus.MPP`
(Machine Previous Privilege) чтобы определить в какой режим возвращаться,
и на `mepc` чтобы знать адрес возврата.

**`sret`** — инструкция возврата из S-mode. CPU смотрит на `sstatus.SPP`
(Supervisor Previous Privilege) и `sepc`.

**`ecall`** — переход вверх по привилегиям. Из U-mode — trap в S-mode
(syscall). Из S-mode — trap в M-mode (SBI-вызов к OpenSBI).

---

## Что каждый режим может и не может

### M-mode

- Доступ ко всем CSR: `mstatus`, `mtvec`, `mepc`, `mie`, `mip`, ...
- Прямое чтение/запись любой физической памяти
- Настройка PMP (Physical Memory Protection) — ограничение доступа к
  памяти для нижестоящих режимов
- Приём всех прерываний и исключений (если не делегированы)

### S-mode

- Доступ к S-mode CSR: `sstatus`, `stvec`, `sepc`, `scause`, `satp`, ...
- Управление MMU через `satp` — включение виртуальной памяти
- Делегирование прерываний в U-mode
- Нет доступа к M-mode CSR: попытка вызывает Illegal Instruction

### U-mode

- Только пользовательские инструкции (RV64GC без CSR кроме `cycle`, `time`,
  `instret` — и то если не запрещено ядром)
- Нет привилегированных инструкций (`mret`, `sret`, `sfence.vma`, ...)
- Попытка выполнить привилегированную инструкцию → Illegal Instruction trap

---

## Делегирование прерываний и исключений

По умолчанию все прерывания и исключения обрабатываются в M-mode. Но
OpenSBI делегирует большинство из них в S-mode через регистры `medeleg`
(Machine Exception Delegation) и `mideleg` (Machine Interrupt Delegation).

```
  Типичная конфигурация делегирования от OpenSBI:

  medeleg делегирует в S-mode:
    - Instruction address misaligned
    - Breakpoint
    - Load/Store address misaligned
    - Environment call from U-mode (ecall из userspace → syscall)
    - Instruction/Load/Store page fault (page fault → обрабатывает Linux)

  mideleg делегирует в S-mode:
    - Supervisor software interrupt
    - Supervisor timer interrupt
    - Supervisor external interrupt (внешние прерывания через PLIC)
```

Это означает что Linux (S-mode) обрабатывает системные вызовы и page fault
самостоятельно, не обращаясь каждый раз к OpenSBI.

---

## Практика: наблюдение trap в S-mode

Linux регистрирует свой обработчик trap в `stvec` при старте. Посмотреть
адрес этого обработчика:

```bash
# Адрес stvec напрямую недоступен из userspace,
# но можно найти в System.map
$ sudo grep " handle_exception" /proc/kallsyms
ffffffff80e9b62c T handle_exception
```

```bash
# Количество trap/прерываний по типам
$ sudo less /proc/interrupts
```

```bash
# Статистика исключений через perf (`sudo apt-get install perf`)
$ sudo perf stat -e alignment-faults,emulation-faults ls
 Performance counter stats for 'ls':

                 0      alignment-faults                                                      
                 0      emulation-faults                                                      

       0.012504251 seconds time elapsed

       0.000000000 seconds user
       0.012571000 seconds sys
```

---

## Как это связано с загрузкой VisionFive2

```
  1. BootROM стартует в M-mode
  2. SPL работает в M-mode, инициализирует DRAM
  3. OpenSBI инициализируется в M-mode:
     - настраивает PMP (Physical Memory Protection)
     - заполняет medeleg/mideleg
     - устанавливает mtvec на свой trap handler
  4. OpenSBI выполняет mret в S-mode → управление переходит к U-Boot
  5. U-Boot работает в S-mode
  6. U-Boot загружает ядро и выполняет booti
  7. Ядро стартует в S-mode:
     - устанавливает stvec = handle_exception (arch/riscv/kernel/entry.S)
     - включает MMU (записывает в satp)
     - начинает принимать прерывания
  8. Ядро запускает init в U-mode
```

Важно: ядро Linux **никогда** не переходит в M-mode. Для операций
требующих M-mode (таймер, IPI — inter-processor interrupt) ядро использует
SBI-вызовы к OpenSBI. Это разбирается в следующем модуле.

---

← [Назад](module1-boot-sequence.md) · [На главную](../INDEX.md) · [Следующий →](module3-opensbi.md)
