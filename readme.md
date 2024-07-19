# Análise e Visualização de Dados Imobiliários no Rio de Janeiro

Este projeto visa analisar e visualizar dados imobiliários da cidade do Rio de Janeiro utilizando `geopandas` e `folium`.

## Requisitos

Certifique-se de ter os seguintes pacotes Python instalados:
- `geopandas`
- `pandas`
- `folium`
- `folium.plugins`

Você pode instalar esses pacotes usando `pip`:
```bash
pip install geopandas pandas folium
```

## Dados Utilizados
- Shapefile dos setores censitários do Rio de Janeiro.
- Arquivo CSV contendo dados imobiliários.
## Estrutura do Código
1.Importação das Bibliotecas:

```python
import geopandas as gpd
import pandas as pd
import folium
from folium.plugins import HeatMap, MarkerCluster, Fullscreen
```
2. Carregamento dos Dados:

- Carrega o shapefile e o arquivo CSV.
```python
rj = gpd.read_file("33SEE250GC_SIR.shp")
imoveis = pd.read_csv("dados.csv", sep="\t")
```
3. Filtragem e Preparação dos Dados:

- Filtra os dados do Rio de Janeiro e cria geometrias necessárias.
```python
cidade_rio = rj[rj["NM_MUNICIP"] == "RIO DE JANEIRO"]
cidade_rio_copy = cidade_rio.copy()
cidade = cidade_rio_copy.dissolve(by="NM_MUNICIP")
bairros_rio = cidade_rio_copy.dissolve(by="NM_BAIRRO")
imoveis = gpd.GeoDataFrame(imoveis, geometry=gpd.points_from_xy(imoveis["Longitude"], imoveis["Latitude"]))
imoveis = imoveis[imoveis["geometry"].within(cidade["geometry"].iloc[0])]
imoveis.insert(0, "cor", pd.qcut(imoveis["Valor"], q=[0, 0.5, 0.75, 1], labels=["red", "orange", "green"]))
imoveis.insert(0, "NM_BAIRRO", "")
```
4. Atribuição de Bairros aos Imóveis:

- Atribui a cada imóvel o bairro correspondente.
```python
bairros_rio_copy = bairros_rio.copy()
bairro_rio = bairros_rio_copy.reset_index()
geo_bairros = bairro_rio[["NM_BAIRRO", "geometry"]]
for index in range(len(bairro_rio)):
    imoveis.loc[imoveis["geometry"].within(bairro_rio["geometry"].iloc[index]), "NM_BAIRRO"] = bairro_rio["NM_BAIRRO"].iloc[index]
```
5. Cálculo de Estatísticas por Bairro:

- Calcula estatísticas descritivas dos imóveis por bairro.
```python
estatisticas_bairros = imoveis.groupby("NM_BAIRRO")[["Valor", "Tipo", "Area"]].agg({"Valor": ['min', "mean", "max"], "Tipo": "count", "Area": ["min", "max"]})
estatisticas_bairros = estatisticas_bairros.droplevel(level=0, axis=1).reset_index()
estatisticas_bairros.columns = ["NM_BAIRRO", "preco_min", "preco_medio", "preco_max", "qtd_imoveis", "area_min", "area_max"]
estatisticas_bairros = gpd.GeoDataFrame(estatisticas_bairros.merge(geo_bairros, on="NM_BAIRRO", how="left"))
```
6. Criação de Mapas:

- Funções para criar diferentes tipos de mapas usando `folium`.
### Mapa Base:

```python
def mapa(cidade, bairros_rio):
    mapa_rio = folium.Map(location=[-22, -43], zoom_start=8, tiles="Cartodb Positron")
    folium.GeoJson(bairros_rio).add_to(mapa_rio)
    mapa_rio_1 = folium.Map(location=[-22, -43], zoom_start=8, tiles="Cartodb Positron")
    folium.GeoJson(cidade).add_to(mapa_rio_1)
    return mapa_rio, mapa_rio_1
```
### Mapa de Calor:

```python
def mapa_calor(imoveis, bairros_rio):
    mapa_rio_calor = folium.Map(location=[imoveis["Latitude"].mean(), imoveis["Longitude"].mean()], zoom_start=10, tiles="Cartodb dark_matter")
    estilo = {"fillOpacity": 0, "color": "#ffffff", "weight": 0.5}
    HeatMap(data=imoveis[["Latitude", "Longitude"]], name="Mapa de Calor", radius=20).add_to(mapa_rio_calor)
    folium.GeoJson(bairros_rio, name="Rio de Janeiro", style_function=lambda x: estilo).add_to(mapa_rio_calor)
    folium.TileLayer("Cartodb Positron", name="Positron").add_to(mapa_rio_calor)
    folium.LayerControl().add_to(mapa_rio_calor)
    return mapa_rio_calor
```
### Mapa de Calor por Bairros:

```python
def mapa_calor_bairros(imoveis, bairros_rio):
    mapa_rio_calor_bairro = folium.Map(location=[imoveis["Latitude"].mean(), imoveis["Longitude"].mean()], zoom_start=10, tiles="Cartodb dark_matter")
    estilo = {"fillOpacity": 0, "color": "#ffffff", "weight": 0.5}
    HeatMap(data=imoveis[["Latitude", "Longitude"]], name="Mapa de Calor", radius=20).add_to(mapa_rio_calor_bairro)
    for indice, linha in bairros_rio.iterrows():
        bairro = gpd.GeoDataFrame(pd.DataFrame(linha).T, geometry="geometry", crs="EPSG:4674")
        folium.GeoJson(bairro, name=bairro.index[0], style_function=lambda x: estilo, tooltip=bairro.index[0]).add_to(mapa_rio_calor_bairro)
    folium.LayerControl().add_to(mapa_rio_calor_bairro)
    mapa_rio_calor_bairro.save("mapa_calor_rio.html")
    return mapa_rio_calor_bairro
```
### Mapa com Marcadores e Cluster:

```python
def mapa_pin_cluster(imoveis, bairros_rio):
    mapa_rio_pin_cluster = folium.Map(location=[imoveis["Latitude"].mean(), imoveis["Longitude"].mean()], zoom_start=10, tiles="Cartodb dark_matter")
    estilo = {"fillOpacity": 0, "color": "#ffffff", "weight": 0.5}
    for indice, linha in bairros_rio.iterrows():
        bairro = gpd.GeoDataFrame(pd.DataFrame(linha).T, geometry="geometry", crs="EPSG:4674")
        folium.GeoJson(bairro, name=bairro.index[0], style_function=lambda x: estilo, tooltip=bairro.index[0]).add_to(mapa_rio_pin_cluster)
    cluster = MarkerCluster()
    imoveis.apply(
        lambda linha: folium.Marker(
            location=[linha["Latitude"], linha["Longitude"]],
            icon=folium.Icon(color=linha["cor"], icon="fa-home", prefix="fa"),
            popup=folium.Popup(f'''<b>Bairro</b>: {linha["NM_BAIRRO"]}<br>
                                   <b>Área</b>: {linha["Area"]} m²<br>
                                   <b>Valor</b>: R$ {linha["Valor"]/1000} k <br>
                                   <b>Quartos</b>: {linha["Quartos"]}''',
                               max_width=200, sticky=True)
        ).add_to(cluster), axis=1
    )
    cluster.add_to(mapa_rio_pin_cluster)
    folium.LayerControl().add_to(mapa_rio_pin_cluster)
    mapa_rio_pin_cluster.save("mapa_calor_rio.html")
    return mapa_rio_pin_cluster
```    
### Mapa Final com Análises:

```python
def mapa_final(imoveis, bairro_rio, estatisticas_bairros):
    mapa_final = folium.Map(location=[imoveis["Latitude"].mean(), imoveis["Longitude"].mean()], zoom_start=10, tiles="Cartodb Positron")
    folium.Choropleth(
        geo_data=bairro_rio,
        data=estatisticas_bairros,
        columns=["NM_BAIRRO", "preco_medio"],
        key_on="feature.properties.NM_BAIRRO",
        fill_color="YlOrRd",
        nan_fill_color="white",
        bins=10,
        highlight=True,
        legend_name="Média do valor do imóvel"
    ).add_to(mapa_final)
    cluster = MarkerCluster()
    imoveis.apply(
        lambda linha: folium.Marker(
            location=[linha["Latitude"], linha["Longitude"]],
            icon=folium.Icon(color=linha["cor"], icon="fa-home", prefix="fa"),
            popup=folium.Popup(f'''<b>Bairro</b>: {linha["NM_BAIRRO"]}<br>
                                   <b>Área</b>: {linha["Area"]} m²<br>
                                   <b>Valor</b>: R$ {linha["Valor"]/1000} k <br>
                                   <b>Quartos</b>: {linha["Quartos"]}''',
                               max_width=200, sticky=True)
        ).add_to(cluster), axis=1
    )
    cluster.add_to(mapa_final)
    Fullscreen().add_to(mapa_final)
    style_function = lambda x: {'fillColor': "#ffffff", "color": "#000000", "fillOpacity": 0.1, "weight": 0.1}
    highlight_function = lambda x: {"fillColor": "#000000", "color": "#000000", "fillOpacity": 0.5, "weight": 0.1}
    config = folium.features.GeoJson(
        estatisticas_bairros,
        style_function=style_function,
        highlight_function=highlight_function,
        tooltip=folium.features.GeoJsonTooltip(
            fields=["NM_BAIRRO", "preco_min", "preco_medio", "preco_max", "qtd_imoveis", "area_min", "area_max"],
            aliases=["Bairro: ", "Preço mínimo: ", "Preço médio: ", "Preço máximo: ", "Quantidade de imóveis", "Área mínima: ", "Área máxima: "],
            style=("background-color: #f0f0f0; border: 1px solid black; border-radius: 3px; box-shadow: 3px;")
        )
    )
    config.add_to(mapa_final)
    mapa_final.save("mapa_final.html")
    return mapa_final
```
## Uso
Para executar o código, certifique-se de que os arquivos de dados estejam no diretório correto e execute o script Python. O script gerará mapas que podem ser visualizados em um navegador da web.

1. Mapas Básicos:

```python
mapa_rio, mapa_rio_1 = mapa(cidade, bairros_rio)
mapa_rio.save("mapa_rio.html")
mapa_rio_1.save("mapa_rio_1.html")
```
2. Mapa de Calor:

```python
mapa_rio_calor = mapa_calor(imoveis, bairros_rio)
mapa_rio_calor.save("mapa_calor.html")
```
3. Mapa de Calor por Bairros:

```python
mapa_rio_calor_bairro = mapa_calor_bairros(imoveis, bairros_rio)
```
4. Mapa com Cluster de Marcadores:

```python
mapa_rio_pin_cluster = mapa_pin_cluster(imoveis, bairros_rio)
```
5. Mapa Final com Análises:

```python
mapa_final = mapa_final(imoveis, bairro_rio, estatisticas_bairros)
```