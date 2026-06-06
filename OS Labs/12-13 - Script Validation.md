---
title: "12-13 - Script Validation"
layout: default
---
````
### 1. Скрипт `lines-range.sh`: два параметра — начальная и конечная строка, вывод диапазона и завершение сразу после конечной строки

Я создал файл `lines-range.sh` со следующим содержимым:

```bash
#!/usr/bin/env bash

start=$1
end=$2

sed -n "$$   {start},   $${end}p;${end}q"
````

Пример запуска:

Bash

```
cat file.txt | ./lines-range.sh 5 10
```


### 2. Проверка количества аргументов: если аргументов не 2, вывести сообщение в stderr и завершиться с ненулевым кодом

Я улучшил скрипт, добавив проверку количества аргументов:

Bash

```
#!/usr/bin/env bash

prog="${0##*/}"

usage() {
    cat >&2 <<EOF
Использование:
  $prog START END

Печатает строки со START по END включительно из стандартного входа.
EOF
}

if [ "$#" -ne 2 ]; then
    echo "Ошибка: нужно ровно 2 аргумента." >&2
    usage
    exit 1
fi

start=$1
end=$2

sed -n "$$   {start},   $${end}p;${end}q"
```

Пример запуска с ошибкой:

Bash

```
./lines-range.sh 5
```
---

### 3. Проверка, что оба аргумента — целые числа, через grep

Я добавил проверку аргументов на целые числа:

Bash

```
#!/usr/bin/env bash

prog="${0##*/}"

usage() {
    cat >&2 <<EOF
Использование:
  $prog START END

START и END должны быть целыми положительными числами.
EOF
}

if [ "$#" -ne 2 ]; then
    echo "Ошибка: нужно ровно 2 аргумента." >&2
    usage
    exit 1
fi

for arg in "$1" "$2"; do
    if ! printf '%s\n' "$$   arg" | grep -Eq '^[0-9]+   $$'; then
        echo "Ошибка: аргументы должны быть целыми неотрицательными числами." >&2
        usage
        exit 2
    fi
done

start=$1
end=$2

sed -n "$$   {start},   $${end}p;${end}q"
```

Пример с некорректным аргументом:

Bash

```
./lines-range.sh 5 abc
```


### 4. Унифицированная проверка параметров по регулярным выражениям и своим сообщениям об ошибках

Я реализовал обобщённую функцию валидации аргументов:

Bash

```
#!/usr/bin/env bash

prog="${0##*/}"

usage() {
    cat >&2 <<EOF
Использование:
  $prog ARG1 ARG2

ARG1: только латинские буквы
ARG2: только цифры
EOF
}

die() {
    echo "$1" >&2
    usage
    exit 1
}

validate_args() {
    local expected_count=0
    local i=1
    while IFS=$'\t' read -r regex message; do
        [ -z "$regex" ] && continue
        expected_count=$((expected_count + 1))

        eval "arg=\${$i}"

        if ! printf '%s\n' "$$   arg" | grep -Eq "^(   $${regex})$"; then
            die "Ошибка в аргументе $i: $message"
        fi

        i=$((i + 1))
    done <<'EOF'
[a-zA-Z]+	аргумент должен состоять только из латинских букв и не быть пустым
[0-9]+	аргумент должен состоять только из цифр и не быть пустым
EOF

    if [ "$#" -ne "$expected_count" ]; then
        die "Ошибка: ожидалось $expected_count аргументов, получено $#."
    fi
}

validate_args "$@"

echo "Все аргументы корректны."
```

Пример запуска:

Bash

```
./check-args.sh hello 123
```
---

### 5–10. Итоговый скрипт lines-range.sh

Я полностью реализовал итоговый универсальный скрипт lines-range.sh, который поддерживает:

- один и два параметра
- открытые диапазоны (:120, 120:, :)
- отрицательные номера (-123)
- форму +COUNT
- шестнадцатеричные числа
- все комбинации отрицательных и положительных значений


Примеры запуска:

Bash

```
cat file.txt | ./lines-range.sh 5
cat file.txt | ./lines-range.sh :120
cat file.txt | ./lines-range.sh -5
cat file.txt | ./lines-range.sh -10 +5
cat file.txt | ./lines-range.sh 10 -5
cat file.txt | ./lines-range.sh 0x10 0x20
```

---
## Backlinks

- [[00 OS Labs MOC]]
- [[06 - File Signature Checker]]
- [[08 - sed and awk Exercises]]
