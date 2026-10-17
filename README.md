# Informe de práctica — Docker (Parte 2)

**Curso:** Arquitectura de Software
**Taller:** 2do Corte — Docker, Parte 2 (Práctica)
**Estudiante:** Daniel de Jesús Carbal Castro
**Proyecto:** biblioteca-api (https://github.com/carrenoutb/biblioteca-api)
**Fecha:** 16 de octubre de 2026

---

## Introducción

En esta práctica seguí la guía del `README.md` del proyecto **biblioteca-api**, una API hecha en Go con Gin y MySQL que usa una arquitectura por capas (`handler` - `service` - `repository`). El proyecto ya viene dockerizado, así que el objetivo fue levantar la API y la base de datos solo con Docker, sin instalar Go ni MySQL en mi computador, y comprobar que todo funciona haciendo peticiones reales a la API.

El informe está organizado en tres fases, igual que la guía:

1. **Fase 1:** instalar Docker y verificar que funciona.
2. **Fase 2:** poner en marcha el proyecto (clonar, crear el `.env` y levantar los contenedores).
3. **Fase 3:** verificar que todo funciona (estado de los servicios, Swagger y el flujo register → login → books).

Al final respondo las 5 preguntas de comprensión y dejo la lista de evidencias.

> Las capturas van en la carpeta `evidencias/`, junto a este archivo. En cada paso indico el nombre del archivo de la captura correspondiente.

---

## Fase 1 — Instalación de Docker

### 1.1 Instalar Docker

Dependiendo del sistema operativo, la guía indica dos caminos:

- **Windows y macOS:** instalar [Docker Desktop](https://www.docker.com/products/docker-desktop/). Es un instalador que trae el motor de Docker, Docker Compose y una interfaz gráfica. Después de instalarlo hay que abrirlo y esperar a que el icono muestre que está corriendo. En Windows, es posible que pida activar WSL 2; si lo pide, se acepta y se reinicia el equipo.
- **Linux (Debian/Ubuntu):** instalar Docker Engine y el plugin de Compose desde la terminal:

```bash
sudo apt update
sudo apt install -y docker.io docker-compose-plugin
sudo systemctl enable --now docker
```

**Sistema operativo que usé:** Windows

### 1.2 Verificar la instalación

Para comprobar que Docker y Compose quedaron instalados ejecuté:

```bash
docker --version
docker compose version
```

La salida esperada se ve parecida a esto (las versiones pueden cambiar):

```
Docker version 29.5.2, build ...
Docker Compose version 5.5.1
```

**Evidencia:**

![](https://i.imgur.com/jibtu0A.jpeg)

---

## Fase 2 — Puesta en marcha

### 2.1 Clonar el repositorio

Descargué el código del proyecto y entré a la carpeta:

```bash
git clone https://github.com/carrenoutb/biblioteca-api.git
cd biblioteca-api
```

**Evidencia:**

![Clonado del repositorio](https://i.imgur.com/YapgKEx.jpeg)

### 2.2 Entender las piezas del proyecto

Antes de ejecutar nada, revisé los archivos principales del proyecto para entender qué aporta cada uno:

| Archivo | Qué aporta |
|---|---|
| `Dockerfile` | Es la receta para construir la imagen de la API. Parte de una imagen base de Go, descarga las dependencias, compila el programa y deja lista la aplicación. Es *multi-stage*: un paso sirve para compilar y otro, más liviano, para ejecutar, así la imagen final queda más pequeña. |
| `docker-compose.yml` | Define que la aplicación son dos contenedores, `mysql` (la base de datos) y `api` (nuestra API). También los conecta en una red propia, expone los puertos y lanza las migraciones automáticamente. |
| `.env.example` → `.env` | Contiene las variables de configuración (usuario y clave de la base de datos, secreto del token, etc.). Se copia a `.env` para no dejar datos sensibles escritos en el código. |
| `migrations/` | Son los scripts SQL que crean las tablas y cargan datos de ejemplo. MySQL los ejecuta solo la primera vez que arranca con la base de datos vacía. |
| `docs/` | Es la documentación de Swagger, que ya viene generada. |

Relacionándolo con la Parte 1: el `Dockerfile` crea la **imagen** de la API, `docker compose` ejecuta esa imagen junto con MySQL como **contenedores**, y las imágenes base (`golang` y `mysql`) se descargan del **registro** (Docker Hub).

### 2.3 Crear el archivo `.env`

El repositorio no trae un `.env`, solo una plantilla llamada `.env.example`. Esto es así porque el `.env` tiene datos sensibles (la contraseña de la base de datos y la clave `JWT_SECRET`) y no se debe subir a un repositorio público. Por eso el `.gitignore` lo excluye y yo debo crearlo en mi máquina copiando la plantilla:

```bash
cp .env.example .env
```

> En Windows con PowerShell o Git Bash el comando `cp` funciona igual. Si se usa el CMD de Windows, el equivalente es `copy .env.example .env`.

Este paso es obligatorio: sin el `.env`, `docker compose up --build` falla, porque el compose está configurado con `env_file: .env`.

**Evidencia:**

![Creación del archivo .env](https://i.imgur.com/QtQkHeP.jpeg)

#### Explicación de cada variable

| Variable | Valor de ejemplo | Qué es |
|---|---|---|
| `DB_HOST` | `mysql` | Dónde está la base de datos. No es una dirección de internet, sino el nombre del servicio de MySQL dentro de la red de Docker. |
| `DB_PORT` | `3306` | Puerto en el que escucha MySQL (el puerto estándar de MySQL). |
| `DB_USER` | `biblioteca_user` | Usuario con el que la API entra a la base de datos. |
| `DB_PASSWORD` | `biblioteca_pass` | Contraseña de ese usuario de la base de datos. |
| `DB_NAME` | `biblioteca` | Nombre de la base de datos que usa la API. |
| `JWT_SECRET` | `TOKEN_BIBLIOTECA` | Clave secreta con la que la API firma los tokens de sesión (JWT). Si alguien la conoce, podría fabricar tokens falsos, por eso se debe cambiar en un proyecto real. |
| `JWT_EXPIRATION_HOURS` | `24` | Cuántas horas dura un token antes de vencerse. |
| `APP_PORT` | `8080` | Puerto en el que escucha la API. |

### 2.4 Levantar todo con Docker Compose

Con el `.env` listo, levanté los dos contenedores con un solo comando:

```bash
docker compose up --build
```

Lo que pasa por dentro, en orden, es lo siguiente:

1. Docker lee el `Dockerfile` y **construye la imagen de la API**: descarga la imagen base de Go desde Docker Hub, ejecuta `go mod download` y luego `go build`.
2. Descarga la imagen de **MySQL 8**.
3. Arranca **MySQL primero** y espera a que esté "sano" (healthy). Para eso hace un *healthcheck* que le "hace ping" a MySQL cada 5 segundos.
4. Cuando MySQL está sano, se ejecutan **automáticamente** las migraciones (`migrations/001_init.sql`) y los datos de ejemplo (seed) sobre la base de datos vacía.
5. Arranca la **API** en el puerto `8080`.

La primera vez se demora unos minutos porque tiene que descargar las imágenes. Si se quiere dejar corriendo en segundo plano se puede usar `docker compose up -d`, y para detenerlo sin borrar nada se usa `Ctrl + C`.

**Evidencia del inicio (descarga y construcción):**

![Inicio de docker compose up --build](https://i.imgur.com/KXsNH0B.jpeg)

**Evidencia del final (ambos servicios corriendo):**

![Final de docker compose up --build](https://i.imgur.com/8PJq5IK.jpeg)

---

## Fase 3 — Verificación

### 3.1 Estado de los servicios

En otra terminal, dentro de la carpeta del proyecto, ejecuté:

```bash
docker compose ps
```

Lo esperado es que los dos contenedores aparezcan en estado `Up` y que MySQL aparezca además como `(healthy)`.

**Evidencia:**

![Salida de docker compose ps](https://i.imgur.com/r7Znqlt.jpeg)

### 3.2 Documentación interactiva (Swagger)

Abrí en el navegador la documentación de la API:

```
http://localhost:8080/swagger/index.html
```

Si carga la página de Swagger con la lista de endpoints, significa que la API está arriba y respondiendo.

**Evidencia:**

![Página de Swagger funcionando](https://i.imgur.com/GTi1REI.jpeg)

### 3.3 Flujo completo: registrar → iniciar sesión → usar el token → listar libros

Para probar la API de verdad usé la extensión **REST Client** de VS Code. También se puede hacer con Swagger UI o con `curl` (al final dejo la versión con `curl`).

#### Paso 1: instalar la extensión

1. Abrir VS Code y entrar a las extensiones con `Ctrl + Shift + X` (`Cmd + Shift + X` en Mac).
2. Buscar **REST Client** (autor: Huachao Mao) e instalarla.
3. Con eso, cualquier archivo `.http` muestra un botón **Send Request** sobre cada petición.

#### Paso 2: crear el archivo de pruebas

Creé en la raíz del proyecto un archivo llamado `pruebas-api.http` con este contenido:

```http
# Dirección base de la API
@baseUrl = http://localhost:8080

### 1. Estado de la API
GET {{baseUrl}}/health

### 2. Registrar un usuario nuevo
# @name register
POST {{baseUrl}}/api/auth/register
Content-Type: application/json

{
  "name": "estudiante-taller",
  "email": "taller-practica@example.com",
  "password": "pass@123"
}

### 3. Iniciar sesión (obtener el token)
# @name login
POST {{baseUrl}}/api/auth/login
Content-Type: application/json

{
  "email": "taller-practica@example.com",
  "password": "pass@123"
}

### 4. Listar libros (usa el token del login automáticamente)
GET {{baseUrl}}/api/books
Authorization: Bearer {{login.response.body.token}}
```

Algunas cosas de la sintaxis que me sirvieron para entenderlo: `###` separa una petición de otra, `#` es un comentario, `@baseUrl` define una variable que se usa como `{{baseUrl}}`, y `# @name login` le pone nombre a la petición para poder usar su respuesta después. Gracias a eso, `{{login.response.body.token}}` toma el campo `token` de la respuesta del login y lo pone en la cabecera `Authorization`, sin tener que copiarlo a mano.

#### Paso 3: probar cada petición

**Petición 1: estado de la API (health).** Hice clic en **Send Request**. La respuesta esperada es:

```json
{"status":"ok"}
```

Esto comprueba que el contenedor `api` está vivo y respondiendo.

![Respuesta de /health](https://i.imgur.com/4Z9JqYl.jpeg)

**Petición 2: registrar un usuario.** La respuesta esperada incluye un token y los datos del usuario nuevo, con rol `student` (el `id` y la fecha pueden variar):

```json
{"token":"eyJhbGciOi...","user":{"id":1,"name":"estudiante-taller","email":"taller-practica@example.com","role":"student","created_at":"..."}}
```

Esto significa que la API guardó el usuario en MySQL y devolvió su token.

![Respuesta de register](https://i.imgur.com/UeYkdwu.jpeg)

**Petición 3: iniciar sesión.** La respuesta esperada es parecida a la del registro: un token y los datos del usuario.

```json
{"token":"eyJhbGciOi...","user":{"id":1,"name":"estudiante-taller","email":"taller-practica@example.com","role":"student",...}}
```

Esto confirma que la API verifica el correo y la contraseña (guardada con bcrypt) y entrega un token válido.

![Respuesta de login](https://i.imgur.com/G6IBozU.jpeg)

**Petición 4: listar libros con el token.** Antes de enviarla hay que haber ejecutado el login, para que la variable del token exista. La respuesta esperada son los libros que cargó el seed:

```json
[
  {"id":1,"title":"Cien años de soledad","author":"Gabriel García Márquez","isbn":"9780307474728","category":"Novela","total_copies":3,"available_copies":3},
  {"id":2,"title":"Clean Code","author":"Robert C. Martin","isbn":"9780132350884","category":"Tecnología","total_copies":2,"available_copies":2}
]
```

Si aparece esta lista, quiere decir que la API, el token y la base de datos funcionan juntos.

![Respuesta de books](https://i.imgur.com/2PMLrNG.jpeg)

#### Alternativa con `curl`

Si no se quiere usar VS Code, el mismo flujo se puede probar desde la terminal (en Windows PowerShell conviene escribir `curl.exe` en lugar de `curl`):

```bash
# 1. Health
curl http://localhost:8080/health

# 2. Registrar usuario
curl -X POST http://localhost:8080/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{"name":"estudiante-taller","email":"taller-practica@example.com","password":"pass@123"}'

# 3. Iniciar sesión (copiar el valor del campo "token" de la respuesta)
curl -X POST http://localhost:8080/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"taller-practica@example.com","password":"pass@123"}'

# 4. Listar libros (reemplazar TU_TOKEN por el token del paso anterior)
curl http://localhost:8080/api/books -H "Authorization: Bearer TU_TOKEN"
```

---

## Problemas comunes y cómo resolverlos

Revisando los anexos del `README.md`, estos son problemas que pueden aparecer y su solución:

1. **Error `port is already allocated` (puerto 3306 u 8080 ocupado).** Pasa cuando otra aplicación del computador ya está usando ese puerto, por ejemplo un MySQL instalado localmente. La solución es cerrar esa aplicación o cambiar el mapeo de puertos en `docker-compose.yml` (por ejemplo `"3307:3306"` para MySQL o `"8081:8080"` para la API).
2. **La API da error de conexión con MySQL.** Para diagnosticarlo se revisan los logs con `docker compose logs api` y `docker compose logs mysql`, se confirma que MySQL esté `(healthy)` antes de que arranque la API y se verifica que en el `.env` esté `DB_HOST=mysql`.
3. **Aparece `404` al abrir `http://localhost:8080/`.** No es un error: la API no tiene una ruta raíz. Hay que usar `/health` o `/swagger/index.html`.
4. **`docker compose up --build` falla porque no encuentra el `.env`.** Se soluciona creando el archivo con `cp .env.example .env`.

> Si durante mi práctica me apareció alguno de estos problemas, lo describo aquí: [escribir qué pasó y cómo lo resolví, o borrar esta nota].

---

## Preguntas de comprensión

### 1. ¿Qué diferencia hay entre una imagen y un contenedor? Usa un ejemplo de este proyecto.

La **imagen** es la plantilla de solo lectura que tiene todo lo necesario para ejecutar una aplicación, y el **contenedor** es esa imagen ya en ejecución. En este proyecto, el `Dockerfile` se usa para construir la imagen de la API (con Go ya compilado), y esa imagen no hace nada por sí sola: solo existe guardada. Cuando `docker compose up` la ejecuta, se crea el contenedor `api`, que es un proceso corriendo y respondiendo en el puerto 8080. Con MySQL pasa igual: la imagen `mysql:8` se descarga de Docker Hub, y el contenedor `mysql` es esa imagen funcionando como base de datos. De una misma imagen se podrían crear varios contenedores.

### 2. ¿Qué hace `docker compose` que no harías con `docker run` por separado?

Con `docker run` tendría que arrancar cada contenedor a mano, con todos sus parámetros escritos (puertos, variables, volúmenes), crear yo mismo la red para que se vean entre sí, y acordarme del orden (primero la base de datos y después la API). `docker compose` hace todo eso por mí a partir de un solo archivo, `docker-compose.yml`: crea la red, levanta los dos servicios, carga las variables del `.env`, monta el volumen de datos, espera a que MySQL esté sano con el healthcheck antes de arrancar la API y lo apaga todo con `docker compose down`. Además, el archivo queda guardado en el repositorio, así que cualquiera puede repetir exactamente lo mismo, que es la reproducibilidad del entorno que vimos en la Parte 1.

### 3. ¿Por qué la API encuentra la base de datos con `DB_HOST=mysql` y no con `127.0.0.1`?

Porque cada contenedor tiene su propia red aislada, y dentro de la API `127.0.0.1` significa "este mismo contenedor", es decir, la propia API, donde no hay ninguna base de datos. Docker Compose crea una red interna donde cada servicio se puede encontrar por su nombre, así que `mysql` es el nombre del servicio de la base de datos y Docker lo traduce a la dirección IP del contenedor de MySQL. Por eso `mysql` no es una dirección de internet, sino el nombre del servicio dentro de esa red.

### 4. ¿Qué borra exactamente `docker compose down -v` y cuándo lo usarías?

`docker compose down` detiene y elimina los contenedores y la red que creó Compose, pero conserva los datos. Con la opción `-v` además elimina el **volumen** de datos de MySQL, o sea, todo lo guardado en la base de datos (por ejemplo, los usuarios que registré). Como la base de datos queda vacía, en el siguiente `docker compose up` MySQL vuelve a ejecutar las migraciones y el seed desde cero. Lo usaría cuando quiero volver a los datos de ejemplo originales o empezar con una base de datos limpia, teniendo claro que se pierde lo que haya guardado.

### 5. ¿Qué pasaría si cambias `DB_PASSWORD` en el `.env` pero no en el `docker-compose.yml`? ¿Y al revés?

La contraseña tiene que ser la misma en los dos lados: la API la usa para conectarse y MySQL la usa para crear el usuario. Si cambio `DB_PASSWORD` en el `.env` pero en el `docker-compose.yml` (donde MySQL define su contraseña) queda la anterior, la API intentaría entrar con una contraseña que MySQL no conoce y daría error de acceso denegado (`Access denied`), por lo que la API no podría conectarse a la base de datos. Si lo hago al revés, cambiar el compose y no el `.env`, pasa lo mismo, porque la API seguiría usando la contraseña vieja. Además, MySQL solo crea el usuario con su contraseña la primera vez que arranca con el volumen vacío; si ya existía una base de datos, cambiar la contraseña después no la actualiza y habría que usar `docker compose down -v` para que se cree de nuevo. (Para confirmar cómo se pasa la contraseña a MySQL, conviene mirar el `docker-compose.yml` del proyecto.)

---

## Conclusión

Con esta práctica pude comprobar lo que investigué en la Parte 1. Con solo tres cosas (copiar el `.env`, ejecutar `docker compose up --build` y abrir el navegador) pasé de un computador sin Go ni MySQL a tener la API y su base de datos funcionando, con datos de ejemplo y todo. Entender la cadena Dockerfile → imagen → contenedor, y cómo Compose conecta los servicios por su nombre dentro de una red, me ayudó a ver por qué Docker es tan útil en una arquitectura de software: cada pieza (API y base de datos) corre en su propio contenedor, el entorno se puede repetir en cualquier máquina y cualquier persona del equipo puede ponerlo a funcionar rápido.

---

## Lista de evidencias (carpeta `evidencias/`)

| Fase | Archivo | Qué muestra |
|---|---|---|
| 1 | `https://i.imgur.com/jibtu0A.jpeg` | Salida de `docker --version` y `docker compose version` |
| 2 | `https://i.imgur.com/YapgKEx.jpeg` | Clonado del repositorio |
| 2 | `https://i.imgur.com/QtQkHeP.jpeg` | Creación del `.env` con `cp .env.example .env` |
| 2 | `https://i.imgur.com/KXsNH0B.jpeg` | Inicio de `docker compose up --build` (descarga y construcción) |
| 2 | `https://i.imgur.com/8PJq5IK.jpeg` | Final de `docker compose up --build` con ambos servicios corriendo |
| 3 | `https://i.imgur.com/r7Znqlt.jpeg` | Salida de `docker compose ps` (ambos `Up`, MySQL `healthy`) |
| 3 | `https://i.imgur.com/GTi1REI.jpeg` | Swagger abierto en el navegador |
| 3 | `https://i.imgur.com/4Z9JqYl.jpeg` | Respuesta de `/health` |
| 3 | `https://i.imgur.com/UeYkdwu.jpeg` | Respuesta de registrar usuario |
| 3 | `https://i.imgur.com/G6IBozU.jpeg` | Respuesta de iniciar sesión |
| 3 | `https://i.imgur.com/2PMLrNG.jpeg` | Respuesta de listar libros con el token |
