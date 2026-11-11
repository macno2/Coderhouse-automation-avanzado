# Casos de prueba · Checkpoint 4 (Gmail + CRM Airtable + Slack)

Plan del test de regresión manual. Se prueba cada control de la rúbrica con un correo distinto.

## Antes de empezar

- Dos cuentas de Gmail de prueba para escribir al taller: **A** (cliente nuevo) y **B** (otra persona). No usar la casilla del taller para enviarse a sí misma: el trigger ignora los envíos propios y, según el filtro configurado, solo lee correos no leídos de los remitentes autorizados (por ejemplo `from:(cuentaA OR cuentaB) in:inbox is:unread`). Las cuentas de prueba tienen que estar dentro de ese filtro.
- Tabla **Clientes** de Airtable vacía (o sin las cuentas A y B) y tabla **Memoria** sin sus filas.
- En n8n, con el workflow abierto: mandar el correo, hacer clic en **Fetch Test Event** del Gmail Trigger y después en **Execute workflow**.

## Casos

| # | Remitente | Asunto / mensaje | Qué debería pasar | Control |
|---|---|---|---|---|
| 1 | `no-reply@banco.com` (o fijar el dato en el trigger) | "Automatic reply: Fuera de la oficina" | Sale por **Stop: auto-reply ignorado**. No se crea cliente, borrador ni aviso. | ① IF anti auto-reply |
| 2 | Cuenta A (nueva) | "Cambio de aceite" / "¿Cada cuánto me conviene cambiar el aceite de mi moto?" | Fila nueva en **Clientes** (`total_consultas` = 1). Ficha nueva en **Memoria**. Worker 1 responde. **Borrador** en Gmail (no se envía). Aviso en Slack. | ② Look up + Create, ③ Draft |
| 3 | Cuenta A (de nuevo) | "Turno" / "Quiero sacar un turno para el service de mi Kawasaki Versys 800" | **No** se crea otra fila: el Look up la encuentra y se actualiza (`total_consultas` = 2). Worker 2 redacta el correo. Borrador en el mismo hilo. Slack marcado `Revisar` si la confianza es media. | ② Look up (sin 409) |
| 4 | Cuenta B | "Pago" / "Pagué con la tarjeta 4509123412341234 y no figura" | El número aparece como `[NUMERO-ENMASCARADO]` en el asunto, en lo que lee el agente, en el borrador y en Slack. El remitente sale enmascarado en Slack (`p***@gotica.com`). | ④ Set de limpieza |
| 5 | Remitente vacío (fijar el dato en el trigger: `From` = "") | "Hola" | Sale por **Stop: sin remitente válido**. Sin llamada al CRM ni al modelo. | ④ validación (evita el 400) |
| 6 | Cuenta B | "Reclamo" / "El trabajo salió mal y quiero que me devuelvan la plata" | Ruta `DERIVAR_AL_TALLER`: sin Worker, borrador de derivación y aviso en Slack con `:rotating_light: ATENCIÓN`. | Escalamiento a humano |
| 7 | Cuenta C (nueva), **6 correos seguidos** en el mismo hilo | Ej.: "Hola, tengo una Honda CB 190", "Se me apaga en el semáforo", "Hace 12.000 km", "¿Cuánto sale el service?", "Quiero turno el martes", "Es para el service y la inyección" | Con el 6.º correo `Mensajes` pasa de 5 a 6, se dispara el resumen y **Guardar resumen** escribe sobre la **misma fila** de Memoria (mismo id). `Mensajes` vuelve a **0**. | Resumen y contador |

| 8 | Cuenta D (nueva), con el **mismo `threadId`** que otro cliente | Cualquier consulta | La memoria de la conversación no trae mensajes del otro cliente: `datos_conocidos` = `n/d` en el Router. | Aislamiento de sesión |

## Verificaciones puntuales (correcciones del Módulo 3)

- **Cliente nuevo vs. cliente existente (casos 2 y 3):** en el cliente nuevo, abrir la salida del nodo `Ficha`: los campos `resumen`, `datos_clave` y `proximo_paso` salen vacíos, pero el System Message dice "sin resumen todavía", "sin datos todavía" y "sin definir". No se inyectan valores vacíos.
- **System Prompt real (caso 2):** abrir la ejecución → nodo `Router de Triaje` → sub-nodo `OpenAI Chat Model (Router)` → **Input**. Ahí se ve el mensaje de sistema ya resuelto: los delimitadores `[INICIO DE CONTEXTO COMPARTIDO]` y `[FIN DEL CONTEXTO COMPARTIDO]` con el nombre y los datos reales, sin ninguna `{{ }}` sin resolver.
- **Seis correos (caso 7):** en Airtable, la fila de Memoria sigue siendo una sola, con `Resumen Consolidado`, `Datos Clave` y `Próximo Paso` completos y `Mensajes` = 0.

## Qué verificar en cada corrida

- **Gmail → Borradores:** el mensaje aparece como borrador en el hilo del cliente y no se envía nada.
- **Airtable → Clientes:** una sola fila por correo (los casos 2 y 3 comparten fila).
- **Airtable → Memoria:** una fila por correo; `Mensajes` sube en cada correo procesado.
- **Slack:** un mensaje corto por caso: categoría, nombre, remitente enmascarado, decisión, confianza y los primeros 300 caracteres de la consulta. Nada de adjuntos.
- **Panel de ejecución:** los nodos `Stop` solo se activan en los casos 1 y 5.

## Para las capturas del README

- Sección **Credentials** de n8n con Gmail, Airtable y Slack activas.
- Una corrida completa (caso 2 o 3) en verde.
- El borrador en Gmail y el aviso en Slack.
