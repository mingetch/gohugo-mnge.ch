---
title: "Instalar herramientas y módulos ZFS en Alma Linux 9"
date: 2025-01-10T00:06:51-06:00
draft: false
---

# Objetivo
Instalar los paquetes y utilerías necesarias para crear y administrar volumenes
y datasets de *ZFS*.
# Condiciones previas
No se necesitan condiciones previas en el servidor para la instalación de las
utilerías, solo acceso como *root* o con sudo. Sin embargo para el ejemplo final
es deseable que exista un disco o partición la cual pueda ser formateada.

# Instalar el repositorio de OpenZFS (ZFSonLinux)

```bash
dnf install -y https://zfsonlinux.org/epel/zfs-release-2-3$(rpm --eval "%{dist}").noarch.rpm
dnf install -y epel-release
dnf makecache
```

# Instalar el modulo del kernel
En este caso se desactiva el método de instalación *DKMS* y se activa *kABI-tracking kmod*. Posterior a esto se instala el módulo.

```bash
dnf config-manager --disable zfs
dnf config-manager --enable zfs-kmod
dnf install -y zfs
```
Para Instalar mediante DKMS vea la referencia *https://zfsonlinux.org* al final de esta guía.

# Activar la carga del módulo en el arranque

```bash
echo zfs >/etc/modules-load.d/zfs.conf
```

# Verificar que el módulo se cargue en el arranque
## Reiniciar servidor

```bash
reboot
```

## Verificar que el módulo se haya cargado

```bash
lsmod | grep zfs
```

Salida de ejemplo:

```
$ lsmod | grep zfs
zfs                  4595712  6
zunicode              335872  1 zfs
zzstd                 630784  1 zfs
zlua                  233472  1 zfs
zavl                   16384  1 zfs
icp                   364544  1 zfs
zcommon               126976  2 zfs,icp
znvpair               147456  2 zfs,zcommon
spl                   155648  6 zfs,icp,zzstd,znvpair,zcommon,zavl
```

# Verificar las particiones disponibles

```bash
lsblk
```

## Salida de ejemplo

```
# lsblk
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda      8:0    0  600G  0 disk
├─sda1   8:1    0    1M  0 part
├─sda2   8:2    0    2G  0 part /boot
├─sda3   8:3    0   90G  0 part /
└─sda4   8:4    0  508G  0 part
```

La partición que se desea usar es /dev/sda4

# Crear un pool *ZFS* y sus datasets iniciales

Se formatea la partición */dev/sda4* como un pool de *ZFS*

```bash
zpool create datos /dev/sda4
```

A continuación se puede verificar el estado mediante los siguientes comandos

```bash
zpool list datos
zpool status datos
```

# Referencias

[gnulinux.ro Install ZFS CentOS Alma Rocky](https://gnulinux.ro/blog-page-8441-install-zfs-in-rhel-centos-alma-linux-rocky-linux)

[openzfs.github.io Getting started RHEL based](https://openzfs.github.io/openzfs-docs/Getting%20Started/RHEL-based%20distro/index.html)
