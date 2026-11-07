# API Gateway: crear APIs REST

[Volver al índice](README.md)

## Qué es y cuándo usarlo

**Amazon API Gateway** publica una **API web** (URLs con `GET`, `POST`, `DELETE`...) y envía cada petición a tu código, normalmente una función [Lambda](40-lambda.md). Se encarga de las rutas, los parámetros y la seguridad.

La combinación **API Gateway + Lambda + DynamoDB** es la forma más común de crear un backend sin servidores. Es lo que construiremos aquí: una API de tareas pendientes.

| Tipo | Cuándo |
|---|---|
| **HTTP API** (v2) | Más simple y rápida. La que usamos en este manual |
| **REST API** (v1) | Más opciones (transformaciones, claves de API, caché). Más compleja |

## Conceptos clave

| Concepto | Qué es |
|---|---|
| **API** | El conjunto de rutas |
| **Ruta** (*route*) | Método + camino: `GET /tareas`, `POST /tareas`, `GET /tareas/{id}` |
| **Parámetro de ruta** | La parte variable entre llaves: `{id}` |
| **Integración** | A dónde va la petición: una Lambda (`AWS_PROXY`), otra URL... |
| **Stage** (etapa) | Una versión publicada de la API: `dev`, `prod` |
| **Evento de la API** | El JSON que recibe la Lambda: ruta, parámetros, cabeceras, cuerpo |

## Antes de empezar

- Terminal configurada ([manual 3](03-configurar-aws-cli.md)) y haber hecho el [manual de Lambda](40-lambda.md).
- Para probar la API usaremos **curl**:
  - **Windows (PowerShell):** escribe siempre **`curl.exe`** (en PowerShell, `curl` a secas es otro comando distinto).
  - **macOS / Linux:** `curl`.

La URL de tu API será:

```text
http://localhost:4566/execute-api/ID_DE_LA_API/prod/RUTA
```

---

## Ejercicio 1: la tabla y la función

```bash
aws dynamodb create-table --table-name tareas --attribute-definitions AttributeName=id,AttributeType=S --key-schema AttributeName=id,KeyType=HASH --billing-mode PAY_PER_REQUEST
```

Crea `api_tareas.py`. Una sola función atiende todas las rutas mirando `routeKey`:

```python
import base64
import json
import uuid
import boto3

tabla = boto3.resource("dynamodb").Table("tareas")


def respuesta(codigo, cuerpo):
    return {
        "statusCode": codigo,
        "headers": {"Content-Type": "application/json"},
        "body": json.dumps(cuerpo, default=str),
    }


def leer_cuerpo(event):
    cuerpo = event.get("body") or "{}"
    if event.get("isBase64Encoded"):
        cuerpo = base64.b64decode(cuerpo).decode("utf-8")
    return json.loads(cuerpo)


def handler(event, context):
    ruta = event["routeKey"]

    if ruta == "GET /tareas":
        return respuesta(200, tabla.scan()["Items"])

    if ruta == "POST /tareas":
        datos = leer_cuerpo(event)
        tarea = {"id": str(uuid.uuid4())[:8], "titulo": datos["titulo"], "hecha": False}
        tabla.put_item(Item=tarea)
        return respuesta(201, tarea)

    if ruta == "GET /tareas/{id}":
        item = tabla.get_item(Key={"id": event["pathParameters"]["id"]}).get("Item")
        return respuesta(200, item) if item else respuesta(404, {"error": "No existe"})

    if ruta == "DELETE /tareas/{id}":
        tabla.delete_item(Key={"id": event["pathParameters"]["id"]})
        return respuesta(204, {})

    return respuesta(400, {"error": f"Ruta no soportada: {ruta}"})
```

Puntos clave:

- La función devuelve `statusCode`, `headers` y `body` (texto): es el formato que API Gateway espera.
- `leer_cuerpo` tiene en cuenta que, según la cabecera `Content-Type`, el cuerpo puede llegar codificado en base64.
- `boto3.resource("dynamodb")` no necesita endpoint dentro de Lambda.

Comprime `api_tareas.zip` y crea la función:

```bash
aws lambda create-function --function-name api-tareas --runtime python3.12 --handler api_tareas.handler --zip-file fileb://api_tareas.zip --role arn:aws:iam::000000000000:role/lambda-role
```

## Ejercicio 2: crear la API

Crea la API y guarda su id:

**Windows (PowerShell):**

```bash
$API = aws apigatewayv2 create-api --name tareas-api --protocol-type HTTP --query ApiId --output text
```

**macOS / Linux:**

```bash
API=$(aws apigatewayv2 create-api --name tareas-api --protocol-type HTTP --query ApiId --output text)
```

Crea la integración con la Lambda:

**Windows (PowerShell):**

```bash
$INT = aws apigatewayv2 create-integration --api-id $API --integration-type AWS_PROXY --integration-uri arn:aws:lambda:us-east-1:000000000000:function:api-tareas --payload-format-version 2.0 --query IntegrationId --output text
```

**macOS / Linux:**

```bash
INT=$(aws apigatewayv2 create-integration --api-id $API --integration-type AWS_PROXY --integration-uri arn:aws:lambda:us-east-1:000000000000:function:api-tareas --payload-format-version 2.0 --query IntegrationId --output text)
```

Crea las cuatro rutas, todas hacia la misma integración:

```bash
aws apigatewayv2 create-route --api-id $API --route-key "GET /tareas" --target integrations/$INT
```

```bash
aws apigatewayv2 create-route --api-id $API --route-key "POST /tareas" --target integrations/$INT
```

```bash
aws apigatewayv2 create-route --api-id $API --route-key "GET /tareas/{id}" --target integrations/$INT
```

```bash
aws apigatewayv2 create-route --api-id $API --route-key "DELETE /tareas/{id}" --target integrations/$INT
```

Publica la API en la etapa `prod` (se redespliega sola con cada cambio):

```bash
aws apigatewayv2 create-stage --api-id $API --stage-name prod --auto-deploy
```

Muestra el id para formar la URL:

```bash
echo $API
```

## Ejercicio 3: usar la API

En los ejemplos, sustituye `ID` por el id de tu API.

**Crear tareas.** Crea `tarea.json`:

```json
{ "titulo": "Estudiar API Gateway" }
```

```bash
curl.exe -X POST http://localhost:4566/execute-api/ID/prod/tareas -H "Content-Type: application/json" -d "@tarea.json"
```

(En macOS / Linux, `curl` en vez de `curl.exe`.) Responde con la tarea creada y su `id`, código `201`.

**Listar todas.** Ábrelo directamente en el navegador:

```text
http://localhost:4566/execute-api/ID/prod/tareas
```

**Leer una:**

```bash
curl.exe http://localhost:4566/execute-api/ID/prod/tareas/ID_DE_LA_TAREA
```

**Borrar una** (`-i` muestra el código de respuesta, `204`):

```bash
curl.exe -i -X DELETE http://localhost:4566/execute-api/ID/prod/tareas/ID_DE_LA_TAREA
```

**Probar errores:** pide una tarea que no existe (`404` con `{"error": "No existe"}`) o una ruta que no creaste, como `/prod/otra` (`404 Not Found`, lo responde API Gateway sin llamar a Lambda).

## Ejercicio 4: ver qué recibe la Lambda

Añade al principio de `handler` la línea `print(json.dumps(event))`, actualiza el código (`update-function-code`), haz una petición y mira los logs:

```bash
aws logs filter-log-events --log-group-name /aws/lambda/api-tareas --query "events[].message"
```

Verás el evento completo: `routeKey`, `rawPath`, `pathParameters`, `queryStringParameters`, `headers`, `body`... Es la mejor forma de entender qué te llega.

## Ejercicio 5: probar la API desde Python

```python
import requests

BASE = "http://localhost:4566/execute-api/ID/prod"

nueva = requests.post(f"{BASE}/tareas", json={"titulo": "Probar desde Python"}).json()
print("Creada:", nueva)

print("Todas:", requests.get(f"{BASE}/tareas").json())

r = requests.delete(f"{BASE}/tareas/{nueva['id']}")
print("Borrada, código:", r.status_code)
```

(Instala `requests` con `pip install requests`.)

## Ver y borrar

```bash
aws apigatewayv2 get-routes --api-id $API --query "Items[].RouteKey"
```

```bash
aws apigatewayv2 delete-api --api-id $API
```

---

## Reto

1. Añade la ruta `PUT /tareas/{id}` que marque la tarea como hecha (`"hecha": true`) usando `update_item`.
2. Añade `GET /tareas?hecha=true` para filtrar (pista: `event["queryStringParameters"]`).
3. Valida que `titulo` no esté vacío en `POST`: si lo está, responde `400` con un mensaje claro.
4. Crea una página `index.html` en S3 (ver [web estática](10-s3.md)) que liste las tareas llamando a tu API con `fetch`.

## Chuleta

| Quiero... | Comando |
|---|---|
| Crear API | `aws apigatewayv2 create-api --name N --protocol-type HTTP` |
| Integración Lambda | `aws apigatewayv2 create-integration --api-id A --integration-type AWS_PROXY --integration-uri ARN_LAMBDA --payload-format-version 2.0` |
| Ruta | `aws apigatewayv2 create-route --api-id A --route-key "GET /ruta" --target integrations/I` |
| Publicar | `aws apigatewayv2 create-stage --api-id A --stage-name prod --auto-deploy` |
| URL | `http://localhost:4566/execute-api/A/prod/ruta` |
| Ver rutas | `aws apigatewayv2 get-routes --api-id A` |
| Borrar | `aws apigatewayv2 delete-api --api-id A` |
