# Amazon SQS

## Conceitos Principais

### O que é o Amazon SQS

O **Amazon SQS** (Amazon Simple Queue Service) é um serviço de filas de mensagens totalmente gerenciado, usado para **desacoplar e escalar** os componentes de sistemas distribuídos. Em vez de um componente chamar outro diretamente, o produtor envia a mensagem para uma fila e o consumidor a lê quando estiver pronto. Assim, produtores e consumidores interagem de forma **assíncrona, sem conexões diretas** entre si: cada lado pode falhar, reiniciar ou escalar de forma independente, e a fila absorve picos de carga como um buffer entre eles ([What is Amazon SQS?](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/welcome.html)).

Por ser totalmente gerenciado, o Amazon SQS não exige que o Leitor provisione ou administre servidores de mensageria: a AWS cuida da disponibilidade, da durabilidade e da escala da infraestrutura de filas.

### Ciclo de vida da mensagem

Uma mensagem percorre estágios ordenados desde o envio até a exclusão ([Como funcionam as filas do Amazon SQS](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-basic-architecture.html)):

1. **Envio (produtor).** Um produtor envia a mensagem para a fila (por exemplo, com a ação `SendMessage`).
2. **Armazenamento na fila.** O Amazon SQS armazena a mensagem de forma redundante e a mantém disponível até que seja recebida ou até expirar o período de retenção.
3. **Recebimento (consumidor).** Um consumidor lê a mensagem (por exemplo, com `ReceiveMessage`). A mensagem **não** é removida da fila nesse momento.
4. **Janela de Visibility_Timeout.** Ao ser recebida, a mensagem entra na janela de **Visibility_Timeout**, período durante o qual ela fica oculta de outros consumidores para que dois consumidores não a processem em paralelo ([Amazon SQS visibility timeout](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-visibility-timeout.html)).
5. **Exclusão explícita.** Após processar a mensagem com sucesso, o consumidor deve **excluí-la explicitamente** (com `DeleteMessage`). Se o consumidor não excluir a mensagem antes de o Visibility_Timeout expirar, ela volta a ficar visível e pode ser recebida novamente.

### Redirecionamento para a DLQ

Cada recebimento de uma mensagem incrementa o seu `ReceiveCount`. Quando uma mensagem é recebida mais vezes do que o **maxReceiveCount** configurado na política de redrive da fila, o Amazon SQS a move da fila de origem para uma **DLQ** (dead-letter queue). A DLQ isola as mensagens que não puderam ser processadas com sucesso, para inspeção e análise posterior, sem bloquear o processamento das demais mensagens da fila de origem ([Using dead-letter queues in Amazon SQS](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html)).

### Standard_Queue vs. FIFO_Queue

O Amazon SQS oferece dois tipos de fila. A tabela compara as três dimensões principais ([Amazon SQS queue types / FIFO queues](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/FIFO-queues.html)):

| Dimensão | Standard_Queue | FIFO_Queue |
|---|---|---|
| Ordenação | Melhor esforço (best-effort); a ordem de entrega pode diferir da ordem de envio | Estritamente primeiro a entrar, primeiro a sair (first-in-first-out) |
| Garantia de entrega | Pelo menos uma vez (at-least-once); duplicatas são possíveis | Exatamente uma vez (exactly-once); sem duplicatas dentro da janela de deduplicação |
| Throughput | Praticamente ilimitado (número quase ilimitado de transações por segundo) | Limitado (número finito de transações por segundo) |

Em resumo: escolha a **Standard_Queue** quando o mais importante for throughput máximo e a ordem exata não for crítica; escolha a **FIFO_Queue** quando a ordem estrita e a ausência de duplicatas forem essenciais.

### Atributos de configuração e seus padrões

Os quatro atributos abaixo controlam o comportamento temporal e de reentrega da fila. Cada um tem valor padrão e faixa configurável publicados pelo Amazon SQS.

| Atributo | O que controla | Valor padrão | Faixa configurável | Citação |
|---|---|---|---|---|
| **Visibility_Timeout** | Tempo em que a mensagem recebida fica oculta de outros consumidores | 30 segundos | 0 segundos a 12 horas | [Amazon SQS visibility timeout](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-visibility-timeout.html) |
| **Período de retenção** | Tempo que o SQS mantém a mensagem na fila até removê-la | 4 dias | 60 segundos a 1.209.600 segundos (14 dias) | [Configuring queue parameters](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-configure-queue-parameters.html) |
| **Atraso de entrega (delivery delay)** | Tempo que a mensagem fica indisponível para consumo após o envio | 0 segundos | 0 segundos a 15 minutos (900 segundos) | [Amazon SQS delay queues](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-delay-queues.html) |
| **DLQ** (via maxReceiveCount) | Fila que recebe mensagens não processadas após o número configurado de recebimentos | Sem DLQ (desabilitada por padrão) | maxReceiveCount: inteiro de 1 a 1000 | [Using dead-letter queues in Amazon SQS](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html) |

> **Nota de verificação:** estes valores padrão e faixas são valores que a AWS pode alterar ao longo do tempo. Confirme o valor atual na documentação oficial da AWS antes de projetar restrições rígidas em torno deles.

### Quando usar Amazon SQS, Amazon SNS ou Amazon MQ

Os três serviços tratam de mensageria, mas atendem a padrões de comunicação e necessidades de compatibilidade diferentes ([Amazon SQS, Amazon SNS, or Amazon EventBridge?](https://docs.aws.amazon.com/decision-guides/latest/decision-guides/sns-or-sqs-or-eventbridge.html)):

- **Amazon SQS** — padrão de **fila ponto a ponto**. Uma mensagem é entregue a um único consumidor (dentre os que fazem polling da fila) e retida até ser processada e excluída. Escolha o Amazon SQS quando precisar desacoplar componentes, absorver picos e garantir que cada mensagem seja processada por um consumidor, sem necessidade de protocolos específicos de mensageria.
- **Amazon SNS** — padrão de **publicação-assinatura (pub/sub) com fan-out**. Um publicador envia a mensagem para um tópico, e o Amazon SNS a entrega a todos os assinantes (endpoints HTTP/HTTPS, e-mail, SMS, filas do Amazon SQS, funções do AWS Lambda). Escolha o Amazon SNS quando a mesma mensagem precisar ser distribuída para vários destinos ao mesmo tempo.
- **Amazon MQ** — **broker de mensagens gerenciado** (Apache ActiveMQ e RabbitMQ) que suporta **APIs e protocolos de mensageria padrão do setor** (por exemplo, JMS, AMQP, MQTT, STOMP). Escolha o Amazon MQ principalmente ao **migrar** aplicações existentes que já dependem desses protocolos, evitando reescrever o código de mensageria. Para aplicações novas na nuvem, a AWS recomenda considerar primeiro o Amazon SQS e o Amazon SNS ([What is Amazon MQ?](https://docs.aws.amazon.com/amazon-mq/latest/developer-guide/welcome.html)).

Critério de decisão resumido: **um consumidor por mensagem → Amazon SQS**; **mesma mensagem para muitos destinos → Amazon SNS**; **compatibilidade com protocolos de mensageria padrão / migração de brokers existentes → Amazon MQ**.

## Introdução / Implementação

Esta seção apresenta a sequência mínima para começar a usar o **Amazon SQS**: criar uma fila, enviar uma mensagem, recebê-la e excluí-la. Cada passo declara as **entradas necessárias** e o **resultado observável esperado**, seguindo a ordem em que as operações ocorrem no ciclo de vida da mensagem. Os exemplos usam a AWS Command Line Interface (AWS CLI); as mesmas operações estão disponíveis nos SDKs da AWS e no Console de Gerenciamento da AWS.

> **Pré-requisito:** a AWS CLI deve estar instalada e configurada com credenciais e uma Região da AWS padrão. Nos exemplos, substitua os valores entre `< >` (como `<URL_DA_FILA>`) pelos valores do seu ambiente.

### Passos ordenados (criar → enviar → receber → excluir)

Os quatro passos abaixo cobrem, em ordem, a criação da fila, o envio, o recebimento e a exclusão de uma mensagem. As ações de fila do SQS seguem o modelo documentado no guia de operações básicas ([Enviar mensagens para uma fila do Amazon SQS](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-send-messages.html) e [Receber e excluir mensagens do Amazon SQS](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-receive-delete-messages.html)).

**Passo 1 — Criar a fila.** Entradas necessárias: um nome de fila (por exemplo, `MinhaFila`).

```bash
aws sqs create-queue --queue-name MinhaFila
```

Resultado observável esperado: a resposta contém o campo `QueueUrl`, que é a URL da fila recém-criada. Guarde esse valor para os próximos passos.

```json
{
    "QueueUrl": "https://sqs.us-east-1.amazonaws.com/111122223333/MinhaFila"
}
```

**Passo 2 — Enviar uma mensagem.** Entradas necessárias: a `QueueUrl` obtida no Passo 1 e o corpo da mensagem (`--message-body`).

```bash
aws sqs send-message \
  --queue-url https://sqs.us-east-1.amazonaws.com/111122223333/MinhaFila \
  --message-body "Olá, SQS!"
```

Resultado observável esperado: a resposta contém um `MessageId` (identificador único da mensagem) e um `MD5OfMessageBody` (hash MD5 do corpo enviado, que confirma a integridade do que foi recebido pelo serviço).

```json
{
    "MD5OfMessageBody": "a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6",
    "MessageId": "12345678-90ab-cdef-1234-567890abcdef"
}
```

**Passo 3 — Receber uma mensagem.** Entradas necessárias: a `QueueUrl`. Opcionalmente, `--max-number-of-messages` (até 10) e `--wait-time-seconds` (para habilitar long polling).

```bash
aws sqs receive-message \
  --queue-url https://sqs.us-east-1.amazonaws.com/111122223333/MinhaFila \
  --max-number-of-messages 1 \
  --wait-time-seconds 10
```

Resultado observável esperado: a resposta traz o array `Messages` com o `Body` da mensagem e, principalmente, um `ReceiptHandle`. O `ReceiptHandle` é um token temporário associado a **este** recebimento e é obrigatório para excluir a mensagem no Passo 4. A partir deste momento, a mensagem entra na janela de **Visibility_Timeout** e fica oculta de outros consumidores.

```json
{
    "Messages": [
        {
            "MessageId": "12345678-90ab-cdef-1234-567890abcdef",
            "ReceiptHandle": "AQEB3s6...token-temporario...Rw==",
            "MD5OfBody": "a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6",
            "Body": "Olá, SQS!"
        }
    ]
}
```

**Passo 4 — Excluir a mensagem.** Entradas necessárias: a `QueueUrl` e o `ReceiptHandle` obtido no Passo 3. A exclusão explícita é obrigatória após o processamento bem-sucedido; sem ela, a mensagem volta a ficar visível quando o Visibility_Timeout expirar e será entregue novamente.

```bash
aws sqs delete-message \
  --queue-url https://sqs.us-east-1.amazonaws.com/111122223333/MinhaFila \
  --receipt-handle "AQEB3s6...token-temporario...Rw=="
```

Resultado observável esperado: o comando é concluído **sem saída** e com código de saída `0`. A mensagem foi removida da fila e não será mais recebida.

### Exemplo completo de envio

O exemplo abaixo envia uma mensagem e mostra a resposta de sucesso esperada — atendendo à necessidade de um exemplo verificável de envio.

```bash
aws sqs send-message \
  --queue-url https://sqs.us-east-1.amazonaws.com/111122223333/MinhaFila \
  --message-body '{"pedidoId": 42, "status": "novo"}'
```

Saída de sucesso esperada (o `MessageId` confirma que a mensagem foi aceita e enfileirada):

```json
{
    "MD5OfMessageBody": "9a0364b9e99bb480dd25e1f0284c8555",
    "MessageId": "aaaa1111-bbbb-2222-cccc-333344445555"
}
```

### Exemplo completo de recebimento

O exemplo abaixo recebe uma mensagem usando long polling (`--wait-time-seconds`) e mostra a resposta de sucesso esperada — atendendo à necessidade de um exemplo verificável de recebimento.

```bash
aws sqs receive-message \
  --queue-url https://sqs.us-east-1.amazonaws.com/111122223333/MinhaFila \
  --wait-time-seconds 20
```

Saída de sucesso esperada (o array `Messages` traz o `Body` e o `ReceiptHandle` necessário para a posterior exclusão):

```json
{
    "Messages": [
        {
            "MessageId": "aaaa1111-bbbb-2222-cccc-333344445555",
            "ReceiptHandle": "AQEBd7n2...token-temporario...9Q==",
            "MD5OfBody": "9a0364b9e99bb480dd25e1f0284c8555",
            "Body": "{\"pedidoId\": 42, \"status\": \"novo\"}"
        }
    ]
}
```

> Se não houver mensagens disponíveis dentro do tempo de espera, a resposta é vazia (sem o campo `Messages`). Isso é um resultado normal, não um erro.

### Configurar uma DLQ e o maxReceiveCount

Uma **DLQ** (dead-letter queue) recebe as mensagens que não puderam ser processadas com sucesso, isolando-as para inspeção. A DLQ é ativada por uma **política de redrive** aplicada à fila de origem, na qual o **maxReceiveCount** define quantas vezes uma mensagem pode ser recebida antes de ser movida para a DLQ ([Usar dead-letter queues no Amazon SQS](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html)).

A configuração tem dois passos: (1) criar (ou identificar) a fila que servirá de DLQ e obter o seu ARN; (2) definir, na fila de origem, o atributo `RedrivePolicy` apontando para o ARN da DLQ e informando o `maxReceiveCount`.

**Passo A — Obter o ARN da DLQ.** Entradas necessárias: a `QueueUrl` da fila que será a DLQ.

```bash
aws sqs get-queue-attributes \
  --queue-url https://sqs.us-east-1.amazonaws.com/111122223333/MinhaDLQ \
  --attribute-names QueueArn
```

Resultado observável esperado: a resposta contém o `QueueArn` da DLQ.

```json
{
    "Attributes": {
        "QueueArn": "arn:aws:sqs:us-east-1:111122223333:MinhaDLQ"
    }
}
```

**Passo B — Aplicar a política de redrive na fila de origem.** Entradas necessárias: a `QueueUrl` da fila de origem, o ARN da DLQ e o valor de `maxReceiveCount`.

```bash
aws sqs set-queue-attributes \
  --queue-url https://sqs.us-east-1.amazonaws.com/111122223333/MinhaFila \
  --attributes '{"RedrivePolicy":"{\"deadLetterTargetArn\":\"arn:aws:sqs:us-east-1:111122223333:MinhaDLQ\",\"maxReceiveCount\":\"5\"}"}'
```

Resultado observável esperado: o comando é concluído **sem saída** e com código de saída `0`. A partir daí, uma mensagem recebida mais de 5 vezes na fila de origem é movida para a DLQ.

O **maxReceiveCount** aceita um **inteiro de 1 a 1000** ([Parâmetros de política de redrive do SQS](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-configure-dead-letter-queue-redrive-allow-policy.html)). Além disso, a DLQ deve ser do **mesmo tipo** da fila de origem (uma Standard_Queue usa uma Standard_Queue como DLQ; uma FIFO_Queue usa uma FIFO_Queue) e residir na mesma conta e Região da AWS.

> **Nota de verificação:** a faixa válida do maxReceiveCount e as regras da política de redrive são valores que a AWS pode alterar ao longo do tempo. Confirme os valores atuais na documentação oficial da AWS antes de projetar restrições rígidas em torno deles.

### Payloads maiores que o tamanho máximo de mensagem (armazenamento externo com Amazon S3)

O tamanho máximo de uma mensagem do Amazon SQS é **1 MiB (1.048.576 bytes)** — valor aumentado a partir dos 256 KiB anteriores e válido para filas Standard_Queue e FIFO_Queue ([Cotas de mensagens do Amazon SQS](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/quotas-messages.html) e [Amazon SQS increases maximum message payload size to 1 MiB](https://aws.amazon.com/about-aws/whats-new/2025/08/amazon-sqs-max-payload-size-1mib/)). **Quando o payload excede o tamanho máximo de mensagem**, ele não cabe em uma única mensagem e é necessário usar **armazenamento externo**: em vez de enviar o conteúdo completo na mensagem, armazene o payload em um objeto no **Amazon S3** e envie na fila apenas uma referência (por exemplo, o bucket e a chave do objeto). O consumidor lê a referência da mensagem e busca o payload no Amazon S3.

Esse padrão é implementado pela Amazon SQS Extended Client Library, que grava o payload no Amazon S3 e transmite pelo SQS apenas um ponteiro para o objeto ([Gerenciar mensagens grandes do Amazon SQS com o Amazon S3](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-s3-messages.html)). O fluxo lógico é:

1. **Produtor:** grava o payload (maior que o tamanho máximo de mensagem) como um objeto no Amazon S3 e envia à fila uma mensagem contendo apenas a referência ao objeto (bucket e chave).
2. **Consumidor:** recebe a mensagem, lê a referência, busca o objeto no Amazon S3 e processa o conteúdo; ao final, exclui a mensagem da fila (e, conforme a política de retenção, o objeto no Amazon S3).

Dessa forma, o payload real fica no Amazon S3 (que suporta objetos muito maiores) e o SQS transporta apenas a referência, respeitando o limite de tamanho por mensagem.

> **Nota de verificação:** o tamanho máximo de mensagem do Amazon SQS foi aumentado de 256 KiB para 1 MiB (anúncio de agosto de 2025) e é um valor que a AWS pode alterar ao longo do tempo. Confirme o valor atual na documentação oficial da AWS.

## Dicas e Observações Importantes

Esta seção reúne boas práticas operacionais e advertências para o uso do **Amazon SQS** no dia a dia. Elas complementam os conceitos e os passos de implementação apresentados nas seções anteriores, ajudando o Leitor a evitar armadilhas comuns de custo, desempenho e correção.

### Escolha do tipo de fila (Standard_Queue vs. FIFO_Queue)

Escolha o tipo de fila **antes** de criá-la: uma vez criada, uma fila **não** pode ser convertida entre Standard_Queue e FIFO_Queue — a conversão exige criar uma nova fila e migrar os produtores e consumidores ([Moving from a standard queue to a FIFO queue](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/FIFO-queues-moving.html)).

- Prefira a **Standard_Queue** como padrão. Ela oferece throughput praticamente ilimitado e é adequada à maioria dos casos em que a ordem exata das mensagens não é crítica ([Amazon SQS queue types](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/welcome.html)).
- Use a **FIFO_Queue** apenas quando a **ordem estrita** (first-in-first-out) e o **processamento exatamente uma vez** (sem duplicatas dentro da janela de deduplicação) forem requisitos reais. Em troca, a FIFO_Queue tem throughput limitado ([Amazon SQS FIFO queues](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/FIFO-queues.html)).
- Como a Standard_Queue entrega **pelo menos uma vez** (at-least-once), duplicatas são possíveis. Projete os consumidores para serem idempotentes, de modo que reprocessar a mesma mensagem não cause efeitos colaterais indevidos.

> **Nota de verificação:** as características de throughput da FIFO_Queue (limites por segundo, com ou sem batching e alto throughput) são valores que a AWS pode alterar ao longo do tempo. Confirme os valores atuais na documentação oficial da AWS.

### Prefira long polling em vez de short polling

Ao receber mensagens, defina `--wait-time-seconds` (o atributo `ReceiveMessageWaitTimeSeconds`) com um valor maior que `0` para habilitar o **long polling**. Com short polling (valor `0`), a chamada `ReceiveMessage` retorna imediatamente, muitas vezes vazia, o que **aumenta o número de solicitações de API** — e, portanto, o custo — e desperdiça consultas. Com long polling, a chamada aguarda até que uma mensagem chegue (ou até o tempo de espera terminar), reduzindo respostas vazias e o número de solicitações ([Amazon SQS short and long polling](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-short-and-long-polling.html)).

- O valor de espera do long polling aceita **de 1 a 20 segundos**; a AWS recomenda usar o valor máximo sempre que possível para minimizar respostas vazias ([Amazon SQS short and long polling](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-short-and-long-polling.html)).
- Habilite o long polling no nível da fila (definindo `ReceiveMessageWaitTimeSeconds`) para que valha por padrão, e/ou por chamada com `--wait-time-seconds`.

> **Nota de verificação:** o tempo máximo de espera do long polling (20 segundos) é um valor que a AWS pode alterar ao longo do tempo. Confirme o valor atual na documentação oficial da AWS.

### Dimensionamento do Visibility_Timeout

O **Visibility_Timeout** deve refletir o **tempo real de processamento** de uma mensagem pelo consumidor ([Amazon SQS visibility timeout](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-visibility-timeout.html)):

- Se o Visibility_Timeout for **curto demais** (menor que o tempo de processamento), a mensagem volta a ficar visível antes de o consumidor concluir e excluí-la, o que leva a **entrega duplicada** e a processamento redundante.
- Se for **longo demais**, uma mensagem cujo processamento falhou (por exemplo, um consumidor que travou) demora mais para ficar novamente disponível, atrasando a reentrega e a recuperação.
- Quando o tempo de processamento é difícil de estimar, use o valor padrão como ponto de partida e ajuste com base em métricas reais; alternativamente, estenda o tempo de uma mensagem específica em andamento com `ChangeMessageVisibility` (heartbeat) em vez de superdimensionar o padrão da fila.

> **Nota de verificação:** o valor padrão do Visibility_Timeout (30 segundos) e sua faixa (0 segundos a 12 horas) são valores que a AWS pode alterar ao longo do tempo. Confirme os valores atuais na documentação oficial da AWS.

### Sempre configure uma DLQ e monitore-a

Configure uma **DLQ** com uma política de redrive para isolar mensagens que falham repetidamente, evitando que uma mensagem "envenenada" (poison message) fique sendo recebida indefinidamente e bloqueie o consumo ([Using dead-letter queues in Amazon SQS](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html)):

- Ajuste o **maxReceiveCount** ao número de tentativas que faz sentido para o seu processamento (lembrando da faixa válida de inteiro de 1 a 1000). Um valor muito baixo pode mandar mensagens à DLQ por falhas transitórias; um valor muito alto atrasa a detecção de mensagens problemáticas.
- A DLQ deve ser do **mesmo tipo** da fila de origem (Standard_Queue → Standard_Queue; FIFO_Queue → FIFO_Queue) e residir na mesma conta e Região da AWS.
- **Monitore a DLQ.** Uma DLQ que acumula mensagens é um sinal de falhas de processamento; configure um alarme (por exemplo, sobre a métrica `ApproximateNumberOfMessagesVisible` no Amazon CloudWatch) para ser avisado quando mensagens começarem a chegar nela.

### Confirme a exclusão e trate payloads grandes com cuidado

- **Exclua a mensagem apenas após o processamento bem-sucedido.** O SQS não remove a mensagem no recebimento; a exclusão explícita com `DeleteMessage` (usando o `ReceiptHandle`) sinaliza que o processamento terminou. Excluir cedo demais (antes de concluir) pode causar perda de trabalho se o consumidor falhar em seguida.
- Para **payloads maiores que o tamanho máximo de mensagem (1 MiB)**, use armazenamento externo no Amazon S3 e envie apenas a referência na mensagem, conforme descrito na seção de Introdução / Implementação. Isso mantém a mensagem dentro do limite de tamanho e reduz o custo de tráfego de payloads grandes pela fila.
- Prefira operações **em lote** (`SendMessageBatch`, `DeleteMessageBatch`) quando processar muitas mensagens: elas reduzem o número de solicitações de API e, portanto, o custo, já que o Amazon SQS é cobrado por solicitação.

> **Nota de verificação:** o tamanho máximo de mensagem (1 MiB, aumentado a partir de 256 KiB em agosto de 2025) e a faixa do maxReceiveCount (1 a 1000) são valores que a AWS pode alterar ao longo do tempo. Confirme os valores atuais na documentação oficial da AWS.

## Limites e Cotas

Esta seção reúne os principais limites e cotas do **Amazon SQS** relevantes para o dimensionamento de mensagens e para a política de retenção. Cada linha declara o valor ou a faixa (com unidade), o valor padrão (quando aplicável), o **Tipo** — `Ajustável` (quando pode ser alterado por configuração da fila ou por solicitação de aumento de cota) ou `Fixo (hard limit)` (quando não pode ser alterado) — e a **Citação** para a fonte oficial da AWS.

| Limite/Cota | Valor / Faixa | Padrão | Tipo | Citação |
|---|---|---|---|---|
| Tamanho máximo de mensagem | 1 MiB (1.048.576 bytes) por mensagem | 1 MiB | `Fixo (hard limit)` | [Amazon SQS message quotas](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/quotas-messages.html) |
| Período de retenção de mensagens | 60 segundos a 1.209.600 segundos (14 dias) | 4 dias (345.600 segundos) | `Ajustável` (por configuração da fila, dentro da faixa) | [Amazon SQS message quotas](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/quotas-messages.html) |

Observações sobre cada limite:

- **Tamanho máximo de mensagem — 1 MiB (`Fixo (hard limit)`).** Uma única mensagem do Amazon SQS não pode exceder **1 MiB** (valor aumentado a partir dos 256 KiB anteriores, em agosto de 2025); esse é um limite rígido que **não** pode ser aumentado por solicitação de cota. **Quando o payload excede o tamanho máximo de mensagem**, use armazenamento externo no Amazon S3 e envie apenas uma referência na mensagem, conforme descrito na seção de Introdução / Implementação ([Amazon SQS message quotas](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/quotas-messages.html) e [Amazon SQS increases maximum message payload size to 1 MiB](https://aws.amazon.com/about-aws/whats-new/2025/08/amazon-sqs-max-payload-size-1mib/)).
- **Período de retenção de mensagens — 60 s a 1.209.600 s, padrão 4 dias (`Ajustável`).** O Amazon SQS mantém uma mensagem na fila até que ela seja excluída ou até expirar o período de retenção. O valor é configurável por fila dentro da faixa de **60 segundos a 1.209.600 segundos (14 dias)**, com padrão de **4 dias**; ele é ajustável dentro dessa faixa (não é uma cota de conta sujeita a solicitação de aumento além do máximo publicado) ([Amazon SQS message quotas](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/quotas-messages.html)).

> **Nota de verificação:** o tamanho máximo de mensagem (aumentado de 256 KiB para 1 MiB em agosto de 2025), a faixa e o padrão do período de retenção, bem como sua classificação como ajustável ou fixa, são valores que a AWS pode alterar ao longo do tempo. Confirme os valores atuais na documentação oficial da AWS antes de projetar restrições rígidas em torno deles.

## Segurança

Esta seção descreve os dois pilares de segurança do **Amazon SQS** relevantes para proteger o conteúdo das mensagens e controlar quem pode operar sobre as filas: a **criptografia do lado do servidor** (proteção dos dados em repouso) e o **controle de acesso** (autorização das ações sobre a fila). Ambos são independentes e complementares: a criptografia protege *o que* está armazenado, e o controle de acesso governa *quem* pode fazer *o quê*.

### Criptografia do lado do servidor (SSE)

A criptografia do lado do servidor (server-side encryption, SSE) permite transmitir dados sensíveis em filas criptografadas, protegendo o conteúdo das mensagens **em repouso**. O Amazon SQS oferece duas opções de SSE, que se diferenciam por **quem gerencia as chaves de criptografia** ([Encryption at rest in Amazon SQS](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-server-side-encryption.html)):

- **SSE-SQS — chaves gerenciadas pelo Amazon SQS.** As chaves de criptografia são criadas e gerenciadas inteiramente pelo próprio Amazon SQS, sem que você precise administrar chaves. É a forma mais simples de habilitar a criptografia em repouso. O Amazon SQS criptografa cada mensagem assim que a recebe, armazena a mensagem em forma criptografada e só a descriptografa ao entregá-la a um consumidor autorizado ([Configuring server-side encryption for a queue using SQS-managed encryption keys](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-configure-sqs-sse-queue.html)).
- **SSE-KMS — chaves gerenciadas pelo cliente no AWS KMS.** As chaves ficam sob o seu controle no **AWS Key Management Service (AWS KMS)**. Nessa opção você usa uma KMS key (uma AWS managed key para o SQS ou uma customer managed key própria), o que dá controle adicional sobre a chave: você define a política da chave, pode auditar seu uso e controlar sua rotação e permissões. O Amazon SQS integra-se ao AWS KMS para gerar as chaves de dados que efetivamente criptografam e descriptografam as mensagens ([Amazon SQS key management](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-key-management.html)).

Em resumo: escolha **SSE-SQS** quando quiser criptografia em repouso sem o overhead de gerenciar chaves; escolha **SSE-KMS** quando precisar de controle explícito sobre a chave (política de acesso, auditoria de uso e rotação), especialmente para atender a requisitos de conformidade.

> **Nota de verificação:** os detalhes de integração com o AWS KMS (por exemplo, quais KMS keys estão disponíveis e o comportamento padrão de criptografia de novas filas) são aspectos que a AWS pode alterar ao longo do tempo. Confirme o comportamento atual na documentação oficial da AWS.

### Controle de acesso (IAM e políticas de fila)

O acesso aos recursos do Amazon SQS é concedido por dois sistemas de políticas complementares: as **políticas baseadas em identidade do IAM** e as **políticas de fila baseadas em recurso** do próprio SQS. Ambas concedem permissões (quais principais podem executar quais ações em quais filas), mas diferem no ponto a que são anexadas ([Overview of managing access in Amazon SQS](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-overview-of-managing-access.html) e [Usar políticas com o Amazon SQS](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-using-identity-based-policies.html)).

- **Políticas baseadas em identidade do IAM.** São documentos de política JSON anexados a uma **identidade** — um IAM user, um grupo de usuários ou uma role. Elas governam **o que aquela identidade pode fazer**: em quais ações do SQS (por exemplo, `sqs:SendMessage`, `sqs:ReceiveMessage`, `sqs:DeleteMessage`), sobre quais filas (recursos) e sob quais condições. São a forma central de controlar, do lado do principal, quem tem permissão para operar sobre as filas ([How Amazon Simple Queue Service works with IAM](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/security_iam_service-with-iam.html)).
- **Políticas de fila baseadas em recurso.** São anexadas diretamente à **fila** (o recurso), e não a uma identidade. Elas governam **quais principais podem acessar aquela fila específica e com quais ações**, inclusive principais de **outras contas da AWS** (acesso entre contas) ou principais de serviço da AWS. Assim, uma política de fila é o mecanismo natural para conceder, por exemplo, que uma fila receba mensagens de outra conta ou de um serviço como o Amazon SNS ([Overview of managing access in Amazon SQS](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-overview-of-managing-access.html)).

O que cada uma governa, em resumo: a **política de identidade do IAM** responde “*a que filas e ações esta identidade tem acesso?*” (perspectiva do principal), enquanto a **política de fila baseada em recurso** responde “*quem pode acessar esta fila e como?*” (perspectiva do recurso). Em uma mesma conta, uma ação costuma ser permitida quando qualquer um dos sistemas a concede; para acesso entre contas, a política de fila baseada em recurso é o ponto em que o proprietário da fila autoriza principais externos. Em ambos os casos, aplique o princípio do **privilégio mínimo**, concedendo apenas as ações e os recursos estritamente necessários.

## Considerações de Preço

O **Amazon SQS** adota um modelo de preço de **pagamento por solicitação (pay-per-request)**: você não paga uma taxa fixa nem provisiona capacidade antecipadamente; paga apenas pelo que usa, com base no número de solicitações feitas ao serviço. Não há custo mínimo nem compromisso inicial, e a cobrança escala de acordo com o volume de tráfego das suas filas ([Amazon SQS pricing](https://aws.amazon.com/sqs/pricing/)).

### Principais fatores de custo

Dois fatores dominam o custo de uma solução baseada em Amazon SQS:

- **Número de solicitações de API.** A unidade de cobrança é a **solicitação** (request) — cada chamada às ações da API do SQS, como `SendMessage`, `ReceiveMessage` e `DeleteMessage`, conta como uma solicitação faturável. É importante notar que a cobrança é medida em **blocos de 64 KB de payload**: cada bloco de 64 KB é contabilizado como uma solicitação (por exemplo, uma chamada com um payload de 256 KB equivale a quatro solicitações de 64 KB para fins de cobrança). Além disso, o Amazon SQS **agrupa** ações em lote (`SendMessageBatch`, `DeleteMessageBatch`) e chamadas de long polling de forma a reduzir o número de solicitações — por isso, práticas como **operações em lote** e **long polling** reduzem diretamente o custo, ao diminuírem a contagem total de solicitações ([Amazon SQS pricing](https://aws.amazon.com/sqs/pricing/)).

- **Tratamento de payloads grandes.** Mensagens maiores geram custo maior por dois caminhos. Primeiro, dentro do SQS, como cada 64 KB de payload é cobrado como uma solicitação, mensagens maiores consomem mais solicitações faturáveis. Segundo, **quando o payload excede o tamanho máximo de mensagem (1 MiB)**, ele não cabe em uma única mensagem e precisa ser armazenado externamente no **Amazon S3** (padrão descrito na seção de Introdução / Implementação). Nesse caso, incorrem os **custos próprios do Amazon S3** — armazenamento dos objetos, solicitações `PUT`/`GET` e transferência de dados — que se somam ao custo do SQS. Ou seja, o tratamento de payloads grandes desloca parte do custo para o serviço de armazenamento externo ([Amazon SQS pricing](https://aws.amazon.com/sqs/pricing/) e [Amazon S3 pricing](https://aws.amazon.com/s3/pricing/)).

Há ainda diferenças de preço por **tipo de fila**: as solicitações a uma **FIFO_Queue** têm preço distinto do das solicitações a uma **Standard_Queue**, e os custos usuais de **transferência de dados** da AWS podem se aplicar ao tráfego que sai da Região da AWS. Consulte a página de preços para os valores por tipo de fila e por Região ([Amazon SQS pricing](https://aws.amazon.com/sqs/pricing/)).

Em resumo, para controlar o custo do Amazon SQS: minimize o número de solicitações (use operações em lote e long polling em vez de short polling) e mantenha os payloads pequenos, recorrendo ao armazenamento externo no Amazon S3 apenas quando o payload exceder o tamanho máximo de mensagem (1 MiB) — ciente de que isso adiciona os custos do próprio Amazon S3.

> **Nota de verificação:** os preços do Amazon SQS (valor por milhão de solicitações, preço por tipo de fila, cota do nível gratuito, tamanho do bloco de payload faturável e custos de transferência de dados) e os preços do Amazon S3 são valores que a AWS pode alterar ao longo do tempo e variam por Região da AWS. Confirme os valores atuais na página oficial de preços do Amazon SQS ([aws.amazon.com/sqs/pricing](https://aws.amazon.com/sqs/pricing/)) e na documentação oficial da AWS antes de estimar custos ou projetar restrições rígidas em torno deles.
