---
title: "17 — Pipes & Fork"
---

## 1. Простейшее общение с ребенком

Скрипт создает pipe, запускает ребенка, пишет байт. Ребенок читает байт из pipe.

```bash
python3 -c '
import os, time

r, w = os.pipe()
pid = os.fork()

if pid == 0:
    os.close(w)
    data = os.read(r, 1)
    print(f"Child: read byte {data}")
    os.close(r)
    os._exit(0)
else:
    os.close(r)
    os.write(w, b"A")
    print(f"Parent: wrote byte A")
    os.close(w)
    os.waitpid(pid, 0)
    print("Parent: child finished")
'
```


---

## 2. Ребенок читает не сразу

Родитель пишет сразу, ребенок ждет 3 секунды перед чтением. Наблюдаем, будет ли родитель ждать.

```bash
python3 -c '
import os, time

r, w = os.pipe()
pid = os.fork()

if pid == 0:
    print(f"Child: sleeping 3 seconds before read")
    time.sleep(3)
    data = os.read(r, 1)
    print(f"Child: read byte {data} at {time.time()}")
    os.close(r)
    os._exit(0)
else:
    os.close(r)
    print(f"Parent: writing byte at {time.time()}")
    os.write(w, b"B")
    print(f"Parent: write returned at {time.time()}")
    os.close(w)
    os.waitpid(pid, 0)
    print("Parent: child finished")
'
```


**Наблюдение:** `write` возвращается мгновенно — родитель не ждет ребенка. Буфер pipe достаточно велик, чтобы вместить один байт.

---

## 3. Родитель пишет не сразу

Перевернутый эксперимент: ребенок пытается читать сразу, родитель ждет 3 секунды перед записью.

```bash
python3 -c '
import os, time

r, w = os.pipe()
pid = os.fork()

if pid == 0:
    print(f"Child: trying to read at {time.time()}")
    data = os.read(r, 1)
    print(f"Child: read byte {data} at {time.time()}")
    os.close(r)
    os._exit(0)
else:
    os.close(r)
    print(f"Parent: sleeping 3 seconds before write")
    time.sleep(3)
    print(f"Parent: writing byte at {time.time()}")
    os.write(w, b"C")
    print(f"Parent: write returned at {time.time()}")
    os.close(w)
    os.waitpid(pid, 0)
    print("Parent: child finished")
'
```


**Наблюдение:** ребенок блокируется на `read()` до тех пор, пока родитель не запишет данные.

---

## 4. Циклическая запись и чтение

Родитель пишет по одному байту с таймаутом, ребенок читает по одному байту с таймаутом.

```bash
python3 -c '
import os, time

r, w = os.pipe()
pid = os.fork()

if pid == 0:
    os.close(w)
    for i in range(5):
        data = os.read(r, 1)
        print(f"Child: read byte {data} (iteration {i+1})")
        time.sleep(0.5)
    os.close(r)
    os._exit(0)
else:
    os.close(r)
    for i in range(5):
        byte = bytes([65 + i])
        os.write(w, byte)
        print(f"Parent: wrote byte {byte} (iteration {i+1})")
        time.sleep(0.5)
    os.close(w)
    os.waitpid(pid, 0)
    print("Parent: child finished")
'
```


**Наблюдение:** поток данных синхронизирован — родитель пишет, ребенок читает, таймауты совпадают.

---

## 5. Родитель пишет по 1000 байт, ребенок читает по 1

Родитель пишет порциями по 1000 байт, ребенок читает по одному байту. Предсказание: низкая пропускная способность ребенка не повлияет на родителя, пока буфер pipe не заполнится.

```bash
python3 -c '
import os, time

r, w = os.pipe()
pid = os.fork()

if pid == 0:
    os.close(w)
    total = 0
    for i in range(10):
        data = os.read(r, 1)
        total += 1
        print(f"Child: read {len(data)} byte, total={total}")
        time.sleep(0.3)
    os.close(r)
    os._exit(0)
else:
    os.close(r)
    for i in range(3):
        chunk = bytes([65 + i]) * 1000
        os.write(w, chunk)
        print(f"Parent: wrote {len(chunk)} bytes (chunk {i+1})")
        time.sleep(0.3)
    os.close(w)
    os.waitpid(pid, 0)
    print("Parent: child finished")
'
```


**Наблюдение:** родитель записал 3000 байт, ребенок прочитал только 10. Буфер pipe (обычно 64 KB) вместил все данные, поэтому родитель не заблокировался.

---

## 6. Случайные байты и таймауты

Родитель пишет случайное число байтов со случайными таймаутами, ребенок читает случайное число байтов со случайными таймаутами. Матожидание скорости передачи ниже, чем матожидание запроса чтения.

```bash
python3 -c '
import os, time, random

r, w = os.pipe()
pid = os.fork()

if pid == 0:
    os.close(w)
    random.seed(42)
    for i in range(5):
        n = random.randint(1, 5)
        data = os.read(r, n)
        print(f"Child: requested {n}, got {len(data)} bytes")
        time.sleep(random.uniform(0.1, 0.5))
    os.close(r)
    os._exit(0)
else:
    os.close(r)
    random.seed(100)
    for i in range(5):
        n = random.randint(1, 3)
        chunk = bytes([65 + i]) * n
        os.write(w, chunk)
        print(f"Parent: wrote {n} bytes")
        time.sleep(random.uniform(0.2, 0.6))
    os.close(w)
    os.waitpid(pid, 0)
    print("Parent: child finished")
'
```


**Наблюдение:** ребенок получает данные по мере их поступления. Если запрошено больше байтов, чем доступно, `read` возвращает то, что есть.

---

## 7. Общение one-to-many

Родитель создает pipe, форкает n детей. Родитель пишет байты, дети читают по одному байту.

```bash
python3 -c '
import os, time

r, w = os.pipe()
children = []
N = 3

for i in range(N):
    pid = os.fork()
    if pid == 0:
        os.close(w)
        for j in range(3):
            data = os.read(r, 1)
            print(f"Child {i}: read byte {data}")
            time.sleep(0.2)
        os.close(r)
        os._exit(0)
    else:
        children.append(pid)

os.close(r)
for i in range(9):
    byte = bytes([65 + i])
    os.write(w, byte)
    print(f"Parent: wrote byte {byte}")
    time.sleep(0.3)
os.close(w)

for pid in children:
    os.waitpid(pid, 0)
print("Parent: all children finished")
'
```


**Наблюдение:** байты распределяются между детьми — каждый ребенок получает некоторые байты из общего потока.

---

## 8. Дети читают разное количество байтов

Дети читают не по одному байту, а разное количество. Разные таймауты и размеры запросов.

```bash
python3 -c '
import os, time

r, w = os.pipe()
children = []
N = 3

for i in range(N):
    pid = os.fork()
    if pid == 0:
        os.close(w)
        read_size = (i + 1) * 2
        for j in range(2):
            data = os.read(r, read_size)
            print(f"Child {i}: requested {read_size}, got {len(data)} bytes: {data}")
            time.sleep(0.3)
        os.close(r)
        os._exit(0)
    else:
        children.append(pid)

os.close(r)
for i in range(12):
    byte = bytes([65 + i])
    os.write(w, byte)
    print(f"Parent: wrote byte {byte}")
    time.sleep(0.2)
os.close(w)

for pid in children:
    os.waitpid(pid, 0)
print("Parent: all children finished")
'
```


**Наблюдение:** дети читают разное количество байтов. `BrokenPipeError` возникает, когда дети завершаются и закрывают свои концы pipe, а родитель продолжает писать.

---

## 9. Двустороннее общение

Два канала: родитель → ребенок и ребенок → родитель. Схема запрос-ответ.

```bash
python3 -c '
import os, time

r1, w1 = os.pipe()
r2, w2 = os.pipe()

pid = os.fork()

if pid == 0:
    os.close(w1)
    os.close(r2)
    for i in range(3):
        request = os.read(r1, 100)
        print(f"Child: received request: {request}")
        response = f"response_{i}".encode()
        os.write(w2, response)
        print(f"Child: sent response: {response}")
        time.sleep(0.3)
    os.close(r1)
    os.close(w2)
    os._exit(0)
else:
    os.close(r1)
    os.close(w2)
    for i in range(3):
        request = f"request_{i}".encode()
        os.write(w1, request)
        print(f"Parent: sent request: {request}")
        response = os.read(r2, 100)
        print(f"Parent: received response: {response}")
        time.sleep(0.3)
    os.close(w1)
    os.close(r2)
    os.waitpid(pid, 0)
    print("Parent: child finished")
'
```


---

## 10. Дедлок в двустороннем общении

Дедлок возникает, когда оба процесса становятся писателями в заполненные каналы одновременно.

```bash
python3 -c '
import os, time

r1, w1 = os.pipe()
r2, w2 = os.pipe()

pid = os.fork()

if pid == 0:
    os.close(w1)
    os.close(r2)
    request = os.read(r1, 100)
    print(f"Child: received request: {request}")
    big_response = b"X" * 70000
    print(f"Child: trying to write {len(big_response)} bytes...")
    os.write(w2, big_response)
    print(f"Child: write completed")
    os.close(r1)
    os.close(w2)
    os._exit(0)
else:
    os.close(r1)
    os.close(w2)
    request = b"request"
    os.write(w1, request)
    print(f"Parent: sent request")
    time.sleep(1)
    print(f"Parent: trying to write more data...")
    more_data = b"Y" * 70000
    os.write(w1, more_data)
    print(f"Parent: write completed")
    response = os.read(r2, 100000)
    print(f"Parent: received response: {len(response)} bytes")
    os.close(w1)
    os.close(r2)
    os.waitpid(pid, 0)
    print("Parent: child finished")
'
```

Проверка дедлока:


**Наблюдение:** процесс завис в состоянии `S` (sleeping) — дедлок. Ребенок пытается записать 70 KB в pipe2, родитель пытается записать 70 KB в pipe1. Оба буфера заполняются, и оба процесса блокируются на `write()`, ожидая, пока другая сторона прочитает. Но никто не читает — дедлок.
