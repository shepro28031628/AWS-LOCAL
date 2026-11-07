# 7. Lambda: funciones

[Volver al índice](README.md)

**Lambda** ejecuta un trozo de código cuando se lo pides, sin que tengas que montar un servidor. Tú escribes la función y AWS (aquí, tu nube local) la ejecuta.

> Antes de empezar, configura la terminal con el [manual 3](03-configurar-aws-cli.md).
>
> La **primera vez** que ejecutes una función, AWS Nube Local descarga la imagen oficial de Python para Lambda (unos cientos de MB). Necesitas internet y tarda un poco. Después es rápido.

## Paso 1: escribir la función

Crea una carpeta nueva, entra en ella y crea el archivo **`saludo.py`**:

```python
def handler(event, context):
    nombre = event.get("nombre", "mundo")
    return {
        "mensaje": f"Hola, {nombre}! Esto se ejecutó en AWS Nube Local."
    }
```

- `handler` es la función que Lambda llama.
- `event` es lo que le mandas al ejecutarla.

## Paso 2: comprimirla en un ZIP

Lambda recibe el código en un archivo ZIP.

**Windows (PowerShell):**

```bash
Compress-Archive -Path saludo.py -DestinationPath funcion.zip -Force
```

**macOS / Linux:**

```bash
zip funcion.zip saludo.py
```

## Paso 3: crear la función

```bash
aws lambda create-function --function-name saludo --runtime python3.12 --handler saludo.handler --zip-file fileb://funcion.zip --role arn:aws:iam::000000000000:role/lambda-role
```

| Parte | Significado |
|---|---|
| `--runtime python3.12` | Lenguaje y versión |
| `--handler saludo.handler` | Archivo `saludo.py`, función `handler` |
| `--zip-file fileb://funcion.zip` | El código (fíjate: `fileb://`, con **b**) |
| `--role ...` | Permisos. En local no se comprueban: copia el valor tal cual |

Comprueba que se creó:

```bash
aws lambda list-functions --query "Functions[].FunctionName"
```

## Paso 4: ejecutarla

Crea **`evento.json`**:

```json
{ "nombre": "Ana" }
```

Y ejecuta:

```bash
aws lambda invoke --function-name saludo --payload fileb://evento.json respuesta.json
```

Mira el resultado:

**Windows (PowerShell):**

```bash
Get-Content respuesta.json
```

**macOS / Linux:**

```bash
cat respuesta.json
```

Debes ver: `{"mensaje": "Hola, Ana! Esto se ejecutó en AWS Nube Local."}`

## Paso 5: cambiar el código

Edita `saludo.py`, vuelve a crear el ZIP (paso 2) y actualiza la función:

```bash
aws lambda update-function-code --function-name saludo --zip-file fileb://funcion.zip
```

Ejecútala otra vez (paso 4) para ver el cambio.

## Paso 6: borrar

```bash
aws lambda delete-function --function-name saludo
```

## ¿Qué pasa por dentro?

Cuando ejecutas la función, AWS Nube Local arranca un **contenedor de Docker real** con Python, mete tu código y lo ejecuta. Si miras Docker Desktop mientras se ejecuta, verás aparecer ese contenedor.

**Siguiente:** [8. Programar con Python](08-python-boto3.md)
