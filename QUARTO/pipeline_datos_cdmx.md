# Pipeline de Datos para Simulaciones de Gentrificación — CDMX

**Autores:** [Tu nombre]
**Fecha:** Abril 2026
**Repositorio:** [URL del repositorio]
**Paper de referencia:** Mauro, G., Pedreschi, N., Lambiotte, R., & Pappalardo, L. (2025). *Dynamic models of gentrification.* Advances in Complex Systems. DOI: 10.1142/S0219525925400065

---

## Resumen

Este documento describe de forma reproducible el pipeline completo de preparación de datos para adaptar el modelo ABM de gentrificación de Mauro et al. (2025) a la Ciudad de México. El pipeline consta de dos fases:

1. **Fase A — Distribución de ingresos:** Construcción de `income_clean_CDMX_v2.csv` (60 tramos de ingreso equiponderados) a partir de microdatos del Concentrado ENIGH 2022.
2. **Fase B — Asignación espacial de celdas:** Clustering K-means sobre AGEBs urbanos del Censo de Población 2020 para justificar empíricamente la subdivisión de alcaldías en celdas del grid 5×5.

Los archivos generados alimentan directamente al modelo `GentModelCDMX`, que ejecuta las simulaciones batch con los mismos parámetros estructurales que el paper original (proporciones L/M/H ≈ 38%/57%/5%).

---

## Tabla de contenidos

1. [Fuentes de datos y requisitos](#1-fuentes-de-datos-y-requisitos)
2. [Fase A: Distribución de ingresos CDMX](#2-fase-a-distribución-de-ingresos-cdmx)
3. [Fase B: Clustering K-means por AGEB](#3-fase-b-clustering-k-means-por-ageb)
4. [Conclusiones y justificación formal](#4-conclusiones-y-justificación-formal)
5. [Archivos generados](#5-archivos-generados)
6. [Instrucciones de reproducción](#6-instrucciones-de-reproducción)

---

## 1. Fuentes de datos y requisitos

### 1.1 Datos requeridos

| Archivo | Fuente | Descarga | Tamaño aprox. |
|---------|--------|----------|---------------|
| `concentradohogar.csv` | ENIGH 2022, INEGI | [Microdatos ENIGH 2022](https://www.inegi.org.mx/programas/enigh/nc/2022/#microdatos) | ~50 MB |
| `resultados_ageb_urbano_09_cpv2020.csv` | Censo de Población y Vivienda 2020, INEGI | [Datos abiertos AGEB urbano](https://www.inegi.org.mx/contenidos/programas/ccpv/2020/datosabiertos/ageb_manzana/resultados_ageb_urbano_09_cpv2020.zip) | ~75 MB |

### 1.2 Preprocesamiento del Concentrado ENIGH

El Concentrado ENIGH 2022 contiene registros a nivel nacional. Se filtra para conservar únicamente los hogares ubicados en las 16 alcaldías de la Ciudad de México (códigos `ubica_geo` entre 9002 y 9017):

```python
import pandas as pd

df = pd.read_csv("concentradohogar.csv")
df_cdmx = df[df['ubica_geo'].between(9002, 9017)]
df_cdmx.to_csv('Concentrado_2022_CDMX.csv', index=False)
# Resultado: 2,576 hogares en muestra → 3,082,330 hogares expandidos
```

> **Nota:** El código `ubica_geo=9001` corresponde al total de la entidad y se excluye. Los códigos 9002–9017 mapean a las 16 alcaldías según el catálogo del Marco Geoestadístico Nacional de INEGI.

### 1.3 Mapeo de códigos a alcaldías

```python
GEO_MAP = {
    9002: "Azcapotzalco",         9003: "Coyoacán",
    9004: "Cuajimalpa de Morelos", 9005: "Gustavo A. Madero",
    9006: "Iztacalco",            9007: "Iztapalapa",
    9008: "La Magdalena Contreras",9009: "Milpa Alta",
    9010: "Álvaro Obregón",       9011: "Tláhuac",
    9012: "Tlalpan",              9013: "Xochimilco",
    9014: "Benito Juárez",        9015: "Cuauhtémoc",
    9016: "Miguel Hidalgo",       9017: "Venustiano Carranza",
}
```

### 1.4 Entorno de software

```
Python 3.11
mesa==1.2.1
numpy==1.26.4
scikit-learn==1.3.2
pandas (última compatible)
```

---

## 2. Fase A: Distribución de ingresos CDMX

### 2.1 Objetivo

Construir una distribución de ingresos con **60 tramos de igual peso poblacional** (~1.67% cada uno) que replique la estructura del paper original. Mauro et al. (2025) utilizan datos de la Social Security Administration (SSA) 2022 de Estados Unidos, que tienen aproximadamente 60 tramos con ancho de percentil ~1.67%. Este diseño genera una distribución de cola pesada con alta resolución en el rango bajo-medio y menor granularidad en los extremos, que es lo que produce la dinámica emergente del modelo (desigualdad → gentrificación).

### 2.2 Variable de ingreso

Se utiliza `ing_cor` (ingreso corriente trimestral del hogar) del Concentrado ENIGH 2022, convertido a ingreso anual:

```
ing_anual = ing_cor × 4
```

Cada hogar tiene asociado un factor de expansión (`factor`) que pondera su representatividad en la población objetivo.

### 2.3 Algoritmo de construcción

```python
def build_income_v2(concentrado_path, n_brackets=60):
    df = pd.read_csv(concentrado_path)
    df["alcaldia"]  = df["ubica_geo"].map(GEO_MAP)
    df["ing_anual"] = df["ing_cor"] * 4

    # Distribución empírica ponderada
    ing = df["ing_anual"].values
    fac = df["factor"].values
    sidx = np.argsort(ing)
    ing_s, fac_s = ing[sidx], fac[sidx]
    cumw = np.cumsum(fac_s)
    total_w = cumw[-1]

    # Cortes en percentiles equiponderados
    qcuts = np.linspace(0, 100, n_brackets + 1)
    rows = []
    for i in range(n_brackets):
        lo_w = qcuts[i]   / 100 * total_w
        hi_w = qcuts[i+1] / 100 * total_w
        mask = (cumw > lo_w) & (cumw <= hi_w)
        if mask.sum() == 0:
            nearest = np.argmin(np.abs(cumw - (lo_w + hi_w) / 2))
            mask = np.zeros(len(ing_s), dtype=bool)
            mask[nearest] = True
        bi, bw = ing_s[mask], fac_s[mask]
        rows.append({
            "bound_low":      round(0.0 if i == 0 else float(bi.min()), 2),
            "bound_high":     round(float(bi.max()), 2),
            "number":         int(bw.sum()),
            "average_amount": round(float(np.average(bi, weights=bw)), 2),
        })

    out = pd.DataFrame(rows)
    tot = out["number"].sum()
    out["percent"]       = out["number"] / tot * 100
    out["percent_cumul"] = out["percent"].cumsum()
    return out
```

**Lógica clave:**
- Se ordenan los hogares por ingreso anual ascendente.
- Se acumula el peso del factor de expansión.
- Se divide la distribución ponderada en 60 segmentos de igual masa probabilística.
- Para cada tramo se registran: límite inferior, límite superior, número de hogares expandidos, porcentaje e ingreso promedio ponderado.

### 2.4 Calibración de cutoffs L/M/H

Los cutoffs entre categorías de ingreso se determinan para replicar las proporciones del paper original:

| Categoría | Paper USA (SSA 2022) | CDMX (ENIGH 2022) | Diferencia |
|-----------|---------------------|--------------------|------------|
| L (bajo ingreso) | 38% | 38.31% | +0.31 pp |
| M (medio ingreso) | 57% | 56.67% | −0.33 pp |
| H (alto ingreso) | 5% | 5.02% | +0.02 pp |

```python
# Determinación de cutoffs
row_L = int(out[out["percent_cumul"] >= 38].index[0])   # → 22
row_H = int(out[out["percent_cumul"] >= 95].index[0])   # → 57
```

**Resultado (`cdmx_cutoffs.json`):**

```json
{
  "row_L_max": 22,
  "row_H_min": 57,
  "L_bound_high": 267945.64,
  "H_bound_low": 1140462.56,
  "pct_L": 38.31,
  "pct_M": 56.67,
  "pct_H": 5.02
}
```

**Interpretación en pesos mexicanos (MXN/año):**
- **L (bajo ingreso):** hogares con ingreso anual ≤ $267,946 MXN (~$22,329/mes)
- **M (medio ingreso):** hogares con ingreso anual entre $267,946 y $1,140,463 MXN
- **H (alto ingreso):** hogares con ingreso anual > $1,140,463 MXN (~$95,039/mes)

### 2.5 Archivo generado: `income_clean_CDMX_v2.csv`

| Columna | Descripción |
|---------|-------------|
| `bound_low` | Límite inferior del tramo (MXN/año) |
| `bound_high` | Límite superior del tramo (MXN/año) |
| `number` | Hogares expandidos en el tramo |
| `percent` | Porcentaje del total |
| `percent_cumul` | Porcentaje acumulado |
| `average_amount` | Ingreso promedio ponderado del tramo |

El archivo contiene 60 filas, donde cada tramo representa aproximadamente 1.67% de la población de hogares de CDMX.

---

## 3. Fase B: Clustering K-means por AGEB

### 3.1 Motivación

El modelo ABM asume que cada **celda del grid** representa un vecindario con perfil socioeconómico internamente homogéneo pero diferente de sus vecinos. Asignar una sola celda por alcaldía violaría este supuesto porque alcaldías como Iztapalapa contienen zonas con perfiles socioeconómicos muy diferentes entre sí (por ejemplo, Iztapalapa Centro vs. Sierra de Santa Catarina).

El clustering K-means sobre los AGEBs (Áreas Geoestadísticas Básicas) de cada alcaldía permite:

1. **Agrupar AGEBs similares** por indicadores socioeconómicos (educación, hacinamiento, acceso a bienes).
2. **Determinar K = número de celdas** por alcaldía, donde K se fija a priori por un score compuesto de población, ingreso y vulnerabilidad.
3. **Justificar empíricamente** que alcaldías heterogéneas (como Iztapalapa con K=3) tienen sub-vecindarios estadísticamente distintos, mientras que alcaldías homogéneas (como Milpa Alta con K=1) no requieren subdivisión.

### 3.2 Fuente de datos: Censo 2020 por AGEB urbano

Se utiliza el archivo `resultados_ageb_urbano_09_cpv2020.csv` del Censo de Población y Vivienda 2020 de INEGI, filtrado a nivel AGEB urbano (excluyendo manzanas, totales de localidad y totales de municipio).

**Variables seleccionadas como proxies socioeconómicos:**

| Variable | Descripción | Proxy de |
|----------|-------------|----------|
| `GRAPROES` | Grado promedio de escolaridad | Nivel educativo |
| `PROM_OCUP` | Promedio de ocupantes por cuarto | Hacinamiento |
| `VPH_AUTOM` | Viviendas particulares habitadas con automóvil | Bienestar material |
| `VPH_INTER` | Viviendas particulares habitadas con internet | Modernización/conectividad |
| `POBTOT` | Población total del AGEB | Tamaño del vecindario |

Estas variables son estándar en la literatura de segregación urbana para CDMX (Duhau & Giglia, 2008; Flores, 2019; Saraví, 2015).

### 3.3 Preprocesamiento de datos AGEB

```python
def load_ageb_data(ageb_path):
    dtype_override = {
        "ENTIDAD": str, "MUN": str, "LOC": str,
        "AGEB": str, "MZA": str, "NOM_LOC": str,
    }
    df = pd.read_csv(ageb_path, dtype=dtype_override, low_memory=False)

    # Filtrar SOLO AGEBs reales (excluir totales agregados)
    ageb = df[
        (df["MZA"]  == "000")  &   # nivel AGEB, no manzana
        (df["AGEB"] != "0000") &   # excluir total de localidad
        (df["LOC"]  != "0000") &   # excluir total de municipio
        (df["MUN"]  != "000")      # excluir total de entidad
    ].copy()

    # Convertir features a numérico (valores "*" en el CSV → NaN)
    for col in FEATURES:
        ageb[col] = pd.to_numeric(ageb[col], errors="coerce")

    # Imputar NaN con la mediana de cada municipio
    for col in FEATURES:
        ageb[col] = ageb.groupby("cve_mun")[col].transform(
            lambda x: x.fillna(x.median())
        )
        # Fallback: mediana global si todo el municipio es NaN
        ageb[col] = ageb[col].fillna(ageb[col].median())

    return ageb  # 2,433 AGEBs urbanos reales
```

**Decisiones de limpieza:**
- El CSV del Censo usa `*` para valores confidenciales o no disponibles; se convierten a `NaN` con `pd.to_numeric(errors="coerce")`.
- La imputación por mediana municipal preserva el contexto geográfico local.
- Se excluyen filas de totales agregados (MZA≠"000", AGEB="0000", etc.) que distorsionarían el clustering.

### 3.4 Asignación de celdas por alcaldía

El grid del modelo es de **5×5 = 25 celdas**. Se asignan celdas a cada alcaldía mediante un score compuesto:

```
score = 0.50 × (peso poblacional normalizado)
      + 0.30 × (peso de ingreso normalizado)
      + 0.20 × (peso de vulnerabilidad NSE AMAI)
```

**Distribución resultante:**

| Alcaldía | Celdas (K) | AGEBs | Justificación |
|----------|-----------|-------|---------------|
| Iztapalapa | 3 | 458 | Mayor población + alta heterogeneidad interna |
| Gustavo A. Madero | 3 | 305 | Segunda en población + heterogeneidad significativa |
| Benito Juárez | 2 | 102 | Alto ingreso, perfil dual (comercial/residencial) |
| Cuauhtémoc | 2 | 153 | Centro histórico + zonas residenciales diversas |
| Tlalpan | 2 | 208 | Zona urbana consolidada + zona rural/periurbana |
| Álvaro Obregón | 2 | 199 | Zona alta (Santa Fe) + zona popular |
| Miguel Hidalgo | 2 | 130 | Polanco/Lomas vs. zonas populares |
| Azcapotzalco | 1 | 103 | Relativamente homogénea |
| Coyoacán | 1 | 156 | Relativamente homogénea |
| Cuajimalpa de Morelos | 1 | 31 | Pocos AGEBs, homogénea |
| Iztacalco | 1 | 110 | Relativamente homogénea |
| La Magdalena Contreras | 1 | 52 | Relativamente homogénea |
| Milpa Alta | 1 | 42 | Muy homogénea (rural-periurbana) |
| Tláhuac | 1 | 111 | Muy homogénea |
| Xochimilco | 1 | 122 | Relativamente homogénea |
| Venustiano Carranza | 1 | 151 | Relativamente homogénea |
| **Total** | **25** | **2,433** | |

### 3.5 Algoritmo K-means

```python
from sklearn.preprocessing import StandardScaler
from sklearn.cluster import KMeans
from sklearn.metrics import silhouette_score

FEATURES = ["GRAPROES", "PROM_OCUP", "VPH_AUTOM", "VPH_INTER", "POBTOT"]

def kmeans_alcaldia(ageb_df, alcaldia_name, k_clusters, random_state=42):
    sub = ageb_df[ageb_df["alcaldia"] == alcaldia_name].copy()
    X = sub[FEATURES].copy()

    # Salvaguarda: imputar NaN residuales
    for col in FEATURES:
        if X[col].isna().any():
            X[col] = X[col].fillna(X[col].median())

    # Si hay muy pocos AGEBs, asignar cluster único
    if len(X) < max(k_clusters * 2, 3):
        sub["cluster"] = 0
        return sub, 1.0

    scaler = StandardScaler()
    Xs = scaler.fit_transform(X)

    if k_clusters == 1:
        sub["cluster"] = 0
        sil = 1.0
    else:
        km = KMeans(n_clusters=k_clusters, random_state=random_state,
                    n_init=10, max_iter=300)
        sub["cluster"] = km.fit_predict(Xs)
        sil = silhouette_score(Xs, sub["cluster"])

    return sub, round(float(sil), 3)
```

**Pasos del algoritmo:**
1. Seleccionar AGEBs de la alcaldía.
2. Estandarizar las 5 features con `StandardScaler` (media=0, varianza=1).
3. Aplicar K-means con K = número de celdas asignadas.
4. Calcular el coeficiente de silueta como medida de calidad del clustering.
5. Asignar un `cell_id` global (0–24) a cada cluster de cada alcaldía.

### 3.6 Resultados del clustering

| Alcaldía | AGEBs | K | Silhouette | Calidad |
|----------|-------|---|------------|---------|
| Álvaro Obregón | 199 | 2 | 0.764 | Excelente |
| Miguel Hidalgo | 130 | 2 | 0.449 | Buena |
| Benito Juárez | 102 | 2 | 0.418 | Buena |
| Cuauhtémoc | 153 | 2 | 0.389 | Aceptable |
| Iztapalapa | 458 | 3 | 0.353 | Aceptable |
| Gustavo A. Madero | 305 | 3 | 0.322 | Aceptable |
| Tlalpan | 208 | 2 | 0.322 | Aceptable |
| Alcaldías con K=1 | — | 1 | 1.000 | N/A (trivial) |

**Interpretación de los coeficientes de silueta:**
- Valores > 0.25 indican separación significativa entre clusters (Rousseeuw, 1987).
- Todas las alcaldías con K > 1 superan este umbral, lo que valida empíricamente la subdivisión.
- Álvaro Obregón (0.764) muestra la separación más clara, consistente con la conocida dualidad Santa Fe (alto ingreso) vs. zonas populares.
- Las alcaldías con K=1 tienen silueta trivial (1.0) porque no se subdividen.

### 3.7 Validación complementaria: Heterogeneidad desde ENIGH

Como validación adicional, se calculó el coeficiente de variación (CV) de ingresos por alcaldía directamente desde los microdatos ENIGH:

| Alcaldía | CV | Ratio P90/P10 | Celdas | Consistencia |
|----------|-----|---------------|--------|-------------- |
| Venustiano Carranza | 1.304 | 5.6 | 1 | ⚠ Alta heterogeneidad con 1 celda |
| Cuajimalpa de Morelos | 1.151 | 5.3 | 1 | ⚠ Alta heterogeneidad con 1 celda |
| Iztapalapa | 0.769 | 5.6 | 3 | ✓ Consistente |
| Gustavo A. Madero | 0.788 | 6.2 | 3 | ✓ Consistente |
| Milpa Alta | 0.440 | 3.9 | 1 | ✓ Consistente |
| Tláhuac | 0.439 | 3.5 | 1 | ✓ Consistente |

> **Nota sobre Venustiano Carranza y Cuajimalpa:** El alto CV sugiere que estas alcaldías podrían beneficiarse de más de 1 celda. Sin embargo, el constraint del grid 5×5 = 25 celdas limita la asignación. Esto se documenta como una limitación del modelo.

### 3.8 Archivos generados

**`ageb_clusters.csv`** (2,433 filas):

| Columna | Descripción |
|---------|-------------|
| `ENTIDAD` | Código de entidad (09 = CDMX) |
| `MUN` | Código de municipio/alcaldía |
| `LOC` | Código de localidad |
| `AGEB` | Clave del AGEB |
| `MZA` | Código de manzana (siempre "000" a nivel AGEB) |
| `POBTOT` | Población total del AGEB |
| `cve_mun` | Código municipal como entero |
| `alcaldia` | Nombre de la alcaldía |
| `cluster` | Cluster K-means dentro de la alcaldía (0-indexed) |
| `cell_id` | Identificador global de celda en el grid (0–24) |

**`alcaldia_to_cells.json`:**

Diccionario que mapea cada alcaldía a su(s) celda(s) en el grid:

```json
{
  "Iztapalapa": [11, 12, 13],
  "Gustavo A. Madero": [7, 8, 9],
  "Benito Juárez": [1, 2],
  "Cuauhtémoc": [5, 6],
  "Tlalpan": [18, 19],
  "Álvaro Obregón": [23, 24],
  "Miguel Hidalgo": [15, 16],
  "Azcapotzalco": [0],
  "Coyoacán": [3],
  "Cuajimalpa de Morelos": [4],
  "Iztacalco": [10],
  "La Magdalena Contreras": [14],
  "Milpa Alta": [17],
  "Tláhuac": [20],
  "Venustiano Carranza": [21],
  "Xochimilco": [22]
}
```

---

## 4. Conclusiones y justificación formal

### 4.1 Por qué esta distribución de ingresos es adecuada para el ABM

La distribución de 60 tramos construida desde microdatos ENIGH 2022 satisface los requisitos estructurales del modelo de Mauro et al. (2025):

1. **Misma granularidad:** 60 tramos de ~1.67% cada uno, idéntico al formato SSA utilizado en el paper original.
2. **Proporciones L/M/H calibradas:** La desviación respecto a las proporciones del paper (38/57/5%) es menor a 0.5 puntos porcentuales en cada categoría, lo que garantiza que la dinámica del modelo (probabilidades de movimiento de agentes C, B y A) se preserve.
3. **Base empírica sólida:** Los microdatos ENIGH con factor de expansión representan la distribución real de ingresos de 3.08 millones de hogares en CDMX.
4. **Resolución en cola pesada:** La estructura de percentiles equiponderados captura la cola derecha de la distribución (hogares de muy alto ingreso) con la misma fidelidad que los datos SSA.

### 4.2 Por qué K-means sobre AGEBs es el método adecuado

1. **Homogeneidad intra-celda:** El modelo ABM asume que cada celda es un vecindario socioeconómicamente homogéneo. K-means agrupa AGEBs con perfiles similares, cumpliendo este supuesto.
2. **Heterogeneidad inter-celda:** Los coeficientes de silueta (todos > 0.25 para K > 1) confirman que los clusters son estadísticamente distinguibles entre sí.
3. **Variables censales como proxies:** Las 5 variables seleccionadas (educación, hacinamiento, acceso a automóvil, internet, y tamaño poblacional) son proxies socioeconómicos establecidos en la literatura de segregación urbana mexicana.
4. **Justificación del número de celdas:** El K de cada alcaldía no es arbitrario; se determina por un score compuesto de factores demográficos y socioeconómicos, y se valida empíricamente tanto con silueta (Censo) como con CV de ingresos (ENIGH).

### 4.3 Limitaciones documentadas

- **Grid fijo de 25 celdas:** El constraint de 5×5 impide dar más celdas a alcaldías con alta heterogeneidad interna (e.g., Venustiano Carranza con CV=1.304 pero K=1).
- **Sin georreferenciación:** Los clusters K-means no consideran contigüidad geográfica entre AGEBs. Un AGEB puede estar asignado al mismo cluster que otro geográficamente distante dentro de la misma alcaldía.
- **Temporalidad de los datos:** La distribución de ingresos es ENIGH 2022 y el clustering es Censo 2020, con 2 años de diferencia. Se asume estabilidad relativa en los patrones socioeconómicos por AGEB en este período.
- **Imputación de datos faltantes:** Los valores `*` del Censo se imputan con mediana municipal, lo que puede suavizar variaciones locales extremas.

---

## 5. Archivos generados

| Archivo | Descripción | Generado por |
|---------|-------------|-------------|
| `income_clean_CDMX_v2.csv` | Distribución de ingresos, 60 tramos equiponderados | Fase A: `build_income_v2()` |
| `cdmx_cutoffs.json` | Cutoffs L/M/H como dict importable | Fase A: `build_income_v2()` |
| `ageb_clusters.csv` | 2,433 AGEBs con cluster y cell_id asignados | Fase B: `run_kmeans_all_alcaldias()` |
| `alcaldia_to_cells.json` | Mapa alcaldía → lista de cell_ids del grid | Fase B: `run_kmeans_all_alcaldias()` |

---

## 6. Instrucciones de reproducción

### 6.1 Desde cero (reproducción completa)

```bash
# 1. Clonar el repositorio
git clone https://github.com/[tu-usuario]/Gentrification.git
cd Gentrification/CDMX

# 2. Crear entorno con uv
uv venv --python 3.11
source .venv/bin/activate
uv pip install mesa==1.2.1 numpy==1.26.4 scikit-learn==1.3.2 pandas openpyxl tqdm

# 3. Descargar datos de INEGI (ver sección 1.1)
# Colocar archivos en income/

# 4. Ejecutar pipeline de datos
python pipeline_cdmx_datos.py
# → Genera: income_clean_CDMX_v2.csv, cdmx_cutoffs.json,
#           ageb_clusters.csv, alcaldia_to_cells.json

# 5. Verificar proporciones
python -c "
import pandas as pd
df = pd.read_csv('income/income_clean_CDMX_v2.csv')
print(f'Tramos: {len(df)}')
print(f'L: {df.iloc[:23][\"percent\"].sum():.2f}%')
print(f'M: {df.iloc[23:57][\"percent\"].sum():.2f}%')
print(f'H: {df.iloc[57:][\"percent\"].sum():.2f}%')
"
```

### 6.2 Usando archivos precalculados

Si no se dispone de los microdatos originales de INEGI, los archivos `income_clean_CDMX_v2.csv`, `cdmx_cutoffs.json`, `ageb_clusters.csv` y `alcaldia_to_cells.json` incluidos en el repositorio son suficientes para ejecutar las simulaciones:

```bash
cd Gentrification/CDMX
python run_batch_cdmx.py \
    --mode improve \
    --num_agents 4096 \
    --rep_start 0 --rep_end 50 \
    --workers 6 \
    --steps 300 --h 20 \
    --width 5 --height 5
```

---

## Referencias

- Duhau, E. & Giglia, Á. (2008). *Las reglas del desorden: habitar la metrópoli.* Siglo XXI.
- Flores, C. A. (2019). Segregación residencial y distribución del ingreso en la CDMX. *Estudios Demográficos y Urbanos*, 34(1).
- INEGI (2020). Censo de Población y Vivienda 2020. Resultados por AGEB urbano.
- INEGI (2022). Encuesta Nacional de Ingresos y Gastos de los Hogares (ENIGH) 2022. Microdatos.
- Mauro, G., Pedreschi, N., Lambiotte, R., & Pappalardo, L. (2025). Dynamic models of gentrification. *Advances in Complex Systems*. DOI: 10.1142/S0219525925400065
- Rousseeuw, P. J. (1987). Silhouettes: A graphical aid to the interpretation and validation of cluster analysis. *Journal of Computational and Applied Mathematics*, 20, 53–65.
- Saraví, G. A. (2015). *Juventudes fragmentadas: socialización, clase y cultura en la construcción de la desigualdad.* FLACSO/CIESAS.
