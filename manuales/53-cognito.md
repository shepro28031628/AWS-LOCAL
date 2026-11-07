# Cognito: registro e inicio de sesión de usuarios

[Volver al índice](README.md)

## Qué es y cuándo usarlo

**Amazon Cognito** gestiona los **usuarios de tu aplicación**: registro, confirmación, inicio de sesión, contraseñas, grupos y tokens. No tienes que programar ni guardar contraseñas tú.

No confundir con IAM:

| IAM | Cognito |
|---|---|
| Para **quien administra** AWS (tú, tu equipo, tus servicios) | Para **los usuarios de tu app** (clientes, alumnos...) |
| Decenas de usuarios | Miles o millones |

Al iniciar sesión, Cognito entrega **tokens JWT**. Tu app (o [API Gateway](41-api-gateway.md)) los comprueba para saber quién es el usuario.

## Conceptos clave

| Concepto | Qué es |
|---|---|
| **User pool** | El "directorio" de usuarios de tu aplicación |
| **App client** | Una aplicación (web, móvil) autorizada a usar el pool |
| **Atributos** | Datos del usuario: `email`, `name`, `phone_number`, personalizados... |
| **Política de contraseñas** | Longitud mínima, números, mayúsculas... |
| **Estado del usuario** | `UNCONFIRMED`, `CONFIRMED`, `FORCE_CHANGE_PASSWORD`... |
| **Tokens** | `IdToken` (quién es), `AccessToken` (qué puede hacer), `RefreshToken` (renovar sin contraseña) |
| **Grupos** | Roles dentro de tu app: `profesores`, `alumnos`, `admins` |

## Antes de empezar

Terminal configurada ([manual 3](03-configurar-aws-cli.md)).

---

## Ejercicio 1: crear el user pool

Una política de contraseñas de 8 caracteres con números y minúsculas:

**Windows (PowerShell):**

```bash
$POOL = aws cognito-idp create-user-pool --pool-name usuarios-app --auto-verified-attributes email --policies "PasswordPolicy={MinimumLength=8,RequireNumbers=true,RequireUppercase=false,RequireLowercase=true,RequireSymbols=false}" --query UserPool.Id --output text
```

**macOS / Linux:**

```bash
POOL=$(aws cognito-idp create-user-pool --pool-name usuarios-app --auto-verified-attributes email --policies "PasswordPolicy={MinimumLength=8,RequireNumbers=true,RequireUppercase=false,RequireLowercase=true,RequireSymbols=false}" --query UserPool.Id --output text)
```

## Ejercicio 2: crear el cliente de la aplicación

Permitimos inicio de sesión con usuario y contraseña, y renovar tokens:

**Windows (PowerShell):**

```bash
$CLIENTE = aws cognito-idp create-user-pool-client --user-pool-id $POOL --client-name web --explicit-auth-flows ALLOW_USER_PASSWORD_AUTH ALLOW_REFRESH_TOKEN_AUTH --query UserPoolClient.ClientId --output text
```

**macOS / Linux:**

```bash
CLIENTE=$(aws cognito-idp create-user-pool-client --user-pool-id $POOL --client-name web --explicit-auth-flows ALLOW_USER_PASSWORD_AUTH ALLOW_REFRESH_TOKEN_AUTH --query UserPoolClient.ClientId --output text)
```

## Ejercicio 3: registro de usuarios

Así se registra un usuario desde tu app:

```bash
aws cognito-idp sign-up --client-id $CLIENTE --username ana@clase.com --password clave1234 --user-attributes Name=email,Value=ana@clase.com Name=name,Value=Ana
```

Responde `"UserConfirmed": false`: falta confirmar el correo. Prueba con una contraseña que no cumpla la política:

```bash
aws cognito-idp sign-up --client-id $CLIENTE --username luis@clase.com --password corta --user-attributes Name=email,Value=luis@clase.com
```

Error `InvalidPasswordException`.

En AWS real, Cognito envía un código al correo y el usuario lo introduce con `confirm-sign-up`. Aquí lo confirmamos como administrador:

```bash
aws cognito-idp admin-confirm-sign-up --user-pool-id $POOL --username ana@clase.com
```

```bash
aws cognito-idp admin-get-user --user-pool-id $POOL --username ana@clase.com --query "[UserStatus,UserAttributes]"
```

## Ejercicio 4: iniciar sesión y obtener tokens

```bash
aws cognito-idp initiate-auth --client-id $CLIENTE --auth-flow USER_PASSWORD_AUTH --auth-parameters USERNAME=ana@clase.com,PASSWORD=clave1234
```

En `AuthenticationResult` llegan `AccessToken`, `IdToken`, `RefreshToken` y `ExpiresIn` (segundos de validez).

Con la contraseña mal:

```bash
aws cognito-idp initiate-auth --client-id $CLIENTE --auth-flow USER_PASSWORD_AUTH --auth-parameters USERNAME=ana@clase.com,PASSWORD=mala1234
```

Error `NotAuthorizedException: Incorrect username or password`.

**Mira dentro de un token:** copia el `IdToken` y pégalo en <https://jwt.io>. Verás el email, el nombre, el emisor (`iss`) y la caducidad (`exp`). Los tokens están **firmados**, no cifrados: cualquiera puede leerlos, pero nadie puede modificarlos sin que se note.

## Ejercicio 5: usar el token

Guarda el token de acceso:

**Windows (PowerShell):**

```bash
$TOKEN = aws cognito-idp initiate-auth --client-id $CLIENTE --auth-flow USER_PASSWORD_AUTH --auth-parameters USERNAME=ana@clase.com,PASSWORD=clave1234 --query AuthenticationResult.AccessToken --output text
```

**macOS / Linux:**

```bash
TOKEN=$(aws cognito-idp initiate-auth --client-id $CLIENTE --auth-flow USER_PASSWORD_AUTH --auth-parameters USERNAME=ana@clase.com,PASSWORD=clave1234 --query AuthenticationResult.AccessToken --output text)
```

Pide los datos del usuario **con el token** (sin credenciales de AWS de por medio):

```bash
aws cognito-idp get-user --access-token $TOKEN
```

Las claves públicas para verificar firmas están en (sustituye el id del pool):

```text
http://localhost:4566/ID_DEL_POOL/.well-known/jwks.json
```

## Ejercicio 6: grupos y usuarios creados por el administrador

```bash
aws cognito-idp create-group --user-pool-id $POOL --group-name profesores
```

```bash
aws cognito-idp admin-add-user-to-group --user-pool-id $POOL --username ana@clase.com --group-name profesores
```

Si inicias sesión de nuevo, el `IdToken` incluye `"cognito:groups": ["profesores"]`. Tu app puede usarlo para mostrar u ocultar opciones.

Crear un usuario como administrador, con contraseña temporal:

```bash
aws cognito-idp admin-create-user --user-pool-id $POOL --username marta@clase.com --temporary-password Temporal123 --user-attributes Name=email,Value=marta@clase.com
```

```bash
aws cognito-idp list-users --user-pool-id $POOL --query "Users[].[Username,UserStatus]" --output table
```

Marta queda en `FORCE_CHANGE_PASSWORD`: al entrar por primera vez deberá cambiarla.

## Ejercicio 7: login desde Python

```python
import boto3

CONFIG = dict(endpoint_url="http://localhost:4566", region_name="us-east-1",
              aws_access_key_id="test", aws_secret_access_key="test")
cognito = boto3.client("cognito-idp", **CONFIG)

CLIENTE = "PEGA_AQUÍ_EL_CLIENT_ID"

try:
    r = cognito.initiate_auth(
        ClientId=CLIENTE,
        AuthFlow="USER_PASSWORD_AUTH",
        AuthParameters={"USERNAME": "ana@clase.com", "PASSWORD": "clave1234"},
    )
    token = r["AuthenticationResult"]["AccessToken"]
    usuario = cognito.get_user(AccessToken=token)
    atributos = {a["Name"]: a["Value"] for a in usuario["UserAttributes"]}
    print("Bienvenida,", atributos.get("name"))
except cognito.exceptions.NotAuthorizedException:
    print("Usuario o contraseña incorrectos")
```

---

## Reto

1. Crea un pool `campus` con política de contraseñas de 10 caracteres que exija mayúsculas.
2. Crea los grupos `alumnos` y `profesores`.
3. Registra 3 usuarios por `sign-up`, confírmalos y repártelos en los grupos.
4. Escribe un script de login que muestre "Panel de profesor" o "Panel de alumno" según el grupo que venga en el `IdToken` (pista: decodifica la parte central del JWT con `base64`).

## Chuleta

| Quiero... | Comando |
|---|---|
| Crear pool | `aws cognito-idp create-user-pool --pool-name N` |
| Cliente | `aws cognito-idp create-user-pool-client --user-pool-id P --client-name C --explicit-auth-flows ALLOW_USER_PASSWORD_AUTH ALLOW_REFRESH_TOKEN_AUTH` |
| Registrar | `aws cognito-idp sign-up --client-id C --username U --password X` |
| Confirmar | `aws cognito-idp admin-confirm-sign-up --user-pool-id P --username U` |
| Login | `aws cognito-idp initiate-auth --client-id C --auth-flow USER_PASSWORD_AUTH --auth-parameters USERNAME=U,PASSWORD=X` |
| Datos con token | `aws cognito-idp get-user --access-token T` |
| Grupos | `create-group` / `admin-add-user-to-group` |
| Listar | `aws cognito-idp list-users --user-pool-id P` |
