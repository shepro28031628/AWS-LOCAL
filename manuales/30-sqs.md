# SQS: colas de mensajes

[Volver al índice](README.md)

## Qué es y cuándo usarlo

**Amazon SQS** (*Simple Queue Service*) es una cola: un programa (**productor**) deja mensajes y otro (**consumidor**) los recoge y procesa cuando puede. Así los programas no dependen uno del otro: si el consumidor se cae o va lento, los mensajes esperan en la cola sin perderse.

Ejemplos: procesar pedidos, enviar correos, generar informes, redimensionar imágenes.

## Conceptos clave

| Concepto | Qué es |
|---|---|
| **Cola estándar** | Muy rápida. El orden no está garantizado y, rara vez, un mensaje puede llegar dos veces |
| **Cola FIFO** | *First In, First Out*: orden garantizado y sin duplicados. Su nombre termina en `.fifo` |
| **Tiempo de visibilidad** | Cuando un consumidor recibe un mensaje, este se oculta unos segundos. Si no se borra a tiempo, vuelve a la cola (por si el consumidor falló) |
| **Receipt handle** | El "ticket" de una recepción, necesario para borrar el mensaje |
| **Cola de mensajes fallidos** (DLQ) | Cola donde van los mensajes que fallan demasiadas veces, para revisarlos aparte |
| **Long polling** | El consumidor espera hasta N segundos a que llegue algo, en vez de preguntar sin parar |
| **Retraso** (*delay*) | El mensaje no se puede recibir hasta pasados unos segundos |

## Antes de empezar

Terminal configurada ([manual 3](03-configurar-aws-cli.md)). Las URLs de las colas tienen la forma `http://localhost:4566/000000000000/NOMBRE`.

> Las respuestas de SQS muestran las URLs con `https://floci:4566/...` (el nombre interno del emulador). Para tus comandos usa siempre `http://localhost:4566/...`.

---

## Ejercicio 1: el ciclo de vida de un mensaje

```bash
aws sqs create-queue --queue-name pedidos
```

Enviar:

```bash
aws sqs send-message --queue-url http://localhost:4566/000000000000/pedidos --message-body "Pedido 1: dos pizzas"
```

Recibir:

```bash
aws sqs receive-message --queue-url http://localhost:4566/000000000000/pedidos
```

Copia el `ReceiptHandle` de la respuesta y borra el mensaje para marcarlo como procesado:

```bash
aws sqs delete-message --queue-url http://localhost:4566/000000000000/pedidos --receipt-handle "PEGA_AQUÍ_EL_RECEIPT_HANDLE"
```

**Experimento:** envía otro mensaje, recíbelo y **no** lo borres. Espera 30 segundos (el tiempo de visibilidad por defecto) y vuelve a recibir: el mensaje reaparece. Así SQS garantiza que nada se pierde si un consumidor falla a mitad.

## Ejercicio 2: enviar en lote y esperar mensajes (long polling)

Crea `lote.json`:

```json
[
  { "Id": "1", "MessageBody": "Pedido A" },
  { "Id": "2", "MessageBody": "Pedido B" },
  { "Id": "3", "MessageBody": "Pedido C" }
]
```

```bash
aws sqs send-message-batch --queue-url http://localhost:4566/000000000000/pedidos --entries file://lote.json
```

Recibe hasta 10 mensajes esperando hasta 10 segundos si la cola está vacía:

```bash
aws sqs receive-message --queue-url http://localhost:4566/000000000000/pedidos --max-number-of-messages 10 --wait-time-seconds 10
```

Mensaje con retraso de 5 segundos:

```bash
aws sqs send-message --queue-url http://localhost:4566/000000000000/pedidos --message-body "Llego tarde" --delay-seconds 5
```

Vaciar la cola entera:

```bash
aws sqs purge-queue --queue-url http://localhost:4566/000000000000/pedidos
```

## Ejercicio 3: cola de mensajes fallidos (DLQ)

Queremos que, si un pago falla **2 veces**, pase a una cola `pagos-errores` para revisarlo.

Crea primero la cola de errores:

```bash
aws sqs create-queue --queue-name pagos-errores
```

Crea `config-pagos.json` (visibilidad de 2 segundos para que el ejercicio sea rápido):

```json
{
  "RedrivePolicy": "{\"deadLetterTargetArn\":\"arn:aws:sqs:us-east-1:000000000000:pagos-errores\",\"maxReceiveCount\":\"2\"}",
  "VisibilityTimeout": "2"
}
```

```bash
aws sqs create-queue --queue-name pagos --attributes file://config-pagos.json
```

Envía un pago:

```bash
aws sqs send-message --queue-url http://localhost:4566/000000000000/pagos --message-body "Pago 99"
```

Simula un consumidor que falla: recibe el mensaje **sin borrarlo**, espera 3 segundos y repite. Haz esto **3 veces**:

```bash
aws sqs receive-message --queue-url http://localhost:4566/000000000000/pagos --attribute-names ApproximateReceiveCount
```

Las dos primeras veces llega con `ApproximateReceiveCount` 1 y 2. La tercera, la cola está vacía: el mensaje se movió a la cola de errores.

```bash
aws sqs receive-message --queue-url http://localhost:4566/000000000000/pagos-errores
```

> Así evitas que un mensaje "envenenado" bloquee el sistema reintentándose para siempre.

## Ejercicio 4: cola FIFO (orden y sin duplicados)

Crea `config-fifo.json`:

```json
{ "FifoQueue": "true", "ContentBasedDeduplication": "true" }
```

```bash
aws sqs create-queue --queue-name turnos.fifo --attributes file://config-fifo.json
```

En las colas FIFO cada mensaje necesita un **grupo** (`--message-group-id`). Los mensajes del mismo grupo salen en orden.

```bash
aws sqs send-message --queue-url http://localhost:4566/000000000000/turnos.fifo --message-body "Turno 1" --message-group-id caja1
```

```bash
aws sqs send-message --queue-url http://localhost:4566/000000000000/turnos.fifo --message-body "Turno 2" --message-group-id caja1
```

```bash
aws sqs send-message --queue-url http://localhost:4566/000000000000/turnos.fifo --message-body "Turno 3" --message-group-id caja1
```

Ahora envía **otra vez** `Turno 1` (un duplicado):

```bash
aws sqs send-message --queue-url http://localhost:4566/000000000000/turnos.fifo --message-body "Turno 1" --message-group-id caja1
```

```bash
aws sqs receive-message --queue-url http://localhost:4566/000000000000/turnos.fifo --max-number-of-messages 10 --query "Messages[].Body"
```

Resultado: `Turno 1, Turno 2, Turno 3`. En orden y **sin el duplicado**, gracias a `ContentBasedDeduplication`.

## Ejercicio 5: un consumidor de verdad en Python

`trabajador.py` procesa la cola sin parar, como haría un servidor real:

```python
import time
import boto3

CONFIG = dict(endpoint_url="http://localhost:4566", region_name="us-east-1",
              aws_access_key_id="test", aws_secret_access_key="test")
sqs = boto3.client("sqs", **CONFIG)
url = sqs.create_queue(QueueName="pedidos")["QueueUrl"]

print("Esperando pedidos... (Ctrl + C para salir)")
while True:
    respuesta = sqs.receive_message(QueueUrl=url, MaxNumberOfMessages=10, WaitTimeSeconds=10)
    for mensaje in respuesta.get("Messages", []):
        print("Procesando:", mensaje["Body"])
        time.sleep(1)  # simula trabajo
        sqs.delete_message(QueueUrl=url, ReceiptHandle=mensaje["ReceiptHandle"])
        print("  listo")
```

```bash
python trabajador.py
```

En otra terminal, envía mensajes con `aws sqs send-message` y míralos procesarse. Prueba a abrir **dos** trabajadores a la vez: se reparten los mensajes. Así se escala en la nube.

## Ver el estado de una cola

```bash
aws sqs get-queue-attributes --queue-url http://localhost:4566/000000000000/pedidos --attribute-names All
```

Fíjate en `ApproximateNumberOfMessages` (esperando) y `ApproximateNumberOfMessagesNotVisible` (en proceso).

---

## Reto

1. Crea una cola FIFO `cocina.fifo` con dos grupos (`mesa1`, `mesa2`) y comprueba que dentro de cada mesa los platos salen en orden.
2. Añádele una DLQ (`cocina-errores.fifo`, que también debe ser FIFO) con `maxReceiveCount` 3.
3. Modifica `trabajador.py` para que, si el mensaje contiene la palabra `error`, **no** lo borre. Comprueba que acaba en la DLQ.

## Chuleta

| Quiero... | Comando |
|---|---|
| Crear | `aws sqs create-queue --queue-name Q [--attributes file://config.json]` |
| Listar | `aws sqs list-queues` |
| Enviar | `aws sqs send-message --queue-url URL --message-body "..."` |
| Enviar varios | `aws sqs send-message-batch --queue-url URL --entries file://lote.json` |
| Recibir | `aws sqs receive-message --queue-url URL --max-number-of-messages 10 --wait-time-seconds 10` |
| Borrar mensaje | `aws sqs delete-message --queue-url URL --receipt-handle RH` |
| Vaciar | `aws sqs purge-queue --queue-url URL` |
| Estado | `aws sqs get-queue-attributes --queue-url URL --attribute-names All` |
| Borrar cola | `aws sqs delete-queue --queue-url URL` |
