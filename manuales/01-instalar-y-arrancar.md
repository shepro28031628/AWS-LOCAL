# 1. Instalar y arrancar

[Volver al índice](README.md)

## Qué necesitas

| Programa | Para qué | Dónde conseguirlo |
|---|---|---|
| **Docker Desktop** | Ejecuta AWS Nube Local | <https://www.docker.com/products/docker-desktop/> |
| **Git** | Descarga el proyecto | <https://git-scm.com/downloads> |
| **AWS CLI** (opcional) | Darle órdenes desde la terminal | <https://aws.amazon.com/cli/> |

> En Windows, Docker Desktop te pedirá activar **WSL 2** la primera vez. Acepta y reinicia si te lo pide.

## Paso 1: abrir Docker Desktop

Abre Docker Desktop y espera a que abajo a la izquierda aparezca **Engine running** (motor en marcha). Si no está en marcha, nada de lo siguiente funcionará.

## Paso 2: descargar el proyecto

Abre una terminal (en Windows: **PowerShell**) y ejecuta:

```bash
git clone https://github.com/shepro28031628/AWS-LOCAL.git
```

```bash
cd AWS-LOCAL
```

## Paso 3: encender

```bash
docker compose up -d --build
```

- La **primera vez** tarda varios minutos (entre 5 y 15): descarga y compila todo. Es normal.
- Las siguientes veces tarda segundos.

## Paso 4: comprobar que funciona

Abre en el navegador: <http://localhost:4566/>

Debes ver el panel de **AWS Nube Local** con un punto verde y la palabra **Ready** (listo).

En **Docker Desktop, pestaña Containers**, verás:

| Contenedor | Estado | ¿Está bien? |
|---|---|---|
| `floci` | Running (verde) | Sí, es el emulador |
| `consola-build` | Exited (0) | Sí, solo prepara la consola web y termina |

## Apagar y volver a encender

| Quiero... | Comando |
|---|---|
| Apagar | `docker compose down` |
| Encender otra vez | `docker compose up -d` |
| Ver qué está pasando (logs) | `docker compose logs -f floci` (sal con `Ctrl + C`) |

También puedes usar los botones de **Stop** y **Start** en Docker Desktop.

> **Ojo:** al apagar, los datos se borran (buckets, tablas, colas...). Si quieres conservarlos, mira la sección *Configuración útil* del [README principal](../README.md#configuración-útil).

**Siguiente:** [2. La consola web](02-consola-web.md)
