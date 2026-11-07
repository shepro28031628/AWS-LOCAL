# Step Functions: flujos de trabajo

[Volver al índice](README.md)

## Qué es y cuándo usarlo

**AWS Step Functions** coordina varios pasos en un **flujo de trabajo** (*workflow*) visual: primero valida, luego decide, espera, ejecuta en paralelo, reintenta si falla... Cada paso puede ser una Lambda u otro servicio de AWS.

En vez de una Lambda gigante que lo hace todo, tienes pasos pequeños y un "director de orquesta" que guarda el estado y el historial de cada ejecución.

Úsalo para: procesar pedidos, aprobar solicitudes, pipelines de datos (ETL), procesos con esperas o reintentos.

## Conceptos clave

| Concepto | Qué es |
|---|---|
| **Máquina de estados** | El flujo, definido en JSON con el lenguaje **ASL** (*Amazon States Language*) |
| **Estado** | Un paso del flujo |
| **Ejecución** | Una vez que corre el flujo con una entrada concreta |
| **Entrada / salida** | JSON que pasa de un estado al siguiente |
| **`$.campo`** | Ruta JSON para leer un dato de la entrada |

Tipos de estado:

| Tipo | Hace |
|---|---|
| `Task` | Ejecuta trabajo (por ejemplo, una Lambda) |
| `Choice` | Decide el siguiente paso según los datos (un `if`) |
| `Wait` | Espera unos segundos o hasta una fecha |
| `Pass` | Pasa o transforma datos sin hacer trabajo |
| `Parallel` | Ejecuta ramas a la vez |
| `Map` | Repite unos pasos para cada elemento de una lista |
| `Succeed` / `Fail` | Termina bien / con error |

## Antes de empezar

Terminal configurada ([manual 3](03-configurar-aws-cli.md)) y haber hecho el [manual de Lambda](40-lambda.md).

---

## Ejercicio 1: la Lambda del primer paso

Crea `validar_pedido.py`:

```python
def handler(event, context):
    total = event["total"]
    return {**event, "valido": total > 0, "descuento": 10 if total >= 100 else 0}
```

Comprímela en `validar_pedido.zip` y créala:

```bash
aws lambda create-function --function-name validar-pedido --runtime python3.12 --handler validar_pedido.handler --zip-file fileb://validar_pedido.zip --role arn:aws:iam::000000000000:role/lambda-role
```

## Ejercicio 2: definir el flujo

Crea `flujo.json`:

```json
{
  "Comment": "Procesa un pedido",
  "StartAt": "ValidarPedido",
  "States": {
    "ValidarPedido": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:000000000000:function:validar-pedido",
      "Next": "EsValido"
    },
    "EsValido": {
      "Type": "Choice",
      "Choices": [
        { "Variable": "$.valido", "BooleanEquals": true, "Next": "EsperarPago" }
      ],
      "Default": "PedidoRechazado"
    },
    "EsperarPago": {
      "Type": "Wait",
      "Seconds": 2,
      "Next": "Confirmar"
    },
    "Confirmar": {
      "Type": "Pass",
      "Parameters": {
        "pedido.$": "$.pedido",
        "estado": "CONFIRMADO",
        "descuento.$": "$.descuento"
      },
      "End": true
    },
    "PedidoRechazado": {
      "Type": "Fail",
      "Error": "PedidoInvalido",
      "Cause": "El total debe ser mayor que cero"
    }
  }
}
```

Léelo como un diagrama:

```text
ValidarPedido ──> EsValido ──(valido)──> EsperarPago ──> Confirmar ──> FIN
                     │
                     └──(no válido)──> PedidoRechazado (error)
```

> En `Parameters`, una clave terminada en `.$` (como `"pedido.$"`) toma su valor de la entrada. Sin `.$` es un valor fijo.

Crea la máquina de estados:

```bash
aws stepfunctions create-state-machine --name procesar-pedido --definition file://flujo.json --role-arn arn:aws:iam::000000000000:role/sfn-role
```

Guarda el `stateMachineArn` que devuelve. Tiene la forma `arn:aws:states:us-east-1:000000000000:stateMachine:procesar-pedido`.

## Ejercicio 3: ejecutar el flujo

Crea `pedido-ok.json`:

```json
{ "pedido": "P-100", "total": 150 }
```

Y `pedido-mal.json`:

```json
{ "pedido": "P-101", "total": 0 }
```

```bash
aws stepfunctions start-execution --state-machine-arn arn:aws:states:us-east-1:000000000000:stateMachine:procesar-pedido --name pedido-100 --input file://pedido-ok.json
```

```bash
aws stepfunctions start-execution --state-machine-arn arn:aws:states:us-east-1:000000000000:stateMachine:procesar-pedido --name pedido-101 --input file://pedido-mal.json
```

> El `--name` de cada ejecución debe ser único. Si repites, cambia el nombre (o quítalo y se genera uno).

Mira cómo terminaron:

```bash
aws stepfunctions list-executions --state-machine-arn arn:aws:states:us-east-1:000000000000:stateMachine:procesar-pedido --query "executions[].[name,status]" --output table
```

| Ejecución | Estado |
|---|---|
| `pedido-100` | `SUCCEEDED` |
| `pedido-101` | `FAILED` |

## Ejercicio 4: resultado e historial

Resultado de la buena (copia su `executionArn` de `list-executions`):

```bash
aws stepfunctions describe-execution --execution-arn ARN_DE_LA_EJECUCION --query "[status,output]"
```

Salida: `{"pedido":"P-100","estado":"CONFIRMADO","descuento":10}`.

Motivo del fallo de la mala:

```bash
aws stepfunctions describe-execution --execution-arn ARN_DE_LA_EJECUCION --query "[status,error,cause]"
```

Historial paso a paso, muy útil para depurar:

```bash
aws stepfunctions get-execution-history --execution-arn ARN_DE_LA_EJECUCION --query "events[].type"
```

Verás la secuencia: `TaskStateEntered`, `LambdaFunctionSucceeded`, `ChoiceStateEntered`, `WaitStateEntered`, `PassStateExited`, `ExecutionSucceeded`...

## Ejercicio 5: procesar una lista con Map, con reintentos y control de errores

Crea `flujo-lote.json`. Recorre una lista de pedidos, valida cada uno (de 2 en 2), **reintenta** si falla y, si aun así falla, lo marca como error **sin parar el lote**:

```json
{
  "StartAt": "ProcesarCadaPedido",
  "States": {
    "ProcesarCadaPedido": {
      "Type": "Map",
      "ItemsPath": "$.pedidos",
      "MaxConcurrency": 2,
      "Iterator": {
        "StartAt": "Validar",
        "States": {
          "Validar": {
            "Type": "Task",
            "Resource": "arn:aws:lambda:us-east-1:000000000000:function:validar-pedido",
            "Retry": [ { "ErrorEquals": ["States.TaskFailed"], "MaxAttempts": 2, "IntervalSeconds": 1 } ],
            "Catch": [ { "ErrorEquals": ["States.ALL"], "Next": "MarcarError" } ],
            "End": true
          },
          "MarcarError": {
            "Type": "Pass",
            "Result": { "valido": false, "nota": "No se pudo validar" },
            "End": true
          }
        }
      },
      "End": true
    }
  }
}
```

```bash
aws stepfunctions create-state-machine --name procesar-lote --definition file://flujo-lote.json --role-arn arn:aws:iam::000000000000:role/sfn-role
```

Crea `lote.json`. El pedido C no tiene `total`, así que la Lambda fallará con él:

```json
{ "pedidos": [ {"pedido": "A", "total": 50}, {"pedido": "B", "total": 300}, {"pedido": "C"} ] }
```

```bash
aws stepfunctions start-execution --state-machine-arn arn:aws:states:us-east-1:000000000000:stateMachine:procesar-lote --input file://lote.json
```

Mira la salida con `describe-execution`. A y B quedan validados; C aparece como `"No se pudo validar"` y la ejecución termina `SUCCEEDED`.

| Campo | Para qué |
|---|---|
| `Retry` | Reintenta el paso ante ciertos errores, con espera entre intentos |
| `Catch` | Si se agotan los reintentos, salta a otro estado en vez de fallar todo |
| `States.ALL` | Cualquier error |

## Actualizar y borrar

```bash
aws stepfunctions update-state-machine --state-machine-arn ARN --definition file://flujo.json
```

```bash
aws stepfunctions delete-state-machine --state-machine-arn ARN
```

---

## Reto

Diseña el flujo de **matrícula** de un alumno:

1. `Task` que comprueba si hay cupo (una Lambda que devuelva `{"cupo": true/false}`).
2. `Choice`: sin cupo, termina en `Fail` con `SinCupo`.
3. Con cupo, un `Parallel` con dos ramas: "enviar correo de bienvenida" y "crear usuario en la plataforma" (pueden ser `Pass` o Lambdas).
4. Añade `Retry` a la comprobación de cupo y prueba con varias entradas.

## Chuleta

| Quiero... | Comando |
|---|---|
| Crear flujo | `aws stepfunctions create-state-machine --name N --definition file://flujo.json --role-arn arn:aws:iam::000000000000:role/sfn-role` |
| Ejecutar | `aws stepfunctions start-execution --state-machine-arn ARN --input file://entrada.json` |
| Ver ejecuciones | `aws stepfunctions list-executions --state-machine-arn ARN` |
| Resultado | `aws stepfunctions describe-execution --execution-arn ARN_EJEC` |
| Historial | `aws stepfunctions get-execution-history --execution-arn ARN_EJEC` |
| Actualizar | `aws stepfunctions update-state-machine --state-machine-arn ARN --definition file://flujo.json` |
