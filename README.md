# Bitácora Miau Platform

Documentación técnica del stack autohospedado usado para construir y operar **Bitácora Miau**, un proyecto editorial sobre salud felina desarrollado con Ghost CMS, Docker, MariaDB, Nginx, imgproxy, correo transaccional, newsletters y herramientas propias para usuarios.

Este repositorio no contiene secretos, credenciales ni configuración privada de producción. Su objetivo es documentar la arquitectura, decisiones técnicas y aprendizajes del proyecto como caso de estudio.

## Objetivo

Construir una plataforma editorial autohospedada que pudiera funcionar como blog, newsletter, sistema de miembros y base para herramientas futuras relacionadas con el cuidado de gatos.

## Arquitectura general

El flujo de solicitudes web principal y la interacción de componentes secundarios se organizan de la siguiente manera:

```text
Internet
   │
   ▼
[ Traefik (Main Gateway / SSL) ]
   │
   ▼
[ Nginx (Caching Accelerator) ]
   │
   ▼
[ Ghost CMS ] ───► [ MariaDB ]
   │
   ├─► [ imgproxy ] (Image Optimization)
   ├─► [ mailgun-proxy ] (Mailgun HTTP API to SMTP Gateway) ──► [ SMTP Relay (Brevo/Mailcow) ]
   └─► [ tools-api ] (Custom Node.js Services - "Mis gatos")
```

---

## Stack principal

* **Ghost CMS** (Plataforma editorial y membresías)
* **Docker & Docker Compose** (Orquestación de servicios)
* **MariaDB** (Base de datos relacional)
* **Nginx** (Motor de caché reverso y proxy de aceleración)
* **Traefik** (Gateway perimetral y renovación SSL automática)
* **imgproxy** (Procesamiento de imágenes al vuelo y compresión a WebP/AVIF)
* **Node.js / Express** (Para la API personalizada de herramientas y el proxy Mailgun)
* **SMTP / Mailcow / Brevo** (Servicios de correo transaccional y masivo)
* **Ghost Members & Portal** (Gestión de suscripciones y magic links)

---

## Retos documentados

1. **Ghost autohospedado**
   Configuración de Ghost CMS en un entorno propio, con persistencia de datos, variables de entorno, base de datos externa y reverse proxy.
2. **Caché con Nginx**
   Separación entre rutas públicas cacheables y rutas dinámicas que deben evitar caché agresiva, como administración, previews, API y miembros.
3. **Optimización de imágenes**
   Integración de imgproxy para transformar y optimizar imágenes usadas en artículos, portadas y recursos visuales.
4. **Correo y newsletters**
   Documentación del flujo de correo transaccional y boletines masivos, integración mediante un adaptador de API compatible con Mailgun, garantizando resiliencia y observabilidad mediante instrumentación activa (Sentry).
5. **Herramientas propias**
   Diseño inicial de una API para funciones como “Mis Gatos”, orientada a miembros autenticados.
6. **Simplificación del stack**
   Evaluación de servicios externos como Remark42 y migración hacia funciones nativas cuando reducen complejidad operativa.

---

## Estructura del repositorio

```text
bitacora-miau-platform/
├── README.md
├── docs/
│   ├── architecture.md
│   ├── mailgun-proxy.md
│   ├── nginx-cache.md
│   ├── imgproxy.md
│   ├── ghost-members.md
│   ├── operations.md
│   └── decisions.md
├── diagrams/
│   ├── architecture.svg
│   ├── email-flow.svg
│   └── request-flow.svg
├── examples/
│   ├── docker-compose.example.yml
│   ├── nginx.example.conf
│   ├── env.example
│   └── mailgun-proxy.example.env
└── screenshots/
    ├── homepage.png
    ├── members-flow.png
    └── tools-page.png
```

> [!NOTE]
> **Nota sobre componentes auxiliares**: Esta documentación incluye ejemplos de integración con servicios complementarios como `mailgun-proxy` y `tools-api`. Dichos componentes pueden mantenerse en repositorios separados o fuera del alcance de esta plantilla pública, ya que este repositorio está centrado exclusivamente en la arquitectura, configuraciones de ejemplo y decisiones de diseño del stack principal.

---

## Qué NO contiene este repositorio

Este repositorio **no incluye** ni debe incluir bajo ninguna circunstancia:
* API Keys o Tokens reales.
* Credenciales SMTP activas.
* Variables de entorno con contraseñas de producción.
* Copias de seguridad de bases de datos.
* Datos reales de usuarios o suscriptores.
* Direcciones de correo electrónico reales de producción.
* Logs con información de depuración privada.
* Claves DKIM privadas o configuraciones de registros DNS internos.
* Configuración exacta o detalles de acceso al VPS físico.

## Estado del proyecto

**Bitácora Miau** es un proyecto en constante evolución. La documentación técnica en este repositorio se actualiza progresivamente a medida que se implementan optimizaciones en la infraestructura, herramientas adicionales o mejoras en la entregabilidad del correo.
