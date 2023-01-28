# Introduzindo concorrência máxima para funções AWS Lambda com Amazon SQS como fonte de eventos

O [AWS Lambda](https://aws.amazon.com/lambda/) agora oferece uma maneira de controlar o número máximo de funções concorrentes invocadas pelo Amazon SQS como fonte de eventos. Você pode usar esse recurso para controlar a concorrência das funções AWS Lambda processando mensagens em filas SQS individuais.

Este artigo descreve como definir a concorrência máxima dos triggers do SQS ao usar o SQS como fonte de eventos com o Lambda. Ele também fornece uma visão geral do comportamento de escalonamento do Lambda usando esse padrão de arquitetura, desafios que esse recurso ajuda a resolver e uma demonstração do recurso de concorrência máxima.

# Visão geral

O AWS Lambda usa um [mapeamento de fonte de evento](https://docs.aws.amazon.com/lambda/latest/dg/invocation-eventsourcemapping.html) para processar itens de um _stream_ ou fila. O mapeamento do evento lê uma origem do evento, como uma fila SQS, opcionalmente filtra as mensagens, agrupa-as em lotes e invoca a função Lambda mapeada.

O comportamento de escala para integração do Lambda com [filas SQS FIFO](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/FIFO-queues.html) é simples. Uma única função do Lambda processa lotes de mensagens em um único grupo de mensagens para garantir que as mensagens sejam processadas em ordem.

Para [filas padrão do SQS](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/standard-queues.html), o mapeamento da origem do evento escaneia a fila para consumir as mensagens recebidas, começando em cinco lotes simultâneos com cinco funções por vez. À medida que as mensagens são adicionadas à fila do SQS, o Lambda continua a expandir para atender à demanda, adicionando até 60 funções por minuto, até 1.000 funções, para consumir essas mensagens. Para saber mais sobre o comportamento de escalabilidade do Lambda, leia ["Entendendo como o AWS Lambda escala com filas padrão do Amazon SQS"](https://aws.amazon.com/blogs/compute/understanding-how-aws-lambda-scales-when-subscribed-to-amazon-sqs-queues/).

![Lambda processando filas SQS padrão](https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2023/01/11/Lambda-processing-standard-SQS-queues.png)

# Desafios

Quando um grande número de mensagens está na fila do SQS, o Lambda escala, adicionando funções adicionais para processar as mensagens. A escala pode consumir a cota de concorrência da sua conta AWS. Para evitar que isso aconteça, você pode definir uma [concorrência reservada](https://docs.aws.amazon.com/lambda/latest/operatorguide/reserved-concurrency.html) para funções Lambda individuais. Isso garante que a função do Lambda especificada sempre possa ser escalada para essa concorrência, mas também não pode exceder esse número.

Quando a concorrência da função Lambda atinge o limite de concorrência reservada, a configuração da fila especifica o comportamento subsequente. A mensagem é retornada à fila e repetida com base na política de redrive, expirada com base em sua política de retenção ou enviada para outra [dead-letter queue (DLQ) SQS](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html). Embora enviar mensagens não processadas para um DLQ seja uma boa opção para preservar mensagens, ele requer um mecanismo separado para inspecionar e processar mensagens do DLQ.

O exemplo a seguir mostra uma função do Lambda atingindo sua cota de concorrência reservada de 10.

![Lambda alcançando simultaneidade reservada de 10.](https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2023/01/11/Lambda-reaching-reserved-concurrency-of-10.-1024x736.png)

# Concorrência máxima do Lambda com SQS como fonte de evento

A nova funcionalidade de concorrência máxima para SQS como uma fonte de evento, permite que você controle a concorrência da função Lambda em cada fonte de evento. Você define a concorrência máxima no mapeamento da origem do evento, não na função do Lambda.

Essa configuração no mapeamento da origem do evento não altera o comportamento de escala ou lote do Lambda com SQS. Você pode continuar a enviar mensagens em lote com tamanho e janelas de lote personalizados. Em vez disso, definimos um limite para o número máximo de chamadas concorrentes de uma função por fonte de evento SQS. Depois que o Lambda escala e atinge a concorrência máxima configurada na origem do evento, o Lambda para de ler as mensagens da fila. Esse recurso também oferece flexibilidade para definir a concorrência máxima para fontes de eventos individuais quando a função do Lambda tem várias fontes de eventos.

![A concorrência máxima é definida como 10 para a fila SQS.](https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2023/01/11/Maximum-concurrency-is-set-to-10-for-the-SQS-queue.-1024x604.png)

Esse recurso pode ajudar a impedir que uma função do Lambda consuma toda a concorrência do Lambda disponível da conta e evita que as mensagens retornem à fila desnecessariamente devido à limitação das funções do Lambda. Ele fornece uma maneira mais fácil de controlar e consumir mensagens no ritmo desejado, controlado pelo número máximo de funções Lambda concorrentes.

A configuração de concorrência máxima não substitui o recurso de concorrência reservada existente. Ambos servem a propósitos distintos e os dois recursos podem ser usados ​​juntos. A concorrência máxima pode ajudar a evitar o sobrecarregamento de _sistemas downstream_ e o _throttled_ de invocações desnecessárias. A concorrência reservada garante um número máximo de instâncias concorrentes para a função.

Quando usadas em conjunto, a função Lambda pode ter sua própria capacidade alocada (concorrência reservada), ao mesmo tempo em que pode controlar a taxa de transferência para cada fonte de evento (concorrência máxima). **Ao usar os dois recursos juntos, você deve definir o valor da concorrência reservada da função mais alta que a concorrência máxima no mapeamento de origem do evento SQS, para evitar _throttling_**.

# Configurando a concorrência máxima para SQS como fonte de evento

Você pode configurar a concorrência máxima para uma fonte de evento SQS por meio do [AWS Management Console](https://console.aws.amazon.com/console/home), [AWS Command Line Interface (CLI)](https://docs.aws.amazon.com/cli/latest/reference/lambda/create-event-source-mapping.html) ou infraestrutura em código, como [AWS Serverless Application Model (AWS SAM)](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/sam-specification.html). O valor mínimo compatível é 2 e o valor máximo é 1.000. Consulte a documentação de cotas do Lambda para obter os limites mais recentes.

![Configurando a concorrência máxima para um trigger do SQS no console.](https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2023/01/11/Configuring-the-maximum-concurrency-for-an-SQS-trigger-in-the-console.-879x1024.png)

Você pode definir a concorrência máxima por meio do comando [`create-event-source-mapping`](https://docs.aws.amazon.com/cli/latest/reference/lambda/create-event-source-mapping.html) da AWS CLI.

```bash
aws lambda create-event-source-mapping --function-name my-function --ScalingConfig {MaxConcurrency=2} --event-source-arn arn:aws:sqs:us-east-2:123456789012:my-queue
```

# Vendo a configuração de concorrência máxima em ação

A demo a seguir compara o Lambda recebendo e processando as mensagens de maneira diferente ao usar a concorrência máxima em comparação com a concorrência reservada.

Este [repositório no GitHub](https://github.com/aws-samples/aws-lambda-amazon-sqs-max-concurrency) contém um template AWS SAM que implanta os seguintes recursos:

- `ReservedConcurrencyQueue` (fila SQS)
- `ReservedConcurrencyDeadLetterQueue` (fila SQS)
- `ReservedConcurrencyFunction` (função Lambda)
- `MaxConcurrencyQueue` (fila SQS)
- `MaxConcurrencyDeadLetterQueue` (fila SQS)
- `MaxConcurrencyFunction` (função Lambda)
- `CloudWatchDashboard` (painel do CloudWatch)

O template do AWS SAM fornece dois conjuntos de arquiteturas idênticas e um [painel do Amazon CloudWatch](https://aws.amazon.com/cloudwatch/) para monitorar os recursos.

Cada arquitetura disponibiliza uma função Lambda que recebe mensagens de uma fila SQS e uma DLQ para a fila SQS.

O `maxReceiveCount` é definido como 1 para as filas SQS, que enviam todas as mensagens retornadas diretamente para o DLQ. O `ReservedConcurrencyFunction` tem sua concorrência reservada definida como 5 e o `MaxConcurrencyFunction` tem a concorrência máxima para a origem do evento SQS definida como 5.

# Pré-requisitos

A execução desta demo requer a [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html) e a [AWS SAM CLI](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/serverless-sam-cli-install.html). Depois de instalar ambas as CLIs, clone o [repositório do GitHub](https://github.com/aws-samples/aws-lambda-amazon-sqs-max-concurrency) e navegue até a raiz do diretório:

```bash
git clone https://github.com/aws-samples/aws-lambda-amazon-sqs-max-concurrency
cd aws-lambda-amazon-sqs-max-concurrency
```

# Fazendo o deploy do template AWS SAM

1. Crie o template AWS SAM com o [comando `build`](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/sam-cli-command-reference-sam-build.html) para se preparar para o deploy em seu ambiente da AWS.

```bash
sam build
```

2. Use a opção de [deploy guiado](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/sam-cli-command-reference-sam-deploy.html) para fazer i deploy dos recursos em sua conta.

```bash
sam deploy --guided
```

3. Dê um nome à pilha e aceite os valores padrão restantes. Depois do deploy, você pode acompanhar o progresso por meio da CLI ou navegando até a [página do AWS CloudFormation](http://console.aws.amazon.com/cloudformation) no Console de gerenciamento da AWS.
4. Observe os URLs da fila na guia _Outputs_ no AWS SAM CLI, no console do CloudFormation ou navegue até o console do SQS para encontrar os URLs da fila.

![A aba Outputs do template AWS SAM iniciado fornece URLs para o painel CloudWatch e filas SQS.](https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2023/01/11/The-Outputs-tab-of-the-launched-AWS-SAM-template-provides-URLs-to-CloudWatch-dashboard-and-SQS-queues.-1024x473.png)

# Executando o demo

O código da função do Lambda simula o processamento inativo por 10 segundos antes de retornar uma resposta 200. Isso permite que a função atinja um número alto de concorrência com apenas um pequeno número de mensagens.

Para adicionar 25 mensagens à fila com concorrência reservada, execute os seguintes comandos. Atualize o valor de `<ReservedConcurrencyQueueURL>` com o URL da fila do seu AWS SAM Outputs.

```bash
for i in {1..25}; do aws sqs send-message --queue-url <ReservedConcurrencyQueueURL> --message-body testing; done
```

Para adicionar 25 mensagens à fila de concorrência máxima, execute os seguintes comandos. Atualize o valor de `<MaxConcurrencyQueueURL>` com o URL da fila do AWS SAM Outputs.

```bash
for i in {1..25}; do aws sqs send-message --queue-url <MaxConcurrencyQueueURL> --message-body testing; done
```

Depois de enviar mensagens para ambas as filas, navegue até a URL do painel CloudWatch, disponível na aba Outputs.

# Validando resultados

Ambas as funções Lambda têm o mesmo número de invocações e as mesmas invocações concorrentes fixas em 5. O painel do CloudWatch que a `ReservedConcurrencyFunction` experênciou _throttling_ e 9 mensagens, conforme visto na métrica superior direita, foram enviadas para o seu DLQ correspondente. O `MaxConcurrencyFunction` não experimentou nenhum _throttling_ e as mensagens não foram enviadas ao DLQ.

![Painel do CloudWatch mostrando throttling e DLQs.](https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2023/01/11/CloudWatch-dashboard-showing-throttling-and-DLQs.-1024x791.png)

# Limpando

Para remover todos os recursos criados nesta demo, use o comando delete e siga os prompts:

```bash
sam delete
```

# Conclusão

Agora você pode controlar o número máximo de funções concorrentes invocadas pelo SQS como uma fonte de eventos para Lambda. Este artigo explica o comportamento de escala do Lambda usando esse padrão de arquitetura, os desafios que esse recurso ajuda a resolver e uma demonstração da concorrência máxima em ação.

Não há cobranças adicionais para usar esse recurso além das cobranças SQS e Lambda padrão. Você pode começar a usar a concorrência máxima para SQS no mapeamentos de fontes de eventos em novas filas ou filas existentes. Esse recurso está disponível em todas as regiões comerciais onde o Lambda está disponível.

Para obter mais recursos de aprendizado serverless, visite [Serverless Land](https://serverlessland.com/).

---

# Créditos

- Escrito originalmente por [John Lee e Jeetendra Vaidya](https://aws.amazon.com/blogs/compute/introducing-maximum-concurrency-of-aws-lambda-functions-when-using-amazon-sqs-as-an-event-source/), em [Introducing maximum concurrency of AWS Lambda functions when using Amazon SQS as an event source](https://aws.amazon.com/blogs/compute/introducing-maximum-concurrency-of-aws-lambda-functions-when-using-amazon-sqs-as-an-event-source/).
