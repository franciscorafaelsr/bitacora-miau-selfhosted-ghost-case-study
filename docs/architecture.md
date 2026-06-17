# Arquitectura del Stack

Este documento describe la arquitectura técnica del stack autohospedado utilizado para operar **Bitácora Miau**. El stack está diseñado para ejecutarse sobre Docker Compose en un servidor virtual privado (VPS), priorizando la eficiencia en el uso de memoria RAM, la velocidad de carga mediante caché perimetral, y la robustez del correo transaccional.

## Componentes del Sistema

El stack se divide en el gateway perimetral, el proxy de aceleración, el gestor de contenidos, la base de datos y un conjunto de servicios complementarios (sidecars).

```text
                               ┌────────────────────────────────┐
                               │            INTERNET            │
                               └────────────────┬───────────────┘
                                                │ HTTPS (Port 443)
                                                ▼
                               ┌────────────────────────────────┐
                               │       Traefik (Gateway)        │
                               └────────────────┬───────────────┘
                                                │ HTTP
                                                ▼
                               ┌────────────────────────────────┐
                               │  Nginx Accelerator (Caching)   │
                               └───────┬────────────────┬───────┘
                                       │                │
                          /ghost/*     │                │ /assets/*, GET /
                          No-cache     │                │ Proxy Cache
                                       │                │
                                       ▼                ▼
                               ┌───────┴────────────────┴───────┐
                     ┌────────►│           Ghost CMS            │◄────────┐
                     │         └───────────────┬────────────────┘         │
                     │                         │                          │
                     ▼                         ▼                          ▼
            ┌────────┴────────┐       ┌────────┴────────┐        ┌────────┴────────┐
            │   imgproxy      │       │    MariaDB      │        │  mailgun-proxy  │
            │  (Optimization) │       │   (Database)    │        │  (SMTP Gateway) │
            └─────────────────┘       └─────────────────┘        └────────┬────────┘
                                                                          │
                                                                          ▼
                                                                 ┌────────┴────────┐
                                                                 │   SMTP Relay    │
                                                                 │ (Brevo/Mailcow) │
                                                                 └─────────────────┘
```

### 1. Traefik (Gateway de Entrada)
* **Función**: Actúa como el proxy inverso principal de la máquina virtual (host level).
* **Responsabilidades**:
  * Escuchar el tráfico entrante en los puertos `80` (HTTP) y `443` (HTTPS).
  * Redirección automática de HTTP a HTTPS.
  * Resolución y renovación automática de certificados SSL/TLS mediante Let's Encrypt (ACME).
  * Enrutamiento del tráfico hacia el contenedor de Nginx Accelerator utilizando nombres de dominio.

### 2. Nginx Accelerator (Aceleración y Cachi)
* **Función**: Capa intermedia encargada de optimizar las solicitudes HTTP que van dirigidas a Ghost CMS.
* **Responsabilidades**:
  * Cachear respuestas GET exitosas (páginas públicas, artículos, feeds RSS) utilizando `proxy_cache`.
  * Servir archivos estáticos directamente o reenviar la solicitud de recursos pesados.
  * Omitir el almacenamiento en caché para peticiones de administración (`/ghost/`), previsualizaciones de artículos (`/p/`), flujos de miembros y cuenta como `/subscribe/`, `/account/` y endpoints relacionados con membresías/portal, y llamadas de la API de Ghost.
  * Reducir la carga de CPU y el uso de memoria en Ghost CMS al evitar la regeneración constante de páginas HTML estáticas.

### 3. Ghost CMS (Motor Editorial)
* **Función**: La aplicación central que provee el panel de redacción, la interfaz web del blog, la API de contenidos y el Portal de miembros.
* **Responsabilidades**:
  * Gestionar los posts, páginas, etiquetas y metadatos.
  * Controlar el registro, inicio de sesión y perfiles de los miembros (suscripciones).
  * Generar y encolar el envío de newsletters.
  * Interactuar con MariaDB para la persistencia del contenido.

### 4. MariaDB (Base de Datos)
* **Función**: Almacenamiento estructurado de datos.
* **Responsabilidades**:
  * Persistir usuarios, posts, configuraciones, tokens de sesión y miembros.
  * Configurada con optimizaciones de memoria (como límites en el tamaño del buffer de InnoDB) para operar de forma eficiente en entornos con recursos de RAM ajustados.

### 5. Servicios Sidecars (Complementarios)

* **imgproxy**:
  * Modifica el tamaño, optimiza y comprime las imágenes (a formatos modernos como WebP o AVIF) bajo demanda.
  * Evita que el servidor gaste recursos guardando múltiples copias estáticas de una imagen y reduce el ancho de banda transferido hacia el usuario final.
* **mailgun-proxy**:
  * Microservicio desarrollado en Node.js que emula la API HTTP de Mailgun.
  * Captura las peticiones de envío masivo que Ghost emite de forma nativa mediante Mailgun y las procesa enviando los correos a través de un relay SMTP alternativo (como Brevo o Mailcow).
  * Otorga observabilidad detallada y control de concurrencia sobre los correos enviados.
* **tools-api**:
  * API personalizada para el soporte de herramientas internas de Bitácora Miau (por ejemplo, interacción y servicios dinámicos de la sección de miembros "Mis Gatos").

---

## Diseño de Redes de Docker

El stack utiliza dos redes principales para garantizar la seguridad y el aislamiento:

1. **Red Pública (`proxy`)**:
   * Red externa compartida por Traefik y Nginx.
   * Traefik solo puede ver y comunicarse con el contenedor Nginx Accelerator. Ghost CMS y la base de datos no están expuestos en esta red externa, bloqueando accesos directos desde internet.
2. **Red Interna (`miau_internal`)**:
   * Red de tipo bridge cerrada para el stack.
   * Conecta Nginx Accelerator, Ghost CMS, MariaDB, imgproxy, mailgun-proxy y tools-api.
   * Evita que la base de datos o el CMS escuchen en puertos del host directamente.

## Gestión de Recursos y KSM (Kernel Samepage Merging)

Para operar este stack de forma económica en un VPS de bajos recursos, se aplican límites de memoria rígidos en Docker Compose y se puede aprovechar el servicio del kernel Linux **KSM** como optimización opcional:

* **Límites de RAM**:
  * MariaDB: Limitado a 256MB.
  * Ghost: Limitado a 384MB (384m).
  * Nginx: Limitado a 64MB.
  * Sidecars: Limitados a 64-128MB.
* **KSM (Opcional - Requiere soporte en el Host)**: Es posible montar la librería de soporte KSM (`libksm_enabler.so`) en los contenedores más pesados (como Ghost y MariaDB) para permitir que el kernel consolide páginas de memoria RAM idénticas entre procesos y reduzca el uso general del host. En la configuración de ejemplo, los bloques relacionados se encuentran comentados por defecto para simplificar el arranque inicial.
