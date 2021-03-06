# Пользователи и группы. Авторизация и аутентификация

## Задачи:
1. Запретить всем пользователям, кроме группы admin логин в выходные (суббота и воскресенье), без учета праздников
2. Дать конкретному пользователю права работать с докером
и возможность рестартить докер сервис



## 1. Настройка ограничения авторизации по времени для групп пользователей

Создаю группу admin, добавляю в нее пользователя root
```
[root@centos8-test ~]# groupadd admin
[root@centos8-test ~]# usermod -aG admin root
[root@centos8-test ~]# id root
uid=0(root) gid=0(root) groups=0(root),1003(admin)
```

Создаю пользователей test1 и duty, обоим назначаю пароли такие же как логин (__bad practice!__).
```
[root@centos8-test ~]# useradd test1 -s /bin/bash
[root@centos8-test ~]# passwd test1
Changing password for user test1.
New password:
BAD PASSWORD: The password is shorter than 8 characters
Retype new password:
passwd: all authentication tokens updated successfully.
[root@centos8-test ~]# useradd duty -s /bin/bash
[root@centos8-test ~]# passwd duty
Changing password for user duty.
New password:
BAD PASSWORD: The password is shorter than 8 characters
Retype new password:
passwd: all authentication tokens updated successfully.
```
 
Пользователя duty добавляю в группу admin
 ```
[root@centos8-test ~]# usermod -aG admin duty
```

Меняю настройки sshd для возможности логиниться по паролю и перезапускаю сервис
```
[root@centos8-test ~]# grep PasswordA /etc/ssh/sshd_config
#PasswordAuthentication yes
PasswordAuthentication no
# PasswordAuthentication.  Depending on your PAM configuration,
# PAM authentication, then enable this but set PasswordAuthentication
[root@centos8-test ~]# sed -i 's/^PasswordAuthentication.*$/PasswordAuthentication yes/' /etc/ssh/sshd_config
[root@centos8-test ~]# grep PasswordA /etc/ssh/sshd_config
#PasswordAuthentication yes
PasswordAuthentication yes
# PasswordAuthentication.  Depending on your PAM configuration,
# PAM authentication, then enable this but set PasswordAuthentication
[root@centos8-test ~]# systemctl restart sshd
```

Создаю скрипт для определения вхождения пользователя в группу admin, устанавливаю ему бит исполнения
```
[root@centos8-test ~]# cat > /usr/local/bin/test_login.sh
#!/bin/bash
ID_USER=$(id $PAM_USER)
CHECKGROUP=$(echo ${ID_USER} | grep -o admin)
echo ${CHECKGROUP}
if [[ ${CHECKGROUP} != admin ]]; then
 if [ $(date +%a) = "Sat" ] || [ $(date +%a) = Sun ]; then
 exit 1
 else
 exit 0
 fi
fi

[root@centos8-test ~]# chmod +x /usr/local/bin/test_login.sh
```

Добавляю в /etc/pam.d/sshd строку
```
account    required     pam_exec.so /usr/local/bin/test_login.sh
```

Проверяю:
```
practice$ ssh duty@192.168.11.102
duty@192.168.11.102's password:
Last login: Fri Mar 20 00:33:50 2020 from 192.168.11.1
```
```
practice$ ssh test1@192.168.11.102
test1@192.168.11.102's password:
/usr/local/bin/test_login.sh failed: exit code 1
Connection closed by 192.168.11.102 port 22
```


## 2. Настройка возможности работы с докером пользователю без sudo

Устанавливаю репозиторий докера и сам docker и запускаю сервис
```
[root@centos8-test ~]# dnf config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo
[root@centos8-test ~]# dnf install -y docker-ce
[root@centos8-test ~]# systemctl start docker
```

Добавляю пользователя duty в группу docker
```
[root@centos8-test ~]# gpasswd -a duty docker
```

Это уже позволяет запускать контейнеры
```
[duty@centos8-test ~]$ docker run hello-world
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
1b930d010525: Pull complete
Digest: sha256:f9dfddf63636d84ef479d645ab5885156ae030f611a56f3a7ac7f2fdd86d7e4e
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

For more examples and ideas, visit:
 https://docs.docker.com/get-started/
```

Но при попытке перезапустить сервис получаю предложение авторизоваться под рутом
```
[duty@centos8-test ~]$ systemctl restart docker
==== AUTHENTICATING FOR org.freedesktop.systemd1.manage-units ====
Authentication is required to restart 'docker.service'.
Authenticating as: root
Password:
```

Добавляю правило
```
[root@centos8-test ~]# cat > /etc/polkit-1/rules.d/01-docker.rules 
polkit.addRule(function(action, subject) {
    if (action.id == "org.freedesktop.systemd1.manage-units") {
        if (action.lookup("unit") == "docker.service" && subject.user === "test1"){
            if (action.lookup("verb") == "restart") {
                return polkit.Result.YES;
            }
        }
    }
});
```

Пробую еще раз
```
[duty@centos8-test ~]$ systemctl restart docker

[duty@centos8-test ~]$ systemctl status docker
● docker.service - Docker Application Container Engine
   Loaded: loaded (/usr/lib/systemd/system/docker.service; disabled; vendor preset: disabl>
   Active: active (running) since Fri 2020-03-20 00:52:35 MSK; 4s ago
     Docs: https://docs.docker.com
 Main PID: 15974 (dockerd)
    Tasks: 8
   Memory: 68.5M
   CGroup: /system.slice/docker.service
           └─15974 /usr/bin/dockerd -H fd://
```

Но при этом, например, возможности остановить сервис по прежнему нет
```
[duty@centos8-test ~]$ systemctl stop docker
==== AUTHENTICATING FOR org.freedesktop.systemd1.manage-units ====
Authentication is required to stop 'docker.service'.
Authenticating as: root
Password:
```

## Доступ в контейнер возможен одним из перечисленных выше пользователей, а также пользователем vagrant (пароль vagrant), который на всякий случай также включен в группу admin.