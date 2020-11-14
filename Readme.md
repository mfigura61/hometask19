# OTUS ДЗ 11 Пользователи и группы. Авторизация и аутентификация  (Centos 7)
-----------------------------------------------------------------------
### Домашнее задание

    1. Запретить всем пользователям, кроме группы admin логин в выходные(суббота и воскресенье), без учета праздников
    2. *Опционально.дать конкретному пользователю права работать с докером и возможность рестартить докер сервис

1. Запретить всем пользователям, кроме группы admin логин в выходные(суббота и воскресенье), без учета праздников

#### запретим конкретному пользователю, пока без привязки к группе.

Чтобы у пользователя не было возможности войти в систему не только через SSH, а также через обычную консоль (монитор) то необходимо откорректировать два файла.

- Заходим в файлы ```nano /etc/pam.d/sshd``` и ```nano /etc/pam.d/login``` и приводим их к следующему виду:
```
[root@lvm system]# cat /etc/pam.d/sshd
#%PAM-1.0
auth       required     pam_sepermit.so
auth       substack     password-auth
auth       include      postlogin
# Used with polkit to reauthorize users in remote sessions
-auth      optional     pam_reauthorize.so prepare
account required pam_access.so # Это модуль, который может давать доступ одной группе и не давать другой, но он не может это сделать по дням и времени
account required pam_time.so # Добавляем вот эту строку для ограничения по пользователю
account    required     pam_nologin.so
account    include      password-auth
password   include      password-auth
# pam_selinux.so close should be the first session rule
session    required     pam_selinux.so close
session    required     pam_loginuid.so
# pam_selinux.so open should only be followed by sessions to be executed in the user context
session    required     pam_selinux.so open env_params
session    required     pam_namespace.so
session    optional     pam_keyinit.so force revoke
session    include      password-auth
session    include      postlogin
# Used with polkit to reauthorize users in remote sessions
-session   optional     pam_reauthorize.so prepare
```
```
[root@lvm system]# cat /etc/pam.d/login
#%PAM-1.0
auth [user_unknown=ignore success=ok ignore=ignore default=bad] pam_securetty.so
auth       substack     system-auth
auth       include      postlogin
account required pam_access.so # Это модуль, который может давать доступ одной группе и не давать другой, но он не может это сделать по дням и времени
account   required   pam_time.so # Добавляем вот эту строку для ограничения по пользователю
account    required     pam_nologin.so
account    include      system-auth
password   include      system-auth
# pam_selinux.so close should be the first session rule
session    required     pam_selinux.so close
session    required     pam_loginuid.so
session    optional     pam_console.so
# pam_selinux.so open should only be followed by sessions to be executed in the user context
session    required     pam_selinux.so open
session    required     pam_namespace.so
session    optional     pam_keyinit.so force revoke
session    include      system-auth
session    include      postlogin
-session   optional     pam_ck_connector.so
```

- Далее заходим в файл ```nano /etc/security/time.conf``` и добавляем в конце файла строку ```*;*;test_user;Wk```
- Создаем пользователя командой ```useradd test_user``` и задаем ему пароль, например, 123456 ```passwd test_user```
- Пытаемся зайти сегодня, в среду, когда у нас работает правило ```ssh test_user@localhost``` и получаем:
```
vagrant@otuslinux ~]$ ssh test_user@localhost
test_user@localhost's password:
Last login: Wed Sep 30 12:29:24 2020 from ::1
[test_user@otuslinux ~]$

```
- А теперь запретим в среду, изменив Wk на !We.

```
vagrant@otuslinux ~]$ ssh test_user@localhost
test_user@localhost's password:
Authentication failed.
```

### Теперь распространим ограничения на группу admin .

- Создаем группу ```groupadd admin```
- Добавляем туда пользователя ```usermod -aG admin test_user```
- Устанавливаем компонент, с помощью которого мы сможем описывать правила входа в виде обычного bash скрипта ```yum install pam_script -y```
- Приводим файл ```/etc/pam.d/sshd``` к следующему виду и добавляем строку ```auth       required     pam_script.so```:
```
#%PAM-1.0

auth       required     pam_script.so # Добавляем сюда эту строку
auth       required     pam_sepermit.so
auth       substack     password-auth
auth       include      postlogin
# Used with polkit to reauthorize users in remote sessions
-auth      optional     pam_reauthorize.so prepare
account required pam_access.so
account required pam_time.so
account    required     pam_nologin.so
account    include      password-auth
password   include      password-auth
# pam_selinux.so close should be the first session rule
session    required     pam_selinux.so close
session    required     pam_loginuid.so
# pam_selinux.so open should only be followed by sessions to be executed in the user context
session    required     pam_selinux.so open env_params
session    required     pam_namespace.so
session    optional     pam_keyinit.so force revoke
session    include      password-auth
session    include      postlogin
# Used with polkit to reauthorize users in remote sessions
-session   optional     pam_reauthorize.so prepare
```
- Далее приводим файл ```/etc/pam_script``` к следующему виду: (у файла должны быть права на исполнение)
```
#!/bin/bash

if [[ `grep $PAM_USER /etc/group | grep 'admin'` ]]
then
exit 0
fi
if [[ `date +%u` > 5 ]]
then
exit 1
fi
```
В этом файле мы проверяем состоит ли пользователь в группе admin и если да то пускаем его (значит его можно пускать всегда). Если он не состоит в этой группе то срабатывает проверка на то, какой сейчас день недели, если он больше 5, т.е. выходные то не пускаем.
