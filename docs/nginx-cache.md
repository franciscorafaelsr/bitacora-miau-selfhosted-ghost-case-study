# Aceleración y Caché con Nginx

Este documento describe la estrategia de almacenamiento en caché implementada en la capa de **Nginx Accelerator** para **Bitácora Miau**. Nginx actúa como un proxy reverso intermedio que protege a Ghost CMS de la sobrecarga de consultas y acelera el tiempo de respuesta para los lectores.

## Principio de Operación

Ghost CMS es una aplicación basada en Node.js que genera páginas web dinámicamente consultando la base de datos MariaDB. Enviar cada petición de un usuario directo a Node.js y a la base de datos reduce drásticamente el rendimiento del servidor ante picos de tráfico.

Para mitigar esto, Nginx almacena en memoria y en disco las copias procesadas del blog (páginas estáticas) y las sirve de forma inmediata.

```text
Solicitud del Lector ──────► [ Nginx ] ──(¿Hay caché válida?)──► SI ──► Retorna HTML
                                │
                               NO
                                │
                                ▼
                         [ Ghost CMS ] ◄──────► [ MariaDB ]
```

---

## Directivas de Caché en Nginx

El sistema utiliza la directiva `proxy_cache` configurando las siguientes directivas principales:

* **`proxy_cache_path`**: Define el directorio del disco donde se almacenan las páginas cacheadas, el límite máximo de almacenamiento (`max_size=1g`) y el tiempo que deben permanecer inactivas antes de ser eliminadas (`inactive=60m`).
* **`keys_zone`**: Reserva un espacio en memoria RAM (ej. `ghost_cache:10m`) para guardar los metadatos e índices de las llaves de caché, acelerando las búsquedas en disco.
* **`proxy_cache_key`**: Establece la cadena identificadora para cada recurso (típicamente `$scheme$proxy_host$request_uri`).

---

## Reglas de Caching (Rutas Públicas vs. Rutas Dinámicas)

Para asegurar que las secciones administrativas y las interacciones de los miembros funcionen en tiempo real sin mostrar datos desactualizados, se implementa una separación lógica estricta.

### 1. Rutas que NUNCA deben cachearse
Nginx establece variables para evitar el almacenamiento (`proxy_cache_bypass`) y la lectura (`proxy_no_cache`) de la caché en los siguientes casos:

* **Administración de Ghost**: `/ghost/` y cualquier ruta relacionada.
* **Vistas de previsualización**: `/p/` (artículos en borrador que los editores revisan antes de publicar).
* **API del CMS**: `/ghost/api/` (consultas de datos dinámicos).
* **Flujos de miembros y cuenta**: Rutas como `/subscribe/`, `/account/` y endpoints relacionados con membresías/portal.
* **Peticiones HTTP que no sean seguras**: Métodos `POST`, `PUT`, `DELETE` o `PATCH`.

### 2. Cookies de Exclusión de Caché
Incluso si se solicita una ruta pública como la portada o un artículo, Nginx omitirá la caché si la petición contiene ciertas cookies que revelan que el usuario está autenticado o tiene una sesión activa:
* `ghost-members-ssr`: Indica que un miembro ha iniciado sesión.
* Cookies de sesión de administradores (identificables por patrones en la cookie de Ghost).

### 3. Rutas con Caching Agresivo
Se configuran para guardarse en caché con tiempos de vida de hasta 24 horas:
* Recursos estáticos de temas: `/assets/` (archivos CSS, JS del frontend, tipografías).
* Imágenes optimizadas que no cambian de contenido frecuentemente.

---

## Entrega de Contenido Obsoleto en Caso de Falla (`stale-while-revalidate`)

Nginx está configurado con directivas de tolerancia a fallos para mantener el sitio en línea incluso si Ghost CMS se cae o está reiniciándose por mantenimiento:

```nginx
proxy_cache_use_stale error timeout updating http_500 http_502 http_503 http_504;
proxy_cache_background_update on;
proxy_cache_lock on;
```

* **`proxy_cache_use_stale`**: Si Ghost CMS responde con un error o tarda demasiado, Nginx servirá una copia obsoleta (stale) de la página que tenga en su disco, en lugar de mostrar un error 502 Bad Gateway al lector.
* **`proxy_cache_background_update`**: Cuando una página cacheada expira, Nginx sirve la versión antigua al primer usuario que la solicita mientras lanza una petición en segundo plano hacia Ghost para refrescar la caché en el disco de manera silenciosa.
* **`proxy_cache_lock`**: Si varias peticiones concurrentes solicitan el mismo recurso ausente en la caché, solo una petición pasará a Ghost CMS para regenerarlo; las demás esperarán su respuesta reduciendo el impacto de carga.

---

## Verificación de la Caché (Cabeceras HTTP)

Para auditar y monitorear el comportamiento de Nginx, se inyecta una cabecera personalizada en la respuesta HTTP devuelta al navegador del lector:

```nginx
add_header X-Cache-Status $upstream_cache_status;
```

Los posibles estados devueltos son:
* **`HIT`**: La página se sirvió directamente desde la caché de Nginx sin tocar Ghost CMS.
* **`MISS`**: El recurso no estaba en caché; Nginx lo solicitó a Ghost y lo guardó para futuras peticiones.
* **`BYPASS`**: Nginx ignoró la caché debido a una cookie de sesión o a una ruta restringida.
* **`EXPIRED`**: El recurso existía en la caché pero su tiempo de validez había expirado; Nginx consultó a Ghost para actualizarlo.
* **`STALE`**: Ghost no estaba disponible y Nginx entregó una copia obsoleta segura.
