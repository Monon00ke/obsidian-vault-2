---
title: "20


---

## 1. Гарантии доставки сигналов

Реализуем процесс, обрабатывающий SIGUSR1 наращиванием счётчика. Отправляем 10 сигналов — сравниваем количество отправленных и увеличение счётчика.

```bash

python3 -c "

import os, time, signal

counter = 0

def handler(sig, frame):

global counter

counter += 1

signal.signal(signal.SIGUSR1, handler)

pid = os.getpid()

print(f'Receiver PID: {pid}')

child = os.fork()

if child == 0:

N = 10

for i in range(N):

os.kill(pid, signal.SIGUSR1)

print(f'Sent {N} signals')

os._exit(0)

else:

os.waitpid(child, 0)

time.sleep(0.5)

print(f'Counter: {counter}')

"

```


**Наблюдение:** отправлено 10 сигналов, счётчик = 1. Стандартные сигналы не накапливаются (coalescing) — несколько одинаковых сигналов, уже ожидающих доставки, схлопываются в один.

---

## 2. Долгий обработчик

Обработчик SIGINT печатает, спит 2 секунды, возвращается. Быстро нажимаем Ctrl-C 10 раз — сколько раз зафиксирован вход?

```bash

python3 -c "

import os, time, signal

count = 0

def handler(sig, frame):

global count

count += 1

print(f'Handler entered #{count}', flush=True)

time.sleep(2)

print(f'Handler exited #{count}', flush=True)

signal.signal(signal.SIGINT, handler)

print(f'PID: {os.getpid()}')

for i in range(10):

time.sleep(1)

print(f'Loop iteration {i}')

" &

PID=$!

sleep 1

for i in 1 2 3 4 5 6 7 8 9 10; do

kill -INT $PID 2>/dev/null

sleep 0.1

done

sleep 4

kill -9 $PID 2>/dev/null

wait $PID 2>/dev/null

```


**Наблюдение:** все 10 сигналов вошли в обработчик, несмотря на долгий sleep. Обработчик SIGINT в Python не блокирует повторные вхождения — сигналы доставляются реентерабельно.

---

## 3. Рентерабельность

Счётчик входов, глобальная переменная `v`. Одиночный и множественный Ctrl-C.

```bash

python3 -c "

import os, time, signal

v = 0

def h(*args):

global v

n = v

print(f'start handler, n={n}', flush=True)

time.sleep(3)

v = n + 1

print(f'finish handler, v now={v}', flush=True)

signal.signal(signal.SIGINT, h)

print(f'PID: {os.getpid()}')

for i in range(10):

time.sleep(1)

print(f'Loop value of v: {v}', flush=True)

" &

PID=$!

sleep 1

kill -INT $PID

sleep 5

kill -INT $PID

sleep 0.2

kill -INT $PID

sleep 0.2

kill -INT $PID

sleep 6

kill -9 $PID 2>/dev/null

wait $PID 2>/dev/null

```


**Наблюдение:**

- Одиночный Ctrl-C: `n=0`, `v=1` — корректно.

- Три быстрых Ctrl-C: все три вложенных вызова читают `v=1`, все пишут `v=2`. Приращение на 1 вместо 3 — классический race condition из-за неатомарной операции чтение-модификация-запись.

---

## 4. Три нажатия

Приложение игнорирует одиночные Ctrl-C, но завершается при трёх нажатиях за секунду.

```bash

python3 -c "

import os, time, signal

press_times = []

def handler(sig, frame):

global press_times

now = time.time()

press_times.append(now)

press_times = [t for t in press_times if now - t < 1.0]

if len(press_times) >= 3:

print('Three Ctrl-C within 1 second, exiting!', flush=True)

os._exit(0)

signal.signal(signal.SIGINT, handler)

print(f'PID: {os.getpid()}')

for i in range(30):

time.sleep(1)

print(f'iteration {i}', flush=True)

" &

PID=$!

sleep 1

kill -INT $PID

sleep 0.3

kill -INT $PID

sleep 0.3

kill -INT $PID

sleep 2

wait $PID 2>/dev/null

```


**Наблюдение:** три нажатия Ctrl-C за 0.6 секунды — приложение завершается с сообщением "Three Ctrl-C within 1 second, exiting!". Одиночные и двойные нажатия игнорируются.

---

## 5. Текущее состояние

При нажатии Ctrl-\ (SIGQUIT) вместо завершения печатаем стек вызовов в stderr.

```bash

python3 -c "

import os, time, signal, traceback, sys

def handler(sig, frame):

traceback.print_stack(frame, file=sys.stderr)

signal.signal(signal.SIGQUIT, handler)

print(f'PID: {os.getpid()}')

for i in range(5):

time.sleep(1)

print(f'iteration {i}', flush=True)

" &

PID=$!

sleep 2

kill -QUIT $PID

sleep 1

kill -9 $PID 2>/dev/null

wait $PID 2>/dev/null

```


**Наблюдение:** Ctrl-\ печатает стек вызовов в stderr. Процесс продолжает работу после обработки сигнала — завершение с core dump отменено.

---

## 6. Группы процессов

Выводим все группы процессов с количеством процессов в каждой.

```bash

ps -eo pid,ppid,pgid,sid,comm --no-headers | \

awk '{pgid[$3]++} END {for(g in pgid) print "PGID", g, "count", pgid[g]}' | \

sort -k3 -rn | head -20

```


**Наблюдение:** группы процессов отсортированы по количеству процессов. Крупнейшая — обычно systemd или браузер с множеством тредов/процессов.

---

## 7. Сеансы

Выводим все сеансы с количеством процессов и групп процессов в каждом.

```bash

ps -eo sid,pgid --no-headers | sort | \

awk '{

sid=$1; pgid=$2

s_procs[sid]++

s_groups[sid","pgid]=1

} END {

for(s in s_procs) {

gcnt=0

for(k in s_groups) {split(k,a,","); if(a[1]==s) gcnt++}

print "SID " s ": " s_procs[s] " procs, " gcnt " groups"

}

}' | sort -k4 -rn | head -20

```


**Наблюдение:** SID 0 (ядро) имеет больше всего процессов. Остальные сеансы содержат от 1 до 34 процессов в 1 группе каждый.

---

## 8. bash-функции

Реализация bash-функций: передача параметров, return, локальные переменные.

```bash

my_func() {

local name="$1"

local age="$2"

echo "Name: $name, Age: $age"

local sum=$((age + 10))

return 0

}

my_func "Ivan" 25

echo "Return code: $?"

echo "stdout result: $(my_func "Maria" 30)"

```


**Наблюдение:** параметры передаются как `$1`, `$2`. `local` ограничивает область видимости. `return` возвращает код (0-255). Для возврата данных используется stdout.

---

## 9. bash-команда trap

Используем `trap` для обработки SIGINT. Маскирование и игнорирование сигналов.

```bash

handle_int() {

echo "Caught SIGINT, but not quitting"

}

trap handle_int SIGINT

echo "Sending SIGINT to self..."

kill -INT $$

sleep 0.5

echo "Still alive after SIGINT"

# Игнорирование сигнала

trap '' SIGINT

echo "Now SIGINT is ignored"

kill -INT $$

sleep 0.5

echo "Still running after ignored SIGINT"

# Возврат к стандартной обработке

trap - SIGINT

```


**Наблюдение:**

- `trap handler SIGINT` — перехват сигнала, выполнение функции

- `trap '' SIGINT` — игнорирование сигнала

- `trap - SIGINT` — сброс к стандартному поведению (завершение)

---

## 10. Делегирование сигналов

Bash-скрипт обрабатывает SIGUSR1. Определяет все группы процессов в том же сеансе и пересылает сигнал, исключая свою группу.

```bash

handler() {

echo "--- SIGUSR1 received ---"

my_pgid=$(ps -o pgid= -p $$ | tr -d " ")

my_sid=$(ps -o sid= -p $$ | tr -d " ")

echo "My PGID=$my_pgid, SID=$my_sid"

echo "Forwarding SIGUSR1 to other process groups in session:"

ps -eo sid,pgid,pid,comm --no-headers | \

awk -v sid="$my_sid" -v pg="$my_pgid" \

'$1==sid && $2!=pg {print " PGID="$2" PID="$3" "$4}'

}

trap handler SIGUSR1

echo "PID=$$"

echo "Sending SIGUSR1..."

kill -USR1 $$

```


**Наблюдение:** скрипт определяет свой PGID и SID через `/proc`/`ps`, находит все группы в том же сеансе и может переслать сигнал, исключая собственную группу (чтобы избежать циклической обработки).

---

## См. также
- [[00 OS Labs MOC|К оглавлению OS Labs]]
