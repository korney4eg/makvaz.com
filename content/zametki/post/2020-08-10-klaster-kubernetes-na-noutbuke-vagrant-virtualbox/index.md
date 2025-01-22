---
layout: post
title: Как запустить кластер Kubernetes на ноутбуке. Настраиваем VirtualBox и Vagrant.
archives: "2020"
tags: [kubernetes, howto, vagrant]
---
<img src="/2020/07/29/kubernetes-the-hard-way-on-laptop-preparation/laptop.png" alt="" width="120" style="float: left; margin-right: 15px">

_В прошлой [статье](/2020/08/05/klaster-kubernetes-na-noutbuke-podgotovka/) мы рассмотрели, что будет представлять из себя кластер Kubernetes, а так же установили некоторый софт. Сегодня будем запускать виртуальные машины(ВМ), для чего нам понадобиться VirtualBox и Vagrant._
<!--more-->
## Содержимое:

1. [Введение. Схема устройства кластера. Подготовка рабочей среды.](/2020/08/05/klaster-kubernetes-na-noutbuke-podgotovka/)
2. [Установка и настройка Vagrant, VirtualBox. Создание виртуальных машин.](#)
4. Создание файлов конфигураций, сертификатов.
5. Установка и настройка `etcd`. Запуск Панели управления Kubernetes(Kubernetes Control Plane)
7. Настройка рабочих нод
8. Настройка kubectl, установка плагинов и аддонов. Проверка работоспособности.


## Установка VirtualBox

Все компоненты будут запускаться на виртуальных машинах VirtualBox.

##### Установка на MacOS

```bash
brew install virtualbox
```

##### Установка на Linux

По этой [ссылке](https://www.virtualbox.org/wiki/Downloads) можно скачать и установить последнюю версию для вашего дистрибутива.

## Vagrant

Vagrant помогает автоматизировать создание и настройку ВМ. Не важно сколько нужно машин 2 или 50, достаточно только создать правильный `Vagrantfile` файл.


##### Установка на MacOS

```bash
brew install vagrant
```

##### Установка на Linux

По [ссылке](https://www.vagrantup.com/downloads) можно скачать *deb* пакет для дистрибутивов на базе Debian, *rpm* пакет - для CentOs. Для всех остальных - скачать и распаковать zip-файл, например в папку `/usr/local/bin/`.


{{< notice tip >}}
Для Vagrant есть плагин - **hostsupdater**. С помощью этого плагина на ноутбуке можно привязать имя виртуалки к её IP-адресу.

Устанавливается командой:

```bash
vagrant plugin update vagrant-hostsupdater
```
{{< /notice >}}

## Создаём Виртуальные машины.

Перед началом работы давайте создадим пустую папку, где будут храниться нужные файлы. 

Набираем в терминале:
```bash
mkdir ~/local-k8s && cd ~/local-k8s
```

Как я уже говорил выше, с помощью файла `Vagrantfile` можно описать какие виртуалки мы хотим. Давайте создадим файл `Vagrantfile` в вашем любимом редакторе и запишем:
{{< highlight ruby "linenos=table" >}}
Vagrant.configure("2") do |config|
  hostnames = {
    'controller0': {:ip:'192.168.50.100',:ram:512 },
    'controller1': {:ip:'192.168.50.101',:ram:512 },
    'controller2': {:ip:'192.168.50.102',:ram:512 },
    'lb'         : {:ip:'192.168.50.200',:ram:512 },
    'worker0'    : {:ip:'192.168.50.10', :ram:1024},
    'worker1'    : {:ip:'192.168.50.11', :ram:1024},
  }

  hosts_file = "127.0.0.1	localhost\n"
  hostnames.each do |hostname,settings|
    hosts_file += "#{settings[:ip]} #{hostname}\n"
  end
  config.vm.provision "shell", inline: "echo \"#{hosts_file}\" > /etc/hosts"

  hostnames.each do |hostname,settings|
    config.vm.define hostname do |instance|
      instance.vm.box = "bento/ubuntu-20.04"
      instance.vm.network "private_network", ip: hostnames[hostname][:ip]
      instance.vm.hostname = hostname
      instance.vm.provider :virtualbox do |v|
        v.customize ["modifyvm", :id, "--memory", hostnames[hostname][:ram]]
        v.customize ["modifyvm", :id, "--name", hostname]
      end
    end
  end
end
{{< / highlight >}}

### Разбираемся с Vagrantfile

#### Начало конфигурации
{{< highlight ruby >}}
Vagrant.configure("2") do |config|
  ...
end
{{< / highlight >}}
Указываем, что будет использоваться конфигурация Vagrant **2-ой** версии.

#### Параметры всех виртуальных машин
{{< highlight ruby >}}
  instances = {
    'controller0': {:ip:'192.168.50.100',:ram:512 },
    'controller1': {:ip:'192.168.50.101',:ram:512 },
    'controller2': {:ip:'192.168.50.102',:ram:512 },
    'lb'         : {:ip:'192.168.50.200',:ram:512 },
    'worker0'    : {:ip:'192.168.50.10', :ram:1024},
    'worker1'    : {:ip:'192.168.50.11', :ram:1024},
  }
{{< / highlight >}}
По сути, это таблица с параметрами для каждой ВМ. Например, у виртуалки `lb` будет `512` МБ оперативной памяти и IP-адрес `192.168.50.200`. Эти параметры запишем в переменную `instances`, они будут использоваться позже.

#### Заполняем файл `/etc/hosts` на всех виртуальных машинах
{{< highlight ruby >}}
  hosts_file = "127.0.0.1	localhost\n"
  instances.each do |hostname,settings|
    hosts_file += "#{settings[:ip]} #{hostname}\n"
  end
  config.vm.provision "shell", inline: "echo \"#{hosts_file}\" > /etc/hosts"
{{< / highlight >}}
На каждой виртуалке будет создан файл вида:
```
127.0.0.1       localhost
192.168.50.10 worker0
192.168.50.11 worker1
192.168.50.100 controller0
192.168.50.101 controller1
192.168.50.102 controller2
192.168.50.200 lb
```
После этого виртуалки смогут соединяться друг с другом по имени.

#### создание каждой виртуальной машины.
{{< highlight ruby >}}
  instances.each do |hostname,settings|
    config.vm.define hostname do |instance|
      instance.vm.box = "bento/ubuntu-20.04"
      instance.vm.network "private_network", ip: instances[hostname][:ip]
      instance.vm.hostname = hostname
      instance.vm.provider :virtualbox do |v|
        v.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]
        v.customize ["modifyvm", :id, "--memory", instances[hostname][:ram]]
        v.customize ["modifyvm", :id, "--name", hostname]
      end
    end
  end
{{< / highlight >}}
Давайте разберём по строчке.

{{< highlight ruby >}}
  instances.each do |hostname,settings|
    ...
  end
{{< / highlight >}}
Проходим циклом по каждой строчке из переменной `instances` и записываем имя машины(`hostname`) и параметры(`settings`)

{{< highlight ruby >}}
    config.vm.define hostname do |instance|
      ...
    end
{{< / highlight >}}
Создаём виртуальную машину с именем из переменной `hostname`.

{{< highlight ruby >}}
      instance.vm.box = "bento/ubuntu-20.04"
{{< / highlight >}}
На всех виртуалках будет установлена Ubuntu 20.04 - последняя стабильная версия Ubuntu на момент написания статьи(август 2020).

{{< highlight ruby >}}
      instance.vm.network "private_network", ip: instances[hostname][:ip]
{{< / highlight >}}
По умолчанию виртуальные машины для Vagrant настроены с сетевым адаптером типа **NAT** для доступа в интернет. Нам нужно, чтобы у виртуальных машин был фиксированный IP-адрес, по которому можно было бы к ним обращаться. Для этого создаём новый адаптер, с типом **Internal network** и задаём IP-адрес.

{{< highlight ruby >}}
      instance.vm.hostname = hostname
{{< / highlight >}}
Задаём имя хоста.

{{< highlight ruby >}}
      instance.vm.provider :virtualbox do |v|
        ...
      end
{{< / highlight >}}
Задаём настройки, специфичные для VirtualBox.

{{< highlight ruby >}}
        v.customize ["modifyvm", :id, "--memory", instances[hostname][:ram]]
{{< / highlight >}}
Сколько будет оперативки. Это значение будет браться из таблицы.

{{< highlight ruby >}}
        v.customize ["modifyvm", :id, "--name", hostname]
{{< / highlight >}}
Задаём имя машины в VirtualBox.

### Запускаем Vagrant

Чтобы создать виртуальные машины теперь запускаем команду:
```bash
vagrant up
```

{{< notice info >}}
Если вы установили плагин для Vagrant - `hostsupdater`, то скорее всего у вас спросят пароль. Не пугайтесь, этот плагин будет менять файл `/etc/hosts`, а для этого ему нужны привилегированные права.
{{< /notice >}}

На экран выведется следующее:
```
Bringing machine 'worker0' up with 'virtualbox' provider...
Bringing machine 'worker1' up with 'virtualbox' provider...
Bringing machine 'controller0' up with 'virtualbox' provider...
Bringing machine 'controller1' up with 'virtualbox' provider...
Bringing machine 'controller2' up with 'virtualbox' provider...
Bringing machine 'lb' up with 'virtualbox' provider...
==> worker0: Importing base box 'bento/ubuntu-20.04'...
==> worker0: Matching MAC address for NAT networking...
==> worker0: Checking if box 'bento/ubuntu-20.04' version '202007.17.0' is up to date...
==> worker0: Setting the name of the VM: local-k8s_worker0_1596841981923_77771
==> worker0: Clearing any previously set network interfaces...
==> worker0: Preparing network interfaces based on configuration...
    worker0: Adapter 1: nat
    worker0: Adapter 2: hostonly
==> worker0: Forwarding ports...
    worker0: 22 (guest) => 2222 (host) (adapter 1)
==> worker0: Running 'pre-boot' VM customizations...
==> worker0: Booting VM...
==> worker0: Waiting for machine to boot. This may take a few minutes...
    worker0: SSH address: 127.0.0.1:2222
    worker0: SSH username: vagrant
    worker0: SSH auth method: private key
    worker0: Warning: Connection reset. Retrying...
    worker0:
    worker0: Vagrant insecure key detected. Vagrant will automatically replace
    worker0: this with a newly generated keypair for better security.
    worker0:
    worker0: Inserting generated public key within guest...
    worker0: Removing insecure key from the guest if it's present...
    worker0: Key inserted! Disconnecting and reconnecting using new SSH key...
==> worker0: Machine booted and ready!
==> worker0: Checking for guest additions in VM...
    worker0: The guest additions on this VM do not match the installed version of
    worker0: VirtualBox! In most cases this is fine, but in rare cases it can
    worker0: prevent things such as shared folders from working properly. If you see
    worker0: shared folder errors, please make sure the guest additions within the
    worker0: virtual machine match the version of VirtualBox you have installed on
    worker0: your host and reload your VM.
    worker0:
    worker0: Guest Additions Version: 6.1.12
    worker0: VirtualBox Version: 6.0
==> worker0: [vagrant-hostsupdater] Checking for host entries
==> worker0: [vagrant-hostsupdater] Writing the following entries to (/etc/hosts)
==> worker0: [vagrant-hostsupdater]   192.168.50.10  worker0  # VAGRANT: c561d73cbe3d4c6c1959c9a662ec99c1 (worker0) / 31bfb971-8060-4ea1-a17a-4751db2764b7
==> worker0: [vagrant-hostsupdater] This operation requires administrative access. You may skip it by manually adding equivalent entries to the hosts file.
==> worker0: Setting hostname...
==> worker0: Configuring and enabling network interfaces...
==> worker0: Mounting shared folders...
    worker0: /vagrant => /Users/akarneyeu/local-k8s
==> worker0: Running provisioner: shell...
    worker0: Running: inline script
...
==> lb: Mounting shared folders...
    lb: /vagrant => /Users/akarneyeu/local-k8s
==> lb: Running provisioner: shell...
    lb: Running: inline script

```

Проверим, что наши виртуалки пингуются:
```bash
$ ping controller1
PING controller1 (192.168.50.101): 56 data bytes
64 bytes from 192.168.50.101: icmp_seq=0 ttl=64 time=0.595 ms
64 bytes from 192.168.50.101: icmp_seq=1 ttl=64 time=0.497 ms
64 bytes from 192.168.50.101: icmp_seq=2 ttl=64 time=0.385 ms
^C
--- controller1 ping statistics ---
3 packets transmitted, 3 packets received, 0.0% packet loss
round-trip min/avg/max/stddev = 0.385/0.492/0.595/0.086 ms
```

Отлично. Как видите с этим плагином не нужно запоминать IP-адреса виртуалок, а обращаться к ним по имени.

Теперь попробуем залогиниться на `controller1` по ssh:
```bash
vagrant ssh controller1

...
vagrant@controller1:~$
```
Ура, виртуальные машины работают как надо.

### Прибираемся за собой

Пока мы можем удалить все виртуалки, чтобы они не занимали ресурсов. Потом их всегда можно пересоздать. Набираем команду:
```bash
vagrant destroy -f
```
## Заключение

Сегодня мы создали виртуальные машины. На них будет работать кластер Kubernetes. В следующей статье создадим сертификаты и конфигурационные файлы для Kubernetes.

Спасибо за внимание.
