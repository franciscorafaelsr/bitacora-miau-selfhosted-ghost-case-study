# Adaptador de API Mailgun (mailgun-proxy)

Este documento detalla el diseño y funcionamiento del componente de compatibilidad **mailgun-proxy** utilizado en **Bitácora Miau**. Este componente actúa como un adaptador de arquitectura que traduce las solicitudes de la API HTTP de Mailgun generadas por Ghost CMS en llamadas SMTP estándar, facilitando el desacoplamiento de proveedores y la observabilidad del canal de distribución de correo masivo.

## Contexto y Desafío

Ghost CMS permite publicar newsletters directamente para los miembros del sitio. Sin embargo, su arquitectura restringe el envío de campañas masivas exclusivamente a través de la API HTTP oficial de **Mailgun**. Aunque las funciones transaccionales (como magic links o notificaciones individuales) pueden configurarse con cualquier servidor SMTP genérico, el botón de "Publicar y enviar newsletter" obliga a usar Mailgun.

En un entorno autohospedado, depender de un único proveedor comercial con configuración cerrada genera los siguientes retos:
1. **Falta de visibilidad**: Dificultad para registrar y analizar qué datos y metadatos envía Ghost exactamente durante un despacho de correos masivo.
2. **Dependencia tecnológica**: Imposibilidad de enrutar newsletters a través de proveedores SMTP alternativos (como Brevo, Amazon SES, Mailcow propio o relés especializados).
3. **Diagnóstico complejo**: Si ocurre una falla de entregabilidad en Mailgun, el CMS no suele ofrecer logs detallados sobre la respuesta exacta del servidor de correo.
4. **Control de concurrencia**: Ghost envía las solicitudes de correos masivos en ráfagas que pueden saturar límites de tasas (rate limits) del proveedor de envío.

---

## Arquitectura de Adaptación: Mailgun HTTP-to-SMTP Gateway

Se desarrolló una capa de compatibilidad (microservicio) en **Node.js y Express** que se ubica entre Ghost CMS y el relé SMTP final. Este gateway expone una interfaz idéntica al endpoint de mensajería esperado por la API de Mailgun, procesa la carga útil formateada como multipart y realiza el reenvío a través de transporte SMTP estándar.

### Flujo simplificado de correo masivo

```text
[ Ghost CMS ]
     │
     │ HTTP POST (Multipart/Form-Data) a /v3/domain/messages
     ▼
[ mailgun-proxy ]
     │
     │ 1. Procesa destinatarios (Lista separada por comas en campo 'to')
     │ 2. Segmenta en lotes de tamaño configurable (Chunking)
     │ 3. Genera un email SMTP por cada lote
     ▼
[ SMTP Relay (Brevo / Mailcow) ]
     │
     │ Distribución de correos
     ▼
[ Destinatarios Finales ]
```

---

## Características de la Implementación

### 1. Compatibilidad de Endpoint
Ghost CMS intenta enviar correos llamando a la siguiente ruta de Mailgun:
`POST https://api.mailgun.net/v3/<domain>/messages`

El proxy simula esta ruta escuchando en:
`POST /v3/:domain/messages`

### 2. Procesamiento de Multipart Form Data
Dado que Ghost envía el contenido del correo utilizando codificación `multipart/form-data` (para adjuntar el asunto, contenido en texto, cuerpo en HTML y los destinatarios), el proxy utiliza el middleware `multer` en Node.js para parsear el cuerpo del request sin necesidad de guardar archivos temporales en disco.

### 3. Segmentación en Lotes (Chunking)
Cuando el boletín se envía a miles de personas, Ghost manda una sola llamada API HTTP con todos los correos de los destinatarios concatenados en el campo `to`. El proxy:
* Lee la lista de destinatarios.
* Los divide en lotes de un tamaño configurable (`CHUNK_SIZE`, por defecto `1000` correos).
* Envía cada lote mediante el transporte de NodeMailer al servidor SMTP de destino de forma secuencial o paralela controlada.

---

## Ventajas Arquitectónicas

* **Observabilidad Centralizada**: Permite capturar e imprimir en la consola del contenedor el estado de cada lote enviado y registrar los identificadores de mensaje de retorno (`Message-ID`) provistos por el servidor SMTP final.
* **Flexibilidad e Independencia de Proveedores**: Permite alternar la entrega masiva entre múltiples servidores SMTP o relés corporativos (como Brevo, Amazon SES o servidores Mailcow propios) modificando únicamente las variables de entorno, sin alterar el CMS.
* **Desacoplamiento de API Propietaria**: Evita que el CMS quede acoplado estrictamente a las condiciones o cuotas de un único proveedor SaaS de entrega HTTP masiva.
* **Diagnóstico y Resiliencia**: Captura de forma estructurada cualquier excepción del servidor SMTP (por ejemplo, rechazo de cabeceras o credenciales inválidas) y las reporta inmediatamente a Sentry para alertamiento temprano.

---

## Consideraciones Operativas y Entregabilidad

El uso del adaptador resuelve un problema técnico de **compatibilidad**, pero la **entregabilidad** (el porcentaje de correos que llegan efectivamente a la bandeja de entrada del usuario en vez de spam) sigue dependiendo de la reputación y configuración del relé SMTP final:
1. **Configuración de DNS (SPF, DKIM, DMARC)**: El dominio remitente debe autorizar expresamente a las IPs del servidor SMTP final (ya sea Brevo, Mailcow, etc.) para enviar correos en su nombre.
2. **Reputación del IP**: Los proveedores como Gmail, Yahoo o Outlook analizan la reputación de la dirección IP que emite la conexión SMTP. Si se usa un relay SMTP compartido de baja reputación o un VPS propio con IP sucia, los correos serán rechazados.
3. **Gestión de Rebotes y Bajas**: El proxy no implementa de forma automática un manejador de bajas (Unsubscribe webhook) ni de rebotes (bounces). Es mandatorio usar la lógica nativa del Portal de Ghost para permitir a los usuarios darse de baja y procesar manualmente o mediante webhooks los rebotes para limpiar la base de datos de correos inválidos.

---

## Observabilidad con Sentry (Sentry Error Tracking)

Para identificar y solucionar errores durante el despacho de boletines de forma inmediata, `mailgun-proxy` cuenta con un sistema de observabilidad integrado mediante **Sentry**.

### 1. Inicialización de Sentry
El SDK oficial de Sentry para Node.js (`@sentry/node`) se integra de forma condicional. Se activa únicamente si la variable de entorno `SENTRY_DSN_MAILGUN_PROXY` está definida:

```javascript
const Sentry = require('@sentry/node');

if (process.env.SENTRY_DSN_MAILGUN_PROXY) {
    try {
        const tracesSampleRate = Number(process.env.SENTRY_TRACES_SAMPLE_RATE || 0);
        Sentry.init({
            dsn: process.env.SENTRY_DSN_MAILGUN_PROXY,
            environment: process.env.NODE_ENV || 'production',
            tracesSampleRate: Number.isFinite(tracesSampleRate) ? tracesSampleRate : 0,
        });
        console.log('[mailgun-proxy] Sentry initialized');
    } catch (err) {
        console.error('[mailgun-proxy] Sentry init failed:', err.message);
    }
}
```

### 2. Captura de Errores en Express
Las excepciones no controladas y los errores de respuesta del relé SMTP final son interceptados por el middleware global de Express y despachados hacia Sentry para su análisis detallado:

```javascript
app.use((err, _req, res, _next) => {
    console.error('[mailgun-proxy] unhandled error:', err);
    if (process.env.SENTRY_DSN_MAILGUN_PROXY) {
        Sentry.captureException(err);
    }
    res.status(500).json({ message: 'Internal server error' });
});
```

### 3. Trazas Opt-in
Para optimizar el uso de recursos y evitar cargos innecesarios en la cuenta de Sentry, las trazas de rendimiento (`tracesSampleRate`) se definen de forma **opt-in** mediante la variable `SENTRY_TRACES_SAMPLE_RATE` (por defecto `0`), garantizando que solo se reporten errores severos a menos que se configure una tasa superior con fines de diagnóstico activo.

