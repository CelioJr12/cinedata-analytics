# CineData Analytics

Pipeline de dados end-to-end no Databricks (PySpark/SQL), seguindo a Arquitetura Medalhão
(Bronze → Silver → Gold), com modelagem dimensional em Star Schema, uma tabela de contexto para
assistente de IA (RAG) e orquestração via Databricks Workflows.

## Estrutura

- `notebooks/Landing_to_Bronze.ipynb` — ingestão dos 5 CSVs de origem + cotação do dólar (API Banco Central)
- `notebooks/Bronze_to_Silver.ipynb` — limpeza, tipagem e regras de qualidade de dados (7 tabelas)
- `notebooks/Silver_to_Gold.ipynb` — Star Schema, tabela de contexto GenAI e Desafio de Analytics
- `job.yaml` — definição do Job de orquestração (Bronze → Silver → Gold, com agendamento)
- `print_execucao_job.png` — print da execução de sucesso do Job

## Resultados do Desafio de Analytics

| # | Pergunta de negócio | Resultado |
|---|---|---|
| 1 | Receita total (R$) de todos os filmes da base | R$ 834.732.810.004,24 |
| 2 | Top 5 filmes por popularidade | Blue Beetle, Gran Turismo, The Nun II, Meg 2: The Trench, Retribution |
| 3 | Filmes por gênero (top 3 do ranking completo) | Drama (32.287), Documentary (18.996), Comedy (18.625) |
| 4 | Filme de maior receita (1º do ranking dos 10 maiores) | Avengers: Endgame — US$ 2.800.000.000,00 / R$ 14.439.320.000,00 |
| 5 | Ator com mais participações (últimos 2 anos) | Kevin Hart (66 participações) |
| 6 | Produtora com maior lucro (últimos 5 anos) | Universal Pictures (US$ 5.772.329.679,00) |

## Principais dificuldades e decisões técnicas

- **Duplicatas na Bronze:** o modo `append` na ingestão causou registros duplicados por `id_filme`
  em algumas tabelas. Solução: deduplicação na Silver via `row_number()` particionado por
  `id_filme`, mantendo sempre o registro mais recente por `ingestion_datetime`.
- **Delimitador real dos gêneros:** o enunciado sugeria vírgula/ponto e vírgula, mas o delimitador
  verdadeiro na fonte era `|`. Identificado analisando os valores distintos após o split, que vinham
  cheios de lixo não separado.
- **Sujeira por Column Shift:** textos de sinopse, tags e caminhos de imagem vazavam para colunas
  numéricas (elenco, produtoras, popularidade). Tratado com filtros de conteúdo (letras, extensões
  de imagem, comprimento do texto) em vez de apenas tentar converter o tipo.
- **Popularidade corrompida:** parte dos valores de popularidade eram, na verdade, sinopses ou anos
  de lançamento vazados. Regra criada: valor com letra vira NULL; número inteiro "redondo" dentro da
  faixa de anos plausíveis (1870–2030) também vira NULL, por ser mais provável ser um ano vazado do
  que uma popularidade real.
- **Sufixos K/M/B no orçamento:** valores como `"34.0M"` (34 milhões) eram convertidos incorretamente
  para `34`. Criada uma detecção do sufixo antes da limpeza numérica, aplicando o multiplicador certo.
- **Nulos na tabela de contexto (GenAI):** `concat()` zera a frase inteira se qualquer campo for
  NULL. Resolvido tratando cada campo individualmente com `coalesce()` (textos de fallback) antes da
  concatenação final, garantindo que nenhum filme desaparecesse da tabela.
- **Modelagem de `dim_people`:** o papel da pessoa (Ator/Diretor/Roteirista) foi modelado como
  atributo da própria dimensão (não da tabela ponte), então uma mesma pessoa com múltiplos papéis
  gera mais de uma linha — decisão alinhada à especificação do projeto.
- **Chaves substitutas:** geradas com `row_number()` sobre janelas ordenadas, desacoplando o modelo
  Gold das chaves naturais da origem, como recomendado pela modelagem dimensional (Star Schema).
