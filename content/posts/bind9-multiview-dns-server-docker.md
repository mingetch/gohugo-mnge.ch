---
title: "BIND 9 - Multiviews DNS Server - Docker"
date: 2025-04-19T23:45:41-06:00
draft: false
---

# Objetivo

Configurar cuatro servidores de *DNS* en los cuales uno funja como servidor
principal (*authoritative*) y los otros tres como secundarios, los servidores
deben manejar las zonas mediante vistas (*BIND9 views*) para lograr una
arquitectura *split horizon*.

El servicio *BIND9* debe ejecutarse en contenedores *docker*.

# Requisitos

En el servidor docker no deben existir otros servicios de *DNS*, como pueden ser
*BIND, DNSmasq*, etc. Estos usualmente son proporcionados por aplicaciones como 
*libvirt* o el mismo sistema operativo en el caso de *Ubuntu*.

# Datos iniciales

## Asunciones
1. Los servidores están en VPN y pueden *verse* a través de ella.
1. Existen tres redes en las que se dividiran los registros *DNS*, las cuales
   son *publica, lan1*, y *vpn*.
1. El servidor primario se encuentra en la red lan1 y no es accesible de manera
pública.

## Nombres de host e IP's

Los servidores tienen los siguientes nombres, IP's públicas y rol

```
Servidor DNS Primario
---------------------
Hostname: darkstar1
IP Address: 192.168.1.100,
            10.8.0.100

Servidor DNS Secundario
---------------------
Hostname: central1
IP Address: 11.11.11.11
            10.8.0.101

Servidor DNS Secundario
---------------------
Hostname: east1
IP Address: 22.22.22.22
            10.8.0.102

Servidor DNS Secundario
---------------------
Hostname: west1
IP Address: 33.33.33.33
            10.8.0.103
```

## Segmentos de las redes
A continuación se enumeran las redes con las que segregarán los registros DNS y
sus segmentos de red.

```
publica, 0.0.0.0/0
vpn, 10.8.0.0/24
lan1, 192.168.1.0/24
```

## Dominios administrados
Los siguientes dominios serán administrados por nuestro sistema de DNS'.

```
ejemplo1.com
ejemplo2.com
ejemplo3.com
ejemplo4.com
```

# Declarar variables de entorno para pruebas
Se crean listas de las zonas, dominios y direcciones IP de cada dominio en las
diferentes zonas.

```bash
domains_list='ejemplo1.com ejemplo2.com ejemplo3.com ejemplo4.com'
zones_list="lan1 vpn publica"
ip_addr_list_lan1='192.168.0.100'
ip_addr_list_vpn='10.8.0.100 10.8.0.101 10.8.0.102 10.8.0.103'
ip_addr_list_publica='172.32.0.2 11.11.11.11 22.22.22.22 33.33.33.33'
```

Estas variables se deben declarar en todos los equipos con los que
interactuemos.

# Instalar docker

```bash
dnf config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo
dnf install -y docker-ce docker-ce-cli containerd.io
systemctl enable --now docker
```

# Obtener la imágen base del contenedor
Descarga la imágen que se usará como base desde el registro de contenedores de
Docker.

```bash
docker pull almalinux:9.5
```

# Crear una carpeta de trabajo
Crear una carpeta para almacenar los archivos de creación de la imágen y los 
datos del contenedor.

```bash
mkdir -p /datos/contenedores/bind9
cd /datos/contenedores/bind9/
```

# Crear un archivo *Dockerfile*
El contenido del Dockerfile indica lo siguiente al momento de crear la imágen:
1. Usar como base la versión 9.5 de almalinux desde *docker registry*
   (descargada en el paso anterior).
1. Instala dependencias con dnf.
1. Crear llave de cifrado para la comunicación *rndc*.
1. Indica que el contenedor expondrá sus puertos 53/TCP, 53/UDP y 953/TCP.
1. Se crearán los volúmenes para almacenar datos permanentes, los cuales se 
   montan en la ubicación que uno desee al momento de iniciar el contenedor ya
   directo desde la línea de comandos o mediante *docker-compose*.
1. Al arrancar el contenedor se iniciará el servicio *named* con su archivo de
   configuración.

```bash
cat << FIN > Dockerfile
FROM almalinux:9.5

RUN dnf update -y && dnf -y install bind9.18 bind9.18-dnssec-utils bind-utils crypto-policies iproute
RUN /usr/sbin/rndc-confgen | sed -n '/^key\ "rndc-key/,/^$/p' > /etc/rndc.key

# Expose Ports
EXPOSE 53/tcp
EXPOSE 53/udp
EXPOSE 953/tcp

VOLUME ["/etc/bind","/var/named/bind"]

# Start the Name Service
CMD ["/sbin/named", "-g", "-c", "/etc/bind/named.conf", "-u", "named"]
FIN
```

# Construir la imágen del contenedor

Esto creará una imágen en nuestro registro local de docker.

```bash
docker build -t bind918 .
```

## Crear un respaldo de la imagen del contenedor
Se crea un respaldo de la imagen para subirla a los servidores esclavos, los
siguientes comandos crean el respaldo, la comprimen y envía al servidor esclavo
11.11.11.11.

```bash
docker save -o bind918.tar bind918
xz bind918.tar
rsync -a --progress bind918.tar.xz root@10.8.0.101:
```

El comando *rsync* se deberá repetir para subir la imágen a los demás esclavos.

# Instalar archivos de configuración de BIDN9
Se instala el servidor de DNS BIDN9 y herramientas para gestión y diagnóstico en
todos los servidores.

```bash
dnf install -y bind9.18 bind9.18-dnssec-utils
```

## Desactivar el servicios BIND del sistema

```bash
systemctl disable --now named
```

# Crear red docker
Crear una subred en docker para el contenedor.

```bash
docker network create --subnet=172.32.0.0/24 bind9-network
```

# Iniciar el contenedor para crear las configuraciones básicas
El inicio del contenedor en este punto fallará debido a que no tiene
configuraciones, sin embargo esto nos ayuda a verificar que la imagen puede ser
usada y a crear las carpetas básicas de configuración.

```bash
docker run \
    --rm \
    --name=ddns-master-918 \
    --net=bind9-network \
    --ip=172.32.0.2\
    -p 53:53/udp \
    -p 53:53/tcp \
    --volume ./config:/etc/bind \
    --volume ./zones:/var/named/bind \
    bind918
```

## Descripción de parámetros
  
    --rm       # El contenedor será destruido cuando se detenga
    --name     # Nombre del contenedor
    --net      # Crea una red llamada *bind9-network*
    --ip       # Define la IP que tendrá el contenedor en su red
    -p         # Espone el puerto 53 UDP/TCP para cada interfáz del host
    --volume   # *Monta* el volumen un las subcarpetas config y zones dentro de
               # la carpeta base del contenedor.
    bind918  # Indica la imágen que se usará para el contenedor

Más adelante en este documento se creará un archivo *docker-compose.yaml* el cual
facilita la administración del contenedor.

# Configuración de servidor principal

## Copiar configuraciones base de el servidor secundario
Copiar configuraciones base */etc/named.conf* desde el servidor secundario y
colocarlo en la ruta *./config/*.

## Escuchar desde cualquier origen o interfaz
Permitir a BIND el recibir peticiones desde cualquier origen cambiando los
siguientes parámetros en el archivo *named.conf*:

```
options {
        listen-on port 53 { 127.0.0.1; };
        listen-on-v6 port 53 { ::1; };
          . . .
        allow-query     { localhost; };
          . . .
```

Por estos otros:

```
options {
       listen-on port 53 { any; };
       listen-on-v6 port 53 { any; };
         . . .
       allow-query     { any; };
         . . .
```

## Configurar las notificaciones hacia los servidores secundarios

Agregar los siguientes parámetros posterior al parámetro *allow-query* para que
los servidores secundarios sean notificados cada que ocurran cambios en las
zonas del primario.

```
notify explicit;
```

## Deshabilitar la recursividad

Debido a que este es un servidor autoritario y para reducir los ataque de
amplificación por DNS, se deshabilitan las consultas recursivas.

Estas consultas son las que pertenecen a cualquier dominio (incluyendo los
ajenos a nosotros) y en nuestro caso solo se desea servir los registros de
nuestros dominios.

Cambiamos el parámetro

```
recursion yes
```

Por

```
recursion no
```

## Comentar la zona '.'

```
//zone "." IN {
//    type hint;
//    file "named.ca";
//};
```

## Incluir las configuraciones de vistas
Agregamos la inclusión de las configuraciones de vistas y comentamos la
inclusión del las zonas del *rfc1912*.

```
//include "/etc/named.rfc1912.zones";
include "/etc/bind/named.views";
```

## Configuraciones en el archivo de vistas

El archivo de las vistas definirá cuales son las zonas a usar de acuerdo al
origen (segmento de red) de las peticiones y las llaves de cifrado que se usen
para hacer dichas peticiones, iniciamos creando un archivo en blanco como sigue:

### Generar llaves de cifrado

La comunicación entre servidores se lleva a cabo de manera cifrada, podemos
generar una llave para cada vista mediante el comando *rndc-confgen* y filtrando
los campos necesarios con *grep*.

```bash
rndc-confgen | grep -E '^[^#].*algorithm|^[^#].*secret'
```

Con la ayuda de un ciclo *for* repetiremos este comando para cada zona y
vaciaremos el resultado en el archivo de las vistas.

```bash
for net in $zones_list
do
    echo -e "key \"${net}\" {
    $(rndc-confgen | grep -E '^[^#].*algorithm|^[^#].*secret')
    };"
    echo
done \
| sudo tee config/named.views
```

### Listas de control de acceso (ACL)

Las listas de control de acceso definen el origen de cada petición con base en
el segmento de red desde el que se hace la petición y la llave de cifrado usada
por parte de los servidores esclavos.

El signo de admiración en las siguientes reglas es un *NO* lógico (negación).

```
acl "lan1" {
                !key vpn;
                !key publica;
                192.168.1.0/24;
                key lan1;
};

acl "vpn" {
                !key lan1;
                !key publica;
                10.80.0.0/16;
                key vpn;
};

acl "publica" {
                !key lan1;
                !key vpn;
                !"private";
                !"vpn";
                key publica;
                any;
};

```

### Crear los bloques de las vistas

En el siguiente ejemplo se muestra la vista para la red *lan1*:

```
view lan1 {
     match-clients { lan1; };
     allow-query     { any;};
     recursion yes;
     allow-transfer { key lan1; };

     zone "." IN {
             type hint;
             file "named.ca";
          };

     };
     zone "ejemplo1.com" {
          type primary;
          file "bind/lan1/ejemplo1.com.zone";
          also-notify {
                        11.11.11.11 key lan1;
                        22.22.22.22 key lan1;
                        33.33.33.33 key lan1;
          };
     };
     zone "ejemplo2.com" {
          type primary;
          file "bind/lan1/ejemplo2.com.zone";
          also-notify {
                        11.11.11.11 key lan1;
                        22.22.22.22 key lan1;
                        33.33.33.33 key lan1;
          };
     };
     zone "ejemplo3.com" {
          type primary;
          file "bind/lan1/ejemplo3.com.zone";
          also-notify {
                        11.11.11.11 key lan1;
                        22.22.22.22 key lan1;
                        33.33.33.33 key lan1;
          };
     };
     zone "ejemplo4.com" {
          type primary;
          file "bind/lan1/ejemplo4.com.zone";
          also-notify {
                        11.11.11.11 key lan1;
                        22.22.22.22 key lan1;
                        33.33.33.33 key lan1;
          };
     };
};
```

Para la red *vpn* la vista sería muy similar ya que se manejan las mismas tres zonas que para la *lan1*.

```
view vpn {
     match-clients { vpn; };
     allow-query     { any;};
     recursion yes;
     allow-transfer { key vpn; };

     zone "." IN {
             type hint;
             file "named.ca";
     };

     zone "ejemplo1.com" {
          type primary;
          file "bind/vpn/ejemplo1.com.zone";
          also-notify {
                        11.11.11.11 key vpn;
                        22.22.22.22 key vpn;
                        33.33.33.33 key vpn;
          };
     };
     zone "ejemplo2.com" {
          type primary;
          file "bind/vpn/ejemplo2.com.zone";
          also-notify {
                        11.11.11.11 key vpn;
                        22.22.22.22 key vpn;
                        33.33.33.33 key vpn;
          };
     };
     zone "ejemplo3.com" {
          type primary;
          file "bind/vpn/ejemplo3.com.zone";
          also-notify {
                        11.11.11.11 key vpn;
                        22.22.22.22 key vpn;
                        33.33.33.33 key vpn;
          };
     };
     zone "ejemplo4.com" {
          type primary;
          file "bind/vpn/ejemplo4.com.zone";
          also-notify {
                        11.11.11.11 key vpn;
                        22.22.22.22 key vpn;
                        33.33.33.33 key vpn;
          };
     };
};
```

Note como en estas dos redes el parámetro *recursion* es igual a *yes* lo cual
permite usar nuestros servidores para resolver cualquier dominio, incluso los de
terceros desde nuestras redes privadas (*lan1* y *vpn*).

Par la vista de la red pública se usa una configuración similar a la siguiente:

```
view publica {
     match-clients { publica; };
     allow-query     { any;};
     recursion no;
     allow-transfer { key publica; };

     zone "." IN {
             type hint;
             file "named.ca";
     };
     zone "ejemplo3.com" {
          type primary;
          file "publica/ejemplo3.com.zone";
          also-notify {
                        11.11.11.11 key publica;
                        22.22.22.22 key publica;
                        33.33.33.33 key publica;
          };
     };
};
```

En esta última red no se permiten las consultas recursivas y solo se expone el
dominio *ejemplo3.com* por lo que no habría resolución para registros de los demás
dominios desde el exterior de las redes privadas.

## Creación de zonas para cada vista

Cada una de las vistas tiene archivos de zonas independientes, esto se almacenan
en subcarpetas de *./zones*.

### Creación de carpetas de las vistas

```bash
sudo mkdir zones/{lan1,vpn,publica}
sudo chown root:root -R zones
sudo chmod 755 -R zones
```

### Creación de archivos de zonas

En los siguientes listados se muestra el contenido de las zonas del dominio
*ejemplo1.com* para las tres vistas, se deja al lector la creación de las zonas
para los otros dominios.

#### Vista *lan1*

```
#./zones/vpn/ejemplo1.com.zone
$ORIGIN ejemplo1.com.
$TTL 10M
@                               IN SOA  @ ns1.ejemplo1.com. (
                                                        2409032044      ; serial
                                                        1H              ; refresh
                                                        1H              ; retry
                                                        1D              ; expire
                                                        3H )            ; minimum
; NS Records
@               IN      NS      ns1.ejemplo1.com.
@               IN      NS      ns2.ejemplo1.com.
@               IN      NS      ns3.ejemplo1.com.

; A Records
@               IN      A       192.168.1.10
ns1             IN      A       11.11.11.11
ns2             IN      A       22.22.22.22
ns3             IN      A       33.33.33.33

nas01           IN      A       192.168.1.101
nas02           IN      A       33.33.33.33
proxmox1        IN      A       192.168.1.103
blog            IN      A       22.22.22.22

; CNAME Records
nextcloud       IN      CNAME   nas01
traefik         IN      CNAME   proxmox1
mail            IN      CNAME   mail.ejemplo3.com.
autodiscover    IN      CNAME   mail.ejemplo3.com.
autoconfig      IN      CNAME   mail.ejemplo3.com.

; TXT Records
networktype     IN      TXT     "lan1_network"

; SRV Records
; Name              Type       Priority Weight Port    Value
_autodiscover._tcp  IN SRV     0        1      443      mail.ejemplo3.com.
_imap._tcp          IN SRV     0        1      143      mail.ejemplo3.com.
_imaps._tcp         IN SRV     0        1      993      mail.ejemplo3.com.
_pop3._tcp          IN SRV     0        1      110      mail.ejemplo3.com.
_pop3s._tcp         IN SRV     0        1      995      mail.ejemplo3.com.
_sieve._tcp         IN SRV     0        1      4190     mail.ejemplo3.com.
_smtps._tcp         IN SRV     0        1      465      mail.ejemplo3.com.
_submission._tcp    IN SRV     0        1      587      mail.ejemplo3.com.
```

#### Vista *vpn*

```
#/var/named/vpn/ejemplo1.com.zone
$ORIGIN ejemplo1.com.
$TTL 10M
@                               IN SOA  @ ns1.ejemplo1.com. (
                                                        2409032044      ; serial
                                                        1H              ; refresh
                                                        1H              ; retry
                                                        1D              ; expire
                                                        3H )            ; minimum
; NS Records
@               IN      NS      ns1.ejemplo3.com.
@               IN      NS      ns2.ejemplo3.com.
@               IN      NS      ns3.ejemplo3.com.

; A Records
@               IN      A       10.8.0.100
ns1             IN      A       11.11.11.11
ns2             IN      A       22.22.22.22
ns3             IN      A       33.33.33.33
nas01           IN      A       10.8.0.101
nas02           IN      A       10.8.0.102
proxmox1        IN      A       10.8.0.103
blog            IN      A       10.8.0.104

; CNAME Records
nextcloud       IN      CNAME   nas01
traefik         IN      CNAME   proxmox1
mail            IN      CNAME   mail.ejemplo3.com.
autodiscover    IN      CNAME   mail.ejemplo3.com.
autoconfig      IN      CNAME   mail.ejemplo3.com.

; TXT Records
networktype     IN      TXT     "vpn_network"

; SRV Records
; Name              Type       Priority Weight Port    Value
_autodiscover._tcp  IN SRV     0        1      443      mail.ejemplo3.com.
_imap._tcp          IN SRV     0        1      143      mail.ejemplo3.com.
_imaps._tcp         IN SRV     0        1      993      mail.ejemplo3.com.
_pop3._tcp          IN SRV     0        1      110      mail.ejemplo3.com.
_pop3s._tcp         IN SRV     0        1      995      mail.ejemplo3.com.
_sieve._tcp         IN SRV     0        1      4190     mail.ejemplo3.com.
_smtps._tcp         IN SRV     0        1      465      mail.ejemplo3.com.
_submission._tcp    IN SRV     0        1      587      mail.ejemplo3.com.
```

#### Vista *publica*

```
;/var/named/bind/publica/ejemplo1.com.zone
$ORIGIN ejemplo1.com.
$TTL 10M
@                               IN SOA  @ ns1.ejemplo1.com. (
                                                        2409032044      ; serial
                                                        1H              ; refresh
                                                        1H              ; retry
                                                        1D              ; expire
                                                        3H )            ; minimum
; NS Records
@               IN      NS      ns1.ejemplo3.com.
@               IN      NS      ns2.ejemplo3.com.
@               IN      NS      ns3.ejemplo3.com.

; CAA Records
@               IN      CAA     0    issue    "letsencrypt.org"

; A Records
@               IN      A       11.11.11.11
ns1             IN      A       11.11.11.11
ns2             IN      A       22.22.22.22
ns3             IN      A       33.33.33.33
blog            IN      A       22.22.22.22

; CNAME Records
www             IN      CNAME   blog
mail            IN      CNAME   mail.ejemplo3.com.
autodiscover    IN      CNAME   mail.ejemplo3.com.
autoconfig      IN      CNAME   mail.ejemplo3.com.

; TXT Records
networktype     IN      TXT     "publica_network"

; SRV Records
; Name              Type       Priority Weight Port    Value
_autodiscover._tcp  IN SRV     0        1      443      mail.ejemplo3.com.
_imap._tcp          IN SRV     0        1      143      mail.ejemplo3.com.
_imaps._tcp         IN SRV     0        1      993      mail.ejemplo3.com.
_pop3._tcp          IN SRV     0        1      110      mail.ejemplo3.com.
_pop3s._tcp         IN SRV     0        1      995      mail.ejemplo3.com.
_sieve._tcp         IN SRV     0        1      4190     mail.ejemplo3.com.
_smtps._tcp         IN SRV     0        1      465      mail.ejemplo3.com.
_submission._tcp    IN SRV     0        1      587      mail.ejemplo3.com.
```

## Crear el archivo docker-compose.yaml para el servidor primario

Este archivo lo creamos para facilitar la administración del contenedor, antes
de proceder debemos asegurarnos de estar en la carpeta en que se ubican los
archivos de configuración.

```bash
cd /datos/contenedores/bind9/
cat > docker-compose.yaml << FIN
services:
  bind918-master:
    image: bind918:latest
    volumes:
      - ./config:/etc/bind
      - ./zones:/var/named/bind
    ports:
      - "53:53/udp"
      - "53:53/tcp"
    restart: unless-stopped
    networks:
      bind9-network:
        ipv4_address: 172.32.0.2
networks:
  bind9-network:
    driver: bridge
    ipam:
     config:
       - subnet: 172.32.0.0/24
         gateway: 172.32.0.1
FIN
```

## Iniciar el contenedor con sus configuraciones

Una vez creado el archivo *docker-compose.yaml* podemos iniciar el contenedor
con el siguiente comando:

```bash
docker compose up
```

Esto iniciará el contenedor acoplado a nuestra consola y mostrará sus logs en pantalla.

Para cambiar su comportamiento podemos iniciarlo en el trasfondo agregando el
parámetro *-d* o deteniendolo con *Ctrl+C* y volviéndolo a iniciar con:

```bash
docker compose start
```

Independientemente de la forma en que se arranque el contenedor, lo podemos
detener con:

```bash
docker compose stop
```

Para que los comandos *docker compose* se debe estar en la misma ruta que el
archivo *docker-compose.yaml*.

Para destruir el contenedor usamos el comando siguiente:

```bash
docker compose down
```

Este último comando no destruye los datos de configuración.


## Iniciar un shell en el contenedor

Para realizar las pruebas es necesario tener un shell en el contenedor docker o
los archivos de configuración en las rutas adecuadas del equipo en el cual se
desean realizar las pruebas. Lo primero es más sencillo y se puede lograr con el
siguiente comando:

```bash
docker exec -ti ddns-master-918 /bin/bash
```

Una vez iniciado el contenedor deberíamos tener un prompt similar al siguiente:

```bash
[root@b0005f055cfa /]#
```

## Verificar el archivo de Configuración

Si no se muestran errores al ejecutar el siguiente comando significa que el
contenido del archivo está correcto.

```bash
named-checkconf /etc/bind/named.conf && echo Ok
```

## Verificar los archivos de zonas

Para cada zona se usa el comando *named-checkzone* pasándole el nombre de la
zona (dominio) y la ruta completa al archivo de zona. A continuación se muestra
un ejemplo con un ciclo for:

```bash
echo -e '\n\n'
for zona in $zones_list
do
    for dominio in $domains_list
    do
        echo -e "\nZona ${zona}, Dominio: ${dominio}"
        named-checkzone ${dominio} /var/named/bind/${zona}/${dominio}.zone
    done
done
```

Un resultado favorable para un archivo de zona sería como el siguiente:

```
Zona lan1, Dominio: ejemplo1.com
zone ejemplo1.com/IN: loaded serial 2504182312
OK

Zona lan1, Dominio: ejemplo2.com
zone ejemplo2.com/IN: loaded serial 2504182312
OK

Zona lan1, Dominio: ejemplo3.com
zone ejemplo3.com/IN: loaded serial 2504182312
OK

[...] 
```

## Abrir puertos en el firewall

Si las configuraciones y pruebas son exitosas se pueden abrir los puertos en el
firewall de nuestro host de contenedores para que el servicio sea accesible
desde Internet.

```bash
firewall-cmd --permanent --add-service=dns
firewall-cmd --permanent --add-service=dns-over-tls
firewall-cmd --reload
```

El servicio *dns-over-tls* solo es necesario si se planea implementar ese tipo
de servicio de DNS.


## Realizar pruebas desde el exterior

Hacemos una *query* a las *IP's* del equipo que aloja nuestro contenedor

```bash
host -t txt networktype ejemplo1.com 192.168.1.100
host -t txt networktype ejemplo1.com 10.8.0.100
```

El resultado esperado sería el siguiente:

```
Using domain server:
Name: 192.168.1.100
Address: 192.168.1.100#53
Aliases:

networktype.ejemplo1.com descriptive text "vpn_network"
```

El texto que está entre comillas en la última línea es el registro TXT
*networktype* de nuestra vista pública.

# Configuración de servidor secundario

## Aceptar tráfico en wireguard
Asegurarse que en las configuraciones de wireguard en el servidor primario se
acepte el tráfico desde el segmento de red de los contenedores secundarios para
cada *peer* que los aloja.

```
*#/etc/wireguard/wg0.conf*
PublicKey = NfwNSKFgI6RXPl3wnobmaYwvSD15rlrAbBwEeZph3Tc=
Endpoint = 11.11.11.11:51820
AllowedIPs = 10.8.0.100/32, 172.37.0.2/32


*#/etc/wireguard/wg0.conf*
PublicKey = jbmgDGY4IjZWP9gINx2ltoRrqKy9nLXnVhbl7IV3GQt=
Endpoint = 22.22.22.22:51820
AllowedIPs = 10.8.0.102/32, 172.37.0.2/32
...
...
```

En el ejemplo anterior se agregó el segmento *172.37.0.0/24* a la configuración
de wireguar.


## Cargar el respaldo de la imagen a docker

```bash
unxz bind918.tar.xz
docker load -i bind918.tar
docker images
```

## Crear y acceder a la carpeta para el contenedor

```bash
mkdir -p /datos/contenedores/bind9/{config,zones}
cd /datos/contenedores/bind9
```

## Copiar configuraciones del contenedor maestro
Desde el contenedor maestro copiar la carpeta *config* a los servidores
esclavos.

```bash
rsync -a \
    /datos/contenedores/bind9/config/ \
    root@10.8.0.101:/datos/contenedores/bind9/config/
```

## Asignar propietarios usuario:grupo a los archivos de configuración

```bash
chown -R root:root {config,zones}
```

Varios puntos de la configuración de los servidores secundarios es similar a la
del primario, los siguientes apartados se pueden ejecutar de la misma manera.

1. Abrir puertos en el firewall

## Modificar los parametros propios del maestro
Cambiar el parlametro notify por el siguiente valor en el archivo
*./config/named.conf* en los servidores esclavos:

```
notify no;
```

## Configurar las vistas

Para configurar las vistas, se toma como base el archivo de configuración de
vistas del servidor autoritativo con las siguientes modificaciones.

### Configurar las llaves de sincronización e IP del maestro

Agregar la siguiente línea después del parámetro *match-clients* para cada una
de las vistas:

```
#Ejemplo para la vista vpn
server 10.8.0.100 { keys vpn; };
```

Esto limita la sincronización de la vista solo a la IP de VPN del servidor
maestro.

### No notificar a otros servidores

Eliminar los bloques *also-notify* de cada vista. No es necesario que los
servidores secundarios notifiquen a otros ya que la *fuente de verdad* es el
servidor primario.

Los bloques son similare al siguiente ejemplo:

```
also-notify {
...
...
}
```

### Definir la IP del servidor maestro para cada zona

Para cada zona de todas las vistas agregar los siguientes parámetros, en los que
se define que las zonas son esclavas, la IP del servidor maestro y el formato en
que se guardarán las zonas sincronizadas.

```
type slave;
masters { 10.8.0.100 key vpn; };
masterfile-format text;
```

## Crear las carpetas para cada zona

```bash
for zona in $zones_list
do
  mkdir zones/${zona}
  chmod 755 zones/${zona}
  chown 25:25 zones/${zona}
done
```

## Crear el archivo docker-compose.yaml para los servidores secundarios

Este archivo lo creamos para facilitar la administración del contenedor, antes
de proceder debemos asegurarnos de estar en la carpeta en que se ubican los
archivos de configuración.

```bash
cd /datos/contenedores/bind9/
cat > docker-compose.yaml << FIN
services:
  bind918-slave:
    image: bind918:latest
    volumes:
      - ./config:/etc/bind
      - ./zones:/var/named/bind
    ports:
      - "53:53/udp"
      - "53:53/tcp"
    restart: unless-stopped
    networks:
      bind9-slave-network:
        ipv4_address: 172.37.0.2
networks:
  bind9-slave-network:
    driver: bridge
    ipam:
     config:
       - subnet: 172.37.0.0/24
         gateway: 172.37.0.1
FIN
```


## Iniciar el contenedor de *BIND*

Los comandos de control se pueden encontrar en la subsección *Iniciar el 
contenedor con sus configuraciones*.

## Realizar pruebas a los servidores esclavos

### Pruebas en la zona *vpn*

Si se realiza la prueba siguiente directamente desde los servidores secundarios.

```bash
for ipaddr in $ip_addr_list_vpn  
do
    for domain in $domains_list
    do
        host -t TXT networktype.$domain $ipaddr
    done
done
```

#### Resultado esperado
Los resultados deben ser similares al siguiente, en el que cambiarían el nombre
del dominio y la IP de los servidores, el registro debe contener el texto
"*vpn_network_<dominio>*".

```
Using domain server:
Name: 10.8.0.101
Address: 10.8.0.101#53
Aliases:

networktype.gl8.ch descriptive text "vpn_network_gl8.ch"
```

### Pruebas en la zona *publica*
Las pruebas para la zona *publica* se realizan mediante el siguiente ciclo.
Estas también se pueden realizar desde nuestra PC de trabajo y deben dar los 
mismos resultados.

```bash
for ipaddr in $ip_addr_list_publica
do
    for domain in $domains_list
    do
        host -t TXT networktype.$domain $ipaddr
    done
done
```

#### Resultado esperado
Los resultados deben ser similares al siguiente, en el que cambiarían el nombre
del dominio y la IP de los servidores, el registro debe contener el texto
"*publica_network_<dominio>*".

```
Using domain server:
Name: 11.11.11.11
Address: 11.11.11.11#53
Aliases:

networktype.gl8.ch descriptive text "publica_network_gl8.ch"
```

# Referencias
[kb.isc.org Understanding views in BIND 9](https://kb.isc.org/docs/aa-00851)

[kb.isc.org ACLs with both addresses and keys](https://kb.isc.org/docs/aa-00723)

[kb.isc.org Avoid view transferred from the same primary view](https://kb.isc.org/docs/aa-00296)

[zytrax.com named.conf configuration](https://www.zytrax.com/books/dns/ch7/)

[zytrax.com named.conf Bind9 List of Statements](https://www.zytrax.com/books/dns/ch7/statements.html)

[linuxconfig.org rndc key for bind](https://linuxconfig.org/configure-rndc-key-for-bind-dns-server-on-centos-7)

[computingforgeeks.com configure slave bind server](https://computingforgeeks.com/configure-slave-bind-dns-server-on-ubuntu)

[cyberciti.biz bind9 named configure views)](https://www.cyberciti.biz/faq/linux-unix-bind9-named-configure-views)

[golinuxcloud.com Security_Challenges_with_DNS_Server](https://www.golinuxcloud.com/secure-master-slave-dns-server-dnssec-linux/#Security_Challenges_with_DNS_Server)

[serverfault.com disable dnsmasq in libvirt](https://serverfault.com/questions/937188/disable-or-change-port-of-dnsmasq-service-in-libvirt)

[jensd.be Basic master and slave dns setup with BIND](https://jensd.be/197/linux/basic-master-and-slave-dns-setup-with-bind)

[mpolinowski.github.io Installing BIND9 with docker](https://mpolinowski.github.io/docs/DevOps/Provisioning/2022-01-25--installing-bind9-docker/2022-01-25/)

[okta.com SOA record](https://www.okta.com/identity-101/soa-record/)

[geeksforgeeks.org Export and import docker containers and images/](https://www.geeksforgeeks.org/export-and-import-docker-containers-and-images/)
