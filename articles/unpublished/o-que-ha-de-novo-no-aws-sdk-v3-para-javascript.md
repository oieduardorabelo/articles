# O que há de novo no AWS SDK v3 para JavaScript?

O SDK v3 da AWS para JavaScript está disponível para todo mundo desde dezembro de 2020. Um desafio que une todos os usuários da AWS: vale a pena investir seu precioso tempo nesta nova versão?

Nesse artigo, mostro os novos recursos e casos de uso em que a versão v3 mais ajuda, não importa se você usa JavaScript no front-end ou no back-end (Node.js). Vamos começar!

## Paginação

Muitas APIs da AWS podem retornar uma longa lista de dados (por exemplo, listar todos os objetos em um bucket S3). Todas as APIs de lista oferecem um mecanismo de paginação para recuperar um lote por vez. Cada lote contém um token para recuperar o próximo lote ou indicar que você atingiu o fim da lista. O bom e velho código da versão v2 com **callbacks** era assim:

```js
const AWS = require("aws-sdk");

const dynamodb = new AWS.DynamoDB({ apiVersion: "2012-08-10" });

function fetchPage(lastEvaluatedKey, cb) {
  const params = {
    TableName: "users",
    Limit: 100,
  };
  if (lastEvaluatedKey) {
    params.ExclusiveStartKey = lastEvaluatedKey;
  }
  dynamodb.scan(params, cb);
}

function fetchAll(cb) {
  fetchPage(null, function (err, data) {
    if (err) {
      cb(err);
    } else {
      data.Items.forEach(console.log);
      if (data.LastEvaluatedKey) {
        process.nextTick(() => fetchPage(data.LastEvaluatedKey, cb));
      } else {
        cb();
      }
    }
  });
}

fetchAll(function (err) {
  if (err) {
    // panic
    process.exit(1);
  } else {
    console.log("done");
  }
});
```

Felizmente, a linguagem JavaScript evoluiu. Agora temos Promises, async / await e [generators](https://hacks.mozilla.org/2015/05/es6-in-depth-generators/) . É por isso que agora podemos escrever o seguinte usando o SDK v3:

```js
import { DynamoDBClient, paginateScan } from "@aws-sdk/client-dynamodb";

const dynamodb = new DynamoDBClient({ apiVersion: "2012-08-10" });

async function fetchAll() {
  const config = {
    client: dynamodb,
    pageSize: 100,
  };
  const input = {
    TableName: "users",
  };
  const paginator = paginateScan(config, input);
  for await (const page of paginator) {
    page.Items.forEach(console.log);
  }
}

fetchAll()
  .then(() => console.log("done"))
  .catch(() => process.exit(1));
```

Estou feliz por não precisar mais escrever código de paginação. Lembre-se de que, em quase todos os cenários voltados para o usuário, você deve evitar obter todos os itens de uma vez só. Vai demorar muito e você pode ficar sem memória na execução! Acredito que os desenvolvedores de back-end se beneficiam mais com esse recurso quando escrevem código de "trabalho em lote" (_batch jobs_).

## Promises

Muitos programadores preferem [Promise](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise) ao invés de **callbacks**. Na versão SDK v2 usávamos callbacks dessa maneira:

```js
const AWS = require("aws-sdk");

const dynamodb = new AWS.DynamoDB({ apiVersion: "2012-08-10" });

dynamodb.getItem(
  {
    /*...*/
  },
  function (err, data) {
    // esse código é invocado assim que os resultados estiverem disponíveis
    dynamodb.updateItem(
      {
        /*...*/
      },
      function (err, data) {
        // ao usar vários callbacks, criamos o chamado "callback hell"
      }
    );
  }
);
```

A abordagem de Promises combinada com async / await na v2 é assim:

```js
const AWS = require('aws-sdk');

const dynamodb = new AWS.DynamoDB({apiVersion: '2012-08-10'});

async function demo() {}
 const data = await dynamodb.getItem({/*...*/}).promise();
 await dynamodb.updateItem({/*...*/}).promise();
}

demo()
 .then(() => console.log('done'))
 .catch(() => process.exit(1));
```

Com o novo SDK v3, agora você pode escrever:

```js
import { DynamoDB } from '@aws-sdk/client-dynamodb';

const dynamodb = new DynamoDB({apiVersion: '2012-08-10'});

async function demo() {}
 const data = await dynamodb.getItem({/*...*/}); // Não precisamos de ".promise()"!
 await dynamodb.updateItem({/*...*/});
}

demo()
 .then(() => console.log('done'))
 .catch(() => process.exit(1));
```

Este novo recurso é ótimo para desenvolvedores front-end e back-end!

## Modularização e Tree Shaking

O SDK v2 empacota todos os serviços da AWS. Com o tempo, o módulo `aws-sdk` cresceu muito em tamanho. E esse é um desafio difícil de resolver se você usar o SDK no front-end. Ele é muito grande e deixa seu site mais lento. Dois recursos do novo SDK v3 que irão ajudar:

- Modularização
- Tree Shaking

Cada serviço da AWS agora é empacotado como seu próprio módulo npm. Você deseja usar o DynamoDB? Instale o pacote `@aws-sdk/client-dynamodb`. Precisa de S3? Instale `@aws-sdk/client-s3` e assim por diante.

JavaScript hoje em dia suporta módulos ES6. Combinado com ferramentas como [esbuild](https://esbuild.github.io/) ou [webpack](https://webpack.js.org/) , podemos eliminar o código JS que "não é usado". Isso é chamado de _tree shaking_. Se você quiser se beneficiar do _tree shaking_, **não use** a sintaxe que é compatível com o SDK v2:

```js
import { DynamoDB } from "@aws-sdk/client-dynamodb";

const dynamodb = new DynamoDB({ apiVersion: "2012-08-10" });

await dynamodb.getItem(/*...*/);
```

Ao invés disso, use:

```js
import { DynamoDBClient, GetItemCommand } from "@aws-sdk/client-dynamodb";

const dynamodb = new DynamoDBClient({ apiVersion: "2012-08-10" });

await dynamodb.send(new GetItemCommand(/*...*/));
```

Agora, o _tree shaking_ pode entrar em ação e remover todos os comandos que você nunca usa! Os desenvolvedores de front-end vão adorar!

> O tempo de execução do AWS Lambda em Node.js não inclui o SDK v3. Se você deseja manter seu código de função lambda o menor possível, você ainda deve usar o SDK v2 porque você não precisa incluí-lo em seu arquivo ZIP.

## As partes ruins

- Suporte para X-Ray ausente (sendo trabalhado [aqui](https://github.com/aws/aws-sdk-js-v3/issues/1795) e [aqui](https://github.com/aws/aws-xray-sdk-node/issues/294)
- Como eu disse, o SDK v3 não faz parte do tempo de execução do AWS Lambda em Node.js. Se você quiser usá-lo, terá que adicioná-lo ao seu ZIP.
- O DynamoDB [DocumentClient](https://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/DynamoDB/DocumentClient.html) está faltando. Ao invés dele, você pode querer usar [essas funções utilitárias](https://docs.aws.amazon.com/sdk-for-javascript/v3/developer-guide/dynamodb-example-dynamodb-utilities.html). Eu nunca usei o `DocumentClient` porque prefiro a importação explícita ao implícito.
- O Client Side Monitoring (CSM) não é suportado, o que torna a [depuração de problemas de IAM mais difícil / impossível](https://cloudonaut.io/record-aws-api-calls-to-improve-iam-policies/) .

## Características adicionais

- Se você deseja escrever plugins para o SDK da AWS (talvez porque você tenha adicionado suporte ao X-Ray), você vai se interessar pelo no novo [middleware](https://aws.amazon.com/blogs/developer/middleware-stack-modular-aws-sdk-js/).
- v3 é implementado em TypeScript. Também temos tipos para o SDK v2. Portanto, não é uma grande mudança para nós, clientes da AWS.

## AWS SDK v3 para JavaScript em ação

[Assista essa vídeo](https://cloudonaut.io/whats-new-about-aws-sdk-for-javascript-v3#AWS-SDK-for-JavaScript-v3-in-Action) para aprender como usamos o novo SDK em nosso [Chatbot para AWS Monitoring](https://marbot.io/).

## Resumo

Os desenvolvedores de front-end devem começar a usar o SDK v3 hoje mesmo. Use objetos de comando para remover todo o código não utilizado! Para desenvolvedores de back-end, não vejo muitos benefícios, a menos que seu código esteja paginando muito :). Se seu back-end é executado no lambda, lembre-se de que o SDK v3 não faz parte do tempo de execução do AWS Lambda! Se você planeja migrar, verifique o [guia oficial de migração](https://docs.aws.amazon.com/sdk-for-javascript/v3/developer-guide/migrating-to-v3.html).

# Créditos

- [What's new about AWS SDK for JavaScript v3?](https://cloudonaut.io/whats-new-about-aws-sdk-for-javascript-v3/#AWS-SDK-for-JavaScript-v3-in-Action), escrito originalmente por [Michael Wittig](https://twitter.com/hellomichibye).
