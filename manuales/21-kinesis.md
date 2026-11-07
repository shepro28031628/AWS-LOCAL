# Kinesis y Firehose: datos en tiempo real

[Volver al índice](README.md)

## Qué es y cuándo usarlo

Cuando los datos llegan **sin parar** (clics en una web, sensores IoT, logs, transacciones), no se guardan en archivos de golpe: se mandan como un **flujo** (*stream*).

| Servicio | Papel |
|---|---|
| **Kinesis Data Streams** | Recibe registros en tiempo real y los guarda unas horas para que uno o varios programas los lean |
| **Data Firehose** | Recibe registros y los **entrega** agrupados a un destino, por ejemplo S3, sin que programes nada |

Ejemplo típico: la web manda cada clic a Kinesis; un programa calcula estadísticas en vivo y Firehose guarda todo en S3 para analizarlo luego con [Athena](20-glue-athena.md).

## Conceptos clave

| Concepto | Qué es |
|---|---|
| **Stream** | El flujo de datos |
| **Shard** | Una "vía" del stream. Más shards = más capacidad |
| **Registro** | Un dato enviado (hasta 1 MB), normalmente JSON |
| **Partition key** | Decide a qué shard va cada registro. Los registros con la misma clave van en orden al mismo shard |
| **Sequence number** | Número de orden de cada registro dentro de su shard |
| **Shard iterator** | Un "marcapáginas" para leer desde un punto del shard |
| **Buffer de Firehose** | Firehose junta registros hasta un tamaño o un tiempo y los escribe de una vez |

## Antes de empezar

Terminal configurada ([manual 3](03-configurar-aws-cli.md)).

> **Importante con la AWS CLI v2:** el campo `--data` se interpreta como base64. Añade siempre `--cli-binary-format raw-in-base64-out` para mandar texto normal, o usa `fileb://archivo`.

---

## Ejercicio 1: crear un stream

```bash
aws kinesis create-stream --stream-name clics --shard-count 1
```

```bash
aws kinesis describe-stream-summary --stream-name clics --query "StreamDescriptionSummary.[StreamStatus,OpenShardCount]" --output text
```

Debe decir `ACTIVE 1`.

## Ejercicio 2: enviar registros

Crea `clic1.json`:

```json
{"usuario": "ana", "pagina": "/inicio", "segundos": 12}
```

Y `clic2.json`:

```json
{"usuario": "luis", "pagina": "/productos", "segundos": 40}
```

```bash
aws kinesis put-record --stream-name clics --partition-key ana --data fileb://clic1.json
```

```bash
aws kinesis put-record --stream-name clics --partition-key luis --data fileb://clic2.json
```

Cada respuesta trae el `ShardId` donde cayó y su `SequenceNumber`.

## Ejercicio 3: leer los registros

Pide un marcapáginas desde el principio del shard (`TRIM_HORIZON`):

**Windows (PowerShell):**

```bash
$IT = aws kinesis get-shard-iterator --stream-name clics --shard-id shardId-000000000000 --shard-iterator-type TRIM_HORIZON --query ShardIterator --output text
```

**macOS / Linux:**

```bash
IT=$(aws kinesis get-shard-iterator --stream-name clics --shard-id shardId-000000000000 --shard-iterator-type TRIM_HORIZON --query ShardIterator --output text)
```

Lee:

```bash
aws kinesis get-records --shard-iterator $IT
```

El campo `Data` viene en **base64**. Para verlo como texto:

**Windows (PowerShell):**

```bash
aws kinesis get-records --shard-iterator $IT --query "Records[].Data" --output text | ForEach-Object { $_ -split "`t" } | ForEach-Object { [Text.Encoding]::UTF8.GetString([Convert]::FromBase64String($_)) }
```

**macOS / Linux:**

```bash
aws kinesis get-records --shard-iterator $IT --query "Records[].Data" --output text | tr '\t' '\n' | while read d; do echo "$d" | base64 -d; echo; done
```

Otros tipos de iterador: `LATEST` (solo lo nuevo a partir de ahora) y `AT_SEQUENCE_NUMBER` (desde un registro concreto).

## Ejercicio 4: productor y consumidor en Python

Abre **dos terminales**.

`productor.py` (simula clics cada segundo):

```python
import json
import random
import time
import boto3

CONFIG = dict(endpoint_url="http://localhost:4566", region_name="us-east-1",
              aws_access_key_id="test", aws_secret_access_key="test")
kinesis = boto3.client("kinesis", **CONFIG)

usuarios = ["ana", "luis", "marta", "pedro"]
paginas = ["/inicio", "/productos", "/carrito", "/pagar"]

while True:
    clic = {"usuario": random.choice(usuarios), "pagina": random.choice(paginas)}
    kinesis.put_record(StreamName="clics", PartitionKey=clic["usuario"],
                       Data=json.dumps(clic).encode())
    print("Enviado:", clic)
    time.sleep(1)
```

`consumidor.py` (cuenta visitas por página en vivo):

```python
import json
import time
from collections import Counter
import boto3

CONFIG = dict(endpoint_url="http://localhost:4566", region_name="us-east-1",
              aws_access_key_id="test", aws_secret_access_key="test")
kinesis = boto3.client("kinesis", **CONFIG)

iterador = kinesis.get_shard_iterator(
    StreamName="clics", ShardId="shardId-000000000000", ShardIteratorType="LATEST"
)["ShardIterator"]

visitas = Counter()
while True:
    respuesta = kinesis.get_records(ShardIterator=iterador, Limit=100)
    for registro in respuesta["Records"]:
        visitas[json.loads(registro["Data"])["pagina"]] += 1
    if respuesta["Records"]:
        print(dict(visitas))
    iterador = respuesta["NextShardIterator"]
    time.sleep(1)
```

Ejecuta primero `python consumidor.py` y luego, en la otra terminal, `python productor.py`. Verás el recuento actualizarse cada segundo. Para con `Ctrl + C`.

## Ejercicio 5: guardar el flujo en S3 con Firehose

Crea el bucket si no lo tienes:

```bash
aws s3 mb s3://lago-datos
```

Crea `destino.json`:

```json
{
  "RoleARN": "arn:aws:iam::000000000000:role/firehose-role",
  "BucketARN": "arn:aws:s3:::lago-datos",
  "Prefix": "streaming/clics/",
  "BufferingHints": { "IntervalInSeconds": 60, "SizeInMBs": 1 }
}
```

```bash
aws firehose create-delivery-stream --delivery-stream-name clics-a-s3 --delivery-stream-type DirectPut --extended-s3-destination-configuration file://destino.json
```

Envía registros. Crea `registro.json` (el campo `Data` va en base64; este es `{"usuario":"ana"}`):

```json
{ "Data": "eyJ1c3VhcmlvIjoiYW5hIn0K" }
```

```bash
aws firehose put-record --delivery-stream-name clics-a-s3 --record file://registro.json
```

Espera un minuto (el buffer es de 60 segundos) y mira S3:

```bash
aws s3 ls s3://lago-datos/streaming/clics/ --recursive
```

Firehose agrupó los registros en archivos. A partir de ahí puedes registrarlos en Glue y consultarlos con Athena.

> Desde Python es más fácil: `boto3.client("firehose", **CONFIG).put_record(DeliveryStreamName="clics-a-s3", Record={"Data": json.dumps(clic).encode() + b"\n"})`.

## Borrar

```bash
aws kinesis delete-stream --stream-name clics
```

```bash
aws firehose delete-delivery-stream --delivery-stream-name clics-a-s3
```

---

## Reto

1. Crea un stream `sensores` y un productor que mande temperaturas aleatorias de 3 sensores (`sensor-1`, `sensor-2`, `sensor-3`) usando el nombre del sensor como partition key.
2. Haz un consumidor que muestre una alerta cuando algún sensor pase de 30 grados.
3. Envía también las lecturas a Firehose y comprueba que llegan a S3.

## Chuleta

| Quiero... | Comando |
|---|---|
| Crear stream | `aws kinesis create-stream --stream-name S --shard-count 1` |
| Enviar | `aws kinesis put-record --stream-name S --partition-key K --data fileb://dato.json` |
| Marcapáginas | `aws kinesis get-shard-iterator --stream-name S --shard-id shardId-000000000000 --shard-iterator-type TRIM_HORIZON` |
| Leer | `aws kinesis get-records --shard-iterator IT` |
| Firehose a S3 | `aws firehose create-delivery-stream ... --extended-s3-destination-configuration file://destino.json` |
| Enviar a Firehose | `aws firehose put-record --delivery-stream-name F --record file://registro.json` |
