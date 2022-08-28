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

*Устанавливаю вспомогательную утилиту, репозиторий, докер и проверяю docker run hello-world*
```
[vagrant@nginx ~]$  sudo yum install -y yum-utils
Failed to set locale, defaulting to C
Loaded plugins: fastestmirror
Determining fastest mirrors
 * base: centos-mirror.rbc.ru
 * extras: mirror.corbina.net
 * updates: mirror.corbina.net
***
yum-utils.noarch 0:1.1.31-54.el7_8                                            

Complete!
[vagrant@nginx ~]$ sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
Failed to set locale, defaulting to C
Loaded plugins: fastestmirror
adding repo from: https://download.docker.com/linux/centos/docker-ce.repo
grabbing file https://download.docker.com/linux/centos/docker-ce.repo to /etc/yum.repos.d/docker-ce.repo
repo saved to /etc/yum.repos.d/docker-ce.repo
[vagrant@nginx ~]$  sudo yum install docker-ce docker-ce-cli containerd.io docker-compose-plugin
Failed to set locale, defaulting to C
Loaded plugins: fastestmirror
Loading mirror speeds from cached hostfile
***
docker-ce-rootless-extras.x86_64 0:20.10.17-3.el7                             
  docker-scan-plugin.x86_64 0:0.17.0-3.el7                                      
  fuse-overlayfs.x86_64 0:0.7.2-6.el7_8                                         
  fuse3-libs.x86_64 0:3.6.1-4.el7                                               
  libcgroup.x86_64 0:0.41-21.el7                                                
  libsemanage-python.x86_64 0:2.5-14.el7                                        
  policycoreutils-python.x86_64 0:2.5-34.el7                                    
  python-IPy.noarch 0:0.75-6.el7                                                
  setools-libs.x86_64 0:3.3.8-4.el7                                             
  slirp4netns.x86_64 0:0.4.3-4.el7_8                                            

Complete!
[vagrant@nginx ~]$ sudo systemctl start docker                  
[vagrant@nginx ~]$ sudo docker run hello-world
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
2db29710123e: Pull complete 
Digest: sha256:7d246653d0511db2a6b2e0436cfd0e52ac8c066000264b3ce63331ac66dca625
Status: Downloaded newer image for hello-world:latest

Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (amd64)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/
sudo docker run hello-world
For more examples and ideas, visit:
 https://docs.docker.com/get-started/

```

*2.1. Добавляю пользователя test1 в группу wheel*
```
[vagrant@nginx ~]$ sudo usermod -G wheel test1
```
*Проверяю*
```
[test1@nginx ~]$ docker run hello-world
docker: Got permission denied while trying to connect to the Docker daemon socket at unix:///var/run/docker.sock: Post "http://%2Fvar%2Frun%2Fdocker.sock/v1.24/containers/create": dial unix /var/run/docker.sock: connect: permission denied.
See 'docker run --help'.
[test1@nginx ~]$ id test1
uid=1001(test1) gid=1001(test1) groups=1001(test1),10(wheel)
[test1@nginx ~]$ sudo docker run hello-world

We trust you have received the usual lecture from the local System
Administrator. It usually boils down to these three things:

    #1) Respect the privacy of others.
    #2) Think before you type.
    #3) With great power comes great responsibility.

[sudo] password for test1: 

Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (amd64)
 3. The Docker daemon created a new container from that image which runs the
    executa*ble that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/get-started/
```
*И проверка рестарта сервиса*
```
[test1@nginx ~]$ systemctl restart docker.service
==== AUTHENTICATING FOR org.freedesktop.systemd1.manage-units ===
Для управления системными службами и юнитами, необходимо пройти аутентификацию.
Authenticating as: test1
Password: 
==== AUTHENTICATION COMPLETE ===
[test1@nginx ~]$ 
```
*Не очень удобный способ, во-первых даёт права рута для пользователя, во-вторых дополнительно вводить sudo и пароль*

**2.2. Установить suid-бит. Установка данного бита позволит рабоать с docker так, будто он запущен от root. Способ имеет низкую гибкость, так как установка бита позволит любому пользователю выполнить команду**

```
[vagrant@nginx ~]$ sudo chmod u+s /usr/bin/docker
[vagrant@nginx ~]$ ll /usr/bin/ | grep docker
-rwsr-xr-x. 1 root root   60507248 Jun  6 23:06 docker
-rwxr-xr-x. 1 root root     849104 Jun  6 23:04 docker-init
-rwxr-xr-x. 1 root root    2990568 Jun  6 23:03 docker-proxy
-rwxr-xr-x. 1 root root   96658176 Jun  6 23:04 dockerd
-rwxr-xr-x. 1 root root      13348 Jun  6 22:59 dockerd-rootless-setuptool.sh
-rwxr-xr-x. 1 root root       5150 Jun  6 22:59 dockerd-rootless.sh
-rwxr-xr-x. 1 root root    7925248 Jun  6 23:10 rootlesskit-docker-proxy
```
*Проверка*
```
[test2@nginx ~]$ docker run hello-world
WARNING: Error loading config file: /home/test2/.docker/config.json: open /home/test2/.docker/config.json: permission denied

Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (amd64)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/get-started/

```
*проверка с перезапуском будет добавлена как только сделаю сделующее задание с докером*

