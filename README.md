# Controlador Fuzzy de Apostas Esportivas (Mamdani)

Mini-Projeto 2 da disciplina de Sistemas Baseados em Conhecimento (SBC).

Controlador fuzzy do tipo Mamdani, implementado em scikit-fuzzy, que recomenda o grau de
confiança para apostar na vitória do time mandante em uma partida de futebol. O projeto
segue o **Caminho B** da especificação: reescreve o mesmo domínio do Mini-Projeto 1
(recomendação de apostas), substituindo as regras IF-THEN nítidas por inferência fuzzy.

- Código principal: `sbc_fuzzy_apostas.ipynb`
- Dependências: `requirements.txt`

## Descrição do domínio

O domínio é a recomendação de apostas esportivas em futebol. Dado o contexto de uma
partida, o sistema avalia quão confiável é apostar na vitória do mandante. Três fatores
de domínio resumem a situação:

- **Forma recente do mandante**: desempenho nas últimas partidas.
- **Força de mando**: aproveitamento do time jogando em casa.
- **Risco contextual**: fatores que pesam contra a aposta, como lesões de titulares,
  clima desfavorável e posição ruim na tabela.

A saída é um número contínuo de **confiança (0 a 100)**, defuzzificado e traduzido em uma
banda qualitativa (BAIXA, MÉDIA, ALTA) com a ação correspondente (evitar, cautela,
apostar).

## SBC construído

O sistema é um **controlador fuzzy Mamdani** com três entradas e uma saída:

| Variável | Papel | Universo | Termos linguísticos |
|----------|-------|----------|---------------------|
| `forma` | entrada | 0 a 100 | ruim, média, boa |
| `mando` | entrada | 0 a 100 | fraco, médio, forte |
| `risco` | entrada | 0 a 10 | baixo, médio, alto |
| `confianca` | saída | 0 a 100 | evitar, cautela, apostar |

O fluxo de inferência é: fuzzificação das entradas, avaliação das 13 regras (operador E =
mínimo, agregação = máximo), e **defuzzificação por centroide** da saída `confianca`.

## Motivação

O Mini-Projeto 1 modelou o mesmo domínio com regras nítidas e limiares rígidos
(por exemplo, "aproveitamento em casa maior ou igual a 70 por cento ativa vantagem de
mando"). Esses cortes secos criam descontinuidades artificiais: 69 por cento e 70 por
cento produzem decisões diferentes, embora sejam praticamente o mesmo cenário. Decisões
esportivas, porém, são graduais e incertas por natureza.

A lógica fuzzy foi escolhida por modelar diretamente essa gradação: um time pode estar
"parcialmente em boa forma", e a recomendação varia de forma suave conforme as entradas
mudam. O objetivo do projeto é demonstrar essa diferença de modelagem em relação ao
Mini-Projeto 1, tanto em complexidade quanto em poder de expressividade.

## Premissas

- O foco é a confiança na **vitória do mandante**; empate e vitória do visitante não são
  saídas separadas, e sim representados por confiança baixa/intermediária.
- As três entradas são tratadas como índices já normalizados nos seus universos. Em
  particular, `risco` agrega em um único eixo (0 a 10) fatores que no Mini-Projeto 1 eram
  fatos booleanos distintos (lesões críticas, clima desfavorável, classificação em risco).
- As funções de pertinência são trapezoidais nos extremos (para saturar nos limites do
  universo) e triangulares no centro, com sobreposição intencional entre termos vizinhos.
- A resolução de conflito entre regras não usa prioridades manuais: é resultado natural da
  agregação fuzzy seguida da defuzzificação por centroide.
- A defuzzificação adotada é o centroide (centro de área), padrão do método Mamdani.

## Arquitetura

### Variáveis linguísticas e funções de pertinência

As funções de pertinência foram definidas para cobrir integralmente cada universo, sem
faixas sem cobertura, e com sobreposições que permitem transições suaves.

`forma` (0 a 100):

- ruim: trapezoidal [0, 0, 20, 40]
- média: triangular [30, 50, 70]
- boa: trapezoidal [60, 80, 100, 100]

`mando` (0 a 100):

- fraco: trapezoidal [0, 0, 30, 50]
- médio: triangular [40, 60, 75]
- forte: trapezoidal [65, 80, 100, 100]

`risco` (0 a 10):

- baixo: trapezoidal [0, 0, 2, 4]
- médio: triangular [3, 5, 7]
- alto: trapezoidal [6, 8, 10, 10]

`confianca` (0 a 100, saída):

- evitar: trapezoidal [0, 0, 20, 40]
- cautela: triangular [30, 50, 70]
- apostar: trapezoidal [60, 80, 100, 100]

Justificativa dos termos: cada variável tem três níveis que correspondem a leituras
naturais do domínio (forma fraca/média/boa, mando fraco/médio/forte, risco baixo/médio/
alto). Os limites das funções refletem os limiares que o Mini-Projeto 1 usava de forma
rígida, agora suavizados pelas sobreposições.

### Base de regras (13 regras, sem lacunas)

As regras R1 a R9 formam a grade completa de `forma` x `mando` (3 x 3 = 9 combinações).
Como os termos cobrem todo o universo de cada variável, qualquer entrada ativa ao menos
uma dessas regras, o que garante ausência de lacunas no espaço de entradas. As regras R10
a R13 tratam o eixo `risco`: R10 é um override dominante e R11 a R13 refinam a confiança
conforme o nível de risco.

| # | SE | ENTÃO |
|---|----|-------|
| R1 | forma ruim E mando fraco | evitar |
| R2 | forma ruim E mando médio | evitar |
| R3 | forma ruim E mando forte | cautela |
| R4 | forma média E mando fraco | cautela |
| R5 | forma média E mando médio | cautela |
| R6 | forma média E mando forte | apostar |
| R7 | forma boa E mando fraco | cautela |
| R8 | forma boa E mando médio | apostar |
| R9 | forma boa E mando forte | apostar |
| R10 | risco alto | evitar |
| R11 | risco baixo E forma boa | apostar |
| R12 | risco médio E forma ruim | evitar |
| R13 | risco baixo E mando forte | apostar |

### Fluxo Mamdani

1. Fuzzificação: cada entrada crisp é convertida em graus de pertinência nos seus termos.
2. Avaliação das regras: o operador E usa o mínimo; cada regra ativa seu termo de saída na
   força correspondente.
3. Agregação: os conjuntos de saída ativados são combinados pelo máximo.
4. Defuzzificação: o centroide do conjunto agregado produz a confiança final (0 a 100).
5. Mapeamento qualitativo: confiança abaixo de 40 é BAIXA (evitar), de 40 a 65 é MÉDIA
   (cautela) e acima de 65 é ALTA (apostar). Esses limiares espelham os do score do
   Mini-Projeto 1.

## Desenvolvimento

A implementação está no notebook `sbc_fuzzy_apostas.ipynb`, organizado em seções:

0. Instalação das dependências.
1. Imports.
2. Definição das variáveis linguísticas e funções de pertinência, com visualização das
   curvas.
3. Definição das 13 regras fuzzy.
4. Montagem do sistema de controle (`ControlSystem`) e das funções auxiliares de
   diagnóstico (graus de pertinência ativados, valor defuzzificado, banda e recomendação).
5. Casos de teste comentados.
6. Superfície de controle (mapa contínuo da confiança em função de forma e mando).
7. Comparativo explícito com o Mini-Projeto 1.

A biblioteca usada é o scikit-fuzzy, com o módulo `skfuzzy.control` para construir o
controlador Mamdani de forma declarativa (antecedentes, consequente, regras e simulação).

## Casos de teste e resultados

Os quatro casos a seguir são executados no notebook, com entradas, saída defuzzificada e
interpretação comentada no código.

| Caso | forma | mando | risco | Confiança | Banda | Leitura |
|------|-------|-------|-------|-----------|-------|---------|
| 1. Favorito em casa | 85 | 80 | 1 | 84.44 | ALTA | apostar no mandante |
| 2. Time em crise | 15 | 55 | 9 | 15.56 | BAIXA | evitar a aposta |
| 3. Jogo equilibrado | 50 | 55 | 5 | 50.00 | MÉDIA | cautela |

O Caso 4 varia `mando` de 60 a 80 (com forma 50 e risco 4) para evidenciar a transição
contínua, que substitui o corte seco em 70 por cento do Mini-Projeto 1:

| mando | 60 | 65 | 68 | 69 | 70 | 71 | 72 | 75 | 80 |
|-------|----|----|----|----|----|----|----|----|----|
| confiança | 50.0 | 50.0 | 60.5 | 63.6 | 66.7 | 69.7 | 72.8 | 83.1 | 84.4 |

A saída cresce de forma suave em torno de 70, sem o salto abrupto que o Mini-Projeto 1
produzia ao cruzar o limiar.

## Comparativo explícito: regras nítidas (MP1) x fuzzy (MP2)

### Complexidade

| Aspecto | MP1 (regras nítidas, experta) | MP2 (fuzzy, scikit-fuzzy) |
|---------|-------------------------------|---------------------------|
| Número de regras | 14 | 13 |
| Estrutura | 3 níveis de encadeamento (fatos brutos, derivados, decisão) | 1 nível (entradas para saída) |
| Resolução de conflito | salience manual (90 maior que 80 maior que 75 ...) | automática (agregação máximo + centroide) |
| Cálculo da saída | 3 fórmulas ad-hoc de score (modos positivo, empate, risco) | defuzzificação única por centroide |
| Fatos intermediários | 8 (vantagem de mando, lesões críticas, momentum, etc.) | nenhum |
| Manutenção | alta (ordenar salience, evitar conflito de regras) | baixa (ajustar funções de pertinência e regras) |

### Expressividade

| Situação | MP1 (regras nítidas) | MP2 (fuzzy) |
|----------|----------------------|-------------|
| Aproveitamento 69 vs 70 por cento | decisões diferentes (penhasco em 70) | transição contínua e suave |
| "Forma quase boa" | inexprimível (booleano: é boa ou não) | grau de pertinência parcial |
| Combinação de evidências fracas | exige regra explícita | emerge da agregação das funções de pertinência |
| Saída | rótulo discreto mais score por fórmula | número contínuo mais banda derivada |

### Síntese

- **Complexidade**: o MP1 precisa de encadeamento em níveis, fatos derivados, prioridades
  (salience) para ordenar os disparos e fórmulas separadas para o score. O MP2 elimina
  esses elementos: a própria matemática fuzzy (mínimo, máximo e centroide) resolve
  conflitos e gera a saída.
- **Expressividade**: o MP1 raciocina em degraus (um valor pertence ou não a uma
  categoria), o que cria descontinuidades em limiares como 70 por cento. O MP2 representa
  gradação e incerteza de forma nativa, produzindo uma superfície de decisão contínua.
- **Trade-off**: o MP1 oferece rastreabilidade passo a passo (a cadeia de regras
  disparadas e a origem de cada ponto de score) e decisões categóricas diretas. O MP2
  oferece robustez a pequenas variações e suavidade, ao custo de uma explicabilidade menos
  literal, já que a saída vem da área agregada e não de uma trilha de regras.

## Estrutura dos arquivos

```
.
├── sbc_fuzzy_apostas.ipynb   # controlador fuzzy Mamdani (código principal)
├── requirements.txt          # dependências
└── README.md                 # este arquivo
```
