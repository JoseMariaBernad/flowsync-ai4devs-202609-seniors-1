## Purpose

Permitir que una persona cree su cuenta en FlowSync, entre y salga con ella, y consulte sus propios datos, de modo que el resto del sistema sepa siempre quién hace cada petición.

## Requirements

### Requirement: Formato de las respuestas de la API

El sistema SHALL responder siempre en JSON. Las respuestas correctas de registro, inicio de sesión y perfil SHALL ir envueltas en un objeto `data`; las respuestas de error SHALL llevar una lista `errors` en la que cada elemento tiene al menos un `message`.

#### Scenario: Respuesta correcta envuelta

- **WHEN** una petición de registro, inicio de sesión o perfil se resuelve correctamente
- **THEN** el cuerpo de la respuesta es `{ "data": ... }` con el resultado dentro

#### Scenario: Error de validación desglosado por campo

- **WHEN** una petición llega con campos que no pasan la validación
- **THEN** la respuesta es un 422 con `{ "errors": [...] }`, con un elemento por cada fallo, y cada uno indica `field`, `rule`, `message` y, cuando aplica, `meta` (por ejemplo `{ "min": 8 }`)

### Requirement: Datos públicos de un usuario

El sistema SHALL representar a un usuario, en todas las respuestas que lo incluyen, con exactamente estos campos: `id`, `fullName` (texto o `null`), `email`, `initials`, `createdAt` y `updatedAt` (fechas ISO 8601). El sistema SHALL NOT devolver nunca la contraseña ni ningún derivado de ella.

#### Scenario: Usuario con nombre y apellido

- **WHEN** se devuelve un usuario cuyo `fullName` es `"Ada Lovelace"`
- **THEN** `initials` vale `"AL"`: la inicial de las dos primeras palabras del nombre, en mayúsculas

#### Scenario: Usuario con nombre de una sola palabra

- **WHEN** se devuelve un usuario cuyo `fullName` es `"Dan"`
- **THEN** `initials` vale `"DA"`: las dos primeras letras de esa palabra, en mayúsculas

#### Scenario: Usuario sin nombre

- **WHEN** se devuelve un usuario sin `fullName` y con email `carol.smith@x.com`
- **THEN** `fullName` es `null` e `initials` vale `"CX"`: la inicial de lo que va antes de la arroba más la inicial de lo que va después

### Requirement: Registro de una cuenta nueva

El sistema SHALL aceptar `POST /api/v1/auth/signup` con `fullName`, `email`, `password` y `passwordConfirmation`, crear la cuenta y devolver en la misma respuesta el usuario creado y un token de acceso, de modo que quien se registra queda ya identificado sin tener que iniciar sesión después.

#### Scenario: Registro correcto

- **WHEN** se envía un `fullName`, un email con formato válido no registrado todavía, una contraseña de 8 a 32 caracteres y la misma contraseña en `passwordConfirmation`
- **THEN** la respuesta es un 200 con `data.user` (los datos públicos del usuario nuevo) y `data.token` (un token opaco que empieza por `oat_`)

#### Scenario: Nombre opcional pero clave obligatoria

- **WHEN** se envía `fullName: null` o `fullName: ""` junto con el resto de campos válidos
- **THEN** la cuenta se crea y el usuario devuelto tiene `fullName: null`

#### Scenario: Falta la clave del nombre

- **WHEN** se envía el cuerpo sin la clave `fullName`
- **THEN** la respuesta es un 422 con un error `required` en el campo `fullName`, y no se crea la cuenta

#### Scenario: Email ya registrado

- **WHEN** se intenta registrar un email que ya existe escrito exactamente igual
- **THEN** la respuesta es un 422 con un error `database.unique` en el campo `email`, y no se crea la cuenta

#### Scenario: Email que solo difiere en mayúsculas

- **WHEN** existe una cuenta con `ada@x.com` y se registra `ADA@x.com`
- **THEN** el sistema lo trata como un email distinto y crea una segunda cuenta

#### Scenario: Email con formato inválido o demasiado largo

- **WHEN** el email no tiene formato de dirección de correo o supera los 254 caracteres
- **THEN** la respuesta es un 422 con un error `email` o `maxLength` en el campo `email`

#### Scenario: Contraseña fuera de longitud

- **WHEN** la contraseña o su confirmación tienen menos de 8 o más de 32 caracteres
- **THEN** la respuesta es un 422 con un error `minLength` o `maxLength` en cada campo afectado, indicando el límite en `meta`

#### Scenario: Confirmación distinta

- **WHEN** `passwordConfirmation` no coincide con `password`
- **THEN** la respuesta es un 422 con un error `sameAs` en el campo `passwordConfirmation`

#### Scenario: Cuerpo vacío

- **WHEN** se envía un cuerpo sin ningún campo
- **THEN** la respuesta es un 422 con un error `required` por cada uno de los cuatro campos

### Requirement: Inicio de sesión

El sistema SHALL aceptar `POST /api/v1/auth/login` con `email` y `password` y, si corresponden a una cuenta, devolver el usuario y un token de acceso nuevo. Cada inicio de sesión SHALL emitir un token distinto, y los anteriores SHALL seguir siendo válidos.

#### Scenario: Credenciales correctas

- **WHEN** se envían el email y la contraseña de una cuenta existente
- **THEN** la respuesta es un 200 con `data.user` y `data.token`

#### Scenario: Contraseña incorrecta o email desconocido

- **WHEN** la contraseña no es la de la cuenta, o el email no pertenece a ninguna cuenta
- **THEN** la respuesta es un 400 con el mismo error genérico en ambos casos (`Invalid user credentials`), sin indicar si el email existe

#### Scenario: Email con distinta capitalización

- **WHEN** la cuenta se registró como `dan@x.com` y se intenta entrar con `Dan@X.com`
- **THEN** la respuesta es un 400 de credenciales inválidas: el email debe coincidir exactamente

#### Scenario: Faltan campos

- **WHEN** se envía la petición sin `email` o sin `password`
- **THEN** la respuesta es un 422 con un error `required` por cada campo que falta

### Requirement: Token de acceso

El sistema SHALL identificar a quien hace una petición protegida mediante la cabecera `Authorization: Bearer <token>`. Los tokens SHALL NOT caducar por el paso del tiempo: solo dejan de valer cuando se cierra la sesión con ellos.

#### Scenario: Petición protegida sin token o con token inválido

- **WHEN** se llama a una ruta de cuenta sin cabecera `Authorization`, con un token inventado o con un token ya revocado
- **THEN** la respuesta es un 401 con `{ "errors": [{ "message": "Unauthorized access" }] }`

#### Scenario: Varias sesiones simultáneas

- **WHEN** la misma persona ha iniciado sesión dos veces y tiene dos tokens
- **THEN** ambos tokens funcionan a la vez en las rutas protegidas

### Requirement: Consulta del propio perfil

El sistema SHALL devolver en `GET /api/v1/account/profile` los datos públicos del usuario dueño del token, y solo los suyos.

#### Scenario: Perfil con token válido

- **WHEN** se pide el perfil con un token válido
- **THEN** la respuesta es un 200 con `data` igual a los datos públicos de ese usuario

#### Scenario: Perfil sin identificar

- **WHEN** se pide el perfil sin token válido
- **THEN** la respuesta es un 401

### Requirement: Cierre de sesión en la API

El sistema SHALL aceptar `POST /api/v1/account/logout` con un token válido y revocar exactamente ese token, sin afectar a los demás tokens del mismo usuario. Esta respuesta SHALL NOT ir envuelta en `data`.

#### Scenario: Cierre de sesión correcto

- **WHEN** se cierra sesión con un token válido
- **THEN** la respuesta es un 200 con `{ "message": "Logged out successfully" }` y, a partir de ese momento, ese token recibe un 401 en cualquier ruta protegida

#### Scenario: Las otras sesiones sobreviven

- **WHEN** un usuario con dos tokens cierra sesión con uno de ellos
- **THEN** el otro token sigue obteniendo el perfil con un 200

#### Scenario: Cerrar dos veces la misma sesión

- **WHEN** se intenta cerrar sesión con un token que ya se revocó
- **THEN** la respuesta es un 401

### Requirement: Pantalla de registro

La aplicación web SHALL ofrecer en `/register` un formulario con nombre completo (marcado como opcional), email, contraseña (con la pista «Entre 8 y 32 caracteres.») y repetición de la contraseña, más un enlace «Inicia sesión» hacia `/login`.

#### Scenario: Registro correcto desde la pantalla

- **WHEN** una persona sin sesión rellena el formulario con datos válidos y pulsa «Crear cuenta»
- **THEN** el botón muestra «Creando cuenta…» y queda deshabilitado mientras espera, y al terminar la persona llega a su perfil ya identificada

#### Scenario: Contraseñas que no coinciden

- **WHEN** la persona escribe dos contraseñas distintas y envía el formulario
- **THEN** bajo «Repite la contraseña» aparece «Las contraseñas no coinciden.» sin que se llegue a crear la cuenta

#### Scenario: Errores de validación en castellano

- **WHEN** el servidor rechaza el registro por validación
- **THEN** cada error aparece bajo su campo en castellano (por ejemplo «Ese email ya está registrado. Inicia sesión en su lugar.», «Introduce una dirección de email válida.», «la contraseña debe tener al menos 8 caracteres.»)

### Requirement: Pantalla de inicio de sesión

La aplicación web SHALL ofrecer en `/login` un formulario con email y contraseña, un botón «Entrar» y un enlace «Crea una» hacia `/register`.

#### Scenario: Inicio de sesión correcto desde la pantalla

- **WHEN** una persona sin sesión introduce credenciales válidas y pulsa «Entrar»
- **THEN** el botón muestra «Entrando…» mientras espera y después la persona llega a su perfil

#### Scenario: Credenciales incorrectas

- **WHEN** la persona introduce un email o una contraseña que no corresponden a ninguna cuenta
- **THEN** aparece un aviso destacado arriba del formulario: «El email o la contraseña no son correctos.»

#### Scenario: Servidor inaccesible

- **WHEN** la persona intenta entrar y el servidor no responde
- **THEN** aparece el aviso «No se pudo conectar con el servidor. Comprueba que el backend está arrancado.»

#### Scenario: Error inesperado del servidor

- **WHEN** el servidor responde con un error que no es de validación ni de credenciales
- **THEN** aparece el aviso «Algo ha ido mal en el servidor. Inténtalo de nuevo en un momento.»

### Requirement: Pantalla de perfil

La aplicación web SHALL mostrar en `/profile`, a quien tiene sesión, un círculo con sus iniciales, su nombre completo (o «Sin nombre» si no lo tiene), su email, la fecha de alta como «Miembro desde» en formato largo en castellano (por ejemplo «1 de octubre de 2026») y un botón «Cerrar sesión».

#### Scenario: Ver el perfil

- **WHEN** una persona con sesión abre `/profile`
- **THEN** ve sus iniciales, su nombre o «Sin nombre», su email y la fecha en que se dio de alta

#### Scenario: Cerrar sesión desde la pantalla

- **WHEN** la persona pulsa «Cerrar sesión»
- **THEN** el botón muestra «Cerrando sesión…», la sesión se cierra en el navegador aunque el servidor no responda, y la persona acaba en `/login`

### Requirement: Acceso a las pantallas según la sesión

La aplicación web SHALL dejar ver el perfil solo a quien tiene sesión, y las pantallas de registro e inicio de sesión solo a quien no la tiene.

#### Scenario: Perfil sin sesión

- **WHEN** una persona sin sesión abre `/profile`
- **THEN** se la lleva a `/login`

#### Scenario: Login o registro con sesión

- **WHEN** una persona con sesión abre `/login` o `/register`
- **THEN** se la lleva a `/profile`

#### Scenario: Dirección desconocida

- **WHEN** alguien abre una dirección de la aplicación que no existe
- **THEN** se le lleva a `/profile`, y de ahí a `/login` si no tiene sesión

### Requirement: Recordar la sesión entre visitas

La aplicación web SHALL recordar la sesión en el navegador, de modo que recargar la página o volver más tarde no obligue a iniciar sesión de nuevo mientras el servidor siga aceptando la sesión.

#### Scenario: Recarga con sesión válida

- **WHEN** una persona con sesión recarga la página
- **THEN** ve brevemente un indicador de carga y después sigue en la pantalla en la que estaba, identificada

#### Scenario: Sesión rechazada por el servidor

- **WHEN** la persona vuelve a la aplicación y el servidor ya no acepta su sesión (por ejemplo, porque se cerró en otra ventana)
- **THEN** la aplicación olvida esa sesión y la lleva a `/login` con el aviso «Tu sesión ha caducado. Vuelve a iniciar sesión.»

#### Scenario: Servidor caído al volver

- **WHEN** la persona vuelve a la aplicación y el servidor no responde
- **THEN** la lleva a `/login` con el aviso «No se pudo conectar con el servidor. Comprueba que el backend está arrancado.», pero no olvida la sesión: al recargar con el servidor ya disponible, vuelve a estar identificada
