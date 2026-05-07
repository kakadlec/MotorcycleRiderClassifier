# Fase 0 - Corpus E Ground Truth

Este documento detalha a Fase 0 do Motorcycle Rider Classifier. A meta desta fase e criar uma base pequena, realista e mensuravel antes de alterar qualquer comportamento do app desktop herdado do FrankSherlock.

## Objetivo

Ao final da Fase 0, deve ser possivel avaliar se um agrupamento sugerido esta certo ou errado usando dados reais de eventos de motocicleta.

Esta fase nao implementa pipeline, OCR, embeddings, clustering ou UI. Ela prepara os dados e os criterios que vao orientar os experimentos de dominio nas proximas fases.

## Estrutura Local Do Corpus

O corpus real deve ficar fora do Git, em uma pasta local ignorada:

```text
_local/
  rider_corpus/
    evt_YYYYMMDD_slug/
      photos/
        day_01/
        day_02/
      ground_truth/
        photos.csv
        riders.csv
        hard_cases.csv
      notes/
        event_notes.md
  rider_outputs/
```

Regras:

- `photos/` contem copias ou symlinks das imagens usadas na validacao.
- `ground_truth/` contem a anotacao manual local preenchida a partir dos templates em `docs/templates/`.
- `notes/` registra contexto do evento, camera, condicoes de pista, pontos conhecidos de erro e decisoes de anotacao.
- `_local/rider_outputs/` fica reservado para thumbnails, crops, embeddings, bancos temporarios, resultados de OCR e relatorios gerados em fases futuras.
- Scripts e experimentos devem tratar a origem das fotos como read-only.
- Nenhum script deve escrever arquivos derivados dentro da pasta original das fotos do usuario.

## Identificadores

### Eventos

Use `evt_YYYYMMDD_slug`.

Exemplos:

- `evt_20260506_interlagos_trackday`
- `evt_20260506_20260507_serra_multi_day`

Um evento pode ter varios dias. O `event_id` continua sendo o mesmo quando o numero do adesivo e unico para todo o evento.

### Fotos

Use `photo_id` estavel por evento:

```text
photo_<event_slug>_<nnnnn>
```

Exemplo:

```text
photo_interlagos_trackday_00042
```

O `relative_path` deve ser relativo a pasta `photos/` do evento. Evite depender de paths absolutos no ground truth.

### Pilotos

Use `rider_id` com escopo por evento:

```text
rider_<event_slug>_<nnn>
```

Exemplo:

```text
rider_interlagos_trackday_007
```

Regras:

- O `rider_id` nao deve ser derivado do numero do adesivo.
- O numero do adesivo fica no campo `sticker_number`, porque pode estar invisivel, parcialmente visivel ou errado.
- Use `unknown` quando a foto contem um piloto, mas a identidade ainda nao foi anotada.
- Use `non_rider` apenas para imagens negativas que forem mantidas no corpus, como paddock sem piloto em acao ou fotos sem motociclista util.

## Ground Truth

Os templates ficam em `docs/templates/` e devem ser copiados para a pasta local do evento antes do preenchimento.

### `photos.csv`

Uma linha por foto.

Campos:

- `event_id`: ID do evento.
- `photo_id`: ID estavel da foto.
- `relative_path`: path relativo a `photos/`.
- `rider_id`: ID manual do piloto, `unknown` ou `non_rider`.
- `sticker_number`: numero lido manualmente, vazio quando ausente ou ilegivel.
- `sticker_visibility`: `visible`, `partial`, `not_visible`, `illegible`, `unknown`.
- `view_angle`: `front`, `front_side`, `side`, `rear_side`, `rear`, `unknown`.
- `quality_flags`: lista separada por `|`, usando valores como `ok`, `motion_blur`, `occluded`, `low_resolution`, `overexposed`, `underexposed`, `distant`, `multiple_riders`.
- `split`: `train`, `validation`, `test` ou `holdout`.
- `notes`: observacoes curtas.

### `riders.csv`

Uma linha por piloto conhecido em um evento.

Campos:

- `event_id`: ID do evento.
- `rider_id`: ID manual do piloto.
- `sticker_number`: numero oficial ou melhor leitura manual, vazio se desconhecido.
- `visual_summary`: descricao curta para ajudar auditoria manual, como cor do capacete, roupa e moto.
- `notes`: ambiguidades conhecidas.

### `hard_cases.csv`

Uma linha por caso dificil relevante.

Campos:

- `event_id`: ID do evento.
- `photo_id`: foto relacionada.
- `case_type`: `similar_rider`, `similar_bike`, `hidden_sticker`, `illegible_sticker`, `motion_blur`, `occlusion`, `rear_view`, `side_view`, `multiple_riders`, `other`.
- `expected_challenge`: por que este caso e dificil.
- `review_priority`: `low`, `medium`, `high`.
- `notes`: observacoes adicionais.

## Tamanho Minimo Da Amostra

Meta inicial:

- pelo menos 20 pilotos, ou todos os pilotos disponiveis se o evento for menor;
- pelo menos 80 fotos no total;
- pelo menos 3 fotos por piloto quando possivel;
- pelo menos 10 fotos frontais com adesivo visivel;
- pelo menos 10 fotos laterais;
- pelo menos 10 fotos de costas ou sem adesivo visivel;
- pelo menos 10 fotos com baixa qualidade, oclusao, motion blur, distancia grande ou exposicao ruim;
- pelo menos 3 pares ou grupos de pilotos/motos visualmente parecidos.

Minimo aceitavel para encerrar a Fase 0 quando o evento for pequeno:

- 50 fotos;
- 10 pilotos, ou todos os pilotos disponiveis;
- cobertura explicita de adesivo visivel, lateral, costas, adesivo invisivel/ilegivel e pilotos/motos parecidos.

## Metricas Iniciais

As metricas abaixo sao os primeiros criterios para os experimentos de Fase 1. Elas nao exigem implementacao nesta fase, mas precisam estar definidas para evitar avaliacao subjetiva.

### Clustering

- Pureza de cluster: para cada cluster sugerido, fracao de fotos que pertencem ao `rider_id` majoritario.
- Recall por piloto: para cada piloto, fracao das fotos do piloto recuperadas no melhor cluster associado a ele.
- Taxa de revisao manual: percentual de fotos ou clusters que precisam de decisao humana antes de serem aceitos.

### Adesivo

- Precisao do adesivo: leituras corretas entre as leituras retornadas.
- Recall do adesivo visivel: fotos com adesivo visivel em que o numero correto foi recuperado.
- Taxa correta de sem leitura: fotos com adesivo invisivel ou ilegivel em que o sistema nao inventou um numero.

### Retrieval Visual

- Top-k por piloto: percentual de queries cuja galeria top-k contem ao menos uma foto do mesmo `rider_id`.
- Separacao de similares: desempenho em casos registrados como `similar_rider` ou `similar_bike`.

## Criterio Inicial De Aceite Do MVP

O primeiro MVP deve mirar, no corpus de validacao inicial:

- pureza de cluster de pelo menos 0.90 nos clusters aceitos automaticamente;
- recall por piloto de pelo menos 0.70;
- taxa de revisao manual de no maximo 35%;
- nenhuma dependencia de API em nuvem;
- nenhum processamento que escreva em pastas de fotos do usuario.

Esses valores sao pontos de partida. Eles podem mudar depois da Fase 1, desde que a decisao seja baseada nos resultados registrados.

## Arquivos No Git

Devem entrar no Git:

- documentacao da Fase 0;
- templates CSV vazios ou com linhas de exemplo sinteticas;
- scripts futuros de validacao, quando existirem;
- configuracoes de ignore que protegem dados locais.

Nao devem entrar no Git:

- fotos reais do evento;
- copias ou symlinks para corpus real;
- thumbnails, crops e imagens derivadas;
- embeddings e vetores;
- bancos SQLite locais;
- caches de modelo;
- resultados gerados por OCR, deteccao, classificacao ou clustering;
- logs pesados;
- dados pessoais ou sensiveis do evento.

## Checklist De Conclusao

- [ ] Corpus local criado em `_local/rider_corpus/<event_id>/`.
- [ ] Fotos reais copiadas ou linkadas em `photos/`, sem modificar a origem original.
- [ ] `photos.csv` preenchido para a amostra minima.
- [ ] `riders.csv` preenchido para pilotos conhecidos.
- [ ] `hard_cases.csv` registra casos de lado, costas, adesivo invisivel, blur e pilotos/motos parecidos.
- [ ] Corpus contem fotos frontais com adesivo visivel.
- [ ] Corpus contem fotos de lado.
- [ ] Corpus contem fotos de costas.
- [ ] Corpus contem fotos com adesivo ilegivel ou invisivel.
- [ ] Corpus contem pilotos ou motos visualmente parecidos.
- [ ] Ground truth identifica pelo menos 20 pilotos, ou o maximo disponivel no evento.
- [ ] Metricas iniciais e criterio de aceite do MVP estao escritos neste documento.
- [ ] `.gitignore` protege corpus local, outputs, bancos, embeddings e imagens derivadas.
- [ ] Nenhuma alteracao funcional foi feita em `sherlock/desktop/`.

## Criterio De Saida

A Fase 0 esta pronta quando qualquer experimento futuro conseguir carregar o corpus local e responder, com base no ground truth manual, se um agrupamento, uma leitura de adesivo ou uma recuperacao visual esta correta.
