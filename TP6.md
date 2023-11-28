# TP6 : Travail autour de la solution NextCloud

# Module 1 : Reverse Proxy

# I. Setup

🌞 **On utilisera NGINX comme reverse proxy**

````
[quentin@proxy ~]$ sudo systemctl status nginx
````

- utiliser la commande `ss` pour repérer le port sur lequel NGINX écoute
````
[quentin@proxy ~]$ sudo ss -altpn | grep nginx
LISTEN 0      511          0.0.0.0:80        0.0.0.0:*    users:(("nginx",pid=1446,fd=6),("nginx",pid=1445,fd=6))
LISTEN 0      511             [::]:80           [::]:*    users:(("nginx",pid=1446,fd=7),("nginx",pid=1445,fd=7))
````

- ouvrir un port dans le firewall pour autoriser le trafic vers NGINX
````
[quentin@proxy ~]$ sudo firewall-cmd --add-port=80/tcp --permanent
success
````

- utiliser une commande `ps -ef` pour déterminer sous quel utilisateur tourne NGINX
````
[quentin@proxy ~]$ ps -ef | grep nginx
nginx       1446    1445  0 16:13 ?        00:00:00 nginx: worker process
````
- vérifier que le page d'accueil NGINX est disponible en faisant une requête HTTP sur le port 80 de la machine

🌞 **Configurer NGINX**

- nous ce qu'on veut, c'pas une page d'accueil moche, c'est que NGINX agisse comme un reverse proxy entre les clients et notre serveur Web
- deux choses à faire :
  - créer un fichier de configuration NGINX
    - la conf est dans `/etc/nginx`
````
quentin@proxy conf.d]$ cat nginx.conf | grep proxy_pass
        proxy_pass http://10.105.1.12:80;
````

  - NextCloud est un peu exigeant, et il demande à être informé si on le met derrière un reverse proxy
    - y'a donc un fichier de conf NextCloud à modifier
    - c'est un fichier appelé `config.php`
````
[quentin@localhost config]$ sudo cat config.php | grep trusted
  'trusted_domains' => 'proxy.tp6.linux'
````

🌞 **Faites en sorte de**

- rendre le serveur `web.tp6.linux` injoignable
- sauf depuis l'IP du reverse proxy
````
[quentin@web ~]$ sudo firewall-cmd --zone=public --add-rich-rule='rule family="ipv4" source address="10.105.1.13/32" invert="True" drop' --permanent
[sudo] password for quentin:
success
````

🌞 **Une fois que c'est en place**

- faire un `ping` manuel vers l'IP de `proxy.tp6.linux` fonctionne
````
PS C:\Users\qcass> ping 10.105.1.13

Envoi d’une requête 'Ping'  10.105.1.13 avec 32 octets de données :
Réponse de 10.105.1.13 : octets=32 temps<1ms TTL=64
Réponse de 10.105.1.13 : octets=32 temps<1ms TTL=64
Réponse de 10.105.1.13 : octets=32 temps<1ms TTL=64

Statistiques Ping pour 10.105.1.13:
    Paquets : envoyés = 3, reçus = 3, perdus = 0 (perte 0%),
Durée approximative des boucles en millisecondes :
    Minimum = 0ms, Maximum = 0ms, Moyenne = 0ms
````

- faire un `ping` manuel vers l'IP de `web.tp6.linux` ne fonctionne pas
````
PS C:\Users\qcass> ping 10.105.1.11

Envoi d’une requête 'Ping'  10.105.1.11 avec 32 octets de données :
Délai d’attente de la demande dépassé.
Délai d’attente de la demande dépassé.
Délai d’attente de la demande dépassé.
Délai d’attente de la demande dépassé.

Statistiques Ping pour 10.105.1.11:
    Paquets : envoyés = 4, reçus = 0, perdus = 4 (perte 100%),
````

# II. HTTPS

🌞 **Faire en sorte que NGINX force la connexion en HTTPS plutôt qu'HTTP**
````
[quentin@proxy nginx]$ sed '41,42!d' nginx.conf
        server_name  _;
        return 301   https://$host$request_uri;
````

````
qcass@LAPTOP-4K0GL7E4 MINGW64 ~
$ curl -iL http://web.tp6.linux
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   169  100   169    0     0  38269      0 --:--:-- --:--:-- --:--:-- 56333
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0HTTP/1.1 301 Moved Permanently
Server: nginx/1.20.1
Date: Fri, 13 Jan 2023 10:12:48 GMT
Content-Type: text/html
Content-Length: 169
Connection: keep-alive
Location: https://web.tp6.linux/


curl: (60) SSL certificate problem: self signed certificate
More details here: https://curl.se/docs/sslcerts.html

curl failed to verify the legitimacy of the server and therefore could not
establish a secure connection to it. To learn more about this situation and
how to fix it, please visit the web page mentioned above.
````


# Module 2 : Sauvegarde du système de fichiers

## I. Script de backup

### 1. Ecriture du script

🌞 **Ecrire le script `bash`**

- il s'appellera `tp6_backup.sh`
````bash
#!/bin/bash
# Le script permet de faire une backup des fichiers impotants de nextcloud si celui-ci est installé, et les sauvegardera dans le dossier /srv/backup

cd /srv
mkdir backup
cd /backup
heure="$(date +%H%M)"
jour="$(date +%y%m%d)"
zip -r "nextcloud_"${jour}""${heure}"" /home/quentin/nextcloud/core/Db /home/quentin/nextcloud/core/Data /home/quentin/nextcloud/config /home/quentin/nextcloud/themes
cd /srv
mv nextcloud* /srv/backup
````

### 3. Service et timer

🌞 **Créez un *service*** système qui lance le script

````bash
[Unit]
Description=BAckup de nextcloud

[Service]
ExecStart=/srv/tp6_nextcloud.sh
Type=oneshot
````

- assurez-vous qu'il fonctionne en utilisant des commandes systemctl
````
[quentin@web system]$ sudo systemctl status backup | grep Jan
Jan 14 12:32:37 web.tp5.linux systemd[1]: Starting BAckup de nextcloud...
Jan 14 12:32:38 web.tp5.linux systemd[1]: backup.service: Deactivated successfully.
Jan 14 12:32:38 web.tp5.linux systemd[1]: Finished BAckup de nextcloud.
````

🌞 **Créez un *timer*** système qui lance le *service* à intervalles réguliers

````
[quentin@web system]$ cat backup.timer
[Unit]
Description=Run service backup.service

[Timer]
OnCalendar=*-*-* 4:00:00

[Install]
WantedBy=timers.target
````

🌞 Activez l'utilisation du *timer*
````
[quentin@web system]$ sudo systemctl list-timers | grep backup
Sun 2023-01-15 04:00:00 CET 15h left  n/a         n/a          backup.timer        backup.service
````


## II. NFS

### 1. Serveur NFS

🌞 **Installer le serveur NFS** (sur la machine `storage.tp6.linux`)

- installer le paquet `nfs-utils`
````
[quentin@storage /]$ sudo dnf install nfs-utils
````

- créer le fichier `/etc/exports`
````
[quentin@storage /]$ cat /etc/exports
/srv/nfs_shares/web.tp6.linux       10.105.1.11(rw,sync,no_root_squash,no_subtree_check)
````

- ouvrir les ports firewall nécessaires
````
[quentin@storage etc]$ sudo firewall-cmd --permanent --list-all | grep services
  services: cockpit dhcpv6-client mountd nfs rpc-bind ssh
````

- démarrer le service
````
[quentin@storage /]$ sudo systemctl enable nfs-server
success
[quentin@storage /]$ sudo systemctl start nfs-server
success
````


### 2. Client NFS

🌞 **Installer un client NFS sur `web.tp6.linux`**

- il devra monter le dossier `/srv/nfs_shares/web.tp6.linux/` qui se trouve sur `storage.tp6.linux`
- le dossier devra être monté sur `/srv/backup/`
````
[quentin@web /]$ sudo mount 10.105.1.15:/srv/nfs_shares/web.tp6.linux /srv/backup/
````

````
[quentin@web /]$ df -h | grep /srv/backup
10.105.1.15:/srv/nfs_shares/web.tp6.linux  6.2G  1.3G  4.9G  21%  /srv/backup 
````

🌞 **Tester la restauration des données** sinon ça sert à rien :)

- livrez-moi la suite de commande que vous utiliseriez pour restaurer les données dans une version antérieure
````
[quentin@storage web.tp6.linux]$ sudo unzip nextcloud_2301162145.zip
````


# Module 3 : Fail2Ban

🌞 Faites en sorte que :

- si quelqu'un se plante 3 fois de password pour une co SSH en moins de 1 minute, il est ban
````
[quentin@db fail2ban]$ grep -e "maxretry = 3" -e "findtime = 1" jail.local
maxretry = 3
findtime = 1m
````

- vérifiez que ça fonctionne en vous faisant ban
````
[quentin@db fail2ban]$ sudo fail2ban-client status
Status
|- Number of jail:      1
`- Jail list:   sshd
````

- utilisez une commande dédiée pour lister les IPs qui sont actuellement ban
````
[quentin@db fail2ban]$ sudo fail2ban-client status sshd | grep -i ip
   `- Banned IP list:   10.105.1.11
````

- afficher l'état du firewall, et trouver la ligne qui ban l'IP en question
````
[quentin@db action.d]$ cat firewallcmd-allports.conf | grep actionban
actionban = firewall-cmd --direct --add-rule <family> filter f2b-<name> 0 -s <ip> -j <blocktype>
````
- lever le ban avec une commande liée à fail2ban
````
[quentin@db fail2ban]$ sudo fail2ban-client set sshd unbanip 10.105.1.11
0
````

# Module 4 : Monitoring

🌞 **Installer Netdata**

````
[quentin@db ~]$ sudo systemctl status netdata | grep PID
   Main PID: 13285 (netdata)
````

````
[quentin@db ~]$ sudo ss -altnp | grep 13285
LISTEN 0      4096       127.0.0.1:8125       0.0.0.0:*    users:(("netdata",pid=13285,fd=54))
LISTEN 0      4096         0.0.0.0:19999      0.0.0.0:*    users:(("netdata",pid=13285,fd=6))
````
````
[quentin@db ~]$ sudo firewall-cmd --add-port=19999/tcp --permanent
success
````

🌞 **Une fois Netdata installé et fonctionnel, déterminer :**

- l'utilisateur sous lequel tourne le(s) processus Netdata
- si Netdata écoute sur des ports
````
tcp      LISTEN    0         4096                                                                     [::1]:8125                               [::]:*        users:(("netdata",pid=13285,fd=47))
````

````
[quentin@db ~]$ sudo ss -alp | grep netdata
u_str LISTEN 0      4096                                                   /tmp/netdata-ipc 37987                         * 0    users:(("netdata",pid=13285,fd=53))
udp   UNCONN 0      0                                                             127.0.0.1:8125                    0.0.0.0:*    users:(("netdata",pid=13285,fd=44))
udp   UNCONN 0      0                                                                 [::1]:8125                       [::]:*    users:(("netdata",pid=13285,fd=39))
````

- comment sont consultables les logs de Netdata
````
[quentin@db netdata]$ cat /var/log/netdata/access.log | head -n 2
2023-01-13 12:15:13: 1: 13447 '[10.105.1.1]:1058' 'CONNECTED'
2023-01-13 12:15:13: 2: 13447 '[10.105.1.1]:1059' 'CONNECTED'
````

🌞 **Configurer Netdata pour qu'il vous envoie des alertes** 

````
[quentin@db ~]$ cat /etc/netdata/health_alarm_notify.conf
###############################################################################

# enable/disable sending discord notifications
SEND_DISCORD="YES"

# Create a webhook by following the official documentation -
# https://support.discordapp.com/hc/en-us/articles/228383668-Intro-to-Webhooks
DISCORD_WEBHOOK_URL="https://discord.com/api/webhooks/1063442238765539401/-LqWUr77GYSKhUJzClfJQxf7YKt_65bwezjg0eCsfWzduylkEOc1vl-qgyfWNu-YvLC6"

# if a role's recipients are not configured, a notification will be send to
# this discord channel (empty = do not send a notification for unconfigured
# roles):
DEFAULT_RECIPIENT_DISCORD="général"
````

🌞 **Vérifier que les alertes fonctionnent**

- en surchargeant volontairement la machine 
````
[quentin@db ~]$ cat prog.py
a=2
for i in range (1,10):
        a=a**a
        print(a)
````
````
[quentin@db ~]$ python3 prog.py
4
256
32317006071311007300714876688669951960444102669715484032130345427524655138867890893197201411522913463688717960921898019494119559150490921095088152386448283120630877367300996091750197750389652106796057638384067568276792218642619756161838094338476170470581645852036305042887575891541065808607552399123930385521914333389668342420684974786564569494856176035326322058077805659331026192708460314150258592864177116725943603718461857357598351152301645904403697613233287231227125684710820209725157101726931323469678542580656697935045997268352998638215525166389437335543602135433229604645318478604952148193555853611059596230656
Killed
````

````[quentin@db netdata]$ cat health.log | tail -n 7
2023-01-13 14:17:44: [db.tp5.linux]: Alert event for [mem.available.ram_available], value [2.06%], status [CRITICAL].
2023-01-13 14:17:44: [db.tp5.linux]: Sending notification for alarm 'mem.available.ram_available' status CRITICAL.
2023-01-13 14:18:03: [db.tp5.linux]: Alert event for [mem.available.ram_available], value [64.1%], status [CLEAR].
2023-01-13 14:19:33: [db.tp5.linux]: Alert event for [system.swapio.30min_ram_swapped_out], value [190.4% of RAM], status [WARNING].
2023-01-13 14:19:33: [db.tp5.linux]: Sending notification for alarm 'system.swapio.30min_ram_swapped_out' status WARNING.
2023-01-13 14:20:33: [db.tp5.linux]: Alert event for [mem.oom_kill.oom_kill], value [1 kills], status [WARNING].
2023-01-13 14:20:33: [db.tp5.linux]: Sending notification for alarm 'mem.oom_kill.oom_kill' status WARNING.
````