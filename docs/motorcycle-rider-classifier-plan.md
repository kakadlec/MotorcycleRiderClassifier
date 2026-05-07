# Motorcycle Rider Classifier - Plano De Produto E Implementação

## Decisão De Planejamento

Este documento deve ser tratado como um **guia macro por fases**, não como uma especificação final fechada.

A recomendação é:

1. Usar este plano para orientar escopo, sequência e critérios de aceite.
2. Antes de iniciar cada fase, gerar um plano detalhado daquela fase com tarefas, schema, comandos Tauri, UI e testes.
3. Validar e aprovar esse plano detalhado antes de executar alterações funcionais ou criar artefatos da fase.
4. Encerrar cada fase com um artefato executável e validável.
5. Só detalhar a fase seguinte depois de observar os resultados reais da fase atual.

Motivo: o problema depende de evidência visual e dados reais. A estratégia ideal para adesivo, OCR, embeddings e clustering vai mudar conforme a qualidade das fotos, variação de ângulos, resolução, iluminação, capacetes, roupas e motos. Um plano grande demais no início tende a cristalizar decisões ruins.

Para fases mais complexas, o fluxo esperado é sempre: planejar a fase em detalhe, revisar o plano, receber OK explícito e só então executar. A Fase 0 é a preparação desse processo; as fases seguintes não devem começar diretamente pela implementação.

## Objetivo

Criar um aplicativo desktop local-first para classificar fotos de motociclistas e agrupar imagens que pertencem ao mesmo motociclista, usando uma combinação de:

- leitura do número em adesivo na windscreen, quando visível;
- similaridade visual do piloto;
- similaridade visual da moto;
- revisão manual para corrigir grupos ambíguos;
- aprendizado incremental a partir das correções do usuário.

O FrankSherlock será usado como base técnica inicial, mas o novo produto deve ter domínio, schema, nomes, UI e pipeline próprios.

## Princípios

- Não criar um FrankSherlock com features acumuladas.
- Reaproveitar infraestrutura, não conceitos de domínio genéricos.
- Começar com schema novo, sem carregar migrations históricas do FrankSherlock.
- Manter o app local-only e read-only sobre as pastas de origem.
- Priorizar pipeline observável e testável sobre automação completa prematura.
- Não alterar o app desktop antes de validar os experimentos de domínio com dados reais.
- Tratar o adesivo como sinal forte, mas não como verdade única.
- Manter revisão manual como parte central do produto desde cedo.

## Estratégia Técnica

### Reaproveitar Do FrankSherlock

- Tauri + React + Vite.
- Scanner incremental de diretórios.
- SQLite local.
- Cache de thumbnails.
- Jobs canceláveis e retomáveis.
- Grid, preview, seleção múltipla e status de progresso.
- Organização básica de comandos Tauri.
- Detecção de ambiente local e diretórios de app data.
- Test setup de Rust e frontend.

### Remover Ou Substituir Cedo

- Classificação genérica por `media_type`.
- Busca textual como centro do produto.
- Albums e smart folders, exceto se forem reaproveitados depois como coleções.
- PDF, vídeo e OCR genérico.
- Face detection.
- Prompts e categorias de anime/documentos.
- Nomes e paths ligados a `FrankSherlock` ou `sherlock`.

### Novo Domínio

Entidades principais propostas:

- `photos`: foto original indexada.
- `photo_assets`: thumbnails, crops e imagens derivadas.
- `rider_detections`: regiões contendo motociclista/moto.
- `sticker_reads`: leituras candidatas do número do adesivo.
- `visual_embeddings`: vetores extraídos de piloto, moto, crop combinado ou foto.
- `rider_clusters`: grupos sugeridos pelo sistema.
- `cluster_members`: associação foto-cluster com score e razão.
- `manual_labels`: correções do usuário, merges, splits e rejeições.
- `evaluation_sets`: conjuntos de validação manual.

## Pipeline Alvo

1. Scan incremental encontra imagens e cria registros mínimos.
2. Thumbnailing gera visual rápido para UI.
3. Detector localiza moto/piloto ou crop principal.
4. OCR tenta ler número do adesivo quando a windscreen está visível.
5. Extratores visuais geram embeddings para piloto, moto e crop combinado.
6. Clustering combina número, embeddings e metadados de confiança.
7. UI apresenta grupos sugeridos.
8. Usuário corrige grupos.
9. Correções viram restrições para as próximas rodadas de clustering.

## Fases

### Fase 0 - Preparação E Corpus De Validação

Objetivo: definir o problema com dados reais antes de alterar muito código.

Escopo:

- Separar um corpus pequeno e representativo de fotos.
- Criar ground truth manual inicial.
- Definir métricas mínimas de sucesso.
- Confirmar formatos de imagem, volume esperado e ambiente alvo.

Entregáveis:

- Pasta local de teste, não versionada, com amostra representativa.
- Arquivo JSON ou CSV de ground truth.
- Lista de casos difíceis: lado, costas, adesivo invisível, motion blur, grupos parecidos.
- Métrica inicial: pureza dos clusters, recall por piloto e taxa de revisão manual.

Checklist:

- [ ] Corpus contém fotos frontais com adesivo visível.
- [ ] Corpus contém fotos de lado.
- [ ] Corpus contém fotos de costas.
- [ ] Corpus contém fotos com adesivo ilegível.
- [ ] Corpus contém pilotos/motos visualmente parecidos.
- [ ] Ground truth identifica pelo menos 20 pilotos ou o mínimo disponível.
- [ ] Métrica de aceite do MVP está escrita.

Critério de saída:

- Conseguimos dizer se um agrupamento sugerido está certo ou errado usando dados reais.

### Fase 1 - Experimentos De Viabilidade Do Domínio

Objetivo: replicar a abordagem experimental do FrankSherlock para o cenário de motociclistas antes de alterar o app desktop.

Escopo:

- Criar laboratórios independentes, fora do app, inspirados em `_classification`, `_research_ab_test` e `_face_ab_test`.
- Testar classificação básica da foto, leitura do adesivo, embeddings visuais, crops e clustering.
- Medir resultados contra o ground truth da Fase 0.
- Decidir quais técnicas entram no MVP do app.

Entregáveis:

- `_rider_classification/`: PoC rápido para classificar fotos de evento.
- `_rider_research_ab_test/`: benchmark principal de adesivo, OCR, modelos visuais, embeddings e similaridade.
- `_rider_ab_test/`: benchmark específico de detecção/crop, embeddings e clustering de motociclistas.
- `docs/RESULTS.md` em cada laboratório com achados, métricas e decisão recomendada.
- Lista objetiva do que será implementado no app e do que será descartado.

Checklist:

- [ ] Nenhuma alteração funcional foi feita em `sherlock/desktop` antes dos experimentos.
- [ ] `_rider_classification` identifica presença de motociclista, moto, adesivo visível, ângulo e qualidade básica.
- [ ] `_rider_research_ab_test` compara OCR genérico, modelo vision-language e eventuais heurísticas para ler o adesivo.
- [ ] `_rider_research_ab_test` compara embedding de imagem inteira contra crop central ou crop piloto+moto.
- [ ] `_rider_ab_test` compara estratégias de detecção/crop de motociclista+moto.
- [ ] `_rider_ab_test` compara clustering por threshold, DBSCAN/HDBSCAN ou estratégia equivalente.
- [ ] Todos os experimentos rodam contra o mesmo corpus e ground truth.
- [ ] Resultados registram precision/recall do adesivo, top-k retrieval, pureza de cluster e recall por piloto.
- [ ] Existe recomendação clara para o pipeline inicial do app.

Critério de saída:

- Sabemos, com dados reais, qual estratégia inicial será usada para adesivo, embeddings, crop e clustering antes de mexer no app.

### Fase 2 - Bootstrap Do Novo Produto

Objetivo: criar o novo app a partir do FrankSherlock sem carregar domínio antigo.

Escopo:

- Criar um fork do FrankSherlock, preservando histórico, autoria e licença.
- Manter o projeto derivado compatível com `GPL-3.0-only`.
- Renomear package, Cargo crate, app id, título, paths e diretórios de cache.
- Remover funcionalidades fora do domínio.
- Manter o app abrindo com shell desktop, scanner básico e grid.

Entregáveis:

- Fork criado a partir do repositório original.
- Novo app com nome próprio.
- `LICENSE` preservado.
- README com atribuição clara ao FrankSherlock original.
- Build de dev funcionando.
- Banco novo criado em path próprio.
- README mínimo do novo produto explicando que este é um projeto derivado.

Checklist:

- [ ] Fork preserva histórico Git do FrankSherlock.
- [ ] `LICENSE` GPL-3.0-only foi mantido.
- [ ] README informa que o projeto é derivado do FrankSherlock.
- [ ] README atribui o projeto original ao autor/repositório original.
- [ ] README explica que o novo produto tem domínio próprio de organização de fotos de motociclistas.
- [ ] Não houve recriação do Git para apagar proveniência.
- [ ] Nome `FrankSherlock` removido de UI, package, Cargo e app data.
- [ ] Banco novo não reutiliza migrations históricas.
- [ ] App inicia sem depender de Ollama.
- [ ] Grid vazio renderiza corretamente.
- [ ] Scanner básico indexa imagens.
- [ ] Thumbnails aparecem.
- [ ] Testes existentes irrelevantes foram removidos ou substituídos.

Critério de saída:

- O novo app existe como produto independente, com proveniência, licença e atribuição corretas, e não parece uma versão parcial do FrankSherlock.

### Fase 3 - Modelo De Dados Limpo E Scanner MVP

Objetivo: estabilizar a fundação de dados para fotos e assets derivados.

Escopo:

- Criar schema novo para `photos`, `photo_assets` e `scan_jobs`.
- Adaptar scanner incremental para o domínio novo.
- Manter read-only nas pastas de origem.
- Preservar cancelamento e retomada.

Entregáveis:

- Migrations novas e pequenas.
- Scanner com fases: discovery, thumbnailing, cleanup.
- Página de status de scan.
- Testes unitários de schema e scanner incremental.

Checklist:

- [ ] `photos` registra root, path relativo, path absoluto, fingerprint, mtime, tamanho e status.
- [ ] `photo_assets` registra thumbnail e futuros crops.
- [ ] Arquivos movidos são detectados por fingerprint.
- [ ] Arquivos removidos são marcados como ausentes ou deletados logicamente.
- [ ] Scan cancelado pode ser retomado.
- [ ] Testes cobrem novo, modificado, movido, inalterado e removido.

Critério de saída:

- É possível escanear uma pasta real, fechar o app, reabrir e continuar sem reprocessar tudo.

### Fase 4 - UI Base De Curadoria

Objetivo: transformar o grid em uma ferramenta de inspeção e rotulagem.

Escopo:

- Grid de fotos com preview.
- Seleção múltipla.
- Painel lateral com metadados.
- Ações manuais mínimas: marcar mesmo piloto, marcar diferente, criar grupo.
- Persistir labels manuais.

Entregáveis:

- Tela principal orientada a fotos de evento.
- Tabelas `rider_clusters`, `cluster_members` e `manual_labels`.
- Primeira experiência de curadoria manual.

Checklist:

- [ ] Usuário cria um grupo manualmente.
- [ ] Usuário adiciona fotos a um grupo.
- [ ] Usuário remove foto de um grupo.
- [ ] Usuário marca duas fotos como "não são o mesmo piloto".
- [ ] Labels sobrevivem a restart do app.
- [ ] UI deixa claro quais agrupamentos são manuais e quais são automáticos.

Critério de saída:

- Mesmo sem IA, o app já serve para organizar um pequeno conjunto de fotos manualmente.

### Fase 5 - Leitura Do Adesivo Como Sinal Forte

Objetivo: extrair número do adesivo quando visível e medir se esse sinal é confiável.

Escopo:

- Criar pipeline `sticker_read`.
- Implementar a estratégia escolhida nos experimentos da Fase 1.
- Começar simples no app, mesmo que os experimentos indiquem um caminho mais sofisticado para fases futuras.
- Persistir múltiplas leituras candidatas por foto com confiança.
- UI para revisar e corrigir número lido.

Entregáveis:

- Tabela `sticker_reads`.
- Job de processamento de adesivo.
- Filtro por número detectado.
- Relatório simples de acurácia no corpus.

Checklist:

- [ ] Fotos com adesivo frontal recebem leitura candidata.
- [ ] Fotos sem adesivo visível podem ficar sem leitura.
- [ ] Usuário corrige leitura errada.
- [ ] Leitura corrigida é usada como verdade manual.
- [ ] Pipeline registra confiança e método.
- [ ] Avaliação mede precision e recall do número.

Critério de saída:

- Sabemos empiricamente se o adesivo é bom o suficiente para ancorar clusters.

### Fase 6 - Embeddings Visuais Baseline

Objetivo: criar similaridade visual independente do adesivo.

Escopo:

- Gerar embeddings para a foto inteira ou crop central.
- Implementar o modelo recomendado pelos experimentos da Fase 1, usando a mesma filosofia do FrankSherlock: detectar hardware local e recomendar o modelo mais adequado.
- Persistir vetores ou arquivos de vetores.
- Criar busca "fotos parecidas com esta".

Entregáveis:

- Tabela ou storage de `visual_embeddings`.
- Comando para processar embeddings.
- Tela ou ação "similar photos".
- Avaliação top-k no corpus.

Checklist:

- [ ] Cada foto processada tem embedding.
- [ ] Seleção de modelo considera GPU, VRAM, memória unificada e fallback CPU.
- [ ] Reprocessamento evita duplicar embeddings.
- [ ] Busca por similaridade retorna resultados estáveis.
- [ ] Avaliação top-5/top-10 mede quantas fotos do mesmo piloto aparecem.
- [ ] Correções manuais são excluídas de pares proibidos quando aplicável.

Critério de saída:

- Dada uma foto sem adesivo, o app consegue sugerir candidatos plausíveis do mesmo motociclista.

### Fase 7 - Detecção Ou Crop De Piloto/Moto

Objetivo: reduzir interferência de cenário e fundo.

Escopo:

- Detectar ou estimar região principal contendo piloto + moto.
- Implementar a estratégia escolhida nos experimentos da Fase 1.
- Gerar crops derivados.
- Comparar embedding da foto inteira contra embedding do crop.
- Manter fallback para fotos onde detector falha.

Entregáveis:

- `rider_detections`.
- Crops em cache.
- Embeddings separados por tipo: full image, rider crop, motorcycle crop ou combined crop.
- Relatório comparativo.

Checklist:

- [ ] Crop é gerado para maioria das fotos úteis.
- [ ] Falha de crop não quebra processamento.
- [ ] UI permite visualizar crop usado.
- [ ] Similaridade por crop supera ou empata foto inteira no corpus.
- [ ] Fundo/cenário perde influência perceptível nos resultados.

Critério de saída:

- Existe uma representação visual mais específica para "motociclista + moto" do que a imagem inteira.

### Fase 8 - Clustering Automático V1

Objetivo: agrupar fotos automaticamente combinando adesivo e similaridade visual.

Escopo:

- Criar algoritmo inicial de cluster.
- Começar pelo algoritmo recomendado pelos experimentos da Fase 1.
- Combinar sinais com pesos configuráveis.
- Respeitar labels manuais como restrições.
- Gerar explicação simples por associação: número, visual, manual ou misto.

Entregáveis:

- Job `cluster_riders`.
- Scores por membro.
- Tela de grupos sugeridos.
- Configuração inicial de thresholds.

Checklist:

- [ ] Fotos com mesmo número confiável agrupam com alta prioridade.
- [ ] Fotos sem número entram por similaridade visual.
- [ ] Pares marcados como diferentes não ficam no mesmo cluster.
- [ ] Grupos manuais não são destruídos sem consentimento.
- [ ] Cada associação mostra score e razão.
- [ ] Avaliação mede pureza, recall e quantidade de revisão necessária.

Critério de saída:

- O app gera grupos úteis o bastante para acelerar a revisão manual.

### Fase 9 - Revisão De Clusters

Objetivo: tornar o trabalho humano rápido e confiável.

Escopo:

- Tela de cluster com representante, membros e candidatos rejeitados.
- Ações de merge e split.
- Atalhos de teclado para aceitar/rejeitar.
- Fila de grupos de baixa confiança.

Entregáveis:

- Workflow completo de revisão.
- Persistência de decisões.
- Histórico simples de alterações.

Checklist:

- [ ] Usuário aceita cluster inteiro.
- [ ] Usuário remove item incorreto.
- [ ] Usuário divide cluster misturado.
- [ ] Usuário junta dois clusters.
- [ ] App prioriza clusters ambíguos.
- [ ] Decisões alteram próxima execução do clustering.

Critério de saída:

- Um lote de fotos pode ser revisado de ponta a ponta dentro do app.

### Fase 10 - Avaliação E Ajuste De Pesos

Objetivo: transformar o pipeline em algo mensurável e ajustável.

Escopo:

- Criar script ou comando de avaliação contra ground truth.
- Medir contribuição do adesivo, embedding full image, crop e labels manuais.
- Ajustar thresholds.
- Registrar resultados por versão do pipeline.

Entregáveis:

- Relatório versionado de benchmark.
- Configuração de pesos.
- Dataset de regressão local.

Checklist:

- [ ] Avaliação roda com um comando.
- [ ] Métricas são salvas em arquivo.
- [ ] Mudanças de modelo/peso podem ser comparadas.
- [ ] Existe baseline manual ou semi-manual.
- [ ] Regressões são visíveis antes de trocar pipeline.

Critério de saída:

- Decisões sobre modelo e algoritmo são baseadas em números, não impressão visual isolada.

### Fase 11 - MVP Operacional

Objetivo: empacotar uma primeira versão utilizável em evento real ou lote grande.

Escopo:

- Fluxo completo: scan, processar, agrupar, revisar e organizar por piloto.
- Definir se a organização final será apenas interna no app ou também exportada para pastas, links, CSV ou JSON.
- Melhorar mensagens de erro e retomada.
- Build desktop reproduzível.

Entregáveis:

- Build local do app.
- Organização revisada por piloto dentro do app.
- Decisão registrada sobre exportação física, caso ela seja necessária.
- Guia de uso curto.
- Checklist operacional para processar um evento.

Checklist:

- [ ] Usuário seleciona uma pasta e inicia processamento.
- [ ] App mostra progresso de scan, OCR, embeddings e clustering.
- [ ] App permite revisar grupos.
- [ ] App organiza fotos por piloto.
- [ ] Se houver exportação física, o formato foi escolhido e validado.
- [ ] Falhas individuais não derrubam o lote inteiro.
- [ ] Build roda em ambiente alvo.

Critério de saída:

- O app consegue processar um lote real e entregar grupos revisados organizados por piloto.

## Checklists De Avanço Entre Fases

Antes de iniciar qualquer fase:

- [ ] O critério de saída da fase anterior foi atingido.
- [ ] Os testes da fase anterior passam.
- [ ] O que será removido, mantido e criado está escrito.
- [ ] Existe uma lista curta de tarefas para a fase.
- [ ] O schema impactado foi desenhado antes de codificar.
- [ ] A UI esperada tem pelo menos um wireframe textual.

Antes de encerrar qualquer fase:

- [ ] O app abre.
- [ ] O fluxo principal da fase funciona com dados reais.
- [ ] Existe pelo menos um teste automatizado para a lógica nova.
- [ ] Decisões técnicas relevantes foram registradas.
- [ ] Dívidas intencionais foram anotadas.
- [ ] A próxima fase foi reavaliada com base nos resultados obtidos.

## Métricas Recomendadas

- Sticker precision: leituras corretas entre leituras feitas.
- Sticker recall: fotos com adesivo visível que receberam leitura.
- Top-k retrieval: para uma foto, quantos candidatos do mesmo piloto aparecem nos primeiros 5, 10 e 20.
- Cluster purity: quanto de cada cluster pertence ao mesmo piloto.
- Cluster recall: quanto das fotos de um piloto foram reunidas.
- Manual review rate: porcentagem de fotos que exigem correção humana.
- Time to review: tempo para revisar um lote.

## Decisões Registradas

- Modelo local para embeddings visuais: manter a filosofia do FrankSherlock, detectando a placa de vídeo, VRAM, memória disponível e ambiente para recomendar o modelo mais adequado.
- Mudança de capacete, roupa ou moto entre sessões: fora do escopo inicial. Dentro de um evento, esses elementos não mudam.
- Número do adesivo: único por evento, e um evento pode durar vários dias.
- Resultado final esperado: organizar as fotos por piloto.

## Decisões Ainda Abertas

- Se a detecção de piloto/moto precisa de modelo dedicado ou se crop heurístico basta para o MVP. Isso precisa ser testado na Fase 1 com o corpus real.
- Se OCR do adesivo será feito por OCR genérico, modelo visual ou detector treinado. Isso também precisa ser testado na Fase 1.
- Se "organizar por piloto" será apenas uma organização interna no app ou se também haverá exportação física para pastas, links, CSV ou JSON.

## Primeiro Próximo Passo

Gerar um plano detalhado apenas da **Fase 0 - Preparação E Corpus De Validação**.

Esse plano deve definir:

- formato do ground truth;
- tamanho mínimo do corpus;
- convenção de IDs de piloto;
- métricas iniciais;
- estrutura de diretórios local;
- comandos ou scripts de validação;
- como armazenar exemplos difíceis.

Depois da Fase 0, gerar o plano detalhado da **Fase 1 - Experimentos De Viabilidade Do Domínio**, incluindo a estrutura dos três laboratórios:

- `_rider_classification/`;
- `_rider_research_ab_test/`;
- `_rider_ab_test/`.

Somente depois da Fase 1, gerar o plano detalhado da **Fase 2 - Bootstrap Do Novo Produto**, com a lista precisa do que remover do FrankSherlock e o novo esqueleto do produto.
