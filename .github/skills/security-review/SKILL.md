---
name: security-review
description: "Use when reviewing application or repository security, auditing code for vulnerabilities, evaluating risk severity, or preparing a false-positive-aware security checklist."
---

# Revisión de seguridad

Realiza análisis de seguridad basado en evidencia y entrega un checklist de riesgos priorizado. Este skill es estrictamente de solo lectura: solo analiza y propone; no modifica código, configuración, documentación ni otros archivos, y no crea issues.

## Procedimiento

1. Delimita el alcance solicitado e identifica las tecnologías, puntos de entrada, límites de confianza y archivos relevantes. Si el alcance es ambiguo y afecta materialmente la revisión, pide aclaración antes de continuar.
2. Inspecciona el código y la configuración relevantes para los riesgos del alcance, incluidos autenticación, autorización, validación, inyección, secretos y datos sensibles, criptografía, dependencias, configuración y manejo de errores.
3. Sigue el flujo de datos desde su origen hasta los usos sensibles. Comprueba las validaciones, controles y mitigaciones que podrían cambiar la explotabilidad antes de informar un riesgo.
4. Registra cada hallazgo con evidencia precisa (ruta, líneas o símbolo), precondiciones, escenario de abuso e impacto. No conviertas ausencia de evidencia en una afirmación de vulnerabilidad.
5. Asigna una de estas severidades, razonando según explotabilidad, privilegios requeridos, alcance e impacto:
   - **Crítico**: explotación directa y de impacto extremo, como compromiso amplio de sistemas o datos.
   - **Alto**: explotación práctica con impacto grave en confidencialidad, integridad o disponibilidad.
   - **Mediano**: impacto significativo, pero limitado por el alcance o las condiciones necesarias.
   - **Low**: impacto bajo o explotación difícil, con alcance reducido o mitigaciones eficaces.
   - **Informativo**: recomendación preventiva o incumplimiento sin vulnerabilidad explotable demostrada.
6. Indica una corrección concreta y una verificación objetiva, como una prueba de regresión, una comprobación de autorización o un criterio de configuración. Distingue entre pruebas propuestas y pruebas realmente ejecutadas.
7. Valora **¿Falso positivo?** como **Sí** si falta evidencia o una mitigación/contexto podría invalidar el hallazgo, y explica qué comprobar. Usa **No** solo cuando el problema se demuestre con la evidencia disponible. No descartes un riesgo únicamente porque no se haya reproducido dinámicamente.
8. Ordena los hallazgos por severidad y evita duplicados que compartan causa raíz. No muestres valores de secretos: describe únicamente su ubicación y tipo.

## Formato de salida

Entrega una fila por riesgo:

| Checklist | Severidad | Hallazgo y evidencia | Impacto / escenario | Corrección verificable | ¿Falso positivo? |
|---|---|---|---|---|---|
| [ ] | Crítico / Alto / Mediano / Low / Informativo | Archivo y líneas o símbolo; evidencia | Consecuencia y precondiciones | Cambio recomendado y criterio de verificación | Sí/No; motivo o comprobación pendiente |

Al terminar, resume el alcance y las limitaciones. Para cada hallazgo crítico o alto confirmado, prepara la propuesta del issue y marca su estado como **pendiente de aprobación humana**. Solicita confirmación explícita para ese issue antes de crearlo; la solicitud de revisión no equivale a aprobación. No implementes correcciones ni edites archivos. Si no encuentras hallazgos, dilo expresamente.
