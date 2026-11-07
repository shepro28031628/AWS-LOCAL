# Programar con Python (boto3)

[Volver al índice](README.md)

Todo lo que hiciste con la terminal se puede hacer desde un programa. En Python se usa la librería oficial de AWS, **boto3**.

## Paso 1: instalar boto3

```bash
pip install boto3
```

## Paso 2: conectarse a tu nube local

La única diferencia con la nube real es `endpoint_url` y las credenciales de prueba. Crea **`conexion.py`**:

```python
import boto3

CONFIG = {
    "endpoint_url": "http://localhost:4566",
    "region_name": "us-east-1",
    "aws_access_key_id": "test",
    "aws_secret_access_key": "test",
}


def cliente(servicio):
    return boto3.client(servicio, **CONFIG)
```

Los demás ejemplos importan este archivo.

## Ejemplo 1: S3

Crea **`ejemplo_s3.py`**:

```python
from conexion import cliente

s3 = cliente("s3")

s3.create_bucket(Bucket="fotos-clase")
s3.put_object(Bucket="fotos-clase", Key="saludo.txt", Body="Hola desde Python")

for archivo in s3.list_objects_v2(Bucket="fotos-clase").get("Contents", []):
    print("Archivo:", archivo["Key"], "-", archivo["Size"], "bytes")

contenido = s3.get_object(Bucket="fotos-clase", Key="saludo.txt")["Body"].read()
print("Contenido:", contenido.decode())
```

```bash
python ejemplo_s3.py
```

## Ejemplo 2: DynamoDB

Crea **`ejemplo_dynamodb.py`**:

```python
from conexion import CONFIG
import boto3

dynamodb = boto3.resource("dynamodb", **CONFIG)

tabla = dynamodb.create_table(
    TableName="notas",
    KeySchema=[{"AttributeName": "id", "KeyType": "HASH"}],
    AttributeDefinitions=[{"AttributeName": "id", "AttributeType": "S"}],
    BillingMode="PAY_PER_REQUEST",
)
tabla.wait_until_exists()

tabla.put_item(Item={"id": "1", "nombre": "Ana", "nota": 9})
tabla.put_item(Item={"id": "2", "nombre": "Luis", "nota": 7})

print(tabla.get_item(Key={"id": "1"})["Item"])

for alumno in tabla.scan()["Items"]:
    print(alumno["nombre"], "tiene", alumno["nota"])
```

```bash
python ejemplo_dynamodb.py
```

Verás algo como `{'id': '1', 'nombre': 'Ana', 'nota': Decimal('9')}`. DynamoDB devuelve los números como `Decimal` para no perder precisión; si necesitas un número normal, usa `int(alumno["nota"])` o `float(alumno["nota"])`.

> Si lo ejecutas dos veces, falla porque la tabla ya existe. Bórrala con `aws dynamodb delete-table --table-name notas` o reinicia AWS Nube Local.

## Ejemplo 3: SQS

Crea **`ejemplo_sqs.py`**:

```python
from conexion import cliente

sqs = cliente("sqs")

url = sqs.create_queue(QueueName="tareas")["QueueUrl"]

for i in range(1, 4):
    sqs.send_message(QueueUrl=url, MessageBody=f"Tarea número {i}")

respuesta = sqs.receive_message(QueueUrl=url, MaxNumberOfMessages=10)
for mensaje in respuesta.get("Messages", []):
    print("Recibido:", mensaje["Body"])
    sqs.delete_message(QueueUrl=url, ReceiptHandle=mensaje["ReceiptHandle"])
```

```bash
python ejemplo_sqs.py
```

## Pasar a la nube real

Cuando quieras usar el mismo código con AWS de verdad, solo quita `endpoint_url` y pon credenciales reales. El resto del código **no cambia**: esa es la gracia de practicar en local.

## Otros lenguajes

La idea es igual en todos: configurar el endpoint `http://localhost:4566` y credenciales `test`. Hay ejemplos para Java, Node.js y Go en el panel principal: <http://localhost:4566/>.

**Siguiente:** [Proyecto integrador](80-proyecto-integrador.md)
