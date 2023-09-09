---
layout: single
title: DarkHole2 - VulnHub
date: 2023-09-04
classes: wide
header:
  tease: 
categories:
  - Vulnhub
  - Easy
  - Linux
tags:
  - Vulnhub
  - Sqli
  - Bash history
  - Sudoers abuse
  - GitHub information leakage
---

 ![](../assets/images/DarkHole2/DarkHole2.png)

# Fase de reconocimiento
---
Empezamos realizando un escaneo con `arp-scan` a la red para averiguar la IP de la máquina:
 
![](../assets/images/DarkHole2/arpScan.png)

Ahora realizamos un `ping` a la máquina para ver si nos responde.
Vemos que el *TTL* es *64* lo que nos indica que es una máquina **Linux**

![](../assets/images/DarkHole2/ping.png)

Empezamos ahora el reconocimiento de los puertos utilizando `nmap`, primero
escaneamos todo el rango de puertos.

![](../assets/images/DarkHole2/nmapAllPorts.png)

Ahora sobre estos puertos, hacemos un escaneo más profundo para detectar versión y servicio.
Vemos que `nmap` detecta que hay un repositorio git bajo la ruta `192.168.1.37:80/.git`

![](../assets/images/DarkHole2/nmapTargeted.png)

# Fase de explotación
---
## GitHub información expuesta
Investigando en el repositorio, encuentro los commits que se han ido haciendo.
Llama la atención el segundo commit, donde dice que ha puesto las credenciales por defecto en el `login.php`.

![](../assets/images/DarkHole2/gitLogs.png)

Nos traemos el repositorio a nuestra máquina con `wget -r <ruta>` para trabajar mejor y ahora dentro del repositorio hacemos `git show <hash_commit>` para ver los cambios que se han realizado en ese commit.
Con esto obtenemos el email `lush@admin.com` y la password `321` que nos sirven para hacer login en `login.php`.

![](../assets/images/DarkHole2/gitCredentials.png)
![](../assets/images/DarkHole2/loginPHP.png)

## SQLI

Después de hacer login, examindo esta nueva página, vemos que se produce un la SQL bajo el parámetro `id`.

![](../assets/images/DarkHole2/Sql1.png)

Una vez que hemos detectado que se produce un SQLI, dectamos el número de columnas que se están representando utilizando `order by` para después ir mostrando información de la base de datos. En este caso es *6*.

![](../assets/images/DarkHole2/Sql2.png)

El siguiente paso es mostrar las bases de datos que existen. En este caso utilizamos `group_concat()` para agrupar toda la salida en una sola columna y así poder verla en la web. (Hacer url encode en la query para que no haya problemas).

```sql
2'union select 1,group_concat(schema_name),3,4,5,6 from information_schema.schemata-- -
```

Después obtenemos las tablas que hay en la base de datos `darkhole_2`.

```sql
2'union select 1,group_concat(table_name),3,4,5,6 from information_schema.tables-- -
```

Ahora obtenemos las columnas de la tabla `ssh`

```sql
2'union select 1,group_concat(column_name),3,4,5,6 from information_schema.columns where table_schema='darkhole_2' and table_name='ssh'-- - 

```

Finalmente mostramos la información de las columnas para la tabla `ssh`

```sql
2'union select 1,group_concat(id,user,":",pass),3,4,5,6 from ssh -- -
```

Llegados a este punto obtenemos el usuario `jehad` y la contraseña `fool` que podremos utilizar para conectarnos por `ssh` a la máquina.

# Escalada de privilegios
---

Una vez concetados a la máquina como el usuario `fool`, vamos a hacer un reconociemintoutilizando la herramienta [Linpeas](https://github.com/carlospolop/PEASS-ng/tree/master/linPEAS).
Vemos que bajo la ruta `/opt/web` el usuario `losy` esta proporcionando un servidor web por el puerto `9000`. Además inspeccionando el archivo `index.php` vemos que se pueden ejecutar comandos a través del parámetro `cmd` en la url.

![](../assets/images/DarkHole2/indexphp.png)

Como tenemos ejecución de comandos bajo el usuario `losy`, entablamos una **Reverse shell**.

![](../assets/images/DarkHole2/RStoLosy.png)

## Escalada a Root

Observando el `.bash_history` de `losy` encontramos una posible credencial.

![](../assets/images/DarkHole2/history.png)

Ahora con la credencial podemos mirar si podemos ejecutar algún binario con `sudo`, para ello, ejecutamos `sudo -l`. Vemos que podemos ejecutar `python3` como si fuesemos `root`

![](../assets/images/DarkHole2/sudol.png)

Ahora convertirse en `root` es tarea sencilla, simplemente ejecutamos `sudo /usr/bin/python3 -c 'import os; os.system("/bin/sh")'` y ya estaríamos como `root`.

![](../assets/images/DarkHole2/root.png)





