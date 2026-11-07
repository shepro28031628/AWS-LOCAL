# IAM y STS: usuarios, permisos y roles

[Volver al índice](README.md)

## Qué es y cuándo usarlo

**AWS IAM** (*Identity and Access Management*) decide **quién** puede hacer **qué** sobre **qué recursos**. Es la base de la seguridad en AWS: todo lo que se hace en la nube pasa por IAM.

**AWS STS** (*Security Token Service*) entrega **credenciales temporales**, por ejemplo cuando alguien "asume un rol".

Úsalo siempre: cada persona y cada aplicación debe tener **solo los permisos que necesita** (principio de mínimo privilegio).

## Conceptos clave

| Concepto | Qué es |
|---|---|
| **Usuario** | Una persona o aplicación con credenciales permanentes |
| **Grupo** | Conjunto de usuarios que comparten permisos (`analistas`, `desarrolladores`) |
| **Política** | Documento JSON con permisos: `Effect` (Allow/Deny), `Action`, `Resource` |
| **Rol** | Identidad **sin** credenciales fijas que alguien o algo "se pone" temporalmente |
| **Política de confianza** | Dice **quién** puede asumir un rol |
| **Access key** | Par de claves (`AccessKeyId` + `SecretAccessKey`) para usar la CLI o el SDK |
| **ARN** | Identificador único de un recurso: `arn:aws:iam::000000000000:user/ana` |
| **Denegación implícita** | Lo que no está permitido expresamente, está prohibido |

Estructura de una política:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:ListBucket"],
      "Resource": ["arn:aws:s3:::docs-clase", "arn:aws:s3:::docs-clase/*"]
    }
  ]
}
```

## Importante: cómo funciona IAM en el simulador

Por defecto, AWS Nube Local **acepta cualquier credencial y no aplica las políticas**: puedes crear usuarios, grupos y políticas, pero no te bloquean. Así es más fácil practicar con los demás servicios.

Para que las políticas **se cumplan de verdad**, activa el modo estricto (ejercicio 6).

## Antes de empezar

Terminal configurada ([manual 3](03-configurar-aws-cli.md)). Crea el bucket que usaremos:

```bash
aws s3 mb s3://docs-clase
```

---

## Ejercicio 1: una política de solo lectura

Crea `politica-lectura.json` con el contenido de arriba (lectura del bucket `docs-clase`).

```bash
aws iam create-policy --policy-name LecturaDocs --policy-document file://politica-lectura.json
```

El ARN será `arn:aws:iam::000000000000:policy/LecturaDocs`.

## Ejercicio 2: grupo y usuario

```bash
aws iam create-group --group-name analistas
```

```bash
aws iam attach-group-policy --group-name analistas --policy-arn arn:aws:iam::000000000000:policy/LecturaDocs
```

```bash
aws iam create-user --user-name ana
```

```bash
aws iam add-user-to-group --group-name analistas --user-name ana
```

Comprueba:

```bash
aws iam get-group --group-name analistas --query "Users[].UserName"
```

```bash
aws iam list-attached-group-policies --group-name analistas
```

> **Buena práctica:** da permisos a **grupos**, no a usuarios sueltos. Si Ana cambia de equipo, solo la mueves de grupo.

## Ejercicio 3: credenciales para Ana

```bash
aws iam create-access-key --user-name ana
```

Guarda `AccessKeyId` y `SecretAccessKey`: el secreto solo se muestra **una vez**.

```bash
aws iam list-access-keys --user-name ana
```

## Ejercicio 4: comprobar permisos con el simulador de políticas

Sin activar nada, puedes preguntar a IAM "¿podría Ana hacer esto?":

```bash
aws iam simulate-principal-policy --policy-source-arn arn:aws:iam::000000000000:user/ana --action-names s3:GetObject s3:DeleteObject --resource-arns arn:aws:s3:::docs-clase/informe.txt --query "EvaluationResults[].[EvalActionName,EvalDecision]" --output table
```

| Acción | Decisión |
|---|---|
| `s3:GetObject` | `allowed` |
| `s3:DeleteObject` | `implicitDeny` (no está permitido en ninguna política) |

## Ejercicio 5: roles y credenciales temporales

Un **auditor externo** necesita acceso de solo lectura durante una hora. En vez de crearle un usuario, creamos un rol.

`confianza.json` (quién puede asumir el rol; aquí, cualquiera de nuestra cuenta):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "AWS": "arn:aws:iam::000000000000:root" },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

```bash
aws iam create-role --role-name auditor --assume-role-policy-document file://confianza.json
```

Dale la política **gestionada por AWS** de solo lectura:

```bash
aws iam attach-role-policy --role-name auditor --policy-arn arn:aws:iam::aws:policy/ReadOnlyAccess
```

Asume el rol:

```bash
aws sts assume-role --role-arn arn:aws:iam::000000000000:role/auditor --role-session-name sesion-auditoria
```

Recibes un `AccessKeyId` (empieza por `ASIA`, las temporales), un `SecretAccessKey`, un `SessionToken` y una fecha de **caducidad**. Pasado ese tiempo, dejan de valer.

Para saber "quién soy" con unas credenciales:

```bash
aws sts get-caller-identity
```

> Los roles son lo que usan los servicios entre sí: una Lambda **asume** su rol para leer de S3, un EC2 **asume** el suyo, etc. Por eso al crear Lambdas pasamos `--role`.

## Ejercicio 6 (opcional): que las políticas se cumplan de verdad

1. Abre `docker-compose.yml`, y en `environment:` del servicio `floci` añade:

   ```yaml
   FLOCI_SERVICES_IAM_ENFORCEMENT_ENABLED: "true"
   ```

2. Reinicia (los datos en memoria se borran):

   ```bash
   docker compose up -d
   ```

3. Repite los ejercicios 1, 2 y 3 (con tus credenciales `test`, que actúan como administrador) y sube un archivo:

   ```bash
   aws s3 cp politica-lectura.json s3://docs-clase/informe.txt
   ```

4. Ahora **actúa como Ana** con sus claves:

   **Windows (PowerShell):**

   ```bash
   $env:AWS_ACCESS_KEY_ID="CLAVE_DE_ANA"; $env:AWS_SECRET_ACCESS_KEY="SECRETO_DE_ANA"
   ```

   **macOS / Linux:**

   ```bash
   export AWS_ACCESS_KEY_ID=CLAVE_DE_ANA AWS_SECRET_ACCESS_KEY=SECRETO_DE_ANA
   ```

5. Prueba:

   ```bash
   aws sts get-caller-identity
   ```

   ```bash
   aws s3 ls s3://docs-clase/
   ```

   ```bash
   aws s3 rm s3://docs-clase/informe.txt
   ```

   ```bash
   aws s3 mb s3://otro-bucket
   ```

| Comando | Resultado |
|---|---|
| `get-caller-identity` | `arn:aws:iam::000000000000:user/ana` |
| `s3 ls` | Funciona |
| `s3 rm` | `AccessDenied ... not authorized to perform: s3:DeleteObject` |
| `s3 mb` | `AccessDenied ... not authorized to perform: s3:CreateBucket` |

6. Vuelve a ser administrador poniendo otra vez `test` / `test` en las variables. Cuando termines, quita la línea del `docker-compose.yml` y ejecuta `docker compose up -d`.

---

## Reto

1. Crea un grupo `desarrolladores` con una política que permita **todo** en DynamoDB (`dynamodb:*`) pero **solo** sobre la tabla `tareas` (`arn:aws:dynamodb:us-east-1:000000000000:table/tareas`).
2. Añade una segunda instrucción con `"Effect": "Deny"` para `dynamodb:DeleteTable` (un `Deny` siempre gana a un `Allow`).
3. Comprueba con `simulate-principal-policy` que puede hacer `PutItem` pero no `DeleteTable`.
4. Con el modo estricto activado, verifícalo con las claves de un usuario de ese grupo.

## Chuleta

| Quiero... | Comando |
|---|---|
| Crear política | `aws iam create-policy --policy-name P --policy-document file://p.json` |
| Grupo | `aws iam create-group --group-name G` |
| Política a grupo | `aws iam attach-group-policy --group-name G --policy-arn ARN` |
| Usuario | `aws iam create-user --user-name U` |
| Usuario a grupo | `aws iam add-user-to-group --group-name G --user-name U` |
| Claves | `aws iam create-access-key --user-name U` |
| Rol | `aws iam create-role --role-name R --assume-role-policy-document file://confianza.json` |
| Asumir rol | `aws sts assume-role --role-arn ARN --role-session-name S` |
| ¿Quién soy? | `aws sts get-caller-identity` |
| ¿Puede hacerlo? | `aws iam simulate-principal-policy --policy-source-arn ARN --action-names ...` |
