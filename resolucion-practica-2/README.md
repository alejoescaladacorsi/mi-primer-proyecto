# Clase práctica 2 — Silver, Gold y orquestación

## Propósito

Transformar las tablas Bronze de la práctica 1 en datos confiables y productos analíticos. El flujo se ejecuta como un Lakeflow Job, admite archivos nuevos y demuestra idempotencia mediante validaciones automáticas.

## Objetivos

- Aplicar contratos de tipos y reglas de calidad.
- Separar registros válidos de una cuarentena explicable.
- Resolver duplicados y correcciones con `MERGE`.
- Construir tablas Gold con un grano de negocio explícito.
- Orquestar notebooks dependientes con Lakeflow Jobs.
- Probar una carga nueva sin cambiar el código del pipeline.
- Verificar reconciliación e idempotencia.

## Requisitos previos

La práctica 1 debe haber creado, en el mismo esquema:

```text
bronze_customers
bronze_products
bronze_transactions
bronze_events
```

Usá en todos los notebooks el mismo `student_id` y la misma escala de la práctica 1. Si trabajaste con `small`, no cambies a `test`: las claves foráneas se generan de acuerdo con la escala.

## Secuencia

1. Ejecutá `00_preflight.ipynb`.
2. Ejecutá manualmente `00_generate_new_batch.ipynb` con `batch_002`.
3. Creá el Job siguiendo `GUIA_CREAR_JOB.md`.
4. Ejecutá el Job con `expected_batch_id=batch_002`.
5. Revisá las cuatro tareas y las tablas resultantes.
6. Ejecutá nuevamente el mismo Job sin crear otro archivo.
7. Confirmá que la segunda validación informa métricas estables.
8. Volvé a ejecutar manualmente `00_generate_new_batch.ipynb`, esta vez con `batch_id=batch_003`.
9. Ejecutá el Job con `expected_batch_id=batch_003` y verificá que el nuevo lote aparezca en `gold_batch_summary`.
10. Ejecutá manualmente `05_visualizacion.ipynb` y resolvé las cuatro preguntas usando las tablas Gold.

### Qué hace `expected_batch_id`

El pipeline **no necesita** `expected_batch_id` para encontrar ni procesar archivos. La tarea de ingesta usa `COPY INTO` sobre el directorio de entrada, detecta automáticamente `batch_003` como un archivo nuevo y lo incorpora de manera incremental. Silver y Gold procesan después los datos resultantes sin requerir cambios en el código.

`expected_batch_id` se utiliza solamente en `04_validate_pipeline.ipynb`: expresa qué lote esperamos comprobar en esa ejecución. Al crear `batch_003`, cambiar el parámetro a `batch_003` permite validar que ese lote llegó a Bronze, fue aceptado o enviado a cuarentena, aparece en Gold y aplicó la corrección esperada.

Si se deposita `batch_003` pero se conserva `expected_batch_id=batch_002`, el ETL igualmente procesará el archivo nuevo. Sin embargo, la tarea `validate` puede fallar porque seguirá evaluando las expectativas de `batch_002` y comparando la corrida como si fuera una reejecución sin datos nuevos. El estado rojo representa entonces una expectativa de prueba incorrecta, no una incapacidad del pipeline para detectar el archivo.

## Tablas resultantes

```text
bronze_transactions_incremental
bronze_transactions_all       (vista)
silver_customers
silver_products
silver_transactions
silver_transactions_quarantine
gold_daily_sales
gold_customer_risk
gold_batch_summary
pipeline_run_audit
```

## Preguntas de análisis y comprensión

Respondé las siguientes preguntas después de completar las ejecuciones con `batch_002` y `batch_003`. En las preguntas de análisis incluí la consulta SQL o PySpark utilizada y el resultado relevante. En las preguntas sobre código, indicá el notebook y la sección que fundamentan tu respuesta.

### Análisis de las tablas

1. ¿Cuántas filas físicas recibió cada lote en `bronze_transactions_incremental`? Escribí una consulta que muestre el resultado por `source_batch_id`.

SELECT source_batch_id,
       COUNT(*)                       AS filas_fisicas,
       COUNT(DISTINCT transaction_id) AS ids_distintos
FROM bronze_transactions_incremental
GROUP BY source_batch_id
ORDER BY source_batch_id;


2. Para cada lote, ¿cuántas transacciones fueron aceptadas y cuántas quedaron en `silver_transactions_quarantine`? Reconciliá tus resultados con `gold_batch_summary`.

SELECT
    g.source_batch_id,
    COUNT(DISTINCT s.transaction_id) AS accepted_transactions,
    COUNT(DISTINCT q.transaction_id) AS rejected_transactions,
    g.accepted_transactions AS gold_accepted,
    g.rejected_transactions AS gold_rejected
FROM bigdata_alejo_escalada1.gold_batch_summary g
LEFT JOIN bigdata_alejo_escalada1.silver_transactions s
    ON g.source_batch_id = s.source_batch_id
LEFT JOIN bigdata_alejo_escalada1.silver_transactions_quarantine q
    ON g.source_batch_id = q.source_batch_id
GROUP BY
    g.source_batch_id,
    g.accepted_transactions,
    g.rejected_transactions
ORDER BY g.source_batch_id;

3. ¿Qué motivos de rechazo aparecen en la cuarentena y cuántos registros tiene cada uno por lote? ¿Los rechazos observados coinciden con los casos introducidos por el generador?

SELECT
    source_batch_id,
    quality_reason,
    COUNT(*) AS rejected_records
FROM bigdata_alejo_escalada1.silver_transactions_quarantine
GROUP BY
    source_batch_id,
    quality_reason
ORDER BY
    source_batch_id,
    rejected_records DESC;

4. Seguí la transacción `42` desde `bronze_transactions_all` hasta `silver_transactions`. ¿Cuántas versiones existen en Bronze y cuál quedó vigente en Silver? Mostrá las columnas que justifican la elección.

SELECT
    transaction_id,
    source_batch_id,
    event_ts,
    updated_at,
    customer_id,
    device_id,
    is_fraud,
    amount
FROM bigdata_alejo_escalada1.bronze_transactions_all
WHERE transaction_id = 42
ORDER BY event_ts, updated_at;
SELECT
    transaction_id,
    source_batch_id,
    event_ts,
    updated_at,
    customer_id,
    device_id,
    is_fraud,
    amount
FROM bigdata_alejo_escalada1.silver_transactions
WHERE transaction_id = 42;
La transacción 42 tiene 3 versiones en Bronze: initial, batch_002 y batch_003. La versión de batch_003 posee el updated_at más reciente (2026-03-11 02:00:00), por lo que fue la que quedó vigente en Silver. La versión seleccionada mantiene amount = 1999.99, device_corrected_42 e is_fraud = 1, confirmando que Silver conserva la última versión de la transacción.

5. Comprobá mediante una consulta que `silver_transactions` tiene una sola fila por `transaction_id`. ¿Qué resultado indicaría que la deduplicación falló?

SELECT
    transaction_id,
    COUNT(*) AS cantidad
FROM bigdata_alejo_escalada1.silver_transactions
GROUP BY transaction_id
HAVING COUNT(*) > 1
ORDER BY cantidad DESC;

Si hay transacciones con un ID que se cuenten mas de una vez en la capa silver.. Se podria ver en la tabla. Eso indicaria que la deduplicación fallo

6. Calculá la tasa de rechazo de cada lote como `rechazadas / (aceptadas + rechazadas)` en `gold_batch_summary`. ¿Es correcto comparar solamente las cantidades absolutas si los lotes tienen tamaños diferentes?

SELECT
    source_batch_id,
    accepted_transactions,
    rejected_transactions,
    rejected_transactions * 1.0 /
        (accepted_transactions + rejected_transactions) AS rejection_rate
FROM bigdata_alejo_escalada1.gold_batch_summary
ORDER BY source_batch_id;

Comparar en terminos absolutos esta mal, deberia hacerse una relacion entre lo que se limpia y el total.. 

7. ¿Qué día presenta el mayor monto total y cuál presenta la mayor cantidad de transacciones? Consultá `gold_daily_sales` y explicá si ambos máximos coinciden.

Esta pregunta se responde en el ejercicio de visualización. Para esto necesitamos dos queries
Por un lado explicar cuando es el dia de mayor monto, y por el otro, el dia de mayor cantidad de transacciones. La respuesta es correcta, ambos coinciden, y el dia es 01.03.2026

SELECT
    sale_date,
    SUM(total_amount) AS total_amount,
    SUM(transaction_count) AS transaction_count
FROM bigdata_alejo_escalada1.gold_daily_sales
GROUP BY sale_date
ORDER BY total_amount DESC;

SELECT
    sale_date,
    SUM(total_amount) AS total_amount,
    SUM(transaction_count) AS transaction_count
FROM bigdata_alejo_escalada1.gold_daily_sales
GROUP BY sale_date
ORDER BY transaction_count DESC;

8. ¿Qué canal de pago tiene la mayor tasa global de fraude? Calculala como `SUM(fraud_transactions) / SUM(transaction_count)` y explicá por qué no corresponde promediar directamente `fraud_rate`.
SELECT
    payment_channel,
    SUM(fraud_transactions) AS fraud_transactions,
    SUM(transaction_count) AS transaction_count,
    SUM(fraud_transactions) * 100.0 /
        SUM(transaction_count) AS fraud_rate_pct
FROM bigdata_alejo_escalada1.gold_daily_sales
GROUP BY payment_channel
ORDER BY fraud_rate_pct DESC;
Porque cada fila de gold daily sales no necesariamente representa la misma cantidad de transacciones

9. ¿Qué combinación de país y categoría concentra el mayor monto vendido? Mostrá también la combinación líder dentro de cada país.

SELECT
    country,
    category,
    SUM(total_amount) AS total_sold
FROM bigdata_alejo_escalada1.gold_daily_sales
GROUP BY country, category
ORDER BY total_sold DESC;
WITH ventas AS (
    SELECT
        country,
        category,
        SUM(total_amount) AS total_sold
    FROM bigdata_alejo_escalada1.gold_daily_sales
    GROUP BY country, category
),
ranking AS (
    SELECT
        country,
        category,
        total_sold,
        ROW_NUMBER() OVER (
            PARTITION BY country
            ORDER BY total_sold DESC
        ) AS posicion
    FROM ventas
)
SELECT
    country,
    category,
    total_sold
FROM ranking
WHERE posicion = 1
ORDER BY total_sold DESC;

10. Compará las dos primeras filas de `pipeline_run_audit` correspondientes a la reejecución de `batch_002`. ¿Qué métricas permanecen iguales y qué columna demuestra que se realizó la comparación de idempotencia?

SELECT *
FROM bigdata_alejo_escalada1.pipeline_run_audit
ORDER BY recorded_at;

La ultima columna de comparativas de idempotencia hace el check.
Todos los campos permanecen iguales excepto el ID del job, y su timestamp (y obviamente el check de idempotencia)

### Interpretación del código y del pipeline

11. En `01_ingest_bronze_incremental.ipynb`, ¿qué problema resuelve `COPY INTO` y qué información utiliza para evitar cargar dos veces el mismo archivo físico?

Copy into permite sumar una ingesta incremental, evitando cargar nuevamente archivos fisicos ya procesados. De esta manera, una reejecución no duplica los datos provenientes del mismo archivo.

12. ¿Por qué la vista `bronze_transactions_all` usa `UNION ALL` en lugar de eliminar duplicados? ¿En qué capa se resuelven los duplicados de negocio y por qué?

Se utiliza UNION ALL porque Bronze busca conservar todos los registros recibidos, incluidas las distintas versiones de una misma transacción. La deduplicación se realiza en Silver. Esto permite tener trazabilidad. 

13. ¿Por qué las transacciones iniciales reciben `source_batch_id='initial'` y usan `event_ts` como `updated_at`? ¿Cómo afecta eso a la corrección de la transacción `42`?

Las transacciones originales reciben source_batch_id = 'initial' para poder identificarlas como parte de la carga inicial. Como esos registros no poseen un updated_at propio, se utiliza event_ts como referencia temporal. Esto permite compararlos con las versiones posteriores. En el caso de la transacción 42, las correcciones de batch_002 y batch_003 tienen un updated_at posterior, por lo que Silver conserva finalmente la versión más reciente, correspondiente a batch_003.

14. En `quality_rules.py`, ¿qué ventaja ofrece `try_cast` frente a un `cast` convencional cuando llega un importe como `N/A`?

try_cast permite intentar la conversión sin interrumpir el procesamiento cuando el valor no puede convertirse. Por ejemplo, un amount = 'N/A' se convierte en NULL, lo que permite que posteriormente la regla de calidad lo identifique como INVALID_AMOUNT y lo envíe a cuarentena. Un cast convencional puede provocar un error de conversión según la configuración y detener el procesamiento, mientras que try_cast permite tratar el problema como un dato inválido.

15. Las reglas de calidad asignan una única `quality_reason`. ¿Qué sucede si un registro viola más de una regla y por qué importa el orden de las condiciones?

Si un registro incumple varias reglas simultáneamente, se registra únicamente la primera condición que resulte verdadera. Por eso el orden de las condiciones es importante: determina cuál será considerada la causa principal del rechazo. Por ejemplo, si una fila posee un transaction_id inválido y además un amount inválido, será clasificada como INVALID_TRANSACTION_ID porque esa validación aparece antes que INVALID_AMOUNT

16. Explicá cómo se construye `_record_key` y cómo se usa junto con `row_number`. ¿Qué caso cubre el hash cuando `transaction_id` no puede convertirse a un número?

_record_key utiliza el transaction_id tipado cuando este es válido. Si no puede convertirse a número, genera un hash SHA-256 a partir de los valores raw del registro. Luego row_number() particiona por _record_key y ordena las versiones por updated_at descendente, conservando _rn = 1. De esta manera se selecciona la versión más reciente de cada transacción. El hash permite distinguir y deduplicar registros cuyo transaction_id es inválido, evitando tratar todos los NULL como una misma transacción.

17. Interpretá las dos cláusulas principales del `MERGE` de `silver_transactions`. ¿Cuándo se actualiza una fila existente y cuándo se inserta una nueva?

El MERGE compara las transacciones utilizando transaction_id. Si la transacción ya existe en Silver, solamente se actualiza cuando el updated_at de la nueva versión es posterior al almacenado. Si el transaction_id todavía no existe, se inserta como una nueva fila. Esto permite incorporar tanto nuevas transacciones como correcciones posteriores sin generar duplicados.

18. ¿Por qué las tablas Gold se reconstruyen completamente en esta práctica mientras Silver se actualiza con `MERGE`? Mencioná una ventaja y una limitación de cada estrategia.

Silver utiliza MERGE porque necesita mantener el estado de cada transacción incorporando registros nuevos y actualizando únicamente aquellos que poseen una versión más reciente. Esto resulta eficiente para cargas incrementales, aunque requiere una lógica de actualización más compleja. Gold, en cambio, se reconstruye completamente desde Silver mediante CREATE OR REPLACE TABLE, lo que simplifica la lógica y garantiza resultados deterministas en cada ejecución. Su limitación es que, con grandes volúmenes de datos, reconstruir todas las agregaciones puede resultar más costoso que una actualización incremental.

19. ¿Por qué `expected_batch_id` no participa en la detección del archivo nuevo? Indicá qué parte del pipeline descubre `batch_003` y qué parte utiliza el parámetro.

expected_batch_id no se utiliza para detectar archivos nuevos. Esa responsabilidad corresponde a COPY INTO en 01_ingest_bronze_incremental, que inspecciona el directorio de entrada e incorpora automáticamente los archivos físicos todavía no procesados. expected_batch_id se utiliza principalmente en 04_validate_pipeline para indicar qué lote se espera validar y comprobar su presencia en Bronze, Silver y Gold, además de las reglas asociadas a esa ejecución.

20. Si la tarea `build_silver` falla, ¿qué ocurre con `build_gold` y `validate` en el Job? Explicá cómo las dependencias del DAG evitan publicar o validar resultados incompletos.

Si build_silver falla, las tareas posteriores que dependen de ella no se ejecutan satisfactoriamente: build_gold queda bloqueada por la dependencia y, en consecuencia, validate tampoco continúa. Las dependencias del DAG garantizan que una tarea solo avance cuando sus predecesoras requeridas finalizaron correctamente. De esta manera se evita construir productos Gold sobre datos Silver incompletos y posteriormente validar o publicar resultados inconsistentes.

## Duración estimada

| Bloque | Minutos |
|---|---:|
| Repaso y preflight | 15 |
| Contratos, calidad y cuarentena | 25 |
| Silver y `MERGE` | 30 |
| Pausa | 10 |
| Gold y reconciliación | 25 |
| Construcción del Job | 25 |
| Archivo nuevo y primera ejecución | 20 |
| Segunda ejecución y cierre | 10 |
| Visualización orientada a preguntas | 20 |

## Entrega

En tu repositorio personal creá `resolucion-practica-2/` con:

```text
resolucion-practica-2/
├── README.md
├── 02_build_silver.ipynb
├── 03_build_gold.ipynb
├── 04_validate_pipeline.ipynb
└── 05_visualizacion.ipynb
```

El `README.md` debe incluir:

- Nombre y `student_id`. Alejo Escalada. alejo_escalada1
- Captura del DAG del Job con las cuatro tareas.

- URL del Job o su nombre exacto.
![image_1791038759471.png](./image_1791038759471.png "image_1791038759471.png")

- Resultados de la primera y segunda ejecución.
|job_run_id|expected_batch_id|silver_rows|quarantine_rows|gold_rows|gold_total_amount|recorded_at|idempotence_compared|
|---|---|---|---|---|---|---|---|
|826449793655254|batch_002|50149|53|598|50286183.33|2026-10-03T13:03:48.888Z|false|

|job_run_id|expected_batch_id|silver_rows|quarantine_rows|gold_rows|gold_total_amount|recorded_at|idempotence_compared|
|---|---|---|---|---|---|---|---|
|355827486323868|batch_002|50149|53|598|50286183.33|2026-10-03T13:06:05.685Z|true|
|826449793655254|batch_002|50149|53|598|50286183.33|2026-10-03T13:03:48.888Z|false|


- Cantidades aceptadas y rechazadas para `batch_002`.

|source_batch_id|accepted_transactions|rejected_transactions|total_amount|fraud_transactions|refreshed_at|
|---|---|---|---|---|---|
|batch_002|201|2|146812.99|28|2026-10-03T13:05:43.737Z|
|initial|49948|51|50139370.34|6560|2026-10-03T13:05:43.737Z|

- Explicación breve de por qué `COPY INTO` y `MERGE` resuelven problemas diferentes.

### COPY INTO
Carga archivos desde el origen a una tabla, llevado un registro de lo que cargó, por lo que no duplica archivos.
No analiza el contenido, por lo que si un mismo id llega en dos archivos distintos, el contenido se duplica.

### MERGE
Compara el origen contra el destino por claves para operaciones de inserción y actualización.
Hace deduplicación por clave, por lo que si se reprocesa un mismo archivo y la lógica es la correcta, no genera duplicados.

- Explicación del grano de cada tabla Gold.

La tabla gold esta a nivel dia, la granularidad seria esa.

- Las cuatro visualizaciones y una respuesta explícita para cada pregunta.
- Respuestas a las 20 preguntas de análisis y comprensión, incluyendo las consultas utilizadas cuando corresponda.

No incluyas datos, credenciales ni tokens.

