# KMS: cifrado de datos

[Volver al índice](README.md)

## Qué es y cuándo usarlo

**AWS KMS** (*Key Management Service*) crea y guarda **claves de cifrado** de forma segura. Tú nunca ves la clave: le pides a KMS que cifre o descifre por ti.

Muchos servicios lo usan por debajo: S3 para cifrar objetos, Parameter Store para los `SecureString`, Secrets Manager para los secretos...

Úsalo para:

- Cifrar datos sensibles (números de documento, datos médicos) antes de guardarlos.
- Cifrar archivos en S3 y que solo quien tenga permiso sobre la clave pueda leerlos.
- **Cifrado de sobre** (*envelope encryption*) para archivos grandes.

## Conceptos clave

| Concepto | Qué es |
|---|---|
| **Clave KMS** | La clave maestra. Nunca sale de KMS |
| **Alias** | Nombre fácil para una clave: `alias/clase` |
| **Cifrar / descifrar** | `encrypt` / `decrypt` (hasta 4 KB de datos directamente) |
| **Data key** | Clave que KMS genera para que **tú** cifres datos grandes en tu programa |
| **Rotación** | Cambiar el material de la clave periódicamente sin romper lo ya cifrado |

## Antes de empezar

Terminal configurada ([manual 3](03-configurar-aws-cli.md)). Crea `confidencial.txt` con un texto cualquiera.

---

## Ejercicio 1: crear una clave y su alias

**Windows (PowerShell):**

```bash
$KEY = aws kms create-key --description "Clave de la clase" --query KeyMetadata.KeyId --output text
```

**macOS / Linux:**

```bash
KEY=$(aws kms create-key --description "Clave de la clase" --query KeyMetadata.KeyId --output text)
```

```bash
aws kms create-alias --alias-name alias/clase --target-key-id $KEY
```

```bash
aws kms list-aliases --query "Aliases[?starts_with(AliasName, 'alias/clase')]"
```

## Ejercicio 2: cifrar y descifrar

Cifra el archivo. El resultado se guarda en binario:

**Windows (PowerShell):**

```bash
aws kms encrypt --key-id alias/clase --plaintext fileb://confidencial.txt --query CiphertextBlob --output text | Out-File -Encoding ascii cifrado.b64
```

```bash
[IO.File]::WriteAllBytes("$PWD\cifrado.bin", [Convert]::FromBase64String((Get-Content cifrado.b64 -Raw).Trim()))
```

**macOS / Linux:**

```bash
aws kms encrypt --key-id alias/clase --plaintext fileb://confidencial.txt --query CiphertextBlob --output text | base64 -d > cifrado.bin
```

Abre `cifrado.bin`: es ilegible. Descífralo:

**Windows (PowerShell):**

```bash
$b64 = aws kms decrypt --ciphertext-blob fileb://cifrado.bin --query Plaintext --output text; [Text.Encoding]::UTF8.GetString([Convert]::FromBase64String($b64))
```

**macOS / Linux:**

```bash
aws kms decrypt --ciphertext-blob fileb://cifrado.bin --query Plaintext --output text | base64 -d
```

Aparece el texto original. Fíjate en que **no hizo falta decir qué clave usar** para descifrar: el texto cifrado lleva dentro esa información.

> Las conversiones base64 son molestas en la terminal. Desde Python (ejercicio 4) es mucho más simple.

## Ejercicio 3: cifrar archivos en S3 con tu clave

```bash
aws s3 mb s3://docs-clase
```

```bash
aws s3 cp confidencial.txt s3://docs-clase/secreto.txt --sse aws:kms --sse-kms-key-id alias/clase
```

```bash
aws s3api head-object --bucket docs-clase --key secreto.txt --query "[ServerSideEncryption,SSEKMSKeyId]"
```

Responde `aws:kms` y tu clave. Al descargarlo, S3 lo descifra solo (si tienes permiso sobre la clave):

```bash
aws s3 cp s3://docs-clase/secreto.txt -
```

## Ejercicio 4: cifrar desde Python

`cifrar.py`:

```python
import boto3

CONFIG = dict(endpoint_url="http://localhost:4566", region_name="us-east-1",
              aws_access_key_id="test", aws_secret_access_key="test")
kms = boto3.client("kms", **CONFIG)

documento = "CC 1.020.304.050"

cifrado = kms.encrypt(KeyId="alias/clase", Plaintext=documento.encode())["CiphertextBlob"]
print("Cifrado:", cifrado[:30], "...")

descifrado = kms.decrypt(CiphertextBlob=cifrado)["Plaintext"].decode()
print("Descifrado:", descifrado)
```

En una app real guardarías `cifrado` en la base de datos en lugar del dato original.

## Ejercicio 5: cifrado de sobre para datos grandes

`encrypt` solo admite 4 KB. Para archivos grandes, KMS te da una **data key**: la usas para cifrar en tu programa y guardas la data key **cifrada** junto al archivo.

```bash
aws kms generate-data-key --key-id alias/clase --key-spec AES_256
```

La respuesta trae:

| Campo | Qué hacer con él |
|---|---|
| `Plaintext` | La clave en claro: úsala para cifrar el archivo y **bórrala de memoria** |
| `CiphertextBlob` | La misma clave cifrada por KMS: **guárdala** junto al archivo |

Para descifrar más tarde: `kms decrypt` del `CiphertextBlob` devuelve la clave y con ella descifras el archivo. Así funciona S3 por dentro.

## Ejercicio 6: rotación y desactivación

```bash
aws kms enable-key-rotation --key-id $KEY
```

```bash
aws kms get-key-rotation-status --key-id $KEY
```

Desactivar una clave (lo cifrado con ella deja de poder descifrarse mientras esté desactivada):

```bash
aws kms disable-key --key-id $KEY
```

Intenta descifrar ahora: falla con `DisabledException`. Reactívala:

```bash
aws kms enable-key --key-id $KEY
```

Programar el borrado (mínimo 7 días; en AWS real es irreversible al cumplirse):

```bash
aws kms schedule-key-deletion --key-id $KEY --pending-window-in-days 7
```

```bash
aws kms cancel-key-deletion --key-id $KEY
```

---

## Reto

1. Crea una clave `alias/datos-personales`.
2. Haz un script en Python que lea un CSV con nombres y documentos, cifre la columna de documentos con KMS y guarde un CSV nuevo.
3. Haz otro script que descifre ese CSV.
4. Desactiva la clave y comprueba que el segundo script falla.

## Chuleta

| Quiero... | Comando |
|---|---|
| Crear clave | `aws kms create-key --description "..."` |
| Alias | `aws kms create-alias --alias-name alias/N --target-key-id ID` |
| Cifrar | `aws kms encrypt --key-id alias/N --plaintext fileb://archivo` |
| Descifrar | `aws kms decrypt --ciphertext-blob fileb://cifrado.bin` |
| Data key | `aws kms generate-data-key --key-id alias/N --key-spec AES_256` |
| S3 cifrado | `aws s3 cp f s3://B/k --sse aws:kms --sse-kms-key-id alias/N` |
| Rotación | `aws kms enable-key-rotation --key-id ID` |
| Desactivar | `aws kms disable-key` / `enable-key --key-id ID` |
