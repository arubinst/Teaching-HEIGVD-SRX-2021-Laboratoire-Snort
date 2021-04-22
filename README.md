Pellissier David et Ruckstuhl Michael

**Remarque** : nous avons dû changer le réseau Docker en 192.168.2.0 parce que le réseau 192.168.1.0 était déjà utilisé par le réseau d'un d'entre nous, causant des problèmes pour accéder à Internet.

# Teaching-HEIGVD-SRX-2021-Laboratoire-Snort

**Ce travail de laboratoire est à faire en équipes de 2 personnes**

**ATTENTION : Commencez par créer un Fork de ce repo et travaillez sur votre fork.**

Clonez le repo sur votre machine. Vous pouvez répondre aux questions en modifiant directement votre clone du README.md ou avec un fichier pdf que vous pourrez uploader sur votre fork.

**Le rendu consiste simplement à répondre à toutes les questions clairement identifiées dans le text avec la mention "Question" et à les accompagner avec des captures. Le rendu doit se faire par une "pull request". Envoyer également le hash du dernier commit et votre username GitHub par email au professeur et à l'assistant**

## Table de matières

[Introduction](#introduction)

[Echéance](#echéance)

[Démarrage de l'environnement virtuel](#démarrage-de-lenvironnement-virtuel)

[Communication avec les conteneurs](#communication-avec-les-conteneurs)

[Configuration de la machine IDS et installation de Snort](#configuration-de-la-machine-ids-et-installation-de-snort)

[Essayer Snort](#essayer-snort)

[Utilisation comme IDS](#utilisation-comme-un-ids)

[Ecriture de règles](#ecriture-de-règles)

[Travail à effectuer](#exercises)

[Cleanup](#cleanup)


## Echéance 

Ce travail devra être rendu au plus tard, **le 29 avril 2021 à 23h59.**


## Introduction

Dans ce travail de laboratoire, vous allez explorer un système de détection contre les intrusions (IDS) dont l'utilisation es très répandue grâce au fait qu'il est gratuit et open source. Il s'appelle [Snort](https://www.snort.org). Il existe des versions de Snort pour Linux et pour Windows.

### Les systèmes de détection d'intrusion

Un IDS peut "écouter" tout le traffic de la partie du réseau où il est installé. Sur la base d'une liste de règles, il déclenche des actions sur des paquets qui correspondent à la description de la règle.

Un exemple de règle pourrait être, en langage commun : "donner une alerte pour tous les paquets envoyés par le port http à un serveur web dans le réseau, qui contiennent le string 'cmd.exe'". En on peut trouver des règles très similaires dans les règles par défaut de Snort. Elles permettent de détecter, par exemple, si un attaquant essaie d'éxecuter un shell de commandes sur un serveur Web tournant sur Windows. On verra plus tard à quoi ressemblent ces règles.

Snort est un IDS très puissant. Il est gratuit pour l'utilisation personnelle et en entreprise, où il est très utilisé aussi pour la simple raison qu'il est l'un des systèmes IDS des plus efficaces.

Snort peut être exécuté comme un logiciel indépendant sur une machine ou comme un service qui tourne après chaque démarrage. Si vous voulez qu'il protège votre réseau, fonctionnant comme un IPS, il faudra l'installer "in-line" avec votre connexion Internet. 

Par exemple, pour une petite entreprise avec un accès Internet avec un modem simple et un switch interconnectant une dizaine d'ordinateurs de bureau, il faudra utiliser une nouvelle machine éxecutant Snort et placée entre le modem et le switch. 


## Matériel

Vous avez besoin de votre ordinateur avec Docker et docker-compose. Vous trouverez tous les fichiers nécessaires pour générer l'environnement pour virtualiser ce labo dans le projet que vous avez cloné.


## Démarrage de l'environnement virtuel

Ce laboratoire utilise docker-compose, un outil pour la gestion d'applications utilisant multiples conteneurs. Il va se charger de créer un réseaux virtuel `lan`, la machine IDS et une machine "Client". Le réseau LAN interconnecte les deux machines (voir schéma ci-dessous).

![Plan d'adressage](images/docker-snort.png)

Nous allons commencer par lancer docker-compose. Il suffit de taper la commande suivante dans le répertoire racine du labo, celui qui contient le fichier [docker-compose.yml](docker-compose.yml). Optionnelement vous pouvez lancer le script [up.sh](scripts/up.sh) qui se trouve dans le répertoire [scripts](scripts), ainsi que d'autres scripts utiles pour vous :

```bash
docker-compose up --detach
```

Le téléchargement et génération des images prend peu de temps. 

Les images utilisées pour les conteneurs sont basées sur l'image officielle Kali. Le fichier [Dockerfile](Dockerfile) que vous avez téléchargé contient les informations nécessaires pour la génération de l'image de base. [docker-compose.yml](docker-compose.yml) l'utilise comme un modèle pour générer les conteneurs. Vous pouvez vérifier que les deux conteneurs sont crées et qu'ils fonctionnent à l'aide de la commande suivante.

```bash
docker ps
```

## Communication avec les conteneurs

Afin de simplifier vos manipulations, les conteneurs ont été configurées avec les noms suivants :

- IDS
- Client

Pour accéder au terminal de l’une des machines, il suffit de taper :

```bash
docker exec -it <nom_de_la_machine> /bin/bash
```

Par exemple, pour ouvrir un terminal sur votre IDS :

```bash
docker exec -it IDS /bin/bash
```

Optionnelement, vous pouvez utiliser les scripts [openids.sh](scripts/openids.sh) et [openclient.sh](scripts/openclient.sh) pour contacter les conteneurs.

Vous pouvez bien évidemment lancer des terminaux avec les deux machines en même temps ou même lancer plusieurs terminaux sur la même machine. ***Il est en fait conseillé pour ce laboratoire de garder au moins deux terminaux ouverts sur la machine IDS en tout moment***.


### Configuration de la machine Client

Dans un terminal de votre machine Client, taper les commandes suivantes :

```bash
ip route del default 
ip route add default via 192.168.1.2
```

Ceci configure la machine IDS comme la passerelle par défaut pour la machine Client.


## Configuration de la machine IDS et installation de Snort

Pour permettre à votre machine Client de contacter l'Internet à travers la machine IDS, il faut juste une petite règle NAT :

```bash
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
```

Cette commande `iptables` définit une règle dans le tableau NAT qui permet la redirection de ports et donc, l'accès à l'Internet pour la machine Client.

On va maintenant installer Snort sur le conteneur IDS.

La manière la plus simple c'est d'installer Snort en ligne de commandes. Il suffit d'utiliser la commande suivante :

```
apt install snort
```

Ceci télécharge et installe la version la plus récente de Snort.

Il est possible que vers la fin de l'installation, on vous demande de fournir deux informations :

- Le nom de l'interface sur laquelle snort doit surveiller - il faudra répondre ```eth0```
- L'adresse de votre réseau HOME. Il s'agit du réseau que vous voulez protéger. Cela sert à configurer certaines variables pour Snort. Vous pouvez répondre ```192.168.1.0/24```.


## Essayer Snort

Une fois installé, vous pouvez lancer Snort comme un simple "sniffer". Pourtant, ceci capture tous les paquets, ce qui peut produire des fichiers de capture énormes si vous demandez de les journaliser. Il est beaucoup plus efficace d'utiliser des règles pour définir quel type de trafic est intéressant et laisser Snort ignorer le reste.

Snort se comporte de différentes manières en fonction des options que vous passez en ligne de commande au démarrage. Vous pouvez voir la grande liste d'options avec la commande suivante :

```
snort --help
```

On va commencer par observer tout simplement les entêtes des paquets IP utilisant la commande :

```
snort -v -i eth0
```

**ATTENTION : le choix de l'interface devient important si vous avez une machine avec plusieurs interfaces réseau. Dans notre cas, vous pouvez ignorer entièrement l'option ```-i eth0```et cela devrait quand-même fonctionner correctement.**

Snort s'éxecute donc et montre sur l'écran tous les entêtes des paquets IP qui traversent l'interface eth0. Cette interface reçoit tout le trafic en provenance de la machine "Client" puisque nous avons configuré le IDS comme la passerelle par défaut.

Pour arrêter Snort, il suffit d'utiliser `CTRL-C` (**attention** : il peut arriver de temps à autres que Snort ne réponde pas correctement au signal d'arrêt. Dans ce cas-là, il faudra utiliser `kill` depuis un deuxième terminal pour arrêter le process).

## Utilisation comme un IDS

Pour enregistrer seulement les alertes et pas tout le trafic, on execute Snort en mode IDS. Il faudra donc spécifier un fichier contenant des règles. 

Il faut noter que `/etc/snort/snort.config` contient déjà des références aux fichiers de règles disponibles avec l'installation par défaut. Si on veut tester Snort avec des règles simples, on peut créer un fichier de config personnalisé (par exemple `mysnort.conf`) et importer un seul fichier de règles utilisant la directive "include".

Les fichiers de règles sont normalement stockes dans le répertoire `/etc/snort/rules/`, mais en fait un fichier de config et les fichiers de règles peuvent se trouver dans n'importe quel répertoire de la machine. 

Par exemple, créez un fichier de config `mysnort.conf` dans le repertoire `/etc/snort` avec le contenu suivant :

```
include /etc/snort/rules/icmp2.rules
```

Ensuite, créez le fichier de règles `icmp2.rules` dans le repertoire `/etc/snort/rules/` et rajoutez dans ce fichier le contenu suivant :

`alert icmp any any -> any any (msg:"ICMP Packet"; sid:4000001; rev:3;)`

On peut maintenant éxecuter la commande :

```
snort -c /etc/snort/mysnort.conf
```

Vous pouvez maintenant faire quelques pings depuis votre "Client" et regarder les résultas dans le fichier d'alertes contenu dans le repertoire `/var/log/snort/`.


## Ecriture de règles

Snort permet l'écriture de règles qui décrivent des tentatives de exploitation de vulnérabilités bien connues. Les règles Snort prennent en charge à la fois, l'analyse de protocoles et la recherche et identification de contenu.

Il y a deux principes de base à respecter :

* Une règle doit être entièrement contenue dans une seule ligne
* Les règles sont divisées en deux sections logiques : (1) l'entête et (2) les options.

L'entête de la règle contient l'action de la règle, le protocole, les adresses source et destination, et les ports source et destination.

L'option contient des messages d'alerte et de l'information concernant les parties du paquet dont le contenu doit être analysé. Par exemple:

```
alert tcp any any -> 192.168.1.0/24 111 (content:"|00 01 86 a5|"; msg: "mountd access";)
```

Cette règle décrit une alerte générée quand Snort trouve un paquet avec tous les attributs suivants :

* C'est un paquet TCP
* Emis depuis n'importe quelle adresse et depuis n'importe quel port
* A destination du réseau identifié par l'adresse 192.168.1.0/24 sur le port 111

Le text jusqu'au premier parenthèse est l'entête de la règle. 

```
alert tcp any any -> 192.168.1.0/24 111
```

Les parties entre parenthèses sont les options de la règle:

```
(content:"|00 01 86 a5|"; msg: "mountd access";)
```

Les options peuvent apparaître une ou plusieurs fois. Par exemple :

```
alert tcp any any -> any 21 (content:"site exec"; content:"%"; msg:"site
exec buffer overflow attempt";)
```

La clé "content" apparait deux fois parce que les deux strings qui doivent être détectés n'apparaissent pas concaténés dans le paquet mais a des endroits différents. Pour que la règle soit déclenchée, il faut que le paquet contienne **les deux strings** "site exec" et "%". 

Les éléments dans les options d'une règle sont traités comme un AND logique. La liste complète de règles sont traitées comme une succession de OR.

## Informations de base pour le règles

### Actions :

```
alert tcp any any -> any any (msg:"My Name!"; content:"Skon"; sid:1000001; rev:1;)
```

L'entête contient l'information qui décrit le "qui", le "où" et le "quoi" du paquet. Ça décrit aussi ce qui doit arriver quand un paquet correspond à tous les contenus dans la règle.

Le premier champ dans le règle c'est l'action. L'action dit à Snort ce qui doit être fait quand il trouve un paquet qui correspond à la règle. Il y a six actions :

* alert - générer une alerte et écrire le paquet dans le journal
* log - écrire le paquet dans le journal
* pass - ignorer le paquet
* drop - bloquer le paquet et l'ajouter au journal
* reject - bloquer le paquet, l'ajouter au journal et envoyer un `TCP reset` si le protocole est TCP ou un `ICMP port unreachable` si le protocole est UDP
* sdrop - bloquer le paquet sans écriture dans le journal

### Protocoles :

Le champ suivant c'est le protocole. Il y a trois protocoles IP qui peuvent être analysés par Snort : TCP, UDP et ICMP.


### Adresses IP :

La section suivante traite les adresses IP et les numéros de port. Le mot `any` peut être utilisé pour définir "n'import quelle adresse". On peut utiliser l'adresse d'une seule machine ou un block avec la notation CIDR. 

Un opérateur de négation peut être appliqué aux adresses IP. Cet opérateur indique à Snort d'identifier toutes les adresses IP sauf celle indiquée. L'opérateur de négation est le `!`.

Par exemple, la règle du premier exemple peut être modifiée pour alerter pour le trafic dont l'origine est à l'extérieur du réseau :

```
alert tcp !192.168.1.0/24 any -> 192.168.1.0/24 111
(content: "|00 01 86 a5|"; msg: "external mountd access";)
```

### Numéros de Port :

Les ports peuvent être spécifiés de différentes manières, y-compris `any`, une définition numérique unique, une plage de ports ou une négation.

Les plages de ports utilisent l'opérateur `:`, qui peut être utilisé de différentes manières aussi :

```
log udp any any -> 192.168.1.0/24 1:1024
```

Journaliser le traffic UDP venant d'un port compris entre 1 et 1024.

--

```
log tcp any any -> 192.168.1.0/24 :6000
```

Journaliser le traffic TCP venant d'un port plus bas ou égal à 6000.

--

```
log tcp any :1024 -> 192.168.1.0/24 500:
```

Journaliser le traffic TCP venant d'un port privilégié (bien connu) plus grand ou égal à 500 mais jusqu'au port 1024.


### Opérateur de direction

L'opérateur de direction `->`indique l'orientation ou la "direction" du trafique. 

Il y a aussi un opérateur bidirectionnel, indiqué avec le symbole `<>`, utile pour analyser les deux côtés de la conversation. Par exemple un échange telnet :

```
log 192.168.1.0/24 any <> 192.168.1.0/24 23
```

## Alertes et logs Snort

Si Snort détecte un paquet qui correspond à une règle, il envoie un message d'alerte ou il journalise le message. Les alertes peuvent être envoyées au syslog, journalisées dans un fichier text d'alertes ou affichées directement à l'écran.

Le système envoie **les alertes vers le syslog** et il peut en option envoyer **les paquets "offensifs" vers une structure de repertoires**.

Les alertes sont journalisées via syslog dans le fichier `/var/log/snort/alerts`. Toute alerte se trouvant dans ce fichier aura son paquet correspondant dans le même repertoire, mais sous le fichier `snort.log.xxxxxxxxxx` où `xxxxxxxxxx` est l'heure Unix du commencement du journal.

Avec la règle suivante :

```
alert tcp any any -> 192.168.1.0/24 111
(content:"|00 01 86 a5|"; msg: "mountd access";)
```

un message d'alerte est envoyé à syslog avec l'information "mountd access". Ce message est enregistré dans `/var/log/snort/alerts` et le vrai paquet responsable de l'alerte se trouvera dans un fichier dont le nom sera `/var/log/snort/snort.log.xxxxxxxxxx`.

Les fichiers log sont des fichiers binaires enregistrés en format pcap. Vous pouvez les ouvrir avec Wireshark ou les diriger directement sur la console avec la commande suivante :

```
tcpdump -r /var/log/snort/snort.log.xxxxxxxxxx
```

Vous pouvez aussi utiliser des captures Wireshark ou des fichiers snort.log.xxxxxxxxx comme source d'analyse por Snort.

## Exercises

**Réaliser des captures d'écran des exercices suivants et les ajouter à vos réponses.**

### Essayer de répondre à ces questions en quelques mots, en réalisant des recherches sur Internet quand nécessaire :

**Question 1: Qu'est ce que signifie les "preprocesseurs" dans le contexte de Snort ?**

---

**Réponse :**  

Les préprocesseurs permettent d'étendre les fonctionnalités de Snort en permettant aux utilisateurs et aux programmeurs autorisés de déposer assez facilement des plugins modulaires dans Snort. Le code du préprocesseur est exécuté avant l'appel du moteur de détection, mais après le décodage du paquet. Le paquet peut être modifié ou analysé de manière hors bande en utilisant ce mécanisme.

Source: http://manual-snort-org.s3-website-us-east-1.amazonaws.com/node17.html



---

**Question 2: Pourquoi êtes vous confronté au WARNING suivant `"No preprocessors configured for policy 0"` lorsque vous exécutez la commande `snort` avec un fichier de règles ou de configuration "fait-maison" ?**

---

**Réponse :**  
Ce message indique qu'aucun préprocesseur snort n'est chargé par le fichier de configuration.

---


### Trouver du contenu :

Considérer la règle simple suivante:

alert tcp any any -> any any (msg:"Mon nom!"; content:"Rubinstein"; sid:4000015; rev:1;)

**Question 3: Qu'est-ce qu'elle fait la règle et comment ça fonctionne ?**

---

**Réponse :**

Décortication de chaque partie :

- alert - génère une alerte et écrit le paquet dans le journal
- tcp – analyse sur le protocole TCP
- any (premier) – traite toutes les adresses IP
- any (deuxième) – traite tous les ports
- flèche : source -> destination

Entre parenthèse, les options :

- msg : L'option de règle msg indique au moteur de journalisation et d'alerte le message à imprimer avec un vidage de paquets ou une alerte.
- content : Il permet à l'utilisateur de définir des règles qui recherchent un contenu spécifique dans la charge utile du paquet et déclenchent une réponse en fonction de ces données.
- sid : Le mot-clé sid est utilisé pour identifier de manière unique les règles de Snort. Ces informations permettent aux plugins de sortie d'identifier facilement les règles. Cette option doit être utilisée avec le mot clé rev.
- rev : Le mot clé rev est utilisé pour identifier de manière unique les révisions des règles Snort.

En résumé :

Pour tout paquet envoyé par le protocole tcp et contenant « Rubinstein », une alerte est générée et le paquet est écrit dans le journal avec le message « Mon nom ! » (dans les fichiers alerts et snort.log.xxxxxxxxxx). 
Dans le fichier d'alerte se trouvent aussi le sid :4000015 et le rev 1 qui permettent d’identifier cette règle. 
source : http://manual-snort-org.s3-website-us-east-1.amazonaws.com/node31.html  

---

Utiliser nano pour créer un fichier `myrules.rules` sur votre répertoire home (```/root```). Rajouter une règle comme celle montrée avant mais avec votre text, phrase ou mot clé que vous aimeriez détecter. Lancer Snort avec la commande suivante :

```
sudo snort -c myrules.rules -i eth0
```

**Question 4: Que voyez-vous quand le logiciel est lancé ? Qu'est-ce que tous ces messages affichés veulent dire ?**

---

![](images/4_1.png)

![](images/4_2.png)

**Réponse :**  

On trouve diverses informations d'initialisation, comme le démarrage des composants du logiciel ou le nombre de règles configurées.

Après le message "Initialization Complete" on voit la version de Snort et la licence.
 
Quand Snort est prêt et commence l'analyse réseau, le message "Commencing packet processing" apparaît.

---

Aller à un site web contenant dans son text la phrase ou le mot clé que vous avez choisi (il faudra chercher un peu pour trouver un site en http... Si vous n'y arrivez pas, vous pouvez utiliser [http://neverssl.com](http://neverssl.com) et modifier votre votre règle pour détecter un morceau de text contenu dans le site).

**Question 5: Que voyez-vous sur votre terminal quand vous chargez le site depuis la machine Client? (vous pouvez utiliser wget pour lancer la requête http ou le navigateur Web lynx - il suffit de taper ```lynx neverssl.com```. Le navigateur lynx est un navigateur basé sur text, sans interface graphique)**

---

![](images/5.png)

**Réponse :**  

Sur l'IDS on voit un certain nombre de warning "no preprocessors configured..." à chaque requête du client

---

Arrêter Snort avec `CTRL-C`.


**Question 6: Que voyez-vous quand vous arrêtez snort ? Décrivez en détail toutes les informations qu'il vous fournit.**

---

![](images/6_1.png)
![](images/6_2.png)

**Réponse :**

On voit le résumé de l'analyse réseau. Il contient par exemple : 

- Le nombre de paquets observés
- Une liste des protocoles utilisés avec leur fréquence
- Toutes les actions effectuées

---


Aller au répertoire /var/log/snort. Ouvrir le fichier `alert`. Vérifier qu'il y ait des alertes pour votre text choisi.

**Question 7: A quoi ressemble l'alerte ? Qu'est-ce que chaque élément de l'alerte veut dire ? Décrivez-la en détail !**

---

![](images/7.png)

**Réponse :**  

Pour chaque alerte il y a les champs suivants (ligne par ligne):

- SID de la règle, message de l'alerte
- Date, ip source/destination et ports
- Information protocole IP
- Information TCP
- Options du protocole

---


### Detecter une visite à Wikipedia

Ecrire une règle qui journalise (sans alerter) un message à chaque fois que Wikipedia est visité **SPECIFIQUEMENT DEPUIS VOTRE MACHINE CLIENT**. Ne pas utiliser une règle qui détecte un string ou du contenu**.

**Question 8: Quelle est votre règle ? Où le message a-t'il été journalisé ? Qu'est-ce qui a été journalisé ?**

---

**Réponse :**  

La règle utilisée: `log tcp 192.168.2.3 any -> 91.198.174.192 443 (msg:"Visite vers fr.wikipedia.org"; sid:4000018; rev:1)`

Le message a été journalisé dans le fichier de log de la session Snort, qui se trouve dans /var/log/snort/snort.log.xxxxxxxxx.

Dans le journal se trouvent tous les paquets qui correspondent à la règle.

![](images/8.png)

---



### Détecter un ping d'un autre système

Ecrire une règle qui alerte à chaque fois que votre machine IDS reçoit un ping depuis une autre machine (la seule autre machine que vous avez à disposition c'est la machine Client). Assurez-vous que **ça n'alerte pas** quand c'est vous qui envoyez le ping depuis l'IDS vers un autre système !

**Question 9: Quelle est votre règle ?**

---

**Réponse :**  
`alert icmp !192.168.2.2 any -> 192.168.2.2 any (itype:8; msg:"Ping vers IDS"; sid:4000009; rev:1)`

---


**Question 10: Comment avez-vous fait pour que ça identifie seulement les pings entrants ?**

---

**Réponse :**

On a filtré seulement les paquets de type "Echo Request" (ICMP 8*) qui arrivent vers la machine locale depuis une autre machine 

*Source: https://www.informit.com/articles/article.aspx?p=101171&seqNum=6
  

---


**Question 11: Où le message a-t-il été journalisé ?**

---

**Réponse :**  
Dans le fichier .log de la session Snort, en plus du fichier alerts. 

Pour rappel ces fichiers se situent dans /var/log/snort

---


**Question 12: Qu'est-ce qui a été journalisé ? (vous pouvez lire les fichiers log utilisant la commande `tshark -r nom_fichier_log` **

---

**Réponse :**  
![](images/12.png)
On voit toutes les requêtes echo qui ont été effectuées

---

--

### Detecter les ping dans les deux sens

Faites le nécessaire pour que les pings soient détectés dans les deux sens.

**Question 13: Qu'est-ce que vous avez fait pour détecter maintenant le trafic dans les deux sens ?**

---

**Réponse :**  

On a ajouté une nouvelle règle qui détectaient aussi les réponses ICMP depuis la machine locale

`alert icmp 192.168.2.2 any -> !192.168.2.2 any (itype:0; msg:"Reponse ping depuis IDS"; sid:4000013; rev:1)`

On peut alors les retrouver aussi dans les logs et ainsi suivre la conversation (s'il n'y a pas d'erreur)

![](images/13.png) 

---


### Detecter une tentative de login SSH

Essayer d'écrire une règle qui Alerte qu'une tentative de session SSH a été faite depuis la machine Client sur l'IDS.

**Question 14: Quelle est votre règle ? Montrer la règle et expliquer en détail comment elle fonctionne.**

---

**Réponse :**  
`alert tcp 192.168.2.3 any -> 192.168.2.2 22 (flags: S;msg : "SSH client to IDS: SYN detected" ; sid:4000015; rev:1;)`

Génère des alertes quand des paquets sont envoyés depuis l’adresse IP du client et depuis n’importe quel port à l’adresse IP de l’IDS sur le port 22 (SSH) avec le protocole TCP. 

Le message d’alerte est "SSH client to IDS: SYN detected". Le flag S a été ajouté pour détecter seulement la tentative de connexion. Ce flag vérifie que qu’il y ait un SYN dans le header. SYN montre une tentative de connexion.

---


**Question 15: Montrer le message enregistré dans le fichier d'alertes.** 

---

**Réponse :**

![](images/15.png)  

---



### Analyse de logs

Depuis l'IDS, servez-vous de l'outil ```tshark```pour capturer du trafic dans un fichier. ```tshark``` est une version en ligne de commandes de ```Wireshark```, sans interface graphique. 

Pour lancer une capture dans un fichier, utiliser la commande suivante :

```
tshark -w nom_fichier.pcap
```

Générez du trafic depuis le deuxième terminal qui corresponde à l'une des règles que vous avez ajoutées à votre fichier de configuration personnel. Arrêtez la capture avec ```Ctrl-C```.

**Question 16: Quelle est l'option de Snort qui permet d'analyser un fichier pcap ou un fichier log ?**

---

**Réponse :**  
L'option `-r <file>` permet d'analyser un fichier .pcap 

---

Utiliser l'option correcte de Snort pour analyser le fichier de capture Wireshark que vous venez de générer.

**Question 17: Quelle est le comportement de Snort avec un fichier de capture ? Y-a-t'il une différence par rapport à l'analyse en temps réel ?**

---

**Réponse :**  
Le comportement est identique à une analyse réseau, hormis que l'analyse de fichier s'arrête à lorsqu'il atteint la fin de ce dernier. 
---

**Question 18: Est-ce que des alertes sont aussi enregistrées dans le fichier d'alertes?**

---

**Réponse :**  
Oui, les alertes sont aussi enregistrées. 

Le fichier journal de l'analyse de fichier est également créé

---

--

### Contournement de la détection

Faire des recherches à propos des outils `fragroute` et `fragrouter`.

**Question 19: A quoi servent ces deux outils ?**

---

**Réponse :**  

`fragroute`

Fragroute intercepte, modifie et réécrit le trafic de sortie destiné à un hôte spécifié, mettant en œuvre la plupart des attaques décrites dans le document Secure Networks «Insertion, Evasion, and Denial of Service: Eluding Network Intrusion Detection» de janvier 1998.

Source : https://tools.kali.org/information-gathering/fragroute

`fragrouter`

Fragrouter est une boîte à outils d'évasion de détection d'intrusion réseau. Il met en œuvre la plupart des attaques décrites dans l'article de janvier 1998 intitulé «Insertion, évasion et déni de service: Eluding Network Intrusion Detection» de Secure Networks.

Source : https://tools.kali.org/information-gathering/fragrouter#:~:text=Fragrouter%20is%20a%20network%20intrusion,Detection%E2%80%9D%20paper%20of%20January%201998.

---


**Question 20: Quel est le principe de fonctionnement ?**

---

**Réponse :**  

`fragroute`

Fragroute dispose d'un langage de jeu de règles simple pour retarder, dupliquer, supprimer, fragmenter, chevaucher, imprimer, réorganiser, segmenter, acheminer la source ou autre monkey avec tous les paquets sortants destinés à un hôte cible, avec une prise en charge minimale du comportement aléatoire ou probabiliste.

Source : https://tools.kali.org/information-gathering/fragroute

`fragrouter`

Conceptuellement, le fragrouter n'est qu'un routeur de fragmentation unidirectionnel - les paquets IP sont envoyés de l'attaquant au fragrouter, ce qui les transforme en un flux de données fragmenté à transmettre à la victime.

Source : https://tools.kali.org/information-gathering/fragrouter#:~:text=Fragrouter%20is%20a%20network%20intrusion,Detection%E2%80%9D%20paper%20of%20January%201998.

---


**Question 21: Qu'est-ce que le `Frag3 Preprocessor` ? A quoi ça sert et comment ça fonctionne ?**

---

**Réponse :**  
Le préprocesseur frag3 est un module de réassemblage de défragmentation IP basé sur la cible pour Snort. Frag3 est conçu avec les objectifs suivants :

1) Une exécution plus rapide avec une gestion des données moins complexe.
2) Techniques anti-évasion de modélisation de l'hôte basées sur la cible.

Frag3 utilise la structure de données sfxhash et les listes liées pour la gestion des données en interne, ce qui lui permet d'avoir des performances beaucoup plus prévisibles et déterministes dans n'importe quel environnement, ce qui devrait nous aider à gérer des environnements fortement fragmentés.

L'idée d'un système basé sur les cibles est de modéliser les cibles réelles sur le réseau au lieu de simplement modéliser les protocoles à la place de rechercher les attaques dans ces derniers. Lorsque les stacks d’IP sont écrites pour différents systèmes d’exploitations, elles sont généralement mis en œuvre par des personnes qui lisent les RFC, et écrivent ensuite leur interprétation de ce que la RFC décrit dans le code.

Source : https://www.snort.org/faq/readme-frag3

---


L'utilisation des outils ```Fragroute``` et ```Fragrouter``` nécessite une infrastructure un peu plus complexe. On va donc utiliser autre chose pour essayer de contourner la détection.

L'outil nmap propose une option qui fragmente les messages afin d'essayer de contourner la détection des IDS. Générez une règle qui détecte un SYN scan sur le port 22 de votre IDS.


**Question 22: A quoi ressemble la règle que vous avez configurée ?**

---

**Réponse :**  
La règle est presque la même que pour la question 14: "Detecter une tentative de login SSH"

`alert tcp !192.168.2.2 any -> 192.168.2.2/24 22 (flags:S; msg:"SSH to IDS: SYN detected"; sid:4000022; rev:1;)` 

---


Ensuite, servez-vous du logiciel nmap pour lancer un SYN scan sur le port 22 depuis la machine Client :

```
nmap -sS -p 22 192.168.1.2
```
Vérifiez que votre règle fonctionne correctement pour détecter cette tentative. 

Ensuite, modifiez votre commande nmap pour fragmenter l'attaque :

```
nmap -sS -f -p 22 --send-eth 192.168.1.2
```

**Question 23: Quel est le résultat de votre tentative ?**

---

**Réponse :**  

Avec un SYN scan fragmenté, il n'y a pas d'alerte ![preuve](images/23.png))

---


Modifier le fichier `myrules.rules` pour que snort utiliser le `Frag3 Preprocessor` et refaire la tentative.


**Question 24: Quel est le résultat ?**

---

**Réponse :**  

Le code à ajouter dans le fichier de règles est le suivant:

```conf
preprocessor frag3_global
preprocessor frag3_engine
```

L'alerte a bien lieu car les paquets sont défragmentés ![](images/24.png)

---


**Question 25: A quoi sert le `SSL/TLS Preprocessor` ?**

---

**Réponse :**  

Le trafic chiffré doit être ignoré par Snort pour des raisons de performances et pour réduire les faux positifs. Ce préprocesseur inspecte le trafic SSL et TLS et détermine éventuellement si et quand arrêter l'inspection de celui-ci.

Source : https://www.snort.org/faq/readme-ssl

---


**Question 26: A quoi sert le `Sensitive Data Preprocessor` ?**

---

**Réponse :**  
Le Sensitive Data preprocessor effectue la détection et le filtrage des informations personnellement identifiables (PII). Ces informations incluent les numéros de carte de crédit, les numéros de sécurité sociale des États-Unis et les adresses e-mail. Une syntaxe d'expression régulière limitée est également incluse pour définir vos propres informations personnelles.

Source : https://www.snort.org/faq/readme-sensitive_data

---

### Conclusions


**Question 27: Donnez-nous vos conclusions et votre opinion à propos de snort**

---

**Réponse :**
Nous avons trouvé que Snort est très efficace et puissant pour la gestion de logs. La syntaxe des règles est également assez simple et plutôt intuitive.

Cependant, celui-ci est complexe à paramétrer sur un environnement. Il faut tout configurer à la main et sélectionner les bonnes règles. Des règles mal configurées risquent de créer des fausses alertes et rendra beaucoup plus compliqué l’analyse des logs.

Dans la majorité des cas, un firewall bien configuré nous semble rester bien plus important. Il vaut mieux prévenir que guérir. Mais la possibilité d’analyser le contenu des paquets après le passage plutôt qu’en directe est une idée très intéressante afin de garder la trace d’intrusion tout en préservant la performance du trafic.

---

### Cleanup

Pour nettoyer votre système et effacer les fichiers générés par Docker, vous pouvez exécuter le script [cleanup.sh](scripts/cleanup.sh). **ATTENTION : l'effet de cette commande est irréversible***.


<sub>This guide draws heavily on http://cs.mvnu.edu/twiki/bin/view/Main/CisLab82014</sub>