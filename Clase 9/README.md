# Documentación de Implementación BGP + OSPF

## 1. Descripción General

En este escenario se implementa un entorno de enrutamiento utilizando:

- **BGP (Border Gateway Protocol)** para comunicación entre sistemas autónomos (AS).
- **OSPF (Open Shortest Path First)** como protocolo interno (IGP).
- **Redistribución entre BGP y OSPF** para intercambio de rutas.

Se tienen tres dispositivos principales:

| Dispositivo | AS (BGP) | Proceso OSPF |
| ----------- | -------- | ------------ |
| MSW2        | 65001    | 1            |
| MS3         | 65002    | 3            |
| MS1         | 65003    | 2            |

## 2. Topología Lógica

Cada equipo pertenece a un sistema autónomo distinto y se interconectan mediante enlaces punto a punto /30:

- MSW2 ↔ MS3 → 10.0.10.0/30
- MS3 ↔ MS1 → 10.0.10.4/30
- MSW2 ↔ MS1 → 10.0.10.8/30

Esto permite redundancia y múltiples rutas entre los AS.

## 3. Configuración de MSW2 (AS 65001)

### 3.1 Interfaces

```bash
interface GigabitEthernet1/0/1
 no switchport
 ip address 10.0.10.1 255.255.255.252
 duplex auto
 speed auto
```

Comentarios:

- Interfaz en capa 3 utilizada para enlazar con MS3.
- La dirección 10.0.10.1/30 pertenece a la red 10.0.10.0/30.
- `duplex auto` y `speed auto` permiten negociación automática.
- Aquí también podría agregarse una descripción para identificar el enlace, por ejemplo:
  `description Enlace hacia MS3`

```bash
interface GigabitEthernet1/0/2
 no switchport
 ip address 10.0.10.9 255.255.255.252
 duplex auto
 speed auto
```

Comentarios:

- Enlace de capa 3 hacia MS1.
- La dirección 10.0.10.9/30 pertenece a la red 10.0.10.8/30.
- En una práctica más completa podría definirse `description Enlace hacia MS1`.

```bash
interface GigabitEthernet1/0/23
 no switchport
 ip address 10.0.10.14 255.255.255.252
 duplex auto
 speed auto
```

Comentarios:

- Esta interfaz representa la red interna/local que luego será anunciada.
- La red asociada es 10.0.10.12/30.
- Aquí podrían conectarse otros dispositivos o una red de salida específica.

### 3.2 Interfaz VLAN

```bash
interface Vlan1
 no ip address
 shutdown
```

Comentarios:

- La VLAN 1 no se utiliza para administración.
- Se mantiene apagada para evitar uso innecesario o tráfico no deseado.
- En escenarios reales se suele usar otra VLAN dedicada para gestión.

### 3.3 OSPF

```bash
router ospf 1
 log-adjacency-changes
 redistribute bgp 65001 subnets
 network 10.0.10.12 0.0.0.3 area 0
```

Comentarios:

- `router ospf 1` inicia el proceso OSPF 1.
- `log-adjacency-changes` registra cambios en vecindades OSPF.
- `redistribute bgp 65001 subnets` permite inyectar rutas BGP en OSPF.
- `network 10.0.10.12 0.0.0.3 area 0` anuncia la red interna en el área backbone.
- Aquí también podría configurarse una métrica para la redistribución si se quisiera mayor control.

### 3.4 BGP

```bash
router bgp 65001
 bgp log-neighbor-changes
 no synchronization
 neighbor 10.0.10.2 remote-as 65002
 neighbor 10.0.10.10 remote-as 65003
 network 10.0.10.12 mask 255.255.255.252
 network 10.0.10.0 mask 255.255.255.252
 network 10.0.10.8 mask 255.255.255.252
 redistribute ospf 1
```

Comentarios:

- `router bgp 65001` define a MSW2 dentro del AS 65001.
- `bgp log-neighbor-changes` registra cambios de estado de vecinos BGP.
- `no synchronization` desactiva la sincronización entre BGP e IGP, común en redes modernas.
- `neighbor 10.0.10.2 remote-as 65002` establece vecindad eBGP con MS3.
- `neighbor 10.0.10.10 remote-as 65003` establece vecindad eBGP con MS1.
- Las sentencias `network` anuncian redes específicas siempre que existan en la tabla de enrutamiento.
- `redistribute ospf 1` permite anunciar hacia BGP las rutas aprendidas por OSPF.
- En una implementación más avanzada podrían agregarse filtros, route-maps o atributos BGP.

### 3.5 Otras configuraciones

```bash
ip classless
ip flow-export version 9
line con 0
line aux 0
line vty 0 4
 login
```

Comentarios:

- `ip classless` permite manejar redes sin limitarse a clases tradicionales.
- `ip flow-export version 9` habilita exportación NetFlow versión 9.
- `line vty 0 4` con `login` habilita acceso remoto, aunque faltaría definir contraseña o método de autenticación para mayor seguridad.

## 4. Configuración de MS3 (AS 65002)

### 4.1 Interfaces

```bash
interface GigabitEthernet1/0/1
 no switchport
 ip address 10.0.10.2 255.255.255.252
 duplex auto
 speed auto
```

Comentarios:

- Enlace hacia MSW2.
- Pertenece a la red 10.0.10.0/30.

```bash
interface GigabitEthernet1/0/3
 no switchport
 ip address 10.0.10.5 255.255.255.252
 duplex auto
 speed auto
```

Comentarios:

- Enlace hacia MS1.
- Pertenece a la red 10.0.10.4/30.

```bash
interface GigabitEthernet1/0/23
 no switchport
 ip address 10.0.10.18 255.255.255.252
 duplex auto
 speed auto
```

Comentarios:

- Red interna/local de MS3.
- Corresponde a la subred 10.0.10.16/30.

### 4.2 Interfaz VLAN

```bash
interface Vlan1
 no ip address
 shutdown
```

Comentarios:

- La VLAN 1 queda deshabilitada para evitar uso no planificado.

### 4.3 OSPF

```bash
router ospf 2
 log-adjacency-changes
 redistribute bgp 65002 subnets
 network 10.0.10.16 0.0.0.3 area 0
```

Comentarios:

- Proceso OSPF 2.
- Se redistribuyen rutas BGP dentro de OSPF.
- Se anuncia únicamente la red interna 10.0.10.16/30.
- Aquí podría usarse `passive-interface` si se quisiera evitar adyacencias en ciertas interfaces.

### 4.4 BGP

```bash
router bgp 65002
 bgp log-neighbor-changes
 no synchronization
 neighbor 10.0.10.1 remote-as 65001
 neighbor 10.0.10.6 remote-as 65003
 network 10.0.10.0 mask 255.255.255.252
 network 10.0.10.4 mask 255.255.255.252
 network 10.0.10.16 mask 255.255.255.252
 redistribute ospf 2
```

Comentarios:

- MS3 pertenece al AS 65002.
- Tiene vecindad BGP con MSW2 y MS1.
- Se anuncian tanto enlaces de tránsito como la red interna.
- `redistribute ospf 2` inyecta rutas OSPF a BGP.
- En este punto podrían configurarse políticas para preferencia de rutas si fuera necesario.

### 4.5 Otras configuraciones

```bash
ip classless
ip flow-export version 9
```

Comentarios:

- Se mantiene comportamiento classless.
- Exportación de flujos para monitoreo.

## 5. Configuración de MS1 (AS 65003)

Nota:
En la configuración proporcionada para MS1 aparece el bloque repetido dos veces. Para fines de documentación se describe una sola vez, ya que ambas secciones contienen la misma configuración.

### 5.1 Interfaces

```bash
interface GigabitEthernet1/0/2
 no switchport
 ip address 10.0.10.10 255.255.255.252
 duplex auto
 speed auto
```

Comentarios:

- Enlace hacia MSW2.
- Pertenece a la red 10.0.10.8/30.

```bash
interface GigabitEthernet1/0/3
 no switchport
 ip address 10.0.10.6 255.255.255.252
 duplex auto
 speed auto
```

Comentarios:

- Enlace hacia MS3.
- Pertenece a la red 10.0.10.4/30.

```bash
interface GigabitEthernet1/0/23
 no switchport
 ip address 10.0.10.22 255.255.255.252
 duplex auto
 speed auto
```

Comentarios:

- Red interna/local.
- Pertenece a la subred 10.0.10.20/30.

### 5.2 Spanning Tree

```bash
spanning-tree mode pvst
```

Comentarios:

- Se habilita PVST.
- Aunque las interfaces usadas para enrutamiento están en capa 3, esta configuración sigue siendo válida para puertos switch que pudieran utilizarse en el equipo.
- Aquí también podrían definirse prioridades si el equipo participara activamente como root bridge en VLANs.

### 5.3 Interfaz VLAN

```bash
interface Vlan1
 no ip address
 shutdown
```

Comentarios:

- VLAN 1 apagada, evitando que quede como interfaz de administración por defecto.

### 5.4 OSPF

```bash
router ospf 3
 log-adjacency-changes
 redistribute bgp 65003 subnets
 network 10.0.10.20 0.0.0.3 area 0
```

Comentarios:

- Proceso OSPF 3.
- Se redistribuyen rutas BGP dentro de OSPF.
- Se anuncia la red interna 10.0.10.20/30 en el área 0.

### 5.5 BGP

```bash
router bgp 65003
 bgp log-neighbor-changes
 no synchronization
 neighbor 10.0.10.9 remote-as 65001
 neighbor 10.0.10.5 remote-as 65002
 network 10.0.10.20 mask 255.255.255.252
 network 10.0.10.8 mask 255.255.255.252
 network 10.0.10.4 mask 255.255.255.252
 redistribute ospf 3
```

Comentarios:

- MS1 pertenece al AS 65003.
- Tiene relaciones eBGP con MSW2 y MS3.
- Se anuncian redes de tránsito y red interna.
- También redistribuye rutas OSPF hacia BGP.
- En una red real podría ser útil agregar autenticación BGP o listas de prefijos.

### 5.6 Otras configuraciones

```bash
ip classless
ip flow-export version 9
line con 0
```

Comentarios:

- Se habilita operación classless.
- Exportación de flujos para monitoreo.
- La línea de consola está presente, aunque no muestra parámetros extra como contraseña o logging sync.

## 6. Relación entre BGP y OSPF

En los tres equipos se observa la misma lógica:

- OSPF anuncia la red interna local de cada equipo.
- BGP intercambia rutas entre los diferentes AS.
- Cada equipo redistribuye BGP dentro de OSPF.
- Cada equipo redistribuye OSPF dentro de BGP.

Esto permite que:

- Las rutas internas locales puedan propagarse a otros sistemas autónomos.
- Exista conectividad total entre redes internas y enlaces de tránsito.

Consideración técnica:

- Aunque funciona para laboratorio, en producción la redistribución mutua debe controlarse cuidadosamente para evitar loops o anuncios innecesarios.

## 7. Observaciones Importantes

### 7.1 Uso de enlaces /30

Los enlaces entre equipos utilizan máscara 255.255.255.252, lo cual es apropiado para enlaces punto a punto porque:

- Solo se necesitan dos direcciones útiles.
- Se optimiza el uso del direccionamiento IPv4.

### 7.2 Vecindades eBGP

Las relaciones BGP configuradas son externas, ya que cada vecino pertenece a un AS distinto:

- MSW2 con MS3
- MSW2 con MS1
- MS3 con MS1

### 7.3 Anuncio de redes

Las redes declaradas con `network` en BGP deben existir en la tabla de enrutamiento local para ser anunciadas correctamente.

### 7.4 Seguridad

No se observa configuración de:

- Contraseñas en vecinos BGP
- Acceso seguro por VTY
- ACLs de protección

Esto no está mal para práctica, pero sí sería recomendable agregarlo en una implementación real.

## 8. Comandos de Verificación Recomendados

Para validar el funcionamiento se pueden usar los siguientes comandos:

```bash
show ip bgp summary
show ip bgp
show ip route
show ip ospf neighbor
show ip ospf database
show running-config
```

Descripción breve:

- `show ip bgp summary`: muestra estado de vecinos BGP.
- `show ip bgp`: muestra tabla BGP.
- `show ip route`: verifica rutas instaladas.
- `show ip ospf neighbor`: muestra vecindades OSPF.
- `show ip ospf database`: permite revisar LSAs.
- `show running-config`: confirma configuración activa.
