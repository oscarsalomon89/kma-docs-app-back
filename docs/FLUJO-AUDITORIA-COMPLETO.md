# 📋 Flujo Completo: Creación de Auditoría hasta Exportación de PDF

**Última actualización:** Enero 2026  
**Versión:** 2.0  
**Ejemplo Real:** `audit_ac4ce4cf-d29d-4b9f-94e0-a9a2efa466c3_20260120220344`  
**Flow:** `flow_curb_ramps_v2_20260106102320` (Curb Ramps)

---

## 🎯 Resumen Ejecutivo

Este documento describe el flujo completo desde que se crea una auditoría en el sistema hasta que se exporta el PDF final, incluyendo todos los estados, transiciones, lambdas involucradas y operaciones de base de datos.

**📌 Nota:** Este documento usa valores reales de la auditoría `audit_ac4ce4cf-d29d-4b9f-94e0-a9a2efa466c3_20260120220344` para facilitar la comprensión del flujo.

**⚠️ IMPORTANTE:** Esta auditoría contiene un **ERROR** documentado en la sección de agrupación (PASO 4): el finding "B22" debería ser "CR-B22" y estar agrupado con los otros barriers.

> [!WARNING]
> **Nota de Vigencia sobre Ingesta (V1 vs V2):**  
> El **Paso 1 (Creación de Auditoría)** de este documento refleja el envío monolítico original (V1). Para el ciclo vigente acordado con Mobile (creación temprana de borrador, uploads en background, guardado progresivo por deltas y submit liviano), consultar [RESPUESTAS-BACKEND-A-MOBILE.md](./RESPUESTAS-BACKEND-A-MOBILE.md) y [RFC-AUDITS-V2-MOBILE-SYNC.md](./RFC-AUDITS-V2-MOBILE-SYNC.md).  
> A partir del **Paso 2 (Enriquecimiento, Revisión de QC y Generación de PDF)**, la arquitectura y los eventos descritos en este documento se mantienen plenamente vigentes.


---

## 📊 Diagrama de Estados

```
┌─────────────────────┐
│ audit_in_progress   │ ← Estado inicial (mobile)
└──────────┬──────────┘
           │ EventSubmitAudit
           ▼
┌──────────────────────────────┐
│ draft_report_pending_review  │ ← Auditoría creada, esperando enriquecimiento
└──────────┬───────────────────┘
           │ EventSendForReview / EventOpenReview
           ▼
┌──────────────────────┐
│ draft_report_in_review│ ← QC revisando y editando
└──────────┬───────────┘
           │ EventCompleteReview
           ▼
┌──────────────────────────────┐
│ final_report_sent_to_client  │ ← PDF generado y enviado
└──────────┬───────────────────┘
           │ EventConfirmReceived
           ▼
┌──────────────┐
│  completed   │ ← Estado final
└──────────────┘
```

---

## 🔄 Flujo Detallado Paso a Paso

### **PASO 1: Creación de Auditoría** 
**Endpoint:** `POST /api/audits`  
**Lambda:** `audits`  
**Handler:** `create-audit`

#### 1.1. Request desde Frontend (Ejemplo Real)

**Auditoría:** `audit_ac4ce4cf-d29d-4b9f-94e0-a9a2efa466c3_20260120220344`

```json
{
  "flow_id": "flow_curb_ramps_v2_20260106102320",
  "flow_version": 1,
  "project_id": "proj_2cc5ad51-5cee-47fb-a2ed-e3595652b1ac_20251220214508",
  "facility_id": "fac_6d56a228-2258-4060-bc91-2792c43ee665",
  "answers": [
    {
      "step_id": "CR-B03",
      "type": "Question",
      "answer": "NO"
    },
    {
      "step_id": "F03",
      "type": "Form",
      "values": {
        "photos": ["s3://kma-audit-bucket/.../F03/photo.jpg"]
      }
    },
    {
      "step_id": "CR-B04",
      "type": "Question",
      "answer": "NO"
    },
    {
      "step_id": "F04",
      "type": "Form",
      "values": {
        "photos": ["s3://kma-audit-bucket/.../F04/photo.jpg"]
      }
    },
    {
      "step_id": "CR-B08",
      "type": "Question",
      "answer": "NO"
    },
    {
      "step_id": "CR-B20",
      "type": "Question",
      "answer": "NO"
    },
    {
      "step_id": "CR-B21",
      "type": "Question",
      "answer": "NO"
    },
    {
      "step_id": "CR-B22",
      "type": "Select",
      "answer": "ISLAND WITH CURB RAMPS"  // ⚠️ Esta respuesta causa el error
    },
    {
      "step_id": "CR-B24",
      "type": "Question",
      "answer": "NO"
    },
    {
      "step_id": "CR-QUANTITY-01",
      "type": "Form",
      "values": {
        "quantity": 6  // Shared quantity aplicado a múltiples barriers
      }
    }
  ]
}
```

#### 1.2. Procesamiento

1. **Validar request:**
   - Verificar que `flow_id` y `flow_version` existan
   - Validar estructura de `answers`

2. **Crear registro en tabla `audits`:**
   ```json
   {
     "id": "audit_ac4ce4cf-d29d-4b9f-94e0-a9a2efa466c3_20260120220344",
     "flow_id": "flow_curb_ramps_v2_20260106102320",
     "flow_version": 1,
     "project_id": "proj_2cc5ad51-5cee-47fb-a2ed-e3595652b1ac_20251220214508",
     "facility_id": "fac_6d56a228-2258-4060-bc91-2792c43ee665",
     "status": "audit_in_progress",
     "answers": [...],  // Array completo de respuestas
     "created_at": "2026-01-20T22:03:51Z",
     "updated_at": "2026-01-20T22:03:51Z"
   }
   ```

3. **Transición de estado:**
   - `audit_in_progress` → `draft_report_pending_review`
   - Actualiza `audits.status`

4. **Enviar mensaje SQS:**
   - Queue: `audit-enrichment-queue`
   - Mensaje:
     ```json
     {
       "audit_id": "audit_ac4ce4cf-d29d-4b9f-94e0-a9a2efa466c3_20260120220344",
       "user_id": "..."
     }
     ```

#### 1.3. Resultado
- ✅ Tabla `audits` creada con estado `draft_report_pending_review`
- ✅ Mensaje SQS enviado para procesamiento asíncrono
- ✅ Respuesta HTTP 201 con `audit_id`

---

### **PASO 2: Enriquecimiento de Findings**
**Lambda:** `audit-enrichment-processor`  
**Trigger:** SQS Message desde PASO 1

#### 2.1. Procesamiento

1. **Recibir mensaje SQS:**
   - Extrae `audit_id` del mensaje
   - Lee auditoría desde tabla `audits`

2. **Obtener flow:**
   - Lee desde tabla `flows` usando `flow_id` + `flow_version`
   - Flow: `flow_curb_ramps_v2_20260106102320` v1

3. **Procesar respuestas:**
   - Itera sobre `answers` del audit
   - Identifica findings (respuestas "NO" o Select con `barrier_id`)
   - Aplica Forms a findings pendientes
   - Maneja `shared_quantity` si existe metadata

4. **Enriquecer con catálogo:**
   - Extrae `barrier_id`s de findings
   - Busca en tabla `flows-catalog` usando `flow_id` + `barrier_id`
   - Obtiene: `barrier_statement`, `code_reference`, `mitigation_statement`, `mitigation_id`, `unit_cost`, `unit_of_measure`
   - Calcula `calculated_cost = quantity × unit_cost`

5. **Crear audits-review:**
   - Tabla: `audits-review`
   - Campos:
     - `audit_id` (PK)
     - `flow_id`, `flow_version`
     - `status`: `draft_report_pending_review`
     - `findings`: Array de findings enriquecidos
     - `created_at`, `updated_at`

#### 2.2. Estructura de Findings Enriquecidos (Ejemplo Real)

**Antes de agrupar (findings individuales):**

```json
[
  {
    "question_code": "CR-B03",
    "answer": "NO",
    "photos": [{"url": "s3://kma-audit-bucket/.../F03/photo.jpg", "include_in_report": true}],
    "barrier_statement": "The cross slope of the curb ramp run is >2%.",
    "code_reference": "ADAS 406.1, 405.3",
    "mitigation_statement": "Rebuild the curb ramp.",
    "mitigation_id": "CR-M02",
    "unit_cost": 1250.0,
    "unit_of_measure": "ea.",
    "quantity": 6,  // Del shared_quantity form
    "calculated_cost": 7500.0  // 6 × 1250
  },
  {
    "question_code": "CR-B04",
    "answer": "NO",
    "photos": [{"url": "s3://kma-audit-bucket/.../F04/photo.jpg", "include_in_report": true}],
    "barrier_statement": "The ground surface of the curb ramp run has abrupt changes in level.",
    "code_reference": "ADAS 406.1, 405.4",
    "mitigation_statement": "Rebuild the curb ramp.",
    "mitigation_id": "CR-M02",  // ← Mismo MitigationID que CR-B03
    "unit_cost": 1250.0,
    "unit_of_measure": "ea.",
    "quantity": 6,
    "calculated_cost": 7500.0
  },
  {
    "question_code": "CR-B08",
    "answer": "NO",
    "barrier_statement": "The counter slope of the adjoining gutter or road surface is >5%.",
    "code_reference": "ADAS 406.2",
    "mitigation_statement": "Rebuild the curb ramp.",
    "mitigation_id": "CR-M02",  // ← Mismo MitigationID
    "unit_cost": 1250.0,
    "unit_of_measure": "ea.",
    "quantity": 6,
    "calculated_cost": 7500.0
  },
  {
    "question_code": "CR-B20",
    "answer": "NO",
    "barrier_statement": "The clear space is not contained within the marked crossings.",
    "code_reference": "ADAS 406.6",
    "mitigation_statement": "Rebuild the curb ramp.",
    "mitigation_id": "CR-M02",  // ← Mismo MitigationID
    "unit_cost": 1250.0,
    "unit_of_measure": "ea.",
    "quantity": 6,
    "calculated_cost": 7500.0
  },
  {
    "question_code": "CR-B21",
    "answer": "NO",
    "barrier_statement": "The segments of curb are not contained within the marked crossings.",
    "code_reference": "ADAS 406.7",
    "mitigation_statement": "Rebuild the curb ramp.",
    "mitigation_id": "CR-M02",  // ← Mismo MitigationID
    "unit_cost": 1250.0,
    "unit_of_measure": "ea.",
    "quantity": 6,
    "calculated_cost": 7500.0
  },
  {
    "question_code": "B22",  // ❌ ERROR: debería ser "CR-B22"
    "answer": "ISLAND WITH CURB RAMPS",
    "barrier_statement": null,  // ❌ No se enriqueció porque "B22" no existe en catálogo
    "mitigation_id": null,  // ❌ No tiene mitigation_id
    "unit_cost": null,
    "calculated_cost": null
  },
  {
    "question_code": "CR-B24",
    "answer": "NO",
    "barrier_statement": "The curb ramp lacks a level ≥48\"x36\" area at the top of the curb ramp in the part of the island intersected by the crossings.",
    "code_reference": "ADAS 406.7",
    "mitigation_statement": "Rebuild the curb ramp.",
    "mitigation_id": "CR-M02",  // ← Mismo MitigationID
    "unit_cost": 1250.0,
    "unit_of_measure": "ea.",
    "quantity": 6,
    "calculated_cost": 7500.0
  }
]
```

**⚠️ PROBLEMA (Doble Dipping):** Si no se agrupa, el costo total sería:
- CR-B03: $7,500
- CR-B04: $7,500
- CR-B08: $7,500
- CR-B20: $7,500
- CR-B21: $7,500
- CR-B24: $7,500
- **Total CR-M02: $45,000** ❌ (INCORRECTO - se cuenta 6 veces la misma mitigación)

**⚠️ PROBLEMA ADICIONAL (Error B22):**
- Finding "B22" no se enriquece porque no existe en el catálogo
- Debería ser "CR-B22" y tener `mitigation_id = "CR-M02"`
- Debería estar agrupado con los otros 6 barriers

#### 2.3. Resultado
- ✅ Tabla `audits-review` creada con findings enriquecidos (aún sin agrupar)
- ✅ Estado: `draft_report_pending_review`
- ✅ Findings listos para revisión de QC
- ⚠️ **Nota:** La agrupación se aplicará en el PASO 4 cuando QC abra el review
- ❌ **Error:** Finding "B22" no se enriqueció correctamente (ver sección de error más abajo)

---

### **PASO 3: Enviar para Revisión (Opcional - puede saltarse)**
**Endpoint:** `POST /api/audits/{id}/send-for-review`  
**Lambda:** `audits`  
**Handler:** `send-for-review`

#### 3.1. Procesamiento
1. **Validar estado:** Debe estar en `draft_report_pending_review`
2. **Obtener findings enriquecidos:**
   - Lee desde `audits-review` usando `audit_id`
   - Valida que haya findings (si no hay, retorna error 422)
3. **Transición de estado:**
   - `draft_report_pending_review` → `draft_report_in_review`
   - Actualiza tabla `audits.status`
4. **Generar presigned URLs:**
   - Convierte URLs de S3 a presigned URLs para las fotos
5. **Respuesta:**
   - Retorna findings enriquecidos con presigned URLs

#### 3.2. Resultado
- ✅ Estado actualizado a `draft_report_in_review`
- ✅ Findings listos para mostrar en UI de review

**Nota:** Este paso es opcional. Si QC abre directamente el review (PASO 4), la transición se hace automáticamente.

---

### **PASO 4: Abrir Revisión (QC)**
**Endpoint:** `POST /api/audits-review/{id}/reviews` o `GET /api/audits-review/{id}`  
**Lambda:** `audits-review`  
**Handler:** `create-review` o `get-review`

#### 4.1. Procesamiento
1. **Validar estado:**
   - Debe estar en `draft_report_pending_review` o `draft_report_in_review`
   - Si está en `pending`, transiciona automáticamente a `in_review`

2. **Obtener review:**
   - Lee desde tabla `audits-review` usando `audit_id`

3. **Agrupar findings por MitigationID:**
   - **CRÍTICO:** Se aplica agrupación para evitar doble dipping
   - Findings con mismo `mitigation_id` se agrupan:
     - Se suman `quantity` y se recalcula `calculated_cost = quantity_total × unit_cost`
     - Se combinan `question_codes`, `barrier_statements`, `code_references`, `notes`
     - Se combinan todas las `photos`
   - Se guarda el review agrupado en `audits-review`

   **Ejemplo Real - Agrupación:**
   
   **ANTES (6 findings individuales con CR-M02 + 1 finding con error):**
   - CR-B03: $7,500
   - CR-B04: $7,500
   - CR-B08: $7,500
   - CR-B20: $7,500
   - CR-B21: $7,500
   - CR-B24: $7,500
   - B22: $0 (sin datos) ❌
   - **Total: $45,000** ❌ (INCORRECTO - doble dipping + error)
   
   **DESPUÉS (1 finding agrupado con CR-M02 + 1 finding con error):**
   ```json
   {
     "question_code": "CR-B03, CR-B04, CR-B08, CR-B20, CR-B21, CR-B24",
     "mitigation_id": "CR-M02",
     "quantity": 6,
     "unit_cost": 1250.0,
     "calculated_cost": 7500.0,  // ✅ Recalculado: 6 × 1250 = 7500
     "barrier_statement": "The cross slope of the curb ramp run is >2%.\nThe ground surface of the curb ramp run has abrupt changes in level.\nThe counter slope of the adjoining gutter or road surface is >5%.\nThe clear space is not contained within the marked crossings.\nThe segments of curb are not contained within the marked crossings.\nThe curb ramp lacks a level ≥48\"x36\" area at the top of the curb ramp in the part of the island intersected by the crossings.",
     "code_reference": "ADAS 406.1, 405.3, ADAS 406.1, 405.4, ADAS 406.2, ADAS 406.6, ADAS 406.7",
     "mitigation_statement": "Rebuild the curb ramp.",
     "photos": [
       // Todas las fotos de F03, F04, etc. combinadas
     ]
   }
   ```
   - **Total: $7,500** ✅ (CORRECTO - una sola mitigación)
   
   **Finding 2 (ERROR - no se agrupa):**
   ```json
   {
     "question_code": "B22",  // ❌ ERROR: debería ser "CR-B22"
     "answer": "ISLAND WITH CURB RAMPS",
     "mitigation_id": null,  // ❌ No tiene mitigation_id
     "quantity": null,
     "unit_cost": null,
     "calculated_cost": null
   }
   ```
   
   **Resultado Final (2 findings):**
   - Finding 1: CR-M02 (Rebuild the curb ramp) - $7,500 ✅
   - Finding 2: B22 (sin datos) - $0 ❌
   - **Total Auditoría: $7,500** ⚠️ (INCORRECTO - falta CR-B22)

#### 4.2. ❌ ERROR DETECTADO: "B22" en lugar de "CR-B22"

**Problema:**
- El finding tiene `question_code = "B22"` cuando debería ser `"CR-B22"`
- No se enriquece porque "B22" no existe en el catálogo
- No se agrupa porque no tiene `mitigation_id`
- Debería estar agrupado con los otros 6 barriers (CR-B03, CR-B04, CR-B08, CR-B20, CR-B21, CR-B24)

**Origen del Error:**

El error nace en el **flow** `lambdas/flows/docs/curb_ramps.json` línea 886:

```json
{
    "id": "CR-B22",
    "type": "Select",
    "text": "Select one of the following island conditions:",
    "options": [
        {
            "label": "Island with cut-through",
            "next": "CR-B23"
            // ❌ NO tiene barrier_id
        },
        {
            "label": "Island with curb ramps",  // ← Usuario seleccionó esto
            "next": "CR-B24"
            // ❌ NO tiene barrier_id
        },
        {
            "label": "Neither",
            "next": "F22",
            "barrier_id": "CR-B22"  // ✅ Esta opción SÍ tiene barrier_id
        }
    ],
    "barrier_id": "B22"  // ❌ ERROR: debería ser "CR-B22"
}
```

**Flujo del Error:**

1. Usuario selecciona: "Island with curb ramps" (opción sin `barrier_id`)
2. El enrichment processor busca: `stepToBarrier["CR-B22:ISLAND WITH CURB RAMPS"]` → no existe
3. Usa fallback: `stepToBarrier["CR-B22"]` → devuelve "B22" (del step) ❌
4. Resultado: `question_code = "B22"` en lugar de `"CR-B22"`

**Impacto:**

| Estado | Finding 1 | Finding 2 | Total |
|--------|-----------|-----------|-------|
| **Actual** | $7,500 (6 barriers) | $0 (B22 sin datos) | $7,500 |
| **Esperado** | $8,750 (7 barriers incluyendo CR-B22) | - | $8,750 |
| **Diferencia** | -$1,250 | - | **-$1,250** |

**Solución:**

Corregir el flow cambiando la línea 886:
```json
"barrier_id": "CR-B22"  // En lugar de "B22"
```

O agregar `barrier_id` a las opciones que no lo tienen.

**Cómo Debería Estar (Correcto):**

```json
{
  "question_code": "CR-B03, CR-B04, CR-B08, CR-B20, CR-B21, CR-B22, CR-B24",  // ← 7 barriers
  "mitigation_id": "CR-M02",
  "quantity": 6,
  "unit_cost": 1250.0,
  "calculated_cost": 7500.0,  // 6 × 1250 = 7500
  "barrier_statement": "...",
  "mitigation_statement": "Rebuild the curb ramp."
}
```

**Total Esperado:** $7,500 (si quantity=6 se aplica a los 7 barriers, o $8,750 si quantity=7)

#### 4.3. Resultado
- ✅ Findings agrupados por `mitigation_id`
- ✅ Costos recalculados correctamente (`quantity_total × unit_cost`)
- ✅ Estado: `draft_report_in_review`
- ❌ **Error:** Finding "B22" no se agrupa (debería ser "CR-B22")

---

### **PASO 5: Actualizar Finding (QC)**
**Endpoint:** `PUT /api/audits-review/{id}/findings/{question_code}`  
**Lambda:** `audits-review`  
**Handler:** `update-finding`

#### 5.1. Procesamiento
1. **Validar estado:** Debe estar en `draft_report_in_review`
2. **Actualizar finding:**
   - Actualiza `quantity`, `notes`, `photos` del finding específico
   - Preserva campos enriquecidos (`barrier_statement`, `mitigation_id`, etc.)
3. **Reagrupar findings:**
   - Después de actualizar, se reagrupa todo el review
   - Se recalcula `calculated_cost` basándose en `quantity_total × unit_cost`
4. **Invalidar PDF:** Borra `report_url` si existía
5. **Mantener estado:** Confirma estado como `draft_report_in_review`

#### 5.2. Resultado
- ✅ Finding actualizado
- ✅ Review reagrupado con costos recalculados
- ✅ PDF invalidado (debe regenerarse)

---

### **PASO 6: Completar Revisión y Generar PDF**
**Endpoint:** `POST /api/audits-review/{id}/complete`  
**Lambda:** `audits-review`  
**Handler:** `complete-review`

#### 6.1. Procesamiento
1. **Validar estado:** Debe estar en `draft_report_in_review`
2. **Obtener review:**
   - Lee desde `audits-review` usando `audit_id`
3. **Agrupar findings (verificación final):**
   - Aplica agrupación una vez más antes de generar PDF
   - Recalcula costos: `calculated_cost = quantity_total × unit_cost`
4. **Transición de estado:**
   - `draft_report_in_review` → `final_report_sent_to_client`
   - Actualiza `audits.status` y `audits-review.status`
5. **Enviar mensaje SQS:**
   - Queue: `reports-queue`
   - Mensaje:
     ```json
     {
       "audit_id": "audit_ac4ce4cf-d29d-4b9f-94e0-a9a2efa466c3_20260120220344",
       "user_id": "..."
     }
     ```

#### 6.2. Resultado
- ✅ Estado actualizado a `final_report_sent_to_client`
- ✅ Mensaje SQS enviado para generación de PDF
- ✅ Review final agrupado y listo para PDF

---

### **PASO 7: Generar PDF**
**Lambda:** `reports-worker`  
**Trigger:** SQS Message desde PASO 6

#### 7.1. Procesamiento
1. **Recibir mensaje SQS:**
   - Extrae `audit_id` del mensaje
2. **Obtener review:**
   - Lee desde `audits-review` usando `audit_id`
   - Findings ya están agrupados y con costos correctos
3. **Generar PDF:**
   - Usa biblioteca de generación de PDFs
   - Incluye todos los findings agrupados
   - Calcula totales basándose en `calculated_cost` de cada finding agrupado
4. **Subir a S3:**
   - Bucket: `kma-audit-bucket`
   - Key: `reports/{audit_id}/report.pdf`
5. **Actualizar auditoría:**
   - Actualiza `audits.report_url` con URL del PDF
   - Genera presigned URL para acceso temporal

#### 7.2. Estructura del PDF (Ejemplo Real)

**Findings en el PDF:**

| Finding | Question Codes | Mitigation | Quantity | Unit Cost | Calculated Cost |
|---------|----------------|------------|----------|-----------|-----------------|
| 1 | CR-B03, CR-B04, CR-B08, CR-B20, CR-B21, CR-B24 | Rebuild the curb ramp (CR-M02) | 6 | $1,250 | **$7,500** ✅ |
| 2 | B22 | (sin datos) | - | - | **$0** ❌ |

**Total en PDF:** $7,500 ⚠️ (INCORRECTO - debería incluir CR-B22)

**⚠️ NOTA:** El PDF refleja el error del finding "B22" que no se enriqueció ni agrupó correctamente.

#### 7.3. Resultado
- ✅ PDF generado y subido a S3
- ✅ `audits.report_url` actualizado
- ✅ Presigned URL disponible para descarga
- ❌ **Error:** PDF incluye finding "B22" sin datos en lugar de "CR-B22" agrupado

---

## 📊 Resumen de la Auditoría Real

### Datos de la Auditoría

- **Audit ID:** `audit_ac4ce4cf-d29d-4b9f-94e0-a9a2efa466c3_20260120220344`
- **Flow:** `flow_curb_ramps_v2_20260106102320` v1
- **Status:** `final_report_sent_to_client`
- **Created:** 2026-01-20T22:03:51Z
- **Updated:** 2026-01-20T22:04:49Z

### Findings Finales

**Finding 1 (Agrupado - CORRECTO):**
- Question codes: `CR-B03, CR-B04, CR-B08, CR-B20, CR-B21, CR-B24`
- Mitigation ID: `CR-M02`
- Quantity: 6
- Unit cost: $1,250
- Calculated cost: $7,500 ✅ (6 × 1,250)

**Finding 2 (ERROR):**
- Question code: `B22` ❌ (debería ser `CR-B22`)
- Mitigation ID: `null` ❌
- Quantity: `null` ❌
- Unit cost: `null` ❌
- Calculated cost: `null` ❌

### Valores Totales

| Estado | Finding 1 | Finding 2 | Total |
|--------|-----------|-----------|-------|
| **Actual** | $7,500 (6 barriers) | $0 (B22 sin datos) | **$7,500** ⚠️ |
| **Esperado** | $7,500 o $8,750 (7 barriers incluyendo CR-B22) | - | **$7,500 o $8,750** ✅ |

### Error Detectado

**Problema:** Finding "B22" debería ser "CR-B22" y estar agrupado con Finding 1.

**Origen:** Error en flow `curb_ramps.json` línea 886: `"barrier_id": "B22"` debería ser `"barrier_id": "CR-B22"`.

**Impacto:** 
- Finding no se enriquece (no encuentra datos en catálogo)
- Finding no se agrupa (no tiene `mitigation_id`)
- Costo potencial perdido: $1,250 (si quantity=1) o incluido en el total si quantity=6 se aplica a 7 barriers

---

## 🔑 Puntos Clave del Flujo

1. **Enriquecimiento:** Los findings se enriquecen con datos del catálogo en el `audit-enrichment-processor`
2. **Agrupación:** Los findings se agrupan por `mitigation_id` cuando QC abre el review (PASO 4)
3. **Recálculo:** Los costos se recalculan como `quantity_total × unit_cost` después de agrupar
4. **Doble Dipping:** Se evita sumando quantities y recalculando costos en lugar de sumar `calculated_cost` existentes
5. **Error B22:** El finding "B22" no se enriquece ni agrupa debido a un error en el flow

---

## 📝 Notas Finales

- Este documento usa valores reales de la auditoría `audit_ac4ce4cf-d29d-4b9f-94e0-a9a2efa466c3_20260120220344`
- La auditoría contiene un error documentado: finding "B22" debería ser "CR-B22"
- El error nace en el flow `curb_ramps.json` y afecta el enriquecimiento y agrupación
- La solución es corregir el flow cambiando `"barrier_id": "B22"` a `"barrier_id": "CR-B22"`

---

**Fin del documento**
