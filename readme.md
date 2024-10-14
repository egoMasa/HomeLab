# Projet HomeLab

## Objectif : Déployer une infrastructure serveurs d'entreprise avec tous les services necessaires
* Liste des services à déployer 
	* Serveur WEB Portfolio : 
        * Front : Famework Tailwind CSS 
        * Backend PHP sur serveur Apache avec fonctionnalité d'envoyer un fichier offre d'emploi
        * Serveur BDD MariaDB
	* Serveur de partage de fichier : NextCloud (Apache&PHP)
	* Serveur mail : PostFix avec BDD et accès ThunderBird
    * Serveur de supervision : Zabbix serveur et Zabbix client
	* Serveur de stockage : BackUp avec Veeam
	* Serveur DNS & DHCP : isc-dhcp et dnsmasq 
	* Firewall + VPN : Pfsense avec Wireguard
* Défi : 
	* Réussir à déployer et configurer chaque services et que tout soit fonctionnel, envoi, reception, stockage, consultation, suppression, mise à jour.
	* Capacité de consulter chaque services depuis le meme réseau interne et ensuite depuis le réseau externe passant par le firewall.
    * Gérer et comprendre la gestion des noms de domaines par services et leur accessibilité
* Autres points à gérer :
	* Mise à jour automatique des versions OS et paquets
	* Gérer les questions d'accès entre les différente services, qui à besoins d'accéder à qui ? 
	* Gestion de l'externalisation des services
	* Conception de l'archictecture interne et la passerelle entre réseau virutalbox NAT vers mon réseau domestique NAT après vers Internet
* Evolution et compléxification en cas de réussite
    * Ajout d'une fôret active Direcotory avec l'annuaire, OU, GPO, DC, Groupe
    * L'idée sera de comprendre comment implémenter AD pour limiter les accès dans un environnemet déjà déployé et fonctionnel

## Méthode de virtualisation 
* Logiciel GNS3 pour emuler et simuler tous les équipements (routeur, switch) et les différents serveurs.
* Il faut télécharge les .ISO pour chaque instance et la configurer directement dans GNS3
* Liste des ISO
    * Pfsense : Firewall + VPN
    * Poste informatique : Windows
    * Serveur Linux : ISO debian graphique (stable et rapide)
    * Routeurs et Switch : ISO cisco


## Architecture réseau de l'entreprise

[SCHEMA RESEAU]

## Configuration réseau (routeur et switch)

