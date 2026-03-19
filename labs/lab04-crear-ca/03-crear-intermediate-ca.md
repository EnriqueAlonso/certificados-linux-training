# Laboratorio 04 — Crear una autoridad certificadora

## Crear una autoridad intermedia

En muchas infraestructuras PKI reales la **autoridad raíz no firma directamente los certificados de servidores**.
En su lugar se utilizan **autoridades intermedias** que firman los certificados operativos.

Esto permite proteger mejor la autoridad raíz y facilita la gestión de certificados.

En este ejercicio crearemos una **autoridad intermedia** firmada por nuestra CA raíz.

---

### Paso 1 — Crear el directorio de la autoridad intermedia

Sitúate dentro del directorio de la PKI.

```bash
cd ~/pki-ca
```

Crea un directorio para la autoridad intermedia.

```bash
mkdir intermediate
cd intermediate
```

Ahora crea la misma estructura de directorios utilizada anteriormente.

```bash
mkdir certs crl csr newcerts private
touch index.txt
echo 2000 > serial
```

Comprueba la estructura creada.

```bash
ls
```

Deberías ver algo similar a:

```
certs
crl
csr
newcerts
private
index.txt
serial
```

Esta estructura permitirá gestionar certificados emitidos por la CA intermedia y sus listas de revocación.

---

### Paso 2 — Crear el archivo de configuración de la CA intermedia

Al igual que la CA raíz tiene `openssl-ca.cnf`, la intermedia necesita su propio
archivo de configuración para poder firmar certificados y generar CRLs.

Vuelve al directorio raíz de la PKI:

```bash
cd ~/pki-ca
```

Crea el archivo `intermediate/openssl-intermediate.cnf`:

```bash
cat > intermediate/openssl-intermediate.cnf <<'EOF'
[ ca ]
default_ca = CA_default

[ CA_default ]
dir               = ./intermediate
certs             = $dir/certs
crl_dir           = $dir/crl
database          = $dir/index.txt
new_certs_dir     = $dir/newcerts
certificate       = $dir/intermediate.crt
serial            = $dir/serial
private_key       = $dir/private/intermediate.key
default_days      = 365
default_crl_days  = 30
default_md        = sha256
policy            = policy_any

[ policy_any ]
countryName             = optional
stateOrProvinceName     = optional
organizationName        = optional
commonName              = supplied
EOF
```

> Este archivo se ejecutará siempre desde `~/pki-ca`, por lo que `dir = ./intermediate`
> apunta correctamente a los directorios de la CA intermedia.

Comprueba que se ha creado:

```bash
ls intermediate/openssl-intermediate.cnf
```

---

### Paso 3 — Generar la clave privada de la CA intermedia

Sitúate dentro del directorio de la intermedia:

```bash
cd ~/pki-ca/intermediate
```

Genera la clave privada que utilizará la autoridad intermedia.

```bash
openssl genrsa -out private/intermediate.key 4096
```

Comprueba que el archivo se ha generado correctamente.

```bash
ls -l private
```

Deberías ver el archivo:

```
intermediate.key
```

Esta clave será utilizada para firmar certificados emitidos por la autoridad intermedia.

---

### Paso 4 — Crear la solicitud de certificado de la CA intermedia

La autoridad intermedia necesitará un certificado firmado por la autoridad raíz.

Genera una **CSR (Certificate Signing Request)**.

```bash
openssl req -new \
-key private/intermediate.key \
-out csr/intermediate.csr
```

Durante el proceso se solicitarán varios campos.

Cuando aparezca **Common Name**, introduce por ejemplo:

```
Training Intermediate CA
```

Comprueba que el archivo CSR se ha creado.

```bash
ls csr
```

Deberías ver:

```
intermediate.csr
```

---

### Paso 5 — Firmar la CA intermedia con la autoridad raíz

Vuelve al directorio de la autoridad raíz.

```bash
cd ~/pki-ca
```

Antes de firmar, crea un archivo de extensiones X.509 para que el certificado emitido sea realmente una **CA intermedia** (si no, en el lab 5 la verificación puede fallar con `untrusted`).

Crea el archivo `intermediate/intermediate-ext.cnf` con este contenido:

```bash
cat > intermediate/intermediate-ext.cnf <<'EOF'
[ v3_intermediate_ca ]
basicConstraints = critical, CA:true, pathlen:0
keyUsage = critical, digitalSignature, cRLSign, keyCertSign
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always,issuer
EOF
```

Utiliza la CA raíz para firmar el CSR de la autoridad intermedia **aplicando estas extensiones**.

```bash
openssl x509 -req \
-in intermediate/csr/intermediate.csr \
-CA ca.crt \
-CAkey private/ca.key \
-CAcreateserial \
-out intermediate/intermediate.crt \
-days 1825 \
-sha256 \
-extfile intermediate/intermediate-ext.cnf \
-extensions v3_intermediate_ca
```

Este comando utiliza el certificado raíz y su clave privada para emitir el certificado de la autoridad intermedia.

Comprueba que el certificado se ha generado.

```bash
ls intermediate
```

Deberías ver:

```
intermediate.crt
```

---

### Paso 6 — Verificar la relación entre la CA raíz y la CA intermedia

Verifica el certificado de la autoridad intermedia utilizando el certificado raíz.

```bash
openssl verify -CAfile ca.crt intermediate/intermediate.crt
```

La salida debería indicar:

```
intermediate/intermediate.crt: OK
```

Esto confirma que la autoridad intermedia está correctamente firmada por la autoridad raíz.

A partir de este punto podremos utilizar la **CA intermedia para firmar certificados de servidores**, manteniendo la CA raíz reservada únicamente para firmar autoridades intermedias.
