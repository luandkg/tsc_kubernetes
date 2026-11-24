# Metodologia Experimental

## 1. Estratégia geral

O projeto utiliza quatro etapas:

1. Revisão comparativa dos dois frameworks.
2. Definição de cenários e hipóteses verificáveis.
3. Experimentação reproduzível com MASA-OpenMP.
4. Análise quantitativa e discussão das ameaças à validade.

A comparação principal é qualitativa e quantitativa a partir dos dados publicados nos artigos. Os testes locais complementares usam o **MASA-OpenMP 1.0.1.1024**, uma aplicação real de alinhamento de sequências com blocos e paralelismo OpenMP.

## 2. Variáveis experimentais

| Tipo | Variável |
|---|---|
| Independente | Framework/mecanismo, número de réplicas ou nós, perfil de carga e número de threads |
| Dependente | Tempo de execução, throughput, latência, taxa de sucesso, desbalanceamento, uso de CPU/memória e migrações |
| Controlada | Sistema operacional, compilador/OpenMP, tamanho das entradas e penalidades de alinhamento |

## 3. Cenários

| Cenário | Condição | Finalidade |
|---|---|---|
| Baseline | Kubernetes padrão ou uma thread MASA | Referência de comparação |
| KSE-Greedy | Rebalanceamento agressivo por maior carga versus menor carga | Avaliar redução de desbalanceamento |
| KSE-Refine | Ajuste fino com menor número de migrações | Avaliar disponibilidade e custo de migração |
| AdaptScale-Hardware | CPU e transmissão de dados | Aproximar o comportamento reativo do HPA |
| AdaptScale-Combinado | CPU/transmissão mais taxa de requisições | Avaliar sinais de aplicação |
| MASA-OpenMP | 1, 2 e 4 threads | Avaliar escalabilidade local |

## 4. Execução dos experimentos

Cada cenário completo deve ser executado pelo menos cinco vezes. Quando o custo impedir esse total, o piloto deve declarar explicitamente o número reduzido e evitar inferências estatísticas fortes.

O piloto local registrado neste projeto utiliza:

- Duas sequências FASTA com aproximadamente 2.000 caracteres;
- alinhamento local com parâmetros SW `match=1`, `mismatch=-3`, `gap open=-3` e `gap extension=-2`;
- blocos de `128 × 128` células;
- `OMP_NUM_THREADS` igual a 1, 2 e 4;
- cinco execuções independentes por configuração.

## 5. Métricas

- Tempo de parede em segundos;
- média, mediana e desvio padrão;
- speedup relativo a uma thread;
- GCUPS, para comparações compatíveis com os experimentos publicados pela MASA;
- taxa de sucesso ou disponibilidade;
- latência e tempo de resposta;
- uso de CPU e memória;
- desbalanceamento médio;
- número de migrações ou mudanças de configuração;
- custo por execução.

## 6. Análise

Os resultados devem combinar estatística descritiva, gráficos de dispersão ou boxplots e intervalos de confiança quando houver amostra suficiente. Na comparação entre frameworks publicados, não se combinam diretamente valores absolutos de ambientes diferentes; a análise considera direção do efeito, magnitude relativa e limitações metodológicas.

## 7. Reprodutibilidade

O experimento deve informar versões, hardware, comandos, entradas, semente aleatória quando aplicável e procedimentos de coleta. Os arquivos intermediários não devem ser considerados evidência sem o respectivo método de geração.

