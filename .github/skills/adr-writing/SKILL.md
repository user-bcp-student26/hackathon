---
name: adr-writing
description: Guía para redactar y revisar Architecture Decision Records (ADR) en este repositorio, alineados con historias de usuario y épicas. Úsalo al crear o mejorar un ADR.
---

# Redacción de ADR

## Estructura obligatoria

1. **Encabezado:** título con número (`ADR-NNN: ...`), estado (`Propuesta`, `Aceptada`, `Rechazada`, `Reemplazada`), fecha, enlace a la historia de usuario y a la épica.
2. **Contexto:** problema y restricciones, con referencia a los criterios de aceptación de la historia. No describir la solución aquí.
3. **Decisión:** una sección por tema. Cada decisión debe ser concreta y verificable (nombre del algoritmo, formato, tabla, endpoint).
4. **Alternativas consideradas:** tabla con alternativa y motivo de descarte. Incluir al menos una alternativa por decisión importante.
5. **Consecuencias:** separar positivas y negativas. Las negativas deben incluir riesgos de seguridad, operación y dependencias.
6. **Pendientes:** decisiones o validaciones que faltan antes de implementar.

## Reglas de calidad

- Cada criterio de aceptación de la historia debe quedar cubierto por una decisión o marcado explícitamente como desviación.
- Si la decisión contradice un criterio, documentarlo como **desviación**, con su riesgo y quién debe aprobarla.
- No inventar requisitos ni cifras. Lo no confirmado va en **Pendientes**.
- Las decisiones de seguridad (hash, sesión, transporte, límites de intentos) deben mencionar la mitigación mínima.
- Escribir en el idioma de la historia y la épica (español en este repositorio).
- Enlaces relativos desde `docs/adr/` a `packages/`: usar `../../packages/...`.

## Ubicación y nombre

- Archivos en `docs/adr/`, nombre `ADR-NNN-tema-corto.md`, numeración secuencial.
- Un ADR por historia o decisión; no mezclar temas no relacionados.
