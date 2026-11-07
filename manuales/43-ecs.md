# ECS: ejecutar contenedores

[Volver al índice](README.md)

## Qué es y cuándo usarlo

**Amazon ECS** (*Elastic Container Service*) ejecuta **contenedores Docker** en la nube y los mantiene funcionando: si uno se cae, arranca otro; si necesitas más, los multiplica.

En AWS Nube Local, cada tarea es un **contenedor real** en tu Docker Desktop.

Úsalo cuando tu aplicación ya está (o puede estar) en una imagen Docker: webs, APIs, procesos largos que no encajan en Lambda (que tiene límite de tiempo).

## Conceptos clave

| Concepto | Qué es |
|---|---|
| **Cluster** | Agrupación lógica donde corren tus contenedores |
| **Definición de tarea** (*task definition*) | La "receta": imagen, memoria, puertos, variables de entorno |
| **Tarea** (*task*) | Una ejecución de esa receta: uno o varios contenedores en marcha |
| **Servicio** (*service*) | Mantiene **N tareas** funcionando siempre, y las reemplaza si fallan |
| **Revisión** | Cada vez que registras una definición con el mismo nombre se crea `:1`, `:2`... |
| **Mapeo de puertos** | Conecta un puerto del contenedor con uno de tu PC |

## Antes de empezar

- Terminal configurada ([manual 3](03-configurar-aws-cli.md)).
- Ten Docker Desktop a la vista: verás aparecer y desaparecer los contenedores.

---

## Ejercicio 1: crear un cluster

```bash
aws ecs create-cluster --cluster-name clase
```

```bash
aws ecs list-clusters
```

## Ejercicio 2: definir una tarea (un servidor web nginx)

Crea `tarea-web.json`:

```json
{
  "family": "web-nginx",
  "networkMode": "bridge",
  "containerDefinitions": [
    {
      "name": "nginx",
      "image": "nginx:alpine",
      "memory": 128,
      "essential": true,
      "portMappings": [ { "containerPort": 80, "hostPort": 8085 } ],
      "environment": [ { "name": "ENTORNO", "value": "clase" } ]
    }
  ]
}
```

- `image`: cualquier imagen de Docker Hub o de [ECR](#ejercicio-6-tu-propia-imagen).
- `portMappings`: el puerto 80 del contenedor se abrirá en el **8085** de tu PC.

```bash
aws ecs register-task-definition --cli-input-json file://tarea-web.json
```

```bash
aws ecs list-task-definitions
```

## Ejercicio 3: ejecutar una tarea

```bash
aws ecs run-task --cluster clase --task-definition web-nginx --count 1
```

Comprueba el estado (repite hasta que diga `RUNNING`; la primera vez descarga nginx):

```bash
aws ecs describe-tasks --cluster clase --tasks $(aws ecs list-tasks --cluster clase --query "taskArns[0]" --output text) --query "tasks[0].[lastStatus,containers[0].networkBindings]"
```

> En PowerShell funciona igual: `$(...)` también ejecuta el comando interior.

Abre en el navegador: <http://localhost:8085>. Verás **Welcome to nginx!**.

En Docker Desktop aparece un contenedor `floci-aws-ecs-...-nginx`.

Detener la tarea:

```bash
aws ecs stop-task --cluster clase --task ARN_DE_LA_TAREA
```

## Ejercicio 4: un servicio que mantiene contenedores vivos

Para un servicio con varias copias, el puerto del PC no puede ser fijo (chocarían). Crea `tarea-api.json` con `"hostPort": 0`:

> En AWS Nube Local, los contenedores de un servicio con `hostPort` 0 **no se publican en tu PC**: no podrás abrirlos en el navegador. Este ejercicio sirve para practicar copias, escalado y autorreparación. Para ver una web en el navegador, usa una tarea con puerto fijo como en el ejercicio 3.


```json
{
  "family": "api-nginx",
  "networkMode": "bridge",
  "containerDefinitions": [
    {
      "name": "nginx",
      "image": "nginx:alpine",
      "memory": 128,
      "essential": true,
      "portMappings": [ { "containerPort": 80, "hostPort": 0 } ]
    }
  ]
}
```

```bash
aws ecs register-task-definition --cli-input-json file://tarea-api.json
```

Crea un servicio con **2 copias**:

```bash
aws ecs create-service --cluster clase --service-name api --task-definition api-nginx --desired-count 2
```

Espera unos segundos:

```bash
aws ecs describe-services --cluster clase --services api --query "services[0].[desiredCount,runningCount]"
```

Responde `[2, 2]`: dos deseadas, dos funcionando.

**Experimento:** en Docker Desktop, detén a mano uno de los dos contenedores. Al poco tiempo, ECS arranca otro para volver a 2. Eso es la *autorreparación*.

**Escalar** a 1 copia:

```bash
aws ecs update-service --cluster clase --service api --desired-count 1
```

## Ejercicio 5: ver las tareas de un servicio y sus eventos

Tareas que forman el servicio:

```bash
aws ecs list-tasks --cluster clase --service-name api
```

Historial del servicio (arranques, reemplazos, "steady state" cuando todo está estable):

```bash
aws ecs describe-services --cluster clase --services api --query "services[0].events[:5].message"
```

## Ejercicio 6: tu propia imagen

Con [ECR](https://docs.aws.amazon.com/es_es/AmazonECR/latest/userguide/what-is-ecr.html) (el registro de imágenes de AWS) puedes subir tus propias imágenes:

```bash
aws ecr create-repository --repository-name mi-app
```

La respuesta incluye un `repositoryUri`. Construye tu imagen con Docker, etiquétala con esa URI y haz `docker push`. Después usa esa URI como `image` en la definición de tarea.

## Borrar

Un servicio debe bajar a 0 antes de borrarse:

```bash
aws ecs update-service --cluster clase --service api --desired-count 0
```

```bash
aws ecs delete-service --cluster clase --service api
```

```bash
aws ecs delete-cluster --cluster clase
```

---

## Reto

1. Crea una definición `web-propia` con la imagen `httpd:alpine` (Apache) en el puerto 8090.
2. Añade dos variables de entorno y comprueba en Docker Desktop (pestaña *Inspect* del contenedor) que llegaron.
3. Crea un servicio con 3 copias de `api-nginx`, detén dos contenedores a mano y comprueba que ECS los repone.
4. Registra una segunda revisión de `api-nginx` con más memoria y actualiza el servicio a ella (`--task-definition api-nginx:2`).

## Chuleta

| Quiero... | Comando |
|---|---|
| Crear cluster | `aws ecs create-cluster --cluster-name C` |
| Registrar receta | `aws ecs register-task-definition --cli-input-json file://tarea.json` |
| Ejecutar tarea | `aws ecs run-task --cluster C --task-definition T --count 1` |
| Ver tareas | `aws ecs list-tasks --cluster C` |
| Detalle | `aws ecs describe-tasks --cluster C --tasks ARN` |
| Parar | `aws ecs stop-task --cluster C --task ARN` |
| Servicio | `aws ecs create-service --cluster C --service-name S --task-definition T --desired-count N` |
| Escalar | `aws ecs update-service --cluster C --service S --desired-count N` |
