# ElastiCache: caché en memoria con Redis

[Volver al índice](README.md)

## Qué es y cuándo usarlo

**Amazon ElastiCache** te da un servidor **Redis** (o Valkey, su versión libre compatible) administrado. Redis guarda datos **en memoria**, así que responde en microsegundos.

En AWS Nube Local es un **Valkey real** en un contenedor Docker, compatible con cualquier cliente de Redis.

Úsalo para:

- **Caché**: guardar resultados de consultas lentas para no repetirlas.
- **Sesiones** de usuarios en una web.
- **Contadores** (visitas, likes) y **rankings** en tiempo real.
- Datos que deben **caducar** solos (códigos de verificación, límites de uso).

## Conceptos clave

| Concepto | Qué es |
|---|---|
| **Cluster** | El servidor Redis que creas |
| **Nodo** | Cada máquina del cluster (aquí usaremos 1) |
| **Clave / valor** | Redis guarda pares `nombre -> dato` |
| **TTL** | Segundos que le quedan a una clave antes de borrarse sola |
| **Tipos** | Texto, contadores, listas, conjuntos, hashes (objetos), conjuntos ordenados |

## Antes de empezar

Terminal configurada ([manual 3](03-configurar-aws-cli.md)). La primera vez se descarga la imagen de Valkey.

---

## Ejercicio 1: crear el cluster

```bash
aws elasticache create-cache-cluster --cache-cluster-id cache-clase --engine redis --cache-node-type cache.t3.micro --num-cache-nodes 1
```

Espera a que esté `available`:

```bash
aws elasticache describe-cache-clusters --cache-cluster-id cache-clase --show-cache-node-info --query "CacheClusters[0].[CacheClusterStatus,CacheNodes[0].Endpoint.Port]" --output text
```

Desde tu PC se conecta en **`localhost`** y el puerto que muestre (el primero es **6379**).

## Ejercicio 2: conectarse

**Con un cliente gráfico:** [Redis Insight](https://redis.io/insight/) (gratuito). Host `localhost`, puerto `6379`.

**Con la línea de comandos desde Docker:**

```bash
docker run --rm -it valkey/valkey:8 valkey-cli -h host.docker.internal -p 6379
```

Si responde con `host.docker.internal:6379>` estás dentro. Escribe `PING`: debe contestar `PONG`.

## Ejercicio 3: claves, contadores y caducidad

Dentro de `valkey-cli`:

```text
SET saludo "hola clase"
GET saludo

SET visitas 0
INCR visitas
INCR visitas
GET visitas

SET codigo 4821 EX 60
TTL codigo
```

- `INCR` suma 1 de forma segura aunque muchos usuarios lo hagan a la vez.
- `EX 60` hace que `codigo` se borre a los 60 segundos. `TTL` dice cuánto le queda.

## Ejercicio 4: objetos (hashes) y listas

Un usuario como objeto:

```text
HSET usuario:1 nombre Ana ciudad Bogota rol admin
HGET usuario:1 nombre
HGETALL usuario:1
```

Una lista de tareas (cola sencilla):

```text
RPUSH tareas "enviar correo" "generar informe" "limpiar logs"
LRANGE tareas 0 -1
LPOP tareas
```

## Ejercicio 5: ranking en tiempo real

Los **conjuntos ordenados** son perfectos para marcadores:

```text
ZADD ranking 120 ana 95 luis 150 marta
ZINCRBY ranking 40 luis
ZREVRANGE ranking 0 2 WITHSCORES
```

Muestra el top 3 de mayor a menor puntuación.

## Ejercicio 6: patrón caché desde Python

Instala el cliente: `pip install redis`. Crea `cache.py`:

```python
import time
import redis

r = redis.Redis(host="localhost", port=6379, decode_responses=True)


def consulta_lenta(ciudad):
    time.sleep(2)  # simula una consulta a base de datos que tarda
    return f"Clima soleado en {ciudad}"


def obtener_clima(ciudad):
    clave = f"clima:{ciudad}"
    guardado = r.get(clave)
    if guardado:
        return guardado + " (desde caché)"
    resultado = consulta_lenta(ciudad)
    r.set(clave, resultado, ex=30)  # se guarda 30 segundos
    return resultado


for _ in range(3):
    inicio = time.time()
    print(obtener_clima("Bogota"), f"{time.time() - inicio:.2f}s")
```

```bash
python cache.py
```

La primera vez tarda 2 segundos; las siguientes, casi nada. Ese es el patrón *cache-aside*, el más usado.

## Borrar

```bash
aws elasticache delete-cache-cluster --cache-cluster-id cache-clase
```

---

## Reto

1. Guarda la sesión de un usuario como hash `sesion:abc123` con `usuario` y `rol`, que caduque en 15 minutos (`EXPIRE sesion:abc123 900`).
2. Implementa un límite de 5 intentos de login por minuto: usa `INCR intentos:ana` y `EXPIRE intentos:ana 60`, y rechaza cuando pase de 5.
3. Crea un ranking de 5 jugadores y muestra solo el primero.

## Chuleta

| Quiero... | Comando |
|---|---|
| Crear cluster | `aws elasticache create-cache-cluster --cache-cluster-id X --engine redis --cache-node-type cache.t3.micro --num-cache-nodes 1` |
| Ver estado y puerto | `aws elasticache describe-cache-clusters --show-cache-node-info` |
| Conectar | `docker run --rm -it valkey/valkey:8 valkey-cli -h host.docker.internal -p 6379` |
| Borrar | `aws elasticache delete-cache-cluster --cache-cluster-id X` |

| Redis | Para qué |
|---|---|
| `SET k v EX 60` / `GET k` | Guardar con caducidad / leer |
| `INCR k` | Contador |
| `HSET` / `HGETALL` | Objetos |
| `RPUSH` / `LPOP` | Listas y colas |
| `ZADD` / `ZREVRANGE` | Rankings |
| `TTL k` / `DEL k` | Tiempo restante / borrar |
