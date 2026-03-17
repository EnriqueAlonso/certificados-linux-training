## Lab 04 — Uso de certtool (GnuTLS)

En este ejercicio utilizarás **certtool**, la herramienta de línea de comandos de **GnuTLS**, para generar claves y certificados.

El objetivo es comparar el flujo de trabajo con `openssl` y comprender cómo trabajar con entornos que utilizan GnuTLS.

---

> **Nota sobre instalación**  
> Si al ejecutar `certtool` obtienes un error de comando no encontrado, instala el paquete correspondiente a GnuTLS en tu distribución.  
> - En Debian/Ubuntu: `sudo apt install gnutls-bin`  
> - En Fedora/RHEL/CentOS: `sudo dnf install gnutls-utils` (o `yum install gnutls-utils`)  
> - En Arch/Manjaro: `sudo pacman -S gnutls`

---

### 1. Generar una clave privada con certtool

1. Crea un directorio de trabajo:

```bash
mkdir -p ~/labs/certtool
cd ~/labs/certtool
```

2. Genera una clave privada RSA:

```bash
certtool --generate-privkey --outfile server-key.pem
```

3. Inspecciona el contenido del archivo:

```bash
openssl pkey -in server-key.pem -text -noout | less
```

Compara el resultado con una clave generada anteriormente con `openssl genpkey` o `openssl genrsa`.

---

### 2. Crear una plantilla de certificado

`certtool` utiliza archivos de plantilla para definir los campos del certificado.

1. Crea un archivo `server.cfg` con contenido similar a:

```bash
cat > server.cfg << 'EOF'
organization = "Laboratorio PKI"
cn = "server.lab.local"
tls_www_server
encryption_key
signing_key
EOF
```

2. Revisa los campos y relaciónalos con los que utilizaste al crear certificados con `openssl req`.

---

### 3. Generar un certificado autofirmado con certtool

1. Genera un certificado autofirmado usando la clave y la plantilla:

```bash
certtool --generate-self-signed \
  --load-privkey server-key.pem \
  --template server.cfg \
  --outfile server-cert.pem
```

2. Analiza el certificado resultante:

```bash
openssl x509 -in server-cert.pem -noout -text | less
```

3. Comprueba:

* el `Subject`
* las extensiones (`Key Usage`, `Extended Key Usage`)
* el período de validez.

---

### 4. Comparar flujos certtool vs openssl

Responde a las siguientes preguntas:

1. ¿Qué diferencias observas entre:
   * generar un certificado con `openssl req -x509`
   * generar un certificado con `certtool --generate-self-signed`?
2. ¿Qué ventajas tiene el uso de plantillas (`server.cfg`)?
3. ¿En qué tipos de sistemas o distribuciones es más probable encontrar `certtool` como herramienta principal?

Puedes anotar tus conclusiones en un archivo `notas-certtool.md` dentro del mismo directorio.

---

### 5. Uso de certtool junto a la PKI del curso

De forma opcional, intenta:

1. Generar una CSR con `certtool` usando tu clave:

```bash
certtool --generate-request \
  --load-privkey server-key.pem \
  --template server.cfg \
  --outfile server.csr
```

2. Firmar esa CSR con tu **Intermediate CA** utilizando `openssl`, igual que en el laboratorio de emisión de certificados.

Esto te permitirá ver cómo se integran herramientas diferentes (`certtool` y `openssl`) dentro de una misma PKI.

---

### 6. Conclusiones

Al finalizar este ejercicio deberías ser capaz de:

* generar claves y certificados con `certtool`
* utilizar plantillas para definir los campos de un certificado
* integrar certificados generados con `certtool` en la PKI creada con `openssl`.

