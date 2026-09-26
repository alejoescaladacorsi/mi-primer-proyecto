Resolución práctica 1 — Ingesta y capa Bronze

Nombre



Alejo Escalada Corsi



Student ID



alejo\_escalada1



Entrega breve

1\. Observaciones sobre los formatos



CSV y JSON: son formatos simples y flexibles para intercambio de datos, pero los tipos pueden llegar como texto o requerir inferencia y validación. En esta práctica, por ejemplo, amount quedó como string al leer el CSV sin inferencia de esquema.



Parquet: es un formato columnar que conserva información de tipos y resulta adecuado para trabajar eficientemente con datos analíticos. En la práctica, products pudo leerse directamente desde Parquet conservando su estructura.



Delta: agrega una capa de gestión sobre almacenamiento basado en archivos, permitiendo trabajar con tablas transaccionales, historial y operaciones trazables. Las tablas Bronze de esta práctica fueron almacenadas en formato Delta y se pudo consultar su historial mediante DESCRIBE HISTORY.



2\. Las cinco V en este caso



Volumen: aparece en la cantidad de datos generados: 5.000 customers, 500 products, 50.011 transactions y 200.000 events.



Velocidad: aparece en la necesidad de ingerir y procesar los datos mediante Spark, especialmente cuando aumenta la cantidad de eventos y transacciones.



Variedad: aparece en las distintas fuentes y formatos utilizados: CSV para customers y transactions, Parquet para products y JSON para events.



Veracidad: aparece en los controles de calidad realizados sobre los datos. En transactions se detectaron 50.011 filas, 50.000 IDs distintos y 52 valores de amount que no pudieron convertirse a DECIMAL(12,2).



Valor: aparece en transformar las fuentes originales en tablas Bronze trazables y reutilizables, que sirven como base para las siguientes etapas del Lakehouse.



Reflexión final



La práctica permitió recorrer el proceso inicial de un Lakehouse desde la generación de fuentes hasta la creación de tablas Bronze. Pude comprobar que el formato de origen influye en cómo Spark interpreta los datos y que la capa Bronze permite conservar las fuentes con metadatos de ingestión. También fue útil comprobar la calidad de los datos antes de avanzar, ya que aparecieron valores inválidos y diferencias entre cantidad de filas e IDs distintos. Finalmente, el uso de Delta permitió consultar el detalle y el historial de una tabla, aportando trazabilidad a la ingesta.

