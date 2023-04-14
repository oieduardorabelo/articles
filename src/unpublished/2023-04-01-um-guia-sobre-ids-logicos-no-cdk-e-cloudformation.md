# Um guia sobre IDs lógicos no CDK e CloudFormation

---

## Créditos

- Escrito originalmente por [Maciej Radzikowski](https://twitter.com/radzikowski_m), em [Understanding Logical IDs in CDK and CloudFormation](https://betterdev.blog/understanding-logical-ids-in-cdk-and-cloudformation/).

---

O CDK gera IDs lógicos usados pelo CloudFormation para rastrear e identificar recursos. Nesse artigo, explicarei o que são IDs lógicos, como são gerados e por que são importantes. Compreender isso ajudará a evitar exclusões de recursos inesperadas e erros desconcertantes de **_"resource already exists"_** durante o deploy.

[CDK](https://betterdev.blog/aws-cdk-pros-and-cons/) fornece uma camada de abstração sobre o CloudFormation, que é usada sob o capô. Com o CDK, a infraestrutura como código é mais fácil e segura. Mas, para usar o CDK de maneira eficaz, você ainda precisa entender como o CloudFormation funciona. Ao não compreender como o CloudFormation funciona, você pode ter consequências terríveis, como a remoção acidental de todos os dados do banco de dados de produção. E nós não queremos isso.

## ID de Construção vs. ID Lógico vs. ID Físico

_Em inglês: "Construct ID", "Logical ID" e "Physical ID"_

Vamos criar uma stack simples com um **_Construct_** – uma fila SQS. Para o **ID de construção**, que é o segundo parâmetro no construtor, definimos `MyQueue`:

```typescript
import { Stack, StackProps } from "aws-cdk-lib";
import { Queue } from "aws-cdk-lib/aws-sqs";

export class MyStack extends Stack {
  constructor(scope: Construct, id: string, props?: StackProps) {
    super(scope, id, props);

    new Queue(this, "MyQueue");
  }
}
```

Depois de executar `cdk deploy` temos uma stack do CloudFormation com um único recurso. O modelo de CloudFormation gerado se parece com o seguinte:

```yaml
Resources:
  MyQueueE6CA6235:
    Type: AWS::SQS::Queue
    UpdateReplacePolicy: Delete
    DeletionPolicy: Delete
    Metadata:
      aws:cdk:path: MyStack/MyQueue/Resource
```

O modelo contém um recurso `AWS::SQS::Queue` com ID lógico `MyQueueE6CA6235`. Como você pode ver, o ID lógico é o ID de construção que fornecemos, com um sufixo extra adicionado pelo CDK.

**_CDK Constructs_** tem relação com os **recursos do CloudFormation** em um relacionamento de um para muitos. Um único **_CDK Construct_** pode criar um ou mais recursos do CloudFormation. Neste exemplo, o **_Construct Queue_** cria um único recurso `AWS::SQS::Queue`.

Ainda outra coisa é o nome do recurso ou o **ID físico**. Se você for à página SQS no AWS Console, encontrará uma fila com um nome como `MyStack-MyQueueE6CA6235-86lqOs0JG5ZC`. É o nome gerado automaticamente pelo CloudFormation, que consiste do nome da stack, o ID lógico do recurso e em um sufixo aleatório adicionado pelo CloudFormation para exclusividade. Isso será importante mais adiante, então continue lendo.

Por enquanto, temos três IDs:

- o ID de construção que definimos no código CDK (`MyQueue`),
- o ID lógico gerado pelo CDK e colocado no recurso do CloudFormation (`MyQueueE6CA6235`),
- o ID físico (nome do recurso) gerado pelo CloudFormation (`MyStack-MyQueueE6CA6235-86lqOs0JG5ZC`).

Além disso, o ID físico faz parte do nome de recurso ARN (_Amazon Resource Name_). Que é usado pelos clientes (SDKs, CLIs, etc.) para fazer chamadas de API para o recurso. O ID lógico é importante para o CloudFormation, mas o ID físico é necessário para uso pelos clientes para acessar esses recursos.

## Como o CloudFormation rastreia recursos

O CloudFormation identifica recursos por seus IDs lógicos. Se alterarmos o ID lógico no modelo CloudFormation, o CloudFormation o verá como duas alterações:

- remoção do recurso antigo,
- e criação do novo.

Em terminologia do CloudFormation, isso é chamado de **substituir o recurso** (_resource replacing_).

Esse comportamento é descrito na [documentação do CloudFormation](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-cfn-updating-stacks-get-template.html):

> Para a maioria dos recursos, alterar o nome lógico de um recurso é equivalente a excluir esse recurso e substituí-lo por um novo. Quaisquer outros recursos que dependem do recurso renomeado também precisam ser atualizados e podem fazer com que sejam substituídos.

A maneira mais simples de provocar uma substituição é alterar o ID de construção:

```typescript
import { Stack, StackProps } from "aws-cdk-lib";
import { Queue } from "aws-cdk-lib/aws-sqs";

export class MyStack extends Stack {
  constructor(scope: Construct, id: string, props?: StackProps) {
    super(scope, id, props);

    new Queue(this, "MyRenamedQueue");
  }
}
```

Quando executarmos `cdk deploy`, o CloudFormation **primeiro cria uma nova fila SQS** e **somente depois remove a antiga**.

A ordem das operações é essencial aqui – O CloudFormation primeiro criará novos recursos e somente depois, sendo bem-sucedido, removerá os antigos. Isso reduz o tempo de inatividade e evita a remoção dos recursos existentes se algo der errado durante a atualização e precisarmos reverter o processo.

Recursos antigos são removidos no fase [UPDATE_COMPLETE_CLEANUP_IN_PROGRESS](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/cfn-console-view-stack-data-resources.html), descrito da seguinte forma:

> Remoção contínua de recursos antigos para uma ou mais stacks após uma atualização bem-sucedida da stack. Para atualizações de stack que exigem a substituição de recursos, o CloudFormation cria os novos recursos primeiro e depois exclui os recursos antigos para ajudar a reduzir quaisquer interrupções na sua stack. Nesse estado, a stack foi atualizada e pode ser usada, mas o CloudFormation ainda está excluindo os recursos antigos.

No exemplo acima, alteramos o ID de construção (e, portanto, o ID lógico), e a atualização ocorreu sem problemas. Mas nem sempre é o caso.

## Perigos ao substituir os recursos do CloudFormation

Ao alterar o ID lógico do nosso recurso em CloudFormation, removemos a fila SQS existente e criamos uma nova. Isso é uma coisa perigosa a se fazer no ambiente de produção!

## Perder dados de produção por acidente

E se a fila tivesse mensagens que ainda não foram processadas? Nós os perderíamos.

Se, em vez de uma fila do SQS, fosse uma tabela do DynamoDB, banco de dados em RDS ou qualquer outro banco de dados – o substituiríamos por um novo e vazio.

Também não é bom quando se lida com recursos sem estados ou estáticos, como as funções da Lambda. Ao substituir um recurso por outro, perdemos a continuidade das métricas.

## Erros no CloudFormation - "resource already exists"

Perder dados não é o único problema em potencial. Às vezes, o CloudFormation pode não fazer a atualização, informando que o recurso que queremos criar já existe.

Vamos modificar a primeira versão da nosso stack e adicionar a propriedade `queueName`. Isso corresponde ao ID físico do recurso `Queue`. Anteriormente, o CloudFormation gerava esse nome para nós, mantendo-o único ao adicionar um sufixo aleatório. Agora, nós codificamos isso:

```typescript
import { Stack, StackProps } from "aws-cdk-lib";
import { Queue } from "aws-cdk-lib/aws-sqs";
import { Construct } from "constructs";

export class MyStack extends Stack {
  constructor(scope: Construct, id: string, props?: StackProps) {
    super(scope, id, props);

    new Queue(this, "MyQueue", {
      queueName: "my-queue",
    });
  }
}
```

Se fizermos o deploy da stack, e como antes, alterarmos o ID de construção do `MyQueue` para `MyRenamedQueue`, deixando o `queueName` como está, a atualização da pilha do CloudFormation falhará:

```bash
CREATE_FAILED | AWS::SQS::Queue | MyRenamedQueue
Resource handler returned message: "Resource of type 'AWS::SQS::Queue'
with identifier 'my-queue' already exists."
(HandlerErrorCode: AlreadyExists)
```

Por isso acontece?

O nome da fila deve ser único em uma determinada conta da AWS em uma determinada região. É semelhante para funções Lambda, tabelas DynamoDB e, francamente, a maioria dos outros recursos da AWS.

"Mas pera aí!" - você pode dizer. Não declaramos uma segunda fila do SQS com o mesmo nome. Nossa pilha ainda contém uma única fila.

Para responde isso, devemos olhar para a ordem das operações do CloudFormation:

1. Criamos um _CDK Construct_ com ID de construção `MyQueue` e nome `my-queue` (ID físico).
   1.1 CloudFormation cria uma fila com o ID lógico `MyQueueE6CA6235` (sufixo adicionado pelo CDK) e nome `my-queue` (ID físico).
2. Alteramos o ID de construção do _CDK Construct_ de `MyQueue` para `MyRenamedQueue`
   2.1 CloudFormation vê isso como a remoção de `MyQueueE6CA6235` e criação de `MyRenamedQueue5E166F18` (sufixo adicionado pelo CDK)
   2.2 Em primeiro lugar, ele tenta criar a nova fila com ID lógico `MyRenamedQueue5E166F18` nomeado `my-queue` (ID físico).
   2.3 A criação da fila falha, pois o nome `my-queue`(ID físico) já existe

Como consertar isso? Existem duas maneiras:

1. Restaure o ID de construção original.
   - No entanto, nem sempre é possível se estamos refatorando o código! (veremos isso em um momento)
2. Remove o _CDK Construct_ da fila, faça o deploy da stack (para que o recurso antigo seja removido), adicione o _CDK Construct_ da fila novamente e faça um novo deploy para criar um novo recurso! (você pode comentar e descomentar o código para fazer isso)

## Prevenindo a substituição de recursos do CloudFormation

Ok, para evitar todos esses problemas, basta não definir os nomes dos recursos manualmente e não modificar os IDs de construção? Infelizmente, não é assim tão simples.

## Deixar o CloudFormation gerar nomes únicos

**A melhor prática é permitir que o CloudFormation gere nomes de recursos únicos**, ao invés de codificá-los. Isso tem dois benefícios:

1. Evitamos os erros como o descrito acima
2. Podemos fazer o deploy de várias instâncias da mesma stack no CloudFormation na mesma conta, por exemplo, para criar vários ambientes do nosso serviço

O item 2 também pode ser alcançado com nomes de recursos definidos manualmente, adicionando o nome do ambiente ao nome do recurso. Mas isso não é tão conveniente quanto deixar o CloudFormation gerar nomes únicos.

Mas, às vezes, o uso de nomes gerados automaticamente não é o mais adequado. Pela minha experiência, os nomes que codificamos são melhores em:

1. Para recursos compartilhados com outras contas da AWS (por exemplo, se um serviço em outra conta da AWS enviar mensagens diretamente para a nossa fila do SQS) porque se removermos e recriarmos a stack, o ARN do recurso não será alterado e nenhuma atualização de clientes externos será necessária
2. para recursos como as tabelas do AWS Glue, onde um nome agradável e curto é muito melhor para usar nas consultas do Athena, e ele precisa ser único apenas no escopo do banco de dados do AWS Glue

## Não alterar os IDs de construção no CDK

Mas, como discutimos anteriormente, substituir recursos provavelmente não é a melhor coisa a se fazer em primeiro lugar. Portanto, para evitarmos essa ação, a melhor maneira é **não modificarmos os IDs de construção de um CDK Construct**. Simples o suficiente, certo?

Bem, você pode adivinhar que – não é realmente simples!

Voltamos a nossa pilha com a criação da fila:

```typescript
import { Stack, StackProps } from "aws-cdk-lib";
import { Queue } from "aws-cdk-lib/aws-sqs";
import { Construct } from "constructs";

export class MyStack extends Stack {
  constructor(scope: Construct, id: string, props?: StackProps) {
    super(scope, id, props);

    new Queue(this, "MyQueue");
  }
}
```

Digamos que, à medida que nosso serviço cresce, adicionamos mais filas SQS e agora, nossas filas precisam ser configuradas com _dead-letter queue (DLQ)_. Então, ao invés de repetirmos nosso código, criamos um _Construct_ separado. Lembre-se, _Constructs_ são blocos de construção abstratos em CDK. Você pode aninhá-los e cada _Construct_ pode criar um ou mais recursos do CloudFormation.

```typescript
import { Stack, StackProps } from "aws-cdk-lib";
import { Queue } from "aws-cdk-lib/aws-sqs";
import { Construct } from "constructs";

export class MyStack extends Stack {
  constructor(scope: Construct, id: string, props?: StackProps) {
    super(scope, id, props);

    new MyQueueWithDLQ(this, "MyCustomQueue");
  }
}

class MyQueueWithDLQ extends Construct {
  constructor(scope: Construct, id: string) {
    super(scope, id);

    const dlq = new Queue(this, "DLQ");

    new Queue(this, "MyQueue", {
      deadLetterQueue: {
        maxReceiveCount: 5,
        queue: dlq,
      },
    });
  }
}
```

Mudamos a fila de do _Construct_ `MyStack` para o _Construct_ `MyQueueWithDLQ`. Mas o ID da fila permanece o mesmo – ainda é `MyQueue`.

Se executarmos o deploy da pilha agora, veremos duas novas filas criadas:

- `MyCustomQueueMyQueue20F468EB`
- `MyCustomQueueDLQE6D3019E`

E a fila existente anteriormente é removida.

Por que isso acontece?

**O CDK gera os IDs lógicos com base no caminho completo do _Construct_**.

Com _Constructs_ aninhados, os IDs de construções de todos os recursos "superiores" são usados para criar o ID lógico único do recurso final.

Então, quando o caminho mudou de `MyQueue` para `MyCustomQueue/MyQueue`.

O ID lógico gerado foi alterado de `MyQueueE6CA6235` para `MyCustomQueueMyQueue20F468EB`.

Portanto, mesmo que não alteremos os IDs de construção, mover construções para outros _Constructs_ irá gerar novos IDs lógicos. É o que geralmente acontece durante o desenvolvimento ou refatoração.

## Fixando IDs lógicos durante a refatoração em CDK

Felizmente, ainda podemos refatorar nosso código CDK evitando alterações nos IDs lógicos dos nossos recursos.

Para fazer isso, podemos configurar manualmente o ID lógico, ao invés de deixar que o CDK crie um.

Obviamente, isso não é recomendável fazer no início do desenvolvimento, pode ser considerado uma optimização prematura. Mas quando refatoramos o código e queremos mover o _Construct_ existente, esse é uma maneira de manter o ID lógico atual e fixá-lo, para que não seja alterado:

```typescript
import { CfnQueue, Queue } from "aws-cdk-lib/aws-sqs";

const queue = new Queue(this, "MyRenamedQueue");
(queue.node.defaultChild as CfnQueue).overrideLogicalId("MyQueueE6CA6235");
```

# Finalizando

Espero que este post esclareça como o CDK e o CloudFormation rastreiam os recursos e o tornam menos confuso.

O importante é que o **CloudFormation identifica os recursos pelo ID lógico**, não o nome ou qualquer outra propriedade. Portanto, se você alterar o ID lógico, o novo recurso será criado e o antigo será removido.

Substituir recursos por novos geralmente é seguro em ambientes de desenvolvimento, mas perigoso na produção, onde pode nos levar a perda de dados.

**CDK gera os IDs lógicos a partir do ID de construção.** Se você tem _Constructs_ aninhados, todos os IDs de construção dos _Constructs_ acima do seu recurso serão usados para gerar o ID lógico. **Mover um _Construct_ para outro irá alterar seu ID lógico.**

Quando refatoramos um código CDK e queremos mover o _Construct_ sem alterar seu ID lógico atual, podemos definir o ID lógico atual manual.

Um problema desagradável é alterar o ID lógico de um recurso com um nome codificado.

O CloudFormation tentará primeiro criar o novo recurso e falhará porque o recurso com o mesmo nome já existe.

A solução é reverter para o ID lógico anterior ou remover temporariamente o _Construct_ da stack, fazer o deploy para remover o recurso antigo, adicionar o _Construct_ novamente, e fazer um novo deploy para criar o recurso.
