# Roadmap de déploiement

Ce fichier est la référence de l'ordre de déploiement de la maquette. Il répond à deux questions : dans quel ordre configure-t-on les équipements, et pourquoi cet ordre plutôt qu'un autre. Chaque étape liste les prérequis, les actions, et les commandes de vérification à exécuter avant de passer à la suite.

---

## Vue d'ensemble

| Phase | Section rapport | Statut |
|---|---|---|
| **Phase 1a** — Architecture réseau | Section 3 | ✅ Finalisée |
| **Phase 1b** — Configuration des équipements | Section 4 | 🔄 En cours |
| **Phase 2** — Services applicatifs | Section 5 | ⬜ À venir |
| **Phase 3** — Conformité SI | Section 6 | ⬜ À venir |

---

## Logique d'enchaînement — pourquoi cet ordre

### Principe fondateur : deux îlots OSPF, un pont FW

L'infrastructure forme deux domaines OSPF area 0 qui partagent le même numéro d'area mais sont physiquement séparés par le firewall :

- **Îlot LAN** : Core ↔ Distribution ↔ (FW côté LAN)
- **Îlot DC** : (FW côté DC) ↔ Spine ↔ Leaf

Le firewall pfSense est le **pont entre les deux îlots**. Sans lui configuré avec FRR OSPF, les deux côtés ne se connaissent pas. Avec lui, une seule area 0 couvre toute l'infra.

Cette architecture impose une stratégie de déploiement claire : **on monte chaque îlot séparément, on le valide en isolation, et on les connecte via le FW en toute dernière étape**. À ce moment, si quelque chose ne fonctionne pas, on sait que le problème vient du FW uniquement — les deux îlots sont déjà prouvés fonctionnels.

### Pourquoi le DC avant le FW

La question centrale est : après avoir fini le LAN, fait-on le FW ou le DC en premier ?

**Réponse : DC d'abord, FW en dernier.**

Raisons :

1. **Le DC est le morceau le plus complexe.** BGP EVPN implique plusieurs couches qui s'empilent : OSPF underlay → BGP peering (RR/clients) → EVPN address families → VXLAN encapsulation → VRF isolation → anycast gateway → ARP suppression. Chaque couche a ses propres commandes de vérification. Il faut les valider une à une, sans perturbation externe. Intercaler pfSense au milieu (avec CARP, pfsync, FRR, sous-interfaces VRF, règles) créerait du bruit lors du troubleshooting.

2. **Le DC se valide en isolation.** Les adjacences BGP EVPN entre Spine et Leaf ne nécessitent pas le FW. Les VTEP, les VNI, la mobilité VM — tout ça se teste depuis les consoles GNS3 des Spine/Leaf/HV directement, sans avoir besoin d'un poste LAN qui passe par le FW.

3. **Proxmox dépend du DC overlay.** Le cluster Corosync (HV-1/HV-2) communique via VLAN 999, étendu en VNI 10999 dans la fabric. Sans EVPN overlay opérationnel, le cluster ne peut pas se former. Proxmox vient donc après le DC overlay, pas avant.

4. **Le FW comme connecteur final.** Quand le LAN est validé et le DC est validé, le FW n'a plus qu'à rejoindre les deux OSPF islands et exposer les VRF via les sous-interfaces VLAN. Si une route ne passe pas, c'est forcément dans le FW.

### Pourquoi Access avant Distribution avant Core

Dans le LAN Tier-3, on monte dans l'ordre inverse de la topologie logique (de bas en haut). C'est la seule façon de valider sans dépendances circulaires :

- Les **Access** ne dépendent de rien en amont pour leur config L2 (VLAN, trunks, port-security, DHCP snooping). On peut les configurer entièrement et vérifier leur état L2 sans que le reste du réseau soit opérationnel.
- Les **Distribution** ont besoin des Access pour valider VRRP (les postes doivent recevoir leur passerelle) et DHCP relay (les postes doivent obtenir une IP). On ne peut pas valider leur SVI sans des Access trunks fonctionnels.
- Le **Core** dépend des Distribution pour que OSPF ait des routes à apprendre. Un Core sans Distribution voisins OSPF n'a rien à router.

---

## Phase 1b — Étapes détaillées

### Étape 0 — Configuration de base (tous les équipements, en parallèle)

**Objectif :** sécuriser le plan d'administration avant toute config fonctionnelle. Cette étape est indépendante du réseau — on configure depuis la console GNS3 directement.

**Prérequis :** GNS3 démarré, équipements bootés, accès console.

**Actions :**

| Action | Cisco IOS | Arista EOS | pfSense | Proxmox |
|---|---|---|---|---|
| Hostname | `hostname X` | `hostname X` | GUI → System → General | `hostnamectl set-hostname X` |
| Bannière | `banner motd` | `banner motd` | — | `/etc/motd` |
| Désactiver CDP/LLDP sortant sur ports non-infrastructure | `no cdp enable` par interface | `no lldp transmit` | N/A | N/A |
| SSH v2 uniquement | `ip ssh version 2` | déjà défaut | GUI activé | déjà actif |
| Utilisateur local admin | `username admin priv 15 secret X` | `username admin priv 15 secret X` | GUI | déjà root |
| VTY restrict SSH | `transport input ssh` + `login local` | idem | N/A | `sshd_config` |
| Service password-encryption | `service password-encryption` | N/A (hash auto) | N/A | N/A |
| Désactiver services inutiles | `no ip http server`, `no ip http secure-server`, `no cdp run` (si non nécessaire) | `no ip http server` | désactiver console HTTP si non besoin | — |
| NTP client | `ntp server <IP>` | `ntp server <IP>` | GUI | `chronyc` |
| Syslog | `logging host <IP>` | `logging host <IP>` | GUI | `/etc/rsyslog.conf` |

**Vérification :**
```
! Cisco / Arista
show running-config | section line vty
show running-config | include hostname
show ssh
```

**État cible :** tous les équipements accessibles en SSH depuis la console GNS3 avec auth locale, pas de services non nécessaires actifs.

---

### Étape 1 — LAN : couche L3 (Core + Distribution — OSPF underlay LAN)

**Objectif :** établir les adjacences OSPF entre Core, Distribution et (côté LAN) les interfaces futures du FW. Le Core doit apprendre les préfixes LAN depuis les Distribution.

**Prérequis :** Étape 0 terminée. Interfaces physiques UP/UP entre Core ↔ Distribution (vérifier dans GNS3).

**Périmètre :**
- Interfaces L3 routées /31 Core ↔ Distribution (bloc `10.0.40.0/24`)
- Interfaces L3 routées /31 Core ↔ FW (bloc `10.0.30.0/24`) — configurées sur Core, le FW est éteint pour l'instant, les interfaces seront juste en `down` côté FW
- Loopbacks Core (`10.0.0.21/32`, `10.0.0.22/32`) et Distribution (`10.0.0.31-34/32`)
- OSPF area 0 sur tous ces liens
- Peer-link LACP Core ↔ Core (agrégation `Po1`)
- Peer-links LACP Distribution ↔ Distribution intra-bloc (`Po1` sur chaque paire)

**Vérification :**
```
show ip ospf neighbor       ! adjacences FULL sur tous les liens L3
show ip route ospf          ! routes O apprises depuis les voisins
show interfaces Po1         ! port-channel UP, membres actifs
ping 10.0.0.22 source lo0  ! Core-1 ping loopback Core-2
```

---

### Étape 2 — LAN : couche L2 (Distribution + Access — VLAN, SVI, VRRP, sécurité L2)

**Objectif :** monter les VLAN utilisateurs, les SVI de passerelle sur les Distribution, VRRP, les trunks vers les Access, et la sécurité L2 sur les ports Access.

**Prérequis :** Étape 1 terminée — OSPF stable entre Core et Distribution.

**Périmètre :**
- Distribution : VLAN 10/20/30/40/41/42/50/90, SVI avec IP /24, VRRP par VLAN (priorités alternées pair/impair), DHCP relay (`ip helper-address` vers futur serveur DHCP)
- Access : ports access par VLAN, trunk vers les deux Distribution, MSTP, BPDU Guard, Port Security, DHCP Snooping, DAI, Storm Control
- Trunk Distribution ↔ Access portant tous les VLAN du bloc

**Vérification :**
```
! Sur Distribution
show vlan brief
show standby brief          ! VRRP : un Active, un Standby par VLAN
show ip interface brief     ! SVI UP

! Sur Access
show interfaces trunk
show ip dhcp snooping binding
show spanning-tree summary
```
Test fonctionnel : configurer une IP statique sur W-CLIENT dans VLAN 10 (`10.1.10.50/24`, GW `10.1.10.1`), vérifier que le ping vers la SVI fonctionne.

---

### Étape 3 — DC underlay (Spine + Leaf — OSPF, loopbacks, MTU jumbo)

**Objectif :** établir les adjacences OSPF entre Spine et Leaf dans la fabric DC. C'est la fondation sur laquelle s'appuie tout l'overlay EVPN.

**Prérequis :** Étape 0 terminée sur Spine et Leaf. Liens physiques Spine ↔ Leaf UP dans GNS3.

**Périmètre :**
- Interfaces L3 routées /31 Spine ↔ Leaf (bloc `10.0.10.0/24`)
- Loopbacks Spine (`10.0.0.1/32`, `10.0.0.2/32`) et Leaf (`10.0.0.11-13/32`)
- OSPF area 0 sur tous ces liens, `network point-to-point`
- MTU 9214 sur toutes les interfaces fabric (jumbo frames pour absorber l'overhead VXLAN de 50 octets)
- Interfaces Spine ↔ FW (`10.0.20.0/24`) configurées sur les Spine, DOWN côté FW pour l'instant

**Point d'attention MTU :** le MTU doit être configuré sur l'interface GNS3 **et** dans vEOS. En GNS3, vEOS hérite du MTU par défaut de QEMU (1500). Il faut forcer le MTU à 9214 dans EOS ET agrandir le MTU dans GNS3 via les propriétés du lien si on veut tester les jumbo frames en réel. Dans un premier temps, configurer le MTU côté EOS suffit pour monter les adjacences OSPF — les jumbo frames seront vérifiées lors des tests VXLAN.

**Vérification :**
```
! Arista EOS
show ip ospf neighbor
show ip route ospf
ping 10.0.0.2 source loopback0    ! Spine-1 ping Spine-2 via Leaf
ping 10.0.0.11 source loopback0   ! Spine-1 ping Leaf-1
show interfaces ethernet1/1 | grep MTU
```

---

### Étape 4 — DC overlay (BGP EVPN — peerings, VRF, VNI, anycast gateway)

**Objectif :** monter l'overlay BGP EVPN. C'est l'étape la plus complexe du projet. On la décompose en sous-étapes.

**Prérequis :** Étape 3 terminée — OSPF DC stable, toutes les loopbacks joignables.

**Sous-étape 4a — BGP EVPN control plane (peerings iBGP)**
- Spine : configurer comme Route Reflector iBGP AS 65000, EVPN address family, peer avec tous les Leaf
- Leaf : configurer comme RR clients, peer avec les deux Spine

```
! Vérification 4a
show bgp evpn summary         ! sur Spine : tous les Leaf en Established
show bgp evpn summary         ! sur Leaf : les deux Spine en Established
```

**Sous-étape 4b — VRF et VNI L3**
- Sur chaque Leaf : créer les VRF (PROD, DMZ, ADMIN, MGMT), configurer le VNI L3 par VRF (50001-50004), Route Distinguisher et Route Target selon le plan

```
! Vérification 4b
show vrf                      ! 4 VRF présentes sur chaque Leaf
show bgp evpn route-type ip-prefix    ! routes type 5 échangées
```

**Sous-étape 4c — VNI L2 et VTEP**
- Sur chaque Leaf : créer les VXLAN mappings VLAN → VNI (10000+VLAN)
- Associer chaque VLAN à sa VRF
- Configurer les VTEP (source = loopback)

```
! Vérification 4c
show vxlan vtep                ! VTEP distant appris
show bgp evpn route-type mac-ip       ! routes type 2 (MAC/IP)
show vxlan address-table              ! table MAC VXLAN locale
```

**Sous-étape 4d — Anycast gateway**
- Sur chaque Leaf : configurer les SVI avec l'IP anycast partagée (même IP + même MAC virtuelle sur tous les Leaf pour chaque VLAN)
- Activer ARP suppression

```
! Vérification 4d (avec une VM de test sur HV)
ping <anycast-gw-IP> from VM    ! réponse depuis le Leaf local
arping -I eth0 <anycast-gw-IP>  ! même MAC anycast de tous les Leaf
```

**Vérification globale Étape 4 :**
Lancer une VM de test sur HV-1 et une sur HV-2 dans le même VLAN. Vérifier que les deux se pinguent via VXLAN sans que le FW soit impliqué.

---

### Étape 5 — Proxmox (bridges, cluster, QDevice, PBS, PDM)

**Objectif :** configurer les hyperviseurs Proxmox, former le cluster HV-1/HV-2, configurer le QDevice sur HV-3.

**Prérequis :** Étape 4 terminée — EVPN overlay fonctionnel, VNI 10999 (MGMT-INFRA) étendu dans la fabric. Les interfaces de management Proxmox (VLAN 999) doivent être joignables via la fabric.

**Périmètre :**
- HV-1 et HV-2 : bridges `vmbr0` (management VLAN 999), `vmbr-prod`, `vmbr-dmz`, `vmbr-admin` (tous VLAN-aware), interface physique `ens3` en trunk 802.1Q vers Leaf
- HV-3 : même bridges, uniquement VLAN 999 + VLAN 300-306 (ADMIN)
- Cluster Corosync HV-1/HV-2 : `pvecm create` sur HV-1, `pvecm add` sur HV-2
- QDevice : installer `corosync-qnetd` sur HV-3, l'ajouter au cluster
- PBS sur HV-3 : VM Proxmox Backup Server, configuration des datastores
- PDM sur HV-3 : VM Proxmox Datacenter Manager

**Point d'attention quorum :** un cluster Proxmox à 2 nœuds sans QDevice passe en read-only si un nœud est perdu (perte de quorum — majorité = 2/2 requis, 1/2 insuffisant). Le QDevice sur HV-3 apporte la troisième voix. **Ne pas tester la HA du cluster avant d'avoir le QDevice configuré.**

**Vérification :**
```bash
pvecm status            # quorum OK, 3 votes (HV-1 + HV-2 + QDevice)
pvecm nodes             # liste des nœuds
# Tester migration live d'une VM test HV-1 → HV-2 sans interruption
```

---

### Étape 6 — pfSense HA (CARP, pfsync, interfaces, FRR OSPF, NAT, règles)

**Objectif :** connecter les deux îlots OSPF (LAN et DC) via le firewall. À cette étape, tout le reste fonctionne — le FW n'a qu'à rejoindre les deux domaines.

**Prérequis :** Étapes 1-5 terminées. LAN validé, DC validé, Proxmox opérationnel.

**Ordre de configuration interne pfSense :**

1. **Installation et interfaces** — assigner les interfaces em0-em5 selon le plan (em0=pfsync, em1=WAN, em2=Spine-1, em3=Spine-2, em4=Core-1, em5=Core-2)

2. **HA CARP + pfsync** — configurer FW-1 en master, FW-2 en backup. Lien pfsync sur em0 (`10.0.255.0/30`). Tester la bascule AVANT de configurer le routing.

3. **Sous-interfaces VLAN (handoff VRF)** — em2.201 à em2.204 et em3.201 à em3.204, avec les IP de transit du plan d'adressage.

4. **VIP CARP** — les VIP publiques simulées (`192.168.122.10-14`, `.200`) sur em1.

5. **FRR OSPF** — activer le package FRR, configurer OSPF area 0 sur em4/em5 (Core LAN) et em2/em3 (Spine DC). Annoncer la default route apprise du WAN. C'est ici que les deux îlots se rejoignent.

6. **NAT outbound** — mode manuel, règles SNAT pour LAN (`10.1.0.0/16`), DC PROD, DC DMZ vers `192.168.122.200`.

7. **Règles firewall** — deny-by-default, puis règles explicites par interface. Commencer par les règles minimales pour valider la connectivité end-to-end, affiner ensuite.

**Vérification :**
```
! pfSense CLI (option 8 shell)
frrctl show ip ospf neighbor    ! adjacences avec Core et Spine
frrctl show ip route            ! routes LAN et DC apprises

! Depuis W-CLIENT (VLAN 10)
ping 10.0.0.11                  ! loopback LEAF-1 via FW
ping 8.8.8.8                    ! Internet via NAT FW

! Bascule CARP
# Éteindre FW-1 dans GNS3 → FW-2 doit reprendre toutes les VIP
# Les sessions TCP en cours (ex. ping continu) ne doivent pas être interrompues (grâce à pfsync)
```

---

### Étape 7 — Validation end-to-end

**Objectif :** valider les scénarios fonctionnels critiques de bout en bout. Chaque scénario est un test de régression qui garantit que l'ensemble de la stack fonctionne ensemble.

| Scénario | Chemin validé | Commande/Action |
|---|---|---|
| LAN → Internet | W-CLIENT → FW → NAT Cloud | `ping 8.8.8.8` depuis W-CLIENT |
| LAN → DC intra-VRF | W-CLIENT → Core → FW → Spine → Leaf → VM PROD | `ping 10.2.100.X` depuis W-CLIENT |
| DC intra-VRF (Cas A) | VM-PROD HV-1 → VM-PROD HV-2 sans FW | capture Wireshark sur Spine : pas de flux vers FW |
| DC inter-VRF U-turn (Cas B) | VM-DMZ → VM-PROD via FW | capture Wireshark : flux voit le FW |
| Bascule CARP | Panne FW-1 → FW-2 reprend | `ping continu` + `shutdown FW-1` dans GNS3 |
| Convergence OSPF | Panne lien Core ↔ Distribution | `shutdown interface` + observer reconvergence |
| Live migration Proxmox | VM migre HV-1 → HV-2 | `qm migrate` sans interruption du ping |
| VPN WireGuard | PC-EXT → VIP CARP 192.168.122.10 | test depuis VM hors fabric |
| 802.1X Access | Poste non-authentifié rejeté | test avec supplicant désactivé |

---

## Phase 2 — Services applicatifs

Démarrera une fois tous les scénarios de validation de la Phase 1b passés.

Chaque service suivra le canevas : notion/concept → état du marché → sélection argumentée → dimensionnement → installation → sécurisation → administration → intégration avec le SI.

Ordre recommandé (à affiner) :
1. Services prérequis pour le reste (DNS interne, NTP, DHCP central)
2. Annuaire (Samba ADDC ou FreeIPA)
3. AAA (FreeRADIUS, Keycloak)
4. Supervision (Prometheus/Grafana, Zabbix)
5. Bastion (Guacamole)
6. CI/CD (GitLab)
7. Secrets & PKI (HashiCorp Vault)
8. SIEM/SOC (Wazuh, Graylog)
9. Services DMZ (HAProxy, BunkerWeb, DNS public, SMTP)
10. Services complémentaires (NextCloud, ITSM, Automation)

---

## Phase 3 — Conformité SI

À venir. Couvrira : référentiels (ISO 27001, ANSSI PGSSI-S, NIS2), cartographie des risques, PCA/PRA, gouvernance, plan de remédiation.
