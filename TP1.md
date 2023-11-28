# TP1 : Are you dead yet ?


## II. Feu

🌞 **Trouver au moins 4 façons différentes de péter la machine**

- elles doivent être **vraiment différentes**
- je veux le procédé exact utilisé
  - généralement une commande ou une suite de commandes (script)
- il faut m'expliquer avec des mots comment ça marche
  - pour chaque méthode utilisée, me faut l'explication qui va avec
- tout doit se faire depuis un terminal
- 
## Méthode 1:
````
cd boot/grub2 -> rm grub.cfg  
````
Supprime Grub donc à l'allumage de la vm, le noyau linux ne peux plus se charger avec grub

## Méthode 2:
````
cd bin -> rm dbus-daemon
````
Supprime dbus daemon qui est une librairie qui sert à faire communiquer des applications entre elles en arrière-plan  

## Méthode 3:
````
cat /dev/urandom > /dev/sda
````
Ecrit des nombres aléatoire sur le disque dur donc il ne fonctionne plus au démarrage  

## Méthode 4: 

Aller sur VMbox -> configuration -> affichage et réduire la mémoire vidéo à 0 pour que plus rien ne puissent s'afficher quand on lance la vm.
 
