# Decisiones a definir con Backend — KMA Mobile

**Fecha:** 22/09/2026 · **Base:** código de la app en producción (v1.0.8), `RFC-AUDITS-V2-MOBILE-SYNC.md`

Este documento lista las definiciones que necesitamos de backend para avanzar, ordenadas **de lo más
crítico a lo más soft**. El criterio de orden es: _qué tan caro sale equivocarse_, no cuánto trabajo
implica. Los primeros puntos son los que hoy pueden **perder o corromper datos de auditorías reales**;
los últimos son detalles que se pueden definir sobre la marcha.

Cada punto explica, en simple:

- **Qué preguntamos**
- **Por qué importa** (qué pasa hoy en el código real)
- **Qué se ve afectado** si no se define
- **Qué asume mobile** mientras tanto, para no frenar el desarrollo

---

## Antes que nada: qué hace la app hoy

Para que las preguntas se entiendan, así funciona el envío de una auditoría en la versión que está
en producción:

1. El auditor completa el flow **entero en el teléfono**. Nada viaja al servidor hasta el final.
2. Al tocar "Finalizar", la auditoría se guarda en la base local y se encola en una cola de envío
   (patrón _outbox_), que se procesa cuando hay conexión.
3. Cuando le toca el turno, la app hace:
   - `POST /uploads` con la lista de fotos → **el backend responde con un `audit_id` nuevo** y las URLs
     firmadas de S3,
   - `PUT` de cada foto a S3,
   - `POST /audits` con ese `audit_id` y todas las respuestas.
4. Si sale bien, se borran la auditoría local y sus fotos del teléfono.

El RFC propone cambiar esto por un ciclo de 4 fases (crear el borrador al empezar, subir fotos mientras
se audita, guardar por deltas, y un submit final vacío). Las preguntas de abajo son las que hacen falta
para que ese cambio no rompa nada.

---

# 🔴 Tier 0 — Críticas: riesgo de perder o corromper datos

Estas cuatro hay que cerrarlas **antes** de escribir el código del nuevo ciclo de auditorías.

## 1. ¿`POST /audits` es idempotente cuando el `id` ya existe?

**Qué preguntamos:** si mandamos dos veces un `POST /audits` con el mismo `id`, ¿el backend responde
`200`/`201` sin duplicar, responde `409`, o sobrescribe lo que había?

**Por qué importa:** hoy, si el POST sufre un timeout o se corta la red, la app **lo reintenta sola**,
y no manda ninguna clave de idempotencia. Si el servidor alcanzó a crear la auditoría pero la respuesta
se perdió en el camino, se crea una segunda auditoría duplicada y nadie se entera.

Con el ciclo nuevo el riesgo es peor: la creación del borrador viaja con `answers: []` (vacío) y
`version: 1`. Si el backend hace un "upsert ciego", **un reintento tardío de esa creación le borra las
respuestas a una auditoría que ya venía cargada**. Es el riesgo más caro de todo el proyecto: silencioso,
del lado del servidor, y el auditor no ve nada raro.

El RFC descarta el header `Idempotency-Key` porque el `audit_id` ya actúa como clave, pero no le pide
explícitamente idempotencia a la creación. Falta cerrar ese punto.

**Qué se ve afectado:** todo el flujo de envío de auditorías, tanto el actual como el nuevo. También la
posibilidad de reintentar de forma segura cuando el auditor tiene mala señal, que es la situación normal
en campo.

**Qué asume mobile mientras tanto:** desactivamos el reintento automático de la creación y lo manejamos
desde la cola de envío, con un solo intento por corrida. Tratamos un eventual `409` como éxito. El caso
"la respuesta se perdió pero el servidor sí guardó" queda sin resolver hasta tener respuesta.

---

## 2. ¿Quién genera el `audit_id`: el móvil o `POST /uploads`?

**Qué preguntamos:** el contrato exacto de `POST /uploads` de ahora en adelante.

**Por qué importa:** hay una contradicción directa entre el RFC y el código real.

- **Hoy:** la app manda `POST /uploads` con `{ files: [{ name, step_id }] }` **sin** `audit_id`, y lee el
  `audit_id` que devuelve la respuesta. Es decir, **hoy `uploads` es quien crea la auditoría**.
- **El RFC:** dice que `uploads` no requiere cambios, pero en su ejemplo el payload **incluye** un
  `audit_id` generado por el móvil.

Las dos cosas no pueden ser ciertas a la vez. Hay que decidir cuál es.

Junto con eso, tres sub-preguntas que van en el mismo hilo:

- **¿Se puede pedir un presign para un `audit_id` que ya existe?** Hoy, cada reintento de envío pide un
  presign nuevo, que genera un `audit_id` nuevo → quedan auditorías y fotos huérfanas en S3. Sin poder
  re-firmar sobre una auditoría existente, no hay forma de reintentar sin ensuciar.
- **¿La respuesta incluye `expires_at`?** Necesitamos saber cuándo vence la URL firmada para pedir una
  nueva antes de que falle, en vez de descubrirlo con un error.
- **¿Acepta `files: []` (array vacío)?** Hoy, una auditoría **sin ninguna foto** igual llama a `/uploads`
  con la lista vacía, solamente para obtener el `audit_id`. Si el lambda valida un mínimo de archivos,
  **esto ya puede estar fallando en producción** para auditorías sin fotos. Vale la pena revisarlo aunque
  se cambie el contrato.

**Qué se ve afectado:** el backup automático de fotos (8.M) depende enteramente de esto — para subir una
foto _mientras_ el auditor todavía está trabajando, la auditoría tiene que existir en el servidor antes.
También afecta los reintentos de envío y la limpieza de archivos huérfanos en S3.

**Qué asume mobile mientras tanto:** que `/uploads` va a recibir `audit_id` + `files[{ name, step_id,
content_type }]`, y que se puede llamar varias veces para el mismo `audit_id`. Si no viene `expires_at`,
asumimos 15 minutos y re-firmamos cuando S3 responda 403.

---

## 3. ¿Cómo se borra una respuesta de una rama abandonada?

**Qué preguntamos:** el RFC define que el merge del servidor es por `step_id`: si el paso ya existía, lo
reemplaza; si es nuevo, lo agrega. Pero no define cómo se **elimina** un paso.

**Por qué importa:** los flows tienen ramas. Si el auditor contesta "NO" en el paso 3, recorre una rama
de 5 pasos; si después vuelve atrás y cambia esa respuesta a "SÍ", la app **descarta automáticamente** los
pasos de la rama vieja (ya funciona así hoy, está implementado en el runner).

Pero el servidor ya recibió esos pasos y el merge solo agrega o reemplaza. Resultado: en la base quedan
**los pasos de las dos ramas a la vez**, contradiciéndose entre sí. Y el proceso de enrichment los toma
como válidos y genera hallazgos sobre una rama que el auditor descartó.

**Qué se ve afectado:** el guardado progresivo por deltas (que es el corazón del RFC) y la edición de
auditorías ya completadas. Sin esto, el guardado progresivo produce datos sucios en cuanto un auditor
corrige una respuesta anterior — que es exactamente el caso de uso que el RFC quiere habilitar.

**Qué asume mobile mientras tanto:** mandamos un campo `deleted_step_ids: string[]` en el `PUT`. Si el
backend no lo soporta, el plan B es que el envío final mande el set **completo** de respuestas (no vacío)
y el backend reemplace todo — pero eso anula la ventaja de los deltas.

---

## 4. 🆕 Regla de compatibilidad para cambios en `GET /flows`

> Este punto **no** estaba en los análisis anteriores. Salió de revisar el código de validación.

**Qué preguntamos:** acordar una regla clara sobre qué se puede cambiar en la respuesta de `GET /flows`
sin romper las apps que ya están instaladas.

**Por qué importa:** la app valida la respuesta de `/flows` con un schema estricto y **de una sola vez,
para todo el catálogo**. Los tipos de campo permitidos están cerrados a una lista fija
(`text`, `number`, `photo`, `button`), igual que los tipos de paso.

Entonces:

- Si backend agrega **un tipo de campo nuevo** (por ejemplo `textarea` o `integer`) o **un tipo de paso
  nuevo**, la validación falla para **toda** la respuesta. La app no muestra ningún error: se cae
  silenciosamente al catálogo guardado en el teléfono.
- Consecuencia práctica: **los flows se congelan**. Los auditores siguen viendo los flows viejos, los
  nuevos nunca aparecen, y nadie se entera hasta que alguien reporta que "falta un flow". Afecta a todos
  los dispositivos con la versión actual instalada.

**La buena noticia**, ya verificada en el código: **agregar propiedades nuevas opcionales es seguro**. La
validación descarta las claves que no conoce sin fallar. Así que backend **puede** agregar `order`,
`required`, `multiline` o lo que haga falta a los campos **sin esperar una release de mobile**.

**Acuerdo propuesto:**

- ✅ **Libre:** agregar propiedades nuevas a objetos existentes.
- ⛔ **Coordinado:** agregar valores nuevos a `type` de campo, o tipos de paso nuevos. Solo después de una
  versión de mobile que los tolere.

**Qué se ve afectado:** todo el catálogo de flows en todos los dispositivos, incluida la app que está hoy
en producción. Es el cambio más barato de prevenir y el más caro de detectar.

**Qué hace mobile:** pasamos a validar flow por flow, descartando solo el flow inválido en lugar del
catálogo completo, y dejamos registro del descarte.

---

# 🟠 Tier 1 — Importantes: baratas de definir ahora, caras de corregir después

## 5. Convención de nombres de archivo en S3

**Qué preguntamos:** para la key `<audit_id>/<step_id>/<nombre>`, ¿un `PUT` sobre la misma key
sobrescribe el objeto? ¿Se espera un nombre estable (`photo_01.jpg`) o uno único?

**Por qué importa:** hoy el nombre de archivo se arma **en el momento del envío** e incluye un timestamp,
así que cada reintento genera una key distinta → fotos huérfanas acumulándose en S3. Queremos generar el
nombre **una sola vez, al sacar la foto**, y guardarlo, para que los reintentos vayan siempre a la misma
key y el `PUT` sea idempotente.

**Qué se ve afectado:** los reintentos de subida de fotos y el costo de almacenamiento en S3. También la
edición de auditorías: si los nombres son estables tipo `photo_01.jpg`, reeditar un paso pisaría la foto
anterior, que puede ser justo lo que no queremos con evidencia ya entregada.

**Qué asume mobile:** nombre **único y estable por foto** (`<step_id>_<field_id>_<id_local>.<ext>`),
generado al capturar y persistido.

---

## 6. ¿El submit devuelve la versión final?

**Qué preguntamos:** ¿la respuesta de `POST /audits/{id}/submit` incluye `version` (y `updated_at`)?

**Por qué importa:** el control de concurrencia del RFC se basa en que el móvil sepa en qué versión está
la auditoría en el servidor. Después del submit, si no nos devuelven la versión, el móvil se queda sin
saberlo.

**Qué se ve afectado:** la edición de auditorías completadas (9.M). Sin la versión, para editar hay que
hacer primero una lectura extra al servidor — lo que significa que **no se puede empezar a editar sin
conexión**, aunque la auditoría se acabe de enviar desde ese mismo teléfono.

**Qué asume mobile:** que después del submit la versión local queda inválida y hace falta una lectura
antes de editar.

---

## 7. Formato exacto del `id` de auditoría

**Qué preguntamos:** ¿el backend valida el formato o el largo del `id`? El RFC lo describe como
`audit_${uuidv4()}_${Date.now()}`, pero los ejemplos reales muestran
`audit_<uuid>_20260120220344`, con el timestamp formateado como fecha legible. No son lo mismo.

**Por qué importa:** es la partition key de la tabla. Si hay validación de formato, el móvil tiene que
generar exactamente ese, y un error acá falla en el momento de crear la auditoría.

**Qué se ve afectado:** la creación de auditorías. Es una definición de 2 minutos que, si se descubre
tarde, invalida las auditorías creadas hasta entonces.

**Qué asume mobile:** `audit_<uuid>_<yyyyMMddHHmmss>` (el formato de los ejemplos reales).

---

# 🟡 Tier 2 — Definen alcance: no hay riesgo de datos, pero sin respuesta hay funcionalidad que no arranca

## 8. Endpoints de lectura de auditorías

**Qué preguntamos:** ¿existen o se van a agregar `GET /audits?project_id&facility_id&auditor` (listado) y
`GET /audits/{id}` (detalle)? ¿Entran en este RFC o van por separado? ¿El detalle devuelve la versión del
flow con la que se hizo la auditoría, y las URLs de las fotos?

**Por qué importa:** hoy la app **no tiene ningún endpoint de lectura de auditorías**, y además borra la
auditoría local apenas se envía con éxito. O sea: una vez enviada, el teléfono no conserva nada.

**Qué se ve afectado:**

- **"Revisar Completados" y editar auditorías en sitio (9.M) no se puede hacer.** Es el requerimiento más
  grande de Phase 3 y depende enteramente de esto.
- Tampoco se puede recuperar una auditoría cuya respuesta de creación se perdió (ver punto 1).
- Tampoco se puede resolver un conflicto de versión de forma informada.

**Qué asume mobile:** sin lectura, el manejo de conflictos se hace a ciegas y la edición queda fuera de
alcance.

---

## 9. ¿Se puede editar una auditoría después del submit?

**Qué preguntamos:** ¿se admite `PUT /audits/{id}` una vez que la auditoría pasó a
`draft_report_pending_review`? Si no, ¿qué transición de estado habilita la edición?

**Por qué importa:** el RFC solo define la transición `audit_in_progress → draft_report_pending_review`, y
nada sobre qué pasa después.

**Qué se ve afectado:** todo el requerimiento de **editar flows completados en sitio**: la pantalla
de facility con "Iniciar Nueva Auditoría" / "Revisar Completados", el listado de completados y el modo
edición del runner. Es el bloque de trabajo más grande de los cuatro requerimientos mobile, y no se puede
ni empezar sin esta respuesta.

También hay una definición de producto detrás: ¿solo puede editar el autor? ¿hay una ventana de tiempo?
¿se puede editar una auditoría ya aprobada?

---

## 10. ¿El `409` de conflicto puede decir qué tiene el servidor?

**Qué preguntamos:** cuando el backend responde `409` por conflicto de versión, ¿puede incluir también
las respuestas que tiene guardadas, o al menos la lista de `step_id`?

**Por qué importa:** hoy el móvil solo recibiría el número de versión nuevo. Adopta esa versión y reenvía
lo suyo, sin saber qué hay del otro lado. Si el conflicto vino de **otro dispositivo** (dos auditores en
la misma auditoría), estaría sobrescribiendo sin avisar.

**Qué se ve afectado:** la calidad de la resolución de conflictos. Con la respuesta enriquecida se puede
avisar al auditor; sin ella, gana el último que escribe.

**Qué asume mobile:** el último que escribe gana, por paso, con un tope de reintentos por conflicto para
no entrar en loop.

---

## 11. ¿Hay límite de escrituras para el guardado automático?

**Qué preguntamos:** ¿hay throttling o rate limit en el `PUT`? Cada guardado es una escritura en la base
y un incremento de versión. Un flow de 30–50 pasos, con varios auditores en paralelo, puede generar
cientos de escrituras por auditoría.

**Por qué importa:** define cada cuánto conviene guardar y si vale la pena agrupar varios pasos por
request.

**Qué se ve afectado:** el costo de infraestructura y el consumo de datos móviles del auditor en campo.

**Qué asume mobile:** **no** usamos el timer de 2 segundos que propone el RFC. Guardamos al completar cada
paso y agrupamos todo lo pendiente de una auditoría en un solo `PUT` por corrida de sincronización — bastante
menos escrituras que la propuesta original.

---

## 12. Contrato final de los campos de formulario

**Qué preguntamos:** para el requerimiento "campos dinámicos en orden personalizado":

- ¿Se agrega un campo de orden? ¿Cómo se llama, de qué tipo es, empieza en 0 o en 1?
- ¿El array de campos ya viene ordenado desde el backend, o hay que ordenarlo en el móvil?
- ¿Se agrega `required` (campo obligatorio)? ¿Y `multiline` para las notas?

**Por qué importa:** hoy la app renderiza los campos **en el orden en que vienen en el array**. Si el
backend ya los manda ordenados, mobile no necesita hacer nada para el orden. Lo que sí falta es la
obligatoriedad: hoy la única validación es "al menos una foto en alguno de los campos de foto", no hay
validación campo por campo, y el mockup pide explícitamente "Tomar Foto (obligatorio)".

**Qué se ve afectado:** solo el requerimiento 4.M, que es chico y está aislado del resto. No bloquea nada
más.

**Nota práctica:** por lo del punto 4, backend **puede shippear estos campos cuando quiera** — la app
actual los ignora sin romperse. No hace falta coordinar releases.

**Qué asume mobile:** respetamos el orden del array (comportamiento actual) y preparamos soporte opcional
para un campo de orden, como defensa.

---

# 🟢 Tier 3 — Soft: se pueden definir sobre la marcha

## 13. ¿Los errores `4xx` traen un mensaje legible?

Cuando el backend rechaza una auditoría con un `4xx`, ¿el body incluye un mensaje entendible para mostrar?

**Afecta:** solo qué texto ve el auditor en la lista de "auditorías que no se pudieron enviar". Sin
mensaje, mostramos el código HTTP y un recorte del body crudo — feo pero funcional.

> Contexto importante: hoy **un `4xx` hace que la auditoría se descarte en silencio**, sin UI ni
> reintento. Eso lo arreglamos del lado de mobile sin depender de backend; esta pregunta es solo
> cosmética.

## 14. ¿Se borran del servidor las fotos que el auditor elimina?

Esta es más de producto y legal que técnica, y conviene decidirla antes de habilitar el backup automático:

- **Trazabilidad:** en una auditoría, una foto subida puede considerarse evidencia. Borrarla definitivamente
  puede ir en contra de requisitos del cliente. Quizás corresponde un borrado lógico (marcarla como
  eliminada y ocultarla).
- **Privacidad:** si el auditor la borró porque salió alguien, una patente o información sensible,
  conservarla sin uso es un riesgo.
- **Costo:** fotos huérfanas acumulándose en S3 cuestan sin aportar nada.

Y define quién la borra: un endpoint `DELETE` (le suma trabajo a mobile, porque tiene que funcionar
offline y encolarse como cualquier otra operación) o una regla de ciclo de vida en S3 sobre objetos que
ninguna auditoría referencia (cero trabajo en mobile).

Ojo: la política puede ser distinta para una foto de un borrador que para una foto de una auditoría ya
entregada que se está editando.

## 15. Borradores abandonados

Si la app muere o el auditor nunca cierra una auditoría, queda un borrador en `audit_in_progress` para
siempre. ¿Hay TTL? ¿Se muestran en la web? Mobile no borra nada del servidor por su cuenta; solo descarta
el borrador local.

## 16. Versión del flow con la que se hizo cada auditoría

Si una auditoría se hizo con la versión 1 de un flow y hoy el flow va por la versión 2, editarla con los
pasos nuevos puede romper la navegación (pasos que ya no existen, condiciones nuevas). Lo ideal es que la
lectura del detalle devuelva los pasos de la versión usada. Se cierra junto con el punto 8.

---

# Resumen para la reunión

| Prioridad | Punto                                                    | Qué se rompe / no arranca sin esto                                   |
| --------- | -------------------------------------------------------- | -------------------------------------------------------------------- |
| 🔴 1      | Idempotencia de `POST /audits`                           | Auditorías duplicadas, o respuestas borradas por un reintento        |
| 🔴 2      | Contrato de `POST /uploads` / quién genera el `audit_id` | Backup automático de fotos (8.M); fotos huérfanas en S3              |
| 🔴 3      | Borrado de pasos de ramas abandonadas                    | Auditorías con datos contradictorios que el enrichment procesa igual |
| 🔴 4      | Regla de compatibilidad de `GET /flows`                  | El catálogo de flows se congela en silencio en todos los teléfonos   |
| 🟠 5      | Nombres de archivo en S3                                 | Reintentos de fotos y costo de S3                                    |
| 🟠 6      | ¿El submit devuelve `version`?                           | Editar una auditoría recién enviada sin conexión                     |
| 🟠 7      | Formato del `id`                                         | Creación de auditorías                                               |
| 🟡 8      | `GET /audits` (listado y detalle)                        | "Revisar Completados" y editar en sitio (9.M)                        |
| 🟡 9      | ¿Se puede editar después del submit?                     | Todo 9.M — el bloque más grande de Phase 3                           |
| 🟡 10     | ¿El `409` dice qué tiene el servidor?                    | Calidad de la resolución de conflictos                               |
| 🟡 11     | Límite de escrituras del guardado automático             | Costo de infra y datos móviles                                       |
| 🟡 12     | Contrato de campos de formulario                         | Solo 4.M (aislado; backend puede adelantarse)                        |
| 🟢 13     | Mensaje legible en los `4xx`                             | Solo el texto del error que ve el auditor                            |
| 🟢 14     | Política de borrado de fotos                             | Decisión de producto/legal                                           |
| 🟢 15     | Borradores abandonados                                   | Limpieza del lado del servidor                                       |
| 🟢 16     | Versión del flow por auditoría                           | Editar auditorías viejas sin romper la navegación                    |

**Sugerencia de secuencia:** los puntos **1, 2, 3 y 4** en la primera reunión — son los que habilitan el
grueso del RFC y los únicos con riesgo real de datos. Los puntos **8 y 9** en la segunda, porque habilitan
la edición de auditorías completadas. El punto **12** se puede resolver por mail.

---

# Anexo: hallazgos de la revisión del código

Todo lo de arriba sale de verificar el código de la versión en producción, no de suposiciones. Lo que se
confirmó:

| Afirmación                                                                      | Dónde está en el código                                                     |
| ------------------------------------------------------------------------------- | --------------------------------------------------------------------------- |
| `POST /audits` se reintenta solo, sin clave de idempotencia                     | `src/core/http/http.ts` — el retry solo excluye `/uploads`                  |
| Un `4xx` hace que la auditoría se borre de la cola sin dejar rastro             | `src/processes/sync/SyncService.ts` — `drop` se trata igual que `success`   |
| El backoff de reintentos vive en memoria y se pierde al cerrar la app           | `src/processes/sync/SyncService.ts` — se usan `Map` en memoria              |
| La cola siempre toma los 10 items más viejos, sin filtrar los que fallan        | `src/core/repos/sqliteOutboxRepo.ts` — `ORDER BY created_at ASC LIMIT ?`    |
| Cada paso del flow inserta una fila nueva de borrador                           | `useFlowRunner.ts` llama a `persistDraft` sin id (el repo sí soporta id)    |
| Si el presign no devuelve URL para una foto, la auditoría se envía sin esa foto | `src/features/flow-runner/application/usecases.ts` — `if (!match) continue` |
| El nombre del archivo se arma al enviar, con timestamp                          | mismo archivo — incluye `Date.now()`                                        |
| Las fotos se leen a memoria como base64 y se decodifican en JS                  | mismo archivo — `readFileAsUint8` / `base64ToUint8Array`                    |
| La conversión HEIC → JPEG existe pero no se usa en ningún lado                  | `src/features/camera/application/ensureWebImage.ts` — 0 referencias         |
| Un refresh de token que falla por falta de señal desloguea al auditor           | `src/core/http/http.ts` — rama del 401                                      |
| `flow_type` ya viaja, se mapea y se guarda localmente                           | DTO → mapper → `sqliteFlowRepo`                                             |
| No existen `order`, `required` ni `multiline` en los campos                     | `StepFieldDtoSchema` — solo id, type, label, unit, placeholder              |
| `/flows` se valida de una sola vez para todo el catálogo                        | `flow.repo.http.ts` — `FlowsResponseDtoSchema.parse(...)`                   |
