# Entrega N.º 1 — Mocking de usuarios y mascotas

API REST de la **Entrega N.º 1 del curso de Backend**. Genera datos falsos (usuarios y mascotas) con Faker, los expone mediante endpoints de mocking y permite insertarlos en **MongoDB Atlas**.

## Tecnologías

- Node.js (v18 o superior)
- Express
- MongoDB Atlas + Mongoose
- @faker-js/faker
- bcryptjs
- dotenv

## Estructura del proyecto

```text
entrega-1-mocking/
├── src/
│   ├── app.js                    # Servidor Express y registro de routers
│   ├── config/
│   │   └── database.js           # Conexión a MongoDB con Mongoose
│   ├── middlewares/
│   │   └── requireDB.js          # Responde 503 si no hay conexión a la base
│   ├── models/
│   │   ├── user.model.js
│   │   └── pet.model.js
│   ├── routes/
│   │   ├── mocks.router.js       # /api/mocks
│   │   ├── users.router.js       # /api/users
│   │   └── pets.router.js        # /api/pets
│   ├── services/
│   │   ├── users.service.js      # Consulta e inserción de usuarios
│   │   └── pets.service.js       # Consulta e inserción de mascotas
│   └── mocks/
│       ├── mockingUsers.js       # Generador de usuarios falsos (no toca la BD)
│       └── mockingPets.js        # Generador de mascotas falsas (no toca la BD)
├── tests/
│   └── mocking.test.js           # Pruebas que no requieren MongoDB
├── .env.example
├── .gitignore
├── package.json
└── README.md
```

La **generación** (carpeta `mocks/`) y la **persistencia** (carpeta `services/`) son responsabilidades separadas.

## Instalación

```bash
npm install
```

## Configuración de MongoDB Atlas

1. En Atlas, andá a **Database Access** y confirmá que existe tu usuario de base de datos con su contraseña.
2. En **Network Access**, agregá tu IP actual (o `0.0.0.0/0` solo para pruebas).
3. Verificá que el cluster esté activo.
4. Copiá `.env.example` a `.env` y completá tus datos.

## Variables de entorno

Archivo `.env` (nunca se sube a GitHub, está en `.gitignore`):

```env
PORT=8080
MONGO_URI=mongodb+srv://<db_user>:<db_password>@<cluster>.mongodb.net/entrega1_mocking?retryWrites=true&w=majority
```

Reemplazá `<db_user>`, `<db_password>` y `<cluster>` por tus valores. Si la contraseña tiene caracteres especiales (`@`, `:`, `/`, `#`), codificalos en formato URL (por ejemplo `@` → `%40`).

La base de datos utilizada se llama `entrega1_mocking`.

## Cómo iniciar el servidor

```bash
npm start        # producción
npm run dev      # desarrollo (reinicia al guardar cambios)
```

El servidor queda disponible en `http://localhost:8080`. Si la conexión a MongoDB falla, el servidor no arranca y muestra el motivo en consola.

## Endpoints

### `GET /api/mocks/mockingpets`

Genera 100 mascotas falsas. **No** las guarda en la base de datos.

```json
{
  "status": "success",
  "total": 100,
  "payload": [
    { "_id": "6710a1f2c3d4e5f607182930", "name": "Luna", "specie": "gato", "age": 4, "adopted": false, "owner": null }
  ]
}
```

### `GET /api/mocks/mockingusers`

Genera **50 usuarios falsos** por defecto. **No** los guarda en la base de datos. Parámetro opcional `quantity` (entero entre 1 y 1000):

```bash
curl http://localhost:8080/api/mocks/mockingusers
curl "http://localhost:8080/api/mocks/mockingusers?quantity=10"
```

Cada usuario tiene `_id` (ObjectId), `first_name`, `last_name`, `email` único dentro del lote, `password` (hash bcrypt de `coder123`), `role` (`user` o `admin`) y `pets: []`.

```json
{
  "status": "success",
  "total": 50,
  "payload": [
    {
      "_id": "6710a1f2c3d4e5f607182931",
      "first_name": "Jacynthe",
      "last_name": "Kertzmann",
      "email": "jacynthe_kertzmann98@gmail.com",
      "password": "$2a$10$...",
      "role": "user",
      "pets": []
    }
  ]
}
```

Valores de `quantity` inválidos (texto, decimales, negativos, 0 o mayores a 1000) responden `400`.

### `POST /api/mocks/generateData`

Genera **e inserta** usuarios y/o mascotas en MongoDB.

```bash
curl -X POST http://localhost:8080/api/mocks/generateData \
  -H "Content-Type: application/json" \
  -d '{ "users": 10, "pets": 15 }'
```

Respuesta exitosa (`201`):

```json
{
  "status": "success",
  "message": "Datos generados correctamente",
  "payload": { "usersInserted": 10, "petsInserted": 15 }
}
```

Reglas:

- Se puede enviar solo `users`, solo `pets` o ambos (el parámetro omitido se toma como 0).
- Deben ser **números enteros** entre 0 y 1000 (no se aceptan strings, decimales ni negativos).
- Si ambos son 0 o no se envía ninguno, responde `400`.
- Los emails nuevos no se repiten con los ya guardados en la base.
- Si la inserción es parcial o falla, responde `500` con las cantidades **realmente insertadas** (nunca un éxito falso).
- Sin conexión a MongoDB responde `503`.

Ejemplo de error (`400`):

```json
{ "status": "error", "message": "\"users\" no puede ser negativo" }
```

### `GET /api/users`

Lista los usuarios guardados en MongoDB. **No** incluye el campo `password`.

### `GET /api/pets`

Lista las mascotas guardadas en MongoDB.

## Cómo comprobar los registros insertados

1. Ejecutá `POST /api/mocks/generateData` con `{ "users": 10, "pets": 15 }`.
2. Consultá `GET /api/users` y `GET /api/pets`: `total` debe reflejar los registros insertados.
3. Reiniciá el servidor y volvé a consultar: los datos siguen ahí (persisten en Atlas).
4. También podés verlos en Atlas: **Database → Browse Collections → `entrega1_mocking` → `users` / `pets`**.

## Pruebas

```bash
npm test
```

Ejecuta pruebas automáticas (sin necesidad de MongoDB) que verifican: mascotas falsas, 50 usuarios con `_id` válido, emails únicos, ambos roles, `pets: []`, hashes bcrypt verificables contra `coder123`, y las validaciones de `generateData` (negativos, texto, decimales, parámetros omitidos, cero registros, límite excedido y falta de conexión).

Las pruebas de inserción real (`generateData` → `GET /api/users` y `GET /api/pets` → persistencia tras reiniciar) se hacen manualmente contra tu base `entrega1_mocking` siguiendo la sección anterior.

## Seguridad

- No se guardan credenciales en el código.
- `.env` está en `.gitignore`.
- Las contraseñas se almacenan con hash bcrypt, nunca en texto plano.
