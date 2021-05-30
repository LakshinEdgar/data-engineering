# Создание веб-сервера и Amazon RDS DB instance
Лабораторная работа должна развить навыки по установке Apache веб-сервера и созданию базы данных MySQL. Веб-сервер будет запущен на Amazon EC2 instance с операционной системой Amazon Linux. База данных MySQL будет запущена еа MySQL DB instance. Оба instance (EC2 и DB) будут работать в виртуальном частном облаке (VPC - virtual private cloud на основе Amazon VPC service.

Когда вы создаете DB instance, вы должны указать VPC, подсеть, группы пользователей. Вы также должны указать их, когда создаете EC2 instance для вашего веб-сервера. VPC, подсеть и группы пользователей необходимы для "общения" междуй DB instance и веб-сервером. 
Далее после настройки VPC мы научимся создавать DB INSTANCE и устанавливать веб-сервер. Мы соединим веб-сервер и DB instance в VPC через endpoint.

## План работ
1. Создание Amazon VPC для использования DB instance 
    1. Создание VPC с приватными и публичными подсетями
    2. Создание дополнительных подсетей
    3. Создание группы безопасности для публичного веб-сервера
    4. Создание группы безопасности для приватного DB instance
    5. Создание группы DB подсети
2. Создание DB instance
3. Создание EC2 instance и установка веб-сервера

## 1. Создание Amazon VPC для использования DB instance 
### 1. Создание VPC с приватными и публичными подсетями
#### Создание VPC и подсети
1.  Если у вас нет Elastic IP адреса для связи со шлюзом трансляции сетевых адресов (NAT), выделите его сейчас. Если у вас есть доступный Elastic IP адрес, перейдите к следующему шагу.
    1. Откройте в консоле Amazon EC2 https://console.aws.amazon.com/ec2/.
    2. В верхнем правом углу консоли AWS Management выберите регион для размещения вашего Elastic IP адреса. Регион Elastic IP адреса должен быть тем же, где вы хотите создать VPC. В примере используется US West (Oregon) Region.
    3. В панели навигации выберите Elastic IPs.
    4. Выберите Allocate Elastic IP address.
    5. Для публичных IPv4 адресов выберите Amazon's pool of IPv4 addresses.
    6. Выберите Allocate.
    Обратите внимание на ID нового Elastic IP адреса, т.к. эта информация вам понадобится для создания VPC.
2. Откройте в консоли Amazon VPC  https://console.aws.amazon.com/vpc/.
3. В верхнем правом углу консоли AWS Management выберите Region для создания VPC. В нашем примере используется регион US West (Oregon).
4. В верхнем левом углу выберите VPC Dashboard. Начните создание VPC, выберите  Launch VPC Wizard.
5. Шаг 1: выберите страницу VPC Configuration, затем VPC with Public and Private Subnets и нажмите Select.
6. Шаг 2: на странице VPC with Public and Private Subnets установите значения:
    * IPv4 CIDR block: 10.0.0.0/16
    * IPv6 CIDR block: No IPv6 CIDR Block
    * VPC name: tutorial-vpc
    * Public subnet's IPv4 CIDR: 10.0.0.0/24
    * Availability Zone: us-west-2a
    * Public subnet name: Tutorial public
    * Private subnet's IPv4 CIDR: 10.0.1.0/24
    * Availability Zone: us-west-2a
    * Private subnet name: Tutorial private 1
    * Elastic IP Allocation ID: An Elastic IP address to associate with the NAT gateway
    * Service endpoints: Skip this field.
    * Enable DNS hostnames: Yes
    * Hardware tenancy: Default
7. Нажмите Create VPC.

### 2. Создание дополнительных подсетей.
Вы должны быть доступны 2 приватные подсети или 2 публичные подсети для создания DB группы для использования DB instance в VPC. Т.к. DB instance приватный в данной лабораторной работе, добавьте вторую приватную подсеть к VPC.
#### Создание дополнительных подсетей.
1.  Откройте в консоли Amazon VPC  https://console.aws.amazon.com/vpc/.
2.  Добавьте вторую приватную подсеть в ваш VPC, выберите VPC Dashboard, выберите Subnets и затем выберите Create subnet.
3.  На странице Create subnet установите значения:
    * VPC ID: Выберите VPC, которую вы создали на предыдущем шаге. Например:vpc-identifier (tutorial-vpc)
    * Subnet name: Tutorial private 2
    * Availability Zone: us-west-2b
    * IPv4 CIDR block: 10.0.2.0/24
4. Выберите Create subnet. Затем кликните Close страницу с конфигурацией.
5. Чтобы убедиться, что вторая приватная подсеть, котору вы создали, используеб ту же таблицу маршрутов, что и первая подсеть, выполните следующие действия:
    1. Выберите VPC Dashboard, выберите Subnets и затем выберите первую приватную подсеть, которую вы создали для VPC,  Tutorial private 1.
    2. Под списком подсетей выберите вкладку Route table и зафиксируйте значени. Например: rtb-98b613fd.
    3. На листе подсетей снимите выделение с первой приватной подсети.
    4. На листе подсетей выберите вторую приватную подсеть Tutorial private 2 и перейдите на вкладку Route table.
    5. Если текущая таблица маршрутов не совпадает с таблицей маршрутов для первой приватной подсети, выберите Edit route table association. Для Route table ID выберите таблицу маршрутов, которую вы записали ранее. Например: rtb-98b613fd. Далее сохраните свой выбор, кликните Save.
### 3. Создание группы безопасности для публичного веб-сервера
Далее создаем группу безопасности для публичного веб-сервера. Чтобы подключиться к публичным instance в вашем VPC, добавьте входящие правила в свою группу безопасности VPC, которые разрешат трафику подключаться из Интернета.
#### Создание VPC группы безопасности.
1. Откройте в консоли  Amazon VPC https://console.aws.amazon.com/vpc/
2. Выберите VPC Dashboard, выберите Security Groups и затем кликните Create security group. 
3. На странице Create security group установите значения:
    * Security group name: tutorial-securitygroup
    * Description: Tutorial Security Group
    * VPC: Выберите VPC, которую вы создали ранее. Например: vpc-identifier (tutorial-vpc)
4. Добавьте правила для входящих подключений в группу бозопасности
    1. Определите IP-адрес, который будет использоваться для подключения к instance в вашем VPC. Определите ваш публичный ID адрес, в другом окне (или вкладке) браузера вы можете использовать сервис https://checkip.amazonaws.com. Например: IP адрес 203.0.113.25/32. Если вы подключаетесь через поставщика услуг Интернета (ISP) или используете файервол без статического IP-адреса, вам необходимо выяснить диапазон IP-адресов, используемых клиентскими компьютерами. Если вы используете 0.0.0.0/0, вы разрешаете всем IP-адресам доступ к вашим общедоступным экземплярам. Такой подход приемлем на короткое время в тестовой среде, но небезопасен для производственной среды. В производственной среде вы авторизуете только определенный IP-адрес или диапазон адресов для доступа к своим экземплярам.
    2. В секции  Inbound rules, кликните Add rule.
    3. Установите следующие значения для вашего нового входящего правила, чтобы разрешить Secure Shell (SSH) доступ к вашему экземпляру EC2. Если вы это сделаете, вы можете подключиться EC2 instance для установки веб-сервера и других утилит, а также для загрузки содержимого на свой веб-сервер.
    * Type: SSH
    * Source: IP-адрес или диапазон из шага 1. Например: 203.0.113.25/32.
    4. Выберите Add rule.
    5. Установите следующие значения для вашего нового правила для входящего трафика, чтобы разрешить HTTP-доступ к вашему веб-серверу.
    * Type: HTTP
    * Source: 0.0.0.0/0.
5. Создайте группу безопасности, нажмите Create security group.
Запишите ID группы безопасности, он понадобится нам позже.
### 4. Создание группы безопасности для приватного DB instance.
Чтобы сохранить частный доступ к DB instance, создайте вторую группу безопасности для приватного доступа. Чтобы подключиться к приватным instance в вашем VPC, добавьте входящие правила в свою группу безопасности VPC, которые разрешат трафик только с вашего веб-сервера.
#### Создание VPC группы безопасности.
1. Откройте в консоли  Amazon VPC https://console.aws.amazon.com/vpc/
2. Выберите VPC Dashboard, выберите Security Groups и затем кликните Create security group. 
3. На странице Create security group установите значения:
    * Security group name: tutorial-db-securitygroup
    * Description: Tutorial DB Instance Security Group
    * VPC: Выберите VPC, которую вы создали ранее. Например: vpc-identifier (tutorial-vpc)
4. Добавьте правила для входящих подключений в группу бозопасности
    1. В секции  Inbound rules, кликните Add rule.
    2. Установите следующие значения для вашего нового входящего правила, чтобы разрешить MySQL трафик на порту 3306 из вашего экземпляа EC2. Если вы это сделаете, вы можете подключить веб-сервер к DB instance для хранения и извлечения данных из вашего веб-приложения в вашу базу данных.
    * Type: MySQL/Aurora
    * Source: Идентификатор tutorial-securitygroup группы безопасности, которую вы создали ранее. Например: sg-9edd5cfb.
5. Создайте группу безопасности, нажмите Create security group.
### 4. Создание группы DB подсети
Группа DB подсети это набор подсетей, которые вы создаете в VPC и затем назначаете для своих DB экземпляров. Группа DB подсети позволяет указать конкретный VPC при создании экземпляров DB.
#### Создание группы DB подсети
1. Откройте в консоли  Amazon RDS  https://console.aws.amazon.com/rds/
Убедитесь, что вы подключаете к Amazon RDS, а не VPC.
2. На панели навигации выберите Subnet groups.
3. Нажмите Create DB Subnet Group.
4. На странице Create DB subnet group установите следующие значения в Subnet group details:
    * Name: tutorial-db-subnet-group
    * Description: Tutorial DB Subnet Group
    * VPC: tutorial-vpc (vpc-identifier)
5. На вкладке Add subnets, выберите  Availability Zones и Subnets.
В данной лабораторной работе в Availability Zones используются us-west-2a и us-west-2b.
Далее для Subnets выберите подсети для блока 10.0.0.0/24, 10.0.1.0/24, and 10.0.2.0/24.
Обратите внимание, если вы включили Local Zone, вы можете выбрать группу  Availability Zone на странице Create DB subnet group. В данном случае выберите Availability Zone group, Availability Zones и Subnets.
6. Нажмите Create.
Ваша новая группа DB подсети появится в списке групп DB подсети в консоли RDS. Вы можете кликнуть на группу DB подсети, чтобы увидеть детали, включая все связанные с этой группой подсети. Детали отображаются в нижней части окна.

## 2. Создание DB instance
#### Создание MySQL DB instance
1. Залогиньтесь в консоли AWS Management и откройте Amazon RDS https://console.aws.amazon.com/rds/.
2. В верхнем правом углу консоли AWS Management выберите AWS Region, где вы хотите создать DB instance. В нашей лабораторной работе используется регион  US West (Oregon).
3. В панели навигации выберите Databases.
4. Нажмите Create database.
5. На странице Create database убедитесь, что выбран Standard create и затем выберите MySQL.
6. В разделе Templates выберите Free tier.
7. В разделе Settings установите следующие значения:
    * DB instance identifier – tutorial-db-instance
    * Master username – tutorial_user
    * Auto generate a password – Disable the option.
    * Master password – Введите пароль.
    * Confirm password – Повторите пароль.
8. В разделе DB instance class включите Include previous generation classes и затем установите значения:
    * Burstable classes (includes t classes)
    * db.t2.micro
9. В разделах Storage и Availability & durability используете значения по умолчанию.
10. В разделе Connectivity установите следующие значения:
    * Virtual private cloud (VPC) - выберите существующую VPC с обеими подсетями приватной и публичной, например: tutorial-vpc (vpc-identifier).
    VPC должен иметь подсети в разных Availability Zones.
    * Subnet group – группа DB подсети VPC, например tutorial-db-subnet-group
    * VPC security group – Выбрать существующий
    * Existing VPC security groups – Выберите существующую VPC группу безопасности, которая сконфигурирована для приватного доступа, например tutorial-db-securitygroup.
    Удалите остальные группы безопасности, например группу безопасности по умолчанию, выбрав X напротив каждой группы.
    * Availability Zone – No preference
    * Откройте Additional configuration, и убедитес, что Database port uиспользует значение по умолчанию 3306.
11. В разделе Database authentication убедитесь, что выбран Password authentication.
12. Откройте раздел Additional configuration и введите sample для Initial database name. Оставьте настройки по умолчаниб для остальных опций.
13. Создайте MySQL DB instance, нажмите Create database.
14. Ваш новый DB instance появится в списке Databases со статусом Creating.
15. В разделе Connectivity & security просмотрите Endpoint и Port DB instance.

##3 . Создание EC2 instance и установка веб-сервера



