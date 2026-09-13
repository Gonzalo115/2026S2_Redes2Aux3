# Ejemplo de clase: redundancia HSRP y enrutamiento EIGRP

## 1. Objetivo

El ejemplo implementa una puerta de enlace redundante para la VLAN 10 mediante **HSRP**. Router0 y Router1 comparten la dirección virtual `192.168.50.1`, que debe configurarse como gateway en los equipos de la VLAN. Además, **EIGRP AS 1** permite intercambiar rutas entre los routers y el switch multicapa.

Si el router activo deja de funcionar, el router en espera asume la IP virtual y los dispositivos de la LAN pueden continuar enviando tráfico sin cambiar su puerta de enlace.

## 2. Esquema de direccionamiento

| Segmento | Dispositivo | Interfaz | Dirección |
| --- | --- | --- | --- |
| VLAN 10 | Gateway virtual HSRP | Grupo 50 | `192.168.50.1/24` |
| VLAN 10 | Router0 | `G0/1.10` | `192.168.50.3/24` |
| VLAN 10 | Router1 | `G0/1.10` | `192.168.50.2/24` |
| Router0–Multilayer | Router0 | `G0/0` | `10.0.1.21/30` |
| Router0–Multilayer | Multilayer0 | Puerto enrutado | `10.0.1.22/30` |
| Router1–Multilayer | Router1 | Puerto enrutado | `10.0.1.13/30`* |
| Router1–Multilayer | Multilayer0 | Puerto enrutado | `10.0.1.14/30`* |

\* Las direcciones `.13` y `.14` se proponen a partir de la red `10.0.1.12/30` mostrada en la topología. Se deben ajustar si en Packet Tracer se utilizaron otras direcciones.

> La máscara de la VLAN 10 debe ser `/24` (`255.255.255.0`). Una máscara `/30` no funciona para `192.168.50.3`, porque en la red `192.168.50.0/30` dicha dirección es broadcast.

## 3. Configuración limpia

### 3.1 Switch3 (2960)

Primero se crea la VLAN 10. Los enlaces hacia Router0 y Router1 deben funcionar como troncales, porque transportan las tramas etiquetadas utilizadas por las subinterfaces.

```cisco
enable
configure terminal

vlan 10
 name VLAN_USUARIOS
exit

interface gigabitEthernet 0/1
 description TRUNK_A_ROUTER1
 switchport mode trunk
 switchport trunk allowed vlan 10
 no shutdown
exit

interface gigabitEthernet 0/2
 description TRUNK_A_ROUTER0
 switchport mode trunk
 switchport trunk allowed vlan 10
 no shutdown
exit

end
write memory
```

Si el servidor DHCP está conectado, por ejemplo, en `FastEthernet0/1`, su puerto se configura como acceso. Se debe sustituir el número de interfaz si se utilizó otro puerto:

```cisco
configure terminal
interface fastEthernet 0/1
 description SERVIDOR_DHCP_VLAN10
 switchport mode access
 switchport access vlan 10
 spanning-tree portfast
 no shutdown
end
write memory
```

`switchport trunk allowed vlan all` es válido, pero permite todas las VLAN. Para este ejercicio es más claro limitar el enlace a la VLAN 10.

### 3.2 Router0 — router HSRP principal

```cisco
enable
configure terminal
hostname Router0

interface gigabitEthernet 0/0
 description ENLACE_A_MULTILAYER0
 ip address 10.0.1.21 255.255.255.252
 no shutdown
exit

interface gigabitEthernet 0/1
 description TRONCAL_A_SWITCH3
 no ip address
 no shutdown
exit

interface gigabitEthernet 0/1.10
 description GATEWAY_VLAN10
 encapsulation dot1Q 10
 ip address 192.168.50.3 255.255.255.0
 standby 50 ip 192.168.50.1
 standby 50 priority 110
 standby 50 preempt
exit

router eigrp 1
 no auto-summary
 network 10.0.1.20 0.0.0.3
 network 192.168.50.0 0.0.0.255
 passive-interface gigabitEthernet 0/1.10
exit

end
write memory
```

### 3.3 Router1 — router HSRP de respaldo

La interfaz hacia el switch multicapa se presenta con `G0/0` y la IP propuesta `10.0.1.13/30`. Si el cable está conectado a otra interfaz, se debe sustituir ese nombre.

```cisco
enable
configure terminal
hostname Router1

interface gigabitEthernet 0/0
 description ENLACE_A_MULTILAYER0
 ip address 10.0.1.13 255.255.255.252
 no shutdown
exit

interface gigabitEthernet 0/1
 description TRONCAL_A_SWITCH3
 no ip address
 no shutdown
exit

interface gigabitEthernet 0/1.10
 description GATEWAY_VLAN10
 encapsulation dot1Q 10
 ip address 192.168.50.2 255.255.255.0
 standby 50 ip 192.168.50.1
 standby 50 priority 100
 standby 50 preempt
exit

router eigrp 1
 no auto-summary
 network 10.0.1.12 0.0.0.3
 network 192.168.50.0 0.0.0.255
 passive-interface gigabitEthernet 0/1.10
exit

end
write memory
```

### 3.4 Multilayer0 (3560)

En un switch multicapa se debe habilitar `ip routing`. Los puertos conectados directamente a los routers se convierten en puertos de capa 3 mediante `no switchport`.

Los números de interfaz siguientes son un ejemplo y deben coincidir con el cableado real de Packet Tracer.

```cisco
enable
configure terminal
hostname Multilayer0
ip routing

interface gigabitEthernet 0/2
 description ENLACE_A_ROUTER0
 no switchport
 ip address 10.0.1.22 255.255.255.252
 no shutdown
exit

interface gigabitEthernet 0/1
 description ENLACE_A_ROUTER1
 no switchport
 ip address 10.0.1.14 255.255.255.252
 no shutdown
exit

router eigrp 1
 no auto-summary
 network 10.0.1.20 0.0.0.3
 network 10.0.1.12 0.0.0.3
exit

end
write memory
```

Si Multilayer0 tiene otros enlaces en las redes `10.0.1.0/30` y `10.0.1.4/30`, también deben configurarse sus puertos de capa 3 y anunciarse en EIGRP:

```cisco
configure terminal
router eigrp 1
 network 10.0.1.0 0.0.0.3
 network 10.0.1.4 0.0.0.3
end
write memory
```

## 4. Funcionamiento y prioridad de HSRP

Los dos routers pertenecen al **grupo HSRP 50** y anuncian la misma IP virtual `192.168.50.1`.

| Router | IP real | Prioridad | Estado esperado |
| --- | --- | ---: | --- |
| Router0 | `192.168.50.3` | **110** | Active |
| Router1 | `192.168.50.2` | **100** | Standby |

Router0 queda como **Active** porque HSRP elige al equipo que tenga la prioridad más alta. La prioridad predeterminada es 100; por eso se asigna 110 a Router0 y se conserva 100 en Router1.

El comando `preempt` permite que Router0 recupere el rol Active cuando vuelva a estar disponible. Sin `preempt`, Router1 podría continuar como Active aunque Router0 ya se haya recuperado.

Todos los clientes y el servidor de la VLAN 10 deben utilizar `192.168.50.1` como puerta de enlace predeterminada; no deben utilizar directamente `.2` ni `.3`.

## 5. Funcionamiento de EIGRP

EIGRP usa el sistema autónomo `1` en todos los dispositivos. El número debe coincidir para que se formen las vecindades.

- Router0 anuncia `10.0.1.20/30` y `192.168.50.0/24`.
- Router1 anuncia `10.0.1.12/30` y `192.168.50.0/24`.
- Multilayer0 anuncia los enlaces conectados hacia ambos routers.
- `passive-interface G0/1.10` evita formar una vecindad EIGRP entre Router0 y Router1 a través de la red de usuarios. La red continúa anunciándose, pero los mensajes Hello no se envían hacia la VLAN.


## 6. Comandos de verificación

### Switch3

```cisco
show vlan brief
show interfaces trunk
show interfaces gigabitEthernet 0/2 switchport
```

### Router0 y Router1

```cisco
show ip interface brief
show standby brief
show standby
show ip eigrp neighbors
show ip route
show ip route eigrp
```

### Multilayer0

```cisco
show ip interface brief
show ip eigrp neighbors
show ip route eigrp
show ip protocols
```

El resultado esperado de `show standby brief` es Router0 en estado **Active** y Router1 en estado **Standby**, ambos en el grupo 50 y con la IP virtual `192.168.50.1`.

## 7. Prueba de redundancia

1. Configurar en un host de la VLAN 10 una IP `192.168.50.x/24` y gateway `192.168.50.1`.
2. Ejecutar un ping continuo hacia una red remota conocida.
3. En Router0, apagar temporalmente la subinterfaz o la interfaz física:

```cisco
configure terminal
interface gigabitEthernet 0/1
shutdown
```

4. Verificar que Router1 cambie de Standby a Active mediante `show standby brief`.
5. Reactivar Router0:

```cisco
configure terminal
interface gigabitEthernet 0/1
no shutdown
```

6. Debido a su prioridad 110 y al comando `preempt`, Router0 debe recuperar el estado Active.

Durante la conmutación pueden perderse uno o varios paquetes; esto es normal mientras HSRP detecta el cambio de estado.

## 8. Cambiar el ip-helper-address en el switch

### Multilayer Switch 1
```bash
interface vlan 10
ip helper-address 192.168.50.4
```
