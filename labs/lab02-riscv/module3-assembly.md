# module3 · Первая программа на ассемблере в userspace

← [Назад](module2-registers.md) · [На главную](../INDEX.md) · [Следующий →](module4-syscalls.md)

---

## Инструментарий

Весь код компилируется и запускается нативно на плате.

```bash
# Проверить наличие инструментов
$ as --version          # GNU Assembler
$ ld --version          # GNU Linker
$ objdump --version     # для дизассемблирования
```

Для сборки ассемблерных файлов используется два пути:
1. Напрямую через `as` + `ld` — минимально, без C runtime
2. Через `gcc -S` — смотреть что компилятор генерирует из C

---

## Программа 1: минимальный exit

Самая простая программа — сделать системный вызов `exit(0)` и завершиться.
Никаких библиотек, никакого C runtime — только syscall напрямую.

Создать файл `exit.s`:

```asm
# exit.s — minimal program: exit(0)
#
# Calling convention for syscalls (RISC-V Linux):
#   a7 = syscall number
#   a0 = first argument
#   ecall → kernel

    .section .text
    .global _start

_start:
    li   a7, 93        # syscall number: __NR_exit = 93
    li   a0, 0         # exit code = 0
    ecall              # invoke kernel
```

Сборка и запуск:

```bash
$ as -o exit.o exit.s
$ ld -o exit exit.o
$ ./exit
$ echo $?               # вывод: 0
```

```bash
# Посмотреть что получилось
$ objdump -d exit
```

Ожидаемый вывод дизассемблера:

```
exit:     формат файла elf64-littleriscv


Дизассемблирование раздела .text:

00000000000100b0 <_start>:
   100b0:       05d00893                li      a7,93
   100b4:       00000513                li      a0,0
   100b8:       00000073                ecall
```

Три инструкции. `li a7, 93` — псевдоинструкция, разворачивается в
`addi a7, x0, 93`. `ecall` — 4 байта, опкод `0x00000073`.

---

## Программа 2: write + exit

Вывести строку в stdout через `write(1, buf, len)`.

```asm
# hello.s — write "Hello, RISC-V!\n" to stdout and exit

    .section .rodata
msg:
    .ascii "Hello, RISC-V!\n"
msg_len = . - msg           # длина = текущий адрес минус начало строки

    .section .text
    .global _start

_start:
    # write(1, msg, msg_len)
    li   a7, 64             # __NR_write = 64
    li   a0, 1              # fd = STDOUT_FILENO
    la   a1, msg            # buf = &msg
    li   a2, msg_len        # count = msg_len
    ecall

    # exit(0)
    li   a7, 93             # __NR_exit = 93
    li   a0, 0
    ecall
```

```bash
$ as -o hello.o hello.s
$ ld -o hello hello.o
$ ./hello
# вывод: Hello, RISC-V!
```

### Что делает `la a1, msg`

`la` (Load Address) — псевдоинструкция. Разворачивается в пару:

```asm
auipc  a1, %pcrel_hi(msg)     # a1 = PC + (msg - PC)[31:12] << 12
addi   a1, a1, %pcrel_lo(msg) # a1 += (msg - PC)[11:0]
```

`auipc` (Add Upper Immediate to PC) — загружает старшие 20 бит
относительного адреса. `addi` добавляет младшие 12 бит. Вместе дают точный
адрес `msg` относительно текущего PC — PC-relative addressing, не зависит
от абсолютного адреса загрузки.

Проверить через `objdump`:

```bash
$ objdump -d hello | sed -n '/<_start>:/,$p'
```

Ожидаемый вывод:
```
00000000000100b0 <_start>:
   100b0:       04000893                li      a7,64
   100b4:       00100513                li      a0,1
   100b8:       00000597                auipc   a1,0x0
   100bc:       01c58593                addi    a1,a1,28 # 100d4 <msg>
   100c0:       00f00613                li      a2,15
   100c4:       00000073                ecall
   100c8:       05d00893                li      a7,93
   100cc:       00000513                li      a0,0
   100d0:       00000073                ecall
```

---

## Программа 3: функция с аргументами

Функция `add(a, b)` на ассемблере, вызываемая из `_start`.

```asm
# add_func.s — function call example

    .section .text
    .global _start

# long add(long a, long b)
# Аргументы: a → a0, b → a1
# Возврат:   a0 = a + b
add:
    add  a0, a0, a1          # a0 = a0 + a1
    ret                      # return (= jalr x0, ra, 0)

_start:
    li   a0, 10              # первый аргумент: 10
    li   a1, 32              # второй аргумент: 32
    call add                 # ra = PC+4; jump to add
    # теперь a0 = 42

    mv   a1, a0              # сохранить результат
    li   a7, 93              # __NR_exit
    mv   a0, a1              # exit code = 42
    ecall
```

```bash
$ as -o add_func.o add_func.s
$ ld -o add_func add_func.o
$ ./add_func
$ echo $?          # вывод: 42
```

### Что делает `call add`

`call` — псевдоинструкция. Разворачивается в:

```asm
auipc  ra, %pcrel_hi(add)
jalr   ra, ra, %pcrel_lo(add)
```

`jalr` (Jump And Link Register): `ra = PC + 4`, затем `PC = ra + offset`.
Именно так `ra` получает адрес возврата.

---

## Программа 4: цикл и работа со стеком

Функция `sum(n)` — сумма чисел от 1 до n.

```asm
# loop.s — loop example

    .section .text
    .global _start

# long sum(long n)  — sum of 1..n
# Leaf function — стек не используется
sum:
    li   t0, 0               # accumulator = 0
    li   t1, 1               # counter i = 1
.loop:
    bgt  t1, a0, .done       # if i > n: goto done
    add  t0, t0, t1          # acc += i
    addi t1, t1, 1           # i++
    j    .loop               # goto loop
.done:
    mv   a0, t0              # return acc
    ret

_start:
    li   a0, 10              # sum(10)
    call sum                 # a0 = 55

    li   a7, 93
    ecall                    # exit(55)
```

```bash
$ as -o loop.o loop.s
$ ld -o loop loop.o
$ ./loop
$ echo $?         # вывод: 55
```

Пошаговая отладка через GDB:

```bash
$ gdb ./loop
(gdb) starti                 # остановиться на _start
(gdb) layout asm             # показать ассемблерное окно
(gdb) info registers t0 t1 a0
(gdb) si                     # step instruction — выполнить одну инструкцию
# повторять si, наблюдая как меняются t0, t1
```

---

## Что компилятор генерирует из C

```c
/* fib.c */
long fibonacci(long n) {
    if (n <= 1) return n;
    return fibonacci(n - 1) + fibonacci(n - 2);
}
```

```bash
# Без оптимизации
$ gcc -O0 -S -o fib.s fib.c
$ cat fib.s
```

В выводе будет видно:
- пролог: `addi sp, sp, -N` + `sd ra, ...` + `sd s0, ...`
- рекурсивные вызовы через `call fibonacci`
- эпилог: `ld ra, ...` + `ld s0, ...` + `addi sp, sp, N` + `ret`

```bash
# С оптимизацией
$ gcc -O2 -S -o fib_opt.s fib.c
$ cat fib_opt.s
```

При `-O2` рекурсия частично превращается в итерацию (tail call
optimization), стек используется иначе. Сравнение двух вариантов наглядно
показывает что делает оптимизатор:
```bash
$ vimdiff fib.s fib_opt.s
```

---

← [Назад](module2-registers.md) · [На главную](../INDEX.md) · [Следующий →](module4-syscalls.md)
