# CloudWatch: logs, métricas y alarmas

[Volver al índice](README.md)

## Qué es y cuándo usarlo

**Amazon CloudWatch** es el sistema de **observabilidad** de AWS: te dice qué está pasando en tus aplicaciones.

| Parte | Para qué |
|---|---|
| **Logs** | Guardar y buscar los mensajes que escriben tus programas (Lambda los manda solos) |
| **Métricas** | Números a lo largo del tiempo: tiempo de respuesta, errores, pedidos por minuto |
| **Alarmas** | Avisar (por ejemplo, a SNS) cuando una métrica pasa un límite |

Sin CloudWatch, si algo falla en la nube no te enteras hasta que un usuario se queja.

## Conceptos clave

| Concepto | Qué es |
|---|---|
| **Grupo de logs** | Contenedor de logs de una app: `/tienda/app`, `/aws/lambda/mi-funcion` |
| **Stream de logs** | Secuencia de mensajes de una fuente (un servidor, una ejecución) |
| **Retención** | Cuántos días se guardan los logs |
| **Logs Insights** | Lenguaje de consultas para buscar en los logs |
| **Namespace** | Agrupación de métricas: `Tienda`, `AWS/Lambda` |
| **Dimensión** | Etiqueta de una métrica: `Servicio=api` |
| **Estadística** | `Average`, `Sum`, `Maximum`, `Minimum`, `SampleCount` |
| **Periodo** | Cada cuántos segundos se agrupan los datos |
| **Estados de alarma** | `OK`, `ALARM`, `INSUFFICIENT_DATA` (sin datos suficientes) |

## Antes de empezar

Terminal configurada ([manual 3](03-configurar-aws-cli.md)).

---

## Parte 1: Logs

### Ejercicio 1: grupo, stream y mensajes

```bash
aws logs create-log-group --log-group-name /tienda/api
```

```bash
aws logs put-retention-policy --log-group-name /tienda/api --retention-in-days 7
```

```bash
aws logs create-log-stream --log-group-name /tienda/api --log-stream-name servidor-1
```

Cada mensaje necesita una marca de tiempo en **milisegundos desde 1970**. Obtén la actual:

**Windows (PowerShell):**

```bash
[DateTimeOffset]::UtcNow.ToUnixTimeMilliseconds()
```

**macOS / Linux:**

```bash
echo $(($(date +%s)*1000))
```

Crea `logs.json` poniendo ese número (y +1000, +2000) en `timestamp`. Usamos mensajes en **JSON**, que es la buena práctica porque permite filtrar por campos:

```json
[
  { "timestamp": 1794013900000, "message": "{\"nivel\": \"INFO\", \"usuario\": \"ana\", \"accion\": \"login\"}" },
  { "timestamp": 1794013901000, "message": "{\"nivel\": \"ERROR\", \"usuario\": \"luis\", \"accion\": \"pago\"}" },
  { "timestamp": 1794013902000, "message": "{\"nivel\": \"ERROR\", \"usuario\": \"ana\", \"accion\": \"pago\"}" }
]
```

```bash
aws logs put-log-events --log-group-name /tienda/api --log-stream-name servidor-1 --log-events file://logs.json
```

> Los programas no hacen esto a mano: Lambda, ECS o la librería de logs de tu lenguaje lo envían solos.

### Ejercicio 2: leer y filtrar

Todos los mensajes de un stream:

```bash
aws logs get-log-events --log-group-name /tienda/api --log-stream-name servidor-1 --query "events[].message"
```

Los que contienen una palabra:

```bash
aws logs filter-log-events --log-group-name /tienda/api --filter-pattern ERROR --query "events[].message"
```

Filtrar por un **campo JSON**. Crea `patron.txt` con este contenido:

```text
{ $.nivel = "ERROR" }
```

```bash
aws logs filter-log-events --log-group-name /tienda/api --filter-pattern file://patron.txt --query "events[].message"
```

### Ejercicio 3: consultas con Logs Insights

Las consultas se lanzan y luego se recogen sus resultados. Necesitan un rango de tiempo en **segundos** desde 1970.

**Windows (PowerShell):**

```bash
$AHORA = [DateTimeOffset]::UtcNow.ToUnixTimeSeconds(); $Q = aws logs start-query --log-group-name /tienda/api --start-time ($AHORA - 3600) --end-time ($AHORA + 60) --query-string "fields @timestamp, usuario, accion | filter nivel = 'ERROR' | sort @timestamp desc | limit 20" --query queryId --output text
```

**macOS / Linux:**

```bash
Q=$(aws logs start-query --log-group-name /tienda/api --start-time $(($(date +%s)-3600)) --end-time $(($(date +%s)+60)) --query-string "fields @timestamp, usuario, accion | filter nivel = 'ERROR' | sort @timestamp desc | limit 20" --query queryId --output text)
```

```bash
aws logs get-query-results --query-id $Q
```

Devuelve los dos errores, de `ana` y `luis`, con sus campos.

> **Limitación del simulador:** Logs Insights admite `fields`, `filter campo = 'valor'` (también `!=`), `sort`, `dedup` y `limit`. No admite `like`, `stats` ni `parse`: si los usas, la consulta termina en `Failed`.

---

## Parte 2: Métricas

### Ejercicio 4: publicar métricas propias

Simulamos 4 tiempos de respuesta de la API (en milisegundos):

```bash
aws cloudwatch put-metric-data --namespace Tienda --metric-name TiempoRespuesta --unit Milliseconds --value 120 --dimensions Servicio=api
```

Repite con los valores `340`, `95` y `410`.

```bash
aws cloudwatch list-metrics --namespace Tienda
```

### Ejercicio 5: estadísticas

Necesitas un rango de fechas en formato ISO. Pon la fecha y hora actuales en UTC (por ejemplo, `2026-11-07T10:00:00Z` y `2026-11-07T11:00:00Z`):

```bash
aws cloudwatch get-metric-statistics --namespace Tienda --metric-name TiempoRespuesta --dimensions Name=Servicio,Value=api --start-time 2026-11-07T10:00:00Z --end-time 2026-11-07T11:00:00Z --period 3600 --statistics Average Maximum SampleCount
```

Verás el promedio (241,25), el máximo (410) y cuántos datos hay (4).

> Las métricas de los servicios de AWS (Lambda, SQS, DynamoDB...) se consultan igual, con namespaces como `AWS/Lambda`.

---

## Parte 3: Alarmas

### Ejercicio 6: una alarma que avisa por SNS

Queremos un aviso cuando el tiempo de respuesta medio pase de **300 ms**.

Crea el tema y una cola "buzón" para ver los avisos:

**Windows (PowerShell):**

```bash
$TEMA = aws sns create-topic --name alertas --query TopicArn --output text
```

**macOS / Linux:**

```bash
TEMA=$(aws sns create-topic --name alertas --query TopicArn --output text)
```

```bash
aws sqs create-queue --queue-name buzon-alertas
```

```bash
aws sns subscribe --topic-arn $TEMA --protocol sqs --notification-endpoint arn:aws:sqs:us-east-1:000000000000:buzon-alertas
```

Crea la alarma (evalúa el promedio cada 60 segundos):

```bash
aws cloudwatch put-metric-alarm --alarm-name api-lenta --namespace Tienda --metric-name TiempoRespuesta --dimensions Name=Servicio,Value=api --statistic Average --period 60 --evaluation-periods 1 --threshold 300 --comparison-operator GreaterThanThreshold --alarm-actions $TEMA
```

### Ejercicio 7: hacer saltar la alarma

Publica tiempos lentos (`520`, `610` y `480`) con `put-metric-data` como en el ejercicio 4. Espera algo más de un minuto y consulta:

```bash
aws cloudwatch describe-alarms --alarm-names api-lenta --query "MetricAlarms[0].[StateValue,StateReason]"
```

Responde `ALARM` con `Threshold Crossed`. Mira el aviso que llegó por SNS:

```bash
aws sqs receive-message --queue-url http://localhost:4566/000000000000/buzon-alertas
```

El mensaje trae `"Subject": "ALARM: \"api-lenta\" ..."`. Si dejas de enviar datos lentos, al siguiente minuto vuelve a `OK`.

Historial de cambios de estado:

```bash
aws cloudwatch describe-alarm-history --alarm-name api-lenta --query "AlarmHistoryItems[].HistorySummary"
```

Para probar las acciones sin esperar, puedes forzar el estado:

```bash
aws cloudwatch set-alarm-state --alarm-name api-lenta --state-value ALARM --state-reason "Prueba manual"
```

### Ejercicio 8: de logs a métricas

Un **filtro de métrica** cuenta automáticamente las líneas de log que cumplen un patrón y lo convierte en métrica (y por tanto, en alarma):

```bash
aws logs put-metric-filter --log-group-name /tienda/api --filter-name errores --filter-pattern ERROR --metric-transformations metricName=Errores,metricNamespace=Tienda,metricValue=1
```

Cada log nuevo con `ERROR` suma 1 a la métrica `Tienda/Errores`. Ahora podrías crear una alarma "más de 5 errores por minuto".

---

## Reto

1. Escribe un script en Python que cada segundo publique un tiempo de respuesta aleatorio entre 50 y 600 ms.
2. Crea una alarma que salte si el **máximo** de un minuto pasa de 550.
3. Haz que la alarma avise a SNS **también cuando vuelva a la normalidad** (pista: `--ok-actions`).
4. Envía logs JSON desde el mismo script con el nivel `ERROR` cuando el tiempo pase de 500, y búscalos con Logs Insights.

## Chuleta

| Quiero... | Comando |
|---|---|
| Grupo de logs | `aws logs create-log-group --log-group-name G` |
| Enviar logs | `aws logs put-log-events --log-group-name G --log-stream-name S --log-events file://logs.json` |
| Buscar | `aws logs filter-log-events --log-group-name G --filter-pattern TEXTO` |
| Insights | `aws logs start-query ...` + `aws logs get-query-results --query-id Q` |
| Publicar métrica | `aws cloudwatch put-metric-data --namespace N --metric-name M --value V` |
| Estadísticas | `aws cloudwatch get-metric-statistics ...` |
| Alarma | `aws cloudwatch put-metric-alarm ... --alarm-actions ARN_SNS` |
| Estado alarmas | `aws cloudwatch describe-alarms` |
