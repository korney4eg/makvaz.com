---
title: Пример бесперебойного развёртывания сервиса на Ansible в AWS. Часть 2. Rolling Deployment
layout: post
preview_image: /assets/img/rolling-deployment-aws-with-ansible/rolling-1.png
categories: []
tags: [devops, aws, ansible, deployment]
twit: Простой пример Rolling Deployment на Ansible в AWS
---

Продолжаем серию статей по бесперебойному развёртыванию сервиса в AWS при помощи [ Ansible ]( https://www.ansible.com/). В предыдущей [статье](/2018/11/11/blue-green-deployment-with-anisble-with-aws.html) мы разобрали Blue-Green deployment.

На этот раз мы рассмотрим более долгий, но и более экономный деплоймент - Rolling deployment.
<!--more-->

## Rolling Deployment

Идея заключается в следующем:

1. У нас есть несколько инстансов на которых работает приложение версии 2.21 и лоад балансер.
![rolling-1](/assets/img/rolling-deployment-aws-with-ansible/rolling-1.png)
2. Добавляем по одной виртуалке с версией 2.22 и подключаем к балансеру.
![rolling-2](/assets/img/rolling-deployment-aws-with-ansible/rolling-2.png)
3. Отключаем одну старую ноду с версией 2.21 от лоад балансера и удаляем её.
![rolling-3](/assets/img/rolling-deployment-aws-with-ansible/rolling-3.png)
4. Повторяем шаги 2 и 3 пока не заменим все виртуалки.
![rolling-4](/assets/img/rolling-deployment-aws-with-ansible/rolling-6.png)

Этот вид развёртывания неприемлем, если есть калечащие изменения в базе данных при переходе на новую версию приложения.


## Реализация

На этот раз я решил упростить архитектуру, ведь для иллюстрации принципа работы достаточно иметь несколько серверов и лоад балансер, поэтому не будет использоваться RDS и дополнительные подсети. В качестве приложения будет использоваться nginx, который будет возвращать версию приложения и _hostname_ инстанса, на котором работает nginx.

Сразу оставлю [ ссылку на репозиторий с полным кодом ](https://github.com/korney4eg/rolling-deployment-ansible), для тех, кто хочет самостоятельно разобраться.

Как же реализовать rolling-deployment на ansible.

### 0. Используемые переменные

```yaml
{% raw %}
    region: eu-west-1
    version: 2.22
    instance_num: 3
{% endraw %}
```

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**region:** - регион AWS, в этом примере используется регион в Ирландии

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**version:** - версия приложения

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**instance_num:** - окончательное число виртуалок, с нашей версией.


### 1. Находим предыдущие инстансы.
 
#### 1.1. Смотрим есть ли инстансы нужной нам версии

Перед тем, как создавать новые виртуальные машины, убедимся, сколько их уже было создано с нужной версией и все остальные, таким образом прерванный деплой этого же приложения пе будет повторяться, а продолжиться..

Собираем данные о виртуалках с нужной версией.

```yaml
{% raw %}
    - name: Get new instances
      ec2_remote_facts:
        filters:
          vpc_id: "{{ vpc.vpc.id }}"
          instance-state-name: running
          "tag:Name": MyTestApp
          "tag:Environment": MyTest
          "tag:Version": "{{ version }}"
        region: "{{ region }}"
      register: new_instances

    - name: set new instances ids if not found previously created
      set_fact:
        new_instances: []
      when: new_instances.instances is not defined

    - name: set new instances ids if found previously created
      set_fact:
        new_instances: "{{ new_instances.instances|map(attribute='id')|list }}"
      when: new_instances.instances is defined
{% endraw%}
```
Так как из всей собранной информации нам нужны только айдишники, то в переменную **new_instances**  записывается только список айдишников виртуалок или пустой список если ничего не нашли.

#### 1.2. Находим все инстансы, включая найденные выше

В этот список будут включены инстансы из п. 1.1., поэтому отдельно удаляем их из этого списка
```yaml
{% raw %}
    - name: Get old instances
      ec2_remote_facts:
        filters:
          vpc_id: "{{ vpc.vpc.id }}"
          instance-state-name: running
          "tag:Name": MyTestApp
          "tag:Environment": MyTest
        region: "{{ region }}"
      register: old_instances

    - name: set old instances ids if not found previously created
      set_fact:
        old_instances: []
      when: old_instances.instances is not defined

    - name: set old instances ids if found previously created
      set_fact:
        old_instances: "{{ old_instances.instances|map(attribute='id')|list }}"
      when: old_instances.instances is defined

    - name: Remove new instances from old ones
      set_fact:
        old_instances: "{{old_instances|difference(new_instances)}}"
{% endraw%}
```
Точно также записываем список айдишников.


### 2. Создаём виртуалки (по очереди)

#### 2.1. Стартовый скрипт виртуальной машины

Как уже упоминалось раннее, сервис был упрощён, теперь скрипт при старте виртуалки выглядит следующим образом.

```
{% raw %}
#!/bin/bash
yum install nginx -y

# nginx configuration
echo '
server {
    listen      8081  default_server;
    location = /healthcheck {
        add_header Content-Type text/plain;
    	return 200 '{{ version }}';
    }
    location = /stack {
        add_header Content-Type text/plain;
    	return 200 '$(hostname)';
    }
}' > /etc/nginx/conf.d/virtual.conf
service nginx restart
{% endraw%}
```

Создаётся два эндпоинта:

- _/healthcheck_ - по нему лоад балансер определяет жива ли нода. Так же он показывает версию, с которой был продеплоен инстанс
- _/stack_ - показывает имя хоста

#### 2.2. По очереди создаём виртуальные машины

Так как в Ansible нельзя просто обьеденить несколько тасков в цикл, то с версии 2.1 добавили возможность запустить группу тасков циклом, обьеденённых в один файл. Выглядит эта конструкция примерно так:

```yaml
{% raw %}
    - include: add_remove.yaml
      with_sequence: start=1 end={{ instance_num }}
      loop_control:
        loop_var: instance_counter
{% endraw%}
```

Получается мы вызываем список заданий `add_remove.yaml` столько раз, сколько планируем запустить новых инстансов(здесь будет запущено **instance_num** раз).

**instance_counter** - это переменная, которая будет передаваться внутрь списка заданий.

Заглянем внутрь `add_remove.yaml`, там происходит следующее:

#### 2.3. Создаётся виртуалка и добавляется в конец списка **old_instances**

```yaml
{% raw %}
- set_fact:
    new_instances_num: "{{new_instances|length|int}}"

- name: Create EC2 server
  ec2:
    image: ami-41505fab
    wait: yes
    instance_type: t2.micro
    region: "{{ region }}"
    group_id: "{{ app_sg.group_id }}"
    vpc_subnet_id: "{{ app_subnet.subnet.id }}"
    key_name: "{{ keypair.key.name  }}"
    user_data: "{{ lookup('file', './setup.sh') }}"
    instance_tags:
      Environment: MyTest
      Name: MyTestApp
      Version: "{{ version }}"
  register: ec2
  when: new_instances_num|int < instance_num

- name: add newly create instance id to new_instances list
  set_fact:
    new_instances: "{{new_instances}}+{{ec2.instance_ids}}"
    new_instances_num: "{{new_instances|int + 1}}"
  when: ec2.changed == True
{% endraw%}
```
Условие используется для того, чтобы не создавать лишние инстансы.

#### 2.2 Добавляем в Elastic Load Balancer и ждём, когда виртуалка запустится

```yaml
{% raw %}
- name: Create ELB with all instances
  ec2_elb_lb:
    name: "app-rolling-lb"
    state: present
    security_group_ids:
      - "{{ elb_sg.group_id }}"
    region: "{{ region }}"
    instance_ids: "{{ new_instances|union ( old_instances)}}"
    subnets:
      - "{{ app_subnet.subnet.id}}"
    listeners:
      - protocol: http
        load_balancer_port: 80
        instance_port: 8081
    health_check:
        ping_protocol: http # options are http, https, ssl, tcp
        ping_port: 8081
        ping_path: "/healthcheck" # not required for tcp or ssl
        response_timeout: 15 # seconds
        interval: 30 # seconds
        unhealthy_threshold: 2
        healthy_threshold: 2
    tags:
      Environment: MyTest

- name: Waiting for instances to become ready
  ec2_elb_facts:
    region: "{{ region }}"
    names: app-rolling-lb
  register: elb_facts
  until: elb_facts.elbs[0].instances_inservice_count  == (new_instances|union ( old_instances)|length)
  retries: 10
  delay: 15
{% endraw%}
```
Добавляем старые и новые инстансы в балансер, на случай если они не были добавлены раньше. Затем ждём когда все смогут ответить на запросы балансера.

#### 2.3 Самое хитрое место 

```yaml
{% raw %}
- name: Terminate instances that were previously launched
  ec2:
    state: 'absent'
    instance_ids: '{{ old_instances.0 }}'
    region: "{{ region }}"
    group_id: "{{ app_sg.group_id }}"
    vpc_subnet_id: "{{ app_subnet.subnet.id }}"
  when: ec2.changed == True and old_instances.0 is defined

- name: remove instance_id from old_instances list
  set_fact:
    old_instances: "{{old_instances[1:]}}"
  when: ec2.changed == True and old_instances.0 is defined
{% endraw%}
```

Удаляем первую виртуалку физически и её айдишник из списка **old_instances**, если новая была создана.

При удалении виртуалки, она автоматически удаляется из лоад балансера, поэтому отдельный таск под это не создавался.

## Запускаем ansible

Ещё раз продублирую [ ссылку ](https://github.com/korney4eg/rolling-deployment-ansible) на полный плейбук.


Проще всего будет клонировать весь репозиторий и запустить скрипт:
```bash
git clone https://github.com/korney4eg/rolling-deployment-ansible
cd rolling-deployment-ansible/
ansible-playbook provision.yaml  -v
```

Чтобы вам долго не мучиться с удалением только что созданных ресурсов я подготовил скрипт, который удаляет всё в обратном порядке, для этого нужно выполнить следующую команду:
```bash
ansible-playbook destroy.yaml  -v
```

## Что можно сделать лучше

1. Разбить один большой файл на несколько мелких, это бы дало лучшую читабельность ([как советуют в Ansible](https://docs.ansible.com/ansible/latest/user_guide/playbooks_best_practices.html)).

## Заключение
На этом примере был рассмотрен один из видов беспрерывного развёртывания (Zero-downtime deployment) - Поэтапное развёртывание (Rolling Deployment). Как уже было сказано выше, этот способ экономит деньги(немного), потому что в один момент времени используется _n+1_ виртуалок, в то время как при Blue-Green Deployment - _2*n_.

К недостаткам этого метода можно отнести следующие пункты:

1. Медленный, занимает времени больше, чем при Blue-Green Deployment в N раз, где N - количество виртуалок.
2. Если что-то сломается в середине процесса, например на 3-й виртуалке из 5-ти, то придётся заранее думать, как всё починить и идти дальше. Хотя можно поставить деплоить виртуалки со старой версией.

Не смотря на недостатки этот способ используется по-умолчанию в оркестраторах контейнеров (например [ kubernetes ](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)).
