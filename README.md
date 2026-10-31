# Agente de soporte técnico para un taller de motos (n8n)

Proyecto integrador del curso **AI Automation Avanzado – Agentes autónomos con n8n** (Coderhouse, comisión 97320).
Autor: Marcelo Carames.

Es un solo proyecto que crece módulo a módulo: en cada checkpoint se importa el `.json` anterior y se extiende, sin rehacer el flujo.

## Qué hace

Un cliente escribe en lenguaje natural ("la moto no arranca, hace clic y la batería es nueva"). El agente consulta la base del taller y cierra cada mensaje con una de tres decisiones:

- **RESUELTO:** responde con la regla de la base y cita el artículo (por ejemplo, TAL-08).
- **FALTA UN DATO:** pide lo que figura en la columna `Datos_necesarios`.
- **DERIVADO AL TALLER:** pasa el caso a una persona (presupuestos no estándar, frenos, garantías, reclamos, turnos).

El agente no da precios salvo los orientativos del cambio de aceite y del service de 10.000 km, no confirma garantías y no autoriza a circular con fallas de frenos.

## Flujo del Checkpoint 1

`Chat Trigger` → `AI Agent` → `Gmail (log_observabilidad)` → `Respuesta al chat`

| Componente | Implementación |
|---|---|
| Trigger | Chat Trigger, texto libre |
| AI Agent | Tools Agent con OpenAI Chat Model (`gpt-5.4-mini`) |
| Guardrail | `maxIterations = 6` |
| System Prompt | 4 bloques: Rol y ámbito / Objetivo / Reglas de herramientas / Escalamiento |
| Tool | `consultar_base_tecnica` (Google Sheets), conectada por `ai_tool`, con description extensa y filtro `Categoria` vía `$fromAI` |
| Observabilidad | Gmail después del agente, por `main`: decisión, moto o modelo, categoría y artículo citado |
| Memoria | Simple Memory (ventana de 10 mensajes) |

## Archivos

- `checkpoint1_marcelo_carames.json`: workflow exportado de n8n.
- `Base del taller - Clase 1.csv`: base de conocimiento (26 artículos en 4 categorías: `mantenimiento`, `falla_tecnica`, `presupuesto`, `servicio_taller`). Todos los datos son ficticios.

## Cómo usarlo

1. Importar `Base del taller - Clase 1.csv` a Google Sheets y anotar el ID del documento y el GID de la hoja.
2. En n8n: **Workflows → Import from file** y elegir el `.json`.
3. Crear y asignar las credenciales de OpenAI, Google Sheets (OAuth2) y Gmail (OAuth2).

4. Probar con el chat de n8n y revisar el Execution Log.

Versión de n8n usada: 2.45.6 (self-hosted, Docker).

## Hoja de ruta

| Módulo | Qué suma al taller | Archivo |
|---|---|---|
| M1 | Agente base con base técnica y log por Gmail | `checkpoint1_marcelo_carames.json` |
| M2 | Manager-Worker con sub-workflows | pendiente |
| M3 | Memoria persistente por cliente y moto | pendiente |
| M4 | Integraciones OAuth2 (turnos, confirmaciones) | pendiente |
| M5 | RAG con manuales de servicio | pendiente |
| M6 | Voz (STT y TTS) | pendiente |
| M8–M9 | AI-as-a-Judge, costos y gobernanza | pendiente |
| M11 | Proyecto final integrador | pendiente |
