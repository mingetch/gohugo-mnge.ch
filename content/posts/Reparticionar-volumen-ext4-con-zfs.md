---
title: "Reparticionar volumen ext4 con ZFS"
date: 2024-12-05T23:05:22-06:00
draft: false
---

# Objetivo

Crear una partición *ZFS* en un disco con particiones existentes.

# Condiciones previas

Se presupone que la partición que deseamos particionar está formateada con el
sistema *ext4*.

Verificar la estructura inicial del sistema de archivos de ejemplo:

```bash
# lsblk /dev/sda
```

Salida de ejemplo:

```
# lsblk /dev/sda
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
sda      8:0    0   600G  0 disk
├─sda1   8:1    0     1M  0 part
├─sda2   8:2    0     2G  0 part
└─sda3   8:3    0   598G  0 part
```

La partición a modificar tiene el siguiente formato (*TYPE="ext4"*)

```bash
# blkid /dev/sda3
```

Salida de ejemplo:

```
# blkid /dev/sda3
/dev/sda3: UUID="REDACTED_UUID" BLOCK_SIZE="4096" TYPE="ext4" PARTUUID="REDACTED_UUID"
```

IMPORTANTE: Recuerde siempre hacer un respaldo de la información ya que factores
como la fragmentación pueden llevar a la pérdida de la misma al realizar este
proceso.

# Iniciar con disco de rescate (opcional)

Si la partición de deseamos dividir contiene el sistema de archvos *ROOT* será
necesario primero iniciar el servidor con un disco de rescate, para este ejemplo
se usa *Debian 11* pero los comandos deberían funcionar para otras
distribuciones.

Si la partición no es *ROOT* se recomienda detener cualquier proceso que pueda
estar usando el disco, para esto se puede apoyar del comando *lsof*, el cual 
lista todos los archivos que están en uso actualmente cuando se ejecuta sin
parámetros.

# Desmontar la partición

Asegúrese que el sistema de archivos a modificar no esté montado.

```bash
umount /dev/sda3
```

# Redimensionar el sistema de archivos

Antes de cambiar el tamaño de la partición es necesario redimensionar el sistema
de archivos, las herramientas dependen del tipo de formato, en este caso es
*ext4*.

Note que no es lo mismo el tamaño de la partición que el del sistema de
archivos. Este último debe ser de un tamaño menor o igual al tamaño de la
partición.

## Verificar la partición con e2fsck

Con esto verificamos que el sistema de archivos esté en un estado consistente y
cuan fragmentado se encuentra (última línea de la salida de ejemplo).

```bash
e2fsck -f /dev/sda3
```

### Salida de ejemplo

```
# e2fsck -f /dev/sda3
e2fsck 1.46.2 (28-Feb-2021)
Pass 1: Checking inodes, blocks, and sizes
Pass 2: Checking directory structure
Pass 3: Checking directory connectivity
Pass 4: Checking reference counts
Pass 5: Checking group summary information
/dev/sda3: 60040/39198720 files (0.1% non-contiguous), 3120026/156773632 blocks
```

## Redimensionar el sistema de archivos

El siguiente comando redimesiona el sistema de ficheros *ext4*.

```bash
resize2fs -p /dev/sda3 90G
```

### Salida de ejemplo

```
resize2fs -p /dev/sda3 90G
resize2fs 1.46.2 (28-Feb-2021)
Resizing the filesystem on /dev/sda3 to 23592960 (4k) blocks.
Begin pass 2 (max = 585517)
Relocating blocks             XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
Begin pass 3 (max = 4785)
Scanning inode table          XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
Begin pass 4 (max = 8842)
Updating inode references     XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
The filesystem on /dev/sda3 is now 23592960 (4k) blocks long.
```

Tomar nota de los valores mostrados en la última línea, ya que se usarán en los
pasos posteriores.

## Probar el sistema redimesionado

Procedemos a montar el sistema de archivos, observe en los últimos dos ejemplos
cómo el tamaño de la partición no ha tenido cambios, sin embargo el sistema de
archivos motado sí se ha reducido.

### Montar el sistema de archivos

```bash
mount /dev/sda3 /mnt
```

### Verificar el tamaño de las particiones

```
# lsblk /dev/sda
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda      8:0    0  600G  0 disk
├─sda1   8:1    0    1M  0 part
├─sda2   8:2    0    2G  0 part
└─sda3   8:3    0  598G  0 part /mnt
```

### Verificar el montaje del sistema de archivos

Se verifica el nuevo tamaño del sistema de archivos:

```
# df | grep -E 'Used|sda'
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda3        88G  1.5G   82G   2% /mnt
```

# Redimensionar la partición

## Verificar el tamaño del sistema de archivos en KB

```
# df | grep -E 'Used|sda'
Filesystem     1K-blocks    Used Available Use% Mounted on
/dev/sda3       91784928 1260144  85789816   2% /mnt
```

## Desmontar la partición

Asegúrese que el sistema de archivos a modificar no esté montado.

```bash
umount /dev/sda3
```

## Calcular el tamaño de la nueva partición

Se obtiene el nuevo tamaño del sistema de archivos (en KB), con base en el
tamaño reportado por *resize2fs* cuando redujimos el sistema de ficheros.

```bash
new_size=$((23592960*4))
echo $new_size
```

## Obtener el número de partición

```bash
parted /dev/sda unit KiB p
```

### Salida de ejemplo

```
parted /dev/sda u KiB p
Model: QEMU QEMU HARDDISK (scsi)
Disk /dev/sda: 629145600kiB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags: pmbr_boot

Number  Start        End           Size          File system  Name  Flags
        17.0kiB      1024kiB       1007kiB       Free Space
 1      1024kiB      2048kiB       1024kiB                          bios_grub
 2      2048kiB      2050048kiB    2048000kiB    ext4
 3      2050048kiB   629145583kiB  627095536kiB  ext4

```

En la Salida de ejemplo notamos, por su tamaño, que la partición que nos interesa
es la número *3*.

## Obtener la posición final de la partición 
Obtener la posición final de la partición que redimensionaremos ya que el
parámetro *resizepart* del comando *parted*, que usaremos para tal fin, recibe
dicho argumento en lugar del nuevo tamaño.

Para esto tomamos el valor de la columna *Start* (2099249) de la salida del comando anterior y lo sumamos con el tamaño calculado para la nueva partición (guardado en la variable $new_size)

```bash
new_end=$((2050048+new_size))
echo $new_end
```

### Salida de ejemplo

```
# new_end=$((2050048+new_size))
# echo $new_end
96421888
```

## Redimensionar la partición

Usamos el tamaño obtenido en el paso anterior (*new_size*) para indicarle a
*parted* en dónde terminará la nueva partición.

```bash
parted /dev/sda resizepart 3 ${new_end}KiB
```

### Salida de ejemplo

```
# parted /dev/sda resizepart 3 ${new_end}KiB
Warning: Shrinking a partition can cause data loss, are you sure you want to continue?
Yes/No? yes
Information: You may need to update /etc/fstab.
```

## Verificar el nuevo tamaño de la partición

Con el parámetro *p* listamos el espacio libre en el disco, cómo se puede ver en
el siguiente ejemplo, la partición *3* ahora tiene un tamaño de *96.6GB* y
quedan *546GB* libres al final del disco.

```
# parted /dev/sda p free
Model: QEMU QEMU HARDDISK (scsi)
Disk /dev/sda: 644GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags: pmbr_boot

Number  Start   End     Size    File system  Name  Flags
        17.4kB  1049kB  1031kB  Free Space
 1      1049kB  2097kB  1049kB                     bios_grub
 2      2097kB  2099MB  2097MB  ext4
 3      2099MB  98.7GB  96.6GB  ext4
        98.7GB  644GB   546GB   Free Space
```

### Verificar la partición con e2fsck

Verificamos que el sistema de archivos esté en un estado consistente.

```
# e2fsck -f /dev/sda3
e2fsck 1.47.0 (5-Feb-2023)
Pass 1: Checking inodes, blocks, and sizes
Pass 2: Checking directory structure
Pass 3: Checking directory connectivity
Pass 4: Checking reference counts
Pass 5: Checking group summary information
/dev/sda3: 55571/5898240 files (0.1% non-contiguous), 961764/23592960 blocks
```

#### Ejemplo de un redimensionado fallido
En el siguiente ejemplo la partición se redimensionó a un tamaño inferior al del
sistema de archivos:

```
# e2fsck -f /dev/sda3
e2fsck 1.47.0 (5-Feb-2023)
The filesystem size (according to the superblock) is 23592960 blocks
The physical size of the device is 23040000 blocks
Either the superblock or the partition table is likely to be corrupt!
Abort<y>? yes
```

## Redimensionar el sistema de archivos

Redimesionamos nuevamente el sistema de ficheros *ext4* para que ocupe todo el
espacio disponible en la nueva partición (si es que existe), ya que en ocaciones
las conversiones entre unidades pueden fallar. Si el tamaño de la partición
fuera menor que el del sistema de archivos, el comando *e2fsck* nos lo
indicaría, en ese caso se deben verificar los tamaño y conversiones de unidades.

```
# resize2fs -p /dev/sda3
resize2fs 1.47.0 (5-Feb-2023)
The filesystem is already 23592960 (4k) blocks long.  Nothing to do!
```

### Ejemplo de un redimensionado en el que la partición tenía un tamaño mayor al
del sistema de archivos:

```
resize2fs -p /dev/sda3
resize2fs 1.46.2 (28-Feb-2021)
Resizing the filesystem on /dev/sda3 to 24391168 (4k) blocks.
```
Note que en este caso no se indica el tamaño nuevo.

## Verificar la partición con e2fsck

Verificamos que el sistema de archivos esté en un estado consistente.

```
# e2fsck -f /dev/sda3
e2fsck 1.47.0 (5-Feb-2023)
Pass 1: Checking inodes, blocks, and sizes
Pass 2: Checking directory structure
Pass 3: Checking directory connectivity
Pass 4: Checking reference counts
Pass 5: Checking group summary information
/dev/sda3: 55571/5898240 files (0.1% non-contiguous), 961764/23592960 blocks
```

## Verificar el nuevo tamaño del sistema de archivos

```
# mount /dev/sda3 /mnt
# df -h | grep sda
/dev/sda3        88G  1.3G   82G   2% /mnt

# df | grep sda
/dev/sda3       91784928 1260144  85789816   2% /mnt
```

Si se compara el tamaño de la partición del segundo comando *df* debería ser
igual al que se sacó antes de iniciar el redimensionamiento.

# Crear la nueva partición
## Verificar el estado actual del disco

```
# parted /dev/sda p free
Model: QEMU QEMU HARDDISK (scsi)
Disk /dev/sda: 644GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags: pmbr_boot

Number  Start   End     Size    File system  Name  Flags
        17.4kB  1049kB  1031kB  Free Space
 1      1049kB  2097kB  1049kB                     bios_grub
 2      2097kB  2099MB  2097MB  ext4
 3      2099MB  98.7GB  96.6GB  ext4
        98.7GB  644GB   546GB   Free Space
```
        
## Crear la nueva partición 
Indicamos al comando *parted* el inicio de la nueva partición, que es el valor correspondiente a la columna *Start* del espacio libre en disco (*98.7GB*) y el final sería el *100%* del espacio contiguo posterior a este.

```bash
parted /dev/sda mkpart zfspool ext2 98.7GB 100%
```

### Salida de ejemplo

```
# parted /dev/sda mkpart zfspool ext2 98.7GB 100%
Information: You may need to update /etc/fstab.
```

El parámetro *zfspool* es opcional y solo funge como un tag en el caso de las tablas de particiones tipo *GPT*.

## Verificar que el kernel haya tomado la nueva partición

```bash
cat /proc/partitions
```

### Salida de ejemplo
```
# cat /proc/partitions
major minor  #blocks  name

   8        0  629145600 sda
   8        1       1024 sda1
   8        2    2048000 sda2
   8        3   94371840 sda3
   8        4  532721664 sda4
   7        0    1237668 loop0
```

En este punto se puede reiniciar el servidor para que arranque con sus sistema,
en caso de que haya iniciado con un disco de rescate.

# Referencias

[https://access.redhat.com](https://access.redhat.com/articles/1196333)

[https://docs.redhat.com](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/managing_storage_devices/getting-started-with-partitions_managing-storage-devices#proc_resizing-a-partition-with-parted_getting-started-with-partitions)

[https://gnulinux.ro](https://gnulinux.ro/blog-page-8441-install-zfs-in-rhel-centos-alma-linux-rocky-linux)
