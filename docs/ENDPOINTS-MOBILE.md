# Endpoints utilizados por la app mobile (KMA)

Listado de todas las llamadas de red que hace la app hoy, para compartir con backend.

## 1. Backend KMA

Base URL: `EXPO_PUBLIC_API_BASE_URL`. Todas las llamadas pasan por el cliente `http` (`src/core/http/http.ts`) y llevan `Authorization: Bearer <idToken>` (Cognito).

| Método | Endpoint | Uso | Código |
|---|---|---|---|
| `GET` | `/flows` | Catálogo de flows con sus steps | `src/features/selector/data/flow.repo.http.ts` |
| `GET` | `/projects` | Lista de proyectos | `src/features/projects/data/project.repo.http.ts` |
| `GET` | `/projects/{id}` | Detalle de proyecto | `src/features/projects/data/project.repo.http.ts` |
| `GET` | `/projects/{projectId}/facilities` | Facilities de un proyecto | `src/features/facility/data/facility.repo.http.ts` |
| `GET` | `/projects/{projectId}/facilities/{facilityId}` | Detalle de facility | `src/features/facility/data/facility.repo.http.ts` |
| `POST` | `/uploads` | Presign de fotos (devuelve también el `audit_id`) | `src/features/flow-runner/application/usecases.ts` |
| `POST` | `/audits` | Crea la auditoría final | `src/features/flow-runner/application/usecases.ts` |

### Query params

- `GET /projects`: `status`, `search`, `limit`, `cursor`, `sortBy` (`created_at` \| `updated_at` \| `name`), `sortOrder` (`asc` \| `desc`).
- `GET /projects/{projectId}/facilities`: `limit`, `cursor`, `status` (`ACTIVE` \| `ARCHIVED`), `search`.

### Bodies

`POST /uploads`

```json
{ "files": [{ "name": "<nombre-archivo>", "step_id": "<step>" }] }
```

Respuesta esperada: `{ "audit_id": "...", "urls": [{ "file_name", "upload_url", "file_url" }] }`.
Se llama también cuando la auditoría no tiene fotos (con `files: []`), porque la app usa este endpoint para obtener el `audit_id`. **No se reintenta** automáticamente.

`POST /audits`

```json
{
  "id": "<audit_id devuelto por /uploads>",
  "flow_id": "...",
  "project_id": "...",
  "facility_id": "...",
  "answers": [ ... ],
  "flow_version": 1
}
```

No se envía `status`.

## 2. Servicios externos (no son el backend KMA)

| Método | Destino | Uso |
|---|---|---|
| `POST` | Cognito `cognito-idp` (`EXPO_PUBLIC_COGNITO_BASE_URL`), `X-Amz-Target: AWSCognitoIdentityProviderService.InitiateAuth`, `USER_PASSWORD_AUTH` | Login |
| `POST` | Cognito, `InitiateAuth` con `REFRESH_TOKEN_AUTH` | Refresh de token |
| `POST` | Cognito, `GlobalSignOut` (con `AccessToken`) | Logout |
| `PUT` | `upload_url` presignada (S3), sin `Authorization` | Subida de cada foto |
| `GET` | URLs de imágenes de los steps (descarga con `FileSystem.downloadAsync`) | Cache local de imágenes |

## 3. Cuándo se llama cada uno

- **`GET` de catálogo** (`/flows`, `/projects`, `/projects/.../facilities`): sync de catálogo al abrir la app (foreground), al recuperar red y al iniciar sesión. La UI lee de SQLite (offline-first).
- **`POST /uploads` y `POST /audits`**: solo desde el outbox, cuando hay una auditoría encolada y red disponible. Orden: `/uploads` → `PUT` a S3 por cada foto → `/audits`.

## 4. Comportamiento de red del cliente

- **Reintentos**: 429, 5xx, timeouts y errores de red, con backoff exponencial. Nunca se reintentan otros 4xx. `POST /uploads` no se reintenta.
- **401**: se intenta renovar el token una vez y se repite el request. Si Cognito rechaza el refresh token se cierra sesión; si el refresh falla por red, no se cierra sesión.
- **Outbox** (auditorías): ante errores recuperables se reintenta con backoff persistido (base 60 s, tope 30 min, máximo 15 intentos). Un 4xx no recuperable deja la auditoría como fallida y visible para el auditor; no se descarta.
- **Timeout** del cliente: 20 s.

## 5. Fuera de este listado

La app **no** usa hoy: `GET /audits`, `GET /audits/{id}`, `PATCH`/`PUT`/`DELETE` sobre auditorías, ni `POST /audits/{id}/submit`. Son de fases futuras del plan de implementación.
