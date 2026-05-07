# Fase 0 - Notas De Execucao

Status: concluida para iniciar a PoC de dominio.

Estas notas registram a execucao inicial da Fase 0 depois da aprovacao da spec em `docs/phase-0-corpus-ground-truth.md`.

## O Que Foi Feito

- Criado um corpus local em `_local/rider_corpus/evt_20260420_experience/`.
- Criados ground truths locais em `_local/rider_corpus/evt_20260420_experience/ground_truth/`.
- Revisadas 10 fotos reais do evento.
- Anotado o piloto `rider_22`, com adesivo `22`.
- Revisados `photos.csv`, `riders.csv` e `hard_cases.csv` locais.
- Registrados casos dificeis reais, incluindo foto lateral sem adesivo visivel, poeira, baixa luz, distancia e multiplos pilotos na cena.
- Confirmado que os dados reais do corpus continuam fora do Git.

## Escopo Versionado

Os arquivos reais do corpus e seus ground truths preenchidos permanecem em `_local/` e nao entram no Git.

Este documento versiona apenas o resultado operacional da fase, sem fotos reais, sem metadados sensiveis de imagem e sem saidas geradas.

## Ressalva Principal

O corpus atual valida o processo de preparacao, leitura de ground truth e repeticao dos experimentos, mas ainda nao valida a eficacia do produto.

Limites atuais:

- O corpus contem apenas um piloto anotado.
- Ainda nao ha base para medir clustering entre pilotos diferentes.
- Metricas como pureza de cluster, recall por piloto e taxa de revisao manual devem ser tratadas como definidas, mas ainda nao conclusivas.

Quando houver mais fotos e mais pilotos, os mesmos artefatos da Fase 0 devem ser ampliados e os experimentos devem ser executados novamente.

## Decisao

A Fase 0 esta suficientemente completa para iniciar a PoC em `_rider_classification/`.

A proxima fase deve validar o fluxo experimental contra o corpus atual, deixando claro que o objetivo inicial e testar o processo, nao provar a eficacia do modelo.
