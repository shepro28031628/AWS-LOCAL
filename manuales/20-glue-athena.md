# Glue + Athena: data lake y consultas SQL

[Volver al índice](README.md)

## Qué es y cuándo usarlo

Juntos forman un **data lake** (lago de datos) sin servidores:

| Servicio | Papel |
|---|---|
| **S3** | Guarda los archivos de datos (CSV, JSON, Parquet) |
| **Glue Data Catalog** | Describe esos archivos como **tablas**: nombre, columnas, tipos y dónde están |
| **Athena** | Ejecuta **SQL** directamente sobre los archivos de S3 usando el catálogo |

No hay base de datos que encender: subes archivos, los describes y los consultas. Es el equivalente en AWS a lo que harías en Databricks con tablas Delta y `%sql`.

En AWS Nube Local, Athena ejecuta el SQL de verdad con **DuckDB**, así que admite `GROUP BY`, `HAVING`, `WITH`, funciones de ventana (`DENSE_RANK() OVER ...`), etc.

## Conceptos clave

| Concepto | Qué es |
|---|---|
| **Arquitectura medallón** | Capas de calidad creciente: **Bronze** (datos crudos), **Silver** (limpios), **Gold** (agregados para informes) |
| **Base de datos de Glue** | Agrupa tablas (aquí, `tienda`) |
| **Tabla externa** | Una tabla cuyos datos son archivos en un prefijo de S3 |
| **Parquet** | Formato **columnar**: comprimido y mucho más rápido para analítica que CSV |
| **Ubicación de resultados** | Prefijo de S3 donde Athena guarda cada resultado como CSV |

## Limitaciones del simulador (léelas)

| En AWS real | En AWS Nube Local |
|---|---|
| Los **crawlers** de Glue detectan el esquema y crean tablas solos | Los crawlers no crean tablas: **defines la tabla a mano** (ejercicio 3) |
| Los **jobs** de Glue ejecutan Spark | Los jobs no ejecutan código: transforma los datos con Python en tu PC |
| `CREATE TABLE AS SELECT` crea tablas | Solo funcionan consultas **`SELECT`**: la capa Gold se crea guardando el resultado de una consulta (ejercicio 6) |

La primera consulta descarga una imagen extra (`floci/floci-duck`). Necesitas internet esa vez.

## Antes de empezar

- Terminal configurada ([manual 3](03-configurar-aws-cli.md)).
- Python con pandas y pyarrow para generar los datos:

```bash
pip install pandas pyarrow
```

---

## Ejercicio 1: generar los datos (capa Bronze y Silver)

Crea `generar_ventas.py`. Genera 10 000 ventas sintéticas, las guarda en CSV (Bronze, crudo) y en Parquet ya limpio y con `monto_total` calculado (Silver):

```python
import random
import pandas as pd

random.seed(42)
filas = []
for i in range(1, 10001):
    filas.append({
        "id_orden": f"ORD-{i:07d}",
        "id_cliente": f"CLI-{random.randint(1, 2000):05d}",
        "categoria": random.choice(["electronica", " hogar", "ROPA", "Deportes "]),
        "region": random.choice(["NORTE", "SUR", "ESTE", "OESTE"]),
        "precio_unitario": round(random.uniform(10, 310), 2),
        "cantidad": random.randint(1, 5),
    })

bronze = pd.DataFrame(filas)
bronze.to_csv("ventas.csv", index=False)

# Silver: limpieza y transformación
silver = bronze.copy()
silver["categoria"] = silver["categoria"].str.strip().str.upper()
silver["monto_total"] = (silver["precio_unitario"] * silver["cantidad"]).round(2)
silver = silver[silver["precio_unitario"] > 0]
silver.to_parquet("ventas.parquet", index=False)

print("Bronze:", len(bronze), "filas | Silver:", len(silver), "filas")
```

```bash
python generar_ventas.py
```

Fíjate en que las categorías del CSV están "sucias" (minúsculas, espacios). En Silver quedan limpias.

## Ejercicio 2: subir los datos al lago

```bash
aws s3 mb s3://lago-datos
```

```bash
aws s3 cp ventas.csv s3://lago-datos/bronze/ventas/ventas.csv
```

```bash
aws s3 cp ventas.parquet s3://lago-datos/silver/ventas/ventas.parquet
```

> Cada tabla apunta a una **carpeta** (prefijo), no a un archivo. Si subes más archivos a esa carpeta, la tabla los incluye automáticamente.

## Ejercicio 3: registrar las tablas en Glue

Crea la base de datos:

```bash
aws glue create-database --database-input Name=tienda
```

**Tabla Bronze (CSV).** Crea `tabla-bronze.json`:

```json
{
  "Name": "ventas_raw",
  "TableType": "EXTERNAL_TABLE",
  "Parameters": { "classification": "csv", "skip.header.line.count": "1" },
  "StorageDescriptor": {
    "Columns": [
      { "Name": "id_orden",        "Type": "string" },
      { "Name": "id_cliente",      "Type": "string" },
      { "Name": "categoria",       "Type": "string" },
      { "Name": "region",          "Type": "string" },
      { "Name": "precio_unitario", "Type": "double" },
      { "Name": "cantidad",        "Type": "int" }
    ],
    "Location": "s3://lago-datos/bronze/ventas/",
    "InputFormat": "org.apache.hadoop.mapred.TextInputFormat",
    "OutputFormat": "org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat",
    "SerdeInfo": {
      "SerializationLibrary": "org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe",
      "Parameters": { "field.delim": "," }
    }
  }
}
```

```bash
aws glue create-table --database-name tienda --table-input file://tabla-bronze.json
```

**Tabla Silver (Parquet).** Crea `tabla-silver.json`:

```json
{
  "Name": "ventas_limpias",
  "TableType": "EXTERNAL_TABLE",
  "Parameters": { "classification": "parquet" },
  "StorageDescriptor": {
    "Columns": [
      { "Name": "id_orden",        "Type": "string" },
      { "Name": "id_cliente",      "Type": "string" },
      { "Name": "categoria",       "Type": "string" },
      { "Name": "region",          "Type": "string" },
      { "Name": "precio_unitario", "Type": "double" },
      { "Name": "cantidad",        "Type": "bigint" },
      { "Name": "monto_total",     "Type": "double" }
    ],
    "Location": "s3://lago-datos/silver/ventas/",
    "InputFormat": "org.apache.hadoop.hive.ql.io.parquet.MapredParquetInputFormat",
    "OutputFormat": "org.apache.hadoop.hive.ql.io.parquet.MapredParquetOutputFormat",
    "SerdeInfo": {
      "SerializationLibrary": "org.apache.hadoop.hive.ql.io.parquet.serde.ParquetHiveSerDe"
    }
  }
}
```

```bash
aws glue create-table --database-name tienda --table-input file://tabla-silver.json
```

Comprueba:

```bash
aws glue get-tables --database-name tienda --query "TableList[].[Name,StorageDescriptor.Location]" --output table
```

## Ejercicio 4: tu primera consulta con Athena

Una consulta de Athena tiene **tres pasos**: lanzarla, esperar a que termine y leer el resultado.

**1. Lanzar** (guarda el identificador en una variable):

**Windows (PowerShell):**

```bash
$Q = aws athena start-query-execution --query-string "SELECT count(*) AS filas FROM ventas_raw" --query-execution-context Database=tienda --result-configuration OutputLocation=s3://lago-datos/resultados/ --query QueryExecutionId --output text
```

**macOS / Linux:**

```bash
Q=$(aws athena start-query-execution --query-string "SELECT count(*) AS filas FROM ventas_raw" --query-execution-context Database=tienda --result-configuration OutputLocation=s3://lago-datos/resultados/ --query QueryExecutionId --output text)
```

**2. Ver el estado** (repite hasta que diga `SUCCEEDED`; si dice `FAILED`, mira el motivo):

```bash
aws athena get-query-execution --query-execution-id $Q --query "QueryExecution.Status.[State,StateChangeReason]" --output text
```

**3. Leer el resultado:**

```bash
aws athena get-query-results --query-execution-id $Q --output table
```

Debe decir `10000`. La primera consulta tarda más porque arranca el motor DuckDB.

> **Más cómodo:** el ejercicio 7 hace los tres pasos con un solo comando desde Python.

## Ejercicio 5: los retos del taller en SQL

Lanza cada consulta como en el ejercicio 4, cambiando solo el texto de `--query-string`.

**Agregación por región y categoría:**

```sql
SELECT region, categoria,
       COUNT(DISTINCT id_cliente) AS clientes_unicos,
       COUNT(id_orden)            AS total_ventas,
       ROUND(SUM(monto_total), 2) AS ingresos_totales,
       ROUND(AVG(monto_total), 2) AS ticket_promedio
FROM ventas_limpias
GROUP BY region, categoria
ORDER BY ingresos_totales DESC
```

**Top 5 clientes por región (función de ventana):**

```sql
WITH ranking AS (
  SELECT region, id_cliente, SUM(monto_total) AS gasto,
         DENSE_RANK() OVER (PARTITION BY region ORDER BY SUM(monto_total) DESC) AS posicion
  FROM ventas_limpias
  GROUP BY region, id_cliente
)
SELECT region, posicion, id_cliente, ROUND(gasto, 2) AS gasto
FROM ranking
WHERE posicion <= 5
ORDER BY region, posicion
```

**Categorías con ticket promedio mayor a 100 (HAVING):**

```sql
SELECT categoria, ROUND(AVG(monto_total), 2) AS ticket
FROM ventas_limpias
GROUP BY categoria
HAVING AVG(monto_total) > 100
ORDER BY ticket DESC
```

**Comparar Bronze con Silver** (ver la limpieza de categorías):

```sql
SELECT categoria, COUNT(*) AS filas FROM ventas_raw GROUP BY categoria
```

> Escribir consultas largas en una sola línea de la terminal es incómodo. Usa Python (ejercicio 7) o la consola web.

## Ejercicio 6: crear la capa Gold

Como `CREATE TABLE AS` no está disponible en el simulador, la capa Gold se crea así:

1. Lanza la consulta de agregación del ejercicio 5 (la primera) y espera a `SUCCEEDED`.
2. Athena guardó el resultado como CSV en `s3://lago-datos/resultados/ID.csv`. Cópialo a la carpeta Gold:

```bash
aws s3 cp s3://lago-datos/resultados/$Q.csv s3://lago-datos/gold/resumen/resumen.csv
```

3. Registra la tabla Gold. Crea `tabla-gold.json`:

```json
{
  "Name": "resumen_gold",
  "TableType": "EXTERNAL_TABLE",
  "Parameters": { "classification": "csv", "skip.header.line.count": "1" },
  "StorageDescriptor": {
    "Columns": [
      { "Name": "region",           "Type": "string" },
      { "Name": "categoria",        "Type": "string" },
      { "Name": "clientes_unicos",  "Type": "bigint" },
      { "Name": "total_ventas",     "Type": "bigint" },
      { "Name": "ingresos_totales", "Type": "double" },
      { "Name": "ticket_promedio",  "Type": "double" }
    ],
    "Location": "s3://lago-datos/gold/resumen/",
    "InputFormat": "org.apache.hadoop.mapred.TextInputFormat",
    "OutputFormat": "org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat",
    "SerdeInfo": { "SerializationLibrary": "org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe" }
  }
}
```

```bash
aws glue create-table --database-name tienda --table-input file://tabla-gold.json
```

4. Consulta la categoría líder sobre la tabla Gold:

```sql
SELECT categoria, SUM(ingresos_totales) AS ingresos
FROM resumen_gold
GROUP BY categoria
ORDER BY ingresos DESC
```

## Ejercicio 7: todo desde Python

Crea `athena.py`:

```python
import time
import boto3

CONFIG = dict(endpoint_url="http://localhost:4566", region_name="us-east-1",
              aws_access_key_id="test", aws_secret_access_key="test")
athena = boto3.client("athena", **CONFIG)


def sql(consulta):
    q = athena.start_query_execution(
        QueryString=consulta,
        QueryExecutionContext={"Database": "tienda"},
        ResultConfiguration={"OutputLocation": "s3://lago-datos/resultados/"},
    )["QueryExecutionId"]
    while True:
        estado = athena.get_query_execution(QueryExecutionId=q)["QueryExecution"]["Status"]
        if estado["State"] in ("SUCCEEDED", "FAILED", "CANCELLED"):
            break
        time.sleep(0.5)
    if estado["State"] != "SUCCEEDED":
        raise RuntimeError(estado.get("StateChangeReason"))
    filas = athena.get_query_results(QueryExecutionId=q)["ResultSet"]["Rows"]
    for fila in filas:
        print(" | ".join(c.get("VarCharValue", "") for c in fila["Data"]))


sql("""
    SELECT region, ROUND(SUM(monto_total), 2) AS ingresos
    FROM ventas_limpias
    GROUP BY region
    ORDER BY ingresos DESC
""")
```

```bash
python athena.py
```

> **Truco con pandas:** cada resultado queda en `s3://lago-datos/resultados/ID.csv`. Para tenerlo como tabla de pandas: `pd.read_csv(boto3.client("s3", **CONFIG).get_object(Bucket="lago-datos", Key=f"resultados/{q}.csv")["Body"])`.

---

## Reto

1. Genera un segundo archivo de ventas (cambia la semilla y los `id_orden` empiezan en 10001) y súbelo a `silver/ventas/`. Comprueba que `SELECT count(*)` ahora da 20 000 **sin tocar la tabla**.
2. Crea una tabla Gold `top_clientes` con el top 5 por región.
3. ¿Qué región tiene el ticket promedio más alto en `ELECTRONICA`?

## Chuleta

| Quiero... | Comando |
|---|---|
| Crear base de datos | `aws glue create-database --database-input Name=DB` |
| Crear tabla | `aws glue create-table --database-name DB --table-input file://tabla.json` |
| Ver tablas | `aws glue get-tables --database-name DB` |
| Borrar tabla | `aws glue delete-table --database-name DB --name T` |
| Lanzar consulta | `aws athena start-query-execution --query-string "..." --query-execution-context Database=DB --result-configuration OutputLocation=s3://B/resultados/` |
| Estado | `aws athena get-query-execution --query-execution-id ID` |
| Resultado | `aws athena get-query-results --query-execution-id ID` |

## Tipos de datos más comunes

| Glue | Para |
|---|---|
| `string` | Texto |
| `int` / `bigint` | Enteros (Parquet de pandas usa `bigint`) |
| `double` | Decimales |
| `boolean` | Verdadero / falso |
| `date` / `timestamp` | Fechas |
