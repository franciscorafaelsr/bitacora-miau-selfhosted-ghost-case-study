# Registro de Decisiones de Arquitectura (ADR)

Este documento registra las decisiones técnicas clave tomadas durante el diseño y evolución de la plataforma **Bitácora Miau**, explicando el contexto, las alternativas evaluadas y las justificaciones de cada elección.

---

## ADR 1: Elección de Stack Autohospedado (Self-Hosted) frente a Ghost Pro (SaaS)

* **Estatus**: Aprobado
* **Contexto**: Bitácora Miau requiere una plataforma editorial robusta con suscripciones y newsletters. Ghost Pro ofrece el software como servicio (SaaS), encargándose del hosting, seguridad y entregabilidad de correos por una tarifa mensual escalable basada en la cantidad de miembros.
* **Decisión**: Se optó por construir un entorno **autohospedado** basado en Docker y VPS.
* **Justificación**:
  * **Predictibilidad Financiera y de Escalabilidad**: Evita el incremento exponencial de costos de licencias SaaS basado en el volumen de suscriptores, permitiendo un crecimiento lineal de la audiencia sobre infraestructura de costo fijo.
  * **Extensibilidad**: La autogestión permite integrar componentes secundarios de forma directa (como bases de datos optimizadas, APIs de servicios personalizados e inyección de middleware de aceleración en Nginx).
  * **Soberanía de Datos**: Absoluta propiedad y control regulatorio sobre la base de datos de correos, perfiles de miembros y analíticas de comportamiento, sin dependencias de políticas de retención de terceros.

---

## ADR 2: Implementación de Adaptador de API Mailgun (mailgun-proxy)

* **Estatus**: Aprobado
* **Contexto**: Ghost CMS obliga a utilizar la API HTTP de Mailgun para el envío de boletines informativos masivos. Esta restricción limita la flexibilidad para enrutar correos a través de infraestructuras SMTP corporativas o relés transaccionales alternativos externos.
* **Decisión**: Implementar un adaptador de compatibilidad (**mailgun-proxy**) que expone una firma de API idéntica a la esperada por Ghost CMS, parsea las peticiones multipart HTTP y las enruta a través de transporte SMTP estándar hacia relés de correo.
* **Justificación**:
  * **Flexibilidad de Proveedores**: Permite enrutar el tráfico de boletines hacia cualquier infraestructura SMTP corporativa o relés transaccionales de alta reputación (como Brevo, Amazon SES o servidores de correo propios).
  * **Observabilidad Avanzada**: Introduce un punto centralizado para interceptar, registrar e inspeccionar metadatos de correos masivos, facilitando el diagnóstico de fallas operativas.
  * **Independencia de Integración**: Traduce llamadas propietarias HTTP a estándares abiertos (SMTP), evitando el acoplamiento directo de la aplicación a nivel de código con un único proveedor SaaS de envío.

---

## ADR 3: Capa de Aceleración de Caché con Nginx (Nginx Accelerator)

* **Estatus**: Aprobado
* **Contexto**: Ghost CMS corre sobre Node.js y, por ende, es susceptible a saturaciones de memoria y CPU cuando se reciben múltiples solicitudes simultáneas de lectores, afectando el tiempo de respuesta y el SEO técnico.
* **Decisión**: Colocar un contenedor de Nginx configurado específicamente para almacenamiento en caché (`proxy_cache`) por delante de Ghost CMS.
* **Justificación**:
  * Nginx procesa miles de conexiones concurrentes consumiendo muy pocos megabytes de memoria RAM.
  * Protege al CMS de ráfagas de tráfico sirviendo las solicitudes estáticas de forma instantánea.
  * Permite lógica avanzada como `stale-while-revalidate` (servir contenido antiguo en caché de forma rápida mientras se actualiza en background, o si Ghost está caído).

---

## ADR 4: Migración y Simplificación (Uso de Funciones Nativas sobre Remark42)

* **Estatus**: Aprobado
* **Contexto**: Inicialmente se consideró utilizar herramientas de comentarios externas como Remark42 para fomentar la comunidad. Esto introducía un contenedor extra en el stack, mayor consumo de memoria RAM y configuraciones de autenticación adicionales.
* **Decisión**: Descartar Remark42 y activar la función nativa de comentarios integrada en las versiones recientes de Ghost CMS.
* **Justificación**:
  * Reducción de la complejidad operativa del stack general (un servicio menos que mantener y monitorear).
  * Experiencia de usuario (UX) 100% cohesionada con la suscripción y portal de miembros nativo de Ghost.
  * Menor consumo de memoria en el VPS host.

---

## ADR 5: Uso de MariaDB en lugar de SQLite o PostgreSQL

* **Estatus**: Aprobado
* **Contexto**: Ghost CMS requiere una base de datos. En desarrollo utiliza SQLite por simplicidad, pero en producción Ghost solo ofrece soporte oficial para MySQL / MariaDB.
* **Decisión**: Seleccionar **MariaDB** ejecutada en contenedor con límites de recursos estrictos.
* **Justificación**:
  * Cumple con la especificación de producción exigida por Ghost para evitar bugs de concurrencia de SQLite.
  * MariaDB es una alternativa de código abierto a MySQL con excelente rendimiento y optimización de huella de memoria RAM, ideal para servidores pequeños.
