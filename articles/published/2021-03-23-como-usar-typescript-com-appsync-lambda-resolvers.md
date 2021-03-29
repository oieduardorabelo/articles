# Como Usar TypeScript com AWS AppSync Lambda Resolvers

## Gerando tipos de TypeScript diretamente do seu esquema!

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1614692037497/pr_Jgcv1q.png?w=1600&h=840&fit=crop&crop=entropy&auto=compress)

Um dos grandes benefícios do GraphQL é que ele é fortemente tipado! Defina seu esquema e GraphQL impõe a "forma" de entrada / saída de seus dados.

Se você estiver usando Lambda como resolvers do AWS AppSync com o tempo de execução Node.js, você também pode usar TypeScript. Se você fizer isso, você poderá definir os tipos de TypeScript que correspondem ao seu esquema automaticamente. Fazer isso manualmente pode ser entediante, e estaremos sujeito a erros e basicamente faremos o mesmo trabalho duas vezes! 🙁 Não seria ótimo se você pudesse importar seus tipos GraphQL para o seu código automaticamente?

Neste artigo, mostrarei como gerar tipos TypeScript diretamente do seu esquema GraphQL, apenas executando uma linha de comando! Em seguida, veremos como usar esses tipos em seus resolvers Lambda.

Vamos começar!

# Pré-requisitos

Você já deve ter uma configuração de projeto do AWS AppSync com um esquema GraphQL definido (se ainda não tiver um, você pode usar o exemplo abaixo).

Para os fins deste tutorial, usaremos esse esquema como exemplo:

```gql
type Query {
  post(id: ID!): Post
}

type Mutation {
  createPost(post: PostInput!): Post!
}

type Post {
  id: ID!
  title: String!
  content: String!
  publishedAt: AWSDateTime
}

input PostInput {
  title: String!
  content: String!
}
```

# Configurando o projeto

## Instale as dependências

Precisaremos instalar três pacotes:

```bash
npm i @graphql-codegen/cli @graphql-codegen/typescript @types/aws-lambda  -D
```

Os primeiros dois pacotes pertencem a suite [graphql-code-generator](https://github.com/dotansimha/graphql-code-generator). O primeiro é a CLI base, enquanto o segundo é o plugin que gera o código TypeScript a partir de um esquema GraphQL.

`@types/aws-lambda` é uma coleção de tipos TypeScript para AWS Lambda. Inclui todos os tipos de definições de eventos Lambda (API Gateway, S3, SNS, etc.), incluindo uma para resolvers do AWS AppSync (`AppSyncResolverHandler`). Usaremos esse último mais tarde, quando construirmos nossos resolvers.

# Crie o arquivo de configuração

É hora de configurar `graphql-codegen` e dizer como gerar nossos tipos TypeScript. Para isso, vamos criar um arquivo `codegen.yml`:

```yaml
overwrite: true
schema:
  - schema.graphql # seu schema graphql

generates:
  appsync.d.ts:
    plugins:
      - typescript
```

Isso diz ao codegen quais arquivos de esquema ele deve usar (no exemplo: `schema.graphql`), qual plugin (`typescript`) e onde a saída deve ser colocada (`appsync.d.ts`). Fique a vontade para alterar esses parâmetros para atender às suas necessidades.

# Suporte para AWS Scalars

Se você estiver usando [AWS AppSync Scalars](https://docs.aws.amazon.com/appsync/latest/devguide/scalars.html), também precisará dizer ao `graphql-codegen` como lidar com eles.

> 💡 Você precisa declarar, no mínimo, os escalares que usa, mas pode ser uma boa ideia apenas declarar todos eles.

Vamos criar um novo arquivo `appsync.graphql` com o seguinte conteúdo:

```gql
scalar AWSDate
scalar AWSTime
scalar AWSDateTime
scalar AWSTimestamp
scalar AWSEmail
scalar AWSJSON
scalar AWSURL
scalar AWSPhone
scalar AWSIPAddress
```

> ⚠️ Não coloque esses tipos no mesmo arquivo de seu esquema principal. Você só precisa deles para geração de código e eles não devem entrar em seu deploy para o AWS AppSync.

Também precisamos dizer ao codegen como mapear esses escalares para o TypeScript. Para isso, iremos modificar o arquivo `codegen.yml`. Adicione / edite as seguintes seções:

```yaml
schema:
  - schema.graphql
  - appsync.graphql # 👈 coloco isso

# e isso 👇
config:
  scalars:
    AWSJSON: string
    AWSDate: string
    AWSTime: string
    AWSDateTime: string
    AWSTimestamp: number
    AWSEmail: string
    AWSURL: string
    AWSPhone: string
    AWSIPAddress: string
```

# Gerando o código

Estamos prontos com a configuração. É hora de gerar algum código! Execute o seguinte comando:

```bash
graphql-codegen
```

> 💡 Você também pode adicionar "codegen": "graphql-codegen" ao seu `package.json` na seção "scripts" e usar `npm run codegen`.

Se você olhar em seu diretório de trabalho, verá um arquivo `appsync.d.ts` que contém os tipos gerados.

```ts
export type Maybe<T> = T | null;
export type Exact<T extends { [key: string]: unknown }> = {
  [K in keyof T]: T[K];
};
export type MakeOptional<T, K extends keyof T> = Omit<T, K> &
  { [SubKey in K]?: Maybe<T[SubKey]> };
export type MakeMaybe<T, K extends keyof T> = Omit<T, K> &
  { [SubKey in K]: Maybe<T[SubKey]> };

/** Todos os AWS AppSync Scalars, mapeados para seus atuais valores */
export type Scalars = {
  ID: string;
  String: string;
  Boolean: boolean;
  Int: number;
  Float: number;
  AWSDate: string;
  AWSTime: string;
  AWSDateTime: string;
  AWSTimestamp: number;
  AWSEmail: string;
  AWSJSON: string;
  AWSURL: string;
  AWSPhone: string;
  AWSIPAddress: string;
};

export type Query = {
  __typename?: "Query";
  post?: Maybe<Post>;
};

export type QueryPostArgs = {
  id: Scalars["ID"];
};

export type Mutation = {
  __typename?: "Mutation";
  createPost: Post;
};

export type MutationCreatePostArgs = {
  post: PostInput;
};

export type Post = {
  __typename?: "Post";
  id: Scalars["ID"];
  title: Scalars["String"];
  content: Scalars["String"];
  publishedAt?: Maybe<Scalars["AWSDateTime"]>;
};

export type PostInput = {
  title: Scalars["String"];
  content: Scalars["String"];
};
```

Observe que, além de alguns tipos auxiliares na parte superior, diferentes tipos estão sendo gerados:

- `Scalars`

Contém todos os escalares básicos (ID, String, etc.) e os escalares personalizados da AWS.

- `Query` e `Mutation`

Esses dois tipos descrevem os tipos completos de consulta e mutação.

- `Post`

Este é o nosso tipo `Post` de nosso esquema traduzido para TypeScript. É também o valor de retorno da query `post` e da mutação `createPost`.

- `QueryPostArgs` e `MutationCreatePostArgs`

Esses tipos descrevem os argumentos de entrada da query `post` e da mutação `createPost`, respectivamente.

> 💡 Você notou o padrão de nome aqui? Os tipos de argumento são sempre nomeados `Query[NomeDoEndpoint]Args` e `Mutation[NomeDoEndpoint]Args` em `PascalCase`. Isso é útil para saber quando você deseja preencher automaticamente os tipos em seu IDE.

# Usando os tipos gerados

Agora que geramos nossos tipos, é hora de usá-los!

Vamos implementar o resolver `Query.post` como exemplo.

Os manipuladores de lambda sempre recebem 3 argumentos:

- `event`: contém informações sobre a consulta de entrada (argumentos, identidade, etc)
- `context`: contém informações sobre a função Lambda executada
- `callback`: uma função que você pode chamar quando seu manipulador for finalizado (se você não estiver usando `async`/`Promise`)

A forma de um manipulador do AWS AppSync é quase sempre a mesma. Acontece que existe um pacote `DefinitelyTyped` que já o define. Nós o instalamos no início deste tutorial. Vamos usar!

O tipo `AppSyncResolverHandler` leva dois argumentos. O primeiro é o tipo do objeto `event.arguments` e o segundo é o valor de retorno do resolver.

No nosso caso, será: `QueryPostArgs` e `Post`, respectivamente.

Aqui está como usá-lo:

```ts
import db from "./db";
import { AppSyncResolverHandler } from "aws-lambda";
import { Post, QueryPostArgs } from "./appsync";

export const handler: AppSyncResolverHandler<QueryPostArgs, Post> = async (
  event
) => {
  const post = await db.getPost(event.arguments.id);

  if (post) {
    return post;
  }

  throw new Error("Not Found");
};
```

Agora, nosso manipulador Lambda se beneficia da verificação de tipo de 2 maneiras:

- `event.arguments`: Será do tipo `QueryPostArgs` (com os benefícios do preenchimento automático!)
- verificação do valor de retorno, ou o segundo argumento de callback, tenha a mesma forma que `Post` (com um id, título, etc); ou o TypeScript mostrará um erro.

# Uso avançado

Existem muitas opções que permitem personalizar os tipos gerados. [Confira a documentação para mais detalhes](https://graphql-code-generator.com/docs/plugins/typescript)!

# Conclusão

Com a geração automática de tipos, você não apenas melhorará sua velocidade de desenvolvimento e experiência, mas também garantirá que seus resolvers façam o que sua API espera. Você também garante que seus tipos de código e seus tipos de esquema estejam sempre em perfeita sincronia, evitando incompatibilidades que podem levar a bugs.

Não se esqueça de executar novamente o comando `graphql-codegen` cada vez que editar seu esquema! Pode ser uma boa ideia automatizar o processo ou validar seus tipos no pipeline de CI / CD.

---

# Créditos

- [How to use TypeScript with AppSync Lambda Resolvers](https://benoitboure.com/how-to-use-typescript-with-appsync-lambda-resolvers), escrito originalmente por [Benoît Bouré](https://twitter.com/Benoit_Boure).
