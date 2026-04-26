# Configurations des équipements

Ce dossier contient les configurations CLI de chaque équipement de la maquette GNS3.

**Convention de nommage :** `NOM-EQUIPEMENT.txt` pour les outputs `show running-config` bruts. Les équipements Proxmox utilisent des fichiers `.md` car leur configuration mêle CLI bash, fichiers de config et commandes GUI.

**Source de vérité adressage :** `../plan_adressage.xlsx` — toute IP dans les configs ci-dessous en est extraite.

---

## État des configurations

| Équipement | Zone | Fichier | Base | L2/L3 | Proto | Sécurité | État |
|---|---|---|:---:|:---:|:---:|:---:|---|
| ACC-1 | LAN | `lan/ACC-1.txt` | ⬜ | ⬜ | ⬜ | ⬜ | À faire |
| ACC-2 | LAN | `lan/ACC-2.txt` | ⬜ | ⬜ | ⬜ | ⬜ | À faire |
| ACC-3 | LAN | `lan/ACC-3.txt` | ⬜ | ⬜ | ⬜ | ⬜ | À faire |
| ACC-4 | LAN | `lan/ACC-4.txt` | ⬜ | ⬜ | ⬜ | ⬜ | À faire |
| DIST-1 | LAN | `lan/DIST-1.txt` | ⬜ | ⬜ | ⬜ | ⬜ | À faire |
| DIST-2 | LAN | `lan/DIST-2.txt` | ⬜ | ⬜ | ⬜ | ⬜ | À faire |
| DIST-3 | LAN | `lan/DIST-3.txt` | ⬜ | ⬜ | ⬜ | ⬜ | À faire |
| DIST-4 | LAN | `lan/DIST-4.txt` | ⬜ | ⬜ | ⬜ | ⬜ | À faire |
| CORE-1 | LAN | `lan/CORE-1.txt` | ⬜ | ⬜ | ⬜ | ⬜ | À faire |
| CORE-2 | LAN | `lan/CORE-2.txt` | ⬜ | ⬜ | ⬜ | ⬜ | À faire |
| SPINE-1 | DC | `dc/SPINE-1.txt` | ⬜ | ⬜ | ⬜ | ⬜ | À faire |
| SPINE-2 | DC | `dc/SPINE-2.txt` | ⬜ | ⬜ | ⬜ | ⬜ | À faire |
| LEAF-1 | DC | `dc/LEAF-1.txt` | ⬜ | ⬜ | ⬜ | ⬜ | À faire |
| LEAF-2 | DC | `dc/LEAF-2.txt` | ⬜ | ⬜ | ⬜ | ⬜ | À faire |
| LEAF-3 | DC | `dc/LEAF-3.txt` | ⬜ | ⬜ | ⬜ | ⬜ | À faire |
| HV-1 | DC | `hv/HV-1.md` | ⬜ | ⬜ | ⬜ | ⬜ | À faire |
| HV-2 | DC | `hv/HV-2.md` | ⬜ | ⬜ | ⬜ | ⬜ | À faire |
| HV-3 | DC | `hv/HV-3.md` | ⬜ | ⬜ | ⬜ | ⬜ | À faire |
| FW-1 | Edge | `fw/FW-1.txt` | ⬜ | ⬜ | ⬜ | ⬜ | À faire |
| FW-2 | Edge | `fw/FW-2.txt` | ⬜ | ⬜ | ⬜ | ⬜ | À faire |

**Légende colonnes :** Base = hostname/SSH/AAA | L2/L3 = VLAN/interfaces/routage | Proto = OSPF/EVPN/VRRP/CARP | Sécurité = port-security/802.1X/règles FW

**Légende état :** ⬜ À faire | 🔄 En cours | ✅ Validé en lab

---

## Organisation du dossier

```
configs/
├── README.md       ← ce fichier (index + état)
├── lan/            ← Cisco IOSvL2 (Access, Distribution, Core)
├── dc/             ← Arista vEOS-lab (Spine, Leaf)
├── fw/             ← pfSense (export XML ou notes CLI FRR)
└── hv/             ← Proxmox (markdown : bridges, cluster, PBS, PDM)
```
