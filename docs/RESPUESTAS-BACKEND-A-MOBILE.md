# Respuestas y Definiciones de Backend para KMA Mobile

**Fecha:** 22/09/2026  
**Referencia:** [docs/DECISIONES-BACKEND-MOBILE.md](file:///h:/Projects/KMA/kma-backend/docs/DECISIONES-BACKEND-MOBILE.md) y [docs/RFC-AUDITS-V2-MOBILE-SYNC.md](file:///h:/Projects/KMA/kma-backend/docs/RFC-AUDITS-V2-MOBILE-SYNC.md)  
**Estado:** Acordado y en proceso de implementación en Backend.

Este documento responde punto por punto a las consultas del equipo Mobile, detallando el **estado actual**, la **decisión técnica acordada**, el **contrato definitivo** y los **cambios que Backend implementa** para garantizar la integridad de datos y simplificar el desarrollo en Mobile.

---

## Índice Rápido de Respuestas

| # | Pregunta Mobile | Respuesta Backend | Acción Backend |
|---|---|---|---|
| 🔴 **1** | ¿`POST /audits` es idempotente si el ID existe? | **No era idempotente (hacía upsert ciego). Ahora responde `409 Conflict` con guarda atómica DynamoDB.** | Agregado `attribute_not_exists(id)` en `Create`. Retorna 409 con mensaje claro. |
| 🔴 **2** | ¿Quién genera el `audit_id`: móvil o `/uploads`? | **El móvil genera el `audit_id` al iniciar el borrador (`POST /audits`).** Luego `/uploads` lo recibe para firmar fotos. | Agregar `expires_at` en respuesta de presign en `/uploads`. Mantener validación de pertenencia. |
| 🔴 **3** | ¿Cómo se borra una respuesta de rama abandonada? | **Se acepta `deleted_step_ids: string[]` en `PUT /api/audits/{id}`.** Backend limpia esos pasos atómicamente en el merge. | Agregar `deleted_step_ids` en DTO, UseCase y Domain `MergeAnswers`. |
| 🔴 **4** | Regla de compatibilidad para `GET /flows` | **Aceptada la regla propuesta.** Propiedades nuevas opcionales son libres. Tipos de campos o pasos nuevos se coordinan previamente. | Documentado como estándar en el monorepo. |
| 🟠 **5** | Convención de nombres de archivo en S3 | **Nombre único y estable por foto:** `<step_id>_<field_id>_<id_local>.<ext>`. El `PUT` a S3 es idempotente. | Confirmado. La key final en S3 es `<audit_id>/<step_id>/<name>`. |
| 🟠 **6** | ¿El submit devuelve versión final? | **Sí.** Se incorporan `version` y `updated_at` en el body de respuesta de `POST /audits/{id}/submit`. | Modificado `SubmitAuditResponse` DTO para exponer `version` y `updated_at`. |
| 🟠 **7** | Formato exacto del `id` de auditoría | **Formato oficial:** `audit_<uuid>_<yyyyMMddHHmmss>` (ej. `audit_7219fcce-13cd-4da6-881e-33dca6b41102_20260922153000`). | Estandarizado en DTOs y validaciones regex. |
| 🟡 **8** | Endpoints de lectura de auditorías | **Ya existen y están disponibles:** `GET /api/audits` (listado con filtros) y `GET /api/audits/{id}` (detalle completo). | Expuesto campo `version` en `AuditListItemResponse` del listado. |
| 🟡 **9** | ¿Se puede editar después del submit? | **Sí, mientras esté en `draft_report_pending_review`.** Se bloquea con `400` una vez que un revisor web la pasa a `draft_report_in_review` o `completed`. | Agregada validación de estado en `UpdateAudit`. |
| 🟡 **10** | ¿El `409` de conflicto dice qué tiene el servidor? | **Devuelve `server_version` y mensaje explicativo.** Mobile puede usar `GET /api/audits/{id}` para obtener el estado completo si requiere resolver conflictos. | Formato estándar `VersionConflictResponse`. |
| 🟡 **11** | Límite de escrituras / guardado automático | **100% de acuerdo con Mobile.** No usar timer ciego de 2s. Agrupar deltas por paso o por lote de sincronización en un solo `PUT`. | Sin rate-limit estricto que bloquee a Mobile; optimiza costos DynamoDB. |
| 🟡 **12** | Contrato de campos de formulario | **El array ya viene ordenado.** Se agregan `required: boolean` y `multiline: boolean` como campos opcionales no-breaking. | Incorporación gradual en modelo de Flow. |
| 🟢 **13** | Mensajes legibles en errores `4xx` | **Sí, garantizado por `httputil.ErrorResponse`:** `{ "error": "...", "message": "...", "code": 4xx }`. | Estándar en todos los endpoints del backend. |
| 🟢 **14** | Política de fotos eliminadas en servidor | **Mobile no debe llamar a un `DELETE` de fotos.** Si una foto sale de `answers`, queda desasociada del reporte. Cleanup gestionado en S3. | Cero esfuerzo para Mobile. |
| 🟢 **15** | Borradores abandonados | **Backend se encarga de la depuración periódica.** Mobile solo descarta su base local. | En Web solo se listan auditorías enviadas o completadas por default. |
| 🟢 **16** | Versión del flow por auditoría | **`GET /api/audits/{id}` ya devuelve `flow_id` y `flow_version`.** El catálogo soporta versionado inmutable. | Ya implementado en modelo de dominio. |

---

# Detalle Punto por Punto

---

## 🔴 Tier 0 — Críticas: Integridad de Datos

### 1. Idempotencia de `POST /audits`
- **Diagnóstico:** El código anterior hacía un `PutItem` sin condición en DynamoDB. Si un reintento tardío del móvil enviaba la creación inicial con `answers: []`, pisaba y borraba todas las respuestas ya guardadas por deltas.
- **Decisión:** 
  1. `Repository.Create` ahora incluye la expresión condicional:
     ```
     attribute_not_exists(id)
     ```
  2. Si el `id` ya existe en DynamoDB, la base rechaza la operación y el backend responde **`HTTP 409 Conflict`**:
     ```json
     {
       "error": "conflict",
       "message": "Audit already exists with id: audit_...",
       "code": 409
     }
     ```
- **Comportamiento en Mobile:** Mobile puede tratar con seguridad un `409` en la creación como éxito de sincronización (la auditoría ya fue creada previamente en el servidor y no fue pisada).

---

### 2. Ciclo de Vida y Generación de `audit_id`
- **Decisión:** El `audit_id` lo genera el **Móvil al presionar "Iniciar Auditoría"** con el formato oficial `audit_<uuid>_<yyyyMMddHHmmss>`.
- **Nuevo Flujo:**
  1. Móvil genera `audit_id`.
  2. Móvil llama a `POST /api/audits` creando el borrador inicial en servidor (`status: audit_in_progress`, `answers: []`, `version: 1`).
  3. Cuando el auditor saca fotos (incluso durante la auditoría), Móvil llama a `POST /api/uploads` pasando:
     ```json
     {
       "audit_id": "audit_1234_20260922153000",
       "files": [
         {
           "name": "step_3_field_photo_loc1.jpg",
           "step_id": "step_3",
           "content_type": "image/jpeg"
         }
       ]
     }
     ```
  4. La lambda `/uploads` valida que la auditoría exista en DynamoDB y que pertenezca al usuario autenticado. Si no existe, devuelve `404 Not Found`. Si existe, emite las URLs firmadas de S3.
- **Mejoras en `/uploads`:**
  - Se agrega `expires_at` (Unix timestamp en segundos) en cada ítem de respuesta para que Mobile sepa con exactitud cuándo expira la URL presignada (por defecto 15 minutos).
  - Si una auditoría no tiene fotos, Mobile **no debe llamar** a `/uploads` (el campo `files` no puede estar vacío).

---

### 3. Borrado de Respuestas en Ramas Abandonadas
- **Diagnóstico:** El merge del servidor solo insertaba o reemplazaba por `step_id`. Al cambiar de rama condicional, los pasos viejos quedaban huérfanos y eran procesados erróneamente por el motor de enriquecimiento.
- **Decisión:** Se habilita formalmente el campo `deleted_step_ids` en el endpoint `PUT /api/audits/{id}` y `PATCH /api/audits/{id}`:
  ```json
  {
    "version": 3,
    "answers": [
      {
        "step_id": "step_branch_b_1",
        "type": "Question",
        "answer": "SI"
      }
    ],
    "deleted_step_ids": [
      "step_branch_a_1",
      "step_branch_a_2"
    ]
  }
  ```
- **Lógica en Backend:** Durante la actualización, Backend primero elimina de la lista de respuestas existentes cualquier respuesta cuyo `step_id` esté en `deleted_step_ids`, y luego aplica el merge de las `answers` nuevas o actualizadas.

---

### 4. Regla de Compatibilidad para `GET /flows`
- **Acuerdo Aceptado:**
  - ✅ **Cambios No Rupturistas (Libres en cualquier momento):** Agregar propiedades opcionales nuevas a pasos o campos existentes (`order`, `required`, `multiline`, `tooltip`, etc.). La app móvil las ignorará o las aprovechará sin fallar.
  - ⛔ **Cambios Rupturistas (Requieren Release Coordinada):** Agregar nuevos valores a los enums `type` de pasos o `type` de campos (más allá de `Question`, `Select`, `Form`, `text`, `number`, `photo`, `button`). Se coordinará con Mobile previo al alta en el catálogo.

---

## 🟠 Tier 1 — Importantes

### 5. Convención de Nombres de Archivo en S3
- **Contrato:**
  - S3 Key generada por backend: `<audit_id>/<step_id>/<file_name>`
  - Mobile debe nombrar cada foto de forma **única y determinística**:
    ```
    <step_id>_<field_id>_<id_local>.<extension>
    ```
    Ejemplo: `step04_photo1_a8f9c2d1.jpg`
  - Un reintento de subida (`PUT` a S3) sobre la misma URL presignada es idempotente y seguro.
  - Al editar un paso y sacar una foto nueva, generar un nuevo `<id_local>` para no sobrescribir evidencia previa hasta confirmar el guardado.

---

### 6. Versión Final en `POST /audits/{id}/submit`
- **Decisión:** El body de respuesta de `POST /api/audits/{id}/submit` ahora incluye la versión incrementada y la fecha de actualización:
  ```json
  {
    "id": "audit_abc_20260922153000",
    "version": 4,
    "status": "draft_report_pending_review",
    "message": "Audit submitted successfully for enrichment",
    "updated_at": "2026-09-22T20:30:00Z"
  }
  ```
- Mobile puede guardar de inmediato `version: 4` en su base local sin necesidad de realizar un `GET` adicional.

---

### 7. Formato Oficial del `audit_id`
- **Formato Obligatorio:**
  ```text
  audit_<uuidv4>_<yyyyMMddHHmmss>
  ```
  Ejemplo: `audit_9b1deb4d-3b7d-4bad-9bdd-2b0d7b3dcb6d_20260922153045`
- Backend valida este prefijo y largo máximo (128 caracteres) tanto en `audits` como en `uploads`.

---

## 🟡 Tier 2 — Definición de Alcance y Funcionalidad

### 8. Endpoints de Lectura de Auditorías (`GET`)
Ya están implementados y desplegados:
1. **Listado:** `GET /api/audits`
   - Query Params:
     - `auditor`: filtra por username/email del auditor.
     - `status`: filtra por estado (`audit_in_progress`, `draft_report_pending_review`, `completed`, etc.).
     - `limit`: número de registros por página.
     - `last_eval_id`: token para paginación consecutiva.
   - Respuesta incluye datos enriquecidos: `flow_name`, `project_name`, `facility_name`, `version`, `findings_count`, `created_at`, `status`.
2. **Detalle:** `GET /api/audits/{id}`
   - Devuelve la auditoría completa con todas sus `answers`, `flow_version`, metadata del proyecto y auditor.

---

### 9. Edición de Auditorías tras el Submit
- **Regla de Negocio:**
  - Se permite `PUT /api/audits/{id}` y `PATCH /api/audits/{id}` mientras el estado sea:
    - `audit_in_progress` (durante la captura en campo).
    - `draft_report_pending_review` (completada por el auditor, en cola o pendiente de revisión).
  - Se bloquea la edición (`400 Bad Request`) si la auditoría ya pasó a:
    - `draft_report_in_review` (el revisor técnico ya está editando el reporte en la Web).
    - `completed` / `final_report_sent_to_client` (reporte final emitido).
- Si se edita en `draft_report_pending_review`, backend vuelve a disparar el evento de enriquecimiento a SQS para recalcular hallazgos.

---

### 10. Respuesta en Conflicto de Versión (`409 Conflict`)
- Cuando la versión enviada en el `PUT` no coincide con la versión actual en la base (`incomingVersion != serverVersion`), backend responde:
  ```json
  {
    "error": "conflict",
    "message": "Audit version conflict detected",
    "server_version": 5,
    "code": 409
  }
  ```
- **Estrategia para Mobile:**
  - Si el móvil estaba trabajando solo en su auditoría, adopta `server_version` y reintenta.
  - Si necesita reconciliar respuestas, llama a `GET /api/audits/{id}` para obtener la foto exacta del servidor.

---

### 11. Estrategia de Guardado Automático (Escrituras)
- **Aprobada la propuesta de Mobile:**
  - Guardar al completar cada paso o agrupar deltas en un `PUT` por ciclo de sincronización (debounce / outbox).
  - No usar timers fijos agresivos (ej. cada 2 segundos).
  - Esto minimiza consumo de batería, datos móviles y provisionamiento de DynamoDB.

---

### 12. Contrato de Campos de Formulario en Flows
- El array `fields` del catálogo de flows ya mantiene un orden estricto de elementos en el JSON. Mobile puede renderizarlos en el orden del array.
- Backend incorpora soporte para:
  - `required: boolean` (opcional, default `false`).
  - `multiline: boolean` (opcional, default `false`).
  - `order: number` (opcional, para ordenamiento explícito).

---

## 🟢 Tier 3 — Definiciones Operativas y Ciclo de Vida

### 13. Formato Uniforme de Errores `4xx`
Todos los microservicios retornan errores con la estructura:
```json
{
  "error": "bad_request | not_found | conflict | unauthorized",
  "message": "Descripción legible para mostrar en logs o alert",
  "code": 400
}
```

### 14. Política de Fotos en S3
- Mobile **no** necesita llamar a ningún endpoint de borrado de fotos.
- Si una foto se retira de `answers`, el reporte oficial ya no la mostrará.
- Backend configurará políticas de ciclo de vida (Lifecycle Rules) en S3 para eliminar fotos huérfanas en buckets temporales tras un período de retención prudencial.

### 15. Borradores Abandonados
- En el portal Web no se visualizan borradores en estado `audit_in_progress`.
- Si el auditor cancela o descarta localmente una auditoría que nunca completó, puede invocar `DELETE /api/audits/{id}` si hay conexión, o simplemente descartarla en el teléfono. Backend aplicará un proceso de limpieza para borradores inactivos con más de 90 días.

### 16. Versión Histórica del Flow
- El objeto de la auditoría persiste permanentemente `flow_id` y `flow_version`.
- Si el catálogo de flows se actualiza a una versión posterior, la auditoría conserva su referencia a la versión histórica con la que fue realizada.

---

## 🧪 Guía Práctica de Pruebas Paso a Paso para Frontend / Mobile

Esta guía permite al equipo de Frontend / Mobile probar y validar el ciclo de vida completo de una auditoría en Postman o en su cliente HTTP, incluyendo **creación con ubicación inicial**, **guardado por deltas**, **cambio de ramas con `deleted_step_ids`**, **control de versiones optimista (OCC)** y **envío final (Submit)**.

### Configuración Previa
- **Base URL (Dev):** `https://{{api_id}}.execute-api.us-east-1.amazonaws.com` (o el dominio configurado)
- **Headers requeridos en todas las peticiones:**
  ```http
  Authorization: Bearer {{jwt_token}}
  Content-Type: application/json
  ```
- **Variables de prueba recomendadas:**
  - `flow_id`: `"flow_ramp_accessibility_verification_20251009190307"` (Flujo de Rampas)
  - `flow_version`: `21` (o la versión activa en dev)
  - `project_id`: `"project_test_001"`
  - `facility_id`: `"facility_test_001"`
  - `audit_id`: `"audit_" + uuid() + "_" + timestamp()` (ej: `audit_7219fcce-13cd-4da6-881e-33dca6b41102_20260930103000`)

---

### Paso 1: Iniciar Auditoría y Crear Borrador (`POST /api/audits`)
Al pulsar "Iniciar Auditoría", el móvil genera el `audit_id` y crea el borrador. Puede enviar la ubicación inicial (`location`) en el paso `R-L01`:

- **Método / Ruta:** `POST /api/audits`
- **Body:**
  ```json
  {
    "id": "audit_7219fcce-13cd-4da6-881e-33dca6b41102_20260930103000",
    "flow_id": "flow_ramp_accessibility_verification_20251009190307",
    "flow_version": 21,
    "project_id": "project_test_001",
    "facility_id": "facility_test_001",
    "answers": [
      {
        "step_id": "R-L01",
        "type": "Form",
        "values": {
          "location": "Entrada Principal - Acceso Este"
        }
      }
    ]
  }
  ```
- **Respuesta esperada (`201 Created`):**
  ```json
  {
    "id": "audit_7219fcce-13cd-4da6-881e-33dca6b41102_20260930103000",
    "flow_id": "flow_ramp_accessibility_verification_20251009190307",
    "flow_version": 21,
    "version": 1,
    "status": "audit_in_progress",
    "created_at": "2026-09-30T13:30:00Z"
  }
  ```
- **Prueba de Idempotencia:** Si se vuelve a ejecutar este mismo `POST` con el mismo `id`, el servidor responderá **`409 Conflict`**, garantizando que un reintento no pise datos existentes.

---

### Paso 2: Guardar Respuestas con Hallazgo (`PATCH /api/audits/{id}`)
A medida que el auditor avanza, el móvil envía deltas mediante `PATCH`.
En este paso, responde `"no"` a la pregunta `R-B01` ("¿La rampa está en una ruta accesible?") y completa el formulario `F01`:

- **Método / Ruta:** `PATCH /api/audits/audit_7219fcce-13cd-4da6-881e-33dca6b41102_20260930103000`
- **Body:**
  ```json
  {
    "version": 1,
    "answers": [
      {
        "step_id": "R-B01",
        "type": "Question",
        "answer": "no"
      },
      {
        "step_id": "F01",
        "type": "Form",
        "values": {
          "quantity": 1,
          "notes": "No existe ruta accesible directa hacia la rampa.",
          "photos": [
            "https://kma-bucket.s3.amazonaws.com/photo1.jpg"
          ]
        }
      }
    ],
    "deleted_step_ids": []
  }
  ```
- **Respuesta esperada (`200 OK`):**
  ```json
  {
    "id": "audit_7219fcce-13cd-4da6-881e-33dca6b41102_20260930103000",
    "version": 2,
    "status": "audit_in_progress",
    "updated_at": "2026-09-30T13:31:00Z"
  }
  ```
  *(El servidor incrementa la versión a `2`)*.

---

### Paso 3: Cambio de Rama y Limpieza con `deleted_step_ids`
Si el auditor vuelve atrás y cambia la respuesta de `R-B01` a `"yes"` (conforme), los pasos de la rama del "NO" (`F01`) quedan huérfanos. Se envían en `deleted_step_ids` para eliminarlos atómicamente:

- **Método / Ruta:** `PATCH /api/audits/audit_7219fcce-13cd-4da6-881e-33dca6b41102_20260930103000`
- **Body:**
  ```json
  {
    "version": 2,
    "answers": [
      {
        "step_id": "R-B01",
        "type": "Question",
        "answer": "yes"
      },
      {
        "step_id": "R-B02",
        "type": "Question",
        "answer": "no"
      },
      {
        "step_id": "F02",
        "type": "Form",
        "values": {
          "measurements": 9.5,
          "quantity": 1,
          "notes": "Pendiente de 9.5% supera el 8.3% permitido."
        }
      }
    ],
    "deleted_step_ids": [
      "F01"
    ]
  }
  ```
- **Respuesta esperada (`200 OK`):**
  ```json
  {
    "id": "audit_7219fcce-13cd-4da6-881e-33dca6b41102_20260930103000",
    "version": 3,
    "status": "audit_in_progress",
    "updated_at": "2026-09-30T13:32:00Z"
  }
  ```
  *(El servidor elimina `F01`, actualiza `R-B01` a `"yes"`, añade `R-B02` y `F02`, e incrementa versión a `3`)*.

---

### Paso 4: Prueba de Conflicto de Versión (OCC — `409 Conflict`)
Si el móvil enviara erróneamente una versión desactualizada (por ejemplo `version: 1` cuando el servidor ya está en `3`):

- **Método / Ruta:** `PATCH /api/audits/audit_7219fcce-13cd-4da6-881e-33dca6b41102_20260930103000`
- **Body:**
  ```json
  {
    "version": 1,
    "answers": []
  }
  ```
- **Respuesta esperada (`409 Conflict`):**
  ```json
  {
    "error": "conflict",
    "message": "Audit version conflict detected",
    "server_version": 3,
    "code": 409
  }
  ```
- **Acción del Móvil:** Adoptar `"version": 3` (o sincronizar vía `GET`) y reintentar.

---

### Paso 5: Enviar la Auditoría para Enriquecimiento (`POST /submit`)
Una vez finalizada la carga en campo, el móvil somete la auditoría:

- **Método / Ruta:** `POST /api/audits/audit_7219fcce-13cd-4da6-881e-33dca6b41102_20260930103000/submit`
- **Body:** `{}` (no requiere payload)
- **Respuesta esperada (`200 OK`):**
  ```json
  {
    "id": "audit_7219fcce-13cd-4da6-881e-33dca6b41102_20260930103000",
    "version": 4,
    "status": "draft_report_pending_review",
    "message": "Audit submitted successfully for enrichment",
    "updated_at": "2026-09-30T13:33:00Z"
  }
  ```
- **Qué sucede en Backend:**
  1. Estado cambia a `draft_report_pending_review`.
  2. Publica mensaje a SQS `kma-audit-ready-to-enrich-dev`.
  3. `audit-enrichment-processor` busca automáticamente en `kma-flows-catalog-dev` la mitigación y el costo para la barrera `R-B02` (costo unitario $15,000 ea.) y genera el review enriquecido en `kma-audits-review-dev`.

---

### Paso 6: Verificación del Estado Final (`GET /api/audits/{id}`)
- **Método / Ruta:** `GET /api/audits/audit_7219fcce-13cd-4da6-881e-33dca6b41102_20260930103000`
- **Respuesta esperada (`200 OK`):**
  - `status: "draft_report_pending_review"`
  - `version: 4`
  - `answers` contiene exactamente:
    - `R-L01`: `values.location: "Entrada Principal - Acceso Este"`
    - `R-B01`: `answer: "yes"`
    - `R-B02`: `answer: "no"`
    - `F02`: `measurements: 9.5`, `quantity: 1`
  - El paso `F01` **no aparece** (confirmando que fue purgado correctamente por `deleted_step_ids`).
