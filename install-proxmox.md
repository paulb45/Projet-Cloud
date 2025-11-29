Installation du cluster Proxmox
============================

Mise en place d'un cluster Proxmox sur 4 Dell PowerEdge T340 avec 4 disques de 1To.

# Installation d'un PVE

Nous avons réalisé l'installation du PVE via une clé bootable avec iso PROXMOX 9.

Pendant l'installation, il est nécessaire de choisir le mode avancé. Dans le mode "avancé" à la sélection des disques. Choisir RAIDZ-1 afin de mettre en place un raid 5.
Sélectionner les disques dur de la machine et valider.

Les autres pages peuvent être laissées tel quel. Excepté le domain et l'adresse mail du compte admin qui doivent être modifiés.

# Mise en place du cluster

Réaliser depuis l'interface graphique.dans un des pve, dans l'onglet cluster, créer un cluster. Puis copier les "join information" que vous copirer dans les autres pve (dans l'onglet cluster).   

# Mise en place du Ceph

Dans l'onglet Ceph cliquer sur installer Ceph pour chaque PVE

## Noeud manager et monitor

Dans l'interface de Proxmox mettre tous les noeuds comme manager et monitor.

## Pool ZFS et OSD

A faire sur tous les PVE du cluster.

Création du pool ZFS pour ceph
```bash
zfs create -V 2T rpool/ceph
```

Récupération des clés et préparation du volume ceph (osd).
```bash
ceph auth get client.bootstrap-osd > /etc/pve/priv/ceph.client.bootstrap-osd.keyring
ceph auth get client.bootstrap-osd > /var/lib/ceph/bootstrap-osd/ceph.keyring
ceph-volume raw prepare --data /dev/zvol/rpool/ceph --bluestore
ceph-volume raw activate --device /dev/zvol/rpool/ceph
```

Démarrage de l'OSD créé.
```bash
systemctl start ceph-osd@X
systemctl enable ceph-osd@X
```

Ajouter une unit systemd pour générer le montage du volume Ceph au démarrage du PVE. Editer le fichier */etc/systemd/system/ceph-activate.service*
```
[Unit]
Description=Activation automatique du volume Ceph
After=zfs.target
Requires=zfs.target

[Service]
Type=oneshot
ExecStart=/usr/sbin/ceph-volume raw activate --device /dev/zvol/rpool/ceph
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

Mettre au démarrage ce service
```
systemctl daemon-reload
systemctl enable ceph-activate.service
```


## Pool ceph

Création du pool ceph pour stocker les disques des VMs directement dans l'interface Proxmox.


## Pool FS ceph

Création d'un pool FS ceph directement sur Proxmox pour stocker les ISOs.


## Haute disponibilité

Dans la base de données qui éberge les pve, navigeur vers HA.
Dans le menu HA, ajouter une ressources pour chaqe VM que vous voulez être en haute dispo. Sélectionner le nombre de migration et de redémarrage souhaité.

Dans le sous menu "Affinity rules", Pour chaque vm, ajouté un "HA node affinity rule". Sélectionner tout les pve et indiquer la priorité pour chacun.