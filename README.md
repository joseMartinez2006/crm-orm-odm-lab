# CRM ORM/ODM Lab

API REST de un CRM básico que combina un ORM (Sequelize + PostgreSQL) y un ODM (Mongoose + MongoDB).

## Stack

- Node.js 22, Express 5, CommonJS
- Sequelize + PostgreSQL 16 (`User`, `Company`, `Contact`)
- Mongoose + MongoDB 7 (`Activity`)
- Jest + Supertest
- GitHub Codespaces, Dev Containers, Docker Compose
- Supervisor (`npm run dev`)

## Arquitectura

```text
GitHub Codespace
│
├── app       Node.js 22  ──┬── Sequelize ──> postgres (PostgreSQL)
│                           └── Mongoose  ──> mongo    (MongoDB)
├── postgres
└── mongo
```

La aplicación se conecta por nombre de servicio (`postgres`, `mongo`). Las credenciales de desarrollo llegan como variables de entorno definidas en `.devcontainer/docker-compose.yml` (ver `.env.example`).

## Iniciar el Codespace

1. En GitHub: **Code → Codespaces → Create codespace on main**.
2. Espera a que se levanten los tres servicios (`app`, `postgres`, `mongo`). `postCreateCommand` ejecuta `npm install`.

## Instalar dependencias

```bash
npm install
```

## Seed y reset

```bash
npm run seed    # inserta datos deterministas (3 users, 4 companies, 8 contacts, 10 activities)
npm run reset   # elimina y recrea tablas/base de datos y vuelve a sembrar
```

## Iniciar la API

```bash
npm start       # node ./bin/www
npm run dev     # supervisor ./bin/www
```

Servidor en el puerto `3000` (variable `PORT`).

## Pruebas

```bash
npm test
```

Cada suite restablece PostgreSQL y MongoDB antes de ejecutarse y cierra las conexiones al terminar.

## Endpoints

| Método | Ruta | Descripción |
|---|---|---|
| GET | `/health` | Health check |
| GET | `/users` | Listar usuarios |
| GET | `/users/:id` | Obtener usuario |
| POST | `/users` | Crear usuario |
| PUT | `/users/:id` | Actualizar usuario |
| DELETE | `/users/:id` | Eliminar usuario |
| GET | `/companies` | Listar compañías (`?industry=`) |
| GET | `/companies/:id` | Obtener compañía |
| POST | `/companies` | Crear compañía |
| PUT | `/companies/:id` | Actualizar compañía |
| DELETE | `/companies/:id` | Eliminar compañía |
| GET | `/contacts` | Listar contactos |
| GET | `/contacts/:id` | Obtener contacto |
| POST | `/contacts` | Crear contacto |
| PUT | `/contacts/:id` | Actualizar contacto |
| DELETE | `/contacts/:id` | Eliminar contacto |
| GET | `/activities` | Listar actividades (`?type=`) |
| GET | `/activities/:id` | Obtener actividad |
| POST | `/activities` | Crear actividad |
| PUT | `/activities/:id` | Actualizar actividad |
| DELETE | `/activities/:id` | Eliminar actividad |

Los errores se devuelven como JSON: `{ "error": "Contact not found" }`.

## Respuestas

### 5.1 Sobre la arquitectura

**1. Dos motores.**  
Activity es buen candidato para MongoDB porque su campo `metadata` puede contener estructuras diferentes según el tipo de actividad. Company y Contact son buenos candidatos para PostgreSQL porque tienen relaciones claras entre sí y necesitan llaves foráneas para mantener esas relaciones.

**2. ORM vs ODM.**  
Un ORM permite trabajar con bases de datos relacionales usando objetos y modelos; en este proyecto se usa Sequelize. Un ODM hace algo similar con bases documentales; aquí se usa Mongoose. La principal diferencia es que Sequelize trabaja con tablas y relaciones, mientras Mongoose trabaja con documentos y colecciones.

**3. Configuración por variables de entorno.**  
Las credenciales se definen mediante las variables de entorno configuradas en el Codespace, en lugar de escribirlas directamente en los archivos `.js`. Esto evita exponer información sensible en el código. La aplicación usa `DB_HOST=postgres` y `MONGODB_URI` con el host `mongo`, porque esos nombres corresponden a los servicios definidos en Docker Compose y no a `localhost`.

### 5.2 Sobre Sequelize y PostgreSQL

**4. Asociaciones.**  
En `models/sequelize/index.js`, Company tiene muchos Contact mediante `Company.hasMany(Contact, { as: 'contacts' })`. La llave foránea es `companyId` y vive en la tabla `Contact`; sirve para indicar a qué compañía pertenece cada contacto. El alias `contacts` permite incluir y acceder a esos contactos mediante ese nombre.

**5. Eager loading.**  
Si primero se trae la compañía y después se hace otra consulta para sus contactos, se necesitan dos consultas a la base de datos. Con `include`, Sequelize puede obtener la compañía y sus contactos en una misma operación. En el Reto 05 es preferible `include` porque simplifica el código y evita una consulta adicional.

**6. Instancia vs consulta.**  
En el Reto 07 primero se busca el contacto y después se usa `contact.update()`, por lo que se trabaja directamente con la instancia encontrada. `Model.update({...}, { where })` realiza la actualización directamente y puede ser más sencillo, pero normalmente devuelve información sobre la operación y no el registro actualizado como instancia.

### 5.3 Sobre Mongoose y MongoDB

**7. Esquema flexible.**  
En `models/mongoose/activity.js`, `metadata` utiliza un tipo de objeto flexible, por lo que puede guardar diferentes propiedades dependiendo de la actividad. Esto permite que CALL, EMAIL y MEETING tengan información adicional distinta. La desventaja es que se pierde parte de la validación que habría al definir cada campo con un tipo específico.

**8. Sin ref.**  
`contactId` y `userId` son números y no tienen `ref` porque los contactos y usuarios están almacenados en PostgreSQL, no en MongoDB. Por eso Mongoose no puede usar `populate()` para resolver esas relaciones. Si se elimina un User en PostgreSQL, MongoDB no elimina automáticamente las actividades que tengan su `userId`.

**9. Documento actualizado.**  
Antes del Reto 08, `findByIdAndUpdate()` devolvía por defecto el documento anterior a la modificación. En `controllers/activities.js` se agregó `{ new: true, runValidators: true }`. Con `new: true`, la API devuelve el documento después de actualizarlo.

### 5.4 Sobre pruebas y proceso

**10. Pruebas de comportamiento.**  
Probar el comportamiento permite comprobar que la API funciona correctamente sin depender de cómo está escrito internamente el código. Por ejemplo, el test verifica que `/contacts` devuelva los contactos esperados, sin obligar a usar específicamente `findAll()`. Esto permite cambiar la implementación sin romper la prueba si el comportamiento sigue siendo correcto.

**11. Repetibilidad.**  
`tests/setup.js` prepara las bases de datos antes de las pruebas y limpia o restablece los datos después de cada suite. Esto evita que los resultados de una prueba afecten a las siguientes. Por eso `npm test` puede ejecutarse varias veces con un estado inicial conocido y obtener resultados consistentes.

**12. Mi experiencia.**  
El reto que me resultó más difícil fue el Reto 05, porque primero tuve que revisar la asociación entre Company y Contact en `models/sequelize/index.js`. También tuve un error de sintaxis al colocar el código fuera de la función `getById`; el mensaje de Jest permitió identificar que el problema estaba en el archivo `controllers/companies.js` y corregirlo.

## Evidencia

![npm test con las 9 suites en verde](<Evidencia npm test.png>)