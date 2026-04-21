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

|#|Équipement|Zone|ISO / Image|Version|RAM|vCPU|Disque principal|Disque additionnel|Interfaces|Console|
|---|---|---|---|---|---|---|---|---|---|---|
|1|Routeur PE FAI|OUT|Cisco IOSv|15.9(3)M4 ou +récent|512 Mo|1|(fourni)|—|4|telnet|
|2|Firewall périmétrique (×2 HA)|OUT ↔ LAN/DMZ/DC|pfSense CE ou OPNsense|pfSense 2.8.x / OPNsense 25.x|2 Go|2|20 Go|—|6|vnc (install) / telnet (run)|
|3|Switch Core LAN (×2)|LAN|Cisco IOSvL2|15.2(20200924:215240)|1 Go|1|(fourni)|—|16|telnet|
|4|Switch Distribution LAN (×2)|LAN|Cisco IOSvL2|15.2(20200924:215240)|1 Go|1|(fourni)|—|16|telnet|
|5|Switch Access LAN (×N)|LAN|Cisco IOSvL2|15.2(20200924:215240)|768 Mo|1|(fourni)|—|16|telnet|
|6|Spine DC (×2)|DC|Arista vEOS-lab|4.33.x (ou +récent stable)|2 Go|1|vmdk fourni|Aboot ISO (bootloader)|8–16|telnet|
|7|Leaf DC (×4)|DC|Arista vEOS-lab|4.33.x|2 Go|1|vmdk fourni|Aboot ISO|8–16|telnet|
|8|Border Leaf (×1 ou ×2)|DC|Arista vEOS-lab|4.33.x|2 Go|1|vmdk fourni|Aboot ISO|8–16|telnet|
|9|Hyperviseur Proxmox VE (×2)|DC / DMZ|Proxmox VE|9.1|16 Go|6|300–500 Go|—|4–6|vnc|
|10|Proxmox Backup Server (×1)|DC (zone Infra)|Proxmox Backup Server|3.x (ou 4.x si sorti)|4 Go|2|32 Go|500 Go (datastore)|1|vnc|


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

### Partie : Ce qu'il faut configurer, couche par couche

Voici le récap structuré par équipement, qui répond directement à ta question "qu'est-ce qu'on configure où".

#### Sur les switches Access

**Protocoles/features à configurer localement** :

- VLAN (access + voice VLAN sur les ports utilisateurs)
- Trunk 802.1Q LACP vers les Distribution
- MSTP + PortFast + BPDU Guard
- Port Security
- DHCP Snooping (untrusted sur ports access, trusted sur uplinks)
- Dynamic ARP Inspection
- Storm Control
- 802.1X + MAB (vers RADIUS central)
- SSH v2, AAA local + RADIUS fallback, logs vers syslog
- SNMPv3 pour supervision
- NTP client vers serveur central
- DNS client (résolveur interne)

#### Sur les switches Distribution

**Protocoles/features** :

- VLAN locaux nécessaires + trunk vers Access
- **SVI utilisateurs** (une par VLAN) avec VRRP (ou VARP Arista)
- LACP peer-link avec l'autre Distribution du bloc (MLAG)
- MSTP (priorité élevée, secondaire au Core)
- **OSPF area 0** sur les uplinks vers Core (L3 /31)
- **DHCP Relay** (`ip helper-address`) sur chaque SVI utilisateur
- SSH v2, AAA, syslog, SNMPv3, NTP, DNS client

#### Sur les switches Core

**Protocoles/features** :

- **MLAG** + peer-link LACP entre Core-1 et Core-2
- **OSPF area 0** sur tous les liens (vers Distribution et vers FW)
- MSTP (priorité root)
- Pas de SVI utilisateur : le Core ne termine aucun VLAN utilisateur
- SSH v2, AAA, syslog, SNMPv3, NTP, DNS client

#### Sur les firewalls pfSense

**Protocoles/features** :

- CARP + pfsync (HA)
- **OSPF via FRR** vers les Core (côté LAN), vers les Spine (côté DC, avec VRF handoff)
- VIP (CARP) publiques côté WAN
- Règles firewall inter-zones
- NAT sortant
- Termination VPN (OpenVPN ou WireGuard)
- Suricata (IPS/IDS en mode inline ou IDS)
- Syslog export, SNMPv3, NTP client, DNS client

---

### Partie 4 — Tableau de synthèse "où ça se configure"

Pour que tu aies une vue d'ensemble :

|Protocole / Feature|Sur équipement réseau|Sur serveur central|Commentaire|
|---|---|---|---|
|VLAN|✓|—|Local, sans VTP|
|MSTP + BPDU Guard|✓|—|Local|
|VRRP (ou VARP)|✓|—|Local sur Distribution|
|LACP|✓|—|Local, peer-links et uplinks Access|
|OSPF|✓|—|Local|
|Port Security|✓|—|Local sur Access|
|DHCP Snooping + DAI|✓|—|Local sur Access|
|Storm Control|✓|—|Local sur Access|
|802.1X (supplicant → switch → RADIUS)|✓ (client)|✓ (FreeRADIUS)|Split|
|DHCP (relay → serveur central)|✓ (relay)|✓ (Kea+Stork)|Split|
|DNS|—|✓ (Unbound/BIND9)|Serveur central, équipements = clients|
|NTP|✓ (client)|✓ (chrony)|Split|
|AAA admin (TACACS+ / RADIUS)|✓ (client)|✓ (FreeRADIUS/tac_plus)|Split|
|Syslog|✓ (expéditeur)|✓ (Graylog/Wazuh)|Split|
|SNMPv3 / Télémétrie|✓ (agent)|✓ (Zabbix/Prometheus)|Split|
|NetFlow / sFlow|✓ (export)|✓ (ntopng/Elastiflow)|Split|

### Tableau synthétique par équipement (Zone datacenter)

Voici le tableau que tu demandes, par catégorie d'équipement :

#### Spine (vEOS)

|Protocole / Feature|But|
|---|---|
|**OSPFv2**|IGP underlay, résout les loopbacks de tous les équipements fabric|
|**BGP EVPN (Route Reflector)**|Réfléchit les routes EVPN entre tous les Leaf pour éviter le full-mesh iBGP|
|**MLAG peer-link**|Synchronise l'état MAC/ARP entre Spine-1 et Spine-2, porte le lien de contrôle du cluster|
|**Jumbo frames (MTU 9214)**|Absorbe l'overhead VXLAN (50 octets) sans fragmentation|
|**ECMP**|Répartition du trafic sur tous les chemins Spine-Leaf équivalents|
|**BFD** (optionnel)|Détection rapide de panne de lien (<1s vs 40s OSPF)|
|**VLAN handoff vers FW**|Présente chaque VRF sous forme de VLAN de transit au FW (pattern Border)|
|**Anycast gateway VIP**|IP partagée pour les VRF côté FW (si HA)|
|**SSH + AAA**|Administration sécurisée|
|**Syslog / SNMPv3 / NTP / DNS**|Intégration aux services centraux|

#### Leaf (vEOS)

|Protocole / Feature|But|
|---|---|
|**OSPFv2**|Participe à l'underlay, annonce sa loopback aux Spine|
|**BGP EVPN**|Client des RR Spine, annonce les MAC/IP de ses VM, apprend celles des autres Leaf|
|**VTEP VXLAN**|Encapsule les trames sortantes, décapsule les trames entrantes. Port UDP 4789|
|**VNI L2** (1 par VLAN)|Identifie chaque segment L2 étendu dans la fabric|
|**VNI L3** (1 par VRF)|Identifie chaque VRF pour le routage inter-subnet à travers la fabric|
|**VLAN locaux**|Mappent les interfaces serveur vers les VNI|
|**VRF-PROD, VRF-DMZ, VRF-ADMIN, VRF-MGMT**|Instanciées localement, avec RD et RT propres|
|**Distributed Symmetric IRB**|Mode de routage inter-VNI retenu|
|**Anycast Gateway**|Tous les Leaf répondent à la même IP de passerelle pour un VLAN donné, permet la mobilité VM|
|**ARP suppression (EVPN)**|Les Leaf répondent localement aux ARP grâce aux annonces EVPN, réduit le trafic BUM|
|**Jumbo frames**|Idem Spine|
|**Trunk 802.1Q** vers les HV|Porte tous les VLAN des zones hébergées sur le Leaf|
|**Storm Control / BPDU Guard**|Protection L2 sur les ports serveurs|
|**MAC move threshold**|Détection de boucles ou d'attaques|
|**SSH + AAA / Syslog / SNMPv3 / NTP / DNS**|Intégration services centraux|

#### Hyperviseur Proxmox (HV-1, HV-2)

|Protocole / Feature|But|
|---|---|
|**Linux Bridge par zone** (`vmbr-prod`, `vmbr-dmz`)|Un bridge dédié par VRF pour éviter la mutualisation des domaines L2 entre zones|
|**VLAN tagging** (802.1Q) sur le trunk vers Leaf|Chaque bridge est attaché à un VLAN du trunk entrant|
|**Cluster Corosync**|Communication de cluster Proxmox pour quorum et synchronisation HA|
|**Lien dédié cluster** (idéalement VLAN séparé)|Isole le trafic Corosync du data|
|**Interface management**|Interface Proxmox sur VLAN de VRF-MGMT, accès admin uniquement|
|**HA Manager**|Orchestration du failover des VM entre nœuds en cas de panne|
|**Affinity rules**|Règles de placement préférentiel des VM (DMZ sur HV-1, PROD sur HV-2 par exemple)|
|**SSH par clé uniquement**|Administration sécurisée|
|**Fail2ban**|Protection contre bruteforce SSH|
|**Firewall Proxmox** (pve-firewall)|Filtrage au niveau hyperviseur et par VM (bonus)|
|**Syslog / NTP / DNS**|Intégration services centraux|

#### Hyperviseur Proxmox standalone (HV-3, management)

|Protocole / Feature|But|
|---|---|
|**Tout ce qui précède** (Linux Bridge, management, SSH, NTP, etc.)|Base commune|
|**QDevice (corosync-qnetd)**|Troisième voix pour le quorum du cluster HV-1/HV-2|
|**Double patte réseau**|Une dans VRF-MGMT (pour atteindre les HV), une dans VRF-ADMIN (pour héberger les VM d'admin)|
|**Proxmox Backup Server** (hébergé en VM)|Sauvegarde des VM du cluster|
|**Proxmox Datacenter Manager** (hébergé en VM)|Administration centralisée multi-cluster|

#### Virtual switches / bridges Proxmox

|Élément|But|
|---|---|
|**vmbr0** (management)|Bridge pour l'interface admin de l'hyperviseur lui-même, connecté au VLAN MGMT|
|**vmbr-prod**|Bridge pour VM de VRF-PROD, connecté au VLAN de VRF-PROD sur le trunk|
|**vmbr-dmz**|Bridge pour VM de VRF-DMZ|
|**vmbr-admin**|Bridge pour VM de VRF-ADMIN|
|**VLAN-aware** (option alternative)|Un seul bridge `vmbr0` avec VLAN-aware activé, et on assigne un tag par VM. Plus compact, moins lisible|
|**Firewall Proxmox par VM**|Règles iptables au niveau hyperviseur pour chaque VM (L2 et L3)|
### 2) Protocoles et features à configurer sur la zone Edge

#### Haute disponibilité FW

- **CARP** : VIP flottantes entre FW-1 et FW-2. Chaque interface data porte une VIP CARP partagée, plus une IP unique par membre.
- **pfsync** : synchronisation des tables d'états (connexions TCP/UDP actives) entre les deux FW. Garantit que le FW backup peut reprendre les connexions sans les couper.
- **XMLRPC Config Sync** : synchronisation de la configuration (règles, NAT, VIP, certificats, users) via HTTPS depuis le master vers le backup. Transparent côté admin.

#### Routage

- **Route par défaut** : vers l'IP du NAT Cloud GNS3 (qui joue le PE). Statique pour la maquette.
- **OSPF via FRR** : adjacences vers les Core LAN et vers les Spine DC. Le FW annonce la default route apprise du WAN à toute l'infra interne. Cette partie relève plus des zones LAN/DC, on la mentionne ici pour complétude.
- **DHCP client** (temporaire) : pour récupérer une IP dynamique depuis le NAT Cloud. À remplacer par une **IP statique** (`192.168.122.11` pour FW-1, `.12` pour FW-2, VIP `.10`) dès que le plan est stable, pour avoir un comportement déterministe.

#### NAT

- **Outbound NAT** : NAT source sortant des réseaux internes vers l'IP WAN (simulée publique). Configuration en mode _Hybrid_ ou _Manual_ dans pfSense — **jamais en Automatic** en prod (trop opaque).
- **Inbound NAT / Port Forward** : pour exposer des services internes qui ne pourraient pas recevoir directement leur IP publique (rare dans notre modèle, puisque les VIP publiques sont portées directement par le FW).
- **Static 1:1 NAT** (bonus) : pour les cas où on veut dédier une IP publique à une VM interne (reverse proxy, serveur SMTP qui a besoin d'une IP source identifiable).

#### Filtrage

- **Rules par interface** : une politique par interface (WAN, LAN, DC_PROD, DC_DMZ, DC_ADMIN, DC_MGMT). pfSense applique un filtrage stateful, deny-by-default en entrée sur l'interface WAN.
- **Aliases** : regroupement d'IP, de réseaux ou de ports dans des objets nommés réutilisables. Obligatoire dès qu'on a plus de 10 règles, sinon c'est ingérable.
- **Floating Rules** : règles transverses qui s'appliquent à plusieurs interfaces à la fois (typiquement pour bloquer les bogons, ou pour du QoS).
- **Anti-bogons / anti-spoof** : pfSense bloque par défaut les IP privées entrantes sur WAN et les préfixes non alloués (bogons). À garder actif.

#### VPN

- **OpenVPN** ou **WireGuard** pour le VPN nomade (télétravailleurs). Le choix retenu dans le cahier des charges est OpenVPN, mais WireGuard est plus moderne, plus rapide et plus simple à maintenir. **Je te recommande WireGuard** pour le VPN nomade, et de documenter OpenVPN en alternative.
- **IPsec** pour un éventuel VPN site-to-site (interco avec un deuxième site fictif). Pas critique pour la maquette initiale.
- **Endpoint VPN** binder à une **VIP CARP publique** dédiée (par exemple `198.51.100.10` simulée), pour que le failover HA soit transparent côté clients.
- **Authentification VPN** : idéalement via RADIUS (FreeRADIUS dans le DC), avec MFA (TOTP) en bonus. Au démarrage, on peut rester sur du user/pass local + certificats, puis basculer sur RADIUS quand le service sera déployé en Phase 2.

#### IPS / IDS

- **Suricata** (préféré à Snort aujourd'hui : multithread, supporte mieux les rules modernes, interface pfSense plus aboutie).
- **Mode inline (IPS)** sur l'interface WAN : bloque les menaces avant qu'elles entrent.
- **Mode passif (IDS)** sur les interfaces internes : détecte sans bloquer pour éviter les faux positifs destructifs sur les flux internes.
- **Rulesets** : ET Open (Emerging Threats gratuit), Snort Community, éventuellement ET Pro (payant). Mise à jour quotidienne automatique.
- **pfBlockerNG** (bonus) : filtrage géographique, listes de réputation IP, DNSBL. Complémentaire à Suricata.

#### Services réseau sur FW

- **DNS Resolver (Unbound)** sur le FW : pour résoudre côté WAN et pour les équipements locaux du FW lui-même. Configurer les forwarders vers les DNS internes du DC pour les zones internes, et vers des résolveurs publics (1.1.1.1, 9.9.9.9) pour le reste.
- **DHCP server** sur le FW : **non**, le DHCP est centralisé dans le DC (Kea+Stork). Le FW fait juste du relay si nécessaire, mais même ça, c'est les Core/Distribution qui relayent, pas le FW.
- **NTP client** : synchronisation sur le serveur NTP interne (chrony dans le DC). **Pas** de serveur NTP sur le FW (les équipements LAN/DC synchronisent sur le serveur central, pas sur le FW).

#### Administration FW

- **HTTPS web admin** : accessible uniquement depuis la VRF-ADMIN via bastion, jamais depuis WAN. Certificat TLS interne signé par la PKI (Phase 2).
- **SSH** : clé uniquement, depuis la VRF-MGMT/ADMIN. Désactivé depuis WAN.
- **AAA** : authentification admin via RADIUS/TACACS+ (FreeRADIUS/tac_plus) une fois déployés.
- **Rôles / Privileges** : pfSense supporte des user groups avec des privileges granulaires. À utiliser pour séparer "admin full" / "admin read-only" / "admin VPN only".

#### Supervision / logging

- **Syslog remote** : export vers Graylog ou Wazuh.
- **SNMPv3** : agent pour Zabbix/Prometheus.
- **Netgate's pfsense_status / Telegraf** (optionnel) : télémétrie détaillée.
- **Logs firewall** : activés sur les règles importantes (pas sur toutes — ça spamme).
- **Logs VPN, NAT, Suricata** : tous exportés.

#### Sécurité du FW lui-même

- **Fail2ban-like** : pfSense a un mécanisme natif de _Login Protection_ contre le bruteforce sur le webGUI et SSH.
- **Anti-DDoS / Rate-limiting** : scrub rules, limites de connexions par source.
- **Bogons update** : mise à jour automatique des listes de bogons et d'IP privées.
- **Certificats et PKI internes** : pfSense a une CA interne pour signer les certs IPsec/OpenVPN. En prod on centraliserait sur une vraie PKI (Vault, Smallstep, Dogtag) — à voir en Phase 2.

#### SW-WAN (switch edge)

C'est le switch entre NAT Cloud et les FW. Très peu de configuration, mais quelques points importants :

- **VLAN unique** (VLAN 1 par défaut, ou VLAN dédié "WAN-transit" pour l'hygiène).
- **Trunk minimal ou access ports** : les trois ports (NAT Cloud, FW-1, FW-2) sont en access dans le même VLAN.
- **Pas de STP advanced** : une seule instance, MSTP par cohérence avec le LAN.
- **Pas de routage** : c'est un pur L2.
- **Port security minimal** : on peut laisser ouvert puisque ce sont des équipements de confiance branchés dessus, mais on peut fixer les MAC autorisées pour le principe.

#### NAT Cloud GNS3 (hors scope de configuration)

C'est un objet GNS3, pas un vrai équipement configurable. Il fournit du NAT vers le host Ubuntu via libvirt. Comportement par défaut :

- Subnet typique `192.168.122.0/24`
- DHCP activé
- NAT masquerading vers l'interface principale de l'host

On le mentionne dans le rapport comme "équivalent fonctionnel d'un PE du FAI dans la modélisation".

---

### 3) Tableau synthétique par équipement

#### Firewall pfSense (FW-1, FW-2)

|Protocole / Feature|But|
|---|---|
|**CARP**|VIP partagées entre FW-1 et FW-2 pour HA, une VIP par interface data + VIP publiques de services|
|**pfsync**|Synchronisation des tables d'états (connexions en cours) pour que le failover soit transparent|
|**XMLRPC Config Sync**|Synchronisation automatique de la configuration (rules, NAT, VIP, users) du master vers le backup|
|**Route statique par défaut**|Default route vers l'IP du NAT Cloud (PE simulé)|
|**DHCP client WAN (temporaire)**|Obtention d'une IP dynamique sur l'interface WAN ; remplacé par IP statique ensuite|
|**OSPF via FRR**|Routage dynamique vers Core LAN et Spine DC ; annonce de la default apprise du WAN|
|**Outbound NAT (SNAT)**|Translation de source des réseaux internes vers l'IP WAN ou une VIP publique dédiée|
|**Inbound NAT / DNAT**|Publication de services internes (reverse proxy, SMTP) si IP publique non directement portée sur le FW|
|**Static 1:1 NAT**|Binding d'une IP publique à une IP interne spécifique (optionnel)|
|**Firewall rules par interface**|Politique de filtrage stateful, deny-by-default, une policy par interface|
|**Aliases**|Regroupement d'IP, réseaux, ports en objets nommés réutilisables|
|**Floating rules**|Règles transverses multi-interface (bogons, QoS, logs)|
|**Anti-bogons / Anti-spoof**|Blocage des IP privées entrantes sur WAN et préfixes non alloués|
|**WireGuard** (ou OpenVPN)|Terminaison des VPN nomades, bindé sur une VIP CARP publique dédiée|
|**IPsec**|Terminaison des VPN site-to-site (non prioritaire pour la maquette initiale)|
|**Authentification VPN RADIUS**|Intégration avec FreeRADIUS pour les credentials VPN|
|**Suricata (IPS/IDS)**|Inspection profonde des paquets, blocage des menaces en mode inline sur WAN, détection en mode passif sur interfaces internes|
|**Rulesets ET Open / Snort Community**|Signatures de menaces mises à jour quotidiennement|
|**pfBlockerNG** (optionnel)|Filtrage géographique, listes de réputation IP, DNSBL|
|**Unbound (DNS resolver)**|Résolution DNS locale du FW, forwarders vers DNS interne et DNS public|
|**NTP client**|Synchronisation sur le serveur NTP interne (chrony dans DC)|
|**Login Protection**|Anti-bruteforce natif sur webGUI et SSH|
|**Scrub rules**|Normalisation des paquets, rate-limiting, anti-DDoS basique|
|**CA interne pfSense**|Signature des certs VPN et TLS (temporaire, à basculer sur PKI centrale en Phase 2)|
|**HTTPS webGUI**|Admin via interface web sécurisée, accessible uniquement depuis VRF-ADMIN via bastion|
|**SSH (clé uniquement)**|Admin CLI depuis VRF-MGMT, clé uniquement, jamais depuis WAN|
|**AAA RADIUS / TACACS+**|Authentification admin via serveur central (après Phase 2)|
|**User privileges groups**|Séparation "admin full" / "read-only" / "VPN only"|
|**Syslog remote**|Export des logs vers Graylog/Wazuh|
|**SNMPv3**|Agent de supervision pour Zabbix/Prometheus|
|**Bogons update automatique**|Mise à jour régulière des listes de préfixes à bloquer|

#### SW-WAN (Cisco IOSvL2)

|Protocole / Feature|But|
|---|---|
|**VLAN dédié WAN-transit**|Isolation du segment WAN des autres VLAN éventuels ; VLAN unique pour NAT Cloud + FW-1 + FW-2|
|**Mode access ports**|Les trois ports (NAT Cloud, FW-1, FW-2) sont en access dans le même VLAN WAN-transit|
|**MSTP**|Cohérence avec le reste du LAN, instance minimale puisqu'il n'y a qu'un VLAN|
|**BPDU Guard**|Filet de sécurité, bloque un port si BPDU non attendu|
|**SSH v2 + SNMPv3**|Administration sécurisée et supervision|
|**Syslog remote**|Export logs vers collecteur central|
|**NTP client**|Synchronisation sur le serveur NTP interne|
|**DNS client**|Résolution depuis le CLI vers le DNS interne|
|**Port security (optionnel)**|Limitation MAC par port, puisque les équipements branchés sont connus et fixes|

---
# 3) Architecture réseau

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

1. **Absence d'isolation** : une tempête de broadcast, un boucle STP, ou la compromission d'un poste se propage à tout le réseau.
2. **Pas de redondance** : la moindre panne matérielle arrête toute l'activité.

On le mentionne ici pour compléteness, mais il est hors périmètre du projet.

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

Le cheminement d'un paquet VPN entrant illustre le mécanisme :

```
1. Client → dst = 198.51.100.10:51820
2. Internet → FAI de l'entreprise (route BGP connue)
3. PE → CE via peering public (IP destination inchangée)
4. CE → table de routage : 198.51.100.0/28 via FW
5. CE → FW via lien privé de transit (IP destination toujours inchangée)
6. FW → reconnaît 198.51.100.10 comme VIP locale → livre au démon
```

À aucun moment le CE n'héberge la VIP. À aucun moment il n'y a de NAT sur le CE. L'IP de destination reste publique de bout en bout, même en traversant un segment L2 privé. **L'adressage du lien et l'adressage des paquets qui y transitent sont décorrélés**.

### 3.4.7) Gestion du NAT

Le NAT source sortant (masquerading des usagers vers Internet) est réalisé **sur le firewall**, pas sur le CE. Trois raisons :

Le FW est déjà stateful pour le filtrage ; la table NAT réutilise la même infrastructure. Dupliquer cette logique sur le CE n'a aucun intérêt technique. La séparation des rôles reste propre (CE = routage L3 pur, FW = sécurité + NAT + VPN). Les logs de filtrage et de traduction restent corrélés au même endroit, ce qui simplifie l'analyse a posteriori.

Le NAT destination entrant est **inexistant dans le modèle routé** : les IP publiques de service sont directement portées par le FW en VIP, il n'y a rien à traduire. Le DNAT n'intervient qu'à l'intérieur du FW, si un service public doit être redirigé vers une VM interne derrière la DMZ (ex. reverse proxy HAProxy en DMZ qui reçoit le trafic HTTPS public).

L'anti-pattern à éviter est le **NAT sur le CE** (DNAT port par port vers les services internes). On le rencontre en PME et sur les box familiales, mais il rompt la séparation des rôles, complique la gestion des services multi-ports, et ne passe pas proprement certains protocoles (IPsec ESP sans port, GRE, SCTP). Il n'est pas retenu ici.

### 3.4.8) Choix retenu pour la maquette

La maquette initiale implémente une **version simplifiée du Scénario 1**, avec les adaptations suivantes :

- Un seul FAI simulé via le **NAT Cloud de GNS3**, qui se comporte comme un PE fictif et fournit une IP DHCP sur le /24 du host Ubuntu (généralement `192.168.122.0/24` via libvirt).
- **Pas de CE** dans la maquette initiale. Les firewalls se raccordent directement au NAT Cloud à travers un switch L2 de transit (SW-WAN), lui-même connecté au NAT Cloud par un lien unique.
- **Deux pfSense en HA CARP** avec lien pfsync dédié. Toutes les VIP publiques simulées sont des IP Alias (ou CARP) sur les FW.
- **Sortie Internet fonctionnelle** depuis les VM internes pour permettre le `apt update`, les mises à jour Proxmox, les pull d'images. Le NAT Cloud GNS3 masquerade vers la vraie connectivité du host, on obtient donc un double NAT (FW → 192.168.122.x → IP hôte publique) qui est transparent pour le trafic sortant.

**Justification du choix** : le CE n'apporte aucune valeur pédagogique en l'absence de BGP. Une fois supprimé, on obtient une maquette plus simple sans rien perdre de ce qu'on veut démontrer côté FW (HA CARP, VIP, NAT, règles inter-zone).

**Évolution possible** : une phase ultérieure pourra réintégrer un CE Cisco IOSv entre le SW-WAN et le NAT Cloud, et configurer une session eBGP factice avec un second IOSv simulant un deuxième PE, pour démontrer le single-homing puis le multi-homing. Rien de ce qui est configuré dans la version initiale n'est invalidé par cette évolution.

### 3.4.9) Schéma de la zone Edge

```
                    Internet (host Ubuntu)
                            │
                    NAT Cloud GNS3
                            │
                       SW-WAN (IOSvL2, VLAN 1)
                       /        \
                      /          \
                  FW-1 ═══════ FW-2    ← HA CARP + pfsync sur em3
                   │             │        (10.255.0.0/30)
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

Chaque bloc héberge deux switches d'accès, ce qui permet de simuler des extensions de bloc sans complexifier le schéma global.

### 3.5.3) Topologie et câblage

La topologie cible est la suivante :

```
              FW-1 ═════ FW-2      (couple HA traité en section 3.7)
              │ │         │ │
              │ │         │ │     (4 liens L3 /31, OSPF)
              └─┼─────────┼─┘
                │         │
             Core-1 ═════ Core-2   (peer-link MLAG 2×LACP)
            ╱ │ │          │ │ ╲
           ╱  │ │          │ │  ╲  (8 liens full-mesh L3 /31, OSPF)
          ╱   │ │          │ │   ╲
       Dist-1 ═ Dist-2  Dist-3 ═ Dist-4
         │╲   ╱│           │╲   ╱│
         │ ╳  │            │ ╳  │    (8 liens L2 trunk LACP)
         │╱   ╲│           │╱   ╲│
       Acc-1 Acc-2        Acc-3 Acc-4
         │     │            │     │
      postes postes        postes postes
```

Les règles de câblage appliquées sont strictes :

**Full-mesh Core ↔ Distribution.** Chaque Core est connecté à chaque Distribution, y compris sur l'autre bloc. Cela donne 8 liens Core-Distribution au total (2 Core × 4 Distribution). La redondance et l'ECMP via OSPF sont ainsi garantis. Le pattern alternatif "chaque Core sert un bloc" est rejeté car il crée une asymétrie (si Core-1 tombe, tout le bloc 1 est isolé) et casse l'ECMP.

**Full-mesh Access ↔ Distribution au sein du bloc.** Chaque Access est connecté aux deux Distribution de son bloc, et uniquement à celles-là. Pas de liens inter-blocs au niveau Access. Pas de liens entre Access. 8 liens Access-Distribution au total.

**Pas de lien Core ↔ Core direct**, sauf le peer-link MLAG. Les deux Core forment un cluster logique via leur peer-link, pas via un chemin de routage normal.

**Pas de lien Distribution ↔ Distribution inter-blocs.** Deux Distribution d'un même bloc forment un MLAG pair et ont un peer-link. Deux Distribution de blocs différents n'ont aucun lien direct.

**Pas de lien Access ↔ Access.**

### 3.5.4) Niveau protocolaire par étage

La frontière L2/L3 dans le Tier 3 moderne est placée **au niveau de la Distribution**. Concrètement :

- Les **liens Access ↔ Distribution** sont des **trunks 802.1Q** portant les VLAN utilisateurs. Ils sont en L2 pur, agrégés en LACP vers le MLAG pair des Distribution.
- Les **liens Distribution ↔ Core** sont des **liens L3 routés point-à-point en /31**. Pas de VLAN, pas de trunk : chaque lien est une interface routée qui porte une adjacence OSPF.
- Les **liens Core ↔ FW** sont également **L3 routés /31** avec OSPF.
- Les **SVI utilisateurs** (Vlan10 "Users", Vlan20 "Voice", etc.) vivent **sur les Distribution**, pas sur le Core. Le Core ne fait aucune terminaison de VLAN utilisateur.

Ce pattern "L3 à partir de la Distribution" est appelé parfois **"routed access boundary at Distribution"**. Il limite les domaines de broadcast à la taille d'un bloc de distribution, supprime le STP actif sur les liaisons inter-étages, et exploite pleinement OSPF + ECMP au cœur.

### 3.5.5) Peer-links MLAG

Les deux Core forment un MLAG pair, reliés par un **peer-link en LAG de 2 liens minimum**. Idem pour chaque paire de Distribution. Ces peer-links servent à :

- Synchroniser les tables MAC et ARP entre les deux membres du cluster.
- Porter les messages de contrôle MLAG (élections, heartbeats).
- Assurer le chemin de secours en cas d'échec d'une adjacence multi-chassis.

Le peer-link est **toujours en LAG**, même si un seul câble logique suffirait en trafic, parce que sa perte provoque un split-brain. Deux liens minimum, en LACP.

Sur plateforme Arista (vEOS), le peer-link est configuré en trunk portant le VLAN MLAG (4094 par convention) et tous les VLAN data nécessaires. Le nom d'interface classique est `Port-Channel1000`.

### 3.5.6) Redondance de passerelle : VRRP + MLAG

Chaque VLAN utilisateur voit ses SVI instanciées sur les deux Distribution du bloc, avec VRRP (ou équivalent) pour présenter une **VIP de passerelle partagée** aux postes. La configuration standard :

- Dist-1A : `Vlan10` en `10.10.10.2/24`, priorité VRRP 120 pour le VLAN 10.
- Dist-1B : `Vlan10` en `10.10.10.3/24`, priorité VRRP 100.
- VIP VRRP partagée : `10.10.10.1`.

Les postes utilisent la VIP `10.10.10.1` comme passerelle. En cas de panne de Dist-1A, Dist-1B prend la VIP en quelques secondes.

Sur plateforme **Arista avec MLAG**, ce pattern est étendu via **VARP** (Virtual ARP) : les deux Distribution répondent simultanément à l'ARP de la VIP, chacune forwarding en L3 à 100%. Il n'y a plus de notion de master/backup — les deux sont actives en permanence. Ce pattern, équivalent au "HSRP+vPC" de Cisco, est plus moderne et plus performant.

Sur plateforme **Cisco IOSvL2** en lab (pas de MLAG), on retombe sur VRRP actif/passif classique avec STP + RSTP sur les uplinks Access, ce qui est moins élégant mais fonctionnel.

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

Le raccordement du LAN au reste du SI se fait par les **Core en L3 OSPF** vers les firewalls. Chaque Core a deux adjacences OSPF (une par FW), chaque FW a deux adjacences OSPF (une par Core), soit **4 liens L3 /31 full-mesh** entre Core et FW.

Le choix du L3 routé (plutôt qu'un trunk L2) est cohérent avec la philosophie "L3 partout" du projet. Les préfixes utilisateurs (SVI des Distribution) sont annoncés vers le Core via OSPF, puis redistribués vers le FW. Le FW redistribue les routes vers DMZ/DC/WAN.

## 3.6) Architecture de la zone Datacenter (Spine-Leaf)

### 3.6.1) Le problème du trafic Est-Ouest

Dans une infrastructure virtualisée moderne, la majorité du trafic ne sort pas vers l'extérieur : il circule **entre les serveurs eux-mêmes**. Réplication de bases de données, synchronisation de clusters, appels d'API internes, sauvegardes, vMotion, trafic CI/CD. On estime qu'au-delà de **80% du trafic d'un datacenter est Est-Ouest** (interne), le reste étant Nord-Sud (vers les utilisateurs ou Internet).

Le modèle Tier 3 classique, conçu pour le trafic Nord-Sud, montre rapidement ses limites dans ce contexte :

Pour que deux serveurs sur des Leaf différents communiquent, le trafic doit **remonter jusqu'à la couche Distribution ou Core**, ce qui introduit de la latence et sature les uplinks. Dans un design L2 avec Spanning Tree Protocol, la moitié des liens redondants est **bloquée par STP** pour éviter les boucles, et cette bande passante perdue est inacceptable pour des services distribués à forte demande inter-rack.

Ces limites poussent à adopter un modèle radicalement différent pour le datacenter : le **Spine-Leaf**.

### 3.6.2) Le paradigme Spine-Leaf

Le Spine-Leaf (ou fabric Clos) repose sur deux couches seulement :

**Les Leaf switches** (Top-of-Rack) qui raccordent les hyperviseurs et les serveurs. Chaque Leaf est connecté à **tous les Spine**.

**Les Spine switches** qui constituent le backbone de la fabric. Leur rôle est exclusivement de transiter le trafic entre Leaf. Un Spine ne connecte jamais directement un serveur.

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

### 3.6.4) Rôle Border : Spine-border retenu

La sortie de la fabric vers l'extérieur (FW, WAN, autres zones) doit passer par un rôle appelé **Border**. Trois implémentations existent :

**Border Leaf dédiés** : un ou deux Leaf sans serveur, dont le rôle unique est le handoff vers le FW. C'est le pattern canonique des grandes fabrics (>8 Leaf, exigences de scale des tables BGP externes, insertion de services L4-L7 multiples). Il **isole les domaines de panne** externe et interne, et permet d'ajouter des Spine sans toucher à la connectivité WAN.

**Border-Spine** : les Spine portent eux-mêmes la sortie vers le FW. Plus simple, approprié aux fabrics de 2 à 8 Leaf. C'est le pattern retenu ici.

**ToR qui fait aussi Border** : un Leaf normal porte des serveurs ET la sortie FW. Valide en très petit déploiement mais mélange les rôles.

**Justification du Border-Spine pour ce projet** : avec 3 Leaf et une sortie FW unique, l'ajout de Border Leaf dédiés serait overkill. Il faudrait doubler les équipements et les adjacences BGP EVPN pour aucun gain pédagogique ou fonctionnel. Le Border-Spine couvre tous les concepts (handoff VRF, insertion FW, U-turn inter-VRF) sans la surcharge.

**Limitation à documenter** : dans une fabric de grande taille ou avec insertion de services multiples, on reviendrait sur des Border Leaf dédiés pour isoler les domaines de panne et scaler indépendamment.

### 3.6.5) Multi-tenancy par VRF et U-turn forcé

La fabric porte plusieurs **VRF** (Virtual Routing and Forwarding), chacune représentant une zone logique isolée :

- **VRF-PROD** : VM de production interne (bases de données, applicatifs métier).
- **VRF-DMZ** : VM de services exposés (reverse proxy, serveurs web, DNS public).
- **VRF-ADMIN** : VM de management (Proxmox Datacenter Manager, Proxmox Backup Server, bastion).

Chaque VRF a ses propres VNI (un par VLAN interne), ses propres tables de routage, ses propres Route Targets. La fabric en garantit l'étanchéité via BGP EVPN : le trafic d'une VRF ne peut pas atteindre une autre VRF **à l'intérieur de la fabric**.

Pour qu'un flux passe d'une VRF à une autre (par exemple, une VM en DMZ qui doit interroger une base de données en PROD), il doit **sortir de la fabric par le Border, être inspecté par le firewall, puis être ré-injecté dans la fabric vers la VRF destination**. Ce pattern est appelé **U-turn** ou **"VRF-aware service insertion"**. Il garantit qu'aucune communication inter-VRF ne peut court-circuiter les règles de sécurité.

Le firewall n'a lui-même **pas de notion native de VRF** dans pfSense. Chaque VRF est présentée au FW comme un **VLAN de transit distinct** via les Spine. Le FW voit donc plusieurs sous-interfaces (une par VRF) et applique ses règles inter-interface comme il le fait classiquement. Ce modèle, parfois appelé **"one-armed VRF router"** ou **"VRF-lite via VLAN handoff"**, est le pattern standard en ETI avec firewall open-source.

### 3.6.6) Les hyperviseurs Proxmox

La fabric héberge **trois hyperviseurs Proxmox VE**, chacun raccordé à un Leaf distinct :

**HV-1 et HV-2** forment un **cluster Proxmox** et hébergent simultanément des VM DMZ et des VM internes, isolées par VLAN et par bridge Linux. Chaque zone a son propre bridge (`vmbr-dmz`, `vmbr-prod`), et les règles d'hygiène interdisent les VM multi-bridge sauf cas documentés (reverse proxy, DNS, etc.). Des affinity rules préférentielles sont utilisées pour que les VM DMZ tournent préférentiellement sur un hôte et les VM PROD sur l'autre, sans empêcher la migration en cas de panne.

**HV-3** est un hyperviseur **standalone** dédié au management. Il héberge le **Proxmox Datacenter Manager** (administration centralisée des hyperviseurs) et le **Proxmox Backup Server** (équivalent open-source de Veeam). Son interface management est dans la VRF-ADMIN, séparée des zones data.

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
                      │ │       │ │            L3 /31, OSPF + eBGP VRF)
                   Spine-1   Spine-2          (pas de lien Spine↔Spine)
                    │╲  ╲     ╱  ╱│
                    │ ╲  ╲   ╱  ╱ │
                    │  ╲  ╳ ╳  ╱  │           (6 liens Spine↔Leaf full-mesh
                    │   ╳ ╳ ╳ ╳   │            L3 /31, OSPF underlay
                    │  ╱  ╳ ╳  ╲  │            + iBGP EVPN overlay)
                    │ ╱  ╱   ╲  ╲ │
                    │╱  ╱     ╲  ╲│
                  Leaf-1   Leaf-2  Leaf-3     (pas de lien Leaf↔Leaf)
                    │        │        │
                  HV-1     HV-2     HV-3       (1 lien trunk 802.1Q par HV)
                  (cluster) (cluster)(mgmt)
```

## 3.7) Placement des firewalls : pattern mutualisé multi-zone

### 3.7.1) Le firewall, hub central du SI

Dans l'architecture retenue, un **unique cluster de firewalls HA** assume la totalité du filtrage inter-zone du SI. Concrètement, le couple FW-1/FW-2 est connecté à :

- Le **SW-WAN** côté Edge (2 liens WAN, un par FW).
- Les **Core LAN** côté utilisateurs (4 liens L3 /31 full-mesh).
- Les **Spine DC** côté fabric datacenter (4 liens L3 /31 full-mesh).
- Le **lien HA pfsync** entre les deux FW.

Soit environ 6 interfaces par FW. Cette topologie en étoile autour des FW garantit que **tout flux inter-zone traverse le firewall** : LAN → Internet, LAN → DC, DC → Internet, DMZ ↔ DC, inter-VRF dans la fabric. Le FW est le point de contrôle unique des politiques de sécurité.

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

Le **lien pfsync**, dédié, sert à synchroniser :

- les **tables d'états** (connexions TCP/UDP ouvertes), ce qui garantit que le FW backup peut reprendre les connexions en cours sans les réinitialiser ;
- la **configuration** (via XMLRPC HTTPS) pour que les règles restent cohérentes entre les deux membres ;
- les **annonces CARP** (heartbeats) pour l'élection du master.

Le lien pfsync est en subnet privé /30 (`10.255.0.0/30`), sur une interface dédiée (`em3` par convention), non agrégé. En production, il serait doublé en LAG pour éviter le split-brain sur perte de câble ; dans la maquette, le dédoublement est omis et documenté comme simplification.

### 3.7.4) Handoff VRF via VLAN

Comme évoqué en section 3.6.5, pfSense ne supporte pas nativement les VRF. Le handoff entre la fabric DC (multi-VRF) et le FW se fait par **présentation de chaque VRF comme un VLAN de transit** du côté Spine :

- VRF-PROD → VLAN 100 sur le trunk Spine ↔ FW
- VRF-DMZ → VLAN 200
- VRF-ADMIN → VLAN 300

Le FW reçoit ces VLAN sur ses sous-interfaces (`em2.100`, `em2.200`, `em2.300` côté Spine-1, et idem côté Spine-2), chacune portant une IP de transit propre à la VRF. Les règles firewall entre ces sous-interfaces jouent le rôle de contrôle inter-VRF.

Ce pattern, parfois appelé **"one-armed VRF router"** ou **"VRF-lite handoff"**, est le standard en ETI avec firewall open-source. Il ne scale pas à 100 VRF (ça devient lourd à configurer), mais pour 3 à 5 VRF il est parfaitement approprié.

## 3.8) Synthèse de l'architecture retenue

### 3.8.1) Vue globale

```
                        Internet (host Ubuntu)
                                │
                          NAT Cloud GNS3
                                │
                             SW-WAN
                             /    \
                            /      \
                        FW-1 ═══ FW-2       ← HA CARP + pfsync
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
                                        │╲   ╱│      │╲   ╱│
                                       Acc-1 Acc-2  Acc-3 Acc-4
                                         │    │       │    │
                                        PC   PC      PC   PC
```

### 3.8.2) Décompte des équipements

| Zone | Équipement | Qté | Plateforme |
|---|---|---:|---|
| Edge | NAT Cloud | 1 | Builtin GNS3 |
| Edge | SW-WAN | 1 | Cisco IOSvL2 |
| Edge / central | Firewall | 2 | pfSense CE |
| LAN | Core | 2 | Cisco IOSvL2 (ou vEOS) |
| LAN | Distribution | 4 | Cisco IOSvL2 (ou vEOS) |
| LAN | Access | 4 | Cisco IOSvL2 |
| DC | Spine | 2 | Arista vEOS-lab |
| DC | Leaf | 3 | Arista vEOS-lab |
| DC | Hyperviseur Proxmox | 3 | Proxmox VE |

Soit **22 équipements réseau et système** au total, hors endpoints utilisateurs simulés.

### 3.8.3) Décompte des liens par zone

| Segment | Nombre | Nature |
|---|---:|---|
| NAT Cloud ↔ SW-WAN | 1 | L2 |
| SW-WAN ↔ FW | 2 | L2 (1 par FW) |
| FW ↔ FW (HA pfsync) | 1 | L3 /30, dédié |
| FW ↔ Core | 4 | L3 /31, OSPF |
| Core ↔ Core (peer-link) | 2 en LAG | L2 MLAG sync |
| Core ↔ Distribution | 8 | L3 /31, OSPF, full-mesh |
| Distribution ↔ Distribution (peer-link intra-bloc) | 2 × 2 en LAG | L2 MLAG sync |
| Access ↔ Distribution | 8 | L2 trunk LACP |
| FW ↔ Spine | 4 | L3 /31, OSPF + trunk VLAN VRF |
| Spine ↔ Leaf | 6 | L3 /31, OSPF underlay |
| Leaf ↔ Hyperviseur | 3 | L2 trunk 802.1Q |

Soit **une quarantaine de liens logiques**, tous avec un rôle précis et justifié.

### 3.8.4) Règles structurantes appliquées

Trois règles transverses pilotent tout le design :

**Règle de l'agrégation.** On fait du LAG (peer-link MLAG) uniquement entre équipements pair qui forment un cluster logique (Core↔Core, Distribution↔Distribution intra-bloc). On ne fait jamais de LAG entre étages (Core↔Distribution, Access↔Distribution en routage, Spine↔Leaf), parce que la redondance y est déjà garantie par la topologie (multiples uplinks exploités par ECMP ou MLAG multi-chassis).

**Règle du L3 partout.** Les liens entre étages de routage sont en L3 routé /31, avec OSPF comme IGP. Le L2 trunk n'est présent qu'aux extrémités : liens Access↔Distribution pour les VLAN utilisateurs, liens Leaf↔HV pour les VLAN de zones datacenter, trunk VLAN pour le handoff VRF FW↔Spine.

**Règle du U-turn forcé.** Aucun flux inter-zone (LAN↔DC, LAN↔Internet, DMZ↔PROD, etc.) ne peut contourner le firewall. La topologie garantit structurellement que tout flux change de zone uniquement en passant par le couple FW HA.

### 3.8.5) Ce qui reste à produire pour clore la Phase 1

L'architecture étant actée, les prochaines étapes de la Phase 1 sont :

1. **Plan d'adressage IP complet** : loopbacks de tous les équipements, subnets de peering /31 et /30, blocs VLAN utilisateur, bloc public simulé, numérotation des AS BGP.
2. **Plan VLAN / VNI** : numérotation cohérente des VLAN par zone, mapping VLAN → VNI pour la fabric VXLAN.
3. **Plan des VRF** : Route Distinguisher et Route Target par VRF, VLAN de handoff correspondants.
4. **Plan OSPF** : areas (area 0 unique ou découpage), authentification, redistribution.
5. **Plan BGP EVPN** : AS iBGP, rôle Route Reflector des Spine, AF configurations.
6. **Configurations concrètes** des équipements (IOSvL2, vEOS, pfSense, IOSv), validées en GNS3.
7. **Scénarios de test** : convergence OSPF, bascule CARP, bascule MLAG, apprentissage EVPN, U-turn inter-VRF.

Les Phases 2 (déploiement des services) et 3 (conformité SI) viendront ensuite, une fois l'infrastructure réseau stabilisée.