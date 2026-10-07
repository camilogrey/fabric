# Trabajar con datos en un eventhouse de Microsoft Fabric

## Historia del laboratorio
En Microsoft Fabric, un eventhouse se utiliza para almacenar datos en tiempo real relacionados con eventos; a menudo capturados desde una fuente de datos de streaming mediante un eventstream.

Dentro de un eventhouse, los datos se almacenan en una o más bases de datos KQL, cada una de las cuales contiene tablas y otros objetos que se pueden consultar mediante Kusto Query Language (KQL) o un subconjunto de Structured Query Language (SQL).

En este ejercicio, crearé y poblaré un eventhouse con datos de ejemplo relacionados con viajes en bicicleta, y luego consultaré los datos con KQL y SQL.

Este ejercicio tiene una duración aproximada de 25 minutos.

---

## Desarrollo (en primera persona)

### 1. Crear un workspace
Navegué a la página de inicio de Microsoft Fabric e inicié sesión con mis credenciales. En la barra de menú de la izquierda, seleccioné **Workspaces** (el icono similar a 🗇). Creé un nuevo workspace con un nombre de mi elección, seleccionando un modo de licencia que incluye capacidad de Fabric (Trial, Premium o Fabric). Cuando se abrió el nuevo workspace, estaba vacío.

> ![Workspace vacío en Fabric](1.png)

### 2. Crear un Eventhouse
En la barra de menú de la izquierda, seleccioné **Workloads**. Luego, seleccioné el mosaico **Real-Time Intelligence**. En la página de Real-Time Intelligence, seleccioné el mosaico **Real-Time Intelligence Sample** y luego **Bike Rental Data**. Esto creó automáticamente un eventhouse llamado `Bike_Database`.

> ![Workloads y acceso a Real-Time Intelligence](2.png)
> ![Creación de samples con Bike Rental Data](3.png)

En el panel de la izquierda, noté que mi eventhouse contenía una base de datos KQL con el mismo nombre que el eventhouse. Verifiqué que también se había creado una tabla `Bikestream`.

> ![Eventhouse creado con carpeta Bike_sample](4.png)
> ![Elementos del sample: Bike_Eventhouse y Bike_Database](5.png)

### 3. Consultar datos con KQL
Kusto Query Language (KQL) es un lenguaje intuitivo y completo que puedo usar para consultar una base de datos KQL.

#### 3.1. Recuperar datos de una tabla con KQL
En el panel izquierdo de la ventana del eventhouse, bajo mi base de datos KQL, seleccioné el archivo queryset predeterminado. Este archivo contiene algunas consultas KQL de ejemplo para comenzar.

Modifiqué la primera consulta de ejemplo de la siguiente manera:

Bikestream
| take 100


Seleccioné el código de la consulta y lo ejecuté para devolver 100 filas de la tabla.

> ![Consulta take 100 ejecutada](6.png)

Puedo ser más preciso añadiendo atributos específicos que quiero consultar usando la palabra clave `project` y luego usando la palabra clave `take` para indicar al motor cuántos registros devolver.

Escribí, seleccioné y ejecuté la siguiente consulta:

// Use 'project' and 'take' to view a sample number of records in the table and check the data.
Bikestream
| project Street, No_Bikes
| take 10


> ![Consulta project Street, No_Bikes](7.png)

Otra práctica común en el análisis es renombrar columnas en el queryset para hacerlas más fáciles de usar.

Probé la siguiente consulta:

Bikestream
| project Street, ["Number of Empty Docks"] = No_Empty_Docks
| take 10


> ![Consulta con columna renombrada Number of Empty Docks](8.png)

#### 3.2. Resumir datos con KQL
Puedo usar la palabra clave `summarize` con una función para agregar y manipular datos.

Probé la siguiente consulta, que usa la función `sum` para resumir los datos de alquiler y ver cuántas bicicletas están disponibles en total:

Bikestream
| summarize ["Total Number of Bikes"] = sum(No_Bikes)


> ![Consulta summarize con sum(No_Bikes)](9.png)

Puedo agrupar los datos resumidos por una columna o expresión específica.

Ejecuté la siguiente consulta para agrupar el número de bicicletas por barrio y determinar la cantidad de bicicletas disponibles en cada barrio:

Bikestream
| summarize ["Total Number of Bikes"] = sum(No_Bikes) by Neighbourhood
| project Neighbourhood, ["Total Number of Bikes"]


> ![Consulta summarize agrupada por Neighbourhood](10.png)

Si alguno de los puntos de bicicletas tiene una entrada nula o vacía para el barrio, los resultados del resumen incluirán un valor en blanco, lo cual nunca es bueno para el análisis.

Modifiqué la consulta como se muestra aquí para usar la función `case` junto con las funciones `isempty` e `isnull` para agrupar todos los viajes cuyo barrio es desconocido en una categoría `Unidentified` para su seguimiento.

Bikestream
| summarize ["Total Number of Bikes"] = sum(No_Bikes) by Neighbourhood
| project Neighbourhood = case(isempty(Neighbourhood) or isnull(Neighbourhood), "Unidentified", Neighbourhood), ["Total Number of Bikes"]


> ![Consulta con case para agrupar Unidentified](11.png)

> **Nota:** Como este conjunto de datos de ejemplo está bien mantenido, es posible que no tenga un campo `Unidentified` en el resultado de la consulta.

#### 3.3. Ordenar datos con KQL
Para dar más sentido a nuestros datos, normalmente los ordenamos por una columna, y este proceso se realiza en KQL con el operador `sort by` o `order by` (actúan de la misma manera).

Probé la siguiente consulta:

Bikestream
| summarize ["Total Number of Bikes"] = sum(No_Bikes) by Neighbourhood
| project Neighbourhood = case(isempty(Neighbourhood) or isnull(Neighbourhood), "Unidentified", Neighbourhood), ["Total Number of Bikes"]
| sort by Neighbourhood asc


> ![Consulta con sort by Neighbourhood asc](12.png)

Modifiqué la consulta de la siguiente manera y la ejecuté de nuevo, notando que el operador `order by` funciona de la misma manera que `sort by`:

Bikestream
| summarize ["Total Number of Bikes"] = sum(No_Bikes) by Neighbourhood
| project Neighbourhood = case(isempty(Neighbourhood) or isnull(Neighbourhood), "Unidentified", Neighbourhood), ["Total Number of Bikes"]
| order by Neighbourhood asc


> ![Consulta con order by Neighbourhood asc](13.png)

#### 3.4. Filtrar datos con KQL
En KQL, la cláusula `where` se utiliza para filtrar datos. Puedo combinar condiciones en una cláusula `where` usando los operadores lógicos `and` y `or`.

Ejecuté la siguiente consulta para filtrar los datos de bicicletas e incluir solo los puntos de bicicletas en el barrio de Chelsea:

Bikestream
| where Neighbourhood == "Chelsea"
| summarize ["Total Number of Bikes"] = sum(No_Bikes) by Neighbourhood
| project Neighbourhood = case(isempty(Neighbourhood) or isnull(Neighbourhood), "Unidentified", Neighbourhood), ["Total Number of Bikes"]
| sort by Neighbourhood asc


> ![Consulta filtrada para Chelsea](14.png)

### 4. Consultar datos con Transact-SQL
KQL Database no admite Transact-SQL de forma nativa, pero proporciona un endpoint T-SQL que emula Microsoft SQL Server y permite ejecutar consultas T-SQL sobre los datos. El endpoint T-SQL tiene algunas limitaciones y diferencias con respecto al SQL Server nativo. Por ejemplo, no admite la creación, alteración o eliminación de tablas, ni la inserción, actualización o eliminación de datos. Tampoco admite algunas funciones y sintaxis T-SQL que no son compatibles con KQL. Fue creado para permitir que los sistemas que no admiten KQL usen T-SQL para consultar los datos dentro de una base de datos KQL. Por lo tanto, se recomienda usar KQL como lenguaje de consulta principal para KQL Database, ya que ofrece más capacidades y rendimiento que T-SQL. También se pueden usar algunas funciones SQL compatibles con KQL, como `count`, `sum`, `avg`, `min`, `max`, etc.

#### 4.1. Recuperar datos de una tabla con Transact-SQL
En mi queryset, añadí y ejecuté la siguiente consulta Transact-SQL:

SELECT TOP 100 * from Bikestream


> ![Consulta SELECT TOP 100 con T-SQL](15.png)

Modifiqué la consulta de la siguiente manera para recuperar columnas específicas:

SELECT TOP 10 Street, No_Bikes
FROM Bikestream


> ![Consulta SELECT TOP 10 Street, No_Bikes](16.png)

Modifiqué la consulta para asignar un alias que renombre `No_Empty_Docks` a un nombre más fácil de usar.

SELECT TOP 10 Street, No_Empty_Docks as [Number of Empty Docks]
from Bikestream


> ![Consulta con alias Number of Empty Docks](17.png)

#### 4.2. Resumir datos con Transact-SQL
Ejecuté la siguiente consulta para encontrar el número total de bicicletas disponibles:

SELECT sum(No_Bikes) AS [Total Number of Bikes]
FROM Bikestream


> ![Consulta sum(No_Bikes) con T-SQL](18.png)

Modifiqué la consulta para agrupar el número total de bicicletas por barrio:

SELECT Neighbourhood, Sum(No_Bikes) AS [Total Number of Bikes]
FROM Bikestream
GROUP BY Neighbourhood


> ![Consulta con GROUP BY Neighbourhood](19.png)

Modifiqué la consulta aún más para usar una declaración `CASE` para agrupar los puntos de bicicletas con origen desconocido en una categoría `Unidentified` para su seguimiento.

SELECT CASE
WHEN Neighbourhood IS NULL OR Neighbourhood = '' THEN 'Unidentified'
ELSE Neighbourhood
END AS Neighbourhood,
SUM(No_Bikes) AS [Total Number of Bikes]
FROM Bikestream
GROUP BY CASE
WHEN Neighbourhood IS NULL OR Neighbourhood = '' THEN 'Unidentified'
ELSE Neighbourhood
END;


> ![Consulta con CASE para Unidentified](20.png)

#### 4.3. Ordenar datos con Transact-SQL
Ejecuté la siguiente consulta para ordenar los resultados agrupados por barrio:

SELECT CASE
WHEN Neighbourhood IS NULL OR Neighbourhood = '' THEN 'Unidentified'
ELSE Neighbourhood
END AS Neighbourhood,
SUM(No_Bikes) AS [Total Number of Bikes]
FROM Bikestream
GROUP BY CASE
WHEN Neighbourhood IS NULL OR Neighbourhood = '' THEN 'Unidentified'
ELSE Neighbourhood
END
ORDER BY Neighbourhood ASC;


> ![Consulta con ORDER BY Neighbourhood ASC](21.png)

#### 4.4. Filtrar datos con Transact-SQL
Ejecuté la siguiente consulta para filtrar los datos agrupados de modo que solo se incluyan las filas con un barrio de "Chelsea" en los resultados.

SELECT CASE
WHEN Neighbourhood IS NULL OR Neighbourhood = '' THEN 'Unidentified'
ELSE Neighbourhood
END AS Neighbourhood,
SUM(No_Bikes) AS [Total Number of Bikes]
FROM Bikestream
GROUP BY CASE
WHEN Neighbourhood IS NULL OR Neighbourhood = '' THEN 'Unidentified'
ELSE Neighbourhood
END
HAVING Neighbourhood = 'Chelsea'
ORDER BY Neighbourhood ASC;


> ![Consulta con HAVING Neighbourhood = 'Chelsea'](22.png)

### 5. Limpiar recursos
En este ejercicio, he creado un eventhouse y he consultado datos usando KQL y SQL.

Cuando terminé de explorar mi base de datos KQL, eliminé el workspace que creé para este ejercicio.

En la barra de la izquierda, seleccioné el icono de mi workspace.
En la barra de herramientas, seleccioné **Workspace settings**.
En la sección **General**, seleccioné **Remove this workspace**.

---

**Laboratorio completado**
