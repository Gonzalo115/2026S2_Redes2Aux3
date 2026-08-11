**DOCUMENTACIÓN DE LA RED**

**Enrutamiento EIGRP • EtherChannel LACP/PAgP • VLAN • 
Router-on-a-Stick** • DHCP


*Figura 1. Topología de la red documentada.*

# 1. Descripción general

La topología está formada por tres switches multicapa (Multilayer Switch
0, 1 y 2), un router (Router 1), un switch de acceso (Switch 3) y un
servidor. Los enlaces entre los switches multicapa utilizan EtherChannel
para agrupar tres enlaces físicos en un único enlace lógico. Se emplean
LACP y PAgP según el segmento. El enrutamiento interno se realiza
mediante EIGRP AS 1.

Las LAN de la parte inferior están separadas lógicamente mediante VLAN y
subredes. Los switches multicapa funcionan como gateways mediante
interfaces SVI y ejecutan ip routing. La red superior corresponde a la
VLAN 10 (192.168.50.0/25), cuyo gateway está configurado en Router 1
mediante una subinterfaz 802.1Q.

# 2. Esquema de direccionamiento

  ----------------------------------------------------------------------------------
  **Segmento**   **Red**           **Direcciones**   **Interfaces**   **Tipo**
  -------------- ----------------- ----------------- ---------------- --------------
  Enlace MS0 ↔   10.0.1.0/30       MS0: 10.0.1.2 \|  Port-channel2 /  LACP
  MS1                              MS1: 10.0.1.1     Port-channel2    

  Enlace MS1 ↔   10.0.1.4/30       MS1: 10.0.1.5 \|  Port-channel1 /  LACP
  MS2                              MS2: 10.0.1.6     Port-channel1    

  Enlace MS0 ↔   10.0.1.8/30       MS0: 10.0.1.10 \| Port-channel3 /  PAgP
  MS2                              MS2: 10.0.1.9     Port-channel3    

  Enlace MS2 ↔   10.0.1.12/30      Router: 10.0.1.13 Gi0/0 / Gi0/1    Enlace L3
  Router 1                         \| MS2: 10.0.1.14                  

  VLAN 10        192.168.50.0/25   Gateway:          Router 1         802.1Q
  superior                         192.168.50.1      Gi0/1.10         

  VLAN 10        192.168.10.0/24   Gateway:          MS1 SVI          LAN
  inferior                         192.168.10.1                       

  VLAN 20        192.168.20.0/24   Gateway:          MS1 SVI          LAN
                                   192.168.20.1                       

  VLAN 30        192.168.30.0/24   Gateway:          MS0 SVI          LAN
                                   192.168.30.1                       

  VLAN 40        192.168.40.0/24   Gateway:          MS0 SVI          LAN
                                   192.168.40.1                       
  ----------------------------------------------------------------------------------


# 4. Configuración de EtherChannel con LACP

LACP (Link Aggregation Control Protocol) permite negociar dinámicamente
la agrupación de enlaces. En la configuración proporcionada se utiliza
mode active en ambos extremos, por lo que los dos lados participan
activamente en la negociación.

## 4.1 Multilayer Switch 0 --- LACP hacia Multilayer Switch 1

enable\
configure terminal\
interface range fastethernet 0/7 - 9\
no switchport\
no ip address\
channel-protocol lacp\
channel-group 2 mode active\
no shutdown\
exit\
\
interface port-channel 2\
no switchport\
ip address 10.0.1.2 255.255.255.252\
no shutdown\
exit

## 4.2 Multilayer Switch 1 --- LACP hacia Multilayer Switch 0

enable\
configure terminal\
interface range fastethernet 0/7 - 9\
no switchport\
no ip address\
channel-protocol lacp\
channel-group 2 mode active\
no shutdown\
exit\
\
interface port-channel 2\
no switchport\
ip address 10.0.1.1 255.255.255.252\
no shutdown\
exit

## 4.3 Multilayer Switch 1 --- LACP hacia Multilayer Switch 2

enable\
configure terminal\
interface range fastethernet 0/1 - 3\
no switchport\
no ip address\
channel-protocol lacp\
channel-group 1 mode active\
no shutdown\
exit\
\
interface port-channel 1\
no switchport\
ip address 10.0.1.5 255.255.255.252\
no shutdown\
exit

## 4.4 Multilayer Switch 2 --- LACP hacia Multilayer Switch 1

enable\
configure terminal\
interface range fastethernet 0/1 - 3\
no switchport\
no ip address\
channel-protocol lacp\
channel-group 1 mode active\
no shutdown\
exit\
\
interface port-channel 1\
no switchport\
ip address 10.0.1.6 255.255.255.252\
no shutdown\
exit

# 5. Configuración de EtherChannel con PAgP

PAgP (Port Aggregation Protocol) es el protocolo propietario de Cisco
utilizado en el tercer enlace entre los switches multicapa. En este caso
ambos extremos usan mode desirable, por lo que ambos intentan negociar
activamente el EtherChannel.

## 5.1 Multilayer Switch 0 --- PAgP hacia Multilayer Switch 2

enable\
configure terminal\
interface range fastethernet 0/4 - 6\
no switchport\
no ip address\
channel-protocol pagp\
channel-group 3 mode desirable\
no shutdown\
exit\
\
interface port-channel 3\
no switchport\
ip address 10.0.1.10 255.255.255.252\
no shutdown\
exit

## 5.2 Multilayer Switch 2 --- PAgP hacia Multilayer Switch 0

enable\
configure terminal\
interface range fastethernet 0/4 - 6\
no switchport\
no ip address\
channel-protocol pagp\
channel-group 3 mode desirable\
no shutdown\
exit\
\
interface port-channel 3\
no switchport\
ip address 10.0.1.9 255.255.255.252\
no shutdown\
exit

# 6. Enrutamiento EIGRP

Los tres switches multicapa y Router 1 utilizan EIGRP con Autonomous
System 1. El objetivo es intercambiar las rutas de las redes LAN y de
los enlaces de tránsito. La interfaz de cada Port-channel funciona como
una interfaz de capa 3.

## 6.1 Multilayer Switch 0

configure terminal\
ip routing\
\
router eigrp 1\
network 10.0.1.0 0.0.0.3\
network 10.0.1.8 0.0.0.3\
network 192.168.30.0 0.0.0.255\
network 192.168.40.0 0.0.0.255\
no auto-summary\
exit

## 6.2 Multilayer Switch 1

configure terminal\
ip routing\
\
router eigrp 1\
network 10.0.1.0 0.0.0.3\
network 10.0.1.4 0.0.0.3\
network 192.168.10.0 0.0.0.255\
network 192.168.20.0 0.0.0.255\
no auto-summary\
exit

## 6.3 Multilayer Switch 2

configure terminal\
ip routing\
\
router eigrp 1\
network 10.0.1.0 0.0.0.255\
network 10.0.1.4 0.0.0.3\
network 10.0.1.8 0.0.0.3\
network 10.0.1.12 0.0.0.3\
no auto-summary\
exit

Nota: en MS2 se recomienda reemplazar el network 10.0.1.0 0.0.0.255 por
las redes específicas 10.0.1.0/30, 10.0.1.4/30 y 10.0.1.8/30, si se
desea que EIGRP sea más preciso y no active el protocolo en interfaces
no previstas.

## 6.4 Router 1

configure terminal\
router eigrp 1\
network 10.0.1.12 0.0.0.3\
network 192.168.50.0 0.0.0.127\
no auto-summary\
exit

# 7. Interfaces VLAN (SVI) y gateways

## 7.1 Multilayer Switch 0

configure terminal\
ip routing\
\
interface vlan 30\
ip address 192.168.30.1 255.255.255.0\
no shutdown\
exit\
\
interface vlan 40\
ip address 192.168.40.1 255.255.255.0\
no shutdown\
exit

## 7.2 Multilayer Switch 1

configure terminal\
ip routing\
\
interface vlan 10\
ip address 192.168.10.1 255.255.255.0\
ip helper-address 192.168.50.2\
no shutdown\
exit\
\
interface vlan 20\
ip address 192.168.20.1 255.255.255.0\
no shutdown\
exit

El comando ip helper-address 192.168.50.2 permite reenviar solicitudes
DHCP desde la VLAN 10 hacia un servidor ubicado en la red
192.168.50.0/25, suponiendo que 192.168.50.2 funciona como servidor
DHCP.

# 8. Red superior: VLAN 10 y Router-on-a-Stick

Router 1 utiliza la interfaz GigabitEthernet0/1 como enlace hacia Switch
3. La subinterfaz Gi0/1.10 encapsula las tramas etiquetadas con 802.1Q
para VLAN 10 y proporciona el gateway 192.168.50.1/25.

configure terminal\
interface gigabitethernet 0/1\
no ip address\
no shutdown\
exit\
\
interface gigabitethernet 0/1.10\
encapsulation dot1Q 10\
ip address 192.168.50.1 255.255.255.128\
no shutdown\
exit

## 8.1 Switch 3

El puerto donde está conectado el servidor se configura como access VLAN
10. El puerto que conecta Switch 3 con Router 1 debe configurarse como
trunk 802.1Q. Como en el running-config entregado no se identifica con
certeza el número de puerto del uplink, se muestra la configuración como
plantilla y debe aplicarse al puerto real.

configure terminal\
vlan 10\
name SERVIDORES\
exit\
\
! Puerto del servidor (en la configuración entregada: Fa0/1)\
interface fastethernet 0/1\
switchport mode access\
switchport access vlan 10\
no shutdown\
exit\
\
! Uplink hacia Router 1: sustituir X por el puerto real\
interface fastethernet 0/X\
switchport mode trunk\
switchport trunk allowed vlan 10\
no shutdown\
exit

# 9. Comandos de verificación y diagnóstico

  -----------------------------------------------------------------------
  **Objetivo**                        **Comando**
  ----------------------------------- -----------------------------------
  Estado de interfaces                show ip interface brief

  Tabla de rutas                      show ip route

  Rutas EIGRP                         show ip route eigrp

  Vecinos EIGRP                       show ip eigrp neighbors

  Interfaces anunciadas por EIGRP     show ip protocols

  Resumen de EtherChannel             show etherchannel summary

  Detalle LACP/PAgP                   show etherchannel detail

  Estado de un Port-channel           show interfaces port-channel 1

  VLAN existentes                     show vlan brief

  Trunks                              show interfaces trunk

  Estado de una SVI                   show interfaces vlan 10

  Configuración actual                show running-config

  Prueba de conectividad              ping \<IP-destino\>

  Ruta de un destino                  traceroute \<IP-destino\>
  -----------------------------------------------------------------------

# 10. Pruebas recomendadas

  -----------------------------------------------------------------------
  **Prueba**              **Comando**             **Resultado esperado**
  ----------------------- ----------------------- -----------------------
  MS0 → MS1               ping 10.0.1.1           Debe responder el
                                                  enlace L3 del
                                                  Port-channel.

  MS1 → MS2               ping 10.0.1.6           Debe responder el
                                                  enlace L3 del
                                                  Port-channel.

  MS0 → MS2               ping 10.0.1.9           Debe responder el
                                                  enlace PAgP.

  MS2 → Router 1          ping 10.0.1.13          Debe responder el
                                                  enlace L3.

  Router 1 → servidor     ping 192.168.50.2       Debe responder si el
                                                  servidor está
                                                  configurado
                                                  correctamente.

  MS1 → VLAN 30           ping 192.168.30.1       Debe existir una ruta
                                                  EIGRP hacia la LAN de
                                                  MS0.

  Host VLAN 10 → VLAN 20  ping 192.168.20.x       Debe funcionar mediante
                                                  las SVI de MS1.

  Host LAN → servidor     ping 192.168.50.2       Debe funcionar si EIGRP
                                                  y el gateway están
                                                  correctamente
                                                  configurados.
  -----------------------------------------------------------------------

# 11. Observaciones importantes sobre la configuración entregada

-   En los tres switches multicapa se usa ip routing, por lo que las SVI
    y Port-channel pueden participar en el enrutamiento de capa 3.

-   Los Port-channel mostrados están configurados como enlaces de capa 3
    mediante no switchport; por ello no se deben configurar como trunks.

-   LACP está implementado entre MS0--MS1 y MS1--MS2. PAgP está
    implementado entre MS0--MS2.

-   En la configuración original aparece auto-summary en EIGRP. Para una
    práctica con subredes modernas y segmentadas se recomienda no
    auto-summary.

-   En MS2 aparece network 10.0.1.0 0.0.0.255. Es funcionalmente más
    amplio de lo necesario; se recomienda declarar únicamente las
    subredes de tránsito utilizadas.

-   La VLAN 10 de la parte superior (192.168.50.0/25) y la VLAN 10 de
    MS1 (192.168.10.0/24) tienen el mismo número de VLAN pero pertenecen
    a segmentos diferentes. El número de VLAN por sí solo no determina
    la subred IP.

-   Para Router-on-a-Stick, el enlace Router 1--Switch 3 debe ser trunk
    y el puerto del servidor debe ser access VLAN 10. En el
    running-config proporcionado no se muestra el puerto de uplink de
    Switch 3, por lo que debe identificarse antes de aplicar el comando.

-   Si 192.168.50.2 es el servidor DHCP, el ip helper-address de MS1
    permite que los clientes de 192.168.10.0/24 soliciten direcciones al
    servidor. El servidor debe tener rutas de retorno hacia las LAN
    internas, ya sea mediante una ruta estática o el mecanismo de
    enrutamiento correspondiente.

# 12. Resumen de la arquitectura

La red utiliza una arquitectura jerárquica de capa 3 en la que los tres
switches multicapa forman el núcleo de interconexión. Los enlaces
redundantes se agrupan mediante EtherChannel, reduciendo la dependencia
de un único enlace físico y proporcionando mayor capacidad. LACP se
utiliza en dos de los lados de la topología y PAgP en el enlace
restante. EIGRP permite distribuir las rutas entre las LAN
192.168.10.0/24, 192.168.20.0/24, 192.168.30.0/24, 192.168.40.0/24 y la
red superior 192.168.50.0/25. Las VLAN inferiores permanecen segmentadas
y se comunican mediante enrutamiento inter-VLAN en los switches
multicapa.

**Fin de la documentación**
