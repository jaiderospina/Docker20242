# Redes en Docker con MySQL y phpMyAdmin

Este ejemplo muestra cómo Docker Networking permite conectar servicios de forma aislada y segura, resolviendo automáticamente los nombres de los servicios dentro de la misma red.

En este ejemplo, crearemos un entorno Docker con:
1. Una red personalizada para aislar nuestros servicios
2. Un contenedor con MySQL (base de datos)
3. Un contenedor con phpMyAdmin (interfaz web para administrar MySQL)
4. Conexión entre estos servicios a través de la red Docker

## Paso 1: Crear una red personalizada en Docker

Las redes en Docker permiten la comunicación entre contenedores de forma aislada. Crearemos una red llamada `mi-red-app`:

```bash
docker network create mi-red-app
```

## Paso 2: Configurar el contenedor MySQL

Vamos a crear un contenedor MySQL con las siguientes características:
- Usará nuestra red personalizada
- Tendrá variables de entorno para configuración
- Los datos persistirán en un volumen

```bash
docker run -d \
  --name mi-mysql \
  --network mi-red-app \
  -e MYSQL_ROOT_PASSWORD=my-secret-pw \
  -e MYSQL_DATABASE=mi_base_datos \
  -e MYSQL_USER=mi_usuario \
  -e MYSQL_PASSWORD=mi_contraseña \
  -v mysql_data:/var/lib/mysql \
  mysql:8.0

****
docker run -d --name mi-mysql --network mi-red-app -e MYSQL_ROOT_PASSWORD=my-secret-pw -e MYSQL_DATABASE=mi_base_datos -e MYSQL_USER=mi_usuario -e MYSQL_PASSWORD=mi_contraseña -v mysql_data:/var/lib/mysql   mysql:8.0

```

Explicación de los parámetros:
- `-d`: Ejecuta el contenedor en segundo plano (detached mode)
- `--name`: Asigna un nombre al contenedor
- `--network`: Conecta el contenedor a nuestra red personalizada
- `-e`: Establece variables de entorno (configuración de MySQL)
- `-v`: Crea un volumen para persistencia de datos

## Paso 3: Configurar phpMyAdmin

Ahora crearemos un contenedor con phpMyAdmin que se conectará a nuestro MySQL:

```bash
docker run -d \
  --name mi-phpmyadmin \
  --network mi-red-app \
  -e PMA_HOST=mi-mysql \
  -e PMA_PORT=3306 \
  -p 8080:80 \
  phpmyadmin/phpmyadmin
```

Explicación de los parámetros:
- `PMA_HOST=mi-mysql`: Le indica a phpMyAdmin el nombre del servicio MySQL (resuelto por la red Docker)
- `PMA_PORT=3306`: Puerto estándar de MySQL
- `-p 8080:80`: Mapea el puerto 80 del contenedor al 8080 de nuestro host

## Paso 4: Verificar la conexión

1. Accede a phpMyAdmin en tu navegador: http://localhost:8080
2. Inicia sesión con:
   - Usuario: root
   - Contraseña: my-secret-pw (la que configuramos en MYSQL_ROOT_PASSWORD)

## Paso 5: Conectar una aplicación (ejemplo con Node.js)

Para completar el ejemplo, aquí está cómo conectar una aplicación Node.js a nuestra base de datos MySQL:

```javascript
// app.js
const mysql = require('mysql2');

const connection = mysql.createConnection({
  host: 'mi-mysql', // Nombre del servicio MySQL en la red Docker
  user: 'mi_usuario',
  password: 'mi_contraseña',
  database: 'mi_base_datos'
});

connection.connect(err => {
  if (err) {
    console.error('Error de conexión: ' + err.stack);
    return;
  }
  console.log('Conectado a MySQL con ID ' + connection.threadId);
});

// Ejemplo de consulta
connection.query('SELECT 1 + 1 AS solution', (err, results) => {
  if (err) throw err;
  console.log('La solución es: ', results[0].solution);
});

connection.end();
```

Para ejecutar esta aplicación en Docker, crearíamos un Dockerfile:

```dockerfile
# Dockerfile
FROM node:14
WORKDIR /usr/src/app
COPY package*.json ./
RUN npm install
COPY . .
EXPOSE 3000
CMD ["node", "app.js"]
```

Y lo ejecutaríamos en nuestra red:

```bash
docker build -t mi-app-node .
docker run -it --rm --network mi-red-app mi-app-node
```

## Estructura final de la red

```
+---------------------------------------------------+
|                  Red Docker: mi-red-app           |
|                                                   |
|  +-------------+    +----------------+            |
|  | Contenedor  |    | Contenedor     |            |
|  | MySQL       |    | phpMyAdmin     |            |
|  | (mi-mysql)  |<-->| (mi-phpmyadmin)|            |
|  +-------------+    +----------------+            |
|                                                   |
|  +-------------+                                  |
|  | Contenedor  |                                  |
|  | Node.js App |                                  |
|  | (mi-app)    |                                  |
|  +-------------+                                  |
+---------------------------------------------------+
```

## Comandos útiles para verificar

1. Listar contenedores en ejecución:
   ```bash
   docker ps
   ```

2. Inspeccionar la red:
   ```bash
   docker network inspect mi-red-app
   ```

3. Ver logs de un contenedor:
   ```bash
   docker logs mi-mysql
   ```

4. Detener y eliminar todos los recursos:
   ```bash
   docker stop mi-mysql mi-phpmyadmin
   docker rm mi-mysql mi-phpmyadmin
   docker network rm mi-red-app
   docker volume rm mysql_data
   ```

