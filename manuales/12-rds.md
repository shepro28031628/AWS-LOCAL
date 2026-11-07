# RDS: bases de datos relacionales

[Volver al índice](README.md)

## Qué es y cuándo usarlo

**Amazon RDS** (*Relational Database Service*) te da bases de datos SQL de siempre (PostgreSQL, MySQL, MariaDB...) sin tener que instalarlas ni mantenerlas. AWS se encarga de copias de seguridad, actualizaciones y réplicas.

En AWS Nube Local, cada base de datos es **un PostgreSQL o MySQL real** dentro de un contenedor Docker. Puedes conectarte con cualquier programa: `psql`, DBeaver, pgAdmin, MySQL Workbench o desde tu código.

Úsalo cuando tus datos tengan **relaciones** (clientes, pedidos, productos) y necesites SQL, JOIN y transacciones.

## Conceptos clave

| Concepto | Qué es |
|---|---|
| **Instancia** (*DB instance*) | Un servidor de base de datos |
| **Motor** (*engine*) | `postgres`, `mysql` o `mariadb` |
| **Clase** (*instance class*) | Tamaño del servidor (`db.t3.micro`...). En local no afecta |
| **Usuario maestro** | El administrador que se crea con la instancia |
| **Endpoint** | Dirección y puerto para conectarse |
| **Snapshot** | Copia de seguridad completa en un momento dado |

## Antes de empezar

- Terminal configurada ([manual 3](03-configurar-aws-cli.md)).
- La primera vez se descarga la imagen de PostgreSQL (unos 100 MB).
- Para conectarte necesitas un cliente SQL. Si no tienes ninguno, usa el de Docker (ver ejercicio 2) o instala [DBeaver](https://dbeaver.io/), que es gratuito.

---

## Ejercicio 1: crear una base de datos PostgreSQL

```bash
aws rds create-db-instance --db-instance-identifier tienda-db --engine postgres --db-instance-class db.t3.micro --master-username admin --master-user-password Clase2026! --db-name tienda --allocated-storage 20
```

Espera unos segundos y comprueba el estado hasta que diga `available`:

```bash
aws rds describe-db-instances --db-instance-identifier tienda-db --query "DBInstances[0].[DBInstanceStatus,Endpoint.Port]" --output text
```

El número que aparece es el **puerto**: la primera base de datos usa el **7001**, la segunda el 7002, etc.

> El campo `Endpoint.Address` muestra una IP interna de Docker. **Desde tu PC conéctate siempre a `localhost` y ese puerto.**

## Ejercicio 2: conectarse y usar SQL

Datos de conexión:

| Campo | Valor |
|---|---|
| Host | `localhost` |
| Puerto | `7001` |
| Base de datos | `tienda` |
| Usuario | `admin` |
| Contraseña | `Clase2026!` |

**Opción A: con DBeaver o pgAdmin.** Crea una conexión PostgreSQL con esos datos.

**Opción B: con `psql` desde Docker** (no necesitas instalar nada):

```bash
docker run --rm -it -e PGPASSWORD=Clase2026! postgres:16-alpine psql -h host.docker.internal -p 7001 -U admin -d tienda
```

> Dentro de un contenedor, `host.docker.internal` significa "mi PC".

Ya dentro de `psql`, crea tablas relacionadas:

```sql
CREATE TABLE clientes (
  id     serial PRIMARY KEY,
  nombre text NOT NULL,
  ciudad text
);

CREATE TABLE pedidos (
  id         serial PRIMARY KEY,
  cliente_id int REFERENCES clientes(id),
  total      numeric(10,2),
  fecha      date DEFAULT current_date
);

INSERT INTO clientes (nombre, ciudad) VALUES ('Ana', 'Bogotá'), ('Luis', 'Medellín');
INSERT INTO pedidos (cliente_id, total) VALUES (1, 120.50), (1, 35.00), (2, 80.00);

SELECT c.nombre, count(p.id) AS pedidos, sum(p.total) AS gastado
FROM clientes c JOIN pedidos p ON p.cliente_id = c.id
GROUP BY c.nombre
ORDER BY gastado DESC;
```

Sal con `\q`.

## Ejercicio 3: copia de seguridad (snapshot) y restauración

Crea una copia:

```bash
aws rds create-db-snapshot --db-instance-identifier tienda-db --db-snapshot-identifier tienda-copia
```

```bash
aws rds describe-db-snapshots --db-snapshot-identifier tienda-copia --query "DBSnapshots[0].Status"
```

Cuando diga `available`, crea **una base nueva** a partir de la copia:

```bash
aws rds restore-db-instance-from-db-snapshot --db-instance-identifier tienda-restaurada --db-snapshot-identifier tienda-copia
```

Lista tus bases:

```bash
aws rds describe-db-instances --query "DBInstances[].[DBInstanceIdentifier,DBInstanceStatus,Endpoint.Port]" --output table
```

Conéctate a la restaurada (puerto `7002`) y comprueba que tiene los mismos datos. Es lo que harías en AWS real para recuperar datos tras un error o crear un entorno de pruebas.

## Ejercicio 4: detener, arrancar y cambiar la contraseña

```bash
aws rds stop-db-instance --db-instance-identifier tienda-restaurada
```

```bash
aws rds start-db-instance --db-instance-identifier tienda-restaurada
```

```bash
aws rds modify-db-instance --db-instance-identifier tienda-db --master-user-password NuevaClave2026! --apply-immediately
```

## Ejercicio 5: MySQL

Lo mismo funciona con MySQL. Solo cambia el motor:

```bash
aws rds create-db-instance --db-instance-identifier blog-db --engine mysql --db-instance-class db.t3.micro --master-username admin --master-user-password Clase2026! --db-name blog --allocated-storage 20
```

Conéctate con MySQL Workbench o:

```bash
docker run --rm -it mysql:8.0 mysql -h host.docker.internal -P PUERTO -u admin -pClase2026! blog
```

(Pon el puerto que muestre `describe-db-instances`.)

## Borrar

```bash
aws rds delete-db-instance --db-instance-identifier tienda-restaurada --skip-final-snapshot
```

`--skip-final-snapshot` evita que AWS cree una copia final antes de borrar.

---

## Reto

1. Crea una base PostgreSQL `biblioteca` con tablas `libros`, `socios` y `prestamos` (con claves foráneas).
2. Inserta datos y escribe una consulta con JOIN que liste qué socio tiene qué libro.
3. Haz un snapshot, borra por error la tabla `prestamos` (`DROP TABLE prestamos;`) y recupérala restaurando el snapshot en una instancia nueva.

## Chuleta

| Quiero... | Comando |
|---|---|
| Crear | `aws rds create-db-instance --db-instance-identifier X --engine postgres ...` |
| Ver estado y puerto | `aws rds describe-db-instances --query "DBInstances[].[DBInstanceIdentifier,DBInstanceStatus,Endpoint.Port]"` |
| Snapshot | `aws rds create-db-snapshot --db-instance-identifier X --db-snapshot-identifier COPIA` |
| Restaurar | `aws rds restore-db-instance-from-db-snapshot --db-instance-identifier NUEVA --db-snapshot-identifier COPIA` |
| Detener / arrancar | `aws rds stop-db-instance` / `start-db-instance` |
| Borrar | `aws rds delete-db-instance --db-instance-identifier X --skip-final-snapshot` |

## En la consola web

Menú **Database**: tus instancias, su estado y su endpoint.
