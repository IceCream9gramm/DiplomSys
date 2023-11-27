#  Дипломная работа по профессии «Системный администратор»

# Дементьев Андрей

Содержание
==========
* [Задача](#Задача)
* [Инфраструктура](#Инфраструктура)
    * [Сайт](#Сайт)
    * [Мониторинг](#Мониторинг)
    * [Логи](#Логи)
    * [Сеть](#Сеть)
    * [Резервное копирование](#Резервное-копирование)
    * [Дополнительно](#Дополнительно)
* [Выполнение работы](#Выполнение-работы)
* [Критерии сдачи](#Критерии-сдачи)
* [Как правильно задавать вопросы дипломному 
руководителю](#Как-правильно-задавать-вопросы-дипломному-руководителю) 

---------

## Задача
Ключевая задача — разработать отказоустойчивую инфраструктуру для сайта, включающую мониторинг, сбор логов и 
резервное копирование основных данных. Инфраструктура должна размещаться в [Yandex 
Cloud](https://cloud.yandex.com/) и отвечать минимальным стандартам безопасности: запрещается выкладывать токен от 
облака в git. Используйте 
[инструкцию](https://cloud.yandex.ru/docs/tutorials/infrastructure-management/terraform-quickstart#get-credentials).

**Перед началом работы над дипломным заданием изучите [Инструкция по экономии облачных 
ресурсов](https://github.com/netology-code/devops-materials/blob/master/cloudwork.MD).**

## Инфраструктура
Для развёртки инфраструктуры используйте Terraform и Ansible.  

Важно: используйте по-возможности **минимальные конфигурации ВМ**:2 ядра 20% Intel ice lake, 2-4Гб памяти, 10hdd, 
прерываемая. 

Так как прерываемая ВМ проработает не больше 24ч, после сдачи работы на проверку свяжитесь с вашим дипломным 
руководителем и договоритесь запустить инфраструктуру к указанному времени.

**Продвинутый вариант:** Вместо создания обычной ВМ, создайте Instance Groups с прерываемыми ВМ. После остановки 
работы ВМ, Instance Groups автоматически их запустит. Подробности в  
[инструкции](https://cloud.yandex.ru/docs/compute/concepts/instance-groups/). 

Ознакомьтесь со всеми пунктами из этой секции, не беритесь сразу выполнять задание, не дочитав до конца. Пункты 
взаимосвязаны и могут влиять друг на друга.

### Сайт
Создайте две ВМ в разных зонах, установите на них сервер nginx, если его там нет. ОС и содержимое ВМ должно быть 
идентичным, это будут наши веб-сервера.

Используйте набор статичных файлов для сайта. Можно переиспользовать сайт из домашнего задания.

Создайте [Target Group](https://cloud.yandex.com/docs/application-load-balancer/concepts/target-group), включите в 
неё две созданных ВМ.

Создайте [Backend Group](https://cloud.yandex.com/docs/application-load-balancer/concepts/backend-group), 
настройте backends на target group, ранее созданную. Настройте healthcheck на корень (/) и порт 80, протокол HTTP.

Создайте [HTTP router](https://cloud.yandex.com/docs/application-load-balancer/concepts/http-router). Путь укажите 
— /, backend group — созданную ранее.

Создайте [Application load balancer](https://cloud.yandex.com/en/docs/application-load-balancer/) для 
распределения трафика на веб-сервера, созданные ранее. Укажите HTTP router, созданный ранее, задайте listener тип 
auto, порт 80.

Протестируйте сайт
`curl -v <публичный IP балансера>:80` 

### Сайт - Решение
Создаю Instance Group с тремя ВМ, Target Group, Backend Group, HTTP router и Application load balancer с помощью Terraform. Для сайта создаю внешний IP адрес и прописываю в DNS A запись сайта, который поддерживает с помощью Yandex Certificate Manager HTTPS. Адрес сайта - https://jo-os.ru

При помощи Ansible устанавливаю на ВМ nginx и копирую папку с сайтом.

![web](https://github.com/joos-net/diplom/blob/main/img/web.png)

### Мониторинг
Создайте ВМ, разверните на ней Zabbix. На каждую ВМ установите Zabbix Agent, настройте агенты на отправление 
метрик в Zabbix. 

Настройте дешборды с отображением метрик, минимальный набор — по принципу USE (Utilization, Saturation, Errors) 
для CPU, RAM, диски, сеть, http запросов к веб-серверам. Добавьте необходимые tresholds на соответствующие 
графики.

### Мониторинг - Решение
Создаю с помощью Terraform 2 ВМ - Zabbix сервер и Zabbix web, так же создаю базу данных через Yandex Managed Service for PostgreSQL.

При помощи Ansible устанавливаю Zabbix сервер с подключением к бд, далее устанавливаю Zabbix web и после установки web сервера через api создаю группу Netology и dashboard с метриками Netology. На остальные ВМ устанавливаю Zabbix Agent и с помощью api добавляю ВМ в Zabbix server.

![zabbix](https://github.com/joos-net/diplom/blob/main/img/zabbix2.png)

### Логи
Cоздайте ВМ, разверните на ней Elasticsearch. Установите filebeat в ВМ к веб-серверам, настройте на отправку 
access.log, error.log nginx в Elasticsearch.

Создайте ВМ, разверните на ней Kibana, сконфигурируйте соединение с Elasticsearch.

### Логи - Решение
Terraform создаю ВМ для Elasticsearch и Kibana.

При помощи Ansible устанавливаю на Elasticsearch - докер контейнеры Elasticsearch и Logstash, на Kibana - докер контейнер Kibana. На ВМ устанавливаю докер контейнеры Filebeat и в зависимости от ВМ собираю логи nginx или syslog.

![logs](https://github.com/joos-net/diplom/blob/main/img/logs.png)
![logs2](https://github.com/joos-net/diplom/blob/main/img/logs2.png)

### Сеть
Разверните один VPC. Сервера web, Elasticsearch поместите в приватные подсети. Сервера Zabbix, Kibana, application 
load balancer определите в публичную подсеть.

Настройте [Security Groups](https://cloud.yandex.com/docs/vpc/concepts/security-groups) соответствующих сервисов 
на входящий трафик только к нужным портам.

Настройте ВМ с публичным адресом, в которой будет открыт только один порт — ssh. Настройте все security groups на 
разрешение входящего ssh из этой security group. Эта вм будет реализовывать концепцию bastion host. Потом можно 
будет подключаться по ssh ко всем хостам через этот хост.

### Сеть - Решение

Terraform создаю один VPC с подсетями для публичных и внутренних серверов (2 внутренние подсети для разных регионов), далее создаю ВМ для Bastion и файл настройки ssh config для использования bastion при подключении по ssh к другим ВМ.
Создаю и присваиваю по именам необходимые группы безопасности. Открываю трафик внутри сетей с ВМ.
Для внешнего трафика по нужным портам открыты только bastion, web сайт, zabbix web и kibana.

![network1](https://github.com/joos-net/diplom/blob/main/img/network1.png)

![security group](https://github.com/joos-net/diplom/blob/main/img/security%20group.png)

### Резервное копирование
Создайте snapshot дисков всех ВМ. Ограничьте время жизни snaphot в неделю. Сами snaphot настройте на ежедневное 
копирование.

### Резервное копирование - Решение

При помощи Terraform - yandex_compute_snapshot_schedule - организуем резервное копирование - делаем снимки дисков ВМ раз в день.

![backup](https://github.com/joos-net/diplom/blob/main/img/backup.png)

### Дополнительно
Не входит в минимальные требования. 

1. Для Zabbix можно реализовать разделение компонент - frontend, server, database. Frontend отдельной ВМ поместите 
в публичную подсеть, назначте публичный IP. Server поместите в приватную подсеть, настройте security group на 
разрешение трафика между frontend и server. Для Database используйте [Yandex Managed Service for 
PostgreSQL](https://cloud.yandex.com/en-ru/services/managed-postgresql). Разверните кластер из двух нод с 
автоматическим failover.
2. Вместо конкретных ВМ, которые входят в target group, можно создать [Instance 
Group](https://cloud.yandex.com/en/docs/compute/concepts/instance-groups/), для которой настройте следующие 
правила автоматического горизонтального масштабирования: минимальное количество ВМ на зону — 1, максимальный 
размер группы — 3.
3. В Elasticsearch добавьте мониторинг логов самого себя, Kibana, Zabbix, через filebeat. Можно использовать 
logstash тоже.
4. Воспользуйтесь Yandex Certificate Manager, выпустите сертификат для сайта, если есть доменное имя. 
Перенастройте работу балансера на HTTPS, при этом нацелен он будет на HTTP веб-серверов.

## Выполнение работы
На этом этапе вы непосредственно выполняете работу. При этом вы можете консультироваться с руководителем по поводу 
вопросов, требующих уточнения.

⚠️ В случае недоступности ресурсов Elastic для скачивания рекомендуется разворачивать сервисы с помощью docker 
контейнеров, основанных на официальных образах.

**Важно**: Ещё можно задавать вопросы по поводу того, как реализовать ту или иную функциональность. И руководитель 
определяет, правильно вы её реализовали или нет. Любые вопросы, которые не освещены в этом документе, стоит 
уточнять у руководителя. Если его требования и указания расходятся с указанными в этом документе, то приоритетны 
требования и указания руководителя.

## Критерии сдачи
1. Инфраструктура отвечает минимальным требованиям, описанным в [Задаче](#Задача).
2. Предоставлен доступ ко всем ресурсам, у которых предполагается веб-страница (сайт, Kibana, Zabbix).
3. Для ресурсов, к которым предоставить доступ проблематично, предоставлены скриншоты, команды, stdout, stderr, 
подтверждающие работу ресурса.
4. Работа оформлена в отдельном репозитории в GitHub или в [Google Docs](https://docs.google.com/), разрешён 
доступ по ссылке. 
5. Код размещён в репозитории в GitHub.
