# Notas — Docker + SonarQube (13-oct-2026)

Resumen de la sesión del 13-oct-2026 en la que se levantó el stack de Docker de Cuidarte+ y se corrigió el escaneo de SonarQube, que solo analizaba 2 archivos CSS en vez de todo el sistema.

## 1. Levantar el stack

Al ejecutar `docker compose up`, el frontend entraba en reinicio constante con `exec format error`.

**Causa:** la imagen `nginx:alpine` (tag "latest" de alpine) estaba rota en esta máquina — incluso `/bin/sh` fallaba dentro del contenedor. La imagen `alpine` a secas sí funcionaba, y una versión fija de nginx también.

**Fix:** en [FRONTEND/Dockerfile](FRONTEND/Dockerfile#L34) se cambió `FROM nginx:alpine` por `FROM nginx:1.29-alpine`. Fijar la versión además hace el build más predecible (evita que un re-pull futuro de `latest` rompa el build de nuevo).

**Resultado — los 4 servicios arriba y respondiendo:**

| Servicio | URL | Estado |
|---|---|---|
| Frontend | http://localhost:3333 | 200 OK |
| Backend (API) | http://localhost:4444 | 200 OK |
| PostgreSQL | `localhost:15442` | healthy |
| SonarQube | http://localhost:9000 | UP |

Comandos de referencia:
```bash
docker compose up -d              # levantar todo
docker compose down                # apagar todo
docker compose --profile tools run --rm scanner   # correr el análisis de SonarQube (no se levanta con el resto)
```

## 2. SonarQube solo escaneaba 2 archivos CSS

Al correr el scanner, el dashboard solo mostraba 2 archivos CSS analizados — ningún JS/JSX de `BACKEND` ni `FRONTEND`.

**Causas, dos problemas distintos:**

1. **El analizador de JavaScript se colgaba.** Recorre todo el proyecto buscando `package.json`/`tsconfig.json`, y terminaba entrando a `node_modules` (~270 MB) pese a estar excluido en `sonar-project.properties`. Como el proyecto vive dentro de OneDrive, leer esos archivos a través de Docker colgó Docker Desktop dos veces durante la sesión (hubo que reiniciarlo manualmente; los contenedores de la app volvieron solos).
2. **`sonar.sources` solo cubría `BACKEND/src` y `FRONTEND/src`**, dejando fuera Dockerfiles y archivos de configuración.

**Fix:**
- En [sonar-project.properties](sonar-project.properties): el escaneo ahora cubre todo `BACKEND` y todo `FRONTEND` (incluye Dockerfiles y configs), excluyendo `node_modules`, `dist`, `build`, `coverage` y `uploads`.
- En [docker-compose.yml](docker-compose.yml): el servicio `scanner` monta los dos `node_modules` (`BACKEND` y `FRONTEND`) como volúmenes vacíos, para que el analizador físicamente no pueda entrar ahí aunque lo intente.

**Resultado tras el fix — de 2 a 65 archivos escaneados:**

| Métrica | Antes | Ahora |
|---|---|---|
| Archivos | 2 | 65 |
| Líneas de código | 53 | 10.245 (JS 10.140 · CSS 53 · Dockerfile 29 · HTML 23) |
| Bugs | 1 | 1 |
| Vulnerabilidades | 0 | 6 |
| Code smells | 0 | 100 |
| Duplicación | — | 14% |

Dashboard: http://localhost:9000/dashboard?id=isy1102-cuidarteplus-equipo011

## Pendiente / a seguir de cerca

- **6 vulnerabilidades y 100 code smells** detectados — falta revisarlos uno por uno en el dashboard de SonarQube.
- **Docker Desktop se colgó dos veces** leyendo node_modules desde OneDrive durante el análisis — si vuelve a pasar, confirmar primero que los volúmenes vacíos de `node_modules` sigan declarados en `docker-compose.yml` antes de re-correr el scanner.
- Para volver a escanear en el futuro: `docker compose --profile tools run --rm scanner` (mismo comando de siempre, ya con la config corregida).
