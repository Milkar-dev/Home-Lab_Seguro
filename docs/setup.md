# Setup inicial del Home-Lab

## VM base
- Distro: Arch Linux (ThinkPad T470 como máquina host).
- Disco asignado: 30-50 GB.
- RAM: 4 GB para VM (host tiene 8 GB).
- Particionado: con LUKS para encriptación de disco.

## Primeros pasos realizados
1. Instalación base de Arch con `pacstrap`.
2. Creación de usuario inicial.
3. Configuración de red mínima (`systemd-networkd` + `systemd-resolved`).
4. Snapshot de estado inicial guardado en `vm_images/`.


# Instalación base de Arch Linux (Home-Lab_Seguro)

## 1. Comprobación UEFI y conexión
#### Se comprueba que el sistema arranca en modo UEFI y hay conexion a conexión
- ls /sys/firmware/efi/efivars      # Si aparece, estás en modo UEFI
- iwctl                             # Entra al gestor de red inalámbrica
- station wlan0 connect NOMBRE_RED  # Conéctate a la red WiFi
- exit
- ping archlinux.org                # Comprueba conexión a Internet



## 2. Particionado del disco
#### Se usa fdisk para crear dos particiones para un HHD (una pequeña de 512 MB para EFI):
- fdisk /dev/sda
- n → nueva partición (EFI)
- Enter → número por defecto (1)
- Enter → sector inicial por defecto
- +512M → tamaño de la partición
- t → cambia el tipo
- 1 → EFI System
- n → nueva partición (resto del disco para el sistema)
- Enter → acepta todo el espacio libre
- w → guardar y salir

#### Para crear las particiones para SSD se utiliza esto:
- fdisk /dev/nvme0n1
- n → nueva partición (EFI)
- Enter → número y sector por defecto
- +512M → tamaño EFI
- t → cambia tipo
- 1 → EFI System
- n → nueva partición (resto del disco para el sistema)
- Enter → acepta todo el espacio restante
- w → guardar y salir



## 3. Particionado del disco (con fdisk)

Ejemplo usando `/dev/sda` como disco principal.  
Crea dos particiones: una **EFI** (512 MB) y una **principal cifrada (resto del disco)**.

---

#### 1. Abrir fdisk

fdisk /dev/sda

#### 2. Si hay particiones viejas creamos una nueva tabla
Command (m for help): g        # crea nueva tabla GPT limpia

#### 3. Crear partición EFI
Command (m for help): n
Partition number: 1
First sector: <Enter>
Last sector: +512M
Command (m for help): t
Partition type: 1    # EFI System

#### 4: Crear partición raiz
Command (m for help): n
Partition number: 2
First sector: <Enter>
Last sector: <Enter> (usa todo el espacio)

#### 5. Verificar la tabla
Command (m for help): p
- Debe verse algo como:
Device      Start       End   Sectors   Size Type
/dev/sda1    2048   1050623   1048576   512M EFI System
/dev/sda2 1050624 234441647 233391024 111.3G Linux filesystem

#### 6. Guardar y salir:
Command (m for help): w





## 📦 4. Instalación del sistema base
#### Se instalan los paquetes mínimos necesarios para que el sistema pueda arrancar y disponer de red funcional.

pacstrap /mnt base linux linux-firmware vim nano sudo networkmanager lvm2 cryptsetup

#### Genera la tabla de sistemas de archivos (fstab) para que se monten automáticamente al inicio.
genfstab -U /mnt >> /mnt/etc/fstab
Para verificar qeu las entradas sean correctas usamos esto: cat /mnt/etc/fstab

#### Entramos sistema
arch-chroot /mnt




## 5. Configuración básica y chroot
#### Se establecen la zona horaria, el idioma, el nombre del equipo y el teclado.
arch-chroot /mnt /bin/bash

#### zona horaria
ln -sf /usr/share/zoneinfo/Europe/Madrid /etc/localtime
hwclock --systohc

#### locales
echo "es_ES.UTF-8 UTF-8" >> /etc/locale.gen
locale-gen
echo "LANG=es_ES.UTF-8" > /etc/locale.conf

#### hostname
echo "mi-host" > /etc/hostname
#### editar /etc/hosts
cat >> /etc/hosts <<EOF
127.0.0.1   localhost
::1         localhost
127.0.1.1   mi-host.localdomain mi-host
EOF

#### root passwd
passwd

#### crear usuario (ejemplo 'mil')
useradd -m -G wheel -s /bin/bash mil
passwd mil

#### permitir sudo para wheel (usa visudo y descomenta %wheel ALL=(ALL) ALL)
pacman -S --noconfirm sudo
EDITOR=nano visudo

#### Aquí se deja el sistema listo con la hora en Madrid, idioma español y teclado español, si deseas cambiarlo, eres libre de hacerlo




## 6. Configurar mkinitcpio y GRUB con cifrado
#### Esta parte es clave: se indica al sistema que el disco está cifrado, para que pida la contraseña al arrancar.

- Si usas el initramfs por defecto (busybox-ish), añade keyboard encrypt lvm2 antes de filesystems.
- Si usas systemd initramfs, usa sd-encrypt y lvm2 (ejemplo abajo).

HOOKS=(base systemd autodetect modconf kms keyboard sd-vconsole block sd-encrypt lvm2 filesystems fsck)
mkinitcpio -P

#### Importante: el orden importa — encrypt/sd-encrypt debe ir antes de lvm2 para que el contenedor se desbloquee antes de activar volúmenes. Revisa la página de dm-crypt y mkinitcpio.




## 7. Configurar cargadoe de arranque, servicios y usuario
#### instalar grub y efibootmgr
pacman -S --noconfirm grub efibootmgr

#### instalar grub en UEFI
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB

#### averigua UUID de la partición LUKS (UUID del superblock)
cryptsetup luksUUID /dev/sda2   # guarda ese UUID en variable

#### editar /etc/default/grub y añadir:
 GRUB_CMDLINE_LINUX="cryptdevice=UUID=TU_UUID_AQUI:cryptroot root=/dev/vg0/root"

#### si usas LVM, ejemplo:
GRUB_CMDLINE_LINUX="cryptdevice=UUID=TU_UUID_AQUI:cryptroot root=/dev/vg0/root"
#### activar que GRUB pueda desbloquear disco (opcional)
GRUB_ENABLE_CRYPTODISK=y

#### generar cfg
grub-mkconfig -o /boot/grub/grub.cfg




## 8. Finalización
- exit
- umount -R /mnt
- swapoff -a
- reboot
#### Se desmonta y se reinicia al nuevo sistema base
#### Tras reiniciar, el sistema pedirá la contraseña LUKS antes de arrancar. 
#### Si todo ha ido bien, tendrás un Arch Linux completamente funcional, cifrado y con red: la base perfecta para seguir con el proyecto de Home-Lab Seguro.



## Pendiente
Lista de tareas que quedan por configurar después de la instalación base.
Estas se irán completando a medida que avance el proyecto Home-Lab Seguro.

- Comprobar servicios activos (systemctl list-unit-files --state=enabled)
- Configurar conexión persistente de red (NetworkManager y nmcli)
- Instalar y configurar sudo correctamente (verificar permisos del grupo wheel)
- Actualizar el sistema (pacman -Syu) y limpiar paquetes innecesarios
- Instalar utilidades básicas: git, base-devel, curl, wget, man-db, openssh, etc.
- Configurar el usuario principal (mil) con su entorno y dotfiles
- Crear snapshot o backup inicial del sistema base
- Documentar la estructura del repositorio (scripts/, configs/, labs/, logs/)
- Configurar el servicio SSH para acceso remoto seguro
- Añadir reglas básicas de firewall (ufw o iptables)
- Configurar partición o directorio para logs del sistema
- Establecer alias y scripts iniciales de mantenimiento
- Empezar configuración de los servicios del Home-Lab (semana 5-6 del plan)
- Añadir comprobación automatizada de arranque y cifrado (test de verificación)
- Revisar documentación y actualizar README.md si cambia el flujo de instalación
