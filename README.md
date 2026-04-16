# Regresi-n-y-correlaci-n-m-ltiple_ARGEDUCATIVO.ipynb
# ===============================================
# CARGA Y UNIÓN DE INDICADORES EDUCATIVOS (SECUNDARIA)
# Salida: base_df con índice Territorio y columnas (Año, Métrica),
# donde CADA Territorio tiene Abandono/Promoción/Repitencia por AÑO.
# ===============================================

import pandas as pd
import numpy as np
import re
import unicodedata
from pathlib import PureWindowsPath

# ---------- Utilidades ----------
def norm_txt(s):
    s = str(s)
    s = unicodedata.normalize("NFKD", s).encode("ascii", "ignore").decode("ascii")
    s = s.strip().lower()
    s = re.sub(r"\s+", " ", s)
    return s

def to_posix(path_str):
    return PureWindowsPath(path_str).as_posix()

def detectar_col_territorio(df):
    candidatos = []
    for c in df.columns:
        cn = norm_txt(c)
        if any(k in cn for k in ["territorio", "provincia", "jurisdiccion"]):
            candidatos.append(c)
    if candidatos:
        for c in candidatos:
            if norm_txt(c) == "territorio":
                return c
        return candidatos[0]
    return df.columns[0]

def detectar_col_valor(df, prefer_keywords=None):
    if prefer_keywords is None:
        prefer_keywords = []
    for c in df.columns:
        cn = norm_txt(c)
        if any(k in cn for k in prefer_keywords):
            return c
    num_cols = [c for c in df.columns if pd.api.types.is_numeric_dtype(df[c])]
    if num_cols:
        return num_cols[-1]
    return df.columns[-1]

def limpiar_df_base(df):
    df = df.copy()
    df = df.dropna(how="all")
    df = df.loc[:, ~df.columns.duplicated(keep="last")]
    return df

def cargar_xlsx_inteligente(path, prefer_keywords):
    try:
        df = pd.read_excel(path, engine="openpyxl")
        if isinstance(df.columns, pd.MultiIndex):
            cols = []
            for tpl in df.columns:
                parts = [str(x) for x in tpl if (x is not None and not str(x).startswith("Unnamed"))]
                cols.append(" - ".join(parts) if parts else "")
            df.columns = cols
        df = limpiar_df_base(df)
        ter_col = detectar_col_territorio(df)
        return df, ter_col
    except Exception:
        pass

    df2 = pd.read_excel(path, header=None, engine="openpyxl")
    df2 = limpiar_df_base(df2)
    header_row = None
    for i, row in df2.iterrows():
        row_norm = [norm_txt(x) for x in row.tolist()]
        if any(x in ("territorio", "provincia", "jurisdiccion") for x in row_norm):
            header_row = i
            break
    if header_row is None:
        header_row = 0
    new_header = df2.iloc[header_row].astype(str).tolist()
    body = df2.iloc[header_row+1:].copy()
    body.columns = new_header
    body = limpiar_df_base(body)
    ter_col = detectar_col_territorio(body)
    return body, ter_col

def preparar_tidy(path, metrica, anio, prefer_keywords):
    df_raw, ter_col = cargar_xlsx_inteligente(path, prefer_keywords=prefer_keywords)
    if ter_col not in df_raw.columns:
        ter_col = detectar_col_territorio(df_raw)
    val_col = detectar_col_valor(df_raw, prefer_keywords=prefer_keywords)

    df = df_raw[[ter_col, val_col]].copy()
    df = df.rename(columns={ter_col: "Territorio", val_col: "Valor"})
    df["Territorio"] = df["Territorio"].astype(str).str.strip()
    df = df[df["Territorio"].notna() & (df["Territorio"] != "")]
    df["Valor"] = pd.to_numeric(df["Valor"], errors="coerce")
    df = df.dropna(subset=["Valor"]).copy()
    df["Métrica"] = metrica
    df["Año"] = anio
    return df[["Territorio", "Año", "Métrica", "Valor"]]

# ---------- Rutas ----------
BASE_DIR = r"C:\Users\Brenda\Desktop\DATA BASE\INDICADORES EDUCATIVOS-DATASET ARGENTINA"

rutas = {
    "abandono": {
        "2019-2020": r"Abandono interanual\2019-2020-Abandono interanual.xlsx",
        "2020-2021": r"Abandono interanual\2020-2021-Abandono interanual.xlsx",
        "2021-2022": r"Abandono interanual\2021-2022-Abandono interanual.xlsx",
    },
    "promocion": {
        "2019": r"Promoción efectiva\2019-Promoción efectiva.xlsx",
        "2020": r"Promoción efectiva\2020-Promoción efectiva.xlsx",
        "2021": r"Promoción efectiva\2021-Promoción efectiva.xlsx",
        "2022": [
            r"Promoción efectiva\2022-Promoción efectiva.xlsx",
            r"Promoción efectiva\2022-Pomoción efectiva.xlsx"  # typo fallback
        ],
    },
    "repitencia": {
        "2019": r"Repitencia\2019-Repitencia.xlsx",
        "2020": r"Repitencia\2020-Repitencia.xlsx",
        "2021": r"Repitencia\2021-Repitencia.xlsx",
        "2022": r"Repitencia\2022-Repitencia.xlsx",
    }
}

# ---------- Build TIDY ----------
tidy_list = []

# Abandono → mapear a AÑO DE CIERRE del periodo (2019-2020 -> 2020, etc.)
map_end_year = {
    "2019-2020": 2020,
    "2020-2021": 2021,
    "2021-2022": 2022,
}

for periodo, rel_path in rutas["abandono"].items():
    full_path = to_posix(BASE_DIR + "/" + rel_path)
    tidy_list.append(preparar_tidy(
        full_path,
        metrica="Abandono",
        anio=map_end_year.get(periodo, periodo),
        prefer_keywords=["abandono", "tasa", "interanual", "secundaria"]
    ))

# Promoción
for anio, rel in rutas["promocion"].items():
    if isinstance(rel, list):
        last_err = None
        loaded = False
        for candidate in rel:
            try:
                full_path = to_posix(BASE_DIR + "/" + candidate)
                tidy_list.append(preparar_tidy(
                    full_path,
                    metrica="Promoción",
                    anio=int(anio),
                    prefer_keywords=["promocion", "tasa", "secundaria", "efectiva"]
                ))
                loaded = True
                break
            except Exception as e:
                last_err = e
        if not loaded:
            raise last_err
    else:
        full_path = to_posix(BASE_DIR + "/" + rel)
        tidy_list.append(preparar_tidy(
            full_path,
            metrica="Promoción",
            anio=int(anio),
            prefer_keywords=["promocion", "tasa", "secundaria", "efectiva"]
        ))

# Repitencia
for anio, rel_path in rutas["repitencia"].items():
    full_path = to_posix(BASE_DIR + "/" + rel_path)
    tidy_list.append(preparar_tidy(
        full_path,
        metrica="Repitencia",
        anio=int(anio),
        prefer_keywords=["repitencia", "tasa", "secundaria"]
    ))

tidy = pd.concat(tidy_list, ignore_index=True)

# Asegurar tipo de Año (numérico si aplica) para ordenar correctamente
tidy["Año"] = pd.to_numeric(tidy["Año"], errors="ignore")

# ---------- Pivot: (Año, Métrica) como columnas ----------
base_df = (
    tidy.pivot_table(index="Territorio",
                     columns=["Año", "Métrica"],
                     values="Valor",
                     aggfunc="first")
        .sort_index(axis=1)
)
base_df = base_df.sort_index()

print("Dimensiones de base_df:", base_df.shape)
display(base_df.head(10))

# =====================================================
# AGREGAR REGIONES A base_df
# - Crea columna "Región" por territorio
# - Agrega filas con promedios regionales
# - Mantiene "Argentina" como "País" (si existe)
# =====================================================

import pandas as pd
import numpy as np
import unicodedata
import re

# --- Normalizador para comparar nombres sin acentos ni mayúsculas ---
def norm_txt(s):
    s = str(s)
    s = unicodedata.normalize("NFKD", s).encode("ascii", "ignore").decode("ascii")
    s = re.sub(r"\s+", " ", s.strip().lower())
    return s

# --- Mapa de miembros por región (se permiten sinónimos/variantes) ---
regiones_raw = {
    "Gran Buenos Aires": [
        "buenos aires", "resto de buenos aires", "conurbano",
        "ciudad de buenos aires", "caba"  # por si tu archivo usa CABA
    ],
    "Noroeste": [
        "catamarca", "jujuy", "salta",
        "santiago del estero", "santiago de estero",  # variantes
        "tucuman"
    ],
    "Noreste": [
        "chaco", "corrientes", "formosa", "misiones"
    ],
    "Cuyo": [
        "la rioja", "mendoza", "mendonza",  # variante
        "san juan", "san luis"
    ],
    "Región Pampeana": [
        "cordoba", "entre rios", "la pampa", "santa fe"
    ],
    "Patagonia": [
        "chubut", "neuquen", "rio negro", "santa cruz",
        "tierra del fuego", "tierra del fuego, antartida e islas del atlantico sur"
    ],
}

# --- Diccionario auxiliar: índice normalizado -> nombre real del índice ---
idx_norm_to_real = {norm_txt(i): i for i in base_df.index}

# --- Construir mapeo Territorio -> Región (solo si el territorio existe en base_df) ---
territorio_a_region = {}
for region, miembros in regiones_raw.items():
    for m in miembros:
        real = idx_norm_to_real.get(norm_txt(m))
        if real is not None:
            territorio_a_region[real] = region

# Evitar mapear "Argentina" a región (se mantendrá como País)
for nombre in list(idx_norm_to_real.values()):
    if norm_txt(nombre) == "argentina":
        territorio_a_region.pop(nombre, None)

# --- Añadir columna "Región" a base_df (para los territorios que mapean) ---
base_with_region = base_df.copy()
base_with_region.insert(
    0,
    "Región",
    [territorio_a_region.get(t, np.nan) for t in base_df.index]
)

# --- Calcular promedios regionales (media simple) ---
#   - Excluye filas sin región y "Argentina"
mask_valid_region = base_with_region["Región"].notna()
regional_means = (
    base_with_region.loc[mask_valid_region]
    .groupby("Región")
    .mean(numeric_only=True)
)

# --- Opcional: verificar qué territorios no fueron mapeados a ninguna región ---
territorios_no_mapeados = sorted(
    [t for t in base_df.index
     if t not in territorio_a_region and norm_txt(t) != "argentina"]
)

if territorios_no_mapeados:
    print("⚠️ Territorios no mapeados a ninguna región (revisar nombres):")
    for t in territorios_no_mapeados:
        print("  -", t)

# --- Armar índice jerárquico (Tipo, Nombre) y concatenar ---
# Territorios
territorial = base_df.copy()
territorial.index = pd.MultiIndex.from_product([["Territorio"], territorial.index])

# Regiones
regional_means_multi = regional_means.copy()
regional_means_multi.index = pd.MultiIndex.from_product([["Región"], regional_means_multi.index])

# País (si existe fila Argentina)
bloques = [territorial, regional_means_multi]
if any(norm_txt(x)== "argentina" for x in base_df.index):
    pais = base_df.loc[[idx_norm_to_real[norm_txt("argentina")]]].copy()
    pais.index = pd.MultiIndex.from_product([["País"], pais.index])
    bloques.append(pais)

base_df_reg = pd.concat(bloques, axis=0).sort_index()

# --- (Opcional) mantener orden de métricas por año: Abandono, Promoción, Repitencia ---
orden_metricas = ["Abandono", "Promoción", "Repitencia"]
cols = sorted(
    base_df_reg.columns,
    key=lambda x: (int(x[0]), orden_metricas.index(x[1]) if x[1] in orden_metricas else 99)
)
base_df_reg = base_df_reg.loc[:, cols]

# --- Vista rápida ---
print("Dimensiones con regiones (base_df_reg):", base_df_reg.shape)
display(base_df_reg.head(12))          # primeras 12 filas (verás Territorios y quizás alguna Región)
display(base_df_reg.loc[("Región", slice(None))].head())  # muestra algunas regiones

# Mostrar todas las regiones con sus tasas
regiones_df = base_df_reg.loc["Región"]

print("Dimensiones de regiones_df:", regiones_df.shape)
display(regiones_df)

# Guardar CSV y Excel en el Escritorio
ruta_csv = "C:/Users/Brenda/Desktop/base_df_reg.csv"
ruta_xlsx = "C:/Users/Brenda/Desktop/base_df_reg.xlsx"

# Guardar en CSV
base_df_reg.to_csv(ruta_csv, index=False, encoding="utf-8-sig")

# Guardar en Excel
ruta_xlsx = "C:/Users/Brenda/Desktop/base_df_reg.xlsx"
base_df_reg.to_excel(ruta_xlsx, index=True, engine="openpyxl")

print(f"Archivos guardados en:\n- {ruta_csv}\n- {ruta_xlsx}")

print(base_df_reg.columns.tolist())
print(base_df_reg.head(5))

print(df_reset.columns)
print(df_reset.head())

# Paso 1: resetear el índice
df_reset = base_df_reg.reset_index()

# Paso 2: aplanar MultiIndex de columnas a strings
df_reset.columns = [
    str(col[0]) if col[0] in ["level_0","level_1"] else f"{col[0]}_{col[1]}"
    for col in df_reset.columns
]

# Paso 3: renombrar las dos primeras columnas
df_reset = df_reset.rename(columns={"level_0": "Tipo", "level_1": "Nombre"})

print(df_reset.head())

df_long = df_reset.melt(
    id_vars=["Tipo","Nombre"],
    var_name="Año_Var",
    value_name="Valor"
)

# Separar año y variable
df_long[["Año","Variable"]] = df_long["Año_Var"].str.split("_", expand=True)

# Pivotear a formato ancho
df_long = df_long.pivot_table(
    index=["Tipo","Nombre","Año"],
    columns="Variable",
    values="Valor"
).reset_index()

# Convertir año a int
df_long["Año"] = df_long["Año"].astype(int)

print(df_long.head(12))

# Correlaciones por año
for year in sorted(df_long["Año"].unique()):
    sub = df_long[df_long["Año"] == year][["Abandono","Promoción","Repitencia"]]
    print(f"\n=== Correlaciones en {year} ===")
    print(sub.corr().round(2))

# Correlaciones por región
for region in df_long[df_long["Tipo"]=="Región"]["Nombre"].unique():
    sub = df_long[(df_long["Tipo"]=="Región") & (df_long["Nombre"]==region)][["Abandono","Promoción","Repitencia"]]
    print(f"\n=== Correlaciones en región {region} ===")
    print(sub.corr().round(2))

    import statsmodels.formula.api as smf

# Filtramos solo las regiones (excluimos el total País para no mezclar)
df_model = df_long[df_long["Tipo"]=="Región"].dropna()

# Ajustar modelo OLS: Abandono explicado por Promoción y Repitencia, con efectos fijos
modelo = smf.ols(
    "Abandono ~ Promoción + Repitencia + C(Año) + C(Nombre)",
    data=df_model
).fit()

print(modelo.summary())

import pandas as pd
import numpy as np
import matplotlib.pyplot as plt

# Extraer coeficientes y CIs
coefs = modelo.params
cis = modelo.conf_int()
pvals = modelo.pvalues

# Filtrar solo términos de región (C(Nombre)[T.X])
mask_reg = coefs.index.str.startswith("C(Nombre)[T.")
reg_effects = pd.DataFrame({
    "term": coefs.index[mask_reg],
    "coef": coefs[mask_reg].values,
    "ci_low": cis.loc[mask_reg, 0].values,
    "ci_high": cis.loc[mask_reg, 1].values,
    "pval": pvals[mask_reg].values
})

# Limpiar nombres → "C(Nombre)[T.Región Pampeana]" → "Región Pampeana"
reg_effects["region"] = reg_effects["term"].str.replace(r"C\(Nombre\)\[T\.", "", regex=True).str.replace("]", "", regex=False)

# Orden por coeficiente
reg_effects = reg_effects.sort_values("coef")

# Plot (matplotlib puro, 1 figura)
plt.figure(figsize=(8, 5))
y = np.arange(len(reg_effects))
plt.errorbar(reg_effects["coef"], y, xerr=[reg_effects["coef"]-reg_effects["ci_low"], reg_effects["ci_high"]-reg_effects["coef"]],
             fmt='o', capsize=4)
plt.axvline(0, linestyle="--")
plt.yticks(y, reg_effects["region"])
plt.xlabel("Efecto sobre Abandono (p.p.) respecto a base = Cuyo")
plt.title("Efectos fijos por región (controlando por año, promoción y repitencia)")
plt.tight_layout()
plt.show()

reg_effects

# Términos de año: C(Año)[T.2021], C(Año)[T.2022] (base = 2020)
mask_year = coefs.index.str.startswith("C(Año)[T.")
year_effects = pd.DataFrame({
    "term": coefs.index[mask_year],
    "coef": coefs[mask_year].values,
    "ci_low": cis.loc[mask_year, 0].values,
    "ci_high": cis.loc[mask_year, 1].values,
    "pval": pvals[mask_year].values
})
year_effects["año"] = year_effects["term"].str.extract(r"C\(Año\)\[T\.(\d+)\]").astype(int)

# Orden cronológico
year_effects = year_effects.sort_values("año")

# Plot
plt.figure(figsize=(6, 4))
x = np.arange(len(year_effects))
plt.errorbar(x, year_effects["coef"], yerr=[year_effects["coef"]-year_effects["ci_low"], year_effects["ci_high"]-year_effects["coef"]],
             fmt='o', capsize=4)
plt.axhline(0, linestyle="--")
plt.xticks(x, year_effects["año"])
plt.ylabel("Efecto sobre Abandono (p.p.) respecto a base = 2020")
plt.title("Efectos fijos de año (controlando región y tasas)")
plt.tight_layout()
plt.show()

year_effects

def stars(p):
    return "***" if p < 0.001 else "**" if p < 0.01 else "*" if p < 0.05 else "." if p < 0.1 else ""

# Unir regiones + años en una sola tabla
summary_tbl = (
    pd.concat([
        reg_effects.assign(grupo="Región", etiqueta=reg_effects["region"])[["grupo","etiqueta","coef","ci_low","ci_high","pval"]],
        year_effects.assign(grupo="Año", etiqueta=year_effects["año"].astype(str))[["grupo","etiqueta","coef","ci_low","ci_high","pval"]]
    ], ignore_index=True)
    .assign(sig=lambda d: d["pval"].map(stars))
    .rename(columns={"coef":"coef_pp","ci_low":"ci_inf","ci_high":"ci_sup"})
    .sort_values(["grupo","coef_pp"], ascending=[True, False])
)

summary_tbl

# Guardar la tabla resumen
summary_path = "C:/Users/Brenda/Desktop/efectos_regiones_y_anios.csv"
summary_tbl.to_csv(summary_path, index=False, encoding="utf-8-sig")

# Guardar gráficos (volver a dibujar y guardar)
# Regiones
plt.figure(figsize=(8, 5))
y = np.arange(len(reg_effects))
plt.errorbar(reg_effects["coef"], y, xerr=[reg_effects["coef"]-reg_effects["ci_low"], reg_effects["ci_high"]-reg_effects["coef"]],
             fmt='o', capsize=4)
plt.axvline(0, linestyle="--")
plt.yticks(y, reg_effects["region"])
plt.xlabel("Efecto sobre Abandono (p.p.) respecto a base = Cuyo")
plt.title("Efectos fijos por región")
plt.tight_layout()
plt.savefig("C:/Users/Brenda/Desktop/efectos_regionales.png", dpi=150)

# Años
plt.figure(figsize=(6, 4))
x = np.arange(len(year_effects))
plt.errorbar(x, year_effects["coef"], yerr=[year_effects["coef"]-year_effects["ci_low"], year_effects["ci_high"]-year_effects["coef"]],
             fmt='o', capsize=4)
plt.axhline(0, linestyle="--")
plt.xticks(x, year_effects["año"])
plt.ylabel("Efecto sobre Abandono (p.p.) respecto a base = 2020")
plt.title("Efectos fijos de año")
plt.tight_layout()
plt.savefig("C:/Users/Brenda/Desktop/efectos_anio.png", dpi=150)

print("✅ Guardados en el Escritorio:\n-",
      summary_path,
      "\n- C:/Users/Brenda/Desktop/efectos_regionales.png",
      "\n- C:/Users/Brenda/Desktop/efectos_anio.png")

      import seaborn as sns

# Usamos df_long (con columnas: Tipo, Nombre, Año, Abandono, Promoción, Repitencia)
df_regiones = df_long[df_long["Tipo"]=="Región"].dropna()

plt.figure(figsize=(10,6))
sns.lineplot(data=df_regiones, x="Año", y="Abandono", hue="Nombre", marker="o")
plt.title("Evolución del abandono escolar por región (2019-2022)")
plt.ylabel("Tasa de abandono (%)")
plt.xlabel("Año")
plt.legend(title="Región", bbox_to_anchor=(1.05, 1), loc="upper left")
plt.tight_layout()
plt.show()

df_prom = df_regiones.groupby("Nombre")["Abandono"].mean().reset_index()

plt.figure(figsize=(8,5))
sns.barplot(data=df_prom, x="Abandono", y="Nombre", palette="viridis")
plt.title("Promedio de abandono escolar por región (2019-2022)")
plt.xlabel("Tasa promedio de abandono (%)")
plt.ylabel("Región")
plt.tight_layout()
plt.show()

