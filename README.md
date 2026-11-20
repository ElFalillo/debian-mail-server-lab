# Laboratorio de correo local en Debian 13

Laboratorio de correo local en Debian 13 con Postfix, Dovecot, Thunderbird,
ClamAV y Rspamd. Incluye análisis antivirus de los correos mediante Rspamd y
pruebas de detección con EICAR.

## 1. Objetivos

- Configurar Postfix para recibir y entregar correo local.
- Configurar Dovecot para consultar los buzones mediante IMAP.
- Compartir el mismo almacenamiento Maildir entre ambos servicios.
- Enviar mensajes de prueba desde la terminal con mailutils.
- Enviar y recibir mensajes desde Thunderbird.
- Integrar Rspamd con Postfix como filtro de correo.
- Integrar ClamAV con Rspamd para analizar adjuntos.
- Verificar el funcionamiento de los servicios y consultar sus registros.

## 2. Entorno del laboratorio

| Elemento | Configuración |
|---|---|
| Sistema operativo | Debian 13 |
| Servidor SMTP | Postfix |
| Servidor IMAP | Dovecot 2.4 |
| Filtro de correo | Rspamd |
| Antivirus | ClamAV |
| Herramientas de terminal | mailutils, rspamc |
| Cliente gráfico | Thunderbird |
| Dominio de pruebas | raulprueba.test |
| Nombre del servidor de correo | mail.raulprueba.test |
| Usuario local de pruebas | adm-n1 |
| Dirección de correo | [adm-n1@raulprueba.test](mailto:adm-n1@raulprueba.test) |
| Formato de almacenamiento | Maildir |
| Acceso a los servicios de correo | Solo localhost |

> Este documento utiliza el usuario `adm-n1`, que ya existe en la máquina.
> Si utilizas otro usuario, sustituye su nombre en los comandos y en Thunderbird.

## 3. Arquitectura

Todos los componentes se ejecutan dentro de la misma máquina virtual:

```text
Máquina virtual Debian
│
├── Thunderbird
│   ├── SMTP → 127.0.0.1:25
│   └── IMAP → 127.0.0.1:143
│
├── Postfix
│   ├── Milter → 127.0.0.1:11332
│   └── Entrega los mensajes en /home/adm-n1/Maildir/
│
├── Rspamd
│   ├── Analiza el correo y calcula una puntuación de spam
│   └── Consulta a ClamAV para analizar los adjuntos
│
├── ClamAV daemon
│   └── Detecta malware en los adjuntos
│
└── Dovecot
    └── Permite consultar ese mismo buzón mediante IMAP
```

Recorrido de un mensaje enviado a la propia cuenta:

```text
Thunderbird → Postfix → Rspamd → Maildir → Dovecot → Thunderbird
```

Si Rspamd detecta malware, el mensaje se rechaza antes de llegar al Maildir:

```text
Thunderbird → Postfix → Rspamd → ClamAV → rechazo SMTP
```

### Administración por SSH y cliente gráfico

La instalación y configuración del servidor se pueden realizar por SSH.

Sin embargo, Thunderbird debe abrirse desde el escritorio de la máquina virtual
para utilizar directamente `127.0.0.1`.

Abrir Thunderbird en el equipo anfitrión no equivale a abrirlo en la VM, aunque
haya una terminal SSH conectada a ella.

## 4. Instalación de paquetes

En Debian:

```bash
sudo apt update
sudo apt install postfix dovecot-core dovecot-imapd mailutils thunderbird
```

Durante el asistente de Postfix:

| Pregunta | Respuesta |
|---|---|
| Tipo de configuración | Sitio de Internet |
| Nombre del sistema de correo | raulprueba.test |

Después se limitará la escucha de Postfix a localhost.

Comprueba Dovecot con la ruta completa:

```bash
sudo /usr/sbin/dovecot --version
```

> Si `dovecot --version` devuelve «orden no encontrada», el ejecutable puede
> estar instalado pero fuera del PATH del usuario.
> Utiliza `/usr/sbin/dovecot` antes de asumir que falta el paquete.

## 5. Configuración de Postfix

### 5.1. Copia de seguridad

```bash
sudo cp /etc/postfix/main.cf /etc/postfix/main.cf.bak
```

### 5.2. Configuración principal

Edita:

```bash
sudo nano /etc/postfix/main.cf
```

A continuación se muestra la configuración utilizada en el laboratorio:

```ini
# Compatibilidad: valor utilizado en esta instalacion
compatibility_level = 3.9

# Identidad del servidor
myhostname = mail.raulprueba.test
mydomain = raulprueba.test
myorigin = /etc/mailname

# Saludo SMTP
smtpd_banner = $myhostname ESMTP $mail_name (Debian)

# Escucha local
inet_interfaces = loopback-only
inet_protocols = all

# Clientes de confianza
mynetworks = 127.0.0.0/8 [::1]/128

# Destinos de entrega local
mydestination = $myhostname, $mydomain, localhost.$mydomain, localhost, localhost.localdomain, vm-N1, vm-N1.internal

# Restricciones de reenvio
smtpd_relay_restrictions = permit_mynetworks, permit_sasl_authenticated, defer_unauth_destination
relay_domains =
relayhost =

# Almacenamiento local
home_mailbox = Maildir/
mailbox_command =
mailbox_size_limit = 0

# Alias
alias_maps = hash:/etc/aliases
alias_database = hash:/etc/aliases

# Otros ajustes
biff = no
recipient_delimiter = +

# SASL
cyrus_sasl_config_path = /etc/postfix/sasl

# TLS del servidor
smtpd_tls_key_file = /etc/ssl/private/ssl-cert-snakeoil.key
smtpd_tls_cert_file = /etc/ssl/certs/ssl-cert-snakeoil.pem
smtpd_tls_security_level = may

# TLS del cliente
smtp_tls_CAfile = /etc/ssl/certs/ca-certificates.crt
smtp_tls_security_level = may
smtp_tls_session_cache_database = btree:${data_directory}/smtp_scache
```

> `compatibility_level` corresponde al valor de la instalación utilizada.
> Si partes de otra versión de Postfix, conserva inicialmente el valor generado
> por su paquete y revisa las diferencias antes de sustituir todo el archivo.
>
> Los nombres `vm-N1` y `vm-N1.internal` son específicos de esta VM:
> adáptalos al nombre de tu máquina o elimínalos si no los necesitas.

### 5.3. Dominio de origen

```bash
echo 'raulprueba.test' | sudo tee /etc/mailname
```

### 5.4. Validación y reinicio

```bash
sudo newaliases
sudo postfix check
sudo systemctl enable postfix
sudo systemctl restart postfix
```

Si la validación muestra errores, corrígelos antes de continuar.

Comprueba los parámetros efectivos:

```bash
sudo postconf myhostname mydomain myorigin mydestination home_mailbox inet_interfaces
```

Los valores relevantes deben incluir:

```ini
myhostname = mail.raulprueba.test
mydomain = raulprueba.test
myorigin = /etc/mailname
home_mailbox = Maildir/
inet_interfaces = loopback-only
```

## 6. Configuración de Dovecot 2.4

> Los ajustes siguientes son para Dovecot 2.4.
> No mezclar directamente con ejemplos de Dovecot 2.3.

### 6.1. Archivo de ajustes locales

Si el archivo ya existe, guarda primero una copia.

Abre:

```bash
sudo nano /etc/dovecot/conf.d/99-local.conf
```

Añade:

```ini
# Solo conexiones desde la propia VM
listen = 127.0.0.1

# Mismo almacenamiento que Postfix
mail_driver = maildir
mail_path = ~/Maildir
mail_inbox_path = ~/Maildir
```

El archivo principal debe incluir los archivos de `conf.d`, como ocurre en la
configuración utilizada para esta práctica.

### 6.2. Importancia de mail_inbox_path

Durante la práctica, la configuración efectiva mostraba:

```ini
mail_inbox_path = /var/mail/%{user}
```

Se sustituyó por:

```ini
mail_inbox_path = ~/Maildir
```

De esta forma, la bandeja de entrada coincide con el buzón donde entrega Postfix.

### 6.3. Autenticación

Se conserva la configuración de usuarios del sistema que trae la instalación,
verificada mediante:

```bash
sudo /usr/sbin/doveconf -n
```

En este laboratorio aparecen:

```ini
passdb pam {
}

userdb passwd {
}
```

La cuenta utiliza el nombre y la contraseña del usuario de Debian.

No añadas estos bloques de nuevo si ya existen.

### 6.4. Validación y reinicio

```bash
sudo /usr/sbin/doveconf -n
```

Si no aparecen errores:

```bash
sudo systemctl enable --now dovecot
sudo systemctl restart dovecot
sudo systemctl status dovecot --no-pager
```

## 7. Preparación del buzón

Desde una sesión del usuario `adm-n1`, sin sudo:

```bash
mkdir -p "$HOME/Maildir"/{cur,new,tmp}
chmod 700 "$HOME/Maildir" "$HOME/Maildir"/{cur,new,tmp}
```

Estructura:

```text
/home/adm-n1/Maildir/
├── cur/
├── new/
└── tmp/
```

> No ejecutes estos comandos desde una sesión de root:
> `$HOME` debe corresponder al usuario del buzón.

## 8. Comprobación de servicios y puertos

```bash
sudo systemctl status postfix dovecot --no-pager
sudo ss -ltnp | grep -E ':(25|143)\b'
```

Escuchas esperadas:

```text
127.0.0.1:25     SMTP
127.0.0.1:143    IMAP
```

Postfix también puede aparecer en:

```text
[::1]:25
```

No deberían aparecer escuchas de estos servicios en todas las interfaces,
como `0.0.0.0:25` o `0.0.0.0:143`.

## 9. Prueba de envío desde la terminal

```bash
echo "Mensaje enviado desde la terminal de Debian." \
  | mail -s "Prueba con mailutils" adm-n1@raulprueba.test
```

Comprueba si se ha entregado:

```bash
ls -l "$HOME/Maildir/new"
```

Si Thunderbird ya está conectado, también puede haber mensajes en `cur`:

```bash
ls -l "$HOME/Maildir/cur"
```

Para consultar el correo del laboratorio se utiliza Thunderbird sobre IMAP.
No se presupone que el comando `mail` lea automáticamente el Maildir configurado.

## 10. Prueba de autenticación

```bash
sudo /usr/bin/doveadm auth test adm-n1
```

Introduce la contraseña cuando se solicite.

Resultado esperado:

```text
passdb: adm-n1 auth succeeded
```

Esta prueba verifica la autenticación de Dovecot, pero no sustituye la prueba
completa de acceso IMAP y lectura del buzón.

> No escribas la contraseña como argumento del comando ni la publiques.

## 11. Prueba del protocolo IMAP

Instala netcat si es necesario:

```bash
sudo apt install netcat-openbsd
```

Conecta:

```bash
nc 127.0.0.1 143
```

Respuesta esperada, abreviada:

```text
* OK [...] Dovecot ready.
```

Escribe:

```text
a1 CAPABILITY
```

El servidor debe responder con sus capacidades y finalizar con una respuesta
etiquetada `a1 OK`.

Cierra la conexión:

```text
a2 LOGOUT
```

Resultado esperado:

```text
* BYE Logging out
a2 OK Logout completed.
```

> Los comandos IMAP necesitan una etiqueta, como `a1` o `a2`.
> Una entrada vacía o un comando no válido puede producir una respuesta `BAD`.

## 12. Configuración de Thunderbird

### 12.1. Abrir el cliente en la VM

Accede al escritorio gráfico de Debian desde la consola de la máquina virtual.

Abre Thunderbird desde su menú de aplicaciones o desde una terminal de ese
escritorio:

```bash
thunderbird
```

No lo ejecutes con sudo.

Si la VM solo tiene consola de texto, será necesario disponer de una sesión
gráfica para seguir esta parte del laboratorio.

### 12.2. Añadir la cuenta

Selecciona la opción para añadir una cuenta de correo existente.
Los nombres exactos de los menús pueden variar según la versión.

| Campo | Valor |
|---|---|
| Full name / Nombre | Raúl |
| Email address / Correo | [adm-n1@raulprueba.test](mailto:adm-n1@raulprueba.test) |
| Password / Contraseña | Contraseña del usuario Debian adm-n1 |

Utiliza la configuración manual.

### 12.3. Servidor entrante

| Campo | Valor |
|---|---|
| Protocolo | IMAP |
| Hostname / Servidor | 127.0.0.1 |
| Puerto | 143 |
| Seguridad de conexión | Ninguna |
| Autenticación | Contraseña normal |
| Usuario | adm-n1 |

### 12.4. Servidor saliente

| Campo | Valor |
|---|---|
| Protocolo | SMTP |
| Hostname / Servidor | 127.0.0.1 |
| Puerto | 25 |
| Seguridad de conexión | Ninguna |
| Autenticación | Sin autenticación |
| Usuario | Vacío |

> Estos ajustes sin TLS se utilizan exclusivamente con Thunderbird y los
> servicios dentro de la misma VM, conectados por loopback.
> No deben trasladarse sin cambios a una conexión entre equipos.

Postfix acepta el envío desde localhost porque esa dirección forma parte de
`mynetworks`.

En esta práctica no se ha configurado autenticación SMTP mediante Dovecot ni
un servicio de envío autenticado en el puerto 587.

## 13. Prueba completa desde Thunderbird

Redacta un mensaje:

```text
Para: adm-n1@raulprueba.test
Asunto: Prueba de correo local

Mensaje de prueba enviado desde Thunderbird.
```

Envía el mensaje y comprueba su recepción en la bandeja de entrada.

La prueba valida el siguiente recorrido:

```text
Cliente SMTP → Postfix → Rspamd → entrega en Maildir → Dovecot IMAP → cliente
```

### Resultado obtenido

Se consiguió enviar y recibir un mensaje en la cuenta local desde Thunderbird.

## 14. Diagnóstico de problemas

### 14.1. Thunderbird no encuentra el servidor

Comprobar, en este orden:

1. Thunderbird se ejecuta dentro de la VM.
2. El servidor configurado es `127.0.0.1`.
3. El puerto IMAP es `143`.
4. La seguridad de conexión es «Ninguna» para esta prueba local.
5. El usuario es `adm-n1`, no la dirección completa.
6. Dovecot está activo y escuchando.

```bash
sudo systemctl status dovecot --no-pager -l
sudo ss -ltnp | grep ':143 '
```

No asumir que el asistente de Thunderbird es la causa sin comprobar primero
la conexión.

### 14.2. Dovecot rechaza la contraseña

```bash
sudo /usr/bin/doveadm auth test adm-n1
```

Revisar que se utiliza la contraseña del usuario Debian y que no se está
intentando autenticar con una cuenta distinta.

### 14.3. El mensaje se envía, pero no aparece

Comprobar que ambos servicios usan el mismo buzón:

```bash
sudo postconf home_mailbox
sudo /usr/sbin/doveconf -n
```

Valores esperados:

```ini
# Postfix
home_mailbox = Maildir/

# Dovecot
mail_driver = maildir
mail_path = ~/Maildir
mail_inbox_path = ~/Maildir
```

Revisar también propietario y permisos:

```bash
ls -ld "$HOME/Maildir" "$HOME/Maildir"/{cur,new,tmp}
```

### 14.4. Consultar registros

Dovecot:

```bash
sudo journalctl -u dovecot -n 50 --no-pager
sudo journalctl -u dovecot -f
```

Postfix:

```bash
sudo journalctl --since "10 minutes ago" --no-pager \
  | grep -i postfix
```

Rspamd:

```bash
sudo journalctl -u rspamd -n 50 --no-pager
sudo journalctl -u rspamd -f
```

Si el sistema registra el correo en `/var/log/mail.log`:

```bash
sudo tail -f /var/log/mail.log
```

La disponibilidad de ese archivo depende de la configuración de registro.

### 14.5. Consultar la cola de Postfix

```bash
mailq
```

Solicitar un nuevo intento de entrega de los mensajes en cola:

```bash
sudo postqueue -f
```

## 15. Alcance y limitaciones de seguridad

- Los servicios de correo escuchan solo en loopback.
- No es necesario abrir puertos del router.
- El dominio `raulprueba.test` se utiliza para el laboratorio.
- No se ha configurado recepción de correo público mediante registros MX.
- No se ha configurado un proveedor de salida o smarthost.
- La prueba realizada no valida la entrega a Gmail, Outlook u otros proveedores.
- Un `relayhost` vacío no bloquea el correo saliente: Postfix puede intentar
  entrega directa a destinos externos.
- Los ajustes TLS conservados en los servicios no significan que la conexión
  de Thunderbird de esta práctica esté cifrada: se ha seleccionado «Ninguna».
- No se deben publicar contraseñas, claves privadas, buzones ni perfiles
  completos de Thunderbird en GitHub.

## 16. Evidencias recomendadas

Se pueden incluir capturas de:

- Estado activo de Dovecot.
- Puertos SMTP e IMAP limitados a localhost.
- Autenticación correcta mediante `doveadm`.
- Respuesta de Dovecot a `CAPABILITY`.
- Mensaje enviado y recibido en Thunderbird.
- Estado activo de Rspamd y ClamAV.
- Análisis de un mensaje con `rspamc`.
- Rechazo de un mensaje con EICAR.
- Registros de detección y rechazo.

Antes de publicarlas, revisar y ocultar información personal, cuentas ajenas
al laboratorio y cualquier dato sensible.

## 17. Añadir ClamAV

Se instaló ClamAV para disponer de un antivirus en el servidor de correo.
Inicialmente se configuró mediante `clamav-milter`, pero posteriormente se
integró directamente en Rspamd.

ClamAV sigue siendo el antivirus utilizado por Rspamd para analizar los
adjuntos de los mensajes.

### 17.1. Instalación

```bash
sudo apt update
sudo apt install clamav clamav-daemon clamav-freshclam clamav-milter
```

Guardar copias de seguridad antes de modificar la configuración:

```bash
sudo cp -a /etc/postfix/main.cf /etc/postfix/main.cf.pre-clamav
sudo cp -a /etc/clamav/clamav-milter.conf /etc/clamav/clamav-milter.conf.bak
```

### 17.2. Actualizar las firmas de virus

Si el actualizador automático está activo, puede bloquear el archivo de registro
y provocar este error al ejecutar `freshclam` manualmente:

```text
ERROR: Failed to lock the log file /var/log/clamav/freshclam.log:
Resource temporarily unavailable
```

Para actualizar manualmente:

```bash
sudo systemctl stop clamav-freshclam
sudo freshclam
sudo systemctl enable --now clamav-freshclam
```

Comprobar que las firmas se han actualizado antes de continuar.

### 17.3. Activar el demonio de análisis

```bash
sudo systemctl enable --now clamav-daemon
sudo systemctl status clamav-daemon --no-pager -l
```

Comprobar el socket que utiliza el demonio:

```bash
sudo grep -E '^(LocalSocket|TCPSocket|TCPAddr)' /etc/clamav/clamd.conf
```

En esta práctica se utiliza el socket Unix:

```text
LocalSocket /var/run/clamav/clamd.ctl
```

Rspamd puede comunicarse con ClamAV mediante ese socket Unix, evitando abrir
un puerto TCP adicional.

## 18. Añadir Rspamd al servidor de correo

Rspamd es un filtro de correo que analiza los mensajes antes de que Postfix los
entregue. Calcula una puntuación de spam, comprueba reglas antispam y puede
aplicar acciones como añadir cabeceras, retrasar la entrega o rechazar mensajes.

Rspamd se conecta a Postfix mediante milter y utiliza ClamAV para analizar los
adjuntos.

### 18.1. Nueva arquitectura

```text
Thunderbird → Postfix → Rspamd → Maildir → Dovecot
                             │
                             └── ClamAV para analizar adjuntos
```

Si se detecta malware:

```text
Thunderbird → Postfix → Rspamd → ClamAV → rechazo SMTP
```

### 18.2. Instalación

```bash
sudo apt update
sudo apt install --no-install-recommends rspamd
sudo apt install redis-server
```

Activar los servicios:

```bash
sudo systemctl enable --now rspamd redis-server
sudo systemctl status rspamd redis-server --no-pager
```

### 18.3. Configurar Redis

Crear el archivo:

```bash
sudo nano /etc/rspamd/local.d/redis.conf
```

Añadir:

```lua
servers = "127.0.0.1";
```

Comprobar que Redis responde:

```bash
redis-cli -h 127.0.0.1 -p 6379 ping
```

Resultado esperado:

```text
PONG
```

### 18.4. Comprobar Rspamd

```bash
sudo rspamadm configtest
sudo systemctl restart rspamd
sudo systemctl status rspamd --no-pager -l
```

Comprobar los puertos:

```bash
sudo ss -ltnp | grep -E ':(11332|11333|11334)\b'
```

Rspamd debe escuchar localmente en:

```text
127.0.0.1:11332
127.0.0.1:11333
127.0.0.1:11334
```

### 18.5. Probar Rspamd con un mensaje limpio

Crear un mensaje de prueba:

```bash
cat > /tmp/test.eml <<EOF
From: Raúl <adm-n1@raulprueba.test>
To: adm-n1@raulprueba.test
Subject: Prueba Rspamd - mensaje limpio
Date: $(date -R)
Message-ID: <test-$(date +%s)@raulprueba.test>
MIME-Version: 1.0
Content-Type: text/plain; charset=utf-8

Mensaje de prueba para comprobar Rspamd.
EOF
```

Analizarlo:

```bash
rspamc /tmp/test.eml
```

Deben aparecer la acción, la puntuación y los símbolos analizados.

### 18.6. Probar Rspamd con GTUBE

GTUBE es una prueba estándar para comprobar el filtrado antispam. No es spam
real y no causa daño al sistema.

```bash
{ cat /tmp/test.eml; echo 'XJS*C4JDBQADN1.NSBN3*2IDNEN*GTUBE-STANDARD-ANTI-UBE-TEST-EMAIL*C.34X'; } > /tmp/gtube.eml
rspamc /tmp/gtube.eml
```

Resultado esperado:

```text
Action: reject
Symbol: GTUBE
```

### 18.7. Conectar Rspamd con Postfix

Antes de modificar Postfix, guardar una copia:

```bash
sudo cp -a /etc/postfix/main.cf /etc/postfix/main.cf.pre-rspamd
```

Como anteriormente se utilizaba `clamav-milter`, se sustituye su milter por
Rspamd para evitar analizar cada mensaje dos veces:

```bash
sudo postconf -e 'smtpd_milters = inet:localhost:11332'
sudo postconf -e 'non_smtpd_milters = inet:localhost:11332'
sudo postconf -e 'milter_protocol = 6'
sudo postconf -e 'milter_default_action = accept'
```

- `smtpd_milters`: analiza el correo recibido mediante SMTP, como el de Thunderbird.
- `non_smtpd_milters`: analiza también el correo introducido mediante herramientas locales.
- `milter_default_action = accept`: si Rspamd no está disponible, Postfix acepta el mensaje sin analizarlo.

Validar y reiniciar Postfix:

```bash
sudo postfix check
sudo systemctl restart postfix
```

Comprobar la configuración efectiva:

```bash
sudo postconf smtpd_milters non_smtpd_milters milter_protocol milter_default_action
```

Valores esperados:

```ini
smtpd_milters = inet:localhost:11332
non_smtpd_milters = inet:localhost:11332
milter_protocol = 6
milter_default_action = accept
```

### 18.8. Añadir cabeceras de análisis

Crear el archivo:

```bash
sudo nano /etc/rspamd/local.d/milter_headers.conf
```

Añadir:

```lua
use = ["x-spamd-bar", "x-spam-level", "x-spam-status", "authentication-results"];
skip_local = false;
skip_authenticated = false;
```

Estas opciones permiten ver las cabeceras de análisis incluso en mensajes
enviados desde localhost o por un usuario autenticado.

Recargar Rspamd:

```bash
sudo systemctl reload rspamd
```

### 18.9. Integrar ClamAV con Rspamd

Crear el archivo:

```bash
sudo nano /etc/rspamd/local.d/antivirus.conf
```

Añadir:

```lua
clamav {
    symbol = "CLAM_VIRUS";
    type = "clamav";
    servers = "/var/run/clamav/clamd.ctl";
    scan_mime_parts = true;
    action = "reject";
    message = '${SCANNER}: virus found: "${VIRUS}"';
}
```

Esta configuración utiliza el socket Unix de ClamAV, analiza los adjuntos y
rechaza los mensajes infectados.

Validar y recargar:

```bash
sudo rspamadm configtest
sudo systemctl reload rspamd
```

Comprobar que ClamAV sigue activo:

```bash
sudo systemctl status clamav-daemon --no-pager -l
```

### 18.10. Desactivar clamav-milter

Como Rspamd ya consulta directamente a ClamAV, `clamav-milter` ya no debe
estar conectado a Postfix.

```bash
sudo systemctl disable --now clamav-milter
```

No es necesario desinstalar el paquete: se conserva como parte de la
instalación de ClamAV y como respaldo de la configuración anterior.

## 19. Prueba antivirus con EICAR

### 19.1. Crear el archivo de prueba

EICAR es un archivo de prueba estandarizado para comprobar la detección
antivirus. No es malware real y no causa daño al sistema.

Desde el usuario `adm-n1`, dentro de la VM, ejecutar:

```bash
mkdir -p ~/practica-rspamd/evidencias

printf '%s' 'X5O!P%@AP[4\PZX54(P^)7CC)7}$EICAR-STANDARD-ANTIVIRUS-TEST-FILE!$H+H*' \
  > ~/practica-rspamd/eicar.com.txt
```

Comprobar el tamaño:

```bash
wc -c ~/practica-rspamd/eicar.com.txt
```

Debe indicar:

```text
68
```

Las comillas simples son importantes para conservar literalmente el contenido
oficial de la prueba.

### 19.2. Observar los registros

Antes de enviar el mensaje, abrir una segunda terminal:

```bash
sudo journalctl -f -o short-iso \
  | grep --line-buffered -Ei 'postfix|rspamd|clamav|eicar|CLAM_VIRUS|milter'
```

Si el sistema dispone de `/var/log/mail.log`, también se puede usar:

```bash
sudo tail -f /var/log/mail.log
```

### 19.3. Enviar el archivo desde Thunderbird

En Thunderbird, abierto dentro de la VM:

1. Redactar un mensaje para `adm-n1@raulprueba.test`.
2. Escribir el asunto `Prueba Rspamd - EICAR`.
3. Adjuntar `eicar.com.txt` desde la carpeta `practica-rspamd`.
4. Enviar el mensaje.

Resultado esperado:

```text
Thunderbird muestra un error al enviar el mensaje.
El mensaje no aparece en la bandeja de entrada.
```

El rechazo debe incluir un mensaje similar a:

```text
Message rejected: virus found
```

El código y el texto exactos pueden variar según la versión de Postfix, Rspamd,
ClamAV y Thunderbird.

### 19.4. Comprobar la detección

Guardar los registros recientes:

```bash
sudo journalctl --since "15 minutes ago" --no-pager -o short-iso \
  | grep -Ei 'postfix|rspamd|clamav|eicar|CLAM_VIRUS|milter' \
  > ~/practica-rspamd/evidencias/registro-prueba.txt
```

Localizar las líneas relevantes:

```bash
grep -Ei 'eicar|CLAM_VIRUS|virus found|milter-reject' \
  ~/practica-rspamd/evidencias/registro-prueba.txt
```

Las evidencias deben mostrar:

- Rspamd recibe el mensaje desde Postfix.
- ClamAV detecta el archivo de prueba EICAR.
- Rspamd rechaza el mensaje.
- Postfix registra el rechazo.

## 20. Guardar evidencias para la entrega

Guardar el estado de los servicios:

```bash
{
  date -Is
  for servicio in rspamd redis-server clamav-daemon clamav-freshclam postfix; do
    printf '\n--- %s ---\n' "$servicio"
    systemctl is-active "$servicio"
    systemctl is-enabled "$servicio"
  done
} > ~/practica-rspamd/evidencias/estado-servicios.txt
```

Guardar la integración de Postfix:

```bash
sudo postconf smtpd_milters non_smtpd_milters milter_protocol milter_default_action \
  > ~/practica-rspamd/evidencias/postfix-rspamd.txt
```

Guardar la configuración de Rspamd:

```bash
cat /etc/rspamd/local.d/redis.conf \
  > ~/practica-rspamd/evidencias/redis-rspamd.txt

cat /etc/rspamd/local.d/milter_headers.conf \
  > ~/practica-rspamd/evidencias/milter-headers-rspamd.txt

cat /etc/rspamd/local.d/antivirus.conf \
  > ~/practica-rspamd/evidencias/antivirus-rspamd.txt
```

Crear un índice de evidencias:

```bash
{
  echo "Índice de evidencias - $(date -Is)"
  echo

  echo "## Archivos guardados"
  ls -lh ~/practica-rspamd/evidencias/

  echo
  echo "## Estado de servicios"
  for servicio in rspamd redis-server clamav-daemon clamav-freshclam postfix; do
    printf '%-20s activo=%s habilitado=%s\n' \
      "$servicio" \
      "$(systemctl is-active "$servicio")" \
      "$(systemctl is-enabled "$servicio" 2>/dev/null)"
  done

  echo
  echo "## Detecciones relevantes"
  grep -Ei 'eicar|CLAM_VIRUS|virus found|milter-reject' \
    ~/practica-rspamd/evidencias/registro-prueba.txt
} > ~/practica-rspamd/evidencias/indice-evidencias.txt
```

Evidencias recomendadas:

| Evidencia | Qué demuestra |
|---|---|
| Actualización correcta de firmas | Base de firmas de ClamAV preparada |
| Estado de los servicios | Rspamd, Redis, ClamAV y Postfix activos |
| Configuración Postfix–Rspamd | Integración mediante milter |
| Configuración Rspamd–ClamAV | Análisis antivirus de adjuntos |
| Análisis con `rspamc` | Rspamd analiza los mensajes |
| Mensaje limpio recibido | El correo legítimo sigue funcionando |
| Rechazo de GTUBE | Rspamd puede rechazar spam |
| Rechazo en Thunderbird | EICAR es bloqueado antes de entregarse |
| Registros de detección y rechazo | ClamAV detecta y Rspamd rechaza |

No subir a GitHub el archivo EICAR, contraseñas, buzones, claves privadas ni
perfiles completos de Thunderbird. Revisar las capturas y ocultar cualquier
dato sensible antes de publicarlas.

## 21. Resultado obtenido

Se integró Rspamd con Postfix mediante milter y se conectó a ClamAV para
analizar los adjuntos.

- El mensaje limpio fue analizado y entregado correctamente.
- El mensaje GTUBE fue detectado como spam y rechazado.
- El mensaje con EICAR fue detectado por ClamAV y rechazado por Rspamd.
- Se guardaron los registros de análisis, detección y rechazo.
- Los servicios quedaron habilitados para el arranque.

## 22. Conclusión

Se ha montado un entorno de correo local en Debian 13 utilizando Postfix para
el transporte y la entrega, Dovecot para el acceso IMAP y Thunderbird como
cliente gráfico.

Posteriormente se ha integrado Rspamd como filtro de correo y ClamAV como
antivirus. Rspamd analiza los mensajes antes de su entrega y consulta a ClamAV
para detectar malware en los adjuntos.

Las pruebas con GTUBE y EICAR han permitido comprobar que Rspamd puede rechazar
mensajes sospechosos y que los archivos infectados son bloqueados antes de
llegar al buzón.
