# Casos de uso — Sistema distribuido de turnos

Autora: Candela González · Programación II · Proyecto integrador 2026

Cada caso de uso (CU) se desarrolla en **una rama propia** creada desde su issue (`N-cu-XX-slug`), y se integra a `develop` por Pull Request.

## Trazabilidad con el alcance obligatorio (§5)

| Ítem | Alcance | CU |
|---|---|---|
| 1 | Registrar y autenticar usuario final | CU-01, CU-02 |
| 2 | Registrar cuenta técnica ante la cátedra (Postman) | CU-03 |
| 3 | Autenticar cuenta técnica y recuperar configuración | CU-04 |
| 4 | Snapshot completo guardado localmente | CU-05, CU-17 |
| 5 | Actualización por Kafka y Redis | CU-06 |
| 6 | Detectar imposibilidad de continuar y reconstruir | CU-07 |
| 7 | Buscar y filtrar con la copia local | CU-08 |
| 8 | Construir y presentar turnos disponibles | CU-09 |
| 9 | Solicitar bloqueo temporal | CU-10 |
| 10 | Iniciar confirmación por REST | CU-11 |
| 11 | Información adicional por Kafka y respuesta | CU-12 |
| 12 | Reflejar el resultado final | CU-13 |
| 13 | Consultar mis reservas | CU-14 |
| 14 | Cancelar una reserva propia | CU-15 |
| 15 | Informar errores y recuperarse | CU-16, CU-17, CU-18 |

## Diagramas

### Contexto

![Contexto](diagramas/01-contexto.svg)

### catalogo-service e integración

![catalogo-service e integración](diagramas/02-catalogo-integracion.svg)

### turnos-service: reservas

![turnos-service: reservas](diagramas/03-turnos-reservas.svg)

### Usuarios y seguridad

![Usuarios y seguridad](diagramas/04-usuarios-seguridad.svg)

### Estados de la reserva (propuesta)

![Estados de la reserva (propuesta)](diagramas/05-estados-reserva.svg)

## CU-01 — Registrar usuario final

- **Servicio:** turnos-service
- **Actor principal:** Usuario final
- **Actores secundarios:** —
- **Ítem de la consigna:** 1 · **Sprint:** Sprint 2 · **Prioridad:** Alta
- **Rama:** `N-cu-01-registro-usuario`

**Precondiciones**

- El login y el email no están registrados.
- La app KMP tiene conectividad con turnos-service.

**Disparador:** El usuario completa el formulario de registro y lo envía.

**Flujo principal**

1. La app envía login, password, firstName, lastName, email, imageUrl (opcional) y langKey.
2. El backend valida cada campo (formato, longitud, obligatorios).
3. Verifica que login y email no existan.
4. Guarda la contraseña con hash (BCrypt); nunca en texto plano.
5. Crea el usuario activo, con autoridad ROLE_USER asignada por el backend.
6. Responde 201 sin devolver la contraseña ni campos internos.

**Flujos alternativos / errores**

- Datos inválidos → 400 con detalle por campo.
- Login o email ya usados → 400/409 con código de error.
- El cliente intenta mandar id, authorities o activated → se ignoran (no se confía en el cliente).

**Postcondiciones**

- Existe un usuario activo que puede iniciar sesión de inmediato (sin verificación por mail).

**Reglas de negocio**

- El modelo de usuario es compatible con el de JHipster.
- Id, estado de activación, autoridades y auditoría los define el backend.

**Evidencia asociada:** Seguridad (§9): hashing, validación en el límite, campos no controlables por el cliente.

**Patrones:** Mapper/DTO, Strategy (hasher como puerto), Repository.

**Pruebas:** Dominio: validaciones. Servicio: registro OK, duplicado, hash ≠ texto plano. Controller: 400/201.

## CU-02 — Iniciar sesión y obtener JWT

- **Servicio:** turnos-service
- **Actor principal:** Usuario final
- **Actores secundarios:** —
- **Ítem de la consigna:** 1 · **Sprint:** Sprint 2 · **Prioridad:** Alta
- **Rama:** `N-cu-02-login-jwt`

**Precondiciones**

- El usuario está registrado y activo.

**Disparador:** El usuario ingresa login y contraseña en la app.

**Flujo principal**

1. La app envía las credenciales a /api/authenticate de turnos-service.
2. El backend verifica la contraseña contra el hash.
3. Emite un JWT firmado con vigencia limitada, con identidad y autoridades.
4. La app guarda el token de forma segura y lo envía en cada request protegida.

**Flujos alternativos / errores**

- Credenciales incorrectas → 401 con mensaje genérico (sin revelar cuál falló).
- Token vencido o inválido en un request posterior → 401 y la app vuelve al login.
- Acceso a endpoint protegido sin token → 401.

**Postcondiciones**

- La app posee un JWT válido para consumir las APIs protegidas.

**Reglas de negocio**

- Se valida firma, vigencia y permisos en cada API protegida.
- La contraseña nunca aparece en respuestas ni logs.
- Secreto de firma externalizado (variable de entorno).

**Evidencia asociada:** Rechazo de accesos no autenticados o no autorizados.

**Patrones:** Adapter (JWT), Strategy (PasswordEncoder).

**Pruebas:** Login OK/KO, token vencido, endpoint sin token, firma adulterada.

## CU-03 — Registrar la cuenta técnica de integración ante la cátedra

- **Servicio:** Operativo (Postman)
- **Actor principal:** Alumna (operadora)
- **Actores secundarios:** Servicio de la cátedra
- **Ítem de la consigna:** 2 · **Sprint:** Sprint 2 · **Prioridad:** Alta (bloqueante)
- **Rama:** `N-cu-03-registro-cuenta-tecnica`

**Precondiciones**

- Conectividad con el servidor de la cátedra (ZeroTier autorizado).
- La cuenta técnica aún no fue registrada.

**Disparador:** La alumna ejecuta una vez POST /api/student/register desde Postman.

**Flujo principal**

1. Arma la request con los datos pedidos por el anexo de integración.
2. La cátedra crea la cuenta técnica del proyecto (groupId).
3. Se guardan usuario y secretos fuera del repositorio (.env, colección sin credenciales).
4. Se documenta el procedimiento en el repo de documentación.

**Flujos alternativos / errores**

- Cuenta ya registrada → no se repite; se usa la existente.
- Sin conectividad → se ejecuta contra un servidor simulado hasta tener acceso (sólo para desarrollo).
- Registro fallido → se interpreta el error application/problem+json.

**Postcondiciones**

- Existe una cuenta técnica; sus credenciales NUNCA llegan a la app KMP.

**Reglas de negocio**

- Se registra una sola vez.
- Cuenta técnica ≠ cuenta de usuario final.

**Evidencia asociada:** Seguridad: credenciales externalizadas.

**Patrones:** —

**Pruebas:** No aplica; se prueba con la colección de Postman exportada (sin secretos).

## CU-04 — Autenticar la cuenta técnica y recuperar su configuración

- **Servicio:** catalogo-service
- **Actor principal:** catalogo-service / turnos-service
- **Actores secundarios:** Servicio de la cátedra
- **Ítem de la consigna:** 3 · **Sprint:** Sprint 2 · **Prioridad:** Alta
- **Rama:** `N-cu-04-auth-cuenta-tecnica`

**Precondiciones**

- CU-03 realizado.
- Credenciales técnicas configuradas por variables de entorno.

**Disparador:** Un servicio necesita llamar a la cátedra y no tiene token vigente.

**Flujo principal**

1. El servicio llama a POST /api/authenticate con la cuenta técnica.
2. Obtiene el JWT técnico (vigencia ~1 año) y lo cachea.
3. Consulta GET /api/student/integration: groupId, topics y estado de provisión.
4. Verifica provisioningStatus = PROVISIONED antes de usar Redis/Kafka.

**Flujos alternativos / errores**

- PENDING → reintento con espera acotada y estado visible.
- FAILED / REVOKED → se informa el error y no se inicia la integración.
- 401 de la cátedra → se reautentica una vez; si falla, se traduce a error de integración.

**Postcondiciones**

- El servicio dispone de token técnico y de la configuración asignada.

**Reglas de negocio**

- Timeouts y reintentos acotados.
- El token técnico no se loguea.

**Evidencia asociada:** Contrato con la cátedra (D4); seguridad.

**Patrones:** Adapter (cliente REST), Proxy/Cache del token.

**Pruebas:** Contrato con servidor simulado: OK, PENDING, FAILED, 401, timeout.

## CU-05 — Inicializar el catálogo desde un snapshot completo

- **Servicio:** catalogo-service
- **Actor principal:** catalogo-service
- **Actores secundarios:** Servicio de la cátedra (REST)
- **Ítem de la consigna:** 4 · **Sprint:** Sprint 2 · **Prioridad:** Alta
- **Rama:** `N-cu-05-sincronizacion-snapshot`

**Precondiciones**

- CU-04 realizado.
- Base local vacía o necesidad de reconstruirla.

**Disparador:** Arranque con base vacía o pedido de reconstrucción (CU-07).

**Flujo principal**

1. Marca el estado de sync como SYNCING.
2. Llama a GET /api/synchronization/snapshot.
3. En UNA transacción reemplaza categorías, profesionales y horarios semanales.
4. Registra como versión local exactamente la versión del snapshot recibido.
5. Marca el estado como UP_TO_DATE y guarda la fecha.

**Flujos alternativos / errores**

- Falla la transacción → rollback completo; la versión local no cambia; estado ERROR.
- Cátedra caída → reintento acotado y error informado.
- Snapshot repetido → mismo resultado final (idempotente).

**Postcondiciones**

- El catálogo local es consistente con la versión registrada.

**Reglas de negocio**

- Los IDs son los de la cátedra (upsert).
- La versión sólo avanza si la unidad de trabajo terminó bien.

**Evidencia asociada:** Inicialización desde snapshot (§11).

**Patrones:** Repository, Unit of Work (transacción), Adapter.

**Pruebas:** Aplicación transaccional con Testcontainers; rollback; idempotencia.

## CU-06 — Actualizar el catálogo de forma incremental (Kafka + Redis)

- **Servicio:** catalogo-service
- **Actor principal:** catalogo-service
- **Actores secundarios:** Kafka y Redis de la cátedra
- **Ítem de la consigna:** 5 · **Sprint:** Sprint 2 · **Prioridad:** Alta
- **Rama:** `N-cu-06-sincronizacion-incremental`

**Precondiciones**

- CU-05 realizado (versión local conocida).

**Disparador:** Llega un evento CatalogUpdated por Kafka.

**Flujo principal**

1. El consumer lee CatalogUpdated y compara con la versión local.
2. Consulta en Redis la metadata (versión actual / más antigua disponible).
3. Lee changes:{versión} de cada versión posterior a la local, en orden.
4. Por cada cambio busca el dato en los hashes de Redis y lo aplica (upsert/baja).
5. Avanza la versión local sólo al terminar cada unidad de trabajo.

**Flujos alternativos / errores**

- Evento duplicado (mismo eventId) → se ignora sin alterar el resultado.
- Falta una versión intermedia → CU-07.
- Redis no disponible → reintento acotado; el consumer no pierde el mensaje.

**Postcondiciones**

- El catálogo local avanzó a la última versión disponible.

**Reglas de negocio**

- At-least-once: el procesamiento debe ser idempotente.
- Los cambios se aplican en orden, uno por versión.

**Evidencia asociada:** Actualización incremental; idempotencia ante duplicado (§11).

**Patrones:** Idempotent Receiver, Repository, Adapter (Kafka/Redis).

**Pruebas:** Evento duplicado, orden, avance de versión, falla a mitad de camino.

## CU-07 — Detectar discontinuidad y reconstruir por snapshot

- **Servicio:** catalogo-service
- **Actor principal:** catalogo-service
- **Actores secundarios:** Kafka, Redis y REST de la cátedra
- **Ítem de la consigna:** 6 · **Sprint:** Sprint 2 · **Prioridad:** Alta
- **Rama:** `N-cu-07-deteccion-discontinuidad`

**Precondiciones**

- Existe versión local.

**Disparador:** Durante CU-06 la versión local queda fuera de la ventana de Redis o falta un cambio.

**Flujo principal**

1. Compara la versión local con oldest-available-version.
2. Si local < oldest-available (o hay un salto de versiones) concluye que no puede seguir de forma segura.
3. Ejecuta CU-05 (snapshot completo).
4. Retoma el flujo incremental desde la nueva versión.

**Flujos alternativos / errores**

- Estado local no verificable → también snapshot.
- El snapshot falla → estado ERROR; se reintenta con backoff acotado.

**Postcondiciones**

- La copia local se reconstruyó sin corrupción.

**Reglas de negocio**

- Nunca avanzar versión con huecos.

**Evidencia asociada:** Recuperación mediante snapshot luego de una discontinuidad (§11).

**Patrones:** Strategy (incremental vs. completo), State (SyncState).

**Pruebas:** Versión local menor a la ventana; hueco de versión; recuperación exitosa.

## CU-08 — Buscar y filtrar profesionales sobre la copia local

- **Servicio:** catalogo-service
- **Actor principal:** Usuario final
- **Actores secundarios:** App KMP
- **Ítem de la consigna:** 7 · **Sprint:** Sprint 3 · **Prioridad:** Alta
- **Rama:** `N-cu-08-buscar-filtrar-profesionales`

**Precondiciones**

- El catálogo local fue inicializado (CU-05).
- El usuario tiene JWT válido.

**Disparador:** El usuario abre la búsqueda y aplica filtros.

**Flujo principal**

1. La app envía los filtros a catalogo-service: categoría, nombre, habilitado y disponibilidad.
2. El servicio resuelve la consulta SOLO con datos locales (nunca llama a la cátedra).
3. Devuelve la lista paginada de profesionales con su categoría y horarios.
4. La app muestra los resultados.

**Flujos alternativos / errores**

- Sin resultados → lista vacía (200).
- Filtros inválidos → 400.
- Sin token → 401.

**Postcondiciones**

- Ninguna (consulta).

**Reglas de negocio**

- Prohibido consultar al servicio central por cada búsqueda.
- Es la única fuente local vigente de profesionales.

**Evidencia asociada:** Búsquedas sobre datos locales y filtros obligatorios (§11).

**Patrones:** Specification/Query object, Repository, DTO.

**Pruebas:** Un test por filtro y combinación; verificar que no hay llamadas a la cátedra.

## CU-09 — Consultar turnos disponibles

- **Servicio:** turnos-service
- **Actor principal:** Usuario final
- **Actores secundarios:** catalogo-service, cátedra (REST)
- **Ítem de la consigna:** 8 · **Sprint:** Sprint 3 · **Prioridad:** Alta
- **Rama:** `N-cu-09-consultar-disponibilidad`

**Precondiciones**

- Usuario autenticado.
- catalogo-service disponible.

**Disparador:** El usuario elige profesional y fecha.

**Flujo principal**

1. turnos-service pide a catalogo-service (JWT servicio a servicio) las reglas semanales vigentes del profesional.
2. Consulta las ocupaciones en GET /api/appointment-occupancies.
3. Genera los horarios posibles de esa fecha según las reglas.
4. Descarta los ocupados y devuelve los libres.

**Flujos alternativos / errores**

- catalogo-service caído → 503 con mensaje claro; no se inicia la operación.
- Profesional inexistente o deshabilitado → 404/422.
- Cátedra no responde → error de integración traducido.

**Postcondiciones**

- Ninguna (consulta).

**Reglas de negocio**

- Turnos NO guarda una réplica del catálogo.
- Las ocupaciones vienen siempre de la cátedra.

**Evidencia asociada:** Construcción de disponibilidad (item 8).

**Patrones:** Adapter (clientes), Domain Service (cálculo de slots).

**Pruebas:** Cálculo de slots (unitario puro); con ocupaciones; con catálogo caído.

## CU-10 — Solicitar un bloqueo temporal (hold)

- **Servicio:** turnos-service
- **Actor principal:** Usuario final
- **Actores secundarios:** Cátedra (REST)
- **Ítem de la consigna:** 9 · **Sprint:** Sprint 3 · **Prioridad:** Alta
- **Rama:** `N-cu-10-solicitar-hold`

**Precondiciones**

- Turno elegido está libre (CU-09).

**Disparador:** El usuario selecciona un horario.

**Flujo principal**

1. turnos-service revalida con catalogo-service que el profesional/horario siguen vigentes.
2. Crea localmente el proceso de reserva asociado al usuario, en estado HOLD_REQUESTED.
3. Llama a POST /api/appointment-holds.
4. Guarda holdId y vencimiento; estado HOLD_CREATED.

**Flujos alternativos / errores**

- SLOT_ALREADY_HELD → se informa y se propone otro horario; el proceso queda FAILED.
- Timeout con respuesta perdida → reintento idempotente.
- Horario fuera de reglas → 422.

**Postcondiciones**

- Hay un hold vigente asociado al usuario; el reloj corre en la cátedra.

**Reglas de negocio**

- Conservarlo localmente no extiende su vigencia.
- El proceso queda ligado al usuario autenticado, no a un id enviado por el cliente.

**Evidencia asociada:** Reserva con concurrencia y vencimiento.

**Patrones:** State, Adapter, Idempotency key.

**Pruebas:** Hold OK, slot tomado, reintento, usuario dueño.

## CU-11 — Confirmar inicialmente el hold por REST

- **Servicio:** turnos-service
- **Actor principal:** Usuario final
- **Actores secundarios:** Cátedra (REST)
- **Ítem de la consigna:** 10 · **Sprint:** Sprint 3 · **Prioridad:** Alta
- **Rama:** `N-cu-11-confirmar-hold-rest`

**Precondiciones**

- Hold vigente (CU-10).

**Disparador:** El usuario acepta continuar con la reserva.

**Flujo principal**

1. Verifica que el proceso pertenece al usuario y que el hold no venció localmente.
2. Llama a POST /api/appointment-holds/{holdId}/confirm.
3. Pasa el proceso a CONFIRMATION_PENDING (espera la solicitud de información adicional).

**Flujos alternativos / errores**

- HOLD_EXPIRED → proceso EXPIRED; se informa a la app.
- Repetición por timeout → respuesta idempotente, sin duplicar.
- Proceso de otro usuario → 403/404.

**Postcondiciones**

- El proceso espera AdditionalInformationRequested.

**Reglas de negocio**

- No retroceder un proceso que ya llegó a un estado final.

**Evidencia asociada:** Reserva confirmada mediante REST + Kafka (parte REST).

**Patrones:** State machine.

**Pruebas:** Transiciones válidas/ inválidas; hold vencido; ajeno.

## CU-12 — Recibir la solicitud de información adicional y responderla

- **Servicio:** turnos-service
- **Actor principal:** turnos-service
- **Actores secundarios:** Kafka de la cátedra; Usuario final (carga el teléfono)
- **Ítem de la consigna:** 11 · **Sprint:** Sprint 3 · **Prioridad:** Alta
- **Rama:** `N-cu-12-info-adicional-kafka`

**Precondiciones**

- Proceso en CONFIRMATION_PENDING.

**Disparador:** Llega AdditionalInformationRequested por Kafka.

**Flujo principal**

1. El consumer lo recibe y verifica eventId y reservationProcessId.
2. Localiza el proceso local y lo pasa a INFO_REQUESTED.
3. La app muestra el pedido de teléfono al usuario.
4. El usuario envía el teléfono; turnos-service publica AdditionalInformationSubmitted en el topic asignado (key = reservationProcessId).
5. El proceso pasa a INFO_SUBMITTED.

**Flujos alternativos / errores**

- Evento duplicado → ignorado (eventId ya procesado).
- Evento de proceso desconocido → se registra y descarta.
- El usuario no responde a tiempo → CU-13 (expiración).

**Postcondiciones**

- La información fue enviada; se espera el resultado final.

**Reglas de negocio**

- At-least-once e idempotencia por eventId.
- Teléfono validado antes de enviarlo.

**Evidencia asociada:** Idempotencia ante mensaje duplicado (§11).

**Patrones:** Idempotent Receiver, Saga por coreografía, Adapter (Kafka).

**Pruebas:** Duplicado, fuera de orden, proceso inexistente.

## CU-13 — Procesar el resultado final de la reserva

- **Servicio:** turnos-service
- **Actor principal:** turnos-service
- **Actores secundarios:** Kafka de la cátedra
- **Ítem de la consigna:** 12 · **Sprint:** Sprint 3 · **Prioridad:** Alta
- **Rama:** `N-cu-13-resultado-final`

**Precondiciones**

- Proceso en INFO_SUBMITTED (o pendiente que puede expirar).

**Disparador:** Llega AppointmentConfirmed, AdditionalInformationRejected, AppointmentProcessExpired o AppointmentProcessInvalid.

**Flujo principal**

1. El consumer deduplica por eventId.
2. Aplica la transición según el evento: CONFIRMED, REJECTED, EXPIRED o INVALID.
3. Si fue confirmada, guarda la reserva local vinculada al usuario.
4. La app consulta el estado y muestra el resultado.

**Flujos alternativos / errores**

- Evento sobre un proceso ya final → se ignora (no retrocede).
- Eventos fuera de orden → se converge al estado final correcto.
- Confirmación + expiración cruzadas → gana la primera final.

**Postcondiciones**

- El estado local refleja el resultado final sin reservas duplicadas.

**Reglas de negocio**

- Convergencia entre lo observado por REST y por Kafka.

**Evidencia asociada:** Reserva confirmada; reserva que expira sin completar la info (§11).

**Patrones:** State machine, Idempotent Receiver, Strategy por tipo de evento.

**Pruebas:** Un test por evento; duplicados; fuera de orden; estado final inmutable.

## CU-14 — Consultar mis reservas

- **Servicio:** turnos-service
- **Actor principal:** Usuario final
- **Actores secundarios:** Cátedra (REST)
- **Ítem de la consigna:** 13 · **Sprint:** Sprint 3 · **Prioridad:** Alta
- **Rama:** `N-cu-14-consultar-mis-reservas`

**Precondiciones**

- Usuario autenticado.

**Disparador:** El usuario abre “Mis reservas”.

**Flujo principal**

1. turnos-service obtiene la identidad del JWT (no de un parámetro).
2. Lista los procesos y reservas locales de ESE usuario.
3. Opcionalmente contrasta con GET /api/appointments (cuenta técnica) para confirmar el estado.
4. Devuelve la lista con estado y datos históricos del profesional.

**Flujos alternativos / errores**

- Cátedra caída → se muestran los datos locales marcados como no verificados.
- Sin reservas → lista vacía.

**Postcondiciones**

- Ninguna (consulta).

**Reglas de negocio**

- Un usuario sólo ve lo propio, aunque la cátedra identifique por groupId.

**Evidencia asociada:** Consulta sin acceso cruzado entre usuarios (§11).

**Patrones:** Repository, DTO.

**Pruebas:** Usuario A no ve reservas de B; lista vacía; cátedra caída.

## CU-15 — Cancelar una reserva propia confirmada

- **Servicio:** turnos-service
- **Actor principal:** Usuario final
- **Actores secundarios:** Cátedra (REST, Kafka)
- **Ítem de la consigna:** 14 · **Sprint:** Sprint 3 · **Prioridad:** Alta
- **Rama:** `N-cu-15-cancelar-reserva`

**Precondiciones**

- Existe una reserva CONFIRMED del usuario.

**Disparador:** El usuario pulsa “Cancelar”.

**Flujo principal**

1. Verifica que la reserva sea del usuario y esté CONFIRMED.
2. Llama a POST /api/appointments/{id}/cancel.
3. Pasa la reserva a CANCELLED y registra el evento AppointmentCancelled recibido.
4. Informa a la app.

**Flujos alternativos / errores**

- Reserva de otro usuario → 403/404 sin revelar su existencia.
- Reserva no confirmada → 409/422.
- Repetición → idempotente (ya cancelada = OK).
- Error de la cátedra → se traduce y el estado local no se corrompe.

**Postcondiciones**

- La reserva queda cancelada y el horario vuelve a estar disponible.

**Reglas de negocio**

- Sólo reservas propias y confirmadas.

**Evidencia asociada:** Cancelación de una reserva propia (§11).

**Patrones:** State machine.

**Pruebas:** Cancelar OK, ajena, ya cancelada, estado inválido.

## CU-16 — Informar errores de integración y recuperar procesos pendientes

- **Servicio:** turnos-service
- **Actor principal:** turnos-service
- **Actores secundarios:** App KMP, cátedra
- **Ítem de la consigna:** 15 · **Sprint:** Sprint 3 · **Prioridad:** Media-Alta
- **Rama:** `N-cu-16-recuperacion-y-errores`

**Precondiciones**

- Existen procesos no finales.

**Disparador:** Reinicio del servicio, timeout o error de la cátedra.

**Flujo principal**

1. Al arrancar, un recuperador lee los procesos no finales de la base propia.
2. Para cada uno consulta a la cátedra y converge al estado correcto (o lo marca EXPIRED).
3. Los errores de la cátedra se traducen a un formato propio (problem+json) y a mensajes comprensibles en la app.
4. Los reintentos son acotados y con backoff; no hay ciclos infinitos.

**Flujos alternativos / errores**

- Cátedra caída → proceso queda pendiente y se reintenta luego.
- Datos inválidos o referencias inexistentes → 4xx con detalle.

**Postcondiciones**

- Ningún proceso queda en un estado imposible de recuperar.

**Reglas de negocio**

- Si catálogo no está disponible no se inician operaciones nuevas que lo necesiten.

**Evidencia asociada:** Robustez y recuperación (§8).

**Patrones:** Retry con backoff, Circuit breaker (opcional), Saga.

**Pruebas:** Reinicio con procesos pendientes; timeout; error traducido.

## CU-17 — Informar el estado y los errores de sincronización

- **Servicio:** catalogo-service
- **Actor principal:** Usuario final / Operadora
- **Actores secundarios:** —
- **Ítem de la consigna:** 4.1 · **Sprint:** Sprint 2 · **Prioridad:** Media
- **Rama:** `N-cu-17-estado-sincronizacion`

**Precondiciones**

- catalogo-service en ejecución.

**Disparador:** Se consulta GET /api/sync/status.

**Flujo principal**

1. Lee la fila de sync_state (versión local, última sincronización, estado).
2. Devuelve versión, fecha, estado (UP_TO_DATE, SYNCING, ERROR) y último error resumido.
3. La app puede mostrar “catálogo actualizado a la versión N”.

**Flujos alternativos / errores**

- Nunca sincronizó → estado NEVER_SYNCED.
- Sin token → 401.

**Postcondiciones**

- Ninguna (consulta).

**Reglas de negocio**

- No exponer secretos ni trazas internas en el error.

**Evidencia asociada:** Informar estado y errores (§4.1, item 15).

**Patrones:** State, DTO.

**Pruebas:** Cada estado; sin información sensible.

## CU-18 — Proteger la comunicación entre los dos servicios

- **Servicio:** catalogo-service
- **Actor principal:** turnos-service
- **Actores secundarios:** catalogo-service
- **Ítem de la consigna:** §9 · **Sprint:** Sprint 2-3 · **Prioridad:** Alta
- **Rama:** `N-cu-18-seguridad-entre-servicios`

**Precondiciones**

- Existe un mecanismo para emitir y validar JWT.

**Disparador:** turnos-service llama a un endpoint de catalogo-service.

**Flujo principal**

1. turnos-service obtiene/propaga un JWT (decisión a confirmar: JWT técnico o JWT del usuario).
2. catalogo-service valida firma, vigencia y permisos del token.
3. Autoriza la operación según el token; conserva trazabilidad (quién y para quién).
4. Si el token es inválido responde 401/403.

**Flujos alternativos / errores**

- Token vencido → 401.
- Token de usuario sin permiso para endpoints internos → 403.
- Llamada anónima → 401.

**Postcondiciones**

- Sólo llamadas autenticadas llegan al servicio de catálogo.

**Reglas de negocio**

- Secretos externalizados.
- CORS sólo para clientes permitidos.

**Evidencia asociada:** Rechazo de accesos no autenticados o no autorizados (§11).

**Patrones:** Adapter, Filter chain.

**Pruebas:** 401/403, token válido, token adulterado.

## Pantallas KMP ↔ casos de uso

| Pantalla | CU |
|---|---|
| Registro | CU-01 |
| Login | CU-02 |
| Búsqueda y filtros de profesionales | CU-08 |
| Estado del catálogo (opcional) | CU-17 |
| Elegir fecha y ver turnos libres | CU-09 |
| Confirmar turno (hold + cuenta regresiva) | CU-10, CU-11 |
| Ingresar teléfono (info adicional) | CU-12 |
| Resultado de la reserva | CU-13 |
| Mis reservas y cancelar | CU-14, CU-15 |
| Mensajes de error / reintento | CU-16 |

## Decisiones a confirmar con el profesor

- **¿Dónde viven los usuarios y quién emite el JWT?** — Propuesta: turnos-service gestiona usuarios y emite el JWT; catalogo-service sólo valida (firma compartida o clave pública). Alternativa: un tercer módulo (no permitido: son exactamente dos servicios).
- **¿Quién llama a la cátedra con la cuenta técnica?** — Propuesta: cada servicio con su propia configuración de la misma cuenta (catálogo: snapshot/Redis/CatalogUpdated; turnos: holds/ocupaciones/reservas/Kafka de reservas). El token técnico nunca sale de los servicios.
- **JWT entre servicios: ¿técnico o propagado del usuario?** — Propuesta: JWT técnico de servicio a servicio, con el id del usuario en un claim para trazabilidad. A justificar en la defensa.
- **Filtro “disponibilidad” del catálogo** — Propuesta: se resuelve con los horarios semanales locales (ej. «atiende el día X / franja Y»). La disponibilidad real (ocupaciones) se calcula en turnos (CU-09).
- **Máquina de estados de la reserva** — Propuesta en el diagrama 5. A validar los nombres de estados con el profesor.
- **Nivel de uso de JHipster** — Hoy el proyecto es Spring Boot puro con arquitectura hexagonal (permitido; JHipster es opcional). Confirmar que el contrato de usuario compatible con JHipster alcanza.