# Laboratorio 07 — Keystores y almacenes de confianza

## Añadir una autoridad certificadora al sistema

En el ejercicio anterior observamos que los sistemas Linux mantienen un almacén de certificados de confianza.
Ahora añadiremos **nuestra autoridad certificadora raíz** a ese almacén para que el sistema pueda confiar en los certificados emitidos por nuestra PKI.

Esto permitirá que herramientas como `curl` o aplicaciones que utilicen el almacén del sistema acepten certificados firmados por nuestra CA.

---

### Paso 1 — Copiar el certificado raíz al directorio de autoridades locales

Sitúate en el directorio donde se encuentra el certificado raíz que creamos anteriormente.

```bash id="e2l0pp"
cd ~/pki-ca
```

Copia el certificado raíz al directorio de autoridades locales del sistema.

```bash id="3kz1a3"
sudo cp ca.crt /usr/local/share/ca-certificates/training-root-ca.crt
```

Comprueba que el archivo se ha copiado correctamente.

```bash id="6cah70"
ls /usr/local/share/ca-certificates
```

Deberías ver el archivo:

```id="5qg3gy"
training-root-ca.crt
```

Este directorio es utilizado para añadir certificados personalizados al sistema.

---

### Paso 2 — Actualizar el almacén de certificados del sistema

Una vez copiado el certificado, es necesario actualizar el almacén del sistema.

```bash id="p6a2h1"
sudo update-ca-certificates
```

Durante la ejecución aparecerá un mensaje indicando que se ha añadido un nuevo certificado.

Algo similar a:

```id="2n8n9j"
1 added, 0 removed
```

Esto indica que el certificado raíz ha sido incorporado al almacén de confianza del sistema.

---

### Paso 3 — Verificar que el certificado aparece en el almacén

El almacén consolidado de certificados de confianza del sistema se encuentra en:

```bash
/etc/ssl/certs/ca-certificates.crt
```

Busca tu autoridad certificadora dentro de este archivo:

```bash id="qpxlfr"
openssl x509 -in /usr/local/share/ca-certificates/training-root-ca.crt -noout -subject
```

Anota el texto que aparece en `subject` (especialmente el `CN` que hayas utilizado al crear la CA raíz, por ejemplo `Training Root CA`) y úsalo para buscar dentro del almacén consolidado:

```bash
grep "<TU_COMMON_NAME_CA>" /etc/ssl/certs/ca-certificates.crt
```

Sustituye `<TU_COMMON_NAME_CA>` por el valor real de `CN` que hayas utilizado.  
Si aparece una coincidencia, el certificado ha sido añadido correctamente al almacén de confianza.

También puedes verificarlo utilizando OpenSSL contra ese almacén:

```bash
openssl verify -CAfile /etc/ssl/certs/ca-certificates.crt ~/pki-labs/web-server/server.crt
```

Si la salida indica `server.crt: OK`, confirma que el sistema reconoce nuestra autoridad certificadora y puede validar certificados emitidos por ella.

---

### Paso 4 — Verificar un certificado emitido por nuestra CA

Ahora intenta verificar el certificado del servidor utilizando OpenSSL.

> Importante: **es normal que falle** si no proporcionas el certificado de la CA intermedia.  
> El almacén del sistema suele contener la **CA raíz**, pero el certificado `server.crt` está firmado por la **CA intermedia**.  
> Con `-untrusted` le indicas a OpenSSL cuál es el certificado intermedio para que pueda construir la cadena (root → intermedia → servidor).

```bash id="dsmn4s"
openssl verify \
  -CAfile /etc/ssl/certs/ca-certificates.crt \
  -untrusted ~/pki-ca/intermediate/intermediate.crt \
  ~/pki-labs/web-server/server.crt
```

Si la cadena de certificados está completa, la salida debería indicar:

```id="p8d0s0"
server.crt: OK
```

Esto significa que el sistema confía en la autoridad certificadora que firmó el certificado.

---

### Paso 5 — Probar una conexión HTTPS con un certificado firmado por nuestra CA

En este paso vas a **levantar tú mismo un servicio HTTPS de prueba con NGINX en Docker**, usando el certificado `server.crt` generado en los laboratorios.

1. Prepara un directorio de trabajo y una página sencilla:

```bash
mkdir -p ~/tls-web
cd ~/tls-web
echo "Servidor HTTPS de prueba (Lab07)" > index.html
```

2. Copia los certificados generados anteriormente:

```bash
cp ~/pki-labs/web-server/server.crt .
cp ~/pki-labs/web-server/server.key .
cp ~/pki-labs/web-server/fullchain.crt .
```

3. Crea el archivo de configuración `nginx.conf` con el siguiente contenido:

```nginx
events {}

http {
    server {
        listen 443 ssl;

        ssl_certificate /etc/nginx/certs/fullchain.crt;
        ssl_certificate_key /etc/nginx/certs/server.key;

        location / {
            root /usr/share/nginx/html;
            index index.html;
        }
    }
}
```

4. Desde `~/tls-web`, arranca el servidor NGINX dentro de un contenedor Docker:

```bash
docker run -d \
  -p 8443:443 \
  -v $(pwd)/index.html:/usr/share/nginx/html/index.html \
  -v $(pwd)/nginx.conf:/etc/nginx/nginx.conf \
  -v $(pwd)/fullchain.crt:/etc/nginx/certs/fullchain.crt \
  -v $(pwd)/server.key:/etc/nginx/certs/server.key \
  nginx
```

5. Comprueba la conexión HTTPS desde tu sistema (que ya confía en tu CA raíz gracias a los pasos anteriores):

```bash id="cb3b6c"
curl https://localhost:8443
```

Si el certificado es válido y la CA está en el almacén de confianza, la conexión debería establecerse sin errores relacionados con certificados.

Esto demuestra cómo añadir una CA al sistema permite que múltiples aplicaciones confíen automáticamente en los certificados emitidos por ella.

---

En este ejercicio hemos añadido nuestra autoridad certificadora al almacén de confianza del sistema y hemos comprobado cómo esto permite validar certificados emitidos por nuestra PKI.
