# Entrega práctica 4 — Streaming

- **Nombre: Alejo Escalada**
- **student_id: alejo_escalada1**
- **Escala: small**

Respondé cada pregunta en una a tres oraciones. Cuando se pide un dato, copiá el valor que te dio el notebook.

## Log y offsets

**1.** Después de consumir tres mensajes, en la cola tradicional quedaron 2 mensajes, mientras que en el log permanecieron los 5 originales. La diferencia es que en una cola tradicional los mensajes se retiran al consumirlos, mientras que en un log como Kafka se conservan según su política de retención 

**2.** Después de la segunda llegada, el consumidor antifraude había leído 180 mensajes, mientras que reportes todavía no había procesado ninguno. El atraso de reportes no afectó a antifraude, ya que cada consumidor mantiene su propio offset y puede avanzar independientemente sobre el mismo log.

**3.** Un offset representa la posición de lectura de un consumidor dentro del log. En esta práctica, cada consumidor guarda su progreso en un checkpoint independiente, lo que permite retomar el procesamiento sin volver a leer los mensajes anteriores.

**4.** El replay procesó 280 mensajes, recuperando todo el historial del log. La arquitectura Kappa propone utilizar un único flujo de procesamiento y reprocesar los datos históricos mediante replay cuando sea necesario. En cambio, Lambda utiliza dos caminos separados: uno para el procesamiento batch y otro para el procesamiento en tiempo real.

## Ingesta y enriquecimiento

**5.** 
llegada	filas en el archivo	nuevas en Bronze	nuevas en Silver
stream_001	60	60	60
stream_002	120	120	120
stream_003	40	40	40
stream_004	40	40	40
stream_005	20	20	20
Bronze procesó 60, 120, 40, 40 y 20 filas nuevas en las cinco llegadas, respectivamente, acumulando 280 registros. Auto Loader utiliza un checkpoint para registrar los archivos procesados y evitar volver a ingerirlos. Por lo tanto, una reejecución sin archivos nuevos debería incorporar 0 filas.

**6.** source_batch_id	quality_reason	count
stream_002	INVALID_AMOUNT	20
stream_002	UNKNOWN_CUSTOMER	20
En la llegada stream_002 se identificaron 40 registros en cuarentena: 20 por INVALID_AMOUNT y 20 por UNKNOWN_CUSTOMER. Bronze conserva todos los registros, incluso los registros incorrectos.

**7.** El enriquecimiento agregó las columnas country, obtenida de silver_customers, y category, obtenida de silver_products. Se trata de un join stream-tabla, porque los eventos de compra se procesan incrementalmente y se cruzan con tablas de referencia para incorporar información adicional. Esto también permite identificar clientes inexistentes, como en el caso de bad_customer_000.

## Tiempo y ventanas

**8.** El event time representa el momento en que ocurrió un evento, mientras que el processing time representa cuándo fue procesado por el sistema. Por ejemplo, late_ok tiene un event time de 12:03, pero ingresó durante la segunda llegada (stream_002), después de otros eventos con horarios posteriores.

**9.**
llegada	lote	máximo event time visto	watermark al terminar
1	stream_001	12:04	11:54
2	stream_002	12:08	11:58
3	stream_003	12:26	12:16
4	stream_004	12:27	12:17
5	stream_005	12:45	12:35
El calculo es el maximo event time -10 minutos

**10.** Las primeras ventanas tumbling se emitieron en stream_003, cuando el watermark avanzó hasta las 12:16, generando cuatro filas de resultados. En las llegadas anteriores no se publicaron porque el watermark todavía no había superado el final de las ventanas. En modo append, Spark espera a que las ventanas se consideren cerradas antes de emitir sus resultados definitivos.

**11.** El evento late_ok fue aceptado porque ingresó en stream_002 con un event time de 12:03, cuando todavía se encontraba dentro del límite permitido por el watermark. En cambio, late_bad llegó en stream_004 con un event time de 12:02, cuando el watermark ya había avanzado hasta las 12:16. Por lo tanto, sus 20 registros fueron descartados por llegar demasiado tarde.

**12.** Las ventanas tumbling agrupan eventos en intervalos fijos sin superposición, por ejemplo, ventas cada 5 minutos. Las ventanas hopping utilizan intervalos superpuestos, como ventanas de 10 minutos que avanzan cada 5, permitiendo calcular ventas móviles. Las ventanas session agrupan eventos según la actividad de cada cliente y se cierran después de un período de inactividad, por ejemplo, 5 minutos sin compras.

**13.** Para la ventana [12:00, 12:05) del canal card, el modo update emitió dos resultados: 40 compras por 6.000 en la llegada 1 y 60 compras por 8.400 en la llegada 2. El modo append emitió una sola vez, en la llegada 3, con 60 compras por 8.400. La ventaja de update es ofrecer resultados tempranos, aunque pueden modificarse; append entrega resultados definitivos, pero requiere esperar al cierre de la ventana mediante el watermark.

**14.** Reducir el watermark de 10 minutos a 1 minuto disminuye la tolerancia a eventos tardíos, aumentando la posibilidad de descartar información válida que llegue con retraso. Como ventaja, permite cerrar ventanas antes y liberar más rápidamente el estado almacenado, reduciendo potencialmente el consumo de memoria y recursos computacionales.

## Garantías y fallas

**15.** Sin deduplicación quedaron 220 filas, mientras que al deduplicar por event_id quedaron 200. La diferencia corresponde a 20 eventos duplicados. Un productor puede reenviar mensajes cuando no recibe confirmación de entrega o necesita reintentar una operación, generando duplicados característicos de sistemas con garantía at-least-once.

**16.** Al reiniciar con el mismo checkpoint, el destino permaneció en 200 filas porque Spark conservó el progreso del procesamiento. Al perder el checkpoint y utilizar una escritura append, se reprocesaron los eventos y el destino aumentó a 400 filas. Esto evidencia el riesgo de duplicación asociado al reprocesamiento con una escritura no idempotente, propio de un comportamiento at least once.

**17.** La escritura con MERGE mantuvo 200 filas incluso después de perder el checkpoint, porque utiliza event_id para insertar únicamente los eventos que todavía no existen en el destino. Esto representa una escritura idempotente: ejecutar la misma operación varias veces produce el mismo resultado que ejecutarla una sola vez. Combinada con checkpoints y una clave única adecuada, permite obtener resultados efectivos de tipo exactly once.

## CDC

**18.** El log CDC generó 13 eventos: 6 insert, 3 update_preimage, 3 update_postimage y 1 delete. Cada actualización produce dos eventos porque registra el estado anterior y posterior del registro, permitiendo reconstruir los cambios realizados.

**19.** Sí, fue posible reconstruir la tabla original a partir del log CDC, obteniendo los mismos cinco registros actuales. El log conserva información histórica sobre inserciones, modificaciones y eliminaciones, incluyendo valores anteriores y posteriores a cada actualización. Por ejemplo, permite identificar que el cliente 2 pasó de UY a CL y finalmente a BR, mientras que la tabla actual únicamente muestra BR.

## Cierre

**20.**
