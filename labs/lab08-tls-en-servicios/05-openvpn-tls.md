## Lab 05 — OpenVPN con certificados X.509

En este ejercicio utilizarás la PKI del curso para configurar una topología sencilla de **OpenVPN** con certificados X.509.

El objetivo es ver cómo se usan los certificados emitidos por tu CA en un servicio de VPN real.

---

### 1. Preparar certificados para OpenVPN

Partiendo de tu PKI del curso:

1. Genera una clave y CSR para el servidor VPN:

```bash
cd ~/pki-ca        # ajusta esta ruta a tu estructura real
openssl genpkey -algorithm RSA -out vpn-server.key
openssl req -new -key vpn-server.key -out vpn-server.csr \
  -subj "/CN=vpn.lab.local"
```

2. Crea un archivo de extensiones para un certificado **de servidor**.

En el lab 04 creamos un archivo de extensiones para la CA intermedia (`intermediate/intermediate-ext.cnf`).
Para firmar certificados de servicios (como OpenVPN) necesitamos otro archivo de extensiones distinto (porque el certificado final **no** debe ser CA).

Crea el archivo `intermediate/server-ext.cnf` con este contenido:

```bash
cat > intermediate/server-ext.cnf <<'EOF'
[ server_cert ]
basicConstraints = critical, CA:false
keyUsage = critical, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid,issuer
EOF
```

> Si tu OpenVPN va a validarse por nombre (DNS), en TLS moderno suele ser necesario `subjectAltName`.
> Puedes añadirlo ampliando el fichero, por ejemplo:
>
> ```text
> subjectAltName = @alt_names
>
> [ alt_names ]
> DNS.1 = vpn.lab.local
> ```

3. Firma la CSR con tu CA intermedia **aplicando estas extensiones**:

```bash
openssl x509 -req -in vpn-server.csr \
  -CA intermediate/intermediate.crt -CAkey intermediate/private/intermediate.key \
  -CAcreateserial -out vpn-server.crt -days 365 \
  -extfile intermediate/server-ext.cnf -extensions server_cert
```

4. Genera también un certificado de cliente (`vpn-client.key`, `vpn-client.crt`) siguiendo el mismo flujo.

Si necesitas extensiones explícitas para cliente, crea también `intermediate/client-ext.cnf`:

```bash
cat > intermediate/client-ext.cnf <<'EOF'
[ client_cert ]
basicConstraints = critical, CA:false
keyUsage = critical, digitalSignature, keyEncipherment
extendedKeyUsage = clientAuth
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid,issuer
EOF
```

Y firma la CSR del cliente con:

```bash
openssl x509 -req -in vpn-client.csr \
  -CA intermediate/intermediate.crt -CAkey intermediate/private/intermediate.key \
  -CAcreateserial -out vpn-client.crt -days 365 \
  -extfile intermediate/client-ext.cnf -extensions client_cert
```

---

### 2. Desplegar un servidor OpenVPN de laboratorio

Puedes utilizar un contenedor Docker de OpenVPN Community Edition o un servidor Linux de pruebas.

Ejemplo con Docker (orientativo):

```bash
docker run --name openvpn-lab -v $PWD:/etc/openvpn -d \
  --cap-add=NET_ADMIN -p 1194:1194/udp openvpn/openvpn-server
```

Asegúrate de:

* copiar `vpn-server.crt`, `vpn-server.key`, la cadena de CA y los certificados de CA necesarios a `/etc/openvpn`
* adaptar el fichero de configuración de OpenVPN para que utilice tus certificados.

---

### 3. Configurar OpenVPN para utilizar tu PKI

En la configuración del servidor (por ejemplo `/etc/openvpn/server.conf`), revisa las directivas:

```text
ca /etc/openvpn/ca.crt
cert /etc/openvpn/vpn-server.crt
key /etc/openvpn/vpn-server.key
dh /etc/openvpn/dh.pem
```

Comprueba que:

* `ca.crt` contiene la CA (o cadena) que emitió los certificados
* `vpn-server.crt` ha sido firmado por tu CA
* la clave privada coincide con el certificado.

Reinicia el servicio OpenVPN para aplicar cambios.

---

### 4. Configurar un cliente OpenVPN

En el lado cliente, prepara un archivo de configuración (por ejemplo `client.ovpn`) que incluya:

* la dirección del servidor (`remote`)
* el puerto y protocolo (`1194 udp`)
* las rutas a:
  * certificado de CA
  * certificado de cliente
  * clave privada de cliente.

Ejemplo simplificado:

```text
client
dev tun
proto udp
remote vpn.lab.local 1194

ca ca.crt
cert vpn-client.crt
key vpn-client.key

remote-cert-tls server
```

Inicia el cliente OpenVPN y verifica que la conexión se establece correctamente.

---

### 5. Verificar la cadena de certificados

Utiliza `openssl` para inspeccionar la cadena y el certificado del servidor OpenVPN.

```bash
openssl s_client -connect vpn.lab.local:1194 -showcerts
```

> Importante:
> - `-starttls none` no es una opción válida en muchas versiones de OpenSSL.
> - El comando `openssl s_client` aplica cuando OpenVPN está configurado sobre **TCP**.
> - Si tu OpenVPN usa **UDP** (caso habitual), valida certificado y cadena con:
>
> ```bash
> openssl x509 -in vpn-server.crt -noout -text
> openssl verify -CAfile ca.crt vpn-server.crt
> ```

Comprueba:

* el certificado del servidor
* la cadena de confianza
* la CA que firma el certificado.

---

### 6. Conclusiones

Al finalizar este ejercicio deberías ser capaz de:

* utilizar tu PKI para emitir certificados de servidor y cliente para OpenVPN
* configurar un servidor OpenVPN para utilizar esos certificados
* verificar la cadena de confianza utilizada en la conexión VPN.

