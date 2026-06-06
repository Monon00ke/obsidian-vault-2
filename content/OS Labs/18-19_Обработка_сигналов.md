---
title: "18-19 — Signal Handling"
---

## 1. Модуль signal

Ознакомление с модулем `signal`:
- Документация Python: https://docs.python.org/3/library/signal.html
- Справочная информация: https://man7.org/linux/man-pages/man7/signal.7.html

---

## 2. Базовый скрипт

Скрипт с бесконечным циклом, тестируем Ctrl-C, Ctrl-Z, fg, Ctrl-\.

```bash
python3 -c '
import time
i = 1
while True:
    print(i)
    time.sleep(i)
    i += 1
'
```


**Наблюдения:**
- `Ctrl-C` → SIGINT → завершение процесса
- `Ctrl-Z` → SIGTSTP → приостановка процесса (STAT=T)
- `fg` → возобновление приостановленного процесса
- `Ctrl-\` → SIGQUIT → завершение с core dump

---

## 3. Заблокированные (замаскированные) сигналы

Маскируем все сигналы через `signal.pthread_sigmask`.

```bash
python3 -c '
import signal, time, os

signal.pthread_sigmask(signal.SIG_BLOCK, range(1, 32))
print(f"PID: {os.getpid()}")
print("All signals blocked. Try Ctrl-C, Ctrl-Z, Ctrl-\\")

i = 1
while i <= 5:
    print(f"iteration {i}")
    time.sleep(1)
    i += 1

print("Loop finished, unmasking signals...")
signal.pthread_sigmask(signal.SIG_UNBLOCK, range(1, 32))
print("Signals unmasked")
' &
PID=$!
sleep 2
kill -INT $PID
sleep 1
kill -TSTP $PID
sleep 1
kill -QUIT $PID
sleep 2
ps -o pid,stat,cmd -p $PID 2>/dev/null || echo "Process not found"
kill -9 $PID 2>/dev/null
```


**Наблюдение:** замаскированные сигналы игнорируются. Процесс не реагирует на SIGINT, SIGTSTP, SIGQUIT. Только `kill -9` (SIGKILL) может завершить процесс, так как SIGKILL нельзя замаскировать.

---

## 4. "Висящие" (pending) сигналы

Наблюдаем за `signal.sigpending()` до и после отправки сигналов.

```bash
python3 -c '
import signal, time, os

signal.pthread_sigmask(signal.SIG_BLOCK, range(1, 32))
print(f"PID: {os.getpid()}")

for i in range(5):
    pending = signal.sigpending()
    print(f"iteration {i}: pending signals = {pending}")
    time.sleep(1)

print("Unmasking...")
signal.pthread_sigmask(signal.SIG_UNBLOCK, range(1, 32))
print("Done")
' &
PID=$!
sleep 1
kill -INT $PID
kill -TERM $PID
sleep 3
wait $PID 2>/dev/null
```


**Наблюдение:** после отправки сигналы появляются в pending-наборе и остаются там до размаскирования.

---

## 5. Отложенная обработка

Цикл конечный, после цикла размаскируем сигналы. Наблюдаем, подействуют ли сигналы, порожденные ранее.

```bash
python3 -c '
import signal, time, os

signal.pthread_sigmask(signal.SIG_BLOCK, range(1, 32))
print(f"PID: {os.getpid()}")
print("Signals blocked. Send signals now...")

for i in range(3):
    print(f"iteration {i}")
    time.sleep(1)

print("Unmasking signals...")
signal.pthread_sigmask(signal.SIG_UNBLOCK, range(1, 32))
print("This print should appear after signal handling")
' &
PID=$!
sleep 1
kill -INT $PID
sleep 3
echo "=== Process finished ==="
```


**Наблюдение:** после размаскирования SIGINT обрабатывается и завершает процесс. Print после размаскирования не появляется — сигнал сработал сразу.

---

## 6. Порядок обработки

Меняем порядок нажатия комбинаций клавиш, наблюдаем порядок обработки.

```bash
python3 -c '
import signal, time, os

signal.pthread_sigmask(signal.SIG_BLOCK, range(1, 32))
print(f"PID: {os.getpid()}")

for i in range(3):
    pending = signal.sigpending()
    print(f"iteration {i}: pending = {pending}")
    time.sleep(1)

print("Unmasking...")
signal.pthread_sigmask(signal.SIG_UNBLOCK, range(1, 32))
print("Done")
' &
PID=$!
sleep 1
kill -TERM $PID
sleep 0.2
kill -INT $PID
sleep 0.2
kill -QUIT $PID
sleep 3
echo "=== Process finished ==="
```


**Наблюдение:** все три сигнала появляются в pending-наборе. Порядок обработки зависит от приоритета сигналов: SIGQUIT (3) обрабатывается первым, затем SIGINT (2), затем SIGTERM (15).

---

## 7. Игнорирование

Устанавливаем SIG_IGN до цикла. Наблюдаем, появляются ли игнорируемые сигналы в pending.

```bash
python3 -c '
import signal, time, os

signal.signal(signal.SIGINT, signal.SIG_IGN)
signal.signal(signal.SIGTERM, signal.SIG_IGN)
signal.pthread_sigmask(signal.SIG_BLOCK, range(1, 32))
print(f"PID: {os.getpid()}")
print("SIGINT and SIGTERM set to SIG_IGN, all signals blocked")

for i in range(3):
    pending = signal.sigpending()
    print(f"iteration {i}: pending = {pending}")
    time.sleep(1)

print("Unmasking...")
signal.pthread_sigmask(signal.SIG_UNBLOCK, range(1, 32))
print("Done - ignored signals should not appear in pending")
' &
PID=$!
sleep 1
kill -INT $PID
kill -TERM $PID
sleep 3
echo "=== Process finished ==="
```


**Наблюдение:** игнорируемые сигналы появляются в pending-наборе, но после размаскирования не обрабатываются — процесс не завершается.

---

## 8. Игнорирование после нажатия

Устанавливаем игнорирование после цикла, но перед размаскированием.

```bash
python3 -c '
import signal, time, os

signal.pthread_sigmask(signal.SIG_BLOCK, range(1, 32))
print(f"PID: {os.getpid()}")
print("All signals blocked. Send signals now...")

for i in range(3):
    pending = signal.sigpending()
    print(f"iteration {i}: pending = {pending}")
    time.sleep(1)

signal.signal(signal.SIGINT, signal.SIG_IGN)
signal.signal(signal.SIGTERM, signal.SIG_IGN)
print("Set SIGINT and SIGTERM to SIG_IGN")

print("Unmasking...")
signal.pthread_sigmask(signal.SIG_UNBLOCK, range(1, 32))
print("Done - ignored signals should be discarded")
' &
PID=$!
sleep 1
kill -INT $PID
kill -TERM $PID
sleep 3
echo "=== Process finished ==="
```


**Наблюдение:** сигналы, установленные в SIG_IGN перед размаскированием, отбрасываются — процесс не завершается.

---

## 9. Ребенок продолжает работать после завершения родителя

Форкаем ребенка, завершаем родителя до ребенка.

```bash
python3 -c '
import os, time

pid = os.fork()

if pid == 0:
    print(f"Child: PID={os.getpid()}, PPID={os.getppid()}")
    print("Child: sleeping 3 seconds...")
    time.sleep(3)
    print(f"Child: still alive, PPID={os.getppid()}")
    os._exit(0)
else:
    print(f"Parent: PID={os.getpid()}, child PID={pid}")
    print("Parent: exiting immediately")
' &
PARENT_PID=$!
sleep 1
echo "=== After parent exited ==="
ps -o pid,ppid,stat,cmd -p $PARENT_PID 2>/dev/null || echo "Parent process gone"
sleep 3
echo "=== Child should still be running ==="
ps -eo pid,ppid,stat,cmd | grep "still alive" | grep -v grep || echo "Child finished"
```


**Наблюдение:** ребенок продолжает работать после завершения родителя. После усыновления init/systemd ребенок завершается нормально.

---

## 10. Завершение терминала

Фоновые процессы в терминале. Что происходит при закрытии терминала?

```bash
python3 -c '
import os, time

print(f"PID: {os.getpid()}")
print("Running in background...")
for i in range(5):
    print(f"iteration {i}")
    time.sleep(1)
print("Done")
' &
BG_PID=$!
echo "=== Background process started ==="
ps -o pid,ppid,stat,cmd -p $BG_PID
sleep 2
echo "=== Process still running ==="
ps -o pid,ppid,stat,cmd -p $BG_PID
wait $BG_PID 2>/dev/null
echo "=== Process finished ==="
```


**Наблюдение:** при закрытии терминала фоновые процессы получают SIGHUP и завершаются. Чтобы этого избежать, можно использовать `nohup` или `disown`:

```bash
nohup python3 script.py &
disown
```

Или игнорировать SIGHUP:

```python
import signal
signal.signal(signal.SIGHUP, signal.SIG_IGN)
```
