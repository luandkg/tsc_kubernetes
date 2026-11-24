# Aplicação: MASA-OpenMP

O MASA é uma arquitetura portável para alinhamento exato de sequências biológicas, com suporte a alinhamentos locais, globais e semiglobal. As variantes mais relevantes para o projeto são:

- **MASA-OpenMP/CPU**: antidiagonais de blocos processados em paralelo por threads OpenMP.
- **MASA-OmpSs/CPU**: execução assíncrona orientada por dependências.

Neste projeto, a carga de teste é **MASA-OpenMP 1.0.1.1024**. O binário local está disponível em:

```text
/home/luan/PPGI_UNB/repos/MASA-OpenMP/releases/masa-openmp-1.0.1.1024/masa-openmp
```

# Repositório

https://github.com/edanssandes/MASA-OpenMP

## Comando-base

```bash
OMP_NUM_THREADS=1 ./masa-openmp \
  -c -v 1 --block-size=128,128 \
  --work-dir=/tmp/masa-work \
  /tmp/seq1.fa /tmp/seq2.fa
```

Repita o comando com `OMP_NUM_THREADS=2` e `OMP_NUM_THREADS=4`. Execute pelo menos cinco vezes por configuração e meça com `/usr/bin/time`.

## Artefatos

- Sequências de entrada em formato FASTA;
- saída textual com score e crosspoints;
- arquivo de tempo por repetição;
- resultados agregados em tabela no documento final.
