
---

## 1. Определение точек подключения

Утилиты `ip addr` и `ip route` для просмотра сетевых интерфейсов и маршрутов.

```bash

ip addr show

ip route show

```


**Наблюдение:**

- `lo` — loopback (127.0.0.1)

- `wlp0s20f3` — Wi-Fi (192.168.43.213/24)

- `lxcbr0` — LXC мост (10.0.3.1/24)

- `tun0` — VPN туннель (172.18.0.1/30)

- Шлюз по умолчанию: 192.168.43.1

---

## 2. Простой TCP-сервер

Сервер слушает на TCP-порту, принимает соединение и печатает запрос.

```bash

python3 -c "

import socket, os, select, time

srv = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

srv.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)

srv.bind(('127.0.0.1', 8888))

srv.listen(5)

print(f'Serving on 127.0.0.1:8888')

srv.setblocking(False)

for _ in range(40):

r, _, _ = select.select([srv], [], [], 0.2)

if r:

conn, addr = srv.accept()

data = conn.recv(4096)

print(data.decode(errors='replace'))

conn.close()

break

srv.close()

" &

curl -s http://127.0.0.1:8888/abc

```


**Наблюдение:** curl отправляет HTTP GET-запрос, сервер выводит его содержимое:

```

GET /abc HTTP/1.1

Host: 127.0.0.1:8888

User-Agent: curl/8.5.0

Accept: */*

```

---

## 3. TCP-клиент к HTTP-серверу

Клиент подключается к `example.com:80`, отправляет GET-запрос и выводит ответ.

```bash

python3 -c "

import socket

s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

s.settimeout(5)

s.connect(('example.com', 80))

s.sendall(b'GET / HTTP/1.0\r\nHost: example.com\r\nConnection: close\r\n\r\n')

resp = b''

while True:

chunk = s.recv(4096)

if not chunk: break

resp += chunk

print(resp.decode(errors='replace')[:400])

s.close()

"

```


**Наблюдение:** получен HTTP-ответ с кодом 200, содержимым HTML-страницы, заголовками (Date, Content-Type, Server и т.д.).

---

## 4. Простой HTTP-сервер для файлов

Сервер принимает GET-запросы и отдаёт содержимое файлов из каталога.

```bash

mkdir -p /tmp/test_a

echo "Hello from a/b.txt" > /tmp/test_a/b.txt

python3 -c "

import socket, os, select

def handle(conn, addr):

data = conn.recv(4096)

req = data.decode()

lines = req.split(chr(13)+chr(10))

parts = lines[0].split()

path = parts[1].lstrip('/')

filepath = '/tmp/test_a/' + path

if os.path.isfile(filepath):

content = open(filepath, 'rb').read()

resp = 'HTTP/1.1 200 OK\r\nContent-Length: {}\r\nContent-Type: text/plain\r\n\r\n'.format(len(content))

conn.sendall(resp.encode() + content)

else:

conn.sendall(b'HTTP/1.1 404 Not Found\r\n\r\n404')

conn.close()

srv = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

srv.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)

srv.bind(('127.0.0.1', 8889))

srv.listen(5)

srv.setblocking(False)

for _ in range(20):

r, _, _ = select.select([srv], [], [], 0.5)

if r:

conn, addr = srv.accept()

handle(conn, addr)

break

srv.close()

" &

curl -s http://127.0.0.1:8889/b.txt

```


**Наблюдение:** сервер отдаёт содержимое `b.txt` по GET-запросу. На несуществующие файлы отвечает 404.

---

## 5. SSH-ключи и сервер

Установка SSH-сервера, генерация ключей, настройка аутентификации.

```bash

# Генерация ключей

ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -N ""

# Публичный ключ

cat ~/.ssh/id_ed25519.pub

# Добавление в authorized_keys

cat ~/.ssh/id_ed25519.pub >> ~/.ssh/authorized_keys

# Проверка подключения

ssh mononoke@localhost whoami

```


**Наблюдение:** SSH-сервер не установлен (ожидаемо), ключи сгенерированы (ed25519). При запущенном сервере и настроенных ключах логин по публичному ключу будет работать без пароля.

---

## 6. Генерация пары ключей

```bash

ssh-keygen -t ed25519 -f /tmp/test_ssh_key -N ""

```


**Наблюдение:** созданы два файла: приватный ключ (`test_ssh_key`) и публичный (`test_ssh_key.pub`). Публичный ключ имеет формат: `ssh-ed25519 <base64> <comment>`.

---

## 7. Регистрация ключей

Добавление публичного ключа в `~/.ssh/authorized_keys` для беспарольного входа.

```bash

cat ~/.ssh/id_ed25519.pub >> ~/.ssh/authorized_keys

chmod 600 ~/.ssh/authorized_keys

ssh mononoke@localhost

```


**Наблюдение:** после добавления ключа в `authorized_keys` и при запущенном SSH-сервере соединение устанавливается без запроса пароля.

---

## 8. Игнорирование SIGHUP (stablerun)

Утилита `stablerun.py` игнорирует SIGHUP и через `exec` запускает переданную команду.

```bash

#!/usr/bin/env python3

import signal, os, sys

signal.signal(signal.SIGHUP, signal.SIG_IGN)

os.execvp(sys.argv[1], sys.argv[1:])

# Использование:

./stablerun.py longrun 1 2 3

```


**Наблюдение:** SIGHUP игнорируется, поэтому процесс продолжает работу после завершения сеанса. Однако это не сработает, если запускаемая программа сама устанавливает обработку SIGHUP на завершение.

---

## 9. Новый сеанс (setsid)

Запуск процесса в собственном сеансе без управляющего терминала через `setsid()`.

```bash

python3 -c "

import os, time, signal

pid = os.fork()

if pid == 0:

os.setsid() # Новый сеанс, без управляющего терминала

signal.signal(signal.SIGHUP, signal.SIG_DFL)

for i in range(3):

print(f'Child iteration {i}')

time.sleep(1)

os._exit(0)

else:

time.sleep(5)

os.waitpid(pid, 0)

"

```


**Наблюдение:** ребёнок создаёт новый сеанс через `os.setsid()`, теряет управляющий терминал и не получает SIGHUP при завершении родительского сеанса.

---

## 10. Утилита lsof

Просмотр открытых файлов процессов, сетевых соединений и сокетов.

```bash

# Открытые файлы текущего bash

lsof -p $$

# Сетевые соединения

lsof -i

# Файлы PID 1

lsof -p 1

```


**Наблюдение:** `lsof` показывает все открытые файловые дескрипторы (включая сокеты, пайпы, inotify) для каждого процесса. Сетевая опция `-i` фильтрует только сетевые соединения (TCP, UDP). Позволяет диагностировать, какие порты слушаются и какие соединения установлены.

---

## См. также
- [[00 OS Labs MOC|К оглавлению OS Labs]]
