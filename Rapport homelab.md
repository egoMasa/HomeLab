# Sommaire
1. Présentation & contexte & environnement
2. Cahier des charges
3. Maquette infrastructure + justifications
4. Déploiement et configuration des services
	1. Service 1 : Notion/Concept, etude de marché, sélection, configuration, sécurisation & administration
	2. Service 2 : Notion, etude de marché, sélection, configuration, sécurisation & administration
	3. ...
5. Mise en conformité des SI

# 1) Présentation du projet

Le but de ce projet est de mettre en place une infrastructure réseau et système complète dans un but d'apprentissage et se préparer au plus près des solutions et infrastructure rencontrés dans les missions d'un ingénieur réseau/système/cybersécurité. Le but est d'avoir à la fin une vision claire, globale et complète de à quoi ressemble en réalité, dans le détails les infrastructures des grandes/moyennes entreprises qui hébergent eux-mêmes leur systèmes d'informations (on-premise). L'autre intérêt de ce projet est de comprendre en profondeur les notions et systèmes déployés dans les infrastructures modernes dont les équipes s'occupent au quotidien. Nous en profiterons pour définir l'ensemble des solutions et systèmes primordiales à déployer dans un environnement réel, connaitre leur concept et les besoins auxquels ils répondent et connaitre les différentes solutions disponibles sur le marché.

Il est important de noter que j'ai volontairement choisi une approche open-source gratuite déployable on-premise afin de se détacher des solutions propriétaires payantes et montrer qu'il est possible d'avoir une infrastructure complète, performante et viable en étant souverain de ses données et de dépendre de personnes ou du moins limiter les dépendances techniques externes et politique.

Nous justifierons chaque choix en expliquant en détails les différentes choix possibles autant en terme de solutions, de configuration mais aussi d'infrastructure réseau, nous verrons que il n'existe aucunes maquettes parfaite mais que l'on peut optimiser et mettre en application des bonnes pratiques nous permettant de nous rapprocher d'une infrastructure complète et redondante. 

Concernant la façon de mettre en place ce projet nous utiliserons le logiciel de virtualisation GNS3 car il nous permettra d'importer l'ensemble des équipement réseau et virtualiser l’ensemble des serveurs. Nous utiliserons un PC central sous Linux (Ubuntu Desktop) afin de n'avoir aucunes limitations sur les applicances qui necessitent KVM et QEMU sans hyperviseur de type 2 entre. Donc pas de GNS3 VM ou de Windows comme host, 100% Linux en bare-metal. 

# 2) Cahier des charges

| Matériel               | Nombre | Modèle/ISO    | Rôle                                        | Zones d'emplacement               |
| ---------------------- | ------ | ------------- | ------------------------------------------- | --------------------------------- |
| Routeur                | 1      | CiscoIOSv     | PE du FAI                                   | OUT                               |
| MultiLayerSwitch (SW3) | 8      | CiscoIOSv     | x                                           | LAN(Core, distrib) DC(Spine)      |
| Switch (SW2)           | 8      | CiscoIOSvL2   | x                                           | LAN(Access), DMZ(Access),DC(Leaf) |
| Firewall               | 4      | Pfsense 2.7.2 | Segmente chaques zones (barrage)            | OUT, DMZ, LAN, DC                 |
| Hyperviseur 1          | 4      | Proxmox       | Heberge les services dans des VM/Conteneurs | DMZ, DC                           |


| Services                          | Placement | Solutions possibles (open-source)                      | Utilisé pour                                                                                                      |
| --------------------------------- | --------- | ------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------- |
| DNS Interne                       | DC        | Unbound, BIND9                                         | Résoudre les noms de domaine interne non externalisés                                                             |
| DHCP Central                      | DC        | ISC Kea+Stork                                          | Fournir les adresses IP sur l'ensemble du réseau                                                                  |
| Annuaire LDAP                     | DC        | Active Directory, OpenLDAP, Samba ADDC                 | Stocker les comptes utilisateurs, droits et permissions utilisé pour les authentifications                        |
| RADIUS                            | DC        | FreeRADIUS                                             | Authentifier les usagers enregistré et autorisé sur le réseau                                                     |
| CAS                               | DC        | Keycloak                                               | Fournir les éléments d'authentification (tokens) aux usagers pour se connecter en SSO aux autres services         |
| Supervision de performances       | DC        | Zabbix, Nagios(Centreon, Icinga, CheckMK) , Prometheus | Superviser et détecter l'état des endpoints&équipements réseau et cartographier                                   |
| Supervision de sécurité (SOC XDR) | DC        | Wazuh ou Graylog+Shuffle, OpenCTI, CrowdSec            | Superviser les comportements et actions sur les endpoints et faire de la remédiation active                       |
| Bastion d'administration          | DC        | Apache Guacamole                                       | Portail pour se connecter en admin sur les différents serveurs                                                    |
| Inventaire (ITSM)                 | DC        | GLPI, OpenNebula/OpenStack                             | Rassembler l'ensemble des équipements, licences et biens supports                                                 |
| Ticketing                         | DC        | GLPI                                                   | Gérer les incidents techniques des usagers                                                                        |
| Stockage cloud                    | DC        | NextCloud                                              | Dépot de fichier pour les usagers (documents, photos)                                                             |
| NTP                               | DC        |                                                        | Synchroniser l'heure des équipements sur l'ensemble du réseau                                                     |
| Developpement & dépot de code     | DC        | GitLab, Jenkins                                        | Environnement de développement CI/CD et stockage de scripts, recette                                              |
| Gestion de projet                 | DC        | Redmine                                                | Fournir un environnement de gestion de projet, avec groupe, tâches, diagramme                                     |
| Reverse Proxy                     | DMZ       | HA-Proxy                                               | Rediriger les requêtes internes/externes vers l'IP réel du serveur/hyperviseur du service demandé + loadbalancing |
| Proxy                             | DMZ       | ?                                                      | Contrôler les requêtes internes vers l'extérieur pour les usagers                                                 |
| WAF                               | DMZ       | BunkerWeb                                              | Détecter et protéger les services WEB d'attaques et DoS                                                           |
| SMTP Inbound MX                   | DMZ       |                                                        |                                                                                                                   |
| SMTP Outbound                     | DMZ       |                                                        |                                                                                                                   |
| Serveur Web                       | DMZ       | Apache, Nginx, Django, Nodejs                          | Serveur WEB vitrine externalisé avec backend légé                                                                 |
| DNS Publique                      | DMZ       | BIND9                                                  | Résoudre les noms de domaine externalisé que l'on gère                                                            |
| VPN                               | FW_OUT    | OpenVPN                                                | Permettre aux usagers à distance d'accéder au réseau/services non externalisés                                    |
| Automatisation de tâches serveurs | DC        | Ansible, Puppet                                        | Executer des actions d'administration sur les serveurs via SSH et des playbook                                    |
| IPS/IDS                           | FW        | Suricata, Snort                                        | Surveiller les flux réseaux, les analyser et bloquer automatiquement                                              |
# 3) Architecture réseau

Très bien, donc nous commençons ce projet en abordant la notion de l'architecture réseau. Ça va être une des briques angulaires de ce projet. Le but étant de voir les différents modèles d'architecture et comment est-ce qu'un architecte réseau approche les différents designs et modèles qui sont possibles de mettre en place et qui sont recommandés, quelles sont les différentes bonnes pratiques. Nous verrons qu'il existe beaucoup de modèles différents et nous allons essayer de trouver un modèle le plus optimisé et qui se rapproche le plus possible des bonnes pratiques que l'on va retrouver. On peut commencer par citer que dans le monde des réseaux, il existe des modèles par tiers, tiers 1, tiers 2 et tiers 3, chacun étant ayant leurs avantages et leurs inconvénients, mais généralement, au plus on monte dans les tiers, au plus haut mieux c'est. Donc de façon naturelle, sans même forcément réfléchir sur le pourquoi du comment, nous partons sur un modèle de tiers 3 car il permet de nombreux avantages, notamment une haute redondance en essayant de viser un full mesh, des différentes couches logiques qui ont chacune une fonction, une fonction notamment de commutation, de routage et qui permet d'avoir de la redondance et de la haute disponibilité. Vous allez du coup venir identifier trois différentes couches, la couche access qui va relier les différents terminaux, vraiment la finalité de ce qui va devoir communiquer, les différents endpoints qui vont être reliés à la couche d'agrégation qui elle-même va être reliée à la couche cœur. Nous verrons aussi qu'il y a, que l'on va parler en termes de zones, on va venir aborder le terme de zone LAN, de zone DMZ, de zone datacenter interne non externalisée, ainsi que des zones internet ou out. Et le but est de montrer qu'il est possible de faire une architecture complète permettant d'externaliser différents services de façon sécurisée et maîtrisée en ayant la redondance et la qualité de services des réseaux LAN standards. Nous aborderons aussi dans une approche orientée datacenter, comme nous le verrons, en abordant l'ensemble des différentes configurations et des bonnes pratiques que l'on retrouve généralement, que ce soit pour les questions de stockage, de sauvegarde, mais aussi de modèles d'architecture spine leaf qui diffère un peu du modèle tier 3 que l'on va retrouver au niveau du LAN. Le but étant de cloisonner l'ensemble de ces différentes zones via des firewalls ainsi que des VRF et des VLAN et de rajouter par-dessus les services dits orientés réseau, pas forcément en système, mais réseau, qui vont permettre le bon fonctionnement ainsi que la sécurité du réseau sur chacune des couches du modèle OSI.

## 3.1) Modèle d'architecture réseau
La conception d'une infrastructure réseau moderne ne peut s'affranchir d'une réflexion profonde sur sa topologie. Pour garantir l'agilité, la performance et surtout la résilience d'un Système d'Information (SI), l'industrie s'appuie sur des **modèles hiérarchiques**. Cette approche consiste à segmenter le réseau en couches logiques distinctes, permettant de passer d'un simple "assemblage de câbles" à une architecture déterministe où chaque flux est maîtrisé, de l'endpoint jusqu'au cœur de l'infrastructure.

### 3.1.1) Le Modèle Tier 1 : L'Architecture à Plat (Non-Hiérarchique)

Le modèle **Tier 1**, souvent qualifié d'architecture "flat", représente le stade embryonnaire du réseau. Dans ce schéma, la distinction entre les fonctions de commutation et de routage est quasi inexistante. L'ensemble des terminaux (postes de travail, serveurs, imprimantes) est raccordé à une couche de commutation unique, souvent dépourvue de mécanismes de redondance.

Bien que cette solution soit économiquement imbattable pour de très petites structures ou des environnements domestiques, elle s'avère rapidement critique en milieu professionnel. L'absence de segmentation (hors VLANs basiques) expose le réseau à des tempêtes de diffusion (broadcast storms) massives et à une incapacité totale à isoler les pannes. En somme, le Tier 1 est un modèle "lab" qui ne répond à aucune exigence de haute disponibilité ou de sécurité granulaire.

### 3.1.2) Le Modèle Tier 2 : L'Architecture à Cœur Fusionné (Collapsed Core)

Le **Tier 2** marque l'entrée dans le monde de l'ingénierie réseau sérieuse. Ici, on commence à structurer la topologie en distinguant deux niveaux :
- **La Couche d'Accès :** Elle assure la connectivité directe des terminaux et gère les problématiques de niveau 2 (VLANs, sécurité de port).
- **La Couche de Cœur/Distribution (Hybride) :** C'est la véritable intelligence du réseau. Elle fusionne les fonctions de routage inter-VLAN, l'application des politiques de sécurité et le transit vers l'extérieur.

Ce modèle est un excellent compromis pour des entreprises de taille moyenne dont l'empreinte physique est limitée (peu de bâtiments ou de salles serveurs). Il offre une première réponse à la redondance via le raccordement des switches d'accès à deux commutateurs centraux. Cependant, sa limite réside dans la **scalabilité**. Dès lors que l'infrastructure s'étend, cette couche hybride devient un goulot d'étranglement, car elle doit gérer simultanément l'agrégation massive de liens et les processus de routage complexes.

### 3.1.3) Le Modèle Tier 3 : L'Architecture Hiérarchique de Référence
C'est le modèle que nous avons retenu pour ce projet, car il incarne l'état de l'art des infrastructures d'entreprise à haute résilience. Le **Tier 3** affine la structure du Tier 2 en isolant physiquement et logiquement trois fonctions critiques.

#### A. La Couche d'Accès (Access Layer)
Sa mission est exclusivement dédiée à l'interaction avec les endpoints. C'est ici que l'on définit l'identité du réseau : affectation dynamique des VLANs, filtrage d'adresses MAC, et alimentation PoE. Elle est conçue pour être simple et modulaire.

#### B. La Couche de Distribution (Aggregation Layer)
Véritable pivot de l'architecture, elle agit comme un point d'agrégation pour les multiples switches d'accès. C'est à ce niveau que s'opère la **commutation de niveau 3**. Elle gère le routage inter-VLAN, la mise en place des listes de contrôle d'accès (ACL) et les protocoles de redondance de passerelle (HSRP/VRRP). Elle isole les problèmes de la couche d'accès pour éviter qu'ils ne remontent jusqu'au cœur.

#### C. La Couche Cœur (Core Layer)
Le "Core" est le backbone, l'autoroute de l'infrastructure. Contrairement à la couche de distribution, le cœur n'est pas là pour filtrer ou inspecter les paquets (ce qui ralentirait le transit), mais pour commuter le trafic à une vitesse extrême entre les différentes zones (LAN, DMZ, Datacenter). En séparant le Cœur de la Distribution, on obtient une architecture capable de supporter une charge massive tout en garantissant un temps de convergence ultra-rapide en cas de coupure de lien

![[Pasted image 20260330214908.png]]
## 3.2) Segmentation du réseau par zones
Une architecture Tier 3 performante n'a de sens que si elle s'accompagne d'une segmentation rigoureuse. L'objectif est d'appliquer le principe du moindre privilège au niveau réseau : chaque flux doit être justifié, identifié et contrôlé. Dans notre infrastructure, nous avons découpé le périmètre en trois blocs fonctionnels majeurs.

### A. La Zone LAN (Local Area Network) : Le Périmètre Utilisateur
La zone LAN est le réceptacle de toute l'activité interne de l'entreprise. C'est ici que l'on retrouve la plus grande diversité d'endpoints : postes de travail fixes, terminaux mobiles (via Wi-Fi), téléphonie IP, caméras de surveillance ou objets connectés (IoT).

Structurellement, cette zone s'étend de la **couche d'Accès** à la **couche de Distribution**. Elle est caractérisée par un fort besoin de mobilité interne et de services réseaux dynamiques (DHCP, authentification Radius). La sécurité y est centrée sur l'utilisateur : nous devons nous assurer qu'un terminal infecté dans un bureau ne puisse pas compromettre l'ensemble du parc grâce à une isolation par VLANs et des politiques de filtrage au niveau des passerelles.

### B. La Zone Internet (Edge / WAN) : La Frontière de Sortie
Cette zone, souvent appelée "OUT" ou "Untrust", représente le point de contact avec le monde extérieur. Elle héberge le routeur de bordure (PE du FAI) et nos firewalls périmétriques. C'est ici que s'opèrent les mécanismes de **NAT (Network Address Translation)** et de **PAT**, permettant à nos adresses privées internes de communiquer sur le réseau public.

C'est une zone de haute vigilance où l'on gère non seulement la sortie des utilisateurs, mais aussi les tunnels VPN pour le télétravail. Elle agit comme le premier rempart, filtrant les tentatives d'intrusion avant même qu'elles n'atteignent nos couches de distribution.

### C. L'Espace Services : Convergence entre DMZ et Datacenter
Traditionnellement, on distinguait physiquement la DMZ (services exposés) du Datacenter (services internes). Dans notre approche, nous privilégions la **mutualisation des ressources physiques** tout en garantissant une **étanchéité logique absolue**.

Plutôt que de multiplier les infrastructures, nous créons une zone de services unique mais multi-facettes. Sur une même infrastructure d'hyperviseurs (Proxmox), nous isolons des segments critiques via des Firewalls et des VRF :

- **La Zone DMZ (Demilitarized Zone) :** Elle contient les serveurs exposés à Internet (Serveur Web, Reverse Proxy, DNS Public). Un compromis sur cette zone ne doit, en aucun cas, permettre un rebond vers le cœur du SI.
- **La Zone Production / Datacenter :** Elle héberge le "cerveau" de l'entreprise (Active Directory, Bases de données, ERP). Cette zone est totalement isolée de l'extérieur.
- **La Zone d'Administration (Management) :** Un segment ultra-sécurisé pour la gestion des équipements (SSH, interfaces Web de configuration).
- **La Zone Infrastructure :** Dédiée aux services de support (Supervision, Sauvegarde, NTP).

Cette approche moderne montre qu'il est possible d'avoir une infrastructure consolidée (On-Premise) tout en appliquant une micro-segmentation stricte.

## 3.3) Architecture datacenter : Modèle Spine-Leaf
Bien que le modèle Tier 3 soit exemplaire pour la gestion des accès utilisateurs (flux Nord-Sud), l'environnement spécifique du Datacenter et de la DMZ impose des contraintes différentes. Dans un centre de données moderne, la majorité du trafic ne sort pas vers l'extérieur, mais circule entre les serveurs eux-mêmes : c'est ce qu'on appelle le **trafic Est-Ouest**.
### A. La problématique du trafic Est-Ouest
Dans notre infrastructure virtualisée sous Proxmox, les services communiquent constamment entre eux (requêtes API, réplication de bases de données, sauvegardes, synchronisation de clusters). On estime aujourd'hui que plus de **80 % du trafic** d'un Datacenter est purement interne.
Le modèle Tier 3 classique montre ici ses limites :
- **Goulot d'étranglement :** Pour que deux serveurs sur des segments différents communiquent, le flux doit souvent remonter jusqu'à la couche de distribution ou de cœur, créant une latence inutile.
- **L'obstacle du Spanning Tree (STP) :** Dans une topologie de niveau 2 (L2), le protocole STP bloque systématiquement les liens redondants pour éviter les boucles. En conséquence, 50 % de la bande passante disponible reste inutilisée ("Blocking state"), ce qui est inacceptable pour un environnement de services haute performance.

### B. Le paradigme Spine-Leaf : L'autoroute déterministe
Pour pallier ces faiblesses, nous adoptons pour notre zone de services l'architecture **Spine-Leaf**. Bien qu'elle semble physiquement proche d'un modèle à deux couches, sa logique de fonctionnement est radicalement différente.
1. **Les Leaf Switches :** Équivalents à la couche d'accès, ils connectent physiquement nos hyperviseurs et serveurs. Chaque Leaf est relié à **tous** les switches Spine.
2. **Les Spine Switches (Épines) :** Ils constituent le backbone ultra-rapide. Leur rôle est uniquement de relier les Leaf entre eux. Un Spine ne connecte jamais directement un serveur.

### C. Pourquoi le Spine-Leaf est-il supérieur pour nos services ?
L'innovation majeure réside dans l'abandon du niveau 2 au profit du **routage de niveau 3 (L3)** dès le haut de la baie (Top-of-Rack).
- **Suppression du Spanning Tree :** En utilisant des liens routés (L3), nous n'avons plus de boucles logiques. Le Spanning Tree n'a plus besoin de bloquer des ports.
- **Utilisation du ECMP (Equal-Cost Multi-Pathing) :** Grâce au routage dynamique (OSPF ou BGP), tous les liens entre les Leaf et les Spine sont **actifs simultanément**. Le trafic est réparti intelligemment sur l'ensemble des câbles disponibles, multipliant ainsi la bande passante réelle.
- **Latence déterministe :** Dans un réseau Spine-Leaf, n'importe quel serveur est situé à exactement **deux sauts** (2 hops) de n'importe quel autre serveur de l'infrastructure. Cette prévisibilité est essentielle pour les performances des applications distribuées.
- **Scalabilité horizontale :** Si nous avons besoin de plus de bande passante, il suffit d'ajouter un switch Spine. Si nous avons besoin de plus de ports serveurs, nous ajoutons un switch Leaf. Le réseau s'étend sans jamais remettre en cause l'existant.

### D. Synthèse du choix architectural
Le modèle **Spine-Leaf** s'est imposé comme la référence absolue, du petit Homelab aux infrastructures des géants du Cloud. Pour notre projet, ce choix garantit que nos services (NextCloud, GitLab, LDAP, etc.) ne seront jamais limités par une congestion réseau interne ou par les lenteurs de convergence d'un protocole obsolète comme le STP. Nous combinons ainsi le meilleur des deux mondes : un **Tier 3 robuste** pour la gestion des accès et du WAN, et un **Spine-Leaf agile** pour la performance de nos services critiques.

### E. Impact sur l'ecosystème (VXLAN et BGP EVPN)

L’adoption d’une topologie physique en **Spine-Leaf** impose, par sa nature même, une rupture avec les méthodes de commutation traditionnelles. En basculant sur une infrastructure où le routage de **Niveau 3 (L3)** descend jusqu'au sommet de chaque rack (Top-of-Rack), nous créons ce que l'on appelle un **Underlay** robuste et performant, utilisant le protocole **ECMP (Equal-Cost Multi-Pathing)** pour exploiter toute la bande passante disponible. Cependant, cette rigidité du routage IP pose un défi majeur : comment maintenir la flexibilité du **Niveau 2 (L2)**, indispensable pour le déplacement de machines virtuelles (vMotion) ou le clustering de serveurs, sans réintroduire les limitations du Spanning-Tree ? La réponse réside dans la mise en place d'un **Overlay**, un réseau virtuel superposé à notre infrastructure physique, s'appuyant sur le couple indissociable **VXLAN** et **BGP EVPN**.

Le **VXLAN (Virtual Extensible LAN)** agit comme le bras armé de cette virtualisation réseau en opérant une **encapsulation MAC-in-UDP**. Concrètement, le commutateur Leaf encapsule la trame Ethernet d'un serveur dans un paquet IP pour le transporter à travers la structure Spine-Leaf comme s'il s'agissait d'un simple flux routé. Cette technologie fait sauter le verrou historique des 4096 VLANs en introduisant le **VNI (VXLAN Network Identifier)** codé sur 24 bits, offrant ainsi la possibilité de segmenter jusqu'à **16 millions de réseaux virtuels**. Pour les serveurs connectés sur différents racks, le réseau physique devient totalement transparent : ils ont l'illusion d'être branchés sur le même switch de niveau 2, alors qu'ils sont séparés par plusieurs bonds de routage complexes. C’est cette abstraction qui permet de construire un Datacenter agile, capable de s'étendre horizontalement sans jamais fragmenter les domaines de diffusion.

Cependant, l'encapsulation seule ne suffit pas ; il faut un "cerveau" capable de cartographier en temps réel la position de chaque adresse MAC et IP sur l'ensemble de la Fabric. C'est ici qu'intervient **BGP EVPN (Ethernet VPN)**, agissant comme le **Control Plane** (plan de contrôle) de notre infrastructure. Plutôt que de laisser les switches apprendre les adresses par inondation de trafic (le fameux mécanisme de "Flood and Learn" qui sature les réseaux classiques), EVPN utilise les extensions du protocole **BGP** pour distribuer intelligemment les informations d'accessibilité. Chaque Leaf informe ses voisins des machines qui lui sont raccordées via des annonces BGP. Il en résulte une réduction drastique du trafic de broadcast, d'unknown unicast et de multicast (BUM), tout en permettant une convergence ultra-rapide en cas de panne. L'association du **VXLAN** pour le transport et de **BGP EVPN** pour l'intelligence transforme alors le réseau en une véritable **Fabric programmable**.

La mise en œuvre d'un tel écosystème n'est pas sans contraintes techniques et nécessite une attention particulière sur deux points critiques : la gestion du **MTU (Maximum Transmission Unit)** et l'automatisation. L'ajout des en-têtes VXLAN augmentant la taille des paquets, il devient impératif de configurer les **Jumbo Frames** sur l'Underlay pour éviter toute fragmentation délétère pour les performances. De plus, la complexité inhérente à la configuration de BGP EVPN sur chaque nœud rend l'erreur humaine probable en cas de gestion manuelle. C'est pourquoi cette architecture appelle naturellement l'usage d'outils d'automatisation comme **Ansible**. En résumé, le passage au **VXLAN / BGP EVPN** n'est pas un simple choix optionnel, mais le fondement indispensable d'un Datacenter moderne, garantissant une séparation parfaite entre l'infrastructure physique stable et les services virtuels dynamiques.

### F. Spécificités des Border Leafs et Segmentation Multi-tenant

L'évolution d'une topologie **Spine-Leaf** standard vers une architecture de production nécessite l'introduction d'un rôle spécifique : le **Border Leaf**. Si les **ToR Leafs** (Top-of-Rack) ont pour mission primaire la connectivité des serveurs et l'encapsulation initiale des flux, le Border Leaf agit comme la passerelle de sortie de la Fabric. L'ajout de cette couche dédiée ne répond pas à une simple volonté de hiérarchisation, mais à une nécessité technique de séparation des plans de transfert. En isolant les fonctions de "bordure" (connexion vers les Firewalls, le routeur PE ou le LAN Tier 3) sur des équipements spécifiques, nous préservons l'intégrité des **Spines**. Ces derniers conservent leur rôle de commutateurs de transit haute performance, totalement agnostiques aux services, ne gérant que le routage de l'**Underlay** (généralement via OSPF ou IS-IS) sans jamais avoir à traiter les politiques de sécurité ou les tables de routage complexes des couches supérieures.

Cette architecture est le socle indispensable à la **scalabilité horizontale** de l'infrastructure. Dans un scénario de croissance où le nombre de **ToR Leafs** augmente pour accueillir de nouveaux hyperviseurs, la pression sur la bande passante peut nécessiter l'ajout de nouveaux **Spines**. Grâce à la séparation des rôles, cette extension du cœur de réseau se fait sans aucune interruption des services de bordure. Le Border Leaf maintient une connectivité constante vers l'extérieur pendant que la capacité de commutation interne de la Fabric s'accroît. Cependant, cette flexibilité impose une rigueur absolue dans la gestion du **Plan de Contrôle (Control Plane)**. Pour que la segmentation soit effective, les **VRF (Virtual Routing and Forwarding)** doivent être instanciées de manière cohérente à chaque extrémité des tunnels **VXLAN**. Une VRF définie sur un ToR Leaf pour isoler le trafic d'une DMZ doit impérativement être présente sur le Border Leaf correspondant. C'est cette continuité de la VRF, propagée via **BGP EVPN**, qui garantit que le contexte de sécurité et l'étanchéité des flux sont maintenus lors de la désencapsulation du paquet en sortie de Fabric.

L'implémentation de ce modèle introduit néanmoins des contraintes d'ingénierie non négligeables, notamment en ce qui concerne le **mappage des Route Targets (RT)** et des **Route Distinguishers (RD)**. Chaque zone isolée (DMZ, Production, Administration) fonctionne comme un système de routage indépendant. Le Border Leaf devient alors le point de convergence où ces VRF sont présentées physiquement ou logiquement aux Firewalls **PfSense**. Pour maintenir une sécurité périmétrique stricte, on utilise souvent une topologie en "U-Turn" : tout flux devant passer d'une VRF à une autre (par exemple, un serveur web en DMZ interrogeant une base de données en zone interne) est forcé de quitter la Fabric via le Border Leaf, d'être inspecté par le Firewall, puis d'être ré-injecté dans la Fabric vers la destination finale. Ce mécanisme garantit qu'aucune communication inter-zone ne peut court-circuiter les règles de sécurité établies.

Enfin, la maîtrise de cette architecture exige une surveillance accrue de la **MTU (Maximum Transmission Unit)** sur l'ensemble du chemin réseau. L'encapsulation VXLAN ajoutant un overhead de 50 octets, le réseau d'Underlay (Spines et liens d'interconnexion) doit être configuré en **Jumbo Frames** pour éviter la fragmentation des paquets, laquelle dégraderait lourdement les performances CPU des Leafs. Bien que la configuration manuelle de ces paramètres sur chaque ToR et Border Leaf représente une charge opérationnelle complexe, elle est la condition _sine qua non_ pour obtenir une infrastructure **souveraine, multi-tenant et résiliente**, capable de supporter des charges de travail critiques avec une latence déterministe, indépendamment de la complexité des services hébergés.