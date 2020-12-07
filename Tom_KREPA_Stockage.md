# Partie 1

* Creation de 3 raid 5 sur les disques
	* sudo mdadm --create /dev/md/raidn1 -l5 -n3 /dev/sdb /dev/sdc /dev/sdd
	* sudo mdadm --create /dev/md/raidn2 -l5 -n3 /dev/sde /dev/sdf /dev/sdg
	* sudo mdadm --create /dev/md/raidn3 -l5 -n3 /dev/sdh /dev/sdi /dev/sdj

* Creation des PV
	* sudo pvcreate /dev/md127
	* sudo pvcreate /dev/md126
	* sudo pvcreate /dev/md125
	
* Creation des VG
	* sudo vgcreate production /dev/md127
	* sudo vgcreate preproduction /dev/md126
	* sudo vgcreate sauvegarde /dev/md125
	
* Creation des LV
	* sudo lvcreate -L 4G -n courant production
	* sudo lvcreate -L 4G -n en-attente production
	* sudo lvcreate -L 4G -n courant preproduction
	* sudo lvcreate -L 4G -n en-attente preproduction
	* sudo lvcreate -L 4G -n courant sauvegarde
	
* Formatage des partitions en ext4 et xfs
	* sudo mkfs -t ext4 /dev/mapper/production-courant
	* sudo mkfs -t ext4 /dev/mapper/production-en--attente
	* sudo mkfs -t ext4 /dev/mapper/preproduction-courant
	* sudo mkfs -t ext4 /dev/mapper/preproduction-en--attente
	* sudo mkfs -t xfs /dev/mapper/sauvegarde-courant
	
* Ajout d'un label pour chaque partitions
	* sudo tune2fs -L courant-fs /dev/mapper/production-courant
	* sudo tune2fs -L en-attente-fs /dev/mapper/production-en--attente
	* sudo tune2fs -L courant-fs /dev/mapper/preproduction-courant
	* sudo tune2fs -L en-attente-fs /dev/mapper/preproduction-en--attente
	* sudo xfs_admin -L courant-fs /dev/mapper/sauvegarde-courant
	
* Creation des points de montage
	* sudo mkdir /mnt/production
	* sudo mkdir /mnt/production/courant
	* sudo mkdir /mnt/production/en-attente
	* sudo mkdir /mnt/preproduction
	* sudo mkdir /mnt/preproduction/courant
	* sudo mkdir /mnt/preproduction/en-attente
	* sudo mkdir /mnt/sauvegarde
	* sudo mkdir /mnt/sauvegarde/courant
	
* Montage des partitions
	* sudo mount -t ext4 /dev/mapper/production-courant /mnt/production/courant
	* sudo mount -t ext4 /dev/mapper/production-en--attente /mnt/production/en-attente
	* sudo mount -t ext4 /dev/mapper/preproduction-courant /mnt/preproduction/courant
	* sudo mount -t ext4 /dev/mapper/preproduction-en--attente /mnt/preproduction/en-attente
	* sudo mount -t xfs /dev/mapper/sauvegarde-courant /mnt/sauvegarde/courant
	
* Modification du fichier fstab et ajout des lignes suivantes avec sudo nano /etc/fstab
	* /dev/mapper/production-courant	/mnt/production/courant			ext4	defaults	0 0
	* /dev/mapper/production-en--attente	/production/en-attente			ext4	defaults	0 0
	* /dev/mapper/preproduction-courant		/preproduction/courant			ext4	defaults	0 0
	* /dev/mapper/preproduction-en--attente		/preproduction/en-attente			ext4	defaults	0 0
	* /dev/mapper/sauvegarde-courant	/mnt/sauvegarde/courant			xfs		defaults	0 0
	
# Partie 2

* Declarer le disque comme defaillant
	* sudo mdadm --manage /dev/md126 --set-faulty /dev/sdb
	
* Retirer le disque defaillant
	* sudo mdadm --manage /dev/md126 --remove /dev/sdb
	
* Ajouter un nouveau disque
	* sudo mdadm --manage /dev/md126 --add /dev/nouveau_disque
	
# Partie 3

* Agrandissement du disque dans sauvegarde
	* sudo lvextend -l60%FREE /dev/mapper/sauvegarde-courant

# Partie 4

* Installation de samba
	* sudo apt install samba samba-common samba-testsuite
	
* Edition du fichier de conf
	* sudo nano /etc/samba/smb.conf
	* [global]
	* netbios name = SAMBA
	* workgroup = WORKGROUP
	* server string = Samba, Debian
	* dsn proxy = no
	* logfile = /var/log/samba/log.%m
	* max log size = 1000
	* syslog = 0
	* server role = standalone server
	* passdb backend = tdbsam
	* obey pam restrictions = yes
	* unix password sync = yes
	* passwd program = /usr/bin/passwd %u
	* pam password change = yes
	* map to guest = bad user
	* usershare allow guests = yes
	* browsable = yes
	* security = user
	
	* [PARTAGE EVAL]
	* path = /mnt/preproduction/courant
	* readonly = no
	* writable = yes
	* browsable = yes
	* valid users = jerome karim fernanda
	
* Verification du fichier de conf avec testparm
	* testparm

* Creation des utilisateurs
	* sudo useradd jerome
	* sudo smbpasswd-a jerome
	* sudo passwd jerome
	* sudo useradd karim
	* sudo smbpasswd-a karim
	* sudo passwd karim
	* sudo useradd fernanda
	* sudo smbpasswd-a fernanda
	* sudo passwd fernanda
	
* Relancement des services nmbd et smbd
	* sudo systemctl restart nmbd.service smbd.service

# Partie 5

* Destruction du LV preproduction/en-attente
	* sudo lvremove /dev/preproduction/en-attente
	
* Creation du partition snapshot qui remplace preproduction-en-attente
	* sudo lvcreate -s -l100%FREE -n snaps_production /dev/mapper/production-courant