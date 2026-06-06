---
title: "06 - File Signature Checker"
layout: default
---

## Цель

Реализовать shell-скрипт для определения типа файла по его **сигнатуре** (магическим байтам), а не по расширению имени файла.

## Входные данные

- Путь к файлу передаётся как **аргумент командной строки**.
- Конфигурационный файл `signatures.conf`, содержащий сигнатуры форматов.

**Формат строк конфигурации:**

<тип>|<сигнатура>

> Строки с `#` и пустые строки игнорируются.

## Требования к работе скрипта

1. Проверить существование файла.  
   Если файл отсутствует — вывести `File not found` и завершить работу.
2. Считать первые 16 байт файла (достаточно для всех сигнатур).
3. Преобразовать считанные байты в шестнадцатеричный вид.
4. Последовательно сравнить заголовок файла с сигнатурами из конфигурационного файла (в порядке строк).
5. Если совпадение найдено — вывести тип файла и завершить работу.
6. Если ни одна сигнатура не подошла — вывести `Unknown format`.

## Ограничения

- Не использовать расширение имени файла
- Запрещено использовать `sed` и `awk`
- Не читать входной поток несколько раз
- Сигнатуры **не захардкожены** в скрипте — только через `signatures.conf`

## Код скрипта (`check_file.sh`)

```bash
#!/bin/sh

CONFIG="signatures.conf"
FILE="$1"

if [ ! -f "$FILE" ]; then
    echo "File not found"
    exit 1
fi

HEADER=$(hexdump -n 16 -e '16/1 "%02x"' "$FILE")

while IFS="|" read -r format signature; do
    case "$format" in
        ""|\#*) continue ;;
    esac

    case "$HEADER" in
        $signature*)
            echo "$format"
            exit 0
            ;;
    esac
done < "$CONFIG"

echo "Unknown format"
```


Конфигурационный файл signatures.conf
# format|hex_signature

# Images
PNG|89504e470d0a1a0a
JPEG|ffd8ff
GIF|47494638
BMP|424d
WEBP|52494646
ICO|00000100

# Documents
PDF|25504446
RTF|7b5c727466
DOC|d0cf11e0a1b11ae1
DOCX|504b0304
ODT|504b0304

# Archives
ZIP|504b0304
RAR4|526172211a0700
RAR5|526172211a070100
7Z|377abcaf271c
GZIP|1f8b08
TAR|7573746172

# Executables
ELF|7f454c46
PE_EXE|4d5a

# Audio & Video
MP3|fffb
WAV|52494646
MP4|0000001866747970


Пример запуска:
# Семинар 6: Пример работы скрипта

**Пользователь:** taburetka


taburetka@DESKTOP-L5144K1:~$ ./check_all.sh
check_all.sh: Unknown format
Git и Github.pptx: DOCX
is_pdf.sh: Unknown format
photo.jpg: JPEG
signatures.conf: Unknown format
Без имени 3.odt: DOCX
Лабораторная работа 2.docx: DOCX
CEM2.pdf: PDF
taburetka@DESKTOP-L5144K1:~$




---
## Backlinks

- [[00 OS Labs MOC]]
- [[08 - sed and awk Exercises]]
- [[12-13 - Script Validation]]
