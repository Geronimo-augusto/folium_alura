# Análise geoespacial — imóveis no Rio

Notebook de análise: cruza setores censitários com anúncios imobiliários e desenha mapas interativos do Rio de Janeiro.

Curso Alura (GeoPandas / Folium), usado no currículo como evidência de manipulação espacial — não como produto.

## O que o notebook faz

- Lê shapefile de setores censitários (`33SEE250GC_SIR.shp`)
- Lê CSV de listagens (`dados.csv`, separado por tabulação)
- Associa cada imóvel a um bairro
- Calcula estatísticas de preço e área
- Gera mapas Folium:
  - heatmap de densidade
  - marcadores clusterizados
  - choropleth da média de preço por bairro

## Stack

Python · GeoPandas · Pandas · Folium (`HeatMap`, `MarkerCluster`, `Fullscreen`)

## Como rodar

```bash
git clone https://github.com/Geronimo-augusto/folium_alura.git
cd folium_alura
python -m venv .venv
source .venv/bin/activate
pip install geopandas pandas folium jupyter
jupyter notebook
```

Abrir o `.ipynb` e executar de cima a baixo. Os shapefiles precisam estar no diretório indicado pelo notebook.

## Resultado esperado

Um HTML Folium com:

- mapa-base da cidade
- calor dos pontos
- clusters clicáveis
- cor por média de preço do bairro

## Próximos passos

- Exportar o HTML final para `/docs` e linkar no README
- Uma célula no topo com as 3 conclusões numéricas (preço mediano, bairro mais caro, n de pontos)
