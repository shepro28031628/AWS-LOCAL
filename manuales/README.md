# Manuales de AWS Nube Local

Guías cortas y paso a paso para usar AWS Nube Local, aunque nunca hayas usado AWS.

Síguelas en orden la primera vez. Cada manual tarda entre 5 y 15 minutos.

| # | Manual | Qué aprendes |
|---|---|---|
| 1 | [Instalar y arrancar](01-instalar-y-arrancar.md) | Descargar el proyecto, encenderlo y apagarlo |
| 2 | [La consola web](02-consola-web.md) | Ver tus recursos desde el navegador |
| 3 | [Configurar la AWS CLI](03-configurar-aws-cli.md) | Preparar la terminal para hablar con tu nube local |
| 4 | [S3: guardar archivos](04-s3-archivos.md) | Crear buckets, subir, descargar y borrar archivos |
| 5 | [DynamoDB: base de datos](05-dynamodb-base-de-datos.md) | Crear una tabla y guardar, leer y borrar registros |
| 6 | [SQS y SNS: mensajes](06-sqs-sns-mensajes.md) | Enviar mensajes entre programas y avisos a varios destinos |
| 7 | [Lambda: funciones](07-lambda-funciones.md) | Crear y ejecutar una función sin servidor |
| 8 | [Programar con Python](08-python-boto3.md) | Usar todo lo anterior desde un programa en Python |
| 9 | [Problemas frecuentes](09-problemas-frecuentes.md) | Qué hacer cuando algo no funciona |

## Antes de empezar: tres ideas clave

1. **Todo vive en tu computadora.** Nada se sube a internet ni a una cuenta real de AWS. No cuesta dinero.
2. **La dirección es siempre `http://localhost:4566`.** Ahí responde tu nube local.
3. **Las credenciales son de mentira.** Usa `test` como usuario y como contraseña.

## Cómo leer los manuales

- Los bloques de código son comandos para copiar y pegar en la terminal.
- Si un comando tiene una versión distinta para Windows y para macOS/Linux, se indica.
- Donde ves `mi-bucket`, `alumnos`, etc., puedes poner tus propios nombres.
