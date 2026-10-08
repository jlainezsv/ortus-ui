# Implementacion de Cloudflare Turnstile y segunda capa de seguridad

## 1. Objetivo

Este documento describe la implementacion aplicada al formulario de contacto del sitio BoxLang y sirve como guia para replicarla en otro sitio ContentBox/ColdBox/BoxLang de la empresa.

La solucion tiene dos capas principales:

1. **Cloudflare Turnstile**: demuestra server-side que la solicitud proviene de una interaccion aceptada por Cloudflare.
2. **Validacion y reglas de negocio en el servidor**: bloquea solicitudes no deseadas aunque un cliente intente saltarse JavaScript o enviar una peticion HTTP directa.

Turnstile no reemplaza la validacion del servidor. El token del navegador siempre se considera una entrada no confiable hasta que Cloudflare lo acepta mediante `siteverify`.

## 2. Alcance actual

La implementacion descrita corresponde al flujo BoxLang:

- Formulario: formulario de contacto BoxLang.
- Endpoint: `POST /bxcontactus`.
- Handler: `Site.doContactbx`.
- Widget: `TurnstileElement`.
- Servicio de verificacion: `TurnstileService`.
- Configuracion: settings por sitio de ContentBox.

Los formularios legacy del sitio, como `/contactus`, `/sendtips` y otros formularios existentes, continuan usando `cbReCaptcha`. No se debe asumir que todos los formularios del sitio usan Turnstile solo porque este modulo existe.

## 3. Archivos de la implementacion

| Responsabilidad | Archivo |
| --- | --- |
| Ruta HTTP | `config/Router.cfc` |
| Handler y reglas server-side | `handlers/Site.cfc` |
| Servicio que consulta Cloudflare | `modules_app/contentbox-custom/models/turnstile/TurnstileService.cfc` |
| Widget del formulario | `modules_app/contentbox-custom/_themes/boxlang/widgets/TurnstileElement.cfc` |
| Registro del script de Cloudflare | `modules_app/contentbox-custom/ModuleConfig.cfc` |
| Configuracion adicional de bloqueo | `config/Coldbox.cfc`, setting `contacts_blacklist` |

## 4. Flujo completo

```text
Usuario
  |
  | 1. Abre el formulario
  v
ContentBox / tema BoxLang
  |
  | 2. El modulo agrega api.js solo si existe site key
  | 3. El widget hace render explicito de Turnstile
  v
Cloudflare Turnstile
  |
  | 4. Inyecta el campo oculto cf-turnstile-response
  v
Navegador
  |
  | 5. POST /bxcontactus con FormData + token
  v
Site.doContactbx
  |
  | 6. Comprueba blacklist y normaliza parametros
  | 7. Envia token a Cloudflare /siteverify
  | 8. Comprueba success, action y hostname
  | 9. Comprueba longitud y contenido del mensaje
  v
MailService
  |
  | 10. Solo despues de todas las validaciones se envia el correo
  v
Respuesta AJAX JSON
```

La condicion importante es que el correo se envia al final. Un cliente puede eliminar el widget, modificar el JavaScript o llamar directamente al endpoint, pero no puede obtener un envio valido sin pasar por las comprobaciones server-side.

## 5. Configuracion por sitio

Se utilizan dos settings de ContentBox:

| Setting | Uso | Sensibilidad |
| --- | --- | --- |
| `cb_turnstile_site_key` | Clave publica usada por el navegador para renderizar Turnstile | Publica |
| `cb_turnstile_secret_key` | Clave privada enviada a Cloudflare en `siteverify` | Secreta |

Los nombres estan definidos en:

- `TurnstileService.cfc`: lectura de site key y secret key.
- `TurnstileElement.cfc`: lectura de la site key para el HTML/JavaScript.
- `ModuleConfig.cfc`: deteccion de la site key para cargar el script.

### Reglas de configuracion

- Crear las claves en Cloudflare Turnstile para el hostname real del sitio.
- Registrar tambien los hostnames de staging si se probaran fuera de produccion.
- Guardar el secret key solo como setting protegido o variable segura de despliegue.
- No incluir el secret key en JavaScript, HTML, repositorio, capturas ni logs.
- Configurar los dos settings para el mismo sitio ContentBox. El servicio resuelve el sitio actual mediante `siteService.discoverSite().getSlug()`.
- Si no existe `cb_turnstile_site_key`, el script no se carga y el widget devuelve HTML vacio.
- Si no existe `cb_turnstile_secret_key`, la verificacion falla de forma segura y no se envia correo.

Ejemplo conceptual de los valores (usar valores reales solo en el administrador o secret store):

```text
Site: BoxLang
cb_turnstile_site_key: 0x4AAAA...
cb_turnstile_secret_key: 0x4AAAA...
```

## 6. Carga condicional del script

`ModuleConfig.cfc` registra un interceptor para `cbui_beforeHeadEnd`. El interceptor consulta la site key actual y agrega una sola vez:

```html
<script src="https://challenges.cloudflare.com/turnstile/v0/api.js?render=explicit"></script>
```

La carga es condicional por dos motivos:

1. No cargar JavaScript de Cloudflare en sitios que no tienen Turnstile configurado.
2. Evitar duplicados si otro componente ya agrego la misma URL.

Patron para replicar:

```cfml
function cbui_beforeHeadEnd(event, interceptData, buffer) {
    if (!len(getCurrentTurnstileSiteKey(arguments.event))) {
        return;
    }

    if (toString(arguments.buffer).findNoCase(variables.turnstileScriptUrl) > 0) {
        return;
    }

    arguments.buffer.append(
        '<script src="' & variables.turnstileScriptUrl & '"></script>'
    );
}
```

## 7. Render del widget

El widget usa la API de render explicito. Esto permite controlar el `sitekey`, `action`, apariencia, tamano y nombre del campo de respuesta.

Configuracion actual:

```text
action: contact
size: flexible
appearance: execute
theme: auto
response-field: true
response-field-name: cf-turnstile-response
```

Ejemplo de render del lado del cliente:

```html
<div id="turnstileWidget"></div>

<script>
    window.turnstileContactWidget = function () {
        var container = document.getElementById("turnstileWidget");

        if (!container || container.dataset.turnstileRendered === "1") {
            return;
        }

        if (!window.turnstile || typeof window.turnstile.render !== "function") {
            return;
        }

        container.dataset.turnstileRendered = "1";

        window.turnstile.render(container, {
            sitekey: "PUBLIC_SITE_KEY",
            action: "contact",
            theme: "auto",
            size: "flexible",
            appearance: "execute",
            "response-field": true,
            "response-field-name": "cf-turnstile-response"
        });
    };

    if (window.turnstile) {
        window.turnstile.ready(window.turnstileContactWidget);
    }
</script>
```

En la implementacion real la site key se inserta desde el setting del sitio. Al producir HTML con valores de configuracion, debe aplicarse encoding para atributo/JavaScript cuando corresponda. Nunca se debe insertar el secret key en este bloque.

El campo oculto generado por Turnstile se envia automaticamente junto con `FormData`:

```text
cf-turnstile-response=<token-de-un-solo-uso>
```

Los tokens son de un solo uso y expiran. Por eso no se deben guardar para reintentos posteriores ni reutilizar despues de una respuesta rechazada.

## 8. Envio AJAX del formulario

El widget conecta el formulario con el endpoint y envia todos sus campos, incluido el token generado por Turnstile:

```javascript
form.addEventListener("submit", function (event) {
    event.preventDefault();

    fetch("/bxcontactus", {
        method: "POST",
        body: new FormData(form),
        headers: {
            "X-Requested-With": "fetch"
        }
    })
    .then(function (response) {
        return response.json();
    })
    .then(function (data) {
        if (data.success) {
            form.reset();
        }
    });
});
```

El header `X-Requested-With: fetch` permite que el handler devuelva JSON para la experiencia AJAX. No es una medida de seguridad: un atacante puede falsificarlo.

El cliente tambien:

- Evita doble submit mientras la peticion esta pendiente.
- Deshabilita el boton si el mensaje tiene menos de 20 caracteres.
- Muestra mensajes de error sin recargar la pagina.

Estas validaciones mejoran la experiencia, pero el servidor las repite porque el navegador no es una frontera de confianza.

## 9. Verificacion server-side contra Cloudflare

El servicio `TurnstileService` es singleton y centraliza toda la verificacion. Recibe:

```cfml
turnstileService.verify(
    token  = rc["cf-turnstile-response"],
    action = "contact"
)
```

Luego hace un `POST` form-encoded a:

```text
https://challenges.cloudflare.com/turnstile/v0/siteverify
```

Parametros enviados:

```text
secret   = secret key del sitio
response = token recibido del navegador
remoteip = IP del cliente, cuando esta disponible
```

Ejemplo reducido y portable:

```cfml
struct function verify(required string token, string action = "") {
    var result = {
        success: false,
        action: "",
        hostname: "",
        errorCodes: [],
        errorMessage: ""
    };

    if (!len(trim(arguments.token))) {
        result.errorCodes = ["missing-input-response"];
        result.errorMessage = "Turnstile token is required.";
        return result;
    }

    var httpService = new http(
        method = "post",
        url = "https://challenges.cloudflare.com/turnstile/v0/siteverify",
        timeout = 10
    );

    httpService.addParam(
        type = "formfield",
        name = "secret",
        value = getSecretKey()
    );
    httpService.addParam(
        type = "formfield",
        name = "response",
        value = arguments.token
    );

    var response = httpService.send().getPrefix();
    var payload = deserializeJSON(response.fileContent ?: "{}");

    if (!isStruct(payload) || !payload.success) {
        result.errorMessage = "Cloudflare rejected the Turnstile token.";
        return result;
    }

    if (len(arguments.action) && payload.action != arguments.action) {
        result.errorCodes = ["action-mismatch"];
        result.errorMessage = "The Turnstile action was invalid.";
        return result;
    }

    result.success = true;
    result.action = payload.action ?: "";
    result.hostname = payload.hostname ?: "";
    return result;
}
```

La implementacion real es mas estricta que este ejemplo y ademas:

- Rechaza tokens de mas de 2048 caracteres.
- Usa timeout HTTP de 10 segundos.
- Maneja status HTTP no exitosos.
- Maneja respuestas vacias o JSON inesperado.
- Conserva `error-codes`, `action`, `hostname`, `challenge_ts` y `cdata`.
- Registra fallos en LogBox sin exponer el token ni el secret.
- Rechaza la respuesta si falta `success` o si `success` es falso.
- Rechaza `action` distinto de `contact`.
- Compara el hostname devuelto por Cloudflare con el hostname publico de la peticion.

## 10. Resolucion de hostname e IP

Para validar el hostname, el servicio usa esta prioridad:

1. `X-Forwarded-Host`.
2. `Host`.
3. `cgi.http_host`.

Se elimina el puerto y se compara en minusculas con el hostname retornado por Cloudflare.

Para enviar `remoteip`, usa esta prioridad:

1. `cf-connecting-ip`.
2. `x-cluster-client-ip`.
3. `X-Forwarded-For`.
4. `cgi.remote_addr`.
5. `127.0.0.1` como ultimo fallback.

Al replicar esto detras de un proxy, load balancer o CDN, solo se deben aceptar headers de IP que el proxy confiable sobrescriba y controle. No se debe confiar ciegamente en headers enviados por el cliente desde Internet.

## 11. Segunda capa: reglas server-side del formulario

### 11.1 Blacklist de correos

Antes de procesar el envio, `doContactbx` compara el email con la setting `contacts_blacklist`. Si encuentra una coincidencia, responde como si la solicitud hubiera sido aceptada, pero no envia correo.

Este comportamiento es deliberadamente silencioso para no confirmar al abusador que su direccion esta bloqueada.

La lista se configura actualmente en `config/Coldbox.cfc`:

```cfml
settings = {
    contacts_blacklist: [
        "blocked@example.com"
    ]
};
```

Para otro sitio se recomienda mantener la lista fuera del codigo cuando sea posible, normalizar el email antes de comparar y registrar metricas internas sin revelar la lista al usuario.

### 11.2 Token obligatorio y verificacion de action/hostname

El endpoint normaliza el parametro:

```cfml
event.paramValue("cf-turnstile-response", "");
```

Un token vacio falla. Un token valido para otra action o hostname tambien falla. Por tanto, no basta con copiar un token generado para otro formulario o dominio.

### 11.3 Validacion server-side del mensaje

Solo si Turnstile es valido se ejecuta la validacion de contenido:

```cfml
var trimmedMessage = trim(rc.message);

if (len(trimmedMessage) < 20) {
    arrayAppend(errors, "Please provide at least 20 characters in your message.");
} else if (
    len(trimmedMessage) >= 15
    && !reFind("\\s", trimmedMessage)
    && reFind("^[[:alnum:]]+$", trimmedMessage)
) {
    arrayAppend(errors, "Please provide a more detailed message.");
}
```

Esto bloquea mensajes vacios, demasiado cortos y cadenas alfanumericas sin espacios que suelen indicar entradas automatizadas o poco utiles. Esta regla no pretende clasificar todo spam; es una segunda barrera de calidad y abuso.

### 11.4 Orden de las comprobaciones

El orden actual es importante:

1. Normalizar parametros.
2. Aplicar blacklist.
3. Verificar Turnstile contra Cloudflare.
4. Validar el mensaje.
5. Construir y enviar el correo.
6. Responder JSON o redirect segun el tipo de solicitud.

Nunca se debe mover el envio de correo antes de la verificacion de Turnstile y las validaciones.

## 12. Handler de referencia

Este es un ejemplo resumido del endpoint server-side:

```cfml
function doContactbx(event, rc, prc) {
    var errors = [];
    var isAjax = event.getHTTPHeader("X-Requested-With", "");

    event
        .paramValue("email", "")
        .paramValue("message", "")
        .paramValue("cf-turnstile-response", "");

    if (arrayFind(blacklist, rc.email)) {
        if (isAjax == "fetch") {
            return {
                success: false,
                message: "Thank you, we will respond shortly!"
            };
        }
        relocate("/");
    }

    var turnstileResult = turnstileService.verify(
        token = rc["cf-turnstile-response"],
        action = "contact"
    );

    if (!turnstileResult.success) {
        arrayAppend(errors, "We couldn't send your message. Please try again.");
    }

    if (turnstileResult.success && len(trim(rc.message)) < 20) {
        arrayAppend(errors, "Please provide at least 20 characters in your message.");
    }

    if (arrayLen(errors)) {
        if (isAjax == "fetch") {
            return {
                success: false,
                message: errors.toList(" ")
            };
        }
        return relocate("/?cbcache=true");
    }

    // Construir y enviar el email aqui, despues de todas las validaciones.
    return {
        success: true,
        message: "Thank you, we will respond shortly!"
    };
}
```

El codigo real conserva los campos completos del formulario, el manejo de errores de mail y las respuestas para solicitudes AJAX y tradicionales.

## 13. Checklist de replica

### Cloudflare

- [ ] Crear un Turnstile widget para cada hostname necesario.
- [ ] Registrar produccion y staging por separado cuando corresponda.
- [ ] Guardar site key y secret key.
- [ ] Confirmar que el hostname retornado por Cloudflare coincide con el hostname publico.

### ContentBox

- [ ] Crear `cb_turnstile_site_key` en los settings del sitio.
- [ ] Crear `cb_turnstile_secret_key` en los settings del sitio.
- [ ] Verificar que ambos settings pertenecen al mismo site slug.
- [ ] No versionar el secret key.

### ColdBox / modulo

- [ ] Registrar `TurnstileService` como modelo singleton o usar auto-mapping.
- [ ] Registrar el interceptor de carga de `api.js`.
- [ ] Incluir el widget en el formulario correcto.
- [ ] Añadir una ruta POST dedicada al handler.
- [ ] Inyectar el servicio en el handler.
- [ ] Mantener timeout, limite de token y manejo de errores.

### Formulario

- [ ] Renderizar Turnstile con `response-field` habilitado.
- [ ] Usar un `action` especifico por formulario.
- [ ] Enviar `FormData` para incluir el token oculto.
- [ ] Evitar doble submit mientras hay una peticion activa.
- [ ] No confiar en las validaciones JavaScript.

### Servidor

- [ ] Rechazar token ausente, demasiado largo, expirado o reutilizado.
- [ ] Comprobar `success`.
- [ ] Comprobar `action`.
- [ ] Comprobar `hostname`.
- [ ] Validar email, mensaje y limites de entrada.
- [ ] Aplicar blacklist o reglas anti-abuso antes de enviar correo.
- [ ] Registrar fallos sin registrar secretos ni tokens completos.
- [ ] Revisar la confianza en `X-Forwarded-*` segun el proxy real.

## 14. Plan de pruebas

### Pruebas funcionales

1. Abrir el formulario con ambos settings configurados.
2. Confirmar que `api.js` aparece una sola vez.
3. Confirmar que el widget renderiza y crea `cf-turnstile-response`.
4. Enviar un mensaje valido y confirmar respuesta JSON de exito.
5. Confirmar que llega un solo correo.

### Pruebas negativas

1. Enviar sin `cf-turnstile-response`: debe fallar y no enviar correo.
2. Enviar token invalido: debe fallar.
3. Reutilizar el mismo token: debe fallar por token de un solo uso.
4. Enviar un token con action distinta de `contact`: debe fallar.
5. Enviar un token emitido para otro hostname: debe fallar.
6. Enviar mensaje de menos de 20 caracteres: debe fallar.
7. Enviar una cadena alfanumerica sin espacios: debe fallar cuando aplique la regla.
8. Enviar un email de la blacklist: no debe producir correo y no debe revelar el bloqueo.
9. Eliminar JavaScript y hacer POST directo: el servidor debe seguir rechazando la peticion sin token valido.
10. Simular error o timeout de Cloudflare: no debe enviarse correo.

### Pruebas operativas

- Revisar LogBox para errores de configuracion, respuestas HTTP no exitosas y rechazos de Cloudflare.
- Verificar que no aparecen tokens o secret keys en logs.
- Probar detras del proxy real y confirmar hostname/IP.
- Probar staging con su hostname registrado en Cloudflare.
- Confirmar que los formularios legacy siguen usando su mecanismo previsto y no dependen accidentalmente de Turnstile.

## 15. Limitaciones y mejoras recomendadas

La implementacion actual protege el flujo contra bots y entradas automatizadas, pero no es una solucion completa contra abuso volumetrico. Para un sitio con mayor riesgo se recomienda agregar:

- Rate limiting por IP, email y ventana de tiempo en el edge o reverse proxy.
- CSRF token para endpoints autenticados o flujos donde el navegador mantenga una sesion sensible.
- Limites de longitud para todos los campos antes de construir el correo.
- Normalizacion de email antes de consultar la blacklist.
- Politica clara de headers confiables en el proxy.
- Metricas de tasa de rechazos y alertas sin guardar datos sensibles.
- Rotacion de secret keys y procedimiento de revocacion.
- Politica CSP compatible con `https://challenges.cloudflare.com`.

Estas mejoras son complementarias. No sustituyen la verificacion server-side de Turnstile, la comprobacion de `action` y hostname, ni la validacion del mensaje.

## 16. Resumen para una nueva implementacion

La receta minima reproducible es:

1. Crear el widget Turnstile y registrar el hostname.
2. Guardar site key y secret key como settings por sitio.
3. Cargar `api.js` solo cuando exista la site key.
4. Renderizar Turnstile explicitamente con una action propia.
5. Enviar el token en un campo conocido, por ejemplo `cf-turnstile-response`.
6. Verificar el token desde el servidor con `/siteverify`.
7. Exigir `success`, `action` y `hostname` correctos.
8. Ejecutar validaciones de email, blacklist y contenido en el servidor.
9. Enviar correo solo despues de todas las validaciones.
10. Probar casos positivos, tokens invalidos, replay, hostname/action incorrectos y POST directo.
