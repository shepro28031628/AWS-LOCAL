# 2. La consola web

[Volver al índice](README.md)

La consola web te deja ver lo que tienes en tu nube local sin escribir comandos.

## Abrirla

1. Asegúrate de que AWS Nube Local está encendido ([manual 1](01-instalar-y-arrancar.md)).
2. Abre en el navegador: <http://localhost:4566/_floci/ui>
3. La primera vez verás una pantalla de "Starting..." unos segundos mientras arranca. Después te lleva sola a <http://localhost:4500>.

## Qué hay en la pantalla

| Zona | Qué muestra |
|---|---|
| **Menú izquierdo** | Los servicios agrupados: Compute, Storage, Databases, Networking... |
| **Console Home** | Resumen: cuántos servicios están activos y cuántos recursos tienes |
| **Arriba a la derecha** | Si la consola está conectada (punto verde, *Connected*) |
| **Barra de búsqueda** | Busca un servicio por nombre, por ejemplo "S3" o "DynamoDB" |

## Prueba rápida

1. Crea un bucket desde la terminal (lo explicamos en el [manual de S3](10-s3.md)):

   ```bash
   aws s3 mb s3://prueba-consola
   ```

2. En la consola, entra en **Storage**. Debe aparecer `prueba-consola`.

## Panel principal

En <http://localhost:4566/> tienes además un panel con los datos de conexión (endpoint, región y cuenta) listos para copiar, y ejemplos para la terminal, la AWS CLI y Python.

**Siguiente:** [3. Configurar la AWS CLI](03-configurar-aws-cli.md)
