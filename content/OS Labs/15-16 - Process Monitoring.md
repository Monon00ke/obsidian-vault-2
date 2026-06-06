---
title: "15-16 - Process Monitoring"
---

# Семинар 15-16. Наблюдение за процессами

Студент: mononoke

---

## 1. Разобрать опции `ps`, особенно `-o`

Изучить основные опции `ps`, особенно `-o` для кастомного вывода. Обратить внимание на: потребление памяти, время запуска, командную строку, время выполнения, объём I/O.

```bash
ps -eo user,pid,pmem,etime,cmd --sort=-pmem | head -20
```


Ключевые поля:
- `%MEM` — потребление физической памяти процессом
- `ELAPSED` — время выполнения (etime)
- `CMD` — командная строка
- `RSS` / `VSZ` — физическая / виртуальная память

---

## 2. Все пользователи с активными процессами

```bash
ps -eo user --no-headers | sort -u
```


---

## 3. Пользователи + количество процессов каждого

```bash
ps -eo user --no-headers | sort | uniq -c | sort -rn
```


Больше всего процессов у `root` (219) и `mononoke` (124).

---

## 4. Пользователи + кол-во процессов + сортировка по времени запуска

```bash
ps -eo user,lstart --no-headers | sort -k6,6n -k3,3M -k4,4n | \
  awk '{user=$1; if (!(user in seen)) {seen[user]=$0; print}}'
```


Первым стартовал `root` (11:51:27), последним — `mononoke` (11:51:52).

---

## 5. Два Python-скрипта: пустой цикл vs `time.sleep(1000)`

**Скрипт 1 — пустой бесконечный цикл:**

```bash
python3 -c '
import time
while True:
    pass
' &
PID1=$!
sleep 2
ps -o pid,stat,pcpu,pmem,etime,cmd -p $PID1
kill $PID1
```


**Скрипт 2 — `time.sleep(1000)` в цикле:**

```bash
python3 -c '
import time
while True:
    time.sleep(1000)
' &
PID2=$!
sleep 2
ps -o pid,stat,pcpu,pmem,etime,cmd -p $PID2
kill $PID2
```


**Сравнение:**
| Параметр | Пустой цикл | sleep(1000) |
|---|---|---|
| STAT | `R` (running) | `S` (sleeping) |
| %CPU | 100% | 0.5% |
| %MEM | 0.0% | 0.0% |

Пустой цикл полностью загружает одно ядро CPU (100%), тогда как `sleep` переводит процесс в состояние сна (STAT=S) с минимальным потреблением CPU.

---

## 6. Скрипт: чтение и запись больших файлов

```bash
python3 -c '
import os, time

data = "x" * (50 * 1024 * 1024)  # 50 MB
with open("/tmp/test_large_write.txt", "w") as f:
    f.write(data)
print(f"Written {len(data)} bytes, PID={os.getpid()}")
time.sleep(10)
with open("/tmp/test_large_write.txt", "r") as f:
    content = f.read()
print(f"Read {len(content)} bytes")
time.sleep(5)
' &
PID3=$!
sleep 2
echo "=== Во время записи/сна ==="
ps -o pid,pmem,rss,vsz,cmd -p $PID3
sleep 8
echo "=== Во время чтения/сна ==="
ps -o pid,pmem,rss,vsz,cmd -p $PID3
wait $PID3
rm -f /tmp/test_large_write.txt
```


RSS ≈ 60 MB, что соответствует размеру данных (50 MB) + оверхед Python.

---

## 7. Скрипт: список растёт в цикле → рост памяти

```bash
python3 -c '
import os, time

lst = []
print(f"PID={os.getpid()}")
for i in range(5):
    lst.extend([0] * 10_000_000)
    print(f"iteration {i+1}, len={len(lst)}")
    time.sleep(2)
' &
PID4=$!
sleep 1
echo "=== После итерации 1 ==="
ps -o pid,pmem,rss,vsz,cmd -p $PID4
sleep 4
echo "=== После итерации 3 ==="
ps -o pid,pmem,rss,vsz,cmd -p $PID4
sleep 4
echo "=== После итерации 5 ==="
ps -o pid,pmem,rss,vsz,cmd -p $PID4
wait $PID4
```


**Рост памяти:**
| Итерация | RSS (KB) | VSZ (KB) | %MEM |
|---|---|---|---|
| 1 | 87 760 | 107 236 | 0.5% |
| 3 | 244 008 | 263 484 | 1.5% |
| 5 | 400 260 | 419 736 | 2.4% |

Память растёт линейно с каждой итерацией — каждый `extend([0] * 10_000_000)` добавляет ~80 MB RSS.

---

## 8. Fork: родитель в цикле, ребёнок завершается → зомби

```bash
python3 -c '
import os, time

pid = os.fork()
if pid == 0:
    print(f"Child: PID={os.getpid()}, PPID={os.getppid()}")
    time.sleep(2)
    print("Child: exiting")
    os._exit(0)
else:
    print(f"Parent: PID={os.getpid()}, child PID={pid}")
    time.sleep(1)
    print("Parent: checking child before it exits...")
    os.system(f"ps -o pid,ppid,stat,cmd -p {pid}")
    time.sleep(3)
    print("Parent: checking child after it exited (zombie state)...")
    os.system(f"ps -o pid,ppid,stat,cmd -p {pid}")
    time.sleep(1)
    print("Parent: exiting")
'
```


До завершения ребёнка: `STAT=S` (sleeping). После завершения: `STAT=Z` (zombie), командная строка заменена на `[python3] <defunct>`.

---

## 9. Несколько дочерних процессов → несколько зомби → убиваем родителя

```bash
python3 -c '
import os, time

children = []
for i in range(3):
    pid = os.fork()
    if pid == 0:
        print(f"Child {i}: PID={os.getpid()}, exiting quickly")
        os._exit(0)
    else:
        children.append(pid)
        print(f"Parent: spawned child {i} with PID={pid}")

print(f"Parent: PID={os.getpid()}, waiting 3 seconds to see zombies...")
time.sleep(3)
print("Parent: checking children (should be zombies)...")
for c in children:
    os.system(f"ps -o pid,ppid,stat,cmd -p {c} 2>/dev/null")
print("Parent: exiting now (zombies will be reaped by init)...")
'
```


Все три ребёнка в состоянии `Z` (зомби). После завершения родителя:


Init/systemd подбирает осиротевших зомби и очищает их.

---

## 10. Родитель вызывает `os.wait()` — зомби не накапливаются

```bash
python3 -c '
import os, time

children = []
for i in range(3):
    pid = os.fork()
    if pid == 0:
        print(f"Child {i}: PID={os.getpid()}, sleeping 2s then exiting")
        time.sleep(2)
        print(f"Child {i}: exiting")
        os._exit(0)
    else:
        children.append(pid)
        print(f"Parent: spawned child {i} with PID={pid}")

print(f"Parent: PID={os.getpid()}, calling os.wait() with timeout...")
for i in range(10):
    try:
        wpid, status = os.waitpid(-1, os.WNOHANG)
        if wpid > 0:
            print(f"Parent: reaped child PID={wpid}, status={status}")
        else:
            print(f"Parent: no child exited yet (iteration {i+1})")
    except ChildProcessError:
        print("Parent: all children reaped")
        break
    time.sleep(1)

print("Parent: exiting")
'
```


Родитель вызывает `os.waitpid(-1, os.WNOHANG)` в цикле с `time.sleep(1)`. Дети завершаются через 2 секунды, и родитель тут же их «подбирает» — зомби не появляются.

---

## 11. Перевёрнутый сценарий: дети вечные, родитель завершается

```bash
python3 -c '
import os, time

children = []
for i in range(3):
    pid = os.fork()
    if pid == 0:
        print(f"Child {i}: PID={os.getpid()}, PPID={os.getppid()}, running forever...")
        while True:
            time.sleep(60)
    else:
        children.append(pid)
        print(f"Parent: spawned child {i} with PID={pid}")

print(f"Parent: PID={os.getpid()}, will exit in 5 seconds...")
time.sleep(5)
print("Parent: exiting")
' &
PARENT_PID=$!
sleep 7
echo "=== После завершения родителя, проверяем новый PPID детей ==="
ps -eo pid,ppid,stat,cmd | grep "running forever" | grep -v grep
```


После завершения родителя (PID=7945) все три ребёнка были усыновлены процессом `systemd --user` (PID=2079). PPID изменился с 7945 на 2079.****