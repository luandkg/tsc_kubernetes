# Comparação de KSE e AdaptScale com Validação da Aplicação MASA-OpenMP

## Resumo

Este trabalho compara o Kubernetes Scheduling Extension (KSE), uma extensão para rebalanceamento dinâmico de Pods, e a Arquitetura de Dimensionamento Adaptativo com Suporte ao Aprendizado (AdaptScale), baseada no ciclo MAPE-K e no Kubernetes Event-driven Autoscaler (KEDA). A questão central é identificar qual proposta oferece melhor equilíbrio entre desempenho, estabilidade, disponibilidade e uso de recursos em diferentes condições de carga.

A evidência publicada pelo artigo do KSE demonstra que rebalanceamento dinâmico reduz desbalanceamento, sobretudo em cargas exponenciais, mas pode adicionar migrações e pequenas perdas de disponibilidade. O artigo do AdaptScale mostra vantagens do dimensionamento com métricas combinadas em relação ao uso exclusivo de indicadores de infraestrutura: a taxa mediana de requisições aumentou de 24,04 para 27,65 RPS, a latência caiu de 0,47 para 0,43 segundos e o tempo de resposta caiu de 0,44 para 0,41 segundos.

A aplicação MASA-OpenMP foi utilizada em um piloto local com sequências de aproximadamente 2.000 caracteres e cinco repetições em 1, 2 e 4 threads. As medianas foram 0,04 s, 0,02 s e 0,04 s, respectivamente. O cenário de duas threads obteve speedup mediano de 2× sobre uma thread, mas quatro threads não apresentaram ganho adicional porque a carga era pequena e dominada por sobrecarga de inicialização e sincronização. Os dados dos artigos e do piloto são complementares e não devem ser comparados como repetições do mesmo experimento.

**Palavras-chave:** Kubernetes; rebalanceamento; autoscaling; KEDA; MAPE-K; MASA; OpenMP.

---

## 1. Introdução

Clusters Kubernetes podem desbalancear carga entre nós depois da criação dos Pods, enquanto alterações na demanda de aplicações podem exigir ajustes no número de réplicas. A literatura oferece soluções reativas e preventivas para esses problemas, mas seus mecanismos são distintos:

- o KSE atua sobre a localização dos Pods;
- o AdaptScale atua sobre a quantidade de réplicas e utiliza sinais da aplicação;
- o MASA-OpenMP fornece uma carga computacional independente para validar escalabilidade local e paralelismo.

O objetivo deste estudo é comparar os dois frameworks propostos em termos de:

1. mecanismo de controle;
2. métricas utilizadas;
3. unidade gerenciada;
4. resultados quantitativos publicados;
5. custo operacional;
6. aplicabilidade e ameaças à validade.

### 1.1 Questão de pesquisa

> Qual estratégia proporciona melhor combinação de desempenho, estabilidade e eficiência de recursos: rebalanceamento dinâmico de Pods com KSE ou dimensionamento adaptativo baseado em hardware e eventos de aplicação com AdaptScale?

### 1.2 Hipóteses

- **H1.** O AdaptScale apresenta menor latência e maior estabilidade sob carga variável por integrar eventos da aplicação.
- **H2.** O KSE reduz mais o desbalanceamento em cargas espacialmente heterogêneas.
- **H3.** Estratégias agressivas reduzem desbalanceamento à custa de mais migrações e menor disponibilidade.
- **H4.** Uma carga MASA pequena apresenta ganho limitado com mais threads devido ao overhead de paralelização.

---

## 2. Fundamentação Teórica

### 2.1 Kubernetes e balanceamento

O Kubernetes cria Pods nos nós por meio do escalonador padrão. Essa decisão é feita principalmente no momento da criação e não migra automaticamente Pods já existentes quando o perfil de carga muda. Dessa forma, o cluster pode ficar desbalanceado mesmo quando no início estava homogêneo.

Migrações têm custos: transferência de estado, reinicialização ou indisponibilidade temporária, interrupção de conexões e troca de tarefas pelo plano de controle. Por isso, uma solução de rebalanceamento precisa equilibrar ganho de distribuição contra estabilidade.

### 2.2 Autoscaling

O HPA ajusta o número de réplicas com base em métricas como CPU e memória. O HPA é uma referência reativa adequada para carga de infraestrutura, mas pode reagir tarde ou de forma indireta a eventos específicos da aplicação. O KEDA amplia o HPA ao consumir gatilhos externos, como métricas Prometheus relacionadas à taxa de requisições.

### 2.3 MAPE-K

O modelo MAPE-K estrutura o controle adaptativo em:

1. **Monitor**: coleta dados;
2. **Analyze**: identifica padrões;
3. **Plan**: define ajustes e thresholds;
4. **Execute**: aplica mudanças;
5. **Knowledge**: mantém informações históricas.

Esse ciclo permite decisões contextualizadas, mas depende de boa calibração dos thresholds e da qualidade das métricas.

### 2.4 MASA

O MASA separa código independente de plataforma de implementações específicas. O problema é dividido em blocos da matriz de programação dinâmica. Dependências entre blocos permitem ordem diagonal ou orientada por dados.

Na variante OpenMP/CPU:

- blocos da mesma diagonal são processados em paralelo;
- a distribuição OpenMP utiliza scheduling dinâmico;
- o pruning evita o cálculo de blocos que não podem contribuir para o alinhamento ótimo.

O MASA suporta alinhamentos local, global e semiglobal e utiliza Smith-Waterman, Needleman-Wunsch e variantes semiglobal. O artigo original apresenta implementações para OpenMP, OmpSs, CUDA e Intel Phi.

---

## 3. Revisão dos Trabalhos

### 3.1 KSE

O KSE é um arcabouço externo ao Kubernetes que monitora nós e Pods, calcula um plano de alocação e aplica migrações pela API do Kubernetes. Ele utiliza CPU e memória por padrão e permite fontes externas.

Os algoritmos implementados são:

| Algoritmo | Estratégia | Principal risco |
|---|---|---|
| KSE-Greedy-LB | Associa o Pod mais carregado ao nó menos carregado | Muitas migrações |
| KSE-Refine-LB | Ajusta apenas pares que aproximam cargas leves e pesadas | Menor ganho em alguns cenários |
| Kube-Scheduler | Alocação padrão sem rebalanceamento posterior | Desbalanceamento persistente |

O KSE foi avaliado em ambiente com quatro nós trabalhadores, 2 CPUs, 2 GB de RAM e 5 GB de disco por nó, sob Minikube. Foram usados geradores sintéticos de CPU e memória e carga HTTP com k6.

### 3.2 AdaptScale

O AdaptScale integra Kubernetes, KEDA, Prometheus e Grafana. Monitor coleta CPU, transmissão de dados, RPS e conexões; Analyze identifica tendências; Plan define thresholds; Execute aplica ajustes por KEDA; Knowledge preserva histórico.

A implementação publicada usou dois cenários:

| Cenário | Métricas | Baseline |
|---|---|---|
| Hardware | CPU e transmissão de dados | Aproxima o HPA |
| Combinado | CPU/transmissão e RPS | KEDA com evento da aplicação |

O threshold foi definido empiricamente em 135 RPS. Foram utilizados 1 a 3 réplicas, polling de 30 segundos e cooldown de 60 segundos.

### 3.3 Comparação dos frameworks

| Dimensão | KSE | AdaptScale |
|---|---|---|
| Objetivo | Rebalancear Pods existentes | Ajustar quantidade de réplicas |
| Tipo de controle | Reativo com migração | Reativo adaptativo |
| Mecanismo | API de Kubernetes | KEDA/HPA |
| Unidade | Pod e nó | Réplica e microsserviço |
| Métricas básicas | CPU e memória | CPU, rede e RPS |
| Extensibilidade | Algoritmos e métricas | MAPE-K e triggers |
| Estado histórico | Não central na proposta básica | Knowledge planejado |
| Benefício típico | Corrige desbalanceamento espacial | Responde à demanda temporal |
| Custo principal | Migrações e indisponibilidade | Oscilações de escala e thresholds |

---

## 4. Metodologia Experimental

### 4.1 Planejamento

O projeto segue os requisitos da metodologia: problema, hipóteses, baseline, cenários, repetições, métricas, custo, reprodutibilidade e ameaça à validade.

### 4.2 Experimento do KSE

O projeto experimental do artigo 1 é fatorial completo:

| Fator | Níveis |
|---|---|
| Quantidade de Pods | 20, 40 |
| Requisições por segundo | 20, 40 |
| Taxa | Constante, crescimento linear |
| Distribuição | Normal, exponencial |
| Carga | CPU, memória |
| Balanceador | Kubernetes, Greedy-LB, Refine-LB |

Foram produzidos 96 cenários. Cada cenário durou 10 minutos e o balanceador foi executado a cada minuto. Os resultados foram obtidos em dez execuções por cenário.

### 4.3 Experimento do AdaptScale

O artigo 2 executou dois cenários de duas horas em cluster Digital Ocean:

- um pool com 2 vCPUs e 4 GB;
- dois pools com 1 vCPU e 2 GB;
- carga regular entre 55 e 65 usuários virtuais;
- sobrecarga de até 700 usuários;
- dois eventos progressivos de sobrecarga.

O mínimo não foi repetido por cenário na descrição do artigo. Por isso, os resultados devem ser tratados como evidência exploratória, não como teste inferencial robusto.

### 4.4 Experimento MASA-OpenMP

#### Ambiente local

| Componente | Valor |
|---|---|
| Sistema | Linux x86_64 |
| CPU | Intel Core i5-7200U |
| Núcleos físicos | 2 |
| Threads disponíveis | 4 |
| MASA | 1.0.1.1024 |
| MASA Core | 1.3.9.1024 |

#### Entradas

- `seq1.fa`: sequência homogênea de 2.000 bases;
- `seq2.fa`: sequência de 1.994 bases, com 1.990 bases `A` seguidas de `CGTA`;
- alinhamento local com bloco de `128 × 128`;
- 1, 2 e 4 threads OpenMP;
- cinco execuções por configuração.

Comando reproducível:

```bash
OMP_NUM_THREADS=1 ./masa-openmp \
  -c -v 1 --block-size=128,128 \
  --work-dir=/tmp/masa-work \
  /tmp/seq1.fa /tmp/seq2.fa
```

Os valores registrados foram obtidos com `/usr/bin/time -f "%e %U %S %M"`.

### 4.5 Métricas

| Dimensão | Métrica |
|---|---|
| Desempenho | Tempo de execução, latência, tempo de resposta, RPS |
| Escalabilidade | Speedup e GCUPS |
| Recursos | CPU, memória e número de réplicas |
| Balanceamento | EMA da carga entre nós |
| Confiabilidade | Taxa de sucesso e disponibilidade |
| Operação | Migrações, cooling period e esforço de configuração |
| Custo | Tempo de CPU, infraestrutura e custo por execução |

---

## 5. Resultados

### 5.1 Resultados do KSE

Em carga com distribuição normal, os KSE não superaram o Kubernetes padrão em muitos cenários. Isso ocorre porque a carga já apresenta desbalanceamento relativamente baixo. Em carga exponencial, os dois balanceadores do KSE obtiveram melhora em pelo menos metade das comparações.

O KSE-Refine apresentou menor variabilidade, pois faz menos migrações. O KSE-Greedy apresentou desbalanceamento baixo em muitos cenários, mas realizou muitas migrações.

| Balanceador | Disponibilidade mínima | Migrações médias |
|---|---|---|
| Kubernetes | Aproximadamente 100% | 0 |
| KSE-Greedy-LB | 98,37% | 171,95 a 195,80 |
| KSE-Refine-LB | 99,87% | 0,48 a 11,60 |

Os valores acima agregam 96 cenários. A disponibilidade dos algoritmos diminuiu à medida que o número de migrações aumentou.

### 5.2 Resultados do AdaptScale

| Métrica | Hardware | Combinado | Direção |
|---|---|---|---|
| RPS mediano | 24,04 | 27,65 | +15,0% |
| Latência mediana | 0,47 s | 0,43 s | -8,5% |
| Tempo de resposta mediano | 0,44 s | 0,41 s | -6,8% |
| Sucesso mediano | 99,85% | 99,90% | +0,05 pp |
| CPU média | 39,14% | 37,34% | -1,80 pp |
| Memória média | 85,45% | 81,15% | -4,30 pp |

O cálculo dos ganhos percentuais usa o cenário de hardware como base. O cenário combinado apresentou distribuições mais compactas para RPS, latência e tempo de resposta, indicando maior estabilidade.

### 5.3 Resultados locais do MASA-OpenMP

| Threads | Média (s) | Mediana (s) | Desvio padrão (s) |
|---:|---:|---:|---:|
| 1 | 0,036 | 0,040 | 0,0167 |
| 2 | 0,018 | 0,020 | 0,0045 |
| 4 | 0,088 | 0,040 | 0,1244 |

Tempos individuais:

| Threads | Repetições (s) |
|---:|---|
| 1 | 0,06; 0,04; 0,04; 0,02; 0,02 |
| 2 | 0,02; 0,01; 0,02; 0,02; 0,02 |
| 4 | 0,04; 0,31; 0,03; 0,02; 0,04 |

Com base nas medianas:

| Comparação | Speedup |
|---|---:|
| 2 threads / 1 thread | 2,00× |
| 4 threads / 1 thread | 1,00× |

O maior desvio em quatro threads indica sensibilidade ao processo do sistema e à sobrecarga da biblioteca. Uma repetição levou 0,31 s e foi mantida no conjunto, pois não houve justificativa para remover um dado válido.

### 5.4 Evidência publicada pela MASA

No artigo original, a MASA-OpenMP em CPU foi comparada com OmpSs, CUDA e Intel Phi em sequências de 10 Kbp a 47 Mbp. Para alinhamentos locais:

- GCUPS variaram de 0,57 a 4,81 no OpenMP/CPU;
- OmpSs/CPU chegou de 0,41 a 5,95 GCUPS;
- CUDAlign apresentou melhor desempenho em várias sequências maiores;
- OmpSs atingiu pruning de até 66,5%;
- Generic BP + dataflow aumentou GCUPS até 23,7% em relação ao OpenMP/CPU em casos favoráveis.

Esses resultados de escala foram obtidos em hardware histórico e não são diretamente comparáveis ao piloto atual.

---

## 6. Comparação Integrada

### 6.1 Desempenho

O AdaptScale tem vantagem aparente em carga variável porque o RPS mede diretamente o volume de trabalho e permite reação antes que a saturação de recursos de infraestrutura degrade o serviço. O KSE age sobre o posicionamento de carga já existente e é mais útil quando o desbalanceamento é persistente.

### 6.2 Estabilidade

O KSE-Refine e o cenário combinado do AdaptScale foram mais estáveis que seus equivalentes agressivos ou exclusivos de hardware. Isso sugere que respostas moderadas tendem a preservar qualidade de serviço.

### 6.3 Uso de recursos

No AdaptScale, métricas combinadas reduziram CPU e memória médias. No KSE, o custo não aparece como uso adicional médio, mas como número de migrações. Portanto, o número de migrações deve integrar o custo operacional de qualquer implantação.

### 6.4 Escalabilidade da aplicação MASA

A aplicação MASA demonstra que paralelismo só é eficaz quando o trabalho supera a sobrecarga. Em sequências pequenas, duas threads reduziram o tempo; quatro threads não reduziram por:

- pouca granularidade;
- overhead de OpenMP;
- sincronização entre blocos;
- limitações do processador local;
- variabilidade do ambiente compartilhado.

Para cargas maiores, a evidência do artigo mostra que dataflow e pruning podem escalar melhor que diagonalização simples.

---

## 7. Discussão

### 7.1 Diferenças fundamentais

O KSE e o AdaptScale não são alternativas diretas:

- o KSE decide **onde** executar os Pods;
- o AdaptScale decide **quantos** Pods executar.

Uma arquitetura completa pode combinar os dois níveis:

1. HPA/KEDA ajusta capacidade;
2. MAPE-K correlaciona sinais históricos e aplica thresholds;
3. KSE redistribui Pods que já existem quando surgem hotspots;
4. MASA fornece uma carga HPC controlada para validar concorrência e throughput.

### 7.2 Trade-offs

| Critério | Greedy | Refine | Hardware-only | Combinado |
|---|---|---|---|---|
| Resposta | Muito rápida | Moderada | Moderada | Mais contextualizada |
| Disponibilidade | Maior risco | Melhor preservação | Risco reativo | Mais estável |
| Uso de recursos | Pode redistribuir | Conserva migrações | Pode reagir tarde | Melhor alocação |
| Complexidade | Média | Média | Baixa | Alta |
| Observabilidade | CPU/memória | CPU/memória | CPU/rede | CPU/rede/RPS |

### 7.3 Interpretação da hipótese

- **H1 é parcialmente apoiada:** o AdaptScale teve latência e estabilidade melhores no cenário publicado.
- **H2 é parcialmente apoiada:** o KSE teve vantagens em carga exponencial, mas não em carga normal.
- **H3 é apoiada:** Greedy realizou mais migrações e reduziu mais a disponibilidade que Refine.
- **H4 é apoiada:** o piloto pequeno não apresentou ganho em quatro threads.

---

## 8. Custo

### 8.1 Custo metodológico do piloto MASA

O custo de execução local foi aproximadamente zero em infraestrutura em nuvem, mas inclui:

- 15 execuções;
- 10 segundos de tempo reservado no shell;
- cerca de 7,8 MB de memória máxima observada por processo;
- armazenamento temporário em `/tmp/masa-work`;
- tempo de preparação de aproximadamente 30 minutos.

### 8.2 Custo estimado para experimentos Kubernetes

| Item | Estimativa |
|---|---|
| Cluster local com 4 nós + 1 mestre | 0,00 por infraestrutura própria |
| 96 cenários do KSE × 10 minutos | 16 horas por rodada completa |
| Plataforma cloud média | Dependente da tarifa da região |
| Processamento MASA grande | CPU-hora = núcleos × tempo × custo por núcleo-hora |
| Migração KSE | Custo adicional por interrupção, reinicialização e transferência |

Uma fórmula prática é:

\[
C = C_{\text{infra}} + C_{\text{rede}} + C_{\text{energia}}
\]

para custo absoluto, e:

\[
C_{\text{execução}} = C / N
\]

para custo por execução.

---

## 9. Reprodutibilidade

### 9.1 Materiais

- PDFs dos dois artigos;
- PDF da arquitetura MASA;
- binário MASA-OpenMP local;
- códigos FASTA;
- arquivos de saída em `/tmp/masa-*`;
- ambiente Linux com GCC/OpenMP.

### 9.2 Procedimento do piloto

1. Criar os arquivos FASTA.
2. Limpar o diretório de trabalho.
3. Executar o MASA com 1, 2 e 4 threads.
4. Realizar cinco repetições por configuração.
5. Registrar tempo real, usuário, sistema e memória máxima.
6. Validar o score produzido.
7. Calcular média, mediana, desvio padrão e speedup.

### 9.3 Rastreabilidade

Os dados publicados foram extraídos dos PDFs disponíveis na pasta `artigos`. O piloto executável utilizou o binário previamente disponível no ambiente local. Nenhum resultado publicado foi reproduzido integralmente; ele foi utilizado como evidência secundária e o MASA foi executado como experimento local independente.

---

## 10. Ameaças à Validade

### Validade interna

- Os artigos usaram ambientes, cargas e thresholds diferentes.
- O piloto MASA não possui intervenção relacionada ao Kubernetes.
- O cenário de 4 threads teve uma variação extrema.
- Thresholds do AdaptScale foram definidos manualmente.
- O KSE pode beneficiar cargas exponenciais mais que cargas normais.

### Validade externa

- A faixa de hardware do MASA original é maior que o processador local.
- O Kubernetes pode se comportar de forma diferente em nuvens públicas.
- Os padrões de carga sintéticos não representam todas aplicações.
- O MASA é uma carga científica, não uma aplicação Kubernetes típica.

### Validade de construção

- EMA mede desbalanceamento, mas não qualidade funcional.
- RPS mede taxa observada, não obrigação garantida.
- Tempo mediano não revela todos os efeitos de tail latency.
- Percentual de sucesso pode esconder latências severas.
- Pruning, GCUPS e tempo incluem etapas diferentes do alinhamento.

### Validade estatística

- O piloto local contém somente cinco repetições por configuração.
- Não é suficiente para teste de significância robusto.
- Uma repetição foi mantida como outlier válido.
- Os artigos não apresentam sempre intervalos de confiança ou repetição completa.

---

## 11. Limitações e Trabalhos Futuros

As seguintes extensões devem fazer parte de uma próxima rodada:

1. Executar o MASA com entradas maiores, incluindo 10 Kbp e 1 Mbp.
2. Medir 1, 2, 4, 8 e todos os threads disponíveis.
3. Comparar OpenMP com OmpSs na mesma máquina.
4. Registrar GCUPS e porcentagem de blocos pruned.
5. Implantar HPA e KEDA em cluster local.
6. Executar cinco repetições independentes para todos os cenários.
7. Calcular intervalos de confiança de 95%.
8. Registrar custo por execução e consumo energético.
9. Combinar rebalanceamento KSE com autoscaling AdaptScale.

---

## 12. Conclusão

O KSE e o AdaptScale resolvem problemas complementares. O KSE corrige desbalanceamento espacial com migrações; o AdaptScale ajusta capacidade com métricas de infraestrutura e aplicação. Os resultados publicados indicam que ambos podem melhorar seu baseline, mas o ganho depende da heterogeneidade da carga e da agressividade da política.

O AdaptScale apresentou melhor estabilidade e uso de recursos no cenário com métricas combinadas. O KSE-Refine apresentou melhor trade-off de disponibilidade e migração do que o Greedy. O piloto MASA-OpenMP confirmou que paralelismo melhora a carga pequena somente até o limite em que a sobrecarga deixa de ser desprezível.

A combinação mais promissora é uma arquitetura híbrida:

1. KEDA/HPA dimensiona réplicas;
2. MAPE-K incorpora métricas históricas e eventos;
3. KSE redistribui hotspots sem destruir disponibilidade;
4. o MASA serve como carga real de avaliação de CPU e paralelismo.

O resultado local não substitui experimentos em Kubernetes em escala; ele demonstra o procedimento executável e estabelece a linha de base para a próxima rodada.

---

## Referências

1. NETO, P. M. C.; CASTRO, M.; SIQUEIRA, F. **Balanceamento de Carga Dinâmico em Ambientes Kubernetes com o Kubernetes Scheduling Extension (KSE)**. SBC, 2023.
2. GONÇALVES, A. L. de J.; FREITAS, L. A.; OLIVEIRA-JR, A. **Arquitetura de Dimensionamento Adaptativo com Suporte ao Aprendizado**. SBC, 2025.
3. SANDES, E. F. de O. et al. **MASA: a Multi-Platform Architecture for Sequence Aligners with Block Pruning**. ACM, 2016. DOI: 10.1145/2858656.
4. DOGANI, J.; NAMVAR, R.; KHUNJUSH, F. **Auto-scaling techniques in container-based cloud and edge/fog computing: Taxonomy and survey**. Computer Communications, v. 209, p. 120–150, 2023.
5. NGUYEN, T.-T. et al. **Horizontal Pod Autoscaling in Kubernetes for Elastic Container-Orchestration**. Sensors, v. 20, n. 16, 2020.
