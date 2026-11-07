# Manuales de AWS Nube Local

Guías prácticas para aprender a **trabajar** con los principales servicios de AWS en tu propia computadora, sin cuenta de AWS y sin costes.

Cada manual explica qué es el servicio y cuándo usarlo, sus conceptos clave, una serie de **ejercicios** que van de lo básico a lo avanzado, un **reto** final y una **chuleta** de comandos. Todos los ejercicios se han probado en AWS Nube Local con la AWS CLI v2.

## Primeros pasos (hazlos primero)

| Manual | Qué aprendes |
|---|---|
| [1. Instalar y arrancar](01-instalar-y-arrancar.md) | Descargar el proyecto, encenderlo y apagarlo |
| [2. La consola web](02-consola-web.md) | Ver tus recursos desde el navegador |
| [3. Configurar la AWS CLI](03-configurar-aws-cli.md) | Preparar la terminal para hablar con tu nube local |

## Almacenamiento y bases de datos

| Manual | Qué aprendes |
|---|---|
| [S3](10-s3.md) | Buckets, versionado, metadatos, URLs prefirmadas, ciclo de vida, eventos, web estática |
| [DynamoDB](11-dynamodb.md) | Claves compuestas, query, índices secundarios, expresiones, escrituras condicionales, TTL, Streams |
| [RDS](12-rds.md) | PostgreSQL y MySQL reales, SQL con JOIN, snapshots y restauración |
| [ElastiCache](13-elasticache.md) | Redis: caché, contadores, caducidad, rankings, patrón *cache-aside* |

## Analítica y datos

| Manual | Qué aprendes |
|---|---|
| [Glue + Athena](20-glue-athena.md) | Data lake con arquitectura medallón (Bronze, Silver, Gold), Parquet y SQL |
| [Kinesis y Firehose](21-kinesis.md) | Datos en tiempo real: productores, consumidores y entrega a S3 |

## Mensajería y eventos

| Manual | Qué aprendes |
|---|---|
| [SQS](30-sqs.md) | Colas estándar y FIFO, visibilidad, colas de errores (DLQ), trabajadores |
| [SNS](31-sns.md) | Reparto a varios destinos (*fan-out*) y políticas de filtro |
| [EventBridge](32-eventbridge.md) | Buses de eventos, reglas por contenido, transformaciones, tareas programadas |

## Cómputo y APIs

| Manual | Qué aprendes |
|---|---|
| [Lambda](40-lambda.md) | Funciones, configuración, logs, versiones, disparadores desde SQS y DynamoDB, Function URLs |
| [API Gateway](41-api-gateway.md) | Una API REST completa con Lambda y DynamoDB |
| [Step Functions](42-step-functions.md) | Flujos de trabajo con decisiones, esperas, bucles, reintentos y control de errores |
| [ECS](43-ecs.md) | Contenedores, definiciones de tarea, servicios, escalado y autorreparación |
| [EC2](44-ec2.md) | Servidores virtuales, claves SSH, firewall, scripts de arranque, redes |

## Seguridad e identidad

| Manual | Qué aprendes |
|---|---|
| [IAM y STS](50-iam-sts.md) | Usuarios, grupos, políticas, roles, credenciales temporales, modo estricto |
| [Secrets Manager y Parameter Store](51-secrets-ssm.md) | Guardar contraseñas y configuración fuera del código |
| [KMS](52-kms.md) | Cifrar y descifrar datos, cifrado en S3, data keys, rotación |
| [Cognito](53-cognito.md) | Registro e inicio de sesión de usuarios con tokens JWT |

## Operación e infraestructura como código

| Manual | Qué aprendes |
|---|---|
| [CloudWatch](60-cloudwatch.md) | Logs, Logs Insights, métricas propias y alarmas que avisan por SNS |
| [CloudFormation](61-cloudformation.md) | Crear, actualizar y borrar entornos completos desde una plantilla |

## Programación y proyecto final

| Manual | Qué aprendes |
|---|---|
| [Programar con Python](70-python-boto3.md) | Usar los servicios desde código con boto3 |
| [Proyecto integrador](80-proyecto-integrador.md) | Sistema de pedidos con API Gateway, Lambda, DynamoDB, EventBridge, SQS y S3 |
| [Problemas frecuentes](90-problemas-frecuentes.md) | Qué hacer cuando algo no funciona |

## Ruta recomendada

Si es tu primera vez con AWS, sigue este orden:

1. Primeros pasos (1, 2, 3)
2. [S3](10-s3.md), [DynamoDB](11-dynamodb.md) y [SQS](30-sqs.md)
3. [Lambda](40-lambda.md) y [API Gateway](41-api-gateway.md)
4. [IAM](50-iam-sts.md) y [CloudWatch](60-cloudwatch.md)
5. El [proyecto integrador](80-proyecto-integrador.md)
6. El resto, según lo que necesites (analítica, contenedores, seguridad avanzada...)

## Tres ideas clave

1. **Todo vive en tu computadora.** Nada se sube a una cuenta real de AWS. No cuesta dinero.
2. **La dirección es siempre `http://localhost:4566`.** Ahí responde tu nube local.
3. **Las credenciales son de prueba.** Usa `test` como usuario y como contraseña.

## Cómo leer los manuales

- Los bloques de código son comandos para copiar y pegar en la terminal.
- Cuando un comando cambia entre **Windows (PowerShell)** y **macOS / Linux**, se muestran los dos.
- Los archivos `.json`, `.py` o `.yaml` que se mencionan se crean en tu carpeta de trabajo con cualquier editor (Bloc de notas, VS Code...).
- Donde ves `ID`, `ARN_...` o `PEGA_AQUÍ...`, sustitúyelo por el valor que te devolvió un comando anterior.
- Por defecto los datos viven **en memoria**: al apagar AWS Nube Local se borran. Es útil para empezar de cero.

## Limitaciones conocidas del simulador

Los manuales ya tienen en cuenta estas diferencias con AWS real:

| Servicio | Diferencia |
|---|---|
| IAM | Las políticas no se aplican salvo que actives el modo estricto ([manual de IAM](50-iam-sts.md)) |
| Glue | Los *crawlers* y *jobs* no procesan datos: las tablas se definen a mano |
| Athena | Solo consultas `SELECT` (no `CREATE TABLE AS`) |
| EventBridge | Los filtros numéricos en patrones no funcionan |
| CloudWatch | Logs Insights admite solo `fields`, `filter =`, `sort`, `dedup` y `limit` |
| ECS | Los contenedores con `hostPort` 0 no se publican en tu PC |
