# EventBridge: eventos y tareas programadas

[Volver al índice](README.md)

## Qué es y cuándo usarlo

**Amazon EventBridge** es un **bus de eventos**. Los programas publican lo que les pasa ("se creó un pedido", "un usuario se registró") sin saber quién está interesado. Las **reglas** deciden qué eventos van a qué destinos.

**EventBridge Scheduler** ejecuta acciones a una hora o cada cierto tiempo, como un *cron* en la nube.

Úsalo para:

- Arquitecturas **orientadas a eventos**: los servicios se comunican sin conocerse.
- Reaccionar a eventos según su **contenido** (más flexible que los filtros de SNS).
- Tareas periódicas: copias de seguridad, informes diarios, limpiezas.

## Conceptos clave

| Concepto | Qué es |
|---|---|
| **Evento** | Un JSON que describe algo que pasó. Campos principales: `source`, `detail-type`, `detail` |
| **Bus** | Canal por donde viajan los eventos. Existe uno `default` y puedes crear los tuyos |
| **Regla** | Un **patrón**: si el evento lo cumple, se envía a los destinos |
| **Destino** (*target*) | A dónde va el evento: SQS, SNS, Lambda, Step Functions... (hasta 5 por regla) |
| **Transformador de entrada** | Cambia la forma del evento antes de entregarlo |
| **Programación** (*schedule*) | Acción que se ejecuta según `rate(...)`, `cron(...)` o una fecha |

Estructura de un evento:

```json
{
  "source": "tienda.pedidos",
  "detail-type": "PedidoCreado",
  "detail": { "pedido": "P-001", "cliente": "ana", "total": 250, "prioridad": "alta" }
}
```

## Antes de empezar

Terminal configurada ([manual 3](03-configurar-aws-cli.md)).

> **Limitación del simulador:** los filtros **numéricos** en patrones (`{"numeric": [">=", 100]}`) todavía no funcionan: la regla nunca coincide. Usa filtros por valor exacto, `prefix`, `anything-but` o `exists`.

---

## Ejercicio 1: un bus propio y una regla

```bash
aws events create-event-bus --name tienda
```

Cola de destino:

```bash
aws sqs create-queue --queue-name pedidos-urgentes
```

Crea `patron.json`. Solo queremos pedidos de **prioridad alta**:

```json
{
  "source": ["tienda.pedidos"],
  "detail-type": ["PedidoCreado"],
  "detail": { "prioridad": ["alta"] }
}
```

```bash
aws events put-rule --name pedidos-urgentes --event-bus-name tienda --event-pattern file://patron.json
```

Crea `destino.json`:

```json
[
  { "Id": "cola-urgentes", "Arn": "arn:aws:sqs:us-east-1:000000000000:pedidos-urgentes" }
]
```

```bash
aws events put-targets --rule pedidos-urgentes --event-bus-name tienda --targets file://destino.json
```

## Ejercicio 2: publicar eventos

Crea `eventos.json` con dos pedidos, uno urgente y otro normal:

```json
[
  {
    "EventBusName": "tienda",
    "Source": "tienda.pedidos",
    "DetailType": "PedidoCreado",
    "Detail": "{\"pedido\": \"P-001\", \"cliente\": \"ana\", \"total\": 250, \"prioridad\": \"alta\"}"
  },
  {
    "EventBusName": "tienda",
    "Source": "tienda.pedidos",
    "DetailType": "PedidoCreado",
    "Detail": "{\"pedido\": \"P-002\", \"cliente\": \"luis\", \"total\": 30, \"prioridad\": \"normal\"}"
  }
]
```

> `Detail` es un texto que contiene JSON, por eso sus comillas van escapadas con `\"`.

```bash
aws events put-events --entries file://eventos.json
```

```bash
aws sqs receive-message --queue-url http://localhost:4566/000000000000/pedidos-urgentes --max-number-of-messages 10 --query "Messages[].Body"
```

Solo llega **P-001**, con el evento completo (incluye `id`, `time`, `region`...).

## Ejercicio 3: probar un patrón sin publicar

Crea `evento-prueba.json`:

```json
{
  "id": "1", "account": "000000000000", "source": "tienda.pedidos",
  "time": "2026-11-07T10:00:00Z", "region": "us-east-1", "resources": [],
  "detail-type": "PedidoCreado",
  "detail": { "pedido": "P-003", "prioridad": "normal" }
}
```

```bash
aws events test-event-pattern --event-pattern file://patron.json --event file://evento-prueba.json
```

Responde `false`. Cambia `normal` por `alta` y responde `true`. Muy útil para depurar reglas.

## Ejercicio 4: más tipos de patrón

Empieza por:

```json
{ "detail": { "cliente": [{ "prefix": "an" }] } }
```

Cualquier valor menos uno:

```json
{ "detail": { "prioridad": [{ "anything-but": "baja" }] } }
```

Que el campo exista:

```json
{ "detail": { "cupon": [{ "exists": true }] } }
```

Varios valores (cualquiera vale):

```json
{ "source": ["tienda.pedidos", "tienda.devoluciones"] }
```

## Ejercicio 5: transformar el evento antes de entregarlo

A veces el destino solo necesita un texto. Crea `destino-transformado.json`:

```json
[
  {
    "Id": "cola-urgentes",
    "Arn": "arn:aws:sqs:us-east-1:000000000000:pedidos-urgentes",
    "InputTransformer": {
      "InputPathsMap": { "p": "$.detail.pedido", "c": "$.detail.cliente" },
      "InputTemplate": "\"Pedido urgente <p> del cliente <c>\""
    }
  }
]
```

```bash
aws events put-targets --rule pedidos-urgentes --event-bus-name tienda --targets file://destino-transformado.json
```

Vacía la cola, publica otra vez `eventos.json` y lee:

```bash
aws sqs purge-queue --queue-url http://localhost:4566/000000000000/pedidos-urgentes
```

```bash
aws events put-events --entries file://eventos.json
```

```bash
aws sqs receive-message --queue-url http://localhost:4566/000000000000/pedidos-urgentes --query "Messages[].Body"
```

Ahora llega: `Pedido urgente P-001 del cliente ana`.

## Ejercicio 6: tareas programadas con Scheduler

Vamos a dejar un recordatorio en una cola **cada minuto**.

```bash
aws sqs create-queue --queue-name recordatorios
```

Crea `programa.json`:

```json
{
  "Arn": "arn:aws:sqs:us-east-1:000000000000:recordatorios",
  "RoleArn": "arn:aws:iam::000000000000:role/scheduler-role",
  "Input": "{\"aviso\": \"Hacer copia de seguridad\"}"
}
```

```bash
aws scheduler create-schedule --name copia-periodica --schedule-expression "rate(1 minute)" --flexible-time-window Mode=OFF --target file://programa.json
```

Espera un minuto y mira la cola:

```bash
aws sqs receive-message --queue-url http://localhost:4566/000000000000/recordatorios
```

Expresiones útiles:

| Expresión | Significa |
|---|---|
| `rate(5 minutes)` | Cada 5 minutos |
| `rate(1 day)` | Cada día |
| `cron(0 8 * * ? *)` | Todos los días a las 8:00 (UTC) |
| `cron(0 18 ? * MON-FRI *)` | De lunes a viernes a las 18:00 |
| `at(2026-12-24T20:00:00)` | Una sola vez, en esa fecha |

Bórrala para que no siga enviando:

```bash
aws scheduler delete-schedule --name copia-periodica
```

## Ejercicio 7: publicar eventos desde Python

```python
import json
import boto3

CONFIG = dict(endpoint_url="http://localhost:4566", region_name="us-east-1",
              aws_access_key_id="test", aws_secret_access_key="test")
events = boto3.client("events", **CONFIG)

pedido = {"pedido": "P-010", "cliente": "marta", "total": 480, "prioridad": "alta"}
respuesta = events.put_events(Entries=[{
    "EventBusName": "tienda",
    "Source": "tienda.pedidos",
    "DetailType": "PedidoCreado",
    "Detail": json.dumps(pedido),
}])
print("Eventos fallidos:", respuesta["FailedEntryCount"])
```

## Borrar

Primero los destinos, luego la regla y el bus:

```bash
aws events remove-targets --rule pedidos-urgentes --event-bus-name tienda --ids cola-urgentes
```

```bash
aws events delete-rule --name pedidos-urgentes --event-bus-name tienda
```

```bash
aws events delete-event-bus --name tienda
```

---

## Reto

1. Crea un bus `usuarios` con eventos `UsuarioRegistrado` y `UsuarioEliminado`.
2. Regla 1: todos los `UsuarioRegistrado` van a una cola `bienvenidas`, transformados a `"Bienvenido, <nombre>"`.
3. Regla 2: los eventos de usuarios con `plan` igual a `premium` (de cualquier tipo) van a una cola `vip`.
4. Programa una limpieza `cron` que deje un mensaje en una cola `mantenimiento` todos los días a las 3:00.

## Chuleta

| Quiero... | Comando |
|---|---|
| Crear bus | `aws events create-event-bus --name B` |
| Crear regla | `aws events put-rule --name R --event-bus-name B --event-pattern file://patron.json` |
| Añadir destinos | `aws events put-targets --rule R --event-bus-name B --targets file://destino.json` |
| Publicar | `aws events put-events --entries file://eventos.json` |
| Probar patrón | `aws events test-event-pattern --event-pattern file://p.json --event file://e.json` |
| Ver reglas | `aws events list-rules --event-bus-name B` |
| Programar | `aws scheduler create-schedule --name N --schedule-expression "rate(...)" --flexible-time-window Mode=OFF --target file://t.json` |
