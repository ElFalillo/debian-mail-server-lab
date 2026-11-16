# Laboratorio de correo local en Debian 13

Instalación y configuración de un servidor de correo de pruebas con Postfix, Dovecot y mailutils, utilizando Thunderbird como cliente.

El laboratorio permite enviar y recibir mensajes entre cuentas locales sin utilizar un proveedor de correo externo.

## 1. Objetivos

- Configurar Postfix para recibir y entregar correo local.
- Configurar Dovecot para consultar los buzones mediante IMAP.
- Compartir el mismo almacenamiento Maildir entre ambos servicios.
- Enviar mensajes de prueba desde la terminal con mailutils.
- Enviar y recibir mensajes desde Thunderbird.
- Verificar el funcionamiento de los servicios y consultar sus registros.

## 2. Entorno del laboratorio

| Elemento | Configuración |
|---|---|
| Sistema operativo | Debian 13 |
| Servidor SMTP | Postfix |
| Servidor IMAP | Dovecot 2.4 |
| Herramientas de terminal | mailutils |
| Cliente gráfico | Thunderbird |
| Dominio de pruebas | raulprueba.test |
| Nombre del servidor de correo | mail.raulprueba.test |
| Usuario local de pruebas | adm-n1 |
| Dirección de correo | adm-n1@raulprueba.test |
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
│   └── Entrega los mensajes en /home/adm-n1/Maildir/
│
└── Dovecot
    └── Permite consultar ese mismo buzón mediante IMAP
```

Recorrido de un mensaje enviado a la propia cuenta:

```text
Thunderbird → Postfix → Maildir → Dovecot → Thunderbird
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
| Email address / Correo | adm-n1@raulprueba.test |
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
Cliente SMTP → Postfix → entrega en Maildir → Dovecot IMAP → cliente
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

Antes de publicarlas, revisar y ocultar información personal, cuentas ajenas
al laboratorio y cualquier dato sensible.

## 17. Conclusión

Se ha montado un entorno de correo local en Debian 13 utilizando Postfix para
el transporte y la entrega, Dovecot para el acceso IMAP y Thunderbird como
cliente gráfico.

Las comprobaciones permiten separar los problemas de servicio, conectividad,
autenticación y almacenamiento.
