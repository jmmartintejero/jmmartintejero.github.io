---
layout: page
title: SQL Window
permalink: /sql-window/
--

# Escenario de ventas

Se quieren implementar análisis avanzados de funciones agregadas sobre ventanas en una base de datos de ventas. 

Se considera un modelo de base de datos con una sola tabla Ventas.

![Comentarios](Comentarios.png)
![Relación reflexiva Comentarios](RelacionReflexivaComentarios.png)

``` sql

-- DDL:
CREATE DATABASE negocio;
USE negocio;

CREATE TABLE Ventas
(
    anio    INT,
    pais VARCHAR(20),
    producto VARCHAR(32),
    venta  INT
);

-- DML: 
INSERT INTO Ventas VALUES (2000 ,'Finlandia', 'Ordenador', 1000);
INSERT INTO Ventas VALUES (2000 ,'Finlandia', 'Ordenador', 500);
INSERT INTO Ventas VALUES (2000 ,'Finlandia', 'Teléfono', 100);

INSERT INTO Ventas VALUES (2001 ,'Finlandia', 'Teléfono', 10);

INSERT INTO Ventas VALUES (2000 ,'India', 'Calculadora', 100);
INSERT INTO Ventas VALUES (2000 ,'India', 'Calculadora', 50);
INSERT INTO Ventas VALUES (2000 ,'India', 'Ordenador', 1100);
INSERT INTO Ventas VALUES (2000 ,'India', 'Ordenador', 500);

INSERT INTO Ventas VALUES (2000 ,'USA', 'Calculadora', 50);
INSERT INTO Ventas VALUES (2000 ,'USA', 'Calculadora', 25);
INSERT INTO Ventas VALUES (2000 ,'USA', 'Ordenador', 1200);
INSERT INTO Ventas VALUES (2000 ,'USA', 'Ordenador', 300);

INSERT INTO Ventas VALUES (2001 ,'USA', 'Calculadora', 10);
INSERT INTO Ventas VALUES (2001 ,'USA', 'Calculadora', 25);
INSERT INTO Ventas VALUES (2001 ,'USA', 'Ordenador', 200);
INSERT INTO Ventas VALUES (2001 ,'USA', 'Ordenador', 300);
```



Con MySQL 8+ podemos implementar consultas con enventanado mediante la cláusula OVER() Y OVER(PARTITION BY):

[OVER()y OVER(PARTITION BY ) Enventanado Mysql](https://dev.mysql.com/doc/refman/8.0/en/window-functions-usage.html)

Por ejemplo, para mostrar el listado de las filas país, año, producto y venta, y a la vez informar en cada fila de resultado agregado de los totales de ventas globales y de países:

```sql
SELECT pais, anio, producto, venta,
SUM(venta) OVER () total_ventas_global,
SUM(venta) OVER (PARTITION BY pais) total_ventas_pais
from Ventas;
```

Cuya salida será:

```console
+-----------+------+-------------+-------+---------------------+-------------------+
| pais      | anio | producto    | venta | total_ventas_global | total_ventas_pais |
+-----------+------+-------------+-------+---------------------+-------------------+
| Finlandia | 2000 | Ordenador   |  1000 |                5470 |              1610 |
| Finlandia | 2000 | Ordenador   |   500 |                5470 |              1610 |
| Finlandia | 2000 | Teléfono    |   100 |                5470 |              1610 |
| Finlandia | 2001 | Teléfono    |    10 |                5470 |              1610 |
| India     | 2000 | Calculadora |   100 |                5470 |              1750 |
| India     | 2000 | Calculadora |    50 |                5470 |              1750 |
| India     | 2000 | Ordenador   |  1100 |                5470 |              1750 |
| India     | 2000 | Ordenador   |   500 |                5470 |              1750 |
| USA       | 2000 | Calculadora |    50 |                5470 |              2110 |
| USA       | 2000 | Calculadora |    25 |                5470 |              2110 |
| USA       | 2000 | Ordenador   |  1200 |                5470 |              2110 |
| USA       | 2000 | Ordenador   |   300 |                5470 |              2110 |
| USA       | 2001 | Calculadora |    10 |                5470 |              2110 |
| USA       | 2001 | Calculadora |    25 |                5470 |              2110 |
| USA       | 2001 | Ordenador   |   200 |                5470 |              2110 |
| USA       | 2001 | Ordenador   |   300 |                5470 |              2110 |
+-----------+------+-------------+-------+---------------------+-------------------+
```


Este ejemplo se puede refinar para añadir a los campos resultado anterior el porcentaje de la venta sobre la venta global total:


```sql
SELECT pais, anio, producto, venta,
SUM(venta) OVER () total_ventas_global,
SUM(venta) OVER (PARTITION BY pais) total_ventas_pais,
ROUND(venta / ( SUM(venta) OVER () ) * 100, 2) porcentaje_venta_sobre_total_global
from Ventas;
```

O escrito de mediante un wrap:

```sql
SELECT *, ROUND(venta / total_ventas_global *100, 2) porcentaje_venta_sobre_total_global
(SELECT pais, anio, producto, venta,
SUM(venta) OVER () total_ventas_global,
SUM(venta) OVER (PARTITION BY pais) total_ventas_pais,
from Ventas ) as T_AUX;
```

```console
+-----------+------+-------------+-------+---------------------+-------------------+-------------------------------------+
| pais      | anio | producto    | venta | total_ventas_global | total_ventas_pais | porcentaje_venta_sobre_total_global |
+-----------+------+-------------+-------+---------------------+-------------------+-------------------------------------+
| Finlandia | 2000 | Ordenador   |  1000 |                5470 |              1610 |                               18.28 |
| Finlandia | 2000 | Ordenador   |   500 |                5470 |              1610 |                                9.14 |
| Finlandia | 2000 | Teléfono    |   100 |                5470 |              1610 |                                1.83 |
| Finlandia | 2001 | Teléfono    |    10 |                5470 |              1610 |                                0.18 |
| India     | 2000 | Calculadora |   100 |                5470 |              1750 |                                1.83 |
| India     | 2000 | Calculadora |    50 |                5470 |              1750 |                                0.91 |
| India     | 2000 | Ordenador   |  1100 |                5470 |              1750 |                               20.11 |
| India     | 2000 | Ordenador   |   500 |                5470 |              1750 |                                9.14 |
| USA       | 2000 | Calculadora |    50 |                5470 |              2110 |                                0.91 |
| USA       | 2000 | Calculadora |    25 |                5470 |              2110 |                                0.46 |
| USA       | 2000 | Ordenador   |  1200 |                5470 |              2110 |                               21.94 |
| USA       | 2000 | Ordenador   |   300 |                5470 |              2110 |                                5.48 |
| USA       | 2001 | Calculadora |    10 |                5470 |              2110 |                                0.18 |
| USA       | 2001 | Calculadora |    25 |                5470 |              2110 |                                0.46 |
| USA       | 2001 | Ordenador   |   200 |                5470 |              2110 |                                3.66 |
| USA       | 2001 | Ordenador   |   300 |                5470 |              2110 |                                5.48 |
+-----------+------+-------------+-------+---------------------+-------------------+-------------------------------------+
```
