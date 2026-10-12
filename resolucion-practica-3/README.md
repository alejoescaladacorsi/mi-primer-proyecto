
# Clase práctica 3 — Bases de datos NoSQL: el mismo dato, cinco modelos

## Propósito

Tomar los clientes, productos y transacciones Silver de la clase 2 y reorganizarlos según los modelos NoSQL vistos en la teoría: clave-valor, documental, grafos, vectorial y columnar. Antes de eso se simula el problema común a todos al distribuirse: CAP y PACELC.

La práctica es **demostrativa y visual**. Databricks no es Redis, MongoDB, Neo4j ni Pinecone: cada modelo se simula con Spark, tablas Delta y Python para observar su idea central (cómo se guarda, cómo se consulta, qué cuesta distribuirlo) sin instalar motores externos.

## Objetivos

- Explicar con un ejemplo concreto qué se pierde ante una partición en un sistema CP y en uno AP, y el trade-off latencia/consistencia de PACELC.
- Reconocer el acceso por clave de un almacén clave-valor y el rol de la función hash dentro de un nodo y entre nodos.
- Diseñar un documento a partir de un patrón de lectura: qué se embebe y qué se referencia.
- Expresar un patrón de grafo en Cypher y en SQL, y entender por qué cada salto es costoso de distribuir.
- Buscar vecinos por similitud y entender el trade-off recall/costo de un índice ANN.
- Relacionar Parquet y Delta con el almacenamiento columnar: column pruning, compresión y data skipping.

## Requisitos previos

- Clases 1 y 2 ejecutadas en el mismo catálogo y esquema personal: se usan `silver_customers`, `silver_products` y `silver_transactions`.
- Mismo `student_id` y misma escala que en la clase 2.
- Databricks Free Edition con compute serverless. No hay que instalar librerías: se usan Spark, NumPy, pandas, matplotlib y pyarrow incluidos en la plataforma.
- Para importación manual, conservar `clase-03/` y `common/` como carpetas hermanas.

## Secuencia

Los notebooks son **demostrativos**: no hay que modificar ni completar código. Se ejecutan en orden, celda por celda, leyendo el texto y mirando cada resultado y gráfico. Las preguntas de la entrega se responden con lo que muestran.

| Notebook | Modelo | Qué se ve | Minutos |
|---|---|---|---:|
| `00_preflight.ipynb` | Relacional | Verifica las tablas Silver y muestra el punto de partida | 5 |
| `01_cap_pacelc.ipynb` | Distribución | Réplicas con una partición, modos CP y AP; latencia vs. `W`; quórum `W+R>N` | 15 |
| `02_clave_valor.ipynb` | Clave-valor | Perfiles `customer:<id>`; GET en memoria vs. Spark; hash mod N vs. hashing consistente | 20 |
| `03_documental.ipynb` | Documental | Un documento por cliente con sus transacciones embebidas; consultas anidadas; esquema flexible; tamaño | 20 |
| `04_grafos.ipynb` | Grafos | Clientes y dispositivos; Cypher vs. SQL; componentes conexos; dibujo; particionamiento | 20 |
| `05_vectorial.ipynb` | Vectorial | Vectores de comportamiento; PCA; k-NN exacto; índice IVF y curva recall/costo | 20 |
| `06_columnar.ipynb` | Columnar | Fila vs. columna; CSV/JSON/Parquet; plan físico; footer Parquet; data skipping | 15 |


## Tablas y archivos que se crean

```text
nosql_kv_customer_profile        (key, value JSON)
nosql_customer_documents         (_id, profile, stats, transactions[])
nosql_graph_nodes / nosql_graph_edges / nosql_graph_components
nosql_graph_cc_a / nosql_graph_cc_b   (auxiliares de la propagación de etiquetas)
nosql_customer_vectors           (customer_id, embedding[], flagged)
landing/nosql_lab/customer_documents/   JSON Lines
landing/nosql_lab/flexible_documents/   JSON con esquemas distintos
landing/nosql_lab/columnar/             CSV, JSON y Parquet de las transacciones
```

Ninguna tabla de las clases 1 y 2 se modifica. Todos los notebooks pueden reejecutarse: sobrescriben sus propias tablas y archivos.

## Preguntas

Las preguntas son cortas: buscan comprobar que entendiste la idea de cada modelo y sus ventajas y desventajas. Cuando piden un dato, copiá el número que te dio el notebook (puede variar un poco según tu escala). Alcanza con una a tres oraciones por respuesta.

### CAP y PACELC (`01_cap_pacelc`)

1. Con la red partida, ¿qué respondió la réplica `C` en modo CP y qué respondió en modo AP? ¿Qué propiedad de CAP sacrifica cada modo?

En modo CP prefiere dar error antes que devolver un dato viejo, mientras que en la version AP prefiere responder aunque sea con un dato desactualizado. En el primero sacrifica disponibilidad, es decir, prefiere responder error antes que dar un dato que no es. En el segundo caso prioriza la disponibilidad antes que la consistencia, no importa si el dato no es el correcto.

2. En modo AP, ¿qué saldo quedó en las tres réplicas después de la reparación y qué escritura se perdió?

Quedo el saldo de 50 bajo el modo AP. Eventualmente se llega a esa consistencia. Pero se perdio la escritura de saldo de 80.

3. Según el gráfico de latencia, ¿cuál es la p99 con `W=1` y con `W=3`? ¿Qué se gana a cambio de esperar más réplicas? (PACELC)

En el caso de W=1 el p99 se obtiene en 10 ms, mientras que para W=3 se obtienen en mas de 50 ms. En definitiva, se obtienen resultados mas consistentes pero se demora mas tiempo en confirmar, es decir, se requiere mas latencia

4. Con cinco réplicas partidas en `ABC | DE`, ¿qué lado pudo seguir escribiendo en modo CP y por qué?

El grupo ABC pudo seguir escribiendo porque contaba con 3 de las 5 réplicas, alcanzando la mayoría necesaria para establecer el quórum. En cambio, DE tenía solamente 2 réplicas y no podía realizar escrituras.

### Clave-valor (`02_clave_valor`)

5. ¿Cuántas veces más rápido fue el GET en memoria que el GET con Spark? ¿Por qué Redis guarda los datos en memoria?

El GET en memoria fue aproximadamente 946.395 veces más rápido que el GET con Spark sobre Delta. Redis almacena los datos principalmente en memoria RAM para permitir accesos directos por clave, evitando gran parte de la sobrecarga de procesamiento y logrando tiempos de respuesta muy bajos.

Clave-valor (Redis): cuando conocés exactamente qué dato necesitás y querés recuperarlo lo más rápido posible.
Spark: cuando necesitás analizar muchos registros, aplicar filtros, agrupar información o calcular estadísticas.

6. ¿Por qué para contar los clientes de AR hubo que leer y parsear todos los valores? Mencioná una ventaja y una desventaja del modelo clave-valor.

Para contar cuántos clientes pertenecían a Argentina (AR), fue necesario recorrer todas las claves y analizar sus valores, ya que el país estaba almacenado dentro del JSON y no era la clave de búsqueda. Una de las ventajas de clave valor es la rapidez que tenes para buscar un dato especifico, pero cuando se trata de analizar informacion masiva, se vuelve complejo y poco eficiente

7. Al pasar de 4 a 5 nodos, ¿qué porcentaje de claves se movió con `hash mod N` y con hashing consistente? ¿Por qué conviene el segundo?

En el primer caso, muchas de las claves se mueve sin ser necesario.. Por ejemplo, si tuviese 100 productos
Producto	Con 4 nodos	Con 5 nodos
101	        Nodo 1	    Nodo 1
102	        Nodo 2	    Nodo 2
103	        Nodo 3	    Nodo 3
104	        Nodo 0	    Nodo 4
105     	Nodo 1	    Nodo 0
106     	Nodo 2	    Nodo 1

Casi un 80% de los datos termina moviendose 

En cambio con hashing consistente, este trabajo es menor, y no alcanza el 22% del movimiento de los datos. El segundo método es más conveniente porque permite agregar nuevos servidores sin tener que redistribuir la mayoría de los datos, reduciendo el tiempo y los recursos necesarios para reorganizarlos.

### Documental (`03_documental`)

8. ¿Cuántas filas devolvió la consulta relacional del cliente 42 y cuántos documentos la documental? ¿Qué datos quedaron embebidos dentro del documento?

En el primer caso devuelve 11 filas, mientras que por el otro lado devuelve un solo documento. Los datos que quedaron embebidos fueron esas 11 transacciones, junto con el perfil, etc.

9. Al leer documentos con campos distintos, ¿qué hizo Spark con los campos que faltaban en algunos? ¿Por qué se dice que el modelo documental tiene esquema flexible?

Spark asignó valores null a los campos que no estaban presentes en algunos documentos, para poder representarlos dentro de un mismo DataFrame. Se dice que el modelo documental tiene un esquema flexible porque permite almacenar documentos con diferentes campos y estructuras sin exigir que todos tengan exactamente el mismo formato.

10. ¿Cuánto pesa el documento más grande? ¿Qué problema aparece si un documento crece sin límite?

Documento más grande: 5.2 KB · ≈225 bytes por transacción
A ese ritmo, un cliente llegaría a 16 MB con ≈74,599 transacciones
MongoDB limita un documento a 16 MB.

11. Para calcular el monto por categoría hubo que usar `explode`. ¿Qué tipo de consultas resuelve bien el modelo documental y cuáles le cuestan más?

El modelo documental resulta eficiente para consultar información relacionada dentro de un mismo documento, como el perfil y las transacciones de un cliente. Sin embargo, las consultas que requieren analizar o agrupar información de muchos documentos, como calcular el monto vendido por categoría, son más costosas porque es necesario recorrer y descomponer los datos embebidos mediante operaciones como explode.

### Grafos (`04_grafos`)

12. ¿Cuántos nodos `Customer`, nodos `Device` y aristas `USES` tiene el grafo?

label	nodos
Device	2361
Customer	5000
5,400 aristas USES  

13. ¿Cuántos joins necesitó SQL para el patrón de 2 saltos y para el de 4? ¿Por qué Neo4j recorre relaciones más eficientemente que una base relacional?

2 Saltos 3 Joins
4 Saltos 5 Joins
¿Por qué Neo4j puede ser más eficiente para estos recorridos? Porque almacena referencias a los vecinos de cada nodo, facilitando seguir relaciones sin reconstruirlas mediante joins generales.

14. ¿Cuántas iteraciones tardó en converger la búsqueda de componentes conexos y cuántos nodos tiene el componente más grande?

7 iteraciones. El componente mas grande tiene 23 nodos.

15. ¿Qué porcentaje de aristas quedó entre máquinas distintas al repartir los nodos con `hash(id) mod 4`? ¿Por qué es difícil distribuir una base de grafos?

Al distribuir los nodos con hash(id) mod 4, el 74,4 % de las aristas quedó entre máquinas distintas. Esto dificulta la distribución de una base de grafos porque recorrer relaciones entre diferentes servidores requiere comunicación por red, aumentando la latencia y el consumo de recursos. Al distribuir por componentes conexos, el porcentaje se redujo al 0 % en esta simulación.

### Vectorial (`05_vectorial`)

16. ¿Qué representa el vector de cada cliente y qué mide la similitud coseno?

El vector representa el comportamiento de compra de cada cliente mediante 10 valores: cinco relacionados con las categorías de gasto, tres con los medios de pago y dos con los importes de sus compras. La similitud coseno mide qué tan parecida es la orientación de dos vectores, permitiendo identificar clientes con patrones de comportamiento similares.

17. ¿Qué porcentaje de clientes marcados hay entre los vecinos de clientes marcados y cuál es la tasa base? ¿Para qué sirve buscar "vecinos parecidos"?

Entre los vecinos de clientes marcados, el 31,6 % también estaba marcado, mientras que la tasa base de toda la población era del 14,8 %. Esto demuestra que los clientes marcados presentan comportamientos similares entre sí, con una concentración aproximadamente 2,14 veces mayor. Buscar vecinos parecidos permite identificar patrones de comportamiento y detectar posibles casos sospechosos para analizarlos.

18. En la curva del índice IVF, ¿qué recall y qué porcentaje de vectores escaneados se obtienen con `nprobe=4`? ¿Qué se gana y qué se pierde con la búsqueda aproximada (ANN)?

Con nprobe=4 se obtuvo un recall aproximado del 87 %, escaneando solamente alrededor del 6 % de los vectores. La búsqueda aproximada (ANN) permite reducir considerablemente el costo computacional y acelerar las consultas, aunque puede perder algunos vecinos que sí encontraría una búsqueda exacta.

### Columnar (`06_columnar`)

19. ¿Cuánto ocupan los datos en CSV, JSON y Parquet? ¿Qué columnas leyó Spark para la consulta de `amount` sobre Parquet (`ReadSchema`)? ¿Cuántos archivos se pueden saltear con los datos ordenados?

![image_1791775762729.png](./image_1791775762729.png "image_1791775762729.png")

 Esto demuestra la eficiencia del almacenamiento columnar y de sus mecanismos de compresión. 

20. ¿Por qué el formato columnar es bueno para analítica y poco conveniente para modificar una fila por vez?

El formato columnar es eficiente para analítica porque permite leer únicamente las columnas necesarias, reduciendo el volumen de datos procesados y aprovechando una mejor compresión. Sin embargo, es menos conveniente para modificar una fila individual, ya que los datos están organizados por columnas y una actualización puede requerir reescribir archivos o bloques de información, aumentando el costo computacional.

## Entrega

La publicación, el acceso público y el envío a los profesores siguen el [formato común de entrega](../README.md#formato-común-de-entrega).

La entrega es **un único `README.md`** con las respuestas a las 20 preguntas, dentro de `resolucion-practica-3/` en tu repositorio personal:

```text
resolucion-practica-3/
└── README.md
```

El README debe incluir nombre, `student_id` y escala. Podés partir de la [plantilla](PLANTILLA_ENTREGA.md). No hace falta exportar los notebooks.

No incluyas datos generados, archivos del volumen, credenciales ni tokens.
