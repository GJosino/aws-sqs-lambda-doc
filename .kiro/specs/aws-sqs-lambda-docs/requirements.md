# Requirements Document

## Introduction

Esta spec define os requisitos para um conjunto de documentação de referência que cobre dois serviços da AWS — Amazon Simple Queue Service (Amazon SQS) e AWS Lambda — e o padrão de integração que os conecta (SQS como fonte de eventos para o Lambda). O entregável é documentação escrita destinada a desenvolvedores e arquitetos que estão aprendendo ou consultando esses serviços. Não é código de aplicação. A documentação deve ser fundamentada na documentação oficial da AWS e cobrir, para cada serviço, os conceitos principais, orientações de introdução/implementação, dicas e observações importantes, limites e cotas, segurança e considerações de preço. Ela também deve cobrir o padrão de integração SQS-para-Lambda em profundidade, incluindo agrupamento em lotes (batching), tratamento de erros e idempotência. Como o entregável é documentação, os requisitos descrevem qual conteúdo, estrutura e padrões de qualidade a documentação deve satisfazer, em vez do comportamento de software em tempo de execução.

## Glossary

- **Documentation_Set**: O conjunto completo de documentação de referência produzido por esta spec, composto pelo SQS_Document, o Lambda_Document e o Integration_Document.
- **SQS_Document**: A seção de documentação de nível superior que cobre o Amazon SQS.
- **Lambda_Document**: A seção de documentação de nível superior que cobre o AWS Lambda.
- **Integration_Document**: A seção de documentação de nível superior que cobre o padrão de integração SQS-para-Lambda.
- **Author**: A pessoa que escreve e mantém o Documentation_Set (Autor).
- **Reader**: Um desenvolvedor ou arquiteto que lê o Documentation_Set para aprender ou consultar os serviços cobertos (Leitor).
- **Amazon SQS**: Amazon Simple Queue Service, um serviço de filas de mensagens totalmente gerenciado usado para desacoplar e escalar componentes de sistemas distribuídos.
- **AWS Lambda**: Um serviço de computação serverless que executa código em resposta a eventos sem exigir que o Leitor provisione ou gerencie servidores.
- **Standard_Queue**: Um tipo de fila do Amazon SQS que oferece ordenação de melhor esforço (best-effort), entrega pelo menos uma vez (at-least-once) e throughput praticamente ilimitado.
- **FIFO_Queue**: Um tipo de fila do Amazon SQS que oferece ordenação estrita primeiro a entrar, primeiro a sair (first-in-first-out), processamento exatamente uma vez (exactly-once) e throughput limitado.
- **DLQ**: Dead-letter queue (fila de mensagens mortas); uma fila do Amazon SQS que recebe mensagens que não puderam ser processadas com sucesso após um número configurado de tentativas de recebimento.
- **Visibility_Timeout**: O período durante o qual uma mensagem recebida de uma fila do Amazon SQS fica oculta de outros consumidores.
- **Event_Source_Mapping**: O recurso do Lambda que faz polling de uma fonte de eventos, como uma fila do Amazon SQS, e invoca uma função Lambda com lotes de mensagens.
- **Partial_Batch_Response**: Um mecanismo de resposta do Lambda no qual a função informa os identificadores das mensagens cujo processamento falhou, de modo que apenas essas mensagens sejam reprocessadas.
- **Idempotency**: A propriedade de uma operação que produz o mesmo resultado e os mesmos efeitos colaterais independentemente de uma determinada mensagem ser processada uma ou várias vezes (idempotência).
- **EARS**: Easy Approach to Requirements Syntax; um conjunto de padrões para escrever requisitos estruturados.
- **Citation**: Um link para a página específica da documentação oficial da AWS que embasa uma afirmação factual (citação).
- **maxReceiveCount**: A contagem da política de redrive do SQS após a qual uma mensagem é enviada para uma DLQ.
- **GB-seconds**: A unidade de faturamento do Lambda correspondente à memória alocada em GB multiplicada pelo tempo de execução em segundos.
- **Ephemeral storage**: O armazenamento temporário /tmp disponível para um ambiente de execução do Lambda (armazenamento efêmero).

## Requirements

### Requisito 1: Estrutura e Escopo do Conjunto de Documentação

**História de Usuário:** Como um Leitor, quero a documentação organizada em seções claras e previsíveis, para que eu possa encontrar rapidamente informações sobre SQS, Lambda ou sua integração.

#### Critérios de Aceitação

1. THE Documentation_Set SHALL incluir um SQS_Document, um Lambda_Document e um Integration_Document como três seções de nível superior distintas e navegáveis separadamente, cada uma acessível a partir de um índice (sumário).
2. THE Documentation_Set SHALL incluir uma seção de introdução que declare o público-alvo, o escopo de cobertura e uma lista enumerada dos serviços da AWS cobertos, contendo no mínimo SQS, Lambda e sua integração.
3. WHERE uma seção cobre um serviço, THE Documentation_Set SHALL organizar essa seção exatamente nas seis subseções a seguir, nesta ordem: Conceitos Principais, Introdução / Implementação, Dicas e Observações Importantes, Limites e Cotas, Segurança e Considerações de Preço.
4. IF uma seção que cobre um serviço estiver sem uma ou mais das seis subseções obrigatórias, THEN THE Documentation_Set SHALL ser considerado não conforme para essa seção.
5. THE Documentation_Set SHALL apresentar os três documentos usando uma hierarquia de cabeçalhos idêntica, onde cada subseção aparece no mesmo nível de cabeçalho em todos os documentos, e usando terminologia consistente para conceitos equivalentes conforme definido no glossário.
6. THE Documentation_Set SHALL incluir um glossário que defina cada termo específico da AWS usado nos três documentos, de modo que cada um desses termos tenha exatamente uma entrada correspondente no glossário.

### Requisito 2: Cobertura dos Conceitos Principais do SQS

**História de Usuário:** Como um Leitor, quero entender o que é o Amazon SQS e seus conceitos principais, para que eu possa decidir quando e como usá-lo.

#### Critérios de Aceitação

1. THE SQS_Document SHALL definir o Amazon SQS como um serviço de filas de mensagens totalmente gerenciado, declarando que ele é usado para desacoplar e escalar componentes de sistemas distribuídos e que produtores e consumidores interagem de forma assíncrona sem conexões diretas.
2. THE SQS_Document SHALL descrever o ciclo de vida da mensagem em estágios ordenados, cobrindo o envio pelo produtor, o armazenamento da mensagem na fila, o recebimento pelo consumidor, a janela de Visibility_Timeout durante a qual uma mensagem recebida fica oculta de outros consumidores e a exclusão explícita da mensagem pelo consumidor após o processamento bem-sucedido.
3. IF uma mensagem é recebida mais vezes do que o maxReceiveCount configurado, THEN THE SQS_Document SHALL descrever que a mensagem é redirecionada para uma DLQ.
4. THE SQS_Document SHALL comparar Standard_Queue e FIFO_Queue em pelo menos as dimensões de ordenação de mensagens (melhor esforço vs. estritamente primeiro a entrar, primeiro a sair), garantia de entrega (pelo menos uma vez com possíveis duplicatas vs. processamento exatamente uma vez) e características de throughput (praticamente ilimitado vs. limitado em transações por segundo).
5. THE SQS_Document SHALL definir Visibility_Timeout, período de retenção de mensagens, atraso de entrega e DLQ, e SHALL declarar para cada um seu valor padrão e sua faixa configurável de mínimo a máximo conforme publicado pelo Amazon SQS.
6. THE SQS_Document SHALL descrever quando usar o Amazon SQS em relação ao Amazon SNS e ao Amazon MQ usando critérios de decisão observáveis, incluindo o padrão de comunicação (fila ponto a ponto vs. fan-out de publicação-assinatura) e as necessidades de compatibilidade de protocolo ou de padrão de mensageria.

### Requisito 3: Orientações de Implementação e Introdução do SQS

**História de Usuário:** Como um Leitor, quero orientações passo a passo para começar a usar o Amazon SQS, para que eu possa criar e usar uma fila.

#### Critérios de Aceitação

1. THE SQS_Document SHALL fornecer uma sequência ordenada e numerada de passos que cubra, em ordem, a criação de uma fila, o envio de uma mensagem, o recebimento de uma mensagem e a exclusão de uma mensagem, com cada passo declarando as entradas necessárias e o resultado observável esperado.
2. THE SQS_Document SHALL fornecer pelo menos um exemplo completo de código ou CLI para enviar uma mensagem e pelo menos um exemplo completo de código ou CLI para receber uma mensagem.
3. IF um exemplo de envio ou recebimento de mensagem é apresentado, THEN THE SQS_Document SHALL incluir a saída ou resposta de sucesso esperada para esse exemplo, para que o Leitor possa verificar o resultado.
4. THE SQS_Document SHALL descrever como configurar uma dead-letter queue (DLQ), incluindo como definir o maxReceiveCount, e SHALL declarar a faixa válida para o maxReceiveCount como um inteiro de 1 a 1000.
5. THE SQS_Document SHALL descrever como armazenar payloads maiores que o tamanho máximo de mensagem de 256 KB referenciando um armazenamento externo como o Amazon S3, incluindo a condição (payload excedendo 256 KB) que exige essa abordagem.

### Requisito 4: Limites, Cotas, Segurança e Preço do SQS

**História de Usuário:** Como um Leitor, quero conhecer os limites operacionais, os controles de segurança e o modelo de preço do Amazon SQS, para que eu possa projetar dentro de suas restrições e estimar o custo.

#### Critérios de Aceitação

1. THE SQS_Document SHALL declarar o tamanho máximo de mensagem de 256 KB, a faixa de retenção de mensagens de 60 segundos a 1.209.600 segundos (14 dias) e o período de retenção padrão de 4 dias.
2. THE SQS_Document SHALL descrever as opções de criptografia do lado do servidor, incluindo a criptografia usando chaves gerenciadas pelo Amazon SQS (SSE-SQS) e a criptografia usando chaves gerenciadas pelo cliente no AWS KMS (SSE-KMS).
3. THE SQS_Document SHALL descrever o controle de acesso usando tanto políticas baseadas em identidade do IAM quanto políticas de fila baseadas em recurso do SQS, incluindo como cada uma governa quais principais podem executar quais ações na fila.
4. THE SQS_Document SHALL descrever o modelo de preço de pagamento por solicitação (pay-per-request) e identificar os principais fatores de custo, incluindo o número de solicitações de API e o tratamento de payloads grandes.
5. WHERE um limite ou cota declarado é ajustável, THE SQS_Document SHALL rotulá-lo explicitamente como ajustável; e WHERE um limite declarado é um limite rígido (hard limit), THE SQS_Document SHALL rotulá-lo como fixo.

### Requisito 5: Cobertura dos Conceitos Principais do Lambda

**História de Usuário:** Como um Leitor, quero entender o que é o AWS Lambda e seus conceitos principais, para que eu possa decidir quando usar computação serverless.

#### Critérios de Aceitação

1. THE Lambda_Document SHALL definir o AWS Lambda como um serviço de computação serverless que executa código em resposta a eventos sem exigir que o Leitor provisione ou gerencie servidores.
2. THE Lambda_Document SHALL descrever o handler da função, as fontes de eventos/gatilhos e o modelo de ambiente de execução, declarando para cada um o que é e seu papel em uma invocação.
3. THE Lambda_Document SHALL descrever o modelo de concorrência, declarando que cada ambiente de execução processa uma solicitação por vez e que o Lambda escala criando ambientes de execução concorrentes adicionais à medida que a demanda aumenta.
4. THE Lambda_Document SHALL descrever cold starts e warm starts, definindo cada um e declarando que um cold start adiciona latência de inicialização a uma invocação, enquanto um warm start reutiliza um ambiente de execução existente.
5. THE Lambda_Document SHALL declarar a duração máxima de invocação de 15 minutos (900 segundos) para uma função Lambda e o valor padrão de timeout da função.

### Requisito 6: Orientações de Implementação e Introdução do Lambda

**História de Usuário:** Como um Leitor, quero orientações passo a passo para criar e implantar uma função Lambda, para que eu possa executar minha primeira função.

#### Critérios de Aceitação

1. THE Lambda_Document SHALL fornecer uma sequência ordenada e numerada de passos que cubra, em ordem, a criação da função, a configuração do gatilho, a implantação do código e a invocação da função, onde cada passo inclui a ação a executar e o resultado observável que confirma que o passo foi bem-sucedido.
2. THE Lambda_Document SHALL mostrar pelo menos um exemplo completo de código de handler em um runtime suportado, incluindo a assinatura da função handler, o parâmetro de entrada e um valor de resposta retornado.
3. THE Lambda_Document SHALL descrever como configurar a alocação de memória e o timeout, declarando a faixa de valores permitida para cada um, e explicar como o aumento da alocação de memória afeta o desempenho de execução e como a alocação de memória e a duração de execução afetam o custo.
4. THE Lambda_Document SHALL descrever como visualizar os logs da função e como visualizar as métricas da função para solução de problemas, identificando pelo menos uma fonte de log e pelo menos uma métrica disponível para uma função.
5. IF um passo descrito falhar quando seguido pelo Leitor, THEN THE Lambda_Document SHALL fornecer orientações de solução de problemas identificando a condição de falha e a ação corretiva correspondente para pelo menos os passos de criação e invocação da função.

### Requisito 7: Limites, Cotas, Segurança e Preço do Lambda

**História de Usuário:** Como um Leitor, quero conhecer os limites do Lambda, os controles de segurança e o modelo de preço, para que eu possa projetar dentro de suas restrições e estimar o custo.

#### Critérios de Aceitação

1. THE Lambda_Document SHALL declarar as cotas principais, incluindo a duração máxima de invocação, a faixa de alocação de memória, a faixa de armazenamento efêmero (ephemeral storage) e o limite padrão de execuções concorrentes por conta, com um valor numérico específico ou faixa delimitada e unidade para cada cota.
2. THE Lambda_Document SHALL indicar, para cada cota declarada, se a cota é ajustável por meio de uma solicitação de aumento de cota de serviço ou é um limite rígido (hard limit) que não pode ser alterado.
3. THE Lambda_Document SHALL descrever o papel de execução da função, incluindo que ele concede à função permissão para acessar outros serviços da AWS e que ele SHALL ser restrito para conceder apenas as permissões específicas de que a função precisa (privilégio mínimo).
4. THE Lambda_Document SHALL descrever como referenciar segredos e variáveis de ambiente, incluindo que as variáveis de ambiente são criptografadas em repouso e que valores sensíveis SHALL ser armazenados em um serviço dedicado de gerenciamento de segredos em vez de em variáveis de ambiente em texto plano.
5. THE Lambda_Document SHALL descrever o modelo de preço como uma função do número de solicitações e da duração de computação medida em GB-seconds, incluindo que a duração de computação é derivada da alocação de memória configurada multiplicada pelo tempo de execução.
6. THE Lambda_Document SHALL declarar as unidades e o comportamento de arredondamento usados para medir o tempo de execução faturável.

### Requisito 8: Padrão de Integração SQS-para-Lambda

**História de Usuário:** Como um Leitor, quero entender como usar o Amazon SQS como fonte de eventos para o Lambda, para que eu possa construir um pipeline de processamento orientado a eventos.

#### Critérios de Aceitação

1. THE Integration_Document SHALL descrever o Event_Source_Mapping no qual o Lambda faz polling de uma fila do SQS e invoca a função de forma síncrona com um lote de entre 1 e 10.000 mensagens para uma Standard_Queue e entre 1 e 10 mensagens para uma FIFO_Queue.
2. THE Integration_Document SHALL declarar que o Lambda suporta tanto Standard_Queue quanto FIFO_Queue como fontes de eventos.
3. THE Integration_Document SHALL descrever o comportamento de agrupamento em lotes (batching), incluindo o tamanho do lote (número máximo de mensagens por lote: até 10.000 para uma Standard_Queue e até 10 para uma FIFO_Queue) e a janela de lote (tempo máximo para reunir mensagens antes da invocação: 0 a 300 segundos).
4. THE Integration_Document SHALL declarar que a função Lambda e a fila do SQS devem estar na mesma Região da AWS, e que elas podem estar em contas da AWS diferentes.
5. THE Integration_Document SHALL descrever como um lote processado com sucesso resulta na exclusão das mensagens desse lote da fila, e como o Visibility_Timeout deve ser definido com um valor maior ou igual ao timeout da função Lambda para que as mensagens permaneçam ocultas de outros consumidores durante o processamento.
6. IF a função Lambda retorna um erro ou expira o tempo (timeout) ao processar um lote, THEN THE Integration_Document SHALL descrever como as mensagens do lote não são excluídas e ficam visíveis novamente após a expiração do Visibility_Timeout, tornando-as disponíveis para reentrega.
7. WHEN o Visibility_Timeout é configurado com um valor menor que o timeout da função Lambda, THE Integration_Document SHALL descrever o risco resultante de processamento duplicado de mensagens.

### Requisito 9: Tratamento de Erros e Idempotência da Integração

**História de Usuário:** Como um Leitor, quero orientações sobre como tratar falhas e mensagens duplicadas no padrão SQS-para-Lambda, para que meu processamento seja confiável e correto.

#### Critérios de Aceitação

1. THE Integration_Document SHALL declarar que o processamento do Event_Source_Mapping é pelo menos uma vez (at-least-once) e que o processamento duplicado da mesma mensagem pode ocorrer uma ou mais vezes.
2. THE Integration_Document SHALL recomendar que o código da função seja idempotente e SHALL definir idempotência como produzir o mesmo resultado e os mesmos efeitos colaterais independentemente de uma determinada mensagem ser processada uma ou várias vezes.
3. THE Integration_Document SHALL descrever o Partial_Batch_Response, incluindo que a função retorna uma lista de identificadores de mensagens que falharam, de modo que apenas essas mensagens que falharam em um lote sejam reprocessadas, enquanto as mensagens processadas com sucesso são removidas da fila.
4. IF uma mensagem falha no processamento após atingir o limite maxReceiveCount da fila, THEN THE Integration_Document SHALL descrever como uma DLQ captura a mensagem, a retém para inspeção posterior e impede reprocessamento adicional na fila de origem.
5. THE Integration_Document SHALL descrever como definir o Visibility_Timeout da fila com um valor maior ou igual ao timeout da função, e SHALL declarar que uma proporção recomendada é de pelo menos 6 vezes o timeout da função, para evitar o reprocessamento prematuro de mensagens em andamento.
6. WHEN o Partial_Batch_Response está habilitado, mas a função retorna uma resposta malformada ou vazia após uma falha, THE Integration_Document SHALL descrever que o lote inteiro é tratado como uma falha e todas as mensagens do lote são reprocessadas.

### Requisito 10: Precisão de Conteúdo, Citações e Qualidade

**História de Usuário:** Como um Leitor, quero que a documentação seja precisa e rastreável até fontes oficiais da AWS, para que eu possa confiar na informação e verificá-la.

#### Critérios de Aceitação

1. THE Documentation_Set SHALL basear todos os fatos, limites e comportamentos dos serviços na documentação oficial da AWS publicada em docs.aws.amazon.com ou aws.amazon.com.
2. WHERE uma afirmação factual sobre um limite, cota, preço ou comportamento é feita, THE Documentation_Set SHALL incluir uma Citation, expressa como um link para a página específica da documentação oficial da AWS que a embasa.
3. THE Documentation_Set SHALL usar cada termo definido no glossário de forma consistente, usando o mesmo termo para o mesmo conceito em todos os três documentos, em vez de sinônimos intercambiáveis.
4. WHERE um valor documentado é um que a AWS pode alterar ao longo do tempo (como um padrão de cota, um preço ou um runtime suportado), THE Documentation_Set SHALL incluir uma nota aconselhando o Leitor a verificar o valor atual na documentação oficial da AWS.
5. THE Documentation_Set SHALL apresentar cada exemplo de código e CLI em um bloco de código delimitado (fenced) anotado com o identificador de linguagem ou de shell.
