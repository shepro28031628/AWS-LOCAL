# 5. DynamoDB: base de datos

[Volver al índice](README.md)

**DynamoDB** es una base de datos NoSQL de AWS. Guarda **registros** (llamados *items*) dentro de **tablas**. Cada registro se identifica con una **clave**.

> Antes de empezar, configura la terminal con el [manual 3](03-configurar-aws-cli.md).

En este manual haremos una tabla de **alumnos** con su nombre y su nota.

## Paso 1: crear la tabla

```bash
aws dynamodb create-table --table-name alumnos --attribute-definitions AttributeName=id,AttributeType=S --key-schema AttributeName=id,KeyType=HASH --billing-mode PAY_PER_REQUEST
```

Qué significa:

| Parte | Significado |
|---|---|
| `--table-name alumnos` | Nombre de la tabla |
| `AttributeName=id,AttributeType=S` | La clave se llama `id` y es texto (`S` = *string*) |
| `KeyType=HASH` | `id` es la clave principal: no puede repetirse |
| `PAY_PER_REQUEST` | Modo de cobro (aquí no se cobra nada, pero AWS lo exige) |

Comprueba que existe:

```bash
aws dynamodb list-tables
```

## Paso 2: guardar un alumno

Los datos se escriben en JSON. Para evitar problemas con las comillas en la terminal, los guardamos en un archivo.

Crea un archivo llamado **`alumno1.json`** (con el Bloc de notas o VS Code) con este contenido:

```json
{
  "id":     { "S": "1" },
  "nombre": { "S": "Ana" },
  "nota":   { "N": "9" }
}
```

`S` significa texto y `N` significa número.

Guárdalo en la base de datos:

```bash
aws dynamodb put-item --table-name alumnos --item file://alumno1.json
```

Repite con otro archivo `alumno2.json` (por ejemplo `"id": "2"`, `"nombre": "Luis"`, `"nota": "7"`).

## Paso 3: leer un alumno por su clave

Crea **`clave1.json`**:

```json
{ "id": { "S": "1" } }
```

```bash
aws dynamodb get-item --table-name alumnos --key file://clave1.json
```

## Paso 4: ver todos los alumnos

```bash
aws dynamodb scan --table-name alumnos
```

Solo contar cuántos hay:

```bash
aws dynamodb scan --table-name alumnos --select COUNT
```

## Paso 5: modificar un alumno

Guardar otra vez con el mismo `id` **reemplaza** el registro. Edita `alumno1.json`, cambia la nota a `10` y vuelve a ejecutar:

```bash
aws dynamodb put-item --table-name alumnos --item file://alumno1.json
```

## Paso 6: borrar

Borrar un alumno:

```bash
aws dynamodb delete-item --table-name alumnos --key file://clave1.json
```

Borrar la tabla completa:

```bash
aws dynamodb delete-table --table-name alumnos
```

## Resumen

| Quiero... | Comando |
|---|---|
| Crear tabla | `aws dynamodb create-table ...` (ver paso 1) |
| Listar tablas | `aws dynamodb list-tables` |
| Guardar o reemplazar | `aws dynamodb put-item --table-name T --item file://archivo.json` |
| Leer por clave | `aws dynamodb get-item --table-name T --key file://clave.json` |
| Ver todo | `aws dynamodb scan --table-name T` |
| Borrar registro | `aws dynamodb delete-item --table-name T --key file://clave.json` |
| Borrar tabla | `aws dynamodb delete-table --table-name T` |

**Siguiente:** [6. SQS y SNS: mensajes](06-sqs-sns-mensajes.md)
