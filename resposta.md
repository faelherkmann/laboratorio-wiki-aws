# 📝 Resposta do Laboratório: A Wiki Perdida dos Arquivos Corporativos

> Proposta de arquitetura para transformar os documentos brutos da pasta `raw/` em uma Wiki Corporativa Inteligente, pesquisável e segura, usando apenas serviços da AWS.

---

## 👤 Identificação

**Nome:**  
Raphael Herkmann

**Data:**  
04/09/2026

**Link do repositório:**  
https://github.com/faelherkmann/laboratorio-wiki-aws

---

# ✅ Quest 1: O Mapa dos Arquivos Perdidos

## 1.1 Formatos encontrados na pasta `raw/`

Abri os três arquivos antes de propor qualquer coisa. Cada um pede um tratamento diferente.

```md
- ata_reuniao_vendas_sa.pdf: PDF digital (nasce com camada de texto), 5 paginas.
  Da para extrair o texto direto, sem OCR. Tem secoes estruturadas (identificacao,
  participantes, pauta) e tabelas de indicadores (meta x realizado x atingimento).

- ata_resultados_vendas_novos_dados.png: imagem escaneada (so pixels), 1 pagina.
  Precisa de OCR. Tem texto impresso, uma tabela de indicadores, anotacoes a mao
  ("conferir CRM" e "acao prioritaria" circulando prazos) e um carimbo. As
  anotacoes manuscritas sao justamente onde mora a decisao, entao nao basta OCR
  de texto impresso, precisa de deteccao de manuscrito.

- vendas_sa_dados_ficticios_laboratorio.csv: exportacao do CRM, 240 oportunidades
  em 19 colunas (oportunidade_id, data_criacao, data_fechamento, cliente_ficticio,
  segmento, regiao, vendedor_ficticio, origem_lead, produto, campanha, status,
  probabilidade_pct, valor_bruto_brl, desconto_pct, valor_liquido_brl, ciclo_dias,
  motivo_perda, proxima_atividade, observacao). Nao e texto corrido, e tabela.
  Nao passa por OCR nem por chunking ingenuo: pede tratamento de dado estruturado.
```

## 1.2 Principais desafios encontrados

```md
- Formatos heterogeneos na mesma pasta: um digital, um so imagem, um tabular.
  Tratar os tres do mesmo jeito quebra a solucao.
- Nao existem subpastas em raw/. A classificacao precisa vir do conteudo do arquivo,
  nao da localizacao dele.
- O PDF ja tem texto: mandar ele para o OCR seria desperdicio de custo e ainda
  degradaria a qualidade do texto que ja esta limpo.
- A imagem tem manuscrito. OCR comum ignora anotacao a mao, e aqui a anotacao
  ("acao prioritaria", "conferir CRM") carrega contexto de decisao.
- A imagem tem tabela e carimbo. Extrair a tabela como texto solto perde a relacao
  entre indicador, meta e realizado.
- O CSV tem 240 linhas e 19 colunas. Jogar isso cru num modelo de linguagem estoura
  contexto e nao vira busca semantica util. Precisa virar fatos consultaveis.
- Rastreabilidade: qualquer resposta da Wiki precisa apontar de qual documento e de
  qual trecho saiu a informacao, senao vira palpite.
- Dado sensivel: nomes, valores de negocio e classificacao de confidencialidade
  exigem controle de acesso e criptografia.
```

## 1.3 Informações importantes a serem extraídas

```md
Das atas (PDF e imagem):
- Data e codigo da reuniao
- Participantes e funcoes
- Objetivo e pauta
- Indicadores (meta, realizado, atingimento, variacao)
- Decisoes e deliberacoes
- Responsaveis por cada acao
- Prazos
- Riscos e observacoes
- Proximos passos e pendencias
- Anotacoes manuscritas (marcacoes de prioridade)
- Nivel de confidencialidade

Do CSV do CRM:
- Metricas agregadas por regiao, segmento, produto, campanha e vendedor
- Funil por status e probabilidade
- Valor bruto, desconto e valor liquido
- Ciclo medio de fechamento
- Principais motivos de perda
- Proximas atividades em aberto
```

## 1.4 Estratégia de classificação inicial

Como tudo cai solto em `raw/`, a classificação nasce do arquivo, não da pasta.

```md
1. Deteccao de tipo tecnico: uma Lambda disparada a cada objeto novo no S3 le a
   extensao e faz "content sniffing" (magic bytes) para confirmar o formato real.
   Roteia em tres trilhas: PDF, imagem, CSV.

2. Sub deteccao no PDF: a Lambda verifica se existe camada de texto embutida.
   - Com texto: extrai direto (sem OCR).
   - Sem texto (PDF que na verdade e digitalizado): manda para o Amazon Textract.

3. Classificacao de negocio: apos extrair o texto, o Amazon Bedrock classifica cada
   documento por tipo (ata, relatorio, base CRM), area (comercial, operacoes) e nivel
   de confidencialidade (publico, interno, restrito).

4. Marcacao: o resultado vira tags de objeto no S3 (S3 Object Tagging) e um registro
   no DynamoDB. Assim a "organizacao" existe em metadado, sem nunca mover ou renomear
   nada dentro de raw/.
```

---

# ✅ Quest 2: O Portal de Entrada na AWS

## 2.1 Armazenamento dos arquivos brutos

```md
Amazon S3 e a porta de entrada e o repositorio central. Proponho um data lake com
buckets separados por estagio, para nunca misturar bruto com processado:

- s3://wiki-raw/         arquivos originais, imutaveis (espelha a pasta raw/)
- s3://wiki-processed/   texto extraido e normalizado (.txt / .json)
- s3://wiki-metadata/    metadados e enriquecimento
- s3://wiki-curated/     conteudo pronto para indexacao (fonte da Knowledge Base)

Envio dos arquivos: via Console (poucos arquivos), AWS CLI (aws s3 sync) ou AWS
DataSync para volumes maiores e cargas recorrentes.

Governanca do bucket raw:
- AWS KMS para criptografia em repouso (SSE-KMS).
- S3 Versioning ligado, para manter historico e permitir rastreio por versao.
- S3 Lifecycle para mover originais antigos para classes mais baratas
  (S3 Standard-IA e depois Glacier), sem perder o dado.
- AWS IAM com politicas de menor privilegio: o pipeline le raw, mas so escreve em
  processed, metadata e curated.
```

## 2.2 Preservação dos arquivos originais

A regra do desafio é não alterar `raw/`. A arquitetura garante isso por design, não por promessa.

```md
- Bucket raw separado, com S3 Object Lock em modo WORM (write once read many), o que
  impede sobrescrever ou apagar o original dentro do periodo de retencao.
- S3 Versioning para preservar o historico caso algo seja reenviado.
- Todo processamento le do raw e escreve em outro bucket. Nada volta para o raw.
- Cada documento processado guarda o ponteiro completo para a origem: bucket, chave
  do objeto e versionId. Isso amarra o dado tratado ao original exato que o gerou.
- Checksum (ETag / SHA-256) registrado no DynamoDB para provar integridade.
```

## 2.3 Extração de texto dos documentos

O coração da Quest: cada formato entra por uma trilha diferente, orquestrada por AWS Step Functions com etapas em AWS Lambda.

```md
Orquestracao:
S3 (novo objeto) -> EventBridge -> Step Functions -> estado de escolha (Choice) que
roteia pelo tipo detectado na Quest 1.

Trilha 1, PDF digital (ata_reuniao_vendas_sa.pdf):
- A Lambda le a camada de texto embutida e extrai o conteudo direto.
- NAO passa pelo Textract. O texto ja nasce limpo, OCR so adicionaria custo e erro.
- As tabelas de indicadores sao preservadas como estrutura.

Trilha 2, imagem escaneada (ata_resultados_vendas_novos_dados.png):
- Amazon Textract faz o OCR.
- Uso AnalyzeDocument com TABLES e FORMS para recuperar a tabela de indicadores como
  linhas e colunas, e os pares chave-valor.
- O Textract tambem detecta texto manuscrito, entao as anotacoes "conferir CRM" e
  "acao prioritaria" entram na base em vez de se perderem.
- Justificativa: sem OCR, essa pagina inteira ficaria invisivel para a busca. Escolhi
  Textract porque parte do acervo e foto com escrita a mao, e so ele, dentro da AWS,
  le manuscrito e tabela na mesma passada.

Trilha 3, CSV do CRM (vendas_sa_dados_ficticios_laboratorio.csv):
- CSV nao e texto corrido, entao nao vai para OCR nem para chunking direto.
- O arquivo e catalogado por um AWS Glue Crawler no AWS Glue Data Catalog e fica
  consultavel via Amazon Athena (SQL).
- Para a Wiki responder em linguagem natural, uma Lambda gera "fichas em texto": para
  cada oportunidade e para agregados relevantes (por regiao, segmento, produto,
  campanha, motivo de perda), monta frases descritivas. Essas fichas vao para
  wiki-curated e entram na indexacao como qualquer outro documento.
- Assim o mesmo dado atende dois usos: SQL exato no Athena e busca semantica na Wiki.

Destino do texto: tudo que e extraido e normalizado vai para s3://wiki-processed/ e,
apos enriquecimento, para s3://wiki-curated/.
Logs de cada etapa em Amazon CloudWatch.
```

## 2.4 Tratamento de falhas

```md
- Step Functions com Retry (backoff exponencial) e Catch por etapa. Um erro em um
  documento nao derruba o lote inteiro.
- Fila de mortos (Amazon SQS DLQ) recebe os documentos que falharam apos as
  tentativas, para reprocessamento manual ou automatico.
- Uma tabela de status no DynamoDB registra o estado por documento (recebido,
  extraido, enriquecido, indexado, erro) com a mensagem de erro.
- Amazon CloudWatch Logs para detalhe e CloudWatch Alarms para avisar quando a taxa
  de erro passa de um limite.
- EventBridge encaminha eventos de falha para notificacao (por exemplo, Amazon SNS).
```

---

# ✅ Quest 3: A Relíquia dos Metadados

## 3.1 Padronização dos textos processados

```md
Uma Lambda normaliza tudo que sai da extracao para um formato unico (JSON), antes de
indexar:
- Junta quebras de linha artificiais e remove ruido tipico de OCR (caracteres soltos,
  hifenizacao de fim de linha).
- Normaliza Unicode e acentuacao, padroniza datas e valores monetarios.
- Remove duplicatas e cabecalhos ou rodapes repetidos.
- Guarda o texto limpo junto de um campo com o texto bruto, para auditoria.
- Estrutura de saida comum: { documento_id, tipo, texto_limpo, blocos, tabelas,
  origem (bucket/chave/versionId), checksum }.
```

## 3.2 Metadados propostos

| Metadado | Por que ele é importante? |
|---|---|
| Nome do documento | Identifica a fonte e aparece na citação da resposta |
| Tipo do documento | Separa ata, relatório e base CRM, e define a trilha de processamento |
| Data identificada | Permite filtrar por período (ex.: riscos do último trimestre) |
| Tema principal | Base para filtro e agrupamento por assunto |
| Participantes | Responde "quem estava" e liga pessoas a decisões |
| Decisões tomadas | É o que a liderança mais pergunta sobre o projeto X |
| Responsáveis | Responde "quem ficou com a ação Y" |
| Próximos passos | Expõe pendências em aberto de reuniões anteriores |
| Nível de confidencialidade | Governa quem pode ver o documento na busca |
| Caminho do arquivo original | Garante rastreabilidade até o objeto exato no S3 (com versionId) |
| Região / segmento / produto | Filtros de negócio vindos do CSV do CRM |
| Prazos | Permite alertar ações com data de vencimento |
| Checksum de integridade | Prova que o conteúdo indexado corresponde ao original |

## 3.3 Uso de IA para enriquecimento dos documentos

```md
Amazon Bedrock (modelo de linguagem gerenciado, por exemplo da familia Claude ou
Titan) le o texto normalizado e devolve os campos de negocio em formato estruturado
(JSON), com um prompt que pede saida validavel:
- Identifica tema principal, decisoes, responsaveis, prazos, participantes, riscos e
  proximos passos.
- Gera um resumo curto de cada documento (util na resposta e na previa da busca).
- Classifica tipo, area e nivel de confidencialidade (alimenta a Quest 1.4).
- Como e um passo separado por documento, o custo escala com o acervo e nao com cada
  consulta.
Campos inferidos pela IA ficam marcados como "inferido" para revisao humana quando
necessario, o que mantem a honestidade da base.
```

## 3.4 Armazenamento dos metadados

```md
- Amazon DynamoDB guarda o registro por documento: chave documento_id, ponteiro para
  o original no S3 (bucket, chave, versionId), tipo, data, tema, participantes,
  decisoes, responsaveis, proximos passos, confidencialidade e checksum.
  Escolhi DynamoDB por ser rapido para leitura por chave e para filtros, e por casar
  com os filtros de metadado da Knowledge Base.
- AWS Glue Data Catalog guarda o schema do CSV do CRM, para consulta SQL via Athena.
- Amazon Bedrock Knowledge Bases recebe os metadados junto de cada trecho, para
  permitir filtros na busca (por tipo, data, confidencialidade).
- A conexao com o original: todo metadado carrega o caminho e o versionId no S3.
  A resposta da Wiki consegue linkar de volta para o documento exato.
```

---

# ✅ Quest 4: O Oráculo da Wiki Inteligente

## 4.1 Estratégia de indexação

```md
- O texto normalizado e dividido em trechos (chunks) com sobreposicao, para nao cortar
  uma decisao no meio. Atas curtas podem virar poucos trechos, o CSV vira as fichas em
  texto geradas na Quest 2.
- Cada trecho carrega seus metadados (documento_id, tipo, data, confidencialidade,
  origem no S3), para busca com filtro e para citacao.
- O Amazon Bedrock Knowledge Bases gerencia a divisao em trechos e a geracao de
  embeddings, o que reduz codigo e mantem tudo dentro da AWS.
- A fonte da Knowledge Base e o bucket s3://wiki-curated/. A base e sincronizada por
  jobs de ingestao.
```

## 4.2 Busca semântica e base vetorial

```md
- Embeddings: modelo de embeddings do Amazon Bedrock (por exemplo Amazon Titan
  Embeddings), gerado pela propria Knowledge Base.
- Base vetorial: Amazon OpenSearch Serverless como armazenamento de vetores, integrado
  nativamente ao Bedrock Knowledge Bases. Alternativas validas: Amazon Aurora
  PostgreSQL com pgvector (bom se ja existe Postgres e time SQL) ou Amazon S3 Vectors
  (opcao mais barata para volume baixo e consulta pouco frequente, que e o caso deste
  laboratorio).
- A busca semantica compara o vetor da pergunta com os vetores dos trechos e traz os
  mais proximos por significado, nao por palavra exata. "Decisao sobre expansao
  comercial" encontra o trecho certo mesmo sem repetir as mesmas palavras.
```

## 4.3 Geração de respostas com IA

```md
Fluxo de uma pergunta (RAG):
1. O usuario faz a pergunta em linguagem natural na interface.
2. A pergunta chega a Knowledge Base via API RetrieveAndGenerate do Amazon Bedrock.
3. A busca semantica recupera os trechos mais relevantes, respeitando o filtro de
   confidencialidade do perfil do usuario.
4. O Amazon Bedrock (modelo de linguagem) gera a resposta usando apenas esses trechos
   como contexto (grounding), o que reduz alucinacao.
5. A resposta volta com as fontes: nome do documento, trecho usado e caminho no S3,
   alem de data e pessoas envolvidas quando existem nos metadados.

Quando a base nao tem a informacao, o sistema responde de forma honesta ("nao
encontrei nas fontes disponiveis") em vez de inventar. Amazon Bedrock Guardrails ajuda
a impor esse comportamento e a filtrar conteudo indevido.
```

## 4.4 Interface de consulta

```md
Opcao principal (mais rapida de entregar):
- Amazon Q Business conectado ao acervo, com login corporativo. Entrega chat com
  citacoes e controle de acesso com pouca infraestrutura.

Opcao custom (mais controle):
- Frontend em AWS Amplify.
- Amazon Cognito para autenticacao dos usuarios.
- Amazon API Gateway expondo o endpoint da Wiki.
- AWS Lambda recebendo a pergunta e chamando o Bedrock Knowledge Bases.
- A mesma resposta com citacoes descrita na 4.3.

Nos dois casos o usuario pergunta em linguagem natural e recebe resposta com fonte.
```

## 4.5 Segurança, auditoria e monitoramento

```md
- AWS IAM: papeis de menor privilegio para cada componente do pipeline.
- AWS KMS: criptografia em repouso em todos os buckets e no DynamoDB.
- Amazon Cognito: autenticacao e grupos de usuario. O grupo define o nivel de
  confidencialidade que a pessoa pode consultar, aplicado como filtro na busca.
- AWS CloudTrail: auditoria de quem consultou o que e quais APIs foram chamadas.
- Amazon CloudWatch: metricas, logs e alarmes de erro, latencia e volume.
- Amazon Macie: varre o S3 em busca de dado sensivel (PII) e ajuda a classificar
  confidencialidade.
- Amazon Bedrock Guardrails: qualidade e seguranca das respostas, e o comportamento de
  nao inventar quando falta contexto.
- AWS Cost Explorer e AWS Budgets: acompanham custo e alertam desvios.
```

---

# 🧩 Arquitetura Final da Solução

## 1. Visão geral

```md
Um pipeline RAG serverless, todo em AWS. Os arquivos entram no Amazon S3 e sao
classificados por conteudo, ja que estao soltos em raw/. Cada formato segue uma
trilha: o PDF digital tem o texto lido direto, a imagem passa por OCR com deteccao de
manuscrito no Amazon Textract, e o CSV do CRM vira dado catalogado (Glue e Athena)
mais fichas em texto. Amazon Bedrock enriquece e classifica os documentos, os
metadados vao para o DynamoDB, e o conteudo e indexado no Bedrock Knowledge Bases com
busca vetorial no OpenSearch Serverless. O usuario pergunta em linguagem natural e
recebe uma resposta com citacao da fonte, com seguranca, rastreabilidade e
monitoramento em toda a jornada.
```

## 2. Serviços AWS utilizados

| Serviço AWS | Papel na solução |
|---|---|
| Amazon S3 | Data lake em estágios (raw, processed, metadata, curated); origem da Knowledge Base |
| Amazon Textract | OCR da imagem escaneada, incluindo manuscrito e tabelas |
| AWS Lambda | Classificação, extração do PDF digital, normalização, fichas do CSV, chamadas ao Bedrock |
| AWS Step Functions | Orquestra as trilhas por tipo de documento, com retry e catch |
| Amazon EventBridge | Dispara o pipeline quando chega arquivo novo no S3 |
| AWS Glue Data Catalog | Cataloga o schema do CSV do CRM |
| Amazon Athena | Consulta SQL exata sobre os dados do CRM |
| Amazon Bedrock | Enriquecimento, classificação, embeddings e geração das respostas |
| Amazon Bedrock Knowledge Bases | Chunking, indexação e RAG com citação de fonte |
| Amazon OpenSearch Serverless | Base vetorial dos embeddings |
| Amazon DynamoDB | Metadados por documento e tabela de status do processamento |
| Amazon Cognito | Autenticação e acesso por perfil (filtro de confidencialidade) |
| Amazon Q Business / AWS Amplify + API Gateway | Interface de consulta em linguagem natural |
| AWS IAM / AWS KMS | Permissões mínimas e criptografia em repouso |
| Amazon CloudWatch / AWS CloudTrail | Monitoramento, logs, alarmes e auditoria |
| Amazon Macie | Detecção de dado sensível no S3 |
| AWS Cost Explorer / AWS Budgets | Controle de custo |

## 3. Fluxo de dados de ponta a ponta

```md
1. Arquivos partem da pasta raw/ (PDF, PNG e CSV).
2. Sao enviados para o Amazon S3 (bucket raw, imutavel, versionado e criptografado).
3. Cada objeto novo dispara EventBridge, que aciona a Step Functions.
4. Uma Lambda classifica o arquivo por conteudo e roteia a trilha:
   a. PDF digital: texto lido direto, sem OCR.
   b. PNG escaneado: Amazon Textract (OCR de texto, manuscrito e tabela).
   c. CSV: Glue Data Catalog e Athena, mais fichas em texto geradas por Lambda.
5. O texto e normalizado e limpo (Lambda).
6. Amazon Bedrock enriquece: tema, decisoes, responsaveis, prazos, resumo e
   classificacao de confidencialidade.
7. Metadados vao para o DynamoDB (com ponteiro e versionId do original no S3).
8. O conteudo curado vai para o S3 e e indexado no Bedrock Knowledge Bases
   (embeddings no OpenSearch Serverless).
9. O usuario autentica (Cognito) e pergunta em linguagem natural na interface.
10. A Knowledge Base recupera os trechos relevantes e o Bedrock gera a resposta.
11. A resposta cita o documento de origem e o trecho usado.
12. IAM, KMS, CloudTrail, CloudWatch, Macie e Budgets cobrem seguranca, auditoria,
    monitoramento e custo em toda a jornada.
```

## 4. Diagrama textual da arquitetura

```md
raw/ (PDF, PNG, CSV)
   |
   v
Amazon S3 (raw, imutavel + KMS + versionamento)
   |
   v
EventBridge -> Step Functions (roteia por tipo)
   |
   +--> PDF digital  --> Lambda le texto direto (sem OCR)
   +--> PNG escaneado --> Amazon Textract (OCR + manuscrito + tabela)
   +--> CSV do CRM    --> Glue Data Catalog + Athena  e  Lambda gera fichas em texto
   |
   v
Lambda normaliza -> Amazon Bedrock enriquece e classifica
   |
   +--> Metadados: Amazon DynamoDB (ponteiro + versionId do original)
   |
   v
S3 curated -> Amazon Bedrock Knowledge Bases -> OpenSearch Serverless (vetores)
   |
   v
Interface (Amazon Q Business  ou  Amplify + API Gateway + Lambda) + Amazon Cognito
   |
   v
Usuario pergunta em linguagem natural -> resposta com citacao da fonte

Transversal: IAM, KMS, CloudTrail, CloudWatch, Macie, Cost Explorer/Budgets.
```

## 5. Riscos e limitações

```md
- OCR pode errar em trechos de baixa qualidade ou em manuscrito ambiguo. Mitigo com
  Textract (que le manuscrito) e marcacao de baixa confianca para revisao.
- Metadados inferidos por IA podem exigir validacao humana. Ficam marcados como
  "inferido".
- Custo cresce com volume e com numero de consultas. Mitigo com S3 Lifecycle, base
  vetorial adequada ao volume e alarmes de orcamento.
- Respostas de IA precisam sempre citar a fonte, senao viram palpite. Grounding e
  Guardrails reduzem esse risco.
- CSV cru pode gerar fichas em texto demais. Priorizo agregados e oportunidades
  relevantes em vez de indexar as 240 linhas sem criterio.
- Dado sensivel exige atencao continua. Macie e Cognito ajudam, mas a classificacao
  precisa de revisao periodica.
```

## 6. Melhorias futuras

```md
- Ingestao automatica ponta a ponta: arquivo novo em raw/ dispara todo o pipeline e a
  sincronizacao da Knowledge Base sem acao manual (detalhado abaixo).
- Classificacao automatica por tipo, area e confidencialidade, com tags no S3 e no
  DynamoDB (detalhado abaixo).
- Interface web com historico de perguntas e favoritos.
- Painel de decisoes e pendencias em aberto, com alertas de prazo (por exemplo, avisar
  quando uma "acao prioritaria" esta perto do vencimento).
- Controle de acesso por departamento, alem do nivel de confidencialidade.
- Avaliacao continua da qualidade das respostas, com feedback do usuario.
```

---

# 🎁 Evoluções detalhadas (bônus)

## Automação da ingestão

```md
Objetivo: um arquivo novo entra na base sozinho, sem ninguem rodar nada.

Como funciona:
1. O upload em s3://wiki-raw/ gera um evento (S3 Event Notification).
2. Amazon EventBridge recebe o evento e aciona a Step Functions.
3. A Step Functions executa classificacao, extracao, normalizacao, enriquecimento e
   gravacao do curado, com retry e catch em cada etapa.
4. Ao final, dispara um Ingestion Job do Bedrock Knowledge Bases, que reindexa apenas
   o que mudou.
5. A tabela de status no DynamoDB registra cada etapa, e um alarme avisa se algo falha.

Resultado: o acervo fica sempre atualizado e a operacao vira event-driven e
serverless, sem servidor ligado esperando arquivo.
```

## Classificação de documentos

```md
Objetivo: organizar sem subpastas, classificando por tipo, area e confidencialidade.

Como funciona:
1. Classificacao tecnica na entrada: uma Lambda usa extensao e magic bytes para
   definir PDF, imagem ou CSV, e roteia a trilha.
2. Classificacao de negocio apos a extracao: o Amazon Bedrock le o texto e retorna:
   - tipo: ata, relatorio, base CRM
   - area: comercial, operacoes, marketing, controladoria
   - confidencialidade: publico, interno, restrito
3. Aplicacao: o resultado vira S3 Object Tagging no objeto e campos no DynamoDB.
4. Uso na busca: os mesmos rotulos viram filtros de metadado na Knowledge Base e
   controlam o acesso por perfil via Cognito.

Assim a pasta raw/ continua intacta e sem subpastas, mas o acervo fica totalmente
organizado e filtravel por metadado.
```

---

# 🧠 Checklist Final

- [x] Como transformar documentos escaneados em texto? Amazon Textract (OCR com manuscrito e tabela).
- [x] Como lidar com diferentes formatos na mesma pasta `raw/`? Classificação por conteúdo e três trilhas na Step Functions.
- [x] Como armazenar os documentos originais? Amazon S3 raw imutável, versionado e criptografado.
- [x] Como preservar a rastreabilidade entre resposta e documento fonte? Ponteiro com versionId no metadado e citação na resposta.
- [x] Como organizar metadados? DynamoDB por documento e Glue Data Catalog para o CSV.
- [x] Como criar busca semântica? Embeddings do Bedrock em base vetorial (OpenSearch Serverless).
- [x] Como usar Amazon Bedrock na solução? Enriquecimento, classificação, embeddings e geração RAG.
- [x] Como proteger documentos sensíveis? IAM, KMS, Cognito, Macie e Guardrails.
- [x] Como monitorar falhas? Step Functions com retry e catch, DLQ, CloudWatch e status no DynamoDB.
- [x] Como a empresa usaria essa Wiki no dia a dia? Chat em linguagem natural (Q Business ou Amplify), com resposta e fonte.

---

# 🏁 Conclusão

```md
Esta proposta leva um acervo baguncado ate uma resposta em linguagem natural sem sair
da AWS. O ponto central e nao tratar os tres arquivos do mesmo jeito: o PDF digital e
lido direto, a imagem passa por OCR com deteccao de manuscrito, e o CSV vira dado
catalogado mais fichas em texto. Sobre essa base, um pipeline serverless e
event-driven extrai, enriquece, indexa e responde, sempre citando a fonte e
respeitando confidencialidade.

Para a lideranca, o ganho e direto: perguntas como "qual foi a decisao sobre o projeto
X" ou "quem ficou responsavel pela acao Y" passam a ter resposta em segundos, com o
documento de origem ao lado. A arquitetura escala com o volume, controla custo e
mantem governanca, entregando a memoria da empresa de forma pesquisavel, rastreavel e
segura.
```
