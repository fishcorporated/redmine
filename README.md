# Инструкция по установке Redmine-Dashboard

Далее будут описаны шаги по установке Redmine и плагина Dashboard. Используемая операционная система - Ubuntu (Ubuntu-server) версии 14*

## Установка Redmine

### Установка зависимостей

```
sudo apt-get update && sudo apt-get upgrade -y

sudo apt-get install apache2 php5 libapache2-mod-php5 mysql-server php5-mysql libapache2-mod-perl2 libcurl4-openssl-dev libssl-dev apache2-prefork-dev libapr1-dev libaprutil1-dev libmysqlclient-dev libmagickcore-dev libmagickwand-dev curl git-core gitolite patch build-essential bison zlib1g-dev libssl-dev libxml2-dev libxml2-dev sqlite3 libsqlite3-dev autotools-dev libxslt1-dev libyaml-0-2 autoconf automake libreadline6-dev libyaml-dev libtool imagemagick apache2-utils ssh zip libicu-dev libssh2-1 libssh2-1-dev cmake libgpg-error-dev subversion libapache2-svn
```

### Настройка Subversion

```
sudo mkdir -p /var/lib/svn
sudo chown -R www-data:www-data /var/lib/svn
sudo a2enmod dav_svn
```

Откройте файл конфигурации
```
sudo nano /etc/apache2/mods-enabled/dav_svn.conf
```

Раскоментируйте следующие строки
```
<Location /svn>
    DAV svn
    SVNParentPath /var/lib/svn
    AuthType Basic
    AuthName "Subversion Repository" 
    AuthUserFile /etc/apache2/dav_svn.passwd
    AuthzSVNAccessFile /etc/apache2/dav_svn.authz
    <LimitExcept GET PROFIND OPTIONS REPORT>
    Require valid-user
    </LimitExcept>
</Location>
```

Активируйте Apache-мод **authz_svn**
```
sudo a2enmod authz_svn
```

Добавьте пользователя для чтения из репозитория
```
sudo htpasswd -c /etc/apache2/dav_svn.passwd redmine

sudo service apache2 restart
```

Создайте репозиторий
```
sudo svnadmin create --fs-type fsfs /var/lib/svn/my_repository
sudo chown -R www-data:www-data /var/lib/svn
```

Откройте файл конфигурации доступа к репозиторию
```
sudo nano /etc/apache2/dav_svn.authz
```

Добавьте права доступа к репозиторию
```
[my_repository:/]
redmine = r
```

### Установка Ruby
```
sudo apt-get install software-properties-common
sudo add-apt-repository ppa:brightbox/ruby-ng
sudo apt-get update
sudo apt-get -y install ruby2.1 ruby-switch ruby2.1-dev ri2.1 libruby2.1 libssl-dev zlib1g-dev
sudo ruby-switch --set ruby2.1
```

### Пользователи и SSH ключи

#### Пользователи
Создание пользователей для Redmine (redmine) и для Gitolite (git)
```
sudo adduser --system --shell /bin/bash --gecos 'Git Administrator' --group --disabled-password --home /opt/gitolite git
sudo adduser --system --shell /bin/bash --gecos 'Redmine Administrator' --group --disabled-password --home /opt/redmine redmine
```

Генерация SSH-ключа для пользователя **redmine**. Он будет использоваться как администратор Gitolite. Имя ключа должно быть **redmine_gitolite_admin_id_rsa**
```
sudo su - redmine
ssh-keygen -t rsa -N '' -f ~/.ssh/redmine_gitolite_admin_id_rsa
exit
```

#### Конфигурация Gitolite
```
sudo dpkg-reconfigure gitolite
```
Установите следующие параметры
 - user: git
 - repos path: /opt/gitolite
 - admin ssh-key: /opt/redmine/.ssh/redmine_gitolite_admin_id_rsa.pub
 
#### Конфигурация Visudo 
```
sudo visudo
```
Добавьте следующие строки
```
# temp - *REMOVE* after installation
redmine    ALL=(ALL)      NOPASSWD:ALL

# redmine gitolite integration
redmine    ALL=(git)      NOPASSWD:ALL
git        ALL=(redmine)  NOPASSWD:ALL
```
Это позволит **redmine** пользователю выполнять команды от имени **root** пользователя, но это необходимо для послудующей установки.

### Установка Redmine
#### Подготовка 
```
sudo su - redmine
gpg --keyserver hkp://pgp.mit.edu --recv-keys 409B6B1796C275462A1703113804BB82D39DC0E3
curl -sSL https://get.rvm.io | bash -s stable
exit
```
Необходимо выйти и зати снова
```
sudo su - redmine
rvm install 2.1.4
exit
```

#### Redmine 
```
sudo su - redmine
wget http://www.redmine.org/releases/redmine-3.2.0.tar.gz
tar zxf redmine-3.2.0.tar.gz
rm redmine-3.2.0.tar.gz
ln -s /opt/redmine/redmine-3.2.0 redmine
exit
```

#### MySQL 
```
sudo mysql -u root -p
```
Выполните следующие запросы
```
CREATE DATABASE redmine character SET utf8;
CREATE user 'redmine'@'localhost' IDENTIFIED BY 'my_password';
GRANT ALL privileges ON redmine.* TO 'redmine'@'localhost';
exit
```
Настройка соединения Redmine с базой данных
```
sudo su - redmine
sudo cp redmine/config/database.yml.example redmine/config/database.yml
```
Откройте файл настроек базы данных
```
sudo nano redmine/config/database.yml
```
Измените имя пользователя и пароль
```
database.yml:
production:
 adapter: mysql2
 database: redmine
 host: localhost
 username: redmine
 password: my_password
 encoding: utf8
```

#### Настройка
```
gem install bundler
cd redmine/
bundle install --without development test postgresql sqlite
rake generate_secret_token
RAILS_ENV=production rake db:migrate 
RAILS_ENV=production rake redmine:load_default_data
exit
```

### Удаление root-прав доступа redmine
```
sudo visudo
```
Удалите следующие строки
```
# temp - *REMOVE* after installation
redmine    ALL=(ALL)      NOPASSWD:ALL
```

### Установка Phusion Passenger
#### Добавление репозитория
Добавьте репозитория для Phusion Passenger
```
sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys 561F9B9CAC40B2F7
sudo apt-get install apt-transport-https ca-certificates
```
Откройте конфигурационный файл репозитория
```
sudo nano /etc/apt/sources.list.d/passenger.list
```
Добавьте следующую строку
```
deb https://oss-binaries.phusionpassenger.com/apt/passenger trusty main
```
Выполните команды
```
sudo chown root: /etc/apt/sources.list.d/passenger.list
sudo chmod 600 /etc/apt/sources.list.d/passenger.list
```
#### Установка
```
sudo apt-get update
sudo apt-get install libapache2-mod-passenger
```
#### Настройка
Откройте файл настроек Phusion Passenger
```
sudo nano /etc/apache2/mods-available/passenger.conf
```
Добавьте следующее содержание
```
PassengerUserSwitching on
PassengerUser redmine
PassengerGroup redmine
```

Откройте файл настроек Apache
```
sudo nano /etc/apache2/sites-available/000-default.conf
```
Добавьте следующее содержание
```
<Directory /var/www/html/redmine>
    RailsBaseURI /redmine
    PassengerResolveSymlinksInDocumentRoot on
</Directory>
```
Установите следующее значение **DocumentRoot**
```
DocumentRoot /var/www/html/redmine
```
Выполните команды
```
sudo a2enmod passenger
sudo ln -s /opt/redmine/redmine/public/ /var/www/html/redmine
sudo service apache2 restart
```

Генерация **secret_token**
```
sudo su - redmine
cd redmine
rake generate_secret_token
rake db:migrate RAILS_ENV=production
rake redmine:plugins:migrate RAILS_ENV=production
rake tmp:cache:clear
rake tmp:sessions:clear
exit
```

### Запуск Redmine
Теперь Redmine будет доступен по адресу
```
http://server-ip-address
```

Имя пользователя: admin
Пароль: admin

### Настройка Redmine
Настройка Redmine default URL

Administration > Settings > General

По умолчанию:  localhost:3000
Изменить на: server-ip-address

### Настройка e-mail
Создайте и откройте файл конфигурации Redmine
```
sudo touch /opt/redmine/redmine/config/configuration.yml
sudo nano /opt/redmine/redmine/config/configuration.yml
```

Добавьте следующие строки
```
# Outgoing email settings

production:
  email_delivery:
    delivery_method: :smtp
    smtp_settings:
      enable_starttls_auto: true
      address: smtp.host.com
      port: 587
      domain: host.com
      authentication: :login
      user_name: myname
      password: mypassword
```

### SSL, HTTPS
#### Создание сертификата
Создание Private Key
```
sudo mkdir /etc/apache2/ssl
cd /etc/apache2/ssl
sudo openssl genrsa -des3 -out server.key 1024
```
Создание CSR (Certificate Signing Request)
```
cd /etc/apache2/ssl
sudo openssl req -new -key server.key -out server.csr
```
Удаление passphrase из Private Key
```
cd /etc/apache2/ssl
sudo cp server.key server.key.org
sudo openssl rsa -in server.key.org -out server.key
```
Генерация самоподписанного сертификата
```
cd /etc/apache2/ssl
sudo openssl x509 -req -days 365 -in server.csr -signkey server.key -out server.crt
```
#### Настройка Apache
Включение SSL
```
sudo a2enmod ssl
```
Изменение настроек Apache
```
sudo nano /etc/apache2/sites-available/000-default.conf
```
Удалите следующее содержимое 
```
<Directory /var/www/html/redmine>
    RailsBaseURI /redmine
    PassengerResolveSymlinksInDocumentRoot on
</Directory>
```
Установите следующее значение DocumentRoot
```
DocumentRoot /var/www/html
```

```
sudo nano /etc/apache2/sites-available/default-ssl.conf
```
Добавьте следующее содержимое 
```
<Directory /var/www/html/redmine>
    RailsBaseURI /redmine
    PassengerResolveSymlinksInDocumentRoot on
</Directory>

Установите следующее значение DocumentRoot
```
DocumentRoot /var/www/html/redmine
```
Перезапустите Apache
```
sudo service apache2 restart
```
Теперь Redmine доступен только по адресу
```
https://server-ip-address
```
