# Implementation Plan: aws-sqs-lambda-docs

## Overview

Este plano converte o design do **Documentation_Set** em uma série de tarefas incrementais de **autoria de conteúdo** (arquivos Markdown), não de codificação de aplicação. Cada tarefa consiste em escrever, criar ou redigir uma seção específica de um documento, com o conteúdo exigido pelos critérios de aceitação correspondentes.

A ordem segue a arquitetura do design: primeiro a estrutura de arquivos, o índice/introdução e o glossário compartilhado (fontes de estrutura e terminologia); em seguida, o conteúdo de cada documento (SQS, Lambda, Integração), subseção por subseção; depois as convenções transversais (citações, exemplos em blocos fenced, notas de verificação); e por último a validação/revisão contra os critérios V1–V5 da Testing Strategy.

Como o entregável é documentação e o design não define uma seção "Correctness Properties", **não há tarefas de testes unitários ou property-based tests (PBT)**. A "validação" é a revisão binária (conforme / não conforme) contra os critérios verificáveis do design.

Convenções de idioma aplicadas a todas as tarefas: conteúdo em português (pt-BR); nomes de serviços da AWS e termos técnicos do glossário em inglês (Amazon SQS, AWS Lambda, Event_Source_Mapping, Visibility_Timeout, Partial_Batch_Response, maxReceiveCount, DLQ, Standard_Queue, FIFO_Queue, GB-seconds, Ephemeral storage).

## Tasks

- [x] 1. Criar a estrutura de arquivos e o índice/introdução (index)
  - [x] 1.1 Criar a estrutura de arquivos do Documentation_Set
    - Criar o diretório de documentação e os arquivos vazios conforme a arquitetura: `docs/index.md`, `docs/glossario.md`, `docs/sqs.md`, `docs/lambda.md`, `docs/integracao-sqs-lambda.md`
    - Estabelecer a hierarquia de cabeçalhos idêntica em todos os documentos: título em `#` (H1), subseções obrigatórias em `##` (H2), detalhamentos em `###` (H3)
    - _Requisitos: 1.1, 1.5_

  - [x] 1.2 Escrever a introdução e o índice/sumário em `docs/index.md`
    - Redigir a seção de introdução declarando público-alvo (desenvolvedores e arquitetos), escopo de cobertura e uma **lista enumerada** dos serviços cobertos, contendo no mínimo Amazon SQS, AWS Lambda e sua integração
    - Redigir o índice (sumário) com links para as três seções de nível superior (SQS_Document, Lambda_Document, Integration_Document) e para suas subseções
    - _Requisitos: 1.1, 1.2_

- [x] 2. Criar o glossário compartilhado (`docs/glossario.md`)
  - [x] 2.1 Redigir o glossário compartilhado com uma entrada por termo
    - Escrever uma entrada única por termo específico da AWS usado nos três documentos, no formato do Modelo 2 (Termo em inglês em negrito + definição em português)
    - Cobrir o conjunto inicial: Amazon SQS, AWS Lambda, Standard_Queue, FIFO_Queue, DLQ, Visibility_Timeout, Event_Source_Mapping, Partial_Batch_Response, Idempotency, maxReceiveCount, GB-seconds, Ephemeral storage
    - Garantir exatamente uma entrada por termo (fonte única de verdade da terminologia)
    - _Requisitos: 1.6, 10.3_

- [x] 3. Escrever o SQS_Document (`docs/sqs.md`)
  - [x] 3.1 Escrever a subseção "Conceitos Principais" do SQS
    - Definir Amazon SQS como serviço de filas totalmente gerenciado (desacoplar/escalar; produtores e consumidores assíncronos sem conexão direta)
    - Descrever o ciclo de vida da mensagem em estágios ordenados (envio → armazenamento → recebimento → janela de Visibility_Timeout → exclusão explícita)
    - Descrever o redirecionamento para DLQ quando a mensagem excede o maxReceiveCount
    - Comparar Standard_Queue vs. FIFO_Queue (ordenação, garantia de entrega, throughput)
    - Definir Visibility_Timeout, período de retenção, atraso de entrega e DLQ, com valor padrão e faixa configurável de cada um
    - Descrever critérios de decisão SQS vs. Amazon SNS vs. Amazon MQ (padrão de comunicação e compatibilidade de protocolo/mensageria)
    - _Requisitos: 2.1, 2.2, 2.3, 2.4, 2.5, 2.6_

  - [x] 3.2 Escrever a subseção "Introdução / Implementação" do SQS
    - Redigir passos numerados e ordenados (criar fila → enviar → receber → excluir), cada um com entradas necessárias e resultado observável esperado (Modelo 4)
    - Incluir ≥1 exemplo completo de envio e ≥1 exemplo completo de recebimento (código ou CLI) em bloco fenced com identificador de linguagem/shell, cada um com a saída/resposta de sucesso esperada
    - Descrever a configuração de DLQ e do maxReceiveCount, declarando a faixa válida como inteiro de 1 a 1000
    - Descrever o armazenamento de payloads > 256 KB via armazenamento externo (Amazon S3), incluindo a condição (payload excedendo 256 KB) que exige essa abordagem
    - _Requisitos: 3.1, 3.2, 3.3, 3.4, 3.5_

  - [x] 3.3 Escrever a subseção "Dicas e Observações Importantes" do SQS
    - Redigir boas práticas operacionais e advertências (ex.: escolha de tipo de fila, uso de long polling, dimensionamento de Visibility_Timeout)
    - Incluir notas de verificação onde houver valores voláteis
    - _Requisitos: apoia 2.x, 3.x_

  - [x] 3.4 Escrever a subseção "Limites e Cotas" do SQS
    - Usar o Modelo 5 (tabela de limites) para declarar: tamanho máximo de mensagem 256 KB; retenção de 60 s a 1.209.600 s (14 dias), padrão 4 dias
    - Rotular explicitamente cada limite como `Ajustável` ou `Fixo (hard limit)`
    - Incluir Citation para cada valor
    - _Requisitos: 4.1, 4.5_

  - [x] 3.5 Escrever a subseção "Segurança" do SQS
    - Descrever criptografia do lado do servidor: SSE-SQS (chaves gerenciadas pelo Amazon SQS) e SSE-KMS (chaves gerenciadas pelo cliente no AWS KMS)
    - Descrever o controle de acesso com políticas baseadas em identidade do IAM e políticas de fila baseadas em recurso, explicando o que cada uma governa
    - _Requisitos: 4.2, 4.3_

  - [x] 3.6 Escrever a subseção "Considerações de Preço" do SQS
    - Descrever o modelo pay-per-request e os principais fatores de custo (número de solicitações de API e tratamento de payloads grandes)
    - Incluir nota de verificação de preço
    - _Requisitos: 4.4, 10.4_

- [x] 4. Checkpoint - Revisão parcial do SQS_Document
  - Garantir que o SQS_Document contém exatamente as seis subseções obrigatórias na ordem correta; ask the user if questions arise.

- [x] 5. Escrever o Lambda_Document (`docs/lambda.md`)
  - [x] 5.1 Escrever a subseção "Conceitos Principais" do Lambda
    - Definir AWS Lambda como computação serverless orientada a eventos (sem provisionar/gerenciar servidores)
    - Descrever handler, fontes de eventos/gatilhos e modelo de ambiente de execução (o que é e papel na invocação)
    - Descrever o modelo de concorrência (um pedido por vez por ambiente; escala criando ambientes concorrentes)
    - Definir cold start vs. warm start
    - Declarar a duração máxima de invocação de 15 min (900 s) e o timeout padrão
    - _Requisitos: 5.1, 5.2, 5.3, 5.4, 5.5_

  - [x] 5.2 Escrever a subseção "Introdução / Implementação" do Lambda
    - Redigir passos numerados (criar função → configurar gatilho → implantar código → invocar), cada um com ação e resultado observável esperado (Modelo 4)
    - Incluir ≥1 exemplo completo de handler em runtime suportado, em bloco fenced, mostrando assinatura, parâmetro de entrada e valor de resposta retornado
    - Descrever configuração de memória e timeout com faixas permitidas e efeito no desempenho/custo (memória × duração)
    - Descrever como ver logs e métricas (≥1 fonte de log e ≥1 métrica)
    - Fornecer solução de problemas para os passos de criação e invocação (condição de falha + ação corretiva)
    - _Requisitos: 6.1, 6.2, 6.3, 6.4, 6.5_

  - [x] 5.3 Escrever a subseção "Dicas e Observações Importantes" do Lambda
    - Redigir recomendações de desempenho, dimensionamento de memória/timeout e reuso de ambiente de execução
    - Incluir notas de verificação onde houver valores voláteis
    - _Requisitos: apoia 5.x, 6.x_

  - [x] 5.4 Escrever a subseção "Limites e Cotas" do Lambda
    - Usar o Modelo 5 (tabela de limites) para declarar, com valor/faixa e unidade: duração máxima de invocação, faixa de alocação de memória, faixa de Ephemeral storage e limite padrão de execuções concorrentes por conta
    - Indicar, para cada cota, se é `Ajustável` (via solicitação de aumento de cota) ou `Fixo (hard limit)`
    - Incluir Citation e nota de verificação para valores voláteis
    - _Requisitos: 7.1, 7.2_

  - [x] 5.5 Escrever a subseção "Segurança" do Lambda
    - Descrever o papel de execução (concede acesso a outros serviços da AWS; deve seguir privilégio mínimo)
    - Descrever segredos e variáveis de ambiente (criptografadas em repouso; valores sensíveis em serviço dedicado de gerenciamento de segredos, não em texto plano)
    - _Requisitos: 7.3, 7.4_

  - [x] 5.6 Escrever a subseção "Considerações de Preço" do Lambda
    - Descrever o preço em função do número de solicitações e da duração de computação em GB-seconds (memória configurada × tempo de execução)
    - Declarar as unidades e o comportamento de arredondamento do tempo de execução faturável
    - Incluir nota de verificação de preço
    - _Requisitos: 7.5, 7.6, 10.4_

- [x] 6. Checkpoint - Revisão parcial do Lambda_Document
  - Garantir que o Lambda_Document contém exatamente as seis subseções obrigatórias na ordem correta e na mesma hierarquia de cabeçalhos do SQS_Document; ask the user if questions arise.

- [x] 7. Escrever o Integration_Document (`docs/integracao-sqs-lambda.md`)
  - [x] 7.1 Escrever a subseção "Event_Source_Mapping e Batching"
    - Descrever o Event_Source_Mapping: Lambda faz polling do SQS e invoca a função de forma síncrona com lote de 1–10.000 mensagens (Standard_Queue) e 1–10 (FIFO_Queue)
    - Declarar suporte a Standard_Queue e FIFO_Queue como fontes de eventos
    - Descrever batching (tamanho do lote e janela de lote de 0 a 300 s)
    - Declarar que função e fila devem estar na mesma Região da AWS, podendo estar em contas diferentes
    - Descrever exclusão das mensagens em lote bem-sucedido; Visibility_Timeout ≥ timeout da função; erro/timeout → mensagens não excluídas e revisíveis após expiração; Visibility_Timeout < timeout → risco de processamento duplicado
    - Incluir Citation para cada valor de limite/comportamento
    - _Requisitos: 8.1, 8.2, 8.3, 8.4, 8.5, 8.6, 8.7_

  - [x] 7.2 Escrever a subseção "Tratamento de Erros e Idempotência"
    - Declarar processamento at-least-once (duplicatas possíveis)
    - Recomendar código idempotente e definir Idempotency
    - Descrever Partial_Batch_Response (lista de IDs de mensagens que falharam; só elas são reprocessadas; as bem-sucedidas saem da fila)
    - Descrever a DLQ ao atingir o maxReceiveCount (captura, retém para inspeção, impede reprocessamento na origem)
    - Descrever Visibility_Timeout ≥ timeout da função, com proporção recomendada de ≥ 6× o timeout
    - Descrever resposta de Partial_Batch_Response malformada/vazia → lote inteiro tratado como falha e todo reprocessado
    - _Requisitos: 9.1, 9.2, 9.3, 9.4, 9.5, 9.6_

- [x] 8. Aplicar as convenções transversais a todos os documentos
  - [x] 8.1 Aplicar o modelo de citações (Citation) a fontes oficiais da AWS
    - Revisar todos os documentos e garantir que toda afirmação sobre limite, cota, preço ou comportamento tenha uma Citation adjacente, no formato `[texto descritivo](https://docs.aws.amazon.com/...)`
    - Garantir que as fontes sejam apenas `docs.aws.amazon.com` ou `aws.amazon.com`, preferindo páginas canônicas estáveis
    - _Requisitos: 10.1, 10.2_

  - [x] 8.2 Aplicar o modelo de exemplos de código em blocos fenced
    - Revisar todos os documentos e garantir que cada exemplo de código/CLI esteja em bloco fenced anotado com identificador de linguagem/shell (ex.: ```bash, ```python, ```json)
    - Garantir que os exemplos de envio/recebimento do SQS e o handler do Lambda incluam a saída/resposta de sucesso esperada
    - _Requisitos: 10.5, 3.3, 6.2_

  - [x] 8.3 Aplicar notas de verificação para valores voláteis e consistência terminológica
    - Adicionar nota de verificação a cada valor volátil (padrões de cota, preços, runtimes suportados) aconselhando confirmar o valor atual na documentação oficial da AWS
    - Garantir que cada conceito seja referido pelo mesmo termo do glossário em todos os documentos (sem sinônimos intercambiáveis) e que nomes de serviços e termos técnicos permaneçam em inglês
    - _Requisitos: 10.4, 10.3_

- [x] 9. Validação e revisão final (critérios V1–V5 do design)
  - [x] 9.1 Validar conformidade estrutural (V1)
    - Verificar V1.1 (três seções navegáveis a partir do índice), V1.2 (introdução com público-alvo, escopo e lista enumerada), V1.3 (seis subseções obrigatórias na ordem correta), V1.4 (hierarquia de cabeçalhos idêntica), V1.5 (uma entrada de glossário por termo)
    - Marcar cada verificação como conforme / não conforme e corrigir não conformidades
    - _Requisitos: 1.1, 1.2, 1.3, 1.4, 1.5, 1.6_

  - [x] 9.2 Validar citações e precisão (V2 e V5)
    - Verificar V2 (toda afirmação de limite/cota/preço/comportamento tem Citation válida a fonte oficial; valores voláteis têm nota de verificação)
    - Verificar V5 (precisão dos fatos do SQS, Lambda e Integração contra as citações correspondentes)
    - Corrigir citações ausentes/quebradas e valores imprecisos
    - _Requisitos: 10.1, 10.2, 10.4, 4.1, 4.2, 4.3, 7.1, 7.2, 7.5, 7.6, 8.x, 9.x_

  - [x] 9.3 Validar exemplos e terminologia (V3 e V4)
    - Verificar V3 (todos os exemplos em blocos fenced com linguagem; exemplos de SQS/Lambda com saída/resposta esperada; passos numerados com entradas e resultado observável)
    - Verificar V4 (consistência terminológica contra o glossário; nomes e termos técnicos em inglês)
    - Corrigir divergências
    - _Requisitos: 10.5, 10.3, 3.1, 3.2, 3.3, 6.1, 6.2_

- [x] 10. Checkpoint final - Revisão completa
  - Garantir que todos os critérios de validação V1–V5 estão conformes; ask the user if questions arise.

## Notes

- Este é um entregável de **documentação** (arquivos Markdown), portanto todas as tarefas são de autoria de conteúdo, não de codificação de aplicação.
- Não há tarefas de testes unitários ou property-based tests: o design não define uma seção "Correctness Properties" (PBT não se aplica a documentação). A "validação" é a revisão binária (conforme / não conforme) contra os critérios verificáveis V1–V5 da Testing Strategy.
- Cada tarefa referencia os requisitos específicos que satisfaz, para rastreabilidade.
- A ordem é incremental: estrutura e glossário primeiro, conteúdo de cada documento em seguida, convenções transversais depois e validação por último.
- Os checkpoints garantem validação incremental da conformidade estrutural ao longo do trabalho.

## Task Dependency Graph

```json
{
  "waves": [
    { "id": 0, "tasks": ["1.1"] },
    { "id": 1, "tasks": ["1.2", "2.1"] },
    { "id": 2, "tasks": ["3.1", "5.1", "7.1"] },
    { "id": 3, "tasks": ["3.2", "5.2", "7.2"] },
    { "id": 4, "tasks": ["3.3", "5.3"] },
    { "id": 5, "tasks": ["3.4", "5.4"] },
    { "id": 6, "tasks": ["3.5", "5.5"] },
    { "id": 7, "tasks": ["3.6", "5.6"] },
    { "id": 8, "tasks": ["8.1"] },
    { "id": 9, "tasks": ["8.2"] },
    { "id": 10, "tasks": ["8.3"] },
    { "id": 11, "tasks": ["9.1"] },
    { "id": 12, "tasks": ["9.2"] },
    { "id": 13, "tasks": ["9.3"] }
  ]
}
```

Notas sobre o grafo: tarefas que escrevem no mesmo arquivo estão em ondas diferentes para evitar conflitos de escrita (ex.: 3.1–3.6 gravam em `docs/sqs.md`; 5.1–5.6 em `docs/lambda.md`; 7.1–7.2 em `docs/integracao-sqs-lambda.md`; as tarefas transversais 8.1–8.3 e de validação 9.1–9.3 revisam todos os arquivos e, por isso, são serializadas). Tarefas de documentos distintos na mesma onda são independentes e podem rodar em paralelo.
