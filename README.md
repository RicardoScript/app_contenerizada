# CRUD Docker Demo

API REST mínima de **tareas** (CRUD) hecha con **Node.js + Express** y **PostgreSQL**, ejecutada con **Docker Compose**. Este README permite que cualquier persona la levante desde cero en pocos minutos.

## 1. Requisitos

| Herramienta | Versión recomendada | Cómo comprobarlo |
|---|---|---|
| Docker Engine / Docker Desktop | 24 o superior | `docker --version` |
| Docker Compose (plugin v2) | 2.20 o superior | `docker compose version` |
| Git | cualquiera reciente | `git --version` |
| curl (para probar la API) | cualquiera | `curl --version` |

No necesitas instalar Node.js ni PostgreSQL: todo corre dentro de contenedores.

Puertos usados en tu equipo: `3000` (API, configurable con `APP_PORT`).

### Instalación de Docker en Fedora

```bash
sudo dnf -y install dnf-plugins-core
sudo dnf config-manager addrepo --from-repofile=https://download.docker.com/linux/fedora/docker-ce.repo
sudo dnf -y install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo systemctl enable --now docker
sudo usermod -aG docker $USER      # cierra sesión y vuelve a entrar para aplicarlo
docker run --rm hello-world        # prueba rápida
```

> En Fedora SELinux viene activo. El volumen de `init.sql` usa la opción `:z` en `docker-compose.yml` para que SELinux permita leerlo. No la quites.

## 2. Estructura del proyecto

```
crud-docker-demo/
├── app/
│   ├── Dockerfile        # Imagen de la API (node:20-alpine)
│   ├── package.json      # Dependencias: express, pg
│   └── server.js         # API con las rutas CRUD
├── db/
│   └── init.sql          # Crea la tabla "tareas" e inserta 2 filas de ejemplo
├── docker-compose.yml    # Orquesta los servicios "db" y "app"
├── .env.example          # Plantilla de variables de entorno (sin credenciales reales)
├── .gitignore            # Evita subir .env y node_modules
└── README.md             # Este documento
```

## 3. Preparación de variables de entorno

1. Clona el repositorio y entra a la carpeta:

   ```bash
   git clone <URL_DEL_REPOSITORIO>
   cd crud-docker-demo
   ```

2. Copia la plantilla y edítala:

   ```bash
   cp .env.example .env
   ```

3. Ajusta los valores de `.env`:

   | Variable | Descripción | Ejemplo |
   |---|---|---|
   | `DB_USER` | Usuario de PostgreSQL | `usuario_demo` |
   | `DB_PASSWORD` | Contraseña de PostgreSQL | `una_clave_segura` |
   | `DB_NAME` | Nombre de la base de datos | `tareas_db` |
   | `APP_PORT` | Puerto del equipo donde se publica la API | `3000` |

> El archivo `.env` real **nunca** se sube al repositorio (está en `.gitignore`). Solo se versiona `.env.example`.

## 4. Construcción, inicio y detención

```bash
# Construir las imágenes y levantar todo en segundo plano
docker compose up -d --build

# Ver el estado de los contenedores (db debe quedar "healthy")
docker compose ps

# Ver los logs de la API
docker compose logs -f app

# Detener los contenedores (conserva los datos)
docker compose down

# Detener y borrar también los datos (volumen de la base de datos)
docker compose down -v
```

## 5. Inicialización de la base de datos

La base de datos se inicializa **automáticamente**: el archivo `db/init.sql` se monta en `/docker-entrypoint-initdb.d/` y PostgreSQL lo ejecuta la **primera vez** que se crea el volumen `db_data`. Crea la tabla `tareas` e inserta dos filas de ejemplo.

Para volver a ejecutarlo desde cero (por ejemplo, si cambias `init.sql`):

```bash
docker compose down -v
docker compose up -d --build
```

Para revisar el contenido de la tabla directamente:

```bash
docker compose exec db psql -U usuario_demo -d tareas_db -c "SELECT * FROM tareas;"
```

(Cambia `usuario_demo` y `tareas_db` por los valores de tu `.env`.)

## 6. Ejemplos de solicitudes al CRUD

Con la API arriba (reemplaza `3000` si cambiaste `APP_PORT`):

```bash
# Comprobar que la API y la base de datos responden
curl http://localhost:3000/health

# CREATE - crear una tarea
curl -X POST http://localhost:3000/tareas \
  -H "Content-Type: application/json" \
  -d '{"titulo": "Estudiar Docker"}'

# READ - listar todas las tareas
curl http://localhost:3000/tareas

# READ - obtener una tarea por id
curl http://localhost:3000/tareas/1

# UPDATE - modificar título y/o estado
curl -X PUT http://localhost:3000/tareas/1 \
  -H "Content-Type: application/json" \
  -d '{"titulo": "Estudiar Docker Compose", "completada": true}'

# DELETE - eliminar una tarea (responde 204 sin cuerpo)
curl -X DELETE http://localhost:3000/tareas/1
```

Respuesta esperada al crear (código 201):

```json
{"id": 3, "titulo": "Estudiar Docker", "completada": false, "creada_en": "2026-11-11T23:30:00.000Z"}
```

> En Windows PowerShell, usa `curl.exe` en lugar de `curl`, y comillas dobles escapadas en el JSON, o ejecuta los comandos desde Git Bash / WSL.

## 7. Errores frecuentes, causas y soluciones

### Error 1: `Bind for 0.0.0.0:3000 failed: port is already allocated`
- **Causa:** otro programa o contenedor ya usa el puerto 3000 de tu equipo.
- **Solución:** cambia `APP_PORT` en `.env` (por ejemplo `APP_PORT=3001`) y ejecuta `docker compose up -d` de nuevo. Para ver quién usa el puerto: `docker ps` o `lsof -i :3000` (Linux/Mac) / `netstat -ano | findstr :3000` (Windows).

### Error 2: `password authentication failed for user "..."` (la API no conecta a la base)
- **Causa:** cambiaste `DB_USER` o `DB_PASSWORD` en `.env` **después** de haber creado el volumen. PostgreSQL solo aplica esas credenciales la primera vez que crea la base.
- **Solución:** borra el volumen y recrea todo: `docker compose down -v && docker compose up -d --build`. (Esto elimina los datos guardados.)

### Error 3: `variable is not set. Defaulting to a blank string` o `invalid reference format`
- **Causa:** no existe el archivo `.env` (solo está `.env.example`) o tiene variables vacías.
- **Solución:** ejecuta `cp .env.example .env`, completa todos los valores y vuelve a levantar con `docker compose up -d --build`.

### Error 4: `relation "tareas" does not exist`
- **Causa:** el volumen ya existía antes de agregar o modificar `db/init.sql`, por lo que el script no se volvió a ejecutar.
- **Solución:** `docker compose down -v && docker compose up -d --build` para reinicializar la base.

### Error 5: `Cannot connect to the Docker daemon` / `failed to connect to the docker API`
- **Causa:** Docker Desktop o el servicio de Docker no está iniciado.
- **Solución:** abre Docker Desktop y espera a que indique que está en ejecución; en Linux: `sudo systemctl start docker`.

### Error 6 (Fedora): `permission denied` al leer `init.sql` o `permission denied while trying to connect to the Docker daemon socket`
- **Causa:** (a) SELinux bloquea que el contenedor lea archivos montados sin etiqueta; (b) tu usuario no pertenece al grupo `docker`.
- **Solución:** (a) mantén la opción `:z` en el volumen de `init.sql` dentro de `docker-compose.yml`; (b) ejecuta `sudo usermod -aG docker $USER`, cierra sesión y vuelve a entrar. Como alternativa temporal puedes anteponer `sudo` a los comandos `docker`.

## 8. Verificación desde una copia limpia

Para comprobar que las instrucciones funcionan sin depender de archivos locales:

```bash
cd /tmp
git clone <URL_DEL_REPOSITORIO> prueba-limpia
cd prueba-limpia
cp .env.example .env        # edita las credenciales si lo deseas
docker compose up -d --build
docker compose ps
curl http://localhost:3000/health
curl http://localhost:3000/tareas
docker compose down -v
```
