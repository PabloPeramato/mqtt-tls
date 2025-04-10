# Configuración de Mosquitto con TLS y autenticación
Esta documentación describe los pasos necesarios para crear los certificados requeridos para habilitar la conexión segura mediante TLS en un servidor
Mosquitto, configurar la autenticación mediante contraseñas y cómo realizar pruebas de publicación y suscripción.

## 1. Instalación de mosquitto y openssl
Para instalar Mosquitto, Mosquitto-Clients y OpenSSL en tu sistema, debes ejecutar el siguiente comando en la terminal:
```ruby
sudo apt update
sudo apt install mosquitto mosquitto-clients openssl -y
```
- `mosquitto`: es el servidor MQTT que permite la gestión de mensajes entre clientes, usando el protocolo de comunicación MQTT.
- `mosquitto-clients`: es el conjunto de herramientas de línea de comandos que incluye los comandos mosquitto_pub y mosquitto_sub, los cuales se utilizan para publicar y suscribirse a mensajes en tópicos MQTT, respectivamente.
- `openssl`: es una biblioteca de herramientas para trabajar con certificados y claves criptográficas, y es esencial para la configuración de conexiones seguras TLS/SSL.

Este comando instalará los paquetes necesarios en tu sistema de forma automática, asegurando que tanto el servidor Mosquitto como las herramientas de cliente y las utilidades de OpenSSL estén disponibles para su uso.
Después de la instalación, podrás proceder con la configuración y prueba del servidor Mosquitto de manera segura.

## 2. Creación de la Autoridad Certificadora (CA)
### Paso 1: Accede al directorio de certificados de la CA
```ruby
cd /etc/mosquitto/ca_certificates
```
### Paso 2: Genera la clave privada de la CA
Genera la clave privada de la CA con cifrado AES-256:
```ruby
sudo openssl genpkey -algorithm RSA -out ca.key -aes256
```
### Paso 3: Crea la solicitud de firma de certificado (CSR) para la CA
```ruby
sudo openssl req -new -key ca.key -out ca.csr -subj "/C=ES/ST=Salamanca/L=Salamanca/O=MiMosquittoCA/CN=MiMosquittoCA"
```
### Paso 4: Crea el certificado de la CA
Firma el CST con la clave privada de la CA para generar el certificado:
```ruby
sudo openssl x509 -req -in ca.csr -signkey ca.key -out ca.crt -days 3650
```

## 3. Creación de los certificados del servidor
### Paso 1: Accede al directorio de certificados del servidor
```ruby
cd /etc/mosquitto/certs
```
### Paso 2: Genera la clave privada del servidor
Genera la clave privada del servidor con cifrado AES-256:
```ruby
sudo openssl genpkey -algorithm RSA -out server.key -aes256
```
### Paso 3: Crea la solicitud de firma de certificado (CSR) para el servidor
```ruby
sudo openssl req -new -key server.key -out server.csr -subj "/C=ES/ST=Salamanca/L=Salamanca/O=MiServidor/CN={ip_address}"
```
- `{ip_address}`: cambiar este valor por la dirección IP del equipo en el que se hospedará el broker.
### Paso 4: Firma el CSR con la CA para generar el certificado del servidor
```ruby
sudo openssl x509 -req -in server.csr -CA /etc/mosquitto/ca_certificates/ca.crt -CAkey /etc/mosquitto/ca_certificates/ca.key -CAcreateserial -out server.crt -days 3650
```

## 4. Creación de los certificados del cliente
### Paso 1: Genera la clave privada del cliente
```ruby
sudo openssl genpkey -algorithm RSA -out client.key -aes256
```
### Paso 2: Crea la solicitud de firma de certificado (CSR) para el cliente
```ruby
sudo openssl req -new -key client.key -out client.csr -subj "/C=ES/ST=Salamanca/L=Salamanca/O=MiCliente/CN=cliente1"
```
### Paso 3: Firma el CSR con la CA para generar el certificado del cliente
```ruby
sudo openssl x509 -req -in client.csr -CA /etc/mosquitto/ca_certificates/ca.crt -CAkey /etc/mosquitto/ca_certificates/ca.key -CAcreateserial -out client.crt -days 3650
```
### Paso 4: Verifica los certificados generados
Verifica que los certificados del servidor y del cliente sean válidos:
```ruby
sudo openssl verify -CAfile /etc/mosquitto/ca_certificates/ca.crt /etc/mosquitto/certs/server.crt
sudo openssl verify -CAfile /etc/mosquitto/ca_certificates/ca.crt /etc/mosquitto/certs/client.crt
```
Ambos deberían devolver:
- `server.crt: OK`
- `client.crt: OK`

## 5. Cambiar propietario de los certificados
Hará que acceder al directorio `/etc/mosquitto` y ejecutar los siguientes comandos:
```ruby
cd /etc/mosquitto
sudo chown mosquitto:mosquitto ca_certificates/*
sudo chown mosquitto:mosquitto certs/*
```

## 6.	Configuración de la autenticación con contraseña
### Paso 1: Generar un archivo de contraseñas para el usuario admin
```ruby
sudo mosquitto_passwd -c /etc/mosquitto/passwd admin
```
### Paso 2: Protege el archivo de contraseñas
Asegúrate de que el archivo de contraseñas sea accesible solo por el usuario y grupo de Mosquitto:
```ruby
sudo chown mosquitto:mosquitto /etc/mosquitto/passwd
sudo chmod 440 /etc/mosquitto/passwd
```

## 7.	Configuración del archivo de Mosquitto
### Paso 1: Edita el archivo de configuración de Mosquitto
Edita el archivo de configuración de Mosquitto para habilitar TLS y la autenticación por contraseña.
```ruby
sudo nano /etc/mosquitto/mosquitto.conf
```
### Paso 2: Configuración en el archivo mosquitto.conf
Agrega las siguientes líneas en el archivo de configuración:
```ruby
# Habilitar TLS
listener 8883
protocol mqtt

cafile /etc/mosquitto/ca_certificates/ca.crt
certfile /etc/mosquitto/certs/server.crt
keyfile /etc/mosquitto/certs/server.key

# Archivo de contraseñas
password_file /etc/mosquitto/passwd

# Requerir certificado del cliente (si es necesario)
require_certificate true
```

## 8.	Eliminar la contraseña de las claves privadas
### Paso 1: Elimina la contraseña de la clave del servidor
```ruby
sudo openssl rsa -in /etc/mosquitto/certs/server.key -out /etc/mosquitto/certs/server.key
```
### Paso 2: Elimina la contraseña de la clave del cliente
```ruby
sudo openssl rsa -in /etc/mosquitto/certs/client.key -out /etc/mosquitto/certs/client.key
```
### Paso 3: Reinicia Mosquitto nuevamente
```ruby
sudo systemctl restart mosquitto
```

## 9.	Pruebas de Publicación y Suscripción
Para realizar pruebas de conexión, utiliza los siguientes comandos de suscripción y publicación en la terminal.
### Comando para Publicar un Mensaje
Este comando publicará un mensaje en el tópico test/prueba:
```ruby
mosquitto_pub -h {ip_address} -p 8883 -t "test/prueba" \
  --cafile /etc/mosquitto/ca_certificates/ca.crt \
  --cert /etc/mosquitto/certs/client.crt \
  --key /etc/mosquitto/certs/client.key \
  -m "Este es un mensaje de prueba" -u "admin" -P "proxmox.rpi" --insecure -d
```
•	`-h {ip_address}`: Dirección IP del servidor Mosquitto. Cambiar este valor por la dirección IP del equipo en el que se hospedará el broker.
•	`-p 8883`: Puerto en el que se conecta (8883 es para TLS).
•	`-t "test/prueba"`: El tópico en el que se publica el mensaje.
•	`--cafile`: Ruta al archivo de certificado de la CA.
•	`--cert`: Ruta al archivo de certificado del cliente.
•	`--key`: Ruta al archivo de clave privada del cliente.
•	`-m`: El mensaje que se desea publicar.
•	`-u "admin"`: Nombre de usuario para autenticación.
•	`-P "proxmox.rpi"`: Contraseña para el usuario admin.
•	`-d`: Modo depuración (debug).
### Comando para Suscribirse a un Tópico
Este comando se suscribirá al tópico #, que suscribe a todos los mensajes publicados en cualquier tópico:
```ruby
mosquitto_sub -h {ip_address} -p 8883 -t "#" \
  --cafile /etc/mosquitto/ca_certificates/ca.crt \
  --cert /etc/mosquitto/certs/client.crt \
  --key /etc/mosquitto/certs/client.key \
  -u "admin" -P "proxmox.rpi" --insecure -d
```
•	`-h {ip_address}`: Dirección IP del servidor Mosquitto. Cambiar este valor por la dirección IP del equipo en el que se hospedará el broker.
•	`-p 8883`: Puerto en el que se conecta (8883 es para TLS).
•	`-t "#"`: Se suscribe a todos los tópicos.
•	`--cafile`: Ruta al archivo de certificado de la CA.
•	`--cert`: Ruta al archivo de certificado del cliente.
•	`--key`: Ruta al archivo de clave privada del cliente.
•	`-u "admin"`: Nombre de usuario para autenticación.
•	`-P "proxmox.rpi"`: Contraseña para el usuario admin.
•	`-d`: Modo depuración (debug).

# RESUMEN
1.	`Generación de certificados`: Se generan los certificados de la CA, servidor y cliente utilizando OpenSSL.
2.	`Autenticación`: Se configura la autenticación de Mosquitto con un archivo de contraseñas.
3.	`Configuración de TLS`: Se habilita TLS en Mosquitto mediante la configuración del archivo mosquitto.conf para usar los certificados generados.
4.	`Eliminación de contraseñas de las claves privadas`: Para evitar que Mosquitto pida las contraseñas de las claves al arrancar, se eliminan las contraseñas de las claves privadas del servidor y del cliente.
5.	`Pruebas de conexión`: Se proporcionan los comandos de terminal para realizar pruebas de publicación y suscripción de mensajes en el broker MQTT.
Si sigues estos pasos, tu configuración de Mosquitto debería estar correctamente habilitada para conexiones seguras con TLS y autenticación por contraseña,
y las pruebas de publicación y suscripción deberían funcionar correctamente.
