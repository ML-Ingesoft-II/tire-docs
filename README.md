# Tire

Sistema para detectar microsueños en conductores con la cámara del teléfono y avisar a tiempo, antes de que ocurra un accidente.

Proyecto de Ingeniería de Software II, Universidad Nacional de Colombia, 2026-2. Este repositorio reúne la documentación general del proyecto y los entregables de cada prototipo. No contiene código de la aplicación.

## Equipo

| Integrante | Correo | Área |
|---|---|---|
| Deivid Farid Ardila Herrera | deardilah@unal.edu.co | Frontend |
| Ángel David Beltrán García | anbeltrang@unal.edu.co | Backend |
| Óscar Iván Ulises Gutiérrez Palacios | osgutierrezp@unal.edu.co | Arquitectura |
| Juan Camilo Vergara Tao | juvergarat@unal.edu.co | Frontend |
| Julián Esteban Cadena Rojas | jcadenar@unal.edu.co | Modelo de visión |

## Qué es Tire

Tire analiza el rostro del conductor con la cámara del teléfono, sin enviar video ni imágenes a la nube, y dispara una alarma en la cabina cuando detecta un microsueño. Si el conductor no la confirma, el estado de alerta pasa a Crítico y el despachador de su empresa recibe un aviso en un panel web.

Usuarios del MVP:

- **Conductor de flota:** usa la app móvil durante el viaje.
- **Conductor particular:** mismo rol que el anterior, pero sin flota ni despachador.
- **Despachador:** supervisa su flota y atiende las alertas críticas desde el panel web.

El copiloto y el administrador de seguridad quedan para una versión posterior.

## Repositorios

El proyecto está organizado como polyrepo: cada componente vive en su propio repositorio, con su propia documentación técnica. Este repositorio no los contiene, los conecta con enlaces.

| Componente | Repositorio | Tecnología |
|---|---|---|
| Documentación | [tire-docs](https://github.com/ML-Ingesoft-II/tire-docs) | Markdown |
| Backend y API | [tire-backend](https://github.com/ML-Ingesoft-II/tire-backend) | Java, Spring Boot, PostgreSQL |
| App del conductor | Por definir | React Native, TypeScript |
| Panel del despachador | [web-dashboard](https://github.com/ML-Ingesoft-II/web-dashboard) | Next.js, TypeScript |
| Modelo de visión | Por definir | Por definir |
| Laboratorio 3 (servicio de alertas) | [lab3-tire-alert-service](https://github.com/ML-Ingesoft-II/lab3-tire-alert-service) | TypeScript, Node.js |

## Cómo ejecutar el sistema en local

El MVP se levanta en dos partes: el **backend** (Spring Boot + PostgreSQL) con
datos de ejemplo y el **panel del despachador** (Next.js). Clona cada repositorio
como carpetas hermanas.

### Requisitos

- JDK 17, Docker y Docker Compose (backend).
- Node.js 18.18+ (panel web).

### 1. Backend con Docker + datos de ejemplo

En [`tire-backend`](https://github.com/ML-Ingesoft-II/tire-backend):

```bash
# 1. Variables de entorno
cp .env.example .env

# 2. Base de datos PostgreSQL en contenedor
docker compose up -d

# 3. Backend con el perfil "seed" (crea el esquema y carga datos de demostración)
SPRING_PROFILES_ACTIVE=seed ./mvnw spring-boot:run
```

El backend queda en `http://localhost:8080` (Swagger UI en
`http://localhost:8080/swagger-ui.html`).

El perfil `seed` ejecuta `SeedDataRunner`, que **recrea el esquema** y siembra la
flota **Transportes Andinos**, la despachadora **Laura Gómez** y 24 conductores
con viajes e incidentes de ejemplo. Es idempotente, pero al usar `create-drop`
borra los datos en cada arranque con ese perfil, así que úsalo solo en desarrollo.
Para arrancar sin sembrar: `./mvnw spring-boot:run`.

> Todo en contenedores: `docker build -t tire-backend .` construye la imagen del
> backend (ver el README de `tire-backend` para el detalle).

### 2. Panel del despachador (Next.js)

En [`web-dashboard`](https://github.com/ML-Ingesoft-II/web-dashboard):

```bash
# La URL del backend se lee de .env.local (TIRE_API_URL=http://localhost:8080)
npm install
npm run dev
```

El panel queda en `http://localhost:3000`. Inicia sesión con las credenciales
sembradas:

| Campo | Valor |
|---|---|
| Correo | `laura.gomez@tandinos.co` |
| Contraseña | `Password123` |

## Documentación

- **Informe del Prototipo 1:** [ENLACE: informe en Google Docs]. Visión, requisitos, experiencia de usuario, arquitectura y diseño.
- **[Hoja de requisitos e historias de usuario](https://docs.google.com/spreadsheets/d/1j88lSVzQi-lEuXvqIYyZnxGGtjXeoJL9Z6ZRn1VVo5E/edit?usp=sharing):** historias, criterios de aceptación y story map. Una copia está en [prototipo-1](prototipo-1/tire-historias-de-usuario.xlsx).
- **[Tire Backlog](https://github.com/orgs/ML-Ingesoft-II/projects/1/views/1):** historias del MVP con su prioridad, puntos y estado.
- **Wireframes** (descargar y abrir en el navegador): [app móvil](prototipo-1/wireframes-app-movil.html) y [panel web](prototipo-1/wireframes-dashboard-web.html).
- **Mockups interactivos** (descargar y abrir en el navegador): [app móvil](prototipo-1/mockup-app-movil.html), [panel web](prototipo-1/mockup-dashboard-web.html) y [panel web en celular](prototipo-1/mockup-dashboard-web-movil.html).
- **Contrato de la API:** [openapi.yaml](https://github.com/ML-Ingesoft-II/tire-backend/blob/HEAD/openapi.yaml) y [documentación en Postman](https://documenter.getpostman.com/view/59414595/2sBYHYtcf5).
- **[Diagrama de componentes](https://drive.google.com/drive/folders/1MKtnm3Z2HIfqIGsD7eh9P0Vk2-dXgZWh?usp=sharing).**

## Alcance del MVP

El MVP incluye cuentas y sesión, gestión de conductores por el despachador, viajes, detección de ojos cerrados, alarma en cabina, confirmación del conductor, estados de alerta (Normal, Precaución y Crítico), envío de ubicación y estado, estado de la flota, y alerta crítica en el panel con historial.

Quedan para una versión posterior la calibración facial, el guardado de datos sin señal, la alerta al copiloto y el aviso por SMS a contactos de emergencia.

## Arquitectura

La arquitectura objetivo es de microservicios. Para el MVP se implementa un único backend con arquitectura hexagonal, dividido en cuatro módulos (acceso, flota, viajes y telemetría, y alertas), con base de datos PostgreSQL.

La app analiza el rostro en el teléfono, sin conexión a internet, y el panel consulta periódicamente el estado de la flota. Ambos clientes se comunican con el backend a través de un contrato OpenAPI, que les permite avanzar con un servidor simulado mientras el backend no esté listo.

Para el detalle de componentes:

- **Arquitectura actual (lo que se implementó en el Prototipo 1):** [ARQUITECTURA_ACTUAL.md](prototipo-1/arquitectura/ARQUITECTURA_ACTUAL.md).
- **Arquitectura objetivo (visión de microservicios para fases posteriores):** [ARQUITECTURA_ANTERIOR.md](prototipo-1/arquitectura/ARQUITECTURA_ANTERIOR.md).
