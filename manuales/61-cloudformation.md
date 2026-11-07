# CloudFormation: infraestructura como código

[Volver al índice](README.md)

## Qué es y cuándo usarlo

Hasta ahora creaste los recursos uno a uno con comandos. **AWS CloudFormation** permite describir **toda la infraestructura en un archivo** (YAML o JSON) y crearla, actualizarla o borrarla de una sola vez.

Esto se llama **infraestructura como código** (IaC):

- El archivo se guarda en Git: hay historial y revisión de cambios.
- Creas entornos idénticos (`dev`, `prod`) con el mismo archivo.
- Borras todo de golpe sin dejar restos.

> Otras herramientas de IaC como **Terraform** o **AWS CDK** también funcionan con AWS Nube Local. Hay una guía de Terraform en [docs/getting-started/terraform.md](../docs/getting-started/terraform.md).

## Conceptos clave

| Concepto | Qué es |
|---|---|
| **Plantilla** | El archivo con la descripción de los recursos |
| **Pila** (*stack*) | El conjunto de recursos creados a partir de una plantilla |
| **Recurso** | Cada pieza: un bucket, una tabla, una cola... (`Type: AWS::S3::Bucket`) |
| **Parámetro** | Valor que se pasa al crear la pila (por ejemplo, el entorno) |
| **Salida** (*output*) | Valor que la pila muestra al terminar (una URL, un nombre) |
| **`!Ref`** | Referencia a un parámetro o al identificador principal de un recurso |
| **`!GetAtt`** | Lee un atributo de un recurso (por ejemplo, su ARN) |
| **`!Sub`** | Inserta variables en un texto: `"pedidos-${Entorno}"` |

## Antes de empezar

Terminal configurada ([manual 3](03-configurar-aws-cli.md)). Conviene conocer S3, DynamoDB, SQS y SNS de los manuales anteriores.

---

## Ejercicio 1: la plantilla

Crea `plantilla.yaml`:

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: Infraestructura de la tienda de la clase

Parameters:
  Entorno:
    Type: String
    Default: dev
    AllowedValues: [dev, prod]

Resources:
  BucketArchivos:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Sub "tienda-archivos-${Entorno}"
      VersioningConfiguration:
        Status: Enabled

  TablaPedidos:
    Type: AWS::DynamoDB::Table
    Properties:
      TableName: !Sub "pedidos-${Entorno}"
      BillingMode: PAY_PER_REQUEST
      AttributeDefinitions:
        - AttributeName: id
          AttributeType: S
      KeySchema:
        - AttributeName: id
          KeyType: HASH

  ColaErrores:
    Type: AWS::SQS::Queue
    Properties:
      QueueName: !Sub "pedidos-errores-${Entorno}"

  ColaPedidos:
    Type: AWS::SQS::Queue
    Properties:
      QueueName: !Sub "pedidos-${Entorno}"
      VisibilityTimeout: 60
      RedrivePolicy:
        deadLetterTargetArn: !GetAtt ColaErrores.Arn
        maxReceiveCount: 3

  TemaAvisos:
    Type: AWS::SNS::Topic
    Properties:
      TopicName: !Sub "avisos-${Entorno}"

  ParametroUrl:
    Type: AWS::SSM::Parameter
    Properties:
      Name: !Sub "/tienda/${Entorno}/cola-pedidos"
      Type: String
      Value: !Ref ColaPedidos

Outputs:
  NombreBucket:
    Value: !Ref BucketArchivos
  UrlCola:
    Value: !Ref ColaPedidos
  ArnTema:
    Value: !Ref TemaAvisos
```

Fíjate en:

- `!GetAtt ColaErrores.Arn` enlaza la cola de pedidos con su cola de errores sin escribir el ARN a mano.
- `!Ref ColaPedidos` guarda la URL de la cola en Parameter Store, para que las apps la lean de ahí.
- CloudFormation calcula el **orden**: crea `ColaErrores` antes que `ColaPedidos` porque una depende de la otra.

Comprueba que la sintaxis es correcta:

```bash
aws cloudformation validate-template --template-body file://plantilla.yaml
```

## Ejercicio 2: crear la pila

```bash
aws cloudformation create-stack --stack-name tienda-dev --template-body file://plantilla.yaml --parameters ParameterKey=Entorno,ParameterValue=dev
```

Comprueba el estado hasta que diga `CREATE_COMPLETE`:

```bash
aws cloudformation describe-stacks --stack-name tienda-dev --query "Stacks[0].StackStatus"
```

Mira las salidas:

```bash
aws cloudformation describe-stacks --stack-name tienda-dev --query "Stacks[0].Outputs" --output table
```

Y los recursos creados:

```bash
aws cloudformation list-stack-resources --stack-name tienda-dev --query "StackResourceSummaries[].[LogicalResourceId,ResourceType,ResourceStatus]" --output table
```

Compruébalo con los comandos de cada servicio:

```bash
aws s3 ls
```

```bash
aws dynamodb list-tables
```

```bash
aws ssm get-parameter --name /tienda/dev/cola-pedidos
```

Seis recursos con un solo comando.

## Ejercicio 3: actualizar la pila

Añade a `plantilla.yaml`, dentro de `Resources:` (antes de `Outputs:`), una cola de facturación suscrita al tema de avisos:

```yaml
  ColaFacturacion:
    Type: AWS::SQS::Queue
    Properties:
      QueueName: !Sub "facturacion-${Entorno}"

  SuscripcionFacturacion:
    Type: AWS::SNS::Subscription
    Properties:
      TopicArn: !Ref TemaAvisos
      Protocol: sqs
      Endpoint: !GetAtt ColaFacturacion.Arn
```

Aplica los cambios manteniendo el mismo parámetro:

```bash
aws cloudformation update-stack --stack-name tienda-dev --template-body file://plantilla.yaml --parameters ParameterKey=Entorno,UsePreviousValue=true
```

Cuando diga `UPDATE_COMPLETE`, mira qué cambió:

```bash
aws cloudformation describe-stack-events --stack-name tienda-dev --query "StackEvents[:6].[LogicalResourceId,ResourceStatus]" --output table
```

Solo se crearon los dos recursos nuevos; lo demás no se tocó.

## Ejercicio 4: un segundo entorno idéntico

Con el **mismo archivo**, crea producción:

```bash
aws cloudformation create-stack --stack-name tienda-prod --template-body file://plantilla.yaml --parameters ParameterKey=Entorno,ParameterValue=prod
```

```bash
aws cloudformation list-stacks --stack-status-filter CREATE_COMPLETE UPDATE_COMPLETE --query "StackSummaries[].[StackName,StackStatus]" --output table
```

Ahora existen `pedidos-dev` y `pedidos-prod`, `tienda-archivos-dev` y `tienda-archivos-prod`, etc.

## Ejercicio 5: borrar todo de golpe

```bash
aws cloudformation delete-stack --stack-name tienda-prod
```

Todos los recursos de producción desaparecen; los de `dev` siguen intactos. Compruébalo con `aws s3 ls`.

> En AWS real, un bucket con archivos no se puede borrar así: hay que vaciarlo antes.

---

## Reto

1. Añade a la plantilla una tabla `clientes-${Entorno}` con clave `email` y un índice secundario `por-ciudad`.
2. Añade una salida con el ARN de la tabla (pista: `!GetAtt TablaClientes.Arn`).
3. Añade un parámetro `DiasRetencion` (número, por defecto 7) y úsalo en un `AWS::Logs::LogGroup` con `RetentionInDays: !Ref DiasRetencion`.
4. Actualiza la pila `tienda-dev` y comprueba los cambios.

## Chuleta

| Quiero... | Comando |
|---|---|
| Validar | `aws cloudformation validate-template --template-body file://p.yaml` |
| Crear | `aws cloudformation create-stack --stack-name S --template-body file://p.yaml --parameters ParameterKey=K,ParameterValue=V` |
| Estado y salidas | `aws cloudformation describe-stacks --stack-name S` |
| Recursos | `aws cloudformation list-stack-resources --stack-name S` |
| Eventos | `aws cloudformation describe-stack-events --stack-name S` |
| Actualizar | `aws cloudformation update-stack --stack-name S --template-body file://p.yaml --parameters ParameterKey=K,UsePreviousValue=true` |
| Borrar | `aws cloudformation delete-stack --stack-name S` |
