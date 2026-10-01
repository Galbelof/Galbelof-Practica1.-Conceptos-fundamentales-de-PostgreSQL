# Práctica 1. Conceptos fundamentales de PostgreSQL

## 1. Creación de la base de datos

```sql
CREATE DATABASE biblioteca;
```

```text
CREATE DATABASE
```

## 2. Creación de usuarios y permisos

### Usuarios, rol y asignación

```sql
CREATE USER admin_biblio
WITH PASSWORD 'AdminPractica_2026!';
```

```text
CREATE ROLE
```

```sql
CREATE USER usuario_biblio
WITH PASSWORD 'LecturaInicial_2026!';
```

```text
CREATE ROLE
```

```sql
CREATE ROLE lectores NOLOGIN;
```

```text
CREATE ROLE
```

```sql
ALTER DATABASE biblioteca OWNER TO admin_biblio;
```

```text
ALTER DATABASE
```

```sql
GRANT lectores TO usuario_biblio;
```

```text
GRANT ROLE
```

```sql
SELECT rolname, rolcanlogin, rolsuper
FROM pg_roles
WHERE rolname IN ('admin_biblio', 'usuario_biblio', 'lectores')
ORDER BY rolname;
```

```text
rolname     | rolcanlogin | rolsuper
----------------+-------------+----------
 admin_biblio   | t           | f
 lectores       | f           | f
 usuario_biblio | t           | f
(3 rows)
```

```sql
ALTER USER usuario_biblio
WITH PASSWORD 'LecturaNueva_2026!';
```

```text
ALTER ROLE
```

```sql
\connect biblioteca
```

```text
You are now connected to database "biblioteca" as user "postgres".
```

```sql
REVOKE ALL PRIVILEGES ON DATABASE biblioteca FROM PUBLIC;
```

```text
REVOKE
```

```sql
GRANT CONNECT ON DATABASE biblioteca
TO admin_biblio, lectores;
```

```text
GRANT
```

```sql
REVOKE ALL PRIVILEGES ON SCHEMA public FROM PUBLIC;
```

```text
REVOKE
```

```sql
GRANT USAGE, CREATE ON SCHEMA public TO admin_biblio;
```

```text
GRANT
```

```sql
GRANT USAGE ON SCHEMA public TO lectores;
```

```text
GRANT
```

### Permisos de lectura sobre las tablas

```sql
REVOKE ALL PRIVILEGES ON ALL TABLES IN SCHEMA public
FROM PUBLIC, lectores, usuario_biblio;
```

```text
REVOKE
```

```sql
GRANT SELECT ON ALL TABLES IN SCHEMA public TO lectores;
```

```text
GRANT
```

```sql
SELECT
    tabla,
    has_table_privilege(
        'usuario_biblio', tabla, 'SELECT'
    ) AS puede_consultar,
    has_table_privilege(
        'usuario_biblio', tabla, 'INSERT'
    ) AS puede_insertar,
    has_table_privilege(
        'usuario_biblio', tabla, 'UPDATE'
    ) AS puede_actualizar,
    has_table_privilege(
        'usuario_biblio', tabla, 'DELETE'
    ) AS puede_eliminar
FROM (
    VALUES
        ('public.autores'),
        ('public.libros'),
        ('public.prestamos')
) AS t(tabla);
```

```text
tabla       | puede_consultar | puede_insertar | puede_actualizar | puede_eliminar
------------------+-----------------+----------------+------------------+----------------
 public.autores   | t               | f              | f                | f
 public.libros    | t               | f              | f                | f
 public.prestamos | t               | f              | f                | f
(3 rows)
```

### Comprobación de eliminación como usuario_biblio

```sql
SELECT current_user, current_database();
```

```text
current_user  | current_database
----------------+------------------
 usuario_biblio | biblioteca
(1 row)
```

```sql
BEGIN;
```

```text
BEGIN
```

```sql
DELETE FROM libros WHERE id_libro = 1;
```

```text
ERROR:  permission denied for table libros
```

```sql
ROLLBACK;
```

```text
ROLLBACK
```

```sql
SELECT * FROM libros WHERE id_libro = 1;
```

```text
id_libro |        titulo        | año_publicacion | id_autor
----------+----------------------+-----------------+----------
        1 | Cien años de soledad |            1967 |        1
(1 row)
```

## 3. Creación de tablas

```sql
CREATE TABLE autores (
    id_autor SERIAL PRIMARY KEY,
    nombre VARCHAR(100) NOT NULL,
    nacionalidad VARCHAR(60) NOT NULL
);
```

```text
CREATE TABLE
```

```sql
CREATE TABLE libros (
    id_libro SERIAL PRIMARY KEY,
    titulo VARCHAR(200) NOT NULL,
    año_publicacion INTEGER,
    id_autor INTEGER NOT NULL,
    CONSTRAINT fk_libros_autores
        FOREIGN KEY (id_autor)
        REFERENCES autores(id_autor)
);
```

```text
CREATE TABLE
```

```sql
CREATE TABLE prestamos (
    id_prestamo SERIAL PRIMARY KEY,
    id_libro INTEGER NOT NULL,
    fecha_prestamo DATE NOT NULL,
    fecha_devolucion DATE,
    usuario_prestatario VARCHAR(100) NOT NULL,
    CONSTRAINT fk_prestamos_libros
        FOREIGN KEY (id_libro)
        REFERENCES libros(id_libro)
        ON DELETE CASCADE,
    CONSTRAINT fechas_validas
        CHECK (
            fecha_devolucion IS NULL
            OR fecha_devolucion >= fecha_prestamo
        )
);
```

```text
CREATE TABLE
```

```sql
\dt
```

```text
List of relations
 Schema |   Name    | Type  |    Owner
--------+-----------+-------+--------------
 public | autores   | table | admin_biblio
 public | libros    | table | admin_biblio
 public | prestamos | table | admin_biblio
(3 rows)
```

## 4. Inserción de datos

```sql
INSERT INTO autores (nombre, nacionalidad)
VALUES
    ('Gabriel García Márquez', 'Colombia'),
    ('Isabel Allende', 'Chile'),
    ('Miguel de Cervantes', 'España'),
    ('J. K. Rowling', 'Reino Unido'),
    ('George Orwell', 'Reino Unido');
```

```text
INSERT 0 5
```

```sql
INSERT INTO libros (titulo, año_publicacion, id_autor)
VALUES
    ('Cien años de soledad', 1967, 1),
    ('El amor en los tiempos del cólera', 1985, 1),
    ('La casa de los espíritus', 1982, 2),
    ('Eva Luna', 1987, 2),
    ('Don Quijote de la Mancha', 1605, 3),
    ('Harry Potter y la piedra filosofal', 1997, 4),
    ('1984', 1949, 5),
    ('Rebelión en la granja', 1945, 5);
```

```text
INSERT 0 8
```

```sql
INSERT INTO prestamos (
    id_libro,
    fecha_prestamo,
    fecha_devolucion,
    usuario_prestatario
)
VALUES
    (1, '2026-09-01', '2026-09-10', 'Ana'),
    (1, '2026-09-15', NULL, 'Luis'),
    (2, '2026-09-02', NULL, 'Ana'),
    (2, '2026-09-05', '2026-09-12', 'Marta'),
    (3, '2026-09-03', NULL, 'Luis'),
    (4, '2026-09-04', '2026-09-11', 'Ana'),
    (7, '2026-09-06', NULL, 'Luis'),
    (8, '2026-09-07', '2026-09-14', 'Marta');
```

```text
INSERT 0 8
```

## 5. Consultas básicas

### a. Libros con su autor

```sql
SELECT
    l.titulo,
    a.nombre AS autor
FROM libros AS l
JOIN autores AS a ON l.id_autor = a.id_autor
ORDER BY l.id_libro;
```

```text
titulo               |         autor
------------------------------------+------------------------
 Cien años de soledad               | Gabriel García Márquez
 El amor en los tiempos del cólera  | Gabriel García Márquez
 La casa de los espíritus           | Isabel Allende
 Eva Luna                           | Isabel Allende
 Don Quijote de la Mancha           | Miguel de Cervantes
 Harry Potter y la piedra filosofal | J. K. Rowling
 1984                               | George Orwell
 Rebelión en la granja              | George Orwell
(8 rows)
```

### b. Préstamos pendientes de devolución

```sql
SELECT
    p.id_prestamo,
    l.titulo,
    p.fecha_prestamo,
    p.usuario_prestatario
FROM prestamos AS p
JOIN libros AS l ON p.id_libro = l.id_libro
WHERE p.fecha_devolucion IS NULL
ORDER BY p.id_prestamo;
```

```text
id_prestamo |              titulo               | fecha_prestamo | usuario_prestatario
-------------+-----------------------------------+----------------+---------------------
           2 | Cien años de soledad              | 2026-09-15     | Luis
           3 | El amor en los tiempos del cólera | 2026-09-02     | Ana
           5 | La casa de los espíritus          | 2026-09-03     | Luis
           7 | 1984                              | 2026-09-06     | Luis
(4 rows)
```

### c. Autores con más de un libro

```sql
SELECT
    a.nombre AS autor,
    COUNT(l.id_libro) AS numero_libros
FROM autores AS a
JOIN libros AS l ON a.id_autor = l.id_autor
GROUP BY a.id_autor, a.nombre
HAVING COUNT(l.id_libro) > 1
ORDER BY a.nombre;
```

```text
autor          | numero_libros
------------------------+---------------
 Gabriel García Márquez |             2
 George Orwell          |             2
 Isabel Allende         |             2
(3 rows)
```

## 6. Consultas con agregación

### a. Número total de préstamos

```sql
SELECT COUNT(*) AS total_prestamos
FROM prestamos;
```

```text
total_prestamos
-----------------
               8
(1 row)
```

### b. Número de libros prestados por usuario

```sql
SELECT
    usuario_prestatario,
    COUNT(*) AS numero_libros_prestados
FROM prestamos
GROUP BY usuario_prestatario
ORDER BY usuario_prestatario;
```

```text
usuario_prestatario | numero_libros_prestados
---------------------+-------------------------
 Ana                 |                       3
 Luis                |                       3
 Marta               |                       2
(3 rows)
```

## 7. Modificación de datos

### a. Actualizar la devolución de un préstamo pendiente

```sql
UPDATE prestamos
SET fecha_devolucion = '2026-09-20'
WHERE id_prestamo = 3
  AND fecha_devolucion IS NULL
RETURNING *;
```

```text
id_prestamo | id_libro | fecha_prestamo | fecha_devolucion | usuario_prestatario
-------------+----------+----------------+------------------+---------------------
           3 |        2 | 2026-09-02     | 2026-09-20       | Ana
(1 row)

UPDATE 1
```

### b. Eliminar un libro y comprobar sus préstamos

```sql
SELECT * FROM prestamos WHERE id_libro = 8;
```

```text
id_prestamo | id_libro | fecha_prestamo | fecha_devolucion | usuario_prestatario
-------------+----------+----------------+------------------+---------------------
           8 |        8 | 2026-09-07     | 2026-09-14       | Marta
(1 row)
```

```sql
DELETE FROM libros
WHERE id_libro = 8
RETURNING *;
```

```text
id_libro |        titulo         | año_publicacion | id_autor
----------+-----------------------+-----------------+----------
        8 | Rebelión en la granja |            1945 |        5
(1 row)

DELETE 1
```

```sql
SELECT * FROM prestamos WHERE id_libro = 8;
```

```text
id_prestamo | id_libro | fecha_prestamo | fecha_devolucion | usuario_prestatario
-------------+----------+----------------+------------------+---------------------
(0 rows)
```

```sql
SELECT COUNT(*) AS libros_restantes FROM libros;
```

```text
libros_restantes
------------------
                7
(1 row)
```

```sql
SELECT COUNT(*) AS prestamos_restantes FROM prestamos;
```

```text
prestamos_restantes
---------------------
                   7
(1 row)
```

## 8. Creación de vistas

### a. Vista de libros prestados

```sql
CREATE VIEW vista_libros_prestados AS
SELECT
    l.titulo,
    a.nombre AS autor,
    p.usuario_prestatario AS nombre_prestatario
FROM prestamos AS p
JOIN libros AS l ON p.id_libro = l.id_libro
JOIN autores AS a ON l.id_autor = a.id_autor;
```

```text
CREATE VIEW
```

```sql
SELECT * FROM vista_libros_prestados
ORDER BY titulo, nombre_prestatario;
```

```text
titulo               |         autor          | nombre_prestatario
-----------------------------------+------------------------+--------------------
 1984                              | George Orwell          | Luis
 Cien años de soledad              | Gabriel García Márquez | Ana
 Cien años de soledad              | Gabriel García Márquez | Luis
 El amor en los tiempos del cólera | Gabriel García Márquez | Ana
 El amor en los tiempos del cólera | Gabriel García Márquez | Marta
 Eva Luna                          | Isabel Allende         | Ana
 La casa de los espíritus          | Isabel Allende         | Luis
(7 rows)
```

### b. Permisos de consulta

```sql
REVOKE ALL PRIVILEGES ON vista_libros_prestados
FROM PUBLIC, lectores;
```

```text
REVOKE
```

```sql
GRANT SELECT ON vista_libros_prestados TO usuario_biblio;
```

```text
GRANT
```

```sql
SELECT
    has_table_privilege(
        'usuario_biblio',
        'vista_libros_prestados',
        'SELECT'
    ) AS usuario_puede_consultar,
    has_table_privilege(
        'lectores',
        'vista_libros_prestados',
        'SELECT'
    ) AS lectores_puede_consultar;
```

```text
usuario_puede_consultar | lectores_puede_consultar
-------------------------+--------------------------
 t                       | f
(1 row)
```

## 9. Funciones y consultas avanzadas

### a. Función que devuelve los libros de un autor

```sql
CREATE FUNCTION libros_por_autor(p_nombre VARCHAR)
RETURNS TABLE (
    id_libro INTEGER,
    titulo VARCHAR,
    año_publicacion INTEGER
)
LANGUAGE SQL
STABLE
AS $$
    SELECT
        l.id_libro,
        l.titulo,
        l.año_publicacion
    FROM libros AS l
    JOIN autores AS a ON l.id_autor = a.id_autor
    WHERE a.nombre = p_nombre
    ORDER BY l.id_libro;
$$;
```

```text
CREATE FUNCTION
```

```sql
SELECT *
FROM libros_por_autor('Gabriel García Márquez');
```

```text
id_libro |              titulo               | año_publicacion
----------+-----------------------------------+-----------------
        1 | Cien años de soledad              |            1967
        2 | El amor en los tiempos del cólera |            1985
(2 rows)
```

### b. Tres libros más prestados

```sql
SELECT
    l.titulo,
    COUNT(p.id_prestamo) AS numero_prestamos
FROM libros AS l
JOIN prestamos AS p ON l.id_libro = p.id_libro
GROUP BY l.id_libro, l.titulo
ORDER BY numero_prestamos DESC, l.id_libro ASC
LIMIT 3;
```

```text
titulo               | numero_prestamos
-----------------------------------+------------------
 Cien años de soledad              |                2
 El amor en los tiempos del cólera |                2
 La casa de los espíritus          |                1
(3 rows)
```

## 10. Exportación e importación de datos

### a. Exportación de libros a CSV

```sql
\copy libros TO 'libros.csv' WITH (FORMAT csv, HEADER true, ENCODING 'UTF8')
```

```text
COPY 7
```

### b. Importación de autores desde CSV

```sql
\copy autores(nombre, nacionalidad) FROM 'autores_extra.csv' WITH (FORMAT csv, HEADER true, ENCODING 'UTF8')
```

```text
COPY 2
```

```sql
SELECT * FROM autores ORDER BY id_autor;
```

```text
id_autor |         nombre         | nacionalidad
----------+------------------------+--------------
        1 | Gabriel García Márquez | Colombia
        2 | Isabel Allende         | Chile
        3 | Miguel de Cervantes    | España
        4 | J. K. Rowling          | Reino Unido
        5 | George Orwell          | Reino Unido
        6 | Julio Cortázar         | Argentina
        7 | Mary Shelley           | Reino Unido
(7 rows)
```
