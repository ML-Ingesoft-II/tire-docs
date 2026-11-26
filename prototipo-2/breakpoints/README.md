# Validación de flujos con breakpoints (Prototipo 2)

Evidencia de la sección 6 de la guía. Se depuraron a la vez el backend (Spring Boot, VS Code),
el servidor de Next.js (route handlers `/api/*`, VS Code) y el navegador (Chrome DevTools),
de modo que una sola acción del usuario se sigue de punta a punta.

- 13 breakpoints en cliente y backend, en distintas capas.
- 2 condicionales: bp10 (`token == null`) y bp13 (`res.status === 401`).
- 1 logpoint: bp04.
- Las capturas están en [`capturas/`](capturas/). Contraseñas tapadas; los tokens solo se ven truncados.

## Tabla de puntos de validación

| # | Flujo | Capa | Ubicación | Qué se verifica (valor esperado) | Valor observado |
|---|---|---|---|---|---|
| [bp01](capturas/bp01-login-cliente.png) | Login | Cliente: vista (navegador) | `web-dashboard/src/app/page.tsx:40` | El correo escrito llega a `enviar()` | `email = "laura.gomez@tandinos.co"` |
| [bp02](capturas/bp02-login-backend.png) | Login | Backend: controlador | `AuthController.java:53` | El caso de uso devuelve un `accessToken` no vacío | `respuesta.accessToken = "eyJhbGciOiJIUzUx…"`, `expiracionSegundos = 3600` |
| [bp03](capturas/bp03-login-next.png) | Login | Cliente: route handler (servidor Next) | `src/app/api/auth/sesion/route.ts:30` | El JWT llega antes de guardarse en cookie HttpOnly y el rol es despachador | `token.accessToken = "eyJhbGciOiJIU…"`, `token.rol = "DESPACHADOR"` |
| [bp04](capturas/bp04-login-logpoint.png) | Login | Cliente: cliente del backend (**logpoint**) | `src/lib/server/backend.ts:84` | La siguiente petición lleva `Authorization: Bearer <JWT>` | `Petición a /flota/estado con Bearer eyJhbGciOiJIUzU...` |
| [bp05](capturas/bp05-crear-cliente.png) | Crear conductor | Cliente: vista (navegador) | `src/components/ConductorForm.tsx:49` | El `payload` lleva los datos del formulario | `nombres "Prueba"`, `numeroDocumento "1012345678"`, `placaVehiculo "XYZ987"` |
| [bp06](capturas/bp06-crear-next.png) | Crear conductor | Cliente: route handler (servidor Next) | `src/app/api/conductores/route.ts:14` | El cuerpo se reenvía al backend sin cambios | `body = '{"nombres":"Prueba","apellidos":…'` |
| [bp07](capturas/bp07-crear-controlador.png) | Crear conductor | Backend: controlador | `ConductorController.java:46` | El DTO llega validado y el despachador sale del JWT | `request.numeroDocumento = "1012345678"`, `principal.email = "laura.gomez@tandinos.co"` |
| [bp08](capturas/bp08-crear-caso-de-uso.png) | Crear conductor | Backend: caso de uso | `GestionConductoresService.java:58` | El comando pasó las reglas (documento y correo no repetidos) | `comando.correo = "prueba.depuracion@tandinos.co"` |
| [bp09](capturas/bp09-crear-repositorio.png) | Crear conductor | Backend: repositorio | `PerfilConductorPersistenceAdapter.java:32` | El agregado llega al repositorio con su id | `perfil.nombres = "Prueba"`, `perfil.activo = true`, `perfil.usuarioId = "07fce34d-…"` |
| [bp10](capturas/bp10-401-filtro-condicional.png) | 401 | Backend: filtro JWT (**condicional** `token == null`) | `JwtAuthenticationFilter.java:70` | Se detiene solo si la petición no trae token | `token = null`, `request.getRequestURI() = "/alertas"` |
| [bp11](capturas/bp11-401-entry-point.png) | 401 | Backend: seguridad (punto del rechazo) | `CustomAuthenticationEntryPoint.java:36` | Spring Security rechaza con 401 desde `ExceptionTranslationFilter` | `authException = InsufficientAuthenticationException` ("Full authentication is required…") |
| [bp12](capturas/bp12-401-next.png) | 401 | Cliente: cliente del backend (servidor Next) | `src/lib/server/backend.ts:104` | Con 401 y sin refresh válido se borra la sesión | `res.status = 401`, `path = "/alertas"`, `token = undefined` |
| [bp13](capturas/bp13-401-cliente-condicional.png) | 401 | Cliente: cliente HTTP (navegador, **condicional** `res.status === 401`) | `src/lib/client.ts:70` | El cliente recibe 401 y lleva al inicio de sesión | `res.status = 401` → `/?motivo=sesion-expirada` ([bp13b](capturas/bp13b-401-login-sesion-expirada.png)) |

Rutas de backend relativas a `tire-backend/src/main/java/com/unal/tire/`; rutas de cliente, a `web-dashboard/`.

## Pasos y prueba depurada

- **Step over / step into:** en `GestionConductoresService.crear`, F10 avanza a la línea 59 con el `id` ya
  generado ([captura](capturas/paso-step-over.png)) y F11 entra a `generarPasswordInicial`
  ([captura](capturas/paso-step-into.png)).
- **Prueba depurada (6d):** `TireBackendIntegrationTest.crearConductorValidoDevuelve201` en modo debug se
  detiene en `GestionConductoresService.java:58`; la pila de llamadas muestra la prueba y `MockMvc.perform`
  debajo del código de producción ([captura](capturas/prueba-depurada.png)).

## Hallazgo

- **Suposición equivocada:** se asumía que el 503 del proxy de Next solo ocurría con el backend caído.
- **Qué mostró el depurador:** con el backend detenido unos 7 minutos entre bp06 y bp09, el `fetch` del
  servidor de Next agotó su espera (5 minutos por defecto en Node) y el panel mostró "No se pudo conectar
  con el servidor" ([captura](capturas/hallazgo-503.png)). Sin embargo, el backend terminó la operación y
  el conductor quedó guardado ([captura](capturas/hallazgo-guardado.png)); un reintento habría dado 409.
- **Corrección:** el proxy distingue el tiempo agotado (504, con el aviso de revisar antes de reintentar)
  de la falta de conexión (503). Commit `a10e561` en `web-dashboard` (`src/lib/server/fallas.ts`).
- **Prueba que evita la regresión:** `src/lib/server/fallas.test.ts`, commit `01ab511`.
