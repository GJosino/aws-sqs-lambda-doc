# AWS Lambda

## Conceitos Principais

O **AWS Lambda** é um serviço de computação *serverless* orientado a eventos: você fornece o código e o Lambda o executa em resposta a eventos, sem que você precise provisionar, escalar ou gerenciar servidores. A infraestrutura de execução (capacidade, sistema operacional, aplicação de patches e escalonamento) é totalmente administrada pela AWS, deixando o Leitor responsável apenas pelo código da função e por sua configuração ([What is AWS Lambda?](https://docs.aws.amazon.com/lambda/latest/dg/welcome.html)).

### Handler, fontes de eventos/gatilhos e ambiente de execução

- **Handler.** É o método do seu código que o Lambda invoca a cada evento. O Lambda passa ao handler um objeto de evento (o payload que descreve o que aconteceu) e um objeto de contexto (informações de tempo de execução), e o valor retornado pelo handler é a resposta da invocação ([Lambda function handler](https://docs.aws.amazon.com/lambda/latest/dg/foundation-progmodel.html)).
- **Fontes de eventos / gatilhos.** Um gatilho é o recurso ou a configuração que faz uma função ser invocada em resposta a um evento. As fontes de eventos podem ser outros serviços da AWS (por exemplo, uma fila do **Amazon SQS**, um bucket do Amazon S3 ou uma API do Amazon API Gateway) ou invocações diretas. No caso do SQS, o polling e a invocação são intermediados por um **Event_Source_Mapping** ([Using AWS Lambda with other services](https://docs.aws.amazon.com/lambda/latest/dg/lambda-services.html)).
- **Ambiente de execução (execution environment).** É o ambiente isolado e seguro no qual a função roda. Ele gerencia o ciclo de vida da função: inicializa o runtime, executa o código de inicialização, invoca o handler e, quando ocioso, pode ser congelado para reutilização em invocações futuras ([Understanding the Lambda execution environment lifecycle](https://docs.aws.amazon.com/lambda/latest/dg/lambda-runtime-environment.html)).

### Modelo de concorrência

Cada ambiente de execução processa **uma solicitação por vez**. Quando a demanda aumenta e não há ambientes disponíveis, o Lambda **escala criando ambientes de execução concorrentes adicionais**, cada um atendendo uma invocação simultânea. A concorrência é, portanto, o número de invocações que a função processa ao mesmo tempo, e o Lambda a ajusta automaticamente conforme o volume de eventos ([Understanding Lambda function scaling](https://docs.aws.amazon.com/lambda/latest/dg/lambda-concurrency.html)).

### Cold start vs. warm start

- **Cold start.** Ocorre quando uma invocação precisa de um ambiente de execução novo. O Lambda provisiona o ambiente, carrega o runtime e executa o código de inicialização antes de chamar o handler, o que **adiciona latência de inicialização** àquela invocação.
- **Warm start.** Ocorre quando uma invocação **reutiliza um ambiente de execução existente** que já foi inicializado. Como as etapas de inicialização são puladas, a invocação não paga a latência adicional de partida.

Após uma invocação, o Lambda pode manter o ambiente congelado por um tempo para reaproveitá-lo em invocações subsequentes, o que favorece os warm starts ([Understanding the Lambda execution environment lifecycle](https://docs.aws.amazon.com/lambda/latest/dg/lambda-runtime-environment.html)).

### Duração máxima de invocação e timeout padrão

O *timeout* é o tempo máximo, em segundos, que uma função pode executar antes de ser encerrada. O valor **padrão é 3 segundos** e pode ser ajustado, em incrementos de 1 segundo, até a **duração máxima de invocação de 900 segundos (15 minutos)** ([Configure Lambda function timeout](https://docs.aws.amazon.com/lambda/latest/dg/configuration-timeout.html)).

> **Nota de verificação:** o timeout padrão (3 s) e a duração máxima de invocação (900 s / 15 min) são valores que a AWS pode alterar ao longo do tempo. Confirme os valores atuais na documentação oficial da AWS antes de projetar dentro dessas restrições.

## Introdução / Implementação

Esta seção apresenta uma sequência ordenada de passos para criar, configurar, implantar e invocar sua primeira função no **AWS Lambda**. Cada passo declara a ação a executar e o resultado observável que confirma que ele foi bem-sucedido (Modelo 4). Os exemplos usam a AWS CLI e o runtime Python; adapte os valores entre `<...>` ao seu ambiente.

### Passos numerados

**Passo 1 — Criar a função.** Ação: criar uma função informando um nome, o runtime, o handler e o papel de execução (execution role) que concede as permissões necessárias. Entradas: nome da função, runtime, handler (`arquivo.metodo`), ARN do papel de execução e o pacote de implantação (arquivo `.zip`).

```bash
aws lambda create-function \
  --function-name minha-funcao \
  --runtime python3.13 \
  --handler lambda_function.lambda_handler \
  --role arn:aws:iam::<ACCOUNT_ID>:role/<EXECUTION_ROLE> \
  --zip-file fileb://funcao.zip
```

Resultado observável esperado: a resposta contém o `FunctionArn`, o campo `State` com valor `Active` e o `LastUpdateStatus` como `Successful`, confirmando que a função foi criada e está pronta para invocação.

> **Nota de verificação:** o identificador de runtime usado no exemplo (`python3.13`) e a própria lista de runtimes suportados são valores que a AWS pode alterar ao longo do tempo (novas versões são adicionadas e versões antigas são descontinuadas). Confirme o runtime suportado e vigente na documentação oficial da AWS ([Lambda runtimes](https://docs.aws.amazon.com/lambda/latest/dg/lambda-runtimes.html)) antes de criar a função.

**Passo 2 — Configurar o gatilho (trigger).** Ação: associar uma fonte de eventos à função. Para uma fila do **Amazon SQS**, cria-se um **Event_Source_Mapping**, que faz o Lambda fazer polling da fila e invocar a função com lotes de mensagens (ver o Integration_Document para o detalhamento). Entradas: ARN da fonte de eventos e nome da função.

```bash
aws lambda create-event-source-mapping \
  --function-name minha-funcao \
  --event-source-arn arn:aws:sqs:<REGIAO>:<ACCOUNT_ID>:minha-fila \
  --batch-size 10
```

Resultado observável esperado: a resposta contém um `UUID` para o mapeamento e o campo `State` evoluindo para `Enabled`, indicando que o gatilho está ativo e fará polling da fila.

**Passo 3 — Implantar o código.** Ação: atualizar o código da função enviando um novo pacote de implantação. Entradas: nome da função e o arquivo `.zip` com o código do handler.

```bash
aws lambda update-function-code \
  --function-name minha-funcao \
  --zip-file fileb://funcao.zip
```

Resultado observável esperado: a resposta traz `LastUpdateStatus` igual a `Successful` e o `CodeSha256` atualizado, confirmando que a nova versão do código foi implantada.

**Passo 4 — Invocar a função.** Ação: acionar a função diretamente para verificar seu comportamento, gravando a resposta em um arquivo. Entradas: nome da função, payload de evento e o arquivo de saída.

```bash
aws lambda invoke \
  --function-name minha-funcao \
  --payload '{"nome": "mundo"}' \
  --cli-binary-format raw-in-base64-out \
  resposta.json
```

Resultado observável esperado: a resposta da CLI mostra `StatusCode` igual a `200` e **não** contém o campo `FunctionError`; o arquivo `resposta.json` contém o valor retornado pelo handler (por exemplo, `{"statusCode": 200, "body": "Olá, mundo"}`).

### Exemplo de handler completo

O exemplo a seguir mostra um handler completo em um runtime suportado (Python), incluindo a **assinatura** da função (`event`, `context`), o **parâmetro de entrada** (`event`, o payload recebido) e o **valor de resposta retornado** ([Lambda function handler in Python](https://docs.aws.amazon.com/lambda/latest/dg/python-handler.html)):

```python
# lambda_function.py
def lambda_handler(event, context):
    # event: parâmetro de entrada com o payload da invocação
    # context: informações de tempo de execução (request ID, tempo restante etc.)
    nome = event.get("nome", "mundo")

    # Valor de resposta retornado pelo handler:
    return {
        "statusCode": 200,
        "body": f"Olá, {nome}"
    }
```

Para o payload de entrada `{"nome": "mundo"}`, o valor de resposta retornado é:

```json
{
  "statusCode": 200,
  "body": "Olá, mundo"
}
```

### Configuração de memória e timeout

- **Memória.** A memória é configurável entre **128 MB e 10.240 MB, em incrementos de 1 MB**. O Lambda aloca CPU (e outros recursos) de forma proporcional à memória: quanto mais memória, mais poder de CPU a função recebe. A partir de aproximadamente **1.769 MB**, a função dispõe do equivalente a **uma vCPU** ([Configure Lambda function memory](https://docs.aws.amazon.com/lambda/latest/dg/configuration-memory.html)). Aumentar a memória pode, portanto, **reduzir a duração** de tarefas limitadas por CPU e, em alguns casos, diminuir o custo total mesmo com preço por GB-second maior.
- **Timeout.** O timeout padrão é **3 segundos** e pode ir até a duração máxima de invocação de **900 segundos (15 minutos)**, em incrementos de 1 segundo ([Configure Lambda function timeout](https://docs.aws.amazon.com/lambda/latest/dg/configuration-timeout.html)).

Comando de exemplo para ajustar ambos:

```bash
aws lambda update-function-configuration \
  --function-name minha-funcao \
  --memory-size 512 \
  --timeout 30
```

**Efeito no custo (memória × duração).** O preço do Lambda é função do número de solicitações e da **duração de computação em GB-seconds** — a memória configurada (em GB) multiplicada pelo tempo de execução (em segundos) ([AWS Lambda Pricing](https://aws.amazon.com/lambda/pricing/)). Assim, dobrar a memória dobra o custo por unidade de tempo, mas se isso reduzir a duração pela metade (por mais CPU), o custo em GB-seconds pode permanecer estável — daí a importância de dimensionar (right-sizing) a memória por medição.

> **Nota de verificação:** faixa de memória (128 MB–10.240 MB), ponto de equivalência de vCPU (~1.769 MB), timeout padrão (3 s), duração máxima (900 s) e o modelo de preço são valores que a AWS pode alterar ao longo do tempo. Confirme os valores atuais na documentação oficial da AWS.

### Logs e métricas

- **Logs (fonte de log).** A função envia automaticamente os logs de execução para o **Amazon CloudWatch Logs**, em um log group nomeado `/aws/lambda/<nome-da-funcao>`. Ali você encontra a saída de `print`/logging do código e as linhas de sistema `START`, `END` e `REPORT` (esta última traz duração, memória configurada e memória máxima usada) ([Sending Lambda function logs to CloudWatch Logs](https://docs.aws.amazon.com/lambda/latest/dg/monitoring-cloudwatchlogs.html)). Para acompanhar em tempo real pela CLI:

```bash
aws logs tail /aws/lambda/minha-funcao --follow
```

- **Métricas (métrica disponível).** O Lambda publica métricas no **Amazon CloudWatch**, entre elas `Invocations` (número de invocações), `Errors` (invocações com falha), `Duration` (tempo de execução) e `Throttles` (invocações limitadas por concorrência). Use a métrica `Errors` combinada com `Invocations` para monitorar a taxa de falhas ([Working with Lambda function metrics](https://docs.aws.amazon.com/lambda/latest/dg/monitoring-metrics.html)).

### Solução de problemas

| Passo | Condição de falha | Ação corretiva |
|---|---|---|
| Criação (Passo 1) | A criação falha com erro de permissão sobre o papel (ex.: `InvalidParameterValueException` mencionando o role ou `AccessDenied`). | Verifique se o ARN do papel de execução existe e se sua identidade tem permissão `iam:PassRole` para esse papel; confirme que a política de confiança (trust policy) do papel permite o serviço `lambda.amazonaws.com`. |
| Criação (Passo 1) | A criação falha com `InvalidParameterValueException` sobre o handler ou o runtime. | Confirme que o handler está no formato `arquivo.metodo` correspondente ao código empacotado e que o valor de `--runtime` é um runtime suportado e vigente. |
| Invocação (Passo 4) | A invocação retorna `StatusCode` 200 porém com o campo `FunctionError` (erro dentro do handler). | Abra os logs em `/aws/lambda/<nome-da-funcao>` no Amazon CloudWatch Logs para ler o stack trace, corrija o código e reimplante (Passo 3). |
| Invocação (Passo 4) | A invocação falha com `Task timed out after N seconds`. | Aumente o timeout com `update-function-configuration` (até 900 s) e/ou aumente a memória para obter mais CPU; otimize o código para concluir dentro do limite. |
| Invocação (Passo 4) | A invocação retorna erro de acesso a outro serviço da AWS (ex.: `AccessDeniedException`). | Adicione ao papel de execução apenas as permissões específicas exigidas (privilégio mínimo) para o serviço acessado e invoque novamente. |

## Dicas e Observações Importantes

Esta seção reúne recomendações de desempenho e advertências operacionais para o uso do **AWS Lambda** no dia a dia. Elas complementam os conceitos e os passos de implementação apresentados nas seções anteriores, ajudando o Leitor a dimensionar recursos de forma consciente, reduzir latência e evitar armadilhas comuns de custo e correção.

### Dimensione a memória por medição (right-sizing)

A alocação de memória é o principal "botão" de desempenho do Lambda: a CPU (e a banda de rede e I/O) é atribuída de forma **proporcional à memória** configurada, entre 128 MB e 10.240 MB. Assim, aumentar a memória entrega mais poder de CPU e pode **reduzir a duração** de funções limitadas por processamento ([Configure Lambda function memory](https://docs.aws.amazon.com/lambda/latest/dg/configuration-memory.html)).

- Não dimensione "no olho". Meça a duração e o uso real de memória (a linha `REPORT` no Amazon CloudWatch Logs traz `Memory Size` e `Max Memory Used`) e ajuste a partir de dados.
- Como o preço é medido em **GB-seconds** (memória em GB × tempo de execução), dobrar a memória dobra o custo por unidade de tempo; mas se isso reduzir a duração pela metade em uma tarefa limitada por CPU, o custo em GB-seconds pode permanecer estável — e a latência cai. Por isso, o menor valor de memória nem sempre é o mais barato.
- Para automatizar a busca do ponto ótimo entre custo e desempenho, considere a ferramenta **AWS Lambda Power Tuning** ([Profiling functions with AWS Lambda Power Tuning](https://docs.aws.amazon.com/lambda/latest/dg/best-practices.html)).

> **Nota de verificação:** a faixa de memória (128 MB–10.240 MB) e o ponto de equivalência de uma vCPU (~1.769 MB) são valores que a AWS pode alterar ao longo do tempo. Confirme os valores atuais na documentação oficial da AWS.

### Defina o timeout pelo comportamento esperado, não pelo máximo

O *timeout* deve refletir o tempo de execução **esperado** da função, com uma margem razoável — não o máximo de 900 segundos (15 minutos) por padrão ([Configure Lambda function timeout](https://docs.aws.amazon.com/lambda/latest/dg/configuration-timeout.html)):

- Um timeout **curto demais** encerra invocações legítimas com `Task timed out`, causando falhas e (em fontes de eventos como o **Amazon SQS**) reprocessamento.
- Um timeout **longo demais** deixa uma função travada (por exemplo, aguardando uma dependência lenta) consumindo tempo faturável até o limite, aumentando o custo e retardando a detecção do problema.
- Configure timeouts de chamadas a serviços e dependências **menores** que o timeout da função, para falhar de forma controlada e liberar o ambiente antes do encerramento abrupto.
- Ao usar o Lambda como consumidor do **Amazon SQS**, o **Visibility_Timeout** da fila deve ser maior ou igual ao timeout da função (a recomendação é de pelo menos 6× o timeout); veja o Integration_Document para o detalhamento.

> **Nota de verificação:** o timeout padrão (3 s) e a duração máxima de invocação (900 s / 15 min) são valores que a AWS pode alterar ao longo do tempo. Confirme os valores atuais na documentação oficial da AWS.

### Aproveite o reuso do ambiente de execução (cold start vs. warm start)

O Lambda pode **reutilizar** um ambiente de execução já inicializado em invocações subsequentes (warm start), pulando as etapas de inicialização e evitando a latência de um cold start ([Understanding the Lambda execution environment lifecycle](https://docs.aws.amazon.com/lambda/latest/dg/lambda-runtime-environment.html)). Estruture o código para tirar proveito disso:

- **Inicialize fora do handler.** Crie clientes de SDK, pools de conexão e carregue configuração no escopo global (código de inicialização), não dentro do handler. Esses objetos são reutilizados enquanto o ambiente permanecer aquecido, reduzindo a latência das invocações seguintes ([Best practices for working with AWS Lambda functions](https://docs.aws.amazon.com/lambda/latest/dg/best-practices.html)).
- **Não guarde estado entre invocações como garantia.** O reuso é oportunista: um ambiente pode ser descartado a qualquer momento e cada nova invocação concorrente cria um novo ambiente (com seu próprio cold start). Não dependa de variáveis globais para persistir dados de negócio — a única fonte durável é um serviço externo (banco de dados, Amazon S3 etc.).
- **Reaproveite o Ephemeral storage com cautela.** O diretório `/tmp` (Ephemeral storage) também pode persistir entre invocações no mesmo ambiente; ele serve como cache temporário, mas não como armazenamento confiável, pois seu conteúdo desaparece quando o ambiente é reciclado.
- **Mitigue cold starts quando a latência for crítica.** Prefira pacotes de implantação enxutos e código de inicialização leve. Para cargas sensíveis à latência de partida, avalie a **provisioned concurrency**, que mantém ambientes pré-inicializados e prontos para responder ([Managing Lambda provisioned concurrency](https://docs.aws.amazon.com/lambda/latest/dg/provisioned-concurrency.html)).

### Escreva funções idempotentes e sem estado

Como cada ambiente processa **uma solicitação por vez** e o Lambda escala criando ambientes concorrentes, trate cada invocação como isolada e sem estado compartilhado. Além disso, fontes de eventos como o **Amazon SQS** entregam **pelo menos uma vez** (at-least-once), o que torna o reprocessamento da mesma mensagem possível. Projete o handler para ser **idempotente**, de modo que processar a mesma entrada mais de uma vez não gere efeitos colaterais indevidos (ver o Integration_Document).

### Observe a concorrência e proteja dependências

O Lambda escala automaticamente criando ambientes concorrentes conforme a demanda, mas isso pode **sobrecarregar recursos a jusante** (bancos de dados, APIs de terceiros) que não escalam na mesma velocidade:

- Monitore as métricas `Throttles` e `ConcurrentExecutions` no Amazon CloudWatch para detectar limitação por concorrência.
- Use **reserved concurrency** para limitar o número de execuções concorrentes de uma função e proteger dependências frágeis contra picos de carga ([Configuring reserved concurrency](https://docs.aws.amazon.com/lambda/latest/dg/configuration-concurrency.html)).

> **Nota de verificação:** o limite padrão de execuções concorrentes por conta é um valor que a AWS pode alterar ao longo do tempo e varia por Região. Confirme o valor atual na documentação oficial da AWS.

## Limites e Cotas

Esta seção reúne as cotas principais do **AWS Lambda** relevantes para dimensionar funções e projetar dentro de suas restrições. A tabela a seguir (Modelo 5) declara, para cada cota, o valor ou a faixa com unidade, o valor padrão (quando aplicável) e se a cota é **`Ajustável`** (pode ser aumentada via solicitação de aumento de cota de serviço no AWS Service Quotas) ou **`Fixo (hard limit)`** (não pode ser alterada).

| Limite/Cota | Valor / Faixa | Padrão | Tipo | Citação |
|---|---|---|---|---|
| Duração máxima de invocação (timeout) | Até 900 s (15 min), em incrementos de 1 s | 3 s | `Fixo (hard limit)` | [Configure Lambda function timeout](https://docs.aws.amazon.com/lambda/latest/dg/configuration-timeout.html) |
| Alocação de memória da função | 128 MB a 10.240 MB, em incrementos de 1 MB | 128 MB | `Fixo (hard limit)` | [Configure Lambda function memory](https://docs.aws.amazon.com/lambda/latest/dg/configuration-memory.html) |
| Ephemeral storage (`/tmp`) | 512 MB a 10.240 MB, em incrementos de 1 MB | 512 MB | `Fixo (hard limit)` | [Configure ephemeral storage for Lambda functions](https://docs.aws.amazon.com/lambda/latest/dg/configuration-ephemeral-storage.html) |
| Execuções concorrentes por conta (por Região) | 1.000 (limite padrão inicial) | 1.000 | `Ajustável` | [Lambda quotas](https://docs.aws.amazon.com/lambda/latest/dg/gettingstarted-limits.html) |

### Detalhamento das cotas

- **Duração máxima de invocação (timeout).** O tempo máximo de execução de uma função é limitado a **900 segundos (15 minutos)**; esse teto é um **`Fixo (hard limit)`** e não pode ser aumentado por solicitação de cota. O valor configurável de timeout vai do **padrão de 3 segundos** até esse máximo, em incrementos de 1 segundo ([Configure Lambda function timeout](https://docs.aws.amazon.com/lambda/latest/dg/configuration-timeout.html)).
- **Alocação de memória.** A memória é configurável na faixa de **128 MB a 10.240 MB**, em incrementos de 1 MB. Os extremos dessa faixa constituem um **`Fixo (hard limit)`**. A CPU é atribuída de forma proporcional à memória ([Configure Lambda function memory](https://docs.aws.amazon.com/lambda/latest/dg/configuration-memory.html)).
- **Ephemeral storage.** O diretório temporário `/tmp` (Ephemeral storage) tem tamanho configurável de **512 MB a 10.240 MB**, sendo **512 MB** o padrão. Os limites da faixa são um **`Fixo (hard limit)`** ([Configure ephemeral storage for Lambda functions](https://docs.aws.amazon.com/lambda/latest/dg/configuration-ephemeral-storage.html)).
- **Execuções concorrentes por conta.** O número de execuções concorrentes por conta, por Região da AWS, tem um **limite padrão inicial de 1.000**. Diferentemente das cotas anteriores, esse limite é **`Ajustável`**: pode ser aumentado abrindo uma solicitação de aumento de cota de serviço no AWS Service Quotas ([Lambda quotas](https://docs.aws.amazon.com/lambda/latest/dg/gettingstarted-limits.html)).

> **Nota de verificação:** o timeout padrão (3 s) e a duração máxima (900 s / 15 min), as faixas de memória (128 MB–10.240 MB) e de Ephemeral storage (512 MB–10.240 MB), bem como o limite padrão de execuções concorrentes por conta (1.000) e sua ajustabilidade, são valores que a AWS pode alterar ao longo do tempo e que podem variar por Região. Confirme os valores atuais na documentação oficial da AWS antes de projetar dentro dessas restrições.

## Segurança

Esta seção descreve os controles de segurança fundamentais de uma função no **AWS Lambda**: o papel de execução (execution role), que governa a quais serviços da AWS a função pode acessar, e o tratamento de segredos e variáveis de ambiente, que determina como valores de configuração — sensíveis ou não — são referenciados pelo código. Ambos devem ser configurados segundo o princípio do privilégio mínimo, concedendo apenas o acesso estritamente necessário.

### Papel de execução (execution role)

O **papel de execução** é uma função (role) do IAM que a função Lambda assume ao ser invocada. É esse papel que **concede à função permissão para acessar outros serviços da AWS** — por exemplo, ler mensagens de uma fila do **Amazon SQS**, gravar objetos no Amazon S3, publicar logs no Amazon CloudWatch Logs ou ler um segredo no AWS Secrets Manager. Sem uma permissão correspondente no papel de execução, a chamada da função ao serviço é negada (tipicamente com `AccessDeniedException`) ([Lambda execution role](https://docs.aws.amazon.com/lambda/latest/dg/lambda-intro-execution-role.html)).

O papel tem dois lados complementares:

- **Política de confiança (trust policy).** Autoriza o serviço `lambda.amazonaws.com` a assumir o papel. É o que permite ao próprio Lambda usar o papel em nome da sua função.
- **Políticas de permissão (permission policies).** Definem *quais ações* em *quais recursos* a função pode executar. São essas políticas que você deve manter enxutas.

Aplique o **privilégio mínimo** (least privilege): conceda ao papel de execução **apenas as permissões específicas de que a função realmente precisa**, restritas aos recursos exatos (ARNs), em vez de permissões amplas como `*` em ações ou recursos ([Security best practices in IAM](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html)). Por exemplo, uma função que apenas consome de uma fila específica do SQS deve receber somente as ações necessárias (como `sqs:ReceiveMessage`, `sqs:DeleteMessage` e `sqs:GetQueueAttributes`) sobre o ARN daquela fila:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "sqs:ReceiveMessage",
        "sqs:DeleteMessage",
        "sqs:GetQueueAttributes"
      ],
      "Resource": "arn:aws:sqs:<REGIAO>:<ACCOUNT_ID>:minha-fila"
    }
  ]
}
```

Restringir o papel dessa forma limita o raio de impacto (blast radius) caso o código seja comprometido: a função só pode fazer aquilo que suas permissões explícitas autorizam.

### Segredos e variáveis de ambiente

As **variáveis de ambiente** permitem parametrizar o comportamento da função (endpoints, nomes de recursos, flags) sem alterar o código. O Lambda **criptografa as variáveis de ambiente em repouso** (encrypted at rest) usando uma chave do AWS KMS; por padrão é uma chave gerenciada pela AWS, mas você pode informar uma chave gerenciada pelo cliente (customer managed key) para controlar a criptografia e, opcionalmente, habilitar criptografia adicional em trânsito no console ([Using Lambda environment variables](https://docs.aws.amazon.com/lambda/latest/dg/configuration-envvars.html)).

Ainda assim, **valores sensíveis** — senhas, chaves de API, strings de conexão de banco de dados, tokens — **não devem ser armazenados em variáveis de ambiente em texto plano**. Mesmo criptografadas em repouso, essas variáveis ficam visíveis em texto claro para quem tiver permissão de leitura da configuração da função (console, `GetFunctionConfiguration`), aparecem em exportações de infraestrutura e são de difícil rotação. Em vez disso, armazene segredos em um **serviço dedicado de gerenciamento de segredos** e recupere-os em tempo de execução ([Security best practices in Lambda](https://docs.aws.amazon.com/lambda/latest/dg/security-best-practices.html)):

- **AWS Secrets Manager.** Armazena segredos criptografados, controla o acesso por IAM e oferece **rotação automática** de credenciais. Indicado para senhas de banco de dados e chaves de API que precisam ser rotacionadas ([What is AWS Secrets Manager?](https://docs.aws.amazon.com/secretsmanager/latest/userguide/intro.html)).
- **AWS Systems Manager Parameter Store.** Guarda parâmetros de configuração, com a opção `SecureString` para valores sensíveis criptografados via AWS KMS ([AWS Systems Manager Parameter Store](https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-parameter-store.html)).

Nesse modelo, a variável de ambiente guarda apenas uma **referência não sensível** (o nome ou ARN do segredo), e a função busca o valor no serviço de segredos durante a execução — usando, para isso, permissões concedidas pelo **papel de execução** segundo o privilégio mínimo (por exemplo, `secretsmanager:GetSecretValue` restrito ao ARN daquele segredo). Para reduzir latência e chamadas repetidas, é possível recuperar o segredo no código de inicialização (fora do handler) e reaproveitá-lo entre invocações no mesmo ambiente de execução (ver "Dicas e Observações Importantes").

> **Nota de verificação:** os detalhes de criptografia das variáveis de ambiente (chave padrão do AWS KMS, opções de chave gerenciada pelo cliente) e os recursos dos serviços de gerenciamento de segredos (AWS Secrets Manager, AWS Systems Manager Parameter Store) são aspectos que a AWS pode alterar ao longo do tempo. Confirme os comportamentos e as opções atuais na documentação oficial da AWS.

## Considerações de Preço

O preço do **AWS Lambda** é baseado no uso: você paga apenas pelo que consome, sem custo fixo por ter uma função ociosa. O custo é composto por dois fatores principais — o **número de solicitações** (requests) e a **duração de computação**, medida em **GB-seconds** ([AWS Lambda Pricing](https://aws.amazon.com/lambda/pricing/)).

### Número de solicitações

Cada invocação da função conta como uma solicitação, independentemente da fonte (invocação direta, gatilho de outro serviço da AWS ou polling de um **Event_Source_Mapping**). O Lambda cobra um valor por milhão de solicitações, e todas as invocações contam para esse total — inclusive as que terminam em erro dentro do handler ([AWS Lambda Pricing](https://aws.amazon.com/lambda/pricing/)).

### Duração de computação em GB-seconds

A duração de computação é medida em **GB-seconds** e corresponde à **memória configurada da função (em GB) multiplicada pelo tempo de execução (em segundos)**:

```text
GB-seconds = (memória configurada em GB) × (tempo de execução faturável em segundos)
```

Como a memória é configurada em MB, converte-se para GB dividindo por 1.024. Por exemplo, uma função com **512 MB** (≈ 0,5 GB) que executa por **2 segundos** consome aproximadamente `0,5 GB × 2 s = 1 GB-second` de duração. É por isso que a memória e a duração de execução, juntas, determinam o custo de computação (o efeito de memória × duração já foi discutido na seção "Introdução / Implementação"): aumentar a memória eleva o custo por unidade de tempo, mas pode reduzir a duração de tarefas limitadas por CPU e, com isso, manter ou baixar o custo total em GB-seconds ([AWS Lambda Pricing](https://aws.amazon.com/lambda/pricing/), [Configure Lambda function memory](https://docs.aws.amazon.com/lambda/latest/dg/configuration-memory.html)).

### Unidades e arredondamento do tempo de execução faturável

- **Unidade de tempo faturável.** O tempo de execução é faturado em **milissegundos (ms)**, medido desde o momento em que o código começa a executar até o retorno do handler ou o encerramento da invocação por outro motivo.
- **Arredondamento.** O tempo de execução faturável é **arredondado para cima até o milissegundo (1 ms) mais próximo**. Assim, uma invocação que dura 1,3 ms é faturada como 2 ms.
- **Unidade de duração de computação.** A duração de computação resultante é expressa em **GB-seconds** (memória em GB × tempo faturável em segundos), que é a base sobre a qual o preço por GB-second é aplicado.

Essas unidades e o comportamento de arredondamento estão descritos na página oficial de preços ([AWS Lambda Pricing](https://aws.amazon.com/lambda/pricing/)).

> **Nota de verificação:** o modelo de preço (fatores de custo, unidade de faturamento em milissegundos, arredondamento para cima até 1 ms, cálculo em GB-seconds), os valores por solicitação e por GB-second, bem como a existência de uma camada gratuita (free tier), são aspectos que a AWS pode alterar ao longo do tempo e que variam por Região da AWS. Confirme os valores e as regras de faturamento atuais na documentação oficial da AWS ([AWS Lambda Pricing](https://aws.amazon.com/lambda/pricing/)) antes de estimar custos.
