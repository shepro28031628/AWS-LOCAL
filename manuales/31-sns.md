# SNS: notificaciones a varios destinos

[Volver al índice](README.md)

## Qué es y cuándo usarlo

**Amazon SNS** (*Simple Notification Service*) reparte un mensaje a **muchos destinos a la vez**. Un programa publica en un **tema** (*topic*) y todos los suscritos lo reciben: colas SQS, funciones Lambda, direcciones HTTP, correos...

Es el patrón **fan-out** (abanico): una venta se publica una vez y le llega a facturación, al almacén y al sistema de correos, cada uno por su lado.

| SQS | SNS |
|---|---|
| Un mensaje lo procesa **un** consumidor | Un mensaje lo reciben **todos** los suscritos |
| El mensaje espera en la cola | Se entrega al momento (si nadie escucha, se pierde) |

Por eso lo habitual es combinarlos: **SNS reparte a varias colas SQS** y cada cola tiene su consumidor.

## Conceptos clave

| Concepto | Qué es |
|---|---|
| **Tema** (*topic*) | El canal donde se publica |
| **Suscripción** | Un destino apuntado a un tema |
| **Protocolo** | Tipo de destino: `sqs`, `lambda`, `http`, `https`, `email`... |
| **Atributos del mensaje** | Etiquetas que acompañan al mensaje (por ejemplo, `categoria=ropa`) |
| **Política de filtro** | Una suscripción solo recibe los mensajes cuyos atributos cumplan una condición |
| **Raw delivery** | Entrega el texto tal cual, sin el "sobre" JSON de SNS |

## Antes de empezar

Terminal configurada ([manual 3](03-configurar-aws-cli.md)). Conviene haber hecho el [manual de SQS](30-sqs.md).

---

## Ejercicio 1: crear un tema y dos colas suscritas

**Windows (PowerShell):**

```bash
$TEMA = aws sns create-topic --name ventas --query TopicArn --output text
```

**macOS / Linux:**

```bash
TEMA=$(aws sns create-topic --name ventas --query TopicArn --output text)
```

Crea dos colas: una para facturación y otra para el almacén de electrónica.

```bash
aws sqs create-queue --queue-name facturacion
```

```bash
aws sqs create-queue --queue-name almacen-electronica
```

Suscribe facturación con **entrega en bruto** (recibe solo el texto):

```bash
aws sns subscribe --topic-arn $TEMA --protocol sqs --notification-endpoint arn:aws:sqs:us-east-1:000000000000:facturacion --attributes RawMessageDelivery=true
```

Suscribe el almacén (formato normal de SNS) y guarda el ARN de la suscripción:

**Windows (PowerShell):**

```bash
$SUB = aws sns subscribe --topic-arn $TEMA --protocol sqs --notification-endpoint arn:aws:sqs:us-east-1:000000000000:almacen-electronica --query SubscriptionArn --output text
```

**macOS / Linux:**

```bash
SUB=$(aws sns subscribe --topic-arn $TEMA --protocol sqs --notification-endpoint arn:aws:sqs:us-east-1:000000000000:almacen-electronica --query SubscriptionArn --output text)
```

```bash
aws sns list-subscriptions-by-topic --topic-arn $TEMA
```

## Ejercicio 2: filtrar qué recibe cada suscriptor

El almacén de electrónica solo quiere ventas de `electronica`. Crea `filtro.json`:

```json
{ "categoria": ["electronica"] }
```

```bash
aws sns set-subscription-attributes --subscription-arn $SUB --attribute-name FilterPolicy --attribute-value file://filtro.json
```

## Ejercicio 3: publicar con atributos

`venta-electronica.json`:

```json
{ "categoria": { "DataType": "String", "StringValue": "electronica" } }
```

`venta-ropa.json`:

```json
{ "categoria": { "DataType": "String", "StringValue": "ropa" } }
```

```bash
aws sns publish --topic-arn $TEMA --message "Venta: portátil" --message-attributes file://venta-electronica.json
```

```bash
aws sns publish --topic-arn $TEMA --message "Venta: camiseta" --message-attributes file://venta-ropa.json
```

Comprueba cada cola:

```bash
aws sqs receive-message --queue-url http://localhost:4566/000000000000/facturacion --max-number-of-messages 10 --query "Messages[].Body"
```

```bash
aws sqs receive-message --queue-url http://localhost:4566/000000000000/almacen-electronica --max-number-of-messages 10 --query "Messages[].Body"
```

| Cola | Recibe |
|---|---|
| `facturacion` | Las **dos** ventas, como texto simple |
| `almacen-electronica` | Solo el **portátil**, dentro de un JSON de SNS (campo `Message`) |

## Ejercicio 4: más tipos de filtro

Las políticas de filtro admiten varias condiciones. Ejemplos de `filtro.json`:

Varias categorías:

```json
{ "categoria": ["electronica", "hogar"] }
```

Todo menos una:

```json
{ "categoria": [{ "anything-but": "ropa" }] }
```

Que empiece por un texto:

```json
{ "tienda": [{ "prefix": "BOG-" }] }
```

Varias condiciones a la vez (deben cumplirse todas):

```json
{ "categoria": ["electronica"], "pais": ["CO", "MX"] }
```

## Ejercicio 5: publicar desde Python

```python
import boto3

CONFIG = dict(endpoint_url="http://localhost:4566", region_name="us-east-1",
              aws_access_key_id="test", aws_secret_access_key="test")
sns = boto3.client("sns", **CONFIG)

tema = sns.create_topic(Name="ventas")["TopicArn"]

for producto, categoria in [("tablet", "electronica"), ("sofá", "hogar"), ("auriculares", "electronica")]:
    sns.publish(
        TopicArn=tema,
        Message=f"Venta: {producto}",
        MessageAttributes={"categoria": {"DataType": "String", "StringValue": categoria}},
    )
    print("Publicado:", producto)
```

> `create_topic` es seguro de repetir: si el tema ya existe, devuelve el mismo.

## Borrar

```bash
aws sns unsubscribe --subscription-arn $SUB
```

```bash
aws sns delete-topic --topic-arn $TEMA
```

---

## Reto

Monta el sistema de avisos de una tienda:

1. Tema `pedidos` con tres colas suscritas: `correos`, `envios` y `fraude`.
2. `envios` solo recibe pedidos con `pais` igual a `CO`.
3. `fraude` solo recibe pedidos con `metodo_pago` distinto de `efectivo` (`anything-but`).
4. Publica 5 pedidos variados y comprueba qué le llega a cada cola.

## Chuleta

| Quiero... | Comando |
|---|---|
| Crear tema | `aws sns create-topic --name T` |
| Suscribir cola | `aws sns subscribe --topic-arn ARN --protocol sqs --notification-endpoint ARN_COLA` |
| Ver suscripciones | `aws sns list-subscriptions-by-topic --topic-arn ARN` |
| Filtro | `aws sns set-subscription-attributes --subscription-arn SUB --attribute-name FilterPolicy --attribute-value file://filtro.json` |
| Publicar | `aws sns publish --topic-arn ARN --message "..." [--message-attributes file://attr.json]` |
| Borrar | `aws sns unsubscribe` / `aws sns delete-topic` |
