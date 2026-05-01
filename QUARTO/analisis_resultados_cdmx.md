# Análisis de Resultados — Simulaciones de Gentrificación CDMX

**Réplica de Mauro et al. (2025) adaptada a Ciudad de México**
**150 réplicas × 5 valores de p_H × 4,096 agentes × grid 5×5**

---

## Definiciones previas: las dos métricas de gentrificación

Antes de interpretar las gráficas, es necesario entender qué mide cada métrica:

**G_bin (count-based, métrica tradicional):** Mide si la proporción de residentes de ingreso medio y alto (M+H) en una celda supera un umbral fijo. Es una fotografía estática: "¿hay más ricos aquí que antes?". Equivale a los indicadores censales tradicionales de cambio demográfico. Se activa cuando la composición de la celda cruza un umbral, pero no distingue *cómo* llegaron esos residentes.

**G_net (network-based, métrica propuesta por Mauro et al.):** Mide simultáneamente la *salida neta* de residentes de bajo ingreso (L) y la *entrada neta* de residentes de medio y alto ingreso (M+H) usando flujos en redes temporales dirigidas. Captura el proceso dinámico: "¿están saliendo los pobres Y entrando los ricos al mismo tiempo?". Es la media geométrica de ambos flujos normalizados, por lo que requiere que *ambas* condiciones ocurran simultáneamente.

La hipótesis central del paper es que **G_net detecta la gentrificación antes que G_bin**, porque captura los flujos de movimiento antes de que se reflejen en los conteos estáticos.

---

## Notebook 1: `batch_analysis_cdmx` — Análisis agregado

### Figura 1: Boxplots G_bin y G_net por p_H (equivalente a Fig. 2 del paper)

**Qué muestra:** Dos boxplots lado a lado. Cada uno tiene 5 cajas (una por valor de p_H = 0.00, 0.01, 0.05, 0.10, 0.15), y cada caja resume la distribución de 150 réplicas. El eje Y es el porcentaje de las 25 celdas del grid que experimentaron al menos un evento de gentrificación.

**Resultados CDMX:**

| p_H | G_bin mediana (%) | G_net mediana (%) |
|-----|------------------|------------------|
| 0.00 | 0.0 | 16.0 |
| 0.01 | 16.0 | 36.0 |
| 0.05 | 20.0 | 38.0 |
| 0.10 | 28.0 | 48.0 |
| 0.15 | 52.0 | 68.0 |

**Interpretación:**

- **p_H = 0.0 (sin movimiento estratégico de H):** G_bin no detecta gentrificación en ninguna celda (mediana = 0%), pero G_net detecta actividad en el 16% de las celdas. Esto indica que incluso sin movimiento estratégico, los movimientos aleatorios de agentes generan flujos que G_net captura pero G_bin ignora. Es un hallazgo menor pero consistente con el paper original.

- **A medida que p_H aumenta, ambas métricas crecen monotónicamente.** Esto confirma el resultado central del paper: los agentes de alto ingreso son el motor de la gentrificación. Cuanto más se mueven estratégicamente (buscando celdas con mayor crecimiento de riqueza), más celdas experimentan gentrificación.

- **G_net siempre es mayor que G_bin.** Para p_H = 0.15, G_net detecta gentrificación en el 68% de las celdas vs. 52% de G_bin. La diferencia promedio es de ~18 puntos porcentuales. Esto replica el hallazgo principal del paper: la métrica basada en redes es más sensible que la basada en conteos.

- **La variabilidad (dispersión de los boxplots) aumenta con p_H.** Esto es esperado: con mayor probabilidad de movimiento estratégico, los resultados dependen más de la semilla aleatoria y la configuración inicial específica de cada réplica.

**Conclusión:** El modelo adaptado a CDMX reproduce el comportamiento cualitativo del paper original. La gentrificación es una función creciente de p_H, y G_net consistentemente detecta más celdas gentrificadas que G_bin.

---

### Figura 2: Sensibilidad al número de agentes (curva de densidad)

**Qué muestra:** Dos paneles con líneas de tendencia: G_bin y G_net en función del número de agentes (N = 1024, 2048, 4096, 8192), con una línea por cada valor de p_H. El eje X está en escala logarítmica base 2.

**Interpretación:**

- **A mayor densidad de agentes, mayor gentrificación.** Para p_H = 0.15, G_net pasa de ~30% con 1024 agentes a ~85% con 8192. Esto replica el hallazgo del paper sobre el efecto de densidad: ciudades más densas son más susceptibles a gentrificación.

- **El efecto de densidad es más pronunciado para p_H altos.** Las curvas de p_H = 0.0 y p_H = 0.01 son relativamente planas, mientras que p_H = 0.10 y p_H = 0.15 muestran pendientes pronunciadas. Esto sugiere que la densidad y el comportamiento estratégico interactúan: la densidad amplifica el efecto del movimiento estratégico de H.

- **4096 agentes (el tamaño principal de nuestras simulaciones) está en la zona media de la curva,** lo que indica que nuestros resultados no están saturados (todavía hay margen para mayor gentrificación con más agentes) ni son demasiado ruidosos (como ocurriría con 1024).

**Conclusión:** La densidad urbana promueve la gentrificación, y este efecto se amplifica con el comportamiento estratégico de los agentes de alto ingreso. CDMX, con su alta densidad real, es un contexto donde este mecanismo sería particularmente relevante.

---

### Figura 3: Réplica individual — G_net y riqueza mediana (equivalente a Fig. 3 del paper)

**Qué muestra:** Dos paneles apilados para una réplica específica (p_H = 0.1, rep = 0):

- **Panel a) G_net por celda:** Cada línea coloreada es una celda del grid (solo las que tuvieron G_net > 0). La línea vertical en x = 21 marca el momento en que los agentes H empiezan a moverse estratégicamente (paso h+1 = 21).

- **Panel b) Riqueza mediana por celda:** Escala logarítmica. Las líneas grises son celdas sin gentrificación; las coloreadas coinciden con las de panel a). Antes de x = 21, todas las celdas son grises (fase de equilibrio). Después de x = 21, algunas celdas saltan a riquezas medianas mucho más altas.

**Interpretación:**

- **12 de 25 celdas (48%) experimentaron al menos un evento G_net > 0** en esta réplica particular. Esto es consistente con la mediana de 48% para p_H = 0.1.

- **Los picos de G_net son transitorios:** las líneas coloreadas suben, alcanzan un máximo, y luego bajan. Esto refleja que la gentrificación es un *proceso*, no un estado permanente. Los flujos netos (salida de L, entrada de M+H) son intensos durante la transición, pero se estabilizan una vez que la composición de la celda se ha transformado.

- **La riqueza mediana (panel b) muestra saltos discretos** en las celdas gentrificadas. Antes de h+1, todas las celdas tienen riquezas similares (~$200k–$400k MXN). Después, las celdas gentrificadas saltan a >$1M MXN, mientras que las no gentrificadas se mantienen o bajan. Esto visualiza la segregación espacial emergente.

- **El timing varía entre celdas:** algunas gentrificar temprano (paso ~25), otras tarde (paso ~65). Esto sugiere un efecto de cascada: las primeras celdas en gentrificarse expulsan agentes L que llegan a celdas vecinas, alterando su composición y potencialmente desencadenando nuevos eventos.

**Conclusión:** La gentrificación en el modelo se manifiesta como picos transitorios de flujos netos que coinciden con saltos discretos en la riqueza mediana de las celdas afectadas, replicando el patrón documentado en el paper original.

---

### Figura 4: Heatmap y barplot de susceptibilidad por alcaldía

**Qué muestra:** Dos paneles:

- **Izquierda: Heatmap** con G_net mediana por alcaldía (filas) y p_H (columnas). Colores de amarillo (0%) a rojo oscuro (100%).
- **Derecha: Barplot horizontal** de G_net mediana ± desviación estándar para p_H = 0.1.

**Resultados clave (p_H = 0.1):**

| Grupo | Alcaldías | G_net med | Interpretación |
|-------|-----------|-----------|----------------|
| Alta susceptibilidad | Azcapotzalco, Coyoacán, Cuajimalpa, Iztacalco, Magdalena C., Tláhuac, V. Carranza, Xochimilco | 100% | Con 1 celda, cualquier evento gentrifica el 100% |
| Susceptibilidad media | Á. Obregón, Benito Juárez, Cuauhtémoc, Miguel Hidalgo, Tlalpan | 50–100% | Con 2 celdas, típicamente 1 de 2 gentrifica |
| Menor susceptibilidad | GAM, Iztapalapa | 33.3% | Con 3 celdas, típicamente 1 de 3 gentrifica |
| Sin gentrificación | Milpa Alta | 0% | Homogénea, rural-periurbana |

**Interpretación crítica:**

- **Las alcaldías con 1 celda muestran 100% de gentrificación** no porque sean intrínsecamente más susceptibles, sino porque tienen una sola celda y cualquier evento cubre el 100% de su representación en el grid. Esto es un **artefacto de la resolución del grid**, no un hallazgo sustantivo.

- **La comparación significativa es entre alcaldías con el mismo número de celdas.** Entre las de 2 celdas, Álvaro Obregón (100%) es más susceptible que Benito Juárez (50%). Entre las de 3 celdas, GAM e Iztapalapa son comparables (33.3%).

- **Milpa Alta (0%) es la única alcaldía sin gentrificación en la mediana.** Esto es consistente con su perfil: la más rural, homogénea (CV = 0.44 en ENIGH), y la de menor ingreso promedio. En el modelo, una celda homogénea y periférica tiene pocas razones para atraer agentes H.

**Conclusión:** El mapeo celda-alcaldía permite interpretar los resultados del ABM en términos de la geografía real de CDMX, pero la resolución del grid (25 celdas para 16 alcaldías) limita la granularidad del análisis. Las alcaldías con más celdas (GAM, Iztapalapa) permiten comparaciones más ricas.

---

## Notebook 2: `flows_viz_cdmx` — Visualización de flujos

### Figura 5: Cherry-pick G_bin vs G_net en una celda (equivalente a Fig. 4a del paper)

**Qué muestra:** Un panel con dos series temporales para la celda (0,0), p_H = 0.1, rep = 0:

- **G_bin (cuadrados negros):** salta de 0 a 1 en un momento y se queda en 1 permanentemente.
- **G_net (círculos amarillos):** muestra un pico transitorio que luego decae a ~0.

**Interpretación:**

- **G_bin es una métrica binaria irreversible:** una vez que la proporción M+H supera el umbral, G_bin = 1 para siempre. No captura la dinámica *durante* la transición, solo el resultado final.

- **G_net captura la dinámica transitoria:** el pico indica el momento de máxima actividad de gentrificación (máximos flujos simultáneos de salida L y entrada M+H). Después del pico, los flujos se estabilizan y G_net baja, no porque la gentrificación se "deshaga", sino porque el proceso de transición ha concluido.

- **G_net detecta antes que G_bin:** el pico de G_net ocurre en el paso 21, mientras que G_bin no cambia hasta más tarde. Esto es el hallazgo central del paper: la métrica basada en redes funciona como un **sistema de alerta temprana**.

**Conclusión:** Para esta celda, G_net detecta la gentrificación al menos varios pasos antes que G_bin, confirmando su utilidad como indicador temprano.

---

### Figuras 6 y 7: Grafos de flujo — salida L y entrada M+H (equivalente a Fig. 4b del paper)

**Qué muestran:** Dos pares de grafos de red sobre el grid 5×5:

- **Durante el pico (t = 21):** Flechas rojas = agentes L saliendo de la celda focal (nodo negro). Flechas verde-azul = agentes M+H entrando a la celda focal.
- **Post-pico (t = 29):** Misma estructura pero 8 pasos después.

**Interpretación:**

- **Durante el pico:** se observan flechas rojas saliendo de la celda (0,0) hacia múltiples celdas vecinas y lejanas. Simultáneamente, flechas verde-azul gruesas entran desde varias direcciones. Este patrón dual — expulsión + atracción — es la firma visual de la gentrificación.

- **Post-pico:** las flechas de entrada M+H son mucho más gruesas y numerosas, indicando que la celda ya ha sido "conquistada" por residentes de mayor ingreso. Las flechas de salida L son menores porque ya quedan pocos agentes L en la celda.

- **El tamaño de los nodos refleja la riqueza mediana.** El nodo focal (negro) crece entre t=21 y t=29, visualizando el aumento de riqueza asociado a la gentrificación.

**Conclusión:** Los grafos de flujo confirman visualmente que la gentrificación es un proceso de *displacement* (desplazamiento): los residentes de bajo ingreso son reemplazados por residentes de mayor ingreso, no complementados.

---

## Notebook 3: `who_first_cdmx` — Análisis de alerta temprana

### Figura 8: Cross-correlación G_bin vs G_net (equivalente a Fig. 4c del paper)

**Qué muestra:** Cuatro paneles (uno por p_H = 0.01, 0.05, 0.10, 0.15). Cada uno tiene:

- **Línea coloreada con marcadores:** correlación cruzada promedio ⟨R⟩ entre las señales binarizadas de G_bin y G_net, en función del desfase temporal τ.
- **Línea gris punteada:** baseline (señales shuffled aleatoriamente).
- **Banda gris:** intervalo de confianza al 90% del baseline.
- **Línea vertical en τ = 0:** sincronía perfecta.

**Interpretación:**

- **El pico de correlación está en τ negativo** (τ ≈ −1 a −3 para todos los p_H). Esto significa que G_net *precede* a G_bin: los eventos de G_net ocurren 1–3 pasos antes que los de G_bin. La correlación es significativamente mayor que el baseline shuffled.

- **A mayor p_H, mayor la correlación pico.** Para p_H = 0.01, el pico es ~0.05; para p_H = 0.15, es ~0.15. Esto indica que la señal de alerta temprana es más fuerte cuando hay más movimiento estratégico.

- **La asimetría del perfil de correlación** (mayor a la izquierda de τ = 0) confirma la direccionalidad temporal: G_net → G_bin, no al revés.

**Conclusión:** G_net detecta la gentrificación de 1 a 3 pasos antes que G_bin en el contexto CDMX, replicando el resultado del paper original. Esto valida a G_net como un potencial sistema de alerta temprana para política urbana.

---

### Figura 9: Información mutua entre G_bin y G_net (suplementario del paper)

**Qué muestra:** Un panel con cuatro líneas (una por p_H), mostrando la información mutua (MI) entre señales binarizadas de G_bin y G_net en función del desfase τ. La MI mide la dependencia estadística sin asumir linealidad.

**Interpretación:**

- **El pico de MI está en τ = −1 para todos los p_H**, confirmando con una métrica no lineal que G_net precede a G_bin.

- **La MI es mayor para p_H altos** (≈ 0.010 para p_H = 0.15 vs. ≈ 0.003 para p_H = 0.01), consistente con la cross-correlación.

- **La MI captura dependencias que la correlación lineal podría perder.** El hecho de que ambas métricas (correlación y MI) coincidan en el desfase y la tendencia refuerza la robustez del hallazgo.

**Conclusión:** La información mutua confirma de forma independiente que G_net es un indicador adelantado de G_bin, con un desfase consistente de 1–3 pasos temporales.

---

### Figura 10: Barcodes — Eventos discretos (suplementario del paper)

**Qué muestra:** Tres paneles apilados para una celda específica (p_H = 0.15, rep = 0):

- **Panel superior:** Series continuas de G_bin (negro) y G_net (amarillo).
- **Panel medio:** Barcode de G_net — líneas verticales amarillas en los momentos exactos de los picos.
- **Panel inferior:** Barcode de G_bin — línea vertical negra en el momento del cambio de estado.

**Interpretación:**

- **G_net genera 2–3 barras (eventos) vs. 1 barra de G_bin.** Esto confirma que G_net es más granular: detecta múltiples olas de flujos, mientras que G_bin solo registra el cruce del umbral.

- **Las barras de G_net aparecen antes que la barra de G_bin,** consistente con el análisis de cross-correlación.

- **G_bin es irreversible (una vez que sube a 1, no baja)**, mientras que G_net puede detectar fluctuaciones post-gentrificación.

**Conclusión:** La visualización de barcodes hace explícito que G_net ofrece una representación más rica y temprana del proceso de gentrificación que G_bin.

---

## Resumen de hallazgos principales

1. **El modelo de gentrificación de Mauro et al. (2025) se replica exitosamente con datos de CDMX.** Las proporciones L/M/H (38.3/56.7/5.0%) son prácticamente idénticas a las del paper USA (38/57/5%), y los patrones cualitativos se preservan.

2. **G_net detecta gentrificación 1–3 pasos antes que G_bin,** confirmando su utilidad como sistema de alerta temprana. Este resultado se valida tanto con cross-correlación como con información mutua.

3. **La gentrificación crece monotónicamente con p_H** (probabilidad de movimiento estratégico de agentes H). Sin movimiento estratégico (p_H = 0), la gentrificación es mínima o nula.

4. **La densidad urbana amplifica la gentrificación,** especialmente para valores altos de p_H. CDMX, con su alta densidad real, es un contexto donde este mecanismo sería particularmente relevante.

5. **El mapeo celda-alcaldía permite una lectura geográfica** de los resultados: Milpa Alta es la única alcaldía consistentemente sin gentrificación (mediana G_net = 0%), mientras que alcaldías periurbanas con perfil mixto (como Álvaro Obregón) muestran alta susceptibilidad. Sin embargo, la resolución del grid (25 celdas) limita las conclusiones alcaldía-por-alcaldía.

6. **Los grafos de flujo confirman visualmente el mecanismo de desplazamiento:** la gentrificación no es solo un cambio de composición, sino un proceso activo de expulsión de residentes de bajo ingreso y atracción de residentes de mayor ingreso.

---

## Limitaciones y trabajo futuro

- **Resolución del grid:** 25 celdas para 16 alcaldías produce resultados dominados por alcaldías con 1 celda (100% o 0% trivialmente). Un grid más grande (7×7 o 9×9) permitiría mayor resolución.

- **Validación externa:** Los resultados del modelo no se han comparado aún con datos observables de cambio demográfico o precios de renta en CDMX. Datos de Inside Airbnb o Softec podrían servir como señal de validación.

- **Temporalidad:** Los "pasos" del modelo no tienen una correspondencia directa con años o meses reales. La calibración temporal requeriría datos longitudinales de movilidad residencial.

- **Heterogeneidad espacial:** El modelo trata todas las celdas como equivalentes estructuralmente. En la realidad, las celdas cercanas al centro (Cuauhtémoc, Benito Juárez) tienen dinámicas diferentes a las periféricas (Milpa Alta, Tláhuac).
