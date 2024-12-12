# Ubuntu 24.04 -- Pequeno e Seguro

Neste repositório compartilharei algumas dicas para usuários interessados em instalações altamente customizadas do Ubuntu (aos moldes do que fazemos no Arch Linux).

Confesso que quando o Ubuntu 24.04 LTS ficou pronto, fiquei decepcionado com o instalador. Apesar dele ter sido totalmente reconstruído em Flutter, faltam funcionalidades básicas. Não é possível fazer particionamentos avançados combinando Luks, BTRFS, ZFS e outras opções, principalmente em layouts com dual boot.

Quero um setup com dual boot, com o sistema em uma partição Luks com BTRFS sem LVM, e sem Swap. Tenho um SSD de 1TB.

## Setup

### 1. Particionamento inicial

Layout desejado:

| Tamanho | Montagem  | Filesystem           |
| ------- | --------- | -------------------- |
| 1 GiB   | /boot/efi | EFI System Partition |
| 250 GiB | Windows   | NTFS                 |
| 700 GiB | /         | LUKS (BTRFS)         |

Para a partição BTRFS, os seguintes subvolumes serão criados:

| @     | /     |
| ----- | ----- |
| @var  | /var  |
| @home | /home |
| @opt  | /opt  |
| @srv  | /srv  |
| @root | /root |

```bash
# Virar root no host para não precisar usar sudo o tempo inteiro
sudo su -

# Instalar pacotes necessários
apt update
apt install arch-install-scripts debootstrap

# Particionar de acordo com a tabela acima
cfdisk /dev/nvme0n1

# Formatar as partições
mkfs.vfat -F 32 -n EFI /dev/nvme0n1p1
mkfs.ntfs --fast --label WINDOWS /dev/nvme0n1p2
cryptsetup luksFormat --label=cryptlinux /dev/nvme0n1p3
cryptsetup open /dev/nvme0n1p3 cryptlinux
mkfs.btrfs --label cryptlinux /dev/mapper/cryptlinux

# Criar os subvolumes
cd /mnt
mkdir target temp
mount /dev/mapper/cryptlinux /mnt/temp
btrfs subvolume create /mnt/temp/@
btrfs subvolume create /mnt/temp/@var
btrfs subvolume create /mnt/temp/@home
btrfs subvolume create /mnt/temp/@opt
btrfs subvolume create /mnt/temp/@srv
btrfs subvolume create /mnt/temp/@root
umount /mnt/temp
mount /dev/mapper/cryptlinux -o defaults,noatime,autodefrag,compress-force=zstd:1,space_cache=v2,discard=async,subvol=@ /mnt/target
mount /dev/mapper/cryptlinux -o defaults,noatime,autodefrag,compress-force=zstd:1,space_cache=v2,discard=async,subvol=@var /mnt/target/var
mount /dev/mapper/cryptlinux -o defaults,noatime,autodefrag,compress-force=zstd:1,space_cache=v2,discard=async,subvol=@home /mnt/target/home
mount /dev/mapper/cryptlinux -o defaults,noatime,autodefrag,compress-force=zstd:1,space_cache=v2,discard=async,subvol=@opt /mnt/target/opt
mount /dev/mapper/cryptlinux -o defaults,noatime,autodefrag,compress-force=zstd:1,space_cache=v2,discard=async,subvol=@srv /mnt/target/srv
mount /dev/mapper/cryptlinux -o defaults,noatime,autodefrag,compress-force=zstd:1,space_cache=v2,discard=async,subvol=@root /mnt/target/root
mkdir -p /mnt/target/boot/efi
mount -o defaults,nosuid,nodev,relatime,errors=remount-ro /dev/nvme0n1p1 /mnt/target/boot/efi

# Bootstrap do Ubuntu 24.04
debootstrap noble /mnt/target http://br.archive.ubuntu.com/ubuntu

# Impedir que alguns pacotes sejam instalados
cat > /mnt/target/etc/apt/preferences.d/ignored-packages <<EOF
Package: snapd cloud-init landscape-common popularity-contest ubuntu-advantage-tools
Pin: release *
Pin-Priority: -1
EOF

# Criar os repositórios do apt
cat > /mnt/target/etc/apt/sources.list <<EOF
deb http://br.archive.ubuntu.com/ubuntu noble           main restricted universe
deb http://br.archive.ubuntu.com/ubuntu noble-security  main restricted universe
deb http://br.archive.ubuntu.com/ubuntu noble-updates   main restricted universe
deb http://br.archive.ubuntu.com/ubuntu noble-backports   main restricted universe
EOF

# Fazer CHROOT para o ambiente Ubuntu
arch-chroot /mnt/target

# Instalar pacotes básicos
apt update
apt dist-upgrade -y
apt install --no-install-recommends \
  linux-{,image-,headers-}generic-hwe-24.04 \
  linux-firmware initramfs-tools efibootmgr \
  btrfs-progs curl wget dmidecode ethtool firewalld fwupd gawk git gnupg htop man \
  needrestart openssh-server patch screen software-properties-common tmux zsh zstd \
  grub-efi-amd64 gnome flatpak gnome-software-plugin-flatpak gdm3 cryptsetup-initramfs \
  plymouth plymouth-label plymouth-theme-spinner plymouth-theme-ubuntu-text 

# Configurar o crypttab com o volume LUKS
echo "cryptlinux /dev/nvme0n1p3 none luks" > /etc/crypttab

# Configurar bootloader
grub-install /dev/nvme0
grub-install /dev/nvme0n1
grub-install /dev/nvme0n1p1
mkdir -p /etc/default/grub.d
echo "GRUB_ENABLE_CRYPTODISK=y" > /etc/default/grub.d/local.cfg
update-grub
update-initramfs -u -k all

# Usuários
useradd -m wesley \
  -c "Wesley Rodrigues" \
  -G sudo \
  -s /bin/zsh

echo "root:12345678" | chpasswd
echo "wesley:12345678" | chpasswd

# Hostname e hosts
echo "linuxdragon" > /etc/hostname
echo "127.0.0.1 llocalhost" > /etc/hosts
echo "127.0.1.1 linuxdragon" >> /etc/hosts

# Configurar locais e idiomas
dpkg-reconfigure tzdata
dpkg-reconfigure locales
dpkg-reconfigure keyboard-configuration

# Fim. Sente-se com sorte? Reinicie e teste :)
exit
reboot
```

This is the end
