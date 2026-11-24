# Tema 9 — Comparação de Frameworks de Orquestração no Kubernetes

## Objetivo

Comparar duas propostas de gerenciamento dinâmico de recursos em clusters Kubernetes:

1. **Kubernetes Scheduling Extension (KSE)**: framework externo para rebalanceamento dinâmico de Pods entre nós.
2. **Arquitetura Adaptativa com Suporte ao Aprendizado (AdaptScale)**: arquitetura baseada no ciclo MAPE-K e no KEDA para dimensionamento adaptativo de réplicas.

A aplicação **MASA-OpenMP** será utilizada como carga de trabalho computacional adicional para avaliar tempo de execução, escalabilidade e uso de recursos em ambiente local.

## Motivação da seleção dos artigos

Os dois artigos desta proposta foram selecionados a partir de uma busca preliminar no repositório de artigos da SBC, utilizando palavras-chave relacionadas ao tema de orquestração de contêineres e escalonamento no Kubernetes. Entre os termos utilizados, destacam-se: **Kubernetes**, **balanceamento de carga**, **dinâmico**, **dimensionamento adaptativo**, **autoscaling**, **orquestração** e **eventos da aplicação**.

Essa seleção corresponde a uma etapa inicial de triagem bibliográfica, na qual a escolha foi baseada em título, resumo, palavras-chave e aderência aparente à proposta do tema. A leitura detalhada será realizada nas próximas etapas do projeto e poderá confirmar, ajustar ou substituir os trabalhos selecionados.

Os artigos foram escolhidos porque representam duas abordagens complementares ao problema de gerenciamento dinâmico no Kubernetes:

- o KSE investiga o posicionamento e o rebalanceamento de Pods entre nós;
- o AdaptScale investiga o dimensionamento adaptativo do número de réplicas.

Essa diferença permite construir uma comparação inicial mais ampla, relacionando controle de posicionamento, escalabilidade, desempenho e uso de recursos. A aplicação MASA-OpenMP foi incorporada como carga experimental complementar para verificar o comportamento paralelo em uma aplicação real de alinhamento de sequências.

## Questão de pesquisa

> Qual framework oferece melhor combinação entre desempenho, estabilidade e eficiência de recursos — o rebalanceamento dinâmico do KSE ou o dimensionamento adaptativo do AdaptScale — quando submetidos a condições comparáveis de carga?

## Hipótese principal

**H1.** O AdaptScale deve apresentar melhor latência e estabilidade sob carga variável, pois combina métricas de infraestrutura e eventos da aplicação. O KSE deve apresentar melhor redução de desbalanceamento em cargas espacialmente heterogêneas, pois realiza migrações explícitas de Pods.

## Hipóteses secundárias

- **H2.** Estratégias conservadoras de realocação devem preservar melhor a disponibilidade do que estratégias agressivas.
- **H3.** O MASA-OpenMP deve apresentar ganho limitado em matrizes pequenas, aumentando o benefício de execução paralela à medida que o tamanho das sequências cresce.
