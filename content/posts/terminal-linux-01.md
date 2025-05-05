---
title: "Consola Linux 01 [borrador]"
date: 2023-02-04T07:26:22-06:00
draft: true
---

# Intro
################################################################################
En este curso introductorio se pretende adentrarnos en el manejo del sistema 
operativo linux desde la interfáz de línea de comandos (CLI).

Debido a que el curso tiene un caracter introductorio no se ahonda mucho en los 
tópicos pero se dan las herramientas para saber en dónde buscar si se desea 
profundizar en los temas.

# Buscando ayuda, defiendete solo

Cuando se trata de investigar como funcionan los comandos la mayoría de los 
sistemas linux cuentan con dos potentes herramientas para arreglarselas uno 
mismo:

```
  * man: Páginas de manual 
  * --help: Ayuda en línea de comandos
```

Estas pueden sacarnos de un apuro en caso de que estemos en un ambiente aislado
o cuando nos sintamos a gusto usandolas.

En cualquier otro caso podemos acudir al poder de google u otras herramientas
en línea.

## Cuando sepas lo que tienes, lee el manual (RTFM)

Para ver el manual de un programa o comando basta con teclear la palabra  *man*
seguida del comando del que deseamos ver el manual, por ejemplo:

```
  man mkdir
  man ls
```

Para abrir la ayuda de un comando usualmente se puede escribir el comando
seguido por el parámetro --help o -h, similar al siguiente ejemplo:

```
  touch --help
  cd --help
```

## ¿Y qué si no sé qué buscar?

El comando apropos es un auxiliar que busca en la lista de manuales y nos arroja
las coincidencias con una palabra clave, por jemplo:

```
  apropos intro
```

Así con base en la salida del comando anterior, podemos por ejemplo,  abrir el
manual que nos interesa.

```
  man 4 intro
```

# Manejo de archivos y carpetas

En linux todo es un archivo, las carpetas, binarios, puertos, unidades de
almacenamiento, dispositivos de red, !todo!

Sin embargo, al igual que en todo, hay diferentes tipos de archivos. En esta
sección nos centrarémos en archivos de datos y carpetas.

Aunque los siguiente comandos pueden funcionar para otro tipo de archivos es
probable que algunos fallen o no den el resultado esperado.

## Obtener información de archivos file, stat, du, ls, find

*file*, obtiene información del tipo de archivo

```
file imagen.png
file /home
```

## Comparar archivos, sumas de verificación y diferencias

```
md5sum
sha256sum
diff, kompare, kdiff
```

## Listar contenido (cat, head, tail)

## Caracteres escurridizos, identifícalos en tu teclado (~ ` | \\ / < > [ ] { } )

## borrar, copiar, tocar, vaciar/crear (rm, shred, scrub, cp, mv, touch, >)

## Navegación en el sistema de ficheros pwd, cd -, cd ~, etc

"cd", "cd -", "cd ..", "cd /", and "cd ~"

## Permisos de archivos

## Filtrado y modificación de la salida de un comando (grep, sed, awk, tee, tr)

## Flujos de datos y redirecciones, &0,&1,&2,|, ¡que lo haga el otro!

/dev/null
flujos de datos:
0: stdin
1: stdout
2: stderr
pipas

# Usuarios

Listar usuarios (lslogins, /etc/passwd)
Identificar el shell del usuario
Listar ID y grupo de un usuario
Sudo, su. Correr comandos como un usuario diferente

# Procesos

Listado de procesos (ps -elf)
Listado de archivos en uso (lsof)
Monitoreo de procesos (top, htop)
Detener procesos (kill, killall, -9)

# Introducción a scripts

Encabezados de scripts, she bang. https://www.delftstack.com/howto/linux/bash-script-header/

# Manejo de servicios

Conocer el manejador de servicios slackware(init scripts), systemV(centos5), upstart(ubuntu <=14.10),  SystemD (distros recientes, ubuntu 15.04+, centos 7, etc.)
Inicio y detención de servicios (init scripts, upstart, sysV, systemD)
Crear scripts para el manejo de servicios (init scripts, upstart, sysV, systemD)

# Dispositivos en linux (/dev/tty, /dev/sdx)

Dispositivos USB, lsusb
Dispositivos PCI, lspci
Listado general de hardware, lshw
Buscando dispositivos en el buffer del kernel, dmesg

# Unidades de almacenamiento

Listar dispositivos de bloques y sus propiedades (lsblk, blkid, fdisk, smartctl)
ls /dev/disk/{by-id, by-uuid, by-label, by-partlabel}
Verificar espacio en unidades de almacenamiento
Puntos de montaje
Introducción a fstab

# Redes

Conceptos de puerta de enlace, dirección IP, mascara de red, alias de red
Listar dispositivos de red (ifconfig, ip)
Obtener configuraciones de red(ip r, route -n, ip a, ifconfig, resolv.conf, ipcalc)
Cambiar la configuración de red de manera temporal
Cambiar la configuración de red de manera permanente
Agregar un alias de red
Pruebas de red, Ping, traceroute, mtr, host, dig
Montaje de carpetas locales y remotas (SFTP, mount, fstab, NFS, SAMBA, SSHFS)
Listar puertos en los que “escuchan” los servicios (ss/netstat  -tunap, ss  -tunap, lsof -i)

# SSH

Conexión directa (puerto estandar y no estandar)
Llaves SSH
Túneles
Reenvío de puertos
Reenvío inverso de puertos
Túnel dinámico

# Aumentando la productividad como en las películas

Editor Vim,
Modos de funcionamiento
Navegación (gg, G, #g, 0, $
Comandos frecuentes
Copia, selección, borrado e inserción de palabras y líneas (ctrl+v, shift+v, i, I, a, A, yw, dw, ciw, dd, o, O, etc)
Números de línea

Tmux
Tecla modificadora (ctrl+a y ctrl+b)
Paneles y ventanas (crear, dividir, separar)
Navegación entre paneles
Navegación entre ventanas
renombrar sesión
renombrar panel
Attach (simple)
Múltiples sesiones
Attach (a sesión por defecto, por ID, por nombre)
Enviar comandos a un panel
Cheatsheet

Autokey
