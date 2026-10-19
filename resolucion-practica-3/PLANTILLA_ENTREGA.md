# Entrega práctica 3 — Bases de datos NoSQL

- **Nombre:** Alejo Escalada 
- **student_id:** alejo_escalada1
- **Escala:** small

Respondé cada pregunta en una a tres oraciones. Cuando se pide un dato, copiá el valor que te dio el notebook.

## CAP y PACELC

**1.** En modo CP prefiere dar error antes que devolver un dato viejo, mientras que en la version AP prefiere responder aunque sea con un dato desactualizado. En el primero sacrifica disponibilidad, es decir, prefiere responder error antes que dar un dato que no es. En el segundo caso prioriza la disponibilidad antes que la consistencia, no importa si el dato no es el correcto.



**2.** Quedo el saldo de 50 bajo el modo AP. Eventualmente se llega a esa consistencia. Pero se perdio la escritura de saldo de 80.

**3.** En el caso de W=1 el p99 se obtiene en 10 ms, mientras que para W=3 se obtienen en mas de 50 ms. En definitiva, se obtienen resultados mas consistentes pero se demora mas tiempo en confirmar, es decir, se requiere mas latencia

**4.** El grupo ABC pudo seguir escribiendo porque contaba con 3 de las 5 réplicas, alcanzando la mayoría necesaria para establecer el quórum. En cambio, DE tenía solamente 2 réplicas y no podía realizar escrituras.

## Clave-valor

**5.** El GET en memoria fue aproximadamente 946.395 veces más rápido que el GET con Spark sobre Delta. Redis almacena los datos principalmente en memoria RAM para permitir accesos directos por clave, evitando gran parte de la sobrecarga de procesamiento y logrando tiempos de respuesta muy bajos.

Clave-valor (Redis): cuando conocés exactamente qué dato necesitás y querés recuperarlo lo más rápido posible. Spark: cuando necesitás analizar muchos registros, aplicar filtros, agrupar información o calcular estadísticas.

**6.** Para contar cuántos clientes pertenecían a Argentina (AR), fue necesario recorrer todas las claves y analizar sus valores, ya que el país estaba almacenado dentro del JSON y no era la clave de búsqueda. Una de las ventajas de clave valor es la rapidez que tenes para buscar un dato especifico, pero cuando se trata de analizar informacion masiva, se vuelve complejo y poco eficiente

**7.** En el primer caso, muchas de las claves se mueve sin ser necesario.. Por ejemplo, si tuviese 100 productos Producto Con 4 nodos Con 5 nodos 101 Nodo 1 Nodo 1 102 Nodo 2 Nodo 2 103 Nodo 3 Nodo 3 104 Nodo 0 Nodo 4 105 Nodo 1 Nodo 0 106 Nodo 2 Nodo 1

Casi un 80% de los datos termina moviendose

En cambio con hashing consistente, este trabajo es menor, y no alcanza el 22% del movimiento de los datos. El segundo método es más conveniente porque permite agregar nuevos servidores sin tener que redistribuir la mayoría de los datos, reduciendo el tiempo y los recursos necesarios para reorganizarlos.

## Documental

**8.** En el primer caso devuelve 11 filas, mientras que por el otro lado devuelve un solo documento. Los datos que quedaron embebidos fueron esas 11 transacciones, junto con el perfil, etc.

**9.** Spark asignó valores null a los campos que no estaban presentes en algunos documentos, para poder representarlos dentro de un mismo DataFrame. Se dice que el modelo documental tiene un esquema flexible porque permite almacenar documentos con diferentes campos y estructuras sin exigir que todos tengan exactamente el mismo formato.

**10.** Documento más grande: 5.2 KB · ≈225 bytes por transacción A ese ritmo, un cliente llegaría a 16 MB con ≈74,599 transacciones MongoDB limita un documento a 16 MB.

**11.** El modelo documental resulta eficiente para consultar información relacionada dentro de un mismo documento, como el perfil y las transacciones de un cliente. Sin embargo, las consultas que requieren analizar o agrupar información de muchos documentos, como calcular el monto vendido por categoría, son más costosas porque es necesario recorrer y descomponer los datos embebidos mediante operaciones como explode.

## Grafos

**12.** label nodos Device 2361 Customer 5000 5,400 aristas USES

**13.**2 Saltos 3 Joins 4 Saltos 5 Joins ¿Por qué Neo4j puede ser más eficiente para estos recorridos? Porque almacena referencias a los vecinos de cada nodo, facilitando seguir relaciones sin reconstruirlas mediante joins generales.

**14.**7 iteraciones. El componente mas grande tiene 23 nodos.

**15.**Al distribuir los nodos con hash(id) mod 4, el 74,4 % de las aristas quedó entre máquinas distintas. Esto dificulta la distribución de una base de grafos porque recorrer relaciones entre diferentes servidores requiere comunicación por red, aumentando la latencia y el consumo de recursos. Al distribuir por componentes conexos, el porcentaje se redujo al 0 % en esta simulación.

## Vectorial

**16.**El vector representa el comportamiento de compra de cada cliente mediante 10 valores: cinco relacionados con las categorías de gasto, tres con los medios de pago y dos con los importes de sus compras. La similitud coseno mide qué tan parecida es la orientación de dos vectores, permitiendo identificar clientes con patrones de comportamiento similares.

**17.**Entre los vecinos de clientes marcados, el 31,6 % también estaba marcado, mientras que la tasa base de toda la población era del 14,8 %. Esto demuestra que los clientes marcados presentan comportamientos similares entre sí, con una concentración aproximadamente 2,14 veces mayor. Buscar vecinos parecidos permite identificar patrones de comportamiento y detectar posibles casos sospechosos para analizarlos.

**18.**Con nprobe=4 se obtuvo un recall aproximado del 87 %, escaneando solamente alrededor del 6 % de los vectores. La búsqueda aproximada (ANN) permite reducir considerablemente el costo computacional y acelerar las consultas, aunque puede perder algunos vecinos que sí encontraría una búsqueda exacta.

## Columnar

**19.**

**20.** El formato columnar es eficiente para analítica porque permite leer únicamente las columnas necesarias, reduciendo el volumen de datos procesados y aprovechando una mejor compresión. Sin embargo, es menos conveniente para modificar una fila individual, ya que los datos están organizados por columnas y una actualización puede requerir reescribir archivos o bloques de información, aumentando el costo computacional.
