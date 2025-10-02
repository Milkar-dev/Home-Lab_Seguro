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

## Pendiente
- Documentar instalación de paquetes base (vim, git, network tools).
- Subir notas de configuración avanzada.
