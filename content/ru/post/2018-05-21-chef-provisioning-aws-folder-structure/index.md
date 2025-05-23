---
layout: post
title: Зачем chef provisioning пересоздал всё в AWS и как это поправить.
archives: "2018"
# featureImg: "image/location"
# thumbnail: "image/location"
tags: [chef, aws, техническое]
---
Для того, что бы накатывать изминения в этот и некоторые другие сайты очень удобно пользоваться инструментами конфигурационного управления(Configuration Management tools), такими как OpsCode Chef, Puppet, RedHad Ansible. Мне ближе всего и удобнее использовать Chef потому что, это DSL к Ruby и код легко читается и пишется, а к тому же несколько лет назад был дополнен ещё надстройкой для создания и конфигурирования облачных ресурсов [Chef Provisioning](https://docs.chef.io/provisioning.html).
<!--more-->

Понадобилось мне как-то кое-что поправить в конфигурации сайта с другого компьютера, и хотелось всё сделать правильно, тоесть в код прописать, закоммитить, нажать кнопку и всё чтобы стало хорошо. Залогинился я в консоль aws, создал там дополнительную пару ключей, потому как старая осталась на другом компе, к которому на тот момент доступа совершенно не было, и прописал эти ключи в aws config. Запустил `chef-client ...` и обалдел...

Не понятно почему, chef стал создавать все ресурсы (VPC, Internet Gateway, Subnets) заново, хотя такие уже существуют. Я полез, проверил конфиги - там всё правильно. Затем стал смотреть по статусу git, там тоже всё без изменений. Пошел в веб консоль, удалил новосозданные ресурсы и полез проверить структуру папок, не появилось ли чего нового. Оказалось, что появилось 2 новые папки: _nodes_ и _data_bags_. Вот в этих папках и храниться метадата ресурсов, о которой будет несколько подробнее.

Как сказано в [документации](https://github.com/chef/chef-provisioning/blob/master/docs/faq.md), в _nodes_ сохранятются все атрибуты о машинах, на которых запускался chef, в том числе и её id.

На cоставе данных в _data_bags_ я бы хотел остановиться по-подробнее, потому что мне пришлось почти вручную воссоздавать структуру с нужными параметрами. Что из себя представляет эта дирректория:

# Структура _data_bags_
```bash
data_bags/
├── aws_internet_gateway
│   └── igw-managed-by-VPCID.json
├── aws_subnet
│   └── mysite_subnet.json
└── aws_vpc
    └── vpc-vpc.json
```

## aws_vpc
Цель - хранение информации о VPC.
Посмотрим на файл _mysite-vpc.json_ по-ближе:

```json
{
  "id": "mysite-vpc",
  "reference": {
    "id": "VPCID"
  },
  "driver_url": "aws:mysite:eu-central-1"
}
```

, в котором:

**id** - название vpc, записанное в рецепте создания инфраструктуры,

**reference.id** - идентификатор vpc в AWS, вида *vpc-123abcde*

**driver_url** - строка драйвера для chef-provisioning, где **aws** - мы работаем с AWS, **mysite** - имя профиля конфигурации AWS, **eu-central-1** - регион AWS, где будет создаваться оборудование.

## aws_internet_gateway
Как нетрудно догадаться из названия, здесь храниться настройки AWS internet gateway. Как устроен файл _igw-managed-by-VPCID.json_:
```json
{
  "id": "igw-managed-by-VPCID",
  "reference": {
    "id": "VPCID"
  },
  "driver_url": "aws:vervedea:eu-central-1"
}

```
Структурно, очень похоже на _mysite-vpc.json_

**id** - идентификатор, под которым chef-provisioning записал internet gateway

Всё остальное точно такое же, как и в конфиге VPC.

## aws_subnet
Ссылка на подсети.

_mysite_subnet.json_:

```json
{
  "id": "vervedea_subnet",
  "reference": {
    "id": "subnet-7b93d410"
  },
  "driver_url": "aws:vervedea:eu-central-1"
}
```
**id** - идентификатор, под которым chef-provisioning записал подсеть (subnet)

**reference.id** - идентификатор подсети в AWS, вида **subnet-123abcde**

**driver_url** - точно так же, как и в первом файле.

# Вывод
Так как _data_bags_ и _nodes_ были сгенерированы chef-ом автоматически, я не стал их добавлять в репозиторий, думая, что в следующий раз они заново сгенерируются. Так и оказалось, просто были созданны новые ресурсы. В принципе нет никакой причины не добавлять _data_bags_ и _nodes_ в репозиторий, тем более, что такие конфигурации не храняться в публичных репозиториях.
