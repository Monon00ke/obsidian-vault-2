---
title: "21-22
 
---

## 1. Практика jobs, bg, fg

Запуск нескольких фоновых задач, просмотр через `jobs`, перевод между состояниями.

```bash

sleep 100 &

sleep 200 &

sleep 300 &

jobs -l

kill %1 %2 %3 2>/dev/null

jobs -l

```


**Наблюдение:** `jobs -l` показывает PID и состояние каждого фонового задания. `%1`, `%2`, `%3` — ссылки на задания по номеру. `kill %1` завершает задание. `bg` переводит остановленное задание в фон, `fg` — в основной план (владельцем терминала).

---

## 2. Фоновый печатающий скрипт и stty tostop

Скрипт печатает сообщения в фоне. Остановка Ctrl-Z, возобновление bg. Режим tostop.

```bash

# Печатающий скрипт в фоне

for i in 1 2 3 4 5; do

echo "Background message $i"

sleep 1

done &

# Приостановка

kill -TSTP $!

# Возобновление в фоне

kill -CONT $!

# Режим tostop

stty tostop

```


**Наблюдение:** через `kill -TSTP` (Ctrl-Z) процесс приостанавливается (STAT=T). `kill -CONT` (bg) возобновляет в фоне. Режим `stty tostop` запрещает фоновым процессам вывод в терминал — при попытке вывода они получают SIGTTOU и останавливаются.

---

## 3. Менеджер процессов (fork + wait + статус)

Скрипт читает команды из файла, форкает для каждой, ждёт завершения, выводит pid, номер строки и статус.

```bash

# Команды в файле

echo "hello"

date

whoami

pwd

ls /tmp | head -3

```

```python

for each line in file:

pid = os.fork()

if child: exec line; os._exit(0)

else: children[pid] = line_number

for each child:

wpid, status = os.wait()

print(f"pid={wpid} line={line_no} status={os.WEXITSTATUS(status)}")

```


**Наблюдение:** каждый ребёнок выполняет команду через `os.system()`. Родитель собирает статусы через `os.wait()`. `os.WEXITSTATUS` извлекает код возврата.

---

## 4. Ограничение числа одновременных процессов

Развитие задания 3 с константой MAX_CONCURRENT (например, 5). Новый процесс запускается только после освобождения слота.

```bash

MAX_CONCURRENT=2

for each line in file:

while running >= MAX_CONCURRENT:

wpid, status = os.wait()

running--

launch(line)

running++

```


**Наблюдение:** при MAX_CONCURRENT=2 сначала запускаются 2 процесса, остальные ждут. После завершения любого из них запускается следующий. Поведение напоминает thread pool / process pool.

---

## 5. Перенаправление stdout в файлы

Стандартный вывод каждой команды сохраняется в `/tmp/job_<line>_pid<PID>.stdout`.

```python

pid = os.fork()

if pid == 0:

outfile = f"/tmp/job_{idx}_pid{os.getpid()}.stdout"

sys.stdout = open(outfile, "w")

os.system(cmd)

sys.stdout.close()

os._exit(0)

```


**Наблюдение:** после `fork()` в ребёнке перенаправляем `sys.stdout` в файл. Вывод команды попадает в файл, а не на терминал. Родитель после `wait()` читает файл и выводит содержимое.

---

## 6. Обработка нарушения контракта (stdin)

Если команда пытается читать из stdin, её нужно принудительно завершить.

```python

if os.fork() == 0:

# Закрываем stdin для защиты

os.close(0)

os.system(cmd)

os._exit(0)

```


**Наблюдение:** закрытие stdin (`os.close(0)`) перед выполнением команды гарантирует, что попытка чтения из stdin вызовет ошибку (EBADF), и команда завершится. Родитель фиксирует это как «нарушение контракта».

---

## 7. Интерактивный REPL

После запуска процессов скрипт переходит в цикл REPL с командами: `quit`, `finished`, `current`, `wait`.

```python

while True:

cmd = input(">>> ")

if cmd == "quit": break

elif cmd == "finished": show_finished()

elif cmd == "current": show_running()

elif cmd == "wait": wait_for_all()

```


**Наблюдение:** REPL позволяет интерактивно управлять процессами: смотреть завершённые, выполняющиеся, дожидаться всех.

---

## 8. Обработка SIGCHLD

Проблема: в REPL процессы завершаются, но мы узнаём об этом только при вводе команды. Решение — обработчик SIGCHLD.

```python

import signal

def handle_sigchld(sig, frame):

while True:

wpid, status = os.waitpid(-1, os.WNOHANG)

if wpid == 0: break

finished.append({"pid": wpid, "status": status})

signal.signal(signal.SIGCHLD, handle_sigchld)

```


**Наблюдение:** SIGCHLD доставляется при каждом завершении дочернего процесса. Обработчик через `os.waitpid(-1, WNOHANG)` собирает всех завершённых детей неблокирующим образом. Это решает проблему пропущенных завершений во время ожидания ввода.

---

## 9. POSIX-очереди сообщений

Знакомство с `posix_ipc.MessageQueue`.

```python

import posix_ipc as ipc

mq = ipc.MessageQueue('/test_mq', ipc.O_CREAT)

mq.send(b'Hello from parent')

msg, prio = mq.receive()

print(f'Received: {msg.decode()}')

mq.close()

ipc.unlink_message_queue('/test_mq')

```


**Наблюдение:** POSIX-очереди — механизм IPC с именованными очередями. Сообщения имеют приоритет. Поддерживают блокирующий и неблокирующий приём. После использования очередь нужно удалить через `unlink_message_queue`.

---

## 10. Семафоры и дедлок

Попытка организовать дедлок через два семафора и два процесса с перекрёстным захватом.

```python

sem_a = ipc.Semaphore('/sem_a', ipc.O_CREAT, initial_value=1)

sem_b = ipc.Semaphore('/sem_b', ipc.O_CREAT, initial_value=1)

pid = os.fork()

if pid == 0: # Child

sem_b.acquire() # Захватывает B

sem_a.acquire() # Пытается захватить A → DEADLOCK

else: # Parent

sem_a.acquire() # Захватывает A

sem_b.acquire() # Пытается захватить B → DEADLOCK

```


**Наблюдение:** родитель захватывает A, ребёнок захватывает B. Затем родитель пытается захватить B (занято), ребёнок — A (занято). Классический deadlock. Решение: использовать таймауты (`acquire(timeout=2)`) или единый порядок захвата всеми процессами.

---

## См. также
- [[00 OS Labs MOC|К оглавлению OS Labs]]
