# Manual técnico: ACL estándar y extendida

## Objetivo de la práctica

Las ACL de esta topología controlarán **únicamente ICMP utilizado por `ping`**. TCP, UDP, HTTP, DNS y los demás protocolos continuarán permitidos.

| VLAN | Subred | Wildcard |
| --- | --- | --- |
| VLAN 10 | `192.168.10.0/25` | `0.0.0.127` |
| VLAN 20 | `192.168.20.0/25` | `0.0.0.127` |
| VLAN 30 | `192.168.30.0/25` | `0.0.0.127` |
| VLAN 40 | `192.168.40.0/25` | `0.0.0.127` |

Política solicitada:

- VLAN 10 y VLAN 20 no pueden hacerse ping en ninguna dirección.
- VLAN 20 sí puede iniciar ping hacia VLAN 30 y recibir la respuesta.
- VLAN 30 no puede iniciar ping hacia VLAN 20.
- VLAN 40 puede hacerse ping con todas las VLAN.
- Todo tráfico que no sea el ICMP indicado continúa permitido.

## Resultados esperados

| Origen | Destino | Ping |
| --- | --- | --- |
| VLAN 10 | VLAN 20 | Bloqueado |
| VLAN 20 | VLAN 10 | Bloqueado |
| VLAN 20 | VLAN 30 | Permitido |
| VLAN 30 | VLAN 20 | Bloqueado al iniciar |
| VLAN 40 | VLAN 10, 20 y 30 | Permitido |
| VLAN 10, 20 y 30 | VLAN 40 | Permitido |

---

# ¿Qué es una ACL?

Una ACL (Access Control List) es una lista ordenada de reglas que permite o deniega paquetes. El equipo examina las reglas de arriba hacia abajo y aplica la primera coincidencia.

- `permit`: permite el paquete.
- `deny`: bloquea el paquete.
- Al final existe un `deny any` implícito.

Por ello, en esta configuración se coloca `permit ip any any` al final para permitir todo lo que no sea un ping expresamente bloqueado.

La wildcard de una red `/25` se calcula así:

```text
255.255.255.255 - 255.255.255.128 = 0.0.0.127
```

## ACL estándar

Una ACL estándar revisa únicamente la dirección IP de origen. Sus rangos son `1-99` y `1300-1999`.

```bash
access-list 10 deny 192.168.10.0 0.0.0.127
access-list 10 permit any

interface <interfaz>
 ip access-group 10 in
```

Normalmente se coloca cerca del destino. No es adecuada para esta práctica porque no puede identificar al mismo tiempo el origen, destino y protocolo ICMP.

## ACL extendida

Una ACL extendida revisa protocolo, origen y destino; también puede revisar puertos TCP o UDP. Sus rangos son `100-199` y `2000-2699`.

```bash
access-list 110 deny icmp 192.168.10.0 0.0.0.127 192.168.20.0 0.0.0.127 echo
access-list 110 permit ip any any
```

- `echo`: solicitud de ping.
- `echo-reply`: respuesta del ping.

Las ACL extendidas se colocan cerca del origen. Por eso se aplicarán en dirección `in` sobre cada interfaz VLAN o SVI.

---

# ¿Dónde se configura?

Las ACL se crean en los **switches multicapa que contienen las SVI utilizadas como gateway**.

Según la topología:

| Equipo | Interfaz | ACL |
| --- | --- | --- |
| Multilayer Switch1, lado izquierdo | `interface vlan 10` | `VLAN10_ICMP_IN` |
| Multilayer Switch1, lado izquierdo | `interface vlan 20` | `VLAN20_ICMP_IN` |
| Multilayer Switch0, lado derecho | `interface vlan 30` | `VLAN30_ICMP_IN` |
| Multilayer Switch0, lado derecho | `interface vlan 40` | Ninguna |

Para confirmar dónde está cada gateway, ejecutar:

```bash
show ip interface brief
```

Si las interfaces `Vlan10`, `Vlan20`, `Vlan30` o `Vlan40` aparecen configuradas en otro switch multicapa, la ACL correspondiente debe colocarse en ese equipo.

---

# Multilayer Switch1: VLAN 10 y VLAN 20

Entrar al switch multicapa del lado izquierdo y configurar:

```bash
enable
configure terminal

ip access-list extended VLAN10_ICMP_IN
 remark BLOQUEAR PING DE VLAN 10 HACIA VLAN 20
 deny icmp 192.168.10.0 0.0.0.127 192.168.20.0 0.0.0.127 echo
 permit ip any any
exit

interface vlan 10
 ip access-group VLAN10_ICMP_IN in
exit

ip access-list extended VLAN20_ICMP_IN
 remark BLOQUEAR PING DE VLAN 20 HACIA VLAN 10
 deny icmp 192.168.20.0 0.0.0.127 192.168.10.0 0.0.0.127 echo
 permit ip any any
exit

interface vlan 20
 ip access-group VLAN20_ICMP_IN in
exit

end
write memory
```

La ACL de VLAN 20 no bloquea el ping hacia VLAN 30; por eso ese tráfico llega a la regla `permit ip any any` y se permite.

---

# Multilayer Switch0: VLAN 30 y VLAN 40

Entrar al switch multicapa del lado derecho y configurar:

```bash
enable
configure terminal

ip access-list extended VLAN30_ICMP_IN
 remark PERMITIR RESPUESTA AL PING INICIADO DESDE VLAN 20
 permit icmp 192.168.30.0 0.0.0.127 192.168.20.0 0.0.0.127 echo-reply
 remark BLOQUEAR PING INICIADO DESDE VLAN 30 HACIA VLAN 20
 deny icmp 192.168.30.0 0.0.0.127 192.168.20.0 0.0.0.127 echo
 permit ip any any
exit

interface vlan 30
 ip access-group VLAN30_ICMP_IN in
exit

end
write memory
```

No se aplica una ACL restrictiva en `interface vlan 40`. De esta manera, VLAN 40 puede iniciar ping hacia todas las redes y las demás VLAN pueden responder.

## ¿Cómo funciona VLAN 20 hacia VLAN 30?

1. VLAN 20 envía una solicitud ICMP `echo` hacia VLAN 30.
2. `VLAN20_ICMP_IN` solo bloquea solicitudes hacia VLAN 10, por lo que permite el paquete.
3. VLAN 30 genera un `echo-reply` hacia VLAN 20.
4. La primera regla de `VLAN30_ICMP_IN` permite esa respuesta.
5. Si VLAN 30 intenta iniciar un nuevo `echo` hacia VLAN 20, la segunda regla lo bloquea.

---

# Verificación

En los switches multicapa ejecutar:

```bash
show access-lists
show ip access-lists
show ip interface vlan 10
show ip interface vlan 20
show ip interface vlan 30
show running-config | section access-list
```

Luego realizar los pings desde las computadoras:

```text
VLAN 10 -> VLAN 20: falla
VLAN 20 -> VLAN 10: falla
VLAN 20 -> VLAN 30: éxito
VLAN 30 -> VLAN 20: falla
VLAN 40 -> VLAN 10: éxito
VLAN 40 -> VLAN 20: éxito
VLAN 40 -> VLAN 30: éxito
```

El primer intento puede perder un paquete mientras se resuelve ARP. Se recomienda repetir el ping.

# Eliminar las ACL

Primero retirar la lista de la SVI y después eliminarla:

```bash
configure terminal
interface vlan 10
 no ip access-group VLAN10_ICMP_IN in
exit
no ip access-list extended VLAN10_ICMP_IN
end
```

El mismo procedimiento se utiliza para `VLAN20_ICMP_IN` y `VLAN30_ICMP_IN`.
