# Lambda: funciones sin servidor

[Volver al índice](README.md)

## Qué es y cuándo usarlo

**AWS Lambda** ejecuta tu código **solo cuando hace falta**: cuando llega una petición, un mensaje, un archivo o un evento. No hay servidor encendido ni que mantener, y en AWS real solo pagas por los milisegundos de ejecución.

En AWS Nube Local cada ejecución corre en un **contenedor Docker real** con el runtime oficial de AWS (Python, Node.js, Java...).

Úsalo para:

- Reaccionar a eventos: un archivo en S3, un mensaje en SQS, un cambio en DynamoDB.
- Backends de APIs (con [API Gateway](41-api-gateway.md)).
- Tareas programadas (con [EventBridge Scheduler](32-eventbridge.md)).
- Pasos de un flujo de trabajo (con [Step Functions](42-step-functions.md)).

## Conceptos clave

| Concepto | Qué es |
|---|---|
| **Función** | Tu código más su configuración |
| **Handler** | La función que Lambda llama: `archivo.funcion` |
| **Evento** (`event`) | Los datos de entrada (JSON) |
| **Contexto** (`context`) | Información de la ejecución: nombre, tiempo restante... |
| **Runtime** | Lenguaje y versión: `python3.12`, `nodejs20.x`... |
| **Variables de entorno** | Configuración que lee el código sin cambiarlo |
| **Timeout / memoria** | Máximo de segundos y MB que puede usar |
| **Disparador** (*event source mapping*) | Hace que Lambda lea sola de una cola o un stream |
| **Versión / alias** | Fotos inmutables del código y nombres que apuntan a ellas (`prod`, `test`) |
| **Function URL** | Dirección HTTP propia para llamar a la función |

## Antes de empezar

- Terminal configurada ([manual 3](03-configurar-aws-cli.md)).
- La **primera** ejecución descarga la imagen de Python para Lambda (unos cientos de MB). Las siguientes son rápidas.
- Para crear los ZIP:
  - **Windows (PowerShell):** `Compress-Archive -Path archivo.py -DestinationPath archivo.zip -Force`
  - **macOS / Linux:** `zip archivo.zip archivo.py`
- El rol `arn:aws:iam::000000000000:role/lambda-role` no se comprueba en local: cópialo tal cual.

---

## Ejercicio 1: función con configuración

Crea `saludo.py`:

```python
import os


def handler(event, context):
    nombre = event.get("nombre", "mundo")
    idioma = os.environ.get("IDIOMA", "es")
    saludo = "Hola" if idioma == "es" else "Hello"
    print(f"Saludando a {nombre} en {idioma}")
    return {"mensaje": f"{saludo}, {nombre}!", "funcion": context.function_name}
```

Comprímelo en `saludo.zip` y crea la función con variables de entorno, 10 segundos de timeout y 256 MB:

```bash
aws lambda create-function --function-name saludo --runtime python3.12 --handler saludo.handler --zip-file fileb://saludo.zip --role arn:aws:iam::000000000000:role/lambda-role --environment "Variables={IDIOMA=en}" --timeout 10 --memory-size 256
```

Crea `evento.json`:

```json
{ "nombre": "Ana" }
```

```bash
aws lambda invoke --function-name saludo --payload fileb://evento.json salida.json
```

Mira `salida.json`: `{"mensaje": "Hello, Ana!", "funcion": "saludo"}`.

Cambia la configuración **sin tocar el código**:

```bash
aws lambda update-function-configuration --function-name saludo --environment "Variables={IDIOMA=es}"
```

Ejecuta otra vez: ahora dice `Hola, Ana!`.

## Ejercicio 2: ver los logs

Todo lo que imprimes con `print` va a **CloudWatch Logs**, al grupo `/aws/lambda/NOMBRE`:

```bash
aws logs filter-log-events --log-group-name /aws/lambda/saludo --query "events[].message"
```

Solo las líneas que contengan una palabra:

```bash
aws logs filter-log-events --log-group-name /aws/lambda/saludo --filter-pattern Saludando --query "events[].message"
```

Más sobre logs en el [manual de CloudWatch](60-cloudwatch.md).

## Ejercicio 3: actualizar código, versiones y alias

Edita `saludo.py` (por ejemplo, añade un emoji o cambia el texto), vuelve a crear el ZIP y súbelo:

```bash
aws lambda update-function-code --function-name saludo --zip-file fileb://saludo.zip
```

Cuando el código esté listo para producción, **publica una versión** (una foto que ya no cambia):

```bash
aws lambda publish-version --function-name saludo
```

Crea un alias `prod` que apunte a la versión 1:

```bash
aws lambda create-alias --function-name saludo --name prod --function-version 1
```

Llama al alias:

```bash
aws lambda invoke --function-name saludo:prod --payload fileb://evento.json salida.json
```

Puedes seguir cambiando el código (`$LATEST`) sin afectar a `prod`. Para pasar a producción la versión nueva: `publish-version` y `update-alias --function-version 2`.

## Ejercicio 4: procesar una cola SQS automáticamente

Crea `procesar_pedidos.py`:

```python
import json


def handler(event, context):
    for registro in event["Records"]:
        pedido = json.loads(registro["body"])
        print(f"Procesando pedido {pedido['id']} de {pedido['cliente']} por {pedido['total']}")
    return {"procesados": len(event["Records"])}
```

Crea el ZIP, la cola y la función:

```bash
aws sqs create-queue --queue-name pedidos-lambda
```

```bash
aws lambda create-function --function-name procesar-pedidos --runtime python3.12 --handler procesar_pedidos.handler --zip-file fileb://procesar_pedidos.zip --role arn:aws:iam::000000000000:role/lambda-role
```

Conéctalas: Lambda leerá la cola sola, en lotes de hasta 5 mensajes:

```bash
aws lambda create-event-source-mapping --function-name procesar-pedidos --event-source-arn arn:aws:sqs:us-east-1:000000000000:pedidos-lambda --batch-size 5
```

Crea `pedido.json`:

```json
{"id": "P-1", "cliente": "ana", "total": 120}
```

```bash
aws sqs send-message --queue-url http://localhost:4566/000000000000/pedidos-lambda --message-body file://pedido.json
```

Espera unos segundos y mira los logs:

```bash
aws logs filter-log-events --log-group-name /aws/lambda/procesar-pedidos --filter-pattern Procesando --query "events[].message"
```

Verás `Procesando pedido P-1 de ana por 120`. El mensaje desapareció de la cola: si la función termina bien, Lambda lo borra; si falla, vuelve a la cola.

## Ejercicio 5: reaccionar a cambios en DynamoDB

Necesitas la tabla `notas` con Streams activados del [manual de DynamoDB](11-dynamodb.md) (ejercicio 8).

Crea `cambios_tabla.py`:

```python
def handler(event, context):
    for registro in event["Records"]:
        tipo = registro["eventName"]
        claves = registro["dynamodb"]["Keys"]
        print(f"Cambio {tipo} en {claves}")
    return {"cambios": len(event["Records"])}
```

```bash
aws lambda create-function --function-name cambios-tabla --runtime python3.12 --handler cambios_tabla.handler --zip-file fileb://cambios_tabla.zip --role arn:aws:iam::000000000000:role/lambda-role
```

Busca el ARN del stream:

```bash
aws dynamodb describe-table --table-name notas --query Table.LatestStreamArn --output text
```

Conecta la función (pega el ARN):

```bash
aws lambda create-event-source-mapping --function-name cambios-tabla --event-source-arn PEGA_EL_ARN_DEL_STREAM --starting-position LATEST
```

Inserta o modifica algo en `notas` y revisa los logs de `/aws/lambda/cambios-tabla`: verás `Cambio INSERT ...` o `Cambio MODIFY ...`.

> Así se construyen auditorías, notificaciones ("avisa al alumno cuando cambie su nota") o copias a otros sistemas.

## Ejercicio 6: una URL propia para la función

Crea `web.py`:

```python
import json


def handler(event, context):
    params = event.get("queryStringParameters") or {}
    nombre = params.get("nombre", "visitante")
    return {
        "statusCode": 200,
        "headers": {"Content-Type": "application/json"},
        "body": json.dumps({"mensaje": f"Hola {nombre}, desde una Function URL"}),
    }
```

```bash
aws lambda create-function --function-name web --runtime python3.12 --handler web.handler --zip-file fileb://web.zip --role arn:aws:iam::000000000000:role/lambda-role
```

```bash
aws lambda create-function-url-config --function-name web --auth-type NONE
```

La respuesta trae una `FunctionUrl` como `http://291ab89b....lambda-url.us-east-1.floci:4566/`. **Cambia `floci` por `localhost`** y ábrela en el navegador añadiendo un parámetro:

```text
http://291ab89b....lambda-url.us-east-1.localhost:4566/?nombre=Ana
```

> Para APIs con varias rutas, mejor usa [API Gateway](41-api-gateway.md).

## Ejercicio 7: usar otros servicios desde dentro de la función

El código de una Lambda puede usar `boto3` **sin configurar el endpoint**: AWS Nube Local se lo inyecta automáticamente. Por ejemplo:

```python
import boto3

s3 = boto3.client("s3")


def handler(event, context):
    buckets = [b["Name"] for b in s3.list_buckets()["Buckets"]]
    return {"buckets": buckets}
```

Gracias a eso, el mismo código funciona igual en tu nube local y en AWS real.

## Borrar

```bash
aws lambda delete-function --function-name saludo
```

Para quitar un disparador: `aws lambda list-event-source-mappings` y `aws lambda delete-event-source-mapping --uuid UUID`.

---

## Reto

1. Crea una función `miniaturas` que se ejecute cada vez que se sube un archivo a un bucket `fotos` (pista: notificación de S3 a una cola SQS, como en el [manual de S3](10-s3.md), y esa cola como disparador de la función).
2. La función debe escribir en los logs el nombre del archivo y su tamaño.
3. Guarda en una tabla DynamoDB `fotos-registro` el nombre y la fecha de cada archivo procesado.

## Chuleta

| Quiero... | Comando |
|---|---|
| Crear | `aws lambda create-function --function-name F --runtime python3.12 --handler archivo.funcion --zip-file fileb://f.zip --role arn:aws:iam::000000000000:role/lambda-role` |
| Ejecutar | `aws lambda invoke --function-name F --payload fileb://evento.json salida.json` |
| Actualizar código | `aws lambda update-function-code --function-name F --zip-file fileb://f.zip` |
| Configuración | `aws lambda update-function-configuration --function-name F --environment "Variables={K=V}" --timeout 30` |
| Logs | `aws logs filter-log-events --log-group-name /aws/lambda/F` |
| Disparador | `aws lambda create-event-source-mapping --function-name F --event-source-arn ARN` |
| URL | `aws lambda create-function-url-config --function-name F --auth-type NONE` |
| Versión / alias | `aws lambda publish-version` / `aws lambda create-alias` |
| Listar | `aws lambda list-functions --query "Functions[].FunctionName"` |
