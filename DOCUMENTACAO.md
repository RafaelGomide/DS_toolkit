# DS Toolkit — Documentação

Caixa de ferramentas para o dia a dia do Cientista de Dados: **66 funções**
reutilizáveis e parametrizáveis, organizadas em 7 seções.

## Instalação e requisitos

Coloque `ds_toolkit.py` na pasta do seu projeto (ou em um diretório no
`PYTHONPATH`) e importe:

```python
import ds_toolkit as dst
```

Dependências principais (obrigatórias):

```
pip install pandas numpy scipy scikit-learn matplotlib seaborn
```

Dependências opcionais (só necessárias para funções específicas):

```
pip install requests      # carregar_api
pip install sqlalchemy    # carregar_sql com bancos além de sqlite
pip install joblib        # salvar_modelo / carregar_modelo
pip install pyarrow       # ler/salvar .parquet e .feather
pip install openpyxl      # ler/salvar .xlsx
pip install lifelines     # seção 7 — Análise de Sobrevivência
```

## Convenções do módulo

- **Imutabilidade**: nenhuma função altera o DataFrame original — todas
  trabalham sobre cópia e **retornam** o resultado. Sempre reatribua:
  `df = dst.tratar_nulos(df, "mediana")`.
- **`verbose=True`** (padrão): imprime um relatório legível do que foi feito.
  Use `verbose=False` em scripts/pipelines silenciosos.
- **Funções de plot** retornam `(fig, ax)` para customização posterior e
  aceitam `salvar_em="caminho/figura.png"`.
- **`random_state=42`** por padrão em tudo que envolve aleatoriedade —
  reprodutibilidade garantida.
- Suporte a **dados brasileiros**: `decimal_brasileiro=True` na ingestão e
  conversão ("R$ 1.234,56"), `dayfirst=True` em datas (dd/mm/aaaa).

## Fluxo de trabalho típico

```python
import ds_toolkit as dst

# 1. Ingestão
df = dst.carregar_dados("dados.csv", decimal_brasileiro=True)

# 2. ETL
df = dst.limpar_nomes_colunas(df)
dst.relatorio_qualidade(df)                       # diagnóstico
df = dst.converter_tipos(df, colunas_data=["data"], colunas_numericas=["preco"])
df = dst.tratar_duplicados(df)
df = dst.tratar_nulos(df, "mediana", criar_indicador=True)
df = dst.remover_outliers(df, ["preco"], metodo="iqr", acao="limitar")

# 3. EDA
dst.resumo_geral(df)
dst.analise_alvo(df, "preco")                     # ranking de features
dst.comparar_grupos(df, "preco", "bairro")        # teste estatístico automático

# 4. Gráficos
dst.configurar_estilo("escuro")
dst.plot_distribuicao(df, "preco", log_x=True)
dst.plot_correlacao(df, metodo="spearman")

# 5. ML
X_tr, X_te, y_tr, y_te = dst.preparar_dados(df, "preco")
pre = dst.pipeline_preprocessamento(X_tr)
ranking = dst.comparar_modelos(X_tr, y_tr, "regressao", preprocessador=pre)
# ... treina o melhor, então:
dst.avaliar_regressao(modelo, X_te, y_te)
dst.salvar_modelo(modelo, "modelos/final.joblib", metadados={"rmse": 123})
```

## Fluxo de Análise de Sobrevivência

```python
# Formato padrão: coluna de TEMPO (duração > 0) e de EVENTO (1=ocorreu, 0=censura)
df = dst.preparar_sobrevivencia(df, "tempo_meses", "status",
                                mapa_evento={"Dead": 1, "Alive": 0})

# Não-paramétrico: curvas, medianas e log-rank entre grupos
res = dst.kaplan_meier(df, "tempo_meses", "status", coluna_grupo="tratamento")
dst.risco_acumulado(df, "tempo_meses", "status")     # diagnóstico da forma do risco

# Semi-paramétrico: Cox com hazard ratios e checagem de premissas
cph = dst.cox_ph(df, "tempo_meses", "status", ["idade", "biomarcador"])

# Alta dimensão (p >> n): Cox-LASSO com CV — seleção de variáveis
res = dst.cox_lasso(df, "tempo_meses", "status", lista_de_genes)
res["selecionadas"]

# Paramétrico: Exponencial vs Weibull vs Log-Normal vs Log-Logístico por AIC
dst.modelos_parametricos_sobrevivencia(df, "tempo_meses", "status")

# Previsão individual: S(t) para novos casos
dst.prever_sobrevivencia(cph, df_novos, tempos=[6, 12, 24])
```

---

## Índice de funções
 
 
**1. Ingestão de Dados**
 
- [`carregar_dados`](#carregar_dados) — Carrega um arquivo de dados detectando o formato pela extensão
- [`carregar_multiplos`](#carregar_multiplos) — Lê todos os arquivos que casam com um padrão glob e concatena em um único DataFrame
- [`carregar_sql`](#carregar_sql) — Executa uma query SQL e retorna DataFrame
- [`carregar_api`](#carregar_api) — Consome uma API REST (GET) com retry automático e devolve DataFrame
- [`salvar_dados`](#salvar_dados) — Salva um DataFrame no formato deduzido pela extensão do caminho
  
**2. ETL — Limpeza e Transformação**
 
- [`relatorio_qualidade`](#relatorio_qualidade) — Gera um diagnóstico completo de qualidade dos dados, coluna a coluna
- [`limpar_nomes_colunas`](#limpar_nomes_colunas) — Padroniza nomes de colunas: remove acentos, espaços e caracteres especiais
- [`converter_tipos`](#converter_tipos) — Converte tipos de colunas de forma robusta, com foco em dados brasileiros
- [`tratar_nulos`](#tratar_nulos) — Trata valores nulos com a estratégia escolhida
- [`tratar_duplicados`](#tratar_duplicados) — Remove linhas duplicadas com relatório do que foi removido
- [`remover_outliers`](#remover_outliers) — Detecta e trata outliers em colunas numéricas
- [`otimizar_memoria`](#otimizar_memoria) — Reduz o uso de memória do DataFrame com downcast de tipos
- [`codificar_categoricas`](#codificar_categoricas) — Codifica variáveis categóricas para uso em modelos
- [`escalar`](#escalar) — Escala/transforma colunas numéricas e retorna também o scaler ajustado
- [`criar_features_data`](#criar_features_data) — Extrai features de uma coluna datetime (feature engineering temporal)
- [`padronizar_texto`](#padronizar_texto) — Padroniza VALORES de colunas de texto (não os nomes — p/ isso use
- [`criar_faixas`](#criar_faixas) — Discretiza uma variável numérica em faixas (binning)
- [`mesclar_seguro`](#mesclar_seguro) — Merge com diagnóstico completo — o antídoto contra joins silenciosamente errados
  
**3. EDA — Análise Exploratória**
 
- [`resumo_geral`](#resumo_geral) — Describe turbinado: estatísticas + assimetria, curtose, CV e nulos
- [`analise_correlacao`](#analise_correlacao) — Matriz de correlação + relatório dos pares mais correlacionados
- [`testar_normalidade`](#testar_normalidade) — Testa normalidade com Shapiro-Wilk, D'Agostino e Kolmogorov-Smirnov
- [`comparar_grupos`](#comparar_grupos) — Compara uma variável numérica entre grupos escolhendo o teste correto
- [`analise_categorica`](#analise_categorica) — Associação entre duas variáveis categóricas: qui-quadrado + Cramér's V
- [`analise_univariada`](#analise_univariada) — Perfil completo de uma única coluna, numérica ou categórica
- [`analise_alvo`](#analise_alvo) — Ranqueia TODAS as features pela força de associação com o alvo,
  
**4. Gráficos**
 
- [`configurar_estilo`](#configurar_estilo) — Configura o estilo global de todos os plots do módulo (e do notebook)
- [`plot_distribuicao`](#plot_distribuicao) — Histograma + KDE + boxplot alinhados — visão completa de uma numérica
- [`plot_correlacao`](#plot_correlacao) — Heatmap de correlação com máscara triangular (sem redundância visual)
- [`plot_categorico`](#plot_categorico) — Gráfico de barras de contagem com % anotada e limite de categorias
- [`plot_dispersao`](#plot_dispersao) — Dispersão com linha de tendência e correlação anotada no título
- [`plot_boxplots_grupo`](#plot_boxplots_grupo) — Boxplots de uma numérica por grupo, ordenados pela mediana
- [`plot_serie_temporal`](#plot_serie_temporal) — Linha temporal com reamostragem e média móvel opcionais
- [`plot_nulos`](#plot_nulos) — Barra horizontal com % de nulos por coluna (visão rápida de faltantes)
- [`plot_qq`](#plot_qq) — QQ-plot contra a normal — o complemento visual de `testar_normalidade`
- [`plot_pares`](#plot_pares) — Pairplot (matriz de dispersões) com amostragem automática
  
**5. Machine Learning**
 
- [`preparar_dados`](#preparar_dados) — Separa X/y e faz train/test split com estratificação inteligente
- [`pipeline_preprocessamento`](#pipeline_preprocessamento) — Monta um ColumnTransformer sklearn com imputação + escala + one-hot
- [`avaliar_classificacao`](#avaliar_classificacao) — Avaliação completa de um classificador já treinado
- [`avaliar_regressao`](#avaliar_regressao) — Avaliação completa de um regressor já treinado
- [`comparar_modelos`](#comparar_modelos) — Compara vários modelos baseline via validação cruzada
- [`otimizar_hiperparametros`](#otimizar_hiperparametros) — Busca de hiperparâmetros com GridSearchCV ou RandomizedSearchCV
- [`importancia_features`](#importancia_features) — Ranking de importância de features de um modelo treinado
- [`avaliar_clustering`](#avaliar_clustering) — Ajuda a escolher o k do KMeans: inércia (elbow) + silhouette por k
- [`salvar_modelo`](#salvar_modelo) — Persiste um modelo (ou pipeline) com joblib + metadados opcionais
- [`carregar_modelo`](#carregar_modelo) — Carrega modelo salvo com `salvar_modelo` e avisa se a versão do
- [`curva_aprendizado`](#curva_aprendizado) — Curva de aprendizado — diagnostica overfitting vs underfitting
- [`selecionar_features`](#selecionar_features) — Seleciona as k melhores features (exige X totalmente numérico e sem nulos)
- [`reduzir_dimensionalidade`](#reduzir_dimensionalidade) — PCA com relatório de variância explicada
- [`balancear_classes`](#balancear_classes) — Balanceia classes por reamostragem simples (sem dependência do imblearn)
  
**6. Matemática / Estatística**
 
- [`intervalo_confianca`](#intervalo_confianca) — Intervalo de confiança para média (t de Student), proporção (Wilson)
- [`bootstrap_estatistica`](#bootstrap_estatistica) — IC bootstrap (percentil) para QUALQUER estatística que você definir
- [`tamanho_amostra`](#tamanho_amostra) — Calcula o n mínimo para estimar média ou proporção com a margem de
- [`teste_ab`](#teste_ab) — Análise completa de um teste A/B de proporções (conversão)
- [`correlacao_com_p`](#correlacao_com_p) — Todos os pares de correlação COM p-valor — o que `df.corr()` não dá
- [`calcular_vif`](#calcular_vif) — VIF (Variance Inflation Factor) — diagnóstico de multicolinearidade
- [`ajustar_distribuicao`](#ajustar_distribuicao) — Ajusta várias distribuições teóricas aos dados e ranqueia pelo
- [`estatisticas_robustas`](#estatisticas_robustas) — Estatísticas resistentes a outliers, lado a lado com as clássicas
- [`derivada_numerica`](#derivada_numerica) — Derivada numérica de 1ª ou 2ª ordem por diferenças centrais
- [`integral_numerica`](#integral_numerica) — Integral definida via quadratura adaptativa (scipy.integrate.quad),
  
**7. Análise de Sobrevivência**
 
- [`preparar_sobrevivencia`](#preparar_sobrevivencia) — Valida e padroniza um DataFrame para análise de sobrevivência
- [`kaplan_meier`](#kaplan_meier) — Curva(s) de Kaplan-Meier com medianas, IC e log-rank automático
- [`risco_acumulado`](#risco_acumulado) — Risco acumulado de Nelson-Aalen H(t) — o complemento do Kaplan-Meier
- [`cox_ph`](#cox_ph) — Regressão de Cox (riscos proporcionais) com relatório interpretado
- [`cox_lasso`](#cox_lasso) — Cox penalizado (LASSO/Elastic-Net) com seleção do penalizador por
- [`modelos_parametricos_sobrevivencia`](#modelos_parametricos_sobrevivencia) — Ajusta modelos paramétricos de sobrevivência e compara por AIC
- [`prever_sobrevivencia`](#prever_sobrevivencia) — Prevê curvas de sobrevivência individuais com um modelo de Cox ajustado
  
---


## 1. Ingestão de Dados

### `carregar_dados`

```python
carregar_dados(
    caminho: str | Path,
    *,
    encodings: Sequence[str] = (utf-8, latin-1, cp1252),
    detectar_separador: bool = True,
    decimal_brasileiro: bool = False,
    aba: str | int | None = 0,
    verbose: bool = True,
    **kwargs: Any
) -> pd.DataFrame
```

Carrega um arquivo de dados detectando o formato pela extensão.

Resolve os problemas mais comuns de ingestão:
- Encoding errado (tenta uma lista de encodings em cascata);
- Separador desconhecido em CSV/TXT (`;`, `,`, `\t`, `|`);
- Números no padrão brasileiro (`1.234,56`) via `decimal_brasileiro=True`.

Formatos suportados: .csv, .txt, .tsv, .xlsx, .xls, .json, .parquet,
.feather, .pkl/.pickle, .html, .xml.

**Parameters**
caminho : str | Path
    Caminho do arquivo.
encodings : sequence of str
    Encodings testados em ordem para arquivos de texto.
detectar_separador : bool
    Se True, usa `sep=None, engine='python'` (sniffer do pandas) em CSV/TXT.
decimal_brasileiro : bool
    Se True, lê CSV/TXT com `decimal=',', thousands='.'`.
aba : str | int | None
    Aba do Excel (nome ou índice). `None` retorna dict com todas as abas.
verbose : bool
    Imprime shape e memória após a leitura.
**kwargs
    Repassados à função `pd.read_*` correspondente
    (ex.: `dtype=`, `usecols=`, `nrows=`, `parse_dates=`).

**Returns**
pd.DataFrame
    (ou dict de DataFrames, se Excel com `aba=None`).

**Examples**
```python
>>> df = carregar_dados("vendas.csv", decimal_brasileiro=True)
>>> df = carregar_dados("relatorio.xlsx", aba="2025")
>>> df = carregar_dados("dados.parquet", columns=["id", "preco"])
```


---

### `carregar_multiplos`

```python
carregar_multiplos(
    pasta: str | Path,
    padrao: str = *.csv,
    *,
    adicionar_origem: bool = True,
    ignorar_erros: bool = False,
    verbose: bool = True,
    **kwargs: Any
) -> pd.DataFrame
```

Lê todos os arquivos que casam com um padrão glob e concatena em um único DataFrame.

Útil para consolidar exportações mensais, partições de scraping, logs etc.

**Parameters**
pasta : str | Path
    Diretório onde procurar.
padrao : str
    Padrão glob (ex.: "*.csv", "vendas_2025_*.xlsx").
adicionar_origem : bool
    Se True, cria a coluna `_arquivo_origem` com o nome do arquivo fonte
    (essencial para depurar de onde veio cada linha).
ignorar_erros : bool
    Se True, arquivos com erro são pulados com aviso, em vez de abortar.
**kwargs
    Repassados para `carregar_dados` (ex.: `decimal_brasileiro=True`).

**Returns**
pd.DataFrame concatenado (index resetado).

**Examples**
```python
>>> df = carregar_multiplos("exports/", "vendas_*.csv", decimal_brasileiro=True)
```


---

### `carregar_sql`

```python
carregar_sql(
    query: str,
    conexao: Any,
    *,
    parse_dates: Sequence[str] | None = None,
    chunksize: int | None = None,
    verbose: bool = True,
    **kwargs: Any
) -> pd.DataFrame
```

Executa uma query SQL e retorna DataFrame.

Aceita qualquer conexão compatível com pandas: engine do SQLAlchemy,
`sqlite3.Connection`, connection string etc.

**Parameters**
query : str
    SQL a executar (SELECT).
conexao : Any
    Engine/conexão/URL. Ex.: `sqlalchemy.create_engine("sqlite:///db.db")`.
parse_dates : sequence of str, optional
    Colunas a converter para datetime na leitura.
chunksize : int, optional
    Se definido, lê em blocos e concatena ao final — evita estourar
    memória em tabelas gigantes.

**Examples**
```python
>>> from sqlalchemy import create_engine
>>> eng = create_engine("sqlite:///imoveis.db")
>>> df = carregar_sql("SELECT * FROM anuncios WHERE preco > 0", eng)
```


---

### `carregar_api`

```python
carregar_api(
    url: str,
    *,
    params: dict | None = None,
    headers: dict | None = None,
    caminho_json: str | None = None,
    tentativas: int = 3,
    timeout: int = 30,
    intervalo_retry: float = 2.0,
    verbose: bool = True
) -> pd.DataFrame
```

Consome uma API REST (GET) com retry automático e devolve DataFrame.

Trata os problemas clássicos: timeout, erro 5xx intermitente, resposta
aninhada (use `caminho_json` para navegar até a lista de registros).

**Parameters**
url : str
    Endpoint da API.
params : dict, optional
    Query params (ex.: {"page": 1, "limit": 100}).
headers : dict, optional
    Headers HTTP (ex.: {"Authorization": "Bearer ..."}).
caminho_json : str, optional
    Caminho separado por pontos até a lista de registros dentro do JSON.
    Ex.: "data.results" para `{"data": {"results": [...]}}`.
tentativas : int
    Número máximo de tentativas em caso de falha de rede/5xx.
timeout : int
    Timeout em segundos por requisição.
intervalo_retry : float
    Espera (com backoff exponencial) entre tentativas.

**Examples**
```python
>>> df = carregar_api(
...     "https://dadosabertos.camara.leg.br/api/v2/proposicoes",
```
...     params={"itens": 100}, caminho_json="dados")

---

### `salvar_dados`

```python
salvar_dados(
    df: pd.DataFrame,
    caminho: str | Path,
    *,
    criar_pastas: bool = True,
    index: bool = False,
    verbose: bool = True,
    **kwargs: Any
) -> Path
```

Salva um DataFrame no formato deduzido pela extensão do caminho.

Suporta: .csv, .xlsx, .json, .parquet, .feather, .pkl/.pickle, .html.

**Parameters**
criar_pastas : bool
    Cria diretórios intermediários automaticamente.
index : bool
    Se True, grava o índice (padrão False — causa nº 1 de coluna
    'Unnamed: 0' fantasma ao reler CSVs).
**kwargs
    Repassados ao `df.to_*` (ex.: `sep=';'`, `sheet_name='dados'`).

**Examples**
```python
>>> salvar_dados(df, "saida/resultado.parquet")
>>> salvar_dados(df, "saida/relatorio.xlsx", sheet_name="Q3")
```


---


## 2. ETL — Limpeza e Transformação

### `relatorio_qualidade`

```python
relatorio_qualidade(
    df: pd.DataFrame,
    *,
    limite_cardinalidade: int = 50,
    verbose: bool = True
) -> pd.DataFrame
```

Gera um diagnóstico completo de qualidade dos dados, coluna a coluna.

Para cada coluna reporta: dtype, nº e % de nulos, nº de valores únicos,
% de valor mais frequente (detecta colunas quase-constantes), exemplos,
e flags de problemas comuns:
    - `constante`        : 1 único valor (inútil para modelagem)
    - `quase_constante`  : valor dominante > 95%
    - `alta_cardinalidade`: nº únicos > limite (cuidado com one-hot!)
    - `possivel_id`      : todos os valores são únicos
    - `muitos_nulos`     : > 50% nulos

Também reporta linhas duplicadas no DataFrame inteiro.

**Returns**
pd.DataFrame com o diagnóstico (uma linha por coluna).

**Examples**
```python
>>> diag = relatorio_qualidade(df)
>>> diag[diag["flags"] != ""]  # só colunas com problemas
```


---

### `limpar_nomes_colunas`

```python
limpar_nomes_colunas(
    df: pd.DataFrame,
    *,
    minusculas: bool = True,
    remover_acentos: bool = True,
    snake_case: bool = True,
    prefixo_numerico: str = col_,
    inplace: bool = False
) -> pd.DataFrame
```

Padroniza nomes de colunas: remove acentos, espaços e caracteres especiais.

'Preço do Imóvel (R$)' -> 'preco_do_imovel_r'

Resolve o clássico problema de colunas com espaço/acento que quebram
`df.query()`, SQL, e acesso por atributo.

**Parameters**
prefixo_numerico : str
    Prefixo adicionado a colunas que começariam com dígito
    ('2024' -> 'col_2024'), pois nomes iniciados por número são inválidos
    em muitos contextos.

**Examples**
```python
>>> df = limpar_nomes_colunas(df)
```


---

### `converter_tipos`

```python
converter_tipos(
    df: pd.DataFrame,
    *,
    colunas_data: Sequence[str] | None = None,
    formato_data: str | None = None,
    dayfirst: bool = True,
    colunas_numericas: Sequence[str] | None = None,
    decimal_brasileiro: bool = True,
    colunas_categoricas: Sequence[str] | None = None,
    auto_detectar: bool = False,
    verbose: bool = True
) -> pd.DataFrame
```

Converte tipos de colunas de forma robusta, com foco em dados brasileiros.

Problemas que resolve:
- Datas em dd/mm/aaaa lidas como string (usa `dayfirst=True` por padrão);
- Números como "R$ 1.234,56" ou "1.234,56" viram float de verdade;
- Colunas de baixa cardinalidade viram `category` (economia de memória).

**Parameters**
colunas_data : sequence of str, optional
    Colunas a converter para datetime. Valores inválidos viram NaT.
formato_data : str, optional
    Formato explícito (ex.: "%d/%m/%Y"). Mais rápido e seguro que inferir.
colunas_numericas : sequence of str, optional
    Colunas string a converter para número. Remove "R$", "%", espaços.
decimal_brasileiro : bool
    Se True, interpreta '.' como milhar e ',' como decimal.
colunas_categoricas : sequence of str, optional
    Colunas a converter para dtype `category`.
auto_detectar : bool
    Se True, tenta converter TODAS as colunas object para número/data
    automaticamente (mantém como estava quando >20% viraria nulo).

**Examples**
```python
>>> df = converter_tipos(df,
...     colunas_data=["data_anuncio"], formato_data="%d/%m/%Y",
```
...     colunas_numericas=["preco", "condominio"])

---

### `tratar_nulos`

```python
tratar_nulos(
    df: pd.DataFrame,
    estrategia: "Literal[media, mediana, moda, constante, ffill, bfill, interpolar, knn, drop_linhas, drop_colunas]" = mediana,
    *,
    colunas: Sequence[str] | None = None,
    valor_constante: Any = 0,
    limite_drop_coluna: float = 0.5,
    knn_vizinhos: int = 5,
    criar_indicador: bool = False,
    verbose: bool = True
) -> pd.DataFrame
```

Trata valores nulos com a estratégia escolhida.

**Estratégias**
- "media"/"mediana" : só em colunas numéricas (mediana é robusta a outliers).
- "moda"            : funciona em numéricas e categóricas.
- "constante"       : preenche com `valor_constante`.
- "ffill"/"bfill"   : propaga valor anterior/posterior (séries temporais).
- "interpolar"      : interpolação linear (séries temporais numéricas).
- "knn"             : KNNImputer do sklearn (usa padrões multivariados).
- "drop_linhas"     : remove linhas com nulo nas colunas alvo.
- "drop_colunas"    : remove colunas com fração de nulos > `limite_drop_coluna`.

**Parameters**
colunas : sequence of str, optional
    Restringe o tratamento a essas colunas (padrão: todas as aplicáveis).
criar_indicador : bool
    Se True, cria coluna booleana `<col>_era_nulo` antes de imputar —
    preserva a informação de "faltava" para o modelo.

**Examples**
```python
>>> df = tratar_nulos(df, "mediana", colunas=["preco", "area"],
...                   criar_indicador=True)
>>> df = tratar_nulos(df, "knn")   # imputação multivariada
```


---

### `tratar_duplicados`

```python
tratar_duplicados(
    df: pd.DataFrame,
    *,
    subset: Sequence[str] | None = None,
    manter: "Literal[first, last, False]" = first,
    verbose: bool = True
) -> pd.DataFrame
```

Remove linhas duplicadas com relatório do que foi removido.

**Parameters**
subset : sequence of str, optional
    Considera duplicata apenas por essas colunas (ex.: chave de negócio
    como ["id_anuncio"]) em vez da linha inteira.
manter : "first" | "last" | False
    Qual ocorrência manter. `False` remove todas as ocorrências duplicadas.

**Examples**
```python
>>> df = tratar_duplicados(df, subset=["url"], manter="last")
```


---

### `remover_outliers`

```python
remover_outliers(
    df: pd.DataFrame,
    colunas: Sequence[str] | None = None,
    metodo: "Literal[iqr, zscore, quantil, isolation_forest]" = iqr,
    *,
    fator_iqr: float = 1.5,
    limite_z: float = 3.0,
    quantis: tuple[float, float] = (0.01, 0.99),
    contaminacao: float = 0.05,
    acao: "Literal[remover, limitar, marcar]" = remover,
    verbose: bool = True
) -> pd.DataFrame
```

Detecta e trata outliers em colunas numéricas.

**Métodos**
- "iqr"              : fora de [Q1 - k*IQR, Q3 + k*IQR] (robusto, padrão).
- "zscore"           : |z| > limite_z (assume ~normalidade).
- "quantil"          : fora dos percentis definidos em `quantis`.
- "isolation_forest" : multivariado (sklearn) — considera combinações
                       de variáveis, não coluna a coluna.

**Ações**
- "remover" : exclui as linhas outlier.
- "limitar" : winsoriza — trunca nos limites (clip). Preserva o nº de linhas.
- "marcar"  : só adiciona coluna booleana `_outlier` (você decide depois).

**Examples**
```python
>>> df = remover_outliers(df, ["preco", "area"], metodo="iqr", acao="limitar")
>>> df = remover_outliers(df, metodo="isolation_forest", acao="marcar")
```


---

### `otimizar_memoria`

```python
otimizar_memoria(
    df: pd.DataFrame,
    *,
    categorizar_objetos: bool = True,
    limite_categoria: float = 0.5,
    verbose: bool = True
) -> pd.DataFrame
```

Reduz o uso de memória do DataFrame com downcast de tipos.

- int64 -> menor int possível (int8/16/32);
- float64 -> float32 (atenção: perde precisão além de ~7 dígitos);
- object -> category quando nº únicos / nº linhas < `limite_categoria`.

Essencial ao trabalhar com datasets grandes no Colab (RAM limitada).

**Examples**
```python
>>> df = otimizar_memoria(df)
```


---

### `codificar_categoricas`

```python
codificar_categoricas(
    df: pd.DataFrame,
    colunas: Sequence[str] | None = None,
    metodo: "Literal[onehot, label, ordinal, frequencia, target]" = onehot,
    *,
    ordem: dict[str, list] | None = None,
    alvo: str | None = None,
    max_categorias_onehot: int = 20,
    drop_first: bool = False,
    verbose: bool = True
) -> pd.DataFrame
```

Codifica variáveis categóricas para uso em modelos.

**Métodos**
- "onehot"     : dummies 0/1. Pula (com aviso) colunas com mais de
                 `max_categorias_onehot` categorias — evita explosão dimensional.
- "label"      : inteiro arbitrário por categoria (ok p/ modelos de árvore).
- "ordinal"    : inteiro respeitando ordem fornecida em `ordem`
                 (ex.: {"tamanho": ["P", "M", "G"]}).
- "frequencia" : substitui pela frequência relativa da categoria.
- "target"     : média do alvo por categoria (target encoding).
                 ATENÇÃO: aplique só no treino ou com validação cruzada
                 para não vazar informação (data leakage).

**Examples**
```python
>>> df = codificar_categoricas(df, ["bairro"], metodo="frequencia")
>>> df = codificar_categoricas(df, ["padrao"], metodo="ordinal",
...                            ordem={"padrao": ["baixo", "medio", "alto"]})
```


---

### `escalar`

```python
escalar(
    df: pd.DataFrame,
    colunas: Sequence[str] | None = None,
    metodo: "Literal[standard, minmax, robust, log, log1p]" = standard,
    *,
    verbose: bool = True
) -> tuple[pd.DataFrame, Any]
```

Escala/transforma colunas numéricas e retorna também o scaler ajustado.

**Métodos**
- "standard" : (x - média) / desvio — padrão p/ modelos lineares, SVM, KNN.
- "minmax"   : [0, 1] — bom p/ redes neurais.
- "robust"   : usa mediana e IQR — resistente a outliers.
- "log"      : log natural (exige valores > 0).
- "log1p"    : log(1 + x) — aceita zeros; ótimo p/ variáveis assimétricas
               como preço e renda.

**Returns**
(DataFrame escalado, scaler ajustado ou None)
    Guarde o scaler para aplicar `.transform()` em dados novos/teste —
    NUNCA ajuste o scaler no conjunto de teste (data leakage).

**Examples**
```python
>>> df_tr, scaler = escalar(df_treino, ["preco", "area"], "robust")
>>> df_te[["preco", "area"]] = scaler.transform(df_teste[["preco", "area"]])
```


---

### `criar_features_data`

```python
criar_features_data(
    df: pd.DataFrame,
    coluna: str,
    *,
    componentes: Sequence[str] = (ano, mes, dia, dia_semana, fim_de_semana, trimestre),
    ciclicas: bool = False,
    prefixo: str | None = None,
    verbose: bool = True
) -> pd.DataFrame
```

Extrai features de uma coluna datetime (feature engineering temporal).

Componentes disponíveis: "ano", "mes", "dia", "dia_semana" (0=segunda),
"dia_ano", "semana", "trimestre", "hora", "minuto", "fim_de_semana",
"inicio_mes", "fim_mes".

**Parameters**
ciclicas : bool
    Se True, adiciona codificação seno/cosseno para mês, dia da semana e
    hora — preserva a circularidade (dezembro é vizinho de janeiro),
    importante para modelos lineares e redes neurais.
prefixo : str, optional
    Prefixo das novas colunas (padrão: nome da coluna original).

**Examples**
```python
>>> df = criar_features_data(df, "data_anuncio", ciclicas=True)
```


---

### `padronizar_texto`

```python
padronizar_texto(
    df: pd.DataFrame,
    colunas: Sequence[str] | None = None,
    *,
    minusculas: bool = True,
    remover_acentos: bool = False,
    remover_espacos_extras: bool = True,
    remover_pontuacao: bool = False,
    modo_titulo: bool = False,
    mapa_substituicoes: dict[str, str] | None = None,
    verbose: bool = True
) -> pd.DataFrame
```

Padroniza VALORES de colunas de texto (não os nomes — p/ isso use
`limpar_nomes_colunas`).

Resolve o clássico "São Paulo" vs "sao paulo " vs "SAO  PAULO" que gera
categorias duplicadas fantasmas em groupby e value_counts.

**Parameters**
colunas : sequence of str, optional
    Padrão: todas as colunas object/category.
remover_acentos : bool
    "São Paulo" -> "Sao Paulo". Cuidado: irreversível.
remover_espacos_extras : bool
    Trim nas pontas + colapsa espaços múltiplos internos.
modo_titulo : bool
    Aplica Title Case ao final ("praia da costa" -> "Praia Da Costa").
mapa_substituicoes : dict, optional
    Substituições exatas aplicadas por último
    (ex.: {"Vv": "Vila Velha", "N/I": ""}).

**Examples**
```python
>>> df = padronizar_texto(df, ["bairro", "cidade"], remover_acentos=True)
```


---

### `criar_faixas`

```python
criar_faixas(
    df: pd.DataFrame,
    coluna: str,
    *,
    metodo: "Literal[quantil, largura, custom]" = quantil,
    n_faixas: int = 4,
    limites: Sequence[float] | None = None,
    rotulos: Sequence[str] | None = None,
    nova_coluna: str | None = None,
    verbose: bool = True
) -> pd.DataFrame
```

Discretiza uma variável numérica em faixas (binning).

**Métodos**
- "quantil" : faixas com o MESMO nº de observações (qcut) — padrão,
              robusto a assimetria;
- "largura" : faixas de mesma amplitude (cut) — interpretável, mas faixas
              podem ficar vazias em dados assimétricos;
- "custom"  : você define os cortes em `limites`
              (ex.: [0, 200_000, 500_000, np.inf]).

Uso típico: transformar preço em "econômico/médio/alto/luxo" para
análise, ou discretizar para testes qui-quadrado.

**Examples**
```python
>>> df = criar_faixas(df, "preco", metodo="custom",
...     limites=[0, 300e3, 600e3, np.inf],
```
...     rotulos=["econômico", "médio", "alto padrão"])

---

### `mesclar_seguro`

```python
mesclar_seguro(
    esquerda: pd.DataFrame,
    direita: pd.DataFrame,
    *,
    on: str | Sequence[str],
    como: "Literal[left, right, inner, outer]" = left,
    sufixos: tuple[str, str] = (, _dir),
    validar: str | None = None,
    verbose: bool = True
) -> pd.DataFrame
```

Merge com diagnóstico completo — o antídoto contra joins silenciosamente errados.

Reporta automaticamente:
- taxa de match (quantas chaves da esquerda encontraram par);
- explosão de linhas (merge N:N duplicando dados sem você perceber);
- chaves duplicadas em cada lado.

**Parameters**
validar : str, optional
    Passa ao pandas p/ ABORTAR se a relação for violada:
    "one_to_one", "one_to_many", "many_to_one". Use sempre que souber a
    cardinalidade esperada.

**Examples**
```python
>>> df = mesclar_seguro(anuncios, bairros, on="bairro",
...                     como="left", validar="many_to_one")
```


---


## 3. EDA — Análise Exploratória

### `resumo_geral`

```python
resumo_geral(
    df: pd.DataFrame,
    *,
    percentis: Sequence[float] = (0.05, 0.25, 0.5, 0.75, 0.95),
    verbose: bool = True
) -> pd.DataFrame
```

Describe turbinado: estatísticas + assimetria, curtose, CV e nulos.

Para colunas numéricas adiciona:
- `assimetria` (skew): |v| > 1 sugere transformação log;
- `curtose`: caudas pesadas (>3 = mais outliers que a normal);
- `cv` (coef. de variação = desvio/média): compara dispersão entre
  variáveis de escalas diferentes.

**Examples**
```python
>>> resumo_geral(df)
```


---

### `analise_correlacao`

```python
analise_correlacao(
    df: pd.DataFrame,
    metodo: "Literal[pearson, spearman, kendall]" = pearson,
    *,
    limite_forte: float = 0.7,
    alvo: str | None = None,
    verbose: bool = True
) -> pd.DataFrame
```

Matriz de correlação + relatório dos pares mais correlacionados.

Dicas de escolha do método:
- "pearson"  : relação LINEAR, sensível a outliers;
- "spearman" : relação MONOTÔNICA (por postos), robusta a outliers —
               preferível quando há assimetria forte;
- "kendall"  : similar ao spearman, melhor p/ amostras pequenas/empates.

**Parameters**
limite_forte : float
    |corr| acima disso entra no relatório de pares fortes — atenção a
    multicolinearidade em modelos lineares.
alvo : str, optional
    Se fornecido, imprime ranking de correlação de todas as variáveis
    com o alvo.

**Returns**
Matriz de correlação (DataFrame).

**Examples**
```python
>>> corr = analise_correlacao(df, "spearman", alvo="preco")
```


---

### `testar_normalidade`

```python
testar_normalidade(
    serie: pd.Series | np.ndarray,
    *,
    alpha: float = 0.05,
    verbose: bool = True
) -> dict[str, Any]
```

Testa normalidade com Shapiro-Wilk, D'Agostino e Kolmogorov-Smirnov.

Regras práticas aplicadas automaticamente:
- Shapiro-Wilk é o mais poderoso, mas limitado a n <= 5000 (acima disso
  usa-se amostra aleatória de 5000);
- Com n muito grande, QUALQUER desvio mínimo rejeita H0 — considere
  também skew/curtose e o histograma antes de decidir.

**Returns**
dict com estatísticas, p-valores e o veredito `normal` (bool) por maioria.

**Examples**
```python
>>> res = testar_normalidade(df["preco"])
>>> if not res["normal"]: ...  # use testes não-paramétricos
```


---

### `comparar_grupos`

```python
comparar_grupos(
    df: pd.DataFrame,
    coluna_numerica: str,
    coluna_grupo: str,
    *,
    alpha: float = 0.05,
    forcar_teste: str | None = None,
    verbose: bool = True
) -> dict[str, Any]
```

Compara uma variável numérica entre grupos escolhendo o teste correto.

Fluxo de decisão automático:
- 2 grupos, ambos ~normais  -> t de Welch (não assume variâncias iguais);
- 2 grupos, não normais     -> Mann-Whitney U;
- 3+ grupos, todos ~normais -> ANOVA one-way;
- 3+ grupos, não normais    -> Kruskal-Wallis.

Também reporta tamanho de efeito (Cohen's d para 2 grupos; eta² p/ 3+),
porque p-valor pequeno não significa efeito relevante.

**Parameters**
forcar_teste : {"t", "mannwhitney", "anova", "kruskal"}, optional
    Ignora a decisão automática e usa o teste indicado.

**Returns**
dict: teste usado, estatística, p-valor, tamanho de efeito, medianas/médias.

**Examples**
```python
>>> comparar_grupos(df, "preco_m2", "cluster")
```


---

### `analise_categorica`

```python
analise_categorica(
    df: pd.DataFrame,
    coluna_a: str,
    coluna_b: str,
    *,
    alpha: float = 0.05,
    verbose: bool = True
) -> dict[str, Any]
```

Associação entre duas variáveis categóricas: qui-quadrado + Cramér's V.

Valida a suposição do qui-quadrado (frequências esperadas >= 5 em pelo
menos 80% das células); se violada em tabela 2x2, usa Fisher exato.

Cramér's V interpreta a força: ~0.1 fraca, ~0.3 média, ~0.5 forte.

**Returns**
dict com tabela de contingência, teste, p-valor e Cramér's V.

**Examples**
```python
>>> analise_categorica(df, "bairro_cluster", "tipo_imovel")
```


---

### `analise_univariada`

```python
analise_univariada(
    df: pd.DataFrame,
    coluna: str,
    *,
    top_n: int = 10,
    verbose: bool = True
) -> dict[str, Any]
```

Perfil completo de uma única coluna, numérica ou categórica.

Numérica: estatísticas, quartis, outliers via IQR, teste de normalidade.
Categórica: contagens, proporções, cardinalidade, categorias raras (<1%).

**Examples**
```python
>>> analise_univariada(df, "preco")
>>> analise_univariada(df, "bairro")
```


---

### `analise_alvo`

```python
analise_alvo(
    df: pd.DataFrame,
    alvo: str,
    *,
    max_categorias: int = 30,
    verbose: bool = True
) -> pd.DataFrame
```

Ranqueia TODAS as features pela força de associação com o alvo,
escolhendo a medida certa para cada par de tipos:

- num x num : |correlação de Spearman|;
- cat x num : eta² da ANOVA (variância do alvo explicada pelos grupos);
- num x cat (alvo categórico) : eta² invertido (feature por grupo do alvo);
- cat x cat : Cramér's V.

Todas as medidas ficam em [0, 1], então o ranking é comparável entre
tipos — ótimo primeiro passo de seleção de features.

**Returns**
DataFrame: feature, tipo_relacao, medida, forca (ordenado desc).

**Examples**
```python
>>> analise_alvo(df, "preco").head(10)
```


---


## 4. Gráficos

### `configurar_estilo`

```python
configurar_estilo(
    tema: "Literal[claro, escuro]" = escuro,
    *,
    paleta: str = viridis,
    tamanho_figura: tuple[float, float] = (10, 6),
    tamanho_fonte: int = 11,
    dpi: int = 100
) -> None
```

Configura o estilo global de todos os plots do módulo (e do notebook).

Chame uma vez no início do notebook. O tema "escuro" combina com o
dark mode do VSCode/Colab e usa fundo transparente-friendly.

**Examples**
```python
>>> configurar_estilo("escuro")
>>> configurar_estilo("claro", paleta="magma", dpi=150)  # p/ relatórios
```


---

### `plot_distribuicao`

```python
plot_distribuicao(
    df: pd.DataFrame,
    coluna: str,
    *,
    bins: int | str = auto,
    kde: bool = True,
    com_boxplot: bool = True,
    log_x: bool = False,
    titulo: str | None = None,
    salvar_em: str | None = None
)
```

Histograma + KDE + boxplot alinhados — visão completa de uma numérica.

Anota média (linha tracejada) e mediana (linha cheia); a distância entre
elas denuncia assimetria.

**Parameters**
log_x : bool
    Escala log no eixo x — indispensável p/ preço, renda, população.
salvar_em : str, optional
    Caminho para salvar a figura (ex.: "figs/dist_preco.png").

**Returns**
(fig, axes)

**Examples**
```python
>>> fig, ax = plot_distribuicao(df, "preco", log_x=True)
```


---

### `plot_correlacao`

```python
plot_correlacao(
    df: pd.DataFrame,
    *,
    metodo: "Literal[pearson, spearman, kendall]" = pearson,
    anotar: bool = True,
    mascara_superior: bool = True,
    cmap: str = coolwarm,
    titulo: str | None = None,
    salvar_em: str | None = None
)
```

Heatmap de correlação com máscara triangular (sem redundância visual).

**Examples**
```python
>>> plot_correlacao(df, metodo="spearman")
```


---

### `plot_categorico`

```python
plot_categorico(
    df: pd.DataFrame,
    coluna: str,
    *,
    top_n: int = 15,
    horizontal: bool = True,
    mostrar_pct: bool = True,
    ordenar: bool = True,
    titulo: str | None = None,
    salvar_em: str | None = None
)
```

Gráfico de barras de contagem com % anotada e limite de categorias.

Categorias além de `top_n` são agrupadas em "(outras)" — evita o gráfico
ilegível de 80 bairros.

**Examples**
```python
>>> plot_categorico(df, "bairro", top_n=12)
```


---

### `plot_dispersao`

```python
plot_dispersao(
    df: pd.DataFrame,
    x: str,
    y: str,
    *,
    hue: str | None = None,
    tamanho: str | None = None,
    linha_tendencia: bool = True,
    log_x: bool = False,
    log_y: bool = False,
    alpha: float = 0.6,
    amostra_max: int = 10000,
    titulo: str | None = None,
    salvar_em: str | None = None
)
```

Dispersão com linha de tendência e correlação anotada no título.

Amostra automaticamente quando há mais de `amostra_max` pontos —
scatter com 500k pontos trava o notebook e vira uma mancha.

**Examples**
```python
>>> plot_dispersao(df, "area", "preco", hue="cluster", log_y=True)
```


---

### `plot_boxplots_grupo`

```python
plot_boxplots_grupo(
    df: pd.DataFrame,
    coluna_numerica: str,
    coluna_grupo: str,
    *,
    top_n: int = 12,
    ordenar_por_mediana: bool = True,
    mostrar_pontos: bool = False,
    titulo: str | None = None,
    salvar_em: str | None = None
)
```

Boxplots de uma numérica por grupo, ordenados pela mediana.

Limita aos `top_n` grupos mais frequentes (os demais poluem o gráfico).
`mostrar_pontos=True` sobrepõe stripplot — bom p/ grupos pequenos.

**Examples**
```python
>>> plot_boxplots_grupo(df, "preco_m2", "bairro", top_n=10)
```


---

### `plot_serie_temporal`

```python
plot_serie_temporal(
    df: pd.DataFrame,
    coluna_data: str,
    coluna_valor: str,
    *,
    frequencia: str | None = None,
    agregacao: str = mean,
    media_movel: int | None = None,
    titulo: str | None = None,
    salvar_em: str | None = None
)
```

Linha temporal com reamostragem e média móvel opcionais.

**Parameters**
frequencia : str, optional
    Regra de resample do pandas: "D", "W", "ME" (mês), "QE", "YE".
agregacao : str
    Função de agregação no resample: "mean", "sum", "median", "count"...
media_movel : int, optional
    Janela da média móvel sobreposta (suaviza ruído).

**Examples**
```python
>>> plot_serie_temporal(df, "data", "preco", frequencia="ME",
...                     agregacao="median", media_movel=3)
```


---

### `plot_nulos`

```python
plot_nulos(
    df: pd.DataFrame,
    *,
    apenas_com_nulos: bool = True,
    titulo: str | None = None,
    salvar_em: str | None = None
)
```

Barra horizontal com % de nulos por coluna (visão rápida de faltantes).

**Examples**
```python
>>> plot_nulos(df)
```


---

### `plot_qq`

```python
plot_qq(
    serie: pd.Series | np.ndarray,
    *,
    titulo: str | None = None,
    salvar_em: str | None = None
)
```

QQ-plot contra a normal — o complemento visual de `testar_normalidade`.

Como ler: pontos na reta = compatível com normal; cauda direita acima
da reta = assimetria positiva (típico de preço — tente log).

**Examples**
```python
>>> plot_qq(df["preco"])
>>> plot_qq(np.log(df["preco"]))  # comparar antes/depois do log
```


---

### `plot_pares`

```python
plot_pares(
    df: pd.DataFrame,
    colunas: Sequence[str] | None = None,
    *,
    hue: str | None = None,
    amostra_max: int = 2000,
    diagonal: "Literal[kde, hist]" = kde,
    salvar_em: str | None = None
)
```

Pairplot (matriz de dispersões) com amostragem automática.

Limita a 8 colunas e `amostra_max` linhas — pairplot completo de um
DataFrame grande demora minutos e sai ilegível.

**Examples**
```python
>>> plot_pares(df, ["preco", "area", "quartos"], hue="cluster")
```


---


## 5. Machine Learning

### `preparar_dados`

```python
preparar_dados(
    df: pd.DataFrame,
    alvo: str,
    *,
    colunas_excluir: Sequence[str] = (),
    test_size: float = 0.2,
    estratificar: bool | None = None,
    random_state: int = 42,
    verbose: bool = True
) -> tuple[pd.DataFrame, pd.DataFrame, pd.Series, pd.Series]
```

Separa X/y e faz train/test split com estratificação inteligente.

- Remove automaticamente do X colunas de ID óbvias e as em `colunas_excluir`;
- `estratificar=None` decide sozinho: estratifica se o alvo for
  categórico/discreto com poucas classes (mantém proporção das classes —
  crucial em bases desbalanceadas).

**Returns**
(X_train, X_test, y_train, y_test)

**Examples**
```python
>>> X_tr, X_te, y_tr, y_te = preparar_dados(df, "preco",
...     colunas_excluir=["url", "id_anuncio"])
```


---

### `pipeline_preprocessamento`

```python
pipeline_preprocessamento(
    X: pd.DataFrame,
    *,
    imputacao_numerica: str = median,
    imputacao_categorica: str = most_frequent,
    escalar_numericas: bool = True,
    metodo_escala: "Literal[standard, minmax, robust]" = standard,
    max_categorias_onehot: int = 20
) -> Any
```

Monta um ColumnTransformer sklearn com imputação + escala + one-hot.

Vantagem sobre pré-processar "na mão": encapsulado num Pipeline, o
fit acontece SÓ no treino — elimina data leakage por construção, e o
mesmo objeto serve para produção (FastAPI, por exemplo).

Colunas categóricas com mais de `max_categorias_onehot` categorias são
descartadas com aviso (trate-as antes com frequency/target encoding).

**Returns**
sklearn.compose.ColumnTransformer (não ajustado).

**Examples**
```python
>>> from sklearn.pipeline import Pipeline
>>> from sklearn.ensemble import RandomForestRegressor
>>> pre = pipeline_preprocessamento(X_train)
>>> modelo = Pipeline([("pre", pre), ("rf", RandomForestRegressor())])
>>> modelo.fit(X_train, y_train)
```


---

### `avaliar_classificacao`

```python
avaliar_classificacao(
    modelo: Any,
    X_test: pd.DataFrame | np.ndarray,
    y_test: pd.Series | np.ndarray,
    *,
    plotar: bool = True,
    salvar_em: str | None = None,
    verbose: bool = True
) -> dict[str, Any]
```

Avaliação completa de um classificador já treinado.

Reporta accuracy, precision, recall, F1 (weighted), classification_report
e, se o modelo tiver `predict_proba` e o problema for binário, ROC-AUC.
Plota matriz de confusão (e curva ROC no caso binário).

Lembrete: em bases desbalanceadas, olhe F1/recall por classe — accuracy
engana (o "accuracy paradox").

**Examples**
```python
>>> metricas = avaliar_classificacao(modelo, X_te, y_te)
```


---

### `avaliar_regressao`

```python
avaliar_regressao(
    modelo: Any,
    X_test: pd.DataFrame | np.ndarray,
    y_test: pd.Series | np.ndarray,
    *,
    plotar: bool = True,
    salvar_em: str | None = None,
    verbose: bool = True
) -> dict[str, Any]
```

Avaliação completa de um regressor já treinado.

Métricas: RMSE, MAE, R², MAPE (ignora divisões por ~zero).
Plots de diagnóstico: previsto vs real e resíduos vs previsto —
padrão nos resíduos (funil, curva) indica heterocedasticidade ou
relação não capturada.

**Examples**
```python
>>> metricas = avaliar_regressao(modelo, X_te, y_te)
```


---

### `comparar_modelos`

```python
comparar_modelos(
    X: pd.DataFrame | np.ndarray,
    y: pd.Series | np.ndarray,
    tipo: "Literal[classificacao, regressao]",
    *,
    modelos: dict[str, Any] | None = None,
    cv: int = 5,
    scoring: str | None = None,
    preprocessador: Any = None,
    random_state: int = 42,
    verbose: bool = True
) -> pd.DataFrame
```

Compara vários modelos baseline via validação cruzada.

Sempre comece por aqui antes de otimizar hiperparâmetros: descobrir QUAL
família de modelo funciona vale mais que ajustar fino o modelo errado.

**Parameters**
modelos : dict, optional
    {"nome": estimador}. Se None, usa um conjunto padrão sensato
    (Dummy como piso, linear, árvore, floresta, gradient boosting, KNN).
scoring : str, optional
    Métrica sklearn (padrão: "f1_weighted" ou "neg_root_mean_squared_error").
preprocessador : ColumnTransformer, optional
    Se fornecido, cada modelo roda dentro de um Pipeline com ele —
    garante pré-processamento sem leakage dentro de cada fold.

**Returns**
DataFrame ordenado por desempenho com média, desvio e tempo de fit.

**Examples**
```python
>>> pre = pipeline_preprocessamento(X)
>>> ranking = comparar_modelos(X, y, "regressao", preprocessador=pre)
```


---

### `otimizar_hiperparametros`

```python
otimizar_hiperparametros(
    modelo: Any,
    grade: dict[str, list],
    X: pd.DataFrame | np.ndarray,
    y: pd.Series | np.ndarray,
    *,
    metodo: "Literal[grid, random]" = random,
    n_iter: int = 50,
    cv: int = 5,
    scoring: str | None = None,
    random_state: int = 42,
    verbose: bool = True
) -> Any
```

Busca de hiperparâmetros com GridSearchCV ou RandomizedSearchCV.

Regra prática: "random" com 50 iterações costuma achar resultado quase
tão bom quanto grid completo em fração do tempo — use "grid" só quando
a grade é pequena (< ~100 combinações).

**Parameters**
grade : dict
    {"param": [valores]}. Se `modelo` for um Pipeline, prefixe com o nome
    do passo: {"modelo__n_estimators": [100, 300]}.

**Returns**
Objeto de busca ajustado. Use `.best_estimator_`, `.best_params_`,
`.best_score_`.

**Examples**
```python
>>> busca = otimizar_hiperparametros(
...     RandomForestRegressor(),
```
...     {"n_estimators": [100, 300, 500], "max_depth": [None, 10, 20]},
...     X_tr, y_tr, scoring="neg_root_mean_squared_error")
```python
>>> melhor = busca.best_estimator_
```


---

### `importancia_features`

```python
importancia_features(
    modelo: Any,
    nomes_features: Sequence[str] | None = None,
    *,
    X_test: pd.DataFrame | np.ndarray | None = None,
    y_test: pd.Series | np.ndarray | None = None,
    metodo: "Literal[auto, permutacao]" = auto,
    top_n: int = 20,
    plotar: bool = True,
    random_state: int = 42,
    verbose: bool = True
) -> pd.DataFrame
```

Ranking de importância de features de um modelo treinado.

- "auto": usa `feature_importances_` (árvores) ou |coef_| (lineares —
  só comparável se as features estiverem escaladas!);
- "permutacao": permutation importance no conjunto de TESTE (mais
  confiável e agnóstico ao modelo; exige X_test e y_test).

**Examples**
```python
>>> imp = importancia_features(modelo, X_tr.columns)
>>> imp = importancia_features(modelo, X_te.columns, X_test=X_te,
...                            y_test=y_te, metodo="permutacao")
```


---

### `avaliar_clustering`

```python
avaliar_clustering(
    X: pd.DataFrame | np.ndarray,
    *,
    k_min: int = 2,
    k_max: int = 10,
    escalar_antes: bool = True,
    random_state: int = 42,
    plotar: bool = True,
    verbose: bool = True
) -> pd.DataFrame
```

Ajuda a escolher o k do KMeans: inércia (elbow) + silhouette por k.

Interpretação do silhouette médio: > 0.5 estrutura razoável; 0.25–0.5
estrutura fraca (comum em dados reais); < 0.25 provavelmente não há
clusters bem separados — reporte isso com honestidade.

**Parameters**
escalar_antes : bool
    Padroniza X antes (StandardScaler) — KMeans usa distância euclidiana
    e é dominado pela variável de maior escala se não escalar.

**Returns**
DataFrame com k, inércia e silhouette; melhor k por silhouette impresso.

**Examples**
```python
>>> resultados = avaliar_clustering(df[["preco_m2", "dist_praia"]], k_max=8)
```


---

### `salvar_modelo`

```python
salvar_modelo(
    modelo: Any,
    caminho: str | Path,
    *,
    metadados: dict | None = None,
    verbose: bool = True
) -> Path
```

Persiste um modelo (ou pipeline) com joblib + metadados opcionais.

Boas práticas embutidas:
- salva junto um dict de metadados (data, métricas, versão de libs) em
  `<caminho>.meta.json` — daqui a 6 meses você vai agradecer;
- cria pastas automaticamente.

**Examples**
```python
>>> salvar_modelo(pipeline, "modelos/rf_precos.joblib",
...               metadados={"rmse": 45000, "features": list(X.columns)})
```


---

### `carregar_modelo`

```python
carregar_modelo(caminho: str | Path, *, verbose: bool = True) -> Any
```

Carrega modelo salvo com `salvar_modelo` e avisa se a versão do
sklearn mudou (causa comum de warnings/incompatibilidades silenciosas).

**Examples**
```python
>>> modelo = carregar_modelo("modelos/rf_precos.joblib")
```


---

### `curva_aprendizado`

```python
curva_aprendizado(
    modelo: Any,
    X: pd.DataFrame | np.ndarray,
    y: pd.Series | np.ndarray,
    *,
    cv: int = 5,
    scoring: str | None = None,
    n_pontos: int = 8,
    plotar: bool = True,
    verbose: bool = True
) -> pd.DataFrame
```

Curva de aprendizado — diagnostica overfitting vs underfitting.

Como interpretar:
- Curvas distantes (treino alto, validação baixa) -> OVERFITTING:
  regularize, simplifique o modelo ou consiga mais dados;
- Curvas juntas e baixas -> UNDERFITTING: modelo mais complexo ou
  melhores features;
- Validação ainda subindo no fim -> mais dados devem ajudar.

**Examples**
```python
>>> curva_aprendizado(pipeline, X, y, scoring="r2")
```


---

### `selecionar_features`

```python
selecionar_features(
    X: pd.DataFrame,
    y: pd.Series | np.ndarray,
    *,
    metodo: "Literal[kbest, rfe, modelo]" = kbest,
    k: int = 10,
    tipo: "Literal[classificacao, regressao]" = regressao,
    estimador: Any = None,
    verbose: bool = True
) -> list[str]
```

Seleciona as k melhores features (exige X totalmente numérico e sem nulos).

**Métodos**
- "kbest"  : testes univariados (f_regression/f_classif) — rápido, mas
             ignora interações;
- "rfe"    : eliminação recursiva com o estimador — mais caro e mais fiel;
- "modelo" : importâncias de um RandomForest (capta não-linearidades).

**Returns**
Lista com os nomes das features selecionadas.

**Examples**
```python
>>> melhores = selecionar_features(X_num, y, metodo="rfe", k=8)
>>> X_reduzido = X[melhores]
```


---

### `reduzir_dimensionalidade`

```python
reduzir_dimensionalidade(
    X: pd.DataFrame | np.ndarray,
    *,
    n_componentes: int | float = 0.95,
    escalar_antes: bool = True,
    plotar: bool = True,
    verbose: bool = True
) -> tuple[pd.DataFrame, Any]
```

PCA com relatório de variância explicada.

**Parameters**
n_componentes : int | float
    Inteiro = nº fixo de componentes; float em (0,1) = mínimo de
    variância acumulada desejada (ex.: 0.95 escolhe o nº automaticamente).
escalar_antes : bool
    Padroniza X primeiro — PCA sem escala é dominado pela variável de
    maior variância. Desligue só se X já estiver escalado.

**Returns**
(DataFrame com colunas PC1..PCn, objeto PCA ajustado)
    Use `pca.transform(novos_dados_escalados)` em produção.

**Examples**
```python
>>> X_pca, pca = reduzir_dimensionalidade(X_num, n_componentes=0.9)
```


---

### `balancear_classes`

```python
balancear_classes(
    X: pd.DataFrame,
    y: pd.Series,
    *,
    metodo: "Literal[undersample, oversample]" = oversample,
    random_state: int = 42,
    verbose: bool = True
) -> tuple[pd.DataFrame, pd.Series]
```

Balanceia classes por reamostragem simples (sem dependência do imblearn).

- "oversample"  : duplica aleatoriamente a(s) classe(s) minoritária(s)
                  até igualar a majoritária (não perde dados; risco de
                  overfitting em modelos que decoram);
- "undersample" : descarta aleatoriamente da majoritária (perde dados;
                  seguro quando a base é grande).

IMPORTANTE: aplique SOMENTE no conjunto de TREINO, nunca no teste —
balancear antes do split infla artificialmente as métricas.

**Examples**
```python
>>> X_tr_bal, y_tr_bal = balancear_classes(X_tr, y_tr, metodo="oversample")
```


---


## 6. Matemática / Estatística

### `intervalo_confianca`

```python
intervalo_confianca(
    serie: pd.Series | np.ndarray,
    *,
    nivel: float = 0.95,
    tipo: "Literal[media, proporcao, mediana]" = media,
    verbose: bool = True
) -> dict[str, float]
```

Intervalo de confiança para média (t de Student), proporção (Wilson)
ou mediana (bootstrap).

- "media"     : IC t — adequado mesmo sem normalidade se n for razoável
                (TCL), mas confira outliers;
- "proporcao" : Wilson score — muito melhor que o IC "normal" clássico
                perto de 0/1 e em amostras pequenas. A série deve conter
                apenas 0/1 (ou bool);
- "mediana"   : bootstrap percentil (2000 reamostras).

**Examples**
```python
>>> intervalo_confianca(df["preco"], tipo="mediana")
>>> intervalo_confianca(df["convertido"], tipo="proporcao")
```


---

### `bootstrap_estatistica`

```python
bootstrap_estatistica(
    serie: pd.Series | np.ndarray,
    estatistica: Callable[[np.ndarray], float] = <function mean at 0x7efd713fa530>,
    *,
    n_reamostras: int = 5000,
    nivel: float = 0.95,
    random_state: int = 42,
    verbose: bool = True
) -> dict[str, Any]
```

IC bootstrap (percentil) para QUALQUER estatística que você definir.

Serve quando não há fórmula fechada: desvio-padrão, quantis, razão de
médias, coeficiente de variação, trimmed mean...

**Parameters**
estatistica : callable
    Função array -> escalar. Ex.: `np.median`,
    `lambda a: np.percentile(a, 90)`, `lambda a: a.std()/a.mean()`.

**Examples**
```python
>>> bootstrap_estatistica(df["preco"], lambda a: np.percentile(a, 90))
```


---

### `tamanho_amostra`

```python
tamanho_amostra(
    *,
    tipo: "Literal[media, proporcao]" = proporcao,
    margem_erro: float = 0.05,
    nivel: float = 0.95,
    desvio_estimado: float | None = None,
    proporcao_estimada: float = 0.5,
    populacao: int | None = None,
    verbose: bool = True
) -> int
```

Calcula o n mínimo para estimar média ou proporção com a margem de
erro desejada.

- "proporcao" : usa p=0.5 por padrão (cenário mais conservador);
- "media"     : exige `desvio_estimado` (de estudo piloto ou literatura);
- `populacao` : se fornecida, aplica correção para população finita
  (faz diferença quando n passa de ~5% da população).

**Examples**
```python
>>> tamanho_amostra(tipo="proporcao", margem_erro=0.03)          # pesquisa
>>> tamanho_amostra(tipo="media", margem_erro=50, desvio_estimado=400)
```


---

### `teste_ab`

```python
teste_ab(
    conversoes_a: int,
    total_a: int,
    conversoes_b: int,
    total_b: int,
    *,
    alpha: float = 0.05,
    verbose: bool = True
) -> dict[str, Any]
```

Análise completa de um teste A/B de proporções (conversão).

Reporta: taxas, lift relativo, teste z bilateral de duas proporções,
IC da diferença e poder estatístico observado — se o poder for baixo
(<80%), um resultado "não significativo" é inconclusivo, não negativo.

**Examples**
```python
>>> teste_ab(conversoes_a=120, total_a=2400, conversoes_b=156, total_b=2350)
```


---

### `correlacao_com_p`

```python
correlacao_com_p(
    df: pd.DataFrame,
    *,
    metodo: "Literal[pearson, spearman]" = pearson,
    corrigir_multiplas: bool = True,
    alpha: float = 0.05,
    verbose: bool = True
) -> pd.DataFrame
```

Todos os pares de correlação COM p-valor — o que `df.corr()` não dá.

Aplica correção de Bonferroni por padrão: testando dezenas de pares,
~5% seriam "significativos" por puro acaso sem correção.

**Returns**
DataFrame longo: var_1, var_2, r, p_valor, p_ajustado, significativo.

**Examples**
```python
>>> pares = correlacao_com_p(df, metodo="spearman")
>>> pares[pares["significativo"]]
```


---

### `calcular_vif`

```python
calcular_vif(
    X: pd.DataFrame,
    *,
    limite_alerta: float = 10.0,
    verbose: bool = True
) -> pd.DataFrame
```

VIF (Variance Inflation Factor) — diagnóstico de multicolinearidade
para modelos lineares, sem depender do statsmodels.

Interpretação usual: VIF < 5 ok; 5–10 atenção; > 10 multicolinearidade
séria (coeficientes instáveis — remova ou combine variáveis).

Exige X numérico e sem nulos.

**Examples**
```python
>>> calcular_vif(df[["area", "quartos", "banheiros", "vagas"]])
```


---

### `ajustar_distribuicao`

```python
ajustar_distribuicao(
    serie: pd.Series | np.ndarray,
    *,
    candidatas: Sequence[str] = (norm, lognorm, expon, gamma, weibull_min, uniform),
    plotar: bool = True,
    verbose: bool = True
) -> pd.DataFrame
```

Ajusta várias distribuições teóricas aos dados e ranqueia pelo
teste de Kolmogorov-Smirnov (maior p-valor = melhor ajuste).

Útil para: escolher a distribuição em simulações Monte Carlo, validar
suposições de modelos, e — no seu caso — famílias paramétricas de
análise de sobrevivência (Weibull, Log-Normal, Exponencial).

**Parameters**
candidatas : sequence of str
    Nomes de distribuições do `scipy.stats`.

**Returns**
DataFrame: distribuicao, parametros, ks_stat, p_valor (ordenado).

**Examples**
```python
>>> ajustar_distribuicao(df["tempo_ate_venda"],
...                      candidatas=["expon", "weibull_min", "lognorm"])
```


---

### `estatisticas_robustas`

```python
estatisticas_robustas(
    serie: pd.Series | np.ndarray,
    *,
    proporcao_apara: float = 0.1,
    verbose: bool = True
) -> dict[str, float]
```

Estatísticas resistentes a outliers, lado a lado com as clássicas.

- MAD (desvio absoluto mediano, escalado x1.4826 p/ comparar com o desvio
  padrão sob normalidade);
- média aparada (descarta `proporcao_apara` de cada cauda);
- média winsorizada (trunca as caudas em vez de descartar).

Se média ≈ média aparada, os outliers não estão distorcendo; se diferem
muito, prefira estatísticas robustas nos seus relatórios.

**Examples**
```python
>>> estatisticas_robustas(df["preco"])
```


---

### `derivada_numerica`

```python
derivada_numerica(
    funcao: Callable[[float], float],
    x0: float,
    *,
    ordem: int = 1,
    h: float = 1e-05
) -> float
```

Derivada numérica de 1ª ou 2ª ordem por diferenças centrais.

Útil para conferir derivadas calculadas à mão (seus estudos de cálculo)
e para analisar sensibilidade de funções de custo.

**Examples**
```python
>>> derivada_numerica(lambda x: x**3, 2.0)        # ~12 (3x²)
>>> derivada_numerica(np.sin, 0.0, ordem=2)       # ~0  (-sin(0))
```


---

### `integral_numerica`

```python
integral_numerica(
    funcao: Callable[[float], float],
    a: float,
    b: float,
    *,
    verbose: bool = True
) -> dict[str, float]
```

Integral definida via quadratura adaptativa (scipy.integrate.quad),
com estimativa de erro.

Aceita limites infinitos (`np.inf`) — ex.: verificar que uma densidade
de probabilidade integra 1.

**Examples**
```python
>>> integral_numerica(lambda x: np.exp(-x**2/2)/np.sqrt(2*np.pi),
...                   -np.inf, np.inf)   # ~1.0
```


---


## 7. Análise de Sobrevivência

### `preparar_sobrevivencia`

```python
preparar_sobrevivencia(
    df: pd.DataFrame,
    coluna_tempo: str,
    coluna_evento: str,
    *,
    mapa_evento: dict | None = None,
    coluna_inicio: str | None = None,
    coluna_fim: str | None = None,
    unidade: "Literal[dias, semanas, meses, anos]" = dias,
    verbose: bool = True
) -> pd.DataFrame
```

Valida e padroniza um DataFrame para análise de sobrevivência.

Checagens e correções aplicadas:
- evento convertido para 0/1 (aceita bool, "sim"/"nao" via `mapa_evento`,
  True/False...);
- tempos <= 0 ou nulos são removidos com aviso (inválidos p/ os modelos);
- se você só tem DATAS (`coluna_inicio`/`coluna_fim`), calcula a duração
  automaticamente na `unidade` pedida;
- reporta a taxa de censura — se for altíssima (>90%), estimativas de
  mediana podem nem existir.

**Parameters**
mapa_evento : dict, optional
    Mapeamento para 0/1. Ex.: {"obito": 1, "vivo": 0} ou {"Dead": 1,
    "Alive": 0} (padrão TCGA).
coluna_inicio, coluna_fim : str, optional
    Se fornecidas, `coluna_tempo` é CRIADA a partir da diferença.

**Returns**
DataFrame limpo com `coluna_tempo` (float > 0) e `coluna_evento` (int 0/1).

**Examples**
```python
>>> df = preparar_sobrevivencia(df, "tempo_meses", "status",
...                             mapa_evento={"Dead": 1, "Alive": 0})
>>> df = preparar_sobrevivencia(df, "duracao", "vendido",
...     coluna_inicio="data_anuncio", coluna_fim="data_venda",
```
...     unidade="semanas")

---

### `kaplan_meier`

```python
kaplan_meier(
    df: pd.DataFrame,
    coluna_tempo: str,
    coluna_evento: str,
    *,
    coluna_grupo: str | None = None,
    intervalo_confianca: bool = True,
    marcar_censuras: bool = True,
    tabela_risco: bool = False,
    plotar: bool = True,
    salvar_em: str | None = None,
    verbose: bool = True
) -> dict[str, Any]
```

Curva(s) de Kaplan-Meier com medianas, IC e log-rank automático.

- Sem `coluna_grupo`: uma curva geral;
- Com `coluna_grupo`: uma curva por grupo + teste de log-rank
  (multivariado se 3+ grupos) comparando as curvas.

**Parameters**
marcar_censuras : bool
    Desenha ticks nos tempos de censura (boa prática de publicação).
tabela_risco : bool
    Adiciona a tabela "at risk" sob o gráfico (padrão em papers médicos).

**Returns**
dict com: `ajustes` ({grupo: KaplanMeierFitter}), `medianas`
({grupo: mediana com IC}), e `logrank` (estatística e p-valor, se grupos).

**Examples**
```python
>>> res = kaplan_meier(df, "tempo", "evento", coluna_grupo="cluster")
>>> res["medianas"]
```


---

### `risco_acumulado`

```python
risco_acumulado(
    df: pd.DataFrame,
    coluna_tempo: str,
    coluna_evento: str,
    *,
    coluna_grupo: str | None = None,
    plotar: bool = True,
    salvar_em: str | None = None,
    verbose: bool = True
) -> dict[str, Any]
```

Risco acumulado de Nelson-Aalen H(t) — o complemento do Kaplan-Meier.

Como ler a curva: a INCLINAÇÃO é a taxa de risco instantânea.
- Reta -> risco constante (compatível com Exponencial);
- Côncava p/ cima -> risco crescente (Weibull com k > 1, envelhecimento);
- Côncava p/ baixo -> risco decrescente (Weibull com k < 1).

Ótimo diagnóstico visual ANTES de escolher o modelo paramétrico.

**Examples**
```python
>>> risco_acumulado(df, "tempo", "evento", coluna_grupo="tratamento")
```


---

### `cox_ph`

```python
cox_ph(
    df: pd.DataFrame,
    coluna_tempo: str,
    coluna_evento: str,
    covariaveis: Sequence[str] | None = None,
    *,
    penalizador: float = 0.0,
    verificar_premissas: bool = True,
    plotar: bool = True,
    verbose: bool = True
) -> Any
```

Regressão de Cox (riscos proporcionais) com relatório interpretado.

Para cada covariável reporta o hazard ratio exp(coef) com IC95%:
- HR > 1 -> aumenta o risco (evento acontece mais cedo);
- HR < 1 -> protege (evento demora mais);
- HR = 1.20 lê-se "+20% de risco por unidade da covariável".

Também reporta o índice de concordância (C-index): 0.5 = aleatório,
>0.7 = bom poder discriminativo.

**Parameters**
covariaveis : sequence of str, optional
    Padrão: todas as colunas numéricas exceto tempo/evento.
    Categóricas devem ser codificadas antes (`codificar_categoricas`).
penalizador : float
    Regularização L2 leve (ex.: 0.1) estabiliza coeficientes quando há
    covariáveis correlacionadas ou poucos eventos.
verificar_premissas : bool
    Roda o teste de riscos proporcionais (Schoenfeld). Violações
    indicam usar estratificação ou termos tempo-dependentes.

**Returns**
lifelines.CoxPHFitter ajustado — use `.summary`, `.predict_median(X)`,
`.predict_survival_function(X)`.

**Examples**
```python
>>> cph = cox_ph(df, "tempo", "evento", ["idade", "estagio", "biomarcador"])
>>> cph.predict_median(df_novos_pacientes)
```


---

### `cox_lasso`

```python
cox_lasso(
    df: pd.DataFrame,
    coluna_tempo: str,
    coluna_evento: str,
    covariaveis: Sequence[str] | None = None,
    *,
    penalizadores: Sequence[float] | None = None,
    l1_ratio: float = 1.0,
    cv: int = 5,
    escalar_antes: bool = True,
    random_state: int = 42,
    plotar: bool = True,
    verbose: bool = True
) -> dict[str, Any]
```

Cox penalizado (LASSO/Elastic-Net) com seleção do penalizador por
validação cruzada — seleção de variáveis em alta dimensão (p >> n).

É exatamente o framework Cox-LASSO usado em genômica (ex.: TCGA):
o L1 zera coeficientes de covariáveis irrelevantes, entregando um
subconjunto esparso e interpretável.

Notas metodológicas importantes:
- as covariáveis são padronizadas por default (obrigatório p/ penalização
  justa entre escalas diferentes);
- o C-index de validação cruzada é a métrica de seleção — mais honesto
  que o C-index de treino;
- `l1_ratio=1.0` é LASSO puro; valores em (0,1) dão Elastic-Net, que
  lida melhor com genes/variáveis fortemente correlacionados.

**Parameters**
penalizadores : sequence of float, optional
    Grade de penalização testada (padrão: 10^-3 a 10^1, 9 valores).

**Returns**
dict com:
- "modelo"       : CoxPHFitter final ajustado com o melhor penalizador;
- "melhor_penalizador", "cindex_cv" : resultado da busca;
- "selecionadas" : covariáveis com coeficiente != 0;
- "trajetoria"   : DataFrame penalizador x C-index médio.

**Examples**
```python
>>> res = cox_lasso(df, "tempo_meses", "obito", lista_500_genes)
>>> res["selecionadas"]
```
['gene_BRCA1', 'gene_TP53', 'idade']

---

### `modelos_parametricos_sobrevivencia`

```python
modelos_parametricos_sobrevivencia(
    df: pd.DataFrame,
    coluna_tempo: str,
    coluna_evento: str,
    *,
    modelos: Sequence[str] = (exponencial, weibull, lognormal, loglogistico),
    plotar: bool = True,
    verbose: bool = True
) -> pd.DataFrame
```

Ajusta modelos paramétricos de sobrevivência e compara por AIC.

Modelos e o que cada um assume sobre a taxa de risco h(t):
- "exponencial"  : risco CONSTANTE no tempo (sem memória);
- "weibull"      : risco monotônico — crescente (rho>1) ou decrescente
                   (rho<1); generaliza a exponencial;
- "lognormal"    : risco sobe e depois cai (não monotônico);
- "loglogistico" : similar à lognormal, caudas mais pesadas.

Menor AIC = melhor equilíbrio ajuste x complexidade (diferenças < 2
são empate técnico). Compare também a curva de cada modelo contra o
Kaplan-Meier no gráfico — AIC bom com curva descolada do KM é alerta.

**Returns**
DataFrame: modelo, AIC, log_likelihood, parametros, mediana_prevista
(ordenado por AIC). O objeto ajustado fica na coluna "ajuste".

**Examples**
```python
>>> ranking = modelos_parametricos_sobrevivencia(df, "tempo", "evento")
>>> melhor = ranking.iloc[0]["ajuste"]
>>> melhor.predict(np.array([12, 24, 60]))   # S(t) nesses tempos
```


---

### `prever_sobrevivencia`

```python
prever_sobrevivencia(
    modelo: Any,
    X_novos: pd.DataFrame,
    *,
    tempos: Sequence[float] | None = None,
    plotar: bool = True,
    max_curvas_plot: int = 12,
    verbose: bool = True
) -> pd.DataFrame
```

Prevê curvas de sobrevivência individuais com um modelo de Cox ajustado.

Recebe novos indivíduos (mesmas covariáveis do treino) e retorna S(t)
para cada um — a base de aplicações como "probabilidade de o imóvel
ainda estar à venda em 90 dias" ou "sobrevida em 5 anos do paciente".

**Parameters**
modelo : CoxPHFitter
    Modelo ajustado por `cox_ph` ou `cox_lasso(...)['modelo']`.
tempos : sequence of float, optional
    Tempos específicos de interesse (ex.: [30, 90, 180]). Padrão: grade
    do próprio modelo.

**Returns**
DataFrame: linhas = tempos, colunas = indivíduos, valores = S(t).

**Examples**
```python
>>> curvas = prever_sobrevivencia(cph, df_novos, tempos=[30, 90, 180])
>>> medianas = cph.predict_median(df_novos)   # tempo mediano por indivíduo
```


---
