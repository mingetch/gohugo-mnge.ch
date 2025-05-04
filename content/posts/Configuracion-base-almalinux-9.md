---
title: "Configuración base - Alma Linux 9"
date: 2025-02-18T21:19:06-06:00
draft: false
---

# Actualizar sistema

```bash
dnf -y update
```

# Configurar firewall para el sistema

Dependiendo del uso del servidor, abrir puertos en firewalld.

## Activar el servicio de firewall

```bash
systemctl enable --now firewalld
```

## Configurar los puertos y servicios que se usarán
Los nombres de los servicios están basados en los definidos en
*/usr/lib/firewalld/services/*. Si se desea agregar un servicio personalizado se
puede agregar en la carpeta */etc/firewalld/services/*.

```
#Puerto SSH alternativo
firewall-cmd --permanent --add-port=65501/tcp
# Acceso HTTP
firewall-cmd --permanent --add-service=http
# Acceso HTTPS
firewall-cmd --permanent --add-service=https
# mail
firewall-cmd --permanent --add-service=smtp
# smtp Submission over TLS [RFC8314]
firewall-cmd --permanent --add-service=smtps
# Submission [RFC4409]
firewall-cmd --permanent --add-service=smtp-submission
# IMAP over SSL
firewall-cmd --permanent --add-service=imaps
# POP-3 over SSL
firewall-cmd --permanent --add-service=pop3s
# Acceso OpenVPN
firewall-cmd --permanent --add-port=65501/udp
```

No es necesario agregar todos los puertos, solo los necesario para los servicios
que correrán en el servidor.

## Activar el servicio firewalld y ecargar las configuraciones

 ```bash
firewall-cmd --reload
```

## Verificar los servicios y puertos configurados

```bash
firewall-cmd --list-all
```

# Instalar los repositorios EPEL y REMI

Estos repositorios cuentan con paquetes no incluidos en la distribución base o
paquetes con versiones más recientes necesarias para algunas de las aplicaciones
que se puedan instalar. No son mandatorios.

```bash
dnf -y install epel-release
dnf update
```

# Agregar un usuario administrador

```bash
useradd -m -G wheel mingetch
passwd mingetch
```

## Cargar llaves ssh para el usuario

```bash
mkdir /home/mingetch/.ssh
vi /home/mingetch/.ssh/authorized_keys 
chown mingetch:mingetch -R /home/mingetch/.ssh
chmod 700 /home/mingetch/.ssh
chmod 600 /home/mingetch/.ssh/authorized_keys 
```

# Reforzar configuraciones de sshd

```bash
cp /etc/ssh/sshd_config{,.orig}
for i in \
    Port \
    LogLevel \
    PermitRootLogin \
    MaxAuthTries \
    PubkeyAuthentication \
    AuthorizedKeysFile \
    PasswordAuthentication \
    PermitEmptyPasswords \
    ChallengeResponseAuthentication \
    X11Forwarding\
do 
    sed -i "s@\#$i@$i@g" /etc/ssh/sshd_config
done

sed -i "s@^\(Port\).*@\1 65501@g" /etc/ssh/sshd_config
sed -i "s@^\(LogLevel\).*@\1 VERBOSE@g" /etc/ssh/sshd_config
sed -i "s@^\(PermitRootLogin\).*@\1 prohibit-password@g" /etc/ssh/sshd_config
sed -i "s@^\(MaxAuthTries\).*@\1 4@g" /etc/ssh/sshd_config
sed -i "s@^\(PubkeyAuthentication\).*@\1 yes@g" /etc/ssh/sshd_config
sed -i "s@^\(AuthorizedKeysFile\).*@\1 .ssh/authorized_keys@g" /etc/ssh/sshd_config
sed -i "s@^\(PasswordAuthentication\).*@\1 no@g" /etc/ssh/sshd_config
sed -i "s@^\(PermitEmptyPasswords\).*@\1 no@g" /etc/ssh/sshd_config
sed -i "s@^\(ChallengeResponseAuthentication\).*@\1 no@g" /etc/ssh/sshd_config
sed -i "s@^\(X11Forwarding\).*@\1 no@g" /etc/ssh/sshd_config
echo -e "\n\n"; for i in Port LogLevel PermitRootLogin MaxAuthTries PubkeyAuthentication AuthorizedKeysFile PasswordAuthentication PermitEmptyPasswords ChallengeResponseAuthentication X11Forwarding;do count=$(grep -c ^$i /etc/ssh/sshd_config); [[ $count -gt 1 ]] && echo "El parametro "${i}" aparece mas de 1 vez en el archivo de configuracion sshd.";done; echo
```


## Permitir el acceso mediante el nuevo puerto en SeLinux

```bash
semanage port -a -t ssh_port_t -p tcp 65501
semanage port -l | grep ssh
```

## Comprobar la configuració de SSH y reiniciar

```bash
sshd -t &&\
echo -e "\x1b[32;1mSSH Config Ok \x1b[0m Reinicia do ssh..." &&\
systemctl restart sshd
systemctl status sshd
```

## Verificar el acceso mediante el nuevo puerto
Es importante verificar el acceso mediante el nuevo puerto SSH desde una consola
diferente. Si cierra la consola actual y algo salió mal podría perder el acceso
a la consola.

Si el acceso es correcto, puede continuar con el siguiente paso.

## Cerrar el puerto 22 en el firewall
Cuando se esté seguro de que el puerto SSH alternativo es funcional se puede
desactivar el puerto estandar cerrandolo en el firewall con las siguientes
instrucciones:

```bash
firewall-cmd --permanent --remove-service=ssh
firewall-cmd --reload
```
# Instalar herramientas de administración

```bash
dnf install -y tmux iotop atop mytop htop apachetop bind-utils vim-enhanced \
               iftop at mlocate telnet expect zip unzip strace mtr traceroute \
               nmap rsync zip bzip2 wget python3 tar tree lsof
```

# Configurar el nombre de host

```bash
hostnamectl hostname server01.example.com
```

# Configurar swap en RAM

## Configurar el módulo del kernel

Crear archivos de configuración para cargar el módulo *ZRAM*, configurarlo y
configurar el dispositivo que se creará.

```bash
echo "zram" > /etc/modules-load.d/zram.conf
echo "options zram num_devices=1" > /etc/modprobe.d/zram.conf
echo 'KERNEL=="zram0", ATTR{disksize}="2G",TAG+="systemd"' > /etc/udev/rules.d/99-zram.rules
```

En este ejemplo se observa en la última línea que el tamaño del dispositivo
será de 2GB.

## Eliminar dispositivos *SWAP* anteriores

En caso de que haya algna partición u otro dispositivo de bloques configurado en
*fstab*, se debe desactivar editando dicho archivo.

```bash
vim /etc/fstab
```

## Crear un servicio de *systemd*

Crear o editar el siguiente archivo para poder manejar la *partición* *swap*
mediante *systemd*.

```bash
vim /etc/systemd/system/zram.service
```

Agregar el contenido siguiente

```
[Unit]
Description=Swap with zram
After=multi-user.target

[Service]
Type=oneshot
RemainAfterExit=true
ExecStartPre=/sbin/mkswap /dev/zram0
ExecStart=/sbin/swapon /dev/zram0
ExecStop=/sbin/swapoff /dev/zram0

[Install]
WantedBy=multi-user.target
```

## Activar el servicio

Se activa e inicia el servicio

```bash
systemctl enable zram
```

## Probar la creación y montaje del dispositivo

```bash
modprobe zram
systemctl start zram
zramctl
free #(opcional)
```

## Verificar que el dispositivo se monte tras el reinicio

```bash
reboot
zramctl
free #(opcional)
```

# Configurar el servicio de tiempo NTP

## Definir la zona horaria

```bash
timedatectl set-local-rtc 0
timedatectl set-timezone America/Mexico_City
```

O para sistemas que no cuenten con *timedatectl*

```bash
ln -sf /usr/share/zoneinfo/America/Mexico_City /etc/localtime
```  

## Cambiar los servidores de tiempo (opcional)

La aplicación chrony se encarga de sincronizar el reloj del servidor de manera
automática mediante el pool de servidores de tiempo del projecto CentOS, si se
desea usar un servidor en particular se debe modificar la siguiente sección del
archivo y añadiendo el pool o servidor que se desea usar.

```
# Use public servers from the pool.ntp.org project.
# Please consider joining the pool (http://www.pool.ntp.org/join.html).
pool 2.centos.pool.ntp.org iburst
```

Por ejemplo:

```
# Use public servers from the pool.ntp.org project.
# Please consider joining the pool (http://www.pool.ntp.org/join.html).

# Servidor de tiempo del cenam
pool cronos.cenam.gob.mx
```

# Referencias
[techrepublic.com How to enable zram Rocky Linux](https://www.techrepublic.com/article/how-to-enable-zram-rocky-linux)
