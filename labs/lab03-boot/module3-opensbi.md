# module3 · OpenSBI: SBI-интерфейс и роль прошивки

← [Назад](module2-privilege-levels.md) · [На главную](../INDEX.md) · [Следующий →](module4-uboot.md)

> **Исходники OpenSBI:** [github.com/riscv-software-src/opensbi](https://github.com/riscv-software-src/opensbi)

> **SBI Specification:** [github.com/riscv-non-isa/riscv-sbi-doc](https://github.com/riscv-non-isa/riscv-sbi-doc)

> **Kernel source:** `arch/riscv/kernel/sbi.c`
> [elixir.bootlin.com/linux/v6.12/source/arch/riscv/kernel/sbi.c](https://elixir.bootlin.com/linux/v6.12/source/arch/riscv/kernel/sbi.c)

---

## Что такое SBI и зачем он нужен

SBI (Supervisor Binary Interface) — это стандартизированный интерфейс между
S-mode (ядро Linux) и M-mode (прошивка). По аналогии с тем как syscall —
интерфейс между userspace и ядром, SBI — интерфейс между ядром и прошивкой.

```
  Userspace (U-mode)
       │  ecall → syscall
       ▼
  Linux Kernel (S-mode)
       │  ecall → SBI call
       ▼
  OpenSBI (M-mode)
       │  прямой доступ к железу
       ▼
  Hardware
```

Без SBI ядро Linux было бы вынуждено либо работать в M-mode (небезопасно),
либо каждый производитель SoC реализовывал бы свой способ доступа к
низкоуровневым функциям. SBI стандартизирует этот интерфейс — ядро Linux
для RISC-V не знает ничего о конкретном SoC на уровне M-mode.

---

## Структура SBI-вызова

SBI-вызов из S-mode выглядит точно так же как syscall из U-mode — через
`ecall`. Разница в регистрах:

```
  Регистр   Роль при SBI-вызове
  ─────────────────────────────────────────────────────
  a7        Extension ID (EID) — номер расширения SBI
  a6        Function ID (FID) — номер функции внутри расширения
  a0–a5     Аргументы функции
  ─────────────────────────────────────────────────────
  После возврата:
  a0        Error code (0 = успех, отрицательное = ошибка)
  a1        Возвращаемое значение
```

Пример из [`arch/riscv/kernel/sbi_ecall.c`](https://elixir.bootlin.com/linux/v6.12/source/arch/riscv/kernel/sbi_ecall.c#L20):

```c
struct sbiret __sbi_ecall(unsigned long arg0, unsigned long arg1,
			  unsigned long arg2, unsigned long arg3,
			  unsigned long arg4, unsigned long arg5,
			  int fid, int ext)
{
	struct sbiret ret;

	trace_sbi_call(ext, fid);

	register uintptr_t a0 asm ("a0") = (uintptr_t)(arg0);
	register uintptr_t a1 asm ("a1") = (uintptr_t)(arg1);
	register uintptr_t a2 asm ("a2") = (uintptr_t)(arg2);
	register uintptr_t a3 asm ("a3") = (uintptr_t)(arg3);
	register uintptr_t a4 asm ("a4") = (uintptr_t)(arg4);
	register uintptr_t a5 asm ("a5") = (uintptr_t)(arg5);
	register uintptr_t a6 asm ("a6") = (uintptr_t)(fid);
	register uintptr_t a7 asm ("a7") = (uintptr_t)(ext);
	asm volatile ("ecall"
		       : "+r" (a0), "+r" (a1)
		       : "r" (a2), "r" (a3), "r" (a4), "r" (a5), "r" (a6), "r" (a7)
		       : "memory");
	ret.error = a0;
	ret.value = a1;

	trace_sbi_return(ext, ret.error, ret.value);

	return ret;
}
```

---

## Основные SBI-расширения

**SBI Spec:** [github.com/riscv-non-isa/riscv-sbi-doc](https://github.com/riscv-non-isa/riscv-sbi-doc)

| EID (hex) | Расширение | Назначение |
|-----------|-----------|------------|
| `0x10` | Base | Обнаружение поддерживаемых расширений, версия SBI |
| `0x54494D45` | TIME | Программирование таймера (`sbi_set_timer`) |
| `0x735049` | IPI | Межпроцессорные прерывания (`sbi_send_ipi`) |
| `0x52464E43` | RFNC | Remote Fence — синхронизация TLB на других ядрах |
| `0x48534D` | HSM | Hart State Management — управление ядрами (старт/стоп) |
| `0x53525354` | SRST | System Reset — перезагрузка, выключение |
| `0x504D55` | PMU | Performance Monitoring Unit |

### Как Linux использует SBI_TIME

При каждом тике таймера Linux вызывает `sbi_set_timer()` чтобы запрограммировать
следующее прерывание таймера. На RISC-V нет прямого доступа к регистру
`mtimecmp` из S-mode — только через SBI.

```bash
# Счётчик прерываний таймера (per-CPU)
$ sudo cat /proc/interrupts | grep -i "timer\|local"
```

### Как Linux использует SBI_HSM для SMP

При загрузке ядро стартует только на одном hart (CPU core). Остальные
переводятся в STOPPED-состояние OpenSBI. Ядро запускает дополнительные
core через `sbi_hart_start()`:

```c
/* arch/riscv/kernel/cpu_ops_sbi.c */
static int sbi_cpu_start(unsigned int cpuid, struct task_struct *tidle)
{
    ...
    return sbi_hsm_hart_start(hartid, boot_addr, hsm_data);
}
```

```bash
# Убедиться что все 4 ядра запущены
$ cat /proc/cpuinfo | grep "processor"
# ожидаемо: processor 0, 1, 2, 3
```

---

## Что OpenSBI делает при старте

Исходник: `firmware/fw_base.S` в репозитории OpenSBI.

```
  1. Выбор boot hart (master hart = S7 или первый U74)
  2. Обнуление BSS-секции прошивки
  3. Инициализация платформо-зависимого кода (platform.c для JH7110)
     - настройка клоков и GPIO
     - инициализация UART для вывода сообщений
  4. Настройка PMP (Physical Memory Protection):
     - область памяти OpenSBI — только M-mode
     - остальная RAM — S/U mode
  5. Заполнение medeleg / mideleg (делегирование в S-mode)
  6. Установка mtvec — обработчик trap для M-mode
  7. Подготовка scratch-пространства для каждого hart
  8. Вызов sbi_hart_switch_mode() → mret в S-mode
     mepc = адрес U-Boot (или ядра если без U-Boot)
```
---

## Практика: SBI-вызовы из userspace

Прямого API для SBI-вызовов из userspace нет (они из S-mode), но ядро
экспортирует информацию о поддерживаемых расширениях:

```bash
# Версия SBI и поддерживаемые расширения
$ dmesg | grep "SBI specification\|SBI impl\|SBI v"
[    0.000000] SBI specification v2.0 detected
[    0.000000] SBI implementation ID=0x1 Version=0x10006
```

---

← [Назад](module2-privilege-levels.md) · [На главную](../INDEX.md) · [Следующий →](module4-uboot.md)
