# DynamoDB: base de datos NoSQL

[Volver al índice](README.md)

## Qué es y cuándo usarlo

**Amazon DynamoDB** es una base de datos NoSQL de clave-valor: muy rápida, sin servidor que administrar y que escala sola. No usa SQL ni tablas relacionadas como MySQL: cada registro (*item*) es un conjunto de atributos que se busca por su **clave**.

Úsala cuando:

- Sabes de antemano **cómo vas a buscar** los datos (por usuario, por pedido, por fecha...).
- Necesitas respuestas muy rápidas con mucho tráfico: carritos de compra, sesiones, perfiles, marcadores de juegos, IoT.

Si necesitas uniones (JOIN) entre muchas tablas o consultas libres, usa una base relacional ([RDS](12-rds.md)).

## Conceptos clave

| Concepto | Qué es |
|---|---|
| **Tabla** | Colección de items |
| **Item** | Un registro. Cada item puede tener atributos distintos |
| **Clave de partición** (*HASH*) | Atributo obligatorio que identifica el grupo del item |
| **Clave de ordenación** (*RANGE*) | Opcional. Ordena los items dentro de un mismo grupo |
| **Clave primaria** | Partición sola, o partición + ordenación. No puede repetirse |
| **Query** | Busca por clave: rápido y barato |
| **Scan** | Recorre toda la tabla: lento y caro, evítalo en tablas grandes |
| **GSI** (índice secundario global) | Otra forma de buscar, usando otros atributos como clave |
| **TTL** | Borra items automáticamente cuando llega una fecha |
| **Streams** | Registro de cada cambio en la tabla, para reaccionar a ellos |

**Tipos de datos** en JSON: `S` texto, `N` número (siempre entre comillas), `BOOL` verdadero/falso, `L` lista, `M` mapa (objeto).

## Antes de empezar

Terminal configurada ([manual 3](03-configurar-aws-cli.md)) y una carpeta de trabajo para los archivos JSON.

> **¿Por qué tantos archivos JSON?** Escribir JSON dentro de un comando da problemas de comillas, sobre todo en PowerShell. Con `file://archivo.json` funciona igual en todos los sistemas.

---

## Ejercicio 1: tabla con clave compuesta

Vamos a guardar **notas de alumnos**. Un alumno tiene varias asignaturas, así que:

- Clave de partición: `alumno`
- Clave de ordenación: `asignatura`

```bash
aws dynamodb create-table --table-name notas --attribute-definitions AttributeName=alumno,AttributeType=S AttributeName=asignatura,AttributeType=S --key-schema AttributeName=alumno,KeyType=HASH AttributeName=asignatura,KeyType=RANGE --billing-mode PAY_PER_REQUEST
```

```bash
aws dynamodb describe-table --table-name notas --query "Table.[TableStatus,KeySchema]"
```

## Ejercicio 2: cargar varios items de una vez

Crea `lote.json`:

```json
{
  "notas": [
    { "PutRequest": { "Item": { "alumno": {"S": "ana"},  "asignatura": {"S": "bigdata"}, "nota": {"N": "9"},  "curso": {"S": "2026"} } } },
    { "PutRequest": { "Item": { "alumno": {"S": "ana"},  "asignatura": {"S": "redes"},   "nota": {"N": "7"},  "curso": {"S": "2026"} } } },
    { "PutRequest": { "Item": { "alumno": {"S": "ana"},  "asignatura": {"S": "sql"},     "nota": {"N": "10"}, "curso": {"S": "2025"} } } },
    { "PutRequest": { "Item": { "alumno": {"S": "luis"}, "asignatura": {"S": "bigdata"}, "nota": {"N": "6"},  "curso": {"S": "2026"} } } },
    { "PutRequest": { "Item": { "alumno": {"S": "luis"}, "asignatura": {"S": "sql"},     "nota": {"N": "8"},  "curso": {"S": "2026"} } } }
  ]
}
```

```bash
aws dynamodb batch-write-item --request-items file://lote.json
```

> `batch-write-item` admite hasta 25 items por llamada.

## Ejercicio 3: buscar con query

Todas las notas de Ana. Crea `ana.json`:

```json
{ ":a": {"S": "ana"} }
```

```bash
aws dynamodb query --table-name notas --key-condition-expression "alumno = :a" --expression-attribute-values file://ana.json --query "Items[].[asignatura.S,nota.N]" --output table
```

- `:a` es un **marcador**: su valor viene del archivo JSON.
- `--query` (de la AWS CLI) elige qué columnas mostrar.

**Solo las asignaturas de Ana con nota 8 o más.** Crea `ana-min.json`:

```json
{ ":a": {"S": "ana"}, ":min": {"N": "8"} }
```

```bash
aws dynamodb query --table-name notas --key-condition-expression "alumno = :a" --filter-expression "nota >= :min" --expression-attribute-values file://ana-min.json --query "Items[].asignatura.S"
```

> **Importante:** `--key-condition-expression` solo puede usar las claves. Lo demás va en `--filter-expression`, que filtra **después** de leer. Por eso conviene diseñar bien las claves.

**Asignaturas de Ana que empiezan por "s".** Crea `ana-s.json`:

```json
{ ":a": {"S": "ana"}, ":p": {"S": "s"} }
```

```bash
aws dynamodb query --table-name notas --key-condition-expression "alumno = :a AND begins_with(asignatura, :p)" --expression-attribute-values file://ana-s.json
```

## Ejercicio 4: índice secundario para buscar de otra forma

Ahora queremos **las notas de una asignatura**, de mayor a menor. Con la clave actual no se puede (la partición es `alumno`). Solución: un **GSI** con partición `asignatura` y ordenación `nota`.

Crea `indice.json`:

```json
[
  {
    "Create": {
      "IndexName": "por-asignatura",
      "KeySchema": [
        { "AttributeName": "asignatura", "KeyType": "HASH" },
        { "AttributeName": "nota", "KeyType": "RANGE" }
      ],
      "Projection": { "ProjectionType": "ALL" }
    }
  }
]
```

```bash
aws dynamodb update-table --table-name notas --attribute-definitions AttributeName=asignatura,AttributeType=S AttributeName=nota,AttributeType=N --global-secondary-index-updates file://indice.json
```

Crea `bigdata.json`:

```json
{ ":asig": {"S": "bigdata"} }
```

```bash
aws dynamodb query --table-name notas --index-name por-asignatura --key-condition-expression "asignatura = :asig" --expression-attribute-values file://bigdata.json --no-scan-index-forward --query "Items[].[alumno.S,nota.N]" --output table
```

`--no-scan-index-forward` ordena de mayor a menor.

## Ejercicio 5: modificar con update-item

Luis aprueba la recuperación: le subimos un punto y añadimos una observación.

`clave-luis.json`:

```json
{ "alumno": {"S": "luis"}, "asignatura": {"S": "bigdata"} }
```

`cambios.json`:

```json
{ ":inc": {"N": "1"}, ":obs": {"S": "Recuperación aprobada"} }
```

```bash
aws dynamodb update-item --table-name notas --key file://clave-luis.json --update-expression "SET nota = nota + :inc, observacion = :obs" --expression-attribute-values file://cambios.json --return-values ALL_NEW
```

Otras acciones de `--update-expression`: `REMOVE atributo` (quitar), `ADD contador :n` (sumar), `SET lista = list_append(lista, :nuevos)`.

## Ejercicio 6: escrituras condicionales (evitar sobrescribir)

`put-item` reemplaza el item si ya existe. Para **impedirlo**, crea `ana-nueva.json`:

```json
{ "alumno": {"S": "ana"}, "asignatura": {"S": "bigdata"}, "nota": {"N": "0"} }
```

```bash
aws dynamodb put-item --table-name notas --item file://ana-nueva.json --condition-expression "attribute_not_exists(alumno)"
```

Resultado: `ConditionalCheckFailedException`. La nota de Ana sigue siendo 9. Así se evitan errores cuando dos programas escriben a la vez.

## Ejercicio 7: consultas con sintaxis SQL (PartiQL)

DynamoDB también entiende un SQL limitado:

**macOS / Linux:**

```bash
aws dynamodb execute-statement --statement "SELECT asignatura, nota FROM notas WHERE alumno = 'luis'"
```

**Windows (PowerShell):** igual, pero el texto entre comillas simples va dentro de comillas dobles, como está arriba. Si da error, guarda la consulta en una variable: `$sql = "SELECT asignatura, nota FROM notas WHERE alumno = 'luis'"` y usa `--statement $sql`.

## Ejercicio 8: caducidad automática (TTL) y registro de cambios (Streams)

Activa el TTL usando el atributo `expira` (fecha en segundos desde 1970):

```bash
aws dynamodb update-time-to-live --table-name notas --time-to-live-specification Enabled=true,AttributeName=expira
```

Los items con `expira` en el pasado se borran solos. Útil para sesiones o códigos temporales.

Activa los Streams:

```bash
aws dynamodb update-table --table-name notas --stream-specification StreamEnabled=true,StreamViewType=NEW_AND_OLD_IMAGES
```

Cada cambio queda registrado con el valor anterior y el nuevo. En el [manual de Lambda](40-lambda.md) verás cómo ejecutar una función por cada cambio.

---

## Reto

Diseña una tabla `pedidos` para una tienda online:

1. Clave: `cliente` (partición) + `fecha` (ordenación, formato `2026-11-07T10:30`).
2. Carga al menos 6 pedidos de 3 clientes con `total` y `estado` (`pendiente` o `enviado`).
3. Consulta los pedidos de un cliente en noviembre de 2026 (pista: `begins_with(fecha, :mes)`).
4. Crea un GSI `por-estado` para listar todos los pedidos `pendiente`.
5. Cambia un pedido a `enviado` con `update-item`, solo si estaba `pendiente` (pista: `--condition-expression "estado = :p"`).

## Chuleta

| Quiero... | Comando |
|---|---|
| Crear tabla | `aws dynamodb create-table ...` |
| Ver tablas | `aws dynamodb list-tables` |
| Guardar | `aws dynamodb put-item --table-name T --item file://item.json` |
| Guardar varios | `aws dynamodb batch-write-item --request-items file://lote.json` |
| Leer uno | `aws dynamodb get-item --table-name T --key file://clave.json` |
| Buscar por clave | `aws dynamodb query --table-name T --key-condition-expression ...` |
| Buscar por índice | `... query --index-name INDICE ...` |
| Modificar | `aws dynamodb update-item --table-name T --key ... --update-expression ...` |
| Leer todo | `aws dynamodb scan --table-name T` |
| SQL | `aws dynamodb execute-statement --statement "SELECT ..."` |
| Borrar | `aws dynamodb delete-item` / `delete-table` |

## En la consola web

Menú **DynamoDB**: tablas, sus items e índices.
