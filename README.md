# Suricata + EveBox + Alertas por correo con Postfix

<img width="1662" height="503" alt="image" src="https://github.com/user-attachments/assets/47d610e4-abab-459a-88a4-7de6eee20a72" />


Este repositorio documenta un mecanismo de notificación por correo electrónico para Suricata. Un script Bash vigila el archivo de eventos `eve.json`, filtra las alertas de mayor prioridad y envía correos mediante Postfix, utilizando un relay SMTP autenticado. EveBox permite consultar los eventos desde una interfaz web, pero el script de notificación funciona de forma independiente.

## Arquitectura

1. Suricata escribe los eventos en `/var/log/suricata/eve.json`.
2. `inotifywait` detecta modificaciones en el archivo de eventos.
3. `jq` filtra los eventos de tipo `alert` con severidad 1 o 2 entre las diez últimas líneas.
4. El script Bash prepara un correo con la firma, severidad, IP y puerto de origen y destino, y la hora de procesamiento.
5. `mail` entrega el mensaje a Postfix.
6. Postfix reenvía el mensaje mediante un relay SMTP autenticado y protegido con TLS.
7. Un servicio `systemd` mantiene el script en ejecución y lo reinicia si finaliza.

## Requisitos

En Ubuntu o Debian, instala los paquetes necesarios:

```bash
sudo apt update
sudo apt install inotify-tools jq mailutils postfix
```

Suricata debe estar configurado para escribir los eventos EVE JSON en `/var/log/suricata/eve.json`.

## Script de alertas por correo

El siguiente script es el utilizado por el servicio `systemd`. Guárdalo en el equipo monitorizado como `/usr/local/bin/suricata-alerta.sh`.

```bash
#!/bin/bash

WATCH_FILE="/var/log/suricata/eve.json"
LAST_HASH=""
LAST_ALERT_TIME=0
INTERVAL=60  # Tiempo mínimo entre alertas (segundos)

while inotifywait -e modify "$WATCH_FILE"; do
    NEW_LINE=$(tail -n 10 "$WATCH_FILE" | jq -c 'select(.event_type == "alert" and (.alert.severity == 1 or .alert.severity == 2))' | tail -n 1)

    if [[ -n "$NEW_LINE" ]]; then
        CURRENT_HASH=$(echo "$NEW_LINE" | jq -r '[.timestamp, .alert.signature, .src_ip, .dest_ip] | join(" ")' | md5sum | awk '{print $1}')
        NOW=$(date +%s)

        if [[ "$CURRENT_HASH" != "$LAST_HASH" && $((NOW - LAST_ALERT_TIME)) -ge "$INTERVAL" ]]; then
            LAST_HASH="$CURRENT_HASH"
            LAST_ALERT_TIME=$NOW

            SIGNATURE=$(echo "$NEW_LINE" | jq -r '.alert.signature')
            SEVERITY=$(echo "$NEW_LINE" | jq -r '.alert.severity')
            SRC_IP=$(echo "$NEW_LINE" | jq -r '.src_ip')
            SRC_PORT=$(echo "$NEW_LINE" | jq -r '.src_port')
            DEST_IP=$(echo "$NEW_LINE" | jq -r '.dest_ip')
            DEST_PORT=$(echo "$NEW_LINE" | jq -r '.dest_port')
            TIMESTAMP=$(TZ="Europe/Madrid" date "+%Y-%m-%d %H:%M:%S")

            echo "[DEBUG] $SIGNATURE (sev.$SEVERITY) de $SRC_IP:$SRC_PORT a $DEST_IP:$DEST_PORT @ $TIMESTAMP" >> /tmp/alerta-debug.log

            echo -e "ALERTA DETECTADA EN SURICATA \n\nFirma: $SIGNATURE\nSeveridad: $SEVERITY\nOrigen: $SRC_IP:$SRC_PORT\nDestino: $DEST_IP:$DEST_PORT\nHora: $TIMESTAMP" | \
            mail -s "[Suricata] Alerta severidad $SEVERITY detectada" DESTINATARIO_ALERTAS@example.com

            sleep 10
        fi
    fi
done
```

Concede permisos de ejecución:

```bash
sudo chmod 750 /usr/local/bin/suricata-alerta.sh
```

Sustituye `DESTINATARIO_ALERTAS@example.com` en el script por el buzón que recibirá las alertas.

### Filtrado de alertas y limitación de frecuencia

- Solo se seleccionan eventos cuyo `event_type` sea `alert` y cuya severidad sea `1` o `2`.
- Tras detectar una modificación del archivo, el script examina las diez últimas líneas y selecciona la última alerta coincidente de esa muestra.
- Impone un intervalo mínimo de 60 segundos entre envíos y, después, espera otros 10 segundos.
- Añade una línea de depuración a `/tmp/alerta-debug.log`.
- La hora incluida en el mensaje es la hora local de procesamiento en la zona `Europe/Madrid`, no la marca de tiempo original del evento de Suricata.
- El hash del último evento y la hora del último envío se guardan en memoria y se reinician cuando se reinicia el script.

Esta implementación es sencilla. Si se generan muchos eventos en poco tiempo, pueden omitirse alertas porque solo se examinan las diez últimas líneas. La rotación o sustitución del archivo también puede requerir adaptar la vigilancia. Para un entorno de producción se recomienda procesar de forma persistente cada evento nuevo y gestionar explícitamente la rotación del registro.

## Servicio systemd

Crea el archivo `/etc/systemd/system/suricata-alerta.service`:

```ini
[Unit]
Description=Script de alertas por correo de Suricata (inotify + mail)
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/suricata-alerta.sh
Restart=always
RestartSec=10
User=root

[Install]
WantedBy=multi-user.target
```

Habilita e inicia el servicio:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now suricata-alerta.service
```

Consulta el estado y los registros:

```bash
sudo systemctl status suricata-alerta.service
sudo journalctl -u suricata-alerta.service -f
sudo tail -f /tmp/alerta-debug.log
```

## Configuración de Postfix como relay SMTP

El siguiente fragmento es un ejemplo de configuración para `main.cf`. Sustituye el nombre de host y el relay SMTP de ejemplo por los valores adecuados para el entorno.

```ini
# Banner y compatibilidad
smtpd_banner = $myhostname ESMTP $mail_name (Ubuntu)
biff = no
append_dot_mydomain = no
readme_directory = no
compatibility_level = 3.6

# Nombre del host
myhostname = HOST_LOCAL
myorigin = $myhostname

# Destinos locales
mydestination = $myhostname, localhost.localdomain, localhost
alias_maps = hash:/etc/aliases

# Permite el envío local desde este equipo
mynetworks = 127.0.0.0/8 [::ffff:127.0.0.0]/104 [::1]/128

# Escucha únicamente en la interfaz local
inet_interfaces = loopback-only
inet_protocols = ipv4

# Relay SMTP
relayhost = [SMTP_PROVEEDOR.example]:587

# TLS
smtp_tls_security_level = encrypt
smtp_tls_CApath = /etc/ssl/certs
smtp_tls_CAfile = /etc/ssl/certs/ca-certificates.crt
smtp_tls_session_cache_database = btree:${data_directory}/smtp_scache

# Autenticación SMTP
smtp_sasl_auth_enable = yes
smtp_sasl_password_maps = hash:/etc/postfix/sasl_passwd
smtp_sasl_security_options = noanonymous

# Restricciones y límites
smtpd_relay_restrictions = permit_mynetworks permit_sasl_authenticated defer_unauth_destination
mailbox_size_limit = 0
recipient_delimiter = +
alias_database = hash:/etc/aliases
notify_classes = bounce, policy, protocol, resource, software
```

### Credenciales SMTP

Crea `/etc/postfix/sasl_passwd` en el equipo. No incluyas este archivo ni su base de datos generada en el control de versiones.

```text
[SMTP_PROVEEDOR.example]:587 USUARIO_SMTP:CONTRASEÑA_O_TOKEN_DE_APLICACION
```

Protege las credenciales y genera la base de datos de Postfix:

```bash
sudo chown root:root /etc/postfix/sasl_passwd
sudo chmod 600 /etc/postfix/sasl_passwd
sudo postmap /etc/postfix/sasl_passwd
sudo chmod 600 /etc/postfix/sasl_passwd.db
sudo systemctl restart postfix
```

Comprueba la configuración y los registros del servicio:

```bash
sudo postfix check
sudo systemctl status postfix
sudo journalctl -u postfix --since "30 minutes ago"
```

## Prueba de envío

Envía un mensaje de prueba independientemente de Suricata:

```bash
echo "Correo de prueba desde Postfix" | mail -s "Prueba SMTP de Suricata" DESTINATARIO_ALERTAS@example.com
```

Si el mensaje no llega, revisa los registros de Postfix y comprueba la autenticación SMTP, los permisos del relay, la resolución DNS y la conectividad de red.

## Consideraciones de seguridad

- Utiliza una contraseña de aplicación o un token cuando el proveedor SMTP lo requiera.
- Limita el acceso a `/etc/postfix/sasl_passwd` y `/etc/postfix/sasl_passwd.db` al usuario `root`.
- No incluyas credenciales reales, direcciones personales, IP públicas ni identificadores internos en el repositorio.
- En este ejemplo, el servicio se ejecuta como `root`. Revisa los permisos y privilegios antes de adaptarlo a un entorno de producción.
- Los correos pueden revelar direcciones IP y firmas de detección fuera de la red monitorizada. Evalúa la sensibilidad de esos datos.

## Relación con EveBox

EveBox proporciona una interfaz web para examinar los eventos de Suricata. En esta configuración, Suricata genera los eventos, EveBox permite visualizarlos y el script Bash independiente envía correos para las alertas seleccionadas. Esta configuración no implica que EveBox envíe los correos por sí mismo.


