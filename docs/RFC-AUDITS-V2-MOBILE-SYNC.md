# 📄 RFC: Auditorías V2 – Guardado Progresivo por Deltas, Control de Versión y Arquitectura Offline-First (React Native)

**Estado:** Propuesto / En Revisión  
**Fecha:** Enero 2026  
**Audiencia:** Equipo Frontend Mobile (React Native) & Backend  
**Módulo:** `audits`, `uploads`, `audit-enrichment-processor`  

---

## 1. 🎯 Motivación: ¿Por qué este cambio?

En la versión actual (V1), la aplicación móvil acumula **todas** las respuestas, formularios, medidas y fotos en el almacenamiento local del teléfono, y únicamente al presionar el botón final de envío se dispara todo en bloque:
1. Pide URLs prefirmadas para decenas de fotos a la vez.
2. Sube todas las fotos a S3 en serie/paralelo.
3. Envía un JSON gigante mediante `POST /api/audits` con todo el contenido.
4. El backend crea la auditoría y dispara de inmediato el enriquecimiento a SQS.

### Problemas detectados en campo:
- ❌ **Riesgo crítico de pérdida de datos:** Los auditores realizan inspecciones de 1 a 3 horas en terreno. Si el teléfono se queda sin batería, el sistema operativo mata la app en segundo plano, o la aplicación crashea antes de llegar al final, **se pierde todo el trabajo realizado**.
- ❌ **Bloqueo y timeouts por red móvil inestable:** Subir 30 a 50 fotos de alta resolución más un JSON enorme con cobertura 3G/4G débil (sótanos, estacionamientos, autopistas) genera pantallas de carga de 5 a 10 minutos, fallos por timeout de red y frustración en el usuario.
- ❌ **Límites de infraestructura:** La lambda de `uploads` restringe a un máximo de 20 archivos por petición (`maxFiles: 20`), obligando a partir la petición al final.
- ❌ **Falta de visibilidad de progreso:** Hasta que el auditor no envía todo, nadie en la plataforma web sabe si una auditoría se inició o qué porcentaje de avance lleva.

---

## 2. 💡 Principios de Sincronización: Deltas, Idempotencia y Versión

### 2.1. Sincronización por Deltas (Solo lo nuevo o modificado)
- **El móvil NO reenvía lo que ya fue guardado con éxito.** Cada respuesta tiene un estado local (`synced` o `pending`). Solo viajan al servidor las respuestas que están en `pending`.
- **Merge en Backend por `step_id`:** El backend busca la auditoría en DynamoDB y fusiona las respuestas recibidas:
  - Si el `step_id` no existía $\rightarrow$ se agrega a la lista.
  - Si el `step_id` ya existía (el auditor editó un paso anterior) $\rightarrow$ se reemplaza con el nuevo valor.
- **El Submit final va sin respuestas (`{}`):** Como los pasos ya se fueron consolidando progresivamente en DynamoDB, el botón final solo envía un disparador de cierre liviano.

### 2.2. Control de Versión Optimista (OCC)
- El front **envía la versión esperada** en cada `PATCH /api/audits/{id}` (o `PUT`, soportado como alias).
- El backend valida atómicamente que la versión en DynamoDB coincida con la que el móvil cree tener antes de actualizar e incrementar.
- **Riesgo de requests desordenadas (*Out-of-order*): CERO.** Si la petición del paso 3 se retrasa en la red y llega después de la del paso 4, el backend la rechazará con **`409 Conflict`**, impidiendo que datos viejos sobreescriban datos nuevos.

### 2.3. ¿Hace falta un Header `Idempotency-Key`?
**No.** El `audit_id` generado por el móvil (UUIDv4) actúa como clave de idempotencia natural para la creación y subidas. El `PATCH` con versión y la máquina de estados del backend previenen duplicados sin necesidad de headers adicionales.

---

## 3. 🔄 El Nuevo Ciclo de Vida: Las 4 Fases

```
[Móvil: Inicia Flujo] ─────────► Fase 1: POST /api/audits (crea borrador, versión 1)
         │
[Toma foto en step]   ─────────► Fase 2: POST /api/uploads + PUT S3 (en background)
         │
[Responde o edita]    ─────────► Fase 3: PATCH /api/audits/{id} (envía SOLO deltas pendientes)
         │
[Pulsa "Finalizar"]   ─────────► Fase 4: POST /api/audits/{id}/submit (payload vacío {}, SQS)
```

---

### FASE 1: Creación Temprana del Borrador
Apenas el auditor selecciona el proyecto, facility y flow y presiona *"Comenzar Auditoría"*, el móvil genera el ID y registra el borrador.

- **Endpoint:** `POST /api/audits`
- **Generación de ID en Front:** `audit_${uuidv4()}_${Date.now()}`
- **Payload:**
```json
{
  "id": "audit_ac4ce4cf-d29d-4b9f-94e0-a9a2efa466c3_20260120220344",
  "flow_id": "flow_curb_ramps_v2_20260106102320",
  "flow_version": 1,
  "project_id": "proj_2cc5ad51-5cee-47fb-a2ed-e3595652b1ac_20251220214508",
  "facility_id": "fac_6d56a228-2258-4060-bc91-2792c43ee665",
  "status": "audit_in_progress",
  "answers": []
}
```
- **Respuesta (201 Created):**
```json
{
  "id": "audit_ac4ce4cf-d29d-4b9f-94e0-a9a2efa466c3_20260120220344",
  "status": "audit_in_progress",
  "version": 1
}
```
> 📌 **En Front:** Se inicializa `localVersion = 1`.  
> ⚠️ **En Backend:** En este estado **NO** se envía mensaje a SQS. Solo se persiste el registro para asegurar ownership y permitir subida de fotos.

---

### FASE 2: Subida de Fotos On-the-Fly (Background Queue)
Cada vez que el auditor toma una foto en un paso de tipo Formulario (`Form`):
1. La foto se almacena en el teléfono y se encola en background.
2. El front solicita la URL prefirmada pasando el `audit_id` y `step_id`.
3. Sube el binario directo a S3 vía `PUT`.
4. Actualiza la respuesta local con la URL definitiva de S3 (`s3://...`) y marca el paso como `pending` para sincronizar la URL.

- **Paso 2.1: Pedir URL Prefirmada**
  - **Endpoint:** `POST /api/uploads`
  - **Payload:**
  ```json
  {
    "audit_id": "audit_ac4ce4cf-d29d-4b9f-94e0-a9a2efa466c3_20260120220344",
    "files": [
      {
        "name": "photo_01.jpg",
        "step_id": "F03",
        "content_type": "image/jpeg"
      }
    ]
  }
  ```
  - **Respuesta:**
  ```json
  {
    "audit_id": "audit_ac4ce4cf-d29d-4b9f-94e0-a9a2efa466c3_20260120220344",
    "urls": [
      {
        "file_name": "photo_01.jpg",
        "upload_url": "https://kma-audit-bucket.s3.us-east-2.amazonaws.com/audit_.../F03/photo_01.jpg?X-Amz-...",
        "file_url": "s3://kma-audit-bucket/audit_ac4ce4cf-d29d-4b9f-94e0-a9a2efa466c3_20260120220344/F03/photo_01.jpg"
      }
    ]
  }
  ```

- **Paso 2.2: Subida a S3**
  - `PUT {upload_url}` con el binario de la imagen (`Content-Type: image/jpeg`).

> 🚀 **Ventaja para el usuario:** El auditor sigue contestando preguntas mientras la foto se sube en segundo plano. Al llegar al final, no hay fotos pendientes por subir.

---

### FASE 3: Auto-Save de Deltas con Merge (`PATCH /api/audits/{id}`)
A medida que el auditor responde o edita pasos, el móvil envía **únicamente los pasos que cambiaron o se agregaron** desde la última sincronización (`pending`). Se utiliza `PATCH` porque representa semánticamente una modificación parcial (delitas), aunque el backend aceptará también `PUT` como alias retrocompatible.

- **Endpoint:** `PATCH /api/audits/{id}` (o `PUT /api/audits/{id}`)
- **Payload (Ejemplo: Solo viaja el paso F03 completado recientemente):**
```json
{
  "version": 1,
  "answers": [
    {
      "step_id": "F03",
      "type": "Form",
      "values": {
        "measurements": 3.2,
        "notes": "Cross slope exceeds limit",
        "photos": [
          "s3://kma-audit-bucket/audit_ac4ce4cf-d29d-4b9f-94e0-a9a2efa466c3_20260120220344/F03/photo_01.jpg"
        ]
      }
    }
  ],
  "status": "audit_in_progress"
}
```

#### Comportamiento del Backend (Merge / Upsert):
1. Obtiene la auditoría de DynamoDB y valida que su versión actual coincida con la que envió el cliente (`version == 1`).
2. **Merge por `step_id`:**
   - Si `F03` ya existía en la lista de DynamoDB (porque el auditor lo corrigió) $\rightarrow$ reemplaza los valores de `F03`.
   - Si `F03` es un paso nuevo $\rightarrow$ lo concatena a la lista existente.
3. Guarda el documento consolidado con `version = 2`.
4. **NO publica en SQS** (porque sigue en `audit_in_progress`).

#### Respuesta Exitosa (200 OK):
```json
{
  "id": "audit_ac4ce4cf-d29d-4b9f-94e0-a9a2efa466c3_20260120220344",
  "version": 2,
  "status": "audit_in_progress",
  "updated_at": "2026-01-20T22:15:30Z"
}
```
> 📌 **En Front:** Los pasos enviados pasan de `pending` a `synced` y se actualiza `localVersion = 2`.

#### Respuesta ante Conflicto / Desincronización (409 Conflict):
```json
{
  "error": "version_conflict",
  "message": "Audit has a newer version on server",
  "server_version": 2
}
```

#### ❓ ¿Qué hace el móvil ante un 409 Conflict o Timeout?
1. Si la petición A (paso 2) sufrió un timeout en la red pero el backend sí la guardó (backend pasó a `v2`).
2. El auditor mientras tanto completó el paso 3 y 4.
3. El móvil intenta enviar los pasos pendientes con `version: 1`.
4. El backend responde `409 Conflict (server_version: 2)`.
5. **Acción del móvil:** Actualiza su puntero local `localVersion = 2` y **reintenta enviar sus pasos pendientes con `version: 2`**. Cero pérdida y total consistencia.

---

### FASE 4: Finalización y Envío (`POST /api/audits/{id}/submit`)
Cuando el auditor llega a la pantalla de resumen y hace clic en *"Completar y Enviar"*:

- **Comprobaciones previas en la app:**
  1. ¿Todas las preguntas requeridas están contestadas?
  2. ¿La cola de subida de fotos está en 0 (todas subidas a S3)?
  3. ¿Se sincronizaron todos los pasos pendientes? (Si queda alguno en `pending`, se sincroniza antes del submit).
- **Endpoint:** `POST /api/audits/{id}/submit`
- **Headers:** `Authorization: Bearer <token>`
- **Payload:** `{}` (**Vacío**. No viaja ninguna respuesta, el backend ya tiene la auditoría consolidada en DynamoDB).
- **Respuesta (200 OK):**
```json
{
  "id": "audit_ac4ce4cf-d29d-4b9f-94e0-a9a2efa466c3_20260120220344",
  "status": "draft_report_pending_review",
  "message": "Audit submitted successfully for enrichment"
}
```
> 🎯 **Comportamiento del Backend:**
> 1. Valida la transición: `audit_in_progress` $\rightarrow$ `draft_report_pending_review`.
> 2. **Publica el mensaje en SQS:** Se despierta la lambda `audit-enrichment-processor` para generar los hallazgos en `audits-review`.

---

## 4. 🛠️ Guía Práctica de Implementación en React Native

### 4.1. Estructura del Estado con Flag de Sync
```ts
// types.ts
export interface LocalAnswer {
  step_id: string;
  type: 'Question' | 'Select' | 'Form';
  answer?: string;
  values?: {
    measurements?: number;
    notes?: string;
    photos?: string[];
    quantity?: number;
    location?: string;
  };
  syncStatus: 'pending' | 'synced'; // Permite trackear qué falta enviar
}

export interface LocalAudit {
  id: string;
  flowId: string;
  flowVersion: number;
  projectId: string;
  facilityId: string;
  version: number; // Versión sincronizada con backend
  status: 'audit_in_progress' | 'draft_report_pending_review';
  answers: Record<string, LocalAnswer>; // step_id -> LocalAnswer
  pendingPhotos: Array<{
    id: string;
    stepId: string;
    localUri: string;
    fileName: string;
    isUploaded: boolean;
    s3Url?: string;
  }>;
}
```

### 4.2. Servicio de Auto-Save por Deltas y Cola Secuencial
```ts
import NetInfo from '@react-native-community/netinfo';
import debounce from 'lodash.debounce';
import { api } from '../services/api';
import { useAuditStore } from '../store/useAuditStore';

let isSaving = false;
let hasPendingChanges = false;

export const syncPendingDeltas = async (auditId: string) => {
  const netState = await NetInfo.fetch();
  if (!netState.isConnected) return; // Si no hay red, queda en local

  if (isSaving) {
    hasPendingChanges = true;
    return;
  }

  const { version, getPendingAnswers, markAnswersAsSynced, setServerVersion } = useAuditStore.getState();
  const pending = getPendingAnswers(); // Solo respuestas con syncStatus === 'pending'

  if (pending.length === 0) return; // Nada que sincronizar

  isSaving = true;
  hasPendingChanges = false;

  // Preparamos payload sin el campo interno syncStatus
  const payloadAnswers = pending.map(({ syncStatus, ...rest }) => rest);

  try {
    const response = await api.patch(`/api/audits/${auditId}`, {
      version: version,
      answers: payloadAnswers, // Solo deltas pendientes
      status: 'audit_in_progress'
    });

    // Éxito: actualizamos versión del server y marcamos los pasos enviados como synced
    setServerVersion(response.data.version);
    markAnswersAsSynced(pending.map(p => p.step_id));
  } catch (error: any) {
    if (error.response?.status === 409) {
      // Conflicto de versión (ej. timeout de petición previa que sí guardó)
      const serverVersion = error.response.data.server_version;
      setServerVersion(serverVersion);
      hasPendingChanges = true; // Reintentar de inmediato con la versión correcta
    } else {
      console.warn('Error en auto-save delta, reintentará luego:', error);
    }
  } finally {
    isSaving = false;
    if (hasPendingChanges) {
      syncPendingDeltas(auditId);
    }
  }
};

// Debounce de 2 segundos para disparar la sincronización tras editar
export const debouncedSync = debounce((auditId: string) => {
  syncPendingDeltas(auditId);
}, 2000);
```

### 4.3. Modificación de un Paso Anterior
Si el auditor vuelve al Paso 2 y cambia un dato:
```ts
function onUpdateAnswer(stepId: string, newValues: any) {
  // 1. Guarda en Zustand/MMKV local
  // 2. Establece syncStatus: 'pending' para ese step_id
  store.updateStepAnswer(stepId, newValues);
  
  // 3. Dispara sincronización debounced
  debouncedSync(auditId);
}
```

---

## 5. 📴 Comportamiento ante Pérdida de Conexión y Timeouts

| Escenario | Comportamiento del Sistema |
| :--- | :--- |
| **Conexión Normal** | Borrador creado ($\text{v1}$) $\rightarrow$ Fotos subidas en background $\rightarrow$ Deltas sincronizados al vuelo ($\text{v2} \rightarrow \text{v3}$) $\rightarrow$ Submit final con payload vacío `{}`. |
| **Timeout en respuesta (Server guardó $\text{v2}$ pero móvil no recibió el 200)** | El móvil sigue en $\text{v1}$. En el siguiente guardado envía los pasos pendientes con $\text{v1}$. Server responde `409 (server_version: 2)`. El móvil actualiza su puntero a $\text{v2}$ y reenvía sus deltas con $\text{v2}$. **No hay pérdida ni reenvío de respuestas viejas ya guardadas.** |
| **Corte de red total (En vuelo hacia el server)** | La petición no llegó al servidor. Server sigue en $\text{v1}$. El móvil sigue respondiendo offline (pasos acumulan `syncStatus = 'pending'`). Al volver la señal, móvil envía todos los pasos acumulados en `pending` con $\text{v1}$. Server hace el merge y responde $\text{v2}$. |
| **100% Offline (Toda la inspección sin señal)** | El auditor completa la auditoría de inicio a fin en local. Al conectarse a WiFi/4G, se ejecuta la secuencia ordenada: <br>1. `POST /api/audits` (crea en server $\text{v1}$).<br>2. Sube todas las fotos pendientes a S3.<br>3. `PATCH /api/audits/{id}` enviando todos los pasos pendientes acumulados ($\text{v2}$).<br>4. `POST /api/audits/{id}/submit` con `{}` (pasa a revisión y encola en SQS). |

---

## 6. 🏗️ Cambios Requeridos en Backend (`kma-backend`)

| Archivo / Componente | Modificación |
| :--- | :--- |
| `lambdas/audits/internal/create-audit/handler.go` | Permitir `status = "audit_in_progress"` como estado válido por defecto si viene del móvil. Establecer `version = 1`. |
| `lambdas/audits/internal/create-audit/usecase.go` | **Condicionar SQS:** Solo llamar a `uc.publisher.Publish` si el status es `draft_report_pending_review`. Si es `audit_in_progress`, no publicar. |
| `lambdas/audits/internal/shared/types/audit.go` | Flexibilizar validación `validate:"required"` en `Answers` al crear la auditoría para aceptar array vacío inicial `[]`. |
| `lambdas/audits/internal/update-audit/usecase.go` | **Lógica de Merge & OCC:**<br>1. Validar versión: Si `audit.Version > 0` y no coincide con DynamoDB, retornar `ErrVersionConflict` (`409`).<br>2. **Merge de respuestas:** Fusionar `existingAudit.Answers` con `incoming.Answers` usando `step_id` como clave (reemplazar si existe, concatenar si es nuevo).<br>3. Desactivar publicación a SQS mientras el estado sea `audit_in_progress`. |
| `lambdas/audits/internal/update-audit/handler.go` | Manejar `ErrVersionConflict` respondiendo `HTTP 409 Conflict` con `{ "error": "version_conflict", "server_version": currentVersion }`. |
| `lambdas/audits/internal/submit-audit/` **(NUEVO)** | Crear nuevo caso de uso y handler para `POST /api/audits/{id}/submit`. Valida estado, actualiza a `draft_report_pending_review` y publica en SQS. No requiere body. |
| `lambdas/audits/cmd/bootstrap/routes.go` | Registrar ruta: `case "POST /api/audits/{id}/submit": return h.submitHandler.Handle(ctx, event)`. |
| `lambdas/uploads/` | **Sin cambios requeridos.** Ya valida pertenencia contra la tabla DynamoDB `audits`. Al existir la auditoría desde la Fase 1, autoriza las subidas sin problemas. |

---

## 7. 📋 Resumen de Endpoints para Front

| Método | Endpoint | Cuándo se llama | Payload principal | Respuesta Clave | SQS Trigger |
| :--- | :--- | :--- | :--- | :--- | :---: |
| `POST` | `/api/audits` | Al pulsar "Comenzar" | ID del móvil, flow, facility, project, `status: audit_in_progress` | `201 Created` (`version: 1`) | ❌ No |
| `POST` | `/api/uploads` | Al tomar cada foto | `audit_id`, `step_id`, `files: [{ name, content_type }]` | `200 OK` (`upload_url`, `file_url`) | ❌ No |
| `PATCH` | `/api/audits/{id}` | Auto-save (cada 2 seg) | `version: N`, `answers: [solo deltas pendientes]`, `status: audit_in_progress` | `200 OK` (`version: N+1`) ó `409 Conflict` (`server_version`) | ❌ No |
| `POST` | `/api/audits/{id}/submit` | Al pulsar "Finalizar" | `{}` (vacío) | `200 OK` (`draft_report_pending_review`) | ✅ **Sí** |
