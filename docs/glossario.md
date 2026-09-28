# Glossário

- **Amazon SQS**: Amazon Simple Queue Service, um serviço de filas de mensagens totalmente gerenciado usado para desacoplar e escalar componentes de sistemas distribuídos.
- **AWS Lambda**: Um serviço de computação serverless que executa código em resposta a eventos sem exigir que o Leitor provisione ou gerencie servidores.
- **Standard_Queue**: Um tipo de fila do Amazon SQS que oferece ordenação de melhor esforço (best-effort), entrega pelo menos uma vez (at-least-once) e throughput praticamente ilimitado.
- **FIFO_Queue**: Um tipo de fila do Amazon SQS que oferece ordenação estrita primeiro a entrar, primeiro a sair (first-in-first-out), processamento exatamente uma vez (exactly-once) e throughput limitado.
- **DLQ**: Dead-letter queue (fila de mensagens mortas); uma fila do Amazon SQS que recebe mensagens que não puderam ser processadas com sucesso após um número configurado de tentativas de recebimento.
- **Visibility_Timeout**: O período durante o qual uma mensagem recebida de uma fila do Amazon SQS fica oculta de outros consumidores.
- **Event_Source_Mapping**: O recurso do Lambda que faz polling de uma fonte de eventos, como uma fila do Amazon SQS, e invoca uma função Lambda com lotes de mensagens.
- **Partial_Batch_Response**: Um mecanismo de resposta do Lambda no qual a função informa os identificadores das mensagens cujo processamento falhou, de modo que apenas essas mensagens sejam reprocessadas.
- **Idempotency**: A propriedade de uma operação que produz o mesmo resultado e os mesmos efeitos colaterais independentemente de uma determinada mensagem ser processada uma ou várias vezes (idempotência).
- **maxReceiveCount**: A contagem da política de redrive do SQS após a qual uma mensagem é enviada para uma DLQ.
- **GB-seconds**: A unidade de faturamento do Lambda correspondente à memória alocada em GB multiplicada pelo tempo de execução em segundos.
- **Ephemeral storage**: O armazenamento temporário /tmp disponível para um ambiente de execução do Lambda (armazenamento efêmero).
