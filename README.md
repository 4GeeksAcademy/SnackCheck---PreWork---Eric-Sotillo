# SnackCheck — Evaluación nutricional de productos con n8n

> Automatización del pre-work **No-Code & Automation** de 4Geeks: recibe un código de barras, consulta datos nutricionales de Open Food Facts, aplica reglas de clasificación y devuelve una recomendación redactada con IA.

## 1. Resumen del proyecto

**Nombre:** SnackCheck.  
**Lema:** Consulta un producto por su código de barras y recibe un veredicto nutricional fácil de interpretar.  
**Problema que resuelve:** Los valores nutricionales de un envase pueden resultar difíciles de comparar. SnackCheck reúne información de una base abierta, aplica reglas explícitas y presenta un resultado resumido.  
**Usuarios objetivo:** Personas que desean una orientación sencilla sobre productos envasados y aplicaciones que necesiten incorporar esta evaluación mediante un webhook.

**Alcance:** Este proyecto implementa el servicio de automatización/API; **no** incluye una aplicación móvil ni lectura de códigos de barras con cámara.

## 2. Cómo usar

### Requisitos y configuración

- Una instancia de **n8n** compatible con los nodos incluidos en `SnackCheck.json`.
- Acceso a **Open Food Facts** (API pública, sin clave para esta consulta).
- Credenciales válidas de **Groq** configuradas en n8n para el nodo `Groq Chat Model`. El export de n8n referencia la credencial, pero no debe distribuirse con secretos.
- Importar `SnackCheck.json` en n8n, configurar la credencial de Groq, revisar los nodos y activar el workflow.
- Para probar cambios sin alterar el webhook publicado, utilizar el webhook de pruebas de n8n mientras el editor esté escuchando eventos de test.

**Método:** `POST`  
**Endpoint documentado en esta entrega:** `https://ericdevs2.app.n8n.cloud/webhook/nutrition-check`  
**Content-Type de entrada:** `application/json`

> El endpoint puede dejar de estar disponible si la instancia o la publicación cambian. No lo utilices como servicio garantizado de producción. Si compartes el repositorio públicamente, considera que cada solicitud válida puede generar tráfico y consumo de Groq.

### Formato de entrada

La clave obligatoria es `barcode`. Se recomienda enviarla como **cadena de texto** para conservar ceros iniciales. El flujo acepta cadenas de **8, 12 o 13 dígitos**, sin caracteres adicionales.

```json
{
  "barcode": "3017620422003"
}
```

Ejemplo con `curl`:

```bash
curl -i -X POST 'https://ericdevs2.app.n8n.cloud/webhook/nutrition-check' \
  -H 'Content-Type: application/json' \
  -d '{"barcode":"3017620422003"}'
```

Si se envía un `POST` **sin cuerpo**, SnackCheck devuelve autodocumentación de uso (HTTP 200). En cambio, un cuerpo con `barcode` ausente o vacío devuelve un error de validación (HTTP 400).

### Formato de salida

### Contrato HTTP y formatos de respuesta (según 4Geeks)

| Escenario | Estado HTTP | Cabecera `Content-Type` |
|---|---|---|
| Veredicto generado | `200 OK` | `text/plain` |
| Autodocumentación (POST sin cuerpo) | `200 OK` | `application/json` |
| Información nutricional insuficiente | `200 OK` | `application/json` |
| Entrada inválida (`barcode` ausente, vacío o no numérico) | `400 Bad Request` | `application/json` |
| Producto no encontrado | `404 Not Found` | `application/json` |
| Errores de servicios externos | Según el fallo simulado/configurado (400, 401, 429, 500 o 504) | `application/json` configurado; la capa de infraestructura puede sustituir el cuerpo 5xx |

`200`, `400` y `404` son **códigos de estado HTTP**; `Content-Type` es una cabecera diferente que identifica el formato del cuerpo. El HTTP 200 con JSON `status: "error"` de `PRODUCT_MISSING_DATA` reproduce la especificación del ejercicio y fue comprobado en TC-016.


**En caso de éxito**, el flujo responde **HTTP 200, `text/plain`**, con datos del producto, semáforos para los nutrientes, calorías, veredicto y recomendación. La estructura observada en la plantilla n8n es:

```text
🦉 ASESOR NUTRICIONAL
-----------------------

Nombre Producto: <nombre devuelto por Open Food Facts>
Marca: <marca>
Nutri Score: <letra o Desconocida>

Nutrientes: (Valores cada 100g)
-------------------------------
<🟢/🟡/🔴> Azucar: <valor> gr/100gr (<Bajo/Medio/Alto>)
<🟢/🔴> Sal: <valor> gr/100gr (<Bajo/Alto>)
<🟢/🔴> Grasas: <valor> gr/100gr (<Bajo/Alto>)

Calorias: <valor> Kcal/100gr

🚨 Veredicto: <Saludable/Moderado/No Saludable>

💡 Recomendación:
<texto variable generado por el modelo de IA>
```

Los campos entre `<...>` son **marcadores explicativos**, no datos de una ejecución medida. La recomendación no es determinista y puede variar entre llamadas.

**En caso de error**, los nodos de respuesta devuelven `application/json` y un mensaje que explica el problema. Ejemplo basado en una respuesta documentada para un código de barras inválido:

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
  "timestamp": "2026-10-19T19:16:07.302+02:00"
}
```

El campo `timestamp` depende de la hora de cada ejecución.

## 3. Funcionalidades

- Autodocumentación cuando el `POST` llega sin cuerpo.
- Validación de presencia, longitud y caracteres numéricos del código de barras.
- Consulta de producto y nutrientes mediante **Open Food Facts**.
- Comprobación de que existen valores nutricionales utilizables antes de clasificar.
- Clasificación mediante semáforos de azúcar, sal y grasas por 100 g.
- Cálculo de `preocupacion`, número de nutrientes etiquetados como altos (0 a 3).
- Clasificación independiente mediante Nutri-Score, cuando está disponible.
- Combinación de ambas clasificaciones mediante el **peor veredicto**.
- Recomendación de lenguaje natural con tono alentador o cauteloso según el veredicto.
- Respuestas explícitas ante entradas inválidas, producto no encontrado, datos insuficientes o errores de servicios externos.

## 4. Detalles técnicos y flujo lógico

**Plataforma:** n8n; último workflow adjunto para revisión con **60 nodos**.  
**API de datos:** `GET https://world.openfoodfacts.org/api/v2/product/{barcode}.json?fields=product_name,brands,nutriscore_grade,nutriments`.  
**IA:** proveedor Groq mediante el nodo `Groq Chat Model`; el JSON exportado selecciona `openai/gpt-oss-20b`.  
**Lógica auxiliar:** un nodo Code en Python cuenta los nutrientes altos y establece el resultado del semáforo.  
**Entradas/salidas:** webhook n8n `POST`; veredicto en texto plano; autodocumentación, datos insuficientes y errores en JSON.

```text
POST Webhook
  ├─ Sin cuerpo → Autodocumentación
  └─ Con cuerpo → Validación del barcode
       ├─ Inválido → Error 400
       └─ Válido → Open Food Facts
            ├─ API / producto no disponible → Error específico
            └─ Producto encontrado → Validación de nutrientes
                 ├─ Incompletos → PRODUCT_MISSING_DATA
                 └─ Completos → Semáforo + Nutri-Score
                      → Peor veredicto → Prompt según tono
                      → Groq → Formato final → Respuesta de texto
```

### Reglas de decisión

| Dato (cada 100 g) | Clasificación del ejercicio |
|---|---|
| Azúcar: menos de 5 g | Bajo |
| Azúcar: desde 5 hasta 22,5 g inclusive | Medio |
| Azúcar: más de 22,5 g | Alto |
| Sal: más de 1,5 g | Alto; en caso contrario, Bajo |
| Grasas: más de 17,5 g | Alto; en caso contrario, Bajo |

**Conteo de semáforo:** 0 altos → `Saludable`; 1 alto → `Moderado`; 2–3 altos → `No Saludable`.

**Nutri-Score:** `a`/`b` → `Saludable`; `c` → `Moderado`; `d`/`e` → `No Saludable`. Si es `unknown` o falta, se utiliza solo el semáforo.

**Resultado final:** se selecciona el peor de los dos veredictos. El modelo de IA **redacta la explicación**, pero **no debe decidir** la clasificación.

> **Corrección verificada:** el nodo `Sugar_100g >= 5 & <= 22.5` del export revisado utiliza `lte` (menor o igual a 22,5), y TC-017 confirmó en Hoppscotch que exactamente **22,5 g** se clasifica como `Medio`. También se verificaron límites de azúcar (5 g), sal (1,5 g) y grasas (17,5 g) en TC-018 a TC-020.

## 5. Manejo de errores

| Situación | Respuesta configurada | Acción sugerida |
|---|---|---|
| `barcode` ausente o longitud incorrecta | HTTP 400, `BARCODE_MISSING` | Enviar 8, 12 o 13 dígitos. |
| Caracteres no numéricos | HTTP 400, `BARCODE_INVALID` | Utilizar solo dígitos. |
| Producto no encontrado | HTTP 404, `PRODUCT_NO_FOUND` | Verificar el código o probar otro producto. |
| Producto con nutrientes insuficientes | HTTP 200, JSON con `status: "error"` y `code: "PRODUCT_MISSING_DATA"` | Probar otro producto; **HTTP 200 es exigido explícitamente en el apartado «Salida» de 4Geeks.** |
| Error de Open Food Facts | HTTP 400/401/429/500 según la rama | Reintentar cuando corresponda; 429 indica límite de solicitudes. |
| Fallo del paso de IA (Groq) | HTTP 504 configurado, JSON de error | Reintentar más tarde; durante las pruebas se observó `error code: 504`, sin el cuerpo JSON personalizado. La causa de sustitución del cuerpo no quedó confirmada. |

**Alcance de evidencia:** el registro `Validation-test.md` contiene casos marcados como aprobados y procedimientos para simular otros fallos. No todos incluyen salida real o latencias, por lo que esta documentación no certifica un SLA ni una disponibilidad de producción.

## 6. Pruebas y rendimiento

Consultar [`Validation-test.md`](Validation-test.md). Incluye **TC-001 a TC-016**, cuatro pruebas adicionales de límites **TC-017 a TC-020**, los tiempos observados en Hoppscotch y los procedimientos de simulación de errores externos. Cada latencia corresponde a **una solicitud medida**, no a un ensayo estadístico de carga. TC-002 está **aprobado** con el producto Testglas (`3017620422003`), dos nutrientes altos y Nutri-Score `e`, con veredicto final `No Saludable` y recomendación cautelosa. Las cuatro pruebas de límites están aprobadas para la clasificación del valor límite, pero no cuentan con evidencia del párrafo final. Los fallos simulados se distinguen de pruebas con salida observada.

## 7. Limitaciones y mejoras futuras

- Es un **prototipo educativo** de evaluación según reglas del ejercicio; no es un diagnóstico nutricional ni un consejo médico individualizado.
- La precisión y cobertura dependen de la base de datos comunitaria Open Food Facts.
- El validador verifica longitud y dígitos, pero **no** el dígito de control/checksum del código ni demuestra todos los formatos comerciales; se debe reforzar si pasa a producción.
- La respuesta exitosa es texto plano, no JSON estructurado; puede requerir adaptación para consumo desde una app.
- Se revisaron ejemplos de recomendaciones para los tres veredictos, pero no se ha realizado una evaluación sistemática de fiabilidad, costes de Groq ni carga/concurrencia; los tiempos individuales constan en `Validation-test.md`.
- No se ha acreditado control de acceso ni limitación de uso del webhook público.
- La presentación de semáforos utiliza símbolos y texto; su accesibilidad y comprensión deben validarse con usuarios.
- Mejorar el control de precisión en las recomendaciones de IA (evitar afirmaciones demasiado categóricas o porciones individualizadas) y revisar periódicamente que los datos y la presentación de los nutrientes resulten comprensibles.
- Añadir reintentos controlados, timeouts, observabilidad y tests automatizados cuando el proyecto requiera mayor robustez.

## 8. Entregables y versiones

- `SnackCheck.json`: exportación de la automatización.
- `README.md`: descripción, uso, arquitectura y limitaciones.
- `Validation-test.md`: registro de verificación y rendimiento.
- `CHANGELOG.md`: historia documentada de cambios.
- `Diagrama de Flujo.excalidraw`: diagrama editable aportado para el entregable. Acompañar a los otros archivos en la entrega.

**Checkpoint documental:** `0.1.0`, primera versión documentada, sin implicar un despliegue de producción ni historial de releases previos. Consultar [`CHANGELOG.md`](CHANGELOG.md).


## 9. Verificación del export final y control de cambios

Se inspeccionó el archivo `SnackCheck.json` **adjuntado nuevamente al cierre**. En este export se comprobaron:

- El umbral de azúcar `>= 5` y `<= 22.5` en el nodo `Sugar_100g >= 5 & <= 22.5`.
- El nodo `Formato Plano - Plantilla Salida` con expresiones `toFixed(2)` para **azúcar, sal, grasas y calorías**, aplicadas solo al texto de presentación, no al cálculo previo del veredicto.
- La URL normal de Open Food Facts (`world.openfoodfacts.org/api/v2/product/...`), y el modelo configurado en Groq `openai/gpt-oss-20b` (no el identificador de simulación inválido).

Por tanto, **el export recibido refleja las últimas correcciones de umbrales y formato**. Esto comprueba la configuración del JSON, no sustituye pruebas de funcionamiento de un import nuevo en otro entorno. El archivo de Excalidraw se incluye para explicar las ramas principales; su texto de condición de azúcar fue normalizado en la copia de entrega para utilizar el operador estándar `<=`.
