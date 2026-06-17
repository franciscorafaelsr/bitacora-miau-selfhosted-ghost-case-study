# Operaciones y Mantenimiento

Este documento describe las tareas habituales de administración, respaldo y mantenimiento del stack autohospedado de **Bitácora Miau**.

## Comandos de Control del Stack

Todas las operaciones del ciclo de vida se ejecutan utilizando Docker Compose en el directorio donde se ubica el archivo descriptivo `docker-compose.yml`:

### 1. Iniciar los Servicios
Inicia todos los contenedores en segundo plano (detached mode):
```bash
docker compose up -d
```

### 2. Detener los Servicios
Detiene la ejecución de los contenedores sin eliminar los datos persistidos en los volúmenes:
```bash
docker compose down
```

### 3. Detener y Limpiar Redes y Huérfanos
Detiene los contenedores, remueve las redes virtuales internas creadas y limpia recursos huérfanos:
```bash
docker compose down --remove-orphans
```

---

## Monitoreo y Lectura de Logs

### 1. Inspeccionar el Estado de los Contenedores
```bash
docker compose ps
```

### 2. Ver Logs en Tiempo Real
Para ver la actividad general de todos los contenedores:
```bash
docker compose logs -f --tail=100
```

Para ver la actividad específica de un solo contenedor (por ejemplo, el proxy de Mailgun):
```bash
docker compose logs -f mailgun_proxy
```

---

## Estrategia de Copias de Seguridad (Backups)

El stack persiste la información en dos lugares: la base de datos relacional (MariaDB) y los archivos cargados por el usuario (directorio `ghost_content` con imágenes, temas e integraciones).

### 1. Respaldo de la Base de Datos (Base de Datos Relacional)
Se utiliza la herramienta nativa de MariaDB para generar un volcado de base de datos sin necesidad de apagar el stack:

```bash
docker exec miau_mysql mariadb-dump \
  -u ghost \
  -p"TU_CONTRASENA_AQUI" \
  ghost > backup_ghost_db_$(date +%F).sql
```
*Reemplazar `miau_mysql` y las credenciales por los valores definidos en el archivo `.env` de producción.*

### 2. Respaldo de Archivos (Ghost Content)
El contenido cargado se almacena en el volumen asociado al contenedor de Ghost CMS. Para respaldar esta carpeta, se puede comprimir el directorio de origen en el host o realizar una copia en caliente utilizando una imagen temporal de Debian/Alpine:

```bash
docker run --rm \
  -v miau_ghost_content:/volume \
  -v $(pwd):/backup \
  alpine tar -czf /backup/backup_ghost_content_$(date +%F).tar.gz -C /volume .
```

---

## Actualización del Sistema (Upgrades)

Actualizar Ghost CMS es crucial para obtener parches de seguridad y nuevas funcionalidades. Gracias a Docker, el proceso es sencillo:

1. **Respaldar**: Generar un backup completo de la base de datos y de la carpeta de imágenes antes de proceder.
2. **Actualizar Tag de Imagen**: En el archivo `.env`, cambiar el tag de la versión de Ghost a la deseada (ej. `GHOST_IMAGE=ghost:6.41.0-alpine`).
3. **Descargar Nueva Imagen**:
   ```bash
   docker compose pull ghost
   ```
4. **Re-crear Contenedor**:
   ```bash
   docker compose up -d --no-deps ghost
   ```
   *El parámetro `--no-deps` asegura que el contenedor de la base de datos MariaDB no se reinicie si ya está operando normalmente.*
5. **Verificar**: Consultar los logs de Ghost para confirmar que las migraciones de base de datos automatizadas de Ghost concluyeron con éxito:
   ```bash
   docker compose logs ghost
   ```

---

## Procedimiento de Recuperación ante Desastres (Restore)

Si se debe restaurar el blog en un VPS completamente nuevo:

1. Clonar el repositorio de configuraciones y copiar el archivo `.env` configurado.
2. Inicializar los volúmenes vacíos e iniciar la base de datos:
   ```bash
   docker compose up -d mysql_ghost
   ```
3. Importar la base de datos SQL respaldada:
   ```bash
   docker exec -i miau_mysql mariadb -u ghost -p"TU_CONTRASENA" ghost < backup_ghost_db.sql
   ```
4. Extraer el respaldo de la carpeta `content` en el volumen de Ghost:
   ```bash
   docker run --rm \
     -v miau_ghost_content:/volume \
     -v $(pwd):/backup \
     alpine tar -xzf /backup/backup_ghost_content.tar.gz -C /volume
   ```
5. Iniciar todo el stack completo:
   ```bash
   docker compose up -d
   ```

---

## Verificación de Integridad de Envío de Newsletters (Preflight Checks)

> [!NOTE]
> **Nota de alcance**: Este script se mantiene fuera de este repositorio documental. Esta sección debe entenderse como una guía operativa de referencia y no como un componente incluido en el repositorio.

En Ghost CMS autohospedado, pueden ocurrir inconsistencias silenciosas en la base de datos que provoquen fallos o duplicados en el envío de newsletters. Por ejemplo:
- Miembros con IDs malformados o no secuenciales (que rompen la paginación interna de batches de Ghost).
- Desajustes entre el conteo total de miembros destinatarios y los que son elegibles para recibir el correo en el batch actual.
- Intentos de re-envío de campañas fallidas que ya han registrado envíos parciales.

Para evitar esto, se utiliza un script de preflight (`newsletter-preflight.js`) antes del envío o re-intento de cualquier newsletter.

### 1. Variables de Entorno del Preflight
- `GHOST_DB_CONTAINER`: Nombre del contenedor Docker de MariaDB (por defecto `miau_mysql` en esta plantilla).
- `NEWSLETTER_EMAIL_ID`: ID único de la campaña en la tabla `emails`. Si no se suministra, el script autodetecta la última registrada.
- `NEWSLETTER_SLUG`: Slug del boletín objetivo (por defecto `default-newsletter`).

### 2. Ejecución del Preflight
El script realiza consultas directas a MariaDB para comprobar la consistencia y retorna una respuesta estructurada en formato JSON:

```bash
node newsletter-preflight.js <email_id>
```

#### Respuesta esperada (Exit Code 0 si es seguro):
```json
{
  "emailId": "666c1b3f...",
  "newsletterSlug": "default-newsletter",
  "email_status": "failed",
  "stored_email_count": 1420,
  "target_newsletter_members": 1420,
  "ghost_batch_eligible_members": 1420,
  "email_recipients": 0,
  "email_batches": 0,
  "misordered_members": 0,
  "currentSendSafe": false,
  "retrySafe": true,
  "ok": true
}
```

### 3. Condiciones de Seguridad Evaluadas
- **`currentSendSafe`**: `true` si es el primer intento de envío, no hay miembros con IDs desordenados (`misordered_members === 0`) y la cantidad de miembros objetivo coincide con los elegibles.
- **`retrySafe`**: `true` si la campaña falló previamente (`email_status === 'failed'`), pero no hay registros parciales en `email_recipients` o `email_batches`, garantizando que reintentar no causará duplicados.
- **`ok`**: Evaluación combinada. Si el preflight sale con código `1` (`ok: false`), el envío debe suspenderse hasta sanear la base de datos o verificar los logs de Sentry del `mailgun-proxy` para diagnosticar fallas de red/SMTP upstream.

