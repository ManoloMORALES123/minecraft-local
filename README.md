# Servidor Local de Minecraft (Dockerizado)

Proyecto de infraestructura para levantar un servidor de Minecraft Vanilla en un entorno local utilizando contenedores Docker.

## 🧰 Requisitos

- Docker Engine o Docker Desktop con el comando `docker compose`.
- WSL2 recomendado si el proyecto se ejecuta desde Windows.
- Puerto local `25565/TCP` disponible.
- Al menos 2 GB de memoria disponibles para el contenedor.

## 🏗️ Arquitectura y Decisiones Técnicas

1. **Imagen base (`itzg/minecraft-server`)**
   - Se utiliza un contenedor en lugar de instalar Java y los binarios del servidor directamente en el host. Esto aísla las dependencias y facilita las actualizaciones.

2. **Entorno de ejecución (WSL)**
   - El proyecto debe clonarse y ejecutarse en el sistema de archivos nativo de Linux, por ejemplo `~/minecraft`, no dentro de `/mnt/c/`.
   - El acceso desde una partición montada de Windows puede tener peor rendimiento de I/O al escribir los chunks del mundo.

3. **Mapeo de permisos (UID/GID 1000)**
   - El contenedor usa los identificadores del usuario de WSL para evitar que los archivos persistentes sean creados como `root`.
   - Esto permite administrar configuraciones y datos locales sin usar `sudo`.

4. **Persistencia**
   - El volumen `./data:/data` conserva el mundo, jugadores, logs y configuración generada.
   - `data/` debe incluirse en `.gitignore`: contiene archivos binarios, logs y `server.properties`, que puede incluir una contraseña RCON. Estos datos no deben enviarse a GitHub.

5. **Configuración declarativa**
   - Las variables de juego se definen en `docker-compose.yml`; la imagen las aplica al generar o actualizar `server.properties`.
   - Compose es la configuración de infraestructura, mientras que `data/server.properties` refleja el estado generado del servidor.

> **Nota de versión:** la configuración actual usa `VERSION: "LATEST"`. Puede descargar una versión nueva al recrear el contenedor. Para conseguir despliegues reproducibles conviene fijar una versión concreta una vez elegida.

## 🚀 Despliegue

1. Abre una terminal en el directorio raíz del proyecto.
2. Levanta el contenedor en segundo plano:

   ```bash
   docker compose up -d
   ```

3. Comprueba que el contenedor esté iniciado:

   ```bash
   docker compose ps
   ```

## 📊 Monitoreo y Administración

### Ver logs en tiempo real

```bash
docker compose logs -f
```

Presiona `Ctrl + C` para salir del flujo de logs; esto no detendrá el servidor.

### Consultar recursos

```bash
docker stats minecraft-local
```

`docker compose ps` muestra el estado del contenedor. La configuración actual no define un `healthcheck`, por lo que ese comando no confirma por sí solo que Minecraft esté listo para aceptar jugadores.

### Acceder a la consola

```bash
docker exec -i minecraft-local rcon-cli
```

## 🎮 Cómo Conectarse

### Opción A: modo offline (configuración actual)

Debido a `ONLINE_MODE: "FALSE"`, el servidor no valida licencias ni cuentas de Microsoft:

1. Abre SKlauncher en Windows.
2. Inicia sesión seleccionando el modo **Offline** con el nombre de usuario elegido.
3. Inicia el juego.
4. En Minecraft, abre **Multijugador** y usa **Conexión Directa** o **Añadir servidor**.
5. Conéctate a `127.0.0.1:25565`.

El puerto `25565` se expone desde WSL2 a `localhost` en Windows. No se requieren reglas adicionales de reenvío para una conexión local.

> ⚠️ **Seguridad:** el modo offline no verifica identidades. Cualquiera puede conectarse usando cualquier nombre si obtiene acceso al servidor. Úsalo únicamente en una red local controlada y no publiques el puerto hacia Internet.

### Opción B: modo online (cuenta oficial)

Si tienes una cuenta oficial de Microsoft:

1. Cambia la variable en `docker-compose.yml`:

   ```yaml
   ONLINE_MODE: "TRUE"
   ```

2. Recrea el contenedor:

   ```bash
   docker compose down
   docker compose up -d
   ```

3. Abre el Minecraft Launcher oficial.
4. Inicia sesión con una cuenta que tenga Minecraft Java Edition.
5. Conéctate a `127.0.0.1:25565`.

> ⚠️ **Nota:** al cambiar entre modos offline y online, Minecraft puede asignar UUID diferentes al usuario. El progreso se conserva en `./data/world/players/`, pero puede aparecer separado según el modo y el UUID utilizado. No cambies de modo sin hacer antes una copia de seguridad del mundo.

## 🛑 Detener el Servidor

Para detener el contenedor de forma segura y guardar los fragmentos del mundo:

```bash
docker compose down
```

Este comando conserva los datos almacenados en `./data`.

## 💾 Copia de Seguridad

Antes de cambiar versiones, reemplazar el mundo o actualizar Docker, crea una copia del directorio `data/`:

```bash
cp -a data "backup-data-$(date +%Y%m%d-%H%M%S)"
```

Detén el servidor antes de copiar el mundo para obtener una copia consistente.

```bash
docker compose down
```

## 📦 Preparación para GitHub

Antes de crear el repositorio o subir cambios:

1. Asegúrate de tener `data/` dentro de `.gitignore`.
2. No hagas seguimiento de mundos, logs, jugadores, archivos RCON ni `server.properties`.
3. No publiques contraseñas ni secretos presentes en `data/`.
4. Conserva una copia local independiente de `data/` para restaurar el mundo.
5. Revisa los archivos añadidos antes del primer commit:

   ```bash
   git status
   git diff --stat
   ```
