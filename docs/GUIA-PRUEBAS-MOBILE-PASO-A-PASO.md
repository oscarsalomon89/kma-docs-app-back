# 📱 Guía de Pruebas Paso a Paso: Flujo Móvil KMA

**Propósito:** Guía técnica detallada para el equipo Frontend Mobile / QA para probar el ciclo de vida completo de la aplicación, desde la autenticación en Cognito hasta la carga progresiva por partes, subida de fotos a Amazon S3 y envío final.

> 🔒 **Nota de Seguridad:** Todos los datos sensibles (contraseñas, Client ID y tokens) han sido enmascarados como marcadores de posición (`<...>`) para que este documento pueda compartirse de forma segura.

---

## 🛠️ 1. Configuración del Entorno y Variables

Configura las siguientes variables en el cliente HTTP (Postman, Bruno, cURL) o en el archivo `.env` de Expo / React Native:

```env
# URL Base del Backend API Gateway (¡Importante incluir /api!)
EXPO_PUBLIC_API_BASE_URL=https://0s46o3anuj.execute-api.us-east-1.amazonaws.com/api

# Configuración AWS Cognito (Dev)
EXPO_PUBLIC_COGNITO_BASE_URL=https://cognito-idp.us-east-1.amazonaws.com
EXPO_PUBLIC_COGNITO_USER_POOL_ID=us-east-1_4qaqKkAJu
EXPO_PUBLIC_COGNITO_CLIENT_ID=<COGNITO_CLIENT_ID>
EXPO_PUBLIC_AWS_REGION=us-east-1
```

### Variables de Prueba para esta sesión:
* **Usuario:** `pablocristo`
* **Password:** `<PASSWORD>`
* **Proyecto:** `proj-dev-001` (*"ADA Compliance 2026 - Philadelphia"*)
* **Facility:** `fac-dev-001` (*"Main Office"*)
* **Flow ID:** `flow_ramp_accessibility_verification_20251009190307` (*Flujo de Rampas*)
* **Flow Version:** `21`

---

## 🔐 2. Paso 1: Autenticación en Cognito (Login)

La app móvil se autentica directamente contra AWS Cognito usando el flujo `USER_PASSWORD_AUTH` (sin necesidad de pasar por backend).

* **Método:** `POST`
* **URL:** `https://cognito-idp.us-east-1.amazonaws.com/`
* **Headers:**
  ```http
  Content-Type: application/x-amz-json-1.1
  X-Amz-Target: AWSCognitoIdentityProviderService.InitiateAuth
  ```
* **Body (JSON):**
  ```json
  {
    "AuthFlow": "USER_PASSWORD_AUTH",
    "ClientId": "<COGNITO_CLIENT_ID>",
    "AuthParameters": {
      "USERNAME": "pablocristo",
      "PASSWORD": "<PASSWORD>"
    }
  }
  ```

* **Respuesta Exitosa (`200 OK`):**
  ```json
  {
    "AuthenticationResult": {
      "AccessToken": "<ACCESS_TOKEN>",
      "ExpiresIn": 3600,
      "IdToken": "<ID_TOKEN>",
      "RefreshToken": "<REFRESH_TOKEN>",
      "TokenType": "Bearer"
    },
    "ChallengeParameters": {}
  }
  ```

> 📌 **Regla General:** Para todas las siguientes llamadas al backend KMA, debes enviar el token en la cabecera:  
> `Authorization: Bearer <ID_TOKEN>`

---

## 📂 3. Paso 2: Sincronización de Catálogo (Offline-First)

Al iniciar sesión o abrir la aplicación, el móvil sincroniza los flujos y proyectos para almacenarlos localmente en SQLite.

### 2.1. Obtener Catálogo de Flujos
* **Método:** `GET`
* **Endpoint:** `{{EXPO_PUBLIC_API_BASE_URL}}/flows`
* **Respuesta (`200 OK`):** Lista de flujos disponibles (`Ramps`, `Restrooms`, `Parking`, etc.) con sus preguntas y opciones.

### 2.2. Obtener Lista de Proyectos
* **Método:** `GET`
* **Endpoint:** `{{EXPO_PUBLIC_API_BASE_URL}}/projects?status=ACTIVE`
* **Respuesta (`200 OK`):**
  ```json
  {
    "status": "success",
    "data": {
      "projects": [
        {
          "project_id": "proj-dev-001",
          "name": "ADA Compliance 2026 - Philadelphia",
          "status": "ACTIVE",
          "facilities": [
            { "facility_id": "fac-dev-001", "name": "Main Office" },
            { "facility_id": "fac-dev-002", "name": "Warehouse North" }
          ]
        }
      ],
      "limit": 10
    }
  }
  ```

### 2.3. Obtener Facilities del Proyecto Seleccionado
* **Método:** `GET`
* **Endpoint:** `{{EXPO_PUBLIC_API_BASE_URL}}/projects/proj-dev-001/facilities`
* **Respuesta (`200 OK`):** Devuelve las instalaciones asociadas al proyecto `proj-dev-001`.

### 2.4. Obtener Detalle de la Facility (Dirección y Coordenadas)
* **Método:** `GET`
* **Endpoint:** `{{EXPO_PUBLIC_API_BASE_URL}}/projects/proj-dev-001/facilities/fac-dev-001`
* **Respuesta (`200 OK`):**
  ```json
  {
    "status": "success",
    "data": {
      "facility_id": "fac-dev-001",
      "project_id": "proj-dev-001",
      "name": "Main Office",
      "address": "1200 Market Street",
      "city": "Philadelphia",
      "geo": { "lat": 39.9526, "lng": -75.1652 }
    }
  }
  ```

---

## 🚀 4. Paso 3: Inicio de Auditoría y Creación del Borrador (Fase 1)

Cuando el auditor selecciona el proyecto, la facility y el flujo de rampas y presiona **"Comenzar Auditoría"**:

1. **Generación del ID en el Cliente:**  
   La app genera un ID único con formato `audit_<uuidv4>_<yyyyMMddHHmmss>`.  
   *Ejemplo:* `audit_7219fcce-13cd-4da6-881e-33dca6b41102_20261007100000`

2. **Crear Borrador Inicial:**
   * **Método:** `POST`
   * **Endpoint:** `{{EXPO_PUBLIC_API_BASE_URL}}/audits`
   * **Body (JSON):**
     ```json
     {
       "id": "audit_7219fcce-13cd-4da6-881e-33dca6b41102_20261007100000",
       "flow_id": "flow_ramp_accessibility_verification_20251009190307",
       "flow_version": 21,
       "project_id": "proj-dev-001",
       "facility_id": "fac-dev-001",
       "status": "audit_in_progress",
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
   * **Respuesta esperada (`201 Created`):**
     ```json
     {
       "id": "audit_7219fcce-13cd-4da6-881e-33dca6b41102_20261007100000",
       "flow_id": "flow_ramp_accessibility_verification_20251009190307",
       "flow_version": 21,
       "version": 1,
       "status": "audit_in_progress",
       "created_at": "2026-10-07T10:00:00Z"
     }
     ```
     > 🛡️ **Garantía de Idempotencia:** Si este `POST` se reintenta por un corte de red, el backend responderá `409 Conflict`, evitando pisar o duplicar datos.

---

## 📸 5. Paso 4: Captura de Foto y Subida a Amazon S3 (Fase 2)

Durante la inspección en terreno, en el paso de formulario `F01` el auditor saca una foto de una pendiente no accesible.

### 4.1. Solicitar URL Prefirmada
* **Método:** `POST`
* **Endpoint:** `{{EXPO_PUBLIC_API_BASE_URL}}/uploads`
* **Body (JSON):**
  ```json
  {
    "audit_id": "audit_7219fcce-13cd-4da6-881e-33dca6b41102_20261007100000",
    "files": [
      {
        "name": "foto_pendiente_rampa_01.jpg",
        "step_id": "F01",
        "content_type": "image/jpeg"
      }
    ]
  }
  ```

* **Respuesta esperada (`200 OK`):**
  ```json
  {
    "audit_id": "audit_7219fcce-13cd-4da6-881e-33dca6b41102_20261007100000",
    "urls": [
      {
        "file_name": "foto_pendiente_rampa_01.jpg",
        "upload_url": "https://kma-audit-bucket-dev.s3.us-east-1.amazonaws.com/audit_7219fcce-13cd-4da6-881e-33dca6b41102_20261007100000/F01/foto_pendiente_rampa_01.jpg?X-Amz-Algorithm=...",
        "file_url": "s3://kma-audit-bucket-dev/audit_7219fcce-13cd-4da6-881e-33dca6b41102_20261007100000/F01/foto_pendiente_rampa_01.jpg",
        "expires_at": 1759845600
      }
    ]
  }
  ```

### 4.2. Subida Binaria Directa a S3
* **Método:** `PUT`
* **URL:** Usar exactamente el valor de `upload_url` recibido.
* **Headers:**
  ```http
  Content-Type: image/jpeg
  ```
  *(⚠️ IMPORTANTE: No enviar cabecera Authorization ni Bearer tokens en esta petición; la autenticación está firmada en los query parameters de la URL).*
* **Body:** Archivo binario de la imagen.
* **Respuesta esperada (`200 OK`):** S3 devuelve status 200 con la cabecera `ETag`.

---

## 🔄 6. Paso 5: Guardado Progresivo por Deltas (Fase 3)

A medida que el auditor completa preguntas y enlaza la foto recién subida, el móvil envía **únicamente los deltas nuevos**:

* **Método:** `PATCH` (o `PUT`)
* **Endpoint:** `{{EXPO_PUBLIC_API_BASE_URL}}/audits/audit_7219fcce-13cd-4da6-881e-33dca6b41102_20261007100000`
* **Body (JSON):**
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
          "notes": "Pendiente de 9.5% supera el límite legal de 8.3%.",
          "photos": [
            "https://kma-audit-bucket-dev.s3.us-east-1.amazonaws.com/audit_7219fcce-13cd-4da6-881e-33dca6b41102_20261007100000/F01/foto_pendiente_rampa_01.jpg"
          ]
        }
      }
    ],
    "deleted_step_ids": []
  }
  ```

* **Respuesta esperada (`200 OK`):**
  ```json
  {
    "id": "audit_7219fcce-13cd-4da6-881e-33dca6b41102_20261007100000",
    "version": 2,
    "status": "audit_in_progress",
    "updated_at": "2026-10-07T10:05:00Z"
  }
  ```
  *(El backend incrementa la versión automáticamente a `2` e incorpora las respuestas).*

---

## 🔀 7. Paso 6 (Opcional): Corrección de Rama con `deleted_step_ids`

Si el auditor vuelve atrás y cambia `R-B01` a `"yes"` (conforme), los pasos de la rama vieja (`F01`) deben descartarse:

* **Método:** `PATCH`
* **Endpoint:** `{{EXPO_PUBLIC_API_BASE_URL}}/audits/audit_7219fcce-13cd-4da6-881e-33dca6b41102_20261007100000`
* **Body (JSON):**
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
        "answer": "yes"
      }
    ],
    "deleted_step_ids": [
      "F01"
    ]
  }
  ```
* **Respuesta esperada (`200 OK`):** El servidor elimina `F01` de la base de datos y pasa a `version: 3`.

---

## 🏁 8. Paso 7: Finalización y Envío (`Submit`)

Cuando el auditor llega a la pantalla final y presiona **"Finalizar y Enviar Auditoría"**:

* **Método:** `POST`
* **Endpoint:** `{{EXPO_PUBLIC_API_BASE_URL}}/audits/audit_7219fcce-13cd-4da6-881e-33dca6b41102_20261007100000/submit`
* **Body:** `{}` *(Payload vacío, el backend ya tiene todas las respuestas consolidadas)*
* **Respuesta esperada (`200 OK`):**
  ```json
  {
    "id": "audit_7219fcce-13cd-4da6-881e-33dca6b41102_20261007100000",
    "version": 3,
    "status": "draft_report_pending_review",
    "message": "Audit submitted successfully for enrichment",
    "updated_at": "2026-10-07T10:10:00Z"
  }
  ```

> ⚙️ **Efecto en Backend:** El estado cambia a `draft_report_pending_review` y se dispara un evento a SQS. El microservicio de enriquecimiento calcula automáticamente los costos de mitigación y deja la auditoría visible en la web para el revisor de control de calidad.

---

## 🔍 9. Paso 8: Verificación Final del Estado de la Auditoría

Para verificar que la auditoría quedó registrada con todas sus respuestas consolidadas:

* **Método:** `GET`
* **Endpoint:** `{{EXPO_PUBLIC_API_BASE_URL}}/audits/audit_7219fcce-13cd-4da6-881e-33dca6b41102_20261007100000`
* **Respuesta esperada (`200 OK`):**
  * `status`: `"draft_report_pending_review"`
  * `version`: `3` (o la versión final alcanzada)
  * `answers`: Contiene exactamente la consolidación de todos los pasos recorridos sin datos huérfanos.
