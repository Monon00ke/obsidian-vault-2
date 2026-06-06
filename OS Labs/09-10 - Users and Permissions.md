---
title: "09-10 - Users and Permissions"
layout: default
---
### 0. Переход в режим обычного пользователя (не root)

**Цель:** Убедиться, что работа ведётся не от root.

Bash

```
whoami
# mononoke

su - mononoke
whoami
# mononoke

exit
```

**Вывод:** Работаем от обычного пользователя mononoke. Постоянное использование root — плохая практика по соображениям безопасности.

---

### 1. Команда su

**Цель:** Переход в root и обратно, изучение опций -c и -p.

Bash

```
su -
whoami          # root
exit

su -c 'whoami'
su - mononoke -c 'id'
su -p mononoke

man su
```

**Ключевые опции:**

- -c — выполнить одну команду и выйти
- -p — сохранить текущее окружение
- - — полный login shell (загрузка профиля пользователя)

---

### 2. Команда sudo

**Цель:** Выполнение команд от root и других пользователей, изучение опций -i и -E.

Bash

```
sudo whoami
sudo id
sudo -u mononoke whoami
sudo -u mononoke -i
sudo -i
sudo -E env | sort
sudo -l

man sudo
```

**Ключевые опции:**

- -i — запуск login-shell от целевого пользователя
- -E — сохранить переменные окружения текущего пользователя

---

### 3. Важные системные файлы

**Цель:** Изучить содержимое и описание основных файлов.

Bash

```
cat /etc/passwd | head -n 15
sudo cat /etc/shadow | head -n 5
cat /etc/group | head -n 15
sudo visudo -c

man 5 passwd
man 5 shadow
man 5 group
man 5 sudoers
```

**Краткое описание:**

- /etc/passwd — основные данные о пользователях (логин, UID, GID, домашний каталог, shell)
- /etc/shadow — хэши паролей и информация о сроке действия паролей
- /etc/group — список групп и их членов
- /etc/sudoers — правила, кто и какие команды может выполнять через sudo

---

### 4. Создание и управление пользователями и группами

Bash

```
sudo useradd -m testuser1
sudo passwd testuser1
sudo userdel -r testuser1

sudo adduser testuser2
sudo deluser --remove-home testuser2

sudo groupadd testgroup1
sudo groupdel testgroup1

sudo usermod -aG docker mononoke
sudo usermod -aG adm mononoke

id
groups

newgrp docker
id
exit
```

**Вывод:** Успешно создавались/удалялись пользователи и группы, добавлялся себя в дополнительные группы, менялась первичная группа.

---

### 5. Выполнение команд от имени другого пользователя (без его пароля)

Bash

```
sudo -u testuser1 -i
sudo su - testuser1 -c 'whoami'
sudo su - testuser1 -c 'id'
```

**Вывод:** При наличии прав в sudoers можно переключаться в сеанс другого пользователя без знания его пароля.

---

### 6. Сложные команды под sudo (пайпы и переменные окружения)

Bash

```
sudo cat /etc/shadow | wc -l          # wc выполняется не от root!

sudo bash -c 'cat /etc/shadow | wc -l > /var/log/count'

sudo bash -c 'echo $USER'      # выводит root
sudo sh -c 'echo $USER'
```

**Вывод:** Для выполнения всей цепочки команд (включая пайпы и перенаправления) в контексте root необходимо использовать sudo bash -c '...'.

---

### 7. Python: real и effective UID/GID

Bash

```
python3 - <<'PY'
import os
print("Real UID:", os.getuid())
print("Effective UID:", os.geteuid())
print("Real GID:", os.getgid())
print("Effective GID:", os.getegid())
print("Groups:", os.getgroups())
PY
```

**Вывод:**

- **Real ID** — идентификатор пользователя, который запустил процесс
- **Effective ID** — идентификатор, права которого используются в данный момент

---

### 8. Скрипт проверки доступа к файлу от имени пользователя и всех его групп

Bash

```
sudo python3 -c "
import os, pwd, grp, sys
user = sys.argv[1]
path = sys.argv[2]
pw = pwd.getpwnam(user)
groups = os.getgrouplist(user, pw.pw_gid)
old_euid = os.geteuid()
old_egid = os.getegid()

for gid in groups:
    try:
        os.setegid(gid)
        os.seteuid(pw.pw_uid)
        open(path).close()
        res = 'READ_OK'
    except:
        res = 'READ_FAIL'
    finally:
        os.seteuid(old_euid)
        os.setegid(old_egid)
    print(f'{user}:{grp.getgrgid(gid).gr_name}:{gid}:{res}')
" mononoke /etc/shadow
```

**Вывод:** Такой скрипт позволяет надёжно проверить реальный доступ пользователя и всех его групп к файлу.

---

### 9.1 Реальные (человеческие) пользователи

Bash

```
awk -F: '$6 ~ /^\/home\// && $7 ~ /(\/bash|\/sh|\/zsh)$/ {print $1}' /etc/passwd
```

**Вывод:** Отобраны пользователи с домашним каталогом в /home и нормальным интерактивным shell.

---

### 9.2 Сравнение списков пользователей в /etc/passwd и /etc/shadow

Bash

```
cut -d: -f1 /etc/passwd | sort > left.txt
cut -d: -f1 /etc/shadow | sort > right.txt

comm -12 left.txt right.txt   # присутствуют в обоих
comm -23 left.txt right.txt   # только в passwd
comm -13 left.txt right.txt   # только в shadow
comm -3  left.txt right.txt   # присутствуют только в одном из файлов
```

---

### 9.3 Пары пользователь–группа, отсортированные по убыванию GID

Bash

```
awk -F: '
FNR==NR {
    gname[$3]=$1
    split($4,a,",")
    for(i in a) if(a[i]!="") print $3, a[i], $1
    next
}
{print $4, $1, gname[$4]}
' /etc/group /etc/passwd | sort -nr -k1,1 | uniq
```

---
## Backlinks

- [[00 OS Labs MOC]]
- [[14]]
