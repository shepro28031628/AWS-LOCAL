# 4. S3: guardar archivos

[Volver al índice](README.md)

**S3** es el "disco en la nube" de AWS. Guardas archivos (imágenes, PDFs, datos...) dentro de **buckets**, que son como carpetas principales.

> Antes de empezar, configura la terminal con el [manual 3](03-configurar-aws-cli.md).

## Paso 1: crear un bucket

```bash
aws s3 mb s3://mis-archivos
```

`mb` = *make bucket* (crear bucket). Los nombres van en minúsculas, sin espacios ni tildes.

## Paso 2: crear un archivo de prueba

**Windows (PowerShell):**

```bash
Set-Content -Path nota.txt -Value "Hola clase"
```

**macOS / Linux:**

```bash
echo "Hola clase" > nota.txt
```

## Paso 3: subirlo

```bash
aws s3 cp nota.txt s3://mis-archivos/
```

## Paso 4: ver qué hay en el bucket

```bash
aws s3 ls s3://mis-archivos
```

Y para ver todos tus buckets:

```bash
aws s3 ls
```

## Paso 5: descargarlo

```bash
aws s3 cp s3://mis-archivos/nota.txt descargada.txt
```

## Paso 6: usar carpetas

En S3 las "carpetas" son parte del nombre del archivo:

```bash
aws s3 cp nota.txt s3://mis-archivos/2026/noviembre/nota.txt
```

```bash
aws s3 ls s3://mis-archivos --recursive
```

## Paso 7: subir una carpeta entera

```bash
aws s3 sync . s3://mis-archivos/copia/
```

Sube todos los archivos de la carpeta actual. Úsalo con cuidado si la carpeta es grande.

## Paso 8: borrar

Borrar un archivo:

```bash
aws s3 rm s3://mis-archivos/nota.txt
```

Borrar el bucket con todo lo que tenga dentro:

```bash
aws s3 rb s3://mis-archivos --force
```

## Resumen

| Quiero... | Comando |
|---|---|
| Crear bucket | `aws s3 mb s3://NOMBRE` |
| Listar buckets | `aws s3 ls` |
| Subir archivo | `aws s3 cp ARCHIVO s3://NOMBRE/` |
| Ver contenido | `aws s3 ls s3://NOMBRE --recursive` |
| Descargar | `aws s3 cp s3://NOMBRE/ARCHIVO destino` |
| Borrar archivo | `aws s3 rm s3://NOMBRE/ARCHIVO` |
| Borrar bucket | `aws s3 rb s3://NOMBRE --force` |

**Siguiente:** [5. DynamoDB: base de datos](05-dynamodb-base-de-datos.md)
