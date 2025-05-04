---
title: "Crear red WireGuard"
date: 2025-03-15T23:11:34-06:00
draft: false
---

# Requerimiento

Crear una red *overlay* que permita la comunicación de manera segura entre
varios dispositivos para acceder a servcios privados que solo estarán
disponibles en dicha red.

# Tecnología y arquitectura seleccionadas

La tecnología seleccionada para montar la red es el protocolo y herramientas
WireGuard, debido que es código abierto, a su simplicidad y velocidad superior
a otras alternativas (por ejemplo OpenVPN).

La arquitectura de la red es tipo estrella, en la cual uno de los dispositivos
fungirá como *Servidor de VPN* ya que todos los túneles de dicha red apuntarán a
él, por ende todo el tráfico será ruteado a través de este dispositivo.

Los problemas derivados de esta arquitectura son que el tráfico tiene que viajar
a través de un solo dispositivo de la red aumentando el trafico consumido en él
y generando retardos (lag) en las comunicaciones.

Esto se podría solventar usando herramientas y metodos *STUN* los cuales
permitirían conectar cada host directamente sin necesidad de un *servidor de
VPN.* Dicho metodo está fuera del alcance de esta guía.

# Datos iniciales

Se cuenta con los siguientes dispositivos que se desean conectar entre si:

* VPS en la nube, accesible desde Internet, aloja parte de los servicios que se desean consumir.
* Servidor NAS, solo accesible desde la red privada *lan1*, aloja parte de los servicios que se desean consumir.
* Laptop, usada para consumir los servicios, se conecta desde distintas redes.
* Teléfono celular, usado para consumir los servicios, se conecta desde distintas redes.

# Instalar las herramientas de administración

```bash
# Debian/Ubuntu
apt install wireguard

# RHEL/Fedora
dnf install wireguard-tools

# Arch
pacman -S wireguard-tools
```

Para otras distribuciones referirse a la documentación oficial(ver referencias).

# Generación de llaves

## Consideraciones de seguridad y privacidad

Para el cifrado de las comunicaciones, WireGuard utiliza una arquitectura de
llaves pública/privada. Idealmente los usuarios que se van a conectar deberían
generar sus pares de llaves, y compartir solo las llaves públicas para ser
instaladas en los hosts a los que tendrán acceso, para así asegurar la
privacidad de las comunicaciones.

En este ejemplo todas las llaves son generadas por el operador de la red ya que
esta es administrada en su totalidad por una sola persona.

## Carpeta de trabajo

Crear carpeta de trabajo en la cual residirán los archivos de configuración

```bash
mkdir wg-config-files
```

## Script para generar llaves

Para hacer la generación de llaves y archivos de configuración nos apoyamos con
el siguiente script.

Este consta de dos secciones principales, en la primera se declaran, la lista de
hosts y el puerto en el que escucharán los demonios de WireGuard, por
simplicidad se usa el mismo para todos los hosts, además se

La lista es una suscesion de los siguientes datos separados por ':'

- Nombre_de_Host.
- IP_Pública ('not_ip' si no son accesibles desde Internet).
- IP_de_VPN, la dirección dentro de la red WireGuard.
- La cantidad de segundos a esperar para enviar paquetes *Keep Alive* los cuales ayudan a mantener la conexión activa (no_keepalive si no son accesibles desde Internet).

En la segunda parte del script se generan las llaves criptográficas y con base
en la lista de hosts, se generan los archivos de configuración para cada host.

```bash
# Puerto en el que escucha WireGuard
wg_port=63260

lista_hosts="\
    server01:11.11.11.11:10.8.0.11:24 \
    server02:22.22.22.22:10.8.0.22:24 \
    laptop01:not_ip:10.8.0.10:no_keepalive \
    cellphone01:not_ip:10.8.0.9:no_keepalive"

for host_string in $lista_hosts
do
    # Separar nombres de hosts e IP's
    IFS=":" read -r host ip_addr wg_ip_addr keep_alive <<< $host_string

    # Obtener los pares de llaves
    priv_key=$(wg genkey)
    pub_key=$(echo $priv_key | wg pubkey)

    # Crea los encabezados de los archivos de configuración para cada host
    echo >> wg0-peers.conf
    cat >> wg0-peers.conf <<-FINAL
        # ${host}
        [peer]
        PublicKey = ${pub_key}
        Endpoint = ${ip_addr}:${wg_port}
        AllowedIPs = ${wg_ip_addr}/32
        PersistentKeepalive = ${keep_alive}
FINAL
    cat > wg0-${host}.conf <<-FINAL
        [interface]
        PrivateKey = ${priv_key}
        ListenPort = ${wg_port}
        Address = ${wg_ip_addr}/24
FINAL
done

# Eliminar los parámetros inválidos
sed -i -e '/no_keepalive/d' -e '/not_ip/d' *conf

# Agrega la lista de peers a cada archivo de configuración
for host_string in $lista_hosts
do
    # Separar nombres de hosts e IP's
    IFS=":" read -r host ip_addr wg_ip_addr keep_alive <<< $host_string
    cat wg0-peers.conf >> wg0-${host}.conf
done
```

# Instalar archivos de configuración

En la mayoría de las distribuciones la ubicación del archivo sería

```bash
/etc/wireguard/wg0.conf
```

En dónde *wg0.conf* es el nombre que le daremos al archivo de configuración del
host, *wg0* también será el nombre que el sistema le dará a la interfaz de red
de WireGuard.

# Iniciar y detener el servicio de WireGuard.

## Iniciar servicio

```bash
wg-quick up wg0
```

## Detener servicio

```bash
wg-quick down wg0
```

## Carpeta de trabajo

Una vez verificado el funcionamiento de la red privada se recomienda eliminar la
carpeta de trabajo, en caso de que necesite volver a crear la red, lo más
recomendable es generar llaves nuevas para todos los equipos de la red.

```bash
shred -u wg0-*.conf
cd ..
rm -rf wg-config-files
```

# Configurar el servicio para que arranque automáticamente

Se configura el servicio de *systemd* para que arranque la interfáz *wg0* cuando
arranque el sistema.

```bash
systemctl enable wg-quick@wg0
```

Con esto la interfáz podrá se controlada como cualquier servicio de *systemd*.

```bash
systemctl stop wg-quick@wg0
systemctl status wg-quick@wg0
systemctl start wg-quick@wg0
```

# Referencias

[Wireguard quickstart](https://www.wireguard.com/quickstart/)

[Wireguard conceptual-overview](https://www.wireguard.com/#conceptual-overview)

[Wireguard install](https://www.wireguard.com/install/)

[serverfault.com](https://serverfault.com/questions/1006595/cannot-setup-wireguard-vpn)

[wikipedia.org STUN](https://en.wikipedia.org/wiki/STUN)
