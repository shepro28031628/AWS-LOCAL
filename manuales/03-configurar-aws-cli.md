# 3. Configurar la AWS CLI

[Volver al índice](README.md)

La **AWS CLI** es el programa oficial para dar órdenes a AWS desde la terminal. Hay que decirle que hable con tu nube local y no con la de Amazon.

## Opción A: variables de entorno (recomendada)

Ejecútalas **cada vez que abras una terminal nueva**.

**Windows (PowerShell):**

```bash
$env:AWS_ENDPOINT_URL="http://localhost:4566"; $env:AWS_ACCESS_KEY_ID="test"; $env:AWS_SECRET_ACCESS_KEY="test"; $env:AWS_DEFAULT_REGION="us-east-1"
```

**macOS / Linux:**

```bash
export AWS_ENDPOINT_URL=http://localhost:4566 AWS_ACCESS_KEY_ID=test AWS_SECRET_ACCESS_KEY=test AWS_DEFAULT_REGION=us-east-1
```

Qué significa cada una:

| Variable | Valor | Para qué |
|---|---|---|
| `AWS_ENDPOINT_URL` | `http://localhost:4566` | Manda las órdenes a tu nube local |
| `AWS_ACCESS_KEY_ID` | `test` | "Usuario" (cualquier valor sirve) |
| `AWS_SECRET_ACCESS_KEY` | `test` | "Contraseña" (cualquier valor sirve) |
| `AWS_DEFAULT_REGION` | `us-east-1` | Región de AWS |

## Opción B: un perfil guardado (para no repetirlo)

Ejecuta una sola vez:

```bash
aws configure --profile local
```

Responde `test`, `test`, `us-east-1` y `json`. Luego añade el endpoint al perfil:

```bash
aws configure set endpoint_url http://localhost:4566 --profile local
```

A partir de ahí, añade `--profile local` a cada comando:

```bash
aws s3 ls --profile local
```

## Comprobar que funciona

```bash
aws sts get-caller-identity
```

Si responde con un `Account` igual a `000000000000`, estás conectado a tu nube local.

## ¿No tienes la AWS CLI instalada?

El contenedor ya trae una. Pon `docker compose exec floci` delante de cada comando, desde la carpeta del proyecto:

```bash
docker compose exec floci aws s3 ls
```

> Con esta opción, los archivos que uses (por ejemplo para subirlos a S3) tienen que estar **dentro** del contenedor, no en tu PC. Para los manuales es más cómodo instalar la AWS CLI.

**Siguiente:** [S3: almacenamiento de archivos](10-s3.md)
