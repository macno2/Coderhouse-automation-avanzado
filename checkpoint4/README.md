# Checkpoint 4 · Integraciones del agente del taller de motos (n8n)

Autor: Marcelo Carames · Curso AI Automation Avanzado – Agentes autónomos con n8n (comisión 97320).

El agente del taller (Módulo 3, con memoria) ahora **recibe los correos del taller por Gmail**, registra al cliente en un **CRM (Airtable)** y avisa al **equipo técnico por Slack**. Nunca envía respuestas solas: deja un **borrador** en Gmail para que una persona lo apruebe.

## Conexiones

| Rol | Herramienta | Autenticación | Uso |
|---|---|---|---|
| Casilla de correo | Gmail | OAuth2 | Entrada (trigger) y borrador de respuesta |
| CRM | Airtable, tabla **Clientes** | OAuth2 | Look up del cliente antes de crear |
| Canal del equipo | Slack, canal `#equipo-tecnico` | OAuth2 | Aviso corto al equipo técnico |

Capturas de las credenciales en n8n, con la conexión activa ("Account connected"):

**Airtable (CRM)**

![Credencial de Airtable](Credenciales%20Airtable.png)

**Gmail (correo), con permisos acotados**

![Credencial de Gmail](GMAIL%20CON%20CUSTOM%20SCOPE.png)

**Slack (canal del equipo), con permisos acotados**

![Credencial de Slack](SLACK%20CON%20CUSTOM%20SCOPE.png)

## Flujo

```
Gmail Trigger
  └─ ① IF anti auto-reply ── sí ─▶ Stop (corta el bucle)
       └─ ④ Set: limpiar y validar payload ─▶ IF remitente válido ── no ─▶ Stop
            └─ ② Look up en el CRM ─▶ ¿existe? ── sí ─▶ Update
                                               └─ no ─▶ Create
                 └─ Memoria del cliente (Módulo 3) ─▶ Router de triaje
                      └─ Worker 1 (consulta técnica) | Worker 2 (redacción) | Supervisor humano
                           └─ ④b Set: payload corto ─▶ ③ Create Draft (HITL) ─▶ Slack
                                └─ Actualizar ficha ─▶ (más de 5 mensajes) resumen con gpt-4o-mini
```

## Cumplimiento de la consigna, punto por punto

| # | Qué pide la consigna | Cómo se cumple | Dónde verlo en el flujo |
|---|---|---|---|
| 1 | Elegir tres conectores nativos: CRM, casilla de correo y canal del equipo | **Airtable** (CRM), **Gmail** (casilla) y **Slack** (canal). Los tres son nodos nativos de n8n | `2 Look up…` y `Create/Update cliente…` (Airtable), `Gmail Trigger` y `3 Create Draft` (Gmail), `Slack: aviso al equipo técnico` |
| 2 | Validar la autenticación de cada conector y ver el semáforo en verde | Cada credencial está probada y activa. Gmail usa OAuth2 | Sección **Credentials** de n8n: captura de arriba |
| 3 | Operaciones de lectura y escritura con permisos acotados (mínimo privilegio) | Ver la tabla de lectura/escritura más abajo. Cada conector usa solo las operaciones y permisos que necesita | Campos **Resource / Operation** de cada nodo y permisos de cada credencial |
| 4 | Un IF inmediatamente después del trigger del correo | `Gmail Trigger` se conecta directo a `1 IF: es auto-reply?`. No hay ningún nodo en el medio | `Gmail Trigger (casilla del taller)` → `1 IF: es auto-reply?` |
| 5 | Condición determinista que ignore auto-reply, Out of office, Undeliverable y `no-reply@` | El IF usa condiciones en **OR**, sin distinguir mayúsculas. Asunto contiene: *auto-reply, automatic reply, out of office, undeliverable, delivery status notification, respuesta automática, fuera de la oficina*. Remitente contiene: *no-reply@, noreply@, donotreply, mailer-daemon*. Si coincide, sale por `Stop: auto-reply ignorado`. Además el trigger solo lee correos **no leídos** de la bandeja de entrada y de remitentes autorizados (búsqueda de Gmail `from:(…) in:inbox is:unread`), que excluye los envíos de la propia casilla | `1 IF: es auto-reply?` y `Stop: auto-reply ignorado` |
| 6 | Look up antes del Create en el CRM (evitar el Error 409) | `2 Look up` busca el correo en **Clientes** (Airtable, operación Search, con la opción de devolver una salida aunque no haya resultados). `IF: el cliente ya existe?` decide: si existe, **Update**; si no, **Create**. El Create solo se ejecuta cuando el Look up no encontró nada | `2 Look up…` → `IF: el cliente ya existe?` → `Update…` / `2b Create…` |
| 7 | Salida de correo solo con Create Draft (Human-in-the-loop) | El único nodo de Gmail de salida es `3 Create Draft`, con **Resource = Draft**. No hay ningún nodo de **Send** en todo el flujo. El borrador queda en el mismo hilo | `3 Create Draft (aprobación humana)` |
| 8 | Nodos Set o filtros mínimos antes de los conectores de mensajería, para limpiar el payload | `4 Set` (entrada) deja solo remitente, nombre, asunto (200 caracteres), cuerpo (2.000 caracteres, con números largos enmascarados) y `threadId`. `4b Set` arma un texto corto para Slack: categoría, nombre, remitente enmascarado (`a***@gmail.com`), decisión, confianza y 300 caracteres de la consulta. Slack recibe solo texto: sin adjuntos, sin binarios y sin el correo completo. Un IF valida que haya remitente (evita el Error 400) | `4 Set: limpiar y validar payload`, `IF: remitente válido?` y `4b Set: preparar borrador y payload para Slack` |
| 9 | Test de regresión manual antes de exportar | Plan de 8 casos con resultado esperado en `casos_de_prueba_checkpoint4.md`. Resultados en la sección siguiente | Panel de ejecución de n8n |

### Mínimo privilegio: lectura y escritura por conector

| Conector | Lectura | Escritura | Permisos acotados |
|---|---|---|---|
| Gmail | Solo la bandeja de entrada, correos no leídos de remitentes autorizados (`from:(…) in:inbox is:unread`) | Solo **borradores** (Create Draft). Nunca envía: el flujo no tiene ningún nodo de Send | `gmail.readonly` y `gmail.compose` (Custom Scopes en la credencial de n8n). Se reemplazó el acceso total que n8n pide por defecto |
| Airtable | Search en **Clientes** y **Memoria** | Create y Update de registros en esas dos tablas | `data.records:read`, `data.records:write` y `schema.bases:read`, sobre una sola base |
| Slack | No lee | Un mensaje de texto por caso a `#equipo-tecnico` | `chat:write` y `channels:read` (Custom Scopes en la credencial de n8n, permisos de usuario de Slack). Se reemplazó la lista amplia que n8n pide por defecto |
| Google Sheets (Worker 1) | Lectura de filas de la base del taller | No escribe | Solo la operación de lectura como herramienta del agente |

### Resultados del test de regresión

Plan completo en `casos_de_prueba_checkpoint4.md`. Completar con el resultado de cada corrida:

| # | Caso | Control que prueba | Resultado |
|---|---|---|---|
| 1 | Auto-reply de `no-reply@banco.com.ar` | ① IF anti auto-reply | **OK.** El IF detectó el remitente `no-reply@` y el flujo salió por `Stop: auto-reply ignorado`. Solo se ejecutaron 3 nodos (Gmail Trigger, IF y Stop): no hubo limpieza, CRM, memoria, modelo, borrador ni aviso en Slack |
| 2 | Cliente nuevo con consulta técnica | ② Look up + Create, ③ Draft | **OK.** El Look up no encontró al cliente: se creó una sola fila en Clientes (`total_consultas` = 1) y una ficha en Memoria. El Router clasificó `CONSULTA_TECNICA` con confianza 0,98 y el Worker 1 respondió **RESUELTO** (TAL-01). Quedó un borrador en el hilo de Gmail (sin enviar) y el aviso llegó a `#equipo-tecnico` con el remitente enmascarado |
| 3 | Mismo cliente, segundo correo | ② Look up (sin duplicar) | **OK.** El Look up encontró al cliente y siguió por **Update**: no se creó otra fila en Clientes y `total_consultas` pasó de 1 a 2. La ficha de Memoria se actualizó (`Mensajes` = 2). El Router clasificó `COMUNICACION_AL_CLIENTE` (confianza 0,99) y el Worker 2 redactó el correo del turno, que quedó como borrador en el hilo. Aviso en `#equipo-tecnico` |
| 4 | Número de tarjeta en el asunto y el cuerpo | ④ Set de limpieza | **OK.** El número de 16 dígitos solo aparece en la salida del Gmail Trigger y en el IF anti auto-reply; desde `4 Set` en adelante figura como `[NUMERO-ENMASCARADO]` (lo que lee el agente, el borrador, el aviso de Slack y las tablas de Airtable). El Router derivó el caso (`DERIVAR_AL_TALLER`, confianza 0,98), la ficha quedó en `escalado` y Slack recibió el aviso con `:rotating_light: ATENCIÓN` y el remitente enmascarado |
| 5 | Correo sin remitente válido | ④ validación | **OK.** Con `From` vacío, el Set dejó `from_email` vacío y `IF: remitente válido?` cortó por `Stop: sin remitente válido`. Solo se ejecutaron 5 nodos: no hubo llamadas al CRM, a la memoria, al modelo ni a Slack, y no se creó ninguna fila (evita el Error 400) |
| 6 | Reclamo con pedido de reembolso | Escalamiento a humano | **OK.** El Router clasificó `DERIVAR_AL_TALLER` (confianza 0,99) y el flujo fue directo a `Derivación a Supervisor Humano`, sin invocar ningún Worker. Se creó el borrador de derivación en el hilo (sin enviar), la ficha quedó en `escalado` y Slack recibió el aviso con `:rotating_light: ATENCIÓN`. El cliente existente se actualizó en el CRM sin duplicarse |
| 7 | Seis correos seguidos del mismo cliente | Resumen sobre la misma fila y contador en 0 | **OK.** El primer correo creó al cliente (Create) y los cinco siguientes lo actualizaron (Update), con `total_consultas` = 6 y una sola fila. En Memoria, `Mensajes` subió de 1 a 6; con el 6.º se cumplió `Mensajes > 5` y se ejecutó `Leer la mesa` → `Resumir` (gpt-4o-mini, JSON con las tres claves) → `Guardar resumen`, que escribió sobre la **misma fila** (mismo id de registro). `Mensajes` volvió a **0** y la memoria de la conversación se reinició. El resumen guardó solo hechos accionables (moto, kilómetros, síntoma, servicio y turno), sin transcripción |
| 8 | Dos remitentes distintos con el mismo `threadId` | Aislamiento de sesión | **OK.** Con el mismo hilo que usó otro cliente, la memoria cargó **0 mensajes previos** y `datos_conocidos` salió `n/d`: no hay datos ajenos. Se crearon su fila en Clientes y su ficha en Memoria. Antes de usar la clave `Session_ID\|threadId`, la memoria sí se compartía entre ambos |

## Validaciones de la memoria (devolución del Módulo 3)

**Modo del agente.** Los nodos AI Agent del flujo (`Router de Triaje`, `Agente Consultor Técnico` y `Agente Redactor de Correos`) trabajan en modo **Tools Agent**. En n8n 2.45.6 el nodo AI Agent solo opera en ese modo, y cada nodo lo indica con una nota visible en el lienzo ("AI Agent · modo Tools Agent").

**Expresiones.** Las 131 expresiones `{{ }}` de los tres workflows se revisaron una por una con un chequeo de sintaxis: 131 válidas y 0 con error. Los System Message y los prompts están en modo **Expression** (se ve el `=` al principio en el JSON).

**Evidencia en una ejecución real** (tomada de una corrida real del Módulo 3 en n8n; el Módulo 4 usa la misma lógica de memoria):

- **Usuario nuevo, sin valores vacíos.** El nodo `Ficha` salió con `resumen`, `datos_clave` y `proximo_paso` vacíos, y el System Message que recibió el agente quedó así:

  ```
  [INICIO DE CONTEXTO COMPARTIDO]
  Cliente: Jane Doe
  Estado del caso: recibido
  Resumen: sin resumen todavía
  Datos clave: sin datos todavía
  Próximo paso: sin definir
  [FIN DEL CONTEXTO COMPARTIDO]
  ```

  Los delimitadores y los valores llegaron resueltos, sin llaves `{{ }}` pendientes.
- **Resumen sobre la misma fila.** Con `Mensajes` = 6 la condición `Mensajes > 5` fue verdadera: se leyó la memoria, el modelo económico devolvió el JSON con las tres claves y `Guardar resumen` escribió sobre el **mismo registro** que había creado `Crear ficha` (mismo id). El contador volvió a **0** y la memoria de la conversación se reinició.

Estas pruebas se repitieron con correos reales en el Módulo 4: ver los casos 2, 3 y 7 en la tabla de resultados de más abajo.

## Evidencia de conexiones y permisos

Los permisos de cada credencial no viajan en el `.json` exportado (n8n guarda solo la referencia a la credencial), por eso se documentan con capturas:

- `Credenciales Airtable.png`: credencial de Airtable (OAuth2) conectada. Usa los tres permisos que n8n pide por defecto (`data.records:read`, `data.records:write`, `schema.bases:read`) y acceso a una sola base.
- `GMAIL CON CUSTOM SCOPE.png`: credencial de Gmail con **Custom Scopes** activado y **Enabled Scopes** en `gmail.readonly` y `gmail.compose`.
- `SLACK CON CUSTOM SCOPE.png`: credencial de Slack con **Custom Scopes** activado y **User Scope** en `chat:write channels:read`.
- `Credenciales Google sheets.png`: credencial de Google Sheets que usa el Worker 1 para leer la base técnica del taller.

## Memoria y datos

- **Aislamiento de sesión:** el `Session_ID` de la ficha es el correo del remitente. La memoria de la conversación (`La mesa`) usa como clave `Session_ID|threadId`, de modo que dos clientes nunca comparten memoria aunque coincida el hilo.
- **Tabla Memoria:** `Session_ID`, `Nombre`, `Fecha de Actualización`, `Mensajes`, `Estado del Caso`, `Resumen Consolidado`, `Datos Clave`, `Próximo Paso`.
- **Tabla Clientes (CRM):** `email`, `nombre`, `fecha_alta`, `ultimo_contacto`, `total_consultas`.

## Archivos

- `checkpoint4_marcelo_carames.json`: workflow Manager exportado de n8n.
- `worker1_modulo3_carames_marcelo.json` y `worker2_modulo3_carames_marcelo.json`: los dos Workers (sub-workflows) que usa el Manager.
- `casos_de_prueba_checkpoint4.md`: plan de pruebas manuales.
- Capturas de las credenciales y sus permisos: `Credenciales Airtable.png`, `GMAIL CON CUSTOM SCOPE.png`, `SLACK CON CUSTOM SCOPE.png` y `Credenciales Google sheets.png`.

## Cómo importarlo

1. Importar primero los dos Workers (**Workflows → Import from file**) y guardarlos.
2. Importar `checkpoint4_marcelo_carames.json`.
3. Crear y asignar las credenciales de **Gmail**, **Airtable**, **Slack** y **OpenAI** (y **Google Sheets** en el Worker 1). En el **Gmail Trigger**, ajustar el filtro de búsqueda a los remitentes de prueba propios; por defecto: `-from:me in:inbox`.
4. En los dos nodos **Execute Workflow**, elegir el Worker que corresponde.
5. En el nodo de Slack, elegir el canal del equipo técnico (`#equipo-tecnico`) y, en el Worker 1, reemplazar `REEMPLAZAR_DOC_ID_BASE_TALLER` y `REEMPLAZAR_GID_HOJA` por la base del taller.
6. Probar con el plan de `casos_de_prueba_checkpoint4.md`.

Versión de n8n usada: 2.45.6 (self-hosted, Docker).
