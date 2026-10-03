# RappiPlus: de datos a decisiones de negocio

> Análisis de punta a punta del servicio RappiPlus: calidad de datos, rentabilidad, funnel de conversión, retención por cohortes, experimento A/B y dashboard en Power BI.

## Resumen

| Área | Hallazgo principal |
|---|---|
| **Calidad de datos** | Se corrigieron cantidades negativas, outliers extremos (10,000 y 20,000 unidades), 100 duplicados exactos e inconsistencias de formato en país y categoría |
| **Rentabilidad** | Revenue de **$9.64 M**, costo de producto de **$3.83 M** y marketing de **$2.87 M**: profit de **$2.94 M** (margen ≈ **30.5%**) |
| **Producto** | **Laptop-Gaming-16GB** tiene margen bruto negativo (costo unitario de $280.68) |
| **Funnel** | La mayor fuga está entre *begin_checkout* y *add_payment_info* (solo 86.71% avanza); conversión final de **80.04%** |
| **Retención** | Estable en torno a **40–44%** en los días 7, 14 y 21 para las 5 cohortes (enero a mayo) |
| **Experimento A/B** | El nuevo checkout convierte 16.29% vs 15.69%, pero la diferencia **no es significativa** (p = 0.416) |

## Preguntas de negocio

1. ¿Se puede confiar en los datos?
2. ¿Es rentable el negocio y qué productos o canales pesan más?
3. ¿En qué etapa del proceso de compra se pierden los usuarios?
4. ¿Los usuarios regresan después de registrarse?
5. ¿El nuevo diseño del checkout mejora la conversión?
6. ¿Cómo comunicar todo esto a un equipo directivo?

## Datos

| Fuente | Contenido | Tamaño |
|---|---|---|
| `rappiplus_orders_raw.csv` | Pedidos: producto, cantidad, precio, descuento y monto total, de enero a junio de 2025 | 25,100 filas |
| `rappiplus_catalog.csv` | Costo unitario, categoría y proveedor | 7 productos |
| `rappiplus_marketing_spend.csv` | Gasto diario por país y canal (organic, paid_search, social) | 1,620 filas |
| Tablas SQL `events`, `users`, `user_activity` | Eventos del funnel, registros y actividad de los usuarios | PostgreSQL |
| `experiment_checkout_ui.csv` | Resultados del experimento A/B del checkout | 10,000 usuarios |

> Los datos provienen de un proyecto de práctica del bootcamp [confirma la fuente]. Los CSV se cargan desde enlaces públicos dentro del notebook. Las tablas SQL requieren acceso a la base de datos del curso, por lo que **los resultados de las consultas quedan guardados en el notebook**.

## Metodología

1. **Calidad de datos (Python):** revisión de tipos, valores numéricos, duplicados, categorías y consistencia de montos (`cantidad × precio − descuento` vs `monto_total`).
2. **Rentabilidad (Python):** revenue, costo (cruce con catálogo), inversión en marketing, profit, ticket promedio y ventas por producto.
3. **Funnel de conversión (SQL):** usuarios únicos por evento, conversión paso a paso y conversión total respecto a la primera visita (CTE, `LAG`, `FIRST_VALUE`).
4. **Retención por cohortes (SQL):** cohorte por mes de registro y porcentaje de usuarios activos a los 7, 14 y 21 días.
5. **Experimento A/B (estadística):** prueba Z de proporciones para dos grupos (α = 0.05).
6. **Dashboard (Power BI):** resumen ejecutivo y detalle por producto.

## Resultados

### 1. Calidad de datos

| Problema | Acción |
|---|---|
| 4 pedidos con cantidad negativa (todos de Phone-Pro-128GB y sin país) | Eliminados |
| 10 pedidos con cantidades de 10,000 y 20,000 unidades (el percentil 99.9 es de 2) | Eliminados (límite de 100 unidades) |
| 100 filas duplicadas exactas | Eliminadas |
| País escrito de dos formas ("Mexico" y "mexico") | Homologado |
| "Electrónica" vs "Electronica" entre pedidos y catálogo | Homologado para poder cruzar tablas |
| 50 pedidos sin cantidad ni precio, pero con `monto_total` válido | Conservados para el análisis de revenue |
| 296 pedidos sin país | Marcados como "Desconocido" |

Resultado: **24,986 pedidos** limpios. La verificación de consistencia de montos no encontró diferencias mayores a 1 peso.

### 2. Rentabilidad

| KPI | Valor |
|---|---|
| Revenue | $9,643,910 |
| Costo de producto | $3,828,869 |
| Inversión en marketing | $2,871,844 |
| **Profit** | **$2,943,197** |
| Margen de rentabilidad | 30.5% |
| Ticket promedio | $385.97 |
| Unidades por pedido | 1.5 |

- **Unidades vendidas:** Vacuum-Pro-Black (6,284), Blender-XL-Red (6,279), Jacket-Winter-M (6,256) y Sneakers-Urban-42 (6,172) están prácticamente empatados. Los tres productos de electrónica venden bastante menos (≈ 4,100 a 4,200 unidades).
- **Marketing:** el gasto está repartido de forma pareja entre canales (social $918 K, organic $914 K, paid_search $863 K). Sin datos de atribución no se puede medir el retorno de cada canal.
- **Laptop-Gaming-16GB** es el único producto con margen bruto negativo.

### 3. Funnel de conversión

| Etapa | Usuarios | Conversión vs. paso anterior | Conversión total |
|---|---|---|---|
| first_visit | 7,796 | — | 100% |
| select_item | 7,582 | 97.26% | 97.26% |
| add_to_cart | 7,634 | 100.69% | 97.92% |
| begin_checkout | 7,208 | 94.42% | 92.46% |
| add_payment_info | 6,250 | **86.71%** | 80.17% |
| purchase | 6,240 | 99.84% | **80.04%** |

- **Mayor fuga:** entre `begin_checkout` y `add_payment_info`. Es la etapa que conviene revisar primero (formulario y métodos de pago).
- **Anomalía explicada:** `add_to_cart` supera a `select_item` (100.69%) porque 401 usuarios (5.25%) agregan al carrito sin registrar `select_item`; existe una ruta alterna que se salta ese evento.
- La conversión final de 80% es inusualmente alta, probablemente porque son usuarios que ya son clientes de la plataforma.

### 4. Retención por cohortes

| Cohorte | Usuarios | Semana 1 (día 7) | Semana 2 (día 14) | Semana 3 (día 21) |
|---|---|---|---|---|
| Enero 2025 | 1,627 | 42.84% | 41.06% | 40.32% |
| Febrero 2025 | 1,444 | 42.31% | 42.17% | 43.98% |
| Marzo 2025 | 1,636 | 41.38% | 43.09% | 42.18% |
| Abril 2025 | 1,606 | 42.34% | 43.40% | 41.28% |
| Mayo 2025 | 1,687 | 41.20% | 40.07% | 41.85% |

La retención es **estable (≈ 40–44%)** y no cae progresivamente. Como la actividad se mide en días puntuales (7, 14 y 21), el porcentaje no tiene por qué ser decreciente. Con cohortes de ≈ 1,600 usuarios, variaciones de 2 a 3 puntos están dentro del margen de error muestral, así que no se observan diferencias claras entre cohortes.

### 5. Experimento A/B del checkout

| Grupo | Conversión |
|---|---|
| Control | 15.69% |
| Tratamiento | 16.29% |

Prueba Z de proporciones: Z = −0.81, **p = 0.416** → no se rechaza H₀. No hay evidencia suficiente de que el nuevo diseño mejore la conversión. Esto no demuestra que sean iguales, sino que el experimento no tiene sensibilidad suficiente para detectar una diferencia tan pequeña.

### 6. Dashboard (Power BI)

Se construyeron dos páginas: un **resumen ejecutivo** (revenue, profit, gasto en marketing, ticket promedio y unidades por pedido) y un **detalle por producto** con indicador de rentabilidad (verde/rojo).

<!-- Agrega capturas del dashboard en /images, por ejemplo:
![Resumen ejecutivo](images/dashboard_resumen.png)
![Detalle por producto](images/dashboard_detalle.png)
-->

[Ver archivos del dashboard](https://drive.google.com/drive/folders/19FhZiG65pjKChRDKX-zGs1VtOcznf9Ng?usp=sharing)

## Recomendaciones

1. **Revisar el margen de Laptop-Gaming-16GB:** ajustar su precio, renegociar el costo con el proveedor o evaluar si conviene seguir vendiéndolo.
2. **Investigar la fricción en el pago** (`add_payment_info`): es el cuello de botella más grande del funnel.
3. **No implementar todavía el nuevo checkout:** el experimento no mostró una mejora significativa. Repetirlo con una muestra mayor (ver límites).
4. **Corregir el registro del evento `select_item`** para que el funnel refleje todas las rutas de compra.

## Limitaciones y próximos pasos

- **Experimento A/B:** con 10,000 usuarios, la diferencia observada (≈ 0.6 puntos) es demasiado pequeña para detectarla. Con una conversión cercana al 16%, harían falta unos **58,000 usuarios por grupo** para detectarla con 80% de potencia (estimación aproximada). Falta reportar el intervalo de confianza de la diferencia.
- **Costos y profit:** los pedidos sin cantidad o sin producto (≈ 80 filas) suman revenue pero no costo. Falta también confirmar que el periodo del marketing coincida con el de los pedidos y que los montos de los tres países estén en una moneda común.
- **Marketing:** 101 registros no tienen `canal`, por lo que el desglose por canal excluye ≈ $177 K del gasto total. El nombre del canal puede recuperarse de `id_campaña`.
- **Nulos pendientes:** categoría de producto (80 pedidos) y país (296 pedidos).
- **Funnel y retención:** el funnel cuenta usuarios únicos por evento sin verificar el orden temporal. La retención se mide en tres cortes puntuales y no incluye cohortes posteriores a mayo.
- **Dashboard:** no se implementó la navegación drill-through por producto.

## Estructura del repositorio

```
├── README.md
├── S12_Estudiante_Proyecto_Final.ipynb     # análisis completo (con resultados guardados)
├── images/                                  # capturas del dashboard y gráficas
└── dashboard/                               # archivo de Power BI (opcional)
```

## Cómo reproducirlo

1. Clona o descarga este repositorio.
2. Instala las dependencias:

```bash
pip install pandas numpy matplotlib statsmodels sqlalchemy psycopg2-binary
```

3. Los CSV se descargan automáticamente desde los enlaces del notebook.
4. Los pasos 3 y 4 (funnel y retención) consultan una base PostgreSQL del curso con credenciales privadas. Si no tienes acceso, consulta los **resultados ya guardados en el notebook**.
5. Abre el notebook y ejecuta las celdas en orden:

```bash
jupyter notebook S12_Estudiante_Proyecto_Final.ipynb
```

## Herramientas

Python 3 · pandas · numpy · matplotlib · statsmodels · SQL (PostgreSQL) · SQLAlchemy · Power BI · Jupyter Notebook

## Autor

**[Carlos Carrillo Aguayo]** · Data Analyst  
[LinkedIn](https://www.linkedin.com/in/carlosagca-data/) · [Más proyectos](https://github.com/carlosagca93-ui)
