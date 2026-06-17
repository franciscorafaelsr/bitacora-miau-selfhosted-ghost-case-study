# Gestión de Miembros y Suscripciones (Ghost Members)

Este documento detalla el funcionamiento del sistema de membresías, autenticación y flujos de correo transaccional en **Bitácora Miau**.

## Flujo de Registro y Autenticación sin Contraseña (Passwordless Magic Links)

Ghost implementa un sistema de membresías y registro basado en **Magic Links** (enlaces mágicos). Los usuarios no ingresan ni registran contraseñas tradicionales, lo que incrementa la seguridad y simplifica la experiencia:

```text
Lector ingresa correo en formulario 
     │
     ▼
[ Ghost CMS ] ──(Genera Token temporal)──► [ SMTP Transaccional ] ──► Correo de Magic Link
                                                                             │
                                                                             ▼
                                                                  Lector abre enlace
                                                                             │
                                                                             ▼
                                                                 Nginx bypasses Cache
                                                                             │
                                                                             ▼
                                                                    Sesión iniciada
```

1. **Petición de Acceso**: El usuario ingresa su correo en el portal del blog.
2. **Generación de Token**: Ghost genera un token temporal seguro y crea una URL única de login.
3. **Envío de Correo Transaccional**: Ghost envía un correo directamente a través del transporte **SMTP transaccional** configurado de forma nativa en sus variables de entorno (`mail__transport=SMTP`).
4. **Validación**: El lector hace clic en el enlace de su bandeja de entrada, regresa al dominio del blog (pasando por Nginx, que al ver las cookies de inicio de sesión de miembros, realiza un BYPASS de la caché), y Ghost autentica al miembro iniciando su sesión.

---

## Separación de Canales de Correo (Transaccional vs. Newsletters)

Para optimizar costos y asegurar una excelente entregabilidad de correo, el stack divide las rutas de salida de correos en dos flujos independientes:

| Tipo de Correo | Caso de Uso | Volumen | Transporte Técnico | Configuración en Ghost |
| :--- | :--- | :--- | :--- | :--- |
| **Transaccional** | Magic links, correos de bienvenida, alertas de suscripción, facturación. | Bajo pero crítico (debe ser instantáneo). | **SMTP** directo (ej. Brevo SMTP, Amazon SES, Mailcow propio). | Se configura mediante variables `mail__options` en `docker-compose`. |
| **Boletines / Newsletters** | Envíos masivos de nuevos artículos a todos los suscriptores. | Alto volumen por ráfagas. | **API HTTP de Mailgun** (redirigida al contenedor `mailgun-proxy`). | Se configura en el panel administrativo (`Settings > Email newsletters`). |

---

## Integración del Portal (Ghost Portal)

El portal de miembros se renderiza en el frontend mediante un script de JavaScript inyectado automáticamente por Ghost. Este widget (conocido como **Portal**) maneja de forma asíncrona la comunicación con los endpoints de la API de miembros en `/ghost/api/v4/members/`.

Nginx Accelerator contiene reglas específicas para **no almacenar en caché** ninguna respuesta proveniente de los endpoints de la API de miembros, asegurando que las solicitudes de login, planes de cobro y datos de cuenta de los usuarios reflejen el estado correcto en tiempo real.

> [!NOTE]
> En la plantilla de Nginx provista (`nginx.example.conf`), el bypass de caché para los flujos de miembros se modela explícitamente a través de:
> - Reglas de exclusión para rutas de páginas de membresía como `/subscribe/` y `/account/`.
> - Detección de la cookie de sesión de miembros `ghost-members-ssr`.
> - Exclusión de la API y rutas internas bajo el prefijo `/ghost/`.

---

## Seguridad y Privacidad

1. **Tokens de sesión**: Los tokens de miembros son de corta duración.
2. **Aislamiento de cookies**: La cookie `ghost-members-ssr` se utiliza para identificar solicitudes autenticadas. Nginx lee esta cabecera `Cookie` y, si detecta dicho valor, salta la caché por completo evitando la fuga accidental de contenido privado o personalizado a otros lectores no suscritos.
3. **Exclusión de Tracking**: Las configuraciones de Ghost se parametrizan para reducir la recopilación excesiva de datos de miembros, manteniendo el enfoque en la privacidad de los lectores de Bitácora Miau.
