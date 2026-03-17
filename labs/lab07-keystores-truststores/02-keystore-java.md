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
grep "Training Root CA" /etc/ssl/certs/ca-certificates.crt
```

Si aparece una coincidencia, el certificado ha sido añadido correctamente al almacén de confianza.

También puedes verificarlo utilizando OpenSSL contra ese almacén:

```bash
openssl verify -CAfile /etc/ssl/certs/ca-certificates.crt ~/pki-labs/web-server/server.crt
```

Si la salida indica `server.crt: OK`, confirma que el sistema reconoce nuestra autoridad certificadora y puede validar certificados emitidos por ella.

---

### Paso 4 — Verificar un certificado emitido por nuestra CA

Ahora intenta verificar el certificado del servidor utilizando OpenSSL.

```bash id="dsmn4s"
openssl verify ~/pki-labs/web-server/server.crt
```

Si la cadena de certificados está completa, la salida debería indicar:

```id="p8d0s0"
server.crt: OK
```

Esto significa que el sistema confía en la autoridad certificadora que firmó el certificado.

---

### Paso 5 — Probar una conexión HTTPS con un certificado firmado por nuestra CA

Si dispones de un servicio HTTPS configurado con el certificado emitido por la CA del laboratorio, prueba a conectarte utilizando `curl`.

```bash id="cb3b6c"
curl https://localhost:8443
```

Si el certificado es válido y la CA está en el almacén de confianza, la conexión debería establecerse sin errores relacionados con certificados.

Esto demuestra cómo añadir una CA al sistema permite que múltiples aplicaciones confíen automáticamente en los certificados emitidos por ella.

---

En este ejercicio hemos añadido nuestra autoridad certificadora al almacén de confianza del sistema y hemos comprobado cómo esto permite validar certificados emitidos por nuestra PKI.
