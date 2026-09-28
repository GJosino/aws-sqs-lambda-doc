# Integração Amazon SQS e AWS Lambda

## Event_Source_Mapping e Batching

O padrão de integração entre o Amazon SQS e o AWS Lambda é construído sobre o **Event_Source_Mapping**: um recurso do Lambda que faz *polling* de uma fila do Amazon SQS e invoca a função automaticamente com lotes de mensagens, sem que o Leitor precise escrever código de *polling*. Esta subseção descreve como o Event_Source_Mapping lê as mensagens, como agrupa mensagens em lotes (*batching*), quais tipos de fila são suportados e como o ciclo de exclusão/reentrega das mensagens interage com o Visibility_Timeout.

### O que é o Event_Source_Mapping

Com um Event_Source_Mapping do Amazon SQS, o Lambda faz *polling* da fila e invoca a função de forma **síncrona**, passando um evento que contém um lote de uma ou mais mensagens; o Lambda recebe esses eventos um lote por vez e invoca a função uma vez para cada lote ([Using Lambda with Amazon SQS](https://docs.aws.amazon.com/lambda/latest/dg/with-sqs.html)). Como a invocação é síncrona, a função processa o lote inteiro e o resultado da invocação (sucesso, erro ou *timeout*) determina o que acontece com as mensagens do lote (ver [Exclusão, reentrega e Visibility_Timeout](#exclusão-reentrega-e-visibility_timeout)).

O Event_Source_Mapping suporta como fontes de eventos tanto **Standard_Queue** quanto **FIFO_Queue** ([Using Lambda with Amazon SQS](https://docs.aws.amazon.com/lambda/latest/dg/with-sqs.html)).

### Tamanho do lote e janela de lote (batching)

O comportamento de *batching* é controlado por dois parâmetros do Event_Source_Mapping:

- **Tamanho do lote (batch size)** — o número máximo de mensagens que o Lambda reúne em um único lote antes de invocar a função. Para uma **Standard_Queue**, o máximo é **10.000 mensagens**; para uma **FIFO_Queue**, o máximo é **10 mensagens**. O intervalo válido, portanto, é de **1 a 10.000** (Standard_Queue) e de **1 a 10** (FIFO_Queue) ([Lambda parameters for Amazon SQS event source mappings](https://docs.aws.amazon.com/lambda/latest/dg/services-sqs-parameters.html)).
- **Janela de lote (batch window / `MaximumBatchingWindowInSeconds`)** — o tempo máximo, em segundos, que o Lambda aguarda para reunir mensagens antes de invocar a função. O valor pode ser configurado de **0 a 300 segundos** (até 5 minutos) ([AWS Lambda now supports batch windows of up to 5 minutes for Amazon SQS](https://aws.amazon.com/about-aws/whats-new/2020/11/aws-lambda-now-supports-batch-windows-of-up-to-5-minutes-for-functions/)).

Um lote é invocado quando **qualquer** um dos limites é atingido primeiro: o tamanho do lote configurado é alcançado ou a janela de lote expira.

> **Nota de verificação:** os valores máximos de tamanho de lote (10.000 para Standard_Queue, 10 para FIFO_Queue) e de janela de lote (300 s) podem ser alterados pela AWS ao longo do tempo. Confirme os valores atuais na documentação oficial da AWS em [Lambda parameters for Amazon SQS event source mappings](https://docs.aws.amazon.com/lambda/latest/dg/services-sqs-parameters.html) antes de projetar em torno desses limites.

| Parâmetro | Standard_Queue | FIFO_Queue | Citação |
|---|---|---|---|
| Tamanho do lote (mensagens por lote) | 1 a 10.000 | 1 a 10 | [Lambda parameters for Amazon SQS](https://docs.aws.amazon.com/lambda/latest/dg/services-sqs-parameters.html) |
| Janela de lote (segundos) | 0 a 300 | 0 a 300 | [Batch windows de até 5 minutos](https://aws.amazon.com/about-aws/whats-new/2020/11/aws-lambda-now-supports-batch-windows-of-up-to-5-minutes-for-functions/) |

### Região e conta da AWS

A função Lambda e a fila do Amazon SQS **devem estar na mesma Região da AWS**, embora possam estar em **contas da AWS diferentes** ([Using Lambda with Amazon SQS](https://docs.aws.amazon.com/lambda/latest/dg/with-sqs.html)). Ou seja, o padrão suporta um cenário *cross-account* (fila em uma conta, função em outra), desde que ambas estejam na mesma Região.

### Exclusão, reentrega e Visibility_Timeout

O que acontece com as mensagens de um lote depende do resultado da invocação síncrona:

- **Lote processado com sucesso →** as mensagens desse lote são **excluídas** da fila pelo Lambda, saindo do ciclo de reentrega ([Using Lambda with Amazon SQS](https://docs.aws.amazon.com/lambda/latest/dg/with-sqs.html)).
- **Erro ou *timeout* da função →** as mensagens do lote **não são excluídas**. Elas permanecem ocultas durante a janela de Visibility_Timeout e, após a expiração desse período, ficam **visíveis novamente** na fila, tornando-se disponíveis para reentrega a uma invocação subsequente ([Using Lambda with Amazon SQS](https://docs.aws.amazon.com/lambda/latest/dg/with-sqs.html)).

Para que as mensagens permaneçam ocultas de outros consumidores durante todo o processamento, o **Visibility_Timeout da fila deve ser maior ou igual ao *timeout* da função Lambda**. A AWS impõe essa relação ao criar ou atualizar o Event_Source_Mapping: o *timeout* da função deve ser menor ou igual ao Visibility_Timeout da fila, e a AWS retorna um erro caso essa condição não seja satisfeita ([Creating and configuring an Amazon SQS event source mapping](https://docs.aws.amazon.com/lambda/latest/dg/services-sqs-configure.html)).

**Risco de processamento duplicado:** quando o Visibility_Timeout é configurado com um valor **menor** que o *timeout* da função, uma mensagem ainda em processamento pode voltar a ficar visível antes de a função concluir. Nesse caso, o Lambda pode entregar a mesma mensagem novamente a outra invocação, resultando em **processamento duplicado** ([Creating and configuring an Amazon SQS event source mapping](https://docs.aws.amazon.com/lambda/latest/dg/services-sqs-configure.html)). O dimensionamento recomendado do Visibility_Timeout e as estratégias para lidar com duplicatas (Idempotency, Partial_Batch_Response e DLQ) são detalhados na subseção [Tratamento de Erros e Idempotência](#tratamento-de-erros-e-idempotência).

## Tratamento de Erros e Idempotência

O Event_Source_Mapping do Amazon SQS entrega mensagens ao AWS Lambda com semântica **at-least-once** (pelo menos uma vez), o que torna o processamento duplicado uma condição normal e esperada — não uma exceção. Esta subseção descreve por que duplicatas ocorrem, como escrever código que as tolere (Idempotency), como reportar falhas por mensagem com o Partial_Batch_Response, como uma DLQ isola mensagens que esgotaram as tentativas, como dimensionar o Visibility_Timeout para evitar reprocessamento prematuro e o que acontece quando a resposta de Partial_Batch_Response é malformada ou vazia.

### Entrega at-least-once e duplicatas

O processamento por meio do Event_Source_Mapping é **at-least-once**: a mesma mensagem pode ser entregue e processada **uma ou mais vezes**. Duplicatas podem surgir de várias formas — por exemplo, a natureza de entrega pelo menos uma vez da Standard_Queue, uma mensagem que volta a ficar visível após a expiração do Visibility_Timeout antes de ser excluída, ou uma reentrega após falha parcial de um lote ([Using Lambda with Amazon SQS](https://docs.aws.amazon.com/lambda/latest/dg/with-sqs.html)). Por isso, o Leitor **não deve** assumir que cada mensagem será processada exatamente uma vez, mesmo ao usar uma FIFO_Queue, cuja deduplicação atua apenas dentro da janela de deduplicação da fila e não substitui código tolerante a duplicatas na função.

### Código idempotente (Idempotency)

Como duplicatas são possíveis, a AWS recomenda que a função Lambda seja **idempotente** ([Using Lambda with Amazon SQS](https://docs.aws.amazon.com/lambda/latest/dg/with-sqs.html)). **Idempotency** é a propriedade de uma operação que produz o **mesmo resultado e os mesmos efeitos colaterais** independentemente de uma determinada mensagem ser processada uma ou várias vezes: processar a mesma mensagem duas vezes deixa o sistema no mesmo estado que processá-la uma única vez.

Na prática, alcança-se Idempotency associando cada mensagem a um identificador estável (por exemplo, o `messageId` do SQS ou uma chave de negócio contida no payload) e registrando, de forma atômica, quais identificadores já foram processados — descartando ou tornando inócua qualquer reentrega já vista. Padrões comuns incluem gravações condicionais (por exemplo, *conditional writes* que só efetivam a operação se o identificador ainda não existir) e tabelas de controle de idempotência com expiração. Isso garante que uma reentrega não gere efeitos colaterais duplicados, como cobrar duas vezes ou emitir dois pedidos.

### Partial_Batch_Response (falhas por mensagem)

Por padrão, se a função lança um erro ao processar um lote, **todo** o lote volta a ficar visível e é reprocessado — inclusive as mensagens que já haviam sido tratadas com sucesso. O **Partial_Batch_Response** evita esse reprocessamento em massa: a função retorna uma lista dos **identificadores das mensagens que falharam** (`batchItemFailures`), de modo que **apenas** essas mensagens sejam reprocessadas, enquanto as mensagens processadas com sucesso são **removidas da fila** ([Reporting batch item failures for SQS event source mappings](https://docs.aws.amazon.com/lambda/latest/dg/services-sqs-batchfailurereporting.html)).

Para habilitar esse comportamento, configura-se `ReportBatchItemFailures` no Event_Source_Mapping e a função retorna um objeto com `batchItemFailures`, listando o `itemIdentifier` (o `messageId`) de cada mensagem que falhou:

```json
{
  "batchItemFailures": [
    { "itemIdentifier": "059f36b4-87a3-44ab-83d2-661975830a7d" },
    { "itemIdentifier": "2e1424d4-f796-459a-8184-9c92662be6da" }
  ]
}
```

Nesse exemplo, apenas as duas mensagens listadas voltam a ficar visíveis para reentrega; todas as demais mensagens do lote são consideradas processadas com sucesso e saem da fila ([Reporting batch item failures for SQS event source mappings](https://docs.aws.amazon.com/lambda/latest/dg/services-sqs-batchfailurereporting.html)).

### DLQ ao atingir o maxReceiveCount

Uma mensagem que continua falhando é reentregue repetidamente. Para impedir reentregas infinitas, associa-se uma **DLQ** à fila de origem por meio de uma *redrive policy* que define o **maxReceiveCount**. Quando o número de recebimentos de uma mensagem excede o maxReceiveCount configurado, o Amazon SQS **move** a mensagem para a DLQ ([Amazon SQS dead-letter queues](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html)). A DLQ, portanto:

- **Captura** a mensagem problemática, retirando-a da fila de origem.
- **Retém** a mensagem para inspeção posterior (análise da causa da falha, depuração, reprocessamento manual).
- **Impede** o reprocessamento adicional na fila de origem, evitando que uma mensagem "envenenada" (*poison message*) bloqueie ou degrade o processamento do restante da fila.

> **Nota de verificação:** o maxReceiveCount é configurável (inteiro de 1 a 1000) e o comportamento de redrive pode ser refinado pela AWS ao longo do tempo. Confirme os valores e o comportamento atuais na documentação oficial da AWS em [Amazon SQS dead-letter queues](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html) antes de definir a política de redrive.

### Dimensionamento do Visibility_Timeout

Para evitar que uma mensagem em andamento volte a ficar visível e seja reprocessada antes de a função concluir, o **Visibility_Timeout** da fila deve ser **maior ou igual ao *timeout* da função** Lambda. A AWS recomenda definir o Visibility_Timeout da fila de origem para **pelo menos 6 vezes** o *timeout* configurado da função; essa margem acomoda o tempo de processamento da função somado à janela de lote (batch window), reduzindo o risco de reprocessamento prematuro de mensagens que ainda estão sendo tratadas ([Configuring a queue to trigger a Lambda function](https://docs.aws.amazon.com/lambda/latest/dg/with-sqs.html)).

> **Nota de verificação:** a proporção recomendada (≥ 6× o *timeout* da função) pode ser ajustada pela AWS. Confirme a recomendação atual na documentação oficial da AWS em [Using Lambda with Amazon SQS](https://docs.aws.amazon.com/lambda/latest/dg/with-sqs.html) ao dimensionar o Visibility_Timeout.

### Resposta de Partial_Batch_Response malformada ou vazia

Quando o Partial_Batch_Response está habilitado (`ReportBatchItemFailures`), mas a função retorna uma resposta **malformada** (por exemplo, um `itemIdentifier` nulo, vazio ou que não corresponde a nenhuma mensagem do lote) **após uma falha**, o Lambda não consegue determinar com segurança quais mensagens tiveram sucesso. Nesse caso, o **lote inteiro é tratado como falha** e **todas** as mensagens do lote são reprocessadas ([Reporting batch item failures for SQS event source mappings](https://docs.aws.amazon.com/lambda/latest/dg/services-sqs-batchfailurereporting.html)).

Os principais casos que levam o lote inteiro a ser reprocessado são:

- A função retorna a lista de `batchItemFailures` com um `itemIdentifier` **vazio ou nulo**.
- A função retorna um `itemIdentifier` que **não corresponde** a nenhuma mensagem do lote recebido.
- A resposta está **malformada** e não pode ser interpretada como um objeto de Partial_Batch_Response válido.

Por outro lado, uma resposta com `batchItemFailures` **vazia** (uma lista sem itens) sinaliza que **todas** as mensagens foram processadas com sucesso e o lote inteiro é removido da fila. Reforça-se, assim, a importância da Idempotency: como um lote pode ser inteiramente reprocessado nesses cenários, mensagens já tratadas com sucesso podem ser entregues novamente à função ([Reporting batch item failures for SQS event source mappings](https://docs.aws.amazon.com/lambda/latest/dg/services-sqs-batchfailurereporting.html)).
