# module4 · Системные вызовы: ecall, таблица, strace

← [Назад](module3-assembly.md) · [На главную](../INDEX.md)

> **Kernel source:** `arch/riscv/kernel/syscall_table.c`
> [elixir.bootlin.com/linux/v6.12/source/arch/riscv/kernel/syscall_table.c](https://elixir.bootlin.com/linux/v6.12/source/arch/riscv/kernel/syscall_table.c)

> **Таблица номеров:** `include/uapi/asm-generic/unistd.h`
> [elixir.bootlin.com/linux/v6.12/source/include/uapi/asm-generic/unistd.h](https://elixir.bootlin.com/linux/v6.12/source/include/uapi/asm-generic/unistd.h)

---

## Что такое системный вызов и зачем нужен ecall

Userspace-программа не может напрямую записывать в файлы, открывать сокеты
или обращаться к устройствам. В U-mode (User mode) привилегии ограничены.
Любое обращение к ресурсам системы — через ядро.

Переход от U-mode к S-mode (где работает Linux) называется **trap**.
В RISC-V trap из userspace вызывается инструкцией `ecall` (Environment CALL).

```
  Userspace (U-mode)                Kernel (S-mode)
  ─────────────────                 ───────────────

  a7 = __NR_write (64)
  a0 = fd
  a1 = buf
  a2 = len
  ecall ──────────────────────────► CPU сохраняет PC → sepc
                                    scause = 8 (Environment call from U-mode)
                                    CPU переходит по адресу в stvec
                                          │
                                          ▼
                                    handle_exception()
                                    (arch/riscv/kernel/entry.S)
                                          │
                                          ▼
                                    do_syscall()
                                          │
                                          ▼
                                    sys_write()
                                    (fs/read_write.c)
                                          │
                                          ▼
                                    результат → a0
                                    sret ────────────────────────► возврат в userspace
                                                                   PC = sepc + 4
```

`sret` (Supervisor Return) восстанавливает режим и продолжает программу
со следующей инструкции после `ecall` (sepc + 4).

---

## Соглашение о системных вызовах в RISC-V Linux

RISC-V использует унифицированную (generic) таблицу системных вызовов,
общую с несколькими другими архитектурами. Это отличие от x86_64, который
имеет свою таблицу.

```
  Регистр   Роль при syscall
  ──────────────────────────────────────────────────────
  a7        Номер системного вызова (__NR_xxx)
  a0        Аргумент 1 / Возвращаемое значение
  a1        Аргумент 2
  a2        Аргумент 3
  a3        Аргумент 4
  a4        Аргумент 5
  a5        Аргумент 6
  ──────────────────────────────────────────────────────
```

После возврата из syscall:
- `a0` содержит результат (≥ 0) или отрицательный код ошибки (например
  `-ENOENT` = -2)
- libc-обёртка (`write()`, `read()` и т.д.) проверяет знак, записывает
  `errno` и возвращает -1

---

## Часто используемые номера syscall

Номера соответствуют `include/uapi/asm-generic/unistd.h`:

```
  Номер   Имя           Прототип (упрощённо)
  ────────────────────────────────────────────────────────────────
  17      getcwd        long getcwd(char *buf, size_t size)
  29      ioctl         long ioctl(int fd, unsigned long request, ...)
  56      openat        long openat(int dfd, const char *fname, int flags, ...)
  57      close         long close(int fd)
  63      read          long read(int fd, char *buf, size_t count)
  64      write         long write(int fd, const char *buf, size_t count)
  93      exit          void exit(int status)
  94      exit_group    void exit_group(int status)
  172     getpid        pid_t getpid(void)
  173     getppid       pid_t getppid(void)
  174     getuid        uid_t getuid(void)
  214     brk           long brk(unsigned long addr)
  215     munmap        long munmap(unsigned long addr, size_t len)
  222     mmap          long mmap(...)
  278     getrandom     long getrandom(char *buf, size_t count, unsigned int flags)
```

Полный список (файл include/uapi/asm-generic/unistd.h из пакета kernel-headers-modules):

```bash
$ grep '#define __NR_' $(rpm -ql kernel-headers-modules-6.12 | grep "asm-generic/unistd.h") | less
```

---

## Практика: syscall вручную через ассемблер

```asm
# getpid.s — вызов getpid(), выход с PID в качестве exit code
    .section .text
    .global _start

_start:
    li   a7, 172        # __NR_getpid = 172
    ecall               # a0 = PID текущего процесса

    mv   a1, a0         # сохранить PID
    li   a7, 93         # __NR_exit
    mv   a0, a1         # exit(PID)
    ecall
```

```bash
$ as -o getpid.o getpid.s
$ ld -o getpid getpid.o
$ ./getpid; echo $?
# вывод: PID текущего процесса
```

---

## strace: наблюдение системных вызовов

`strace` перехватывает все `ecall` которые делает процесс и выводит их
в читаемом виде.

```bash
$ strace ./hello
```

Ожидаемый вывод:

```
execve("./hello", ["./hello"], 0x... /* N vars */) = 0
write(1, "Hello, RISC-V!\n", 15)        = 15
exit(0)                                  = ?
+++ exited with 0 +++
```

Без C runtime видно только три syscall: `execve` (запуск процесса, делается
ядром при `exec`), `write`, `exit`.

```bash
# Только имена syscall и статистика
$ strace -c ls

# Фильтрация: только openat/read/write
$ strace -e trace=openat,read,write ls
```

Вывод `-c` показывает таблицу: сколько раз какой syscall вызывался, сколько
времени заняло, сколько ошибок.

---

## Разница между exit и exit_group

```bash
$ strace ./hello 2>&1 | grep exit
exit(0)                                 = ?
+++ exited with 0 +++
```

Если программа скомпилирована с C runtime (glibc), последним будет
`exit_group(0)`, а не `exit(0)`. `exit_group` завершает все потоки
процесса, `exit` — только текущий поток. В многопоточных программах
`pthread_exit` использует `exit`, а `return` из `main` — `exit_group`:

```bash
# Простейшая программа на Си
$ echo "int main(){}" > main.c
$ gcc main.c -o main
$ strace ./main 2>&1 | grep exit
exit_group(0)                           = ?
+++ exited with 0 +++
```

---

## Путь через исходники ядра при ecall

```
  ecall в userspace
       │
       ▼
  arch/riscv/kernel/entry.S :: handle_exception
       │  (сохранение всех регистров в struct pt_regs на стеке ядра)
       │
       ▼
  arch/riscv/kernel/traps.c :: do_trap_ecall_u
       │
       ▼
  arch/riscv/kernel/syscall_table.c
       │  (индекс по a7 в таблице sys_call_table[])
       │
       ▼
  конкретный обработчик, например fs/read_write.c :: ksys_write
       │
       ▼
  результат записывается в pt_regs.a0
       │
       ▼
  arch/riscv/kernel/entry.S :: ret_from_exception
       │  (восстановление регистров из pt_regs)
       ▼
  sret → возврат в userspace
```

Проследить этот путь через ftrace:

```bash
# Включить трассировку событий syscall и запись завершения выхода
$ sudo sh -c 'echo 1 > /sys/kernel/debug/tracing/events/syscalls/enable'
$ sudo sh -c 'echo 1 > /sys/kernel/debug/tracing/events/syscalls/sys_exit_exit/enable'
$ sudo sh -c 'echo 1 > /sys/kernel/debug/tracing/tracing_on'

$ ./hello

$ sudo sh -c 'echo 0 > /sys/kernel/debug/tracing/tracing_on'
# Вывести начало записи и ответ ядра
$ sudo cat /sys/kernel/debug/tracing/trace | grep -A 1 "hello.*sys_write"
           hello-1800    [003] .....  4704.400223: sys_write(fd: 1, buf: 100d4, count: f)
           hello-1800    [003] .n...  4704.400291: sys_write -> 0xf
           hello-1800    [003] .....  4704.400371: sys_exit(error_code: 0)
```
В выводе видна запись входа в `write` с аргументами и выхода с
результатом. Это первое знакомство с tracing — полноценно будет разбираться
в будущих уроках.

---

← [Назад](module3-assembly.md) · [На главную](../INDEX.md)
