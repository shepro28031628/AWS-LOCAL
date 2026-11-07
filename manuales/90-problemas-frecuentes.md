# Problemas frecuentes

[Volver al índice](README.md)

## Al arrancar

**"failed to connect to the docker API" o "the system cannot find the file specified"**
Docker Desktop no está abierto o todavía está arrancando. Ábrelo y espera a ver **Engine running**.

**"port is already allocated" (puerto ocupado)**
Otro programa usa el puerto 4566 (por ejemplo, LocalStack u otra copia de este proyecto). Apágalo, o busca qué contenedor lo usa en Docker Desktop y detenlo.

**La primera vez tarda muchísimo**
Es normal: compila el proyecto y descarga dependencias. Puede tardar hasta 15 minutos con una conexión lenta. Las siguientes veces es rápido.

**No veo nada del proyecto en Docker Desktop**
Mientras se está construyendo no aparece en *Containers*. Mira la pestaña **Builds** para ver el progreso.

## Al usar la AWS CLI

**"Could not connect to the endpoint URL: https://s3.amazonaws.com/..."**
La terminal está hablando con la nube real de Amazon. Abriste una terminal nueva y olvidaste las variables del [manual 3](03-configurar-aws-cli.md). Vuelve a ejecutarlas.

**"Unable to locate credentials"**
Igual que el anterior: faltan las variables `AWS_ACCESS_KEY_ID` y `AWS_SECRET_ACCESS_KEY`.

**"aws: command not found" o "aws no se reconoce como un comando"**
No tienes la AWS CLI instalada, o tienes que cerrar y abrir la terminal después de instalarla. Mientras tanto, usa `docker compose exec floci aws ...`.

**Error de comillas o de JSON en PowerShell**
PowerShell trata las comillas de forma distinta. Guarda el JSON en un archivo y usa `file://archivo.json`, como en el [manual de DynamoDB](11-dynamodb.md).

## Al usar los servicios

**Mis buckets, tablas o colas desaparecieron**
Por defecto los datos viven en memoria y se borran al apagar. Para conservarlos, usa el modo `hybrid` (ver *Configuración útil* en el [README principal](../README.md#configuración-útil)).

**"BucketAlreadyExists" o "ResourceInUseException"**
Ya existe algo con ese nombre. Usa otro nombre o bórralo antes.

**La función Lambda tarda mucho la primera vez**
Está descargando la imagen de Python para Lambda. Solo pasa la primera vez.

**La consola web se queda en "Starting..."**
Espera unos segundos y recarga. Si no avanza, revisa los logs con `docker compose logs -f floci`.

## Último recurso: empezar de cero

Apaga todo, borra los datos y vuelve a encender:

```bash
docker compose down
```

```bash
docker compose up -d --build
```

[Volver al índice](README.md)
