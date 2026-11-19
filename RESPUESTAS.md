# Respuestas de la práctica: Spring Many-to-Many

> Entregable de la práctica. El README pide responder las preguntas de los pasos
> 1 a 3 y comparar los resultados con este archivo, pero no indica un nombre de
> archivo; lo dejé como `RESPUESTAS.md` en la raíz del repo.

## Qué hice

1. Creé el modelo `Department` (categorías) con su `categoriaRepo`.
2. Agregué la relación de categorías al modelo `Producto`.
3. Creé el `CategoriaController` con sus rutas.
4. Levanté la aplicación contra PostgreSQL, ejecuté las pruebas del paso 4 y
   revisé las tablas generadas.

---

## Pregunta 1: ¿Cómo afecta `@ManyToMany` + `@JoinTable` a la base de datos?

Con `spring.jpa.hibernate.ddl-auto=update`, JPA crea **dos tablas**:

**1. `categorias`** (la entidad `Department`). Columnas reales en la base:

| Columna | Tipo |
|---|---|
| `id` | varchar(255), PK |
| `nombre` | varchar(255) |
| `descripcion` | varchar(255) |
| `tipo` | varchar(255) |
| `icono` | varchar(255) |
| `imagen_navbar` | varchar(255) |

**2. `producto_deparments`** (la tabla de unión que define `@JoinTable`):

| Columna | Tipo |
|---|---|
| `product_id` | varchar(255), parte de la PK |
| `deparments_id` | varchar(255), parte de la PK |

- La llave primaria es **compuesta**: `(product_id, deparments_id)`, así que la
  misma pareja no se puede repetir.
- Tiene **dos llaves foráneas**:
  `product_id → categorias(id)` y `deparments_id → producto(id)`.

**Importante (detalle que confunde):** los nombres están al revés de lo que
parecen. `joinColumns` pertenece al **lado propietario** (`Department`), por eso
`product_id` guarda el UUID **de la categoría** y `deparments_id` guarda el UUID
**del producto**. Lo comprobé con una fila real de la tabla:

```text
              product_id              |            deparments_id
--------------------------------------+--------------------------------------
 10458906-5fcf-44c4-967f-8c6323bdb993 | 469a5cf6-25dc-4e44-90de-ad420a14c0b9
      (id de la categoría)            |        (id del producto)
```

Además, la tabla `producto` **no** gana ninguna columna para las categorías: la
asociación vive completa en la tabla de unión. Ese es el sentido de `mappedBy =
"productos"` en `Producto`: le dice a JPA que esa colección solo se lee, porque
la escritura la controla `Department`.

---

## Pregunta 2: ¿Qué significa `@JsonIgnore` y por qué se usa en `Producto`?

`@JsonIgnore` es una anotación de Jackson (la librería que convierte objetos Java
a JSON). Marca una propiedad para que **no se incluya al serializar** el objeto.

En `Producto` se usa porque la relación es **bidireccional**: `Department`
tiene `productos` y `Producto` tiene `deparments`, y ambos apuntan al mismo par
de filas. Si se serializara todo, al pedir `GET /categorias` Jackson entraría en
`categoría → productos → cada producto → sus categorías → ...` y terminaría en
una recursión infinita (`StackOverflowError`), además de mandar respuestas
enormes.

Comprobado en ejecución: `GET /producto/{id}` devuelve el producto **sin** el
campo `deparments`, mientras que `GET /categorias` sí muestra sus `productos`:

```json
{
  "id": "469a5cf6-25dc-4e44-90de-ad420a14c0b9",
  "nombre": "Proteína de fresa",
  "descripcion": "2.2 kg",
  "lote": "123456",
  "precioUnitario": 1500.5,
  "fechaCaducidad": "2026-05-04",
  "comentarios": [],
  "imagenes": []
}
```

---

## Pregunta 3: el endpoint `addProductoCategoria/{productId}/{categoriaId}`

**¿Cuál es su función?** Crear la asociación muchos-a-muchos entre un producto y
una categoría **que ya existen**: recibe los dos IDs por la URL, los busca en la
base, agrega el producto al `Set<Producto>` de la categoría y guarda. Al guardar,
Hibernate inserta una fila en `producto_deparments`. Como la colección es un
`Set`, si se repite la llamada no se duplica la relación.

**¿Por qué dos repositorios?** Porque cada repositorio administra **una tabla
distinta**: `productoRepo` lee la tabla `producto` y `categoriaRepo` lee la tabla
`categorias`. Para crear la relación hacen falta las dos entidades completas, y
cada una se obtiene por su propio repositorio. Además, la relación la posee
`Department`, por eso el `save` se hace con `categoriaRepo`: es quien controla
la tabla de unión.

---

## Pruebas del paso 4 (resultados reales)

Servidor en `http://localhost:8080`, base `chochos`, datos creados con los
`curl` del README.

| Método | Ruta | Resultado |
|---|---|---|
| `GET` | `/productos` | `200` — lista de productos (arreglo JSON) |
| `POST` | `/producto` | `200` — producto guardado con `id` UUID |
| `GET` | `/producto/{id}` | `200` — producto encontrado |
| `PUT` | `/producto/{id}` | `200` — producto actualizado |
| `DELETE` | `/producto/{id}` | `200` — `{"success":true}` |
| `GET` | `/categorias` | `200` — categorías con sus `productos` |
| `POST` | `/categoria` | `200` — categoría guardada con `id` UUID |
| `POST` | `/addProductoCategoria/{productId}/{categoriaId}` | `200` — categoría con el producto asociado |
| `GET` | `/categoria/producto/{productId}` | `200` — `[{"nombre":"Suplementos", ...}]` |
| `GET` | `/categoria/{id}` | `200` — categoría encontrada |
| `PUT` | `/categoria/{id}` | `200` — categoría actualizada |
| `DELETE` | `/categoria/{id}` | `200` — `{"success":true}` |

Verificación en la base después de asociar: aparece **una** fila en
`producto_deparments`, y al borrar el producto y la categoría la tabla queda en
**0 filas** (Hibernate limpia la tabla de unión).

---

## Observaciones (casos límite que encontré al probar)

Estos comportamientos vienen del código tal como lo indica el README; los anoto
porque conviene saberlos:

1. **Categoría inexistente al asociar** → `500 Internal Server Error`.
   En `addcategoria` se hace `return repo.save(d.get())` sin revisar si `d`
   está vacío, así que el `Optional.get()` lanza `NoSuchElementException`.
2. **Producto inexistente al asociar** → `200 OK`, pero **no** crea la relación
   (no se inserta fila). El `if` anidado simplemente no agrega nada.
3. **`GET /categoria/producto/{id}` con producto inexistente** → `200` con
   cuerpo vacío, porque el método devuelve `null` en ese caso.

Una mejora posible sería devolver `404 Not Found` en esos tres casos, igual que
en las prácticas de códigos HTTP.
