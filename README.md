# 🗺️ Score Locacional — Análise de Pontos de Interesse para Terrenos

Case study de uma funcionalidade desenvolvida em **Power BI** para um dashboard de análise de terrenos, que calcula automaticamente um **Score Locacional (0–100)** com base na proximidade a metrô, escolas, hospitais, áreas de lazer e comércio.

> ⚠️ Este repositório contém apenas a documentação técnica e os trechos de código (DAX/HTML) da solução. Os dados reais da empresa (planilha Excel, arquivo `.pbix` e prints completos) não são compartilhados por conterem informações sensíveis.

---

## 📌 Contexto

O dashboard de terrenos precisava responder, para cada terreno cadastrado, uma pergunta simples: **"esse terreno é bem localizado?"**

Em vez de depender de uma análise manual e subjetiva, foi criado um processo que:

1. Estrutura os dados de cada terreno (endereço, latitude, longitude) em uma base no Excel;
2. Usa o Google Maps para levantar os pontos de interesse mais próximos de cada terreno (metrô, escola, hospital, lazer, comércio) e suas respectivas distâncias a pé;
3. Traz essa base para o Power BI, onde cada categoria recebe uma nota (0–100) com base na distância;
4. Combina essas notas em um **Score Locacional único**, ponderado por relevância;
5. Exibe tudo isso em **cards visuais customizados** (via HTML Content), dentro de um mapa interativo com filtros (segmento, empreendimento, linha do metrô, região, distância ao metrô).

---

## 🧱 Estrutura da base de dados

A tabela `Análise de Localizações` contém, por terreno, os seguintes campos:

| Campo | Descrição |
|---|---|
| `Empreendimento` | Nome do terreno/empreendimento |
| `Endereço` | Endereço completo do terreno |
| `CEP` | CEP do terreno |
| `Latitude` / `Longitude` | Coordenadas geográficas do terreno |
| `Estação` | Estação de metrô mais próxima |
| `Linha` | Linha de metrô correspondente |
| `Distância a pé até a estação (m)` | Distância a pé até a estação, em metros |
| `Escola/Faculdade mais próxima` | Instituição de ensino mais próxima |
| `Distância a pé até escola (m)` | Distância a pé até a escola, em metros |
| `Hospital mais próximo` | Hospital/unidade de saúde mais próxima |
| `Distância a pé até hospital (m)` | Distância a pé até o hospital, em metros |
| `Lazer mais próximo (parque/shopping)` | Ponto de lazer mais próximo |
| `Distância a pé até lazer (m)` | Distância a pé até o ponto de lazer, em metros |
| `Comércio mais próximo (mercado)` | Comércio mais próximo |
| `Distância a pé até comércio (m)` | Distância a pé até o comércio, em metros |

As distâncias foram levantadas manualmente no Google Maps, com base na latitude/longitude/endereço de cada terreno.

---

## 🎯 Cálculo do Score Locacional

O score final pondera 5 categorias, cada uma com um peso diferente de acordo com sua relevância para a decisão de investimento:

| Categoria | Peso |
|---|---|
| 🚇 Metrô | 50% |
| 🎓 Educação | 15% |
| 🌳 Lazer | 15% |
| 🏥 Saúde | 10% |
| 🛒 Comércio | 10% |

### Notas por categoria

Cada categoria tem sua própria régua de distância → nota, definida via `SWITCH(TRUE(), ...)` em DAX. Exemplo — Score Metrô:

```dax
Score Metro = 
VAR Dist =
    MAX('Análise de Localizações'[Distância a pé até a estação (m)])

RETURN
SWITCH(
    TRUE(),
    Dist <= 500, 100,
    Dist <= 1000, 90,
    Dist <= 1500, 80,
    Dist <= 2000, 70,
    Dist <= 3000, 50,
    30
)
```

As demais categorias (Educação, Lazer, Saúde, Comércio) seguem a mesma lógica, com faixas de distância ajustadas à realidade de cada tipo de ponto de interesse — ver [`dax/scores_por_categoria.dax`](./dax/scores_por_categoria.dax).

### Score final ponderado

O índice final não usa as medidas de score individuais diretamente — ele recalcula uma nota linear por categoria (`100 - distância/fator`) e aplica os pesos:

```dax
Índice Localização = 
VAR Metro = MAX('Análise de Localizações'[Distância a pé até a estação (m)])
VAR Escola = MAX('Análise de Localizações'[Distância a pé até escola (m)])
VAR Hospital = MAX('Análise de Localizações'[Distância a pé até hospital (m)])
VAR Lazer = MAX('Análise de Localizações'[Distância a pé até lazer (m)])
VAR Comercio = MAX('Análise de Localizações'[Distância a pé até comércio (m)])

VAR NotaMetro = MAX(0, 100 - (Metro / 50))
VAR NotaEscola = MAX(0, 100 - (Escola / 75))
VAR NotaHospital = MAX(0, 100 - (Hospital / 100))
VAR NotaLazer = MAX(0, 100 - (Lazer / 75))
VAR NotaComercio = MAX(0, 100 - (Comercio / 100))

RETURN
ROUND(
    NotaMetro * 0.50 +
    NotaEscola * 0.15 +
    NotaLazer * 0.15 +
    NotaHospital * 0.10 +
    NotaComercio * 0.10,
0
)
```

E formatado para exibição:

```dax
Índice Localização Texto = 
FORMAT([Índice Localização], "0") & "/100"
```

Código completo em [`dax/indice_localizacao.dax`](dax/dax/indice_localizacao.dax).

---

## 🎨 Visualização — Cards com HTML Content

Os pontos de interesse são exibidos em 5 cards visuais (Metrô, Educação, Saúde, Lazer, Comércio), construídos via **HTML Content** do Power BI. Cada card mostra a distância em destaque e o nome do ponto de interesse mais próximo.

```dax
HTML Infraestrutura = 
VAR Comercio = MAX('Análise de Localizações'[Comércio mais próximo (mercado)])
VAR DistComercio = [Distância Comércio Formatada]
VAR Escola = MAX('Análise de Localizações'[Escola/Faculdade mais próxima])
VAR DistEscola = [Distância Escola Formatada]
VAR Hospital = MAX('Análise de Localizações'[Hospital mais próximo])
VAR DistHospital = [Distância Hospital Formatada]
VAR Lazer = MAX('Análise de Localizações'[Lazer mais próximo (parque/shopping)])
VAR DistLazer = [Distância Lazer Formatada]
VAR Metro = MAX('Análise de Localizações'[Estação ])
VAR DistMetro = FORMAT(MAX('Análise de Localizações'[Distância a pé até a estação (m)]), "#,##0 m")

RETURN "<div style='display:flex; ...'> ... </div>"
```

Código completo (5 cards) em [`dax/html_cards.dax`](dax/dax/dax/html_cards.dax).

---

## 🔎 Filtros interativos

O mapa permite cruzar as informações através dos seguintes slicers:

- Segmento
- Empreendimento
- Linha do metrô
- Região
- Distância ao metrô

---

## 🛠️ Tech stack

| Ferramenta | Papel no projeto |
|---|---|
| **Excel** | Base de dados (empreendimento, endereço, latitude, longitude, CEP) |
| **Google Maps** | Levantamento manual dos pontos de interesse e distâncias a pé |
| **Power BI** | Modelagem, cálculos DAX e construção dos cards visuais (HTML Content) |
| **Microsoft Copilot (IA)** | Apoio na construção e otimização do processo e das fórmulas |

---

## 📁 Estrutura deste repositório

```
├── README.md
├── dax/
│   ├── scores_por_categoria.dax
│   ├── indice_localizacao.dax
│   └── html_cards.dax
└── prints/
    └── (imagens do dashboard, sem dados sensíveis)
```

---

## 📄 Licença

Este repositório tem fins exclusivamente demonstrativos/educacionais. Os trechos de código podem ser reaproveitados livremente como referência para projetos similares em Power BI.
