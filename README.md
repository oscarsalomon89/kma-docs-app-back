# KMA — Hub de Contratos y Especificaciones Backend / Frontend

Bienvenido al repositorio central de **contratos de integración, definiciones de API y arquitectura de sincronización** entre **Backend (Go Serverless)** y **Frontend (Mobile React Native / Web QC)** de la plataforma **KMA**.

---

## 📱 ¿De qué trata la aplicación KMA?

**KMA** es una plataforma integral para la **gestión, ejecución en terreno y control de calidad (QC) de auditorías de accesibilidad e infraestructura física** (cumplimiento normativo ADA y estándares afines en instalaciones como rampas, vías de acceso, áreas de reunión, puertas, estacionamientos, etc.).

El ecosistema KMA está compuesto por tres pilares:

```
┌────────────────────────────────────────────────────────────────────────┐
│                              Ecosistema KMA                            │
└────────────────────────────────────────────────────────────────────────┘
          │                                            │
          ▼                                            ▼
┌──────────────────────────────┐              ┌──────────────────────────────┐
│       KMA Mobile App         │              │         KMA Web App          │
│       (React Native)         │              │          (Frontend)          │
│ • Inspecciones en terreno    │              │ • Control de Calidad (QC)    │
│ • Flujo Offline-First        │              │ • Revisión de hallazgos      │
│ • Captura fotográfica        │              │ • Aprobación y exportación   │
│ • Sincronización progresiva  │              │ • Gestión de instalaciones   │
└──────────────┬───────────────┘              └──────────────┬───────────────┘
               │                                             │
               │            API Gateway + JWT                │
               └──────────────────────┬──────────────────────┘
                                      ▼
               ┌─────────────────────────────────────────────┐
               │              KMA Backend API                │
               │          (Go + AWS Serverless)              │
               │ • Lambdas (Auth, Audits, Uploads, Catalog)  │
               │ • DynamoDB (Audits, Flows, Facilities)      │
               │ • S3 (Fotos, Evidencias, Reportes)          │
               │ • SQS / EventBridge (Enrichment, Workers)   │
               │ • Motor de generación de reportes PDF       │
               └─────────────────────────────────────────────┘
```

1. **KMA Mobile (React Native)**: Aplicación para auditores en campo. Permite ejecutar formularios dinámicos con árboles de decisión ("flows"), tomar mediciones, capturar evidencias fotográficas y operar en entornos de conectividad nula o intermitente mediante un enfoque **Offline-First**.
2. **KMA Web**: Plataforma de administración y Control de Calidad (QC). Permite a los revisores validar las respuestas de los auditores, editar hallazgos, reclasificar barreras normativas y aprobar el reporte final para los clientes.
3. **KMA Backend (Go Serverless)**: Microservicios implementados en Go desplegados en AWS (Lambda, DynamoDB, API Gateway, S3 y SQS). Provee gestión de proyectos/instalaciones, catálogo de flows, ingesta idempotente de auditorías, enriquecimiento automático de barreras/mitigaciones y generación asíncrona de reportes PDF.

---

## 🤝 Decisiones y Definiciones entre Backend y Frontend

Las decisiones técnicas más importantes acordadas entre los equipos de Frontend (Mobile) y Backend están documentadas detalladamente en:
- [docs/RESPUESTAS-BACKEND-A-MOBILE.md](./docs/RESPUESTAS-BACKEND-A-MOBILE.md) — **Fuente de la Verdad oficial**: Contratos definitivos, DTOs y reglas implementadas en Backend.
- [docs/DECISIONES-BACKEND-MOBILE.md](./docs/DECISIONES-BACKEND-MOBILE.md) — Consultas críticas, casos de borde y requerimientos planteados desde Mobile.
- [docs/RFC-AUDITS-V2-MOBILE-SYNC.md](./docs/RFC-AUDITS-V2-MOBILE-SYNC.md) — Especificación técnica del protocolo de sincronización progresiva.

### Resumen de los Acuerdos Principales

#### 1. Ciclo de Vida de Auditorías V2 (Guardado Progresivo)
Para mitigar pérdidas de información y bloqueos por red móvil inestable en campo, el flujo de auditorías opera bajo un ciclo de **4 fases**:

| Fase | Acción / Endpoint | Responsabilidad y Contrato |
|---|---|---|
| **Fase 1: Creación temprana** | `POST /api/audits` | El móvil genera el `audit_id` (`audit_<uuid>_<timestamp>`) e inicia el borrador (`version: 1`). Backend valida con guarda condicional `attribute_not_exists(id)` en DynamoDB y responde `409 Conflict` si ya existía (Mobile lo interpreta como éxito de sync). |
| **Fase 2: Subida de fotos** | `POST /api/uploads` + `PUT S3` | El móvil solicita URLs prefirmadas a S3 enviando `audit_id` y lista de archivos. Backend devuelve URLs con `expires_at` (15 min). La subida ocurre en background durante la inspección. |
| **Fase 3: Guardado por Deltas** | `PUT` / `PATCH /api/audits/{id}` | El móvil envía **únicamente** los pasos modificados (`answers`) y la versión esperada para Control de Concurrencia Optimista (OCC). Si se descartan ramas en el árbol de preguntas, envía `deleted_step_ids: string[]`. |
| **Fase 4: Finalización / Submit** | `POST /api/audits/{id}/submit` | El móvil envía un payload vacío `{}`. Backend valida que el borrador esté completo, transiciona el estado a `draft_report_pending_review` y encola a SQS para enriquecimiento automático. |

#### 2. Máquina de Estados de una Auditoría
```
┌─────────────────────────┐
│    audit_in_progress    │  ◄── Creado por Mobile (borrador local y en servidor)
└────────────┬────────────┘
             │ EventSubmitAudit (Fase 4 - submit liviano)
             ▼
┌─────────────────────────┐
│ draft_report_pending_   │  ◄── Enriqueciendo hallazgos / Esperando QC
│ review                  │      (Aún editable por Mobile si fuera necesario)
└────────────┬────────────┘
             │ EventOpenReview (Revisor Web toma la auditoría)
             ▼
┌─────────────────────────┐
│  draft_report_in_review │  ◄── Revisor Web editando (Mobile bloqueado: 400/409)
└────────────┬────────────┘
             │ EventCompleteReview
             ▼
┌─────────────────────────┐
│ final_report_sent_to_   │  ◄── Reporte PDF generado y remitido
│ client                  │
└────────────┬────────────┘
             │ EventConfirmReceived
             ▼
┌─────────────────────────┐
│        completed        │  ◄── Auditoría archivada / cerrada
└─────────────────────────┘
```

#### 3. Reglas de Compatibilidad y Contratos
- **Evolución de `GET /flows`**: El agregado de propiedades opcionales (`order`, `required`, `multiline`) es seguro y no rompe versiones anteriores. La incorporación de nuevos tipos de pasos o tipos de campos de formulario debe coordinarse formalmente para evitar romper validadores locales de la app.
- **Nomenclatura en S3**: Llave única de archivos: `<audit_id>/<step_id>/<name>`, donde el nombre sigue el patrón `<step_id>_<field_id>_<id_local>.<ext>` garantizando idempotencia en los reintentos.
- **Formato Estándar de Errores**: Todos los endpoints responden con la estructura uniforme:
  ```json
  {
    "error": "conflict",
    "message": "Audit already exists with id: audit_...",
    "code": 409
  }
  ```

---

## 📚 Mapa de Documentación del Repositorio

La documentación vigente dentro del directorio `docs/` está organizada de la siguiente manera:

### 🔄 Sincronización y Contratos API (Núcleo)
- [**`docs/RESPUESTAS-BACKEND-A-MOBILE.md`**](./docs/RESPUESTAS-BACKEND-A-MOBILE.md): **⭐ Fuente de la Verdad (30/09/2026).** Contratos definitivos acordados: idempotencia, presigns con expiración, borrado de ramas con `deleted_step_ids`, OCC y mensajes de error.
- [**`docs/DECISIONES-BACKEND-MOBILE.md`**](./docs/DECISIONES-BACKEND-MOBILE.md): **Contexto y Casos de Borde (22/09/2026).** Análisis detallado de los problemas de terreno (red inestable, riesgo de pérdida de datos) que justifican cada decisión.
- [**`docs/RFC-AUDITS-V2-MOBILE-SYNC.md`**](./docs/RFC-AUDITS-V2-MOBILE-SYNC.md): Especificación técnica del protocolo de sincronización progresiva por deltas, control de versión y subidas en background.
- [**`docs/PROJECT-FACILITY-DETAIL.md`**](./docs/PROJECT-FACILITY-DETAIL.md): Especificación del contrato para `GET /api/projects/{id}/facilities/{facility_id}` con formato estandarizado `{ "status": "success", "data": { ... } }`.

### 📋 Flujos y Ciclo de Vida Completo
- [**`docs/FLUJO-AUDITORIA-COMPLETO.md`**](./docs/FLUJO-AUDITORIA-COMPLETO.md): Trazabilidad paso a paso con un ejemplo real desde la ingesta hasta el enriquecimiento, revisión de QC en Web y exportación del PDF final *(Nota: la sección de ingesta inicial fue superada por el ciclo V2 detallado en `RESPUESTAS-BACKEND-A-MOBILE.md`)*.

---

## 🛠️ Buenas Prácticas para Mantener los Contratos

1. **API-First**: Todo cambio en endpoints, DTOs o códigos de estado debe reflejarse en esta documentación y comunicarse entre equipos antes de impactar el código cliente.
2. **Compatibilidad Hacia Atrás**: El backend nunca debe eliminar campos ni alterar tipos de datos sin versionado explícito, preservando el funcionamiento de dispositivos móviles en versiones anteriores en producción.
3. **Validación de Idempotencia y Concurrencia**: Toda mutación en servidor debe respetar las guardas condicionales (`attribute_not_exists` y verificación de `version`).
