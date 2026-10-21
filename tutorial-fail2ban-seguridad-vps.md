# 🛡️ Guía Completa de Fail2ban: Protección contra Ataques de Fuerza Bruta en VPS

**Fail2ban** es una herramienta de seguridad fundamental para cualquier servidor VPS Linux. Su función principal es inspeccionar continuamente los archivos de registro del sistema (`/var/log/auth.log`, logs de Nginx, MySQL, etc.) en busca de actividad maliciosa —como intentos fallidos repetidos de inicio de sesión por SSH o escaneo de exploits web— y bloquear automáticamente la dirección IP del atacante en el firewall (`iptables` o `ufw`) durante un tiempo determinado.

---

## 📑 Tabla de Contenidos
1. [¿Cómo Funciona Fail2ban?](#1-cómo-funciona-fail2ban)
2. [Instalación en Ubuntu / Debian](#2-instalación-en-ubuntu--debian)
3. [Estructura y Regla de Oro de Configuración (`jail.local`)](#3-estructura-y-regla-de-oro-de-configuración-jaillocal)
4. [Configuración Global y Parámetros Clave](#4-configuración-global-y-parámetros-clave)
5. [Protección de Servicios Críticos](#5-protección-de-servicios-críticos)
   - [5.1 Proteger SSH (Acceso al Servidor)](#51-proteger-ssh-acceso-al-servidor)
   - [5.2 Proteger Nginx (Ataques HTTP y Bad Bots)](#52-proteger-nginx-ataques-http-y-bad-bots)
   - [5.3 Proteger MySQL / MariaDB](#53-proteger-mysql--mariadb)
6. [Gestión con `fail2ban-client` (Comandos Diarios)](#6-gestión-con-fail2ban-client-comandos-diarios)
   - [Ver cárceles y estadísticas](#ver-cárceles-y-estadísticas)
   - [Ver IPs bloqueadas actualmente](#ver-ips-bloqueadas-actualmente)
   - [Desbanear una IP manualmente (Unban)](#desbanear-una-ip-manualmente-unban)
   - [Banear una IP de forma manual](#banear-una-ip-de-forma-manual)
7. [Cómo Evitar Quedarte Bloqueado Tú Mismo (`ignoreip`)](#7-cómo-evitar-quedarte-bloqueado-tú-mismo-ignoreip)
8. [Resolución de Problemas y Diagnóstico de Logs](#8-resolución-de-problemas-y-diagnóstico-de-logs)
9. [Chuleta / Resumen Rápido de Comandos](#9-chuleta--resumen-rápido-de-comandos)

---

## 1. ¿Cómo Funciona Fail2ban?

Fail2ban opera mediante tres conceptos centrales:
1. **Filtro (`Filter`):** Expresiones regulares que buscan patrones de intentos fallidos en los archivos `.log`.
2. **Acción (`Action`):** Qué hacer cuando se detecta un atacante (por ejemplo, insertar una regla de rechazo en `iptables`/`ufw` y opcionalmente enviar un correo de alerta).
3. **Cárcel (`Jail`):** La combinación de un Filtro + una Acción para un servicio específico (ejemplo: la cárcel `sshd`).

```text
[ Atacante intenta passwords por SSH ]
                 │
                 ▼
     [/var/log/auth.log] (Guarda intento fallido)
                 │
                 ▼
           [ Fail2ban ] (Detecta 5 fallos en 10 minutos)
                 │
                 ▼
       [ Regla Firewall UFW/iptables ] (Bloquea la IP por 1 hora)
```

---

## 2. Instalación en Ubuntu / Debian

Asegúrate de tener actualizados los repositorios e instala Fail2ban:

```bash
sudo apt update
sudo apt install fail2ban -y
```

### Habilitar y arrancar el servicio:
```bash
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
```

### Comprobar su estado:
```bash
sudo systemctl status fail2ban
```

---

## 3. Estructura y Regla de Oro de Configuración (`jail.local`)

Fail2ban incluye un archivo de configuración por defecto llamado `/etc/fail2ban/jail.conf`.

> [!CAUTION]
> **NUNCA edites directamente `/etc/fail2ban/jail.conf`**. Las actualizaciones del paquete sobrescribirán ese archivo y perderás todos tus cambios.
>
> La forma correcta es crear una copia de trabajo llamada **`jail.local`**, la cual tiene precedencia y preserva tus ajustes:

```bash
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
```

---

## 4. Configuración Global y Parámetros Clave

Abre tu archivo `jail.local`:

```bash
sudo nano /etc/fail2ban/jail.local
```

Busca la sección `[DEFAULT]`. Aquí se definen los parámetros base que aplican a todas las cárceles:

```ini
[DEFAULT]
# IPs o subredes que NUNCA serán bloqueadas (separadas por espacio)
ignoreip = 127.0.0.1/8 ::1 192.168.1.0/24 TU_IP_PUBLICA_LOCAL

# Tiempo que la IP permanecerá bloqueada (ej: 1h, 1d, 3600 = 1 hora)
bantime  = 1h

# Ventana de tiempo en la que se cuentan los intentos fallidos
findtime  = 10m

# Cantidad de fallos permitidos dentro de findtime antes de banear
maxretry = 5

# Backend para leer los logs (systemd en versiones modernas de Debian/Ubuntu)
backend = auto

# Acción de bloqueo (banaction = ufw o iptables-multiport)
banaction = ufw
```

#### 🔍 Explicación práctica de la regla:
*"Si una misma IP falla **5 veces (`maxretry`)** dentro de un lapso de **10 minutos (`findtime`)**, será expulsada y bloqueada en el firewall durante **1 hora (`bantime`)**"*.

---

## 5. Protección de Servicios Críticos

### 5.1 Proteger SSH (Acceso al Servidor)

Dentro de `/etc/fail2ban/jail.local`, localiza o añade el bloque `[sshd]`:

```ini
[sshd]
enabled  = true
port     = ssh
# Si cambiaste el puerto SSH estándar (ej: 2222), especifícalo aquí:
# port   = 2222
filter   = sshd
logpath  = /var/log/auth.log
maxretry = 3
bantime  = 24h
```

*(En este ejemplo, quien falle 3 veces en SSH queda vetado por 24 horas).*

---

### 5.2 Proteger Nginx (Ataques HTTP y Bad Bots)

Fail2ban puede mitigar escaneos de vulnerabilidades web (como peticiones automáticas buscando `wp-login.php`, `phpmyadmin`, etc.) o abusos de autenticación:

#### Proteger accesos fallidos HTTP (`nginx-http-auth`):
```ini
[nginx-http-auth]
enabled  = true
port     = http,https
filter   = nginx-http-auth
logpath  = /var/log/nginx/error.log
maxretry = 3
bantime  = 1h
```

#### Proteger contra escaneos maliciosos 404 repetitivos (`nginx-botsearch`):
```ini
[nginx-botsearch]
enabled  = true
port     = http,https
filter   = nginx-botsearch
logpath  = /var/log/nginx/access.log
maxretry = 5
findtime = 5m
bantime  = 12h
```

---

### 5.3 Proteger MySQL / MariaDB

Si tu base de datos tiene expuesto el puerto 3306 a conexiones remotas:

```ini
[mysqld-auth]
enabled  = true
port     = 3306
filter   = mysqld-auth
logpath  = /var/log/mysql/error.log
maxretry = 4
bantime  = 6h
```

---

### Aplicar los cambios:
Cada vez que modifiques `/etc/fail2ban/jail.local`, recarga el servicio:

```bash
sudo systemctl restart fail2ban
```

---

## 6. Gestión con `fail2ban-client` (Comandos Diarios)

La herramienta `fail2ban-client` es la interfaz de línea de comandos para consultar y controlar el demonio en vivo sin necesidad de reiniciar el servicio.

### Ver cárceles y estadísticas
```bash
sudo fail2ban-client status
```
*Salida típica:*
```text
Status
|- Number of jail:      2
`- Jail list:   nginx-http-auth, sshd
```

---

### Ver IPs bloqueadas actualmente en una cárcel
```bash
sudo fail2ban-client status sshd
```
*Salida:*
```text
Status for the jail: sshd
|- Filter
|  |- Currently failed: 1
|  |- Total failed:     42
`- Actions
   |- Currently banned: 2
   |- Total banned:     15
   `- Banned IP list:   198.51.100.25 203.0.113.84
```

---

### Desbanear una IP manualmente (Unban)
Si tú mismo o un compañero introdujo la clave mal y quedó bloqueado:

```bash
# Desbanear de una cárcel específica:
sudo fail2ban-client set sshd unbanip 198.51.100.25

# Desbanear de todas las cárceles del servidor:
sudo fail2ban-client unban 198.51.100.25
```

---

### Banear una IP de forma manual
Si detectas una IP hostil en los logs y deseas bloquearla de inmediato:

```bash
sudo fail2ban-client set sshd banip 203.0.113.99
```

---

## 7. Cómo Evitar Quedarte Bloqueado Tú Mismo (`ignoreip`)

El mayor temor al configurar Fail2ban es bloquearse a uno mismo tras equivocarse al escribir la clave SSH.

### Paso 1: Conoce tu IP pública actual
En tu computadora de trabajo, entra a [ifconfig.me](https://ifconfig.me) o corre en consola:
```bash
curl ifconfig.me
```

### Paso 2: Añádela a la lista blanca
En `/etc/fail2ban/jail.local`:
```ini
[DEFAULT]
ignoreip = 127.0.0.1/8 ::1 181.45.22.100
```
*(Donde `181.45.22.100` es tu IP).*

Recarga con:
```bash
sudo fail2ban-client reload
```

---

## 8. Resolución de Problemas y Diagnóstico de Logs

### Ver el registro de actividad de Fail2ban:
```bash
sudo tail -f /var/log/fail2ban.log
```
Aquí verás exactamente cuándo se activa un baneo (`[sshd] Ban 198.51.100.25`) y cuándo expira (`[sshd] Unban 198.51.100.25`).

### Verificar reglas activas en el Firewall:
Si usas `ufw`:
```bash
sudo ufw status verbose
```
Si usas `iptables`:
```bash
sudo iptables -L -n -v
```

---

## 9. Chuleta / Resumen Rápido de Comandos

| Comando | Acción |
| :--- | :--- |
| `sudo systemctl status fail2ban` | Comprueba si el servicio está corriendo. |
| `sudo fail2ban-client status` | Lista las cárceles (jails) activas. |
| `sudo fail2ban-client status <cárcel>` | Muestra las IPs bloqueadas en esa cárcel específica. |
| `sudo fail2ban-client set <cárcel> unbanip <IP>` | Desbloquea una IP en esa cárcel. |
| `sudo fail2ban-client unban <IP>` | Desbloquea una IP en todas las cárceles activas. |
| `sudo fail2ban-client set <cárcel> banip <IP>` | Bloquea manualmente una IP. |
| `sudo fail2ban-client reload` | Aplica cambios de configuración sin tirar el servicio. |
| `sudo tail -n 50 -f /var/log/fail2ban.log` | Monitorea en vivo los baneos y advertencias. |
