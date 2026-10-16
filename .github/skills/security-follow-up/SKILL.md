---
name: security-follow-up
description: "Use only after explicit human approval to create a proposed issue for a confirmed critical or high security vulnerability. Never use to edit or implement code changes."
---

# Creación aprobada de issues de seguridad

Este skill solo crea issues de seguridad después de la aprobación humana. Invocarlo no constituye aprobación. No modifica código, configuración, documentación ni otros archivos, ni implementa correcciones: se limita a crear el issue autorizado.

## Requisitos de aprobación

- La aprobación debe ser humana, posterior a la presentación del hallazgo y la propuesta de issue, y especificar inequívocamente que se autoriza crear ese issue concreto.
- La aprobación general para revisar, para aplicar una política o para corregir riesgos no autoriza crear issues.
- Si no está claro qué hallazgo o acción está aprobado, detente y pregunta. No interpretes silencio como aprobación.
- La aprobación para corregir no autoriza crear un issue.
- No amplíes la acción aprobada a otros hallazgos ni a cambios colaterales.
- Aunque el usuario solicite una modificación de código, no la realices: informa que este agente solo puede proponerla.

## Crear un issue

1. Antes de crear nada, verifica que el usuario aprobó explícitamente la creación del issue para este hallazgo después de revisar su propuesta. Si no hay aprobación clara, no uses herramientas de creación; deja el issue pendiente y solicítala.
2. Crea issues únicamente para vulnerabilidades **confirmadas** de severidad **Crítico** o **Alto** y solo con la aprobación humana descrita arriba.
3. Usa el repositorio actual y el tipo **Seguridad**. Si no se puede asignar ese tipo, no lo sustituyas silenciosamente por otro; informa la limitación y presenta el título y cuerpo propuestos.
4. Incluye título conciso, severidad, impacto, evidencia con rutas y líneas, condiciones o pasos de reproducción no destructivos, corrección sugerida y criterios de verificación. No incluyas secretos ni datos sensibles.
5. Informa el enlace o identificador del issue si se creó. No afirmes que se creó cuando la operación falló o no había herramienta disponible.

## Propuesta de corrección

Formula recomendaciones y criterios de verificación para que el usuario u otro agente implemente la corrección. No edites archivos ni ejecutes la implementación, aunque el usuario apruebe la propuesta.
