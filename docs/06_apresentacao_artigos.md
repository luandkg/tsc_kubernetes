# Apresentação dos Artigos

# Parte 1 — KSE

## Título

## Balanceamento de Carga Dinâmico em Ambientes Kubernetes

**Kubernetes Scheduling Extension (KSE)**

Pedro Moritz de Carvalho Neto, Márcio Castro e Frank Siqueira

---

## Contexto

O Kubernetes distribui Pods entre os nós durante a criação.

Isso cria um problema:

- a carga pode mudar durante a execução;
- a distribuição inicial pode deixar de ser eficiente;
- nós podem ficar sobreutilizados;
- outros nós podem ficar subutilizados;
- o escalonador padrão não migra Pods automaticamente.

O artigo trata o rebalanceamento dinâmico como uma necessidade de orquestração.

---

## Problema

O Kubernetes não possui, de forma nativa no experimento analisado, um mecanismo integrado para:

- monitorar o desbalanceamento ao longo do tempo;
- migrar Pods já em execução;
- usar políticas configuráveis de balanceamento;
- substituir algoritmos sem alterar o Kubernetes central.

**Questão do trabalho**

> Como um framework externo pode melhorar o balanceamento da carga entre nós sem afetar significativamente a disponibilidade?

---

## Objetivos

O trabalho apresenta o KSE com os seguintes objetivos:

- atuar durante o ciclo de vida dos Pods;
- obter métricas do Kubernetes;
- admitir métricas externas;
- permitir algoritmos customizáveis;
- executar planos de alocação;
- migrar Pods quando necessário;
- avaliar desbalanceamento e disponibilidade.

---

## Fundamentação

Os componentes principais de um cluster são:

- **Pod:** unidade de carga de trabalho;
- **Node:** máquina que hospeda Pods;
- **Cluster:** conjunto de nós e serviços;
- **Scheduler:** escolhe o nó de novos Pods;
- **KSE:** monitora e reajusta a distribuição existente.

O KSE é uma extensão de software baseada nas APIs do Kubernetes.

---

## Arquitetura do KSE

### Classes principais

- **Cluster:** obtém nós, Pods pendentes e Pods indisponíveis;
- **Node:** fornece métricas e Pods associados;
- **Pod:** permite obtenção de métricas, remoção e reassociação.

### Métricas padrão

- CPU dos nós e Pods;
- memória dos nós e Pods.

O framework permite incorporar:

- APIs externas;
- bases históricas;
- métricas do sistema operacional;
- contadores de hardware.

---

## Fluxo de execução

O KSE executa periodicamente:

1. coleta métricas do cluster;
2. calcula o plano de alocação;
3. compara a distribuição atual;
4. envia migrações pela API;
5. repete a operação no intervalo configurado.

O intervalo utilizado foi de **um minuto**.

---

## Algoritmos propostos

### KSE-Greedy-LB

O algoritmo guloso associa:

- o Pod de maior carga;
- ao nó de menor carga.

Pode gerar:

- forte redução do desbalanceamento;
- muitas migrações;
- maior impacto temporário na disponibilidade.

---

## Algoritmos propostos

### KSE-Refine-LB

O Refine identifica:

- nós pesados;
- nós leves;
- cargas acima da média;
- Pods candidatos à migração.

A operação executa ajustes finos:

- migra somente quando necessário;
- tenta aproximar os nós da carga média;
- reduz o número de mudanças.

Pode produzir menor desbalanceamento em alguns cenários, mas com menor agressividade.

---

## Extensibilidade

A principal contribuição é transformar o balanceamento em um mecanismo extensível.

O desenvolvedor pode:

- escolher a métrica;
- implementar um novo algoritmo;
- definir o intervalo de execução;
- estabelecer o nível de sobrecarga;
- integrar fontes externas.

Isso torna o KSE mais flexível que uma estratégia fixa no escalonador padrão.

---

## Metodologia experimental

**Ambiente**

- Linux com processador Intel Xeon E5-2640 v4;
- 128 GB de RAM;
- Ubuntu 16.04;
- Kubernetes 1.27.3;
- Minikube;
- quatro nós trabalhadores;
- um nó mestre.

**Carga**

- geradores sintéticos de CPU;
- geradores sintéticos de memória;
- k6 para gerar requisições HTTP.

**Métricas coletadas**

- uso de CPU;
- uso de memória;
- número de requisições;
- taxa de sucesso;
- número de migrações;
- desbalanceamento dos nós.

---

## Projeto experimental

| Fator | Níveis |
|---|---|
| Quantidade de Pods | 20 ou 40 |
| Requisições por segundo | 20 ou 40 |
| Taxa das requisições | Constante ou crescimento linear |
| Distribuição | Normal ou exponencial |
| Carga sintética | Memória ou CPU |
| Balanceador | Kubernetes, Greedy-LB ou Refine-LB |

O plano possui 96 cenários experimentais.

Cada execução durou 10 minutos.

---

## Distribuição das cargas

### Distribuição normal

- carga distribuída de forma mais uniforme;
- desbalanceamento inicial menor;
- menor vantagem dos algoritmos do KSE.

### Distribuição exponencial

- concentração maior de carga em alguns Pods;
- desbalanceamento inicial maior;
- maior vantagem do rebalanceamento dinâmico.

---

## Métrica de desbalanceamento

O grau de desbalanceamento foi medido pelo Erro Médio Absoluto:

\[
EMA=\frac{\sum_{i=1}^{D}|x_i-y_i|}{D}
\]

**Interpretação**

- quanto maior o EMA;
- maior é a diferença entre nós;
- pior é o balanceamento.

Quanto mais próximo de zero, melhor.

---

## Resultados — carga normal

Com distribuição normal:

- o escalonador padrão obteve bom balanceamento;
- o KSE apresentou vantagens em apenas 3 dos 16 grupos de cenários;
- o KSE-Refine teve baixa variabilidade;
- o KSE-Greedy apresentou resultados mais instáveis.

**Análise**

Quando o desbalanceamento inicial já é pequeno, migrar Pods adiciona custo sem benefício proporcional.

---

## Resultados — carga exponencial

Com distribuição exponencial:

- o desbalanceamento foi maior;
- pelo menos um balanceador do KSE apresentou melhoria em 7 dos 16 grupos;
- ambos os algoritmos convergiram melhor as cargas dos nós;
- o Refine atuou menos vezes;
- o Greedy realizou mais migrações.

**Análise**

O rebalanceamento é especialmente útil em cenários com concentração desigual de carga.

---

## Migrações e disponibilidade

| Balanceador | Migrações médias | Disponibilidade |
|---|---:|---:|
| Kube-Scheduler | 0 | Aproximadamente 100% |
| KSE-Greedy-LB | 172–196 | 98,37%–99,01% |
| KSE-Refine-LB | 0,48–11,60 | 99,87%–100% |

O Greedy realizou até cerca de 196 migrações em média.

O Refine manteve disponibilidade próxima a 100%.

---

## Conclusões do artigo

O KSE:

- promoveu melhor balanceamento em cargas desbalanceadas;
- obteve melhores resultados em parte dos cenários;
- mostrou maior eficácia com distribuição exponencial;
- apresentou custo-benefício aceitável;
- manteve elevada disponibilidade;
- teve melhor performance com o algoritmo refinado em relação à migração mínima.

**Limitação**

O estudo não avaliou, em igual profundidade:

- carga de aplicações reais;
- outros números de nós;
- outras distribuições;
- exaustão intensa de recursos.

---

# Parte 2 — AdaptScale

## Título

## Arquitetura de Dimensionamento Adaptativo

**Com Suporte ao Aprendizado**

Andre Luiz de J. Gonçalves, Leandro A. Freitas e Antonio Oliveira-Jr

---

## Contexto

Aplicações sensíveis à latência exigem:

- baixa resposta;
- alta disponibilidade;
- uso eficiente de recursos;
- adaptação rápida a picos;
- estabilidade durante a carga variável.

O dimensionamento baseado apenas em hardware pode não representar plenamente o comportamento da aplicação.

---

## Problema

O uso exclusivo de métricas como CPU e memória pode:

- reagir tarde;
- interpretar incorretamente o motivo da sobrecarga;
- produzir replicação desnecessária;
- provocar oscilações;
- atrasar ajustes sob mudanças de demanda.

O trabalho combina métricas de infraestrutura e eventos da aplicação.

---

## Objetivos

A proposta busca:

- melhorar a agilidade do dimensionamento;
- aumentar a estabilidade do sistema;
- correlacionar hardware e aplicação;
- construir conjuntos de dados estruturados;
- permitir evolução futura por modelos preditivos;
- operar sobre microsserviços em Kubernetes.

---

## Arquitetura proposta

O framework é baseado no ciclo **MAPE-K**:

- **Monitor**
- **Analyze**
- **Plan**
- **Execute**
- **Knowledge**

Os componentes são integrados com Kubernetes, KEDA, Prometheus e Grafana.

---

## Monitor

O Monitor:

- coleta métricas do Prometheus;
- padroniza dados;
- sincroniza séries temporais;
- registra uso de CPU;
- registra transmissão de dados;
- registra RPS;
- registra conexões simultâneas.

O script possui objetivo de garantir consistência e reprodutibilidade.

---

## Analyze

O Analyze:

- processa dados históricos;
- identifica padrões;
- calcula estatísticas descritivas;
- gera análises gráficas;
- verifica tendências;
- detecta variações relevantes.

Os dados são organizados em dois conjuntos:

- métricas de hardware;
- métricas combinadas.

---

## Plan

O módulo Plan:

- define thresholds;
- ajusta regras de escalamento;
- utiliza observações empíricas;
- recebe dados históricos;
- está planejado para incorporar modelos preditivos.

Neste trabalho, os thresholds foram definidos manualmente por análise gráfica e testes sucessivos.

---

## Execute

O módulo Execute utiliza o KEDA.

O KEDA:

- amplia o HPA;
- consome métricas externas;
- converte Prometheus em gatilhos;
- solicita ajustes ao Kubernetes;
- controla o número de réplicas.

---

## Knowledge

O Knowledge foi projetado para:

- armazenar conhecimento histórico;
- organizar execuções anteriores;
- apoiar novos thresholds;
- fornecer dados à etapa de planejamento.

No estágio atual, o módulo Knowledge está projetado conceitualmente, enquanto Monitor e Analyze estão consolidados.

---

## Implementação

**Tecnologias**

- Kubernetes;
- KEDA;
- Prometheus;
- Grafana;
- Pandas;
- k6;
- Hubs VR.

**Microsserviços analisados**

- Reticulum;
- Dialog;
- Coturn;
- Nearspark;
- Photomnemonic;
- Hubs;
- Spoke;
- PgSQL;
- PgBouncer.

---

## Cenários experimentais

### Cenário 1 — Métricas de hardware

- uso de CPU;
- transmissão de dados.

### Cenário 2 — Métricas combinadas

- uso de CPU;
- transmissão de dados;
- taxa de requisições por segundo.

O cenário 1 aproxima o comportamento de referência do HPA.

O cenário 2 utiliza eventos da aplicação.

---

## Infraestrutura

**Cluster Digital Ocean**

- primeiro pool:
  - 2 vCPUs;
  - 4 GB de RAM;
- segundo pool:
  - duas instâncias;
  - 1 vCPU;
  - 2 GB de RAM por instância.

A configuração busca resiliencia e distribuição da carga.

---

## Carga aplicada

- carga regular com 55–65 usuários virtuais;
- sobrecarga com até 700 usuários;
- dois eventos progressivos de sobrecarga;
- duração de duas horas por cenário;
- alternância entre carga regular e sobrecarga.

---

## Configuração do KEDA

| Parâmetro | Valor |
|---|---|
| Mínimo de réplicas | 1 |
| Máximo de réplicas | 3 |
| Polling interval | 30 segundos |
| Cooldown | 60 segundos |
| Trigger RPS | 135 |

Os valores foram calibrados com Prometheus e Grafana.

---

## Métricas avaliadas

- requisições por segundo;
- latência;
- tempo de resposta;
- percentual de sucesso;
- número de réplicas;
- uso de CPU;
- uso de memória;
- correlação entre métricas.

---

## Resultados — requisições

| Métrica | Hardware | Combinado |
|---|---:|---:|
| RPS mediano | 24,04 | 27,65 |

O cenário combinado apresentou maior taxa mediana de requisições.

Ganho aproximado:

\[
\frac{27,65-24,04}{24,04}=15\%
\]

---

## Resultados — latência

| Cenário | Latência mediana |
|---|---:|
| Hardware | 0,47 s |
| Combinado | 0,43 s |

Redução aproximada:

\[
\frac{0,47-0,43}{0,47}=8,5\%
\]

O cenário combinado apresentou menor latência e distribuição mais compacta.

---

## Resultados — tempo de resposta

| Cenário | Tempo mediano |
|---|---:|
| Hardware | 0,44 s |
| Combinado | 0,41 s |

Redução aproximada:

\[
\frac{0,44-0,41}{0,44}=6,8\%
\]

O sistema combinado manteve menor tempo de resposta entre réplicas.

---

## Resultados — sucesso e recursos

| Métrica | Hardware | Combinado |
|---|---:|---:|
| Sucesso mediano | 99,85% | 99,90% |
| CPU média | 39,14% | 37,34% |
| Memória média | 85,45% | 81,15% |

A abordagem combinada apresentou:

- maior sucesso;
- menor consumo médio de CPU;
- menor consumo médio de memória.

---

## Correlação das métricas

No cenário de hardware:

- latência e memória do Dialog: correlação 0,1;
- latência e CPU do Spoke: correlação 0,2;
- latência e transmissão no Nearspark: correlação 0,3.

No cenário combinado:

- latência e memória do Dialog: 0,8;
- latência e CPU do Spoke: 0,7;
- latência e transmissão no Nearspark: 0,8.

Os eventos da aplicação revelaram relações não visíveis apenas em hardware.

---

## Estabilidade

O cenário combinado apresentou:

- menor dispersão;
- menor presença de outliers;
- resposta mais consistente;
- latência menor;
- tempo de resposta mais estável.

Isso indica maior previsibilidade em situações de carga variável.

---

## Conclusões do artigo

O trabalho conclui que:

- métricas combinadas apresentam potencial superior;
- a resposta a sobrecargas foi mais rápida;
- o dimensionamento apresentou maior estabilidade;
- o consumo de memória foi menor;
- a relação entre métricas tornou-se mais representativa;
- thresholds precisos continuam sendo um desafio.

Os módulos Plan, Execute e Knowledge ainda exigem consolidação futura.

---

# Aplicação MASA-OpenMP

## O que é o MASA

**MASA — Multi-Platform Architecture for Sequence Aligners**

O MASA é uma arquitetura para alinhamento de sequências biológicas que separa:

- código independente de plataforma;
- implementação específica;
- estratégia de paralelização;
- estratégia de pruning.

É uma aplicação adequada para testes de processamento paralelo em CPU.

---

## Problema resolvido pelo MASA

O alinhamento de sequências utiliza programação dinâmica.

Cada célula depende de:

- célula acima;
- célula à esquerda;
- célula diagonal superior.

Em sequências longas:

- o tempo é elevado;
- a matriz é grande;
- o uso de memória é alto;
- o paralelismo pode reduzir o tempo.

---

## Variantes MASA

- **MASA-OpenMP/CPU:** threads OpenMP;
- **MASA-OmpSs/CPU:** dataflow assíncrono;
- **MASA-CUDAlign:** GPU NVIDIA;
- **MASA-OpenMP/Phi:** Intel Xeon Phi.

A apresentação utiliza a variante:

**MASA-OpenMP 1.0.1.1024**

---

## Estratégia OpenMP

Na variante OpenMP/CPU:

- blocos são associados à matriz dinâmica;
- blocos da mesma diagonal são independentes;
- threads processam os blocos em paralelo;
- a distribuição usa OpenMP com scheduler dinâmico;
- blocos pruned não são calculados.

---

## Aplicação no projeto

O MASA será usado como carga computacional para testar:

- tempo de execução;
- escalabilidade;
- uso de CPU;
- throughput;
- efeito do número de threads;
- overhead de paralelização.

Não substitui os experimentos dos dois artigos Kubernetes.

Complementa a análise com uma carga real de programação dinâmica.

---

## Ambiente utilizado

| Componente | Valor |
|---|---|
| Sistema | Linux x86_64 |
| CPU | Intel Core i5-7200U |
| Núcleos físicos | 2 |
| Threads disponíveis | 4 |
| MASA | 1.0.1.1024 |
| MASA Core | 1.3.9.1024 |

---

## Entrada do experimento

**Sequências**

- `seq1.fa`: 2.000 caracteres;
- `seq2.fa`: 1.994 caracteres.

**Parâmetros Smith-Waterman**

- match: `+1`;
- mismatch: `-3`;
- gap open: `-3`;
- gap extension: `-2`.

**Blocos**

- tamanho de `128 × 128`.

---

## Cenários MASA

Cada cenário usa:

- 1 thread;
- 2 threads;
- 4 threads.

Todas as configurações devem ser executadas pelo menos cinco vezes.

---

## Comando de execução

```bash
OMP_NUM_THREADS=1 ./masa-openmp \
  -c -v 1 --block-size=128,128 \
  --work-dir=/tmp/masa-work \
  /tmp/seq1.fa /tmp/seq2.fa
```

Repita substituindo:

```bash
OMP_NUM_THREADS=2
OMP_NUM_THREADS=4
```

---

## Métricas

- tempo real de execução;
- tempo de usuário;
- tempo de sistema;
- memória máxima;
- média;
- mediana;
- desvio padrão;
- speedup;
- GCUPS, em testes maiores.

---

## Resultados locais

| Threads | Média | Mediana | Desvio padrão |
|---:|---:|---:|---:|
| 1 | 0,036 s | 0,040 s | 0,0167 s |
| 2 | 0,018 s | 0,020 s | 0,0045 s |
| 4 | 0,088 s | 0,040 s | 0,1244 s |

**Speedup mediano**

- duas threads: 2×;
- quatro threads: 1×.

---

## Interpretação

Com a carga pequena:

- duas threads apresentaram aceleração;
- quatro threads não apresentaram ganho adicional;
- uma execução chegou a 0,31 s;
- a inicialização e sincronização tiveram impacto relevante;
- somente dois núcleos físicos estavam disponíveis.

O comportamento não indica que mais threads sejam sempre ineficientes.

Ele indica que a carga não é grande o suficiente para amortizar o overhead.

---

## Evidência publicada do MASA

No artigo original:

- sequências de 10 Kbp até 47 Mbp;
- 12 núcleos para OpenMP/CPU;
- CUDAlign apresentou os melhores tempos em muitas sequências maiores;
- OmpSs obteve até 66,5% de pruning;
- OmpSs apresentou vantagens sobre OpenMP em vários casos.

Esses resultados não são diretamente comparáveis ao piloto local, pois o hardware e o tamanho das entradas são diferentes.

---

## Conclusão do MASA

O MASA-OpenMP oferece uma carga real para avaliar:

- paralelismo;
- escalabilidade;
- pruning;
- sincronização;
- dependências entre blocos;
- tempo de execução.

Para cargas pequenas, o ganho pode ser limitado.

Para cargas maiores, o artigo MASA indica que block pruning e dataflow podem produzir maior benefício.
