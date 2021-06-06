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

## 3 . Создание EC2 instance и установка веб-сервера
#### Запуск EC2 instance
1. Залогиньтесь в консоли AWS Management и откройте Amazon EC2 https://console.aws.amazon.com/ec2/.
2. Выберите EC2 Dashboard и запустите instance, как показано на скрине.
3. Выберите Amazon Linux 2 AMI.
4. Выберите t2.micro тип instance, как показано на скрине, и затем выберите Next: Configure Instance Details.
5. На странице Configure Instance Details установите следующие значения.
    * Network: выберите VPC с обеими подсетями публичной и приватной, которые мы создали для DB instance vpc-identifier | tutorial-vpc
    * Subnet: Выберите существующую подсеть subnet-identifier | Tutorial public | us-west-2a (Созданную в разделе Создание VPC группы безопасности.)
    * Auto-assign Public IP: Выберите Enable

6. Выберите Next: Add Storage
7. На странице Add Storage оставьте значения по умолчанию и выбреите Next: Add Tags
8. На странцие Add Tags выберите Add tag? затем введите Name для ключа и tutorial-web-server для значения
9. Выберите Next: Configure Security Group
10. На странице Configure Security Group выберите Select an existing security group. Затем выберите существующую группу безопасности tutorial-securitygroup, созданную в разделе Создание группы безопасности для публичного веб-сервера.
11. Выбериет Review and Launch
12. На странице review Instance Launch подтвердите свои настройки и нажмите Launch.
13. На странице Select an existing key pair or create a new key pair выберите Create a new key pair и установите в Key pair name значение tutorial-key-pair. Выберите Download Key Pair и сохраните файле key pair на свой локальный компьютер. Мы будем использовать этот файл для установки соединение с EC2 instance.
14. Запустите EC2 instance, выберите Launch Instance. На стрианице Launch Status запишите ID EC2 instance (Например: i-0288d65fd4470b6a9).
15. Выберите View Instnce, чтобы найти ваш instance.
16. Прежде чем продолжить, дождитесь, пока  Instance Status  не станет Running.

### Установка Apache web-server с PHP.
1. Подключитесь к EC2 instnace.
2. Получите последние исправления ошибок и обновления безопасности, обновив ПО на EC2 instance. Для этого используйте следующую команду.
sudo yum update -y
3. После завршения обновления установите PHP, используя команду amazon-linux-extras install. Эта команда устанавливает несколько пакетов и заивисомстей одновременно.
amazon-linux-extras install 
Если у вас появится ошибка amazon-linux-extras: command not, значит ваш EC2 instance не был запущен с AMI Amazon Linux 2 (возможно, вместо него вы используете AMI Amazon Linux). Вы можете посмотреть свою версию с помощью команды:
cat /etc/system-release
4. Установите Apache web-сервер
sudo yum install -y httpd
5. Запустите web-сервер
sudo systemctl start httpd
Вы можете проверить правильно ли ваш сервер установлен и запущен.
Введите значение вашего публичного DNS имени (Domain name System) EC2 instance в адресную строку браузера. Например: http://ec2-42-8-168-21.us-west-1.compute.amazonaws.com. Если web-сервер запущен, то вы увидите тестовую страницу Apache.
Если вы не видите тестовую страницу Apache, проверьте правила группы безопасности VPC. Убедитесь, что правила активирован доступ к 80 порту для IP адресов, которые вы используете для соединения к web-серверу.
6. Настройте web-сервер для запуска при каждой загрузке системы с помощью команды systemctl.
sudo systemctl enable httpd
Чтобы позволить пользователю EC2 управлять файлами в корневом каталоге по умоланию для вашего веб-сервера Apache, измените владельца и права доступа к каталогу /var/www directory.
Есть много способов выполнить эту задачу. В этом руководстве вы добавляете пользователя EC2 в группу Apache, чтобы предоставить группе Apache права владельца и редактора на каталог /var/www.


### Установка прав доступа к файлам для web-сервера Apache.
1. Добавьте ec2-user в группу apache.
sudo usermod -a -G apache ec2-user
2. Выйдите из системы, чтобы обновить свои разрешения и включить новую группу apache.
exit
3. Снова войдите и убедитесь, что группа apache существует.
groups
В ответ вы должны получить следующее сообщение
ec2-user adm wheel apache systemd-journal
4. Измените групповое владение каталогом /var/www и его содержимым на группу apache.
sudo chown -R ec2-user:apache /var/www
5. Измените права доступа к каталогу /var/www и его подкаталогам, чтобы добавить права записи для группы и установить идентификатор группы для подкаталогов, созданных в будущем.
sudo chmod 2775 /var/www
find /var/www -type d -exec sudo chmod 2775 {} \;
6. Рекурсивно измените разрешения для фпйлов в каталоге/var/www и его подкаталогах, чтобы добавить права записи для группы.
find /var/www -type f -exec sudo chmod 0664 {} \;
Теперь ec2-user (и любые будущие члены группы apache) могут добавлять, удалять и редактировать файлы в корне документа Apache, что позволяет вам добавлять контент. Например: статический веб-сайт или PHP-приложение.
### Подлючите ваш Apache web-server к DB instance 
1. Пока вы все еще подключены к вашему EC2 instance измените каталог на /var/www и создайте новый подкаталог с именем inc.
cd /var/www
mkdir inc
cd inc
2. Создайте новый файл в категории inc и назовите его dbinfo.inc. Затем вызовите редактор nano
>dbinfo.inc
nano dbinfo.inc
3. Добавьте следующее содержимое в файл dbinfo.inc. 
Здесь db_instance_endpoint endpoint DB instance без порта и master password пароль от DB instance
<?php

define('DB_SERVER', 'db_instance_endpoint');
define('DB_USERNAME', 'tutorial_user');
define('DB_PASSWORD', 'master password');
define('DB_DATABASE', 'sample');

?>

4. Сохраните и закройте файл dbinfo.inc.
5. Измените каталог на /var/www/html.
cd /var/www/html
6. Создайте новый файл в директории html и назовите его SamplePage.php. Вызовите nano для редактирования файла
>SamplePage.php
nano SamplePage.php
7. Добавьте следующее содердимое в файл SamplePage.php

<?php include "../inc/dbinfo.inc"; ?>
<html>
<body>
<h1>Sample page</h1>
<?php

  /* Connect to MySQL and select the database. */
  $connection = mysqli_connect(DB_SERVER, DB_USERNAME, DB_PASSWORD);

  if (mysqli_connect_errno()) echo "Failed to connect to MySQL: " . mysqli_connect_error();

  $database = mysqli_select_db($connection, DB_DATABASE);

  /* Ensure that the EMPLOYEES table exists. */
  VerifyEmployeesTable($connection, DB_DATABASE);

  /* If input fields are populated, add a row to the EMPLOYEES table. */
  $employee_name = htmlentities($_POST['NAME']);
  $employee_address = htmlentities($_POST['ADDRESS']);

  if (strlen($employee_name) || strlen($employee_address)) {
    AddEmployee($connection, $employee_name, $employee_address);
  }
?>

<!-- Input form -->
<form action="<?PHP echo $_SERVER['SCRIPT_NAME'] ?>" method="POST">
  <table border="0">
    <tr>
      <td>NAME</td>
      <td>ADDRESS</td>
    </tr>
    <tr>
      <td>
        <input type="text" name="NAME" maxlength="45" size="30" />
      </td>
      <td>
        <input type="text" name="ADDRESS" maxlength="90" size="60" />
      </td>
      <td>
        <input type="submit" value="Add Data" />
      </td>
    </tr>
  </table>
</form>

<!-- Display table data. -->
<table border="1" cellpadding="2" cellspacing="2">
  <tr>
    <td>ID</td>
    <td>NAME</td>
    <td>ADDRESS</td>
  </tr>

<?php

$result = mysqli_query($connection, "SELECT * FROM EMPLOYEES");

while($query_data = mysqli_fetch_row($result)) {
  echo "<tr>";
  echo "<td>",$query_data[0], "</td>",
       "<td>",$query_data[1], "</td>",
       "<td>",$query_data[2], "</td>";
  echo "</tr>";
}
?>

</table>

<!-- Clean up. -->
<?php

  mysqli_free_result($result);
  mysqli_close($connection);

?>

</body>
</html>


<?php

/* Add an employee to the table. */
function AddEmployee($connection, $name, $address) {
   $n = mysqli_real_escape_string($connection, $name);
   $a = mysqli_real_escape_string($connection, $address);

   $query = "INSERT INTO EMPLOYEES (NAME, ADDRESS) VALUES ('$n', '$a');";

   if(!mysqli_query($connection, $query)) echo("<p>Error adding employee data.</p>");
}

/* Check whether the table exists and, if not, create it. */
function VerifyEmployeesTable($connection, $dbName) {
  if(!TableExists("EMPLOYEES", $connection, $dbName))
  {
     $query = "CREATE TABLE EMPLOYEES (
         ID int(11) UNSIGNED AUTO_INCREMENT PRIMARY KEY,
         NAME VARCHAR(45),
         ADDRESS VARCHAR(90)
       )";

     if(!mysqli_query($connection, $query)) echo("<p>Error creating table.</p>");
  }
}

/* Check for the existence of a table. */
function TableExists($tableName, $connection, $dbName) {
  $t = mysqli_real_escape_string($connection, $tableName);
  $d = mysqli_real_escape_string($connection, $dbName);

  $checktable = mysqli_query($connection,
      "SELECT TABLE_NAME FROM information_schema.TABLES WHERE TABLE_NAME = '$t' AND TABLE_SCHEMA = '$d'");

  if(mysqli_num_rows($checktable) > 0) return true;

  return false;
}
?>                        
                
8. Сохраните и закройте файл SamplePage.php.
9. Убедитесь, что ваш web-сервер успешно подключается к вашему DB instance, открыв веб-браузер и перейдя по адресу http://EC2 instance endpoint/SamplePage.php. Например: http://ec2-55-122-41-31.us-west-2.compute.amazonaws.com/SamplePage.php

Вы можете использовать SamplePage.php для добавления данных в свой DB instance. Добавленные данные затем отобразятся на странице. 
Чтобы убедиться, что данные были вставлены в таблицу, вы можете установить MySQL в EC2 instance, подключиться к DB instance и запросить таблицу.

Чтобы убедиться, что ваш DB instance максимально безопасен, проверьте, чтобы источники за пределами VPC не могли подключиться к вашему DB instance.

После завершения тестирования web-сервера и базы данных следует удалить DB instance и EC2 instance.
