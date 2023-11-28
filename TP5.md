# Partie 1 : Mise en place et maîtrise du serveur Web

## 1. Installation

🌞 **Installer le serveur Apache**

- paquet `httpd`

````
[quentin@web ~]$ sudo dnf install httpd
````

🌞 **Démarrer le service Apache**

- démarrez-le
````
[quentin@web conf]$ sudo systemctl start httpd
````
````
[quentin@web conf]$ sudo systemctl status httpd
● httpd.service - The Apache HTTP Server
     Loaded: loaded (/usr/lib/systemd/system/httpd.service; disabled; vendor preset: disabled)
     Active: active (running) since Mon 2022-12-12 16:06:46 CET; 5s ago
````

 - faites en sorte qu'Apache démarre automatiquement au démarrage de la machine
````
[quentin@web conf]$ sudo systemctl enable httpd
Created symlink /etc/systemd/system/multi-user.target.wants/httpd.service → /usr/lib/systemd/system/httpd.service.
````

 - ouvrez le port firewall nécessaire
````
[quentin@web conf]$ sudo firewall-cmd --add-port=80/tcp --permanent
[sudo] password for quentin:
success
````

 - utiliser une commande `ss` pour savoir sur quel port tourne actuellement Apache
````
  [quentin@web conf]$ sudo ss -alpnt | grep httpd
LISTEN 0      511                *:80              *:*    users:(("httpd",pid=1388,fd=4),("httpd",pid=1387,fd=4),("httpd",pid=1386,fd=4),("httpd",pid=1384,fd=4))
````

🌞 **TEST**

- vérifier que le service est démarré
````
[quentin@web conf]$ sudo journalctl -xe -u httpd

Dec 12 16:06:46 web.linux.tp5 httpd[1384]: Server configured, listening on: port 80
````
- vérifier qu'il est configuré pour démarrer automatiquement
````
[quentin@web conf]$ sudo systemctl status httpd | grep enabled
     Loaded: loaded (/usr/lib/systemd/system/httpd.service; enabled; vendor preset: disabled)
````
- vérifier avec une commande `curl localhost` que vous joignez votre serveur web localement
````
[quentin@web conf]$ curl --silent localhost | head -n 5
<!doctype html>
<html>
  <head>
    <meta charset='utf-8'>
    <meta name='viewport' content='width=device-width, initial-scale=1'>
````

- vérifier depuis votre PC que vous accéder à la page par défaut
  - avec une commande `curl` depuis un terminal de votre PC (je veux ça dans le compte-rendu, pas de screen)
````
qcass@LAPTOP-4K0GL7E MINGW64 ~
$ curl --silent 10.105.1.11
<!doctype html>
<html>
  <head>
    <meta charset='utf-8'>
    <meta name='viewport' content='width=device-width, initial-scale=1'>
    <title>HTTP Server Test Page powered by: Rocky Linux</title>
    <style type="text/css">
````

## 2. Avancer vers la maîtrise du service

🌞 **Le service Apache...**

- affichez le contenu du fichier `httpd.service` qui contient la définition du service Apache
````
[quentin@web system]$ cat /usr/lib/systemd/system/httpd.service
````

🌞 **Déterminer sous quel utilisateur tourne le processus Apache**

- mettez en évidence la ligne dans le fichier de conf principal d'Apache (`httpd.conf`) qui définit quel user est utilisé
````
[quentin@web conf]$ cat httpd.conf | grep -i user
User apache
````

- utilisez la commande `ps -ef` pour visualiser les processus en cours d'exécution et confirmer que apache tourne bien sous l'utilisateur mentionné dans le fichier de conf
````
[quentin@web conf]$ ps -ef | grep apache
apache      1385    1384  0 16:06 ?        00:00:00 /usr/sbin/httpd -DFOREGROUND
apache      1386    1384  0 16:06 ?        00:00:00 /usr/sbin/httpd -DFOREGROUND
apache      1387    1384  0 16:06 ?        00:00:00 /usr/sbin/httpd -DFOREGROUND
apache      1388    1384  0 16:06 ?        00:00:00 /usr/sbin/httpd -DFOREGROUND
apache      1722    1384  0 16:32 ?        00:00:00 /usr/sbin/httpd -DFOREGROUND
quentin     1836     863  0 16:43 pts/0    00:00:00 grep --color=auto apache
````

- la page d'accueil d'Apache se trouve dans `/usr/share/testpage/`
  - vérifiez avec un `ls -al` que tout son contenu est **accessible en lecture** à l'utilisateur mentionné dans le fichier de conf
````
[quentin@web share]$ ls -la | grep testpage
drwxr-xr-x.   2 root root    24 Dec 12 15:49 testpage
````

🌞 **Changer l'utilisateur utilisé par Apache**

- créez un nouvel utilisateur
  - pour les options de création, inspirez-vous de l'utilisateur Apache existant
````
[quentin@web etc]$ sudo adduser user1 -d /usr/share/httpd -s /sbin/nologin
[sudo] password for quentin:
adduser: warning: the home directory /usr/share/httpd already exists.
adduser: Not copying any file from skel directory into it.
Creating mailbox file: File exists
````

````
[quentin@web httpd]$ cat /etc/passwd | grep user1
user1:x:1001:1001::/usr/share/httpd:/sbin/nologin
````

- modifiez la configuration d'Apache pour qu'il utilise ce nouvel utilisateur
````
[quentin@web conf]$ cat httpd.conf | grep user1
User user1
````

- utilisez une commande `ps` pour vérifier que le changement a pris effet
````
[quentin@web conf]$ ps -ef | grep httpd
root        1931       1  0 17:12 ?        00:00:00 /usr/sbin/httpd -DFOREGROUND
user1       1932    1931  0 17:12 ?        00:00:00 /usr/sbin/httpd -DFOREGROUND
user1       1933    1931  0 17:12 ?        00:00:00 /usr/sbin/httpd -DFOREGROUND
user1       1934    1931  0 17:12 ?        00:00:00 /usr/sbin/httpd -DFOREGROUND
user1       1935    1931  0 17:12 ?        00:00:00 /usr/sbin/httpd -DFOREGROUND
````

🌞 **Faites en sorte que Apache tourne sur un autre port**

- modifiez la configuration d'Apache pour lui demander d'écouter sur un autre port de votre choix
````
[quentin@web conf]$ cat httpd.conf | grep -i listen
Listen 12510
````
- prouvez avec une commande `ss` que Apache tourne bien sur le nouveau port choisi
````
[quentin@web conf]$ sudo ss -alntp | grep httpd
LISTEN 0      511                *:12510            *:*    users:(("httpd",pid=2404,fd=4),("httpd",pid=2403,fd=4),("httpd",pid=2402,fd=4),("httpd",pid=2399,fd=4))
````

- vérifiez avec `curl` en local que vous pouvez joindre Apache sur le nouveau port
````
[quentin@web conf]$ curl --silent localhost:12510 | head 5
head: cannot open '5' for reading: No such file or directory
[quentin@web conf]$ curl --silent localhost:12510 | head -5
<!doctype html>
<html>
  <head>
    <meta charset='utf-8'>
    <meta name='viewport' content='width=device-width, initial-scale=1'>
````

- vérifiez avec votre navigateur que vous pouvez joindre le serveur sur le nouveau port
````
qcass@LAPTOP-4K0GL7E MINGW64 ~
$curl --silent 10.105.1.11:12510
<!doctype html>
<html>
  <head>
    <meta charset='utf-8'>
    <meta name='viewport' content='width=device-width, initial-scale=1'>
    <title>HTTP Server Test Page powered by: Rocky Linux</title>
    <style type="text/css">
      /*<![CDATA[*/
````


# Partie 2 : Mise en place et maîtrise du serveur de base de données

🌞 **Install de MariaDB sur `db.tp5.linux`**

- je veux dans le rendu **toutes** les commandes réalisées
````
[quentin@db ~]$ sudo dnf install mariadb-server
````
````
[quentin@db ~]$ sudo mysql_secure_installation
````

- faites en sorte que le service de base de données démarre quand la machine s'allume
````
[quentin@db ~]$ sudo systemctl enable mariadb
Created symlink /etc/systemd/system/mysql.service → /usr/lib/systemd/system/mariadb.service.
Created symlink /etc/systemd/system/mysqld.service → /usr/lib/systemd/system/mariadb.service.
Created symlink /etc/systemd/system/multi-user.target.wants/mariadb.service → /usr/lib/systemd/system/mariadb.service.
````

````
[quentin@db ~]$ sudo systemctl start mariadb
````

🌞 **Port utilisé par MariaDB**

- vous repérerez le port utilisé par MariaDB avec une commande `ss` exécutée sur `db.tp5.linux`
````
[quentin@db ~]$ sudo ss -altnp | grep mariadb
LISTEN 0      80                 *:3306            *:*    users:(("mariadbd",pid=3799,fd=19))
````
- il sera nécessaire de l'ouvrir dans le firewall
````
[quentin@db ~]$ sudo firewall-cmd --add-port=3306/tcp --permanent
success
````

🌞 **Processus liés à MariaDB**

- repérez les processus lancés lorsque vous lancez le service MariaDB
- utilisz une commande `ps`
````
[quentin@db ~]$ ps -ef | grep mariadb
mysql       3799       1  0 14:28 ?        00:00:00 /usr/libexec/mariadbd --basedir=/usr
quentin     4084     877  0 15:04 pts/0    00:00:00 grep --color=auto mariadb
````

# Partie 3 : Configuration et mise en place de NextCloud

## 1. Base de données

🌞 **Préparation de la base pour NextCloud**

- une fois en place, il va falloir préparer une base de données pour NextCloud :
  - connectez-vous à la base de données à l'aide de la commande `sudo mysql -u root -p`
  - exécutez les commandes SQL suivantes :

````sql
MariaDB [(none)]> CREATE USER 'nextcloud'@'10.105.1.11' IDENTIFIED BY 'pewpewpew';
Query OK, 0 rows affected (0.003 sec)
````
````sql
MariaDB [(none)]> CREATE DATABASE IF NOT EXISTS nextcloud CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;
Query OK, 1 row affected (0.000 sec)
````
````sql
MariaDB [(none)]> GRANT ALL PRIVILEGES ON nextcloud.* TO 'nextcloud'@'10.105.1.11';
Query OK, 0 rows affected (0.003 sec)
````
````sql
MariaDB [(none)]> FLUSH PRIVILEGES;
Query OK, 0 rows affected (0.000 sec)
````

🌞 **Exploration de la base de données**

- afin de tester le bon fonctionnement de la base de données, vous allez essayer de vous connecter
  - depuis la machine `web.tp5.linux` vers l'IP de `db.tp5.linux`
  - utilisez la commande `mysql` pour vous connecter à une base de données depuis la ligne de commande
````
[quentin@web ~]$ mysql -u nextcloud -h 10.105.1.12 -p
Enter password:
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 23
````

    - si vous ne l'avez pas, installez-là
````
[quentin@web ~]$ sudo dnf install mysql-8.0.30-3.el9_0.x86_64
````
- **donc vous devez effectuer une commande `mysql` sur `web.tp5.linux`**
- une fois connecté à la base, utilisez les commandes SQL fournies ci-dessous pour explorer la base

````sql
mysql> SHOW DATABASES;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| nextcloud          |
+--------------------+
2 rows in set (0.00 sec)
````

🌞 **Trouver une commande SQL qui permet de lister tous les utilisateurs de la base de données**

````sql
MariaDB [(none)]> SELECT User FROM mysql.user;
+-------------+
| User        |
+-------------+
| nextcloud   |
| mariadb.sys |
| mysql       |
| root        |
+-------------+
4 rows in set (0.001 sec)
````

## 2. Serveur Web et NextCloud

🌞 **Install de PHP**

````
[quentin@web conf]$ sudo dnf config-manager --set-enabled crb

[quentin@web conf]$ sudo dnf install dnf-utils http://rpms.remirepo.net/enterprise/remi-release-9.rpm -y

[quentin@web conf]$ dnf module list php

[quentin@web conf]$ sudo dnf module enable php:remi-8.1 -y

[quentin@web conf]$ sudo dnf install -y php81-php
```

🌞 **Install de tous les modules PHP nécessaires pour NextCloud**

```
[quentin@web conf]$ sudo dnf install -y libxml2 openssl php81-php php81-php-ctype php81-php-curl php81-php-gd php81-php-iconv php81-php-json php81-php-libxml php81-php-mbstring php81-php-openssl php81-php-posix php81-php-session php81-php-xml php81-php-zip php81-php-zlib php81-php-pdo php81-php-mysqlnd php81-php-intl php81-php-bcmath php81-php-gmp
```

🌞 **Récupérer NextCloud**

- récupérer le fichier suivant avec une commande `curl` ou `wget` ````
[quentin@web www]$ sudo curl https://download.nextcloud.com/server/prereleases/nextcloud-25.0.0rc3.zip -O
````
````
[quentin@web www]$ ls /var/www/tp5_nextcloud/
3rdparty  config       core      index.html  occ           ocs-provider  resources   themes
apps      console.php  cron.php  index.php   ocm-provider  public.php    robots.txt  updater
AUTHORS   COPYING      dist      lib         ocs           remote.php    status.php  version.php
````

- **assurez-vous que le dossier `/var/www/tp5_nextcloud/` et tout son contenu appartient à l'utilisateur qui exécute le service Apache**
  - utilisez une commande `chown` si nécessaire
````
[quentin@web tp5_nextcloud]$ sudo chown apache:apache /var/www/tp5_nextcloud/ -R
````

🌞 **Adapter la configuration d'Apache**

````apache
[quentin@web conf]$ sudo cat httpd.conf | tail -n 18
IncludeOptional conf.d/*.conf

<VirtualHost *:80>
  # on indique le chemin de notre webroot
  DocumentRoot /var/www/tp5_nextcloud/
  # on précise le nom que saisissent les clients pour accéder au service
  ServerName  web.tp5.linux

  # on définit des règles d'accès sur notre webroot
  <Directory /var/www/tp5_nextcloud/>
    Require all granted
    AllowOverride All
    Options FollowSymLinks MultiViews
    <IfModule mod_dav.c>
      Dav off
    </IfModule>
  </Directory>
</VirtualHost>
````

🌞 **Redémarrer le service Apache** pour qu'il prenne en compte le nouveau fichier de conf

````
[quentin@web conf]$ sudo systemctl restart httpd
````

🌞 **Exploration de la base de données**

````sql
mysql> SELECT Count(*) FROM INFORMATION_SCHEMA.TABLES WHERE TABLE_TYPE = 'BASE TABLE';
+----------+
| Count(*) |
+----------+
|       95 |
+----------+
1 row in set (0.00 sec)
````

# Partie 4 : Automatiser la résolution du TP

Script database:
````bash
#!/bin/bash

setenforce 0
echo db.tp5.linux > /etc/hostname
hostname
dnf install mariadb-server -y
sudo systemctl enable mariadb
sudo systemctl start mariadb
sudo mysql_secure_installation
sudo firewall-cmd --add-port=3306/tcp --permanent
sudo firewall-cmd --reload
sudo firewall-cmd --list-all
mysql -u root -p -e "SHOW DATABASES;"
mysql -u root -p -e "CREATE USER 'nextcloud1'@'10.105.1.11' IDENTIFIED BY 'pewpewpew';"
sudo mysql -u root -p -e "CREATE DATABASE IF NOT EXISTS nextcloud CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;"
sudo mysql -u root -p -e "GRANT ALL PRIVILEGES ON nextcloud.* TO 'nextcloud1'@'10.105.1.11';"
sudo mysql -u root -p -e "FLUSH PRIVILEGES"
````

Script webserver:
````bash
#!/bin/bash

setenforce 0
echo web.tp5.linux > /etc/hostname
hostname
dnf install httpd -y
systemctl start httpd
systemctl enable httpd
firewall-cmd --add-port=80/tcp --permanent
firewall-cmd --add-port=3306/tcp --permanent
firewall-cmd --reload
firewall-cmd --list-all
a=$(dnf provides mysql | grep 64 | cut -d' ' -f1) # sert à chercher dans quel paquet est disponible la commande mysql
sudo dnf install ${a}
MYSQL_PASSWORD=$"pewpewpew"
mysql -u nextcloud1 -h 10.105.1.12 -p${MYSQL_PASSWORD} -e "SHOW DATABASES;" # test
mysql -u nextcloud1 -h 10.105.1.12 -p${MYSQL_PASSWORD} -e "USE nextcloud;" -e "SHOW TABLES;" # test
sudo dnf config-manager --set-enabled crb # ajoute le dépôt CRB
dnf module list php
sudo dnf install dnf-utils http://rpms.remirepo.net/enterprise/remi-release-9.rpm -y
sudo dnf module enable php:remi-8.1 -y
sudo dnf install -y php81-php
sudo dnf install -y libxml2 openssl php81-php php81-php-ctype php81-php-curl php81-php-gd php81-php-iconv php81-php-json php81-php-libxml php81-php-mbstring php81-php-openssl php81-php-posix php81-php-session php81-php-xml php81-php-zip php81-php-zlib php81-php-pdo php81-php-mysqlnd php81-php-intl php81-php-bcmath php81-php-gmp
cd /tmp
sudo curl https://download.nextcloud.com/server/prereleases/nextcloud-25.0.0rc3.zip -O
cd /var/www
sudo mkdir tp5_nextcloud
sudo dnf install unzip
cd /tmp
sudo unzip *.zip
sudo mv /tmp/nextcloud /var/www # unzip dans /tmp puis déplacer le dossier dans /var/www
mv /var/www/nextcloud /var/www/tp5_nextcloud
sudo chown apache:apache /var/www/tp5_nextcloud/ -R # faire appartenir le dossier tp5_nextcloud à l'utilisateur apache
ls -al /var/www
mv /tmp/httpd.conf /etc/httpd/conf.d
sudo chown apache:apache /etc/httpd/conf.d/httpd.conf
ls -la /etc/httpd/conf.d/httpd.conf
echo Fin installation Nextcloud
````

Logs de la base de données:
````
[quentin@db ~]$ sudo cat mariadb.log-20221221 | tail -n 2
2022-12-21 12:31:02 0 [Note] /usr/libexec/mariadbd: ready for connections.
Version: '10.5.16-MariaDB'  socket: '/var/lib/mysql/mysql.sock'  port: 3306  MariaDB Server
````