Installation du cluster Proxmox
============================

# Installation d'un PVE

# Mise en place du cluster

# Mise en place du Ceph

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


### Haute disponibilité