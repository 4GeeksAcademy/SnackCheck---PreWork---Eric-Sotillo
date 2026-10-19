# SnackCheck — Registro de pruebas y resultados

> **Contexto:** pruebas manuales con Hoppscotch sobre `POST /webhook-test/nutrition-check`, salvo indicación contraria. Las duraciones representan el tiempo total observado desde el cliente en una ejecución aislada. No son pruebas de carga ni constituyen un SLA. Se distinguen claramente resultados vistos, resultados esperados y procedimientos de simulación.


## Contrato de salida comprobado contra el enunciado

El apartado **«Salida»** de 4Geeks establece: `200` para veredictos, autodocumentación y datos insuficientes; `400` para errores de validación; `404` para producto no encontrado. El veredicto se presenta como **texto plano**; autodocumentación, datos insuficientes y errores se devuelven como **JSON**, con la cabecera `Content-Type` correspondiente. Por eso **TC-016 (200 + `PRODUCT_MISSING_DATA`) cumple el requisito**, aunque el campo del cuerpo sea `status: "error"`. Los errores `401`, `429`, `500` y `504` son ramas adicionales implementadas en SnackCheck y documentadas como procedimientos de simulación.


## Caso de Prueba: Solicitud Vacia - Autodocumentacion (Verificado)

**ID:** TC-001
**Objetivo:** Verifico que al enviar una solicitud al webhook sin cuerpo, se envie la autodocumentacion de la API.
**Entrada:** Mediante Hoppscotch: POST https://ericdevs2.app.n8n.cloud/webhook-test/nutrition-check — sin cuerpo (ejecución de comprobación).
**Salida Esperada:**

```json
{
"title": "API SnackCheck - Evaluacion de Productos",
"objetivo": "El objetivo de esta API es poder deteminar si un producto, mediante su codigo de barra, es Saludable - Moderador o No Saludable, y poder entregar una recomendacion al usuario. Utilizamos la API de Open Food Facs para obtener los datos nutricionales de los productos.",
"method": "POST",
"endpoint": "https://ericdevs2.app.n8n.cloud/webhook/nutrition-check",
"required_fields": {
"barcode": "string or Int"
},
"barcode_format_supported": [
"EAN-8",
"EAN-13",
"UPC-A",
"UPC-E"
],
"responses": {
"200": "Evaluación realizada correctamente",
"400": "Datos de entrada inválidos",
"401": "Unauthorized - API (Open Food Facts)",
"404": "Producto no encontrado",
"429": "To many Request - API (Open Food Facts)",
"500": "Error interno de Servidor"
}
}
```

**Pasos de Prueba:**

1. En Hoppscotch, configurar el método POST y la URL de pruebas https://ericdevs2.app.n8n.cloud/webhook-test/nutrition-check.
2. Dejo esta solicitud sin Cuerpo
3. Envio Solicitud y recibo un Mensaje JSON con la autodocumentacion configurada

**Resultado observado:** HTTP 200 OK. Se devuelve un objeto JSON de autodocumentación con `title`, `objetivo`, `method: POST`, `endpoint`, `required_fields.barcode`, `barcode_format_supported` y `responses`. Los campos visibles coinciden con lo esperado. La URL indicada dentro de la respuesta es el endpoint publicado (`/webhook/`), aunque el ensayo se ejecutó en el endpoint de pruebas (`/webhook-test/`).
**Estado:** Aprobado — reconfirmado mediante captura de Hoppscotch.
**Tiempo de respuesta (Hoppscotch):** 1280 ms.
**Tamaño de respuesta (Hoppscotch):** 2.24 KB.
**Notas:** Solicitud POST sin cuerpo. Los códigos HTTP incluidos dentro de la autodocumentación son información de la API; este caso no demuestra que se hayan ejecutado los escenarios correspondientes a esos códigos.

# Pruebas de entrada - Webhook Codigo de Barra

Valido las entradas y salidas posibles, con los posibles errores esperados:

## Pruebas de Entradas Validas (Encontrados) y diferentes caracteristicas:

### Caso de Prueba: Formato EAN-13 — Dos nutrientes altos y Nutri-Score e

**ID:** TC-002  
**Objetivo:** Verificar que la API acepta un código EAN-13 válido, recupera sus datos de Open Food Facts y clasifica un producto con **dos nutrientes `Alto`** (azúcar y grasas) como `No Saludable` mediante el semáforo. Comprobar además que su Nutri-Score `e` coincide con esa clasificación y que el sistema devuelve el veredicto final `No Saludable` junto con una **recomendación de tono cauteloso**.  
**Entrada:** `{"barcode":"3017620422003"}` — 13 dígitos (EAN-13).  
**Endpoint utilizado:** `POST https://ericdevs2.app.n8n.cloud/webhook-test/nutrition-check`.  
**Salida esperada:** HTTP `200 OK`, veredicto en texto plano `No Saludable`, dos nutrientes `Alto`, Nutri-Score `e` y recomendación cautelosa.

**Resultado observado (ejecución completa aportada):**
- **HTTP:** `200 OK`.
- **Tiempo de respuesta (Hoppscotch):** **2928 ms** (2,928 s); una ejecución anterior del mismo producto mostró `2915 ms`.
- **Tamaño de respuesta:** `2.32 KB`.
- **Producto:** `Testglas`.
- **Marca:** `Nutella, Ferrero`.
- **Nutri-Score:** `e` → `No Saludable`.
- **Azúcar:** `56.3 g/100 g` → `Alto`.
- **Sal:** `0.11 g/100 g` → `Bajo`.
- **Grasas:** `30.9 g/100 g` → `Alto`.
- **Calorías:** `539 kcal/100 g`.
- **Semáforo calculado:** `No Saludable` (dos nutrientes altos).
- **Veredicto final observado:** **`No Saludable`**.
- **Recomendación observada:** aconseja limitar el consumo, explica la contribución del azúcar y las grasas y sugiere alternativas y consumo ocasional. El tono es cauteloso y concuerda con el veredicto.

**Estado:** **Aprobado** — formato de entrada, HTTP, nutrientes, semáforo, Nutri-Score, veredicto y recomendación contrastados con la captura y el texto de respuesta compartidos.  
**Notas:** La recomendación menciona «grasas saturadas», pero el dato mostrado por SnackCheck corresponde a **grasas totales**. Es una observación menor de precisión del texto de IA, no un fallo de clasificación. TC-004 también comprueba dos nutrientes altos, pero con un tercero `Medio`, mientras que TC-002 presenta un tercero `Bajo`. Este TC-002 se documenta según la ejecución real de Testglas; el intento exploratorio con Nestea no forma parte de su aprobación.

### Caso de Prueba: Formato EAN-13 - 1 nutriente alto y Nutri-Score restrictivo

**ID:** TC-003  
**Objetivo:** Comprobar que un EAN-13 correspondiente a un producto con un nutriente alto produce semáforo `Moderado`, pero que un Nutri-Score `d` da como resultado final `No Saludable`.  
**Entrada:** `{"barcode":"4001686301524"}` (13 dígitos).  
**Salida esperada:** `No Saludable`; recomendación cautelosa y respetuosa.  
**Resultado observado:** HTTP `200 OK`; producto *Haribo Goldbären 20 Minis* (Haribo); Nutri-Score `d`; azúcar `46 g/100 g` (`Alto`); sal `0.1 g/100 g` (`Bajo`); grasas `0.5 g/100 g` (`Bajo`); calorías `343 kcal/100 g`. El veredicto final mostrado es `No Saludable`, coherente con elegir el resultado más restrictivo. El texto recomienda moderar el consumo y no contradice los valores mostrados.  
**Tiempo de respuesta (Hoppscotch):** `2671 ms`.  
**Tamaño de respuesta:** `2.34 KB`.  
**Estado:** **Aprobado** — ejecución completa y salida textual aportadas.  
**Notas:** En versiones anteriores figuraba por error Nutri-Score `e`; la ejecución observada devuelve `d`. Ambos grados llevan a `No Saludable`, pero el registro debe reflejar el valor real.

### Caso de Prueba: Formato EAN-13 - 2 nutrientes altos y 1 medio, Nutri-Score e

**ID:** TC-004  
**Objetivo:** Comprobar que dos nutrientes altos determinan `No Saludable` por semáforo y que Nutri-Score `e` coincide con el veredicto.  
**Entrada registrada:** `{"barcode":"8076809513753"}` (EAN-13; entrada confirmada en la captura).  
**Salida esperada:** `No Saludable`; recomendación cautelosa.  
**Resultado observado (texto aportado):** *Barilla PESTO alla GENOVESE* (Barilla), Nutri-Score `e`; azúcar `5 g/100 g` (`Medio`), sal `3.2 g/100 g` (`Alto`), grasas `47 g/100 g` (`Alto`), calorías `492 kcal/100 g`; veredicto final `No Saludable`. La recomendación advierte sobre grasas, sal y porciones y mantiene un tono cauteloso.  
**Tiempo de respuesta (Hoppscotch):** `3260 ms` (captura de la repetición aportada).  
**Estado:** **Aprobado funcionalmente** — la salida textual completa coincide con el escenario.  
**Notas:** Tiempo transcrito de la captura de Hoppscotch.

### Caso de Prueba: Formato EAN-13 - 0 nutrientes altos, Nutri-Score c

**ID:** TC-005  
**Objetivo:** Comprobar que cero nutrientes altos producen semáforo `Saludable`, pero que Nutri-Score `c` determina el veredicto final `Moderado`.  
**Entrada registrada:** `{"barcode":"5449000147417"}` (EAN-13; entrada confirmada en la captura).  
**Salida esperada:** `Moderado`; recomendación alentadora y prudente.  
**Resultado observado (texto aportado):** *Cappy PULPY* (Cappy), Nutri-Score `c`; azúcar `4.4 g/100 g` (`Bajo`), sal `0.04 g/100 g` (`Bajo`), grasas `0 g/100 g` (`Bajo`), calorías `21 kcal/100 g`; veredicto final `Moderado`. El tono es alentador.  
**Tiempo de respuesta (Hoppscotch):** `2685 ms` (captura de la repetición aportada).  
**Estado:** **Aprobado funcionalmente, con observación menor de texto de IA**.  
**Notas:** La frase «contribuirá a tu bienestar sin exceder tus necesidades nutricionales» es demasiado categórica sin conocer a la persona. No cambia el veredicto, pero conviene evitar este tipo de garantías en mejoras futuras.

### Caso de Prueba: Formato EAN-13 - 0 nutrientes altos, Nutri-Score b

**ID:** TC-006  
**Objetivo:** Comprobar que cero nutrientes altos y Nutri-Score `b` producen el veredicto final `Saludable`.  
**Entrada registrada:** `{"barcode":"3168930163480"}` (EAN-13; entrada confirmada en la captura).  
**Salida esperada:** `Saludable`; recomendación alentadora.  
**Resultado observado (texto aportado):** *Alvalle Gazpacho l'original* (ALVALLÉ), Nutri-Score `b`; azúcar `3.3 g/100 g` (`Bajo`), sal `0.67 g/100 g` (`Bajo`), grasas `2.2 g/100 g` (`Bajo`), calorías `40 kcal/100 g`; veredicto final `Saludable`. La recomendación mantiene un tono positivo.  
**Tiempo de respuesta (Hoppscotch):** `2167 ms` (captura de la repetición aportada).  
**Estado:** **Aprobado funcionalmente, con observación menor de texto de IA**.  
**Notas:** «Inclúyelo en tus comidas diarias» es una recomendación más prescriptiva de lo necesario; bastaría con sugerir integrarlo dentro de una alimentación variada.

### Caso de Prueba: Formato EAN-13 - 1 nutriente alto, Nutri-Score favorable

**ID:** TC-007  
**Objetivo:** Verificar que un EAN-13 con un nutriente `Alto` obtiene semáforo `Moderado` y que este prevalece frente a Nutri-Score `a` (`Saludable`).  
**Entrada:** `{"barcode":"4068262073381"}` — 13 dígitos.  
**Endpoint:** POST `/webhook-test/nutrition-check`.  
**Salida esperada:** Veredicto `Moderado` y recomendación alentadora con moderación.  
**Resultado observado:** HTTP `200 OK`; producto *Beurre De Cacahuètes* (Aldi, Trader Joe's); Nutri-Score `a`; azúcar `5.5 g/100 g` (`Medio`); sal `0.01 g/100 g` (`Bajo`); grasas `50 g/100 g` (`Alto`); calorías `625.956022944551 kcal/100 g`; veredicto final **`Moderado`**. La recomendación alienta el consumo moderado y reconoce el alto contenido de grasa.  
**Tiempo de respuesta (Hoppscotch):** `2871 ms` (última ejecución); una ejecución anterior registró `2183 ms`.  
**Tamaño de respuesta (Hoppscotch):** `2.45 KB`.  
**Estado:** **Aprobado funcionalmente** — entrada, clasificación, resultado y recomendación verificados.  
**Notas:** La propuesta de «una o dos cucharadas al día» es demasiado específica sin conocer las necesidades individuales. En esta ejecución histórica las calorías aparecen con muchos decimales; el **export final revisado** ya incorpora redondeo de nutrientes y calorías en la plantilla de presentación.

### Caso de Prueba: Formato UPC-A de 12 dígitos - 1 nutriente alto, Nutri-Score c

**ID:** TC-008  
**Objetivo:** Verificar que un UPC-A de 12 dígitos conserva el cero inicial y que un nutriente `Alto` con Nutri-Score `c` produce veredicto final `Moderado`.  
**Entrada:** `{"barcode":"044000081867"}` — 12 dígitos; dígito de control UPC-A válido. **Importante:** el barcode se envía como cadena para conservar el cero inicial.  
**Salida esperada:** Veredicto `Moderado` y recomendación alentadora con moderación.  
**Resultado observado (texto aportado):** producto *Chocolate Sandwich Cookies* (Oreo); Nutri-Score `c`; azúcar `0 g/100 g` (`Bajo`); sal `0.94 g/100 g` (`Bajo`); grasas `19.91 g/100 g` (`Alto`); calorías `398.230088495575 kcal/100 g`; veredicto final **`Moderado`**. La recomendación indica consumo moderado y es globalmente coherente con la clasificación.  
**Tiempo de respuesta (Hoppscotch):** `3275 ms` (captura aportada).  
**Estado:** **Aprobado** — salida y tiempo de respuesta observados.  
**Notas:** El resultado observado muestra `0 g` de azúcar; con esta captura por sí sola no se puede verificar la variante comercial exacta del producto ni la calidad de todos los campos de Open Food Facts. No se atribuye automáticamente ese dato a un fallo de SnackCheck. La recomendación no debe prometer beneficios de salud personalizados. En esta ejecución histórica las calorías se mostraron con muchos decimales; el **export final revisado** ya redondea ese valor en la plantilla de presentación.

### Caso de Prueba: Formato EAN-8 de 8 dígitos — sin nutrientes altos, Nutri-Score e

**ID:** TC-009  
**Objetivo:** Verificar que el endpoint acepta un EAN-8 de 8 dígitos y que Nutri-Score `e` determina `No Saludable` aunque el semáforo nutricional no tenga nutrientes `Alto`.  
**Entrada:** `{"barcode":"54490000"}` — EAN-8.  
**Endpoint:** POST `/webhook-test/nutrition-check`.  
**Salida esperada:** Veredicto `No Saludable` y recomendación cautelosa.  
**Resultado observado:** HTTP `200 OK`; producto *Coca-Cola Original Taste*, marca Coca-Cola, Nutri-Score `e`; azúcar `10.6 g/100 g` (`Medio`), sal `0 g/100 g` (`Bajo`), grasas `0 g/100 g` (`Bajo`); `42 kcal/100 g`; semáforo `Saludable` (0 nutrientes altos); veredicto final **`No Saludable`**. La recomendación aportada es cautelosa y reconoce el contenido de azúcar.  
**Tiempo de respuesta (Hoppscotch):** `2963 ms`.  
**Estado:** **Aprobado** — formato, clasificación final y tono de la recomendación verificados.  
**Notas:** La versión previa del objetivo describía erróneamente el semáforo y el Nutri-Score como `Moderado`; se corrigió para coincidir con el caso efectivamente probado.

## Pruebas de Casos de errores

### Caso de Prueba: Solicitud sin variable `barcode`

**ID:** TC-010  
**Objetivo:** Comprobar que la API rechaza una solicitud POST con cuerpo JSON que no contiene el campo requerido `barcode`.  
**Entrada:** `{"bar":1}` (se envía `bar`, no `barcode`).  
**Endpoint:** POST `/webhook-test/nutrition-check`.  
**Salida esperada:** HTTP `400 Bad Request`, `status: "error"`, `code: "BARCODE_MISSING"`, `barcode_send: null` y `barcode_status: "Invalid"`.  
**Resultado observado:** HTTP `400 Bad Request`. Se recibió el siguiente cuerpo JSON:

```json
{
  "status": "error",
  "http_code": 400,
  "code": "BARCODE_MISSING",
  "barcode_send": null,
  "barcode_status": "Invalid",
  "error": "No se ha encontrado un codigo de barras proporcionado a la API o no cumple con los estandares (EAN-8, EAN-13, UPC-A, UPC-E)",
  "example": "Envia: {'barcode': '3017620422003'} ",
  "hint": "Verifica que el barcode se este enviando y que cuente con 8,12, o 13 numeros sin caracteres",
  "timestamp": "2026-10-19T23:52:11.319+02:00"
}
```

**Tiempo de respuesta (Hoppscotch):** `1251 ms`.  
**Tamaño de respuesta (Hoppscotch):** `1.84 KB`.  
**Estado:** **Aprobado** — código HTTP y mensaje de error coinciden con lo esperado.  
**Notas:** `barcode_send` es `null` (valor JSON, no texto), porque el parámetro requerido no fue suministrado. La ejecución termina correctamente en la validación de entrada, sin necesidad de consultar Open Food Facts ni generar una recomendación.

### Caso de Prueba: Solicitud con `barcode` vacío

**ID:** TC-011  
**Objetivo:** Verificar que una solicitud con el campo `barcode` presente, pero vacío, se rechaza con error de entrada.  
**Entrada:** `{"barcode":""}`.  
**Endpoint:** POST `/webhook-test/nutrition-check`.  
**Salida esperada:** HTTP `400 Bad Request`, `code: "BARCODE_MISSING"`, `barcode_send: ""` y `barcode_status: "Invalid"`.  
**Resultado observado:** HTTP `400 Bad Request`; la respuesta contiene `status: "error"`, `http_code: 400`, `code: "BARCODE_MISSING"`, `barcode_send: ""`, `barcode_status: "Invalid"`, además de `error`, `example`, `hint` y `timestamp`.  
**Tiempo de respuesta (Hoppscotch):** `1254 ms` (1,254 s).  
**Tamaño de respuesta (Hoppscotch):** `1.84 KB`.  
**Estado:** **Aprobado** — coincide con el caso esperado.  
**Notas:** La propiedad `barcode_send` conserva la cadena vacía `""`, a diferencia de TC-010, en el cual el campo no existe y se devuelve `null`. Ambas variantes son rechazadas mediante HTTP 400.

### Caso de Prueba: Solicitud `barcode` de 13 caracteres con letras

**ID:** TC-012  
**Objetivo:** Verificar que una entrada de 13 caracteres que contiene letras se rechaza, aunque su longitud sea admisible para un EAN-13.  
**Entrada:** `{"barcode":"3017620422aa3"}`.  
**Endpoint:** POST `/webhook-test/nutrition-check`.  
**Salida esperada:** HTTP `400 Bad Request`, `code: "BARCODE_INVALID"`, `barcode_send: "3017620422aa3"` y `barcode_status: "Invalid"`.  
**Resultado observado:** HTTP `400 Bad Request` con los siguientes campos en el cuerpo JSON:

```json
{
  "status": "error",
  "http_code": 400,
  "code": "BARCODE_INVALID",
  "barcode_send": "3017620422aa3",
  "barcode_status": "Invalid",
  "error": "El codigo de barra no cumple los estandares (EAN-8, EAN-13, UPC-A, UPC-E), deben ser solo numeros y no incluir letras",
  "example": "Envia: {'barcode': '3017620422003'} ",
  "hint": "El codigo de barra debe tener 8, 12 o 13 caracteres unicamente numericos",
  "timestamp": "2026-10-20T00:00:14.101+02:00"
}
```

**Tiempo de respuesta (Hoppscotch):** `1174 ms` (1,174 s).  
**Tamaño de respuesta (Hoppscotch):** `1.82 KB`.  
**Estado:** **Aprobado** — se rechaza la entrada en validación de formato y se conserva el valor incorrecto en `barcode_send`.  
**Notas:** La prueba confirma la detección de letras en un valor de longitud permitida; no comprueba el dígito de control de un código EAN válido.

### Caso de Prueba: Solicitud "Barcode" 12 Caracteres Invalida

**ID:** TC-013  
**Objetivo:** Verificar que un código de barras de 12 caracteres con letras sea rechazado por la validación de formato.  
**Entrada:** `{"barcode":"3017620422a3"}`  
**Salida esperada:** HTTP `400 Bad Request`, código `BARCODE_INVALID` y conservación del dato recibido en `barcode_send`.

**Resultado observado (Hoppscotch):**

```json
{
  "status": "error",
  "http_code": 400,
  "code": "BARCODE_INVALID",
  "barcode_send": "3017620422a3",
  "barcode_status": "Invalid",
  "error": "El codigo de barra no cumple los estandares (EAN-8, EAN-13, UPC-A, UPC-E), deben ser solo numeros y no incluir letras",
  "example": "Envia: {'barcode': '3017620422003'} ",
  "hint": "El codigo de barra debe tener 8, 12 o 13 caracteres unicamente numericos",
  "timestamp": "2026-10-20T00:02:19.565+02:00"
}
```

**HTTP observado:** `400 Bad Request`.  
**Tiempo de respuesta (Hoppscotch):** `1300 ms` (1,300 s).  
**Tamaño de respuesta (Hoppscotch):** `1.83 KB`.  
**Estado:** **Aprobado** — se rechaza un valor con longitud admitida (12 caracteres) que contiene una letra (`a`).  
**Notas:** El mensaje de error explica el formato requerido y reproduce el valor recibido. Esta prueba cubre la validación de caracteres, no el dígito de control UPC-A.

### Caso de Prueba: Solicitud "Barcode" 8 Caracteres Invalida

**ID:** TC-014  
**Objetivo:** Verificar que un código de barras de 8 caracteres que contiene letras se rechaza durante la validación de formato.  
**Entrada:** `{"barcode":"301762a4"}`  
**Salida esperada:** HTTP `400 Bad Request`, código `BARCODE_INVALID`, `barcode_send` igual a la entrada y `barcode_status` igual a `Invalid`.

**Resultado observado (Hoppscotch, nueva ejecución):**

```json
{
  "status": "error",
  "http_code": 400,
  "code": "BARCODE_INVALID",
  "barcode_send": "301762a4",
  "barcode_status": "Invalid",
  "error": "El codigo de barra no cumple los estandares (EAN-8, EAN-13, UPC-A, UPC-E), deben ser solo numeros y no incluir letras",
  "example": "Envia: {'barcode': '3017620422003'} ",
  "hint": "El codigo de barra debe tener 8, 12 o 13 caracteres unicamente numericos",
  "timestamp": "2026-10-20T00:03:11.818+02:00"
}
```

**HTTP observado:** `400 Bad Request`.  
**Tiempo de respuesta (Hoppscotch):** **752 ms** (0,752 s).  
**Tamaño de respuesta:** `1.82 KB`.  
**Estado:** **Aprobado**.  
**Notas:** La longitud (8 caracteres) está permitida, pero la letra `a` provoca el rechazo esperado; no se comprueba el dígito de control.

### Caso de Prueba: Solicitud Error 404 - Producto no encontrado

**ID:** TC-015  
**Objetivo:** Verificar la respuesta cuando Open Food Facts no encuentra el producto.  
**Entrada:** `{"barcode":"3011793003685"}`  
**Salida esperada:** HTTP `404 Not Found`, código `PRODUCT_NO_FOUND`.

**Resultado observado (Hoppscotch, nueva ejecución):**

```json
{
  "status": "error",
  "http_code": 404,
  "barcode_send": "3011793003685",
  "code": "PRODUCT_NO_FOUND",
  "error": "Producto no Encontrado en nuestra base de datos - Vuelva a intentar con otro producto",
  "example": "Envia: {'barcode': '3017620422003'} ",
  "timestamp": "2026-10-20T00:03:54.408+02:00"
}
```

**HTTP observado:** `404 Not Found`.  
**Tiempo de respuesta (Hoppscotch):** **1403 ms** (1,403 s).  
**Tamaño de respuesta:** `1.62 KB`.  
**Estado:** **Aprobado**.  
**Notas:** Se devuelve un error estructurado cuando la consulta no encuentra el código solicitado.

### Caso de Prueba: Error - Datos Insuficientes

**ID:** TC-016  
**Objetivo:** Verificar que se informa cuando un producto existe pero los datos nutricionales son insuficientes o inválidos.  
**Entrada:** `{"barcode":"6111035000058"}`  
**Salida esperada:** **HTTP `200 OK` exigido en «Salida» por el enunciado de 4Geeks**, con cuerpo JSON, `status: "error"` y código `PRODUCT_MISSING_DATA`; no debe emitirse un veredicto nutricional.

**Resultado observado (Hoppscotch, nueva ejecución):**

```json
{
  "status": "error",
  "http_code": 200,
  "code": "PRODUCT_MISSING_DATA",
  "barcode_send": 6111035000058,
  "error": "Los datos en la API Open Foods Facts son insuficientes o invalidos - Intenta con otro producto",
  "example": "Envia: {'barcode': '3017620422003'} ",
  "hint": "Vuelve a probar en scannear otro producto, actualmente nuestra base de dato tiene incompletos o los datos invalidos de este producto.",
  "timestamp": "2026-10-20T00:04:34.106+02:00"
}
```

**HTTP observado:** `200 OK`.  
**Tiempo de respuesta (Hoppscotch):** **1085 ms** (1,085 s).  
**Tamaño de respuesta:** `1.81 KB`.  
**Estado:** **Aprobado — coincide con el requisito explícito del entregable**.  
**Notas:** La especificación diferencia el estado HTTP (`200`, operación procesada) del estado funcional del cuerpo (`status: "error"`, no se puede clasificar). En este ejercicio **no debe sustituirse el 200 por otro código**, porque el requisito solicita expresamente 200 para los datos insuficientes.

## Prueba adicional — valor límite de azúcar

### Caso de Prueba: Azúcar exactamente 22,5 g/100 g (regresión de límite)

**ID:** TC-017  
**Objetivo:** Comprobar que el valor exacto `22,5 g/100 g` se clasifica como `Medio` después de corregir la condición de azúcar en n8n.  
**Entrada:** `{"barcode":"8436606892497"}` — POST al endpoint de pruebas `/webhook-test/nutrition-check`.  
**Salida esperada:** Azúcar `22,5 g/100 g` → `Medio`; con sal `Alto` y grasas `Bajo`, el semáforo corresponde a `Moderado`. Nutri-Score `e` debe producir veredicto final `No Saludable` si se aplica la regla del caso más restrictivo.  
**Resultado observado (captura Hoppscotch):**
- HTTP: **200 OK**.
- Tiempo de respuesta: **3242 ms** (3,242 s).
- Tamaño de respuesta: **2,27 KB**.
- Producto: **Hoisin Sauce**; marca: **The Holy Sauce**.
- Nutri-Score: **e**.
- Azúcar: **22,5 g/100 g — Medio**, confirmado visualmente.
- Sal: **4,8375 g/100 g — Alto**.
- Grasas: **0 g/100 g — Bajo**.
- **Veredicto final y recomendación:** no visibles en la captura; pendientes de contrastar con la parte inferior de la respuesta.

**Estado:** **Aprobado para la condición límite de azúcar; verificación integral parcial** (falta comprobar veredicto final y recomendación).  
**Notas:** Prueba adicional de regresión, diferente de TC-001 a TC-016. No mide rendimiento de carga; `3242 ms` es la latencia observada para esta solicitud individual desde Hoppscotch.

### Caso de Prueba: Azúcar exactamente 5 g/100 g (límite inferior de Medio)

**ID:** TC-018  
**Objetivo:** Verificar que un valor exacto de `5 g/100 g` de azúcar se clasifica como `Medio`, y no como `Bajo`.  
**Entrada:** `{"barcode":"5000328038209"}` — POST al endpoint de prueba `/webhook-test/nutrition-check`.  
**Salida esperada:** Azúcar `5 g/100 g` → `Medio`; sal y grasas `Alto` → semáforo `No Saludable`. Con Nutri-Score `unknown`, el veredicto final debería derivarse del semáforo.  
**Resultado observado (captura Hoppscotch):**
- HTTP: **200 OK**.
- Tiempo de respuesta: **3122 ms** (3,122 s).
- Tamaño de respuesta: **2,36 KB**.
- Producto mostrado: **Saturates Sugars 5g Salt 0.50g ee ORIGINAL CHEESE**; marca: **Cheetos**.
- Nutri-Score: **unknown**.
- Azúcar: **5 g/100 g — Medio**; clasificación del límite confirmada visualmente.
- Sal: **1,64166666666667 g/100 g — Alto**.
- Grasas: **31,3333333333333 g/100 g — Alto**.
- **Veredicto final y recomendación:** no visibles en la captura; pendientes de contrastar.

**Estado:** **Aprobado para la condición límite de azúcar; verificación integral parcial**.  
**Notas:** Registro de una ejecución aislada, no una prueba estadística de rendimiento. Mantener los valores nutricionales originales para las decisiones; redondear, si se desea, solo en la plantilla de salida.

### Caso de Prueba: Sal exactamente 1,5 g/100 g (límite superior de Bajo)

**ID:** TC-019  
**Objetivo:** Verificar que `salt_100g = 1,5` se clasifica como `Bajo`, ya que `Alto` comienza con valores estrictamente superiores a 1,5.  
**Entrada:** `{"barcode":"5054268138673"}` — POST a `/webhook-test/nutrition-check`.  
**Salida esperada:** Sal `1,5 g/100 g` → `Bajo`; azúcar `2,2` y grasas `5,7` → `Bajo`; semáforo `Saludable`. Nutri-Score `d` debería prevalecer con un veredicto final `No Saludable`.  
**Resultado observado (captura Hoppscotch):**
- HTTP: **200 OK**.
- Tiempo de respuesta: **2808 ms** (2,808 s).
- Tamaño de respuesta: **2,15 KB**.
- Producto: **Rice Cakes Salt and Vinegar Flavoured 145G**; marca **Tesco**.
- Nutri-Score: **d**.
- Azúcar: **2,2 g/100 g — Bajo**.
- Sal: **1,5 g/100 g — Bajo** (límite confirmado visualmente).
- Grasas: **5,7 g/100 g — Bajo**.
- **Veredicto final y recomendación:** no visibles en la captura, aún no contrastados.

**Estado:** **Aprobado para la condición límite de sal; verificación integral parcial**.  
**Notas:** La latencia corresponde a una sola solicitud observada desde Hoppscotch; no constituye prueba de carga.

### Caso de Prueba: Grasas exactamente 17,5 g/100 g (límite superior de Bajo)

**ID:** TC-020  
**Objetivo:** Verificar que `fat_100g = 17,5` se clasifica como `Bajo` y que no entra en la rama `Alto`, reservada a valores estrictamente superiores a 17,5.  
**Entrada:** `{"barcode":"8411320332900"}` — POST a `/webhook-test/nutrition-check`.  
**Salida esperada:** Grasas `17,5 g/100 g` → `Bajo`; azúcar `0` y sal `0,9` → `Bajo`; semáforo `Saludable`. Nutri-Score `b` → `Saludable`; veredicto final esperado: `Saludable`.  
**Resultado observado (captura Hoppscotch):**
- HTTP: **200 OK**.
- Tiempo de respuesta: **2812 ms** (2,812 s).
- Tamaño de respuesta: **2,11 KB**.
- Producto: **Tonno all’Olio di Oliva**; marca: **Ortiz**.
- Nutri-Score: **b**.
- Azúcar: **0 g/100 g — Bajo**.
- Sal: **0,9 g/100 g — Bajo**.
- Grasas: **17,5 g/100 g — Bajo**; condición límite confirmada.
- Formato de salida: **no aparecen decimales innecesarios** en los valores visibles (`0`, `0.9`, `17.5`).
- **Veredicto final y recomendación:** no visibles en la captura; pendientes de contrastar.

**Estado:** **Aprobado para la condición límite de grasas y la presentación observada; verificación integral parcial**.  
**Notas:** Ejecución observada en Hoppscotch. La modificación del redondeo se confirmó visualmente para los valores mostrados; no demuestra por sí sola el comportamiento con una fracción larga, que puede comprobarse al repetir TC-018.

## Pruebas controladas de errores de servicios externos — Open Food Facts y Groq

> **Alcance y evidencia:** esta sección describe cómo se prepararon o pueden reproducirse las simulaciones documentadas y qué debe verificarse. No constituye, por sí sola, evidencia de que cada ejecución haya terminado correctamente. Se distingue entre **código HTTP del servicio simulado**, **rama ejecutada en n8n** y **respuesta HTTP que recibe Hoppscotch**. Las pruebas se basan en la configuración del archivo `SnackCheck.json` revisado el 20/10/2026.

### A. Simulación de errores HTTP 400, 401, 429 y 500 — Open Food Facts

**Objetivo.** Verificar que, cuando la llamada del nodo `Solicitud API - Open Food Facts` obtiene un código HTTP de error, el workflow selecciona la rama correspondiente y devuelve un mensaje JSON controlado, sin continuar hasta la clasificación nutricional o la generación de IA.

**Por qué se usa una simulación.** Es difícil provocar de forma reproducible errores 400, 401, 429 y 500 en la API pública con un código de barras corriente. Para verificar **el manejo interno de esos estados**, las notas de prueba emplean `https://httpbin.org/status/<CODIGO>` como sustituto temporal de la respuesta del servicio. Esto **no** prueba que Open Food Facts esté fallando ni reproduce fielmente sus cuerpos de error: prueba el enrutamiento que realiza SnackCheck ante un determinado HTTP status.

#### Preparación y ejecución

1. En n8n, abre una **copia de prueba** de SnackCheck o utiliza un entorno de ejecución cuyo cambio temporal no se publique. Conserva la URL original del nodo `Solicitud API - Open Food Facts` para restaurarla después. **Atención:** usar una URL `/webhook-test/` no garantiza por sí solo que las modificaciones guardadas en un flujo activo estén aisladas de producción.
2. Localiza el nodo `Solicitud API - Open Food Facts`. La URL original es:

   ```text
   https://world.openfoodfacts.org/api/v2/product/{{ $json.barcode }}.json?fields=product_name,brands,nutriscore_grade,nutriments
   ```

3. Mantén en el nodo HTTP Request las opciones que ya aparecen en el JSON exportado: respuesta completa (`fullResponse = true`) y no interrumpir el flujo por respuestas HTTP de error (`neverError = true`; `onError = continueRegularOutput`). Esto permite que el `Switch` inspeccione `statusCode`.
4. Sustituye **temporalmente** la URL de ese nodo por `https://httpbin.org/status/400`. Después repite la misma prueba con `/401`, `/429` y `/500`, de una en una.
5. Antes de cada solicitud, activa **Listen for test event** en el Webhook del editor n8n. Desde Hoppscotch envía un POST a:

   ```text
   https://ericdevs2.app.n8n.cloud/webhook-test/nutrition-check
   ```

   Con `Content-Type: application/json` y este cuerpo (barcode válido para pasar las validaciones de entrada):

   ```json
   {"barcode": "3017620422003"}
   ```

   Como alternativa, desde una terminal:

   ```bash
   curl -i -X POST 'https://ericdevs2.app.n8n.cloud/webhook-test/nutrition-check' \
     -H 'Content-Type: application/json' \
     -d '{"barcode":"3017620422003"}'
   ```

6. Verifica **dos lugares** en cada ejecución: (a) en n8n, la salida del nodo `Solicitud API - Open Food Facts` y qué salida toma `Switch - Status Code API Open Food`; (b) en Hoppscotch, el código HTTP recibido y el contenido de la respuesta. Anota el tiempo que muestra Hoppscotch si está disponible.
7. Cuando termines, **restaura la URL original**, verifica que está guardada en el flujo que corresponda y no publiques la configuración temporal. En la copia de prueba, puede descartarse la modificación al finalizar.

#### Matriz de casos de error simulados

| ID de simulación | URL temporal del HTTP Request | Salida del `Switch` esperada | Nodo de respuesta configurado | Respuesta configurada en SnackCheck |
|---|---|---|---|---|
| SIM-OFF-400 | `https://httpbin.org/status/400` | `400` | `Responder Error 400 - Bad Request API Open Food` | HTTP **400**, JSON `status: "error"`, `http_code: 400`, mensaje sobre solicitud incorrecta |
| SIM-OFF-401 | `https://httpbin.org/status/401` | `401` | `Responder Error 401 - Unauthorized API Open Food` | HTTP **401**, JSON `status: "error"`, `http_code: 401`, mensaje de acceso no autorizado |
| SIM-OFF-429 | `https://httpbin.org/status/429` | `429` | `Responder Error 429 - Too Many Request API Open Food` | HTTP **429**, JSON `status: "error"`, `http_code: 429`, `code: "RATE_LIMITED_OPEN_FOOD_API"` |
| SIM-OFF-500 | `https://httpbin.org/status/500` | `500 / otros` (fallback) | `Responder Error 500 - Internal Server Error API Open Food` | HTTP **500**, JSON `status: "error"`, `http_code: 500`, mensaje de error del servidor |

**Nota sobre la última fila:** la rama `500 / otros` es la salida de reserva del `Switch`; no se limita a un upstream HTTP 500. Un error inesperado o un cuerpo distinto del esperado también podría alcanzar esa salida. El workflow devuelve entonces el HTTP 500 genérico que configuraste. Esto comprueba el *fallback*, no una categorización exacta de todos los códigos posibles.

**Criterios de aprobación para cada caso:**

- El barcode supera las validaciones de entrada; no se confunde un HTTP 400 del servicio externo con `BARCODE_MISSING` o `BARCODE_INVALID`.
- El nodo HTTP Request expone el `statusCode` simulado y el `Switch` elige la salida indicada en la tabla.
- Se ejecuta el nodo `Responder Error ...` previsto, con el código y los campos JSON indicados.
- La clasificación nutricional y Groq no se ejecutan después de esa rama de error.
- Si el cliente recibe una página de error genérica en un caso 5xx, se contrasta en n8n el nodo ejecutado y se documenta **lo que vio realmente el cliente**; no se afirma que Hoppscotch recibió el JSON si no lo recibió.

**Estado de evidencia:** los cuatro procedimientos estaban descritos en el documento original, pero no se adjuntaron aquí registros individuales de ejecución (captura/log, respuesta efectiva y latencia) que permitan marcar `SIM-OFF-400/401/429/500` como *verificados*. Registrar cada uno como **procedimiento reproducible; resultado observado pendiente de adjuntar**, si no se conserva esa evidencia. No confundirlos con TC-010–TC-014, que sí prueban errores de **entrada**, ni con TC-015, que sí prueba producto no encontrado.

### B. Simulación del fallo en la generación de texto — Groq (respuesta configurada 504)

**Objetivo.** Verificar que un error durante la generación de la recomendación no provoca que SnackCheck devuelva una recomendación inexistente o continúe como si la IA hubiera respondido; el workflow debe intentar pasar a `Responder Error 504 - Error API Groq`.

**Funcionamiento actual verificado en el JSON:**

```text
Generacion Texto Amigable (onError: continueRegularOutput)
  → Verifico - Error API IA
    → si el resultado contiene $json.error: Responder Error 504 - Error API Groq
    → si no contiene error: Config Color Nutrientes → Formato Plano → respuesta normal
```

El nodo `Responder Error 504 - Error API Groq` está configurado para emitir HTTP **504** y un JSON con `status: "error"`, `http_code: 504`, explicación de error de conexión con Groq y `timestamp`.

#### Método documentado de provocación del fallo

1. Trabaja en una **copia de prueba** o configuración aislada; conserva el modelo y credenciales originales para restaurarlos. El modelo identificado en el JSON revisado es `openai/gpt-oss-20b` dentro del nodo `Groq Chat Model`.
2. En `Groq Chat Model`, reemplaza **temporalmente** el identificador del modelo por uno inexistente, por ejemplo `test-modelo-no-existe`. En las notas originales también se menciona una segunda variante: usar credenciales de Groq incorrectas. No es necesario realizar ambas; la alternativa de modelo inválido evita manipular credenciales. Si se realiza la alternativa de credenciales, no publiques ni compartas tokens o secretos.
3. Activa `Listen for test event` y envía al endpoint de prueba el POST con un producto válido y datos nutricionales completos, por ejemplo `{"barcode":"3017620422003"}`, para que el flujo llegue hasta el paso de IA.
4. Revisa en n8n la salida de `Generacion Texto Amigable`, el resultado de `Verifico - Error API IA` y si se ejecutó `Responder Error 504 - Error API Groq`.
5. En Hoppscotch registra el **HTTP real recibido**, el cuerpo real (si es visible) y la latencia. Comprueba que no aparece una recomendación como si la IA hubiera completado correctamente la generación.
6. Restaura el identificador del modelo y las credenciales originales. Ejecuta una prueba normal para comprobar que volvió a generarse una recomendación. No mantengas publicado el fallo simulado.

#### Resultado y límite de interpretación

- **Salida configurada en el workflow:** HTTP `504`, con un cuerpo JSON como el siguiente (timestamp variable):

  ```json
  {
    "status": "error",
    "http_code": 504,
    "error": "Error al conectar con la API de IA (Groq). Vuelve a intentarlo mas tarde.",
    "timestamp": "<fecha y hora de ejecución>"
  }
  ```

- **Observación descrita originalmente por el autor:** al forzar un fallo mediante un modelo inexistente y, en otra ocasión, mediante credenciales incorrectas, Hoppscotch mostró el texto `error code: 504`, sin el JSON personalizado completo. Las notas lo atribuyen a la capa de Cloudflare. Se conserva esta **observación del entorno de prueba**, no como regla universal de Cloudflare o n8n.
- **Precisión técnica:** un modelo inexistente o una credencial inválida puede provocar errores de origen distintos de HTTP 504. En SnackCheck se **normalizan** los errores detectados por la rama de IA a una respuesta configurada como 504. Por tanto, esta prueba no acredita que **Groq haya respondido HTTP 504** ni que se haya simulado un timeout auténtico de gateway: acredita el manejo del fallo en el paso de IA y la respuesta visible al cliente.
- **Qué evidencia falta para cierre completo:** una captura o registro de ejecución de n8n donde se vea efectivamente la rama `Responder Error 504 - Error API Groq`, junto con la respuesta observada en el cliente. La observación `error code: 504` ya está descrita, pero por sí sola no prueba qué nodo la originó.

**Estado de evidencia:** **fallo forzado y HTTP 504 observado según las notas del autor; ruta interna y cuerpo JSON no verificados con evidencia adjunta**. No confundir una salida **configurada** con una salida **observada**.

### C. Cierre de las simulaciones y control de cambios

Al cerrar las pruebas, registrar en el mismo documento, por cada simulación, la fecha, la URL o el modelo de prueba, el código devuelto por el servicio simulado, la rama ejecutada en n8n, el HTTP visto en Hoppscotch, el tiempo de respuesta y cualquier limitación (por ejemplo, sustitución del JSON por una página genérica). La restauración de la URL de Open Food Facts y del modelo/credenciales Groq debe quedar confirmada antes del export JSON definitivo.

No es necesario volver a ejecutar las simulaciones si se dispone de evidencia suficiente de las ejecuciones anteriores. Tampoco es necesario provocar un auténtico `429` de Open Food Facts ni bloquear llamadas reales al servicio: `httpbin` cumple el objetivo limitado de comprobar el enrutamiento de ese código.


## Resumen de resultados observados

| Caso | Qué verifica | HTTP observado | Tiempo Hoppscotch | Resultado documentado |
|---|---|---|---:|---|
| TC-001 | Autodocumentación sin cuerpo | 200 | **1280 ms** | Aprobado |
| TC-002 | EAN-13, dos nutrientes altos y Nutri-Score e | 200 | **2928 ms** | Aprobado |
| TC-003 | 1 nutriente alto, Nutri-Score d | 200 | **2671 ms** | Aprobado |
| TC-004 | 2 nutrientes altos, Nutri-Score e | 200 | **3260 ms** | Aprobado |
| TC-005 | Nutri-Score c prevalece | 200 | **2685 ms** | Aprobado |
| TC-006 | Veredicto Saludable | 200 | **2167 ms** | Aprobado |
| TC-007 | Semáforo Moderado prevalece | 200 | **2871 ms** | Aprobado |
| TC-008 | UPC-A con cero inicial | 200 | **3275 ms** | Aprobado |
| TC-009 | EAN-8 y Nutri-Score e | 200 | **2963 ms** | Aprobado |
| TC-010 | Campo barcode ausente | 400 | **1251 ms** | Aprobado |
| TC-011 | Campo barcode vacío | 400 | **1254 ms** | Aprobado |
| TC-012 | EAN-13 con caracteres | 400 | **1174 ms** | Aprobado |
| TC-013 | UPC-A 12 caracteres con letra | 400 | **1300 ms** | Aprobado |
| TC-014 | EAN-8 con letra | 400 | **752 ms** | Aprobado |
| TC-015 | Producto no encontrado | 404 | **1403 ms** | Aprobado |
| TC-016 | Datos nutricionales insuficientes | 200 | **1085 ms** | Aprobado según 4Geeks |
| TC-017 | Azúcar = 22,5 g → Medio | 200 | **3242 ms** | Límite aprobado; final no visible |
| TC-018 | Azúcar = 5 g → Medio | 200 | **3122 ms** | Límite aprobado; final no visible |
| TC-019 | Sal = 1,5 g → Bajo | 200 | **2808 ms** | Límite aprobado; final no visible |
| TC-020 | Grasas = 17,5 g → Bajo | 200 | **2812 ms** | Límite aprobado; final no visible |

**Interpretación:** `Aprobado` indica correspondencia entre el resultado observado y el escenario descrito. En TC-017 a TC-020, la aprobación se limita a la **clasificación del valor límite**, porque el veredicto y la recomendación completos no aparecen en las capturas. Las pruebas de errores externos (`SIM-OFF-*` y `SIM-GROQ-504`) se registran aparte: los procedimientos por sí solos no demuestran su ejecución.
