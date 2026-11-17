
## ¿Qué es este proyecto?

En la universidad, reconocer a alguien por una buena acción casi siempre se queda en un "gracias" que se dice de paso y se olvida. No queda registro, la persona no siempre se entera y mucho menos hay forma de volver a verlo después. UCO Kudos nace para resolver eso: es una plataforma donde un estudiante o un profesor le publica un Kudo (un reconocimiento corto, con archivos adjuntos si quiere) a otra persona de la comunidad.

Lo importante es lo que pasa cuando se publica. El Kudo se guarda, le llega un correo a quien lo recibe, le llega un mensaje de WhatsApp y, si en ese momento tiene la aplicación abierta, le aparece al instante en pantalla. Además, todo queda auditado: se puede saber qué Kudo se publicó y qué pasó con cada aviso, incluso cuando uno de los canales falló.

Este repositorio es el backend. Está construido con arquitectura limpia, con las decisiones de diseño justificadas, desplegado en Azure con HTTPS y con un pipeline que prueba, analiza y despliega solo. Este documento cuenta todo el proceso: qué se pidió, qué se priorizó, cómo se diseñó, cómo se probó y qué quedó pendiente.

---

## Objetivo del sistema

Permitir que las personas de la comunidad universitaria se reconozcan entre sí publicando Kudos con adjuntos, y que quien lo recibe se entere por correo, por WhatsApp y en tiempo real, dejando un rastro auditable de todo lo ocurrido, con un diseño que se pueda explicar, probar y cambiar en una sustentación.

### Objetivos específicos

- Publicar un Kudo para otra persona, con mensaje y hasta cinco archivos adjuntos.
- Avisar al destinatario por correo, por WhatsApp y por notificación en vivo (push) si está conectado.
- Garantizar que un canal de aviso caído no impida publicar el Kudo ni afecte a los otros canales.
- Registrar en una auditoría cada Kudo creado y el resultado de cada aviso.
- Controlar quién entra: nadie usa la plataforma sin que el administrador apruebe su cuenta.
- Mostrar los Kudos recibidos rápido y de forma paginada, pensando en quien entra desde el celular.
- Mantener los textos y las plantillas fuera del código, y toda la configuración en variables de entorno.
- Entregar el sistema desplegado en la nube, con pruebas automáticas, análisis estático y análisis de vulnerabilidades.

---

## Alcance

### Incluye

- Solicitud de cuenta, aprobación o rechazo por parte del administrador, e ingreso con correo y clave.
- Cambio de clave por parte de cada persona.
- Publicación de Kudos con adjuntos validados por tipo, tamaño y cantidad.
- Avisos por correo (Gmail), WhatsApp (sandbox de Twilio) y push en tiempo real (SSE).
- Cola de notificaciones en Redis con reintentos, un trabajo por cada canal.
- Lista de Kudos recibidos con caché y paginación por cursor.
- Auditoría de Kudos y de avisos.
- Textos y plantillas administrados desde Strapi.
- Aplicación web instalable (PWA) pensada primero para el celular.
- Despliegue en Azure con HTTPS, secretos en Key Vault y entrega continua desde GitHub Actions.

### No incluye

- Editar o borrar un Kudo ya publicado.
- Responder a un Kudo (réplicas). El enunciado pide que no tengan límite de profundidad; esto quedó pendiente (ver Restricciones técnicas).
- Recuperación de clave olvidada. La persona cambia la clave temporal desde dentro, pero no hay "olvidé mi clave".
- Inicio de sesión institucional con Microsoft: no hay acceso al tenant de la universidad.
- Métricas y alertas automáticas. Hoy hay un endpoint de salud y registros.
- Almacenamiento permanente de adjuntos: hoy se guardan en el disco del contenedor y se pierden si este se reinicia.
- API Management y WAF: están diseñados y justificados, pero todavía no están montados.

---

## Usuarios del sistema

En el sistema existen tres roles de acceso:

| Rol del sistema | Qué puede hacer |
|-----------------|-----------------|
| Estudiante | Publicar Kudos, ver los que recibe, recibir avisos y cambiar su clave. |
| Profesor | Exactamente lo mismo que el estudiante. La diferencia es solo el rol con el que se registra. |
| Administrador | Lo mismo que los anteriores, más aprobar o rechazar solicitudes de cuenta. |

### Tipos de personas que se tuvieron en cuenta

Para priorizar la calidad pensé en cinco tipos de personas. Los usé después para repartir los puntos de cada atributo.

| Tipo de persona | Cómo entra al sistema | Qué espera principalmente |
|-----------------|-----------------------|---------------------------|
| Estudiante | Como Estudiante | Publicar y recibir Kudos sin esfuerzo, desde el celular y con datos móviles. |
| Docente | Como Profesor | Reconocer a sus estudiantes y enterarse rápido de lo que le llega. |
| Coordinadores y directivos | Como Profesor o Administrador | Una plataforma confiable, segura y de la que quede rastro. |
| Administrador del sistema | Como Administrador | Aprobar cuentas, mantener la aplicación viva y enterarse si algo falla. |
| Evaluador (docente del curso) | Como Profesor | Entender el código y poder pedir cambios en vivo en la sustentación. |

---

## Requisitos funcionales

### Cuentas y acceso

| Código | Descripción | Prioridad |
|--------|-------------|-----------|
| RF-01 | Una persona sin cuenta debe poder solicitarla indicando nombre, correo, teléfono (10 números) y rol. | Alta |
| RF-02 | Al solicitar una cuenta, la solicitud debe quedar pendiente y al administrador le debe llegar un correo. | Media |
| RF-03 | El administrador debe poder consultar las solicitudes pendientes. | Alta |
| RF-04 | El administrador debe poder aprobar una solicitud; el sistema crea la cuenta con una clave temporal y se la envía a la persona por correo y por WhatsApp. | Alta |
| RF-05 | El administrador debe poder rechazar una solicitud; el sistema avisa a la persona por correo. | Media |
| RF-06 | La persona debe poder ingresar con su correo y su clave, y recibir una sesión. | Alta |
| RF-07 | Si la persona no puede entrar, el sistema debe decirle cuál es el caso: no tiene cuenta, su solicitud está pendiente o fue rechazada. | Media |
| RF-08 | La persona debe poder cambiar su clave indicando la actual y la nueva. | Media |
| RF-09 | La persona debe poder cerrar su sesión. | Media |
| RF-10 | La cuenta del administrador debe crearse sola al arrancar el sistema, si todavía no existe. | Baja |

### Kudos

| Código | Descripción | Prioridad |
|--------|-------------|-----------|
| RF-11 | La persona debe poder publicar un Kudo para otro usuario activo, con un mensaje de hasta 500 caracteres. | Alta |
| RF-12 | La persona debe poder adjuntar hasta cinco archivos de los tipos permitidos y del tamaño permitido. | Alta |
| RF-13 | Al publicar, el sistema debe guardar el Kudo y sus adjuntos. | Alta |
| RF-14 | Al publicar, el sistema debe enviar un correo al destinatario. | Alta |
| RF-15 | Al publicar, el sistema debe enviar un mensaje de WhatsApp al destinatario. | Alta |
| RF-16 | Al publicar, si el destinatario está conectado, el sistema debe mostrarle el Kudo al instante (push). | Alta |
| RF-17 | Al publicar, el sistema debe auditar el Kudo creado y el resultado de cada canal de aviso. | Alta |
| RF-18 | La persona debe poder consultar los Kudos que ha recibido, de más nuevo a más viejo y con paginación por cursor. | Alta |
| RF-19 | La persona debe poder consultar los usuarios activos (solo id y nombre) para escoger a quién enviarle un Kudo. | Media |
| RF-20 | La persona debe poder abrir o descargar un adjunto de un Kudo. | Media |

### Catálogos y operación

| Código | Descripción | Prioridad |
|--------|-------------|-----------|
| RF-21 | Los textos de los mensajes y las plantillas de las notificaciones deben poder editarse en Strapi sin recompilar ni redesplegar. | Media |
| RF-22 | El sistema debe exponer un endpoint de salud para saber si la API está viva. | Media |

---

## Requisitos no funcionales

| Código | Descripción | Prioridad |
|--------|-------------|-----------|
| RNF-01 | El sistema debe estar desplegado en la nube y todo el tráfico debe ir por HTTPS. | Alta |
| RNF-02 | La aplicación debe estar pensada primero para el celular (Mobile First) y poder instalarse como PWA. | Alta |
| RNF-03 | La consulta de Kudos recibidos debe ser rápida aunque muchas personas lean a la vez (carga de lectura). | Alta |
| RNF-04 | Si falla un canal de aviso, el Kudo debe publicarse igual y los otros canales deben entregarse. | Alta |
| RNF-05 | Las claves de los usuarios deben guardarse cifradas (BCrypt) y la sesión debe vencer sola. | Alta |
| RNF-06 | Ningún secreto puede estar en el código ni en el repositorio; se usan Azure Key Vault y GitHub Secrets. | Alta |
| RNF-07 | Ningún texto ni dato de configuración puede estar quemado en el código: los textos vienen de Strapi y la configuración de variables de entorno. | Alta |
| RNF-08 | El código debe seguir arquitectura limpia, principios SOLID y los 12 factores, y ser explicable por una sola persona. | Alta |
| RNF-09 | El pipeline debe correr las pruebas, SonarCloud y Trivy, y desplegar solo cuando todo pasa. | Alta |
| RNF-10 | La API debe quedar detrás de un API Gateway y un WAF administrados, y el WAF debe bloquear inyecciones. | Alta |
| RNF-11 | El administrador debe poder saber si la API está viva y leer sus registros. | Media |

---

## Requisitos de información

| Código | Entidad | Datos que maneja |
|--------|---------|------------------|
| RI-01 | Usuario | Nombre, correo, teléfono, rol, estado de la cuenta (pendiente, activa o rechazada), clave cifrada. |
| RI-02 | Kudo | Autor, destinatario, mensaje, fecha de creación y sus adjuntos. |
| RI-03 | Adjunto | Nombre original, tipo, tamaño y ruta donde quedó guardado. |
| RI-04 | Sesión | Token aleatorio y usuario al que pertenece, con vencimiento (se guarda en Redis, no en la base de datos). |
| RI-05 | Trabajo de notificación | Kudo, canal y número de intentos (vive en la cola de Redis). |
| RI-06 | Auditoría | Evento, canal, estado (OK, OMITIDO o ERROR), detalle y fecha. |
| RI-07 | Catálogos | Mensajes de la aplicación y plantillas de cada notificación (viven en Strapi). |

---

## Atributos de calidad priorizados

Antes de diseñar quise saber qué le importa a cada persona. Cada tipo de persona repartió los números del 1 al 10 entre los diez atributos, usando cada número una sola vez: 10 es lo más importante para esa persona y 1, lo menos importante. Después sumé las columnas y calculé el peso de cada atributo.

**Importante:** los números los puse yo, pensando en lo que le importaría a cada uno. No hice encuestas, así que están por validar.

### Puntos que dio cada persona

| Atributo | Estudiante | Docente | Coordinadores y directivos | Administrador del sistema | Evaluador (docente del curso) | Total |
|---|---|---|---|---|---|---|
| Rendimiento | 9 | 8 | 4 | 2 | 5 | 28 |
| Usabilidad móvil | 10 | 10 | 6 | 4 | 4 | 34 |
| Seguridad | 6 | 6 | 10 | 10 | 9 | 41 |
| Confiabilidad | 7 | 9 | 8 | 9 | 6 | 39 |
| Capacidad de ser auditado | 3 | 3 | 9 | 7 | 7 | 29 |
| Disponibilidad | 8 | 7 | 7 | 6 | 1 | 29 |
| Interoperabilidad | 4 | 5 | 3 | 1 | 3 | 16 |
| Mantenibilidad | 1 | 1 | 1 | 5 | 10 | 18 |
| Escalabilidad | 5 | 4 | 5 | 3 | 2 | 19 |
| Observabilidad | 2 | 2 | 2 | 8 | 8 | 22 |
| **Total** | 55 | 55 | 55 | 55 | 55 | 275 |

### Resultado de la priorización

| Prioridad | Atributo | Puntaje total | Ponderado global | Por qué importa |
|---|---|---|---|---|
| 1 | Seguridad | 41 | 0.1491 | Una plataforma donde se publica contenido de personas reales no puede dejar entrar a cualquiera ni aceptar archivos peligrosos. Aquí solo entra quien el administrador aprobó, las claves van cifradas y los adjuntos se validan. |
| 2 | Confiabilidad | 39 | 0.1418 | El corazón del sistema es avisar. Si Gmail o Twilio fallan, el Kudo no puede perderse ni la publicación quedar a medias. |
| 3 | Usabilidad móvil | 34 | 0.1236 | Se estima que el 90% entra desde el celular. Si publicar un Kudo es incómodo en una pantalla pequeña, nadie lo usa. |
| 4 | Capacidad de ser auditado | 29 | 0.1055 | Coordinadores y directivos quieren poder responder qué pasó con un Kudo y con cada aviso, incluso cuando algo falló. |
| 4 | Disponibilidad | 29 | 0.1055 | Si la API se cae y no vuelve sola, el docente no puede probar el sistema. Se busca que la plataforma se recupere sin intervención. |
| 6 | Rendimiento | 28 | 0.1018 | Es una plataforma de mucha lectura: todos consultan sus Kudos. La lista tiene que salir rápido aunque lean muchos a la vez. |
| 7 | Observabilidad | 22 | 0.0800 | Para arreglar algo hay que saber que está roto. El administrador necesita saber si la API está viva y poder leer sus registros. |
| 8 | Escalabilidad | 19 | 0.0691 | Si la plataforma crece, debería poder atender a más personas. Hoy se queda corta porque el push guarda a los conectados en memoria. |
| 9 | Mantenibilidad | 18 | 0.0655 | En la sustentación hay cambios de código en vivo. El sistema tiene que ser fácil de entender y de modificar. |
| 10 | Interoperabilidad | 16 | 0.0582 | Cambiar de proveedor de correo o de WhatsApp no debería tocar el negocio. A los usuarios no se les nota, pero a quien mantiene el sistema sí. |

### Escenarios seleccionados para desarrollar

Para cada atributo pensé situaciones concretas que le pasarían a alguien. Cada situación tiene un **impacto** y un **riesgo** del 1 al 3 (3 es lo más alto) y su puntaje es impacto por riesgo. No podía desarrollar y probar todas, así que apliqué unas reglas de selección, que están más abajo. Esta es la lista completa de lo que se me ocurrió y si entró o no.

| Código | Atributo | Escenario | Impacto | Riesgo | Puntaje | Entra |
|---|---|---|---|---|---|---|
| ESC-CAL-REN-0001 | Rendimiento | Ver los Kudos recibidos rápido aunque muchos consulten a la vez | 3 | 3 | 9 | Sí |
| ESC-CAL-CON-0001 | Confiabilidad | Un canal de aviso caído no tumba la publicación ni a los otros canales | 3 | 3 | 9 | Sí |
| ESC-CAL-SEG-0003 | Seguridad | Subir un archivo peligroso o demasiado grande como adjunto | 3 | 3 | 9 | Sí |
| ESC-CAL-SEG-0004 | Seguridad | Enviar una inyección (SQL o script) contra la API pública | 3 | 3 | 9 | Sí |
| ESC-CAL-USA-0001 | Usabilidad móvil | Publicar un Kudo desde el celular sin salir de la pantalla principal | 3 | 2 | 6 | Sí |
| ESC-CAL-SEG-0002 | Seguridad | Usar la API sin sesión o con una sesión vencida | 3 | 2 | 6 | No |
| ESC-CAL-AUD-0001 | Capacidad de ser auditado | Saber qué pasó con cada Kudo y con cada canal de aviso | 3 | 2 | 6 | Sí |
| ESC-CAL-SEG-0001 | Seguridad | Que solo entren las cuentas que el administrador aprobó | 3 | 2 | 6 | Sí |
| ESC-CAL-SEG-0005 | Seguridad | Que las claves y los secretos no queden en el repositorio | 3 | 2 | 6 | No |
| ESC-CAL-MAN-0001 | Mantenibilidad | Cambiar un texto de la aplicación sin tocar código ni redesplegar | 3 | 2 | 6 | Sí |
| ESC-CAL-MAN-0002 | Mantenibilidad | Agregar un canal de aviso nuevo en vivo, creando solo una clase | 3 | 2 | 6 | Sí |
| ESC-CAL-REN-0002 | Rendimiento | Recorrer una lista larga de Kudos sin que se ponga lenta | 2 | 2 | 4 | No |
| ESC-CAL-REN-0003 | Rendimiento | Enterarse del Kudo al instante cuando está conectado | 2 | 2 | 4 | No |
| ESC-CAL-CON-0002 | Confiabilidad | Si el catálogo de textos no responde, la aplicación sigue funcionando | 2 | 2 | 4 | No |
| ESC-CAL-ESCL-0001 | Escalabilidad | Que lean muchos más usuarios y se puedan levantar más copias de la API | 2 | 2 | 4 | No |
| ESC-CAL-OBS-0001 | Observabilidad | Saber si la API está viva y ver sus registros | 2 | 2 | 4 | Sí |
| ESC-CAL-DIS-0001 | Disponibilidad | Que la API vuelva a levantarse sola si el contenedor se reinicia | 2 | 2 | 4 | Sí |
| ESC-CAL-MAN-0003 | Mantenibilidad | Cambiar en vivo una regla de negocio, por ejemplo el largo del mensaje | 2 | 2 | 4 | Sí |
| ESC-CAL-USA-0002 | Usabilidad móvil | Pedir la cuenta escribiendo solo 10 números del celular, sin el prefijo | 2 | 1 | 2 | No |
| ESC-CAL-INT-0001 | Interoperabilidad | Cambiar de proveedor de correo sin tocar la lógica del negocio | 2 | 1 | 2 | No |

#### Reglas de selección

| Regla | Nombre | Qué dice |
|---|---|---|
| R1 | Medible | El escenario tiene que poder comprobarse con un número, una respuesta o una prueba. |
| R2 | Puntaje | Se prefieren los de mayor puntaje (impacto por riesgo). |
| R3 | Cobertura | Cada uno de los seis atributos que más pesan tiene al menos un escenario. |
| R4 | Máximo por atributo | Ningún atributo tiene más de tres escenarios, para que no se llene de seguridad. |
| R5 | Lo que pide el enunciado | Entra al menos un escenario de mantenibilidad (hay cambios de código en vivo) y uno de observabilidad. |
| R6 | Desempate | Si hay empate, gana el que toca el flujo de Publicar Kudo. |
| R7 | Tope | Máximo 12 escenarios. |

---

## Caracterización de atributos de calidad

Cada atributo se baja a características concretas (CAR) y cada característica se comprueba con uno o más escenarios.

### Seguridad

| Código característica | Descripción característica | Categoría | Código de escenario | Descripción escenario |
|---|---|---|---|---|
| CAR-SEG-0001 | El sistema debe dejar entrar únicamente a las cuentas que el administrador aprobó. | Autenticación y autorización | ESC-CAL-SEG-0001 | Solo entran las cuentas aprobadas |
| CAR-SEG-0002 | Los archivos adjuntos deben validarse por tipo, tamaño y cantidad, y guardarse con un nombre aleatorio. | Validación de la entrada | ESC-CAL-SEG-0003 | Subida segura de archivos |
| CAR-SEG-0003 | Las peticiones con inyecciones deben bloquearse antes de llegar a la API. | Protección del perímetro | ESC-CAL-SEG-0004 | Inyección contra la API pública |

### Confiabilidad

| Código característica | Descripción característica | Categoría | Código de escenario | Descripción escenario |
|---|---|---|---|---|
| CAR-CON-0001 | El fallo de un canal de aviso no debe impedir publicar el Kudo ni afectar a los demás canales. | Tolerancia a fallos | ESC-CAL-CON-0001 | Un canal caído no tumba a los demás |

### Usabilidad móvil

| Código característica | Descripción característica | Categoría | Código de escenario | Descripción escenario |
|---|---|---|---|---|
| CAR-USA-0001 | Publicar un Kudo debe poder hacerse desde el celular, desde la pantalla principal y sin hacer zoom. | Facilidad de operación | ESC-CAL-USA-0001 | Publicar un Kudo desde el celular |

### Capacidad de ser auditado

| Código característica | Descripción característica | Categoría | Código de escenario | Descripción escenario |
|---|---|---|---|---|
| CAR-AUD-0001 | Cada Kudo creado y cada aviso enviado deben dejar un registro con su estado. | Trazabilidad | ESC-CAL-AUD-0001 | Rastro de cada Kudo y de cada aviso |

### Disponibilidad

| Código característica | Descripción característica | Categoría | Código de escenario | Descripción escenario |
|---|---|---|---|---|
| CAR-DIS-0001 | Si el contenedor de la API se reinicia, debe volver a funcionar sin que nadie intervenga. | Recuperación | ESC-CAL-DIS-0001 | La API se recupera sola |

### Rendimiento

| Código característica | Descripción característica | Categoría | Código de escenario | Descripción escenario |
|---|---|---|---|---|
| CAR-REN-0001 | La lista de Kudos recibidos debe responder rápido bajo mucha lectura. | Consulta de información | ESC-CAL-REN-0001 | Consultar los Kudos recibidos rápido |

### Mantenibilidad

| Código característica | Descripción característica | Categoría | Código de escenario | Descripción escenario |
|---|---|---|---|---|
| CAR-MAN-0001 | Los textos de la aplicación deben poder cambiarse sin tocar código ni redesplegar. | Datos fuera del código | ESC-CAL-MAN-0001 | Cambiar un texto sin tocar código |
| CAR-MAN-0002 | Agregar un canal de aviso nuevo debe requerir crear una sola clase. | Extensibilidad | ESC-CAL-MAN-0002 | Agregar un canal de aviso nuevo |
| CAR-MAN-0003 | Las reglas del negocio deben vivir en un solo lugar para poder cambiarlas rápido. | Reglas aisladas | ESC-CAL-MAN-0003 | Cambiar una regla de negocio en vivo |

### Observabilidad

| Código característica | Descripción característica | Categoría | Código de escenario | Descripción escenario |
|---|---|---|---|---|
| CAR-OBS-0001 | El administrador debe poder saber si la API está viva y leer sus registros. | Monitoreo | ESC-CAL-OBS-0001 | Saber si la API está viva y ver sus registros |


---

## Escenarios de calidad

Cada escenario tiene una ficha con sus datos generales y una tabla con las seis partes que lo describen: fuente del estímulo, estímulo, ambiente, artefacto, respuesta y medida de la respuesta. El campo **Estado** es honesto: *Hecho* si está implementado y probado, *Parcial* si falta algo, *Pendiente* si todavía no lo monté y *Por verificar* si lo hice pero no lo he medido.

### Seguridad

#### ESC-CAL-SEG-0001

| Campo | Valor |
|---|---|
| **Código** | ESC-CAL-SEG-0001 |
| **Nombre** | Solo entran las cuentas aprobadas |
| **Atributo de calidad** | Seguridad |
| **Categoría** | Autenticación y autorización |
| **Característica** | El sistema debe dejar entrar únicamente a las cuentas que el administrador aprobó. |
| **Objetivo** | Impedir que entre alguien cuya solicitud no fue aprobada, y decirle claramente por qué no puede entrar. |
| **Criterios de éxito** | No se emite ninguna sesión y la persona ve cuál es su caso. |
| **Prerrequisitos** | 1. La persona no tiene cuenta, o su solicitud está pendiente o rechazada.<br>2. La API está disponible. |
| **Requisito relacionado** | RF-03 / RF-04 / RF-06 / RF-07 |
| **Tipo de escenario** | Restricción |
| **Estado** | Hecho (prueba unitaria `IniciarSesionTest`) |

| Fuente del estímulo | Estímulo | Ambiente | Artefacto | Respuesta | Medida de la respuesta |
|---|---|---|---|---|---|
| Persona sin cuenta aprobada | Intenta ingresar con un correo y una clave. | Operación normal | Caso de uso `IniciarSesion` | Rechaza el ingreso y responde con el texto del catálogo según el caso (sin cuenta, pendiente o rechazada). | No se crea ningún token en Redis y la respuesta es 404 o 401 con el mensaje del caso. |

---

#### ESC-CAL-SEG-0003

| Campo | Valor |
|---|---|
| **Código** | ESC-CAL-SEG-0003 |
| **Nombre** | Subida segura de archivos |
| **Atributo de calidad** | Seguridad |
| **Categoría** | Validación de la entrada |
| **Característica** | Los archivos adjuntos deben validarse por tipo, tamaño y cantidad, y guardarse con un nombre aleatorio. |
| **Objetivo** | Evitar archivos peligrosos o demasiado grandes |
| **Criterios de éxito** | Solo se aceptan los tipos y tamaños permitidos |
| **Prerrequisitos** | `ARCHIVOS_TIPOS_PERMITIDOS` y `ARCHIVOS_TAMANO_MAXIMO` configuradas |
| **Requisito relacionado** | RF-12 |
| **Tipo de escenario** | Fallo controlado |
| **Estado** | Hecho (prueba de integración del archivo no permitido) |

| Fuente del estímulo | Estímulo | Ambiente | Artefacto | Respuesta | Medida de la respuesta |
|---|---|---|---|---|---|
| Usuario | Intenta subir un ejecutable o un archivo de 50 MB | Operación normal | `KudosControlador` y `AlmacenArchivos` | Rechaza el archivo con error 400; los permitidos se guardan con nombre aleatorio | Un `.exe` o un archivo mayor al límite (5 MB) nunca llega a la carpeta de subidas |

---

#### ESC-CAL-SEG-0004

| Campo | Valor |
|---|---|
| **Código** | ESC-CAL-SEG-0004 |
| **Nombre** | Inyección contra la API pública |
| **Atributo de calidad** | Seguridad |
| **Categoría** | Protección del perímetro |
| **Característica** | Las peticiones con inyecciones deben bloquearse antes de llegar a la API. |
| **Objetivo** | Bloquear ataques de inyección antes de que lleguen a la API |
| **Criterios de éxito** | Las peticiones maliciosas no llegan a la aplicación |
| **Prerrequisitos** | WAF delante de la API, en modo prevención y con las reglas OWASP |
| **Requisito relacionado** | RNF-10 |
| **Tipo de escenario** | Restricción |
| **Estado** | Pendiente (falta montar el WAF) |

| Fuente del estímulo | Estímulo | Ambiente | Artefacto | Respuesta | Medida de la respuesta |
|---|---|---|---|---|---|
| Atacante externo | Envía una petición con una inyección SQL o un script | Producción en Azure | Azure Application Gateway con WAF y Azure API Management | El WAF detecta el patrón y devuelve 403; la petición no llega a la API | Una petición con `' OR 1=1--` en un parámetro responde 403 y aparece en los registros del WAF |

---

### Confiabilidad

#### ESC-CAL-CON-0001

| Campo | Valor |
|---|---|
| **Código** | ESC-CAL-CON-0001 |
| **Nombre** | Un canal caído no tumba a los demás |
| **Atributo de calidad** | Confiabilidad |
| **Categoría** | Tolerancia a fallos |
| **Característica** | El fallo de un canal de aviso no debe impedir publicar el Kudo ni afectar a los demás canales. |
| **Objetivo** | Que el Kudo se publique aunque falle el correo o WhatsApp |
| **Criterios de éxito** | El Kudo queda guardado y los canales sanos se entregan |
| **Prerrequisitos** | Cola en Redis activa |
| **Requisito relacionado** | RF-14 / RF-15 / RF-16 / RNF-04 |
| **Tipo de escenario** | Fallo controlado |
| **Estado** | Hecho (`EnviarNotificacionTest`) |

| Fuente del estímulo | Estímulo | Ambiente | Artefacto | Respuesta | Medida de la respuesta |
|---|---|---|---|---|---|
| Proveedor externo (Gmail o Twilio) | El proveedor no responde o devuelve error | Operación normal | `ProcesadorNotificaciones`, `EnviarNotificacion` y la cola en Redis | Cada canal es un trabajo aparte con 3 reintentos y espera creciente; el error queda en la auditoría | La API responde 201 aunque falle un canal, y la tabla `auditoria` muestra el estado ERROR de ese canal |

---

### Usabilidad móvil

#### ESC-CAL-USA-0001

| Campo | Valor |
|---|---|
| **Código** | ESC-CAL-USA-0001 |
| **Nombre** | Publicar un Kudo desde el celular |
| **Atributo de calidad** | Usabilidad móvil |
| **Categoría** | Facilidad de operación |
| **Característica** | Publicar un Kudo debe poder hacerse desde el celular, desde la pantalla principal y sin hacer zoom. |
| **Objetivo** | Que un estudiante publique un Kudo con un adjunto desde el celular sin esfuerzo |
| **Criterios de éxito** | Publica sin hacer zoom ni desplazarse de lado |
| **Prerrequisitos** | Aplicación publicada por HTTPS y sesión iniciada |
| **Requisito relacionado** | RF-11 / RF-12 / RNF-02 |
| **Tipo de escenario** | Éxito |
| **Estado** | Por verificar (falta medirlo en un celular real) |

| Fuente del estímulo | Estímulo | Ambiente | Artefacto | Respuesta | Medida de la respuesta |
|---|---|---|---|---|---|
| Estudiante en el celular, con datos móviles | Abre la aplicación y publica un Kudo con un adjunto | Operación normal, pantalla pequeña y red móvil | Frontend (`FormularioKudo` y `estilos.css`) y `POST /kudos` | Muestra el formulario ajustado a la pantalla, envía el Kudo y confirma sin recargar la página | El formulario está en la pantalla principal y se ve completo, sin desplazamiento horizontal, desde 360 px de ancho |

---

### Capacidad de ser auditado

#### ESC-CAL-AUD-0001

| Campo | Valor |
|---|---|
| **Código** | ESC-CAL-AUD-0001 |
| **Nombre** | Rastro de cada Kudo y de cada aviso |
| **Atributo de calidad** | Capacidad de ser auditado |
| **Categoría** | Trazabilidad |
| **Característica** | Cada Kudo creado y cada aviso enviado deben dejar un registro con su estado. |
| **Objetivo** | Poder saber qué pasó con un Kudo y con cada canal |
| **Criterios de éxito** | Cada paso importante queda registrado |
| **Prerrequisitos** | Base de datos disponible |
| **Requisito relacionado** | RF-17 |
| **Tipo de escenario** | Éxito |
| **Estado** | Hecho (prueba de integración de publicar con adjunto) |

| Fuente del estímulo | Estímulo | Ambiente | Artefacto | Respuesta | Medida de la respuesta |
|---|---|---|---|---|---|
| Coordinador o administrador | Pregunta qué pasó con un Kudo publicado | Operación normal | `AuditoriaRepositorioPostgres`, `PublicarKudo` y `EnviarNotificacion` | Guarda `KUDO_CREADO` al publicar y una fila `NOTIFICACION` por cada canal, con su estado | Después de publicar, la tabla `auditoria` tiene una fila `KUDO_CREADO` y una por canal (correo, whatsapp y push) con OK, ERROR u OMITIDO |

---

### Disponibilidad

#### ESC-CAL-DIS-0001

| Campo | Valor |
|---|---|
| **Código** | ESC-CAL-DIS-0001 |
| **Nombre** | La API se recupera sola |
| **Atributo de calidad** | Disponibilidad |
| **Categoría** | Recuperación |
| **Característica** | Si el contenedor de la API se reinicia, debe volver a funcionar sin que nadie intervenga. |
| **Objetivo** | Que si el contenedor se reinicia, la API vuelva sin que nadie intervenga |
| **Criterios de éxito** | La API responde de nuevo por sí sola |
| **Prerrequisitos** | Azure Container Apps con una réplica |
| **Requisito relacionado** | RNF-01 |
| **Tipo de escenario** | Fallo controlado |
| **Estado** | Por verificar (todavía no lo he provocado) |

| Fuente del estímulo | Estímulo | Ambiente | Artefacto | Respuesta | Medida de la respuesta |
|---|---|---|---|---|---|
| La plataforma (reinicio o falla del contenedor) | El contenedor de la API se reinicia | Producción en Azure | Azure Container Apps y `GET /salud` | La plataforma levanta otro contenedor con la misma imagen y las mismas variables | `GET /salud` vuelve a responder `ok` en menos de 2 minutos, sin intervención |

---

### Rendimiento

#### ESC-CAL-REN-0001

| Campo | Valor |
|---|---|
| **Código** | ESC-CAL-REN-0001 |
| **Nombre** | Consultar los Kudos recibidos rápido |
| **Atributo de calidad** | Rendimiento |
| **Categoría** | Consulta de información |
| **Característica** | La lista de Kudos recibidos debe responder rápido bajo mucha lectura. |
| **Objetivo** | Que ver los Kudos recibidos sea rápido aunque haya mucha gente consultando |
| **Criterios de éxito** | La consulta repetida no vuelve a golpear la base de datos |
| **Prerrequisitos** | Redis disponible |
| **Requisito relacionado** | RF-18 / RNF-03 |
| **Tipo de escenario** | Éxito |
| **Estado** | Hecho (`ListarKudosTest`) |

| Fuente del estímulo | Estímulo | Ambiente | Artefacto | Respuesta | Medida de la respuesta |
|---|---|---|---|---|---|
| Usuario desde el celular | Pide la lista de sus Kudos varias veces seguidas | Operación normal con alta carga de lectura | Caso de uso `ListarKudos` y `CacheKudosRedis` | Devuelve la lista desde la caché si existe; si no, consulta la base y la guarda | La segunda consulta idéntica dentro de la vida de la caché (60 s) no ejecuta consulta SQL |

---

### Mantenibilidad

#### ESC-CAL-MAN-0001

| Campo | Valor |
|---|---|
| **Código** | ESC-CAL-MAN-0001 |
| **Nombre** | Cambiar un texto sin tocar código |
| **Atributo de calidad** | Mantenibilidad |
| **Categoría** | Datos fuera del código |
| **Característica** | Los textos de la aplicación deben poder cambiarse sin tocar código ni redesplegar. |
| **Objetivo** | Poder corregir un mensaje de la aplicación sin redesplegar |
| **Criterios de éxito** | El texto nuevo aparece sin recompilar |
| **Prerrequisitos** | Strapi arriba y el catálogo sembrado |
| **Requisito relacionado** | RF-21 / RNF-07 |
| **Tipo de escenario** | Éxito |
| **Estado** | Hecho |

| Fuente del estímulo | Estímulo | Ambiente | Artefacto | Respuesta | Medida de la respuesta |
|---|---|---|---|---|---|
| Administrador | Edita un mensaje en el panel de Strapi | Operación normal | Módulo `catalogos`, `StrapiCliente` y Strapi | El backend lee el texto nuevo cuando vence su caché | El texto nuevo se ve en menos de `CACHE_SEGUNDOS` (60 s) sin tocar Java ni redesplegar |

---

#### ESC-CAL-MAN-0002

| Campo | Valor |
|---|---|
| **Código** | ESC-CAL-MAN-0002 |
| **Nombre** | Agregar un canal de aviso nuevo |
| **Atributo de calidad** | Mantenibilidad |
| **Categoría** | Extensibilidad |
| **Característica** | Agregar un canal de aviso nuevo debe requerir crear una sola clase. |
| **Objetivo** | Poder sumar, por ejemplo, Telegram sin tocar la lógica del negocio |
| **Criterios de éxito** | Solo se agrega una clase |
| **Prerrequisitos** | Contrato `Notificador` definido |
| **Requisito relacionado** | RNF-08 |
| **Tipo de escenario** | Éxito |
| **Estado** | Hecho |

| Fuente del estímulo | Estímulo | Ambiente | Artefacto | Respuesta | Medida de la respuesta |
|---|---|---|---|---|---|
| Desarrollador | Se pide un canal nuevo | Desarrollo | Interfaz `Notificador` y `EnviarNotificacion` | Se crea una clase que cumple el contrato y Spring la registra | `PublicarKudo` y `EnviarNotificacion` no cambian |

---

#### ESC-CAL-MAN-0003

| Campo | Valor |
|---|---|
| **Código** | ESC-CAL-MAN-0003 |
| **Nombre** | Cambiar una regla de negocio en vivo |
| **Atributo de calidad** | Mantenibilidad |
| **Categoría** | Reglas aisladas |
| **Característica** | Las reglas del negocio deben vivir en un solo lugar para poder cambiarlas rápido. |
| **Objetivo** | Poder cambiar en la sustentación una regla simple, por ejemplo el largo máximo del mensaje. |
| **Criterios de éxito** | El cambio se hace en un solo lugar y las pruebas lo confirman. |
| **Prerrequisitos** | 1. El proyecto compila y las pruebas pasan. |
| **Requisito relacionado** | RF-11 / RNF-08 |
| **Tipo de escenario** | Éxito |
| **Estado** | Hecho. El límite está en una constante (`LIMITE_CARACTERES = 500`); queda como pendiente pasarlo a una variable de entorno y quitar el tope (RT-16) |

| Fuente del estímulo | Estímulo | Ambiente | Artefacto | Respuesta | Medida de la respuesta |
|---|---|---|---|---|---|
| Evaluador (docente del curso) | Pide cambiar el largo máximo del mensaje de un Kudo. | Desarrollo, durante la sustentación | Clase `ReglasMensaje` y `ReglasMensajeTest` | Se edita una sola constante y se vuelven a correr las pruebas. | El cambio toca una clase de negocio y una prueba; no toca controladores ni base de datos. |

---

### Observabilidad

#### ESC-CAL-OBS-0001

| Campo | Valor |
|---|---|
| **Código** | ESC-CAL-OBS-0001 |
| **Nombre** | Saber si la API está viva y ver sus registros |
| **Atributo de calidad** | Observabilidad |
| **Categoría** | Monitoreo |
| **Característica** | El administrador debe poder saber si la API está viva y leer sus registros. |
| **Objetivo** | Que el administrador se entere si algo falla |
| **Criterios de éxito** | Puede comprobar la salud y leer los registros |
| **Prerrequisitos** | API desplegada |
| **Requisito relacionado** | RF-22 / RNF-11 |
| **Tipo de escenario** | Éxito |
| **Estado** | Parcial (no tengo métricas ni alertas) |

| Fuente del estímulo | Estímulo | Ambiente | Artefacto | Respuesta | Medida de la respuesta |
|---|---|---|---|---|---|
| Administrador | Quiere saber si la API está funcionando | Producción en Azure | `GET /salud`, tabla `auditoria` y registros por la salida estándar | `/salud` responde `ok` y los registros se leen en el Flujo de registros del contenedor | `/salud` responde `{"estado":"ok"}` en menos de 1 segundo |

---


## Restricciones de negocio

| Tipo | Restricción de negocio | Justificación | Plan de acción |
|---|---|---|---|
| Humano | El equipo es una sola persona y es un trabajo académico. Nadie queda dando soporte después de la entrega. | Si el diseño fuera muy complejo, no habría quién lo explicara ni lo mantuviera. | Monolito modular, una sola cola y nada de microservicios, con todo documentado. |
| Tiempo | Entrega y sistema desplegado el viernes 13 de noviembre de 2026 a las 12:00 m., con sustentación y cambios de código en vivo. | Es un proyecto con fechas fijas de la universidad. | Se priorizó el flujo completo de Publicar Kudo y las cuentas; lo demás quedó diseñado y documentado. |
| Tiempo | Cuatro días para diseñar, construir, desplegar y documentar. | El tiempo no alcanza para todo. | Se hizo primero lo que guía el diseño y el flujo principal. |
| Presupuesto | Cuenta Azure for Students con 100 USD de crédito. | Un servicio caro gasta el crédito antes de la sustentación. | Una sola réplica y los tamaños más pequeños; se borran los recursos costosos después de evaluar. |
| Organizacional | No hay dominio propio para enviar correo ni acceso al tenant institucional de Microsoft. | No se puede usar el inicio de sesión de la universidad ni un correo con dominio propio. | El correo sale por Gmail con contraseña de aplicación; las cuentas las pide la persona y las aprueba el administrador. |
| Proceso | Los números de celular para pruebas son limitados (sandbox de WhatsApp). | Solo reciben mensajes los celulares que se unieron al sandbox de Twilio, con el mensaje `join twilio-trial`, durante unas 72 horas. | Se prueba con los celulares unidos y se documenta la limitación. |
| Proceso | El enunciado pide que los Kudos y las réplicas no tengan límite de longitud, profundidad ni cantidad (RT-16). | Es una regla pedida por el docente. | **No se cumplió del todo:** el mensaje tiene un límite de 500 caracteres y todavía no hay réplicas. Está en los pendientes. |
| Tecnológico | Primera vez que despliego en Azure Container Apps con Key Vault, Redis administrado y credenciales federadas. | Varias decisiones cambiaron durante el despliegue por políticas de la cuenta de estudiante. | Se documentaron los problemas y las soluciones en la sección de despliegue. |

---

## Restricciones técnicas

| Tipo | Categoría | Restricción técnica | Cómo se resolvió |
|---|---|---|---|
| Impuesta por el enunciado | Tecnología base | Backend en Java con Spring Boot. | Se usó Java 21, Spring Boot 3.3 y Maven (ADR-03). |
| Impuesta por el enunciado | Arquitectura | Clean Architecture, análisis del Manifiesto de Sistemas Reactivos y matriz de los 12 factores. | El dominio no depende de frameworks y todo está explicado en las secciones de diseño y de sistemas reactivos. |
| Impuesta por el enunciado | Funcional | Publicar un Kudo guarda, envía correo, envía WhatsApp, envía push si el destinatario está conectado y audita. | Son cinco efectos que se coordinan con una cola; el guardado no depende de los envíos. |
| Impuesta por el enunciado | Datos | Los Kudos admiten archivos adjuntos. | Se guarda el archivo aparte y en la base solo su nombre, tipo, tamaño y ruta. |
| Impuesta por el enunciado | Experiencia de usuario | Mobile First: se estima que el 90% entra desde el celular. | Frontend responsivo y liviano (PWA) con respuestas paginadas. |
| Impuesta por el enunciado | Rendimiento | Carga predominantemente de lectura. | Caché en Redis, índice en la base y paginación por cursor. |
| Impuesta por el enunciado | Calidad | Pruebas unitarias, de integración y de extremo a extremo; SonarCloud y Trivy en el pipeline. | Los análisis corren en el CI y frenan el pipeline si fallan. |
| Impuesta por el enunciado | Seguridad | API Gateway y WAF como servicios ya hechos que se consumen (no se programan); el WAF debe bloquear inyecciones. | Azure API Management y Application Gateway con WAF. Siguen pendientes de montar. |
| Impuesta por el enunciado | Despliegue | URL pública funcional con HTTPS y el flujo usable por el docente. | Azure Container Apps y Static Web Apps entregan HTTPS; PostgreSQL y Redis se conectan con TLS. |
| Impuesta por el entorno | Nube | Azure for Students solo permite cinco regiones (Chile Central, North Central US, South Central US, West US y West US 2). | El App Service B1 tenía cuota 0 y PostgreSQL estaba bloqueado en South Central US. Terminé con Container Apps y recursos repartidos entre West US 2 y North Central US. |
| Propia del proyecto | Seguridad | Las claves y los secretos no pueden estar en el código ni en el repositorio. | Azure Key Vault para los secretos en ejecución y GitHub Secrets para el CI. |
| Propia del proyecto | Mantenibilidad | Nada quemado en el código: textos desde Strapi y configuración desde variables de entorno. | Módulo `catalogos` y clase `ConfiguracionKudos`. |
| Propia del proyecto | Prácticas de diseño | Principios SOLID, bajo acoplamiento y un puerto por cada integración externa. | Los canales de aviso se cambian sin tocar el dominio. |
| Propia del proyecto | Código limpio | Código simple, con nombres claros en español y sin comentarios. | Facilita explicarlo y cambiarlo en la sustentación. |

---

## Estilo y tácticas

### Estilo de arquitectura

Combiné tres ideas:

- **Monolito modular:** una sola aplicación con módulos separados (`kudos`, `notificaciones`, `auditoria`). Es lo más simple para un trabajo individual, y los módulos quedan listos para separarse si hace falta.
- **Arquitectura limpia (hexagonal):** el negocio no depende de la base de datos ni de los proveedores. Habla con puertos y las implementaciones se enchufan por fuera.
- **Orientada a eventos:** publicar un Kudo genera trabajos en una cola, y cada canal los consume por su cuenta.

### Tácticas por escenario

Para cada escenario escogí una o varias tácticas. La columna "Categoría" las clasifica por el atributo que ayudan a cumplir y la última dice en qué estado está cada una.

| Escenario | Táctica | Categoría | Dónde se aplica | Estado |
|---|---|---|---|---|
| ESC-CAL-AUD-0001 | Registro de auditoría por evento y por canal | Seguridad: dejar rastro | `AuditoriaRepositorioPostgres` | Hecho |
| ESC-CAL-CON-0001 | Un trabajo en la cola por cada canal | Disponibilidad: aislar fallos | `PublicadorEventosCola` | Hecho |
| ESC-CAL-CON-0001 | Reintentos con espera creciente | Disponibilidad: recuperación | `ProcesadorNotificaciones` | Hecho |
| ESC-CAL-CON-0001 | Procesamiento asíncrono: la API responde sin esperar los envíos | Rendimiento: desacoplar | Cola en Redis | Hecho |
| ESC-CAL-DIS-0001 | Reinicio automático del contenedor por la plataforma | Disponibilidad: recuperación | Azure Container Apps | Por verificar |
| ESC-CAL-DIS-0001 | Comprobación de salud | Disponibilidad: detectar fallos | `GET /salud` | Hecho |
| ESC-CAL-MAN-0001 | Textos fuera del código, en un catálogo externo | Modificabilidad: aplazar el enlace | Módulo `catalogos` y Strapi | Hecho |
| ESC-CAL-MAN-0001 | Configuración por variables de entorno | Modificabilidad: aplazar el enlace | `ConfiguracionKudos`, `application.yml` | Hecho |
| ESC-CAL-MAN-0002 | Puertos y adaptadores (Strategy y Adapter) | Modificabilidad: reducir el acoplamiento | `Notificador`, `EnviarNotificacion` | Hecho |
| ESC-CAL-MAN-0003 | Reglas del negocio aisladas en el dominio | Modificabilidad: aislar los cambios | `ReglasMensaje` | Hecho |
| ESC-CAL-OBS-0001 | Endpoint de salud y registros por la salida estándar | Observabilidad (Twelve-Factor, factor XI) | `SaludControlador`, Spring Boot | Parcial |
| ESC-CAL-REN-0001 | Caché cache-aside de la lista de cada destinatario | Rendimiento: mantener copias | `CacheKudosRedis` | Hecho |
| ESC-CAL-REN-0001 | Invalidar la caché cuando llega un Kudo nuevo | Rendimiento: evitar datos viejos | `PublicarKudo` | Hecho |
| ESC-CAL-REN-0001 | Paginación por cursor e índice en `kudo (destinatarioId, creadoEn)` | Rendimiento: limitar el trabajo por petición | `ListarKudos`, `KudoEntidad` | Hecho |
| ESC-CAL-SEG-0001 | Estado de la cuenta (pendiente, activa o rechazada) revisado antes de abrir la sesión | Seguridad: autenticar y autorizar | `IniciarSesion`, `EstadoCuenta` | Hecho |
| ESC-CAL-SEG-0001 | Claves cifradas con BCrypt y sesión opaca en Redis con vencimiento | Seguridad: proteger los datos | `CifradorClaves`, `SesionesRedis` | Hecho |
| ESC-CAL-SEG-0003 | Lista de tipos permitidos, tamaño máximo y cantidad máxima | Seguridad: validar la entrada | `AlmacenArchivos` | Hecho |
| ESC-CAL-SEG-0003 | Guardar con nombre aleatorio | Seguridad: resistir ataques | `AlmacenArchivos` | Hecho |
| ESC-CAL-SEG-0004 | WAF con reglas OWASP en modo prevención | Seguridad: resistir ataques | Application Gateway con WAF | Pendiente |
| ESC-CAL-SEG-0004 | Consultas parametrizadas y validación de entradas | Seguridad: resistir ataques | Spring Data JPA y Bean Validation | Hecho |
| ESC-CAL-USA-0001 | Formulario único en la pantalla principal | Usabilidad: reducir pasos | `FormularioKudo` | Hecho |
| ESC-CAL-USA-0001 | Diseño responsivo y PWA instalable (service worker y manifiesto) | Usabilidad: adaptarse al dispositivo | `estilos.css`, `sw.js`, `manifest.webmanifest` | Hecho |


---

## Tradeoffs de calidad

Hay atributos que se pelean entre sí: mejorar uno casi siempre cuesta algo del otro. Estas son las peleas que encontré y cómo resolví cada una.

| Código | Atributos en tensión | Tensión arquitectónica | Resolución | Impacto aceptado | Escenarios afectados |
|---|---|---|---|---|---|
| T-01 | Seguridad vs Usabilidad móvil | La sesión vence sola (`SESION_SEGUNDOS`): protege la cuenta, pero en el celular hay que volver a ingresar. | Prima la seguridad; la vida de la sesión se ajusta con una variable, sin tocar código. | Un paso extra de inicio de sesión cuando la sesión venció. | ESC-CAL-USA-0001, ESC-CAL-SEG-0001 |
| T-02 | Confiabilidad vs Rendimiento | Si esperara a que salgan el correo y el WhatsApp, la publicación sería lenta y frágil. | Primero guardo el Kudo y confirmo; los envíos van a una cola y se reintentan. | El aviso puede llegar con retraso, y hay que guardar el estado de cada envío en la auditoría. | ESC-CAL-CON-0001 |
| T-03 | Rendimiento vs Consistencia | La caché de 60 segundos (lista de Kudos y catálogo de textos) acelera, pero puede mostrar datos viejos. | Cacheo solo datos de baja volatilidad y borro la caché de la lista cuando llega un Kudo. | Un texto editado en Strapi puede tardar hasta 60 segundos en verse. | ESC-CAL-REN-0001, ESC-CAL-MAN-0001 |
| T-04 | Capacidad de ser auditado vs Rendimiento | Registrar cada paso da trazabilidad, pero cuesta una escritura por evento. | Audito los eventos importantes con datos mínimos, sin guardar el mensaje completo. | Más filas en la base de datos y una escritura extra por canal. | ESC-CAL-AUD-0001 |
| T-05 | Mantenibilidad vs Simplicidad | Separar en capas con puertos y adaptadores agrega clases y conversiones, pero permite cambiar proveedores y probar por separado. | Prima la mantenibilidad, porque en la sustentación hay cambios en vivo y pruebas. | Más código y conversiones entre capas. | ESC-CAL-MAN-0002 |
| T-06 | Disponibilidad vs Dependencia de proveedores externos | Gmail, Twilio y Strapi no están bajo mi control y el sandbox de pruebas tiene límites. | Cada canal queda detrás de un contrato propio; si uno cae, no detiene la publicación ni la lectura. | Se acepta que un canal falle y se reintente; dependo de los límites del proveedor. | ESC-CAL-CON-0001 |
| T-07 | Escalabilidad vs Tiempo real | El push (SSE) guarda en memoria quién está conectado, así que con varias copias de la API un aviso podría no llegar. | Dejo una sola réplica y anoto la solución: repartir los eventos por Redis. | La API no escala a varias copias sin cambiar el push. | ESC-CAL-REN-0001 |
| T-08 | Costo vs Disponibilidad | Con 100 USD de crédito no puedo pagar réplicas ni servicios grandes. | Una sola réplica (mínimo y máximo 1), PostgreSQL y Redis del tamaño más chico. | Si el contenedor se reinicia, hay unos segundos sin servicio. | ESC-CAL-DIS-0001 |
| T-09 | Seguridad vs Simplicidad del despliegue | Guardar los secretos en Key Vault y leerlos por referencia agrega pasos al despliegue. | Prima la seguridad: ninguna clave queda en el repositorio ni en el pipeline. | Más pasos de configuración la primera vez. | ESC-CAL-SEG-0004, ESC-CAL-DIS-0001 |


---

## Cómo correrlo

Esta parte es para quien quiera levantar el proyecto en su computador o entender cómo está configurado. Primero explico cómo funcionan las cuentas, luego las rutas de la API y por último cómo probar todo el flujo paso a paso.

### Cuentas

En UCO Kudos nadie entra sin que el administrador lo apruebe. Para solicitar la cuenta sirve cualquier correo que exista (uco.net.co, gmail.com, hotmail.com, etc.): la clave temporal llega a ese correo y al WhatsApp registrado. El teléfono se escribe con 10 números y el prefijo del país sale de `TELEFONO_PREFIJO`.

El flujo es este:

1. Quien no tiene cuenta usa "Solicitar cuenta" (nombre, correo, rol y teléfono). La solicitud queda `PENDIENTE` y al administrador le llega un correo.
2. El administrador entra al panel y aprueba o rechaza. Al aprobar se crea la cuenta con una clave temporal que le llega a la persona por correo (y por WhatsApp). Al rechazar le llega un correo avisando.
3. La persona ingresa con correo y clave. La clave se guarda con BCrypt y la sesión es un token opaco guardado en Redis que vence a las `SESION_SEGUNDOS`.
4. Si intenta ingresar sin cuenta, con la solicitud pendiente o rechazada, la pantalla le dice exactamente cuál de los tres casos es.

Después de entrar, cualquier persona (incluido el administrador) puede cambiar la clave temporal desde el botón "Cambiar clave": escribe la clave actual y la nueva dos veces, y la API la guarda cifrada con BCrypt (`PUT /sesion/clave`).

La cuenta del administrador se crea sola al arrancar con `ADMIN_NOMBRE`, `ADMIN_CORREO` y `ADMIN_CLAVE`, si todavía no existe.

### Endpoints

La API se puede entender en tres grupos.

**Lo que puede usar cualquiera, sin sesión:**

- `GET /salud` dice si la API está viva.
- `POST /cuentas/solicitudes` sirve para pedir una cuenta con nombre, correo, teléfono y rol.
- `POST /sesion` ingresa con correo y clave y devuelve el token.
- `GET /archivos/{nombre}` descarga un adjunto.

**Lo que puede usar cualquier persona con sesión:**

- `GET /sesion` devuelve los datos de quien tiene la sesión, y `DELETE /sesion` la cierra.
- `PUT /sesion/clave` cambia la clave.
- `GET /usuarios` lista los usuarios activos (solo id y nombre) para escoger a quién enviarle un Kudo.
- `POST /kudos` publica un Kudo. Es multipart: `destinatarioId`, `mensaje` y los archivos. El autor es quien tiene la sesión.
- `GET /kudos?limite=10&cursor=...` devuelve los Kudos recibidos, con paginación por cursor.
- `GET /push` abre las notificaciones en tiempo real (SSE). El token va en `?token=` porque el navegador no deja mandar cabeceras en un EventSource.

**Lo que solo puede usar el administrador:**

- `GET /cuentas/solicitudes` muestra las solicitudes pendientes.
- `POST /cuentas/solicitudes/{id}/aprobar` aprueba y manda la clave temporal.
- `POST /cuentas/solicitudes/{id}/rechazar` rechaza y avisa por correo.

Las rutas con sesión llevan la cabecera `Authorization: Bearer TOKEN`.

### Probar el flujo completo

Primero pido una cuenta (el correo tiene que cumplir el formato):

```
curl -X POST http://localhost:3000/cuentas/solicitudes -H "Content-Type: application/json" -d "{\"nombre\":\"Ana Gomez\",\"correo\":\"ana.gomez@gmail.com\",\"telefono\":\"+573000000001\",\"rol\":\"ESTUDIANTE\"}"
```

Ingreso como administrador (los datos de `ADMIN_CORREO` y `ADMIN_CLAVE` del `.env`) y copio el `token` de la respuesta:

```
curl -X POST http://localhost:3000/sesion -H "Content-Type: application/json" -d "{\"correo\":\"ADMIN_CORREO\",\"clave\":\"ADMIN_CLAVE\"}"
```

Veo las solicitudes pendientes y apruebo la de Ana con su `id`:

```
curl http://localhost:3000/cuentas/solicitudes -H "Authorization: Bearer TOKEN_ADMIN"
curl -X POST http://localhost:3000/cuentas/solicitudes/ID_DE_ANA/aprobar -H "Authorization: Bearer TOKEN_ADMIN"
```

A Ana le llega la clave temporal por correo. Repito el proceso con un segundo usuario (Luis) y los dos ingresan con `POST /sesion`. Con el token de Ana publico un Kudo para Luis (el `id` de Luis sale de `GET /usuarios`):

```
curl -X POST http://localhost:3000/kudos -H "Authorization: Bearer TOKEN_ANA" -F destinatarioId=ID_DE_LUIS -F mensaje="Gracias por ayudarme con el proyecto" -F "archivos=@nota.pdf;type=application/pdf"
```

Para ver los Kudos de Luis uso su token:

```
curl "http://localhost:3000/kudos?limite=10" -H "Authorization: Bearer TOKEN_LUIS"
```

Para ver la auditoría:

```
docker compose exec base-datos psql -U kudos -d kudos -c "select evento, canal, estado, detalle from auditoria order by creado_en"
```

Debe aparecer una fila `KUDO_CREADO` y una fila `NOTIFICACION` por cada canal (correo, whatsapp y push).

Para probar el push dejo este comando abierto en otra terminal antes de publicar:

```
curl -N "http://localhost:3000/push?token=TOKEN_LUIS"
```

Cuando publico el Kudo para Luis, esa terminal muestra un evento `kudo-recibido`. Si nadie está conectado, el canal push queda como OMITIDO en la auditoría.

### Catálogo (Strapi)

Los textos de los mensajes y las plantillas de las notificaciones viven en Strapi, un servicio aparte con su propia base de datos. Se levanta con Docker y se siembra solo la primera vez, desde `strapi/datos/mensajes.json` y `strapi/datos/notificaciones.json`.

```
docker compose up -d base-datos redis base-datos-strapi strapi
```

Tarda cerca de un minuto en arrancar. Cuando termine, `http://localhost:1337/api/mensajes` devuelve los textos. El panel para editarlos está en `http://localhost:1337/admin`: la primera vez pide crear el usuario administrador, y después se editan las colecciones Mensaje y Notificacion (el backend las lee cada 60 segundos, sin redesplegar).

### Correo y WhatsApp

**Correo (Gmail):** activo la verificación en dos pasos, creo una contraseña de aplicación y en `.env` pongo `CORREO_HOST=smtp.gmail.com`, `CORREO_PUERTO=587`, `CORREO_USUARIO` con mi correo, `CORREO_CLAVE` con la contraseña de aplicación y `CORREO_REMITENTE` con mi correo.

**WhatsApp (sandbox de Twilio):** creo una cuenta en Twilio, activo el sandbox de WhatsApp y mando el código de unión desde el celular que va a recibir los mensajes. En `.env` pongo `WHATSAPP_CUENTA_SID`, `WHATSAPP_TOKEN` y `WHATSAPP_NUMERO_ORIGEN`. El teléfono del destinatario tiene que ser el que se unió al sandbox, con el prefijo del país.

### Variables de entorno

Están todas en `.env.example`. Ninguna está escrita dentro del código.

| Variable | Para qué sirve |
|---|---|
| `PUERTO` | Puerto de la API |
| `BD_HOST`, `BD_PUERTO`, `BD_USUARIO`, `BD_CLAVE`, `BD_NOMBRE` | Conexión a PostgreSQL |
| `BD_SSLMODE` | `prefer` en local y `require` en Azure |
| `BD_ESQUEMA` | `update` crea y actualiza las tablas solo; `validate` solo revisa |
| `REDIS_HOST`, `REDIS_PUERTO`, `REDIS_CLAVE`, `REDIS_SSL` | Conexión a Redis (en Azure Managed Redis: puerto 10000, clave de acceso y SSL en `true`) |
| `CORREO_*` | Servidor SMTP |
| `WHATSAPP_*` | Cuenta de Twilio, la ruta de envío (`WHATSAPP_RUTA_ENVIO`) y el prefijo del número (`WHATSAPP_PREFIJO`) |
| `CANAL_CORREO`, `CANAL_WHATSAPP`, `CANAL_PUSH` | Nombre de cada canal, que también es la clave de su plantilla en el catálogo |
| `STRAPI_URL`, `STRAPI_TOKEN` | Dirección del catálogo y token opcional (la lectura es pública) |
| `STRAPI_RUTA_*`, `STRAPI_CAMPO_*` | Rutas y campos de las colecciones de Strapi (tienen valores por defecto) |
| `STRAPI_BD_*`, `STRAPI_APP_KEYS`, `STRAPI_*_SALT`, `STRAPI_*_SECRET`, `STRAPI_ENCRYPTION_KEY` | Base de datos y secretos del propio Strapi |
| `ARCHIVOS_CARPETA`, `ARCHIVOS_TAMANO_MAXIMO`, `ARCHIVOS_TIPOS_PERMITIDOS` | Reglas de los adjuntos |
| `ADMIN_NOMBRE`, `ADMIN_CORREO`, `ADMIN_TELEFONO`, `ADMIN_CLAVE` | Cuenta del administrador, que se crea sola al arrancar; su correo recibe los avisos de solicitudes |
| `TELEFONO_PREFIJO` | Prefijo del país que el sistema le agrega al teléfono de 10 números que escribe el usuario (por defecto `+57`) |
| `SESION_SEGUNDOS`, `SESION_PREFIJO` | Vida de la sesión en Redis y prefijo de sus llaves |
| `CLAVE_TEMPORAL_LONGITUD` | Largo de la clave temporal que se manda al aprobar |
| `CLAVE_LARGO_MINIMO` | Largo mínimo de la clave nueva cuando la persona la cambia (por defecto 8) |
| `PLANTILLA_SOLICITUD`, `PLANTILLA_APROBADA`, `PLANTILLA_RECHAZADA`, `PLANTILLA_APROBADA_WHATSAPP` | Nombres de las plantillas de cuentas en Strapi (tienen valores por defecto) |
| `CACHE_SEGUNDOS` | Vida de la caché |
| `CORS_ORIGENES` | Orígenes permitidos (en producción, el dominio del frontend) |
| `COLA_NOMBRE`, `COLA_INTENTOS`, `COLA_ESPERA_BASE`, `COLA_CANALES` | Cola de notificaciones (tienen valores por defecto) |

### Problemas que me han salido

| Problema | Qué reviso |
|---|---|
| `Could not resolve placeholder` al arrancar | Que `.env` exista en la carpeta desde donde ejecuto y tenga todas las variables de `.env.example` |
| Error de conexión a la base | Que el contenedor `base-datos` esté arriba (`docker compose ps`) |
| `no existe el rol kudos` | Que no haya otro Postgres instalado en el computador usando el puerto 5432 |
| Maven usa otra versión de Java | Que `JAVA_HOME` apunte a la 21 |
| Los mensajes de error salen como código (`AUTOR_NO_EXISTE`) | Que Strapi esté arriba (`docker compose ps`) y que `STRAPI_URL` apunte a él |
| El correo falla | Que `CORREO_HOST`, `CORREO_PUERTO` y la contraseña de aplicación sean los reales |
| WhatsApp falla | Que el teléfono se haya unido al sandbox de Twilio |
| Error 400 al subir un archivo | Que el tipo esté permitido, no pase el tamaño máximo y sean máximo 5 archivos |

---

## ABB y SBB

Primero definí qué se necesita (ABB) y después con qué lo resolví (SBB).

| ABB (qué se necesita) | SBB (con qué lo resolví) | Estado |
|---|---|---|
| ABB-01 Gestión de Kudos | Módulo `kudos` | Hecho |
| ABB-02 Gestión de adjuntos | `AlmacenArchivos`, carpeta de subidas | Hecho |
| ABB-03 Notificaciones | Módulo `notificaciones` con cola en Redis | Hecho |
| ABB-04 Canal de correo | `NotificadorCorreo` (JavaMailSender) | Hecho |
| ABB-05 Canal de WhatsApp | `NotificadorWhatsapp` (Twilio) | Hecho |
| ABB-06 Canal push | `NotificadorPush` y `ConexionesPush` (Server-Sent Events) | Hecho |
| ABB-07 Auditoría | Módulo `auditoria`, tabla `auditoria` | Hecho |
| ABB-08 Mensajería asíncrona | Lista de Redis como cola | Hecho |
| ABB-09 Caché | Redis, `CacheKudosRedis` | Hecho |
| ABB-10 Identidad y acceso | Cuentas aprobadas por el administrador, BCrypt, sesión en Redis y Spring Security | Hecho |
| ABB-11 Observabilidad | Endpoint `/salud`, auditoría | Parcial |
| ABB-13 Catálogos de mensajes y notificaciones | Módulo `catalogos` consumiendo Strapi (CMS) | Hecho |
| ABB-12 Entrega continua | GitHub Actions (CI y CD), SonarCloud, Trivy y GitHub Container Registry | Hecho |
| ABB-14 Baúl de secretos | Azure Key Vault, leído por Container Apps con referencias a secretos; en local, `.env` | Hecho |
| ABB-15 API Gateway | Azure API Management (plan de consumo), consumido como servicio | Pendiente |
| ABB-16 Firewall de aplicaciones web (WAF) | Azure Application Gateway con WAF en modo prevención y reglas OWASP | Pendiente |
| ABB-17 Despliegue de contenedores | Azure Container Apps (API y Strapi) y Static Web Apps (frontend) | Hecho |

---

## Tecnologías

| Pieza | Tecnología | Por qué |
|---|---|---|
| Backend | Java 21 con Spring Boot 3.3 y Maven | Es lo que mejor domino y ya trae inyección de dependencias, JPA, validación y pruebas |
| Frontend | React con Vite (PWA) | Liviano e instalable en el celular |
| Base de datos | PostgreSQL con Spring Data JPA | Datos relacionales |
| Caché y cola | Redis (Spring Data Redis) | Un solo servicio para las dos cosas |
| Tiempo real | Server-Sent Events (SSE) | Canal simple del servidor al navegador que se reconecta solo |
| Correo | SMTP con Spring Mail | Funciona con cualquier proveedor |
| WhatsApp | API de Twilio (con `RestClient`) | Tiene sandbox gratis |
| Catálogos | Strapi 5 (CMS) con su propia PostgreSQL | Los textos se editan en un panel, sin tocar código ni redesplegar el backend |
| Calidad | SonarCloud | Análisis estático |
| Seguridad | Trivy | Vulnerabilidades en dependencias e imagen |
| CI/CD | GitHub Actions | Está dentro del repositorio |
| Contenedores | Docker | Corre igual en mi computador y en la nube |
| Nube (API y catálogo) | Azure Container Apps | Corre contenedores sin pedir cuota de App Service, que en mi cuenta de estudiante estaba en 0 |
| Nube (frontend) | Azure Static Web Apps | Sirve la PWA por HTTPS |
| Datos en la nube | Azure Database for PostgreSQL y Azure Managed Redis | Son los mismos motores que uso en local, administrados |
| Secretos | Azure Key Vault | Las claves no viven en el código ni en las variables del repositorio |
| API Gateway y WAF | Azure API Management y Application Gateway con WAF | Se consumen como servicio, no se programan (pendientes de montar) |

---

## Diseño

### Arquitectura limpia

```
src/main/java/co/edu/uco/kudos/
  kudos/
    dominio/          Entidades y reglas (Kudo, Usuario, Adjunto)
    aplicacion/       Casos de uso (PublicarKudo, ListarKudos) y puertos
    infraestructura/  Postgres, Redis, la cola y los controladores
  notificaciones/     Canales de correo, WhatsApp y push
  catalogos/          Cliente de Strapi y catálogos de mensajes y plantillas de notificación
  auditoria/          Registro de cada paso
  compartido/         Configuración leída de variables de entorno
```

Las dependencias apuntan hacia adentro: la infraestructura conoce a la aplicación, la aplicación conoce al dominio, y el dominio no conoce a nadie.

Puertos principales: `KudoRepositorio`, `UsuarioRepositorio`, `AuditoriaRepositorio`, `PublicadorEventos`, `CacheKudos` y `Notificador`.

### Qué pasa al publicar un Kudo

1. El controlador recibe el mensaje y los archivos.
2. `PublicarKudo` verifica que existan el autor y el destinatario, y que el mensaje cumpla las reglas.
3. Guarda el Kudo y los adjuntos en Postgres.
4. Deja la auditoría `KUDO_CREADO`.
5. Borra la caché del destinatario.
6. Crea en Redis un trabajo por cada canal.
7. Responde al cliente.
8. `ProcesadorNotificaciones` toma cada trabajo y `EnviarNotificacion` lo manda por correo, WhatsApp o push, y deja la auditoría con OK, ERROR u OMITIDO.

### Modelo de datos

| Tabla | Campos |
|---|---|
| usuario | id, nombre, correo, telefono, rol (ESTUDIANTE, PROFESOR, ADMINISTRADOR), estado (PENDIENTE, ACTIVO, RECHAZADO), claveCifrada (BCrypt) |
| kudo | id, autorId, destinatarioId, mensaje, creadoEn |
| adjunto | id, nombre, tipo, tamano, url, kudo |
| auditoria | id, kudoId, evento, canal, estado, detalle, creadoEn |

### SOLID

| Principio | Dónde se ve |
|---|---|
| Responsabilidad única | Cada caso de uso hace una sola cosa |
| Abierto/cerrado | Un canal nuevo es una clase nueva, sin cambiar lo existente |
| Sustitución de Liskov | Correo, WhatsApp y push cumplen el mismo contrato `Notificador` |
| Segregación de interfaces | Puertos pequeños y separados |
| Inversión de dependencias | Los casos de uso piden puertos en el constructor y Spring inyecta la implementación |

### Patrones de diseño

| Patrón | Dónde |
|---|---|
| Repository | `KudoRepositorioPostgres`, `UsuarioRepositorioPostgres` |
| Adapter | Cada notificador adapta un servicio externo |
| Strategy | `EnviarNotificacion` elige el notificador según el canal |
| Publicador/Suscriptor | `PublicadorEventosCola` y `ProcesadorNotificaciones` |

---

### Catálogos

Los textos no están escritos dentro del código. Están en Strapi, que corre como un servicio aparte (carpeta `strapi/`, con su propia base PostgreSQL) y expone dos colecciones por HTTP. Al arrancar por primera vez se siembra desde `strapi/datos/*.json`, y el rol público solo tiene permiso de lectura: escribir sin token responde 403.

| Catálogo | Colección en Strapi | Qué guarda |
|---|---|---|
| Mensajes | `mensajes` | Código y texto de cada respuesta de error (por ejemplo `AUTOR_NO_EXISTE`) |
| Notificaciones | `notificaciones` | Canal, asunto y cuerpo, con las marcas `{nombre}` y `{mensaje}` |
| Parámetros | Variables de entorno | Límites y valores de configuración, en `ConfiguracionKudos` |

Los casos de uso lanzan una excepción con un **código** y `ManejadorDeErrores` lo convierte en texto pidiéndolo al catálogo. Los notificadores de correo y WhatsApp piden su plantilla por canal en vez de armar el texto a mano. Las dos consultas pasan por `StrapiCliente` y se guardan en una caché de `CACHE_SEGUNDOS` en el backend.

Las rutas, los campos y los nombres de los canales salen de variables de entorno (`STRAPI_RUTA_*`, `STRAPI_CAMPO_*`, `CANAL_*`), no del código. Si Strapi no responde, el mensaje de error sale como su código y el envío de la notificación queda auditado como ERROR y se reintenta desde la cola. Para cambiar un texto se edita la fila en el panel de Strapi, sin tocar Java ni volver a desplegar.

---

## Diagramas

Aquí van los diagramas de arquitectura. Cada uno tiene su espacio marcado con el nombre de la imagen que debe ir ahí. Primero están los de arquitectura general (contexto y contenedores), luego los de componentes, los de paquetes y por último los de secuencia.

### Diagrama 1: Contexto

Quién usa UCO Kudos y con qué sistemas de afuera se conecta. Las flechas punteadas son los avisos que le llegan a la persona.

> **Espacio para el diagrama 1** (imagen `evidencias/diagramas/1-contexto.png`)

<!-- Pegar aquí: ![Diagrama 1: Contexto](evidencias/diagramas/1-contexto.png) -->

<br>

<br>

<br>

### Diagrama 2: Contenedores

Las piezas que se despliegan en Azure y cómo se hablan. Los recuadros punteados de amarillo (WAF y API Gateway) todavía están pendientes; hoy la aplicación web habla directo con la API.

> **Espacio para el diagrama 2** (imagen `evidencias/diagramas/2-contenedores.png`)

<!-- Pegar aquí: ![Diagrama 2: Contenedores](evidencias/diagramas/2-contenedores.png) -->

<br>

<br>

<br>

### Diagrama 3: Componentes del backend: Kudos y notificaciones

Se lee de izquierda a derecha: el controlador llama a un caso de uso, el caso de uso habla con un puerto (una interfaz) y un adaptador implementa ese puerto contra el servicio real. El procesador toma los trabajos de la cola y ejecuta `EnviarNotificacion`.

> **Espacio para el diagrama 3** (imagen `evidencias/diagramas/3a-componentes-backend-kudos.png`)

<!-- Pegar aquí: ![Diagrama 3: Componentes del backend: Kudos y notificaciones](evidencias/diagramas/3a-componentes-backend-kudos.png) -->

<br>

<br>

<br>

### Diagrama 4: Componentes del backend: cuentas y sesión

Solicitud de cuenta, aprobación del administrador, ingreso y cambio de clave. Mismo patrón: casos de uso, puertos y adaptadores.

> **Espacio para el diagrama 4** (imagen `evidencias/diagramas/3b-componentes-backend-cuentas.png`)

<!-- Pegar aquí: ![Diagrama 4: Componentes del backend: cuentas y sesión](evidencias/diagramas/3b-componentes-backend-cuentas.png) -->

<br>

<br>

<br>

### Diagrama 5: Componentes del backend: catálogos

Los textos de error y las plantillas de notificación no están en el código. Se piden a Strapi y se guardan en una caché corta.

> **Espacio para el diagrama 5** (imagen `evidencias/diagramas/3c-componentes-backend-catalogos.png`)

<!-- Pegar aquí: ![Diagrama 5: Componentes del backend: catálogos](evidencias/diagramas/3c-componentes-backend-catalogos.png) -->

<br>

<br>

<br>

### Diagrama 6: Componentes del frontend

La aplicación web es una PWA en React. `App.jsx` decide qué pantalla mostrar y todos los componentes hablan con la API a través de `api.js`.

> **Espacio para el diagrama 6** (imagen `evidencias/diagramas/4-componentes-frontend.png`)

<!-- Pegar aquí: ![Diagrama 6: Componentes del frontend](evidencias/diagramas/4-componentes-frontend.png) -->

<br>

<br>

<br>

### Diagrama 7: Paquetes del backend: capas dentro de cada módulo

Esta es la regla de dependencia de la arquitectura limpia, y se cumple dentro de cada módulo: la infraestructura conoce a la aplicación, la aplicación conoce al dominio y el dominio no conoce a nadie.

> **Espacio para el diagrama 7** (imagen `evidencias/diagramas/5a-paquetes-capas.png`)

<!-- Pegar aquí: ![Diagrama 7: Paquetes del backend: capas dentro de cada módulo](evidencias/diagramas/5a-paquetes-capas.png) -->

<br>

<br>

<br>

### Diagrama 8: Paquetes del backend: módulos

Cómo se apuntan entre sí los módulos. Hay algunas dependencias en los dos sentidos (por ejemplo, `compartido` y `kudos`), porque `compartido` arma la configuración y el manejo de errores de todos. Lo dejo dibujado tal como está y lo anoto como mejora en los pendientes.

> **Espacio para el diagrama 8** (imagen `evidencias/diagramas/5b-paquetes-modulos-backend.png`)

<!-- Pegar aquí: ![Diagrama 8: Paquetes del backend: módulos](evidencias/diagramas/5b-paquetes-modulos-backend.png) -->

<br>

<br>

<br>

### Diagrama 9: Paquetes del frontend

Es una aplicación pequeña: un punto de entrada, la pantalla principal, los componentes y un solo archivo que habla con la API.

> **Espacio para el diagrama 9** (imagen `evidencias/diagramas/6-paquetes-frontend.png`)

<!-- Pegar aquí: ![Diagrama 9: Paquetes del frontend](evidencias/diagramas/6-paquetes-frontend.png) -->

<br>

<br>

<br>

### Diagrama 10: Secuencia: publicar un Kudo

El flujo principal del proyecto. Lo que está arriba del `loop` ocurre antes de responder al usuario; lo de abajo pasa por la cola, un trabajo por canal.

> **Espacio para el diagrama 10** (imagen `evidencias/diagramas/7-secuencia-publicar-kudo.png`)

<!-- Pegar aquí: ![Diagrama 10: Secuencia: publicar un Kudo](evidencias/diagramas/7-secuencia-publicar-kudo.png) -->

<br>

<br>

<br>

### Diagrama 11: Secuencia: solicitar y aprobar una cuenta

Nadie entra sin que el administrador lo apruebe. La clave temporal llega por correo y por WhatsApp.

> **Espacio para el diagrama 11** (imagen `evidencias/diagramas/8-secuencia-cuentas.png`)

<!-- Pegar aquí: ![Diagrama 11: Secuencia: solicitar y aprobar una cuenta](evidencias/diagramas/8-secuencia-cuentas.png) -->

<br>

<br>

<br>

---

## Lecturas, sistemas reactivos y Twelve-Factor

### Cómo respondo a muchas lecturas

1. Caché en Redis de 60 segundos para la lista de cada destinatario.
2. Esa caché se borra cada vez que llega un Kudo nuevo.
3. Paginación por cursor (la fecha del último Kudo) en vez de `OFFSET`.
4. Índice en la tabla de Kudos para esa consulta.
5. Los envíos pesados quedan fuera del camino de lectura.
6. La API no guarda estado, así que se pueden levantar varias copias.

Limitación: los archivos hoy los sirve la misma API. Con mucho tráfico lo correcto sería un almacenamiento de objetos o un CDN.

### Manifiesto de Sistemas Reactivos

| Pilar | Cómo lo apliqué |
|---|---|
| Responsivo | La API responde sin esperar los envíos |
| Resiliente | Cada canal tiene su trabajo, con 3 reintentos y espera creciente |
| Elástico | API sin estado; el procesador puede correr en otro proceso |
| Orientado a mensajes | Publicar y enviar se comunican por una cola |

### The Twelve-Factor App

| Factor | Cómo lo cumplo |
|---|---|
| I. Código base | Un repositorio por componente |
| II. Dependencias | Declaradas en el `pom.xml` |
| III. Configuración | Todo sale de variables de entorno (`.env.example`) |
| IV. Servicios de respaldo | Postgres, Redis, SMTP, Twilio y Strapi se conectan por configuración |
| V. Build, release, run | El pipeline compila, construye la imagen y después despliega |
| VI. Procesos | La API no guarda estado entre peticiones |
| VII. Vinculación de puertos | Escucha el puerto de la variable `PUERTO` |
| VIII. Concurrencia | Se escala levantando más procesos |
| IX. Desechabilidad | Cierre ordenado con `server.shutdown: graceful` |
| X. Paridad | El mismo Docker en local y en la nube |
| XI. Logs | Spring Boot escribe a la salida estándar |
| XII. Administración | Tareas con comandos de Maven |

---

## Observabilidad

- **Salud:** `GET /salud` responde si la API está viva.
- **Auditoría:** cada Kudo deja rastro en la tabla `auditoria` (evento, canal y estado OK, ERROR u OMITIDO).
- **Logs:** salen por la salida estándar. En local se ven con `docker logs` y en Azure, en el Flujo de registros de la aplicación de contenedores (portal de Azure, Supervisión).
- **Falta:** un endpoint de métricas y alertas.

---

## Pruebas

Tengo muchas pruebas rápidas abajo, algunas de integración en el medio y pocas de extremo a extremo arriba.

| Nivel | Qué prueba | Herramienta | Cantidad | Estado |
|---|---|---|---|---|
| Unitarias | Casos de uso, reglas del dominio, cuentas, sesión, notificadores, controladores, push y catálogos (con un Strapi falso en memoria) | JUnit 5 y Mockito | 102 | Pasan (`mvn test` y el CI) |
| Integración | La API completa con Postgres y Redis reales | Spring Boot Test y MockMvc | 11 | Corren en el CI con servicios de Postgres y Redis |
| Extremo a extremo | Flujo completo desde el frontend | Pendiente | 0 | Pendiente |

### Unitarias

Terminan en `Test` y no necesitan base de datos. Se corren con `mvn test`.

| Clase | Qué comprueba |
|---|---|
| `PublicarKudoTest` | Publicación correcta, mensaje vacío, destinatario inexistente y autor inexistente |
| `EnviarNotificacionTest` | Auditoría OK, OMITIDO y ERROR (con el error vuelto a lanzar para que reintente), canal inexistente y kudo inexistente |
| `ListarKudosTest` | Respuesta desde caché, consulta a la base cuando no hay caché, siguiente cursor, uso del cursor y cursor inválido |
| `ReglasMensajeTest` | Regla del mensaje: vacío, nulo, en el límite y por encima del límite |
| `NotificadorPushTest` | Push enviado si está conectado, omitido si no, y manejo de las conexiones |
| `PlantillaNotificacionTest` | Reemplazo del nombre y del mensaje en la plantilla |
| `StrapiClienteTest` | La consulta que se manda a Strapi, la primera fila de la respuesta, vacío si no hay filas y el token solo cuando está configurado |
| `CatalogoMensajesStrapiTest` | Texto del código, el código mismo si no existe o si Strapi falla, y que la caché evita volver a preguntar |
| `CatalogoNotificacionesStrapiTest` | Plantilla con asunto y cuerpo, asunto vacío si el canal no lo tiene y error si el canal no tiene plantilla |
| `ReglasCuentaTest` | Formato del correo (cualquier dominio), teléfono de 10 números y nombre |
| `SolicitarCuentaTest` | Solicitud pendiente con aviso al administrador, correo con formato inválido, rol administrador y correo repetido |
| `CambiarClaveTest` | Clave nueva guardada cifrada, clave actual incorrecta, clave nueva corta e igual a la actual |
| `ResolverSolicitudTest` | Aprobar (clave cifrada y aviso), rechazar, solicitud ya resuelta y solicitud inexistente |
| `IniciarSesionTest` | Ingreso correcto, sin cuenta, pendiente, rechazada y clave incorrecta |
| `ManejadorDeErroresTest` | Que cada error responda con el estado correcto y el texto sacado del catálogo |

### Integración

Terminan en `IT` y están en `FlujoKudoIT`. Levantan la aplicación completa contra Postgres y Redis reales; solo reemplazo los proveedores externos (correo, WhatsApp, los avisos de cuentas y el catálogo de mensajes). Se corren con `mvn verify`, con Postgres y Redis arriba y las variables de `.env` cargadas.

| Prueba | Qué comprueba |
|---|---|
| Publicar con adjunto | Con sesión, guarda el Kudo y el adjunto, y la auditoría tiene `KUDO_CREADO` y una fila por cada canal |
| Paginación por cursor | Primera página con 2 Kudos y siguiente cursor; segunda página con el último y sin cursor |
| Caché | Un Kudo nuevo aparece aunque la lista ya estuviera cacheada (se invalida) |
| Mensaje vacío | Responde 400 |
| Archivo no permitido | Un `.exe` se rechaza con 400 |
| Destinatario inexistente | Responde 404 |
| Sin sesión | Publicar sin token responde 401 con el texto del catálogo |
| Panel de solicitudes | Un estudiante recibe 403 y el administrador 200 |
| Aprobar solicitud | Pendiente no puede ingresar; el administrador aprueba y la cuenta queda activa |
| Cuenta inexistente | Ingresar con un correo sin cuenta responde 404 con su mensaje |
| Salud | `GET /salud` responde `ok` |

### Comandos

| Comando | Qué corre |
|---|---|
| `mvn test` | Solo las pruebas unitarias (no necesitan base de datos) |
| `mvn verify` | Unitarias y de integración, con cobertura (reporte en `target/site/jacoco/index.html`) |

### Falta

- Guardar las capturas del CI con las 113 pruebas (102 unitarias y 11 de integración) y la cobertura.
- Prueba de extremo a extremo con el frontend.
- Probar las notificaciones contra el sandbox real de Twilio y un SMTP real.

---

## Seguridad y DevSecOps

### Desarrollo seguro

| Medida | Dónde |
|---|---|
| Validación de datos de entrada (ids, correo, longitud del mensaje) | Bean Validation y reglas del dominio |
| Consultas parametrizadas, sin SQL armado a mano | Spring Data JPA con Hibernate |
| Límite de tamaño, de cantidad y lista de tipos permitidos en adjuntos | `AlmacenArchivos` |
| Archivos guardados con nombre aleatorio | `AlmacenArchivos` |
| Cabeceras de seguridad y sesión sin estado | Spring Security (`SeguridadConfig`) |
| Solo entran cuentas aprobadas por el administrador | `SolicitarCuenta`, `ResolverSolicitud` y `ReglasCuenta` |
| Claves guardadas con BCrypt y claves temporales aleatorias (`SecureRandom`) | `CifradorClavesBcrypt` y `GeneradorClavesSeguras` |
| Sesión con token aleatorio en Redis que vence solo; rutas por rol | `SesionesRedis`, `FiltroSesion` y `SeguridadConfig` |
| El autor del Kudo sale de la sesión, no del cuerpo de la petición | `KudosControlador` |
| Secretos fuera del código y del repositorio | Variables de entorno, `.env` en `.gitignore` y Azure Key Vault en producción |
| Todo por HTTPS en producción y conexiones con TLS a PostgreSQL y Redis | Static Web Apps, Container Apps, `BD_SSLMODE=require` y `REDIS_SSL=true` |
| Sin claves de Azure guardadas en GitHub | Credencial federada (OIDC) para el CD |
| Mensajes de error sin detalles internos | `ManejadorDeErrores` |
| Usuario sin privilegios dentro del contenedor | `Dockerfile` (`USER kudos`) |
| Imagen final solo con el JRE y el JAR | `Dockerfile` en dos etapas |

El CORS ya está restringido al dominio del frontend en producción (`CORS_ORIGENES`). Falta: límite de intentos de ingreso y de peticiones por usuario, y forzar el cambio de la clave temporal.

### Integración continua (CI)

Archivo `.github/workflows/ci.yml`. Se ejecuta en cada push y pull request a `main`.

| Paso | Herramienta | Falla si |
|---|---|---|
| Compilar | `mvn verify` | Hay errores de compilación |
| Pruebas y cobertura | JUnit y JaCoCo con servicios de Postgres y Redis | Falla alguna prueba |
| Análisis estático | SonarCloud con espera de la puerta de calidad | No pasa la puerta de calidad |
| Vulnerabilidades en dependencias | Trivy (modo sistema de archivos) | Hay vulnerabilidades críticas o altas con arreglo disponible |
| Imagen | Docker | La imagen no construye |
| Vulnerabilidades en la imagen | Trivy (modo imagen) | Hay vulnerabilidades críticas o altas con arreglo disponible |

Los reportes de cobertura y de Trivy se guardan como artefactos de cada ejecución.

Trivy corre desde la imagen oficial `aquasec/trivy:0.69.3` y no desde la acción `trivy-action`: en marzo de 2026 las etiquetas de esa acción fueron comprometidas y las versiones viejas se borraron, así que el pipeline fija una versión conocida y limpia del escáner.

### Entrega continua (CD)

Archivo `.github/workflows/cd.yml`. Se ejecuta solo cuando el CI termina bien en `main`, dentro del entorno `produccion`.

1. Construye la imagen Docker de la API y la publica en GitHub Container Registry (con el identificador del commit y `latest`).
2. Inicia sesión en Azure con la credencial federada (OIDC), sin ninguna clave guardada.
3. Le dice a Azure Container Apps que use la imagen nueva (`az containerapp update --image`).
4. Verifica `GET /salud` en la URL pública, con hasta 10 intentos.
5. En un segundo trabajo hace lo mismo con la imagen de `strapi/` (el despliegue en Azure solo corre si existe la variable `AZURE_STRAPI_NOMBRE`).

### Secretos y variables en GitHub

Se crean en Settings, Secrets and variables, Actions. Los identificadores de Azure no son secretos, por eso son variables.

| Nombre | Tipo | Qué es |
|---|---|---|
| `SONAR_TOKEN` | Secreto | Token generado en SonarCloud |
| `SONAR_PROJECT_KEY` | Variable | Clave del proyecto en SonarCloud |
| `SONAR_ORGANIZATION` | Variable | Organización en SonarCloud |
| `AZURE_CLIENT_ID` | Variable | Identificador de la aplicación registrada con la credencial federada |
| `AZURE_TENANT_ID` | Variable | Identificador del tenant de Azure |
| `AZURE_SUSCRIPCION_ID` | Variable | Identificador de la suscripción |
| `AZURE_GRUPO` | Variable | Grupo de recursos (`rg-kudos`) |
| `AZURE_API_NOMBRE` | Variable | Nombre de la aplicación de contenedores de la API (`kudos-api`) |
| `AZURE_STRAPI_NOMBRE` | Variable | Nombre de la aplicación de contenedores de Strapi (`kudos-catalogos`) |
| `URL_PUBLICA` | Variable | URL pública de la API, sin la barra final |

La credencial federada confía en el entorno `produccion` de este repositorio. El frontend tiene su propio pipeline (`ci-cd.yml` en su repositorio), con la variable `URL_API` y el secreto `AZURE_STATIC_WEB_APPS_API_TOKEN`.

---

## Despliegue en Azure

Todo está en un solo grupo de recursos llamado `rg-kudos`. La API y Strapi corren como contenedores en Azure Container Apps y el frontend en Static Web Apps; todo se atiende por HTTPS.

### Qué desplegué y dónde

| Recurso | Nombre | Servicio | Región |
|---|---|---|---|
| Frontend | `kudos-front-diego26` | Static Web Apps | West US 2 |
| API | `kudos-api` | Container Apps (1 CPU, 2 GiB, una réplica) | West US 2 |
| Catálogo | `kudos-catalogos` | Container Apps (misma imagen que `strapi/`) | West US 2 |
| Entorno de contenedores | `entorno-kudos-c` | Container Apps, solo consumo | West US 2 |
| Base de datos | `psql-kudos-diego26` (bases `kudos` y `strapi`) | Azure Database for PostgreSQL Flexible Server | North Central US |
| Caché, cola y sesiones | `redis-kudos-diego26` | Azure Managed Redis (en memoria, tamaño B0) | West US 2 |
| Secretos | `kv-kudos-diego26` | Azure Key Vault (permisos por roles) | Una de las cinco regiones permitidas |

Direcciones públicas:

| Pieza | Dirección |
|---|---|
| Frontend | https://zealous-pebble-01cbc5c1e.4.azurestaticapps.net |
| API | https://kudos-api.redsmoke-2305f8cc.westus2.azurecontainerapps.io |
| Salud de la API | https://kudos-api.redsmoke-2305f8cc.westus2.azurecontainerapps.io/salud |
| Catálogo (Strapi) | https://kudos-catalogos.redsmoke-2305f8cc.westus2.azurecontainerapps.io |

### Por qué estas regiones

La suscripción de Azure for Students trae una política que solo deja crear recursos en Chile Central, North Central US, South Central US, West US y West US 2. Dentro de esas cinco probé varias combinaciones: PostgreSQL estaba bloqueado en South Central US, así que quedó en North Central US; el resto quedó en West US 2.

### Secretos en Key Vault

Cada secreto se creó en `kv-kudos-diego26`. Las aplicaciones de contenedores los leen por referencia con su identidad del sistema, que tiene el rol "Key Vault Secrets User" sobre el baúl. Ningún valor aparece en el repositorio ni en las variables.

| Secreto en Key Vault | Lo usa | Variable |
|---|---|---|
| `bd-clave` | API y Strapi | `BD_CLAVE` y `DATABASE_PASSWORD` |
| `redis-clave` | API | `REDIS_CLAVE` |
| `correo-clave` | API | `CORREO_CLAVE` |
| `whatsapp-cuenta-sid` y `whatsapp-token` | API | `WHATSAPP_CUENTA_SID` y `WHATSAPP_TOKEN` |
| `admin-clave` | API | `ADMIN_CLAVE` |
| `strapi-app-keys`, `strapi-api-token-salt`, `strapi-admin-jwt-secret`, `strapi-transfer-token-salt`, `strapi-encryption-key` y `strapi-jwt-secret` | Strapi | `APP_KEYS`, `API_TOKEN_SALT`, `ADMIN_JWT_SECRET`, `TRANSFER_TOKEN_SALT`, `ENCRYPTION_KEY` y `JWT_SECRET` |

### Variables de la API

Las que no son secretas se cargan directo en la aplicación de contenedores:

| Variable | Valor |
|---|---|
| `PUERTO` | `3000` |
| `BD_HOST`, `BD_PUERTO`, `BD_USUARIO`, `BD_NOMBRE` | `psql-kudos-diego26.postgres.database.azure.com`, `5432`, `kudosadmin` y `kudos` |
| `BD_SSLMODE`, `BD_ESQUEMA` | `require` y `update` |
| `REDIS_HOST`, `REDIS_PUERTO`, `REDIS_SSL` | `redis-kudos-diego26.westus2.redis.azure.net`, `10000` y `true` |
| `CORREO_HOST`, `CORREO_PUERTO` | `smtp.gmail.com` y `587` |
| `CORREO_USUARIO`, `CORREO_REMITENTE`, `CORREO_DOMINIOS` | El correo de la cuenta de envío (los dos) y `uco.net.co,gmail.com,hotmail.com` |
| `WHATSAPP_NUMERO_ORIGEN` | El número del sandbox de Twilio |
| `ARCHIVOS_CARPETA`, `ARCHIVOS_TAMANO_MAXIMO`, `ARCHIVOS_TIPOS_PERMITIDOS` | `subidas`, `5242880` y `image/png,image/jpeg,application/pdf` |
| `CACHE_SEGUNDOS`, `TELEFONO_PREFIJO` | `60` y `+57` |
| `CORS_ORIGENES` | La dirección del frontend |
| `ADMIN_NOMBRE`, `ADMIN_CORREO` | El nombre y el correo de la cuenta del administrador |
| `STRAPI_URL` | La dirección de `kudos-catalogos` |

### Variables de Strapi

| Variable | Valor |
|---|---|
| `HOST`, `PORT`, `NODE_ENV` | `0.0.0.0`, `1337` y `production` |
| `PROXY` | `true` (Azure termina el HTTPS y Strapi tiene que confiar en la cabecera del proxy) |
| `DATABASE_CLIENT`, `DATABASE_HOST`, `DATABASE_PORT`, `DATABASE_NAME`, `DATABASE_USERNAME` | `postgres`, el mismo servidor de la API, `5432`, `strapi` y `kudosadmin` |
| `DATABASE_SSL` | `true` |

Strapi no necesita token para que la API lea los catálogos: al arrancar siembra los datos y deja la lectura pública abierta (escribir sin token responde 403).

### Cómo lo monté, paso a paso

1. Crear el grupo `rg-kudos`, el Key Vault (con permisos por roles), el PostgreSQL con las bases `kudos` y `strapi`, y el Redis, todos en las regiones permitidas.
2. Crear los secretos en el Key Vault, uno por uno.
3. Crear el entorno de contenedores en modo solo consumo (importante: los entornos Express no sirven):

```
az containerapp env create -n entorno-kudos-c -g rg-kudos -l westus2 --environment-mode ConsumptionOnly --logs-destination none
```

4. Crear la aplicación de la API con identidad del sistema y darle permiso para leer los secretos:

```
az containerapp create -n kudos-api -g rg-kudos --environment entorno-kudos-c --image ghcr.io/rambbit10/kudos:latest --target-port 3000 --ingress external --cpu 1 --memory 2Gi --min-replicas 1 --max-replicas 1 --system-assigned
az role assignment create --assignee ID_DE_LA_IDENTIDAD --role "Key Vault Secrets User" --scope ID_DEL_KEY_VAULT
```

5. Enlazar los secretos y cargar las variables:

```
az containerapp secret set -n kudos-api -g rg-kudos --secrets "bd-clave=keyvaultref:https://kv-kudos-diego26.vault.azure.net/secrets/bd-clave,identityref:system"
az containerapp update -n kudos-api -g rg-kudos --set-env-vars "BD_CLAVE=secretref:bd-clave" "PUERTO=3000"
```

(Se repite con cada secreto y cada variable de las tablas de arriba.)

6. Abrir el servidor PostgreSQL a la dirección de salida de la API, que se ve en `outboundIpAddresses` de la aplicación:

```
az postgres flexible-server firewall-rule create -g rg-kudos -s psql-kudos-diego26 --name api-kudos --start-ip-address IP_DE_SALIDA --end-ip-address IP_DE_SALIDA
```

7. Hacer lo mismo para Strapi (`kudos-catalogos`, puerto 1337) y apuntar `STRAPI_URL` de la API a su dirección.
8. Crear la aplicación del frontend en Static Web Apps, guardar su clave de despliegue como secreto del repositorio del frontend y la dirección de la API como variable `URL_API`.
9. Registrar la credencial federada para el CD (sección «Seguridad y DevSecOps») y crear las variables de Azure en GitHub.
10. Con un push a `main`, el CI y el CD publican la imagen y actualizan la aplicación. Se verifica con `GET /salud`.

### Problemas que me salieron

| Problema | Qué hice |
|---|---|
| La política de Azure bloqueaba East US 2, West US 3 y South Central US (PostgreSQL) | Leí la lista de regiones permitidas y repartí los recursos entre West US 2 y North Central US |
| El App Service B1 tenía cuota 0 y la solicitud de aumento no respondió a tiempo | Pasé a Azure Container Apps (ADR-11) |
| Azure Cache for Redis clásico ya no deja crear instancias nuevas | Usé Azure Managed Redis, tamaño en memoria B0 (cerca de 24 USD al mes), con acceso por clave y TLS |
| Los entornos de Container Apps que crea el portal por defecto (Express) no admiten Key Vault ni identidad del sistema | Los creé con la línea de comandos en modo solo consumo |
| La API no veía la base de datos | Agregué una regla de firewall con la dirección de salida de la API |
| El CD no podía iniciar sesión en Azure: el tenant rechazaba claves secretas | Usé credencial federada (ADR-13) |
| El login federado fallaba por el `subject` | Registré la credencial con los identificadores numéricos del repositorio |
| El workflow que crea Azure solo al armar el frontend no usaba mi variable de la API | Lo borré y dejé el pipeline propio del frontend |

### API Gateway y WAF (pendiente)

Siguen siendo parte del diseño (diagrama 2). Van como servicios administrados de Azure: Azure API Management en el plan de consumo y Azure Application Gateway con WAF en modo prevención y reglas OWASP. La prueba que falta es la del escenario COO-SEG-02: una petición con `' OR 1=1--` tiene que responder 403 y quedar en los registros del WAF. Cuando la haga, actualizo esta sección y el estado de ABB-15 y ABB-16.

### Costos

Es crédito de estudiante, así que apago o borro lo que cuesta cuando termine la evaluación. Lo que más gasta es Redis (cerca de 24 USD al mes) y, después, PostgreSQL y Container Apps con una réplica siempre encendida.

---

## Evidencias

Las imágenes están en la carpeta `evidencias/`.

| Evidencia | Qué muestra | Archivo |
|---|---|---|
| Pruebas | Salida de `mvn verify` con las pruebas en verde | `pruebas.png` |
| Flujo completo | Publicar un Kudo y ver la auditoría con los tres canales | `flujo-publicar-kudo.png` |
| CI | Ejecución del pipeline de integración en GitHub Actions, en verde | `ci.png` |
| CD | Ejecución del pipeline de despliegue en verde | `cd.png` |
| SonarCloud | Panel del proyecto con la puerta de calidad aprobada | `sonar.png` |
| Trivy | Reporte de dependencias e imagen sin vulnerabilidades críticas | `trivy.png` |
| Despliegue | La URL pública respondiendo en `/salud` | `despliegue.png` |
| Recursos en Azure | El grupo `rg-kudos` con todos los recursos | `azure-recursos.png` |
| Key Vault | Los secretos creados (solo los nombres) | `key-vault.png` |
| Container Apps | La API y Strapi en ejecución | `container-apps.png` |
| Frontend | La aplicación abierta por HTTPS en el celular | `frontend.png` |
| Strapi | `GET /api/mensajes` respondiendo en la nube | `strapi.png` |
| WAF | Una inyección bloqueada con 403 (pendiente) | `waf.png` |
| WhatsApp y correo | El mensaje recibido en el celular y el correo en la bandeja | `notificaciones.png` |
| Push | La notificación llegando en tiempo real | `push.png` |

---

## Uso de inteligencia artificial

Usé IA como ayuda, y me aseguré de entender y poder explicar todo lo que está en el proyecto.

| Tarea | Qué hizo la IA | Qué hice yo |
|---|---|---|
| Documentación | Ayudó a ordenar y redactar con la estructura que ya usamos en otros proyectos | Escogí el alcance y las decisiones, y revisé que el texto dijera solo lo que el proyecto realmente hace |
| Código del backend | Generó el esqueleto inicial con arquitectura limpia, en Java con Spring Boot | Escogí el stack, reviso el código, lo compilo, lo corro y lo pruebo en mi equipo y en el CI |
| Despliegue en Azure | Me fue dando los pasos y los comandos | Hice cada paso en mi cuenta, resolví los bloqueos de la política y de la cuota y decidí pasar a Container Apps |
| Análisis de calidad | Ayudó a armar las tablas de restricciones, escenarios y tradeoffs y a preparar los diagramas | Los puntajes de las prioridades y de los escenarios son propuestas mías que debo validar, y revisé que cada artefacto exista en el código |
| Decisiones de arquitectura | Propuso alternativas con sus ventajas | Elegí y justifiqué cada decisión (sección «Decisiones de arquitectura (ADR)») |

Lo que puedo explicar sin ayuda:

- El flujo de publicar un Kudo desde el controlador hasta la auditoría.
- Por qué cada capa depende solo de la de adentro.
- Cómo agregar un canal de notificación nuevo creando una clase que cumpla el contrato `Notificador`.
- Qué falta y por qué: está en la sección de pendientes.

---

## Pendientes

- Contenido del Kudo sin límite de longitud y réplicas (hoy el mensaje tiene un límite de 500 caracteres y no hay réplicas; RT-16)
- API Management y WAF delante de la API, con la prueba de inyección que responde 403
- Adjuntos en un almacenamiento que no se pierda al reiniciar el contenedor (Azure Files o Blob Storage)
- Límite de intentos de ingreso y de peticiones por usuario, y forzar el cambio de la clave temporal
- Endpoint de métricas y alertas
- Migraciones de base de datos en lugar de crear tablas automáticamente
- Pruebas de extremo a extremo automatizadas
- Cola con confirmación de entrega (broker) si crece la exigencia
- Push con varias réplicas de la API (repartir los eventos por Redis)
- Quitar las dependencias en los dos sentidos entre los módulos `compartido`, `kudos` y `cuentas`
- Rotar las claves que usé durante el desarrollo (contraseña de aplicación de Gmail, token de Twilio, token de SonarCloud) y la clave de despliegue del frontend
- Validar con otras personas los puntajes de prioridades y escenarios (secciones 3 y 4)

---

## Decisiones de arquitectura (ADR)

**ADR-01 Monolito modular.** Contexto: trabajo individual con pocos días y sin presupuesto. Decisión: una sola aplicación Spring Boot con paquetes separados por dominio. Alternativa: microservicios. Consecuencias: más fácil de desarrollar, desplegar y explicar; escala como una sola unidad, pero los módulos están separados para poder dividirlos después.

**ADR-02 Arquitectura limpia (hexagonal).** Contexto: debe poder cambiarse de proveedor de correo o WhatsApp, y los casos de uso deben poder probarse sin base de datos. Decisión: dominio, aplicación e infraestructura separados, con puertos entre ellos. Alternativa: capas tradicionales donde el servicio usa directamente el repositorio y el proveedor. Consecuencias: hay más archivos, pero las pruebas unitarias son simples y cambiar un proveedor es cambiar una clase.

**ADR-03 Java 21 con Spring Boot y Maven.** Contexto: necesito un framework con inyección de dependencias, persistencia, validación y pruebas, y debo poder explicarlo y modificarlo en vivo en la sustentación. Decisión: Java 21 con Spring Boot 3.3 y Maven; para el frontend, una PWA. Alternativas: NestJS con TypeScript (llegué a construir una primera versión) y Express sin estructura. Consecuencias: es la tecnología que mejor domino, así que puedo explicar cada parte y hacer cambios en vivo con más seguridad. Costó reescribir el backend, pero la arquitectura y los puertos se mantuvieron. Se requiere JDK 21.

**ADR-04 Redis para caché y cola.** Contexto: necesito caché para las lecturas y una cola para las notificaciones. Decisión: un solo Redis, con una lista como cola y un procesador propio que la lee. Alternativas: Kafka o RabbitMQ. Consecuencias: menos piezas que montar y explicar. Limitación: una lista de Redis no tiene confirmación de entrega, así que si la aplicación se cae justo después de sacar un trabajo, ese trabajo se pierde (la auditoría deja ver qué canal no quedó registrado). Si el volumen o la exigencia crecen, se puede reemplazar por un broker como RabbitMQ gracias al puerto `PublicadorEventos`.

**ADR-05 Notificaciones asíncronas, un trabajo por canal.** Contexto: enviar correo y WhatsApp es lento y puede fallar. Decisión: al publicar se crea un trabajo en la cola por cada canal, con 3 reintentos y espera creciente. Alternativa: enviar todo dentro de la misma petición. Consecuencias: la API responde rápido y un canal caído no afecta a los demás; la entrega es eventual y por eso la auditoría es importante.

**ADR-06 PWA para el frontend.** Contexto: el 90% de los usuarios entra desde el celular. Decisión: una aplicación web instalable con React. Alternativa: app nativa. Consecuencias: un solo código para todos los dispositivos y sin tiendas de apps; algunas funciones nativas quedan limitadas.

**ADR-07 Server-Sent Events para el push.** Contexto: el destinatario debe enterarse al instante si está conectado. Decisión: el navegador abre `GET /push?token=...` y el servidor le envía eventos; la aplicación guarda en memoria qué usuarios están conectados. Alternativas: WebSocket o Socket.IO, o consultar al servidor cada cierto tiempo. Consecuencias: llega al instante, el navegador lo soporta con `EventSource` y reconecta solo. Solo va del servidor al cliente, que es todo lo que necesita una notificación. Limitación: el mapa de conectados vive en memoria; con varias copias de la API habría que repartir los eventos por Redis.

**ADR-08 Paginación por cursor.** Contexto: habrá muchas lecturas y listas largas. Decisión: el cursor es la fecha del último Kudo recibido. Alternativa: `OFFSET` y `LIMIT`. Consecuencias: mantiene el rendimiento con muchos registros; no permite saltar a una página arbitraria, algo que en un feed no hace falta.

**ADR-09 Adjuntos en disco con metadatos en la base.** Contexto: los Kudos pueden llevar archivos, pero no quiero guardar binarios en la base. Decisión: los archivos van en una carpeta con nombre aleatorio y en la base solo nombre, tipo, tamaño y ruta. Alternativa: almacenamiento de objetos. Consecuencias: simple para esta entrega. En Azure Container Apps el disco del contenedor es temporal: si el contenedor se reinicia o se despliega una imagen nueva, los adjuntos guardados se pierden y quedan solo los datos en la base. Lo correcto a futuro es montar Azure Files o usar Blob Storage.

**ADR-10 SonarCloud y Trivy en el pipeline.** Contexto: el enunciado exige análisis estático y de vulnerabilidades. Decisión: SonarCloud para el código y Trivy para dependencias e imagen Docker, ambos en el CI. Consecuencias: son las herramientas que pide el enunciado y tienen capa gratuita. Si fallan, el pipeline se detiene.

**ADR-11 Azure Container Apps en vez de App Service.** Contexto: mi cuenta de Azure for Students solo permite cinco regiones y el plan B1 de App Service tenía cuota 0 en las que probé (pedí un aumento de cuota y no tuve respuesta a tiempo). Decisión: correr la API y Strapi como aplicaciones de Azure Container Apps, con la misma imagen de Docker que ya publicaba el pipeline. Alternativas: App Service, una máquina virtual o Azure Container Instances. Consecuencias: no necesita cuota especial y cobra solo por lo que usa. Dos cuidados que aprendí: el entorno tiene que crearse en modo "solo consumo" (los entornos Express que crea el portal por defecto no admiten referencias a Key Vault ni identidad del sistema) y el disco del contenedor es temporal (ver ADR-09).

**ADR-12 Secretos en Azure Key Vault con referencias.** Contexto: las claves no pueden estar en el código ni en variables que se vean en texto plano. Decisión: guardar cada secreto en Key Vault y que el contenedor lo lea por referencia, usando su identidad del sistema con el rol "Key Vault Secrets User". Alternativas: variables de entorno directas en el contenedor o secretos de GitHub copiados a Azure. Consecuencias: rotar una clave es cambiar el secreto en un solo lugar; cuesta unos pasos más de configuración la primera vez.

**ADR-13 Despliegue con credencial federada (OIDC).** Contexto: el CD necesita permiso para actualizar la imagen en Azure. Mi primer intento fue un service principal con clave secreta, pero el tenant lo rechazó con "Credential type not allowed as per assigned policy". Decisión: una aplicación registrada con credencial federada que confía en el entorno `produccion` de este repositorio, y `azure/login` con `client-id`, `tenant-id` y `subscription-id` (que son variables, no secretos). Alternativa: guardar una clave en GitHub. Consecuencias: GitHub no guarda ninguna clave de Azure. Un detalle: el token de GitHub trae el repositorio con sus identificadores numéricos (`RAMBBIT10@180295870/KUDOS@1454847854`), así que registré la credencial con ese `subject` exacto además del nombre normal.
