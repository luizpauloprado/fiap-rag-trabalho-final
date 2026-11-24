# Quantum Commerce: base de conhecimento para um assistente RAG

Projeto final da disciplina **Knowledge Management and Prompt Engineering (FIAP)**. A Quantum Commerce usa um assistente com RAG para dar a primeira resposta a reclamações de clientes. Este projeto melhora a **base de conhecimento** e a **recuperação** desse assistente, com experimentos controlados: uma mudança por vez, medida contra a linha de base.

> **Resumo da entrega:** [`entrega-final.md`](entrega-final.md) traz a configuração escolhida, os números das 62 perguntas, do RAGAS e dos 200 chamados, onde o sistema ainda erra e os próximos passos. O caminho completo, com a análise de cada medição, está no notebook.

## Objetivos

- Usar a política **vigente na data do chamado**.
- Recuperar o **prazo e o valor corretos**.
- Usar **só regras aplicáveis ao cliente** (sem regras de vendedor).
- **Reconhecer quando a base não tem a informação** e não inventar.

## Abordagem e resultado

Recuperação medida nas 62 perguntas de referência, com top-k = 5 em todas as configurações.

| Etapa | O que muda | Hit@5 | Hit@1 | MRR |
|---|---|---|---|---|
| Linha de base | chunks fixos (800/100) nos 44 documentos, similaridade por cosseno, sem considerar data | 50 | 32 | 0,618 |
| C1 | filtro de vigência por metadado | 52 | 38 | 0,699 |
| C2 | curadoria por público (sai vendedor, institucional e blog; os revogados ficam) e correção de versão | 53 | 42 | 0,744 |
| C3 | chunking por estrutura (seções, tabelas, FAQ) com cabeçalho contextual | 58 | 39 | 0,747 |
| C4 | busca híbrida BM25 + densa, fundida por RRF | 57 | 43 | 0,792 |
| **C5** | **re-ranking com cross-encoder (configuração final)** | **59** | **47** | **0,819** |

Também tem: variante de "curadoria ingênua" (apaga os revogados, para mostrar a perda), ablação da busca, ajuste do prompt com abstenção, avaliação **RAGAS** em quatro rodadas e rodada final nos **200 chamados** julgada por um LLM.

Na rodada dos 200 chamados (C5 + prompt novo): desfecho correto em 97%, prazo ou valor correto em 55%, invenção em 6% e abstenção em 25% (100% nos chamados `outros`, que a base não cobre). O ponto fraco é a recuperação das tabelas de prazo e valor; os detalhes estão em `entrega-final.md`.

**Stack:** Gemini `gemini-3.5-flash` (temperatura 0) como LLM, juiz e avaliador do RAGAS · embeddings locais `intfloat/multilingual-e5-base` · re-ranker `cross-encoder/mmarco-mMiniLMv2-L12-H384-v1` · `rank_bm25`, `ragas`, `pandas`.

## Arquivos

| Arquivo | O que é |
|---|---|
| `Projeto_Final_QuantumCommerce.ipynb` | **Notebook principal** (a entrega), com todas as saídas salvas: configuração, exploração, linha de base, melhorias, RAGAS, 200 chamados e conclusão |
| [`entrega-final.md`](entrega-final.md) | **Resumo da entrega**: configuração escolhida e por quê, resultados numéricos, erros que sobraram, limitações e próximos passos |
| `painel_resultados.html` | Painel visual dos resultados (abrir no navegador): comparação das configurações, ablação, RAGAS, 200 chamados e matriz pergunta × configuração |
| `resultados_200_chamados.csv` | Saída da rodada final, gerada pelo notebook: para cada chamado, os documentos recuperados, a resposta do assistente e os quatro critérios do juiz |
| `plano-estudos.md` | Plano de estudos e roteiro do experimento: achados da exploração, técnicas, ordem das configurações e decisões |
| `FIAP_Knowledge_Management_and_Prompt_Eng_Projeto_Final_Esqueleto.ipynb` | Esqueleto original fornecido pelo professor |
| `Enunciado Projeto Final - v3.pdf` | Enunciado do trabalho |
| `central-documentos/` | Base de conhecimento: 44 documentos `.md` e o `manifest.json` (título, público, estrutura e vigência de cada um) |
| `central-documentos.zip` | A mesma central, compactada |
| `chamados-200.csv` | 200 reclamações de clientes para a rodada final (separador `;`, UTF-8) |
| `perguntas-referencia.csv` | 62 perguntas com data e resposta de referência, usadas para medir a recuperação |
| `cache/` | Respostas do LLM, embeddings e notas do re-ranker salvos em disco (gerado na execução, fora do git) |

Prefixos em `central-documentos/`: `pol-*` políticas · `com-*` comunicados · `bol-*` boletins · `rev-*` versões revogadas · `ven-*` regras do vendedor · `ins-*` institucional · `blog-*` editorial.

## Como executar

> [!IMPORTANT]
> É preciso ter uma **chave de API do Gemini**. Gere a sua de graça no Google AI Studio: **https://aistudio.google.com/apikey**

1. Gere a chave no [Google AI Studio](https://aistudio.google.com/apikey).
2. Abra `Projeto_Final_QuantumCommerce.ipynb` no Google Colab.
3. Cadastre a chave com o nome `GOOGLE_API_KEY` em **Secrets** (ícone de chave na barra lateral do Colab) e libere o acesso para o notebook. Rodando localmente, use a variável de ambiente `GOOGLE_API_KEY`.
4. **Ambiente de execução → Executar tudo.**

O notebook baixa os dados pelo link do repositório do professor, então não depende dos CSVs desta pasta.

As respostas do LLM, os embeddings e as notas do re-ranker ficam em cache na pasta `cache/`, então uma segunda execução não repete chamadas à API. A pasta não vai para o git: numa máquina nova, a primeira execução faz todas as chamadas (cerca de 1.000 ao Gemini, somando RAGAS e os 200 chamados). No plano gratuito isso pode esgotar a cota do dia; nesse caso a célula para, mostra o que já foi feito e continua de onde parou na próxima execução. O primeiro uso também baixa o modelo de embedding (cerca de 1 GB) e o re-ranker do Hugging Face.
