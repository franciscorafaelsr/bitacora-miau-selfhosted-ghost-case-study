# Optimización de Imágenes con imgproxy

Este documento detalla la integración de **imgproxy** en el stack de **Bitácora Miau** para realizar procesamiento y optimización de imágenes al vuelo.

## Problema y Objetivos

Las plataformas editoriales modernas requieren múltiples tamaños de imagen para renderizar diseños adaptativos (responsive). Por ejemplo, el banner principal del artículo, la miniatura de la tarjeta en el feed de inicio y la imagen optimizada para dispositivos móviles.

Guardar estas variantes de forma estática en disco consume espacio de almacenamiento de manera innecesaria. Adicionalmente, forzar al navegador a descargar la imagen original (muchas veces de varios megabytes) afecta negativamente a los tiempos de carga del sitio, arruinando el SEO móvil y la experiencia de usuario.

El objetivo de integrar **imgproxy** es:
1. **Compresión automática**: Convertir formatos pesados (`PNG`, `JPEG`) a formatos modernos altamente eficientes (`WebP` o `AVIF`).
2. **Dimensionamiento al vuelo**: Modificar el tamaño y recortar las imágenes basándose en parámetros incluidos en la URL.
3. **Desacoplamiento**: Evitar que Ghost gaste CPU procesando imágenes.

---

## Flujo de Solicitud de Imágenes

Cuando un usuario entra al blog, el navegador solicita una imagen formateada. El flujo funciona de la siguiente manera:

```text
Navegador del Lector
     │
     │ GET https://bitacoramiau.com/img/unsigned/resize:fill:400:300/plain/https://bitacoramiau.com/content/images/cat.jpg@webp
     ▼
[ Nginx (Routing) ] ──► Redirecciona a ──► [ imgproxy ]
                                                │
                                                │ 1. Descarga 'cat.jpg' desde Ghost Storage
                                                │ 2. Procesa la imagen (redimensiona a 400x300, comprime a WebP)
                                                ▼
                                         Retorna binario optimizado
```

---

## Firma de URLs y Seguridad (Cryptographic Signatures)

Permitir que cualquiera modifique el tamaño de las imágenes libremente en una URL abre el servidor a ataques de **Denegación de Servicio (DDoS)**. Un atacante podría escribir un script que solicite una imagen en miles de resoluciones aleatorias consecutivamente (ej. `resize:fill:401:300`, `resize:fill:402:300`), obligando al CPU del servidor a procesar infinitas imágenes y llenando el almacenamiento del caché.

Para bloquear esto, imgproxy admite la firma criptográfica de URLs utilizando una clave hexadecimal (`IMGPROXY_KEY`) y una sal criptográfica (`IMGPROXY_SALT`) compartida entre el generador de la URL (el CMS o el proxy Nginx) y el contenedor de imgproxy:

* **URL no firmada (Modo Desarrollo/Pruebas)**:
  `/img/unsigned/resize:fill:400:300/plain/https://example.com/image.jpg@webp`
* **URL firmada (Modo Producción)**:
  `/img/8hG1K_FshJ9Kj-Z.../resize:fill:400:300/plain/https://example.com/image.jpg@webp`

Si un cliente intenta modificar la URL alterando un solo píxel de resolución sin conocer la firma secreta, imgproxy rechazará inmediatamente la petición con un error `403 Forbidden` sin realizar procesamiento en el CPU.

---

## Configuración y Variables Clave

El contenedor de imgproxy en Docker Compose se configura utilizando variables de entorno críticas expuestas en el archivo de secretos:

* `IMGPROXY_KEY`: Clave secreta hexadecimal (generada de forma aleatoria en el despliegue).
* `IMGPROXY_SALT`: Sal secreta hexadecimal.
* `IMGPROXY_MAX_SRC_RESOLUTION`: Límite máximo en megapíxeles (ej. `20` MP) de las imágenes de origen para evitar que archivos corruptos o maliciosos de resoluciones gigantescas saturen la memoria RAM.
* `IMGPROXY_TTL`: Tiempo de caché en el cliente (cabeceras `Cache-Control`).
