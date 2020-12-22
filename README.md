# Gateway Appliance Juniper - Configuración :cloud:

IBM Cloud Juniper vSRX le permite enrutar selectivamente el tráfico de red pública y privada, a través de un firewall de nivel empresarial que funciona con características de software de JunOS, como stack de enrutamiento completo, tráfico compartido, enrutamiento basado en políticas y VPN.

A continuación se detalla la configuración de una VPN basada en ruta entre dos zonas, la  Zona A y la Zona B, junto con la descripción del enrutamiento necesario para permitir el tráfico entre las máquinas de las diferentes zonas y el paso a paso para la traducción de IPs mediante NAT en una de las zonas.

![](https://user-images.githubusercontent.com/60897075/102897440-ffc52e00-4435-11eb-8541-e16510e73c39.png)


### Contenido

1.  [Acceso al Gateway Appliance](#acceso-al-gateway-appliance)
2.  [Configuración de la VPN](#configuración-de-la-vpn)
3.  [Enrutamiento](#enrutamiento)
4.  [Políticas de seguridad](#políticas-de-seguridad)
5.  [NAT](#nat)
6.  [Comprobación](#comprobación)
7.  [Referencias](#referencias)

## Acceso al Gateway Appliance

Para configurar el gateway appliance necesitará conocer su IP pública, para esto acceda al menú de tres líneas de su cuenta de IBM Cloud, seleccione **Infraestructura clásica**, luego en la sección **Network** seleccione **Gateway appliance**, al seleccionar su dispositivo encontrará una descripción de este, tome nota de la dirección ip pública que en el diagrama corresponde a **ip-gw-1-pub** y de la **clave-de-acceso**, luego de esto podrá acceder a el mediante las siguientes dos modalidades:

### A. Interfaz J-Web

En el buscador ingrese la IP de su gateway appliance junto con el puerto 8443 de la siguiente forma \<ip-gw-1-pub>:8443. De este modo ingresará a la interfaz gráfica y deberá especificar el usuario y contraseña, estos parámetros los encontrará en la descripción del dispositivo.

![](https://user-images.githubusercontent.com/60897075/102824888-b16b4d00-43ab-11eb-8806-083b1c769f28.png)

### B. Conexión SSH

En la línea de comandos de su equipo ingrese el comando de conexión SSH:

```shell
ssh admin@<ip-gw-1-pub>
```

Cuando se le pida la contraseña ingrese la clave anotada como **clave-de-acceso**. Podrá rectificar que se encuentra en la consola del dispositivo al ver la etiqueta con el nombre que le dio a su instancia.

## Configuración de la VPN

La Junos OS CLI trabaja en dos modalidades diferentes, la primera es la modalidad de operación y la segunda la de configuración, todos los pasos a continuación se harán en la modalidad de configuración, para acceder a esta ingrese:

```shell
configure
```

Para la configuración del túnel VPN ipsec, Juniper hace uso del protocolo de intercambio de claves de internet (IKE) el cual trabaja en dos fases:

### **FASE 1:**

En esta fase se establece un canal seguro para la comunicación entre dispositivos. Ingrese los siguientes comandos en el gateway para la configuración de la fase 1 del protocolo, teniendo en cuenta que, como se muestra en el diagrama, la dirección **ip-gw-2-pr** es la dirección ip privada del peer de la zona B. Recuerde que los valores tomados en esta guía son valores a modo de ejemplo, los parámetros reales se deben acordar entre las entidades a cargo de la configuración del túnel en cada zona.

```shell
set security ike proposal IKE-PROP lifetime-seconds 3600
set security ike proposal IKE-PROP authentication-method pre-shared-keys
set security ike proposal IKE-PROP authentication-algorithm sha1
set security ike proposal IKE-PROP encryption-algorithm aes-128-cbc

set security ike proposal IKE-PROP dh-group group5
set security ike policy IKE-POL proposals IKE-PROP
set security ike policy IKE-POL mode main
set security ike policy IKE-POL pre-shared-key ascii-text <pre-shared-key>

set security ike gateway IKE-GW ike-policy IKE-POL
set security ike gateway IKE-GW address <ip-gw-2-pr>
set security ike gateway IKE-GW external-interface ge-0/0/1

set security zones security-zone SL-PUBLIC host-inbound-traffic system-services ike
```

### **FASE 2:**

En esta fase se establece el túnel VPN para el tráfico de red. Ingrese los siguientes comandos para la configuración de la fase 2 del protocolo, fíjese que se hará uso de la interfaz st0.1 y esto podría cambiar dependiendo de su configuración. Recuerde que los valores tomados en esta guía son valores a modo de ejemplo, los parámetros reales se deben acordar entre las entidades a cargo de la configuración del túnel en cada zona.

```shell
set security ipsec proposal IPSEC-PROP lifetime-seconds 3600
set security ipsec proposal IPSEC-PROP protocol esp
set security ipsec proposal IPSEC-PROP authentication-algorithm hmac-sha1-96
set security ipsec proposal IPSEC-PROP encryption-algorithm aes-128-cbc

set security ipsec policy IPSEC-POL proposals IPSEC-PROP
set security ipsec policy IPSEC-POL perfect-forwad-secrecy keys group5

set security ipsec vpn IPSEC-VPN ike gateway IKE-GW
set security ipsec vpn IPSEC-VPN ike ipsec-policy IPSEC-POL
set security ipsec vpn IPSEC-VPN vpn-monitor
set security ipsec vpn IPSEC-VPN establish-tunnels immediately

set security ipsec vpn IPSEC-VPN bind-interface st0.1
```

## Enrutamiento

### Enrutamiento del Gateway Appliance

El enrutamiento desde el Juniper permite que el tráfico entre las instancias de la zona A dirigido a las instancias de la zona B salga por la interfaz del gateway appliance conectado con el peer de la zona B. Para lograr esto puede crear una nueva ruta desde la interfaz J-Web donde la **dirección IP** sea: 0.0.0.0/0 y el **next-hop** sea la dirección ip gateway de la subred pública del gateway 1.

![](https://user-images.githubusercontent.com/60897075/102827738-1f664300-43b1-11eb-81f2-53ba496fb620.gif)

### Enrutamiento de las instancias

Es necesaria la configuración de las tablas de enrutamiento de las instancias que hacen parte del dominio de encripción, para que de esta forma el tráfico de la zona A con destino a la Zona B salga mediante la VPN. Para esto se deben agregar las rutas persistentes que en el caso de las máquinas windows puede hacerse mediante el siguiente comando:

```shell
route add -p <dominio-encripcion-3> mask <mascara-dominio-encripcion-3> <vlan-gateway>
route add -p <dominio-encripcion-4> mask <mascara-dominio-encripcion-4> <vlan-gateway>
```

El comando anterior se ejecuta en una instancia de la Zona A y en el se especifica el dominio de encripción de la Zona B, la máscara del dominio y finalmente la dirección IP gateway de la vlan de la zona A. Es necesario correr el comando para cada uno de los dominios de encripción relacionados a la VPN. Puede verificar que el enrutamiento se haya hecho de forma correcta con el comando a continuación, deberá ver las direcciones configuradas en la sección **Rutas persistentes**.

```shell
netstat -r
```

Tenga en cuenta que los pasos anteriormente mencionados permiten la configuración desde el punto de vista de la Zona A, sin embargo es necesario hacerlo también para la zona B.

### Enrutamiento de la VLAN

Para configurar el enrutamiento de la VLAN de la zona A cuyo ID en esta guía es id-vlan, se ejecutan los siguientes comandos, teniendo en cuenta que la vlan-gateway es la dirección IP gateway de la VLAN privada de la zona A.

```shell
set security zones security-zone SL-PRIVATE interfaces ae0.<id-vlan> host-inbound-traffic system-services all 
set security zones security-zone SL-PRIVATE interfaces ae0.<id-vlan> host-inbound-traffic protocols all 
set interfaces ae0 unit <id-vlan> vlan-id <id-vlan>
set interfaces ae0 unit <id-vlan> family inet address <gw-vlan>
```

El siguiente comando es utilizado para dar una descripción a la interfaz y de esta forma facilitar su reconocimiento.

```shell
set interfaces ae0 unit <id-vlan> description VLAN-SERVERS
```

## Políticas de seguridad

Las políticas de seguridad pueden configurarse en uno de los peers o en ambos y serán establecidas según las necesidades de la implementación. A continuación, se listan los comandos para la creación de las políticas de seguridad que permiten el tráfico de la red privada a la VPN y de la VPN a la red privada.

```shell
set security policies from-zone SL-PRIVATE to-zone VPN policy Trust-to-VPN match source-address any destination-address any application any
set security policies from-zone SL-PRIVATE to-zone VPN policy Trust-to-VPN then permit
set security policies from-zone VPN to-zone SL-PRIVATE policy VPN-to-Trust match source-address any destination-address any application any
set security policies from-zone VPN to-zone SL-PRIVATE policy VPN-to-Trust then permit
```

Adicionalmente puede ser útil el uso de libros de direcciones para dar un nombre a cada subred o IP de la arquitectura y que de esta forma sean más fáciles de identificar. Para crearlo puede ejecutar el siguiente comando:

```shell
set security address-book global address <nombre> <subred-o-ip>
```
## Selector de tráfico

Un selector de tráfico es un acuerdo entre pares IKE para permitir el tráfico a través de un túnel si el tráfico coincide con un par específico de direcciones locales y remotas. Dado que, como se mencionó en la sección anterior, todo el tráfico pasará a través de la dirección **ip_nat**, para cada dirección remota perteneciente al dominio de encripción de la zona B, le asignamos la dirección local **ip_nat**.

Para configurar el selector de tráfico desde la interfaz J-Web vaya a **Security services**, luego en **IPsec VPN** seleccione **IPsec (phase II)** y haga clic en el símbolo de añadir; una vez allí ingrese a la pestaña **IPsec VPN Options** para habilitar la opción **Traffic selector**, finalmente añada un selector por cada dominio de encripción de la zona B. En la guía el parámetro **Local IP/Netmask** toma el valor de **ip_nat** y el parámetro **Remote IP/Netmask** serían **dominio-encripcion-3** y **dominio-encripcion-4**.

![](https://user-images.githubusercontent.com/60897075/102900585-7fed9280-443a-11eb-9ae7-974ab6f1cb01.gif)

## NAT

En esta sección se muestra la configuración de **NAT Source** en el gateway appliance para traducir las IPs de origen de todos los paquetes que salen de la zona A a la zona B a una sola dirección ip que en el diagrama se referencia como **ip\_nat**.

Para configurar NAT debe ingresar mediante la interfaz J-Web y en la pestaña **security services** elegir **NAT**, luego **Source** y finalmente crear una nueva **nat pool** desde la **ip\_nat** hasta la **ip\_nat** debido a que todas las IPs del tráfico que sale de la zona A serán traducidas a esta única dirección. En este caso la máscara utilizada es de 32 por tratarse de una sola ip, sin embargo también se puede configurar traducción a varias IPs.

![](https://user-images.githubusercontent.com/60897075/102830104-994cfb00-43b6-11eb-9ec3-b87359166136.gif)

Una vez creado el Pool Nat, procedemos a crear las reglas del NAT Source. Para este paso, en el parámetro **Routing instance** seleccionamos Zone, luego para **From** elegimos la zona **SL-PRIVATE** y en **To** elegimos la zona **VPN**. Finalmente creamos una nueva regla por cada dominio de encripción de la zona B. Al crear las reglas se debe elegir como **Source Address and Ports** la subred correspondiente a la vlan de la zona A y como **Destination Address and Ports** cada uno de los dominios de encripción de la zona B, finalmente en marcamos **Do Source NAT With Pool** y en el menú desplegable elegimos la pool nat creada anteriormente. Los nombres a seleccionar en  **Source Address and Ports** y **Destination Address and Ports** dependerán de los nombres que haya configurado en el libro de direcciones de la sección anterior, para el gift a continuación la Network-A corresponde a la vlan y la Network-B corresponde al dominio-encripcion-3 mostrado en el diagrama.

![](https://user-images.githubusercontent.com/60897075/102831329-84259b80-43b9-11eb-913b-e60c0a770351.gif)

## Comprobación
Utilice los siguientes comandos para ver que cada una de las fases del tunel han sido creadas y que se encuentren activas, además de identificar las configuraciones anteriormente establecidas.

```shell
run show security ipsec security-associations
run show security ike security-associations
```
El resultado esperado se muestra en la siguiente imagen, fíjese en los valores **Total active tunels** y **State**.

![](https://user-images.githubusercontent.com/60897075/102897442-ffc52e00-4435-11eb-9b3c-eea2bfc8ac99.png)

En caso de que el estado se encuentre en DOWN siga los pasos de la [documentación](https://kb.juniper.net/InfoCenter/index?page=content&id=KB10100&actp=search) proporcionada por juniper.

Si en algún momento desea revertir una configuración aplicada puede utilizar los siguientes comandos para ver el historial de los commits realizados, elegir a cúal debe regresar y finalmente con el útlimo comando podrá volver al commit deseado.

```shell
show system commit
rollback <commit-number>
```

## Referencias

La descripción detallada del IBM Cloud Juniper vSRX la puede encontrar en este [link](https://cloud.ibm.com/docs/vsrx?topic=vsrx-about-ibm-cloud-juniper-vsrx).

La guía base sobre la creación de una conexión segura con un igual de Juniper vSRX remoto la puede encontrar en este [link](https://cloud.ibm.com/docs/vpc-on-classic-network?topic=vpc-on-classic-network-creating-a-secure-connection-with-a-remote-juniper-vsrx-peer&locale=es).

## Autores
IBM Cloud Tech Sales :cloud:
