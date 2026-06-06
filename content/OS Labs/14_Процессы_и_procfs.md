---
title: "14 - Processes & procfs"
---

# Семинар 14. Работа с утилитами

Студент: mononoke

---

## 1. Утилита `ps`

Запуск `ps` с разными опциями:

```bash
ps
```


```bash
ps -f
```


```bash
ps -e
```


```bash
ps auxw
```


```bash
ps -o pid,ppid,stat,tty,cmd
```


```bash
ps -o pid,ppid,stat,etime,cmd -p $$
```


---

## 2. Процесс, ожидающий входа, и просмотр через `ps` из другого терминала

**Терминал 1**

```bash
python3 -c 'input("waiting for input> ")'
```

**Терминал 2**

```bash
ps -ef | grep '[p]ython3'
```


```bash
ps -o pid,ppid,stat,wchan,cmd -C python3
```


---

## 3. Фоновый процесс в том же терминале

```bash
sleep 100000 &
jobs -l
ps -o pid,ppid,stat,cmd -p $!
```


Альтернатива:

```bash
tail -f /dev/null &
jobs -l
ps -o pid,ppid,stat,cmd -p $!
```


---

## 4. Утилита `pstree`

```bash
pstree
```


```bash
pstree -p
```


```bash
pstree -p $$
```


```bash
pstree -sp $$
```


---

## 5. Определение PID текущего bash-процесса

```bash
echo "$BASHPID"
```


```bash
echo "$$"
```


```bash
python3 -c 'import os; print("python pid =", os.getpid()); print("bash pid =", os.getppid())'
```


---

## 6. Знакомство с `procfs`

```bash
ls /proc | head -n 50
```


```bash
ls -d /proc/[0-9]* | head
```


---

## 7. Python-скрипт: чтение файлов из собственного `/proc/<pid>`

```bash
python3 - <<'PY'
import os

pid = os.getpid()
path = f"/proc/{pid}/cmdline"

with open(path, "rb") as f:
    data = f.read()

print("pid:", pid)
print("raw:", data)
print("decoded argv:", data.replace(b"\x00", b" ").decode(errors="replace"))
PY
```


---

## 8. То же для родительского процесса

```bash
python3 - <<'PY'
import os

ppid = os.getppid()
path = f"/proc/{ppid}/cmdline"

with open(path, "rb") as f:
    data = f.read()

print("ppid:", ppid)
print("raw:", data)
print("decoded argv:", data.replace(b"\x00", b" ").decode(errors="replace"))
PY
```


---

## 9. Повторить задания 7 и 8 средствами `bash`

```bash
echo "self BASHPID=$BASHPID"
tr '\0' ' ' < /proc/$BASHPID/cmdline; echo

echo "self $$=$$"
tr '\0' ' ' < /proc/$$/cmdline; echo

echo "parent PPID=$PPID"
tr '\0' ' ' < /proc/$PPID/cmdline; echo
```


---

## 10. Python-скрипт: пройти до корневого процесса

```bash
python3 - <<'PY'
import os

def read_ppid(pid: int) -> int:
    with open(f"/proc/{pid}/status", "r", encoding="utf-8", errors="replace") as f:
        for line in f:
            if line.startswith("PPid:"):
                return int(line.split()[1])
    raise RuntimeError(f"PPid not found for pid {pid}")

def read_cmdline(pid: int) -> str:
    try:
        with open(f"/proc/{pid}/cmdline", "rb") as f:
            data = f.read()
        text = data.replace(b"\x00", b" ").decode(errors="replace").strip()
        return text or "[empty cmdline]"
    except FileNotFoundError:
        return "[process disappeared]"

pid = os.getpid()
seen = set()

while pid > 0 and pid not in seen:
    seen.add(pid)
    print(f"pid={pid} cmdline={read_cmdline(pid)}")
    if pid == 1:
        break
    pid = read_ppid(pid)
PY
```


---

## 11. Python-скрипт: ближайший общий предок двух PID

```bash
python3 - <<'PY'
import os
import sys

def exists(pid: int) -> bool:
    return os.path.isdir(f"/proc/{pid}")

def get_ppid(pid: int) -> int:
    with open(f"/proc/{pid}/status", "r", encoding="utf-8", errors="replace") as f:
        for line in f:
            if line.startswith("PPid:"):
                return int(line.split()[1])
    raise RuntimeError(f"PPid not found for pid {pid}")

def ancestors(pid: int):
    chain = []
    seen = set()
    while pid > 0 and pid not in seen and exists(pid):
        chain.append(pid)
        seen.add(pid)
        if pid == 1:
            break
        pid = get_ppid(pid)
    return chain

if len(sys.argv) != 3:
    print("usage: python3 lca.py PID1 PID2")
    sys.exit(1)

pid1 = int(sys.argv[1])
pid2 = int(sys.argv[2])

if not exists(pid1):
    print(f"pid {pid1} does not exist")
    sys.exit(2)

if not exists(pid2):
    print(f"pid {pid2} does not exist")
    sys.exit(2)

a1 = ancestors(pid1)
a2 = ancestors(pid2)

s2 = set(a2)
for p in a1:
    if p in s2:
        print("nearest common ancestor:", p)
        break
else:
    print("no common ancestor found")
PY 1 $$
```


Для произвольных PID:

```bash
python3 lca.py 1234 5678
```


---

## 12. Реализовать задания 10 и 11 на `bash`

### 12.1. Все предки текущего процесса

```bash
pid=$BASHPID
while [[ "$pid" -gt 0 && -r /proc/$pid/status ]]; do
    printf 'pid=%s cmdline=' "$pid"
    tr '\0' ' ' < /proc/$pid/cmdline
    echo
    [[ "$pid" -eq 1 ]] && break
    ppid=$(grep '^PPid:' /proc/$pid/status | awk '{print $2}')
    pid=$ppid
done
```


### 12.2. Ближайший общий предок двух PID

```bash
pid1=$$
pid2=$PPID

declare -A anc
p=$pid1
while [[ "$p" -gt 0 && -r /proc/$p/status ]]; do
    anc[$p]=1
    [[ "$p" -eq 1 ]] && break
    p=$(grep '^PPid:' /proc/$p/status | awk '{print $2}')
done

p=$pid2
while [[ "$p" -gt 0 && -r /proc/$p/status ]]; do
    if [[ -n "${anc[$p]}" ]]; then
        echo "nearest common ancestor: $p"
        break
    fi
    [[ "$p" -eq 1 ]] && break
    p=$(grep '^PPid:' /proc/$p/status | awk '{print $2}')
done
```


---

## 13. Файл `maps`

```bash
cat /proc/$$/maps
```


Или компактно:

```bash
grep -E '\[heap\]|\[stack\]|vdso|python|bash|libc' /proc/$$/maps
```


---

## 14. Python-скрипт: объём памяти, выделенной под кучу

```bash
python3 - <<'PY'
import sys

pid = sys.argv[1] if len(sys.argv) > 1 else "self"
if pid == "self":
    import os
    pid = str(os.getpid())

total = 0

with open(f"/proc/{pid}/maps", "r", encoding="utf-8", errors="replace") as f:
    for line in f:
        if line.rstrip().endswith("[heap]"):
            addr = line.split()[0]
            start_s, end_s = addr.split("-")
            start = int(start_s, 16)
            end = int(end_s, 16)
            total += end - start
            print(line.rstrip())

print(f"heap bytes = {total}")
print(f"heap KiB   = {total / 1024:.1f}")
print(f"heap MiB   = {total / 1024 / 1024:.3f}")
PY
```


---

## 15. Файл `pagemap`

```bash
ls -l /proc/$$/pagemap
```


Проверить размер записи:

```bash
python3 - <<'PY'
import os
print("page size:", os.sysconf("SC_PAGE_SIZE"))
print("pagemap entry size: 8 bytes")
PY
```


---

## 16. Проверка утверждения про `id()` и поиск адреса в `maps`

```bash
python3 - <<'PY'
import os

obj = [1, 2, 3]
addr = id(obj)
pid = os.getpid()

print("pid =", pid)
print("id(obj) =", hex(addr))

with open(f"/proc/{pid}/maps", "r", encoding="utf-8", errors="replace") as f:
    for line in f:
        rng = line.split()[0]
        start_s, end_s = rng.split('-')
        start = int(start_s, 16)
        end = int(end_s, 16)
        if start <= addr < end:
            print("address belongs to mapping:")
            print(line.rstrip())
            break
PY
```


---

## 17. Найти этот адрес в `pagemap`

```bash
python3 - <<'PY'
import os
import struct

obj = [1, 2, 3]
addr = id(obj)
pid = os.getpid()
page_size = 4096

page_index = addr // page_size
offset = page_index * 8

print("pid       =", pid)
print("obj addr  =", hex(addr))
print("page idx  =", page_index)
print("pm offset =", offset)

with open(f"/proc/{pid}/pagemap", "rb") as f:
    f.seek(offset)
    data = f.read(8)

value, = struct.unpack("<Q", data)
print("raw 8 bytes =", data.hex())
print("value       =", hex(value))
PY
```


---

## 18. Интерпретация записи `pagemap`

```bash
python3 - <<'PY'
import os
import struct

obj = [1, 2, 3]
addr = id(obj)
pid = os.getpid()
page_size = 4096
offset = (addr // page_size) * 8

with open(f"/proc/{pid}/pagemap", "rb") as f:
    f.seek(offset)
    raw = f.read(8)

value, = struct.unpack("<Q", raw)

present = (value >> 63) & 1
swapped = (value >> 62) & 1
file_or_shared_anon = (value >> 61) & 1
uffd_wp = (value >> 57) & 1
exclusive = (value >> 56) & 1
soft_dirty = (value >> 55) & 1
pfn = value & ((1 << 55) - 1)

print(f"pid={pid}")
print(f"obj addr={hex(addr)}")
print(f"raw={raw.hex()}")
print(f"value={hex(value)}")
print(f"present={present}")
print(f"swapped={swapped}")
print(f"file/shared_anon={file_or_shared_anon}")
print(f"userfaultfd_wp={uffd_wp}")
print(f"exclusive={exclusive}")
print(f"soft_dirty={soft_dirty}")
print(f"pfn={hex(pfn)}")
PY
```


---

## Backlinks

- [[00 OS Labs MOC]]
- [[09-10 - Users and Permissions]]
