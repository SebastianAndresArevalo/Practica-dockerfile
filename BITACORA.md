# Bitacora: Calculadora Full-Stack con Docker Compose

**Curso:** Infrastructure Computing Course  
**Universidad:** Universidad del Quindio  
**Fecha:** 5 de noviembre de 2026

## 1. Objetivo de la actividad

Aplicar los conceptos de la guia de Docker para crear una aplicacion full-stack con frontend, backend y base de datos usando Docker Compose.

La aplicacion desarrollada es una calculadora web que realiza sumas, restas, multiplicaciones y divisiones. Cada operacion se almacena en PostgreSQL para consultar el historial.

## 2. Conceptos aprendidos

Un contenedor empaqueta codigo, dependencias y configuracion. Se ejecuta aislado, pero comparte el kernel del sistema operativo anfitrion.

| Caracteristica | Maquina virtual | Contenedor |
|---|---|---|
| Peso | Gigabytes | Megabytes |
| Inicio | Minutos | Segundos |
| Overhead | Alto | Bajo |
| Portabilidad | Limitada | Alta |

Las imagenes son plantillas inmutables y los contenedores son instancias de esas imagenes. Un Dockerfile es la receta para crear una imagen y Docker Compose coordina varios contenedores.

## 3. Comandos iniciales de Docker

La guia propone verificar Docker y practicar con imagenes oficiales:

```bash
docker --version
docker info
docker run hello-world
docker search ubuntu
docker search nginx
docker search node
docker pull ubuntu:22.04
docker run -it ubuntu:22.04 /bin/bash
docker ps
docker ps -a
docker start <container_id>
docker stop <container_id>
docker rm <container_id>
```

Para ejecutar Nginx y publicar un puerto:

```bash
docker run -d --name web -p 8080:80 nginx
docker logs web
docker exec -it web /bin/sh
```

## 4. Dockerfile

Un Dockerfile define como se construye una imagen. Las instrucciones principales son `FROM`, `RUN`, `COPY`, `WORKDIR`, `ENV`, `EXPOSE`, `CMD` y `HEALTHCHECK`.

Ejemplo de la guia:

```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY server.js .
EXPOSE 3000
CMD ["node", "server.js"]
```

Buenas practicas aplicadas en este proyecto:

- Usar imagenes oficiales y pequenas.
- Separar frontend, backend y base de datos.
- No incluir credenciales en GitHub.
- Usar `.gitignore` para excluir `.env` y `node_modules`.
- Validar las imagenes antes de ejecutarlas.

## 5. Ejercicio integrador de monitoreo de la guia

La guia tambien presenta un sistema con API de metricas, PostgreSQL, Redis y Nginx. Al revisar ese ejercicio se encontraron errores de indentacion en el archivo Compose, un `package.json` vacio y un Dockerfile de Nginx incompleto.

El error principal era:

```text
mapping key "build" already defined
mapping key "networks" already defined
mapping key "restart" already defined
```

Se corrigieron esos archivos y se probaron los endpoints `/api/` y `/health`. Luego se creo este proyecto nuevo para resolver la tarea de la casa con una aplicacion diferente.

## 6. Aplicacion desarrollada

### Servicios

1. `frontend`: Nginx sirve la interfaz y funciona como reverse proxy.
2. `backend`: Node.js y Express procesan las operaciones.
3. `db`: PostgreSQL guarda el historial.

### Estructura

```text
calculadora-docker-compose/
├── backend/
│   ├── Dockerfile
│   ├── package.json
│   └── server.js
├── frontend/
│   ├── Dockerfile
│   ├── nginx.conf
│   ├── index.html
│   ├── styles.css
│   └── app.js
├── docker-compose.yml
├── .env
├── .env.example
├── .gitignore
├── BITACORA.md
└── contenedores-guia-docker-unquindio.pdf
```

### Endpoints de la API

```text
GET  /api/health
GET  /api/calculations
POST /api/calculations
```

Ejemplo de solicitud:

```json
{
  "number1": 10,
  "number2": 5,
  "operation": "multiply"
}
```

Resultado:

```json
{
  "number1": 10,
  "number2": 5,
  "operation": "multiply",
  "result": 50
}
```

## 7. Requisitos de la tarea cumplidos

- **Al menos 3 servicios:** `frontend`, `backend` y `db`.
- **Volumen persistente:** `calculator-data` montado en `/var/lib/postgresql/data`.
- **Red personalizada:** `calculator-network` con driver `bridge`.
- **Health checks:** `pg_isready` para PostgreSQL y `wget` para el backend.
- **Variables de entorno:** credenciales, nombre de base de datos y puertos definidos en `.env`.
- **Frontend:** interfaz web para ingresar numeros y seleccionar operaciones.
- **Backend:** API REST con Express.
- **Base de datos:** tabla `calculations` creada automaticamente.

## 8. Archivos Docker principales

### Backend Dockerfile

```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package.json .
RUN npm install --omit=dev
COPY server.js .
EXPOSE 3001
CMD ["node", "server.js"]
```

### Frontend Dockerfile

```dockerfile
FROM nginx:alpine
COPY nginx.conf /etc/nginx/conf.d/default.conf
COPY index.html styles.css app.js /usr/share/nginx/html/
```

### Persistencia y red

```yaml
volumes:
  calculator-data:

networks:
  calculator-network:
    driver: bridge
```

## 9. Ejecucion y pruebas

Validar Compose:

```bash
docker compose config
```

Construir y ejecutar:

```bash
docker compose up -d --build
docker compose ps
```

Probar el health check:

```bash
curl http://localhost:8081/health
```

Respuesta esperada:

```json
{"status":"healthy","database":"connected"}
```

Crear una operacion:

```bash
curl -X POST http://localhost:8081/api/calculations \
  -H 'Content-Type: application/json' \
  -d '{"number1":10,"number2":5,"operation":"multiply"}'
```

Consultar el historial:

```bash
curl http://localhost:8081/api/calculations
```

La interfaz web queda disponible en:

```text
http://localhost:8081
```

## 10. Verificacion de persistencia

Para verificar el volumen:

```bash
docker compose down
docker compose up -d
curl http://localhost:8081/api/calculations
```

La operacion debe continuar en el historial porque PostgreSQL utiliza el volumen `calculator-data`.

Para eliminar tambien los datos se usa:

```bash
docker compose down -v
```

Este comando debe utilizarse con cuidado porque elimina el volumen.

## 11. Respuestas de repaso

### Diferencia entre CMD y ENTRYPOINT

`CMD` define el comando por defecto y puede ser reemplazado al ejecutar el contenedor. `ENTRYPOINT` define el proceso principal y es menos flexible.

### Importancia de `.dockerignore`

Evita copiar dependencias, logs, archivos temporales y credenciales al contexto de construccion. Esto mejora el tiempo de build y la seguridad.

### Persistencia de datos

Los datos se conservan montando un volumen Docker en el directorio de datos de PostgreSQL.

### Funcion de EXPOSE

Documenta el puerto interno que usa la aplicacion. No publica el puerto por si solo; la publicacion se configura con `ports`.

### Comunicacion entre contenedores

Los servicios de una misma red se comunican usando el nombre del servicio. En este proyecto el backend se conecta a PostgreSQL usando `DB_HOST=db`.

## 12. Publicacion en GitHub

Antes de publicar se debe confirmar que `.env` esta excluido por `.gitignore`. La plantilla segura es `.env.example`.

```bash
git init
git add .
git commit -m "Calculadora full-stack con Docker Compose"
git branch -M main
git remote add origin git@github.com:USUARIO/calculadora-docker-compose.git
git push -u origin main
```

La autenticacion SSH utiliza una clave publica agregada en GitHub. La clave privada nunca debe compartirse.

## 13. Conclusion

La tarea se resolvio creando una aplicacion full-stack sencilla y funcional. El proyecto demuestra la construccion de imagenes, la ejecucion de tres servicios, la comunicacion por una red personalizada, el uso de variables de entorno, los health checks y la persistencia de datos con PostgreSQL.
