# Secrets Manager y Parameter Store: configuración y secretos

[Volver al índice](README.md)

## Qué es y cuándo usarlo

Las contraseñas, claves de API y configuraciones **nunca** deben escribirse dentro del código ni subirse a GitHub. AWS ofrece dos servicios para guardarlas aparte:

| Servicio | Para qué | Ejemplo |
|---|---|---|
| **Secrets Manager** | Secretos sensibles, con versiones y rotación | Usuario y contraseña de la base de datos |
| **SSM Parameter Store** | Configuración en general, organizada por rutas, opcionalmente cifrada | URL de una API, límites, banderas de funciones |

Tu aplicación los **lee al arrancar** con el SDK. Cambias un valor en AWS y no tienes que tocar ni redesplegar el código.

## Conceptos clave

| Concepto | Qué es |
|---|---|
| **Secreto** | Un valor guardado (normalmente JSON) con nombre, por ejemplo `tienda/db` |
| **Versión** y **etapa** | Cada cambio crea una versión. `AWSCURRENT` es la actual y `AWSPREVIOUS` la anterior |
| **Parámetro** | Un valor con nombre jerárquico: `/tienda/prod/url-api` |
| **Tipos de parámetro** | `String`, `StringList` (separado por comas) y `SecureString` (cifrado con KMS) |
| **Jerarquía** | Las rutas permiten leer todos los parámetros de un entorno de golpe |

## Antes de empezar

Terminal configurada ([manual 3](03-configurar-aws-cli.md)).

---

## Parte 1: Secrets Manager

### Ejercicio 1: guardar un secreto

Crea `secreto.json`:

```json
{ "usuario": "admin", "password": "Clase2026!", "host": "localhost", "puerto": 7001 }
```

```bash
aws secretsmanager create-secret --name tienda/db --description "Credenciales de la base de datos" --secret-string file://secreto.json
```

### Ejercicio 2: leerlo

```bash
aws secretsmanager get-secret-value --secret-id tienda/db --query SecretString --output text
```

Listar los secretos (sin mostrar sus valores):

```bash
aws secretsmanager list-secrets --query "SecretList[].[Name,Description]" --output table
```

### Ejercicio 3: cambiarlo y ver las versiones

Crea `secreto-nuevo.json` con otra contraseña y actualiza:

```bash
aws secretsmanager put-secret-value --secret-id tienda/db --secret-string file://secreto-nuevo.json
```

```bash
aws secretsmanager list-secret-version-ids --secret-id tienda/db --query "Versions[].[VersionId,VersionStages[0]]" --output table
```

Hay dos versiones: `AWSCURRENT` (la nueva) y `AWSPREVIOUS` (la anterior). Leer la anterior, por si hay que volver atrás:

```bash
aws secretsmanager get-secret-value --secret-id tienda/db --version-stage AWSPREVIOUS --query SecretString --output text
```

### Ejercicio 4: generar contraseñas seguras

```bash
aws secretsmanager get-random-password --password-length 20 --exclude-punctuation --query RandomPassword --output text
```

### Ejercicio 5: usarlo desde Python

`conectar.py`:

```python
import json
import boto3

CONFIG = dict(endpoint_url="http://localhost:4566", region_name="us-east-1",
              aws_access_key_id="test", aws_secret_access_key="test")
secrets = boto3.client("secretsmanager", **CONFIG)

credenciales = json.loads(secrets.get_secret_value(SecretId="tienda/db")["SecretString"])
print(f"Conectando a {credenciales['host']}:{credenciales['puerto']} como {credenciales['usuario']}")
# aquí usarías credenciales["password"] para conectarte, sin escribirla nunca en el código
```

### Borrar

AWS no borra al momento: deja un periodo para recuperarlo. Para borrarlo ya:

```bash
aws secretsmanager delete-secret --secret-id tienda/db --force-delete-without-recovery
```

---

## Parte 2: SSM Parameter Store

### Ejercicio 6: parámetros organizados por entorno

```bash
aws ssm put-parameter --name /tienda/prod/url-api --value "https://api.tienda.com" --type String
```

```bash
aws ssm put-parameter --name /tienda/prod/max-items --value 50 --type String
```

```bash
aws ssm put-parameter --name /tienda/prod/paises --value "CO,MX,ES" --type StringList
```

Y uno cifrado:

```bash
aws ssm put-parameter --name /tienda/prod/api-key --value "sk-123456" --type SecureString
```

Crea también uno para desarrollo:

```bash
aws ssm put-parameter --name /tienda/dev/url-api --value "http://localhost:3000" --type String
```

### Ejercicio 7: leer parámetros

Uno solo:

```bash
aws ssm get-parameter --name /tienda/prod/url-api --query Parameter.Value --output text
```

El cifrado **sin** y **con** descifrar:

```bash
aws ssm get-parameter --name /tienda/prod/api-key --query Parameter.Value --output text
```

```bash
aws ssm get-parameter --name /tienda/prod/api-key --with-decryption --query Parameter.Value --output text
```

El primero muestra un texto cifrado ilegible; el segundo, `sk-123456`.

**Todos los de producción** de golpe:

```bash
aws ssm get-parameters-by-path --path /tienda/prod --with-decryption --query "Parameters[].[Name,Value]" --output table
```

> Así una app lee **toda su configuración** con una sola llamada, y cambiando `/prod` por `/dev` tiene la de desarrollo.

### Ejercicio 8: historial de cambios

```bash
aws ssm put-parameter --name /tienda/prod/max-items --value 80 --type String --overwrite
```

```bash
aws ssm get-parameter-history --name /tienda/prod/max-items --query "Parameters[].[Version,Value,LastModifiedDate]" --output table
```

### Ejercicio 9: desde Python

```python
import boto3

CONFIG = dict(endpoint_url="http://localhost:4566", region_name="us-east-1",
              aws_access_key_id="test", aws_secret_access_key="test")
ssm = boto3.client("ssm", **CONFIG)

entorno = "prod"
respuesta = ssm.get_parameters_by_path(Path=f"/tienda/{entorno}", WithDecryption=True)
config = {p["Name"].split("/")[-1]: p["Value"] for p in respuesta["Parameters"]}

print(config)
print("Máximo de artículos:", int(config["max-items"]))
```

### Borrar

```bash
aws ssm delete-parameter --name /tienda/dev/url-api
```

---

## ¿Cuál uso?

| Necesito... | Usa |
|---|---|
| Contraseñas de bases de datos, tokens de terceros | Secrets Manager |
| Rotar contraseñas automáticamente | Secrets Manager |
| URLs, límites, nombres de buckets, banderas | Parameter Store (`String`) |
| Una clave de API sencilla, sin rotación | Parameter Store (`SecureString`) |

## Reto

1. Guarda en Parameter Store la configuración de dos entornos (`/miapp/dev/...` y `/miapp/prod/...`) con al menos 3 parámetros cada uno, uno de ellos `SecureString`.
2. Escribe un script en Python que reciba el entorno como argumento (`python app.py dev`) y muestre su configuración.
3. Guarda en Secrets Manager las credenciales de la base de datos del [manual de RDS](12-rds.md) y modifica el script para conectarse leyéndolas de ahí.

## Chuleta

| Quiero... | Comando |
|---|---|
| Crear secreto | `aws secretsmanager create-secret --name N --secret-string file://s.json` |
| Leer secreto | `aws secretsmanager get-secret-value --secret-id N` |
| Cambiar secreto | `aws secretsmanager put-secret-value --secret-id N --secret-string file://s.json` |
| Contraseña aleatoria | `aws secretsmanager get-random-password --password-length 20` |
| Crear parámetro | `aws ssm put-parameter --name /ruta/p --value V --type String` |
| Leer parámetro | `aws ssm get-parameter --name /ruta/p [--with-decryption]` |
| Leer por ruta | `aws ssm get-parameters-by-path --path /ruta --with-decryption` |
| Historial | `aws ssm get-parameter-history --name /ruta/p` |
