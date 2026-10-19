# CHANGELOG — SnackCheck

Registro de cambios relevantes por versión. Los cambios internos de desarrollo previos al primer checkpoint se agrupan en la versión inicial; este registro no sustituye los commits de Git ni el registro de pruebas.

## [Unreleased]

Sin cambios registrados después del checkpoint inicial.

## [0.1.0] — 2026-10-20 · Primera entrega académica

**Alcance:** baseline de documentación y artefactos del proyecto de pre-work No-Code & Automation de 4Geeks. Esta etiqueta es una **versión documental de referencia**, no una afirmación de despliegue de producción ni de creación de un tag Git.

### Added — Primera versión funcional

- Webhook POST `/nutrition-check`, autodocumentación JSON y validación del código de barras (longitud y caracteres numéricos).
- Consulta de Open Food Facts, comprobación de datos nutricionales y manejo de producto no encontrado.
- Semáforo de azúcar, sal y grasas; uso opcional de Nutri-Score; selección del veredicto más restrictivo.
- Recomendación con Groq: tono cauteloso para `No Saludable` y alentador para `Saludable` o `Moderado`.
- Contrato de salida del ejercicio: HTTP 200 para veredicto en texto plano, autodocumentación JSON y datos insuficientes JSON; HTTP 400/404 en errores de validación/producto inexistente.
- README, explicación detallada del flujo, diagrama editable y registro de TC-001 a TC-020.

### Fixed — Ajustes incorporados antes de la entrega

- Límite de azúcar: `22.5 g/100 g` se clasifica como `Medio` (`<= 22.5`).
- Redondeo de los nutrientes y las calorías en el texto de salida, a un máximo de dos decimales, sin alterar el cálculo de clasificación.
- Alineación de objetivos y entradas del registro de pruebas con las ejecuciones realmente documentadas (por ejemplo, TC-002 con Testglas).

### Known limitations — Limitaciones conocidas

- No hay validación del dígito de control EAN/UPC, solo longitud y contenido numérico.
- Las recomendaciones de IA pueden variar y ocasionalmente resultar demasiado categóricas; no sustituyen asesoramiento nutricional individual.
- Los procedimientos para simular algunos fallos externos están documentados, pero no todas las ramas tienen evidencia independiente completa.
- No se han demostrado requisitos profesionales de disponibilidad, autenticación, monitorización ni pruebas de carga.
