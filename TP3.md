# TP 3 : We do a little scripting

# I. Script carte d'identité

## Rendu

Script

````bash
#!/bin/bash

echo Machine name : $(hostnamectl --static)
echo OS $(uname) and kernel version is $(cat /etc/redhat-release | cut -d'(' -f1)
echo IP : $(ip a | grep enp0s* | grep inet | tail -1 |  cut -d' ' -f6 | cut -d'/' -f1)
echo RAM : $(free -h --mega | grep Mem: | tr -s ' ' | cut -d' ' -f7) memory available on $(free -h --mega | grep Mem: | tr -s ' ' | cut -d' ' -f2) total memory
echo Disque : $(df -h | tr -s ' ' | grep '/$' | cut -d' ' -f2) space left
echo Top 5 processes by RAM usage :

p="$(ps -o command= -e --sort -%mem | head -n5)"
while read line ; do echo "  - $line" ; done <<< "${p}"

echo Listening ports :
g="$(ss -lnp4H)"
while read line ;
do
  port_type=$(cut -d' ' -f1 <<< "${line}")
  port_number=$(echo $line | cut -d' ' -f5 | cut -d':' -f2)
  process=$(echo $line |tr -s ' ' | cut -d'"' -f2)
  echo "  - ${port_number} ${port_type} ":" ${process}"
done <<< "${g}"

cat_pic=$(curl -s https://cataas.com/cat > potichat)
ext=$(file --extension potichat | cut -d' ' -f2 | cut -d'/' -f1)
if [[ ${ext} == "jpeg" ]]
then
  cat_ext="cat.${ext}"
elif [[ ${ext} == "png" ]]
then
  cat_ext="cat.${ext}"
else
  cat_ext="cat.gif"
fi
mv potichat ${cat_ext}
chmod +x ${cat_ext}
echo " "
echo "Here is your random cat : ./${cat_ext}"
````

Exemple d'utilisation

````
[quentin@TP3 idcard]$ sudo ./idcard.sh
Machine name : TP3
OS Linux and kernel version is Rocky Linux release 9.0
IP : 10.3.1.51
RAM : 639M memory available on 960M total memory
Disque : 6.2G space left
Top 5 processes by RAM usage :
  - /usr/bin/python3 -s /usr/sbin/firewalld --nofork --nopid
  - /usr/sbin/NetworkManager --no-daemon
  - /usr/lib/systemd/systemd --switched-root --system --deserialize 27
  - /usr/lib/systemd/systemd --user
  - sshd: quentin [priv]
Listening ports :
  - 323 udp : chronyd
  - 22 tcp : sshd

Here is your random cat : ./cat.gif
````

# II. Script youtube-dl

## Rendu

Script
````bash
#!/bin/bash

video_title="$(youtube-dl -e $1)"
cd /srv/yt/downloads/
mkdir "${video_title}" > /dev/null
cd "${video_title}"
youtube-dl $1 > /dev/null
echo "Vidéo" $1 "was downloaded."
nom_vid="$(ls *.mp4)"
youtube-dl --get-description $1 > description
echo "File path : /srv/yt/downloads/""${video_title}""/""${nom_vid}"
cd /var/log/yt
echo "[""$(date "+%Y/%m/%d"" ""%T")""]" >> /var/log/yt/download.log
echo "Vidéo" $1 "was downloaded. File path : /srv/yt/downloads/""${video_title}""/""${nom_vid}" >> /var/log/yt/download.log
````

Fichier des logs
````
[quentin@TP3 yt]$ cat download.log
[2022/12/04 14:31:14]
Vidéo https://www.youtube.com/watch?v=Wch3gJG2GJ4 was downloaded. File path : /srv/yt/downloads/1 Second Video/1 Second Video-Wch3gJG2GJ4.mp4
[2022/12/04 14:31:48]
Vidéo https://www.youtube.com/watch?v=jjs27jXL0Zs was downloaded. File path : /srv/yt/downloads/SI LA VIDÉO DURE 1 SECONDE LA VIDÉO S'ARRÊTE/SI LA VIDÉO DURE 1 SECONDE LA VIDÉO S'ARRÊTE-jjs27jXL0Zs.mp4
````

Exemple d'utilisation

````
[root@TP3 yt]# ./yt.sh https://www.youtube.com/watch?v=Wch3gJG2GJ4
Vidéo https://www.youtube.com/watch?v=Wch3gJG2GJ4 was downloaded.
File path : /srv/yt/downloads/1 Second Video/1 Second Video-Wch3gJG2GJ4.mp4
````

# III. MAKE IT A SERVICE

## Rendu

Main script
````bash
[yt@TP3 yt]$ cat yt-v2.sh
#!/bin/bash

while true ; do
  vue="$(cat /srv/yt/url_vid.txt)"
  while read line ;
  do
    vue="$(head -1 /srv/yt/url_vid.txt)"
    video_title="$(youtube-dl -e ""${vue}"" 2> /dev/null )"
    cd /srv/yt
    vue="$(cat /srv/yt/url_vid.txt)"
    if [[ "$vue" == "https://www.youtube.com/watch?v="* ]]; then
      cd /srv/yt/downloads/
      mkdir "${video_title}" 2> /dev/null
      cd "${video_title}"
      youtube-dl "${vue}" > /dev/null
      echo "Vidéo" "${vue}" "was downloaded."
      nom_vid="$(ls *.mp4)"
      youtube-dl --get-description "${vue}" > description
      echo "File path : /srv/yt/downloads/""${video_title}""/""${nom_vid}"
      cd /var/log/yt
      echo "[""$(date "+%Y/%m/%d"" ""%T")""]" >> /var/log/yt/download.log
      echo "Vidéo" "${vue}" "was downloaded. File path : /srv/yt/downloads/""${video_title}""/""${nom_vid}" >> /var/log/yt/download.log
      sed -i '1d' /srv/yt/url_vid.txt
  fi
  done <<< "${vue}"
  sleep 20
done
```` 


Script yt.service
````
[yt@TP3 system]$ cat yt.service
[Unit]
Description=Test

[Service]
ExecStart=/srv/yt/yt-v2.sh
User=yt

[Install]
WantedBy=multi-user.target
````

Le statut du service
````
[yt@TP3 system]$ sudo systemctl status yt
● yt.service - Test
     Loaded: loaded (/etc/systemd/system/yt.service; disabled; vendor preset: disabled)
     Active: active (running) since Mon 2022-12-05 00:01:30 CET; 3min 28s ago
   Main PID: 3408 (yt-v2.sh)
````

Extrait du Journalctl

````s
[yt@TP3 system]$ sudo journalctl -xe -u yt -f
Dec 05 00:01:30 TP3 systemd[1]: Started Test.
░░ Subject: A start job for unit yt.service has finished successfully
░░ Defined-By: systemd
░░ Support: https://access.redhat.com/support
░░
░░ A start job for unit yt.service has finished successfully.
░░
░░ The job identifier is 2763.
Dec 05 00:04:03 TP3 yt-v2.sh[3408]: Vidéo https://www.youtube.com/watch?v=jjs27jXL0Zs was downloaded.
Dec 05 00:04:05 TP3 yt-v2.sh[3408]: File path : /srv/yt/downloads/SI LA VIDÉO DURE 1 SECONDE LA VIDÉO S'ARRÊTE/SI LA VIDÉO DURE 1 SECONDE LA VIDÉO S'ARRÊTE-jjs27jXL0Zs.mp4
````