# Partie 1 : Partitionnement du serveur de stockage

🌞 **Partitionner le disque à l'aide de LVM**
````
[quentin@VMstorage ~]$ sudo pvcreate /dev/sdb
[sudo] password for quentin:
  Physical volume "/dev/sdb" successfully created.
````
````
[quentin@VMstorage ~]$ sudo vgcreate storage /dev/sdb
  Volume group "storage" successfully created
````
````
[quentin@VMstorage ~]$ sudo lvcreate -l 100%FREE storage -n storage1
  Logical volume "storage1" created.
````

🌞 **Formater la partition**
````
[quentin@VMstorage ~]$ sudo mkfs -t ext4 /dev/storage/storage1
mke2fs 1.46.5 (30-Dec-2021)
Creating filesystem with 523264 4k blocks and 130816 inodes
Filesystem UUID: 0378c4d0-73c7-4873-b7fb-9f3bb89bc02a
Superblock backups stored on blocks:
        32768, 98304, 163840, 229376, 294912

Allocating group tables: done
Writing inode tables: done
Creating journal (8192 blocks): done
Writing superblocks and filesystem accounting information: done
````

🌞 **Monter la partition**
````
[quentin@VMstorage ~]$ df -h | grep storage_1
/dev/mapper/storage-storage1  2.0G   24K  1.9G   1% /mnt/storage_1
````

````
[quentin@VMstorage ~]$ sudo cat /dev/mapper/storage-storage1 | head -n 2
��3f������~�c~�c��S���c
                                   <�kx��s�Hs���;���*V���N������9@
                                                                           ��c
�   F:��!�^�8��-�b���� �~��P����L��~��n�    �����
����.��~����   �����
````

````
[quentin@VMstorage ~]$ df -h | grep storage_1
/dev/mapper/storage-storage1  2.0G   24K  1.9G   1% /mnt/storage_1
````

# Partie 2 : Serveur de partage de fichiers

🌞 **Donnez les commandes réalisées sur le serveur NFS `storage.tp4.linux`**
````
[quentin@VMstorage storage_1]$ sudo cat /etc/exports
/mnt/storage_1/site_web_1       10.3.1.53(rm,sync,no_subtree_check)
/mnt/storage_1/site_web_2       10.3.1.53(rm,sync,no_subtree_check)
````

🌞 **Donnez les commandes réalisées sur le client NFS `web.tp4.linux`**
````
[quentin@web site_web_2]$ df -h | grep 10.3.1.52
10.3.1.52:/mnt/storage_1/site_web_1  2.0G     0  1.9G   0% /var/www/site_web_1
10.3.1.52:/mnt/storage_1/site_web_2  2.0G     0  1.9G   0% /var/www/site_web_2
````

````
[quentin@web site_web_2]$ sudo cat /etc/fstab | grep 10.3.1.52
10.3.1.52:/mnt/storage_1/site_web_1    /var/www/site_web_1   nfs defaults 0 0
10.3.1.52:/mnt/storage_1/site_web_2    /var/www/site_web_2   nfs defaults 0 0
````

# Partie 3 : Serveur web

## 2. Install

🌞 **Installez NGINX**

````
[quentin@web ~]$ sudo systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
     Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
     Active: active (running) since Thu 2022-12-08 14:24:19 CET; 4s ago
````

🌞 **Analysez le service NGINX**

- avec une commande `ps`, déterminer sous quel utilisateur tourne le processus du service NGINX
````
[quentin@web ~]$ ps -ef | grep nginx
nginx       1907    1906  0 15:39 ?        00:00:00 nginx: worker process
````

- avec une commande `ss`, déterminer derrière quel port écoute actuellement le serveur web
````
[quentin@web ~]$ sudo ss -tunlp |grep nginx
tcp   LISTEN 0      511          0.0.0.0:80         0.0.0.0:*    users:(("nginx",pid=1257,fd=6),("nginx",pid=1256,fd=6))
tcp   LISTEN 0      511             [::]:80            [::]:*    users:(("nginx",pid=1257,fd=7),("nginx",pid=1256,fd=7))
````

- en regardant la conf, déterminer dans quel dossier se trouve la racine web
````
[quentin@web ~]$ cat /etc/nginx/nginx.conf | grep root
        root         /usr/share/nginx/html;
````

- inspectez les fichiers de la racine web, et vérifier qu'ils sont bien accessibles en lecture par l'utilisateur qui lance le processus
````
[quentin@web html]$ ls -la
total 12
drwxr-xr-x. 3 root root  143 Dec  8 14:16 .
drwxr-xr-x. 4 root root   33 Dec  8 14:16 ..
-rw-r--r--. 1 root root 3332 Oct 31 16:35 404.html
-rw-r--r--. 1 root root 3404 Oct 31 16:35 50x.html
drwxr-xr-x. 2 root root   27 Dec  8 14:16 icons
lrwxrwxrwx. 1 root root   25 Oct 31 16:37 index.html -> ../../testpage/index.html
-rw-r--r--. 1 root root  368 Oct 31 16:35 nginx-logo.png
lrwxrwxrwx. 1 root root   14 Oct 31 16:37 poweredby.png -> nginx-logo.png
lrwxrwxrwx. 1 root root   37 Oct 31 16:37 system_noindex_logo.png -> ../../pixmaps/system-noindex-logo.png
````

## 4. Visite du service web

🌞 **Configurez le firewall pour autoriser le trafic vers le service NGINX**

````
[quentin@web html]$ sudo firewall-cmd --add-port=80/tcp --permanent
success
````
````
[quentin@web html]$ sudo firewall-cmd --zone=public --add-service=https --permanent
success
````

🌞 **Accéder au site web**

````
[quentin@web html]$ curl http://10.3.1.53 | head -n 7
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0<!doctype html>
<html>
  <head>
    <meta charset='utf-8'>
    <meta name='viewport' content='width=device-width, initial-scale=1'>
    <title>HTTP Server Test Page powered by: Rocky Linux</title>
    <style type="text/css">
````

🌞 **Vérifier les logs d'accès**

````
[quentin@web /]$ sudo cat /var/log/nginx/access.log | tail -n 3
10.3.1.53 - - [08/Dec/2022:14:53:17 +0100] "GET / HTTP/1.1" 200 7620 "-" "curl/7.76.1" "-"
10.3.1.53 - - [08/Dec/2022:14:53:42 +0100] "GET / HTTP/1.1" 200 7620 "-" "curl/7.76.1" "-"
10.3.1.53 - - [08/Dec/2022:14:54:28 +0100] "GET / HTTP/1.1" 200 7620 "-" "curl/7.76.1" "-"
````

## 5. Modif de la conf du serveur web

🌞 **Changer le port d'écoute**

- une simple ligne à modifier, vous me la montrerez dans le compte rendu
````
[quentin@web /]$ sudo cat /etc/nginx/nginx.conf | grep 8080
        listen       8080;
````

- prouvez-moi que le changement a pris effet avec une commande ss
````
[quentin@web /]$ sudo ss -tulnp | grep 8080
tcp   LISTEN 0      511          0.0.0.0:8080       0.0.0.0:*    users:(("nginx",pid=1486,fd=6),("nginx",pid=1485,fd=6))
````

- n'oubliez pas de fermer l'ancien port dans le firewall, et d'ouvrir le nouveau
````
[quentin@web home]$ sudo firewall-cmd --list-all | grep port
  ports: 22/tcp 8080/tcp
````

- prouvez avec une commande curl sur votre machine que vous pouvez désormais visiter le port 8080
````
[quentin@web /]$ curl http://10.3.1.53:8080 | head -n 7
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0<!doctype html>
<html>
  <head>
    <meta charset='utf-8'>
    <meta name='viewport' content='width=device-width, initial-scale=1'>
    <title>HTTP Server Test Page powered by: Rocky Linux</title>
    <style type="text/css">
100  7620  100  7620    0     0  1240k      0 --:--:-- --:--:-- --:--:-- 1240k
curl: (23) Failed writing body
````

---

🌞 **Changer l'utilisateur qui lance le service**

- pour ça, vous créerez vous-même un nouvel utilisateur sur le système : web
````
[root@web web]# useradd web -m -s /bin/sh
useradd: user 'web' already exists
````

- modifiez la conf de NGINX pour qu'il soit lancé avec votre nouvel utilisateur
````
[quentin@web home]$ sudo cat /etc/nginx/nginx.conf | grep web
user web;
````

- vous prouverez avec une commande ps que le service tourne bien sous ce nouveau utilisateur
````
[quentin@web home]$ ps -ef | grep nginx
root        1936       1  0 15:41 ?        00:00:00 nginx: master process /usr/sbin/nginx
web         1937    1936  0 15:41 ?        00:00:00 nginx: worker process
````

---

**Il est temps d'utiliser ce qu'on a fait à la partie 2 !**

🌞 **Changer l'emplacement de la racine Web**

- configurez NGINX pour qu'il utilise une autre racine web que celle par défaut
````
[quentin@web www]$ cat /var/www/site_web_1/index.html
<h1> Meow </h1>
````

- vous me montrerez la conf effectuée dans le compte-rendu, avec un grep
````
[quentin@web www]$ sudo cat /etc/nginx/nginx.conf | grep root
        root         /var/www/site_web_1/;
````

- prouvez avec un `curl` depuis votre hôte que vous accédez bien au nouveau site
````
[quentin@web www]$ curl http://10.3.1.53:8080
<h1> Meow </h1>
````

## 6. Deux sites web sur un seul serveur

🌞 **Repérez dans le fichier de conf**

- la ligne qui inclut des fichiers additionels contenus dans un dossier nommé `conf.d`
````
[quentin@web ~]$ sudo cat /etc/nginx/nginx.conf | grep conf.d
    # Load modular configuration files from the /etc/nginx/conf.d directory.
    include /etc/nginx/conf.d/*.conf;
````

🌞 **Créez le fichier de configuration pour le premier site**

````
[quentin@web conf.d]$ cat site_web_1.conf
server {
    listen       8080;
    listen       [::]:80;
    server_name  _;
    root         /var/www/site_web_1/;

    # Load configuration files for the default server block.
    include /etc/nginx/default.d/*.conf;

    error_page 404 /404.html;
    location = /404.html {
    }

    error_page 500 502 503 504 /50x.html;
    location = /50x.html {
    }
}
````

🌞 **Créez le fichier de configuration pour le deuxième site**

````
[quentin@web conf.d]$ cat site_web_2.conf
server {
    listen       8888;
    listen       [::]:80;
    server_name  _;
    root         /var/www/site_web_2/;

    # Load configuration files for the default server block.
    include /etc/nginx/default.d/*.conf;

    error_page 404 /404.html;
    location = /404.html {
    }

    error_page 500 502 503 504 /50x.html;
    location = /50x.html {
    }
}
````

🌞 **Prouvez que les deux sites sont disponibles**

````
PS C:\Users\qcass> curl http://10.3.1.53:8080


StatusCode        : 200
StatusDescription : OK
Content           : <h1> Meow </h1>

RawContent        : HTTP/1.1 200 OK
                    Connection: keep-alive
                    Accept-Ranges: bytes
                    Content-Length: 16
                    Content-Type: text/html
                    Date: Thu, 08 Dec 2022 15:54:49 GMT
                    ETag: "6391f9e3-10"
                    Last-Modified: Thu, 08 Dec 2022 14...
Forms             : {}
Headers           : {[Connection, keep-alive], [Accept-Ranges, bytes], [Content-Length, 16], [Content-Type,
                    text/html]...}
Images            : {}
InputFields       : {}
Links             : {}
ParsedHtml        : mshtml.HTMLDocumentClass
RawContentLength  : 16
````

````
PS C:\Users\qcass> curl http://10.3.1.53:8888


StatusCode        : 200
StatusDescription : OK
Content           : <h1> Meow site_web_2 <h/2>

RawContent        : HTTP/1.1 200 OK
                    Connection: keep-alive
                    Accept-Ranges: bytes
                    Content-Length: 27
                    Content-Type: text/html
                    Date: Thu, 08 Dec 2022 15:55:13 GMT
                    ETag: "63920833-1b"
                    Last-Modified: Thu, 08 Dec 2022 15...
Forms             : {}
Headers           : {[Connection, keep-alive], [Accept-Ranges, bytes], [Content-Length, 27], [Content-Type,
                    text/html]...}
Images            : {}
InputFields       : {}
Links             : {}
ParsedHtml        : mshtml.HTMLDocumentClass
RawContentLength  : 27
````

