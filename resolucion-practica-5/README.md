# Entrega práctica 5 — MLOps con MLflow

- **Nombre:** Alejo Escalada

- **student_id:** alejo_escalada1

- **Escala:** small

Respondé cada pregunta en una a tres oraciones. Cuando se pide un dato, copiá el valor que te dieron tus notebooks.

## Mis runs

| Run | Modelo | Features | Class weight | Umbral | Hiperparámetros | Train PR-AUC | Test PR-AUC | Accuracy | Precision | Recall | F1 |
|---|---|---|---|---:|---|---:|---:|---:|---:|---:|---:|
| max_depth=3 | gradient_boosting | completas | balanced | 0.4 | learning_rate=0.1, max_depth=3 | .602 | .586 | .757 | .282 | .868 | .426 |
| learning_rate=0.03 | gradient_boosting | completas | ninguno | 0.3 | learning_rate=0.03, max_depth=8 | .640 | .585 | .906 | .546 | .580 | .563 |
| gradient_boosting-completas-u0.3 | gradient_boosting | completas | ninguno | 0.3 | learning_rate=0.03, max_depth=8 | .640 | .585 | .906 | .546 | .580 | .563 |
| gradient_boosting-completas-u0.3 | gradient_boosting | completas | ninguno | 0.3 | learning_rate=0.03, max_depth=8 | .640 | .585 | .906 | .546 | .580 | .563 |
| umbral=0.2 | gradient_boosting | completas | ninguno | 0.2 | learning_rate=0.1, max_depth=5 | .669 | .582 | .884 | .460 | .678 | .548 |
| umbral=0.3 | gradient_boosting | completas | ninguno | 0.3 | learning_rate=0.1, max_depth=5 | .669 | .582 | .908 | .554 | .569 | .561 |
| umbral=0.4 | gradient_boosting | completas | ninguno | 0.4 | learning_rate=0.1, max_depth=5 | .669 | .582 | .916 | .636 | .445 | .523 |
| umbral=0.5 | gradient_boosting | completas | ninguno | 0.5 | learning_rate=0.1, max_depth=5 | .669 | .582 | .918 | .709 | .353 | .471 |
| gradient_boosting-completas-u0.5 | gradient_boosting | completas | ninguno | 0.5 | learning_rate=0.1, max_depth=5 | .669 | .582 | .918 | .709 | .353 | .471 |
| learning_rate=0.1 | gradient_boosting | completas | ninguno | 0.3 | learning_rate=0.1, max_depth=8 | .660 | .581 | .907 | .551 | .574 | .563 |
| max_depth=3 | gradient_boosting | completas | balanced | 0.4 | learning_rate=0.3, max_depth=3 | .598 | .579 | .758 | .282 | .864 | .426 |
| umbral=0.2 | gradient_boosting | completas | balanced | 0.2 | learning_rate=0.1, max_depth=5 | .614 | .579 | .599 | .198 | .939 | .327 |
| umbral=0.3 | gradient_boosting | completas | balanced | 0.3 | learning_rate=0.1, max_depth=5 | .614 | .579 | .684 | .235 | .908 | .374 |
| umbral=0.4 | gradient_boosting | completas | balanced | 0.4 | learning_rate=0.1, max_depth=5 | .614 | .579 | .753 | .278 | .861 | .420 |
| umbral=0.5 | gradient_boosting | completas | balanced | 0.5 | learning_rate=0.1, max_depth=5 | .614 | .579 | .802 | .320 | .800 | .457 |

**Run registrado como `champion`:** `max_depth=3` (Gradient Boosting, features `completas`, `learning_rate=0.1`, `max_depth=3`, `balanced`, umbral `0.4`), PR-AUC test **0.586**. Modelo: `workspace.bigdata_alejo_escalada1.fraud_model`, versión **3**.

## Datos y tracking

**1.** La tabla `ml_features` contiene **49.948 registros**: train **35.654** (11,05% de contracargos), test **7.129** (10,38%) y nuevas **7.165** (11,05%). La separación temporal respeta el orden cronológico, evita fuga de información futura y simula la evaluación en producción.

**2.** Predecir siempre «sin contracargo» daría **89,62% de accuracy** en test, pero **0% de recall** para los contracargos. Por el desbalance de clases, **PR-AUC** es más útil porque evalúa precision y recall a través de distintos umbrales.

**3.** En `01_manual_logistica` registré parámetros (modelo, `basicas`, `C=1.0`, `class_weight=ninguno`, umbral `0.5` y datos), métricas (PR-AUC, accuracy, precision, recall, F1 y PR-AUC train) y artefactos (`matriz_confusion.png` y modelo con signature). Un **experimento** agrupa ejecuciones; un **run** representa una ejecución con su configuración y resultados.

**4.** MLflow autolog registró **46 parámetros** del pipeline y Random Forest, y **7 métricas de entrenamiento**: accuracy, F1, log loss, precision, recall, ROC-AUC y score. Las métricas de test como `test_pr_auc`, el umbral y las alertas se registraron manualmente porque son específicas del problema.

**5.** Los runs `01_repro_a` y `01_repro_b` reprodujeron las mismas métricas: **PR-AUC test 0,510** y **accuracy 0,909**. Para reproducir un experimento se deben mantener datos y versión, código, semilla (`random_state=42`) y entorno/dependencias.

**6.** Al eliminar la columna `amount`, la predicción falló con **“Failed to enforce schema of data”**. La firma (*signature*) valida columnas y tipos de entrada y salida para evitar inferencias con datos incompatibles.

## Tus experimentos

**7.** El mejor run fue **`max_depth=3`**, Gradient Boosting con features `completas`, `balanced`, `learning_rate=0.1`, profundidad `3` y umbral `0.4`: **PR-AUC test 0,586** (train 0,602). Frente a `solo_regla` (**0,170**), mejoró **0,416 puntos**.

**8.** Con Gradient Boosting, el PR-AUC test fue **0,328** con `basicas`, **0,514** con `cliente` y **0,579** con `completas`. Las variables más importantes fueron `amount` (**0,3265**), `payment_channel` (**0,1750**) y `hour` (**0,1028**), lo que indica valor predictivo sin demostrar causalidad.

**9.** Con features `completas`, el PR-AUC test fue **0,540** para regresión logística, **0,537** para Random Forest y **0,579** para Gradient Boosting, el mejor de los tres. Los árboles capturan relaciones no lineales e interacciones sin definirlas manualmente, a diferencia de una regresión logística sin términos adicionales.

**10.** Con Gradient Boosting, features `completas`, `learning_rate=0.1`, `max_depth=5` y umbral `0.5`, `balanced` obtuvo **PR-AUC 0,579**, precision **0,320** y recall **0,800**, frente a **0,582**, **0,709** y **0,353** con `ninguno`. El balanceo detecta más contracargos, pero genera más falsos positivos; la elección depende del costo de errores y la capacidad de revisión.

**11.** Al bajar el umbral de **0,5 a 0,3**, la precision pasó de **0,709 a 0,554**, el recall de **0,353 a 0,569** y el F1 de **0,471 a 0,561**. El **PR-AUC permaneció en 0,582** porque usa scores a través de umbrales, mientras que F1 depende del umbral elegido.

**12.** En el barrido de `max_depth`, el mejor resultado fue **3** (PR-AUC test **0,586**, train **0,602**), frente a **0,579** con profundidad 5 y **0,577** con 8. En el barrido de `learning_rate`, **0.03** obtuvo el mejor PR-AUC test (**0,585**; train **0,640**), frente a **0,581** con 0.1 y **0,567** con 0.3; las brechas train/test sugieren posible sobreajuste al aumentar la complejidad.

## Comparar y elegir

**13.** La accuracy puede engañar con clases desbalanceadas: predecir siempre «sin contracargo» daría **89,62%** sin detectar ninguno. El mejor run por PR-AUC obtuvo **0,586** y accuracy **75,7%**, mientras que otro alcanzó **91,8% de accuracy** pero PR-AUC **0,582** y recall **35,3%**, por lo que prioricé PR-AUC.

**14.** El run `max_depth=3` tuvo PR-AUC **0,602 train / 0,586 test** (brecha **0,016**), frente a `learning_rate=0.03`, depth 8 (**0,640 / 0,585**, brecha **0,055**) y `learning_rate=0.1`, depth 8 (**0,660 / 0,581**, brecha **0,079**). Las brechas mayores son indicios de posible sobreajuste, no de mejor generalización.

**15.** Para buscar runs de Random Forest con PR-AUC test superior a **0,55**, usaría `mlflow.search_runs(experiment_ids=[experiment_id], filter_string="params.modelo = 'random_forest' AND metrics.test_pr_auc > 0.55", order_by=["metrics.test_pr_auc DESC"])`. Las claves del filtro deben coincidir con las registradas en MLflow.

**16.** Los **falsos positivos** consumen capacidad de revisión y los **falsos negativos** dejan contracargos sin detectar. Al bajar el umbral de **0,5 a 0,2**, el recall subió de **35,3% a 67,8%**, pero la precision bajó de **70,9% a 46%**; elegiría el umbral según costos y capacidad operativa.

## Registro, uso y monitoreo

**17.** Registré `workspace.bigdata_alejo_escalada1.fraud_model`, **versión 3**, con alias **`champion`**, features `completas` y umbral **0.4**, sin errores; se carga como `models:/workspace.bigdata_alejo_escalada1.fraud_model@champion`. `runs:/` referencia artefactos de un run específico, mientras que `models:/` referencia versiones o alias de un modelo registrado.

**18.** Se evaluaron **7.165 transacciones nuevas** con el modelo versión **3** (`champion`) y umbral **0.4**, generando **2.278 alertas**. La salida incluye `transaction_id`, `score`, `alerta`, `threshold`, `model_name`, `model_version`, `run_id` y `scored_at` para auditoría; el umbral se define fuera del modelo para ajustarlo sin reentrenar.

**19.** El PR-AUC pasó de **0,586 en test** a **0,579 en nuevas**, manteniendo un rendimiento similar. En la campaña simulada se observó drift: las alertas subieron del **31,8% al 50,1%**, lo que puede detectarse monitoreando datos y scores antes de conocer los contracargos reales.

## Cierre

**20.** La práctica recorrió el ciclo MLOps: preparación de datos, entrenamiento, tracking en MLflow, comparación, selección, registro, scoring y monitoreo. Para producción real faltaría automatizar pipelines, monitorear drift y rendimiento, gestionar alertas y definir mecanismos de reentrenamiento.
