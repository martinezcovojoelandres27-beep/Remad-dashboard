# Remad-dashboard
Análisis de siniestros marinos - DIMAR 2023
import pandas as pd
import plotly.express as px
import plotly.graph_objects as go
import gradio as gr
import re

COLOR_FONDO        = "#070b14"
COLOR_TARJETA      = "#0c1423"
COLOR_PANEL        = "#101c30"
COLOR_AZUL         = "#3db8ff"
COLOR_AZUL_CLARO   = "#60a5fa"
COLOR_DORADO       = "#f59e0b"
COLOR_VERDE        = "#10d9a0"
COLOR_ROJO         = "#f4476b"
COLOR_NARANJA      = "#ff8c42"
COLOR_MORADO       = "#a78bfa"
COLOR_TEXTO        = "#dce8ff"
COLOR_GRIS         = "#6b82a8"
FUENTE             = "Sora, sans-serif"
TEMA_PLOTLY        = "plotly_dark"

df = pd.read_excel("/content/Siniestros database.xlsx")
df.columns = [str(c).strip() for c in df.columns]
def limpiar_fallecidos(valor):
    if pd.isna(valor):
        return 0
    texto = str(valor).strip()
    if texto in ["0", "N.I."]:
        return 0
    try:
        return int(texto)
    except:
        return 0

df["Fallecidos"] = df["Cantidad Fallecidos / Desaparecidos"].apply(limpiar_fallecidos)
df["Causa"]    = df["Causa Principal Accidente"].str.strip().str.title()
df["Actividad"] = df["Actividad en Desarrollo"].str.strip().str.title()
def clasificar_riesgo(causa):
    causa = str(causa).lower()
    palabras_criticas   = ["volc", "zozobra", "hundim", "naufr", "ahog"]
    palabras_alto       = ["colis", "vara", "incend", "falla"]
    if any(p in causa for p in palabras_criticas):
        return "Critico"
    if any(p in causa for p in palabras_alto):
        return "Alto"
    return "Operacional"

df["Riesgo"] = df["Causa"].apply(clasificar_riesgo)

def clasificar_punto_ciego(valor):
    valor = str(valor).lower()
    if valor.startswith("alta"):
        return "Alta"
    if valor.startswith("moder"):
        return "Moderada"
    if valor.startswith("baja"):
        return "Baja"
    return "No aplica"

df["PuntoCiego"] = df["Punto Ciego de Oleaje"].apply(clasificar_punto_ciego)

def clasificar_sobrecarga(valor):
    valor = str(valor).lower()
    if "alta" in valor or "probable" in valor or "migrante" in valor:
        return "Riesgo"
    if "no aplica" in valor or "no report" in valor:
        return "No aplica"
    return "Sin riesgo"

df["Sobrecarga"] = df["Riesgo de Sobrecarga / Sobrecupo"].apply(clasificar_sobrecarga)

col_formalidad = [c for c in df.columns if "formalidad" in c.lower()][0]

def clasificar_formalidad(valor):
    valor = str(valor).lower()
    if "sin cert" in valor or "ilegal" in valor:
        return "Sin certificacion"
    if "extran" in valor:
        return "Extranjera"
    if "no aplica" in valor:
        return "No aplica"
    return "Certificada"

df["Formalidad"] = df[col_formalidad].apply(clasificar_formalidad)

col_condicion    = [c for c in df.columns if "condici" in c.lower()][0]
col_embarcacion  = [c for c in df.columns if "embarcaci" in c.lower()][0]
col_regimen      = [c for c in df.columns if "r" in c.lower() and "gimen" in c.lower()][0]

total_siniestros  = len(df)
total_fallecidos  = int(df["Fallecidos"].sum())
tasa_mortalidad   = round(total_fallecidos / total_siniestros * 100, 1)
pct_criticos      = round((df["Riesgo"] == "Critico").mean() * 100, 1)
pct_sin_cert      = round((df["Formalidad"] == "Sin certificacion").mean() * 100, 1)
pct_punto_ciego   = round((df["PuntoCiego"] == "Alta").mean() * 100, 1)

def aplicar_estilo(figura, altura=420, titulo=""):
    figura.update_layout(
        template        = TEMA_PLOTLY,
        paper_bgcolor   = "rgba(0,0,0,0)",
        plot_bgcolor    = "rgba(0,0,0,0)",
        font            = dict(family=FUENTE, color=COLOR_TEXTO, size=12),
        height          = altura,
        margin          = dict(l=12, r=20, t=50, b=12),
        legend          = dict(bgcolor="rgba(0,0,0,0)"),
        title           = dict(
            text   = titulo,
            font   = dict(size=13, color=COLOR_DORADO, family=FUENTE),
            x      = 0.02
        )
    )
    figura.update_xaxes(gridcolor="#1a2540", zeroline=False)
    figura.update_yaxes(gridcolor="#1a2540", zeroline=False)
    return figura

def normalizar_embarcacion(texto):
    texto = re.sub(r"\s*\(.*\)", "", str(texto)).strip().title()
    if re.search(r"lancha.*pesca|pesca.*artesanal", texto, re.I):
        return "Lancha / Pesca Artesanal"
    if re.search(r"lancha.*pasaje", texto, re.I):
        return "Lancha De Pasaje"
    return texto


# ─── Gráfica 1:
causas_top = df["Causa"].value_counts().reset_index().head(12)
causas_top.columns = ["Causa", "Casos"]
causas_top = causas_top.sort_values("Casos")

fig_causas = go.Figure(go.Bar(
    x             = causas_top["Casos"],
    y             = causas_top["Causa"],
    orientation   = "h",
    text          = causas_top["Casos"],
    textposition  = "outside",
    marker        = dict(
        color      = causas_top["Casos"],
        colorscale = [[0, "#0d2859"], [0.5, COLOR_AZUL], [1, COLOR_DORADO]],
        showscale  = False,
        line_width = 0
    )
))
aplicar_estilo(fig_causas, 450, "Causas Principales de Siniestros")


# ─── Gráfica 2: 
embarcaciones = df[col_embarcacion].apply(normalizar_embarcacion).value_counts().reset_index().head(10)
embarcaciones.columns = ["Tipo", "Casos"]

fig_embarcaciones = px.treemap(
    embarcaciones, path=["Tipo"], values="Casos", color="Casos",
    color_continuous_scale=[[0, "#0d2859"], [0.5, COLOR_AZUL], [1, COLOR_VERDE]]
)
fig_embarcaciones.update_traces(
    textinfo        = "label+value",
    textfont_size   = 13,
    marker_line_width = 1,
    marker_line_color = COLOR_FONDO
)
aplicar_estilo(fig_embarcaciones, 400, "Tipos de Embarcacion Mas Afectadas")


# ─── Gráfica 3: 
conteo_riesgo = df["Riesgo"].value_counts().reset_index()
conteo_riesgo.columns = ["Nivel", "N"]

fig_riesgo = go.Figure(go.Pie(
    labels     = conteo_riesgo["Nivel"],
    values     = conteo_riesgo["N"],
    hole       = 0.62,
    marker     = dict(
        colors = [COLOR_VERDE, COLOR_NARANJA, COLOR_ROJO],
        line   = dict(color=COLOR_FONDO, width=3)
    ),
    textinfo   = "label+percent",
    textfont   = dict(size=12, family=FUENTE)
))
fig_riesgo.add_annotation(
    text        = f"<b>{total_siniestros}</b><br>eventos",
    x=0.5, y=0.5,
    font_size   = 18,
    showarrow   = False,
    font_color  = COLOR_TEXTO,
    font_family = FUENTE
)
aplicar_estilo(fig_riesgo, 380, "Nivel de Riesgo Naval")


# ─── Gráfica 4: 
actividades = df["Actividad"].value_counts().reset_index().head(8)
actividades.columns = ["Actividad", "Casos"]

fig_actividad = go.Figure(go.Bar(
    x             = actividades["Actividad"],
    y             = actividades["Casos"],
    text          = actividades["Casos"],
    textposition  = "outside",
    marker        = dict(
        color      = actividades["Casos"],
        colorscale = [[0, "#0d2859"], [1, COLOR_AZUL_CLARO]],
        showscale  = False,
        line_width = 0
    )
))
fig_actividad.update_xaxes(tickangle=-30)
aplicar_estilo(fig_actividad, 380, "Actividad al Momento del Siniestro")


# ─── Gráfica 5: 
fallecidos_causa = df.groupby("Causa")["Fallecidos"].sum().reset_index()
fallecidos_causa = fallecidos_causa.sort_values("Fallecidos", ascending=False).head(10)
fallecidos_causa.columns = ["Causa", "Fallecidos"]
fallecidos_causa = fallecidos_causa[fallecidos_causa["Fallecidos"] > 0].sort_values("Fallecidos")

fig_fallecidos = go.Figure(go.Bar(
    x             = fallecidos_causa["Fallecidos"],
    y             = fallecidos_causa["Causa"],
    orientation   = "h",
    text          = fallecidos_causa["Fallecidos"],
    textposition  = "outside",
    marker        = dict(
        color      = fallecidos_causa["Fallecidos"],
        colorscale = [[0, "#3d0d1a"], [1, COLOR_ROJO]],
        showscale  = False,
        line_width = 0
    )
))
aplicar_estilo(fig_fallecidos, 380, "Fallecidos y Desaparecidos por Causa")


# ─── Gráfica 6: 
condicion_nave = df[col_condicion].str.strip().value_counts().reset_index().head(10)
condicion_nave.columns = ["Condicion", "Casos"]
condicion_nave = condicion_nave.sort_values("Casos")

def color_por_condicion(condicion):
    condicion = str(condicion).lower()
    if "pérdida" in condicion or "hurtada" in condicion:
        return COLOR_ROJO
    if "no apta" in condicion or "daño" in condicion:
        return COLOR_NARANJA
    if "operativa" in condicion:
        return COLOR_VERDE
    return COLOR_AZUL_CLARO

colores_condicion = [color_por_condicion(c) for c in condicion_nave["Condicion"]]

fig_condicion = go.Figure(go.Bar(
    x             = condicion_nave["Casos"],
    y             = condicion_nave["Condicion"],
    orientation   = "h",
    text          = condicion_nave["Casos"],
    textposition  = "outside",
    marker        = dict(color=colores_condicion, line_width=0)
))
aplicar_estilo(fig_condicion, 400, "Condicion Final de la Nave")


# ─── Gráfica 7: 
escenarios = df["Escenario Siniestro"].value_counts().reset_index()
escenarios.columns = ["Escenario", "Casos"]
escenarios["Sigla"] = escenarios["Escenario"].str.extract(r"^([A-Z]+)").fillna(escenarios["Escenario"])

fig_escenarios = go.Figure(go.Pie(
    labels       = escenarios["Sigla"],
    values       = escenarios["Casos"],
    hole         = 0.50,
    customdata   = escenarios["Escenario"],
    hovertemplate = "<b>%{customdata}</b><br>Casos: %{value}<extra></extra>",
    textinfo     = "label+percent",
    textfont     = dict(size=12, family=FUENTE),
    marker       = dict(
        colors = [COLOR_AZUL, COLOR_DORADO, COLOR_VERDE, COLOR_ROJO,
                  COLOR_NARANJA, COLOR_MORADO, COLOR_AZUL_CLARO, "#f472b6", "#34d399"],
        line   = dict(color=COLOR_FONDO, width=2)
    )
))
aplicar_estilo(fig_escenarios, 400, "Escenarios Operacionales DIMAR")


cruce_pc_form = df.groupby(["PuntoCiego", "Formalidad"]).size().reset_index(name="N")
tabla_calor = cruce_pc_form.pivot(index="PuntoCiego", columns="Formalidad", values="N").fillna(0)

fig_heatmap = go.Figure(go.Heatmap(
    z             = tabla_calor.values,
    x             = tabla_calor.columns.tolist(),
    y             = tabla_calor.index.tolist(),
    colorscale    = [[0, COLOR_TARJETA], [0.4, COLOR_AZUL], [1, COLOR_DORADO]],
    showscale     = True,
    text          = tabla_calor.values,
    texttemplate  = "%{text:.0f}",
    textfont      = dict(size=13, family=FUENTE)
))
aplicar_estilo(fig_heatmap, 360, "Punto Ciego x Formalidad de Tripulacion")

df_copia = df.copy()
df_copia["TipoNave"] = df[col_embarcacion].apply(normalizar_embarcacion)
top6_tipos = df_copia["TipoNave"].value_counts().head(6).index.tolist()
df_top6 = df_copia[df_copia["TipoNave"].isin(top6_tipos)]

sobrecarga_tipo = df_top6.groupby(["TipoNave", "Sobrecarga"]).size().reset_index(name="N")

fig_sobrecarga = px.bar(
    sobrecarga_tipo,
    x      = "TipoNave",
    y      = "N",
    color  = "Sobrecarga",
    barmode = "stack",
    color_discrete_map = {
        "Riesgo":    COLOR_ROJO,
        "Sin riesgo": COLOR_VERDE,
        "No aplica":  COLOR_NARANJA
    }
)
fig_sobrecarga.update_xaxes(title_text="", tickangle=-28)
fig_sobrecarga.update_yaxes(title_text="Casos")
aplicar_estilo(fig_sobrecarga, 400, "Riesgo de Sobrecarga por Embarcacion")
fig_sobrecarga.update_layout(font=dict(family=FUENTE))

regimenes = df[col_regimen].str.strip().value_counts().reset_index().head(10)
regimenes.columns = ["Regimen", "Casos"]
regimenes = regimenes.sort_values("Casos")

fig_regimen = go.Figure(go.Bar(
    x             = regimenes["Casos"],
    y             = regimenes["Regimen"],
    orientation   = "h",
    text          = regimenes["Casos"],
    textposition  = "outside",
    marker        = dict(
        color      = regimenes["Casos"],
        colorscale = [[0, "#1a0d4a"], [1, COLOR_MORADO]],
        showscale  = False,
        line_width = 0
    )
))
aplicar_estilo(fig_regimen, 400, "Regimen de Navegacion Normativo")

estilos_css = f"""
@import url('https://fonts.googleapis.com/css2?family=Sora:wght@300;400;600;700;800&display=swap');

*, *::before, *::after {{
    box-sizing: border-box;
    margin: 0;
    padding: 0;
}}

body, .gradio-container {{
    font-family: 'Sora', sans-serif !important;
    background: {COLOR_FONDO} !important;
}}

.gradio-container * {{
    font-family: 'Sora', sans-serif !important;
    color: {COLOR_TEXTO} !important;
}}

.remad-header {{
    background: linear-gradient(135deg, #060d20, #091733, #060d20);
    border-bottom: 1px solid {COLOR_AZUL}44;
    padding: 2rem 2.5rem 1.6rem;
    margin-bottom: 1.2rem;
}}

.remad-title {{
    font-size: 1.75rem;
    font-weight: 800;
    letter-spacing: 1px;
    line-height: 1.2;
}}

.remad-title span {{
    background: linear-gradient(90deg, {COLOR_DORADO}, {COLOR_AZUL_CLARO});
    -webkit-background-clip: text;
    -webkit-text-fill-color: transparent;
}}

.remad-sub {{
    font-size: .78rem;
    color: {COLOR_GRIS};
    letter-spacing: 1.5px;
    margin-top: .4rem;
}}

.remad-badge {{
    display: inline-block;
    background: {COLOR_AZUL}22;
    border: 1px solid {COLOR_AZUL}55;
    color: {COLOR_AZUL_CLARO} !important;
    font-size: .68rem;
    font-weight: 600;
    letter-spacing: 1.5px;
    text-transform: uppercase;
    padding: .2rem .7rem;
    border-radius: 99px;
    margin-top: .6rem;
    margin-right: .4rem;
}}

.kpi-grid {{
    display: grid;
    grid-template-columns: repeat(auto-fit, minmax(175px, 1fr));
    gap: .9rem;
    padding: 0 1rem 1.4rem;
}}

.kpi-card {{
    background: linear-gradient(145deg, {COLOR_TARJETA}, {COLOR_PANEL});
    border: 1px solid {COLOR_AZUL}30;
    border-top: 3px solid var(--accent);
    border-radius: 12px;
    padding: 1.1rem 1.2rem 1rem;
    cursor: default;
    position: relative;
    transition: border-color .2s, box-shadow .2s;
}}

.kpi-card:hover {{
    box-shadow: 0 0 24px var(--accent)33;
    border-color: var(--accent);
}}

.kpi-label {{
    font-size: .66rem;
    font-weight: 700;
    letter-spacing: 1.8px;
    text-transform: uppercase;
    color: {COLOR_GRIS} !important;
    margin-bottom: .5rem;
}}

.kpi-number {{
    font-size: 2.1rem;
    font-weight: 800;
    color: var(--accent) !important;
    line-height: 1;
}}

.kpi-sub {{
    font-size: .67rem;
    color: {COLOR_GRIS} !important;
    margin-top: .3rem;
    line-height: 1.4;
}}

.kpi-card .tooltip-texto {{
    visibility: hidden;
    opacity: 0;
    position: absolute;
    bottom: calc(100% + 8px);
    left: 50%;
    transform: translateX(-50%);
    background: #0e1e3a;
    border: 1px solid {COLOR_AZUL}66;
    color: {COLOR_TEXTO} !important;
    font-size: .72rem;
    line-height: 1.5;
    padding: .6rem .9rem;
    border-radius: 8px;
    width: 220px;
    z-index: 9999;
    transition: opacity .2s;
    pointer-events: none;
    box-shadow: 0 8px 24px #00000088;
}}

.kpi-card:hover .tooltip-texto {{
    visibility: visible;
    opacity: 1;
}}

.seccion-titulo {{
    font-size: .68rem;
    font-weight: 700;
    letter-spacing: 2.5px;
    text-transform: uppercase;
    color: {COLOR_DORADO} !important;
    padding: .8rem 1.2rem .4rem;
    border-left: 3px solid {COLOR_DORADO};
    margin-left: 1rem;
    margin-bottom: .4rem;
}}

.descripcion-grafica {{
    font-size: .78rem;
    line-height: 1.65;
    color: {COLOR_GRIS} !important;
    padding: .9rem 1rem 1rem;
    border-top: 1px solid {COLOR_AZUL}18;
    background: {COLOR_FONDO}cc;
}}

.descripcion-grafica strong {{
    color: {COLOR_DORADO} !important;
    font-weight: 600;
    display: block;
    margin-bottom: .2rem;
    font-size: .72rem;
    letter-spacing: 1px;
    text-transform: uppercase;
}}

.insights-grid {{
    display: grid;
    grid-template-columns: repeat(auto-fit, minmax(240px, 1fr));
    gap: .9rem;
    padding: 0 1rem 1rem;
}}

.insight-card {{
    background: linear-gradient(145deg, {COLOR_TARJETA}, {COLOR_PANEL});
    border-left: 4px solid {COLOR_AZUL_CLARO};
    border-radius: 10px;
    padding: 1rem 1.1rem;
    font-size: .82rem;
    line-height: 1.65;
}}

.insight-card.dorado  {{ border-left-color: {COLOR_DORADO}; }}
.insight-card.rojo    {{ border-left-color: {COLOR_ROJO}; }}
.insight-card.verde   {{ border-left-color: {COLOR_VERDE}; }}

.insight-titulo {{
    font-weight: 700;
    font-size: .7rem;
    letter-spacing: 1.5px;
    text-transform: uppercase;
    margin-bottom: .35rem;
    color: {COLOR_AZUL_CLARO} !important;
}}

.insight-card.dorado .insight-titulo {{ color: {COLOR_DORADO} !important; }}
.insight-card.rojo   .insight-titulo {{ color: {COLOR_ROJO} !important; }}
.insight-card.verde  .insight-titulo {{ color: {COLOR_VERDE} !important; }}

.pie-pagina {{
    text-align: center;
    padding: 1.2rem;
    font-size: .68rem;
    color: {COLOR_GRIS} !important;
    letter-spacing: 1.2px;
    border-top: 1px solid {COLOR_AZUL}22;
    margin-top: 1rem;
}}
"""


columnas_calculadas = ["Fallecidos", "Causa", "Actividad", "Riesgo", "PuntoCiego", "Sobrecarga", "Formalidad"]
df_tabla = df.drop(columns=columnas_calculadas, errors="ignore")

with gr.Blocks(theme=gr.themes.Base(), css=estilos_css) as dashboard:

    
    gr.HTML(f"""
    <div class="remad-header">
        <div style="display:flex; align-items:center; gap:1.5rem; flex-wrap:wrap">
            <img src="data:image/png;base64,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"
                 style="height:52px; width:auto; object-fit:contain; flex-shrink:0">
            <div style="flex:1; min-width:200px">
                <div class="remad-title">ANALISIS DE SINIESTROS EN COLOMBIA — REMAD</div>
                <div class="remad-sub">REGISTRO MARITIMO · DIMAR · COLMAR · {total_siniestros} siniestros analizados</div>
            </div>
            <div style="font-size:.75rem; color:{COLOR_GRIS}; text-align:right; line-height:1.8">
                Colombia · Cartagena de Indias<br>caribe colombiano
            </div>
        </div>
    </div>
    """)

    # KPIs
    gr.HTML(f"""
    <div class="kpi-grid">
        <div class="kpi-card" style="--accent:{COLOR_AZUL}">
            <div class="kpi-label">Siniestros Analizados</div>
            <div class="kpi-number">{total_siniestros}</div>
            <div class="kpi-sub">Base consolidada COLMAR</div>
            <div class="tooltip-texto">Total de eventos maritimos registrados en la base COLMAR para embarcaciones en territorio colombiano.</div>
        </div>
        <div class="kpi-card" style="--accent:{COLOR_ROJO}">
            <div class="kpi-label">Fallecidos / Desaparecidos</div>
            <div class="kpi-number">{total_fallecidos}</div>
            <div class="kpi-sub">Victimas confirmadas</div>
            <div class="tooltip-texto">Personas fallecidas o desaparecidas confirmadas. Excluye casos marcados como N.I. (No Informado).</div>
        </div>
        <div class="kpi-card" style="--accent:{COLOR_NARANJA}">
            <div class="kpi-label">Tasa de Mortalidad</div>
            <div class="kpi-number">{tasa_mortalidad}%</div>
            <div class="kpi-sub">Fallecidos sobre total eventos</div>
            <div class="tooltip-texto">Porcentaje de siniestros que resultaron en al menos una victima fatal o desaparecida sobre el total de eventos registrados.</div>
        </div>
        <div class="kpi-card" style="--accent:{COLOR_ROJO}">
            <div class="kpi-label">Eventos Criticos</div>
            <div class="kpi-number">{pct_criticos}%</div>
            <div class="kpi-sub">Volcamiento · naufragio · ahogamiento</div>
            <div class="tooltip-texto">Siniestros clasificados como criticos: volcamiento, naufragio, zozobra o ahogamiento. Son los de mayor riesgo de perdida de vidas.</div>
        </div>
        <div class="kpi-card" style="--accent:{COLOR_DORADO}">
            <div class="kpi-label">Sin Certificacion</div>
            <div class="kpi-number">{pct_sin_cert}%</div>
            <div class="kpi-sub">Tripulacion no certificada</div>
            <div class="tooltip-texto">Tripulaciones que operaban sin certificacion vigente o con operacion ilegal al momento del siniestro registrado.</div>
        </div>
        <div class="kpi-card" style="--accent:{COLOR_MORADO}">
            <div class="kpi-label">Punto Ciego Alto</div>
            <div class="kpi-number">{pct_punto_ciego}%</div>
            <div class="kpi-sub">Zona sin cobertura SAR/VHF</div>
            <div class="tooltip-texto">Eventos ocurridos en zonas de alta exposicion sin cobertura SAR plena, sin VHF o sin vigilancia efectiva institucional.</div>
        </div>
    </div>
    """)

    gr.HTML('<div class="seccion-titulo">Causas y Tipologia de Siniestros</div>')
    with gr.Row():
        with gr.Column(scale=3):
            gr.Plot(fig_causas)
            gr.HTML('<div class="descripcion-grafica"><strong>Que muestra</strong>Ranking de las causas mas frecuentes de siniestros en el caribe colombiano. El volcamiento encabeza la lista y es la causa mas critica para embarcaciones menores. Cada barra es el total de eventos por causa.</div>')
        with gr.Column(scale=2):
            gr.Plot(fig_embarcaciones)
            gr.HTML('<div class="descripcion-grafica"><strong>Que muestra</strong>Distribucion de siniestros por tipo de embarcacion. Las areas mas grandes indican mayor numero de eventos. Lanchas de pesca artesanal y de pasaje concentran la mayor vulnerabilidad operacional.</div>')

    gr.HTML('<div class="seccion-titulo">Clasificacion de Riesgo y Actividad Operacional</div>')
    with gr.Row():
        with gr.Column(scale=1):
            gr.Plot(fig_riesgo)
            gr.HTML('<div class="descripcion-grafica"><strong>Que muestra</strong>Clasificacion en tres niveles. Critico: volcamiento, naufragio, ahogamiento. Alto: colision, varada, incendio, falla. Operacional: otros eventos. El numero central es el total de siniestros.</div>')
        with gr.Column(scale=2):
            gr.Plot(fig_actividad)
            gr.HTML('<div class="descripcion-grafica"><strong>Que muestra</strong>Actividad que realizaba la embarcacion cuando ocurrio el siniestro. Transporte de personas y pesca son las mas frecuentes, indicando donde debe concentrarse la prevencion.</div>')
        with gr.Column(scale=2):
            gr.Plot(fig_escenarios)
            gr.HTML('<div class="descripcion-grafica"><strong>Que muestra</strong>Distribucion por escenario operacional DIMAR. COLMAR = embarcaciones nacionales en mar, EXTMAR = extranjeras, SARCOL = busqueda y salvamento, MIGCOL = migrantes. Pase el cursor para ver nombre completo.</div>')

    gr.HTML('<div class="seccion-titulo">Impacto Humano</div>')
    with gr.Row():
        with gr.Column(scale=2):
            gr.Plot(fig_fallecidos)
            gr.HTML('<div class="descripcion-grafica"><strong>Que muestra</strong>Acumulado de fallecidos y desaparecidos agrupados por causa. Permite identificar cuales causas no solo son frecuentes sino tambien las mas letales en el caribe colombiano.</div>')
        with gr.Column(scale=2):
            gr.Plot(fig_condicion)
            gr.HTML('<div class="descripcion-grafica"><strong>Que muestra</strong>Estado final de la embarcacion tras el siniestro. Rojo = perdida total, naranja = no apta o con danos, verde = operativa. Refleja la severidad material del evento registrado.</div>')

    gr.HTML('<div class="seccion-titulo">Factores de Riesgo Operacional</div>')
    with gr.Row():
        with gr.Column(scale=2):
            gr.Plot(fig_heatmap)
            gr.HTML('<div class="descripcion-grafica"><strong>Que muestra</strong>Cruce entre cobertura SAR/VHF (Punto Ciego) y formalidad de la tripulacion. Las celdas mas brillantes son las combinaciones de mayor riesgo: tripulaciones sin certificacion en zonas sin vigilancia.</div>')
        with gr.Column(scale=2):
            gr.Plot(fig_sobrecarga)
            gr.HTML('<div class="descripcion-grafica"><strong>Que muestra</strong>Riesgo de sobrecarga o sobrecupo por tipo de embarcacion. Barras rojas indican riesgo confirmado o probable. La sobrecarga es un factor directo en la perdida de estabilidad transversal.</div>')

    with gr.Row():
        with gr.Column():
            gr.Plot(fig_regimen)
            gr.HTML('<div class="descripcion-grafica"><strong>Que muestra</strong>Categoria normativa bajo la cual operaba la embarcacion al momento del siniestro. La mayoria de eventos ocurre en Aguas No Protegidas, zonas de mayor exposicion al oleaje y menor supervision institucional.</div>')

    gr.HTML('<div class="seccion-titulo">Interpretacion Estrategica</div>')
    gr.HTML(f"""
    <div class="insights-grid">
        <div class="insight-card">
            <div class="insight-titulo">Causa Dominante</div>
            El volcamiento encabeza los eventos criticos y es directamente prevenible mediante calculo de estabilidad transversal en tiempo real, el corazon del software REMAD en desarrollo.
        </div>
        <div class="insight-card dorado">
            <div class="insight-titulo">Sobrecarga Critica</div>
            Un numero significativo de siniestros presenta sobrecupo, especialmente en lanchas de pasaje y pesca artesanal. El sistema debe calcular el limite de carga segura segun condicion del mar.
        </div>
        <div class="insight-card rojo">
            <div class="insight-titulo">Punto Ciego de Oleaje</div>
            El {pct_punto_ciego}% de eventos ocurre en zonas sin cobertura SAR ni VHF plena. La integracion de alertas meteorologicas en tiempo real es prioritaria para la plataforma REMAD.
        </div>
        <div class="insight-card verde">
            <div class="insight-titulo">Vision de la Plataforma</div>
            Simulacion dinamica de estabilidad, alertas de volcamiento, Digital Twin naval, integracion meteorologica, monitoreo en tiempo real y reduccion de perdidas humanas en el caribe colombiano.
        </div>
    </div>
    """)

    gr.HTML('<div class="seccion-titulo">Base de Datos Completa</div>')
    gr.Dataframe(df_tabla, wrap=True)
    gr.HTML(f'<div class="pie-pagina">ANALISIS DE SINIESTROS EN COLOMBIA — REMAD · Registro Maritimo · DIMAR · Cartagena de Indias · {total_siniestros} siniestros analizados</div>')

dashboard.launch(share=True)
