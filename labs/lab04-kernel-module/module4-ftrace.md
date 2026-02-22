# module4 · ftrace: трассировка ядра и ассемблерная инструментация RISC-V

← [Назад](module3-counter-module.md) · [На главную](../INDEX.md)

> **Kernel docs:** [kernel.org/doc/html/latest/trace/ftrace.html](https://www.kernel.org/doc/html/latest/trace/ftrace.html)

> **Kernel source:** `kernel/trace/ftrace.c`
> [elixir.bootlin.com/linux/v6.12/source/kernel/trace/ftrace.c](https://elixir.bootlin.com/linux/v6.12/source/kernel/trace/ftrace.c)

> **RISC-V ftrace:** `arch/riscv/kernel/ftrace.c`
> [elixir.bootlin.com/linux/v6.12/source/arch/riscv/kernel/ftrace.c](https://elixir.bootlin.com/linux/v6.12/source/arch/riscv/kernel/ftrace.c)

---

## Что такое ftrace и зачем он нужен

ftrace (Function Tracer) — встроенный в ядро Linux механизм трассировки
функций. Позволяет в реальном времени наблюдать какие функции ядра
вызываются, в каком порядке, сколько времени занимают, и с какими
аргументами.

Это не отладчик — не останавливает выполнение. Это трассировщик — фиксирует
события в кольцевой буфер с минимальным влиянием на производительность.

ftrace работает через трассировочную файловую систему (`tracefs`), обычно
смонтированную в `/sys/kernel/debug/tracing/`.

---

## Как ftrace инструментирует функции: RISC-V ассемблер

Это ключевой вопрос для понимания механизма. Проверим прямо на собранном
модуле `my_counter.ko`.

### Флаг компилятора

При сборке модуля в `.my_counter.o.cmd` видна следующая строка:

```
-fpatchable-function-entry=4
```

Синтаксис флага: `-fpatchable-function-entry=N[,M]`, где `N` — общее
количество NOP-инструкций, `M` — сколько из них разместить **после** точки
входа (по умолчанию `M=0`). Значит: **4 NOP перед прологом функции, 0
после**.

Кроме того, компилятор определяет макрос `CC_USING_PATCHABLE_FUNCTION_ENTRY`
и записывает адрес каждой инструментированной функции в секцию
`.patchable_function_entries` — именно из неё ядро строит список функций,
доступных для трассировки.

### Что компилятор вставляет в бинарный код

Реальный вывод `objdump -d my_counter.ko` для двух функций:

```
0000000000000000 <my_counter_do_increment>:  ← символ стоит на первом NOP
   0:   0001                    nop          ← c.nop (2 байта, opcode 0x0001)
   2:   0001                    nop
   4:   0001                    nop
   6:   0001                    nop

0000000000000008 <.LFB...>:                  ← GCC internal label: начало пролога
   8:   ...                     addi sp, sp, -N
   ...

00000000000000XX <my_counter_increment_store>:
  XX:   0001                    nop
  XX:   0001                    nop
  XX:   0001                    nop
  XX:   0001                    nop

00000000000000XX <.LFB...>:
  XX:   ...                     addi sp, sp, -N
  ...
```

Каждый NOP — это **сжатая инструкция** `c.nop` (opcode `0x0001`), **2
байта**. Это стало возможно потому что ядро собрано с расширением `C`
(compressed instructions) через `-march=rv64imac_zicsr_zifencei`.

Обратите внимание на структуру меток: символ функции стоит на первом
`c.nop`, а метка `.LFBxxxx` стоит уже на прологе. Ftrace патчит именно по
адресу символа — то есть по адресу первого `c.nop`.

Итого: **4 × 2 байта = 8 байт** пустого пространства перед каждым прологом.

### noinline и почему без него функций нет в ftrace

Функции `my_counter_do_increment`, `my_counter_do_reset`,
`my_counter_do_trigger` помечены `noinline`:

```c
static noinline void my_counter_do_increment(void) { ... }
```

Без этого атрибута компилятор при `-O2` **встраивает** (inlines) тело
небольшой функции прямо в вызывающий код. Отдельного символа не возникает,
NOP-заглушки не генерируются, и функция не появляется в
`available_filter_functions`. Фильтр `my_counter_do_*` ничего бы не нашёл.

С `noinline` компилятор обязан сгенерировать отдельный пролог и символ —
и функция получает свои 4 NOP-заглушки.

Это главная причина для единого префикса `my_counter_` во всех именах:
фильтр `my_counter_*` выбирает точно наш модуль без шума от ядра.
Короткие имена вроде `reset_store` или `value_show` встречаются во многих
драйверах ядра и делают фильтрацию бесполезной.

### Важное исключение: `__exit`-функции не инструментируются

В `objdump` видно:

```
Дизассемблирование раздела .exit.text:

0000000000000000 <cleanup_module>:
   0:   1101                    addi    sp,sp,-32   ← нет NOP, сразу пролог
   ...
```

`my_counter_exit` помечена `__exit` — компоновщик помещает её в секцию
`.exit.text`. Функции из `.exit.text` не инструментируются ftrace: после
выгрузки модуля их код уже не существует в памяти.

`my_counter_init` (`.init.text`) — инструментируется, но в
`available_filter_functions` не попадает: секция `.init.text` физически
освобождается ядром (`do_free_init`) сразу после того как `my_counter_init`
вернул управление.

### Три стадии: от c.nop до вызова трассировщика

Функция проходит три состояния. Рассмотрим `my_counter_do_increment`.

---

**Стадия 1: в файле `.ko` после компиляции** — стоят `c.nop`:

```
  0x0000  c.nop (0x0001)  ←
  0x0002  c.nop (0x0001)   │ 8 байт, 4 × 16-bit c.nop
  0x0004  c.nop (0x0001)   │ вставлены компилятором
  0x0006  c.nop (0x0001)  ←
  0x0008  addi sp, sp, -N   ← пролог функции
```

---

**Стадия 2: сразу после `insmod`** — `ftrace_init_nop` заменяет `c.nop` на
`NOP4`.

Из `arch/riscv/kernel/ftrace.c`:

```c
int ftrace_make_nop(struct module *mod, struct dyn_ftrace *rec,
		    unsigned long addr)
{
	unsigned int nops[2] = {NOP4, NOP4};

	if (patch_insn_write((void *)rec->ip, nops, MCOUNT_INSN_SIZE))
		return -EPERM;

	return 0;
}
```

`NOP4` — это **32-битный** NOP (`addi x0, x0, 0`, opcode `0x00000013`).
Два таких слова = 8 байт = ровно столько, сколько занимали четыре `c.nop`.
`ftrace_init_nop` берёт `text_mutex` сам, без `stop_machine` — в этот
момент модуль ещё не запущен.

```
  0x0000  NOP4 (0x00000013)  ← 4 байта
  0x0004  NOP4 (0x00000013)  ← 4 байта
  0x0008  addi sp, sp, -N    ← пролог (не тронут)
```

---

**Стадия 3: трассировка включена** (`echo function > current_tracer`) —
`ftrace_make_call` заменяет `NOP4×2` на вызов трассировщика.

Из `arch/riscv/kernel/ftrace.c`:

```c
int ftrace_make_call(struct dyn_ftrace *rec, unsigned long addr)
{
	unsigned int call[2];

	make_call_t0(rec->ip, addr, call);

	if (patch_insn_write((void *)rec->ip, call, MCOUNT_INSN_SIZE))
		return -EPERM;

	return 0;
}
```

`make_call_t0` формирует пару `auipc t0` + `jalr zero, t0`. Целевой
регистр перехода — `zero`, а не `ra`. Это означает что **`ra` не
портится**: при входе в `ftrace_caller` регистр `ra` содержит адрес
возврата из трассируемой функции — именно его трассировщик читает чтобы
узнать имя вызывающей функции (caller).

```
  0x0000  auipc  t0, %hi(ftrace_caller)       ← 4 байта
  0x0004  jalr   zero, %lo(ftrace_caller)(t0)  ← 4 байта
  0x0008  addi   sp, sp, -N                    ← пролог (не тронут)
```

Когда трассировка выключается (`echo nop > current_tracer`),
`ftrace_make_nop` возвращает пару `{NOP4, NOP4}` обратно.

---

### Как патчинг происходит безопасно

`ftrace_make_call`/`ftrace_make_nop` вызываются через
`arch_ftrace_update_code`, который использует `stop_machine`:

```c
void arch_ftrace_update_code(int command)
{
	struct ftrace_modify_param param = { command, ATOMIC_INIT(0) };

	stop_machine(__ftrace_modify_code, &param, cpu_online_mask);
}

static int __ftrace_modify_code(void *data)
{
	struct ftrace_modify_param *param = data;

	if (atomic_inc_return(&param->cpu_count) == num_online_cpus()) {
		ftrace_modify_all_code(param->command);
		/* release: гарантирует видимость записи до сброса icache */
		atomic_inc_return_release(&param->cpu_count);
	} else {
		while (atomic_read(&param->cpu_count) <= num_online_cpus())
			cpu_relax();
		/* acquire: синхронизируется с release выше */
		local_flush_icache_all();
	}

	return 0;
}
```

Один CPU выполняет `ftrace_modify_all_code` и делает `release`-инкремент.
Остальные CPU ждут в спинлупе, затем сбрасывают I-cache с
`acquire`-семантикой — гарантируя что сброс произойдёт **после** того как
патч стал виден. Именно поэтому ftrace можно включать и выключать на
работающей системе без перезагрузки.

---

### Итоговая схема

```
  Компилятор         insmod                       echo function >
  (сборка .ko)       (ftrace_init_nop             current_tracer
                      под text_mutex)             (ftrace_make_call
                                                   через stop_machine)
  ┌──────────┐       ┌──────────────┐           ┌──────────────────────────────┐
  │ c.nop ×4 │ ────► │ NOP4 × 2     │ ────────► │ auipc t0, %hi(ftrace_caller) │
  │ 4×2=8 б  │       │ 2×4=8 б      │           │ jalr  zero, %lo(...)(t0)     │
  └──────────┘       └──────────────┘           └──────────────────────────────┘
                                                            │
                      ◄─────────────────────────────────────
                      echo nop > current_tracer
                      (ftrace_make_nop → NOP4×2)
```

---

### Проверка: убедиться что функции модуля инструментированы

```bash
$ sudo insmod ~/labs/lab04/my_counter/my_counter.ko step=5

$ sudo cat /sys/kernel/debug/tracing/available_filter_functions | grep my_counter
```

Ожидаемо:

```
my_counter_trigger_store [my_counter]
my_counter_trigger_show [my_counter]
my_counter_reset_show [my_counter]
my_counter_increment_show [my_counter]
my_counter_value_show [my_counter]
my_counter_reset_store [my_counter]
my_counter_increment_store [my_counter]
my_counter_value_store [my_counter]
my_counter_do_trigger [my_counter]
my_counter_do_reset [my_counter]
my_counter_do_increment [my_counter]
```

Все нужные функции видны, включая `my_counter_do_*` — благодаря `noinline`.
`my_counter_exit` в списке не появится (`.exit.text`, нет NOP-заглушек).

---

## Точки входа: tracefs

```bash
$ mount | grep tracefs
# если не смонтирован:
$ sudo mount -t tracefs nodev /sys/kernel/debug/tracing

$ sudo cat /sys/kernel/debug/tracing/available_tracers
# ожидаемо: function_graph function nop
```

---

## Трассировщик function

Показывает каждый вызов каждой (отфильтрованной) функции ядра.

```bash
$ echo function | sudo tee /sys/kernel/debug/tracing/current_tracer

# Очистить буфер
$ echo | sudo tee /sys/kernel/debug/tracing/trace

# Включить трассировку
$ echo 1 | sudo tee /sys/kernel/debug/tracing/tracing_on

# Выполнить действие
$ sudo insmod ~/labs/lab04/my_counter/my_counter.ko step=5

# Остановить трассировку
$ echo 0 | sudo tee /sys/kernel/debug/tracing/tracing_on

$ sudo cat /sys/kernel/debug/tracing/trace | head -20
$ sudo cat /sys/kernel/debug/tracing/trace | grep -B 3 -A 3 "my_counter"
```

Вывод выглядит примерно так:

```bash
# tracer: function
#
# entries-in-buffer/entries-written: 2008530/2574569   #P:4
#
#                                _-----=> irqs-off/BH-disabled
#                               / _----=> need-resched
#                              | / _---=> hardirq/softirq
#                              || / _--=> preempt-depth
#                              ||| / _-=> migrate-disable
#                              |||| /     delay
#           TASK-PID     CPU#  |||||  TIMESTAMP  FUNCTION
#              | |         |   |||||     |         |
          <idle>-0       [003] d.... 13624.435438: irq_enter_rcu <-handle_riscv_irq
          <idle>-0       [003] d.h.. 13624.435441: tick_irq_enter <-irq_enter_rcu
          <idle>-0       [003] d.h.. 13624.435443: tick_check_oneshot_broadcast_this_cpu <-tick_irq_enter
          <idle>-0       [003] d.h.. 13624.435445: ktime_get <-tick_irq_enter
          <idle>-0       [003] d.h.. 13624.435447: riscv_clocksource_rdtime <-ktime_get
          <idle>-0       [003] d.h.. 13624.435450: tick_nohz_stop_idle <-tick_irq_enter
          <idle>-0       [003] d.h.. 13624.435452: nr_iowait_cpu <-tick_nohz_stop_idle
          <idle>-0       [003] d.h.. 13624.435454: tick_do_update_jiffies64 <-tick_irq_enter
```

Каждая строка: `процесс-PID [CPU] флаги время: функция <- вызывающий`

Из-за переполнения буфера и большого кол-ва трассируемых функций мы можем
не "поймать" выполнение ожидаемой функции, поэтому лучше увеличить буфер и
запускать последовательно с минимальными задержками:

```bash
# Очистить и увеличить буфер
$ echo | sudo tee /sys/kernel/debug/tracing/trace
$ echo 16384 | sudo tee /sys/kernel/debug/tracing/buffer_size_kb

# Выгрузить модуль
$ sudo rmmod my_counter

$ echo 1 | sudo tee /sys/kernel/debug/tracing/tracing_on   &&\
  sudo insmod ~/labs/lab04/my_counter/my_counter.ko step=5 &&\
  echo 0 | sudo tee /sys/kernel/debug/tracing/tracing_on   &&\
  sudo cat /sys/kernel/debug/tracing/trace | head -12      &&\
  sudo cat /sys/kernel/debug/tracing/trace | grep "my_counter"
```
Ожидаемый вывод:

```bash
# tracer: function
#
# entries-in-buffer/entries-written: 2089201/2429053   #P:4
#
#                                _-----=> irqs-off/BH-disabled
#                               / _----=> need-resched
#                              | / _---=> hardirq/softirq
#                              || / _--=> preempt-depth
#                              ||| / _-=> migrate-disable
#                              |||| /     delay
#           TASK-PID     CPU#  |||||  TIMESTAMP  FUNCTION
#              | |         |   |||||     |         |
          insmod-7773    [003] ..... 14060.803621: my_counter_init <-do_one_initcall
          insmod-7773    [003] ..... 14060.803660: sysfs_create_group <-my_counter_init
          insmod-7773    [003] ..... 14060.803767: _printk <-my_counter_init
          insmod-7773    [003] ..... 14060.803887: _printk <-my_counter_init
```
---

## Фильтрация по имени функции

```bash
# Сбросить фильтры
$ echo | sudo tee /sys/kernel/debug/tracing/set_ftrace_filter

# Трассировать только функции нашего модуля — точный фильтр без конфликтов
$ echo 'my_counter_*' | sudo tee /sys/kernel/debug/tracing/set_ftrace_filter

# Проверить что фильтр применился
$ sudo cat /sys/kernel/debug/tracing/set_ftrace_filter

$ echo function | sudo tee /sys/kernel/debug/tracing/current_tracer
$ echo | sudo tee /sys/kernel/debug/tracing/trace
$ echo 1 | sudo tee /sys/kernel/debug/tracing/tracing_on

# Взаимодействовать с модулем из sysfs
$ echo 1 | sudo tee /sys/kernel/my_counter/increment
$ echo 1 | sudo tee /sys/kernel/my_counter/increment
$ echo 1 | sudo tee /sys/kernel/my_counter/reset
$ echo 1 | sudo tee /sys/kernel/my_counter/trigger

$ echo 0 | sudo tee /sys/kernel/debug/tracing/tracing_on
$ sudo cat /sys/kernel/debug/tracing/trace
```

В выводе будет видно только вызовы `my_counter_*`:
```bash
# tracer: function
#
# entries-in-buffer/entries-written: 8/8   #P:4
#
#                                _-----=> irqs-off/BH-disabled
#                               / _----=> need-resched
#                              | / _---=> hardirq/softirq
#                              || / _--=> preempt-depth
#                              ||| / _-=> migrate-disable
#                              |||| /     delay
#           TASK-PID     CPU#  |||||  TIMESTAMP  FUNCTION
#              | |         |   |||||     |         |
             tee-7899    [001] ..... 14214.305697: my_counter_increment_store <-kobj_attr_store
             tee-7899    [001] ..... 14214.305702: my_counter_do_increment <-my_counter_increment_store
             tee-7911    [003] ..... 14217.482638: my_counter_increment_store <-kobj_attr_store
             tee-7911    [003] ..... 14217.482643: my_counter_do_increment <-my_counter_increment_store
             tee-7923    [001] ..... 14220.819192: my_counter_reset_store <-kobj_attr_store
             tee-7923    [001] ..... 14220.819199: my_counter_do_reset <-my_counter_reset_store
             tee-7935    [002] ..... 14224.198952: my_counter_trigger_store <-kobj_attr_store
             tee-7935    [002] ..... 14224.198956: my_counter_do_trigger <-my_counter_trigger_store
```

---

## Трассировщик function_graph

`function_graph` показывает дерево вызовов с временем выполнения каждой
функции. Для нашего модуля особенно наглядно — видна вся цепочка от
sysfs-store до do_*.

```bash
$ echo function_graph | sudo tee /sys/kernel/debug/tracing/current_tracer

# Ограничить глубину графа (иначе вывод огромный)
$ echo 5 | sudo tee /sys/kernel/debug/tracing/max_graph_depth

# Трассировать все функции
$ echo | sudo tee /sys/kernel/debug/tracing/set_ftrace_filter

# Ограничить граф функциями нашего модуля
$ echo 'my_counter_*' | sudo tee /sys/kernel/debug/tracing/set_graph_function

$ echo | sudo tee /sys/kernel/debug/tracing/trace
$ echo 1 | sudo tee /sys/kernel/debug/tracing/tracing_on

$ echo 1 | sudo tee /sys/kernel/my_counter/increment

$ echo 0 | sudo tee /sys/kernel/debug/tracing/tracing_on
$ sudo cat /sys/kernel/debug/tracing/trace
```

Ожидаемый вывод — видна вся цепочка:

```bash
# tracer: function_graph
#
# CPU  DURATION                  FUNCTION CALLS
# |     |   |                     |   |   |   |
 1)               |  my_counter_increment_store [my_counter]() {
 1)               |    my_counter_do_increment [my_counter]() {
 1)               |      _printk() {
 1)               |        vprintk() {
 1)   3.250 us    |          is_printk_legacy_deferred();
 1) + 95.500 us   |          vprintk_default();
 1) ! 106.000 us  |        }
 1) ! 110.250 us  |      }
 1) ! 114.500 us  |    }
 1) ! 124.250 us  |  }
```

Число слева от `us` — время выполнения функции в микросекундах.

---

## Использование trace-cmd

`trace-cmd` — утилита командной строки для удобной записи и просмотра ftrace-трасс.

```bash
# Трассировать конкретные функции при выполнении команды
$ sudo trace-cmd record -p function -l 'my_counter_*' \
    sh -c 'echo 1 > /sys/kernel/my_counter/increment'
$ sudo trace-cmd report
cpus=4
              sh-10345 [002] ..... 18576.043404: function:             .LPFE3181
              sh-10345 [002] ..... 18576.043408: function:             .LPFE3175
```

```bash
# function_graph через trace-cmd — видна вся цепочка вызовов
$ sudo trace-cmd record -p function_graph -g 'my_counter_*' \
    sh -c 'echo 1 > /sys/kernel/my_counter/increment && \
           echo 1 > /sys/kernel/my_counter/reset'
$ sudo trace-cmd report
cpus=4
              sh-10662 [001] ..... 18697.852901: funcgraph_entry:                   |  .LPFE3181() {
              sh-10662 [001] ..... 18697.852905: funcgraph_entry:                   |    .LPFE3175() {
              sh-10662 [001] ..... 18697.852908: funcgraph_entry:                   |      _printk() {
              sh-10662 [001] ..... 18697.852910: funcgraph_entry:                   |        vprintk() {
              sh-10662 [001] ..... 18697.852912: funcgraph_entry:        3.250 us   |          is_printk_legacy_deferred();
              sh-10662 [001] ..... 18697.852918: funcgraph_entry:      + 94.250 us  |          vprintk_default();
              sh-10662 [001] ..... 18697.853014: funcgraph_exit:       ! 104.750 us |        }
              sh-10662 [001] ..... 18697.853016: funcgraph_exit:       ! 109.000 us |      }
              sh-10662 [001] ..... 18697.853019: funcgraph_exit:       ! 113.500 us |    }
              sh-10662 [001] ..... 18697.853021: funcgraph_exit:       ! 123.000 us |  }
              sh-10662 [001] ..... 18697.853456: funcgraph_entry:                   |  .LPFE3183() {
              sh-10662 [001] ..... 18697.853458: funcgraph_entry:                   |    .LPFE3176() {
              sh-10662 [001] ..... 18697.853460: funcgraph_entry:                   |      _printk() {
              sh-10662 [001] ..... 18697.853461: funcgraph_entry:                   |        vprintk() {
              sh-10662 [001] ..... 18697.853463: funcgraph_entry:        2.250 us   |          is_printk_legacy_deferred();
              sh-10662 [001] ..... 18697.853468: funcgraph_entry:      + 79.250 us  |          vprintk_default();
              sh-10662 [001] ..... 18697.853549: funcgraph_exit:       + 88.000 us  |        }
              sh-10662 [001] ..... 18697.853551: funcgraph_exit:       + 91.750 us  |      }
              sh-10662 [001] ..... 18697.853553: funcgraph_exit:       + 95.500 us  |    }
              sh-10662 [001] ..... 18697.853556: funcgraph_exit:       ! 100.500 us |  }
```

### Проблема: нечитаемые имена функций в выводе trace-cmd

При трассировке модуля ядра `trace-cmd report` может показывать технические
метки компилятора вместо имён функций:

```
sh-10662  [001] .....  funcgraph_entry:  |  .LPFE3181() {
sh-10662  [001] .....  funcgraph_entry:  |    .LPFE3175() {
sh-10662  [001] .....  funcgraph_entry:  |      _printk() {
...
sh-10662  [001] .....  funcgraph_entry:  |  .LPFE3183() {
sh-10662  [001] .....  funcgraph_entry:  |    .LPFE3176() {
```

#### Причина

На RISC-V механизм динамического ftrace основан на атрибуте компилятора
`patchable_function_entry`. GCC генерирует для каждой функции метку вида
`.LPFExxxx`, указывающую на точку патчинга. Эти метки регистрируются в
`/proc/kallsyms` при загрузке модуля наравне с именами функций:

```
ffffffff0273822a t .LPFE3181    [my_counter]
ffffffff0273822a t my_counter_increment_store    [my_counter]
ffffffff02738034 t .LPFE3176    [my_counter]
ffffffff02738034 t my_counter_do_reset    [my_counter]
```

Метка `.LPFExxxx` и имя функции имеют **одинаковый адрес**. При резолвинге
`trace-cmd record` встраивает `/proc/kallsyms` в файл `trace.dat`, и если
метка стоит в таблице раньше имени функции, она «побеждает».

> Ядерный интерфейс `/sys/kernel/debug/tracing/trace` этой проблемы
> лишён — ftrace хранит имена функций напрямую в shadow-стеке вызовов, не
> делая поиска по адресу в kallsyms.

#### Решение: подменить kallsyms перед записью

Нужно убрать `.LPFE*`-метки из таблицы символов **до** вызова `trace-cmd
record`, потому что именно в момент записи утилита встраивает kallsyms в
`trace.dat`.

```bash
# 1. Создать очищенную таблицу символов без .LPFE* и прочих меток компилятора
$ sudo grep -Ev '[[:space:]]\.[A-Z]' /proc/kallsyms > /tmp/kallsyms_clean

# 2. Подменить /proc/kallsyms через bind-mount
$ sudo mount --bind /tmp/kallsyms_clean /proc/kallsyms

# 3. Записать трассу — теперь trace.dat будет содержать чистую таблицу
$ sudo trace-cmd record -p function_graph -g 'my_counter_*' \
    sh -c 'echo 1 > /sys/kernel/my_counter/increment && \
           echo 1 > /sys/kernel/my_counter/reset'

# 4. Снять подмену
$ sudo umount /proc/kallsyms

# 5. Просмотреть трассу
$ sudo trace-cmd report
```

Вывод теперь содержит корректные имена:
```bash
cpus=4
              sh-10777 [002] ..... 18946.755205: funcgraph_entry:                   |  my_counter_increment_store() {
              sh-10777 [002] ..... 18946.755210: funcgraph_entry:                   |    my_counter_do_increment() {
              sh-10777 [002] ..... 18946.755212: funcgraph_entry:                   |      _printk() {
              sh-10777 [002] ..... 18946.755214: funcgraph_entry:                   |        vprintk() {
              sh-10777 [002] ..... 18946.755215: funcgraph_entry:        3.250 us   |          is_printk_legacy_deferred();
              sh-10777 [002] ..... 18946.755222: funcgraph_entry:      + 93.750 us  |          vprintk_default();
              sh-10777 [002] ..... 18946.755318: funcgraph_exit:       ! 104.750 us |        }
              sh-10777 [002] ..... 18946.755320: funcgraph_exit:       ! 108.750 us |      }
              sh-10777 [002] ..... 18946.755322: funcgraph_exit:       ! 113.000 us |    }
              sh-10777 [002] ..... 18946.755325: funcgraph_exit:       ! 121.750 us |  }
              sh-10777 [002] ..... 18946.755680: funcgraph_entry:                   |  my_counter_reset_store() {
              sh-10777 [002] ..... 18946.755682: funcgraph_entry:                   |    my_counter_do_reset() {
              sh-10777 [002] ..... 18946.755684: funcgraph_entry:                   |      _printk() {
              sh-10777 [002] ..... 18946.755686: funcgraph_entry:                   |        vprintk() {
              sh-10777 [002] ..... 18946.755687: funcgraph_entry:        2.500 us   |          is_printk_legacy_deferred();
              sh-10777 [002] ..... 18946.755692: funcgraph_entry:      + 79.500 us  |          vprintk_default();
              sh-10777 [002] ..... 18946.755774: funcgraph_exit:       + 88.000 us  |        }
              sh-10777 [002] ..... 18946.755776: funcgraph_exit:       + 91.750 us  |      }
              sh-10777 [002] ..... 18946.755778: funcgraph_exit:       + 95.750 us  |    }
              sh-10777 [002] ..... 18946.755780: funcgraph_exit:       ! 100.250 us |  }
```

#### Рекомендации

При работе с `trace-cmd` на RISC-V с модулями ядра:

- Если нужен именно файл `trace.dat` (для передачи, архивирования,
  повторного просмотра) — использовать bind-mount чистого kallsyms перед
  записью.
- Если нужен быстрый интерактивный просмотр без сохранения — предпочтительнее
  читать трассу напрямую через `/sys/kernel/debug/tracing/trace`: ядерный
  ftrace всегда резолвит имена корректно и не зависит от содержимого kallsyms.
- Проблема специфична для RISC-V. На x86 механизм патчинга устроен иначе
  (`__fentry__`), и `.LPFE*`-меток в kallsyms не возникает.

---

## Трассировка событий syscall

```bash
# Вернуться к трассировщику nop (без трассировки функций)
$ echo nop | sudo tee /sys/kernel/debug/tracing/current_tracer

# Включить события записи
$ echo 1 | sudo tee /sys/kernel/debug/tracing/events/syscalls/sys_enter_write/enable
$ echo 1 | sudo tee /sys/kernel/debug/tracing/events/syscalls/sys_exit_write/enable

$ echo | sudo tee /sys/kernel/debug/tracing/trace
$ echo 1 | sudo tee /sys/kernel/debug/tracing/tracing_on

$ echo 1 | sudo tee /sys/kernel/my_counter/increment

$ echo 0 | sudo tee /sys/kernel/debug/tracing/tracing_on

$ sudo cat /sys/kernel/debug/tracing/trace | grep write

# Выключить события
$ echo 0 | sudo tee /sys/kernel/debug/tracing/events/syscalls/sys_enter_write/enable
$ echo 0 | sudo tee /sys/kernel/debug/tracing/events/syscalls/sys_exit_write/enable

```

В выводе будет виден `write()` из userspace (`tee`) — оболочка записала
`"1\n"` в файл sysfs, что вызвало `my_counter_increment_store()` в ядре.

---

## Очистка после работы с ftrace

```bash
$ echo 0 | sudo tee /sys/kernel/debug/tracing/tracing_on

# Вернуть трассировщик nop (минимальные накладные расходы)
$ echo nop | sudo tee /sys/kernel/debug/tracing/current_tracer

# Сбросить все фильтры
$ echo | sudo tee /sys/kernel/debug/tracing/set_ftrace_filter
$ echo | sudo tee /sys/kernel/debug/tracing/set_graph_function

# Очистить буфер
$ echo | sudo tee /sys/kernel/debug/tracing/trace

$ sudo rmmod my_counter
```

---

← [Назад](module3-counter-module.md) · [На главную](../INDEX.md)
