# Sistema Tire — Detección Temprana de Microsueños y Alertas en Flotas
## Informe Técnico de Entrega — Prototipo 2 (Etiqueta `v0.2.0`)

---

### Metadatos del Proyecto

- **Asignatura:** Ingeniería de Software II — Semestre 2026-2
- **Institución:** Universidad Nacional de Colombia — Sede Bogotá
- **Entrega:** Prototipo 2 (Etiqueta `v0.2.0`)
- **Video de Demostración:** `[PENDIENTE: enlace al video de demostración]`
- **Repositorios Oficiales:**
  - **Backend:** [`tire-backend`](https://github.com/ML-Ingesoft-II/tire-backend) (`v0.2.0`)
  - **Panel Web:** [`web-dashboard`](https://github.com/ML-Ingesoft-II/web-dashboard) (`v0.2.0`)
  - **Documentación y Artefactos:** [`tire-docs`](https://github.com/ML-Ingesoft-II/tire-docs) (`v0.2.0`)

### Integrantes del Equipo

| Integrante | Correo Institucional | Rol Principal en Prototipo 2 |
|---|---|---|
| **Ángel David Beltrán García** | `anbeltrang@unal.edu.co` | Backend Lead & QA Engineer |
| **Julián Esteban Cadena Rojas** | `jcadenar@unal.edu.co` | Domain Test Engineer & Model Specialist |
| **Óscar Iván Ulises Gutiérrez Palacios** | `osgutierrezp@unal.edu.co` | Software Architect & Backend Engineer |
| **Juan Camilo Vergara Tao** | `juvergarat@unal.edu.co` | Frontend Lead & Full-Stack Debugger |
| **Deivid Farid Ardila Herrera** | `deardilah@unal.edu.co` | Frontend QA & UI/UX Specialist |

---

## 1. Resumen Ejecutivo y Arquitectura del Sistema

Durante la fase del **Prototipo 2**, el equipo consolidó la integración horizontal y vertical entre los tres repositorios del proyecto, garantizando el flujo de punta a punta entre la interfaz del usuario, el intermediario BFF y el núcleo de negocio hexagonal:

```
[ Navegador / Cliente Web ]
         │ HTTPS / JSON
         ▼
[ web-dashboard: Next.js 14 BFF ]
   ├── Route Handlers (/api/*)
   ├── Cookies HttpOnly (JWT Session)
   └── Vitest + MSW
         │ HTTP / REST (OpenAPI 3.0.3)
         ▼
[ tire-backend: Spring Boot 3.2.5 Hexagonal ]
   ├── Inbound Adapters (REST Controllers)
   ├── Application Layer (4 Casos de Uso + Mockito)
   ├── Domain Layer (Modelos Puros + GoF State)
   ├── Outbound Adapters (JPA PostgreSQL/H2, Clima, JWT)
   └── JaCoCo (87.75 % Cobertura Dominio + Aplicación)
```

1. **Backend Hexagonal (`tire-backend`):** Monolito modular en Java 17 y Spring Boot 3.2.5 con estricta inversión de dependencias: `domain` no posee dependencias externas; `application` expone puertos de entrada y orquesta puertos de salida; e `infrastructure` implementa persistencia Spring Data JPA, integración con la API de clima Open-Meteo y seguridad stateless JJWT / BCrypt.
2. **Panel Web (`web-dashboard`):** Aplicación en Next.js 14 (App Router), React 18 y TypeScript que implementa el patrón **Backend for Frontend (BFF)**. Los componentes del cliente nunca contactan directamente la API externa ni almacenan tokens en `localStorage`; consumen route handlers locales (`/api/*`), los cuales gestionan los tokens JWT y refresh tokens en cookies seguras con atributo `HttpOnly`.
3. **Contrato OpenAPI 3.0.3:** Especificación formal compartida que desacopla el ciclo de vida de los clientes y el backend, validada por colecciones de Postman y pruebas automatizadas.

---

## 2. Matriz de Trazabilidad de Requisitos (HU y Criterios de Aceptación)

### Tabla 2.1: Trazabilidad de Historias de Usuario frente a Pruebas Automatizadas

| Historia de Usuario | Criterio de Aceptación (Formato Given-When-Then) | Prueba Automatizada Asociada | Capa / Herramienta | Resultado |
|---|---|---|---|:---:|
| **HU-02:** Iniciar sesión (Despachador) | **Dado** un despachador registrado con credenciales válidas en la base de datos, **cuando** envía su correo y contraseña a `/auth/sesion`, **entonces** el sistema responde `200 OK` con un `accessToken`, `refreshToken` y la información de su flota. | `loginDespachadorDevuelveDatosDeFlota` | Integración API (`MockMvc` + H2) | **Pasa** |
| **HU-02:** Rechazo de credenciales / sesión | **Dado** una petición dirigida a un recurso protegido (`/conductores`) sin cabecera de autenticación, **cuando** el filtro de seguridad intercepta la solicitud, **entonces** el sistema rechaza la petición con `401 Unauthorized` bajo formato RFC 9457. | `listarConductoresSinTokenDevuelve401` | Integración API (`MockMvc` + H2) | **Pasa** |
| **HU-05:** Registrar conductor en flota | **Dado** un despachador autenticado de una flota, **cuando** registra un conductor con nombres, documento y correo únicos, **entonces** el sistema responde `201 Created`, asocia el perfil a la flota y asigna la contraseña inicial calculada. | `crearConductorValidoDevuelve201` | Integración API (`MockMvc` + H2) | **Pasa** |
| **HU-05:** Validación de datos al registrar | **Dado** un despachador autenticado, **cuando** envía el formulario de registro con nombres en blanco o documento inválido, **entonces** la API responde `400 Bad Request` indicando los errores de validación de los campos. | `crearConductorDatosInvalidosDevuelve400` | Integración API (`MockMvc` + H2) | **Pasa** |
| **HU-05:** Conflicto por documento duplicado | **Dado** un conductor previamente registrado con un número de documento, **cuando** se intenta registrar otro conductor con el mismo documento, **entonces** el sistema responde `409 Conflict` y preserva la integridad de la base de datos. | `crearConductorDocumentoDuplicadoDevuelve409` | Integración API (`MockMvc` + H2) | **Pasa** |
| **HU-05:** Control de acceso por rol (403) | **Dado** un usuario autenticado con rol `CONDUCTOR`, **cuando** intenta crear un nuevo conductor en `/conductores`, **entonces** el sistema responde `403 Forbidden` impidiendo la operación administrativa. | `conductorCreandoConductorDevuelve403` | Integración API (`MockMvc` + H2) | **Pasa** |
| **HU-06:** Consultar detalle de conductor | **Dado** un despachador autenticado y un conductor registrado en su flota, **cuando** consulta `/conductores/{id}`, **entonces** el sistema responde `200 OK` con la información completa del perfil y estado del conductor. | `obtenerConductorExistenteDevuelve200` | Integración API (`MockMvc` + H2) | **Pasa** |
| **HU-06:** Edición parcial de conductor | **Dado** un despachador autenticado, **cuando** envía una actualización parcial vía `PATCH /conductores/{id}`, **entonces** el sistema responde `200 OK` y persiste los cambios de teléfono y licencia. | `actualizarConductorParcialmente_conTokenDespachador_devuelve200` | Integración API (`MockMvc` + H2) | **Pasa** |
| **HU-06:** Desactivación de conductor | **Dado** un conductor sin viajes activos en curso, **cuando** el despachador ejecuta `DELETE /conductores/{id}`, **entonces** el sistema desactiva el perfil y responde `204 No Content`. | `desactivarConductorSinViajeActivo_devuelve204` | Integración API (`MockMvc` + H2) | **Pasa** |

---

## 3. Estrategia e Inventario de Pruebas del Backend

### Tabla 3.1: Inventario Completo de Pruebas en `tire-backend`

| Nivel / Tipo de Prueba | Ubicación en el Código | Herramientas / Frameworks | Cantidad | Comando de Ejecución | Estado |
|---|---|---|:---:|---|:---:|
| **Unitarias del Dominio** | `src/test/java/com/unal/tire/domain/` | JUnit 5 puro (sin Spring) | **47** | `.\mvnw.cmd test` / `./mvnw test` | **Pasan** (100 %) |
| **Casos de Uso con Dobles** | `src/test/java/com/unal/tire/application/service/` | JUnit 5 + Mockito 5 (`@Mock`, `@InjectMocks`) | **43** | `.\mvnw.cmd test` / `./mvnw test` | **Pasan** (100 %) |
| **Integración de API / Persistencia** | `src/test/java/com/unal/tire/TireBackendIntegrationTest.java` | Spring Boot Test, MockMvc, H2 en memoria | **15** | `.\mvnw.cmd test` / `./mvnw test` | **Pasan** (100 %) |
| **TOTAL BACKEND** | Toda la suite automatizada | **Java 17, Spring Boot 3.2.5** | **105** | `.\mvnw.cmd test` / `./mvnw test` | **0 fallos, 0 errores, 0 omitidos** |

### Distribución Detallada de Pruebas Unitarias

- **Dominio (47 pruebas):**
  - `ViajeTest` (15 pruebas): invariantes de inicio, registro de telemetría, eventos de fatiga y finalización de viaje.
  - `EstadoAlertaStateTest` (9 pruebas): transiciones de máquina de estados GoF (`NORMAL` $\to$ `PRECAUCION` $\to$ `CRITICO`).
  - `EventoFatigaTest` (8 pruebas): validación de umbrales temporales (1.5 s microsueño, 5.0 s rostro no visible) y respuestas.
  - `UbicacionTest` (4 pruebas): inmutabilidad de Value Object y cálculo de distancia geodésica con fórmula de Haversine.
  - `PerfilConductorTest` (3 pruebas): actualización de perfiles y vehículos asignados.
  - `AlertaCriticaTest` (3 pruebas): ciclo de atención por despachador, registro de notas e invariantes de estado.
  - `UsuarioTest` (3 pruebas): cambio de contraseña, flags de expiración y roles funcionales.
  - `FlotaTest` (2 pruebas): encapsulación e identidad de la entidad Flota.

- **Aplicación con Mockito (43 pruebas):**
  - `AutenticacionServiceTest` (13 pruebas): login con credenciales válidas/inválidas, emisión y renovación de refresh tokens, flujo de recuperación de contraseña (solicitud y confirmación) y aislamiento de tokens vencidos.
  - `GestionConductoresServiceTest` (11 pruebas): creación con unicidad de correo/documento (`verify(..., never())`), consultas multitenant por flota, actualización parcial y desactivación bloqueada si hay viaje en curso.
  - `GestionViajesServiceTest` (11 pruebas): inicio con consulta a API meteorológica (`ConsultorClimaPort`), escalamiento a alerta crítica ante evento no confirmado, finalización de viaje y confirmación de alerta en cabina.
  - `GestionAlertasServiceTest` (8 pruebas): agregación de métricas de flota en tiempo real, atención de alertas críticas con despachador y notas, prevención de doble atención e historial paginado de eventos.

---

## 4. Cobertura de Código y Calidad (Reporte JaCoCo)

### Resumen Oficial de Cobertura

La configuración del plugin `org.jacoco:jacoco-maven-plugin:0.8.12` en `pom.xml` se integró para instrumentar las clases del paquete `com.unal.tire.**`, excluyendo la infraestructura de persistencia JPA, DTOs de transporte y configuración Spring.

Al ejecutar `./mvnw test` o `./mvnw clean test`, JaCoCo genera el reporte interactivo en:
**`target/site/jacoco/index.html`**

```
[PENDIENTE: captura Figura 4.1 de target/site/jacoco/index.html]
```
*Figura 4.1: Reporte interactivo de cobertura JaCoCo generado en `target/site/jacoco/index.html` mostrando 87.75 % de cobertura de líneas.*

### Tabla 4.1: Métricas Cuantitativas de Cobertura (JaCoCo)

| Capa Arquitectónica | Líneas Totales | Líneas Cubiertas | Cobertura de Líneas (%) | Ramas Totales | Ramas Cubiertas | Cobertura de Ramas (%) |
|---|:---:|:---:|:---:|:---:|:---:|:---:|
| **Capa de Dominio (`com.unal.tire.domain`)** | 441 | 353 | **80.05 %** | 122 | 92 | **75.41 %** |
| **Capa de Aplicación (`com.unal.tire.application`)** | 465 | 442 | **95.05 %** | 207 | 128 | **61.84 %** |
| **CONSOLIDADO DOMINIO + APLICACIÓN** | **906** | **795** | **87.75 %** | **329** | **220** | **66.87 %** |

> **Cumplimiento de la Meta:** El proyecto supera ampliamente el umbral mínimo del **70 % de cobertura de líneas** exigido para la entrega del Prototipo 2, alcanzando un **87.75 % global**, con un **80.05 %** en la capa de dominio y un **95.05 %** en la capa de casos de uso de aplicación.

### Análisis Técnico de Clases con Menor Cobertura

1. **Dominio — `PerfilConductor` (60.0 % de líneas):**
   - *Causa técnica:* La entidad cuenta con métodos mutadores y accesores de campos secundarios opcionales (`setFotoUrl`, `setLicenciaVigencia`, `setTelefonoEmergencia`, `setTipoVehiculo`) que no son requeridos en el flujo crítico del MVP pero forman parte del modelo extensible de datos. Asimismo, excepciones de dominio como `AccesoDenegadoDominioException` registran 0 % debido a que el control de acceso es interceptado y resuelto previamente por la capa de seguridad (`JwtAuthenticationFilter` y `CustomAccessDeniedHandler`).
2. **Aplicación — `GestionAlertasService` (49.4 % de ramas):**
   - *Causa técnica:* El método `historialEventos` contiene lógica de consulta con múltiples parámetros opcionales (`fechaInicio`, `fechaFin`, `tipoEvento`, `conductorId`, `soloPendientes`, `sortDirection`). La combinatoria completa de evaluación de nulidad de filtros produce un alto número de ramas condicionales ($2^5$ caminos lógicos). Los casos de uso principales fueron verificados sin incurrir en pruebas redundantes sobre caminos puramente delegatorios.

---

## 5. Verificación de Criterios del Cliente Web (`web-dashboard`)

El panel del despachador en Next.js 14 fue contrastado contra los cinco criterios obligatorios de la guía del Prototipo 2:

- **(a) Inicio de sesión real + `Authorization: Bearer` + guardado seguro:**
  - Login contra `POST /auth/sesion`. El proxy almacena el JWT en una cookie con bandera `HttpOnly`, inaccesible por scripts cliente. En cada petición saliente al backend se inyecta automáticamente el encabezado `Authorization: Bearer <token>`. Cuenta con soporte de renovación transparente mediante `POST /auth/renovar` ante respuestas 401.
- **(b) Flujo principal del MVP y operación adicional:**
  - Recurso principal `Conductor`: listar (`GET /conductores`), consultar detalle (`GET /conductores/{id}`), registrar (`POST /conductores`), editar (`PATCH /conductores/{id}`) y desactivar (`DELETE /conductores/{id}`).
  - Operación adicional de negocio: atención de alerta crítica (`PATCH /alertas/{id}/atender`) con registro del despachador responsable y notas.
- **(c) Manejo exhaustivo de estados de interfaz:**
  - Estados visuales desacoplados en `src/lib/estados.ts`: `cargando`, `vacío`, `error` y `éxito` en vistas de flota, conductores e historial.
  - Respuestas `401 Unauthorized` borran la sesión y redirigen a `/?motivo=sesion-expirada`.
  - Respuestas `403 Forbidden` muestran aviso de permisos insuficientes.
  - Caída del backend es gestionada respondiendo `503 Service Unavailable` con opción de reintentar.
- **(d) Arquitectura desacoplada en capas:**
  - Cadena estricta: `Vista (React Component) → api() (lib/client.ts) → Route Handler (/api/*) → backendAutenticado() (lib/server/backend.ts) → tire-backend (Spring Boot)`. La interfaz jamás invoca directamente al backend.
- **(e) Configuración desacoplada del código:**
  - La URL del backend se define mediante la variable de entorno `TIRE_API_URL` en `.env.local` (plantilla en `.env.example`), sin direcciones hardcodeadas en código fuente.

---

## 6. Validación de Flujos con Breakpoints y Hallazgo de Depuración

Se llevó a cabo una sesión de depuración integral de extremo a extremo inspeccionando concurrentemente el navegador (Chrome DevTools), el servidor BFF de Next.js (VS Code Node Debugger) y el backend en Spring Boot (VS Code Java Debugger):

- **13 Puntos de validación (breakpoints):**
  - Flujo Login: `bp01` (vista cliente), `bp02` (controlador backend), `bp03` (servidor Next.js BFF) y `bp04` (**logpoint** en cliente backend).
  - Flujo Crear Conductor: `bp05` (vista), `bp06` (Next.js route handler), `bp07` (controlador REST), `bp08` (caso de uso de aplicación) y `bp09` (adaptador JPA persistencia).
  - Flujo 401: `bp10` (**condicional** `token == null` en filtro JWT), `bp11` (`CustomAuthenticationEntryPoint`), `bp12` (limpieza de cookies en Next.js) y `bp13` (**condicional** `res.status === 401` en navegador).
- **Mecanismos de paso:** Ejecución de *Step Over* (F10) en `GestionConductoresService.crear` y *Step Into* (F11) hacia `generarPasswordInicial`.
- **Prueba depurada en backend:** `TireBackendIntegrationTest.crearConductorValidoDevuelve201` depurada paso a paso validando la pila de llamadas MockMvc y el contexto transaccional.

### Hallazgo de Depuración (503 vs. 504 por Tiempo Agotado)

- **Comportamiento inesperado:** Se asumía originalmente que el código `503` del proxy de Next.js solo ocurría ante la caída total del backend. Durante la sesión de depuración con el backend pausado por más de 5 minutos, la llamada `fetch` de Node.js agotó el tiempo de espera, emitiendo un error 503 genérico ("No se pudo conectar"). Sin embargo, el backend completó la transacción al reanudar y guardó el registro; un posterior reintento del usuario generaba un conflicto `409` desconcertante.
- **Corrección aplicada:** Se modificó el módulo de fallas del proxy (`web-dashboard/src/lib/server/fallas.ts`, commit `a10e561`) para discriminar explícitamente el tiempo de espera agotado (`504 Gateway Timeout`) de la indisponibilidad de conexión (`503 Service Unavailable`), alertando al despachador de verificar el estado antes de reintentar. Se incluyó la prueba de regresión `src/lib/server/fallas.test.ts` (commit `01ab511`).

### Tabla 6.1: Mejoras y Evolución desde el Prototipo 1

| Componente / Repositorio | Cambio / Mejora Implementada | Evidencia (Commit / PR) | Beneficio Técnico |
|---|---|---|---|
| `tire-backend` | Configuración de JaCoCo en Maven (`pom.xml`) | Commits `3d933d9`, `612f178` (tag `v0.2.0`) | Medición y auditoría automatizada de cobertura de código ($\ge 70\%$). |
| `tire-backend` | 43 pruebas unitarias con dobles Mockito en `application.service` | Commits `3d933d9`, `612f178` / PR #2 | Verificación rigurosa de reglas de negocio e interacciones de puertos. |
| `tire-backend` | Ampliación de suite de integración API a 15 tests MockMvc | Commits `3d933d9`, `612f178` | Cobertura integral de endpoints consumidos por el panel web. |
| `tire-backend` | Actualización de endpoints REST en README y Postman (`GET /eventos`) | Commit `612f178` (tag `v0.2.0`) | Alineación total de documentación y contrato de pruebas de caja negra. |
| `web-dashboard` | Discriminación de errores de proxy 503 vs 504 | Commits `a10e561`, `01ab511` / PR #5 | Manejo robusto de timeouts y prevención de reintentos duplicados. |
| `web-dashboard` | Configuración de Vitest y MSW con pruebas de cliente HTTP | Commits `ed38c4f`, `5279a7a` / PR #4 | Validación automatizada del cliente web sin acoplamiento a red. |
| `web-dashboard` | Estados visuales de carga, error, vacío y sesión expirada | Commits `60eb7ac`, `80f4b14` / PR #3 | Cumplimiento estricto del criterio (c) de UX del Prototipo 2. |
| `web-dashboard` | Creación de plantilla `.env.example` y actualización de README con scripts de tests | Commit `3fe0ee3` (tag `v0.2.0`) | Cumplimiento de criterios de configuración desacoplada y reproducibilidad. |
| `tire-docs` | Sincronización completa de diagramas de clases, API y componentes | Commits `9819194`, `480c187` / PR #27 | Consistencia 100 % entre documentación arquitectónica y código real. |

---

## 7. Aportes Individuales y Trabajo en Equipo

### Tabla 7.1: Aportes Individuales al Prototipo 2

| Integrante | Rol / Especialidad | Aportes Principales y Commits Clave en el Prototipo 2 |
|---|---|---|
| **Ángel David Beltrán García** | Backend Lead & QA | - Configuración del plugin JaCoCo en `tire-backend/pom.xml` con exclusiones selectivas.<br>- Implementación de 43 pruebas unitarias con dobles Mockito en los cuatro servicios de aplicación (`GestionConductoresServiceTest`, `AutenticacionServiceTest`, `GestionViajesServiceTest`, `GestionAlertasServiceTest`).<br>- Ampliación de `TireBackendIntegrationTest` con 5 nuevos escenarios MockMvc.<br>- Implementación del módulo desacoplado de demostración OCP y patrones GoF (`tire_alert_service`).<br>- Actualización de la documentación REST en README y colección Postman (Commits `3d933d9`, `612f178`, tag `v0.2.0`). |
| **Julián Esteban Cadena Rojas** | Domain Testing & Modeling | - Diseño e implementación de la suite de 41 pruebas unitarias puras en `tire-backend/src/test/java/com/unal/tire/domain/` (PR #2, commit `b49d260`).<br>- Sincronización y actualización de diagramas Mermaid (`diagrama_clases.mmd`, `diagrama_clases_aplicacion.mmd`, `diagrama_api.mmd`).<br>- Modelado de invariantes de microsueños y umbrales de fatiga. |
| **Óscar Iván Ulises Gutiérrez Palacios** | Arquitectura & Backend | - Alineación del modelo hexagonal con la especificación OpenAPI 3.0.3 (PR #1).<br>- Implementación del runner de datos semilla (`SeedDataRunner`) con flota, conductores y eventos para pruebas de desarrollo.<br>- Elaboración de instrucciones de despliegue local con Docker Compose y PostgreSQL. |
| **Juan Camilo Vergara Tao** | Frontend Lead & Debugging | - Arquitectura de BFF en Next.js 14 (`web-dashboard`), route handlers proxy y manejo de cookies HttpOnly.<br>- Implementación de la máquina de estados de UI (`cargando`, `vacío`, `error`, `sin permiso`) y aviso de sesión vencida (PR #3).<br>- Conducción de la sesión de depuración multicapa con 13 breakpoints, condicionales y logpoint, y resolución del hallazgo 503/504 (PR #5, commits `a10e561`, `01ab511`). |
| **Deivid Farid Ardila Herrera** | Frontend QA & UI/UX | - Configuración del arnés de pruebas de interfaz con Vitest y Mock Service Worker (MSW) (PR #4, commits `5279a7a`, `ed38c4f`).<br>- Aplicación del sistema de diseño Tire, iconos SVG y diseño accesible sin dependencia de modo claro forzado.<br>- Revisión y validación de PRs y aseguramiento de calidad del frontend. |

---

## 8. Conclusiones y Próximos Pasos (Hacia el Prototipo 3)

1. **Alineación de Calidad:** La suite de pruebas de `tire-backend` alcanza 105 pruebas exitosas con un 87.75 % de cobertura de líneas, cumpliendo holgadamente los estándares del curso.
2. **Robustez de la Solución:** La depuración interactiva de flujos end-to-end permitió detectar y corregir anomalías en el proxy BFF antes de la entrega de producción.
3. **Próximos Pasos:**
   - Incorporar la aplicación móvil del conductor (`React Native`) conectando el modelo de visión embebido en el dispositivo.
   - Migrar la gestión de telemetría hacia eventos asíncronos y websockets para soporte de monitoreo en vivo con latencia mínima.
