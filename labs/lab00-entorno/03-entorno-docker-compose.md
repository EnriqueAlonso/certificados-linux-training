# Laboratorio 00 — Entorno con Docker Compose

En el curso se usan contenedores Docker para levantar servicios (servidores web, correo, etc.) donde aplicar certificados y practicar sin tocar el sistema base del Codespace.

Para que sea **reproducible** y **fácil de usar**, en lugar de lanzar contenedores sueltos con `docker run`, se sigue esta convención con **Docker Compose**.

---

## Convención del curso

### 1. Carpeta única para escenarios Docker

En la raíz del repositorio hay una carpeta **`docker/`**. Por defecto solo contiene un README; **tú vas creando** dentro de ella **subcarpetas por escenario** (y sus `docker-compose.yml`) siguiendo las instrucciones de cada lab.

Estructura orientativa (las subcarpetas las crearás cuando llegues al lab correspondiente):

```text
docker/
├── README.md
├── lab08-nginx/          # escenario para practicar HTTPS con NGINX
│   ├── docker-compose.yml
│   ├── certs/            # certificados (mapeados como volumen)
│   └── config/           # configuración del servicio
├── lab08-apache/
│   ├── docker-compose.yml
│   ├── certs/
│   └── config/
├── lab10-ansible/        # escenario para rotación de certificados con Ansible
│   ├── docker-compose.yml
│   ├── certs/
│   └── ...
└── ...
```

Cada subcarpeta tiene su propio `docker-compose.yml` y, si aplica, carpetas para **configs** y **certificados** que se montan como volúmenes.

### 2. Volúmenes: editar en el host, sin entrar al contenedor

Los archivos que hay que editar (configuración, certificados, claves) se dejan en **carpetas del host** (dentro de `docker/<escenario>/`) y se montan en el contenedor con **volúmenes** en el `docker-compose.yml`.

Así puedes:

* Editar con tu editor en el Codespace (en `docker/lab10-ansible/certs/`, etc.).
* Los contenedores ven esos mismos archivos montados; no hace falta copiarlos ni hacer `docker cp`.

Ejemplo de volumen en `docker-compose.yml`:

```yaml
volumes:
  - ./certs:/etc/ssl/local:ro
  - ./config/nginx.conf:/etc/nginx/nginx.conf:ro
```

### 3. Servicios en segundo plano y “attach” cuando haga falta

* Los servicios se levantan en **segundo plano** con:

  ```bash
  docker compose up -d
  ```

* Cuando necesites **ejecutar comandos dentro de un contenedor** (revisar config, reiniciar un servicio, etc.), usas:

  ```bash
  docker compose exec <nombre_del_servicio> bash
  ```

  (o `sh` si la imagen no tiene bash). Salir con `exit`.

* Para parar el escenario:

  ```bash
  docker compose down
  ```

### 4. Resumen del flujo

1. Entrar en la subcarpeta del escenario: `cd docker/lab10-ansible` (o la que toque).
2. Crear las carpetas que use el compose (p. ej. `certs/`, `config/`) si no existen.
3. Poner en esas carpetas los archivos que quieras (certs, configs) **editando en el host**.
4. Levantar servicios: `docker compose up -d`.
5. Cuando necesites terminal en un contenedor: `docker compose exec <servicio> bash`.
6. Al terminar: `docker compose down`.

Con esta convención puedes **reproducir** cada escenario con un solo `docker compose up -d` y trabajar cómodamente editando en el repo y usando "attach shell" solo cuando haga falta.

**Nota:** En el **Lab08** (TLS en servicios), los ejercicios de NGINX e inspección TLS usan un directorio en tu home (**`~/tls-web`**) y `docker run` directamente, para mantener los pasos más sencillos. Los archivos se crean en el host y se montan en el contenedor con `-v`. En el Lab10 (Ansible) y en escenarios que tú crees, se usa la convención anterior: carpeta `docker/<escenario>/` y Docker Compose.
