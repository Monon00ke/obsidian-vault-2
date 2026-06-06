---
title: "11 - Filesystem Pseudodisk"
layout: default
---


````
## **1. Создание псевдодиска**

Создать файл, имитирующий диск, заполненный нулями.

```bash
head -c 20M </dev/zero > disk.img
````

или

Bash

```
dd if=/dev/zero of=disk.img bs=1M count=20
```
---

## **2. Анализ содержимого до форматирования**

Проверить содержимое псевдодиска до форматирования.

Bash

```
od -x disk.img | head
```

---

## **3. Форматирование псевдодиска (ext2)**

Отформатировать файл как файловую систему ext2.

Bash

```
mkfs.ext2 disk.img
```
---

## **4. Анализ содержимого после форматирования**

Проверить изменения в файле после форматирования.

Bash

```
od -x disk.img | head
```
---

## **5. Монтирование псевдодиска**

Смонтировать псевдодиск как файловую систему.

Bash

```
mkdir -p /mnt/testfs
mount -o loop disk.img /mnt/testfs
```
---

## **6. Работа с файловой системой**

Создать файлы и каталоги внутри смонтированной файловой системы.

Bash

```
touch /mnt/testfs/file1
mkdir /mnt/testfs/dir1
cp /etc/passwd /mnt/testfs/

ls -l /mnt/testfs
```
---

## **7. Размонтирование и повторное монтирование**

Размонтировать и смонтировать файловую систему в другую точку.

Bash

```
umount /mnt/testfs

mkdir -p /mnt/testfs2
mount -o loop disk.img /mnt/testfs2
```

---

## **8. Анализ структуры ext2**

Изучить внутреннюю структуру файловой системы.

Bash

```
dumpe2fs disk.img | less
```
---

## **9. Эксперимент с заполнением файловой системы**

Создать множество файлов и определить ограничения.

Bash

```
#!/bin/bash

i=3
while true
do
  touch /mnt/testfs/file_$i 2>/dev/null || break
  i=$((i+1))
done

echo "Создано файлов: $i"
```
---

## **10. Автоматизация**

Скрипт, создающий псевдодиск, форматирующий его, монтирующий и копирующий файлы.

Bash

```
#!/bin/bash

DISK=$1
SIZE=$2
SRC=$3

dd if=/dev/zero of=$DISK bs=1M count=$SIZE

mkfs.ext2 $DISK

mkdir -p /mnt/auto
mount -o loop $DISK /mnt/auto

cp -r $SRC/* /mnt/auto/

umount /mnt/auto
```

Запуск:

Bash

```
bash script.sh disk.img 20 /etc
```

---
## Backlinks

- [[00 OS Labs MOC]]
