# Self hosting de Github Actions dans des machines virtuelles
Self-hosting de Github Actions, avec accès à distance

*Problème initial:*
L'utilisation des Github Actions sur des dépots mirroirs privés peut dépasser les quotas offerts par Github.
La seule solution pour palier le problème est de self-host un runner GA ou de passer le dépôt en public (ce qui n'est pas idéal), ou de payer.

**Cette installation sera utilisé sur les futurs projets tel que R-Type et l'EIP.**

## Materiel:
- Un PC portable Thinkpad T460 fera office de serveur (Core I7 6600U - 2c/4t - Upgradé à 16GB de DDR3 - Upgrade SSD à 480GB)
- Une connexion internet stable (de préférence fibre, n'utilisant pas de mode CGNAT comme une box 4G).

## Installation d'un hyperviseur
Afin de faciliter la maintenance et de pouvoir self-host plusiurs types de machines pour les Github Actions (Linux et Windows), il est nécessaire de passer par un hyperviseur.
Le choix c'est porté sur [Proxmox Virtual Environment](https://www.proxmox.com), un OS de virtualisation Open Source basé sur Debian, avec une interface web de management.
Proxmox offre de trés bonne performance pour les VMs, puisqu'il s'appuie sur QEMU/KVM.

*L'installation se fait facilement depuis le LiveCD en interface graphique.*

Une adreses ipv4 static locale doit être configuré afin d'accéder facilement au serveur et aux VMs.

Une fois l'installation terminé, Proxmox est prêt à créer et lancer des VMs.

## Config spécifique à un PC portable
- Par défaut lorque le clapet du PC portable est fermé, il se met en veille automatiquement. Cela pose un problème dans notre cas.
  Afin de désactiver ce comportmenet, il faut éditer le fichier `/etc/systemd/logind.conf` et modifier le lid switch vers: `HandleLidSwitch=ignore`
- Il est également judicieux d'éteindre l'écran après un certain délai pour économiser de l'énergie.
  Pour cela il faut éditer les arguments de lancement du noyaux dans `/etc/default/grub` et ajouter `consoleblank=60` dans `GRUB_CMDLINE_LINUX_DEFAULT`
  Puis appliquer les modifications avec `update-grub`


## Installation d'une VM Linux (Debian) pour host les github actions
Allocation des ressources:
2 vcpus, 6GB Ram, 100GB disque.

Debian, rapide et facile à installer. Léger en ressource.
- Installation des paquets du dump EPITECH.
- Instalation de docker en rootless.
- Installation du runner Github action, configuration d'un service.

Fix: création d'un symlink (en tant que service au démarrage) vers le socket docker car les conteneurs dockers n'arrivaient pas à se lancer en rootless.


## Installation d'une VM Windows pour host les github actions
Allocation des ressources:
2 vcpus, 6GB Ram, 100GB disque.

- Installation de windows 10: *Nécessite d'installer les pilotes [VirtIO](https://pve.proxmox.com/wiki/Windows_VirtIO_Drivers) durant l'installation pour détecter le disque et le réseau.*
- Installation de [Chocolatey](https://chocolatey.org/), Visual Studio 2022, MinGW (pour GNU, GCC ...)
- Installation d'un serveur OpenSSH et configuration de PowerShell en interpréteur de command par défaut.
- Installation du service Github Action


# Accès à distance
Il est nécessaire de pouvoir administrer, et faires des changements sur les VMs à distance car le PC n'est pas chez moi (car je passe par une box 4G donc impossible de faire un DDNS à cause du CGNAT).
Tout cela necessite donc l'installation d'une ultime VM Debian (1vcpu, 2GB Ram, 30GB disque) dédiée à la gestion interne du réseau.

## VPN
Pour se connecter à distance, la solution que j'ai retenu à été la mise en place d'un serveur VPN (PI-VPN/Wireguard).
Cependant, l'address IP publique de la livebox change régulièrement, ce qui rends la connexion innacessible au bout d'un certain moment si l'on code l'IP publique de l'endpoint (la box) en dur dans la config du VPN.

## DDNS (API Cloudflare)
Pour palier à ce problème, il faut avoir (ou dans mon cas acheter) un nom de domaine et mettre en place un DDNS.
Ce nom de domaine permet aux serveurs DNS (tel que cloudflare, sur lequel j'ai lié mon domaine) de pointer sur l'adresse IP publique du réseau sur lequel est hébergé le PC.

L'adresse IP publique vers laquelle le domaine pointe devra être mise à jour à chaque changement d'IP grâce à un DDNS:
Pour ce projet, j'utilise un script Bash: [cloudflare-ddns-updater](https://github.com/K0p1-Git/cloudflare-ddns-updater) lancé toutes les minutes par le service Cron. Ce script se connecte à l'API de Cloudflare et met à jour l'adresse IP. 

## Ouverture des ports

Enfin, afin d'autoriser les connexions distantes sur le PC, il faut ouvrir le port 51820 de la box vers la VM debian interne pour faire fonctionner le VPN.


# Fin et améliorations possibles
- Ajout des runners sur une organisation Gitub pour pouvoir l'autoriser sur plusieurs dépots.
- Configuration du PC pour booter dés qu'il est branché (en cas de coupure d'électricité, il redémarre tout seul) et activation du Wake On Lan.
- Désactivation des périphériques inutiles (Wifi, Bluetooth, Lecteur de carte et Audio) pour économiser de l'énergie.

Améliorations :

- Une amélioration notable serait de créer des VLans internes aux machines virtuelles qui hébergent les github actions pour les séparer du réseau local afin d'augmenter la sécurité du réseau.
La livebox ne proposant pas cette fonctionnalité, il faudrait probablement installer un OS routeur tel que [OPNsens](https://opnsense.org/).

- La VM hébergeant le VPN et le DDNS est configurée sur le même subnet que la majorité des autres subnets fournis sur les box des fournisseurs d'accès à internet (192.168.1.0/24). Or, cela pose un problème de conflit quand je me connecte depuis un réseau avec le même subnet. Il faudrait donc changer le subnet du réseau sur lequel le PC est hébergé vers un subnet moins commun pour éviter ces problèmes.

## Screenshots
![image](https://github.com/user-attachments/assets/0b49e017-4ff8-4245-9cc1-5ff5d6975064)


![image](https://github.com/user-attachments/assets/566e27b6-5d13-4327-915d-ada503787d69)


