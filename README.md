# Installation ArchLinux

## Avant l'installation

- Vérifier que la carte mère supporter l'UEFI (Sur VirtualBox il faut cocher l'option EFI lors de la création de la VM).

## Configuration et vérifications de base

```bash
# Défini le layout du clavier en : FR
loadkeys fr

# Vérifier le nombre de bits UEFI (doit retour 64, les systèmes 32 bits ne sont plus supportés par ArchLinux)
cat /sys/firmware/efi/fw_platform_size

# Vérifier que la connexion internet est active
ping archlinux.org
```

## Partionnement du disque

```bash
# Commencer le partitionnement du disque qui contiendra linux (en supposant que c'est /dev/sda)
fdisk /dev/sda
```

Faire cette suite de saisies dans l'ordre pour créer les partitions sur le disque :

1. n, e, Enter, Enter, +300M, t, EF
2. n, e, Enter, Enter, +8G, t, Enter, 82
3. n, e, Enter, Enter, Enter, t, Enter, 83
4. w

Ces actions on permis de créer 3 partitions sur le disque comme suit : 

```
|--- /dev/sda1 (EFI - 300 MB)
|--- /dev/sda2 (SWAP - 8 G)
|--- /dev/sda3 (LINUX - XXX G)
```

- La partition `/dev/sda1` sera la partition les programmes de démarrage comme le grub.
- La partition `/dev/sda2` sera la partition de swap.
- La partition `/dev/sda3` contiendra les fichiers du système.

## Formattage des partitions

```bash
mkfs.fat -F 32 /dev/sda1
mkswap /dev/sda2
mkfs.ext4 /dev/sda3

# Activation de la partition de swap
swapon /dev/sda2
```

## Montage des fichiers

```bash
# Monte la partition LINUX dans le répertoire /mnt
mount /dev/sda3 /mnt

# Monte la partition EFI dans le répertoire /mnt/boot
mount --mkdir /dev/sda1 /mnt/boot
```

## Installation des packages

```bash
# Recherche et tri les serveurs les plus à même de télécharger rapidement depuis les 12 dernières heures
reflector --country France --age 12 --protocol https --sort rate --save /etc/pacman.d/mirrorlist

# Installation des paquets de base absolument nécessaire (Networkmanager est important pour pouvoir se connecter à internet une fois le ArchLinux installé)
pacstrap -K /mnt base linux linux-firmware grub efibootmgr vim networkmanager
```

## Configuration du système

```bash
# Générer le fichier fstab (pour monter automatiquement les partitions au démarrage) à partir des montages précedemment fait
genfstab -U /mnt >> /mnt/etc/fstab

# Se connecter en temps que root sur le système nouvellement installé disponible sur /mnt
arch-chroot /mnt

# Activer le gestionnaire réseau. Permettra d'être connecté à internet dès le reboot
systemctl enable NetworkManager

# Selectionner et synchroniser l'heure
ln -sf /usr/share/zoneinfo/Europe/Paris /etc/localtime
hwclock --systohc

# Décommenter la ligne de locale voulue dans /etc/locale.gen puis exécuter la commande 
locale-gen

# Créer le fichier /etc/locale.conf et y mettre le contenu suivant
LANG=fr_FR.UTF-8

# Éditer le fichier /etc/vconsole.conf et y mettre le contenu suivant
KEYMAP=fr

# Créer le fichier /etc/hostname et y mettre le contenu suivant
<nom_ordinateur>

# Définir un mot de passe pour l'utilisateur root
passwd
```

## Installation du secteur d'amorçage (grub)

```bash
# Installation de grub
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=grub --recheck

# Création du fichier de config de grub
grub-mkconfig -o /boot/grub/grub.cfg
```

## Redémarrage

```bash
# Quitter l'utilisateur root du système et utiliser l'utilisateur root du système "live" (Ctrl + D fonctionne aussi)
exit

# Démonte tous les points de montage dans /mnt
umount -R /mnt

# Redémarrage
reboot
```

## Ajout d'un utilisateur non-root

```bash
# Ajouter un utilisateur au group sudo et définir son mot de passe
useradd -m -G wheel -s /bin/bash <username>
passwd <username>

# Décommenter la ligne suivante dans /etc/sudoers
%wheel ALL=(ALL:ALL) ALL
```

## Installation des systèmes graphiques

À partir de maintenant, toutes les actions doivent être effectuées avec l'utilisateur non-root précédemment créé.

### Installation du serveur graphique Xorg

```bash
# Installation du serveur Xorg
sudo pacman -S xorg-apps xorg-server xorg-xinit xf86-input-libinput

# Installation d'un driver Xorg correspondant au matériel graphique (sur Virtual Box xf86-video-vesa fonctionne bien)
sudo pacman -S <xorg_driver>
```

### Installation d'un gestionnaire de connexion (lightdm)

```bash
# Installation du serveur Xorg
sudo pacman -S lightdm-gtk-greeter

# Éditer le fichier /etc/lightdm/lightdm.conf et dans la section [Seat:*] modifier la ligne
# greeter-session=example-gtk-gnome
sudo greeter-session=lightdm-gtk-greeter

# Activer le service lightdm, comme ça il sera lancé dès le reboot
sudo systemctl enable lightdm.service
```

### Installation d'un gestionnaire de fenêtre

```bash
# Installation du gestionnaire i3wm
sudo pacman -S i3

# Éditer le fichier ~/.config/i3/config et remplacer la touche utliser pour fermer la fenêtre ayant le focus
bindsym $mod+Shift+A kill

# Éditer le fichier ~/.config/i3/config et remplacer la touche pour passer en mod "tabbed"
bindsym $mod+z tabbed
```

### Installation et configuration d'autres outils

#### Additions invité (Virtual Box)

```bash
# Installer les additions invité de Virtual Box si notre système est installé sur une machine virtuelle
sudo pacman -S virtualbox-guest-utils
```

#### Layout du clavier pour Xorg

```bash
# Définir la layout du clavier dans Xorg à FR. La session Xorg doit être rechargée pour que ça prenne effet (logout/login avec i3 ou reboot)
 sudo localectl set-x11-keymap fr
```

#### Terminal virtuel

```bash
# Installation d'un terminal
sudo pacman -S alacritty

# Éditer le fichier ~/.config/i3/config et remplacer le terminal utilisé i3-sensible-terminal
bindsym $mod+Return exec alacritty
```

#### Launcher d'application

```bash
# Installation de rofi
sudo pacman -S rofi

# Éditer le fichier ~/.config/i3/config et remplacer dmenu_run par rofi drun
#
# Remplacer : bindsym $mod+d exec --no-startup-id demenu_run
# par
bindsym $mod+d exec --no-startup-id rofi -show drun
```

#### XDG-USER-DIRS

```bash
# Installation de xdg-user-dirs
sudo pacman -S xdg-user-dirs

# Création de la config par défaut pour l'utilisateur courant
xdg-user-dirs-update
```


Éditer la config précedemment créée pour modifier les répertoires par défaut : 

`~/.config/user-dirs.dirs`

```conf
XDG_DESKTOP_DIR="$HOME"
XDG_DOCUMENTS_DIR="$HOME"
XDG_DOWNLOAD_DIR="$HOME"
XDG_MUSIC_DIR="$HOME"
XDG_PICTURES_DIR="$HOME"
XDG_PUBLICSHARE_DIR="$HOME"
XDG_TEMPLATES_DIR="$HOME"
XDG_VIDEOS_DIR="$HOME"
```

#### Gestionnaire de fichier 

```bash
# Installation de nemo
sudo pacman -S nemo
```

#### Base-Devel (pour AUR)

Ce paquet est utile si on veut installer des paquets provenants d'AUR.

```bash
pacman -S base-devel
```

#### Audio

```bash
# Installer les outils audio
sudo pacman -S pipewire pipewire-pulse pavucontrol
```

Ajouter ces lignes au fichier  `~/.xinitrc`

```
/usr/bin/pipewire &
/usr/bin/pipewire-pulse &
/usr/bin/pipewire-pulse-session &
```

Le menu audio sera accessible avec la commande/runner `pavucontrol`.