# DS Toolkit

![Python](https://img.shields.io/badge/python-3.10%2B-blue)
![Funções](https://img.shields.io/badge/funções-66-brightgreen)
![Seções](https://img.shields.io/badge/seções-7-orange)
![Licença](https://img.shields.io/badge/licença-MIT-lightgrey)

Um módulo Python único com **66 funções** que resolvem os problemas recorrentes
do dia a dia em Data Science — da ingestão bagunçada de um CSV até modelos de
análise de sobrevivência.

Nasceu de uma constatação simples: em todo projeto novo eu reescrevia o mesmo
código para ler um arquivo com encoding errado, converter `"R$ 1.234,56"` em
número, decidir qual teste estatístico usar e montar a mesma matriz de confusão.
Este repositório é essa camada repetitiva, escrita uma vez e bem escrita.

```python
import ds_toolkit as dst

df = dst.carregar_dados("vendas.csv", decimal_brasileiro=True)
df = dst.limpar_nomes_colunas(df)
dst.relatorio_qualidade(df)
```

---

## Por que usar

**Foco em dados brasileiros.** Datas em `dd/mm/aaaa`, números com vírgula
decimal, `R$` no meio da string, encoding `latin-1` que veio do sistema legado —
tudo tratado por padrão, sem gambiarra.

**Decisões estatísticas automatizadas, mas transparentes.** `comparar_grupos()`
escolhe entre t de Welch, Mann-Whitney, ANOVA e Kruskal-Wallis conforme
normalidade e número de grupos, e **sempre reporta o tamanho de efeito junto do
p-valor** — porque significância não é relevância.

**Armadilhas comuns viram avisos explícitos.** O merge avisa quando um `left
join` multiplicou linhas silenciosamente. O Cox alerta quando há menos de 10
eventos por covariável. O teste A/B diz quando "não significativo" na verdade
significa "amostra pequena demais para concluir". A curva de aprendizado
diagnostica overfitting e responde se mais dados ajudariam.

**Documentação de verdade.** Cada função tem docstring completa com parâmetros,
retornos, exemplos e — o mais importante — *quando não usar* aquela abordagem.

---

## Instalação

```bash
git clone https://github.com/RafaelGomide/ds-toolkit.git
cd ds-toolkit
pip install -r requirements.txt
```

Ou simplesmente copie `ds_toolkit.py` para a pasta do seu projeto.

**Dependências obrigatórias:**

```bash
pip install pandas numpy scipy scikit-learn matplotlib seaborn
```

**Opcionais** (o módulo importa sem elas; cada uma habilita funções
específicas):

| Biblioteca | Habilita |
|---|---|
| `lifelines` | Seção 7 inteira — análise de sobrevivência |
| `joblib` | `salvar_modelo` / `carregar_modelo` |
| `requests` | `carregar_api` |
| `sqlalchemy` | `carregar_sql` com bancos além do SQLite |
| `pyarrow` | Leitura e escrita de `.parquet` / `.feather` |
| `openpyxl` | Leitura e escrita de `.xlsx` |

---

## As 7 seções

| # | Seção | Funções | O que cobre |
|---|---|---|---|
| 1 | **Ingestão** | 5 | CSV, Excel, JSON, Parquet, SQL, APIs REST com retry, leitura em lote de pastas |
| 2 | **ETL** | 13 | Diagnóstico de qualidade, nulos, duplicatas, outliers, encoding de categóricas, otimização de memória, merge com auditoria |
| 3 | **EDA** | 7 | Estatísticas descritivas, correlações, testes de normalidade, comparação de grupos, ranking de features vs alvo |
| 4 | **Gráficos** | 10 | Distribuições, correlação, séries temporais, QQ-plot, pairplot — tema claro/escuro |
| 5 | **Machine Learning** | 14 | Pipelines sem vazamento, comparação de baselines, tuning, importância de features, curva de aprendizado, PCA, balanceamento |
| 6 | **Matemática / Estatística** | 10 | Intervalos de confiança, bootstrap, tamanho de amostra, teste A/B com poder, VIF, ajuste de distribuições, estatísticas robustas |
| 7 | **Análise de Sobrevivência** | 7 | Kaplan-Meier, Nelson-Aalen, Cox PH, **Cox-LASSO**, modelos paramétricos, previsão individual de S(t) |

A referência completa de todas as funções está em
**[DOCUMENTACAO.md](DOCUMENTACAO.md)**.

---

## Uso

### Fluxo padrão de um projeto

```python
import ds_toolkit as dst

# 1. Ingestão — detecta formato, encoding e separador sozinho
df = dst.carregar_dados("dados.csv", decimal_brasileiro=True)

# 2. ETL
df = dst.limpar_nomes_colunas(df)          # "Preço (R$)" -> "preco_r"
dst.relatorio_qualidade(df)                # flags: constante, ID, muitos nulos...
df = dst.converter_tipos(df, colunas_data=["data"], colunas_numericas=["preco"])
df = dst.tratar_duplicados(df)
df = dst.tratar_nulos(df, "mediana", criar_indicador=True)
df = dst.remover_outliers(df, ["preco"], metodo="iqr", acao="limitar")

# 3. EDA
dst.analise_alvo(df, "preco")              # ranking de todas as features
dst.comparar_grupos(df, "preco", "bairro") # escolhe o teste certo sozinho

# 4. Gráficos
dst.configurar_estilo("escuro")
dst.plot_distribuicao(df, "preco", log_x=True)
dst.plot_correlacao(df, metodo="spearman")

# 5. Modelagem
X_tr, X_te, y_tr, y_te = dst.preparar_dados(df, "preco")
pre = dst.pipeline_preprocessamento(X_tr)   # imputação + escala + one-hot
ranking = dst.comparar_modelos(X_tr, y_tr, "regressao", preprocessador=pre)
dst.avaliar_regressao(modelo, X_te, y_te)
dst.salvar_modelo(modelo, "modelos/final.joblib", metadados={"rmse": 45000})
```

### Análise de sobrevivência

```python
# Formato: coluna de TEMPO (duração > 0) e de EVENTO (1 = ocorreu, 0 = censura)
df = dst.preparar_sobrevivencia(df, "tempo_meses", "status",
                                mapa_evento={"Dead": 1, "Alive": 0})

# Não-paramétrico: curvas, medianas com IC e log-rank entre grupos
dst.kaplan_meier(df, "tempo_meses", "status", coluna_grupo="tratamento")

# Semi-paramétrico: hazard ratios + checagem da premissa de riscos proporcionais
cph = dst.cox_ph(df, "tempo_meses", "status", ["idade", "biomarcador"])

# Alta dimensão (p >> n): Cox-LASSO com penalizador escolhido por validação cruzada
res = dst.cox_lasso(df, "tempo_meses", "status", lista_de_genes)
res["selecionadas"]   # subconjunto esparso de covariáveis relevantes

# Previsão individual de S(t)
dst.prever_sobrevivencia(cph, df_novos, tempos=[6, 12, 24])
```

### Estatística aplicada

```python
# Teste A/B completo: lift, IC da diferença e poder observado
dst.teste_ab(conversoes_a=120, total_a=2400, conversoes_b=156, total_b=2350)
# -> avisa se um resultado "não significativo" é apenas inconclusivo

# Quantas amostras eu preciso?
dst.tamanho_amostra(tipo="proporcao", margem_erro=0.03)

# IC para qualquer estatística, sem fórmula fechada
dst.bootstrap_estatistica(df["preco"], lambda a: np.percentile(a, 90))

# Multicolinearidade antes de rodar regressão linear
dst.calcular_vif(df[["area", "quartos", "banheiros", "vagas"]])
```

---

## Convenções do módulo

- **Imutabilidade** — nenhuma função altera o DataFrame original. Sempre
  reatribua: `df = dst.tratar_nulos(df, "mediana")`.
- **`verbose=True`** por padrão — cada função imprime um relatório legível do
  que fez. Use `verbose=False` em pipelines silenciosos.
- **Funções de plot** retornam `(fig, ax)` para customização e aceitam
  `salvar_em="figs/grafico.png"`.
- **`random_state=42`** em tudo que envolve aleatoriedade — reprodutibilidade
  por padrão.
- **Sem vazamento de dados** — `pipeline_preprocessamento()` encapsula o
  pré-processamento num `ColumnTransformer`, garantindo que o `fit` aconteça
  apenas no treino, inclusive dentro de cada fold da validação cruzada.

---

## Estrutura do repositório

```
ds-toolkit/
├── ds_toolkit.py       # o módulo (~3.980 linhas, 66 funções)
├── DOCUMENTACAO.md     # referência completa de todas as funções
├── requirements.txt
├── LICENSE
└── README.md
```

---

## Roadmap

- [ ] Testes automatizados com `pytest` e cobertura em CI
- [ ] Publicação no PyPI (`pip install ds-toolkit`)
- [ ] Seção de séries temporais (decomposição, estacionariedade, ARIMA)
- [ ] Suporte a `polars` como backend alternativo ao pandas
- [ ] Modelos de sobrevivência de machine learning (Random Survival Forest)

Sugestões e issues são bem-vindas.

---

## Licença

MIT — use, modifique e distribua livremente. Veja [LICENSE](LICENSE).

## Autor

**Rafael Gomide** — Ciência de Dados @ UVV | Consultor freelance de dados e BI

Construído a partir de projetos reais: capstone de precificação imobiliária,
sistema de monitoramento legislativo, digitalização de arquivo clínico e
pesquisa em análise de sobrevivência.
