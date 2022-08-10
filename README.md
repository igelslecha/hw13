# hw13
Авторизация и аутентификация

***Домашнее задание***

**PAM**

**Описание/Пошаговая инструкция выполнения домашнего задания:**

**Запретить всем пользователям, кроме группы admin логин в выходные (суббота и воскресенье), без учета праздников**

**дать конкретному пользователю права работать с докером и возможность рестартить докер сервис**

***Решение***
**1. Запретить всем пользователям, кроме группы admin логин в выходные (суббота и воскресенье), без учёта праздников**
*В качестве стенда использую ВМ из прошлых домашек на CentOS 7 (установка nginx).*
*Создаю несколько тестовых пользователей*
```
[vagrant@nginx ~]$ sudo useradd test1
[vagrant@nginx ~]$ sudo useradd test2
[vagrant@nginx ~]$ sudo useradd test3
[vagrant@nginx ~]$ echo "Otus2022" |sudo passwd --stdin test1 && echo "Otus2022" |sudo passwd --stdin test2 && echo "Otus2022" |sudo passwd --stdin test3   
Changing password for user test1.
passwd: all authentication tokens updated successfully.
Changing password for user test2.
passwd: all authentication tokens updated successfully.
Changing password for user test3.
passwd: all authentication tokens updated successfully.
[vagrant@nginx ~]$ 
```
*Создаю группу admin и добавляю пользователя test1*
```
[vagrant@nginx ~]$ sudo groupadd "admin"
[vagrant@nginx ~]$ sudo usermod -G admin test1
[vagrant@nginx ~]$ id test1
uid=1001(test1) gid=1001(test1) groups=1001(test1),1004(admin)
```
*1.1. Используя модуль pam_time.so*
```
[vagrant@nginx ~]$ sudo vi /etc/security/time.conf
```
```
#
*;*;!wheel;AlWd
#
```
*1.2. Используя модуль pam_exec.so*

*1.3. Используя модуль pam_script.so*
