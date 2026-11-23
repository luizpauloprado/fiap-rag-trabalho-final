# Entrega final: base de conhecimento do atendimento da Quantum Commerce

Knowledge Management and Prompt Engineering. MBA AI Engineering & Multi-Agents (FIAP).

**Notebook:** `Projeto_Final_QuantumCommerce.ipynb` (roda do início ao fim, com todas as saídas salvas).

**Alunos e turma:**
- Turma: 4AIER
- Luiz Paulo Prado - RM377637
- Renato Romão - RM377019
- Diego Moreira - RM376480
- Orlando Valverde - RM376801

---

## 1. O problema

O assistente de primeira resposta usava versões antigas das políticas, misturava regras de vendedor com regras de cliente, inventava regras e respondia mesmo sem informação na base. O trabalho foi melhorar a base e a recuperação, medindo cada mudança contra a linha de base.

## 2. Configuração fixa em todas as medições

| Item | Valor |
|---|---|
| LLM (geração, juiz e RAGAS) | `gemini-3.5-flash`, temperatura 0, thinking desligado |
| Embedding | `intfloat/multilingual-e5-base` (local, 768 dim) |
| Re-ranker (só na C5) | `cross-encoder/mmarco-mMiniLMv2-L12-H384-v1` (local) |
| top-k | 5 em todas as configurações |
| Seed | 42 |
| Pacotes | google-genai 3.6.1, sentence-transformers 6.3.0, rank-bm25 0.2.2, ragas 0.4.4 |

**Por que k = 5.** Na linha de base, de k = 3 para 5 entram 7 perguntas. De 5 para 10 entram só 4, uma delas por sorte, e o contexto dobra (de ~860 para ~1.760 tokens). A fração de trechos fora da vigência fica em 37% a 40% em qualquer k, então um k maior só põe mais trecho errado na frente do modelo.

**Embedding.** Começamos com o embedding do Gemini, mas o limite do plano gratuito (100 textos por minuto) travava a indexação. Trocamos pelo e5 local antes de medir qualquer melhoria.

## 3. O que a exploração dos dados mostrou

- Só 23 dos 44 documentos são de cliente. Vendedor, institucional e editorial somam 31% do texto e não deveriam responder a um cliente.
- Os 5 documentos revogados são 27% do texto, mas **precisam ficar**: 5 das 6 perguntas históricas só se respondem por eles, e 134 dos 200 chamados foram abertos quando valia a POL DEV 03 v2.
- A vigência confiável está no manifesto, não no texto. A `rev-dev-v1` e a `rev-dev-v2` trazem no cabeçalho a frase copiada da v3 ("Vigente a partir de 01/07/2026").
- 11 respostas de referência aparecem em 5 ou mais documentos ("retido", "5 dias úteis"), então dá para acertar por sorte. Por isso medimos também **fonte certa** (trecho de cliente ou revogado, vigente na data).
- A categoria informada pelo cliente concorda com o classificador em só 66% dos casos, e as tags do sistema não indicam a classe. Usamos a `classe`.
- Nada na base confirma que "premium" seja assinante do Clube Q+. O prompt não assume isso.
- Os 20 chamados `outros` são 10 temas (cobrança em duplicidade, boleto, bivolt, cupom...) sem nenhuma cobertura na base. São o teste natural da abstenção.

## 4. Caminho percorrido (uma mudança por vez)

Métricas nas 62 perguntas de referência, k = 5. **Hit@5** é a métrica oficial do enunciado.

| Configuração | Mudança | Hit@5 | Fonte certa | Hit@1 | MRR |
|---|---|---|---|---|---|
| Linha de base | chunk fixo 800/100, similaridade, sem data | 50 | 48 | 32 | 0,618 |
| C1 | + filtro de vigência | 52 | 51 | 38 | 0,699 |
| C2 | + curadoria por público e correção de versão | 53 | 53 | 42 | 0,744 |
| C3 | + chunk por seção com cabeçalho (título, seção, público, vigência) | 58 | 58 | 39 | 0,747 |
| C4 | + busca híbrida (BM25 + densa, RRF) | 57 | 57 | 43 | 0,792 |
| **C5** | **+ re-ranking com cross-encoder** | **59** | **59** | **47** | **0,819** |

O que cada resultado mostrou e qual decisão motivou:

**Linha de base (50/62).** 38% dos trechos recuperados eram de documento fora da vigência, e em 20 perguntas o 1º trecho estava fora da vigência. Em 11 das 12 erradas havia trecho revogado ocupando vaga. *Decisão:* começar pelo filtro de vigência.

**C1, vigência (52/62).** Trechos fora da vigência no contexto: de 117 para 0. Hit@1 +6. Nenhuma pergunta perdida. As históricas q47 e q49 subiram de 5º para 1º, e q19 e q22 passaram a acertar pela versão certa (antes acertavam com o texto da política atual numa pergunta de 2025). Efeito colateral: as vagas liberadas foram para vendedor e editorial, que não têm vigência e passam pelo filtro. *Decisão:* manter e fazer a curadoria por público.

**C2, curadoria (53/62).** Trechos de outro público no contexto: de 58 para 0. Acertos por sorte: 0. A remoção das seções "Reafirmação" piorou o resultado, mas porque com chunk fixo qualquer edição desloca todos os cortes seguintes do documento (só 6 de 27 chunks da POL DEV 03 ficaram iguais). *Decisão:* manter público + correção de versão e levar a remoção das reafirmações para a C3.

**C3, chunk por estrutura (58/62).** O maior salto. As tabelas viraram um chunk inteiro (q12, q41 e q48 entram; a q48 sobe de 28º para 4º). O cabeçalho é o que mais pesa: põe o código do documento em todo trecho dele. Com chunk por seção, a remoção das reafirmações não teve efeito, o que confirma que a perda na C2 era do chunk fixo. FAQ em blocos ficou melhor que FAQ em pares (pares de FAQ "parecidos" com a pergunta tomavam o 1º lugar), ao custo de +18% de contexto. *Decisão:* seguir com a C3 e atacar as perguntas que citam código ("COM 2026 019") com busca por palavra.

**C4, híbrida (57/62).** Hit@5 caiu 1, mas o MRR subiu de 0,747 para 0,792. O BM25 resolveu as perguntas com código (q23, q52, q53). Perdemos q07, q44 e q62 por causa do RRF: um trecho em 1º só na densa vale menos que um trecho em 10º nas duas listas. Em compensação, o trecho certo ficou entre os 20 candidatos em 62 de 62 perguntas. *Decisão:* manter e reordenar os candidatos com re-ranking.

**C5, re-ranking (59/62).** Recuperou as perdas da C4 (q07, q62) e trouxe q51 e q64 para 1º. Perdeu q39 e q40. *Decisão:* C5 é a configuração final.

**Variante: curadoria ingênua (apagar os revogados).** Fora da cadeia, para medir uma decisão tentadora e errada. Nas 56 perguntas atuais, resultado idêntico ao da C2 (o filtro já tirava os revogados). Nas históricas, perdeu q19, q22, q47 e q49; três delas ficaram com **zero trechos**. Nos chamados seria pior: os 134 abertos no 1º semestre ficariam sem nenhuma política de devolução. **Revogado não é lixo, é a política de outra data.** Quem decide qual versão vale é o filtro.

**Ablação da C5.**

| Variante | Hit@5 | Hit@1 | MRR |
|---|---|---|---|
| C5 completa | 59 | 47 | 0,819 |
| sem híbrida | 58 | 46 | 0,803 |
| sem re-ranking (= C4) | 57 | 43 | 0,792 |
| sem as duas (= C3) | 58 | 39 | 0,747 |

O re-ranking é a peça que mais contribui (+4 no Hit@1). A híbrida contribui com uma pergunta só (q51, código de comunicado), mas custa quase nada. Mantivemos as duas.

## 5. Configuração escolhida e por quê

**C5 + prompt novo:** filtro de vigência, base só com documentos de cliente e revogados, chunk por seção com cabeçalho, busca híbrida com 20 candidatos e re-ranking para os 5 finais.

- Melhor Hit@5 (59 de 62), melhor Hit@1 (47) e melhor MRR (0,819).
- Todos os acertos de fonte certa. Nenhum trecho fora da vigência ou de outro público chega ao modelo.
- Melhor precisão do contexto no RAGAS.

**Prompt novo:** data do chamado, tipo de cliente, categoria e faixa de valor já calculada em código; citação do código do documento; comunicado mais recente prevalece; frase fixa de abstenção quando nada é coberto; não presume que premium é assinante do Clube.

**Ressalva:** contra a C3, a vantagem é de uma pergunta no Hit@5. A escolha se apoia na ordem dos trechos (mais acertos em 1º), não numa diferença clara na qualidade das respostas.

## 6. RAGAS

Amostra fixa de 20 perguntas (seed 42), mesmo Gemini como avaliador. Rodamos quatro configurações por limite de cota.

| Rodada | Fidelidade | Precisão do contexto | Relevância da resposta |
|---|---|---|---|
| Linha de base | 0,898 | 0,581 | 0,786 |
| C3 | **0,963** | 0,700 | **0,930** |
| C5 (prompt antigo) | 0,865 | **0,784** | 0,918 |
| C5 + prompt novo | 0,869 | 0,784 | 0,916 |

- **Precisão do contexto** segue a mesma ordem do MRR. C5 contra a linha de base: +0,20 (IC95 +0,04 a +0,37). C5 contra C3: +0,08, sem significância com 20 perguntas.
- **Relevância** só separa quem não respondeu: a linha de base respondeu "não há informação" em q19 e q49, justamente as duas históricas da amostra.
- **Fidelidade da C5 abaixo da C3:** abrimos as 484 afirmações julgadas. 12 das 21 reprovações da C5 são frases que citam "Quantum Commerce", nome que está no prompt e não nos trechos. Sem elas: C3 0,968, C5 0,923. O resto é o avaliador quebrando a resposta e perdendo qualificadores (no q19, a mesma resposta certa tirou 1,0 na C3 e 0,33 na C5).
- **Prompt novo × antigo empatam** na amostra. As 20 perguntas não têm valor, categoria nem tipo de cliente, que é onde o prompt novo atua.
- **Onde está o ganho:** na recuperação. A geração ficou praticamente igual entre as rodadas.
- O RAGAS pegou um acerto falso da métrica oficial: no q12 a C5 conta como acerto porque "5 dias úteis" aparece no 5º trecho, mas o trecho é da POL AUT 02, não da tabela de faixas.

## 7. Rodada nos 200 chamados (C5 + prompt novo)

Consulta = "Assunto: <classe>. <texto da reclamação>". Data de referência = data de abertura. Juiz com o prompt fornecido, sem alteração.

Acrescentar a classe na consulta foi medido antes: a política certa no top-5 subiu de 80,6% para 99,4% dos chamados.

| Classe | Desfecho | Parâmetro certo¹ | Inventou | Abstenção |
|---|---|---|---|---|
| **Total (200)** | **97,0%** | **54,9%** (50 de 91) | **6,0%** (12) | **25,0%** (50) |
| artigo_proibido (30) | 100% | n/a | 0% | 0% |
| devolucao_ressarcimento (30) | 93,3% | 53,8% | 13,3% | 43,3% |
| divergente_defeito (30) | 100% | **0%** (0 de 10) | 3,3% | 16,7% |
| entrega (30) | 96,7% | 75,9% | 10,0% | 13,3% |
| falsificado (30) | 100% | 55,6% | 10,0% | 20,0% |
| fraude_vendedor (30) | 100% | 66,7% | 3,3% | 6,7% |
| outros (20) | 85,0% | 40,0% | 0% | **100%** |

¹ Só sobre os chamados em que o juiz disse haver prazo ou valor aplicável.

Premium e padrão ficaram praticamente iguais.

## 8. Onde o sistema ainda erra

A maior parte dos erros está na **recuperação**. A geração erra pouco.

**Recuperação**
- **O documento certo vem, a seção com o número não.** A política certa aparece em 99,4% dos chamados, mas a seção com o parâmetro aparece em só 10 de 30 chamados de devolução e em 0 de 30 de defeito (tabela de garantia).
- **Seções genéricas ocupam as vagas.** Definições, Objetivo e Âmbito e Disposições Gerais repetem o assunto da política e casam com o "Assunto: ..." da consulta (média de 1,67 por chamado de devolução).
- **O re-ranking rebaixa tabelas.** A tabela de garantia, quando entra nos 20 candidatos, vai para a 20ª posição depois do cross-encoder, mesmo quando estava em 1º na híbrida. Mesmo padrão do q12 e do q40.
- **Consequência:** "parâmetro certo" em 55% e abstenção desnecessária em 43% dos chamados de devolução. O modelo obedece o prompt (não inventa e encaminha), mas a tabela não chegou.
- Nas 62 perguntas, seguem erradas q39 (blocos de FAQ na frente), q40 (encaixar R$ 350 numa faixa da tabela) e q44 ("conta de streaming" contra "assinaturas digitais").

**Geração**
- Abstenção correta nos 20 chamados `outros`, sem nenhuma invenção.
- Invenção em 12 chamados (6%), de três tipos: erro de regra (R00151 conta o arrependimento a partir da compra, não do recebimento), desobediência ao prompt (R00103 cita o Clube Q+ sem trecho que justifique) e completar o que a recuperação não trouxe (R00166).
- Frases de encaminhamento que prometem verificações que os documentos não descrevem.

## 9. Limitações

- As escolhas finas (FAQ em blocos, híbrida, k) foram feitas nas mesmas 62 perguntas. Os 200 chamados servem de validação, mas não dá para descartar algum ajuste ao conjunto de teste.
- 56 das 62 perguntas são da mesma data. O conjunto quase não pune erros de vigência (a curadoria ingênua perde só 4).
- RAGAS com 20 perguntas, só 2 históricas. Uma pergunta move a média em até 0,05.
- Juiz e gerador são o mesmo modelo, e o juiz não justifica os vereditos. O número de "inventou" é o menos confiável dos quatro.
- A métrica oficial (substring nos trechos) não vê se o trecho realmente responde. O q12 é um acerto falso que ela não pega.

## 10. Próximos passos

1. **Tirar do índice as seções genéricas** (76 de 313 chunks). Já medido sem LLM: a seção do parâmetro em devolução passa de 10 para 23 de 30, e as 62 perguntas ficam em 59, com MRR 0,823.
2. **Consultar as tabelas pelos metadados do chamado.** Garantia pela categoria; faixa de reembolso pelo valor e pela versão vigente na data. A linha vai direto para o gerador, como já fazemos com a faixa de valor. É o que pode tirar divergente_defeito do zero.
3. **Proteger tabelas no re-ranking** (manter o 1º da híbrida) ou testar outro re-ranker.
4. **Ajustar o prompt:** encaminhamento genérico, prazo de arrependimento contado do recebimento, tirar o nome da empresa da resposta.
5. **Melhorar a avaliação:** juiz de outro modelo com justificativa, conferência manual dos 12 casos de "inventou", RAGAS com mais históricas e métrica de recuperação por seção, não só por documento.
