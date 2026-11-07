# 6. SQS y SNS: mensajes

[Volver al índice](README.md)

Sirven para que unos programas se manden mensajes a otros sin estar conectados directamente.

| Servicio | Idea | Ejemplo de la vida real |
|---|---|---|
| **SQS** (colas) | Un programa deja mensajes en una cola y otro los recoge cuando puede | La fila de pedidos de una cocina |
| **SNS** (temas) | Un mensaje se reparte a la vez a todos los suscritos | Un grupo de WhatsApp: uno escribe, todos lo reciben |

> Antes de empezar, configura la terminal con el [manual 3](03-configurar-aws-cli.md).

## Parte 1: SQS (colas)

### Paso 1: crear una cola

```bash
aws sqs create-queue --queue-name pedidos
```

La respuesta trae un `QueueUrl` (la dirección de la cola). Guárdala para no tener que copiarla cada vez.

**Windows (PowerShell):**

```bash
$COLA = aws sqs get-queue-url --queue-name pedidos --query QueueUrl --output text
```

**macOS / Linux:**

```bash
COLA=$(aws sqs get-queue-url --queue-name pedidos --query QueueUrl --output text)
```

### Paso 2: enviar mensajes

```bash
aws sqs send-message --queue-url $COLA --message-body "Pedido 1: dos pizzas"
```

```bash
aws sqs send-message --queue-url $COLA --message-body "Pedido 2: una ensalada"
```

### Paso 3: recibir un mensaje

```bash
aws sqs receive-message --queue-url $COLA
```

La respuesta trae el texto (`Body`) y un `ReceiptHandle`, que es como el "ticket" del mensaje.

### Paso 4: marcar el mensaje como atendido

Si no lo borras, el mensaje **vuelve a aparecer** en la cola pasados unos segundos (por si el programa que lo recogió falló). Para borrarlo, copia el `ReceiptHandle`:

```bash
aws sqs delete-message --queue-url $COLA --receipt-handle "PEGA_AQUÍ_EL_RECEIPT_HANDLE"
```

### Paso 5: ver cuántos mensajes quedan

```bash
aws sqs get-queue-attributes --queue-url $COLA --attribute-names ApproximateNumberOfMessages
```

## Parte 2: SNS (avisos a varios destinos)

Vamos a crear un tema de **avisos** y suscribir la cola `pedidos` para que reciba todo lo que se publique.

### Paso 1: crear el tema

**Windows (PowerShell):**

```bash
$TEMA = aws sns create-topic --name avisos --query TopicArn --output text
```

**macOS / Linux:**

```bash
TEMA=$(aws sns create-topic --name avisos --query TopicArn --output text)
```

### Paso 2: suscribir la cola al tema

Primero, el identificador (ARN) de la cola:

**Windows (PowerShell):**

```bash
$COLA_ARN = aws sqs get-queue-attributes --queue-url $COLA --attribute-names QueueArn --query Attributes.QueueArn --output text
```

**macOS / Linux:**

```bash
COLA_ARN=$(aws sqs get-queue-attributes --queue-url $COLA --attribute-names QueueArn --query Attributes.QueueArn --output text)
```

Y la suscripción:

```bash
aws sns subscribe --topic-arn $TEMA --protocol sqs --notification-endpoint $COLA_ARN
```

### Paso 3: publicar un aviso

```bash
aws sns publish --topic-arn $TEMA --message "La cocina cierra a las 22:00"
```

### Paso 4: comprobar que llegó a la cola

```bash
aws sqs receive-message --queue-url $COLA
```

El aviso llega envuelto en un JSON de SNS. Tu texto está en el campo `Message`.

Puedes suscribir **varias** colas al mismo tema: todas recibirán cada aviso.

## Limpiar

```bash
aws sns delete-topic --topic-arn $TEMA
```

```bash
aws sqs delete-queue --queue-url $COLA
```

**Siguiente:** [7. Lambda: funciones](07-lambda-funciones.md)
