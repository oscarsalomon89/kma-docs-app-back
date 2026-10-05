# Detalle de instalación dentro del proyecto

`GET /api/projects/{id}/facilities/{facility_id}` pertenece a la Lambda `projects` (función AWS `Projects`). La ruta ya está registrada en API Gateway con autorización JWT; faltaba el handler. La app móvil la declara en su repositorio HTTP y valida la envoltura `{ "status": "success", "data": { ... } }`.

La relación vigente está en `projects.facilities`, no en un atributo `project_id` de `facilities`. El caso de uso comprueba esa asociación y luego lee la instalación por su `id`, con lecturas consistentes. Devuelve los datos actuales de la instalación, no el nombre cacheado en el proyecto: nombre, dirección, ciudad, descripción, foto, ubicación, estado y metadatos. `project_id` se agrega únicamente a la respuesta por el contexto de la ruta; no se persiste.

- Proyecto inexistente, instalación inexistente o no asociada: `404`.
- Falta de identificadores: `400`.
- Fallo de almacenamiento: `500`, sin detalles internos en la respuesta.
- Instalaciones archivadas: se pueden consultar y conservan `ARCHIVED`, como en el detalle normal.
- No se agregan `notes`, asignaciones de usuarios ni reglas nuevas de roles. La autenticación sigue a cargo del gateway/runtime; se conserva el acceso de lectura vigente.

La plantilla SAM propone `dynamodb:GetItem` sobre la tabla indicada por `FacilitiesTableName` y conecta `FACILITIES_TABLE_NAME`. No crea tablas, modifica registros, añade índices ni cambia rutas en AWS. No se aplicó el permiso ni se desplegó la Lambda como parte de esta corrección.

Las pruebas cubren ruta, asociación, campos, coordenadas cero, archivado, errores y solicitudes del SDK de sólo lectura. La validación por HTTP local compara la respuesta con `GET /api/facilities/{id}` y usa el esquema real de la app móvil. No sustituye una prueba de UI móvil ni certifica un despliegue AWS. El listado de instalaciones sigue siendo el endpoint actual; este cambio sólo corrige el detalle.
