# Guía de diseño de APIs HTTP

## Introducción

Esta guía describe una serie de buenas prácticas para el diseño de APIs HTTP+JSON originalmente extraído del trabajo en el API de la [plataforma Heroku](https://devcenter.heroku.com/articles/platform-api-reference).

Esta guía incluye añadidos a ese API y también sirve de guía para nuevas APIs internas en Heroku. Esteramos que sea de interés a los diseñadores de APIs que no son de Heroku.

Los objetivos que nos hemos marcado son consistencia y foco en la lógica de negocio, a la vez que evitamos centrarnos en detalles supérfluos (_N.T. el término en ingles es "design bikeshedding" explicado en [este artículo](http://es.wikipedia.org/wiki/Ley_de_Parkinson_de_la_trivialidad))._
Buscamos un _método bueno, consistente y bien documentado_ para el diseño de APIs, aunque no necesariamente _el método único o ideal_.

Asumimos que los conceptos básicos de APIs HTTP+JSON son familiares para tí, así que no entraremos en sus fundamentos en esta guía.

Agradecemos también [contribuciones](CONTRIBUTING.md) a la versión inglesa de esta guía o a su traducción al castellano.

## Contenidos

* [Fundamentos](#fundamentos)
  *  [Separar responsabilidades](#separar-responsabilidades)
  *  [Conexiones seguras requeridas](#conexiones-seguras-requeridas)
  *  [Versionado con la cabecera Accepts requerido](#versionado-con-la-cabecera-accepts-requerido)
  *  [Soportar ETags para el cacheado](#soportar-etags-para-el-cacheado)
  *  [Proporcionar identificadores de petición para introspección](#Proporcionar-identificadores-de-petición-para-introspeccion)
  *  [Dividir respuestas largas en varias peticiones usando rangos](#dividir-respuestas-largas-en-varias-peticiones-usando-rangos)
* [Peticiones](#peticiones)
  *  [Retornar códigos de estado apropiados](#retornar-codigos-de-estado-apropiados)
  *  [Proporcionar recursos completos si están disponibles](#proporcionar-recursos-completos-si-estan-disponibles)
  *  [Aceptar JSON serializado en el cuerpo de las peticiones](#aceptar-json-serializado-en-el-cuerpo-de-las-peticiones)
  *  [Usar formatos de ruta consistentes](#usar-formatos-de-ruta-consistentes)
    *  [Utilizar minúsculas para rutas y atributos](#utilizar-minusculas-para-rutas-y-atributos)
    *  [Soportar referencias por atributos no-identificadores como ayuda](#soportar-referencias-por-atributos-no-identificadores-como-ayuda)
    *  [Minimizar anidado de rutas](#minimizar-anidad-de-rutas)
* [Respuestas](#respuestas)
  *  [Proporcionar identificadores únicos (UUIDs) de recursos](#proporcionar-identificadores-unicos-uuids-de-recursos)
  *  [Proporcionar fechas y horas (timestamps) estándar](#proporcionar-fechas-y-horas-timestamps-estandar)
  *  [Usar horas en UTC con formato ISO8601](#usar-horas-en-utc-con-formato-ISO8601)
  *  [Anidar relaciones de clave foránea](#anidar-relaciones-de-clave-foranea)
  *  [Generar errores estructurados](#generar-errores-estructurados)
  *  [Mostrar el estado del límite de peticiones](#mostrar-el-estado-del-limite-de-peticiones)
  *  [Mantener los JSON minificados en todas las respuestas](#mantener-los-json-minificados-en-todas-las-respuestas)
* [Artefactos](#artifactos)
  *  [Proporcionar un esquema de JSON procesable](#proporcionar-un-esquema-de-json-procesable)
  *  [Proporcionar docmentación para desarrolladores](#proporcionar-docmentacion-para-desarrolladores)
  *  [Proporcionar ejemplos ejecutables](#proporcionar-ejemplos-ejecutables)
  *  [Describir la estabilidad](#describir-la-estabilidad)
* [Traducciones](#tranducciones)

### Fundamentos

#### Separa responsabilidades

Mientras diseñas, simplifica al máximo separando las responsabilidades de las diferentes partes del ciclo de petición y respuesta. Mantener reglas sencillas en esto te permite enfocarte en problemas mayores y más complejos.

Las peticiones y respuestas se realizarán para obtener un recurso o colección en concreto. Usa la ruta (_N.T. el path_) para especificar la entidad, el cuerpo para transferir los contenidos y las cabeceras para especificar metadatos. Los parámetros (_N.T. query params_) pueden ser usados como un medio de pasar información de cabecera en casos extremos, pero es preferible usar las cabeceras ya que son más flexibles y pueden transmitir información más variada.

#### Conexiones seguras requeridas

Haz requerido el acceso seguro al API usando TLS, sin excepciones.
No merece la pena intentar averiguar o explicar cuando se debe permitir acceso con TLS y cuando no. Haz obligatorio TLS para todo.

Idealmente, rechaza cualquier petición sin TLS sin responder a peticiones HTTP o al puerto 80, y así evitar cualquier intercambio de información insegura. En entornos en los que esto no sea posible, retorna con `403 Forbidden`.

Las redirecciones están desaconsejadas ya que permiten a los clientes tener comportamientos incorrectos/chapuceros sin ofrecer ninguna ventaja clara. Los clientes que dependen de las redirecciones duplican el tráfico del servidor y hacen que el TLS sea inutil ya que la información sensible ya ha sido expuesta durante la primera llamada.

#### Versionado con la cabecera Accepts requerido

El versionado y la transición entre versiones puede ser uno de los aspectos más difíciles en el diseño y mantenimiento de un API. Por eso, es mejor empezar con mecanismos para facilitarlo desde el principio.

Para evitar sorpresas, como por ejemplo cambios incompatibles para los usuarios, es mejor que la versión sea requerida en todas las peticiones. Las versiones por defecto deben ser evitadas ya que, en el mejor de los casos, son muy difíciles de cambiar en el futuro.

Es mejor especificar la versión en las cabeceras, junto a otros metadatos, usando la cabecera `Accept` junto con un _content type_ personalizado, por ejemplo:

```
Accept: application/vnd.heroku+json; version=3
```

#### Soportar ETags para el cacheado

Incluye una cabecera `ETag` en todas las respuestas, identificando la versión específica del recurso retornado. Esto permite a los usuarios cachear recursos y usar peticiones con este valor en la cabecera `If-None-Match` para especificar si la caché debe ser actualizada.

#### Proporcionar identificadores de petición para introspección

Incluye una cabecera `Request-Id` en cada respuesta del API, junto con un valor UUID. _Logueando_ estos valores en el cliente, en el servidor y cualquier otro servicio de apoyo, se ofrece un mecanismo para _tracear_, diagnosticar y depurar las peticiones.

#### Dividir respuestas largas en varias peticiones usando rangos

Las respuestas largas deben ser troceadas en múltiples peticiones usando la cabecera `Range` para especificar cuando están disponibles más datos y cómo recuperarlos. Consulta en [el debate sobre rangos (en inglés)](https://devcenter.heroku.com/articles/platform-api-reference#ranges) los detalles sobre las cabeceras de las peticiones y respuestas, los códigos de estado, límites, ordenación e iteración.

### Peticiones

#### Retornar códigos de estado apropiados

Retorna códigos de estado HTTP apropiados para cada respuesta. Las respuetas con éxito deben retornar códigos según la siguiente guía:

* `200`: Petición exitosa para una llamada `GET`, `DELETE` o
  `PATCH` que se completó de forma síncrona, o para una llamada `PUT` que actualizó un recurso existente de forma síncrona.
* `201`: Petición exitosa para una llamada `POST` que se completó de forma síncrona, o para una llamada `PUT` que creó un nuevo recurso de forma síncrona.
* `202`: Petición aceptada para una llamada `POST`, `PUT`, `DELETE`, o `PATCH` que será procesada de forma asíncrona.
* `206`: Petición exitosa para una llamada `GET`, pero que sólo retorna una respuesta parcial: ver [sección sobre respuestas largas](#dividir-respuestas-largas-en-varias-peticiones-usando-rangos).

Presta atención al uso de código de error de autenticación y autorización:

* `401 Unauthorized`: Petición fallida porque el usuario no está autenticado.
* `403 Forbidden`: Petición fallida porque el usuario no tiene autorización para acceder al un recurso en concreto.

Retorna códigos adecuados para ofrecer información adicional cuando ocurran errores:

* `422 Unprocessable Entity`: Tu petición es correcta, pero contiene parámetros inválidos.
* `429 Too Many Requests`: Has superado el límite de consumo. Inténtalo de nuevo más tarde.
* `500 Internal Server Error`: Algo falló en el servidor. Comprueba el estado del sitio y/o reporta la incidencia.

Consulta la [especificación de códigos de respuesta HTTP](https://tools.ietf.org/html/rfc7231#section-6) para una guía sobre los códigos de estado para errores de usuario o de servidor.

#### Proporcionar recursos completos si están disponibles

Incluye en la respuesta la representación completa del recurso (p.e. el objeto con todos sus atributos) siempre que sea posible. Siempre incluye el recurso completo en respuestas `200` y `201`, incluidas peticiones tipo `PUT`/`PATCH` y `DELETE`, p.e.:

```bash
$ curl -X DELETE \  
  https://service.com/apps/1f9b/domains/0fd4

HTTP/1.1 200 OK
Content-Type: application/json;charset=utf-8
...
{
  "created_at": "2012-01-01T12:00:00Z",
  "hostname": "subdomain.example.com",
  "id": "01234567-89ab-cdef-0123-456789abcdef",
  "updated_at": "2012-01-01T12:00:00Z"
}
```

Las respuetas `202` no deben incluir la representación completa del recurso, p.e.:

```bash
$ curl -X DELETE \  
  https://service.com/apps/1f9b/dynos/05bd

HTTP/1.1 202 Accepted
Content-Type: application/json;charset=utf-8
...
{}
```

#### Aceptar JSON serializado en el cuerpo de las peticiones

Soporta JSON serializados en los cuerpos de las peticiones `PUT`/`PATCH`/`POST`, tanto en lugar de, como junto a datos de formularios HTML. Esto es equivalente a los cuerpos de las respuestas con JSON serializado, p.e.:

```bash
$ curl -X POST https://service.com/apps \
    -H "Content-Type: application/json" \
    -d '{"name": "demoapp"}'

{
  "id": "01234567-89ab-cdef-0123-456789abcdef",
  "name": "demoapp",
  "owner": {
    "email": "username@example.com",
    "id": "01234567-89ab-cdef-0123-456789abcdef"
  },
  ...
}
```

#### Usar formatos de ruta consistentes

##### Nombre de recurso

Usa nombres de recurso en plural excepto si el recurso que estás nombrando es único (_N.T. singleton_) en el sistema (por ejemplo, en la mayoría de los sistemas un usuario dado sólo puede tener una cuenta). Esto permite que la forma de acceder a un recurso en particular sea consistente.

##### Acciones

Son preferibles esquemas de llamada (_N.T. endpoint layouts_) que no necesiten ninguna acción especial para recursos individuales. En aquellos casos donde se necesitan acciones especiales, ponlas tras el prefijo estándar `actions`, para diferenciarlas claramente:

```
/resources/:resource/actions/:action
```

p.e.

```
/runs/{run_id}/actions/stop
```

#### Utilizar minúsculas para rutas y atributos

Usa nombres de rutas en minúsculas y separados por guión (-), para que sea igual que los nombres de dominio, p.e.:

```
service-api.com/users
service-api.com/app-setups
```

Para los atributos, utiliza también minúsculas, pero usa guión bajo (_) para que los nombres de atributos puedan ser tecleados sin comillas en JavaScript, p.e.:

```
service_class: "first"
```

#### Soportar referencias por atributos no-identificadores como ayuda

En algunos casos, puede ser conveniente para los usuarios finales ofrecer identificadores para acceder a recursos. Por ejemplo, un usuario puede pensar en los nombres de las apps en Heroku, pero una app puede estar identificada por un UUID. En estos casos, podrías aceptar tanto el identificador como el nombre, p.e.:

```bash
$ curl https://service.com/apps/{app_id_or_name}
$ curl https://service.com/apps/97addcf0-c182
$ curl https://service.com/apps/www-prod
```

No aceptes sólo nombres omitiendo los identificadores.

#### Minimizar anidado de rutas

En modelos de datos con relaciones entre recursos padre/hijo, las rutas pueden acabar estando muy anidadas, p.e.:

```
/orgs/{org_id}/apps/{app_id}/dynos/{dyno_id}
```

Limita el anidamiento excesivo poniendo recursos en el raíz de la ruta. Usa anidado para indicar colecciones. Por ejemplo, para el caso anterior en que un _dyno_ pertenece a una app, que pertenece a una organización:

```
/orgs/{org_id}
/orgs/{org_id}/apps
/apps/{app_id}
/apps/{app_id}/dynos
/dynos/{dyno_id}
```

### Respuestas

#### Proporcionar identificadores únicos (UUIDs) de recursos

Dale a cada recurso un atributo `id` por defecto. Usa Identificadores Únicos Universaeles (UUIDs) excepto si tienes una buena razón para no hacerlo. No uses identificadores que no son únicos entre instancias del servicio o con otros recursos del servicio, especialmente identificadores auto-incrementales.

Presenta los UUIDs en minusculas, en formato `8-4-4-4-12`, p.e.:

```
"id": "01234567-89ab-cdef-0123-456789abcdef"
```

#### Proporcionar fechas y horas (timestamps) estándar

Por defecto, incluye atributos de fecha-hora `created_at` y `updated_at` en tus recursos, p.e.:

```javascript
{
  // ...
  "created_at": "2012-01-01T12:00:00Z",
  "updated_at": "2012-01-01T13:00:00Z",
  // ...
}
```

Estas fechas/horas podrían no tener sentido en algunos recursos, en cuyo caso pueden ser omitidos.

#### Usar horas en UTC con formato ISO8601

Acepta y retorna fechas y horas sólo en formato UTC. Presenta las fechas utilizando el formato ISO8601, p.e.:

```
"finished_at": "2012-01-01T12:00:00Z"
```

#### Anidar relaciones de clave foránea

Serializa las relaciones de clave foránea usando objetos anidados, p.e.:

```javascript
{
  "name": "service-production",
  "owner": {
    "id": "5d8201b0..."
  },
  // ...
}
```

En lugar de, p.e.:

```javascript
{
  "name": "service-production",
  "owner_id": "5d8201b0...",
  // ...
}
```

Este enfoque hace posible incluir más información del recurso relacionado sin tener que cambiar la estructura de la respuesta o introducir más atributos de primer nivel, p.e.:

```javascript
{
  "name": "service-production",
  "owner": {
    "id": "5d8201b0...",
    "name": "Alice",
    "email": "alice@heroku.com"
  },
  // ...
}
```

#### Generar errores estructurados

En caso de error, genera cuerpos de respuesta consistentes y estructurados. Incluye identificadores de error para su procesado, mensajes error para usuarios y, opcionalmente, una `url` para que el cliente pueda obtener más información sobre el error y sobre cómo resolverlo, p.e.:

```
HTTP/1.1 429 Too Many Requests

{
  "id":      "rate_limit",
  "message": "Account reached its API rate limit.",
  "url":     "https://docs.service.com/rate-limits"
}
```

Documenta tu formato de error y los identificadores de error que los clientes pueden recibir.

#### Mostrar el estado del límite de peticiones

Limita el consumo de las peticiones de los clientes para asegurar la salud de tu servicio y para mantener un nivel de calidad alto para otros clientes.  Puedes usar un algoritmo como [token bucket](http://en.wikipedia.org/wiki/Token_bucket) para calcular los límites de las peticiones.

Retorna el número de peticiones restantes en la cabecera de respuesta
`RateLimit-Remaining`.

#### Mantener los JSON minificados en todas las respuestas

Los espacios extra incrementan innecesariamente el tamaño de las respuestas y muchos clientes para desarrolladores "maquillarán" (_N.T. prettify_) las respuestas JSON automaticamente. Así que es mejor que las respuestas JSON estén minificadas, p.e.:

```json
{"beta":false,"email":"alice@heroku.com","id":"01234567-89ab-cdef-0123-456789abcdef","last_login":"2012-01-01T12:00:00Z","created_at":"2012-01-01T12:00:00Z","updated_at":"2012-01-01T12:00:00Z"}
```

Instead of e.g.:

```json
{
  "beta": false,
  "email": "alice@heroku.com",
  "id": "01234567-89ab-cdef-0123-456789abcdef",
  "last_login": "2012-01-01T12:00:00Z",
  "created_at": "2012-01-01T12:00:00Z",
  "updated_at": "2012-01-01T12:00:00Z"
}
```

Podrías ofrecer un método opcional para que los clientes recuperen respuestas más completas, ya sea a través de un parámetro (p.e. `?pretty=true`) o en la cabecera `Accept` (p.e.
`Accept: application/vnd.heroku+json; version=3; indent=4;`).

### Artefactos

#### Proporcionar un esquema de JSON procesable

Proporciona un esquema procesable (_machine-readable_) para especificar con exactitud el uso de tu API. Usa [prmd](https://github.com/interagent/prmd) para gestionar tu esquema, y asegúrate que es válido usando `prmd verify`.

#### Proporcionar docmentación para desarrolladores

Proporciona una documentación para los desarrolladores puedan usar para entender tu API.

Si creas un esquema con `prmd` como hemos dicho antes, puedes generar fácilmente documentos Markdown para todos tus servicios (_endpoints_) usando `prmd doc`.

Además de los detalles de los servicios, ofrece un resumen de tu API con información sobre:

* Autenticacaión, incluyendo obtención y uso de tokens de autenticación.
* Estabilidad y versionado del API, incluyendo cómo usar una versión específica del API.
* Cabeceras de petición y respuesta genéricas.
* Formato de respuestas de error.
* Ejemplos de uso del API con clientes en distintos lenguajes.

#### Proporcionar ejemplos ejecutables

Ofrece ejemplos ejecutables que los usuarios puedan probar directamente en sus terminales para ver las llamadas al API en funcionamiento.

En la medida de lo posible, estos ejemplos deben ser utilizables tal cual, para minimizar la cantidad de trabajo necesario para probar el API, p.e.:

```bash
$ export TOKEN=... # acquire from dashboard
$ curl -is https://$TOKEN@service.com/users
```

Si usas [prmd](https://github.com/interagent/prmd) para generar documentación en Markdown, tendrás ejemplos para cada servicio sin esfuerzo.

#### Describir la estabilidad

Describe la estabilidad de tu API o sus servicios en función de su madurez y estabilidad, por ejemplo con etiquetas prototipo/desarrollo/producción.

Ver la [política de compatibilidad del API de Heroku](https://devcenter.heroku.com/articles/api-compatibility-policy)
para ver un posible enfoque de estabilidad y gestión del cambio.

Una vez que tu API está declarada como lista para producción y estable, no hagas cambios incompatibles en una misma versión del API. Si necesitas hacer cambios incompatibles, crea una nueva API con un número de versión superior.


### Traducciones

 * [Versión original (inglés)](https://github.com/interagent/http-api-design/commit/2a74f45b9afaf6c951352f36c3a4e1b0418ed10b)
 * [Versión en coreano](https://github.com/yoondo/http-api-design) (a partir de [f38dba6](https://github.com/interagent/http-api-design/commit/f38dba6fd8e2b229ab3f09cd84a8828987188863)), realizada por [@yoondo](https://github.com/yoondo/)
 * [Versión en chino simplificado](https://github.com/ZhangBohan/http-api-design-ZH_CN) (a partir de [337c4a0](https://github.com/interagent/http-api-design/commit/337c4a05ad08f25c5e232a72638f063925f3228a)), realizada por [@ZhangBohan](https://github.com/ZhangBohan/)
 * [Versión en chino traducional](https://github.com/kcyeu/http-api-design) (a partir de [232f8dc](https://github.com/interagent/http-api-design/commit/232f8dc6a941d0b25136bf64998242dae5575f66)), realizada por [@kcyeu](https://github.com/kcyeu/)

