# Controlador Fuzzy de Apostas Esportivas (Mamdani)

Mini-Projeto 2 da disciplina de Sistemas Baseados em Conhecimento (SBC).

Controlador fuzzy do tipo Mamdani, implementado em scikit-fuzzy, que recomenda o grau de
confianca para apostar na vitoria do time mandante em uma partida de futebol. O projeto
segue o **Caminho B** da especificacao: reescreve o mesmo dominio do Mini-Projeto 1
(recomendacao de apostas), substituindo as regras IF-THEN nitidas por inferencia fuzzy.

- Codigo principal: `sbc_fuzzy_apostas.ipynb`
- Dependencias: `requirements.txt`


## Descricao do dominio

O dominio e a recomendacao de apostas esportivas em futebol. Dado o contexto de uma
partida, o sistema avalia quao confiavel e apostar na vitoria do mandante. Tres fatores
de dominio resumem a situacao:

- **Forma recente do mandante**: desempenho nas ultimas partidas.
- **Forca de mando**: aproveitamento do time jogando em casa.
- **Risco contextual**: fatores que pesam contra a aposta, como lesoes de titulares,
  clima desfavoravel e posicao ruim na tabela.

A saida e um numero continuo de **confianca (0 a 100)**, defuzzificado e traduzido em uma
banda qualitativa (BAIXA, MEDIA, ALTA) com a acao correspondente (evitar, cautela,
apostar).

## SBC construido

O sistema e um **controlador fuzzy Mamdani** com tres entradas e uma saida:

| Variavel | Papel | Universo | Termos linguisticos |
|----------|-------|----------|---------------------|
| `forma` | entrada | 0 a 100 | ruim, media, boa |
| `mando` | entrada | 0 a 100 | fraco, medio, forte |
| `risco` | entrada | 0 a 10 | baixo, medio, alto |
| `confianca` | saida | 0 a 100 | evitar, cautela, apostar |

O fluxo de inferencia e: fuzzificacao das entradas, avaliacao das 13 regras (operador E =
minimo, agregacao = maximo), e **defuzzificacao por centroide** da saida `confianca`.

## Motivacao

O Mini-Projeto 1 modelou o mesmo dominio com regras nitidas e limiares rigidos
(por exemplo, "aproveitamento em casa maior ou igual a 70 por cento ativa vantagem de
mando"). Esses cortes secos criam descontinuidades artificiais: 69 por cento e 70 por
cento produzem decisoes diferentes, embora sejam praticamente o mesmo cenario. Decisoes
esportivas, porem, sao graduais e incertas por natureza.

A logica fuzzy foi escolhida por modelar diretamente essa gradacao: um time pode estar
"parcialmente em boa forma", e a recomendacao varia de forma suave conforme as entradas
mudam. O objetivo do projeto e demonstrar essa diferenca de modelagem em relacao ao
Mini-Projeto 1, tanto em complexidade quanto em poder de expressividade.

## Premissas

- O foco e a confianca na **vitoria do mandante**; empate e vitoria do visitante nao sao
  saidas separadas, e sim representados por confianca baixa/intermediaria.
- As tres entradas sao tratadas como indices ja normalizados nos seus universos. Em
  particular, `risco` agrega em um unico eixo (0 a 10) fatores que no Mini-Projeto 1 eram
  fatos booleanos distintos (lesoes criticas, clima desfavoravel, classificacao em risco).
- As funcoes de pertinencia sao trapezoidais nos extremos (para saturar nos limites do
  universo) e triangulares no centro, com sobreposicao intencional entre termos vizinhos.
- A resolucao de conflito entre regras nao usa prioridades manuais: e resultado natural da
  agregacao fuzzy seguida da defuzzificacao por centroide.
- A defuzzificacao adotada e o centroide (centro de area), padrao do metodo Mamdani.

## Arquitetura

### Variaveis linguisticas e funcoes de pertinencia

As funcoes de pertinencia foram definidas para cobrir integralmente cada universo, sem
faixas sem cobertura, e com sobreposicoes que permitem transicoes suaves.

`forma` (0 a 100):

- ruim: trapezoidal [0, 0, 20, 40]
- media: triangular [30, 50, 70]
- boa: trapezoidal [60, 80, 100, 100]

`mando` (0 a 100):

- fraco: trapezoidal [0, 0, 30, 50]
- medio: triangular [40, 60, 75]
- forte: trapezoidal [65, 80, 100, 100]

`risco` (0 a 10):

- baixo: trapezoidal [0, 0, 2, 4]
- medio: triangular [3, 5, 7]
- alto: trapezoidal [6, 8, 10, 10]

`confianca` (0 a 100, saida):

- evitar: trapezoidal [0, 0, 20, 40]
- cautela: triangular [30, 50, 70]
- apostar: trapezoidal [60, 80, 100, 100]

Justificativa dos termos: cada variavel tem tres niveis que correspondem a leituras
naturais do dominio (forma fraca/media/boa, mando fraco/medio/forte, risco baixo/medio/
alto). Os limites das funcoes refletem os limiares que o Mini-Projeto 1 usava de forma
rigida, agora suavizados pelas sobreposicoes.

### Base de regras (13 regras, sem lacunas)

As regras R1 a R9 formam a grade completa de `forma` x `mando` (3 x 3 = 9 combinacoes).
Como os termos cobrem todo o universo de cada variavel, qualquer entrada ativa ao menos
uma dessas regras, o que garante ausencia de lacunas no espaco de entradas. As regras R10
a R13 tratam o eixo `risco`: R10 e um override dominante e R11 a R13 refinam a confianca
conforme o nivel de risco.

| # | SE | ENTAO |
|---|----|-------|
| R1 | forma ruim E mando fraco | evitar |
| R2 | forma ruim E mando medio | evitar |
| R3 | forma ruim E mando forte | cautela |
| R4 | forma media E mando fraco | cautela |
| R5 | forma media E mando medio | cautela |
| R6 | forma media E mando forte | apostar |
| R7 | forma boa E mando fraco | cautela |
| R8 | forma boa E mando medio | apostar |
| R9 | forma boa E mando forte | apostar |
| R10 | risco alto | evitar |
| R11 | risco baixo E forma boa | apostar |
| R12 | risco medio E forma ruim | evitar |
| R13 | risco baixo E mando forte | apostar |

### Fluxo Mamdani

1. Fuzzificacao: cada entrada crisp e convertida em graus de pertinencia nos seus termos.
2. Avaliacao das regras: o operador E usa o minimo; cada regra ativa seu termo de saida na
   forca correspondente.
3. Agregacao: os conjuntos de saida ativados sao combinados pelo maximo.
4. Defuzzificacao: o centroide do conjunto agregado produz a confianca final (0 a 100).
5. Mapeamento qualitativo: confianca abaixo de 40 e BAIXA (evitar), de 40 a 65 e MEDIA
   (cautela) e acima de 65 e ALTA (apostar). Esses limiares espelham os do score do
   Mini-Projeto 1.

## Desenvolvimento

A implementacao esta no notebook `sbc_fuzzy_apostas.ipynb`, organizado em secoes:

0. Instalacao das dependencias.
1. Imports.
2. Definicao das variaveis linguisticas e funcoes de pertinencia, com visualizacao das
   curvas.
3. Definicao das 13 regras fuzzy.
4. Montagem do sistema de controle (`ControlSystem`) e das funcoes auxiliares de
   diagnostico (graus de pertinencia ativados, valor defuzzificado, banda e recomendacao).
5. Casos de teste comentados.
6. Superficie de controle (mapa continuo da confianca em funcao de forma e mando).
7. Comparativo explicito com o Mini-Projeto 1.

A biblioteca usada e o scikit-fuzzy, com o modulo `skfuzzy.control` para construir o
controlador Mamdani de forma declarativa (antecedentes, consequente, regras e simulacao).

## Casos de teste e resultados

Os quatro casos a seguir sao executados no notebook, com entradas, saida defuzzificada e
interpretacao comentada no codigo.

| Caso | forma | mando | risco | Confianca | Banda | Leitura |
|------|-------|-------|-------|-----------|-------|---------|
| 1. Favorito em casa | 85 | 80 | 1 | 84.44 | ALTA | apostar no mandante |
| 2. Time em crise | 15 | 55 | 9 | 15.56 | BAIXA | evitar a aposta |
| 3. Jogo equilibrado | 50 | 55 | 5 | 50.00 | MEDIA | cautela |

O Caso 4 varia `mando` de 60 a 80 (com forma 50 e risco 4) para evidenciar a transicao
continua, que substitui o corte seco em 70 por cento do Mini-Projeto 1:

| mando | 60 | 65 | 68 | 69 | 70 | 71 | 72 | 75 | 80 |
|-------|----|----|----|----|----|----|----|----|----|
| confianca | 50.0 | 50.0 | 60.5 | 63.6 | 66.7 | 69.7 | 72.8 | 83.1 | 84.4 |

A saida cresce de forma suave em torno de 70, sem o salto abrupto que o Mini-Projeto 1
produzia ao cruzar o limiar.

## Comparativo explicito: regras nitidas (MP1) x fuzzy (MP2)

### Complexidade

| Aspecto | MP1 (regras nitidas, experta) | MP2 (fuzzy, scikit-fuzzy) |
|---------|-------------------------------|---------------------------|
| Numero de regras | 14 | 13 |
| Estrutura | 3 niveis de encadeamento (fatos brutos, derivados, decisao) | 1 nivel (entradas para saida) |
| Resolucao de conflito | salience manual (90 maior que 80 maior que 75 ...) | automatica (agregacao maximo + centroide) |
| Calculo da saida | 3 formulas ad-hoc de score (modos positivo, empate, risco) | defuzzificacao unica por centroide |
| Fatos intermediarios | 8 (vantagem de mando, lesoes criticas, momentum, etc.) | nenhum |
| Manutencao | alta (ordenar salience, evitar conflito de regras) | baixa (ajustar funcoes de pertinencia e regras) |

### Expressividade

| Situacao | MP1 (regras nitidas) | MP2 (fuzzy) |
|----------|----------------------|-------------|
| Aproveitamento 69 vs 70 por cento | decisoes diferentes (penhasco em 70) | transicao continua e suave |
| "Forma quase boa" | inexprimivel (booleano: e boa ou nao) | grau de pertinencia parcial |
| Combinacao de evidencias fracas | exige regra explicita | emerge da agregacao das funcoes de pertinencia |
| Saida | rotulo discreto mais score por formula | numero continuo mais banda derivada |

### Sintese

- **Complexidade**: o MP1 precisa de encadeamento em niveis, fatos derivados, prioridades
  (salience) para ordenar os disparos e formulas separadas para o score. O MP2 elimina
  esses elementos: a propria matematica fuzzy (minimo, maximo e centroide) resolve
  conflitos e gera a saida.
- **Expressividade**: o MP1 raciocina em degraus (um valor pertence ou nao a uma
  categoria), o que cria descontinuidades em limiares como 70 por cento. O MP2 representa
  gradacao e incerteza de forma nativa, produzindo uma superficie de decisao continua.
- **Trade-off**: o MP1 oferece rastreabilidade passo a passo (a cadeia de regras
  disparadas e a origem de cada ponto de score) e decisoes categoricas diretas. O MP2
  oferece robustez a pequenas variacoes e suavidade, ao custo de uma explicabilidade menos
  literal, ja que a saida vem da area agregada e nao de uma trilha de regras.

## Estrutura dos arquivos

```
.
├── sbc_fuzzy_apostas.ipynb   # controlador fuzzy Mamdani (codigo principal)
├── requirements.txt          # dependencias
└── README.md                 # este arquivo
```
