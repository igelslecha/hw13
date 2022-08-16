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
*;*;test2|test3;!Wd #я делал проверку в пятницу, поэтому у меня стояло *;*;test2|test3:!Fr
#
```
*Активируем модуль pam_time.so в /etc/pam.d/sshd , если нужно ещё ограничить вход в общем то так же делаем в файле /etc/pam.d/login*
```
[vagrant@nginx ~]$ sudo vi /etc/pam.d/sshd 
***
ccount    required     pam_nologin.so
account    required     pam_time.so
***
```
*Проверка*
```
igels@LaptopAll:~/hw13$ ssh test1@192.168.56.150
test1@192.168.56.150's password: 
Last login: Fri Aug 12 10:38:44 2022 from 192.168.56.1
[test1@nginx ~]$ exit
logout
Connection to 192.168.56.150 closed.
igels@LaptopAll:~/hw13$ ssh test2@192.168.56.150
test2@192.168.56.150's password: 
Connection closed by 192.168.56.150 port 22
igels@LaptopAll:~/hw13$ ssh test3@192.168.56.150
test3@192.168.56.150's password: 
Connection closed by 192.168.56.150 port 22
```
**К сожалению, для использования большого количества пользователей очень громозкое решение, потому что правило работает не по группе (как в задании, а по принципу добавки вручную каждого пользователя)**

*1.2. Используя модуль pam_exec.so*
*Удаляю предыдущий модуль и активирую pam_exec.so со ссылкой на скрипт обработки входа*
```
***
account    required     pam_nologin.so
account    required     pam_exec.so    /usr/local/bin/test_login.sh
account    include      password-auth
***
```
*Прописываю скрипт, в процессе выполнения этого задания, осознал как важно, сначала написать правильный скрипт и только потом прописывать его на загрузку, так как может случится непредвиденная перезагрузка и как следствие полная потеря подключения к ВМ по ssh*
```
[vagrant@nginx ~]$ sudo vi /usr/local/bin/test_login.sh
#!/bin/bash
if [ `grep $PAM_USER /etc/group | grep 'admin'` ]; then
 if [ $(date +%a) = "Sat" ] || [ $(date +%) = "Sun" ]; then #Я проверял в понедельник и у меня вместо Sat был Mon
  exit 0
 else
  exit 1
 fi
 else
  exit 1
fi
```
*Сделал скрипт исполняемым для рута и проверил*
```
[vagrant@nginx ~]$ sudo chmod u+x /usr/local/bin/test_login.sh
```
*Для всех пользователей*
```
[vagrant@nginx ~]$ sudo chmod ugo+x /usr/local/bin/test_login.sh
```
*Проверка*
```
igels@LaptopAll:~/hw13$ ssh test1@192.168.56.150
test1@192.168.56.150's password: 
Last failed login: Mon Aug 15 12:33:43 UTC 2022 from 192.168.56.1 on ssh:notty
There were 5 failed login attempts since the last successful login.
Last login: Mon Aug 15 12:14:00 2022 from 192.168.56.1
[test1@nginx ~]$ exit
logout
Connection to 192.168.56.150 closed.
igels@LaptopAll:~/hw13$ ssh test2@192.168.56.150
test2@192.168.56.150's password: 
/usr/local/bin/test_login.sh failed: exit code 1
Connection closed by 192.168.56.150 port 22
igels@LaptopAll:~/hw13$ ssh test3@192.168.56.150
test3@192.168.56.150's password: 
/usr/local/bin/test_login.sh failed: exit code 1
Connection closed by 192.168.56.150 port 22
```
[vagrant@nginx ~]$ sudo chmod ugo+x /usr/local/bin/test_login.sh
```

*1.3. Используя модуль pam_script.so*
*Устанавливаю модуль*
```
[vagrant@nginx ~]$ sudo -i
[root@nginx ~]# for pkg in epel-release pam_script; do yum install -y $pkg; done
```
*Прописываю в файле 
```
[vagrant@nginx ~]$ sudo vi /etc/pam.d/sshd 
***
account    required     pam_nologin.so
account    required     pam_script.so   /usr/local/bin/test_login.sh
account    include      password-auth
***
```
*Проверка*
```
gels@LaptopAll:~/hw13$ ssh test1@192.168.56.150
test1@192.168.56.150's password: 
Last login: Mon Aug 15 17:00:53 2022 from 192.168.56.1
[test1@nginx ~]$ exit
logout
Connection to 192.168.56.150 closed.
igels@LaptopAll:~/hw13$ ssh test2@192.168.56.150
test2@192.168.56.150's password: 
/usr/local/bin/test_login.sh failed: exit code 1
Connection closed by 192.168.56.150 port 22
igels@LaptopAll:~/hw13$ ssh test3@192.168.56.150
test3@192.168.56.150's password: 
/usr/local/bin/test_login.sh failed: exit code 1
Connection closed by 192.168.56.150 port 22
```

***Решение***
**2. Дать конкретному пользователю права работать с докером и возможность рестартить докер сервис**
