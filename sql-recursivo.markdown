---
layout: page
title: SQL Recursivo
permalink: /sql-recursivo/
---

# Escenario de un foro con múltiples hilos de comentarios

Se quiere implementar un foro donde los lectores de un hilo pueden contribuir con comentarios e incluso responderse formándose hilos de discusión.

Se considera un modelo de base de datos con una sola tabla Comentarios y una relación reflexiva 1:N id_comentario -> id_padre.

![Comentarios](Comentarios.png)
![Relación reflexiva Comentarios](RelacionReflexivaComentarios.png)

``` sql
CREATE TABLE Comentarios (
    id_comentario   SERIAL PRIMARY KEY,
    id_padre        BIGINT UNSIGNED,
    comentario      TEXT NOT NULL,
    FOREIGN KEY (id_padre) REFERENCES Comentarios(id_comentario)
);

INSERT INTO Comentarios
(id_comentario,
id_padre,
comentario)
VALUES
(1,
NULL,
'Primer comentario');

INSERT INTO Comentarios
(id_comentario,
id_padre,
comentario)
VALUES
(2,
1,
'Segundo comentario');

INSERT INTO Comentarios
(id_comentario,
id_padre,
comentario)
VALUES
(3,
2,
'Tercer comentario');

INSERT INTO Comentarios
(id_comentario,
id_padre,
comentario)
VALUES
(4,
1,
'Cuarto comentario');

INSERT INTO Comentarios
(id_comentario,
id_padre,
comentario)
VALUES
(5,
4,
'Quinto comentario');

INSERT INTO Comentarios
(id_comentario,
id_padre,
comentario)
VALUES
(6,
2,
'Sexto comentario');

INSERT INTO Comentarios
(id_comentario,
id_padre,
comentario)
VALUES
(7,
3,
'Séptimo comentario');
```

```
Estructura de ejemplo de comentarios
------------------------------------

                  ┌─────┐
                  │  1  ├───────────┐
                  │     │           │
                  └──┬──┘           │
                     │              │
                     │              │
                     │              │
                     │              │
                  ┌──┴──┐        ┌──┴──┐
                  │  2  │        │  4  │
        ┌─────────┤     │        │     │
        │         └──┬──┘        └──┬──┘
        │            │              │
        │            │              │
        │            │              │
        │            │              │
     ┌──┴──┐      ┌──┴──┐        ┌──┴──┐
     │  6  │      │  3  │        │  5  │
     │     │      │     │        │     │
     └─────┘      └──┬──┘        └─────┘
                     │
                     │
                     │
                     │
                  ┌──┴──┐
                  │  7  │
                  │     │
                  └─────┘


```

Con MySQL 8+ tenemos disponible la opción de implementar consultas recursivas con la cláusula WITH:

[With Recursivo Mysql](https://dev.mysql.com/doc/refman/8.0/en/with.html#common-table-expressions-recursive)

Con esta cláusula podemos cargar los comentarios para cualquier nivel con una recursión como la siguiente:

```sql
WITH RECURSIVE hilos AS (
  SELECT     id_comentario,
             id_padre,
             comentario
  FROM       Comentarios
  WHERE      id_padre = 2
  UNION ALL
  SELECT     c.id_comentario,
             c.id_padre,
             c.comentario
  FROM       Comentarios c INNER JOIN hilos h ON 
			 c.id_padre = h.id_comentario)
SELECT * FROM hilos ORDER BY id_padre, id_comentario;
```

Cuya salida será todo el árbol de comentarios para el nodo padre 2:

```console
+---------------+----------+---------------------+
| id_comentario | id_padre | comentario          |
+---------------+----------+---------------------+
|             3 |        2 | Tercer comentario   |
|             6 |        2 | Sexto comentario    |
|             7 |        3 | Séptimo comentario  |
+---------------+----------+---------------------+
```
En este listado se excluye el comentario 2. Es fácil modificar la consulta para que se muestre también dicho comentario:

```sql
WITH RECURSIVE hilos AS (
  SELECT     id_comentario,
             id_padre,
             comentario
  FROM       Comentarios
  WHERE      id_comentario = 2
  UNION ALL
  SELECT     c.id_comentario,
             c.id_padre,
             c.comentario
  FROM       Comentarios c INNER JOIN hilos h ON 
			 c.id_padre = h.id_comentario)
SELECT * FROM hilos ORDER BY id_padre, id_comentario;
```

Es decir:

```
Estructura de comentarios para el nodo 2 incluyéndolo
-----------------------------------------------------
                    │
                 ┌──┴──┐
                 │  2  │
       ┌─────────┤     │
       │         └──┬──┘
       │            │
       │            │
       │            │
       │            │
    ┌──┴──┐      ┌──┴──┐
    │  6  │      │  3  │
    │     │      │     │
    └─────┘      └──┬──┘
                    │
                    │
                    │
                    │
                 ┌──┴──┐
                 │  7  │
                 │     │
                 └─────┘

```

En esta query recursiva se pueden identificar  partes:

 - Declaración

```sql
WITH RECURSIVE hilos AS (
```

- Semilla

```sql
  SELECT     id_comentario,
             id_padre,
             comentario
  FROM       Comentarios
  WHERE      id_padre = 2
```

- Recursión

```sql
  UNION ALL
  SELECT     c.id_comentario,
             c.id_padre,
             c.comentario
  FROM       Comentarios c INNER JOIN hilos h ON 
			 c.id_padre = h.id_comentario)
```
- Parada

```sql
Comentarios c INNER JOIN hilos h ON 
			 c.id_padre = h.id_comentario
```

- Resultado

```sql
SELECT * FROM hilos ORDER BY id_padre, id_comentario;
```

Una forma de visualizar la recursión es introduciendo una columna de nivel que se va incrementando recursivamente con cada fila que se añade a la tabla base CTE (Common Table Expression) *hilos*.

```sql
WITH RECURSIVE hilos AS (
  SELECT     id_comentario,
             id_padre,
             comentario,
             0 as nivel
  FROM       Comentarios
  WHERE      id_comentario = 2
  UNION ALL
  SELECT     c.id_comentario,
             c.id_padre,
             c.comentario, 
             h.nivel+1 as nivel
  FROM       Comentarios c INNER JOIN hilos h ON 
			 c.id_padre = h.id_comentario)
SELECT * FROM hilos ORDER BY id_padre, id_comentario;
```

```console
+---------------+----------+---------------------+-------+
| id_comentario | id_padre | comentario          | nivel |
+---------------+----------+---------------------+-------+
|             2 |        1 | Segundo comentario  |     0 |
|             3 |        2 | Tercer comentario   |     1 |
|             6 |        2 | Sexto comentario    |     1 |
|             7 |        3 | Séptimo comentario  |     2 |
+---------------+----------+---------------------+-------+
```

