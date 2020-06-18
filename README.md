## компоненты системы

### Список
- kafka - сервис доставки сообщений  
- zookepeer - кластер хранения настроек используется kafka как бд
- mongo - бд используется для хранения логов
- postgres - бд
- nginx - вэб сервер  
- configservice - сервис выдачи конфигурации 
- servicerregistry - сервис дискавери 
- serviceadmin - сервис для отображения и администрирования сервисами на актуатурах
- ribapi - входящей api по протоколу rib
- jsonapi - сервис используется для личного кабинета 
- backServer - сервис обработки платежа 
- gatewayabs - сохраняет в бд абс проводки

### kafka и zookeeper

Kafka cервис для асинхроной доставки сообщения, zookeeper используется для хранения, 
оба сервиса работают в режиме кластера, кафка и zookeeper имеет id машин
настройки кафки прописаны в файле
/opt/kafka/kafka/config/real-server.properties
важные настройки   
```properties
 broker.id=3
listeners=PLAINTEXT://testdb:9092
#Здесь должн имя ноды на которой запускается кафка
log.dirs=/opt/kafka/data
zookeeper.connection.timeout.ms=6000
``` 
Настройки zookeeper
/opt/zookeeper/zookeeper/conf/zoo.cfg
```properties
dataDir=/opt/zookeeper/data
# The port at which the clients will connect
clientPort=2181

server.3=10.10.33.3:2888:3888
server.2=10.10.33.5:2888:3888
server.1=10.10.33.7:2888:3888
# здесь номер должен точно совпадать с номером прописанным  в файле
#/opt/zookeeper/data/myid
```

####
- порты подключения: 2181, 9092
- порты работы: 2888,3888 
- протокол: tcp

### mongo 
используется для хранения логов не должны иметь аутификацию,
 сейчас на тестовом сервере запущена в контейнере с ограничением ресурсов

####
- порты подключения: 27017
- протокол: tcp

### postgres
основна бд используется для хранения бизнес информации

на тестовом сервере установлена в папке /opt/dbase/pgsql/12/data

конфигурация pg_hba.conf настройки доступа 
```text
local   all             all                                     peer
host    all             all             127.0.0.1/32            md5
host    all             all             ::1/128                 md5
local   replication     all                                     peer
host    replication     all             127.0.0.1/32            md5
host    replication     all             ::1/128                 md5
host    all             all             0.0.0.0/0               md5
host    all             all             192.168.18.0/24         md5
``` 
конфигурация postgresql.conf
измененные параметры  
```properties
listen_addresses = '*'
max_connections = 300
shared_preload_libraries = 'pg_stat_statements'
pg_stat_statements.max = 10000
pg_stat_statements.track = all
```

Добавить расширения 
```shell script
yum install postgresql12-contrib
```
####
- порты подключения: 5432
- протокол: tcp

### nginx
вэб сервер занимается балансировкой нагрузки, аутификаций, и отдает статику

//todo добавить настройки сюда

### configservice
сервис берет конфигурацию с открытого хранилища и отдает расшифрованные конфиги
 
[подробней можно прочитать]( https://cloud.spring.io/spring-cloud-config/reference/html/)

[исходный код](https://dev.agria.pro/service/config)

[jar файл ](https://dev.agria.pro/nexus/repository/maven-public/ru/nyrk/configservice/2.0.4/configservice-2.0.4.jar)  

[конфигурация](https://github.com/OlegNyr/config-proc)

####
- порты подключения: 8888
- протокол: http

### servicerregistry
сервис регистрации сервисов, каждый сервис при запуске региструется в нем, он содержит полный список запученных сервисов 

[подробней можно прочитать]( https://spring.io/projects/spring-cloud-netflix#overview)

[исходники](https://dev.agria.pro/service/serviceregistry)

[jar файл](https://dev.agria.pro/nexus/repository/maven-public/ru/nyrk/eureka/2.0.1/eureka-2.0.1.jar)

####
- порты подключения: 8761
- протокол: http

### serviceadmin
сервис админка по настройкам и т.д.

[боевой контур](https://billinger-new.rnkorib.ru/admin-mon#/applications)

[тестовый контур](https://billinger-new-test.rnkorib.ru/admin-mon#/applications)

[подробней можно прочитать](https://codecentric.github.io/spring-boot-admin/current/)

[исходники](https://dev.agria.pro/service/admin)

[Jar файл](https://dev.agria.pro/nexus/repository/maven-public/ru/nyrk/admin/2.0.3/admin-2.0.3.jar)

####
- порты подключения: 9999
- протокол: http

### ribapi
реализует протокол РИБ для платежей агрегаторами

[Jar файл](https://dev.agria.pro/nexus/repository/maven-public/biz/ep/proc/ribapi/2.1.0/ribapi-2.1.0-boot.jar)

####
- порты подключения: 8081
- протокол: http
 
### jsonapi
сервис отдает сущьности бд в формате [jsonapi](https://jsonapi.org/) используя компонент [crnk](http://www.crnk.io/)  

[Jar файл](https://dev.agria.pro/nexus/repository/maven-public/biz/ep/proc/jsonapi/2.1.0/jsonapi-2.1.0-boot.jar)

####
- порты подключения: 8082
- протокол: http

### backServer
сервис обрабатывает платеж и вызывает скрипты для отправки в шлюз  

[Jar файл](https://dev.agria.pro/nexus/repository/maven-public/biz/ep/proc/backServer/2.1.0/backServer-2.1.0-boot.jar)

[исходники](https://dev.agria.pro/service/admin)

####
- порты подключения: 8083
- протокол: http

### gatewayyabs
Сохраняет данные в БД абс, получаем параметры для сохранения по rest

[исходниики](https://dev.agria.pro/rib/gatewayyabs)

####
- порт: в заависимости от стенда
- протокол: http

## диаграмма развертываения 
![диаграмма развертывания](http://www.plantuml.com/plantuml/png/dOx1QW8n48RlynGUxM7NgeWNQn_2On6P9BDB6ZOJ9h5KdzwuQsJ15iHXBly_0z_dsT0aKKmOmdpo1VFtEkzoMQ-XgYfms3Y4CxCZ2YJmaGSsQXj9Vgmc4MfjJ7BQpDHsrAFfh2-TM8N1blGTsPTOvm6V3Gxq6rZIW2jX3_sjs2t6TgkNx1HgjyitakVm1XCgZ8E2KME1p_y5ElOjJmrhH_9snMhsZMrlKd-J2-afanxmADFNWq8GxYZHCK8hsA37Hs9dTVyaTye5)


## взаимодействие платежа
![Взаимодействие](http://www.plantuml.com/plantuml/png/VP9TJi9058NVPnMp060193GnSGF6Lr-cC986KamfAc-q-1iVM89RA9LgGnMsS6T7dZjjqM8YVPcvlUVhkUSIUk5OItgbp2meemqbvf4IINigS8nHUgT4dFJ3II3xOq_xeN0dCt-WWhdqXvv_AyggR3lblIlD4-Nc3hWTfPJG5-MKjUI5Bm5S0ecNA6rnUy1v9QoCzO7dgYklYpvAqRS38gUFkeyH5bYWka6bL9-gKBbggkMWEeWI8ziWrZMOYi2x1v4DPMpAh8wdF9lfcPOZTJbI1uXJthlSsuo8YNZMvhdc2TlA6bVrH-SwkDQ15UwemEGGeVFpZv0oKVkqfXjYZ4zlPc6MxD3AVviMFVmiYmTROyIOWpqQhDSJFyosCrIXlb5pvKuh_fvtPI9fnziYOquzvmrGodip4Rndk2SjYOzmWlOZkJHnXpQT1hqvUlKluEIsDZEcc1XfH01BX5dm7bBpKbjkmFxR7sMLjmEHFHrdaNy3)