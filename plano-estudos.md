# Plano de estudos: Projeto Final, base de conhecimento da Quantum Commerce

> Como usar este arquivo: ele foi escrito para ser lido por você **e** por uma IA que vai te ajudar a construir o notebook etapa por etapa. Cada etapa tem objetivo, o que construir, o que observar, o que registrar e um prompt pronto para colar na IA. Antes de cada etapa, entregue à IA este arquivo inteiro como contexto e diga em qual etapa você está.

---

## Sumário

0. [Contexto do trabalho (resumo do enunciado)](#0-contexto-do-trabalho)
1. [O que a exploração dos dados já revelou](#1-o-que-a-exploração-dos-dados-já-revelou)
2. [Catálogo de técnicas, da mais fácil à mais difícil](#2-catálogo-de-técnicas-da-mais-fácil-à-mais-difícil)
3. [Encadear ou isolar as técnicas?](#3-encadear-ou-isolar-as-técnicas)
4. [Trilhas de configuração propostas (escolha aqui)](#4-trilhas-de-configuração-propostas)
5. [Etapas do trabalho](#5-etapas-do-trabalho)
6. [Estrutura do novo notebook](#6-estrutura-do-novo-notebook)
7. [Código-base com Gemini](#7-código-base-com-gemini)
8. [Modelos de registro e de análise](#8-modelos-de-registro-e-de-análise)
9. [Armadilhas e cuidados](#9-armadilhas-e-cuidados)

---

## 0. Contexto do trabalho

### O cenário
A Quantum Commerce usa um assistente com RAG para dar a primeira resposta a reclamações (entrega, divergente/defeito, fraude do vendedor, falsificado, item proibido, devolução/ressarcimento). Em produção ele:

- usa versões antigas de políticas;
- mistura regras do vendedor com regras do cliente;
- inventa regras que não estão nos documentos;
- responde mesmo quando a base não cobre o caso.

**Nosso papel:** melhorar a **base de conhecimento** e a **recuperação**.

### Os quatro objetivos
| # | Objetivo | Técnica que mais ataca |
|---|---|---|
| O1 | usar a política **vigente na data do chamado** | filtro de vigência por metadado |
| O2 | recuperar o **prazo e o valor corretos** | chunking por estrutura (tabelas), busca híbrida, re-ranking, metadados do chamado na consulta |
| O3 | usar **só informação aplicável ao cliente** | curadoria por público |
| O4 | **reconhecer quando a base não tem informação** | prompt de geração, limiar de abstenção, checagem de suficiência |

### O que precisa ser feito
1. **Linha de base:** chunks de tamanho fixo sobre os 44 documentos, busca por similaridade, geração com os trechos. Ignora vigência.
2. **Pelo menos 3 configurações de melhoria**, **uma mudança por vez**. A equipe escolhe quais e em que ordem.
3. **Avaliação controlada:** para cada configuração, quantas das 62 perguntas de referência têm a `resposta_referencia` dentro dos trechos recuperados (com `data_referencia` como data da vigência).
4. **RAGAS** nas configurações candidatas: fidelidade (faithfulness), precisão do contexto (context precision) e relevância da resposta (answer relevancy), numa amostra, mesmo modelo, temperatura 0.
5. **Rodada final nos 200 chamados** com a configuração escolhida, avaliada por um juiz LLM em 4 critérios: desfecho, parâmetro (prazo/valor), inventou, abstenção.

### Regras de comparação (obrigatórias)
- Medir a linha de base **antes** de qualquer alteração.
- **Mesmo modelo de embedding e mesmo LLM** em todas as configurações.
- **Temperatura fixa = 0.**
- O top-k pode ser ajustado, mas **o mesmo top-k em todas as configurações** da comparação (use o melhor).
- Registrar modelo, versão, parâmetros e resultados.
- Depois de cada medição: **o que o resultado mostrou e que decisão ele motivou.**

### Entregas
- **Notebook** que roda do início ao fim **sem intervenção manual**, entregue com as saídas: linha de base, configurações, resultados, análise após cada resultado, comparação final e conclusão.
- **Apresentação** (ppt/doc/pdf), curta: configuração escolhida e por quê; números das 62 perguntas e do RAGAS; números dos 200 chamados; onde ainda erra; limitações e próximos passos.

### O que vale nota
> "Qualidade do raciocínio experimental, capacidade de comparar configurações e justificar decisões com evidências. Um resultado que não seja o melhor, mas cuja causa tenha sido corretamente investigada, pode valer mais do que um resultado aparentemente alto sem análise consistente."

Ou seja: **hipótese → medição → investigação da causa → decisão.** O caminho é o produto.

### Dados (dicionário)
**Central de documentos** (`central-documentos/`, 44 `.md` + `manifest.json`). Cada entrada do manifesto tem:
- `titulo`
- `publico`: `cliente`, `vendedor`, `institucional`, `editorial`, `revogada`
- `estrutura`: `prosa`, `prosa_tabela`, `tabela`, `lista`, `passos`, `comunicado`
- `vigencia`: `inicio/fim`; fim vazio = em vigor; campo vazio = sem vigência definida

**Chamados** (`chamados-200.csv`, `;`, UTF-8): `id_reclamacao`, `id_cliente`, `tipo_cliente` (padrao/premium), `id_produto`, `nome_produto`, `descricao_produto`, `categoria_produto` (brinquedo, game, livro, maquiagem, outros), `preco` (vírgula decimal), `vendedor`, `tipo_vendedor`, `avaliacao_produto`, `data_compra`, `data_abertura`, `texto_reclamacao`, `categoria_informada_cliente`, `tags_sistema` (critério desconhecido), `classe` (do classificador).

**Perguntas de referência** (`perguntas-referencia.csv`, 62 linhas): `id`, `pergunta`, `data_referencia`, `resposta_referencia` (trecho que precisa aparecer).

---

## 1. O que a exploração dos dados já revelou

Fiz uma exploração inicial dos arquivos desta pasta. Estes achados orientam o plano. Na Etapa 1 você vai **reproduzir** isso no notebook, com código e saídas, porque o professor quer ver o raciocínio.

### 1.1 Composição da central (44 documentos)
| público | qtd | documentos | comentário |
|---|---|---|---|
| cliente | 23 | 8 políticas `pol-*`, 12 comunicados `com-*`, 3 boletins `bol-*` | a base certa |
| revogada | 5 | `rev-dev-v1`, `rev-dev-v2`, `rev-gar-v0`, `rev-ent-v1`, `rev-clu-v0` | **versões antigas, necessárias para perguntas históricas** |
| vendedor | 8 | `ven-*` | regras do vendedor, ruído para o cliente (O3) |
| institucional | 5 | `ins-*` | termos, privacidade, imprensa, relatório |
| editorial | 3 | `blog-*` | texto genérico ("confira a política"), sem regra |

Tamanhos: comunicados têm ~150–200 caracteres (uma frase); boletins ~500; blogs ~1.000; políticas 12–19 mil. **Um chunk fixo de 800–1000 caracteres engole um comunicado inteiro junto com nada, e quebra uma política no meio de uma tabela.**

### 1.2 Onde está cada resposta (mapa resposta → documento)
Procurei a `resposta_referencia` de cada pergunta dentro de cada documento:

- **Perguntas históricas dependem de documentos `revogada`:**
  - q19 (2026-05-15, teto padrão R$ 2.000,00) → `rev-dev-v2` (e `com-2026-014`, que diz "passa de R$ 2.000,00 para R$ 4.000,00", mas só vale a partir de 2026-07-01)
  - q22 (2025-11-01, garantia brinquedo 30 dias) → `rev-gar-v0`
  - q47 (2025-11-01, teto padrão R$ 1.500,00) → **só** `rev-dev-v1`
  - q48 (2026-05-15, reembolso acima de R$ 100 = 15 dias úteis) → `rev-dev-v2`
  - q49 (2025-11-15, teto do assinante R$ 5.000,00) → **só** `rev-clu-v0`
  - q21 (2026-07-22, "acrescidos de 2 dias úteis") → **só** `com-2026-016` (vigência 2026-07-15 a 2026-07-31)
- **Perguntas que só os boletins/comunicados respondem:** q55 (horário do chat, `bol-2026-04`), q56 (ponto parceiro, `bol-2026-07`), q57 (Troca Verde, `bol-2026-08`), q58, q59, q61 (comunicados).
- **Respostas que dependem de tabela:** q12, q40, q41 (faixas de valor na tabela da POL DEV 03), q13, q14, q42, q46 (tabela da POL GAR 01).
- **Perguntas que citam códigos literais:** "COM 2026 019", "POL AUT 02", "COM 2026 014", "COM 2026 007", "POL ENT 02", além de valores ("R$ 80", "R$ 350", "R$ 500"). Isso favorece **BM25 / busca híbrida**.

> **Consequência importante:** uma curadoria ingênua que **apaga os documentos revogados** derruba q19, q22, q47, q48 e q49. O certo é **manter os revogados e deixar o filtro de vigência decidir quando cada um vale.** Isso pode virar um experimento didático (ver Trilha C, seção 4).

### 1.3 Problemas de qualidade da base (as "coisas simples para arrumar")
1. **Vigência em formatos diferentes:** `pol-dev-03` tem fim `9999-12-31`; as outras políticas em vigor têm fim vazio. Normalizar.
2. **Vigência vazia** em `ven-*`, `ins-*`, `blog-*`: decidir a regra (sugestão: vazio = sem restrição de data; esses documentos saem pela curadoria de público de qualquer forma).
3. **`rev-dev-v1` tem o cabeçalho errado no texto:** diz "Versão 1, vigente até 31/12/2025" e logo abaixo "Vigente a partir de 01/07/2026. Substitui a versão 1" (cópia malfeita da v3). O LLM pode ler isso e confundir as versões. Corrigir o texto ou confiar no manifesto e injetar a vigência correta no cabeçalho do chunk.
4. **Políticas com "Data de vigência: conforme publicação oficial"** no texto (POL ATD 01, POL AUT 02, POL GAR 01...). O texto não ajuda; a vigência tem de vir do manifesto.
5. **Seções redundantes** na POL DEV 03 ("1.4 Reafirmação do prazo", "4.4 Reafirmação da regra...", "5.4 Reafirmação do cancelamento...") e **anexos-resumo** que repetem o corpo. Ocupam vagas no top-k com o mesmo conteúdo.
6. **Tabelas em markdown** (`| ... |`) na POL DEV 03, POL GAR 01, POL CLU 01, POL DIS 04, `ven-repasse`. Chunk fixo separa o cabeçalho da tabela das linhas.
7. **Seções de "Perguntas frequentes"** nas políticas: pares pergunta-resposta, ótimos como chunks individuais (casam com o jeito do cliente perguntar).

### 1.4 Os 200 chamados
- 7 classes: 30 de cada (`entrega`, `divergente_defeito`, `fraude_vendedor`, `falsificado`, `artigo_proibido`, `devolucao_ressarcimento`) + 20 `outros`.
- **`categoria_informada_cliente` diverge da `classe` em ~60 chamados.** A `classe` (do classificador) é a que vale como metadado; a do cliente é ruidosa.
- **`tags_sistema` são ruído** ("urgente", "ajuda", "pedido"...), critério desconhecido. Bom candidato a ser **ignorado**, com justificativa.
- **`outros` (20) são, em sua maioria, casos sem cobertura na base:** cobrança em duplicidade, nota fiscal não recebida, produto é bivolt?, trocar boleto por cartão, cupom de primeira compra não aplicado, garantia estendida da loja, acesso à conta. **É aqui que a abstenção (O4) deveria aparecer.** Confirmar na Etapa 1 procurando esses temas na base.
- **Datas de abertura de 2026-01-07 a 2026-10-20:** 134 abertos antes de 2026-07-01 (vale a POL DEV 03 **v2**, teto padrão R$ 2.000,00); 8 abertos durante o recesso de 20 a 27/07 (com-2026-016).
- **13 chamados com compra antes de 01/07/2026 e abertura depois.** A POL DEV 03 v3 diz valer "para pedidos realizados a partir de 01/07/2026", mas o enunciado manda usar a data do chamado. Decisão a registrar: seguir o enunciado (data de abertura) e citar o caso como limitação.
- **Preços:** 18 acima de R$ 500 (análise documental), nenhum acima de R$ 2.000. Faixa de valor (até 100 / 100–500 / >500) e categoria do produto (garantia) definem o parâmetro correto: **colocar preço, categoria, tipo de cliente e data na consulta/prompt é essencial para O2.**
- **44 clientes `premium`.** Hipótese: premium = assinante do Clube Q+ (teto R$ 8.000,00, metade dos prazos, fila prioritária). Verificar na Etapa 1.

### 1.5 Limitações da métrica de recuperação (anotar desde já)
A métrica é "a string `resposta_referencia` aparece em algum trecho recuperado?". Ela é útil, mas:
- **Falso acerto:** "30 dias" (q22, brinquedo em 2025) aparece na linha de **livro** da POL GAR 01 atual; "5 dias úteis", "10 dias úteis" e "retido" aparecem em vários documentos que não respondem a pergunta; "R$ 2.000,00" aparece no `com-2026-014`, que fala do valor novo.
- **Falso erro:** com chunk fixo, a string pode ser cortada na fronteira entre dois chunks.
- **Inflação por trechos maiores:** parent-document retrieval e top-k maior aumentam a chance de a string aparecer "por acaso".
- **Não mede geração:** mudanças só no prompt não mexem nesse número; elas aparecem no RAGAS e no juiz dos 200 chamados.

**Proposta:** manter a célula oficial intacta e acrescentar uma **célula de diagnóstico** que mostra, para cada acerto, **de qual documento veio** e se esse documento estava vigente na data. Isso dá base para a análise ("o ganho foi real ou foi sorte?").

---

## 2. Catálogo de técnicas, da mais fácil à mais difícil

Legenda: **Custo** = esforço de implementação. **LLM?** = se exige chamadas de LLM (custo de API). **Mede no 62?** = se mexe na métrica de recuperação.

| # | Técnica | O que é | Objetivo | Custo | LLM? | Mede no 62? |
|---|---|---|---|---|---|---|
| T1 | **Normalização do manifesto** | converter vigência em datas (`date`), fim vazio/`9999-12-31` → "em aberto", vazio → sem restrição | pré-requisito de O1 | muito baixo | não | indireto |
| T2 | **Filtro de vigência** | antes da busca, manter só chunks cujo documento estava vigente na data de referência | O1 | baixo | não | sim |
| T3 | **Curadoria por público** | tirar `vendedor`, `institucional`, `editorial`; **manter** `revogada` (o T2 cuida) | O3 | baixo | não | sim |
| T4 | **Curadoria de conteúdo** | corrigir o cabeçalho de `rev-dev-v1`; remover seções "Reafirmação" e anexos redundantes; remover cabeçalhos vazios | O1, O2 | baixo | não | sim |
| T5 | **Ajuste de chunk/overlap e top-k** | variar tamanho (300/500/800/1200) e overlap (0/100/200) no chunk fixo; varrer k | O2 | baixo | não | sim |
| T6 | **Chunking por estrutura** | quebrar por cabeçalho markdown (`##`/`###`); comunicado e boletim = 1 chunk (ou 1 por `##`); tabela inteira com cabeçalho; FAQ = 1 par por chunk; usar o campo `estrutura` do manifesto para escolher a regra | O2 | médio | não | sim |
| T7 | **Cabeçalho contextual no chunk (metadados no texto)** | prefixar cada chunk com "Documento: título · Seção: X · Público · Vigência: início–fim" antes de gerar o embedding e no texto enviado ao LLM | O1, O2, O3 | baixo (depois do T6) | não | sim |
| T8 | **Linearização de tabelas** | transformar cada linha de tabela numa frase ("Compras de até R$ 100,00: reembolso integral em 5 dias úteis") | O2 | médio | não | sim |
| T9 | **Busca híbrida (BM25 + densa)** | duas listas (BM25 com `rank_bm25` e embeddings), fundidas por RRF (Reciprocal Rank Fusion, k=60) ou soma ponderada de scores normalizados | O2 (códigos, valores) | médio | não | sim |
| T10 | **Re-ranking** | buscar top-20/30 e reordenar com cross-encoder multilingue (ex.: `BAAI/bge-reranker-v2-m3` ou `cross-encoder/mmarco-mMiniLMv2-L12-H384-v1`) ou com o próprio Gemini; ficar com o top-k | O2 | médio | opcional | sim |
| T11 | **Parent-document retrieval** | indexar chunks pequenos (frase/parágrafo) e devolver a seção-pai inteira | O2 | médio | não | sim, mas **infla a métrica** (ver 1.5) |
| T12 | **Metadados do chamado na consulta e no prompt** | enriquecer a consulta com `classe`, `categoria_produto`, `preco`, `tipo_cliente` (premium → Clube Q+) e usar `data_abertura` na vigência; opcional: filtrar/priorizar documentos por classe (mapa classe → políticas) | O2, O3 | médio | não | **não** (as 62 perguntas não têm classe); mede nos 200 |
| T13 | **Prompt de geração com abstenção** | papel, escopo, data do chamado, tipo de cliente, "responda só com os trechos", "cite o documento", "se não houver, diga que a base não cobre e encaminhe"; saída estruturada | O4, "inventou" | baixo | sim | não; mede no RAGAS e nos 200 |
| T14 | **Resolução de conflito entre versões/comunicados** | ordenar trechos por vigência e instruir: "comunicado mais recente vigente prevalece sobre texto anterior" | O1 | baixo | sim | indireto |
| T15 | **Limiar de abstenção** | se o melhor score (rerank/similaridade) < limiar, não gerar e responder "sem informação suficiente"; calibrar o limiar nos `outros` | O4 | médio | não | não; mede nos 200 |
| T16 | **Reescrita de consulta (query rewriting / HyDE)** | o LLM transforma a reclamação informal e longa numa pergunta curta de política (ou gera uma resposta hipotética para buscar) | O2 | alto | sim | sim |
| T17 | **Checagem de suficiência (self-check)** | um passo do LLM decide "os trechos respondem? sim/não" antes de responder | O4 | alto | sim | não; mede nos 200 |

**Sobre as escolhas que você já cogitou:**
- *Chunking + metadados (com filtro)* = T1, T2, T3, T6, T7 (e T8 se quiser). É a parte de maior ganho esperado e a mais barata. **Fortemente recomendado.**
- *Busca híbrida* = T9. Faz sentido aqui por causa dos códigos ("COM 2026 019") e valores literais. **Recomendado.**
- *Re-ranking* = T10. Bom candidato depois da híbrida: a híbrida aumenta o recall, o re-ranking corrige a ordem. **Recomendado.**
- *Parent-document* = T11. Com chunking por seção (T6) os chunks já são "seções", então o ganho tende a ser menor e a métrica fica inflada. **Opcional; só faça se tiver tempo e compare com orçamento de caracteres igual.**

---

## 3. Encadear ou isolar as técnicas?

**Proposta: cadeia cumulativa.** Cada configuração = configuração anterior + **uma** mudança. Exemplo: C2 = C1 + curadoria; C3 = C2 + chunking por estrutura.

**Por que encadear:**
1. Atende à regra "uma mudança por vez": a diferença entre C(n) e C(n-1) é atribuível a uma única mudança.
2. Reflete como se constrói um sistema de verdade: consertar a base, depois a busca, depois o refinamento.
3. Várias técnicas só fazem sentido sobre outras: o cabeçalho contextual (T7) depende do chunking por seção (T6); o re-ranking (T10) precisa de um bom conjunto de candidatos (T9).

**Quando isolar (e por quê), mesmo encadeando:**
1. **Curadoria por público × filtro de vigência:** as duas removem documentos e podem se sobrepor (os revogados). Para separar o efeito, cada uma entra numa configuração diferente. Se quiser mostrar a armadilha, rode a variante "curadoria ingênua" (apaga revogados) **isolada** e mostre a perda em q19/q22/q47/q48/q49.
2. **Busca híbrida × re-ranking:** atacam o mesmo sintoma (trecho certo fora do top-k). Encadeados, o ganho do re-ranking depende da híbrida. **Ablação no fim:** rodar a configuração final **sem** a híbrida e **sem** o re-ranking, para mostrar a contribuição de cada uma.
3. **Parent-document:** muda a unidade devolvida, então a métrica não é comparável diretamente. Rodar isolado sobre a melhor configuração e comparar com um limite de caracteres igual ao das outras.

**Truque barato:** a métrica das 62 perguntas **não chama o LLM** (só embeddings, que ficam em cache). Dá para fazer ablações e varreduras quase de graça. O custo de API está só no RAGAS e nos 200 chamados.

---

## 4. Trilhas de configuração propostas

> **É aqui que você escolhe.** Abaixo há três trilhas. A recomendada é a **B**. Para cada configuração, a hipótese a ser escrita **antes** de medir.

### Pré-configuração (não conta como melhoria): escolha do top-k
Varrer k ∈ {3, 5, 8, 10} na linha de base, olhar acertos e tamanho do contexto, fixar um k (provável: 5) e **usar o mesmo em todas as configurações.** Registrar a escolha e o motivo (k maior infla a métrica e o custo de geração).

### Trilha A, mínima (3 configurações, o exigido)
| Config | Mudança | Hipótese |
|---|---|---|
| C1 | filtro de vigência (T1+T2) | acerta as históricas (q19, q21, q22, q47, q48, q49) e tira ruído de versões antigas nas atuais |
| C2 | curadoria por público + conteúdo (T3+T4) | menos ruído de vendedor/institucional/blog → mais vagas no top-k para a política certa |
| C3 | chunking por estrutura + cabeçalho contextual (T6+T7) | tabelas e comunicados inteiros → acerta prazo/valor (q12, q40, q41, q13, q14...) |

### Trilha B, recomendada (5 configurações + ablação)
| Config | Mudança | Hipótese |
|---|---|---|
| C1 | filtro de vigência (T1+T2) | idem |
| C2 | curadoria por público + conteúdo (T3+T4) | idem |
| C3 | chunking por estrutura + cabeçalho contextual (T6+T7) | idem |
| C4 | busca híbrida BM25 + densa com RRF (T9) | acerta perguntas com códigos/valores literais (q23, q26, q51, q52, q53, q41) |
| C5 | re-ranking com cross-encoder (T10) sobre top-20 da C4 | sobe o trecho certo dentro do top-k fixo |
| Ablação | C5 sem T9; C5 sem T10 | mostra a contribuição de cada técnica de busca |
| Geração | prompt com abstenção + metadados do chamado (T12+T13+T14) | entra na etapa do RAGAS/200 chamados, medida pelo RAGAS e pelo juiz |

### Trilha C, completa (para quem quer ir fundo)
Trilha B, mais:
- **C0-bis (experimento didático):** curadoria ingênua que apaga revogados → mostrar a queda e explicar a causa.
- **C6:** parent-document retrieval (T11), com orçamento de caracteres igual.
- **C7:** linearização de tabelas (T8) ou reescrita de consulta (T16) para os chamados.
- **Abstenção por limiar (T15)** calibrada nos `outros`.

### Por que esta ordem
1. **Primeiro consertar a base, depois sofisticar a busca.** Filtro e curadoria são baratos, atacam diretamente O1 e O3 e reduzem o ruído que qualquer técnica posterior teria de enfrentar.
2. **Vigência antes de curadoria:** se a curadoria viesse antes, seria tentador apagar os revogados. Com a vigência já funcionando, a curadoria pode ficar só com vendedor, institucional e editorial.
3. **Chunking antes da busca:** BM25, embeddings e re-ranking operam sobre chunks. Mudar o chunking depois obrigaria a refazer as conclusões sobre a busca.
4. **Híbrida antes de re-ranking:** o re-ranking só reordena o que foi recuperado; primeiro aumentar o recall, depois a precisão.

---

## 5. Etapas do trabalho

Cada etapa tem: **Objetivo · O que construir · O que observar · O que registrar · Critério de saída · Prompt para a IA.**

### Métricas e onde cada uma aparece

| Métrica | O que mede | Custo | Onde aparece |
|---|---|---|---|
| **Hit@k** (célula oficial) | em quantas das 62 perguntas a resposta aparece nos k trechos | grátis (sem LLM) | **todas** as configurações |
| **Hit@1, Hit@3** | o mesmo, olhando só os 1 ou 3 primeiros trechos | grátis | todas, na célula de diagnóstico |
| **MRR** (Mean Reciprocal Rank) | média de 1/posição do primeiro trecho certo (1º lugar = 1; 2º = 0,5; fora do top-k = 0). Mostra se o trecho certo **subiu**, mesmo quando o Hit@k não muda; é a métrica que revela o efeito do re-ranking | grátis | todas, na célula de diagnóstico |
| **Acertos de fonte certa** | acertos cujo trecho veio de documento vigente e de público cliente (descarta acerto "por acaso") | grátis | todas, na célula de diagnóstico |
| **RAGAS** (fidelidade, precisão do contexto, relevância da resposta) | qualidade da resposta gerada e dos trechos, julgada pelo LLM | ~200 chamadas por configuração | **Etapa 7**, para a linha de base e **todas** as configurações, na mesma amostra |
| **Juiz dos 4 critérios** | desfecho, parâmetro, inventou, abstenção | ~400 chamadas | Etapa 8, só na configuração final |

**Por que o RAGAS fica numa etapa só, cobrindo todas as configurações:** é caro e precisa gerar respostas; rodar todas as configurações juntas garante a **mesma amostra, o mesmo prompt e a mesma versão do avaliador**, o que torna a tabela de evolução comparável. As métricas grátis (Hit@k, MRR, fonte certa) acompanham o progresso etapa a etapa; o RAGAS confirma no fim se o ganho na recuperação virou ganho na resposta.

**Painel de progresso:** uma função `painel()` (criada na Etapa 2) imprime a tabela configuração × Hit@1 × Hit@3 × Hit@k × MRR × fonte certa, acumulando a cada configuração.

### Etapa 0. Ambiente, Gemini e reprodutibilidade
**Objetivo:** notebook que roda de ponta a ponta sem intervenção, com Gemini, cache e registro de parâmetros.

**O que construir:**
- Instalação: `google-genai`, `sentence-transformers`, `rank_bm25`, `ragas`, `langchain-google-genai`, `pandas`. Fixar versões depois que funcionar (`pip freeze | grep ...` registrado no notebook).
- Chave: ler de **Colab Secrets** (`google.colab.userdata.get('GOOGLE_API_KEY')`) ou variável de ambiente; `getpass` só como último recurso (é intervenção manual).
- Constantes de experimento num só lugar: `LLM_MODEL`, `EMB_MODEL`, `TEMPERATURE = 0`, `TOP_K`, `SEED`.
- `get_llm_response(messages, ...)` com a mesma assinatura do esqueleto, agora em Gemini (seção 7).
- **Cache** em disco de embeddings e de respostas do LLM (chave = hash de modelo + parâmetros + texto). Rodar de novo fica barato e determinístico.
- **Retry com backoff** para limites de taxa da API.
- Carregamento de dados: manter o carregamento por URL do esqueleto (o professor roda assim) e, opcionalmente, conferir que os arquivos locais desta pasta são idênticos (hash).

**Observar:** a célula de teste responde; uma segunda execução usa o cache (sem chamada à API).
**Registrar:** tabela "Configuração experimental" com modelo, versão, embedding, dimensão, temperatura, top-k, versões das bibliotecas, data.
**Saída:** "Reiniciar e executar tudo" funciona até a carga dos dados.

> **Prompt para a IA:** "Leia o plano-estudos.md (seções 0, 6 e 7). Estou na Etapa 0. Adapte as células de configuração do esqueleto para usar Gemini via `google-genai`, mantendo a assinatura `get_llm_response(messages, max_tokens)`. Inclua cache em disco para LLM e embeddings, retry com backoff e leitura da chave via Colab Secrets com fallback para variável de ambiente. Não altere a célula de carga dos dados."

---

### Etapa 1. Exploração dos dados (EDA)
**Objetivo:** conhecer a base e os conjuntos de avaliação, e transformar os achados da seção 1 em evidência dentro do notebook.

**O que construir (células curtas, cada uma com uma pergunta):**
1. Manifesto como `DataFrame`: contagem por `publico`, `estrutura`; tamanho em caracteres de cada documento; vigências em linha do tempo (gráfico de barras horizontais por documento ajuda muito).
2. Inconsistências: formatos de vigência; documentos sem vigência; texto do cabeçalho de `rev-dev-v1` vs manifesto.
3. **Mapa resposta → documento:** para cada pergunta, quais documentos contêm a `resposta_referencia`, com público e vigência. Marcar quais dependem de revogados, de comunicados/boletins e de tabelas.
4. Perguntas por `data_referencia`: quantas são históricas.
5. Chamados: distribuição de `classe`, `tipo_cliente`, `categoria_produto`, `tipo_vendedor`; matriz `classe` × `categoria_informada_cliente`; frequência de `tags_sistema`; histograma de `data_abertura` com as datas de corte (2026-07-01, recesso 20–27/07); faixas de preço.
6. **Hipótese premium = Clube Q+:** procurar menções a "Clube"/"assinante" nos textos de chamados premium e ver o que a POL CLU 01 diz sobre o plano.
7. **Cobertura dos `outros`:** buscar na base termos como "duplicidade", "nota fiscal", "bivolt", "boleto", "cupom de primeira compra", "garantia estendida", "acesso à conta". Se não houver nada, esses casos são o teste de abstenção.

**Observar:** quais perguntas a linha de base deve errar (previsão!), quais documentos são ruído, que metadados existem.
**Registrar:** um markdown "O que a exploração mostrou" com 5–8 achados e a **previsão** de quais perguntas a linha de base vai errar. Comparar a previsão com o resultado na Etapa 3 é ótimo material de raciocínio.
**Saída:** lista de achados e lista de hipóteses que justificam a ordem das configurações.

> **Prompt para a IA:** "Leia o plano-estudos.md (seções 0, 1 e 5, Etapa 1). Crie as células de EDA listadas, uma pergunta por célula, com pandas e gráficos simples em matplotlib. Ao final, gere um rascunho do markdown 'O que a exploração mostrou' com base nas saídas reais, não no plano."

---

### Etapa 2. Infraestrutura de avaliação (antes da linha de base)
**Objetivo:** ter as ferramentas para medir, comparar e investigar qualquer configuração com uma linha de código.

**O que construir:**
- **Estrutura de chunk padrão:** `dict` com `texto` (o que vai para o LLM e para a métrica), `doc_id`, `titulo`, `publico`, `estrutura`, `vig_ini`, `vig_fim`, `secao`, `chunk_id`. Todas as configurações usam a mesma estrutura.
- **Fábrica de recuperadores:** cada configuração define uma função `recupera(pergunta, data, **meta) -> list[str]`, como exige a célula de medição. Guardar também uma versão `recupera_detalhado` que devolve os chunks com metadados e scores.
- **Célula de medição oficial:** copiar exatamente a do esqueleto (não alterar).
- **Registro:** `RESULTADOS[nome_config] = {"acertos": ..., "erradas": [...], "params": {...}}`.
- **Diagnóstico:** função `diagnostico(nome_config)` que, para cada pergunta, diz se acertou, **em que posição** veio o primeiro trecho certo, **de qual documento veio o acerto** e se esse documento estava vigente na data. Calcula Hit@1, Hit@3, Hit@k, MRR e "acertos de fonte certa".
- **Painel de progresso:** `painel()` imprime a tabela de todas as configurações medidas até agora.
- **Comparador:** `compara(cfg_a, cfg_b)` → perguntas ganhas, perdidas, mantidas.
- **Inspetor:** `inspeciona(qid, cfg)` → imprime os top-k com `doc_id`, seção, vigência e score. É a ferramenta para "investigar a causa".
- **Matriz final:** pergunta × configuração (✔/✘), para a comparação final.

**Saída:** funções testadas com uma recuperação aleatória boba (deve acertar quase nada).

> **Prompt para a IA:** "Leia o plano-estudos.md (Etapa 2 e seção 8). Implemente a estrutura de chunk, o registro RESULTADOS, as funções diagnostico, compara, inspeciona e a matriz pergunta × configuração. A célula de medição do esqueleto não pode mudar; ela usa `recupera(pergunta, data)`."

---

### Etapa 3. Linha de base (a solução da equipe anterior)
**Objetivo:** reproduzir fielmente o sistema antigo e medir **antes** de qualquer melhoria.

**O que construir:**
- Chunk de tamanho fixo (sugestão: 800 caracteres, overlap 100) sobre **os 44 documentos**, sem metadados no texto.
- Embeddings com o modelo fixado (Gemini `gemini-embedding-001` com `task_type` RETRIEVAL_DOCUMENT/RETRIEVAL_QUERY, ou um modelo local multilingue; ver seção 7). **Este modelo não muda mais.**
- Similaridade por cosseno, top-k fixo. **Ignora a data** (o parâmetro `data` é recebido e não usado).
- Prompt de geração da equipe anterior: papel, escopo, "responda só com os trechos". Simples, como no esqueleto.
- Medição + diagnóstico + 2 ou 3 inspeções de perguntas erradas.

**Observar:** quantas acertou; quais erraram; o que as erradas têm em comum (histórica? tabela? comunicado curto afogado? código literal?); quantos acertos vieram de documento **não vigente** ou **de vendedor** (acerto "sujo").
**Registrar:** análise no formato da seção 8. Decisão: "a primeira mudança será X porque Y".
**Saída:** número da linha de base registrado e congelado.

> **Prompt para a IA:** "Leia o plano-estudos.md (Etapa 3). Implemente a indexação da linha de base (chunk fixo 800/overlap 100 sobre os 44 documentos, embedding fixado, cosseno) e a função `recupera(pergunta, data)` que ignora a data. Implemente também a geração com o prompt simples. Depois rode a medição e o diagnóstico e me mostre as perguntas erradas agrupadas por motivo provável."

---

### Etapa 4. Escolha do top-k
**Objetivo:** fixar o k usado em todas as comparações, com justificativa.
**O que construir:** loop sobre k ∈ {3, 5, 8, 10} na linha de base; tabela k × acertos × caracteres médios de contexto.
**Observar:** onde o ganho satura; o custo (tokens) de cada k.
**Registrar:** "k = X porque o ganho de X para Y foi de N perguntas, com M% mais contexto". Lembrar que k maior infla a métrica.
**Saída:** `TOP_K` fixado nas constantes.

---

### Etapa 5. Configurações de melhoria (repetir o bloco para cada uma)
Para **cada** configuração da trilha escolhida (seção 4):

1. **Markdown "Configuração N. Nome":** o que muda (só uma coisa), a hipótese e a previsão (quais perguntas devem virar).
2. **Código:** só a mudança. Reaproveitar tudo da anterior.
3. **Medição oficial** (mesma célula).
4. **Diagnóstico + comparador** contra a anterior.
5. **Inspeção** de 1 ou 2 perguntas que ainda erram, e de qualquer pergunta que **piorou**.
6. **Análise** (modelo da seção 8): número vs anterior, ganhas/perdidas, causa investigada, decisão.

**Dicas por técnica:**
- **Filtro de vigência:** `vig_ini <= data <= vig_fim` (fim aberto = sempre). Filtrar **antes** do ranking (pré-filtro), não depois; senão o top-k fica com menos de k trechos.
- **Curadoria:** usar o `publico` do manifesto. Manter `cliente` + `revogada`. Registrar quantos chunks saíram.
- **Chunking por estrutura:** regras por `estrutura`: `comunicado` → doc inteiro; boletim → um chunk por `##`; `prosa`/`prosa_tabela`/`passos` → por `###` (ou `##` quando não houver `###`), quebrando seções muito longas com overlap; `tabela`/tabelas → tabela inteira com o cabeçalho e a frase que a introduz; FAQ → um par pergunta-resposta por chunk. Remover seções redundantes (T4) se ainda não removidas.
- **Cabeçalho contextual:** `"[{titulo} | {secao} | público: {publico} | vigência: {ini} a {fim}]\n{texto}"`. O cabeçalho entra no embedding, no BM25 e no texto do LLM.
- **Busca híbrida:** BM25 com tokenização em minúsculas, sem acento, sem stopwords em português; buscar top-20 em cada uma; fundir por RRF `score = Σ 1/(60 + rank)`; aplicar o filtro de vigência nas duas listas.
- **Re-ranking:** candidatos = top-20 da híbrida; cross-encoder multilingue; devolver top-k. Registrar o modelo de re-ranking também (ele também precisa ficar fixo).
- **Parent-document:** filhos = frases/parágrafos; pai = seção. Deduplicar pais. Limitar o total de caracteres para comparar com justiça.

> **Prompt para a IA (por configuração):** "Leia o plano-estudos.md (seções 3, 4 e Etapa 5). Estou na Configuração N da Trilha [A/B/C]: [nome]. A configuração anterior é [N-1]. Implemente **só** essa mudança, reaproveitando o código anterior. Depois de eu rodar a medição, me ajude a escrever a análise no formato da seção 8, usando as saídas reais do diagnóstico e do comparador."

---

### Etapa 6. Ajuste da geração (prompt)
**Objetivo:** atacar "inventa regra" e "responde sem informação" (O4), que a métrica das 62 não enxerga.

**O que construir:**
- Prompt de sistema com: papel (atendente de primeira resposta da Quantum Commerce), escopo (só regras do cliente), **data do chamado**, **tipo de cliente**, "use apenas os trechos; cite o documento de cada regra; se os trechos não cobrem o caso, diga isso explicitamente e encaminhe para atendimento humano; não use regras de vendedor; quando houver versões diferentes, use a vigente na data".
- Trechos numerados, com o cabeçalho contextual (documento e vigência visíveis ao LLM).
- Para os chamados: incluir `data_abertura`, `tipo_cliente`, `categoria_produto`, `preco`, `classe` no prompt (T12).
- Opcional: saída estruturada (desfecho, prazo/valor, documento citado, `cobertura: sim/nao`).

**Medir:** isso é uma mudança de geração. Comparar o prompt antigo e o novo **na mesma recuperação** via RAGAS (fidelidade) e, se quiser, numa amostra de 20 chamados com o juiz. Registrar como mudança separada, para não misturar com o efeito da recuperação.

---

### Etapa 7. Avaliação com RAGAS
**Objetivo:** comparar a linha de base e **todas** as configurações (C1 a C5) na mesma amostra, montando a tabela de evolução do RAGAS, e localizar onde está o ganho ou a perda: **recuperação ou geração**. Também comparar o prompt antigo e o novo (Etapa 6) sobre a configuração final. Se o custo ou o tempo apertar, reduzir para linha de base, C3 e C5 e justificar.

**O que construir:**
- Amostra fixa (ex.: 20 perguntas, `random.seed(42)`, estratificada para incluir históricas, de tabela e de comunicado). **A mesma amostra para todas.**
- Para cada configuração: `user_input` = pergunta, `retrieved_contexts` = trechos, `response` = resposta gerada, `reference` = `resposta_referencia`.
- Métricas: `Faithfulness`, `LLMContextPrecisionWithReference` (ou `WithoutReference`), `ResponseRelevancy` (answer relevancy).
- Avaliador = **o mesmo Gemini, temperatura 0** (via `langchain-google-genai`), embeddings do mesmo modelo.

**Leitura:**
- Precisão de contexto sobe e fidelidade não → o problema está na geração.
- Fidelidade alta e precisão de contexto baixa → o LLM é fiel a trechos ruins (recuperação).
- Relevância baixa com abstenção correta pode ser esperado (resposta "não sei" é pouco "relevante"); comentar isso.

**Registrar:** tabela configuração × métrica, análise e a decisão de qual configuração vai para os 200 chamados.

> **Prompt para a IA:** "Leia o plano-estudos.md (Etapa 7 e seção 7). Monte a avaliação RAGAS com Gemini como avaliador (temperatura 0) para as configurações [lista], numa amostra fixa e estratificada de 20 perguntas. Use cache para não repetir chamadas. Me devolva uma tabela configuração × métrica."

---

### Etapa 8. Rodada nos 200 chamados
**Objetivo:** avaliar a configuração final em situação real.

**O que construir:**
- Para cada chamado: consulta = `texto_reclamacao` (+ metadados, se a configuração usar T12); data = `data_abertura`; gerar resposta; julgar com `PROMPT_JUIZ` (pode ajustar o texto, **mantendo os 4 critérios**), JSON forçado (`response_mime_type="application/json"`), temperatura 0.
- Cache de respostas e de julgamentos (são ~400 chamadas).
- Contagens: % desfecho = sim; % parâmetro = sim (sobre os não-`na`); % inventou = sim (quanto menor, melhor); abstenção: % sim nos `outros` (deveria ser alto) e % sim nas demais classes (deveria ser baixo).
- **Tabela por classe** dos quatro critérios. **Cruzamentos úteis:** antes vs depois de 2026-07-01; padrão vs premium; faixa de preço.
- **Checagem do juiz:** ler à mão 10 julgamentos (alguns "sim", alguns "não") e anotar se você concorda. O juiz também erra, e mostrar isso conta pontos.

**Registrar:** análise: em que classes erra o desfecho, onde inventa, se a abstenção aparece onde deveria, o que você mudaria na base a partir disso.

---

### Etapa 9. Comparação final, conclusão e apresentação
- **Tabela final:** configuração × acertos (62) × acertos de fonte certa × RAGAS (candidatas).
- **Matriz pergunta × configuração** e um gráfico de barras da evolução.
- **Ablação** (se a trilha incluir).
- Conclusão respondendo às 4 perguntas do esqueleto, **com números**:
  1. qual configuração venceu e por quê;
  2. onde ainda erra e em que etapa (recuperação ou geração);
  3. se algum resultado ficou fraco, qual a causa investigada;
  4. o que faria em seguida.
- Apresentação curta (5–8 slides): problema, caminho percorrido, números (62, RAGAS, 200), onde erra, próximos passos.
- **Teste final:** "Reiniciar e executar tudo" limpo, salvar com as saídas.

---

## 6. Estrutura do novo notebook

Baseado no esqueleto `FIAP_Knowledge_Management_and_Prompt_Eng_Projeto_Final_Esqueleto.ipynb`, mantendo a ordem do enunciado. Sugestão de nome: `Projeto_Final_QuantumCommerce.ipynb`.

```
Cabeçalho (título, integrantes, créditos)                     ← do esqueleto
0. Configuração e dados
   0.1 Instalação (versões fixadas)
   0.2 Provedor LLM: Gemini (chave via Secrets)                ← substitui Anthropic/OpenAI
   0.3 get_llm_response + embed + cache + retry
   0.4 Constantes do experimento (modelos, temperatura, top-k)  ← tabela de registro
   0.5 Carga dos dados (célula do esqueleto, intacta)
0.6 Exploração dos dados (Etapa 1)                             ← nova seção
   achados + previsões
0.7 Infraestrutura de avaliação (Etapa 2)                      ← nova seção
1. Linha de base
   indexação | recuperação e geração | medição (intacta) | diagnóstico | análise
1.1 Escolha do top-k                                           ← Etapa 4
2. Melhorias, uma de cada vez
   Configuração 1..N: [mudança | medição | diagnóstico/compara | análise]
   Ablação (opcional)
   Ajuste da geração (Etapa 6)
3. Avaliação com RAGAS
4. Rodada nos 200 chamados (PROMPT_JUIZ do esqueleto)
5. Conclusão (+ tabela e matriz final)
```

Regras do esqueleto que precisam ser preservadas:
- A célula de medição é **idêntica** em todas as seções e usa `recupera(pergunta, data)`.
- Células `SUA VEZ` são as suas; as de análise em markdown vêm logo depois de cada medição.
- O notebook roda do início ao fim sem célula manual.

---

## 7. Código-base com Gemini

> Os nomes de modelo abaixo são exemplos. Antes de fixar, liste os modelos disponíveis na sua conta (`for m in client.models.list(): print(m.name)`) e registre o nome exato no notebook. Escolha um modelo **Flash** (rápido e barato) para geração, juiz e RAGAS; o mesmo em tudo.

### 7.1 Instalação e cliente
```python
!pip install -q google-genai sentence-transformers rank_bm25 ragas langchain-google-genai pandas

import os
from google import genai
from google.genai import types

def _chave_gemini():
    try:
        from google.colab import userdata          # Colab Secrets: sem intervenção manual
        k = userdata.get("GOOGLE_API_KEY")
        if k: return k
    except Exception:
        pass
    if os.environ.get("GOOGLE_API_KEY"):
        return os.environ["GOOGLE_API_KEY"]
    from getpass import getpass                    # último recurso
    return getpass("Chave da API do Gemini: ")

os.environ["GOOGLE_API_KEY"] = _chave_gemini()
cliente = genai.Client(api_key=os.environ["GOOGLE_API_KEY"])

MODEL_PROVIDER = "Gemini"
LLM_MODEL   = "gemini-2.5-flash"        # confirmar com client.models.list() e registrar
EMB_MODEL   = "gemini-embedding-001"
EMB_DIM     = 768
TEMPERATURE = 0.0
MAX_TOKENS  = 2000
```

### 7.2 `get_llm_response` com a mesma assinatura do esqueleto, cache e retry
```python
import json, hashlib, time, pathlib

CACHE_DIR = pathlib.Path("cache"); CACHE_DIR.mkdir(exist_ok=True)

def _hash(*partes):
    return hashlib.sha256(json.dumps(partes, ensure_ascii=False, sort_keys=True).encode()).hexdigest()

def _com_retry(fn, tentativas=6):
    for i in range(tentativas):
        try:
            return fn()
        except Exception as e:                     # 429/503: espera e tenta de novo
            if i == tentativas - 1: raise
            time.sleep(2 ** i)

def get_llm_response(messages, max_tokens=MAX_TOKENS, system=None, json_mode=False):
    """Mesma interface do esqueleto: messages = [{'role': 'user'|'assistant'|'system', 'content': str}]."""
    chave = _hash(LLM_MODEL, TEMPERATURE, max_tokens, system, json_mode, messages)
    arq = CACHE_DIR / f"llm_{chave}.json"
    if arq.exists():
        return json.loads(arq.read_text())["texto"]

    contents = []
    for m in messages:
        if m["role"] == "system":
            system = (system + "\n" if system else "") + m["content"]
            continue
        role = "model" if m["role"] == "assistant" else "user"
        contents.append(types.Content(role=role, parts=[types.Part(text=m["content"])]))

    cfg = types.GenerateContentConfig(
        temperature=TEMPERATURE,
        max_output_tokens=max_tokens,
        system_instruction=system,
        response_mime_type="application/json" if json_mode else "text/plain",
        # Nos modelos 2.5 Flash o "thinking" gasta tokens de saída; budget 0 desliga.
        # Em outras famílias o parâmetro pode ser diferente: conferir a documentação.
        thinking_config=types.ThinkingConfig(thinking_budget=0),
    )
    resp = _com_retry(lambda: cliente.models.generate_content(model=LLM_MODEL, contents=contents, config=cfg))
    texto = resp.text or ""
    arq.write_text(json.dumps({"texto": texto}, ensure_ascii=False))
    return texto
```

### 7.3 Embeddings com cache
```python
import numpy as np

def embed(textos, task="RETRIEVAL_DOCUMENT"):
    """task: RETRIEVAL_DOCUMENT para chunks, RETRIEVAL_QUERY para perguntas."""
    arq = CACHE_DIR / f"emb_{EMB_MODEL}_{EMB_DIM}_{task}.json"
    cache = json.loads(arq.read_text()) if arq.exists() else {}
    faltam = [t for t in dict.fromkeys(textos) if _hash(t) not in cache]
    for i in range(0, len(faltam), 100):                 # lotes de até 100
        lote = faltam[i:i+100]
        r = _com_retry(lambda: cliente.models.embed_content(
            model=EMB_MODEL, contents=lote,
            config=types.EmbedContentConfig(task_type=task, output_dimensionality=EMB_DIM)))
        for t, e in zip(lote, r.embeddings):
            cache[_hash(t)] = e.values
    arq.write_text(json.dumps(cache))
    M = np.array([cache[_hash(t)] for t in textos], dtype=np.float32)
    return M / np.linalg.norm(M, axis=1, keepdims=True)  # normaliza (necessário com dim < 3072)
```

**Alternativa sem API para embeddings:** `sentence-transformers` com um modelo multilingue (ex.: `intfloat/multilingual-e5-base`, usando os prefixos `"query: "` e `"passage: "`). Vantagens: grátis, sem limite de taxa, totalmente reprodutível. Desvantagem: talvez um pouco pior em português. **Escolha um e não troque mais** (regra do enunciado).

### 7.4 RAGAS com Gemini
```python
from langchain_google_genai import ChatGoogleGenerativeAI, GoogleGenerativeAIEmbeddings
from ragas.llms import LangchainLLMWrapper
from ragas.embeddings import LangchainEmbeddingsWrapper
from ragas import evaluate, EvaluationDataset
from ragas.metrics import Faithfulness, LLMContextPrecisionWithReference, ResponseRelevancy

avaliador_llm = LangchainLLMWrapper(ChatGoogleGenerativeAI(model=LLM_MODEL, temperature=0))
avaliador_emb = LangchainEmbeddingsWrapper(GoogleGenerativeAIEmbeddings(model=f"models/{EMB_MODEL}"))

def avalia_ragas(amostra, recupera_fn, gera_fn):
    linhas = []
    for p in amostra:
        trechos = recupera_fn(p["pergunta"], p["data_referencia"])
        linhas.append({
            "user_input": p["pergunta"],
            "retrieved_contexts": trechos,
            "response": gera_fn(p["pergunta"], trechos, p["data_referencia"]),
            "reference": p["resposta_referencia"],
        })
    ds = EvaluationDataset.from_list(linhas)
    return evaluate(ds, metrics=[Faithfulness(), LLMContextPrecisionWithReference(), ResponseRelevancy()],
                    llm=avaliador_llm, embeddings=avaliador_emb).to_pandas()
```
> A API do RAGAS muda entre versões (os nomes acima são da linha 0.2.x). Se der erro de importação, peça à IA para ajustar à versão instalada e **fixe a versão** no `pip install`.

### 7.5 Juiz dos 200 chamados
Usar `get_llm_response([{"role": "user", "content": PROMPT_JUIZ.format(texto=..., resposta=..., docs=...)}], json_mode=True)` e `json.loads` no resultado, com tratamento para JSON inválido (contar e reportar quantos falharam).

---

## 8. Modelos de registro e de análise

### 8.1 Tabela de registro (atualizar a cada configuração)
| Config | Mudança | Modelo emb. | LLM | Temp. | Top-k | Chunk | Hit@k /62 | Hit@1 | MRR | Fonte certa | Ganhou | Perdeu |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| Base | – | | | 0 | | fixo 800/100 | | | | | – | – |
| C1 | filtro de vigência | = | = | 0 | = | = | | | | | | |

### 8.2 Modelo de análise (markdown depois de cada medição)
```
**Análise da configuração N.**
- Hipótese: [o que esperava e por quê]
- Resultado: X/62 (antes Y/62). Ganhou: [ids]. Perdeu: [ids].
- Fonte dos acertos: Z vieram de documento vigente e de público cliente (antes W).
- Causa investigada: [ao inspecionar qNN, o trecho certo estava na posição 7 / foi cortado no meio /
  o top-k foi tomado por 'ven-trocas' / a tabela ficou separada do cabeçalho ...]
- A hipótese se confirmou? [sim/parcialmente/não, e por quê]
- Decisão: [manter / descartar / ajustar parâmetro] e [próxima mudança], porque [evidência].
```

### 8.3 Frases que mostram raciocínio (use como guia)
- "O ganho de N perguntas veio todo de perguntas históricas, o que confirma que o problema era vigência e não ranking."
- "A configuração piorou em qXX porque o chunk que continha a resposta agora vem com outra seção que a dilui. Decidimos..."
- "Esse acerto é falso: a string '30 dias' veio da linha de livro, não de brinquedo. Contando só acertos de fonte certa, o número é..."
- "O RAGAS mostra fidelidade alta e precisão de contexto baixa na linha de base: o modelo foi fiel a trechos errados. O problema era a recuperação."

---

## 9. Armadilhas e cuidados

1. **Não apague os revogados.** Eles respondem perguntas históricas. Use a vigência.
2. **Não mude o modelo de embedding nem o LLM no meio.** Se precisar trocar, refaça tudo desde a linha de base.
3. **Mesmo top-k em tudo.** Se o re-ranking busca 20 candidatos, ele devolve o mesmo k das outras.
4. **Filtro de vigência é pré-filtro.** Pós-filtrar o top-k devolve menos trechos e distorce a comparação.
5. **Parent-document e k maior inflam a métrica.** Registre o tamanho médio do contexto ao lado dos acertos.
6. **Mudança de prompt não aparece nas 62.** Meça no RAGAS e nos chamados, como mudança separada.
7. **A classe do chamado não existe nas 62 perguntas.** Uma técnica baseada em `classe` só pode ser medida nos 200 chamados; diga isso na análise.
8. **Juiz LLM erra.** Faça a checagem manual de uma amostra.
9. **Determinismo:** temperatura 0 reduz, mas não elimina, a variação no Gemini. O cache garante que o notebook entregue mostre sempre as mesmas saídas.
10. **Sem intervenção manual:** chave via Secrets; nada de `input()`; nada de montar o Drive no meio (dados por URL, como no esqueleto).
11. **Custo e limite de taxa:** use cache e retry; rode primeiro em 10 chamados, depois nos 200.
12. **Data de compra × data de abertura:** o enunciado manda usar a data do chamado (abertura). Registre os 13 casos limítrofes como limitação.

---

## Decisões tomadas (e por quê)

Estas são as escolhas do plano. Cada uma tem um motivo que pode ir para o notebook como justificativa.

| Decisão | Escolha | Por quê |
|---|---|---|
| Trilha | **B** (C1 vigência → C2 curadoria → C3 chunking por estrutura + metadados → C4 híbrida → C5 re-ranking) | cobre exatamente as técnicas que você queria estudar (chunking/metadados/filtro, híbrida, re-ranking), passa do mínimo de 3 configurações e cada passo ataca um problema identificado na exploração |
| Embedding | **`intfloat/multilingual-e5-base`, local (sentence-transformers), 768 dimensões, com prefixos `query: `/`passage: `** (trocado em 2026-11-13, antes de qualquer melhoria; a primeira escolha foi o `gemini-embedding-001`) | o plano gratuito do Gemini limita a 100 textos de embedding por minuto, o que travou a indexação e tornava o notebook frágil (o Colab desconecta). O modelo local não tem limite de taxa, reproduz os mesmos vetores e a base é pequena (428 trechos na linha de base). O Gemini segue como LLM e avaliador do RAGAS |
| Re-ranking | **cross-encoder local `cross-encoder/mmarco-mMiniLMv2-L12-H384-v1`** | multilingue, pequeno e rápido em CPU no Colab, determinístico e sem custo de API. Usar o Gemini como reranker custaria uma chamada por pergunta por configuração |
| Parent-document | **não** na trilha principal | depois do chunking por seção (C3), o chunk já é praticamente a seção-pai; o ganho esperado é pequeno e a métrica fica inflada. Vai para "próximos passos" na conclusão |
| Experimento da curadoria ingênua | **sim**, como variante isolada (fora da cadeia) | não custa nada (não chama o LLM) e mostra, com números, por que não se deve apagar os revogados. É o tipo de "causa investigada" que o professor valoriza |
| Amostra do RAGAS | **20 perguntas**, fixa (`seed=42`), estratificada | equilibra custo e representatividade; a estratificação garante perguntas históricas, de tabela e de comunicado na amostra |
| Ablação final | **sim** (C5 sem híbrida; C5 sem re-ranking) | não custa nada (só recuperação) e separa a contribuição de duas técnicas que atacam o mesmo sintoma |
| Top-k | decidir na Etapa 4 (provável k = 5) | depende do número medido; a regra é usar o mesmo k em todas as configurações |

### Glossário rápido (para estudar)
- **Chunk:** pedaço de documento que vira uma unidade de busca.
- **Embedding:** vetor que representa o significado de um texto; textos parecidos têm vetores próximos (similaridade por cosseno).
- **Top-k:** quantos trechos a busca devolve para o LLM.
- **Metadados:** informações sobre o chunk (documento, público, vigência, seção) usadas para filtrar ou dar contexto.
- **Filtro de vigência:** só deixa entrar na busca trechos de documentos válidos na data do chamado.
- **Curadoria:** decidir o que entra na base (tirar o que não é para o cliente, corrigir erros, remover repetição).
- **BM25:** busca por palavras exatas (como um buscador clássico); boa para códigos ("COM 2026 019") e valores.
- **Busca híbrida:** junta a busca por significado (embedding) com a busca por palavras (BM25). O RRF é a fórmula que combina as duas listas pela posição de cada trecho.
- **Re-ranking:** busca muitos candidatos (ex.: 20) e usa um modelo mais preciso (cross-encoder), que lê pergunta e trecho juntos, para reordenar e ficar com os k melhores.
- **Parent-document:** busca em trechos pequenos e devolve o trecho maior que os contém.
- **Ablação:** tirar uma peça do sistema final para ver quanto ela contribuía.
- **RAGAS:** biblioteca que usa um LLM para dar notas: *faithfulness* (a resposta se apoia nos trechos?), *context precision* (os trechos recuperados são os relevantes?), *answer relevancy* (a resposta responde à pergunta?).
