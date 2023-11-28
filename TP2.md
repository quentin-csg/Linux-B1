# TP2 : Appréhender l'environnement Linux

## 1. Analyse du service

On va, dans cette première partie, analyser le service SSH qui est en cours d'exécution.

🌞 **S'assurer que le service `sshd` est démarré**

```
[quentin@TP2 ~]$ systemctl status sshd
● sshd.service - OpenSSH server daemon
     Loaded: loaded (/usr/lib/systemd/system/sshd.service; enabled; vendor preset: enabled)
     Active: active (running) since Tue 2022-11-22 16:13:04 CET; 22min ago
```

🌞 **Analyser les processus liés au service SSH**

```bash
[quentin@TP2 ~]$ ps -ef | grep sshd
root         685       1  0 16:15 ?        00:00:00 sshd: /usr/sbin/sshd -D [listener] 0 of 10-100 startups
root         998     685  0 16:21 ?        00:00:00 sshd: quentin [priv]
quentin     1003     998  0 16:21 ?        00:00:00 sshd: quentin@pts/0
quentin    32881    1004  0 16:38 pts/0    00:00:00 grep --color=auto sshd
```

🌞 **Déterminer le port sur lequel écoute le service SSH**

```
[quentin@TP2 ~]$ ss | grep ssh
tcp   ESTAB  0      52                       10.3.1.50:ssh           10.3.1.1:63122
```

🌞 **Consulter les logs du service SSH**

```
[quentin@TP2 log]$ sudo cat secure | grep sshd | tail -n 10
Nov 22 16:21:20 localhost sshd[998]: Accepted password for quentin from 10.3.1.1 port 63122 ssh2
Nov 22 16:21:20 localhost sshd[998]: pam_unix(sshd:session): session opened for user quentin(uid=1000) by (uid=0)
Nov 22 16:55:47 localhost sshd[1003]: Received disconnect from 10.3.1.1 port 63122:11: disconnected by user
Nov 22 16:55:47 localhost sshd[1003]: Disconnected from user quentin 10.3.1.1 port 63122
Nov 22 16:55:47 localhost sshd[998]: pam_unix(sshd:session): session closed for user quentin
Nov 22 16:55:55 localhost sshd[32933]: Accepted password for quentin from 10.3.1.1 port 63389 ssh2
Nov 22 16:55:55 localhost sshd[32933]: pam_unix(sshd:session): session opened for user quentin(uid=1000) by (uid=0)
Nov 22 17:05:53 localhost sshd[33003]: Accepted password for quentin from 10.3.1.1 port 63487 ssh2
Nov 22 17:05:53 localhost sshd[33003]: pam_unix(sshd:session): session opened for user quentin(uid=1000) by (uid=0)
Nov 22 17:09:37 localhost sudo[33040]: quentin : TTY=pts/1 ; PWD=/etc/ssh ; USER=root ; COMMAND=/bin/cat sshd_config
````
## 2. Modification du service

🌞 **Identifier le fichier de configuration du serveur SSH**

`sshd_config`

🌞 **Modifier le fichier de conf**

````
[quentin@TP2 ssh]$ sudo cat sshd_config | grep Port
Port 8105
````
````
[quentin@localhost ssh]$ sudo firewall-cmd --list-all | grep 8105
ports: 80/tcp 8105/tcp
````

🌞 **Redémarrer le service**

````
[quentin@TP2 ~]$ sudo systemctl restart firewalld
````

🌞 **Effectuer une connexion SSH sur le nouveau port**

PS C:\Users\qcass> ssh quentin@quentin -p 8105   
\
✨ **Bonus : affiner la conf du serveur SSH**  

Pour rendre la connexion en ssh plus secure on a besoin de:

- ne pas laisser le port 22 qui est celui par défaut, et qui va rendre la tâche plus facile à l'attaquant 
  ````
  Port $RANDOM
  ````
- ne pas autoriser les connexions en root sinon il pourrait faire tout ce qu'il veut si l'attaquand parvient à se connecter
  ````
  PermitRootLogin no
  ````
- utiliser une clé ssh plutôt que un mot de passe
  ````
  PasswordAuthentication no
  ````
- et enfin on peut aussi autoriser seulement certaines personnes à pouvoir avoir une connexion à distance
  ````
  AllowUsers username1 etc
  ````

# II. Service HTTP

## 1. Mise en place

🌞 **Installer le serveur NGINX**

````
[quentin@TP2 /]$ sudo dnf install nginx
````
````
[quentin@TP2 /]$ sudo systemctl enable nginx
Created symlink /etc/systemd/system/multi-user.target.wants/nginx.service → /usr/lib/systemd/system/nginx.service.
````

🌞 **Démarrer le service NGINX**

````
[quentin@TP2 /]$ sudo systemctl start nginx
````

🌞 **Déterminer sur quel port tourne NGINX**

````
[quentin@TP2 /]$ cat etc/nginx/nginx.conf | grep listen
        listen       80;
        listen       [::]:80;
````
````
[quentin@TP2 /]$ sudo firewall-cmd --add-port=80/tcp --permanent
success
````

🌞 **Déterminer les processus liés à l'exécution de NGINX**

````
[quentin@TP2 /]$ ps -ef | grep nginx
root        1074       1  0 11:10 ?        00:00:00 nginx: master process /usr/sbin/nginx
nginx       1075    1074  0 11:10 ?        00:00:00 nginx: worker process
quentin     1107     878  0 11:23 pts/0    00:00:00 grep --color=auto nginx
````

🌞 **Euh wait**

````
qcass@LAPTOP-4K0GL7E4 MINGW64 ~
$ curl http://10.3.1.50:80 | head  -n 7
<!doctype html>
<html>
  <head>
    <meta charset='utf-8'>
    <meta name='viewport' content='width=device-width, initial-scale=1'>
    <title>HTTP Server Test Page powered by: Rocky Linux</title>
    <style type="text/css">
````

## 2. Analyser la conf de NGINX

🌞 **Déterminer le path du fichier de configuration de NGINX**

````
[quentin@TP2 /]$ ls -al /etc/nginx/nginx.conf
-rw-r--r--. 1 root root 2334 May 16  2022 /etc/nginx/nginx.conf
````

🌞 **Trouver dans le fichier de conf**

````
[quentin@TP2 nginx]$ cat nginx.conf | grep "server {" -A 16
    server {
        listen       80;
        listen       [::]:80;
        server_name  _;
        root         /usr/share/nginx/html;

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
````
[quentin@TP2 nginx]$ cat nginx.conf | grep include
include /usr/share/nginx/modules/*.conf;
include /etc/nginx/mime.types;
include /etc/nginx/conf.d/*.conf;
include /etc/nginx/default.d/*.conf;
````

## 3. Déployer un nouveau site web

🌞 **Créer un site web**

````
[quentin@TP2 tp2_linux]$ pwd index.html
/var/www/tp2_linux
````
```` html
<!doctype html>
<html lang="fr">
<head>
  <meta charset="utf-8">
  <title>MEOW</title>
  <link rel="stylesheet" href="style.css">
  <script src="script.js"></script>
</head>
<body>
  <h1>MEOW mon premier serveur web</h1>
</body>
</html>
````

🌞 **Adapter la conf NGINX**

````
[quentin@TP2 conf.d]$ cat page.conf
server {
  listen 6716;

  root /var/www/tp2_linux;
}
````
````
[quentin@TP2 nginx]$ sudo rm nginx.conf.default
````
````
[quentin@TP2 nginx]$ sudo systemctl restart nginx
````
````
[quentin@TP2 nginx]$ sudo firewall-cmd --add-port=6716/tcp --permanent
````

🌞 **Visitez votre super site web**

````
qcass@LAPTOP-4K0GL7E4 MINGW64 ~
$ curl http://10.3.1.50:6716
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   158  100   158    0     0  11550      0 --:--:-- --:--:-- --:--:-- 12153<!doctype html>
<html lang="fr">
<head>
  <meta charset="utf-8">
  <title>MEOW</title>
</head>
<body>
  <h1>MEOW mon premier serveur web</h1>
</body>
</html>

````

# III. Your own services

🌞 **Afficher le fichier de service SSH**

````
[quentin@TP2 system]$ cat sshd.service | grep ExecStart=
ExecStart=/usr/sbin/sshd -D $OPTIONS
````

🌞 **Afficher le fichier de service NGINX**

````
[quentin@TP2 system]$ cat nginx.service |grep ExecStart=
ExecStart=/usr/sbin/nginx
````

## 3. Création de service

🌞 **Créez le fichier `/etc/systemd/system/tp2_nc.service`**

````
[quentin@TP2 system]$ cat tp2_nc.service
[Unit]
Description=Super netcat tout fou

[Service]
ExecStart=/usr/bin/nc -l 666
````

🌞 **Indiquer au système qu'on a modifié les fichiers de service**

````
[quentin@TP2 system]$ sudo systemctl daemon-reload
````

🌞 **Démarrer notre service de ouf**

````
[quentin@TP2 system]$ sudo systemctl start tp2_nc
````

🌞 **Vérifier que ça fonctionne**

````
[quentin@TP2 system]$ sudo systemctl status tp2_nc
● tp2_nc.service - Super netcat tout fou
     Loaded: loaded (/etc/systemd/system/tp2_nc.service; static)
     Active: active (running) since Fri 2022-11-25 17:38:32 CET; 1s ago
````

````
[quentin@TP2 ~]$ ss -alnpt | grep 666
LISTEN 0      10           0.0.0.0:666       0.0.0.0:*
LISTEN 0      10              [::]:666          [::]:*
````

🌞 **Les logs de votre service**

````
[quentin@TP2 ~]$ sudo journalctl -xe -u tp2_nc | grep start
 A start job for unit tp2_nc.service has finished successfully.
````
````
[quentin@TP2 ~]$ sudo journalctl -xe -u tp2_nc | grep "SALUT CV"
Nov 26 07:24:53 TP2 nc[2518]: SALUT CV
````

````
[quentin@TP2 ~]$ sudo journalctl -xe -u tp2_nc | grep exit
Nov 26 07:25:41 TP2 systemd[1]: tp2_nc.service: Failed with result 'exit-code'.
````

🌞 **Affiner la définition du service**

````
[quentin@TP2 system]$ sudo systemctl daemon-reload
````
````
Nov 26 09:26:58 TP2 systemd[1]: tp2_nc.service: Failed with result 'exit-code'.
░░ Subject: Unit failed
░░ Defined-By: systemd
░░ Support: https://access.redhat.com/support
░░
░░ The unit tp2_nc.service has entered the 'failed' state with result 'exit-code'.
Nov 26 09:26:58 TP2 systemd[1]: tp2_nc.service: Scheduled restart job, restart counter is at 2.
░░ Subject: Automatic restarting of a unit has been scheduled
░░ Defined-By: systemd
░░ Support: https://access.redhat.com/support
░░
░░ Automatic restarting of the unit tp2_nc.service has been scheduled, as the result for
░░ the configured Restart= setting for the unit
````

