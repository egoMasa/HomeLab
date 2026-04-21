# Sommaire

1. Présentation & contexte & environnement
2. Cahier des charges
3. Architecture réseau et justifications
	1. Approche méthodologique
	2. Modèles hiérarchiques
	3. Segmentation en zones
	4. Architecture de la zone Edge
	5. Architecture de la zone LAN (Tier 3)
	6. Architecture de la zone Datacenter (Spine-Leaf)
	7. Placement des firewalls
	8. Plan d'adressage IPv4
	9. Plan VLAN, VNI et VRF
	10. Cas particulier des flux datacenter (intra-VRF vs inter-VRF)
	11. Synthèse de l'architecture retenue
4. Déploiement et configuration des services
	1. Service 1 : Notion/Concept, étude de marché, sélection, configuration, sécurisation & administration
	2. Service 2 : idem
	3. ...
5. Mise en conformité des SI

---

# 1) Présentation du projet

Le but de ce projet est de mettre en place une infrastructure réseau et système complète dans un but d'apprentissage et se préparer au plus près des solutions et infrastructure rencontrés dans les missions d'un ingénieur réseau/système/cybersécurité. Le but est d'avoir à la fin une vision claire, globale et complète de à quoi ressemble en réalité, dans le détails les infrastructures des grandes/moyennes entreprises qui hébergent eux-mêmes leur systèmes d'informations (on-premise). L'autre intérêt de ce projet est de comprendre en profondeur les notions et systèmes déployés dans les infrastructures modernes dont les équipes s'occupent au quotidien. Nous en profiterons pour définir l'ensemble des solutions et systèmes primordiales à déployer dans un environnement réel, connaitre leur concept et les besoins auxquels ils répondent et connaitre les différentes solutions disponibles sur le marché.

Il est important de noter que j'ai volontairement choisi une approche open-source gratuite déployable on-premise afin de se détacher des solutions propriétaires payantes et montrer qu'il est possible d'avoir une infrastructure complète, performante et viable en étant souverain de ses données et de dépendre de personnes ou du moins limiter les dépendances techniques externes et politique.

Nous justifierons chaque choix en expliquant en détails les différentes choix possibles autant en terme de solutions, de configuration mais aussi d'infrastructure réseau, nous verrons que il n'existe aucunes maquettes parfaite mais que l'on peut optimiser et mettre en application des bonnes pratiques nous permettant de nous rapprocher d'une infrastructure complète et redondante. 

Concernant la façon de mettre en place ce projet nous utiliserons le logiciel de virtualisation GNS3 car il nous permettra d'importer l'ensemble des équipement réseau et virtualiser l'ensemble des serveurs. Nous utiliserons un PC central sous Linux (Ubuntu Desktop) afin de n'avoir aucunes limitations sur les applicances qui necessitent KVM et QEMU sans hyperviseur de type 2 entre. Donc pas de GNS3 VM ou de Windows comme host, 100% Linux en bare-metal. 


# 2) Cahier des charges

Cette section définit les briques matérielles et logicielles retenues pour le projet. Elle se décompose en quatre volets : l'inventaire des équipements réseau et système, le catalogue des services applicatifs prévus, un tableau de synthèse des protocoles et fonctionnalités par catégorie d'équipement, et les contraintes transversales qui structurent les choix. Les justifications détaillées des topologies et des patterns sont traitées en section 3 (Architecture réseau).

## 2.1) Inventaire des équipements

La maquette matérialise 26 équipements GNS3 couvrant l'ensemble des zones fonctionnelles du SI simulé. Les images logicielles retenues sont celles qui offrent le meilleur compromis entre représentativité réelle, empreinte mémoire acceptable sur le host et support des fonctionnalités ciblées par le projet. Le tableau ci-dessous liste chaque équipement avec la configuration retenue.

| # | Équipement | Qté | Zone | ISO / Image | Version | RAM | vCPU | Disque principal | Disque additionnel | Interfaces | Console |
|---|---|---:|---|---|---|---:|---:|---|---|---:|---|
| 1 | NAT Cloud | 1 | Edge | GNS3 builtin (libvirt) | — | — | — | — | — | 1 | — |
| 2 | SW-WAN | 1 | Edge | GNS3 Ethernet Switch builtin | — | — | — | — | — | 3 ports | — |
| 3 | Firewall central HA | 2 | Edge / central | pfSense CE (ou OPNsense) | pfSense 2.8.x | 2 Go | 2 | 20 Go | — | 6 | vnc (install) / telnet (run) |
| 4 | Switch Core LAN | 2 | LAN | Cisco IOSvL2 | 15.2(20200924:215240) | 1 Go | 1 | qcow2 fourni | — | 16 | telnet |
| 5 | Switch Distribution LAN | 4 | LAN | Cisco IOSvL2 | 15.2(20200924:215240) | 1 Go | 1 | qcow2 fourni | — | 16 | telnet |
| 6 | Switch Access LAN | 4 | LAN | Cisco IOSvL2 | 15.2(20200924:215240) | 768 Mo | 1 | qcow2 fourni | — | 16 | telnet |
| 7 | Spine DC | 2 | DC | Arista vEOS-lab | 4.35.3.1F | 2 Go | 1 | vEOS64-lab-4.35.3.1F.qcow2 | Aboot-veos-serial-8.0.2.iso | 13 | telnet |
| 8 | Leaf DC | 3 | DC | Arista vEOS-lab | 4.35.3.1F | 2 Go | 1 | vEOS64-lab-4.35.3.1F.qcow2 | Aboot-veos-serial-8.0.2.iso | 13 | telnet |
| 9 | Hyperviseur Proxmox | 3 | DC | Proxmox VE | 9.1 | 16 Go | 6 | 300–500 Go | — | 4–6 | vnc |
| 10 | Poste client Windows | 1 | LAN | Win 11 LTSC IoT | 24H2 | 4 Go | 2 | 30 Go | — | 1 | vnc |
| 11 | Poste client Linux | 1 | LAN | Xubuntu | 24.04 | 2 Go | 2 | 15 Go | — | 1 | vnc |
| 12 | Postes simulés VPCS | 2 | LAN | VPCS | — | — | — | — | — | 1 | telnet |

Quelques précisions complémentaires sur certains choix.

**pfSense CE 2.8.x plutôt qu'OPNsense.** Les deux solutions dérivent du même ancêtre m0n0wall et se valent fonctionnellement. pfSense a une base utilisateur plus large, une documentation plus abondante, et un écosystème de packages plus riche (pfBlockerNG en particulier). OPNsense est plus moderne dans son interface, avec une gouvernance plus saine côté communauté (suite au rachat de Netgate, pfSense a perdu en ouverture). Le choix pfSense est pragmatique : le savoir-faire et la documentation en ligne sont plus abondants, ce qui compte dans un contexte pédagogique. Basculer sur OPNsense reste possible sans repenser l'architecture.

**Arista vEOS-lab plutôt qu'Arista cEOS ou Nokia SR OS.** Pour la fabric DC, il fallait une image supportant VXLAN et BGP EVPN sans limitation. vEOS-lab est téléchargeable gratuitement depuis le compte Arista et supporte l'ensemble des features EVPN. cEOS nécessite un hôte Docker dédié et une gymnastique de lancement peu adaptée à GNS3. Les alternatives commerciales (Cisco Nexus 9000v, Juniper vQFX) existent en lab mais demandent des licences d'évaluation limitées dans le temps.

**Cisco IOSvL2 plutôt que vEOS pour le LAN.** Le LAN aurait pu être également basé sur vEOS pour bénéficier du MLAG, mais deux raisons ont conduit à garder IOSvL2. D'abord, la majorité des formations et certifications réseau enseignent encore la syntaxe Cisco IOS, donc l'utilisateur apprend une syntaxe plus directement réutilisable en entreprise. Ensuite, garder un mix Cisco/Arista reflète la réalité des SI d'ETI où cohabitent souvent plusieurs générations et plusieurs constructeurs. La limite assumée est l'absence de MLAG : on retombe sur du VRRP classique + STP actif sur les uplinks Access, ce qui est parfaitement défendable et reste le pattern le plus répandu en ETI non-DC-native.

**Trois hyperviseurs Proxmox dont deux en cluster.** HV-1 et HV-2 forment un cluster actif-actif et hébergent VM PROD + VM DMZ + VM ADMIN. HV-3 est standalone, dédié à l'administration pure : il héberge Proxmox Backup Server, Proxmox Datacenter Manager, et joue le rôle de QDevice (troisième voix Corosync) pour garantir le quorum du cluster HV-1/HV-2. Ce découpage reflète le pattern "compute productif + compute management séparé" qu'on trouve dans la plupart des DSI structurées.

**Pas de routeur CE dans la maquette initiale.** Le Cisco IOSv est conservé dans les templates GNS3 mais n'est pas déployé. Il serait réintégré uniquement pour simuler le Scénario 2 (single-homing avec eBGP) ou le Scénario 3 (multi-homing), qui ne sont pas pertinents dans la première version de la maquette. Voir section 3.4 pour le détail.

## 2.2) Catalogue des services applicatifs

Les services applicatifs sont la matière de la Phase 2. On les liste ici à titre indicatif pour que le lecteur ait la vision de ce qui sera hébergé sur les hyperviseurs, sans rentrer dans l'arbitrage solution par solution qui interviendra plus tard.

| Service | Placement | Solutions open-source envisagées | Besoin couvert |
|---|---|---|---|
| DNS Interne | DC (VRF-PROD) | Unbound, BIND9 | Résolution des noms internes non externalisés |
| DHCP Central | DC (VRF-PROD) | ISC Kea + Stork | Attribution dynamique des adresses IP sur tout le SI |
| Annuaire LDAP | DC (VRF-PROD) | Samba ADDC, OpenLDAP | Stockage des comptes, groupes, droits et permissions |
| RADIUS | DC (VRF-ADMIN) | FreeRADIUS | Authentification 802.1X, VPN, admin équipements |
| SSO / IdP | DC (VRF-ADMIN) | Keycloak | Authentification unique par tokens pour les services |
| Supervision de performance | DC (VRF-ADMIN) | Zabbix, Prometheus, Grafana | État et métriques des équipements et services |
| SOC / SIEM / XDR | DC (VRF-ADMIN) | Wazuh, Graylog + Shuffle, OpenCTI, CrowdSec | Supervision de sécurité, remédiation active |
| Bastion d'administration | DC (VRF-ADMIN) | Apache Guacamole | Portail d'accès admin aux équipements |
| ITSM / Ticketing / Inventaire | DC (VRF-ADMIN) | GLPI | Inventaire, tickets, gestion des biens |
| Stockage cloud | DC (VRF-PROD) | NextCloud | Dépôt de fichiers utilisateurs |
| NTP interne | DC (VRF-PROD) | chrony | Synchronisation horaire de tout le SI |
| CI/CD et dépôt de code | DC (VRF-ADMIN) | GitLab, Jenkins | Environnement de développement et dépôt |
| Gestion de projet | DC (VRF-ADMIN) | Redmine | Suivi de projet, tâches, diagrammes |
| Reverse Proxy | DMZ (VRF-DMZ) | HAProxy | Publication HTTPS, load balancing L7 |
| Proxy sortant | DMZ (VRF-DMZ) | Squid, TinyProxy | Contrôle des flux sortants des usagers |
| WAF | DMZ (VRF-DMZ) | BunkerWeb | Protection des applications web exposées |
| SMTP Inbound / Outbound | DMZ (VRF-DMZ) | Postfix, Rspamd | Mail entrant et sortant |
| Serveur web exposé | DMZ (VRF-DMZ) | Nginx, Apache | Sites vitrines et applicatifs publics |
| DNS Public | DMZ (VRF-DMZ) | BIND9, NSD | Résolution autoritaire des zones publiques |
| VPN nomade | FW (Edge) | WireGuard (retenu), OpenVPN (alternatif) | Accès distant au SI pour télétravailleurs |
| Automatisation | DC (VRF-ADMIN) | Ansible, AWX | Exécution de playbooks sur les serveurs |
| IPS/IDS | FW (Edge) | Suricata | Détection et blocage des menaces réseau |
| PKI interne | DC (VRF-ADMIN) | HashiCorp Vault, Smallstep | Émission de certificats internes |
| Gestion de secrets | DC (VRF-ADMIN) | HashiCorp Vault | Coffre-fort de secrets applicatifs |

Chaque solution retenue fera l'objet d'un arbitrage détaillé en début de Phase 2, selon le canevas annoncé dans le prompt système : notion, état du marché, sélection argumentée, dimensionnement, installation, sécurisation, administration, intégration avec le reste du SI.

## 2.3) Synthèse des protocoles et fonctionnalités par catégorie d'équipement

Le tableau ci-dessous récapitule, catégorie par catégorie, les éléments techniques à configurer sur chaque type d'équipement. Il complète les tableaux par zone donnés plus loin en section 3, en offrant une vue consolidée.

### 2.3.1) Switches d'accès LAN (ACC-1 à ACC-4)

| Domaine | Protocole / Feature | Rôle |
|---|---|---|
| VLAN | Ports access + voice VLAN | Affectation des postes et téléphones IP sur leurs VLAN respectifs |
| Uplink | Trunk 802.1Q (2 × uplinks non-agrégés) | Liens vers les deux Distribution du bloc, STP actif pour bloquer le chemin redondant |
| STP | MSTP + PortFast + BPDU Guard | Évite les boucles, accélère la convergence, bloque les switches parasites |
| Sécurité L2 | Port Security | Limitation du nombre de MAC apprises par port (typiquement 2 pour PC + ToIP) |
| Sécurité L2 | DHCP Snooping | Rejet des DHCP offers non autorisés, construction de la binding table |
| Sécurité L2 | Dynamic ARP Inspection | Appuyé sur DHCP Snooping, bloque l'ARP spoofing |
| Sécurité L2 | Storm Control | Limite les tempêtes broadcast/multicast/unknown-unicast |
| AAA | 802.1X + MAB | Authentification par port avec RADIUS, MAB pour équipements sans supplicant |
| Admin | SSH v2, AAA local + RADIUS fallback | Accès CLI sécurisé |
| Supervision | Syslog + SNMPv3 + NTP + DNS client | Intégration aux services centraux |

### 2.3.2) Switches de distribution LAN (DIST-1 à DIST-4)

| Domaine | Protocole / Feature | Rôle |
|---|---|---|
| VLAN | SVI utilisateurs (10, 20, 30, 40, 41, 42, 50, 90) | Interfaces IP de chaque VLAN utilisateur |
| VRRP | VIP partagée par VLAN, priorité alternée pair/impair | Redondance de passerelle, équilibrage entre Dist-A et Dist-B |
| Trunk | LACP peer-link 2 × liens vers l'autre Dist du bloc | Synchronisation logique intra-bloc (cluster LAN) |
| Uplink | L3 routé /31 × 2 vers les Core | Uplinks avec OSPF area 0 |
| STP | MSTP (priorité élevée, secondaire au Core) | Topologie L2 saine sur les trunks vers Access |
| Trunk vers Access | Trunk 802.1Q vers les 2 Access du bloc | Portage des VLAN utilisateurs vers l'Access |
| DHCP | DHCP Relay (`ip helper-address`) sur chaque SVI | Relais vers le serveur DHCP central en VRF-PROD |
| Routage | OSPF area 0 + redistribute connected | Annonce des subnets utilisateurs vers le Core |
| Admin | SSH v2, AAA, Syslog, SNMPv3, NTP, DNS client | Intégration services centraux |

### 2.3.3) Switches cœur LAN (CORE-1, CORE-2)

| Domaine | Protocole / Feature | Rôle |
|---|---|---|
| Trunk | LACP peer-link 2 × liens Core-Core | Synchronisation logique (cluster Core) |
| Uplink vers FW | L3 routé /31 × 2 vers chaque FW | Full-mesh 4 liens FW↔Core, OSPF area 0 |
| Downlink vers Dist | L3 routé /31 × 4 vers chaque Dist | Full-mesh 8 liens Core↔Distribution, OSPF area 0 |
| Routage | OSPF area 0 | IGP commun avec Distribution, FW et Spine |
| STP | MSTP (root primary) | Rôle root STP pour les VLAN éventuels traversant (pas en nominal) |
| SVI | Aucun SVI utilisateur | Le Core ne termine aucun VLAN utilisateur |
| Admin | SSH v2, AAA, Syslog, SNMPv3, NTP, DNS client | Intégration services centraux |

### 2.3.4) Spine DC (SPINE-1, SPINE-2)

| Domaine | Protocole / Feature | Rôle |
|---|---|---|
| Underlay | OSPFv2 area 0 | Résolution des loopbacks fabric, ECMP |
| Overlay | iBGP EVPN (Route Reflector) | Distribution des routes MAC/IP entre Leaf |
| Transit | ECMP 100 % actif sur tous les chemins | Répartition du trafic Leaf↔Leaf |
| MTU | Jumbo frames MTU 9214 | Absorption de l'overhead VXLAN (50 octets) |
| BFD | Détection rapide de panne (optionnel) | Convergence sub-seconde sur rupture de lien |
| Rôle Border | Trunk 802.1Q vers FW avec sous-interfaces par VRF | Handoff des VRF vers le FW via VLAN 201-204 |
| VRF | default (underlay) + PROD + DMZ + ADMIN + MGMT | Tables de routage isolées |
| Anycast | IP anycast partagée entre Spine-1 et Spine-2 sur les sous-interfaces FW | Redondance du handoff côté FW |
| Admin | SSH + AAA, Syslog, SNMPv3, NTP, DNS, Management1 dans VRF-MGMT | Accès admin in-band sécurisé |

### 2.3.5) Leaf DC (LEAF-1, LEAF-2, LEAF-3)

| Domaine | Protocole / Feature | Rôle |
|---|---|---|
| Underlay | OSPFv2 area 0 | Annonce de la loopback vers les Spine |
| Overlay | iBGP EVPN (RR client) | Client des Spine, annonce MAC/IP des VM locales |
| VTEP | VXLAN encapsulation (UDP 4789) | Encapsulation/décapsulation des trames overlay |
| VNI L2 | 1 VNI par VLAN étendu dans la fabric | Segment L2 virtuel (convention 10000 + VLAN) |
| VNI L3 | 1 VNI par VRF | Transport du routage inter-subnet dans la VRF |
| VRF | PROD, DMZ, ADMIN, MGMT | Tables de routage par tenant |
| IRB | Distributed Symmetric IRB | Routage inter-subnet distribué sur chaque Leaf |
| Anycast Gateway | IP de passerelle partagée entre tous les Leaf | Mobilité VM sans changement de passerelle |
| ARP suppression | EVPN ARP/ND suppression | Réduction du trafic BUM |
| Trunk HV | Trunk 802.1Q vers hyperviseur Proxmox | Portage des VLAN des zones hébergées |
| Sécurité L2 | Storm Control, BPDU Guard, MAC move threshold | Protection contre boucles et attaques |
| Admin | Management1 dans VRF-MGMT, SSH + AAA, Syslog, SNMPv3 | Accès admin in-band |

### 2.3.6) Firewall pfSense (FW-1, FW-2)

| Domaine | Protocole / Feature | Rôle |
|---|---|---|
| HA | CARP + pfsync + XMLRPC Config Sync | VIP partagées, synchronisation d'états et de config |
| Lien HA | Interface em0 dédiée en 10.0.255.0/30 | Trafic pfsync isolé |
| Routage | OSPF via FRR (package) | Adjacences vers Core LAN et Spine DC |
| Routage | Default route vers NAT Cloud | Sortie Internet simulée |
| NAT | Outbound NAT (SNAT) en mode manuel | Masquerading des réseaux internes vers les VIP publiques |
| NAT | Inbound NAT / DNAT | Publication de services DMZ (si nécessaire) |
| VIP | CARP sur chaque interface data + VIP publiques sur em1 | HA et exposition des services publics |
| Handoff VRF | Sous-interfaces 802.1Q em2/em3.201-204 | Présentation des 4 VRF sous forme de 4 VLAN transit |
| Filtrage | Rules par interface + Aliases + Floating rules | Stateful, deny-by-default, granulaire |
| Anti-spoof | Anti-bogons + Anti-spoof | Blocage des IP privées et bogons en entrée WAN |
| VPN | WireGuard (retenu) ou OpenVPN (alternatif) | VPN nomade bindé sur VIP CARP 192.168.122.10 |
| IPS/IDS | Suricata + pfBlockerNG | Inspection profonde, rulesets ET Open, filtrage géographique |
| DNS | Unbound en mode forwarder | Résolution locale, forwarders vers DNS interne et public |
| Admin | HTTPS webGUI (bastion uniquement) + SSH clé | Accès admin restreint |
| Supervision | Syslog export, SNMPv3, Login Protection | Intégration services centraux, anti-bruteforce |

### 2.3.7) Hyperviseurs Proxmox (HV-1, HV-2 cluster / HV-3 standalone)

| Domaine | Protocole / Feature | Rôle |
|---|---|---|
| Interface physique | `ens3` en trunk 802.1Q | Unique lien vers le Leaf, porte toutes les zones hébergées |
| Bridge management | `vmbr0` VLAN-aware dans VLAN 999 | Interface admin de l'hyperviseur (SSH, webGUI 8006) |
| Bridges data | `vmbr-prod`, `vmbr-dmz`, `vmbr-admin` VLAN-aware | Un bridge dédié par zone pour isoler les domaines L2 |
| Cluster (HV-1, HV-2) | Corosync + HA Manager | Synchronisation, failover VM entre nœuds |
| QDevice (HV-3) | corosync-qnetd | Troisième voix de quorum pour HV-1/HV-2 |
| PBS (sur HV-3) | Proxmox Backup Server en VM | Sauvegarde déduppliquée des VM du cluster |
| PDM (sur HV-3) | Proxmox Datacenter Manager en VM | Administration centralisée multi-cluster |
| Sécurité | SSH par clé, Fail2ban, pve-firewall | Durcissement plan d'administration |
| Supervision | Syslog, NTP, DNS client | Intégration services centraux |

### 2.3.8) Équipements d'infrastructure simple (SW-WAN)

| Domaine | Protocole / Feature | Rôle |
|---|---|---|
| Type | GNS3 Ethernet Switch builtin | Pas de configuration CLI (équipement intégré GNS3) |
| Topologie | 3 ports access dans le même segment L2 | NAT Cloud + FW-1 + FW-2 |
| Limitation | Pas de SNMP, pas de port-security, pas de MSTP | Remplaçable par un IOSvL2 si ces fonctions sont souhaitées |

## 2.4) Contraintes transversales

Trois contraintes structurent l'ensemble des choix :

**Open-source prioritaire absolu.** Aucune solution propriétaire payante n'est retenue comme composant de l'infrastructure. Les solutions commerciales sont mentionnées uniquement à titre comparatif. Ce choix répond à l'objectif de souveraineté et de pédagogie : montrer qu'une infrastructure complète et crédible peut être bâtie sans dépendre d'éditeurs.

**On-premise exclusif.** Aucun service cloud public n'est utilisé comme brique de l'infrastructure. La sortie Internet est simulée par le NAT Cloud de GNS3 uniquement pour permettre les mises à jour logicielles, pas pour y déporter des fonctions.

**Virtualisation native KVM/QEMU.** GNS3 tourne en bare-metal sur un host Ubuntu Desktop, sans GNS3 VM ni hyperviseur de type 2 intermédiaire. Cela garantit un accès natif à KVM/QEMU et évite les limitations d'imbrication de virtualisation sur les appliances gourmandes (pfSense, Proxmox).

# 3) Architecture réseau

Cette section pose l'architecture réseau complète du projet. Elle précède toute configuration d'équipement et toute installation de service : on fixe ici la topologie, les rôles de chaque équipement, les modèles retenus par zone et les compromis assumés. L'objectif est que le lecteur comprenne **pourquoi** l'infrastructure ressemble à ce qu'elle est avant de voir **comment** elle est configurée.

La démarche suivie est systématique. Pour chaque zone, on présente d'abord le concept et les modèles courants du marché, on liste les options possibles avec leurs tradeoffs, on justifie le choix retenu, et on acte les simplifications éventuelles imposées par le contexte maquette. Le parti pris est celui d'une **entreprise moyenne à grande hébergeant son SI en on-premise** : les choix penchent vers les patterns qu'on rencontre en ETI et grands comptes, non vers ceux d'une PME ou d'un hyperscaler.

## 3.1) Approche méthodologique

Avant d'entrer dans les zones, il faut poser quelques principes qui structurent l'ensemble du design.

**Pas de silver bullet.** Il n'existe pas d'architecture idéale universelle. Chaque choix relève d'un compromis entre résilience, performance, complexité, coût d'exploitation et empreinte matérielle. On justifie chaque décision en exposant ce qu'elle apporte et ce qu'elle coûte, plutôt qu'en invoquant des "bonnes pratiques" floues.

**Le réseau doit précéder les services.** On ne déploie pas un LDAP avant d'avoir décidé où il vit, quelles machines peuvent le joindre et avec quel IGP. Toute la Phase 1 du projet est consacrée à l'architecture pure, avant que la moindre VM applicative ne soit allumée. Cette contrainte force à anticiper les dépendances.

**Lab GNS3, mais configurations de production.** La maquette est virtualisée, mais on applique les mêmes standards qu'en environnement réel : segmentation stricte, redondance là où c'est justifié, sécurisation des protocoles de contrôle, documentation des choix. Les seules simplifications tolérées sont celles dictées par les limites logicielles des images utilisées (par exemple l'absence de MLAG sur Cisco IOSvL2) ou par un rapport coût/valeur défavorable à la taille de la maquette (par exemple le dual-homing des hyperviseurs, qu'on évoque mais qu'on n'implémente pas).

**Ce qui est simplifié est documenté.** Quand on retire un équipement ou qu'on omet une redondance pour alléger la maquette, on précise ce qui aurait été mis en production et pourquoi la simplification est acceptable ici. Le lecteur doit pouvoir extrapoler du lab vers le réel sans ambiguïté.

## 3.2) Les modèles hiérarchiques : un prérequis conceptuel

La conception d'une infrastructure réseau d'entreprise repose sur des modèles hiérarchiques éprouvés. Ces modèles segmentent le réseau en **couches logiques distinctes** où chaque couche porte une fonction précise, ce qui permet de passer d'un simple agrégat de câbles à une architecture déterministe où chaque flux est maîtrisé, de l'endpoint jusqu'au cœur.

On parle classiquement de modèles par **Tier** (niveau), avec une échelle qui va de 1 à 3 selon le nombre de couches distinctes. Le choix du Tier dépend avant tout de la taille de l'entreprise et du niveau d'exigence en résilience et en scalabilité.

### 3.2.1) Tier 1 — l'architecture à plat

Le Tier 1 représente le stade minimal. La distinction entre commutation et routage y est quasi-inexistante : tous les terminaux (postes, serveurs, imprimantes) sont raccordés à une unique couche de commutation, sans mécanisme de redondance structurante et avec peu ou pas de segmentation logique.

Ce modèle convient à des environnements très petits (TPE, labs domestiques, sites distants) où l'absence de trafic critique et la simplicité priment. Il est inacceptable en milieu professionnel dès qu'on dépasse quelques dizaines de postes, pour deux raisons :

1. **Absence d'isolation** : une tempête de broadcast, une boucle STP, ou la compromission d'un poste se propage à tout le réseau.
2. **Pas de redondance** : la moindre panne matérielle arrête toute l'activité.

On le mentionne ici pour complétude, mais il est hors périmètre du projet.

### 3.2.2) Tier 2 — le collapsed core

Le Tier 2 introduit la notion de hiérarchie en séparant deux fonctions :

- **Couche d'accès** qui raccorde les terminaux et gère les problématiques de niveau 2 (VLAN, sécurité de port, alimentation PoE).
- **Couche cœur/distribution fusionnée** ("collapsed core") qui assume simultanément le routage inter-VLAN, l'application des politiques de sécurité et le transit vers les autres zones.

C'est un modèle très répandu en PME et petite ETI (50 à 500 postes, site unique). Il offre un premier niveau de redondance en doublant la couche cœur et en raccordant chaque Access à deux cœurs. Sa limite apparaît dès que l'infrastructure grossit : la couche fusionnée devient un goulot d'étranglement, car elle doit agréger massivement les liens tout en portant les traitements L3 coûteux.

Le Tier 2 est techniquement propre et pédagogiquement simple. Il n'est pas retenu pour ce projet parce qu'il ne permet pas de démontrer la séparation cœur/distribution, qui est centrale dans les architectures d'ETI et de grands comptes.

### 3.2.3) Tier 3 — le modèle hiérarchique de référence

Le Tier 3 est le modèle retenu pour la zone LAN du projet. Il affine le Tier 2 en **isolant physiquement et logiquement trois fonctions** :

La **couche d'accès** reste dédiée à l'interaction avec les endpoints : affectation dynamique des VLAN, ports access, alimentation PoE, déploiement du 802.1X, filtrage MAC. Elle est conçue pour être simple, nombreuse et remplaçable.

La **couche de distribution** agit comme pivot. Elle agrège les nombreux switches d'accès, opère la commutation de niveau 3 (routage inter-VLAN, ACL), et porte les protocoles de redondance de passerelle (VRRP, HSRP). Elle isole les problèmes de la couche d'accès pour éviter qu'ils ne remontent jusqu'au cœur. C'est la couche où vivent les SVI utilisateurs, où on fait les annonces OSPF vers le haut, et où on concentre l'essentiel de l'intelligence réseau.

La **couche cœur** est l'autoroute du réseau. Elle ne filtre pas, ne fait pas de politique applicative : elle commute le trafic à très grande vitesse entre blocs de distribution et zones (LAN, DC, Edge). Sa séparation de la distribution garantit un temps de convergence rapide en cas de panne et un passage à l'échelle sans repenser le pivot.

Ce modèle est l'état de l'art pour les infrastructures d'entreprise à haute résilience. Il est le choix naturel pour simuler une infrastructure d'ETI.

## 3.3) Segmentation en zones

Une architecture hiérarchique performante n'a de sens que si elle s'accompagne d'une segmentation rigoureuse. L'objectif est d'appliquer le **principe du moindre privilège au niveau réseau** : chaque flux doit être justifié, identifié et contrôlé par un équipement de filtrage. On découpe ainsi le SI en zones fonctionnelles, chacune avec son rôle, son niveau de confiance et sa politique de sécurité.

### 3.3.1) Zone LAN — périmètre utilisateur

La zone LAN héberge toute l'activité interne de l'entreprise. On y retrouve la plus grande diversité d'endpoints : postes fixes, terminaux Wi-Fi, téléphones IP, caméras de surveillance, imprimantes, objets connectés. Structurellement, elle s'étend de la couche Access à la couche Distribution du Tier 3, sous un backbone Core commun avec les autres zones.

Elle se caractérise par un fort besoin de mobilité interne (DHCP dynamique, Wi-Fi roaming), par une authentification sur le port (802.1X + RADIUS) et par une isolation fine en VLAN selon la nature de l'équipement. La sécurité y est centrée sur l'utilisateur : on part du principe qu'un poste peut être compromis, et on s'assure que cette compromission ne se propage pas horizontalement.

### 3.3.2) Zone Edge (OUT / WAN) — frontière de sortie

Cette zone, parfois appelée "OUT" ou "Untrust", représente le point de contact avec l'extérieur. Elle héberge le peering avec le ou les FAI, les firewalls périmétriques, et les mécanismes de terminaison VPN. C'est ici que s'opère le NAT sortant pour les usagers internes, et que se termine le trafic entrant vers les services publics.

C'est une zone de haute vigilance. Toutes les tentatives d'intrusion, les scans, les attaques DDoS s'y concentrent. Elle est traitée en détail en section 3.4.

### 3.3.3) Zone DMZ — services exposés

La DMZ ("Demilitarized Zone") héberge les services que l'entreprise expose à Internet : serveurs web vitrine, reverse proxy, DNS public, serveurs SMTP inbound. Par convention, tout service joignable depuis Internet est en DMZ, jamais dans la zone interne.

La règle d'or est qu'un compromis sur un service DMZ ne doit **en aucun cas** permettre un rebond vers le cœur du SI. Cela impose un filtrage strict DMZ → Interne (en général : aucune initiation de connexion autorisée depuis la DMZ vers la prod interne, sauf exceptions explicites contrôlées).

### 3.3.4) Zone Datacenter — services internes

Le Datacenter héberge le "cerveau" du SI : annuaire, bases de données, ERP, stockage, SSO, supervision, ITSM, dépôts de code, backups. Cette zone est invisible depuis Internet et accessible uniquement depuis le LAN et la DMZ à travers des politiques de filtrage.

Elle est elle-même subdivisée logiquement en sous-zones fonctionnelles via des VRF distinctes, traitées en section 3.6.

### 3.3.5) Convergence DMZ/DC sur hyperviseurs mutualisés

Historiquement, la DMZ et le Datacenter étaient séparés **physiquement** : des serveurs dédiés DMZ, des serveurs dédiés prod interne, parfois des racks ou des salles distinctes. Ce modèle reste appliqué dans les infrastructures les plus sensibles (opérateurs d'importance vitale, certains environnements PCI-DSS à scope strict).

Dans ce projet, on fait le choix **moderne de la mutualisation compute avec isolation logique forte** : une seule fabric Spine-Leaf, un même pool d'hyperviseurs Proxmox hébergeant à la fois des VM DMZ et des VM internes, mais séparées par VRF + VLAN + firewall en U-turn. C'est le pattern qu'appliquent aujourd'hui la plupart des cloud providers privés et des DSI modernes.

Ce choix est assumé et documenté. Il implique :

- Un **durcissement obligatoire des hyperviseurs** (séparation du plan de management, SSH par clé, patch cadencé).
- Un **plan de management dédié**, dans une VRF Administration séparée, jamais mutualisée avec le plan data.
- Une **hygiène de bridges Proxmox** : chaque zone (DMZ, Prod, Admin) a son propre bridge dédié, pas de VM multi-zone sauf cas explicite documenté.
- Des **affinity rules préférentielles** pour limiter les domaines de panne tout en bénéficiant de la souplesse de migration.

L'alternative "compute séparé par zone" est évoquée dans le rapport comme pattern renforcé pour les infrastructures à criticité maximale. Elle n'est pas retenue ici.

## 3.4) Architecture de la zone Edge

### 3.4.1) Rôle et enjeux

La bordure WAN concentre les responsabilités les plus sensibles de l'infrastructure : l'exposition publique, la terminaison des VPN, l'essentiel du NAT et toute l'interface avec les fournisseurs d'accès. Sa conception a des répercussions directes sur le plan d'adressage, la politique de routage interne, et la résilience globale du SI face à une panne FAI.

Trois décisions structurent cette zone :
- Le **nombre de fournisseurs d'accès** (mono-FAI ou multi-FAI).
- Le **protocole de routage** avec le FAI (route par défaut statique ou eBGP).
- La **topologie interne** (présence ou non d'un routeur CE distinct du firewall, placement des IP publiques, modèle de NAT).

### 3.4.2) Terminologie de référence

| Terme | Définition |
|---|---|
| **PE** (Provider Edge) | Routeur de bordure du FAI. Toujours administré par le FAI. Hors du périmètre client. |
| **CE** (Customer Edge) | Routeur de bordure du client. Interface entre le SI et le PE. 100 % sous responsabilité du client. |
| **AS** (Autonomous System) | Numéro identifiant un domaine de routage unique dans BGP. Public (alloué par un RIR type RIPE) ou privé (64512–65534, 4200000000+). |
| **Bloc PA** (Provider Aggregatable) | Préfixe IP attribué par un FAI, non portable. |
| **Bloc PI** (Provider Independent) | Préfixe IP attribué au client par un RIR, portable. Indispensable pour du multi-homing propre. |
| **VIP** (Virtual IP) | IP configurée sur un équipement sans attache exclusive à une interface physique. |
| **CARP** (Common Address Redundancy Protocol) | Protocole de VIP HA de pfSense/OpenBSD. Équivalent fonctionnel de VRRP/HSRP. |

### 3.4.3) Notion fondamentale : propriété d'une IP vs connectivité physique

Un point conceptuel est à clarifier avant d'aller plus loin, car il conditionne tout le placement des IP publiques. Une méprise fréquente consiste à supposer qu'une IP publique doit nécessairement être configurée sur un équipement physiquement raccordé à Internet. **Ce n'est pas le cas.**

Une adresse IP est un **identifiant logique** : elle est attribuée à un équipement, qui la configure sur l'interface — physique ou virtuelle — de son choix. Une IP est dite "publique" parce qu'elle appartient à un bloc annoncé sur Internet par un AS, et que le reste de la table de routage globale sait donc comment l'atteindre. Le **type de média physique** sur lequel cette IP est configurée n'intervient nulle part dans cette définition.

Le corollaire est essentiel : **rien n'interdit de porter une IP publique sur un équipement dont toutes les interfaces physiques sont en adressage privé**, à condition qu'un routage correct amène les paquets jusqu'à lui. Les mécanismes techniques pour le faire s'appellent selon les contextes "loopback", "IP Alias", "secondary IP", "VIP CARP", "Virtual IP", "floating IP". Tous désignent la même idée : une IP logiquement rattachée à un système, sans contrainte d'interface physique exclusive.

Cette notion est le pilier du modèle retenu pour la zone Edge.

### 3.4.4) Le bloc public fourni par le FAI

Une offre Internet professionnelle standard fournit deux éléments IP distincts au client :

**Un petit préfixe de peering** (typiquement /30 ou /31) pour le lien point-à-point entre le PE du FAI et le CE du client. Ces IP, techniquement publiques, ne servent qu'à faire fonctionner l'adjacence L3 entre les deux routeurs. On n'héberge aucun service dessus.

**Un bloc client routé** (généralement /29, /28 voire plus selon le contrat) que le FAI route vers le client. C'est ce bloc dans lequel le client puise pour poser ses IP de services exposés : endpoints VPN, reverse proxy publics, MX entrants, DNS public. Le FAI ne sait rien de ce qu'il y a derrière — il sait seulement que ce bloc est joignable via le CE du client.

La séparation entre ces deux éléments est **contractuelle et systématique** sur les offres pro (Orange Business, SFR Business, Completel, Adista, Hexanet, opérateurs alternatifs). La pénurie IPv4 affecte le coût pour le FAI mais pas le principe d'allocation d'un bloc routé aux clients professionnels. Les offres qui ne fournissent qu'un /30 de peering sont des offres de transit BGP pur, destinées à des clients qui apportent déjà leur propre bloc PI.

Dans la modélisation du projet, on suppose une offre pro standard fournissant un /30 de peering et un /28 routé.

**Simplification retenue pour la maquette.** Dans une infrastructure réelle, le bloc public client est distinct du subnet de peering (par exemple un /28 public routé vers le client, séparé du /30 de peering). Dans cette maquette, pour éviter de devoir maintenir une route statique supplémentaire sur le host et garantir que les tests depuis un client externe fonctionnent de bout en bout sans bidouille réseau, on **fusionne les deux rôles dans le subnet libvirt `192.168.122.0/24`**. Les VIP publiques de services (VPN, reverse proxy, MX, DNS, API Gateway) sont portées directement dans cette plage, qui fait donc office à la fois de subnet de peering PE↔FW **et** de bloc public client. En production, cette conflation ne serait pas acceptable — on aurait bien deux plages distinctes comme décrit plus haut — mais elle est pédagogiquement neutre dans le cadre du lab tant que la distinction conceptuelle est rappelée.

### 3.4.5) Les trois scénarios de connectivité WAN

On distingue trois scénarios de référence, ordonnés par complexité croissante. Chacun impose des contraintes distinctes.

#### Scénario 1 — Route par défaut mono-FAI

```
Internet ── PE (FAI) ── FW HA ── LAN / DMZ / DC
```

Le CE est **fusionné** avec le FW ou absent : le FW joue directement le rôle de routeur de bordure. Aucun BGP, aucun AS, aucun peering dynamique. Une route par défaut statique pointe du FW vers l'IP du PE, et le FAI route le bloc client vers l'IP WAN du FW.

**Avantages** : simplicité extrême, faible empreinte, convergence instantanée, aucun prérequis administratif (pas d'AS, pas de PI).

**Limites** : aucune résilience inter-FAI, aucune maîtrise du routage, dépendance forte à l'opérateur (bloc PA non portable, changement de FAI = renumérotation complète des services publiés).

**Usage typique** : PME, sites distants, agences, contextes où l'Internet n'est pas business-critical.

#### Scénario 2 — Single-homing avec eBGP

```
Internet ── PE (FAI) ── CE ── FW HA ── LAN / DMZ / DC
```

Un **routeur CE dédié** est inséré entre le PE et le FW. Il porte la session eBGP avec le FAI, annonce le bloc client (idéalement PI) et reçoit soit la default route, soit une table partielle, soit la full-table Internet. Le FW reste responsable de la sécurité, des VPN et du NAT. Les IP publiques restent portées en VIP sur le FW, le CE étant IP-transparent.

**Avantages** : portabilité du préfixe si bloc PI (changement de FAI sans renumérotation), annonce active possible (communities, politiques sortantes), préparation au multi-homing, séparation claire des rôles CE (routage) / FW (sécurité).

**Limites** : bénéfice limité en mono-FAI (le SPOF reste le FAI unique), prérequis administratifs (AS public, bloc PI), dimensionnement CPU/mémoire du CE si full-table (~950k préfixes IPv4 en 2026).

**Usage typique** : état transitoire vers du multi-homing, ou exigence de portabilité / identification AS.

#### Scénario 3 — Multi-homing avec eBGP

```
Internet ── PE1 (FAI A) ── CE1 ──┐
                                  ├── FW HA ── LAN / DMZ / DC
Internet ── PE2 (FAI B) ── CE2 ──┘
                iBGP CE1 ↔ CE2
```

Deux CE, un par FAI, avec une session eBGP chacun. iBGP entre les deux CE pour partager les routes apprises. Le même bloc PI est annoncé aux deux FAI. Le FW HA est raccordé aux deux CE, et utilise ECMP via un IGP interne pour répartir le trafic.

**Avantages** : résilience Internet réelle (panne d'un FAI = bascule auto), continuité des IP publiques même en cas de changement d'opérateur, traffic engineering sortant et entrant via `local-preference`, `AS-path prepending`, MED, communities.

**Limites** : prérequis administratifs lourds (AS + PI obligatoires), coût (deux contrats FAI avec diversité de boucle locale, deux CE), complexité d'exploitation, convergence non-instantanée sans BFD.

**Usage typique** : ETI, grands comptes, secteurs régulés (banque, santé, énergie), infrastructures où la continuité Internet est business-critical.

#### Comparatif synthétique

| Critère | Scénario 1 | Scénario 2 | Scénario 3 |
|---|---|---|---|
| Nombre de FAI | 1 | 1 | 2+ |
| CE dédié | Non | Oui | Oui (×2) |
| Routage avec FAI | Default statique | eBGP | eBGP × N |
| Résilience Internet | Non | Non | Oui |
| AS requis | Non | Recommandé | Obligatoire |
| PI requis | Non | Recommandé | Obligatoire (IPv4) |
| Complexité | Faible | Modérée | Élevée |
| Usage typique | PME | Transitoire | ETI / GE |

### 3.4.6) Placement des IP publiques : VIP sur le firewall

Dans une architecture edge propre (Scénarios 2 et 3), **les IP publiques de services sont portées par le firewall sous forme de VIP**, alors même que toutes les interfaces physiques du FW sont en adressage privé vis-à-vis du CE. Cette configuration est rendue possible par :

- le **routage du CE**, qui sait envoyer les paquets destinés au bloc public vers le FW via le lien privé de transit,
- la **capacité du FW à se déclarer propriétaire** de ces IP via des VIP (CARP pour l'HA, ou IP Alias pour une IP non-redondée).

En pfSense, la configuration se fait via *Firewall → Virtual IPs*. Quatre types sont disponibles : **IP Alias** (IP supplémentaire sur le système), **CARP** (VIP partagée en HA entre deux pfSense), **Proxy ARP** (pour des scénarios rares), **Other** (IP déclarée sans réponse ARP). Le type retenu pour ce projet est **CARP**, pour préparer la haute disponibilité FW1/FW2.

Chaque IP publique de service est déclarée en VIP CARP et **bindée explicitement** par le démon correspondant (WireGuard, HAProxy, BIND, etc.). Ce binding explicite garantit qu'un service ne répond pas sur une IP qui ne lui est pas destinée.

Les VIP publiques effectivement allouées dans la maquette sont les suivantes (toutes dans le subnet libvirt `192.168.122.0/24`, conformément à la simplification retenue en 3.4.4) :

| IP                 | Service associé        | Rôle                                        |
| ------------------ | ---------------------- | ------------------------------------------- |
| `192.168.122.1`    | NAT sortant LAN/DC     | Source NAT des usagers et VM vers Internet  |
| `192.168.122.2`    | NAT sortant DMZ        | Source NAT dédié aux flux DMZ               |
| `192.168.122.10`   | VPN WireGuard          | Endpoint VPN pour télétravailleurs          |
| `192.168.122.11`   | Reverse proxy HTTPS    | Publication HTTPS (HAProxy derrière en DMZ) |
| `192.168.122.12`   | SMTP MX inbound        | Mail entrant                                |
| `192.168.122.13`   | DNS autoritaire public | Résolution des zones publiques              |
| `192.168.122.14`   | API Gateway            | Exposition d'APIs publiques                 |
| `192.168.122.200`  | VIP CARP WAN           | Adresse partagée d'exposition WAN de base   |
| `192.168.122.201`  | FW-1 physique WAN      | IP master (em1)                             |
| `192.168.122.202`  | FW-2 physique WAN      | IP backup (em1)                             |

Le cheminement d'un paquet VPN entrant illustre le mécanisme **tel qu'implémenté dans la maquette** :

```
1. PC-EXT → dst = 192.168.122.10:51820
2. Paquet émis sur le bridge libvirt virbr0
3. Le FW master (FW-1, advskew=0) répond à l'ARP pour 192.168.122.10
   via CARP sur em1
4. FW → reconnaît 192.168.122.10 comme VIP CARP locale → livre au démon WireGuard
```

**En infrastructure réelle**, le cheminement serait un peu plus long et impliquerait le CE :

```
1. Client Internet → dst = <IP publique VPN, ex. 203.0.113.10>:51820
2. Internet → FAI de l'entreprise (route BGP connue)
3. PE → CE via peering public (IP destination inchangée)
4. CE → table de routage : <bloc public client /28> via FW
5. CE → FW via lien privé de transit (IP destination toujours inchangée)
6. FW → reconnaît l'IP publique comme VIP locale → livre au démon
```

À aucun moment le CE n'héberge la VIP. À aucun moment il n'y a de NAT sur le CE. L'IP de destination reste publique de bout en bout, même en traversant un segment L2 privé. **L'adressage du lien et l'adressage des paquets qui y transitent sont décorrélés**.

### 3.4.7) Gestion du NAT

Le NAT source sortant (masquerading des usagers vers Internet) est réalisé **sur le firewall**, pas sur le CE. Trois raisons :

Le FW est déjà stateful pour le filtrage ; la table NAT réutilise la même infrastructure. Dupliquer cette logique sur le CE n'a aucun intérêt technique. La séparation des rôles reste propre (CE = routage L3 pur, FW = sécurité + NAT + VPN). Les logs de filtrage et de traduction restent corrélés au même endroit, ce qui simplifie l'analyse a posteriori.

Le NAT destination entrant est **inexistant dans le modèle routé** : les IP publiques de service sont directement portées par le FW en VIP, il n'y a rien à traduire. Le DNAT n'intervient qu'à l'intérieur du FW, si un service public doit être redirigé vers une VM interne derrière la DMZ (ex. reverse proxy HAProxy en DMZ qui reçoit le trafic HTTPS public).

L'anti-pattern à éviter est le **NAT sur le CE** (DNAT port par port vers les services internes). On le rencontre en PME et sur les box familiales, mais il rompt la séparation des rôles, complique la gestion des services multi-ports, et ne passe pas proprement certains protocoles (IPsec ESP sans port, GRE, SCTP). Il n'est pas retenu ici.

### 3.4.8) Choix retenu pour la maquette

La maquette initiale implémente une **version simplifiée du Scénario 1**, avec les adaptations suivantes :

- Un seul FAI simulé via le **NAT Cloud de GNS3**, qui se comporte comme un PE fictif et fournit une IP DHCP sur le /24 du host Ubuntu. Subnet imposé par libvirt : **`192.168.122.0/24`**, gateway `192.168.122.1`.
- **Pas de CE** dans la maquette initiale. Les firewalls se raccordent directement au NAT Cloud à travers un switch L2 de transit (SW-WAN), lui-même connecté au NAT Cloud par un lien unique.
- **Deux pfSense en HA CARP** avec lien pfsync dédié en `10.0.255.0/30` sur l'interface `em0`. Toutes les VIP publiques simulées sont des VIP CARP portées par le cluster.
- **IP physiques WAN** des FW en statique, dans la fourchette haute du /24 libvirt pour ne pas chevaucher la plage DHCP dynamique : **FW-1 en `192.168.122.201`**, **FW-2 en `192.168.122.202`**, **VIP CARP WAN en `192.168.122.200`**.
- **Sortie Internet fonctionnelle** depuis les VM internes pour permettre le `apt update`, les mises à jour Proxmox, les pull d'images. Le NAT Cloud GNS3 masquerade vers la vraie connectivité du host, on obtient donc un double NAT (FW → 192.168.122.x → IP hôte publique) qui est transparent pour le trafic sortant.

**Justification du choix** : le CE n'apporte aucune valeur pédagogique en l'absence de BGP. Une fois supprimé, on obtient une maquette plus simple sans rien perdre de ce qu'on veut démontrer côté FW (HA CARP, VIP, NAT, règles inter-zone).

**Évolution possible** : une phase ultérieure pourra réintégrer un CE Cisco IOSv entre le SW-WAN et le NAT Cloud, et configurer une session eBGP factice avec un second IOSv simulant un deuxième PE, pour démontrer le single-homing puis le multi-homing. Rien de ce qui est configuré dans la version initiale n'est invalidé par cette évolution.

### 3.4.9) Schéma de la zone Edge

```
                    Internet (host Ubuntu)
                            │
                    NAT Cloud GNS3
                    (192.168.122.1)
                            │
                       SW-WAN (builtin)
                       /        \
                      /          \
                  FW-1 ═══════ FW-2    ← HA CARP + pfsync sur em0
              .201 / .200         .202    (10.0.255.0/30)
                   │             │
                   └──────┬──────┘
                          │
                    vers Core LAN
                    vers Spine DC
```

## 3.5) Architecture de la zone LAN (Tier 3)

### 3.5.1) Palier d'entreprise simulée

La zone LAN est dimensionnée pour refléter un **site unique de taille moyenne à grande**, typiquement 300 à 1500 postes répartis sur plusieurs bâtiments ou zones fonctionnelles. Ce palier justifie le Tier 3 complet (trois couches distinctes) et la présence d'un Core séparé de la Distribution. En dessous de ce palier, un Tier 2 collapsed core serait suffisant ; au-dessus, on multiplierait les blocs de distribution ou on passerait à un Core géo-redondé multi-site.

### 3.5.2) Découpage en blocs de distribution

Le LAN est organisé en **deux blocs de distribution indépendants**, chacun composé d'une paire de switches de distribution formant un cluster logique. Chaque bloc porte ses propres switches d'accès et dessert sa propre population d'utilisateurs. Ce découpage permet plusieurs choses :

Il **isole les domaines de panne** entre blocs. Un problème sur un bloc (boucle L2, erreur de configuration VRRP, saturation) ne se propage pas à l'autre.

Il **représente concrètement une séparation fonctionnelle ou géographique** : bâtiment A vs bâtiment B, utilisateurs bureautiques vs zones sensibles (Wi-Fi + IoT + ToIP), plateaux de direction vs zones ouvertes.

Il **démontre le vrai rôle du Core** comme agrégateur de blocs, là où un Tier 2 fusionnerait Core et Distribution.

Chaque bloc héberge deux switches d'accès (2 × 2 = 4 Access au total), ce qui permet de simuler des extensions de bloc sans complexifier le schéma global.

### 3.5.3) Topologie et câblage

La topologie cible est la suivante :

```
              FW-1 ═════ FW-2      (couple HA traité en section 3.7)
              │ │         │ │
              │ │         │ │     (4 liens L3 /31, OSPF)
              └─┼─────────┼─┘
                │         │
             Core-1 ═════ Core-2   (peer-link LAG 2×LACP)
            ╱ │ │          │ │ ╲
           ╱  │ │          │ │  ╲  (8 liens full-mesh L3 /31, OSPF)
          ╱   │ │          │ │   ╲
       Dist-1 ═ Dist-2  Dist-3 ═ Dist-4
         │╲   ╱│           │╲   ╱│
         │ ╳  │            │ ╳  │    (8 liens L2 trunk, STP actif)
         │╱   ╲│           │╱   ╲│
       Acc-1 Acc-2        Acc-3 Acc-4
         │     │            │     │
      postes postes        postes postes
```

Les règles de câblage appliquées sont strictes :

**Full-mesh Core ↔ Distribution.** Chaque Core est connecté à chaque Distribution, y compris sur l'autre bloc. Cela donne 8 liens Core-Distribution au total (2 Core × 4 Distribution). La redondance et l'ECMP via OSPF sont ainsi garantis. Le pattern alternatif "chaque Core sert un bloc" est rejeté car il crée une asymétrie (si Core-1 tombe, tout le bloc 1 est isolé) et casse l'ECMP.

**Full-mesh Access ↔ Distribution au sein du bloc.** Chaque Access est connecté aux deux Distribution de son bloc, et uniquement à celles-là. Pas de liens inter-blocs au niveau Access. Pas de liens entre Access. 8 liens Access-Distribution au total.

**Pas de lien Core ↔ Core direct**, sauf le peer-link en LAG. Les deux Core forment un cluster logique via leur peer-link, pas via un chemin de routage normal.

**Pas de lien Distribution ↔ Distribution inter-blocs.** Deux Distribution d'un même bloc ont un peer-link intra-bloc. Deux Distribution de blocs différents n'ont aucun lien direct.

**Pas de lien Access ↔ Access.**

### 3.5.4) Niveau protocolaire par étage

La frontière L2/L3 dans le Tier 3 moderne est placée **au niveau de la Distribution**. Concrètement :

- Les **liens Access ↔ Distribution** sont des **trunks 802.1Q** portant les VLAN utilisateurs. Chaque Access a deux uplinks, un vers chaque Distribution du bloc. Ces uplinks ne sont pas agrégés en MLAG (voir 3.5.6) : STP (MSTP) bloque un des deux en nominal.
- Les **liens Distribution ↔ Core** sont des **liens L3 routés point-à-point en /31**. Pas de VLAN, pas de trunk : chaque lien est une interface routée qui porte une adjacence OSPF.
- Les **liens Core ↔ FW** sont également **L3 routés /31** avec OSPF.
- Les **SVI utilisateurs** (Vlan10 "USERS", Vlan20 "VOICE", etc.) vivent **sur les Distribution**, pas sur le Core. Le Core ne fait aucune terminaison de VLAN utilisateur.

Ce pattern "L3 à partir de la Distribution" est appelé parfois **"routed access boundary at Distribution"**. Il limite les domaines de broadcast à la taille d'un bloc de distribution, supprime le STP actif sur les liaisons inter-étages (mais pas sur les trunks Access-Distribution), et exploite pleinement OSPF + ECMP au cœur.

### 3.5.5) Peer-links intra-cluster

Les deux Core forment un cluster logique, reliés par un **peer-link en LAG de 2 liens LACP**. Idem pour chaque paire de Distribution au sein d'un bloc. Ces peer-links servent à :

- Synchroniser les tables MAC et ARP entre les deux membres du cluster (en mode MLAG s'il était disponible).
- Porter les messages de contrôle du cluster (heartbeats).
- Assurer le chemin de secours en cas d'échec d'une adjacence multi-chassis.

Le peer-link est **toujours en LAG**, même si un seul câble logique suffirait en trafic, parce que sa perte provoque un split-brain. Deux liens minimum, en LACP.

Sur plateforme vEOS avec MLAG, le peer-link porte en plus le VLAN MLAG de contrôle (4094 par convention). Sur plateforme Cisco IOSvL2 utilisée ici, le peer-link est un simple Etherchannel LACP en mode trunk portant tous les VLAN.

### 3.5.6) Redondance de passerelle : le cas de l'absence de MLAG sur IOSvL2

Chaque VLAN utilisateur voit ses SVI instanciées sur les deux Distribution du bloc, avec VRRP pour présenter une **VIP de passerelle partagée** aux postes. La configuration standard :

- Dist-A : `Vlan10` en `10.1.10.2/24`, priorité VRRP 120 pour le VLAN 10.
- Dist-B : `Vlan10` en `10.1.10.3/24`, priorité VRRP 100.
- VIP VRRP partagée : `10.1.10.1`.

Les postes utilisent la VIP `10.1.10.1` comme passerelle. En cas de panne de Dist-A, Dist-B prend la VIP en quelques secondes.

Pour équilibrer la charge, les priorités VRRP sont alternées par parité de VLAN : Dist-A est master sur les VLAN pairs, Dist-B sur les VLAN impairs. Chaque Distribution traite donc environ la moitié des flux en nominal, tout en pouvant reprendre la totalité en cas de panne du pair.

**Le cas du MLAG et son absence dans la maquette.** Dans les infrastructures datacenter et LAN modernes basées sur Arista ou Cisco Nexus, les deux switches Distribution peuvent se présenter à l'Access comme **un seul switch logique** via MLAG (Multi-Chassis Link Aggregation). L'Access peut alors agréger ses deux uplinks en un seul Port-Channel LACP, avec les deux liens actifs simultanément. C'est le pattern moderne, plus performant, plus rapide en convergence.

Cisco IOSvL2 **ne supporte pas le MLAG**. En conséquence, dans cette maquette, les deux uplinks d'un Access vers les deux Distribution **ne peuvent pas être agrégés**. STP (MSTP) bloque un des deux liens en nominal. Ce n'est pas optimal en bande passante, mais c'est parfaitement fonctionnel et reste le pattern historiquement le plus répandu dans les LAN d'entreprise (avant la démocratisation du MLAG). 

L'alternative aurait été de passer tout le LAN en vEOS (comme le DC) pour bénéficier du MLAG et du VARP. Ce choix n'a pas été retenu pour garder la syntaxe Cisco IOS sur le LAN (plus répandue dans les formations et certifications), et parce que le gain pédagogique du MLAG vs le surcoût de conversion de 10 équipements n'est pas décisif pour ce projet. La limite est assumée et documentée.

**Conséquence pratique** : les peer-links Core-Core et Distribution-Distribution intra-bloc restent configurés comme des Etherchannel LACP classiques (pas du MLAG), et les uplinks Access → Distribution restent en trunks individuels avec STP actif.

### 3.5.7) Sécurité de port et hygiène L2 sur les Access

Les switches Access portent l'essentiel des mécanismes de sécurité L2 :

- **Port security** sur les ports utilisateurs, avec limitation du nombre d'adresses MAC apprises (typiquement 2 pour autoriser PC + ToIP sur un même port).
- **DHCP Snooping** activé sur les VLAN utilisateurs, avec ports Access en "untrusted" et uplinks vers Distribution en "trusted".
- **Dynamic ARP Inspection (DAI)** appuyée sur le binding DHCP Snooping pour empêcher l'ARP spoofing.
- **Storm Control** pour limiter les tempêtes broadcast/multicast/unknown-unicast.
- **BPDU Guard** sur les ports Access : un utilisateur qui branche un petit switch en cascade déclenche un port-disable immédiat.
- **802.1X** avec RADIUS sur les ports Access pour l'authentification des postes, avec **MAB** (MAC Authentication Bypass) pour les équipements sans supplicant (imprimantes, caméras IoT).

Ces mécanismes sont détaillés dans les sections de configuration dédiées.

### 3.5.8) Handoff LAN ↔ FW

Le raccordement du LAN au reste du SI se fait par les **Core en L3 OSPF** vers les firewalls. Chaque Core a deux adjacences OSPF (une par FW), chaque FW a deux adjacences OSPF (une par Core), soit **4 liens L3 /31 full-mesh** entre Core et FW, dans le bloc `10.0.30.0/24`.

Le choix du L3 routé (plutôt qu'un trunk L2) est cohérent avec la philosophie "L3 partout" du projet. Les préfixes utilisateurs (SVI des Distribution) sont annoncés vers le Core via OSPF par redistribution des SVI connectées, puis relayés vers le FW. Le FW annonce en retour la default route apprise du WAN et les routes vers DMZ/DC.

## 3.6) Architecture de la zone Datacenter (Spine-Leaf)

### 3.6.1) Le problème du trafic Est-Ouest

Dans une infrastructure virtualisée moderne, la majorité du trafic ne sort pas vers l'extérieur : il circule **entre les serveurs eux-mêmes**. Réplication de bases de données, synchronisation de clusters, appels d'API internes, sauvegardes, vMotion, trafic CI/CD. On estime qu'au-delà de **80% du trafic d'un datacenter est Est-Ouest** (interne), le reste étant Nord-Sud (vers les utilisateurs ou Internet).

Le modèle Tier 3 classique, conçu pour le trafic Nord-Sud, montre rapidement ses limites dans ce contexte :

Pour que deux serveurs sur des Leaf différents communiquent, le trafic doit **remonter jusqu'à la couche Distribution ou Core**, ce qui introduit de la latence et sature les uplinks. Dans un design L2 avec Spanning Tree Protocol, la moitié des liens redondants est **bloquée par STP** pour éviter les boucles, et cette bande passante perdue est inacceptable pour des services distribués à forte demande inter-rack.

Ces limites poussent à adopter un modèle radicalement différent pour le datacenter : le **Spine-Leaf**.

### 3.6.2) Le paradigme Spine-Leaf

Le Spine-Leaf (ou fabric Clos) repose sur deux couches seulement :

**Les Leaf switches** (Top-of-Rack) qui raccordent les hyperviseurs et les serveurs. Chaque Leaf est connecté à **tous les Spine**.

**Les Spine switches** qui constituent le backbone de la fabric. Leur rôle est exclusivement de transiter le trafic entre Leaf. Un Spine ne connecte jamais directement un serveur (sauf pour le rôle Border, voir 3.6.4).

Les caractéristiques clés :

- **Full-mesh Spine ↔ Leaf**, pas de lien Leaf ↔ Leaf, pas de lien Spine ↔ Spine.
- **L3 routé dès le Leaf** (Top-of-Rack routing), avec des liens point-à-point /31 portant un IGP (OSPF ou IS-IS) ou eBGP unnumbered.
- **ECMP actif** sur tous les chemins : le trafic est réparti entre tous les Spine, pas de lien bloqué par STP.
- **Latence déterministe** : deux serveurs de racks différents sont séparés d'exactement 2 sauts (Leaf → Spine → Leaf).
- **Scalabilité horizontale** : ajouter un Spine double la bande passante, ajouter un Leaf ajoute des ports serveurs. Le réseau s'étend sans repenser l'existant.

### 3.6.3) Underlay et Overlay : VXLAN + BGP EVPN

L'adoption d'un Spine-Leaf L3 résout le problème ECMP/STP mais en crée un autre : **comment maintenir la flexibilité L2** indispensable à la virtualisation (migration live de VM, clustering, adjacence L2 partagée entre racks) sur une infrastructure entièrement routée ?

La réponse moderne est de **séparer l'infrastructure physique (underlay) de la couche logique des services (overlay)**. Le underlay est un pur réseau L3 routé, stable et performant. L'overlay est un réseau virtuel superposé, qui simule les adjacences L2 nécessaires aux VM par encapsulation.

#### Underlay L3

L'underlay est l'infrastructure physique routée qui porte le transport. Il est composé :

- d'un IGP (OSPF ou IS-IS) qui résout la connectivité entre toutes les loopbacks des équipements fabric,
- d'ECMP activé pour exploiter tous les chemins équivalents,
- de jumbo frames (typiquement MTU 9000+) sur tous les liens de fabric pour absorber l'overhead d'encapsulation sans fragmentation.

Dans le projet, on retient **OSPF en area 0** comme IGP underlay, pour sa simplicité de mise en œuvre et sa cohérence avec l'IGP utilisé dans le LAN et l'Edge. eBGP unnumbered (pattern "RFC 7938 BGP as IGP") aurait été une alternative crédible, plus utilisée chez les hyperscalers, mais moins pédagogique pour un premier lab EVPN.

#### Overlay VXLAN

**VXLAN** (Virtual Extensible LAN) est le protocole d'encapsulation qui fait vivre l'overlay. Il encapsule des trames Ethernet dans des paquets UDP (port 4789), ce qui leur permet d'être transportées à travers la fabric L3 comme de simples flux IP. À l'arrivée, le Leaf destination décapsule et délivre la trame Ethernet originale au serveur destinataire.

VXLAN fait sauter deux verrous historiques :

- **Limite des VLAN à 4096** : VXLAN introduit le **VNI** (VXLAN Network Identifier) codé sur 24 bits, soit 16 millions de réseaux virtuels possibles.
- **Limitation géographique L2** : les VM peuvent résider sur des racks différents tout en ayant l'illusion d'être sur le même segment Ethernet.

L'équipement qui réalise l'encapsulation/décapsulation s'appelle un **VTEP** (VXLAN Tunnel Endpoint). Dans le projet, les VTEP sont portés par les **Leaf**. Les Spine restent de purs équipements de transit, agnostiques à l'overlay.

#### Control plane BGP EVPN

L'encapsulation VXLAN seule ne suffit pas : il faut un mécanisme pour **distribuer les informations d'accessibilité** (quelle adresse MAC/IP est derrière quel VTEP, quelle VRF appartient à quel VNI). Laisser cela à du "flood-and-learn" Ethernet classique serait désastreux en performance.

**BGP EVPN** (Ethernet VPN) joue ce rôle. C'est une extension de BGP qui distribue des routes de type MAC/IP, MAC-only, Inclusive Multicast, Ethernet Segment. Chaque Leaf annonce à ses voisins les MAC/IP qu'il apprend localement. Le résultat :

- **Réduction drastique du trafic BUM** (Broadcast, Unknown unicast, Multicast) grâce à la connaissance a priori des mappings MAC-VTEP.
- **Convergence rapide** en cas de panne ou de migration de VM.
- **Multi-tenancy propre** via les Route Distinguishers et Route Targets, qui permettent d'isoler des VRF partagées sur la même fabric.

Les **Spine** jouent le rôle de **Route Reflector** (RR) BGP EVPN : chaque Leaf peer avec les deux Spine, les Spine réfléchissent les routes entre Leaf. Cela évite un full-mesh iBGP entre tous les Leaf.

L'AS retenu pour toute la fabric est **65000** (AS privé 16 bits, RFC 6996). Tous les équipements (Spine RR et Leaf RR Client) sont dans le même AS, ce qui confirme un design **iBGP EVPN** (et non eBGP EVPN qui utiliserait un AS par équipement).

### 3.6.4) Rôle Border : Spine-border retenu

La sortie de la fabric vers l'extérieur (FW, WAN, autres zones) doit passer par un rôle appelé **Border**. Trois implémentations existent :

**Border Leaf dédiés** : un ou deux Leaf sans serveur, dont le rôle unique est le handoff vers le FW. C'est le pattern canonique des grandes fabrics (>8 Leaf, exigences de scale des tables BGP externes, insertion de services L4-L7 multiples). Il **isole les domaines de panne** externe et interne, et permet d'ajouter des Spine sans toucher à la connectivité WAN.

**Border-Spine** : les Spine portent eux-mêmes la sortie vers le FW. Plus simple, approprié aux fabrics de 2 à 8 Leaf. C'est le pattern retenu ici.

**ToR qui fait aussi Border** : un Leaf normal porte des serveurs ET la sortie FW. Valide en très petit déploiement mais mélange les rôles.

**Justification du Border-Spine pour ce projet** : avec 3 Leaf et une sortie FW unique, l'ajout de Border Leaf dédiés serait overkill. Il faudrait doubler les équipements et les adjacences BGP EVPN pour aucun gain pédagogique ou fonctionnel. Le Border-Spine couvre tous les concepts (handoff VRF, insertion FW, U-turn inter-VRF) sans la surcharge.

**Limitation à documenter** : dans une fabric de grande taille ou avec insertion de services multiples, on reviendrait sur des Border Leaf dédiés pour isoler les domaines de panne et scaler indépendamment.

### 3.6.5) Multi-tenancy par VRF et U-turn forcé

La fabric porte **quatre VRF** de tenancy, chacune représentant une zone logique isolée :

- **VRF-PROD** : VM de production interne (bases de données, applicatifs métier, services d'infra internes).
- **VRF-DMZ** : VM de services exposés (reverse proxy, serveurs web, DNS public, SMTP inbound, API Gateway).
- **VRF-ADMIN** : VM de management (bastion, PBS, PDM, ITSM, CI/CD, supervision, SIEM, Vault, SSO).
- **VRF-MGMT** : interfaces d'administration des équipements (switches, FW, HV). Aucune VM applicative.

À ces quatre VRF de tenancy s'ajoute la **VRF `default`** qui porte l'underlay (loopbacks, liens /31, OSPF fabric, iBGP EVPN).

Chaque VRF a ses propres VNI (un VNI L2 par VLAN interne, un VNI L3 pour le transport inter-subnet), ses propres tables de routage, ses propres Route Targets. La fabric en garantit l'étanchéité via BGP EVPN : le trafic d'une VRF ne peut pas atteindre une autre VRF **à l'intérieur de la fabric**.

Pour qu'un flux passe d'une VRF à une autre (par exemple, une VM en DMZ qui doit interroger une base de données en PROD), il doit **sortir de la fabric par le Border, être inspecté par le firewall, puis être ré-injecté dans la fabric vers la VRF destination**. Ce pattern est appelé **U-turn** ou **"VRF-aware service insertion"**. Il garantit qu'aucune communication inter-VRF ne peut court-circuiter les règles de sécurité.

Le firewall n'a lui-même **pas de notion native de VRF** dans pfSense. Chaque VRF est présentée au FW comme un **VLAN de transit distinct** via les Spine :

- **VRF-PROD → VLAN 201** sur le trunk Spine ↔ FW
- **VRF-DMZ → VLAN 202**
- **VRF-ADMIN → VLAN 203**
- **VRF-MGMT → VLAN 204**

Le FW reçoit ces VLAN sur ses sous-interfaces (`em2.201`, `em2.202`, `em2.203`, `em2.204` côté Spine-1, et idem côté Spine-2), chacune portant une IP de transit propre à la VRF dans les blocs `10.0.21.0/24` (PROD), `10.0.22.0/24` (DMZ), `10.0.23.0/24` (ADMIN), `10.0.24.0/24` (MGMT). Les règles firewall entre ces sous-interfaces jouent le rôle de contrôle inter-VRF.

Ce pattern, parfois appelé **"one-armed VRF router"** ou **"VRF-lite via VLAN handoff"**, est le standard en ETI avec firewall open-source. Il ne scale pas à 100 VRF (ça devient lourd à configurer), mais pour 4 VRF il est parfaitement approprié.

### 3.6.6) Les hyperviseurs Proxmox

La fabric héberge **trois hyperviseurs Proxmox VE**, chacun raccordé à un Leaf distinct :

**HV-1 et HV-2** forment un **cluster Proxmox** et hébergent simultanément des VM DMZ, des VM PROD et des VM ADMIN, isolées par VLAN et par bridge Linux. Chaque zone a son propre bridge (`vmbr-prod`, `vmbr-dmz`, `vmbr-admin`), plus un bridge management `vmbr0`. Les règles d'hygiène interdisent les VM multi-bridge sauf cas documentés (reverse proxy, DNS, etc.). Des affinity rules préférentielles sont utilisées pour que les VM DMZ tournent préférentiellement sur un hôte et les VM PROD sur l'autre, sans empêcher la migration en cas de panne.

**HV-3** est un hyperviseur **standalone** dédié au management. Il héberge le **Proxmox Datacenter Manager** (administration centralisée des hyperviseurs) et le **Proxmox Backup Server** (équivalent open-source de Veeam). Son interface management est dans la VRF-MGMT comme les autres HV, et il héberge uniquement des VM de la VRF-ADMIN, pas de VM PROD ni DMZ. Pour cette raison, le trunk Leaf-3 ↔ HV-3 ne porte que les VLAN 300-306 et 999, alors que Leaf-1 et Leaf-2 portent en plus les VLAN 100-107 et 200-204.

**Quorum du cluster** : un cluster Proxmox à deux nœuds souffre d'un défaut de quorum — si un nœud tombe, l'autre perd la majorité et passe en lecture seule. La solution standard est un **QDevice** externe (démon `corosync-qnetd`) qui sert de troisième voix sans être membre du cluster. Dans ce projet, le QDevice est **hébergé sur HV-3** qui n'est pas dans le cluster mais peut parfaitement porter ce rôle. Cette décision est documentée comme bonne pratique cluster Proxmox.

**Connexion Leaf** : chaque HV est **single-homed** sur son Leaf, via un trunk 802.1Q portant tous les VLAN des zones hébergées. En production, le pattern serait du dual-homing des HV sur deux Leaf via EVPN multi-homing (ESI-LAG), pour éliminer le SPOF Leaf. Dans cette maquette, le single-homing est accepté comme simplification, avec la mention explicite dans le rapport.

### 3.6.7) Stockage

Le projet ne simule **pas de SAN physique** — tout est virtualisé au niveau Proxmox. Plusieurs modèles restent possibles côté plateforme Proxmox :

- **Local ZFS** (simple, pas de live migration sans replication).
- **ZFS + replication** (semi-HA async).
- **Ceph hyperconvergé** (shared storage natif, HA complet, pattern cloud privé moderne).
- **NFS ou iSCSI depuis une VM cible** (pattern DSI classique).

Ce choix n'est pas bloquant pour l'architecture réseau actuelle. Il sera tranché en début de Phase 2 (services), en cohérence avec les besoins du cluster (live migration, HA).

### 3.6.8) Schéma de la zone Datacenter

```
                     FW-1 ════ FW-2           (HA CARP, section 3.7)
                      │ │       │ │
                      │ │       │ │           (4 liens FW↔Spine full-mesh
                      │ │       │ │            L3 /31 underlay + trunk VLAN 201-204
                      │ │       │ │            pour handoff VRF)
                   Spine-1   Spine-2           (pas de lien Spine↔Spine)
                    │╲  ╲     ╱  ╱│
                    │ ╲  ╲   ╱  ╱ │
                    │  ╲  ╳ ╳  ╱  │            (6 liens Spine↔Leaf full-mesh
                    │   ╳ ╳ ╳ ╳   │             L3 /31, OSPF underlay
                    │  ╱  ╳ ╳  ╲  │             + iBGP EVPN overlay AS 65000)
                    │ ╱  ╱   ╲  ╲ │
                    │╱  ╱     ╲  ╲│
                  Leaf-1   Leaf-2  Leaf-3      (pas de lien Leaf↔Leaf)
                    │        │        │
                  HV-1     HV-2     HV-3        (1 lien trunk 802.1Q par HV)
                 (cluster) (cluster)(mgmt)
              PROD+DMZ+ADMIN     ADMIN uniquement
```

## 3.7) Placement des firewalls : pattern mutualisé multi-zone

### 3.7.1) Le firewall, hub central du SI

Dans l'architecture retenue, un **unique cluster de firewalls HA** assume la totalité du filtrage inter-zone du SI. Concrètement, le couple FW-1/FW-2 est connecté à :

- Le **SW-WAN** côté Edge (2 liens WAN, un par FW, sur `em1`).
- Les **Core LAN** côté utilisateurs (4 liens L3 /31 full-mesh, sur `em4` et `em5`).
- Les **Spine DC** côté fabric datacenter (4 liens L3 /31 avec sous-interfaces VLAN VRF, sur `em2` et `em3`).
- Le **lien HA pfsync** entre les deux FW (sur `em0`).

Soit 6 interfaces physiques par FW (`em0` à `em5`). Cette topologie en étoile autour des FW garantit que **tout flux inter-zone traverse le firewall** : LAN → Internet, LAN → DC, DC → Internet, DMZ ↔ PROD, inter-VRF dans la fabric. Le FW est le point de contrôle unique des politiques de sécurité.

**Répartition des interfaces par FW** :

| Interface | Rôle | IP (FW-1) | IP (FW-2) |
|---|---|---|---|
| `em0` | HA pfsync + CARP + XMLRPC | `10.0.255.1/30` | `10.0.255.2/30` |
| `em1` | WAN | `192.168.122.201/24` | `192.168.122.202/24` |
| `em2` | Underlay vers SPINE-1 + trunk VRF (sous-intf .201-.204) | `10.0.20.1/31` | `10.0.20.5/31` |
| `em3` | Underlay vers SPINE-2 + trunk VRF (sous-intf .201-.204) | `10.0.20.3/31` | `10.0.20.7/31` |
| `em4` | LAN vers CORE-1 | `10.0.30.0/31` | `10.0.30.4/31` |
| `em5` | LAN vers CORE-2 | `10.0.30.2/31` | `10.0.30.6/31` |

### 3.7.2) Pattern A (mutualisé) vs Pattern B (par périmètre)

Deux écoles existent pour le placement des firewalls en entreprise.

**Pattern A — FW mutualisé multi-zone.** Une seule paire de FW HA, multi-interface, qui sert simultanément Edge, LAN et DC. Pattern majoritaire en ETI classique. Simple à administrer (un seul point de politique), économique en matériel, riche pédagogiquement (multi-interface, multi-VRF via VLAN handoff, U-turn).

**Pattern B — FW par périmètre.** Des paires de FW distinctes selon la zone : FW-edge pour le WAN, FW-LAN pour le périmètre utilisateurs, FW-DC pour le datacenter. Plus cher, plus segmenté, justifiable dans les grands comptes où les équipes sécurité sont elles-mêmes segmentées par périmètre, ou dans des environnements où la scalabilité FW par zone est critique.

**Choix retenu : Pattern A.** Raisons :

- Pattern le plus fréquent en ETI, crédible en simulation.
- Pédagogiquement plus riche : un seul FW à configurer, mais touchant plus de notions (multi-interface, trunk VLAN pour VRF handoff, règles inter-zone, NAT sortant multi-zone, VPN, IPS).
- Économie de ressources GNS3 (2 à 4 VM pfSense en moins).
- Le Pattern B est documenté comme alternative grand-compte, non retenue.

### 3.7.3) Haute disponibilité CARP + pfsync

Le couple FW-1 / FW-2 fonctionne en HA active-passive via **CARP** (Common Address Redundancy Protocol), l'équivalent pfSense de VRRP/HSRP. Chaque interface data porte une **VIP CARP** qui flotte entre les deux FW. En nominal, FW-1 est master et porte toutes les VIP ; FW-2 est backup. En cas de panne de FW-1, FW-2 reprend toutes les VIP en quelques secondes.

Le **lien pfsync**, dédié sur `em0` en `10.0.255.0/30`, sert à synchroniser :

- les **tables d'états** (connexions TCP/UDP ouvertes), ce qui garantit que le FW backup peut reprendre les connexions en cours sans les réinitialiser ;
- la **configuration** (via XMLRPC HTTPS) pour que les règles restent cohérentes entre les deux membres ;
- les **annonces CARP** (heartbeats) pour l'élection du master.

Le lien pfsync est non agrégé. En production, il serait doublé en LAG pour éviter le split-brain sur perte de câble ; dans la maquette, le dédoublement est omis et documenté comme simplification.

### 3.7.4) Handoff VRF via VLAN

Comme évoqué en section 3.6.5, pfSense ne supporte pas nativement les VRF. Le handoff entre la fabric DC (multi-VRF) et le FW se fait par **présentation de chaque VRF comme un VLAN de transit** du côté Spine :

- **VRF-PROD → VLAN 201** sur le trunk Spine ↔ FW (bloc IP `10.0.21.0/24` en /30 par lien)
- **VRF-DMZ → VLAN 202** (bloc `10.0.22.0/24`)
- **VRF-ADMIN → VLAN 203** (bloc `10.0.23.0/24`)
- **VRF-MGMT → VLAN 204** (bloc `10.0.24.0/24`)

Le FW reçoit ces VLAN sur ses sous-interfaces (`em2.201` à `em2.204` côté Spine-1, `em3.201` à `em3.204` côté Spine-2). Chaque sous-interface porte une IP de transit propre à la VRF. Les règles firewall entre ces sous-interfaces jouent le rôle de contrôle inter-VRF.

Ce pattern, parfois appelé **"one-armed VRF router"** ou **"VRF-lite handoff"**, est le standard en ETI avec firewall open-source. Il ne scale pas à 100 VRF (ça devient lourd à configurer), mais pour 4 VRF il est parfaitement approprié.

Le détail complet de l'allocation des /30 par lien de handoff figure dans la feuille *Liens-L3-Infra* du plan d'adressage (section 3.8.3).

## 3.8) Plan d'adressage IPv4

Cette section fige le plan d'adressage de toute l'infrastructure. Il est la référence pour toutes les configurations d'équipement. Le plan détaillé (loopbacks, liens /31, /30, VLAN, handoff VRF, MGMT, VIP publiques) est maintenu dans le fichier `plan_adressage.xlsx` au format tableur. La présente section en donne la synthèse et justifie les choix structurants.

### 3.8.1) Philosophie de découpage

Le plan d'adressage est construit sur un principe **hiérarchique par zone** : chaque zone fonctionnelle majeure a son propre bloc /16 dans `10.0.0.0/8`, ce qui permet de reconnaître instantanément la zone à laquelle appartient une IP. Ce principe a deux vertus.

D'abord, il **facilite le débogage** : quand on voit une IP `10.2.x.x` dans une trace, on sait immédiatement qu'on est en VRF-PROD, sans avoir à consulter une table. Ensuite, il **préserve la scalabilité** : chaque /16 offre 256 /24, et on n'en utilise que 5 à 16 par zone. La marge de croissance est immense, ce qui évite toute renumérotation future si le projet grossit.

Le coût de cette approche est une "consommation" apparente d'IP plus large que strictement nécessaire. Sur un réseau privé RFC 1918 de 16 millions d'adresses, ce coût est nul. Le bénéfice de lisibilité et de scalabilité surpasse largement cette non-contrainte.

### 3.8.2) Vue globale des blocs

| Bloc | Préfixe | Rôle | /24 utilisés | /24 libres |
|---|---|---|---:|---:|
| Infrastructure L3 | `10.0.0.0/16` | Loopbacks, liens /31 inter-équipements, pfsync, handoff VRF | 5 | 251 |
| LAN utilisateurs | `10.1.0.0/16` | VLAN Users, Voice, IoT, WiFi×3, Printers, Lab (×2 blocs) | 16 | 240 |
| DC — VRF-PROD | `10.2.0.0/16` | VM production segmentées par tier applicatif | 8 | 248 |
| DC — VRF-DMZ | `10.3.0.0/16` | VM services exposés | 5 | 251 |
| DC — VRF-ADMIN | `10.4.0.0/16` | Outillage admin, CICD, monitoring, SOC, AAA, Vault | 7 | 249 |
| Management | `10.254.0.0/16` | VRF-MGMT in-band équipements | 1 | 255 |
| WAN + bloc public simulé | `192.168.122.0/24` | Subnet libvirt qui sert à la fois de peering PE↔FW et de bloc public (VIP CARP de services) | 1 (imposé) | — |

**Note sur la fusion WAN + public.** Dans une infrastructure réelle, on aurait un /30 de peering **plus** un /28 public routé, comme détaillé en 3.4.4. Dans le lab, par simplicité et pour éviter toute bidouille de routage statique sur le host, les deux rôles sont fusionnés dans le subnet libvirt `192.168.122.0/24`. Le détail de l'allocation est donné en 3.8.6.

### 3.8.3) Conventions par type de lien

- **Liens L3 point-à-point inter-équipements** : `/31` (RFC 3021). 2 IP utilisables par lien, zéro gaspillage.
- **Lien HA pfsync** : `/30` (convention pfSense), subnet `10.0.255.0/30`.
- **Loopbacks équipements fabric** : `/32` dans `10.0.0.0/24`, numérotation par rôle (1-2 Spine, 11-13 Leaf, 21-22 Core, 31-34 Distribution, 41-42 FW).
- **VLAN de postes et de VM** : `/24` systématique pour les segments broadcast.
- **Handoff VRF FW ↔ Spine** : `/30` par lien logique, bloc `/24` dédié par VRF (PROD=`10.0.21.0/24`, DMZ=`10.0.22.0/24`, ADMIN=`10.0.23.0/24`, MGMT=`10.0.24.0/24`).
- **Bloc public** : dans une vraie offre pro, /28 ou /29 routé par le FAI. Dans la maquette, fusionné avec le subnet de peering libvirt `192.168.122.0/24` (voir 3.4.4 et 3.8.6).

### 3.8.4) Allocation des sous-blocs dans `10.0.0.0/16`

Le bloc infrastructure est subdivisé en sous-blocs fonctionnels pour rester lisible :

| Sous-bloc | Rôle | Convention |
|---|---|---|
| `10.0.0.0/24` | Loopbacks équipements | /32 par équipement, numérotation par rôle |
| `10.0.10.0/24` | Underlay Spine ↔ Leaf | 6 liens /31 |
| `10.0.20.0/24` | Underlay FW ↔ Spine (interface parent du trunk) | 4 liens /31 |
| `10.0.21.0/24` | Handoff VRF-PROD sur VLAN 201 | 4 /30 (1 par lien FW-Spine) |
| `10.0.22.0/24` | Handoff VRF-DMZ sur VLAN 202 | 4 /30 |
| `10.0.23.0/24` | Handoff VRF-ADMIN sur VLAN 203 | 4 /30 |
| `10.0.24.0/24` | Handoff VRF-MGMT sur VLAN 204 | 4 /30 |
| `10.0.30.0/24` | Liens Core ↔ FW | 4 liens /31 |
| `10.0.40.0/24` | Liens Core ↔ Distribution (full-mesh) | 8 liens /31 |
| `10.0.255.0/30` | Lien pfsync FW ↔ FW dédié | 1 lien /30 |

Le reste du /16 (environ 245 /24 libres) est disponible pour des extensions futures : fabric DC secondaire, site distant, scission des liens internes, etc.

### 3.8.5) Allocation des VLAN LAN

Chaque bloc de Distribution a ses propres subnets, avec un offset `+100` sur le 3e octet pour le bloc 2. Le numéro de VLAN reste identique entre les deux blocs (même rôle fonctionnel) pour faciliter la lecture des configurations.

| VLAN | Nom | Bloc 1 (Dist-1/2) | Bloc 2 (Dist-3/4) |
|---|---|---|---|
| 10 | USERS | `10.1.10.0/24` | `10.1.110.0/24` |
| 20 | VOICE | `10.1.20.0/24` | `10.1.120.0/24` |
| 30 | IOT | `10.1.30.0/24` | `10.1.130.0/24` |
| 40 | WIFI-CORP | `10.1.40.0/24` | `10.1.140.0/24` |
| 41 | WIFI-GUEST | `10.1.41.0/24` | `10.1.141.0/24` |
| 42 | WIFI-BYOD | `10.1.42.0/24` | `10.1.142.0/24` |
| 50 | PRINTERS | `10.1.50.0/24` | `10.1.150.0/24` |
| 90 | LAB-DEV | `10.1.90.0/24` | `10.1.190.0/24` |

Dans chaque /24 utilisateur, le découpage interne est standardisé : `.1` VIP VRRP (passerelle), `.2`/`.3` SVI des deux Distribution, `.4-.9` réserve, `.10-.99` DHCP dynamique, `.100-.199` réservations DHCP, `.200-.254` IP statiques.

### 3.8.6) Allocation WAN et VIP publiques (fusionnées)

Le subnet libvirt `192.168.122.0/24` porte à la fois les IP physiques WAN des FW **et** les VIP CARP publiques de services. Cette fusion est une simplification assumée du lab (voir 3.4.4). Dans une infrastructure réelle, les VIP publiques seraient dans un /28 séparé, routé par le FAI.

Les IP physiques des FW sont placées en fourchette haute (.201, .202) pour rester en dehors de la plage DHCP dynamique libvirt (`.2-.199`), qui reste disponible pour des clients externes (PC-EXT en particulier).

| IP | Rôle | Commentaire |
|---|---|---|
| `192.168.122.1` | Gateway (port NAT Cloud = PE simulé) | Fournie par libvirt |
| `192.168.122.2`-`192.168.122.199` | Plage DHCP dynamique libvirt | Réservée aux clients externes branchés sur le NAT Cloud (PC-EXT, tests) |
| `192.168.122.10` | VIP CARP VPN WireGuard | Endpoint VPN télétravailleurs |
| `192.168.122.11` | VIP CARP Reverse Proxy HTTPS | Publication HTTPS via HAProxy en DMZ |
| `192.168.122.12` | VIP CARP SMTP MX inbound | Mail entrant |
| `192.168.122.13` | VIP CARP DNS autoritaire public | Résolution des zones publiques |
| `192.168.122.14` | VIP CARP API Gateway | Exposition d'APIs publiques |
| `192.168.122.200` | VIP CARP WAN | Source NAT par défaut depuis LAN/DC vers Internet |
| `192.168.122.201` | FW-1 physique WAN (em1) | IP master |
| `192.168.122.202` | FW-2 physique WAN (em1) | IP backup |
| `192.168.122.203`-`192.168.122.254` | Réserve statique | Extensions futures (CE, VIP supplémentaires) |

**Conflit potentiel avec DHCP libvirt.** Les IP fixes choisies (`.10`-`.14`, `.200`-`.202`) sont dans la plage DHCP libvirt par défaut. libvirt attribue ses IP DHCP séquentiellement depuis `.2`, donc en pratique il n'y aura pas de collision tant qu'on a moins de 8 clients DHCP simultanés. Pour garantir zéro chevauchement, on peut réduire la plage DHCP libvirt à `.2-.9` via `virsh net-edit default` (modifier la balise `<range>`). À faire si des collisions apparaissent en exploitation.

### 3.8.7) Traçabilité et maintenance

Le fichier `plan_adressage.xlsx` est la source de vérité. Il contient 13 feuilles :

1. Synthèse
2. Loopbacks
3. Liens-L3-Infra (underlay, handoff VRF, Core-FW, Core-Distribution, pfsync)
4. WAN-et-VIP-publiques (fusion WAN libvirt + VIP CARP de services)
5. VLAN-LAN
6. VLAN-DC-PROD
7. VLAN-DC-DMZ
8. VLAN-DC-ADMIN
9. MGMT
10. VRF (RD, RT, VNI L3)
11. Routage (OSPF, iBGP EVPN, VRRP, CARP)
12. Scalabilité
13. Légende

Toute modification du plan d'adressage doit passer par ce fichier avant d'être reportée sur un équipement. En cas d'incohérence, **ce fichier prévaut** sur toute configuration d'équipement.

## 3.9) Plan VLAN, VNI et VRF

Cette section décrit la segmentation logique complète : VLAN utilisateurs et VM, VRF de tenancy, mapping VLAN → VNI dans la fabric VXLAN, Route Distinguishers et Route Targets. Elle complète le plan d'adressage de la section 3.8 en expliquant les choix de segmentation et leurs justifications.

### 3.9.1) Approche de segmentation

La segmentation retenue suit le niveau de rigueur d'une **ETI ou d'un grand compte moderne** : on privilégie la segmentation fine par rôle fonctionnel plutôt que des blocs fourre-tout. Cela se traduit par 29 VLAN au total, répartis sur 4 VRF de tenancy plus la VRF underlay. Ce niveau de détail est défendable dans un SI de 300 à 1500 postes.

Un niveau de segmentation plus grossier (10-15 VLAN) aurait été suffisant pour un site de PME. À l'inverse, une très grande entreprise monterait à 50-100 VLAN avec des subdivisions par métier, par environnement ou par client hébergé. Le choix ici est un **compromis pédagogique** qui reflète un ETI sérieuse sans tomber dans la complexité d'un opérateur.

### 3.9.2) Matrice VRF de tenancy

| VRF | Rôle | Nb VLAN | VNI L3 | RD convention | RT import/export |
|---|---|---:|---:|---|---|
| `default` | Underlay, loopbacks, OSPF fabric, iBGP EVPN | — | — | — | — |
| `VRF-PROD` | Production interne (web, app, DB, infra, files, messaging, ToIP, intégration) | 8 | 50001 | `65000:100` | `65000:100` |
| `VRF-DMZ` | Services exposés (front, web, mail, DNS public, API) | 5 | 50002 | `65000:200` | `65000:200` |
| `VRF-ADMIN` | Outillage admin (CICD, monitoring, AAA, automate, Vault, SOC) | 7 | 50003 | `65000:300` | `65000:300` |
| `VRF-MGMT` | Management in-band équipements (switches, FW, HV) | 1 | 50004 | `65000:999` | `65000:999` |

La convention RT `65000:XXX` où `XXX` est le centaine du numéro de VRF laisse une marge très large pour ajouter d'autres VRF (jusqu'à `65000:999`). Les conventions VNI L2 (10000 + numéro VLAN) et VNI L3 (50000 + offset VRF) sont propres, traçables et laissent des millions de valeurs possibles.

**Justification du nombre de VRF retenues.** Trois VRF de tenancy (PROD, DMZ, ADMIN) sont les zones de confiance classiques d'une DSI mono-tenant. La VRF-MGMT séparée est une bonne pratique de plus en plus répandue pour isoler les interfaces d'administration des équipements. On aurait pu fusionner MGMT et ADMIN (souvent fait en PME), mais les sensibilités sont différentes : la MGMT est **orientée équipements**, l'ADMIN est **orientée outillage d'administration**. La séparation simplifie les règles firewall et le périmètre d'audit.

**Contexte multi-tenant non retenu.** Dans une entreprise hébergeant des clients tiers (MSP, cloud privé loué, hébergeur), on aurait une VRF par client plutôt qu'une VRF par zone de confiance. Ce n'est pas le cas ici : le projet simule une entreprise mono-tenant qui héberge **son propre SI**, donc les VRF représentent des zones de confiance internes.

### 3.9.3) VLAN LAN utilisateurs (8 VLAN par bloc)

Les 8 VLAN LAN reflètent la segmentation classique d'une ETI avec quelques ajouts modernes :

| VLAN | Nom | Rôle | Justification |
|---|---|---|---|
| 10 | USERS | Postes bureautiques | Cœur LAN, DHCP dynamique |
| 20 | VOICE | Téléphonie IP | VLAN voice tagué sur les ports access, QoS dédiée |
| 30 | IOT | Caméras, capteurs, IoT | Isolation stricte, pas d'accès ressources internes sans filtrage |
| 40 | WIFI-CORP | Wi-Fi salariés WPA3-Enterprise + RADIUS | Accès ressources internes autorisé |
| 41 | WIFI-GUEST | Wi-Fi invités captif | Internet uniquement, zéro accès interne |
| 42 | WIFI-BYOD | Téléphones/tablettes persos salariés | Accès limité, pas de ressources sensibles |
| 50 | PRINTERS | Imprimantes et MFP | Pool d'imprimantes partagées, isolation forte |
| 90 | LAB-DEV | Tests, postes dev, bancs | Séparé de la prod pour éviter pollutions |

Le VLAN **WIFI-BYOD** reflète une pratique devenue standard : les salariés apportent leurs téléphones/tablettes persos sur le Wi-Fi pro, mais ne doivent pas accéder aux ressources internes. Isolation entre WIFI-CORP et WIFI-BYOD.

Le VLAN **LAB-DEV** est présent dans la plupart des ETI à composante technique : postes de développement, bancs de test, machines de qualification, qui manipulent parfois des données non-productives mais qui ne doivent pas pouvoir accéder aux serveurs de prod.

Ces 8 VLAN sont dupliqués entre les deux blocs de Distribution (16 subnets /24 au total), avec un offset `+100` sur le 3e octet pour éviter les chevauchements.

### 3.9.4) VLAN DC — VRF-PROD (8 VLAN segmentés par tier applicatif)

La VRF-PROD applique une segmentation **par rôle applicatif** plutôt qu'un bloc unique. Le principe est le tiering classique "présentation → application → données → infra" :

| VLAN | Nom | VNI L2 | Rôle |
|---|---|---:|---|
| 100 | PROD-WEB-INT | 10100 | Frontaux web internes (intranet, portails applicatifs) |
| 101 | PROD-APP | 10101 | Serveurs applicatifs métier (backends, ERP) |
| 102 | PROD-DB | 10102 | Bases de données |
| 103 | PROD-INFRA | 10103 | DNS interne, DHCP, LDAP/AD, Kerberos, NTP, PKI |
| 104 | PROD-FILES | 10104 | Partages de fichiers (NextCloud, SMB, NFS) |
| 105 | PROD-MESSAGING | 10105 | Mail interne et messagerie |
| 106 | PROD-TOIP | 10106 | Backend ToIP (Asterisk, SIP) |
| 107 | PROD-INTEGRATION | 10107 | Middlewares, brokers (Kafka, RabbitMQ) |

**Justification du tiering par VLAN.** Un front compromis ne peut pas joindre la DB directement au niveau L3 : il faut que les règles firewall (si on met du filtrage intra-VRF via host-firewall ou ACL Leaf) autorisent explicitement le flux. En cas de compromis, les mouvements latéraux sont freinés. C'est le principe de **défense en profondeur** au niveau réseau.

### 3.9.5) VLAN DC — VRF-DMZ (5 VLAN)

La DMZ est elle aussi segmentée par fonction, pour qu'un compromis sur une fonction n'impacte pas les autres :

| VLAN | Nom | VNI L2 | Rôle |
|---|---|---:|---|
| 200 | DMZ-FRONT | 10200 | Reverse proxy HAProxy, WAF BunkerWeb |
| 201 | DMZ-WEB | 10201 | Serveurs web exposés (vitrine, apps publiques) |
| 202 | DMZ-MAIL | 10202 | SMTP inbound MX + outbound + antispam |
| 203 | DMZ-DNS | 10203 | DNS autoritaire public |
| 204 | DMZ-API | 10204 | API Gateway (exposition APIs publiques) |

**Point d'attention terminologique.** Les numéros VLAN 201-204 sont utilisés à deux endroits différents du projet : **VLAN de VM dans VRF-DMZ** (sur les Leaf et HV) et **VLAN de handoff VRF côté FW** (sur le trunk Spine-FW). Ce sont bien **deux contextes séparés**, mais pour la lisibilité on aurait pu choisir une plage différente pour l'un ou l'autre. Le choix retenu de conserver ces numéros vient du fait que dans la fabric EVPN, ces VLAN sont locaux à chaque équipement et identifiés par leurs VNI. Un VLAN 201 côté Leaf (VM DMZ-WEB, VNI 10201) n'a rien à voir avec un VLAN 201 sur une sous-interface FW (handoff VRF-PROD). C'est en pratique sans impact fonctionnel, mais à signaler pour éviter la confusion à la lecture des configurations.

### 3.9.6) VLAN DC — VRF-ADMIN (7 VLAN)

La VRF-ADMIN regroupe tout l'outillage d'administration du SI, segmenté selon la sensibilité et le rôle :

| VLAN | Nom | VNI L2 | Rôle |
|---|---|---:|---|
| 300 | ADMIN-CORE | 10300 | Bastion Guacamole, PBS, PDM, ITSM GLPI |
| 301 | ADMIN-CICD | 10301 | GitLab, Jenkins, runners CI/CD |
| 302 | ADMIN-MONITOR-PERF | 10302 | Zabbix, Prometheus, Grafana, Telegraf |
| 303 | ADMIN-AAA | 10303 | FreeRADIUS, Keycloak, SSO |
| 304 | ADMIN-AUTOMATE | 10304 | Ansible, AWX |
| 305 | ADMIN-VAULT | 10305 | HashiCorp Vault, gestion des secrets, PKI centrale |
| 306 | ADMIN-SOC | 10306 | Wazuh, Graylog, TheHive, SIEM (sensibilité maximale) |

**Segmentation MONITOR-PERF vs SOC.** C'est une segmentation importante en ETI et grand compte. Le VLAN ADMIN-MONITOR-PERF porte les outils de supervision perf (Zabbix, Prometheus), avec des agents déployés partout et des flux relativement ouverts. Le VLAN ADMIN-SOC porte les outils de SIEM/investigation, qui manipulent des données extrêmement sensibles (logs d'investigation, IOC, rapports d'incidents). Un compromis sur le monitoring perf ne doit pas donner accès aux données SOC. Dans une très grande organisation, on irait jusqu'à mettre le SOC dans une **VRF-SOC dédiée** ; ici on se limite à un VLAN isolé avec politique de filtrage spécifique.

**VLAN Vault séparé.** Le coffre-fort de secrets (tokens API, mots de passe, clés privées, certs) est l'un des outils les plus sensibles du SI. Il mérite un VLAN isolé avec des règles d'accès strictes, pour minimiser la surface d'attaque.

### 3.9.7) VLAN MGMT (1 VLAN)

Un seul VLAN suffit pour toutes les interfaces d'administration des équipements :

| VLAN | Nom | Subnet | VNI L2 | Rôle |
|---|---|---|---:|---|
| 999 | MGMT-INFRA | `10.254.0.0/24` | 10999 | Management in-band équipements |

Ce VLAN est étendu dans la fabric via VXLAN (VNI 10999) pour que toutes les interfaces MGMT (Spine, Leaf, Core, Distribution, Access, FW, HV) soient joignables. L'accès à ce VLAN est **strictement filtré** par le FW depuis la VRF-ADMIN via bastion. Aucun utilisateur final n'y a accès direct.

### 3.9.8) Trafic BUM et anycast gateway

Grâce à BGP EVPN, le trafic **BUM** (Broadcast, Unknown Unicast, Multicast) est fortement réduit :

- Les Leaf apprennent les MAC/IP via les annonces EVPN type 2 (MAC/IP advertisement), pas via du flood-and-learn classique.
- L'**ARP suppression** permet aux Leaf de répondre localement aux requêtes ARP des VM, sans flood à travers la fabric.
- Le **BUM résiduel** (par exemple DHCP discover initial, multicast) est transporté via BGP EVPN type 3 (Inclusive Multicast Ethernet Tag) sans flooding brut.

Côté routage, la **passerelle anycast** permet à chaque VLAN d'avoir **la même IP de passerelle sur tous les Leaf qui hébergent ce VLAN**. Par exemple, la passerelle du VLAN 101 (PROD-APP, `10.2.101.0/24`) est `10.2.101.1` configurée identiquement sur LEAF-1 et LEAF-2. Une VM qui migre de HV-1 vers HV-2 conserve sa passerelle sans changement, son trafic continue à fonctionner sans interruption. C'est le pattern clé pour la mobilité de VM dans une fabric EVPN.

## 3.10) Cas particulier des flux datacenter (intra-VRF vs inter-VRF)

Une subtilité essentielle du design Spine-Leaf + EVPN + VRF mérite d'être traitée explicitement : **tous les flux DC ne remontent pas au firewall**. Deux cas radicalement différents coexistent, et leur compréhension conditionne la perception qu'on peut avoir des performances et de la sécurité du datacenter.

### 3.10.1) Cas A — Flux intra-VRF (majoritaire, ~80 % du trafic Est-Ouest)

Quand deux VM appartiennent à la **même VRF** (même tenant, même zone de confiance) mais à des VLAN différents, leur trafic **reste entièrement dans la fabric**. Aucune remontée au firewall, aucun U-turn.

Exemple typique : dans VRF-PROD, une chaîne applicative `FRONT → BACKEND → DB`.

```
  FRONT (10.2.100.50)              BACKEND (10.2.101.50)          DB (10.2.102.50)
   VLAN 100                         VLAN 101                       VLAN 102
     │                                ▲                              ▲
     │ dst=10.2.101.50                │                              │
     ▼                                │                              │
  HV-1 (vmbr-prod) ──trunk──► LEAF-1 (Anycast GW PROD) ──VXLAN──► SPINE ──VXLAN──► LEAF-2 ──► HV-2 ──► BACKEND
                                                                                                  │
                                                                                                  │ dst=10.2.102.50
                                                                                                  ▼
                                                                                              DB (local)
```

**Mécanisme** :

1. FRONT envoie un paquet vers `10.2.101.50`, gateway = `10.2.100.1` (anycast).
2. LEAF-1, hôte de FRONT, reconnaît `10.2.100.1` comme son anycast gateway PROD. Il fait le routage L3 dans sa table de routage VRF-PROD.
3. `10.2.101.0/24` est annoncé en EVPN par LEAF-2 (ou LEAF-1 lui-même s'il héberge aussi BACKEND). Le Leaf source encapsule en VXLAN avec le VNI L3 de VRF-PROD (50001), destination = loopback du Leaf cible.
4. Le Spine fait du transit IP pur — il ne décapsule pas, ne lit pas le VNI. Il forward selon la table de routage underlay.
5. LEAF cible décapsule, applique la table de routage VRF-PROD, livre la trame au bon VLAN local.

**Résultat** :

- 2 à 3 sauts maximum dans la fabric (Leaf-source → Spine → Leaf-destination).
- Latence sub-milliseconde (les Spine et Leaf sont des équipements matériels line-rate en vrai DC ; en lab, on reste très rapide).
- **Le firewall n'est pas sollicité.**
- La bande passante est la bande passante native de la fabric, sans bottleneck.

### 3.10.2) Cas B — Flux inter-VRF (U-turn forcé par le FW)

Quand deux VM appartiennent à des **VRF différentes**, le trafic **doit obligatoirement traverser le FW**. C'est le pattern U-turn décrit en section 3.6.5.

Exemple : VM-Reverse-Proxy (VLAN 200, **VRF-DMZ**, sur LEAF-1) veut parler à VM-App (VLAN 101, **VRF-PROD**, sur LEAF-2).

```
VM-RP (10.3.200.50, VRF-DMZ)
    │ dst=10.2.101.50
    ▼
LEAF-1 (VRF-DMZ)
    │ Lookup table VRF-DMZ :
    │   10.2.101.0/24 n'est pas dans VRF-DMZ
    │   default route → FW (via Spine)
    ▼
Encapsulation VXLAN avec VNI L3 = 50002 (VRF-DMZ)
    │
    ▼
SPINE (Border) décapsule et présente au FW
sur le trunk VLAN 202 (handoff DMZ)
    │
    ▼
FW em2.202 (interface VRF-DMZ côté FW)
    │
    │ INSPECTION FIREWALL :
    │   src 10.3.200.50 → dst 10.2.101.50 tcp/443 : ACCEPT
    │   (selon la politique configurée)
    │   Logging + éventuelle inspection IPS Suricata
    │
    │ FW route selon sa table :
    │   10.2.101.0/24 → via em2.201 (VLAN 201 = handoff VRF-PROD)
    ▼
FW émet sur em2.201 (tagué 201)
    │
    ▼
SPINE reçoit sur VLAN 201, fait partie de VRF-PROD
    │ Ré-encapsule VXLAN avec VNI L3 = 50001 (VRF-PROD)
    ▼
SPINE → underlay → LEAF-2
    │
    ▼
LEAF-2 décapsule VRF-PROD, livre à VM-App
```

**Résultat** :

- Environ 7 sauts logiques (VM → HV → Leaf-src → Spine → FW → Spine → Leaf-dst → HV → VM).
- Latence plus élevée (mais reste négligeable sur un vrai DC ; en lab, quelques ms).
- **Le FW applique ses règles** : c'est le point de contrôle unique pour tout flux inter-zone.
- Si le FW refuse le flux, il est bloqué — et c'est exactement ce qu'on veut pour la sécurité.

### 3.10.3) Tableau récapitulatif des cas de flux

| Scénario de flux | Chemin | FW impliqué ? | Nb sauts typiques |
|---|---|:---:|---|
| Intra-VLAN (même VLAN, même subnet) | Local au Leaf ou bridging L2 dans la fabric | Non | 1 |
| Intra-VRF inter-VLAN, même Leaf | Routage L3 local au Leaf (IRB intégré) | Non | 1 |
| Intra-VRF inter-VLAN, Leaf différents | Leaf-src → Spine → Leaf-dst (VXLAN) | Non | 2-3 |
| **Inter-VRF** (DMZ ↔ PROD, PROD ↔ ADMIN, etc.) | Leaf-src → Spine → FW → Spine → Leaf-dst (U-turn) | **Oui** | 5-7 |
| VM → Internet | Leaf → Spine → FW → NAT → SW-WAN → NAT Cloud | Oui | 5-7 |
| VM → LAN utilisateur | Leaf → Spine → FW → Core → Distribution → poste | Oui | 6-8 |
| Poste LAN → VM DC | Poste → Access → Distribution → Core → FW → Spine → Leaf → HV → VM | Oui | 7-9 |

### 3.10.4) Conséquences sur le design

Cette distinction Cas A / Cas B a des implications concrètes sur la conception :

**Le regroupement applicatif en VRF est décisif.** Une chaîne applicative qu'on met entièrement dans la même VRF (ex. FRONT + BACKEND + DB dans VRF-PROD) bénéficie de la vitesse fabric, pas de passage FW. Si on la répartit entre plusieurs VRF, on force tout le trafic inter-composants à passer par le FW, ce qui tue la performance. **Le bon réflexe** : regrouper dans une même VRF les éléments qui se parlent beaucoup et ont des niveaux de confiance équivalents.

**Le FW n'est pas le bottleneck qu'on imagine.** Dans un DC moderne bien conçu, le FW voit environ 20 à 30 % du trafic (Nord-Sud + inter-VRF). Le reste (70 à 80 %) est géré par la fabric. C'est exactement l'inverse d'un design tout-L2 derrière un FW central où tout remonte.

**Le filtrage fin intra-VRF** (ex. empêcher le FRONT d'accéder directement à la DB sans passer par le BACKEND) se fait **à d'autres niveaux** :

- **Host-based firewall** sur les VM (iptables, nftables, Windows Firewall) : le pattern le plus commun.
- **ACL au niveau Leaf** : Arista et Cisco Nexus supportent des ACL L3/L4 à appliquer sur les SVI des Leaf. Performant mais plus complexe à gérer.
- **Service mesh applicatif** pour les applications conteneurisées (Istio, Linkerd).
- **Micro-segmentation dédiée** (Cisco Tetration, VMware NSX Distributed Firewall, Illumio) dans les très grands environnements.

Dans ce projet, on s'en tient au filtrage inter-VRF par FW (Cas B). Le filtrage intra-VRF sera envisagé en Phase 2 si un besoin spécifique apparaît, typiquement via host-firewall sur les VM les plus sensibles (DB, Vault, SIEM).

### 3.10.5) Ce qui rend ce design puissant

Le combo Spine-Leaf + VXLAN + EVPN + VRF donne **le meilleur des deux mondes** :

- Pour l'**intra-VRF** (majorité du trafic) : routage distribué, latence minimale, ECMP sur tous les chemins, pas de bottleneck central.
- Pour l'**inter-VRF** (trafic inter-zones sensibles) : contrôle centralisé au FW, inspection profonde, règles granulaires, traçabilité complète.

Sans VRF, il faudrait choisir : soit tout ouvert (pas de sécurité), soit tout au FW (bottleneck catastrophique pour le trafic Est-Ouest). Avec VRF + EVPN, on fait du **fin** : on ouvre là où on a confiance, on ferme et on contrôle là où il faut de la sécurité.

Ce pattern est devenu le **standard de facto du datacenter moderne** depuis 2018 et se retrouve chez la plupart des hyperscalers, cloud providers privés, et DSI modernes de grandes organisations.

## 3.11) Synthèse de l'architecture retenue

### 3.11.1) Vue globale

```
                        Internet (host Ubuntu)
                                │
                          NAT Cloud GNS3
                          (192.168.122.1)
                                │
                             SW-WAN
                             /    \
                            /      \
                        FW-1 ═══ FW-2       ← HA CARP + pfsync sur em0
                  (.201/.200)    (.202)       (10.0.255.0/30)
                       / │ │ \   / │ │ \
                      /  │ │  X  │ │  \
                     /   │ │ / \ │ │   \
                Spine-1  │ │/   \│ │  Spine-2
                  ╲ ╳ ╱  │ X     X │  ╲ ╳ ╱
                   ╳     │/│     │\│     ╳
                  ╱ ╳ ╲  │ │     │ │  ╱ ╳ ╲
              Leaf-1    Leaf-2  Leaf-3   Core-1═══Core-2
                │        │       │       /│ │ \    /│ │ \
               HV-1    HV-2    HV-3     / │ │  \  / │ │  \
              (clust)  (clust) (mgmt)  Dist-1═Dist-2 Dist-3═Dist-4
               PROD+    PROD+  ADMIN    │╲   ╱│      │╲   ╱│
               DMZ+     DMZ+   only    Acc-1 Acc-2  Acc-3 Acc-4
               ADMIN    ADMIN           │    │       │    │
                                       PC1  W-CLIENT L-CLIENT PC4
                                       VPCS Windows  Linux    VPCS
```

### 3.11.2) Décompte exact des équipements

La maquette GNS3 instancie **26 équipements** au total, répartis comme suit :

| Zone | Équipement | Qté | Plateforme |
|---|---|---:|---|
| Edge | NAT Cloud | 1 | GNS3 builtin (libvirt) |
| Edge | SW-WAN | 1 | GNS3 Ethernet Switch builtin |
| Edge / central | Firewall HA | 2 | pfSense CE 2.8.x |
| LAN | Core | 2 | Cisco IOSvL2 15.2 |
| LAN | Distribution | 4 | Cisco IOSvL2 15.2 |
| LAN | Access | 4 | Cisco IOSvL2 15.2 |
| DC | Spine | 2 | Arista vEOS-lab 4.35.3.1F |
| DC | Leaf | 3 | Arista vEOS-lab 4.35.3.1F |
| DC | Hyperviseur Proxmox | 3 | Proxmox VE 9.1 |
| LAN (endpoints) | Poste Windows (W-CLIENT) | 1 | Win 11 LTSC IoT |
| LAN (endpoints) | Poste Linux (L-CLIENT) | 1 | Xubuntu 24.04 |
| LAN (endpoints) | Postes VPCS (PC1, PC4) | 2 | VPCS |
| **Total** | | **26** | |

### 3.11.3) Décompte exact des liens

La topologie GNS3 compte **47 liens** au total, tous inventoriés dans le fichier `cartographie.xlsx` :

| Segment | Nombre | Nature |
|---|---:|---|
| NAT Cloud ↔ SW-WAN | 1 | L2 |
| SW-WAN ↔ FW (2 × 1 par FW) | 2 | L2 |
| FW ↔ FW (pfsync dédié `em0`) | 1 | L3 /30 |
| FW ↔ Core (full-mesh 4 liens) | 4 | L3 /31, OSPF |
| Core ↔ Core (peer-link LAG 2×LACP) | 2 | L2 |
| Core ↔ Distribution (full-mesh 8 liens) | 8 | L3 /31, OSPF |
| Distribution ↔ Distribution intra-bloc (peer-link 2×LACP × 2 blocs) | 4 | L2 |
| Access ↔ Distribution (2 par Access × 4 Access) | 8 | L2 trunk, STP actif |
| Access ↔ Client (1 par Access) | 4 | L2 access + voice |
| FW ↔ Spine (full-mesh 4 liens underlay + trunk VLAN 201-204) | 4 | L3 /31 + trunk 802.1Q |
| Spine ↔ Leaf (full-mesh 6 liens) | 6 | L3 /31, OSPF underlay + iBGP EVPN overlay |
| Leaf ↔ Hyperviseur (1 par HV) | 3 | L2 trunk 802.1Q |
| **Total** | **47** | |

### 3.11.4) Règles structurantes appliquées

Trois règles transverses pilotent tout le design :

**Règle de l'agrégation.** On fait du LAG (peer-link) uniquement entre équipements pair qui forment un cluster logique (Core↔Core, Distribution↔Distribution intra-bloc). On ne fait jamais de LAG entre étages (Core↔Distribution, Spine↔Leaf), parce que la redondance y est déjà garantie par la topologie multi-chemin exploitée par ECMP. Sur les uplinks Access↔Distribution, on aurait pu faire un LAG multi-chassis (MLAG) mais l'image IOSvL2 ne le supporte pas — on retombe sur STP actif, ce qui est fonctionnel mais moins optimal.

**Règle du L3 partout (sauf extrémités).** Les liens entre étages de routage sont en L3 routé /31, avec OSPF comme IGP. Le L2 trunk n'est présent qu'aux extrémités : liens Access↔Distribution pour les VLAN utilisateurs, liens Leaf↔HV pour les VLAN des zones DC, trunk VLAN pour le handoff VRF FW↔Spine. Cette règle limite les domaines de broadcast et exploite pleinement ECMP.

**Règle du U-turn forcé pour les flux inter-zone.** Aucun flux inter-zone (LAN↔DC, LAN↔Internet, DMZ↔PROD, inter-VRF, etc.) ne peut contourner le firewall. La topologie garantit structurellement que tout flux change de zone uniquement en passant par le couple FW HA. À l'inverse, les flux intra-VRF (Cas A de la section 3.10) restent dans la fabric et ne sollicitent pas le FW, ce qui préserve les performances Est-Ouest du DC.

### 3.11.5) État d'avancement du projet

La **Phase 1 — Architecture réseau** est en voie de finalisation. Les livrables produits à ce jour :

| Livrable | État | Localisation |
|---|---|---|
| Cahier des charges complet | ✅ Finalisé | Section 2 du présent rapport |
| Choix topologiques justifiés | ✅ Finalisé | Sections 3.1 à 3.7 |
| Maquette GNS3 fonctionnelle (26 équipements, 47 liens) | ✅ Topologie câblée | `homelab.gns3` |
| Plan d'adressage IPv4 complet | ✅ Finalisé | `plan_adressage.xlsx` (13 feuilles) |
| Plan VLAN / VNI / VRF | ✅ Finalisé | Section 3.9 + `plan_adressage.xlsx` |
| Cartographie détaillée par équipement | ✅ Finalisée | `cartographie.xlsx` (26 feuilles) |

Les étapes restantes pour clore la Phase 1 :

| Étape | État |
|---|---|
| Installation et boot de tous les équipements GNS3 | En cours (vEOS validé, pfSense à installer) |
| Configurations de base (hostnames, loopbacks, MGMT in-band) | À faire |
| Configuration de l'underlay OSPF (LAN Tier 3 + fabric DC) | À faire |
| Configuration de l'overlay iBGP EVPN (Spine = RR, Leaf = clients) | À faire |
| Configuration des VRF, VNI L2/L3, anycast gateway | À faire |
| Configuration des Access (VLAN, STP, sécurité L2, 802.1X) | À faire |
| Configuration des Distribution (SVI, VRRP, DHCP relay) | À faire |
| Configuration des FW (CARP, pfsync, NAT, FRR, VPN, rules) | À faire |
| Configuration des HV Proxmox (bridges, cluster, QDevice) | À faire |
| Scénarios de test (convergence OSPF, bascule CARP, apprentissage EVPN, U-turn inter-VRF) | À faire |

Une fois la Phase 1 validée, on enchaînera sur la **Phase 2** (déploiement des services applicatifs selon le canevas : notion → état du marché → sélection → dimensionnement → installation → sécurisation → administration → intégration). La **Phase 3** (mise en conformité SI : référentiels, cartographie des risques, PCA/PRA, gouvernance) interviendra en clôture.

