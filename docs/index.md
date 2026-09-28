# Documentação de Referência: Amazon SQS, AWS Lambda e Integração

## Introdução

Esta documentação de referência reúne, em um único conjunto, os conceitos, as orientações de implementação, os limites, os controles de segurança e o modelo de preço do Amazon SQS e do AWS Lambda, além do padrão de integração que conecta os dois serviços (o SQS como fonte de eventos do Lambda).

**Público-alvo.** Este material destina-se a **desenvolvedores e arquitetos** que estão aprendendo ou consultando esses serviços. O conteúdo pressupõe familiaridade geral com a AWS, mas não com os serviços específicos aqui cobertos.

**Escopo de cobertura.** Para cada serviço, a documentação cobre seis áreas: conceitos principais, introdução/implementação, dicas e observações importantes, limites e cotas, segurança e considerações de preço. Para o padrão de integração, cobre o Event_Source_Mapping e o batching, além do tratamento de erros e da idempotência. Todos os fatos sobre limites, cotas, preço e comportamento são fundamentados na documentação oficial da AWS. Esta documentação **não** é código de aplicação; é material de referência escrito.

**Serviços cobertos.** Esta documentação cobre os seguintes serviços e padrões da AWS:

1. **Amazon SQS** — serviço de filas de mensagens totalmente gerenciado para desacoplar e escalar componentes de sistemas distribuídos.
2. **AWS Lambda** — serviço de computação serverless que executa código em resposta a eventos, sem provisionar ou gerenciar servidores.
3. **Integração SQS-para-Lambda** — o padrão em que uma fila do Amazon SQS atua como fonte de eventos para uma função AWS Lambda.

## Índice

### 1. [SQS_Document](sqs.md)

- [Conceitos Principais](sqs.md#conceitos-principais)
- [Introdução / Implementação](sqs.md#introdução--implementação)
- [Dicas e Observações Importantes](sqs.md#dicas-e-observações-importantes)
- [Limites e Cotas](sqs.md#limites-e-cotas)
- [Segurança](sqs.md#segurança)
- [Considerações de Preço](sqs.md#considerações-de-preço)

### 2. [Lambda_Document](lambda.md)

- [Conceitos Principais](lambda.md#conceitos-principais)
- [Introdução / Implementação](lambda.md#introdução--implementação)
- [Dicas e Observações Importantes](lambda.md#dicas-e-observações-importantes)
- [Limites e Cotas](lambda.md#limites-e-cotas)
- [Segurança](lambda.md#segurança)
- [Considerações de Preço](lambda.md#considerações-de-preço)

### 3. [Integration_Document](integracao-sqs-lambda.md)

- [Event_Source_Mapping e Batching](integracao-sqs-lambda.md#event_source_mapping-e-batching)
- [Tratamento de Erros e Idempotência](integracao-sqs-lambda.md#tratamento-de-erros-e-idempotência)

### Referência

- [Glossário](glossario.md)
