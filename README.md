# NPS (Net Promoter Score) System

Sistema completo para la gestión de encuestas NPS (Net Promoter Score). Este repositorio monorepo contiene tanto la API backend como la aplicación frontend, además de la orquestación mediante Docker para desplegar ambos servicios junto a la base de datos.

## Arquitectura y Tecnologías

El proyecto está dividido en dos aplicaciones principales independientes dockerizadas, comunicadas a través de una red de contenedores:

- **Frontend (`nps.frontend`):** Aplicación SPA (Single Page Application) construida con **Angular 21** y estilizada con **TailwindCSS 4**. Servida mediante **Nginx** en Docker.
- **Backend (`nps.backend`):** API RESTful desarrollada con **.NET 10**. Utiliza SQL Server como base de datos y Dapper como ORM de alto rendimiento.
- **Base de Datos (`sqlserver`):** Contenedor Microsoft SQL Server 2022.

## Estructura del Proyecto

```text
/NPS
├── nps.backend/        # Código fuente del backend (.NET 10, Clean Architecture)
├── nps.frontend/       # Código fuente del frontend (Angular 21)
├── docker-compose.yml  # Orquestador del ecosistema completo
└── README.md           # Este archivo
```

## Flujo de Trabajo y Funcionalidades del Sistema

La herramienta aborda el flujo completo de evaluación de lealtad de clientes bajo las siguientes reglas y características principales implementadas en el código:

1. **Gestión de Accesos:**
   - Acceso seguro mediante Login y contraseña.
   - Seguridad con bloqueo temporal de cuenta tras **3 intentos fallidos**.
   - Comunicación API fuertemente securizada con **JWT (JSON Web Tokens)** generados en cada solicitud.
   - Renovación segura de credenciales mediante el uso de **Refresh Tokens**.

2. **Perfiles Funcionales (Roles):**
   - **Votante:** Es el único que, al ingresar, observa de manera central y resaltada la pregunta de medición. Clasifica su lealtad seleccionando (no digitando) en una escala estricta del **0 al 10**. Su votación queda bloqueada/registrada para hacerse solo **una única vez**. 
   - **Administrador:** Diseñado para supervisión. Al entrar, es el responsable de visualizar en tiempo real los tableros con los resultados recolectados.

3. **Almacenamiento Confiable:** Toda la información recolectada se consolida consistentemente en una base de datos relacional **SQL Server**.

## ¿Cómo funciona el Control de Inactividad de Sesión (5 minutos)?

El sistema incluye un estricto mecanismo de seguridad que cierra la sesión si el usuario no interactúa con la plataforma durante 5 minutos. *¿Cómo sucede esto por detrás sin afectar el uso normal?*

Para lograr esta protección de forma transparente para el usuario final, el sistema trabaja en equipo (Frontend y Backend):

**Del lado del Navegador (Frontend - Angular):**
- El navegador actúa como un "cronómetro" interno (`activity.service.ts`). Cada vez que haces clic en un menú o navegas entre pantallas (sin contar procesos invisibles de sistema), la plataforma nota tu actividad mediante un interceptor (`activity.interceptor.ts`) y vuelve a reiniciar silenciosamente tu cuenta regresiva de 5 minutos.
- Si el reloj llega a cero, el sistema bloquea inmediatamente la pantalla por seguridad, destruye los accesos guardados y te redirige a una pantalla especial informativa, usando un guardia protector (`session-expired.guard.ts`).

**Del lado del Servidor (Backend - .NET):**
- El servidor es el juez final, en caso de que un usuario malintencionado intente pausar el cronómetro de su navegador. Cada vez que tú le hablas al servidor y este se da cuenta que eres tú, anota silenciosamente tu última hora de conexión real (`SessionActivityMiddleware.cs`).
- Constantemente renovamos tu sesión de fondo pidiendo un nuevo pase de acceso (`RefreshTokenCommandHandler.cs`). Durante este cambio de pase, el servidor verifica: *"A ver, ¿cuándo fue la última vez que hizo algo real? Si pasaron más de 5 minutos, este pase de acceso queda cancelado"*.
- Si te excedes de los 5 minutos, el servidor destruye irrevocablemente tu sesión vigente y te obliga a iniciar sesión con contraseña nuevamente como medida de protección anti-fraudes.

## Buenas Prácticas y Patrones Globales

1. **Dockerización (Containerization):** Uso de `Dockerfile` multi-stage tanto para backend como frontend, permitiendo builds optimizados y livianos para producción.
2. **Docker Compose:** Definición de multi-contenedor mediante `docker-compose.yml`, manejando la persistencia de datos (volúmenes de SQL Server), redes privadas, inyección de variables de entorno seguras cruzadas y control de dependencias mediante `healthcheck`.

## Requisitos Previos

- Docker 
- Docker Compose v2+

## Ejecución Local

Para levantar todo el ecosistema localmente:

```bash
# 1. Abre este directorio raíz en la terminal
cd /ruta/a/NPS

# 2. Levanta los contenedores y constrúyelos
docker-compose up --build -d
```

- **Frontend:** Estará disponible en `http://localhost:4200`
- **Backend Swagger/Scalar:** Estará disponible en `http://localhost:5000/scalar/v1` (o la ruta predefinida para tu UI de OpenAPI).
- **Base de Datos:** Estará accesible mediante SSMS o DataGrid en `localhost:1433` usando el usuario `sa` y la contraseña `TuPasswordFuerte123!`.
