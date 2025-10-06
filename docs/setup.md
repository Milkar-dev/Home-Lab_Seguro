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
### Se comprueba que el sistema arranca en modo UEFI y hay conexion a conexión
ls /sys/firmware/efi/efivars      # Si aparece, estás en modo UEFI
iwctl                             # Entra al gestor de red inalámbrica
station wlan0 connect NOMBRE_RED  # Conéctate a la red WiFi
exit
ping archlinux.org                # Comprueba conexión a Internet


## 2. Particionado del disco
### Se usa fdisk para crear dos particiones para un HHD (una pequeña de 512 MB para EFI):
fdisk /dev/sda
n → nueva partición (EFI)
Enter → número por defecto (1)
Enter → sector inicial por defecto
+512M → tamaño de la partición
t → cambia el tipo
1 → EFI System
n → nueva partición (resto del disco para el sistema)
Enter → acepta todo el espacio libre
w → guardar y salir

### Para crear las particiones para SSD se utiliza esto:
fdisk /dev/nvme0n1
n → nueva partición (EFI)
Enter → número y sector por defecto
+512M → tamaño EFI
t → cambia tipo
1 → EFI System
n → nueva partición (resto del disco para el sistema)
Enter → acepta todo el espacio restante
w → guardar y salir


## 3. Cifrado y montaje
### Aquí se cifra la partición principal y se monta para preparar la instalación. Esto protege los datos incluso si alguien accede físicamente al disco.
cryptsetup luksFormat /dev/sda2         # Cifra la partición principal
cryptsetup open /dev/sda2 cryptroot     # Desbloquea la partición cifrada
mkfs.ext4 /dev/mapper/cryptroot         # Formatea el sistema de archivos principal
mkfs.fat -F32 /dev/sda1                 # Formatea la partición EFI
mount /dev/mapper/cryptroot /mnt        # Monta la raíz
mkdir /mnt/boot
mount /dev/sda1 /mnt/boot


## 4. Instalación del sistema base
### Se instalan los paquetes mínimos para arrancar el sistema y manejar la red.
pacstrap /mnt base linux linux-firmware vim networkmanager
genfstab -U /mnt >> /mnt/etc/fstab
arch-chroot /mnt


## 5. Configuración básica
### Se establecen la zona horaria, el idioma, el nombre del equipo y el teclado.
ln -sf /usr/share/zoneinfo/Europe/Madrid /etc/localtime
hwclock --systohc
echo "LANG=es_ES.UTF-8" > /etc/locale.conf
echo "KEYMAP=es" > /etc/vconsole.conf
echo "homelab" > /etc/hostname
sed -i 's/#en_US.UTF-8 UTF-8/en_US.UTF-8 UTF-8/' /etc/locale.gen
sed -i 's/#es_ES.UTF-8 UTF-8/es_ES.UTF-8 UTF-8/' /etc/locale.gen
locale-gen
### Aquí se deja el sistema listo con la hora en Madrid, idioma español y teclado español, si deseas cambiarlo, eres libre de hacerlo


## 6. Configurar mkinitcpio y GRUB con cifrado
### Esta parte es clave: se indica al sistema que el disco está cifrado, para que pida la contraseña al arrancar.
sed -i 's/HOOKS=(.*)/HOOKS=(base udev autodetect modconf block encrypt filesystems keyboard fsck)/' /etc/mkinitcpio.conf
mkinitcpio -P

pacman -S grub efibootmgr --noconfirm
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB

UUID=$(blkid -s UUID -o value /dev/sda2)
sed -i "s|^GRUB_CMDLINE_LINUX=.*|GRUB_CMDLINE_LINUX=\"cryptdevice=UUID=$UUID:cryptroot root=/dev/mapper/cryptroot\"|" /etc/default/grub
grub-mkconfig -o /boot/grub/grub.cfg

### `mkinitcpio` crea la imagen del sistema con los hooks necesarios para abrir el disco cifrado.
### `GRUB` luego añade los parámetros para descifrarlo al iniciar.


## 7. Servicios y usuario
systemctl enable NetworkManager
passwd
useradd -m -G wheel -s /bin/bash mil
passwd mil
EDITOR=vim visudo
## Descomentar: %wheel ALL=(ALL:ALL) ALL
#### Esto deja el sistema con el usuario mil listo para trabajar y con permisos para administrar el sistema, si deseas cambiar de usuario, eres libre de hacerlo


## 8. Finalización
exit
umount -R /mnt
swapoff -a
reboot
#### Se desmonta y se reinicia al nuevo sistema base

### Tras reiniciar, el sistema pedirá la contraseña LUKS antes de arrancar.
### Si todo ha ido bien, tendrás un Arch Linux completamente funcional, cifrado y con red: la base perfecta para seguir con el proyecto de Home-Lab Seguro.



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
