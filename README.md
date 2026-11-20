# Quantum Commerce: base de conhecimento para um assistente RAG

Projeto final da disciplina **Knowledge Management and Prompt Engineering (FIAP)**. A Quantum Commerce usa um assistente com RAG para dar a primeira resposta a reclamações de clientes. Este projeto melhora a **base de conhecimento** e a **recuperação** desse assistente, com experimentos controlados: uma mudança por vez, medida contra a linha de base.

## Objetivos

- Usar a política **vigente na data do chamado**.
- Recuperar o **prazo e o valor corretos**.
- Usar **só regras aplicáveis ao cliente** (sem regras de vendedor).
- **Reconhecer quando a base não tem a informação** e não inventar.

## Abordagem

| Etapa | O que muda |
|---|---|
| Linha de base | chunks fixos (800/100) nos 44 documentos, similaridade por cosseno, sem considerar data |
| C1 | filtro de vigência por metadado |
| C2 | curadoria por público (sai vendedor, institucional e blog; os revogados ficam) e correção de conteúdo |
| C3 | chunking por estrutura (seções, tabelas, FAQ) com cabeçalho contextual |
| C4 | busca híbrida BM25 + densa, fundida por RRF |
| C5 | re-ranking com cross-encoder |

Também tem: variante de "curadoria ingênua" (apaga os revogados, para mostrar a perda), ablação da busca, ajuste do prompt com abstenção, avaliação **RAGAS** e rodada final nos **200 chamados** julgada por um LLM. Os resultados e a conclusão estão no próprio notebook (seção 5).

**Stack:** Gemini `gemini-3.5-flash` (temperatura 0) como LLM, juiz e avaliador do RAGAS · embeddings locais `intfloat/multilingual-e5-base` · re-ranker `cross-encoder/mmarco-mMiniLMv2-L12-H384-v1` · `rank_bm25`, `ragas`, `pandas`.

## Arquivos

| Arquivo | O que é |
|---|---|
| `Projeto_Final_QuantumCommerce.ipynb` | **Notebook principal** (a entrega): configuração, exploração, linha de base, melhorias, RAGAS, 200 chamados e conclusão |
| `FIAP_Knowledge_Management_and_Prompt_Eng_Projeto_Final_Esqueleto.ipynb` | Esqueleto original fornecido pelo professor |
| `plano-estudos.md` | Plano de estudos e roteiro do experimento: achados da exploração, técnicas, ordem das configurações e decisões |
| `Enunciado Projeto Final - v3.pdf` | Enunciado do trabalho |
| `central-documentos/` | Base de conhecimento: 44 documentos `.md` e o `manifest.json` (título, público, estrutura e vigência de cada um) |
| `central-documentos.zip` | A mesma central, compactada |
| `chamados-200.csv` | 200 reclamações de clientes para a rodada final (separador `;`, UTF-8) |
| `perguntas-referencia.csv` | 62 perguntas com data e resposta de referência, usadas para medir a recuperação |

Prefixos em `central-documentos/`: `pol-*` políticas · `com-*` comunicados · `bol-*` boletins · `rev-*` versões revogadas · `ven-*` regras do vendedor · `ins-*` institucional · `blog-*` editorial.

## Como executar

> [!IMPORTANT]
> É preciso ter uma **chave de API do Gemini**. Gere a sua de graça no Google AI Studio: **https://aistudio.google.com/apikey**

1. Gere a chave no [Google AI Studio](https://aistudio.google.com/apikey).
2. Abra `Projeto_Final_QuantumCommerce.ipynb` no Google Colab.
3. Cadastre a chave com o nome `GOOGLE_API_KEY` em **Secrets** (ícone de chave na barra lateral do Colab) e libere o acesso para o notebook. Rodando localmente, use a variável de ambiente `GOOGLE_API_KEY`.
4. **Ambiente de execução → Executar tudo.**

As respostas do LLM e os embeddings ficam em cache na pasta `cache/`, então uma segunda execução não repete chamadas à API.
