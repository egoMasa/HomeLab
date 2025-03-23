# Setup du logiciel GNS3 
* GNS3 permet d'émuler entièrement les IOS des équipement et non de simplement les simuler. 
* On utilise la VM GNS3 pour hébergé les nodes (qui sont des VM elles aussi). Ca permet de limiter que notre PC hôte se retrouve à 100% on veut controler et limiter la charge. 
* Pour une VM performante on va mettre les specs suivante : 16 Go (16,384 Mo) et 4 coeur CPU

# Mise en place de l'environnement 
## 1) Hyperviseur VirtualBox
1. Installer virutalbox
2. Créer une interface host en auto avec dhcp actif
3. Importer VM GNS3 VirtualBox
4. Configurer deux interfaces réseaux de la VM :
	1. Adapter 1 : Réseau privé hôte > VirtualBox Host-Only Ethernet
	2. Adapter 2 : NAT
5. Configurer les répartitions ressources de la VM directement dans VirtualBox
	1. Coeur : 4
	2. RAM : 16384 Mo

* Logique générale avec la GNS3 VM : GNS3 est composé de deux éléments pour fonctionner (un interface client qui intéragit avec un serveur GNS3 qui émule les instances et configurations), on est sur un fonctionnement client (graphique) et serveur (Emulateur Hebergeur). De base GNS3 considère que le client peut être aussi le serveur, c'est pour ca qu'on renseigne le serveur GNS3 sur le localhost (c'est notre machine local) mais on peut configurer n'importe quelle machine comme serveur GNS3 il faut juste que la machine serveur soit joignable (ping) depuis le client. Pour cela GNS3 fournit une VM pour chaque hyperviseur que l'on peut déployer comme serveur et ou chaque action faite sur le client de facon graphique seront transmises et executer sur le serveur GNS3 VM. Le fonctionnement de la VM est plutôt simple, quand elle va boot et elle automatiquement déployer un IP et un réseau relié au réseau de l'interface Virutal-Host-only de l'hyperviseur. En gros on a notre hyperviseur sur notre PC qui créer une interface réseau spécifique dans un réseau spécifique qui fait aussi DHCP et ou chaque VM avec une interface relié à ce réseau va obtenir une IP dans ce sous réseau spécifique. On a créer un sous réseau dans un machine local et ou les VM sont. Donc on boot notre VM dans ce sous réseau elle nous affiche une IP et un port qui seront les coordonnées que l'on va devoir renseigner dans le client GNS3. 
* Autre point, la GNS3 VM (serveur GNS3) possède un port web avec une interface graphique qui permet de visualiser l'état de la VM et de voir les saves du projet sur lequel on travail. D'ailleurs sur le client GNS3, quand on veut ouvrir un projet et qu'on a déjà renseigné le serveur, les projets hébergés sur le serveur apparaitront et seront disponible.
* Autre point, la GNS3 grossit au fur et a mesure que l'on importe des appliances (ISO équipement) et cette VM peut être exporter et importable sur tout autre PC, plutot pratique car on perd rien.
## 2) Logiciel GNS3
* Ne pas créer de projet 
* Dans préférences > Server : 
	* Désactiver "Enable local server" car notre serveur n'est pas notre machine local mais la VM
	* Renseigner l'IP de notre VM, Port 80, User/MP : gns3
* Dans préférences > GNS3 VM
	* Activer Enable the GNS3 VM

# Importer les appliances necessaires
* Plusieurs facons de faire 
	* Se rendre sur le marketplace/GNS3 Server
	* Importer les recettes 
	* Télécharger les composants necessaires, une recheche google pour trouver un dépot des IOS
* Voici ce qu'il faut importer :
	* Switch Ethernet de couche 2 : Cisco IOSvL2
	* Multilayer Switch : Cisco IOSvL2 (supporte aussi L3)
	* Router : Cisco IOSv (L3)
	* Firewall : Pfsense
	* Serveur : 
		* Debian (sans interface graphique)
		* Windows Server 2022 (avec interface)


# Inventaire des services : 

# Questions d'architecture réseau 

* Placement des IDPS (inline, passif), centralisé ou indépendant
* Choix du placement d'externalisation avec routeur BGB en sortie
	* Choix 1 : 1 IP publique disponible sur routeur BGP donc IP OUT FW PRIV donc on externalise sur le routeur BGP
	* Choix 2 : Demande de deux IP pub donc IP OUT FW PUB donc on externalise sur le FW en pub. 
	* Règle générale : On externalise a chaque fois sur le premier équipement qui une IP PUB. 
