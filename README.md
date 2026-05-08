# adventureworks-rentabilidad-de-inversion

Este proyecto analiza el desempeño financiero de Adventure Works utilizando SQL y Google Sheets, con el objetivo de identificar los mercados más rentables y optimizar la inversión en marketing. Se trabajó con datos de ventas, productos, clientes, territorios y campañas para calcular indicadores clave como ingresos, costos, beneficio bruto, margen y ROI.


###Para el análisis se utilizaron los siguientes tablas: 

•	ventas_2017: transacciones de líneas de pedido (2017). Grano: una línea por producto y pedido.
•	productos: catálogo con atributos, costo y precio unitario por ClaveProducto.
•	productos_categorias: jerarquía categoría/subcategoría para enriquecer productos.
•	clientes: maestro de clientes con segmento y ubicación.
•	territorios: mapa de ClaveTerritorio → país y continente.
•	campanas: gasto de marketing por territorio/campaña.

###Este análisis fue desarrollado para responder las siguientes preguntas de negocio planteadas por el equipo directivo:
¿Cuánto estamos ganando por país?
¿Qué tan rentable es cada mercado considerando los gastos de marketing?

##Extracción y limpieza de datos

En esta etapa se construyó una tabla base unificando información de ventas, productos, categorías, territorios y campañas de marketing mediante consultas SQL y JOINs. El objetivo fue centralizar los datos necesarios para analizar ingresos, costos y rentabilidad por mercado.
Durante el proceso se realizó limpieza y validación de datos, incluyendo tratamiento de valores nulos, selección de variables relevantes y estandarización de categorías. Además, se crearon columnas calculadas para medir indicadores financieros clave:

```sql
SELECT 
v.numero_pedido,
v.clave_producto,
p.nombre_producto,
pc.clave_categoria,
coalesce(p.precio_producto, 0),
coalesce(v.cantidad_pedido, 0),
coalesce(p.costo_producto,0),
t.pais,
t.continente,
v.clave_territorio
FROM ventas_2017 v

left JOIN productos p
ON v.clave_producto = p.clave_producto

left JOIN productos_categorias pc
ON p.clave_subcategoria = pc.clave_subcategoria

left JOIN territorios t
ON v.clave_territorio = t.clave_territorio;
```

```sql
SELECT
    v.numero_pedido,
    v.clave_producto,
    p.nombre_producto,
    pc.clave_categoria,
    COALESCE(p.precio_producto, 0)  AS precio_producto,
    COALESCE(v.cantidad_pedido, 0)  AS cantidad_pedido,
    COALESCE(p.costo_producto, 0)   AS costo_producto,
    t.pais,
    t.continente,
    v.clave_territorio,
coalesce(p.precio_producto,0) * coalesce(v.cantidad_pedido,0) as ingreso_total,
coalesce(p.costo_producto,0) * coalesce(v.cantidad_pedido,0) as costo_total    
FROM ventas_2017 AS v
JOIN productos AS p
  ON v.clave_producto = p.clave_producto
LEFT JOIN productos_categorias AS pc
  ON p.clave_subcategoria = pc.clave_subcategoria
LEFT JOIN territorios AS t
  ON v.clave_territorio = t.clave_territorio;
```

##Cálculo de ingresos y costos por país

En esta etapa se utilizó la tabla ventas_clean como base para calcular el desempeño financiero por país y territorio. El objetivo fue agrupar la información para identificar qué mercados generan mayores ingresos y cuáles representan mayores costos para la empresa.
Se calcularon los ingresos y costos totales mediante funciones de agregación en SQL, sumando los valores de cada territorio y ordenando los resultados de mayor a menor para facilitar la identificación de los mercados más importantes. Los valores fueron presentados como enteros para mejorar la legibilidad y el análisis de la información.

```sql
select
vc.pais,
vc.clave_territorio,
sum(vc.ingreso_total) ::integer as ingresos,
sum(vc.costo_total) ::integer as costos
from ventas_clean as vc
group by vc.pais, vc.clave_territorio
order by ingresos desc
```

##Integración de gastos de marketing

En esta etapa se incorporó la inversión en campañas de marketing al análisis financiero para obtener una visión más completa del desempeño de cada mercado. Para ello, se combinaron los ingresos y costos calculados previamente con la información de gasto en campañas por territorio mediante un LEFT JOIN.

El análisis permitió comparar en una misma tabla los ingresos generados, los costos operativos y la inversión en marketing de cada país y territorio, facilitando la identificación de mercados con mayores niveles de rentabilidad y eficiencia en la inversión publicitaria. Además, se utilizaron funciones de agregación y manejo de valores nulos para asegurar resultados consistentes y legibles.

```sql
SELECT
    v.pais,
    v.clave_territorio,
    SUM(v.ingreso_total)::integer AS ingresos,
    SUM(v.costo_total)::integer AS costos,
    SUM(COALESCE(c.costo_campana::integer, 0)) AS costo_campana
FROM ventas_clean AS v
LEFT JOIN campanas AS c
    ON v.clave_territorio = c.clave_territorio::integer
GROUP BY
    v.pais,
    v.clave_territorio
ORDER BY ingresos DESC;
```

##Cálculo de beneficio bruto, margen y ROI

En esta etapa se calcularon indicadores financieros clave para evaluar la rentabilidad de cada mercado. A partir de los ingresos, costos y gastos de marketing, se obtuvo el beneficio bruto, el margen de ganancia y el ROI, permitiendo identificar qué territorios generan mayores ganancias y qué tan eficiente es la inversión en campañas.

```sql
SELECT
    p.pais,
    p.clave_territorio,
    SUM(p.ingresos)::integer AS ingresos,
    SUM(p.costos)::integer AS costos,
    COALESCE(SUM(c.costo_campana), 0)::integer AS costo_campana,
    SUM(p.ingresos)::integer - SUM(p.costos)::integer AS beneficio_bruto,
    ((SUM(p.ingresos) - SUM(p.costos)) * 100.0)
    / NULLIF(SUM(p.ingresos), 0) AS margen_pct,
    ((SUM(p.ingresos) - SUM(p.costos)) * 100.0)
    / NULLIF(SUM(c.costo_campana), 0) AS roi_pct
FROM pais_ingreso_costo AS p
LEFT JOIN pais_campanas AS c
    ON p.clave_territorio = c.clave_territorio
GROUP BY
    p.pais,
    p.clave_territorio
ORDER BY
    p.clave_territorio, ingresos, costos;
```

##Resultados financieros por país

La siguiente tabla resume el desempeño financiero de cada mercado, incluyendo ingresos, costos, gasto en campañas de marketing, beneficio bruto, margen de ganancia y ROI. Estos indicadores permiten comparar la rentabilidad y eficiencia de cada territorio, facilitando la identificación de los mercados con mejor desempeño y mayores oportunidades de optimización.

<img width="797" height="145" alt="image" src="https://github.com/user-attachments/assets/208208dc-fc35-4b0c-8a71-c6ec8ce314de" />

##Principales hallazgos financieros

El análisis mostró que Estados Unidos es el mercado con mayores ingresos y beneficio bruto, además de presentar el ROI más alto (75.75%), lo que indica una inversión de marketing más eficiente en comparación con otros territorios.

Australia ocupó el segundo lugar en ingresos y ganancias, manteniendo un margen sólido del 41.75% y un ROI del 49.16%. Por otro lado, países como Reino Unido, Alemania y Francia mostraron márgenes similares, pero un ROI considerablemente menor debido a mayores gastos en campañas de marketing.

En general, los resultados evidencian que algunos mercados generan altos ingresos, pero no siempre convierten esa inversión en rentabilidad eficiente, lo que sugiere oportunidades para optimizar el presupuesto de marketing y priorizar los territorios con mejor retorno.

##Hipótesis

Los países con mayores ingresos presentan también mayores niveles de rentabilidad y ROI.
Un mayor gasto en campañas de marketing no garantiza necesariamente un mejor retorno de inversión.
Los mercados con márgenes más altos tienden a ser más eficientes en la gestión de costos operativos.
Existen diferencias significativas en el desempeño financiero entre territorios, lo que puede influir en la asignación estratégica del presupuesto de marketing.

##Conclusiones

Estados Unidos se posicionó como el mercado más rentable, registrando los mayores ingresos, beneficio bruto y ROI, lo que refleja una estrategia de inversión en marketing más eficiente.
Australia mostró un desempeño sólido tanto en ingresos como en margen de ganancia, consolidándose como uno de los mercados más importantes para la compañía.
Reino Unido, Alemania y Francia presentaron márgenes similares, pero un ROI menor debido a niveles elevados de inversión en campañas de marketing.
Los resultados demuestran que altos ingresos no siempre significan mayor rentabilidad, ya que el costo de marketing impacta directamente el retorno obtenido en cada territorio.
El análisis permitió identificar oportunidades para optimizar la inversión en campañas y priorizar mercados con mejor rendimiento financiero.
