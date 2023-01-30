# Aplicativos da Web totalmente tipados

O TypeScript é uma parte enorme da indústria da web. E há uma boa razão para isso. É incrível. E não estou falando só disso:

```typescript
function add(a: number, b: number) {
  return a + b;
}

add(1, 2); // validação de tipos válida
add("one", 3); // validação de tipos mostra erros
```

Quero dizer, isso é legal e tudo, mas o que estou falando é um código mais parecido com este:

![workshop type diagram](https://res.cloudinary.com/epic-web/image/upload/v1666194729/epicweb.dev/blog/Fully%20Typed%20Web%20Apps/workshop-type-diagram.svg)

Um tipo que flui por todo o programa (inclusive entre o front-end e o back-end). No mundo real, é assim que funciona e pode ser uma perspectiva assustadora decidir um dia que você deseja transformar o campo "Seats Left" em uma combinação dos campos "Total Seats" e "Sold Seats" sem os tipos para guiá-lo através dessa refatoração. Você terá dificuldades. Espero que você tenha alguns testes de unidade sólidos.

Mas não quero gastar este artigo convencendo você de como os tipos em JavaScript são ótimos. Em vez disso, quero compartilhar minha empolgação sobre como a tipagem de ponta-a-ponta tem sido excelente e mostrar como você pode usá-la em seus próprios aplicativos.

Primeiro, o que quero dizer com tipagem de ponta-a-ponta? Estou falando sobre ter tipagem dos dados do banco de dados, passando pelo código de back-end, até a interface do usuário e vice-versa. Agora, eu sei que cada um de nós está em uma situação diferente. Você pode não ter controle sobre seu banco de dados. Quando eu estava no PayPal, consumi uma dúzia de serviços criados por equipes diferentes. Nunca toquei em um banco de dados diretamente. Portanto, entendo que obter a verdadeira tipagem de ponta-a-ponta pode exigir colaboração. Mas espero poder ajudá-lo a trilhar o caminho certo para chegar o mais longe possível em sua própria situação.

A principal coisa que dificulta a tipagem de ponta-a-ponta é simples: contexto delimitados de negócio.

**O segredo para aplicativos da web totalmente tipados é criar os tipos de cada contexto delimitados de negócio.**

Na web, temos muitos contextos de negócios. Alguns deles você pode ter em mente e outros você pode não considerar. Aqui estão alguns exemplos de contextos que você encontrará na web:

```javascript
// sincronizando o estado do "ticket" para o local storage
const ticketData = JSON.parse(localStorage.get("ticket"));
//    ^? any 😱

// pegando valores de um formulário
// <form>
//   ...
//   <input type="date" name="workshop-date" />
//   ...
// </form>
const workshopDate = form.elements.namedItem("workshop-date");
//    ^? Element | RadioNodeList | null 😵

// buscando dados de uma API
const data = await fetch("/api/workshops").then((r) => r.json());
//    ^? any 😭

// buscando configuração e/ou parâmetros convencionais (para enviar apra o Remix ou React Router)
const { workshopId } = useParams();
//      ^? string | undefined 🥴

// lead/parseando uma string do módulo "fs"
const workshops = YAML.parse(await fs.readFile("./workshops.yml"));
//    ^? any 🤔

// lendo dados do banco de dados
const data = await SQL`select * from workshops`;
//    ^? any 😬

// lendo dados de um formulário de uma requisição web
const description = formData.get("description");
//    ^? FormDataEntryValue | null 🧐
```

Existem muitos outros exemplos, mas estes são alguns contextos comuns que você encontrará. Alguns contextos que temos aqui são (existem outros):

- Armazenamento local
- Entrada do usuário
- Rede
- Comportamento baseado em configuração ou convenções
- Sistema de arquivo
- Solicitações de banco de dados

O problema é que é impossível ter 100% de certeza de que o que você está recebendo de um contexto é o que você espera na aplicação. **Vale a pena repetir: é impossível.** Você certamente pode "deixar o TypeScript feliz" usando uma tipagem de `as Workshop` e tudo mais, mas você está apenas escondendo o problema. O arquivo pode ter sido mudado por outro processo, a API pode ter mudado, o usuário pode ter modificado o DOM manualmente, pelo amor de Deus. Simplesmente não há como saber com certeza se o resultado que chegou até você através desse contexto é o que você esperava.

Mas, há algumas coisas que você pode fazer para contornar isso. Você também pode:

1. Escreva guardas de tipo/funções de asserção de tipo
2. Use uma ferramenta que gere os tipos (te dá 99% de confiança)
3. Ajude a informar o TypeScript sobre sua convenção/configuração

Portanto, vamos dar uma olhada no uso dessas estratégias para obter tipagem de ponta-a-ponta, abordando os contexto delimitados de negócio dos aplicativos da web.

# Funções de guarda/proteção/asserção de tipos

Esta é definitivamente a maneira mais eficaz de garantir que, os dados que você passo de um contexto para outro é o que você esperava. Você literalmente escreve o código para verificá-lo! Aqui está um exemplo simples de proteção de tipo:

```typescript
const { workshopId } = useParams();
if (workshopId) {
  // aqui você tem um "workshopId" eo TypeScript sabe disso
} else {
  // faça outra coisa quando o "workshopId" não estiver presente
}
```

Neste ponto, alguns de vocês que estão irritados por ter que deixar o compilador do TypeScript feliz. Se você é tem tanta certeza que `workshopId` será o que você espera, então apenas lance um erro (será mais útil do que o erro que você obteria se ignorasse esse problema de qualquer maneira).

```typescript
const { workshopId } = useParams();
if (!workshopId) {
  throw new Error("workshopId not available");
}
```

Há um utilitário útil que uso em quase todos os meus projetos para tornar isso um pouco mais agradável aos olhos:

```typescript
import invariant from "tiny-invariant";

const { workshopId } = useParams();
invariant(workshopId, "workshopId not available");
```

Uma citação do README do [`tiny-invariant`](https://npm.im/tiny-invariant):

> Um função invariante recebe um valor e, se o valor for falso, a função invariante será lançada. Se o valor for verdadeiro, a função não será lançada.

Ter que adicionar o código extra é irritante. Este é apenas um problema complicado porque o TypeScript não conhece suas convenções ou configurações. Dito isso, talvez possamos informar o TypeScript sobre nossas convenções e configurações para ajudá-lo um pouco. Aqui estão alguns projetos trabalhando neste problema:

- [route-gen](https://github.com/sandulat/routes-gen) de [Stratulat Alexandru](https://twitter.com/sandulat) e [remix-routes](https://github.com/yesmeck/remix-routes) de [Wei Zhu](https://twitter.com/yesmeck), ambos geram tipos com base em sua configuração Remix/rotas convencionais (falaremos sobre isso mais tarde)
- (WIP) [TanStack Router](https://tanstack.com/router) por [Tanner Linsley](https://twitter.com/tannerlinsley) que garante todos os utilitários (como `useParams`) têm acesso às rotas que você definiu (informando efetivamente o TypeScript sobre sua configuração, que é outra solução alternativa que abordaremos).

Este é apenas um exemplo da perspectiva do router, que está abordando o contexto de URL para seu aplicativo, mas a ideia de ensinar suas convenções ao TypeScript também pode ser aplicada a outras áreas.

Vamos ver outro exemplo mais complexo de proteção de tipo:

```typescript
type Ticket = {
  workshopId: string;
  attendeeId: string;
  discountCode?: string;
};

// essa é uma função de proteção de tipo (type guard)
function isTicket(ticket: unknown): ticket is Ticket {
  return (
    Boolean(ticket) &&
    typeof ticket === "object" &&
    typeof (ticket as Ticket).workshopId === "string" &&
    typeof (ticket as Ticket).attendeeId === "string" &&
    (typeof (ticket as Ticket).discountCode === "string" ||
      (ticket as Ticket).discountCode === undefined)
  );
}

const ticket = JSON.parse(localStorage.get("ticket"));
//    ^?  any
if (isTicket(ticket)) {
  // agora você sabe que tem um ticket
} else {
  // cuida do caso quando não é um ticket
}
```

Parece muito trabalho, mesmo para um tipo relativamente simples. Imagine um tipo mais complexo do mundo real! Se você faz muito isso, pode achar útil uma ferramenta como o [zod](https://github.com/colinhacks/zod#safeparse)!

```typescript
import { z } from "zod";

const Ticket = z.object({
  workshopId: z.string(),
  attendeeId: z.string(),
  discountCode: z.string().optional(),
});
type Ticket = z.infer<typeof Ticket>;

const rawTicket = JSON.parse(localStorage.get("ticket"));
const result = Ticket.safeParse(rawTicket);
if (result.success) {
  const ticket = result.data;
  //    ^? Ticket
} else {
  // "result.error" terá uma mensagem informativa de erro
}
```

Minha maior preocupação com o zod (por que não o uso o tempo todo) é que o tamanho do pacote é muito grande ([42 KB não compactado](https://bundlejs.com/?q=zod) no momento em que este artigo foi escrito). Mas se você estiver usando apenas no servidor ou se acredita que vai se beneficiar muito com isso, pode valer a pena esse custo.

Uma ferramenta que tira proveito do zod para ajudar bastante com aplicativos da web totalmente tipados é o [tRPC](https://trpc.io/), que compartilha os tipos definidos com o zod para o servidor com o código do lado do cliente para fornecer tipagem segura nos contextos de comunicação entre as redes. Como eu uso o Remix, eu pessoalmente não uso o tRPC (embora você definitivamente possa, se quiser), mas se eu não estivesse usando o Remix, estaria 100% procurando usar o tRPC por causa desse recurso.

Funções de proteção/asserção de tipo também é a abordagem que você deseja usar para lidar com o `FormData` de seus formulários. Pessoalmente, estou gostando muito de usar [`remix-validity-state`](https://github.com/brophdawg11/remix-validity-state), mas a ideia é a mesma: código que realmente verifica os tipos em tempo de execução e fornece tipagem segura nos contexto delimitados de negócio do seu aplicativo.

# Geração de tipo

Eu falei sobre duas ferramentas que geram tipos para suas rotas convencionais em Remix, essas são uma forma de geração de tipos para ajudar a resolver o problema de tipagem de ponta-a-ponta. Outro exemplo alternativo popular dessa solução é o [Prisma](https://www.prisma.io/) (meu ORM favorito). Muitas ferramentas GraphQL também fazem isso. A ideia é permitir que você defina um esquema e o Prisma garante que suas tabelas de banco de dados correspondam a esse esquema. Em seguida, ele também gera definições de tipo TypeScript que também correspondem ao esquema. Mantendo efetivamente os tipos e o banco de dados em sincronia. Por exemplo:

```typescript
const workshop = await prisma.user.findFirst({
  // ^? { id: string, title: string, date: Date } 🎉
  where: { id: workshopId },
  select: { id: true, title: true, date: true },
});
```

Sempre que você fizer uma alteração em seu esquema e criar um script de migração, o prisma atualizará os tipos que ele possui em seu diretório `node_modules`, portanto, quando você interage com o prisma ORM, os tipos correspondem ao esquema atual. Aqui está um exemplo do mundo real da tabela `User` em [kentcdodds.com](https://kentcdodds.com/):

```prisma
model User {
  id           String     @id @default(uuid())
  createdAt    DateTime   @default(now())
  updatedAt    DateTime   @updatedAt
  email        String     @unique(map: "User.email_unique")
  firstName    String
  discordId    String?
  convertKitId String?
  role         Role       @default(MEMBER)
  team         Team
  calls        Call[]
  sessions     Session[]
  postReads    PostRead[]
}
```

E isso é o que é gerado a partir disso:

```typescript
/**
 * Model User
 *
 */
export type User = {
  id: string;
  createdAt: Date;
  updatedAt: Date;
  email: string;
  firstName: string;
  discordId: string | null;
  convertKitId: string | null;
  role: Role;
  team: Team;
};
```

Isso proporciona uma experiência de desenvolvedor fantástica e serve como ponto de partida para meus tipos que fluem por meio de meu aplicativo no back-end.

O principal perigo aqui é se o esquema do banco de dados e os dados no banco de dados de alguma forma ficarem fora de sincronia. Mas ainda não tive esse problema com o Prisma e espero que seja bastante raro, então me sinto bastante confiante em não adicionar funções de asserção em minhas interações com o banco de dados. Porém, se você não tem condições de usar uma ferramenta como o Prisma ou se você não é a equipe responsável pelo esquema do banco de dados, sugiro que encontre uma forma de gerar tipos para seu banco de dados baseado no esquema porque é simplesmente fantástico. Caso contrário, você pode querer adicionar funções de asserção aos resultados da consulta do banco de dados.

Lembre-se de que não estamos fazendo isso apenas para deixar o TypeScript feliz. Mesmo que não tivéssemos o TypeScript, seria uma boa ideia ter algum nível de confiança de que os dados que passam entre o contexto do seu aplicativo e o contexto do banco de dados, são o que você espera. Lembre-se:

![](https://www.epicweb.dev/_next/image?url=https%3A%2F%2Fres.cloudinary.com%2Fepic-web%2Fimage%2Fupload%2Fv1666201083%2Fepicweb.dev%2Fblog%2FFully%2520Typed%2520Web%2520Apps%2Ftweet_2x.png&w=1920&q=100)

# Ajudando o TypeScript com convenções/configurações de tipo

Um dos contextos mais desafiadores é o contexto de rede. Verificar o que o servidor está enviando para sua UI é complicado. `fetch` não tem suporte a tipos genéricos, e mesmo se tivesse você estaria mentindo para si mesmo:

```typescript
// isso não funciona, não faça isso
const data = fetch<Workshop>("/api/workshops/123").then((r) => r.json());
```

Vou contar um segredinho sujo sobre tipos genéricos... quase qualquer função que efetivamente faz isso provavelmente é uma má idéia:

```typescript
function getData<DataType>(one, two, three) {
  const data = doWhatever(one, two, three);
  return data as DataType; // <-- ISSO É UMA MÁ IDÉIA
}
```

Sempre que você está vendo `as Whatever` (chamado de _type cast_), você deve pensar: "Isso é mentir para o compilador TypeScript" que às vezes é o que é necessário para fazer o trabalho, mas eu não recomendo fazer isso com a função `getData` acima. Você tem duas escolhas:

```typescript
const a = getData<MyType>(); // 🤮
const b = getData() as MyType; // 😅
```

Em ambos os casos, você está mentindo para o TypeScript (e para si mesmo), mas no primeiro caso você não sabe disso! Se você vai mentir para si mesmo, pelo menos deveria saber que está fazendo isso.

Então, o que fazemos se não queremos mentir para nós mesmos? Bem, você pode estabelecer uma forte convenção para sua busca de dados com `fetch` e, em seguida, informar o TypeScript sobre essa convenção. Isso é o que é feito com o Remix. Aqui está um exemplo rápido disso:

```typescript
import type { LoaderArgs } from "@remix-run/node";
import { json } from "@remix-run/node";
import { useLoaderData } from "@remix-run/react";
import { prisma } from "~/db.server";
import invariant from "tiny-invariant";

export async function loader({ params }: LoaderArgs) {
  const { workshopId } = params;
  invariant(workshopId, "Missing workshopId");
  const workshop = await prisma.workshop.findFirst({
    where: { id: workshopId },
    select: { id: true, title: true, description: true, date: true },
  });
  if (!workshop) {
    // o Remix CatchBoundary irá cuidar do erro
    throw new Response("Not found", { status: 404 });
  }
  return json({ workshop });
}

export default function WorkshopRoute() {
  const { workshop } = useLoaderData<typeof loader>();
  //      ^? { title: string, description: string, date: string }
  return <div>{/* Workshop form */}</div>;
}
```

A função `useLoaderData` é um genérico que aceita um tipo de função Remix `loader` e é capaz de determinar todas as respostas JSON possíveis (muito obrigado ao criador do zod, [Colin McDonnell](https://twitter.com/colinhacks), por esta contribuição). O `loader` roda no servidor e a função `WorkshopRoute` é executada no servidor e no cliente, mas obter esses tipos através do contexto de rede acontece graças a convenção do tipo genérico da função Remix `loader`. O Remix garantirá que os dados retornados do `loader` acabem sendo retornados por `useLoaderData`. Tudo em um arquivo. Não são necessárias rotas de API 🥳 .

Se você ainda não teve essa experiência, você precisa acreditar em mim que esta é uma experiência incrível. Imagine que decidimos que queremos exibir o campo `price` em nossa interface do usuário. Isso é tão simples quanto atualizar nosso `select` para a consulta do banco de dados e, de repente, temos isso disponível em nosso código de interface do usuário sem alterar mais nada. Tipagem completamente segura! E se decidirmos que não precisamos mais do campo `decription`? Nós simplesmente removemos isso do `select` e obteremos erros em vermelho (e erros de verificação de tipo na compilação) em todos os lugares em que estávamos usando o `description` antes, o que ajuda com refatorações.

**E tudo isso cobrindo todo o contexto de rede (e o contexto entre banco de dados, back-end e front-end).**

Você deve ter notado que a propriedade `date` em nosso código de interface do usuário é um tipo `string` mesmo que na verdade seja um `Date` no back-end. Isso ocorre porque os dados precisam passar pelo contexto da rede e, no processo, tudo é serializado em uma string (o JSON não oferece suporte a datas). Os utilitários de tipo impõem esse comportamento que é estelar.

Se você planeja exibir essa data, provavelmente deve formatá-la no `loader` antes de enviá-la, para evitar a estranheza do fuso horário quando o aplicativo é hidratado na interface do usuário. Dito isso, se você não gostar dessa abordagem, pode usar uma ferramenta como [`superjson`](https://github.com/blitz-js/superjson) de [Matt Mueller](https://twitter.com/mattmueller) e [Simon Knott](https://twitter.com/skn0tt) ou [remix-typedjson](https://github.com/kiliman/remix-typedjson) de [Michael Carter](https://twitter.com/kiliman) para restaurar esses tipos de dados na interface do usuário.

No Remix, também obtemos tipagem segura com as funções `action` também. Aqui está um exemplo disso:

```typescript
import type { ActionArgs } from "@remix-run/node";
import { redirect, json } from "@remix-run/node";
import { useActionData, useLoaderData } from "@remix-run/react";
import type { ErrorMessages, FormValidations } from "remix-validity-state";
import { validateServerFormData } from "remix-validity-state";
import { prisma } from "~/db.server";
import invariant from "tiny-invariant";

// ... coisas da função "loader" aqui

const formValidations: FormValidations = {
  title: {
    required: true,
    minLength: 2,
    maxLength: 40,
  },
  description: {
    required: true,
    minLength: 2,
    maxLength: 1000,
  },
};

const errorMessages: ErrorMessages = {
  tooShort: (minLength, name) =>
    `The ${name} field must be at least ${minLength} characters`,
  tooLong: (maxLength, name) =>
    `The ${name} field must be less than ${maxLength} characters`,
};

export async function action({ request, params }: ActionArgs) {
  const { workshopId } = params;
  invariant(workshopId, "Missing workshopId");
  const formData = await request.formData();
  const serverFormInfo = await validateServerFormData(
    formData,
    formValidations
  );
  if (!serverFormInfo.valid) {
    return json({ serverFormInfo }, { status: 400 });
  }
  const { submittedFormData } = serverFormInfo;
  //      ^? { title: string, description: string }
  const { title, description } = submittedFormData;
  const workshop = await prisma.workshop.update({
    where: { id: workshopId },
    data: { title, description },
    select: { id: true },
  });
  return redirect(`/workshops/${workshop.id}`);
}

export default function WorkshopRoute() {
  // ... coisas da função "loader" aqui
  const actionData = useActionData<typeof action>();
  //    ^? { serverFormInfo: ServerFormInfo<FormValidations> } | undefined
  return <div>{/* Formulário do Workshop */}</div>;
}
```

Mais uma vez, seja qual for o retorno do nosso `action`, acaba sendo do tipo (serializado) que nosso `useActionData` referencia. Neste caso, estou usando `remix-validity-state` que também terá propriedades com tipagem segura. Além disso, os dados enviados são analisados ​​com segurança por `remix-validity-state` seguindo o esquema que forneci, então o tipo `submittedFormData` tem todos os meus dados analisados ​​e prontos para uso. Existem outras bibliotecas para isso, mas o ponto é que, com alguns utilitários simples, podemos obter uma tipagem segura fantástica e termos nossos contextos tipados e aumentar nossa confiança no envio e dados. Bem, a API para os utilitários é simples. Às vezes, os próprios utilitários são bastante complexos 😅.

Devo mencionar que isso também funciona em outros utilitários Remix. A exportação `meta` pode ser totalmente tipada, assim como `useFetcher`, e `useMatcher`. É um mundo glorioso, meus amigos.

Sério, esse `loader` é uma coisa maravilhosa. Quero dizer, caramba, assista isso!

![](https://res.cloudinary.com/epic-web/video/upload/v1666195258/epicweb.dev/blog/Fully%20Typed%20Web%20Apps/across-the-network-typesafety.mp4)

E tudo em um arquivo. Caramba aí sim 🔥

# Conclusão

O ponto que estou tentando enfatizar aqui é que a tipagem segura é algo que não é apenas valioso, mas alcançável através dos contextos da sua aplicação e de ponta-a-ponta. Esse último exemplo do `loader` vai desde o banco de dados até a interface do usuário. Esses dados estão fortemente tipados entre o `database` → `node` → `browser` e isso me torna incrivelmente produtivo como engenheiro.

Qualquer que seja o projeto em que você esteja trabalhando, pense em como pode deixar algumas chamadas mentirosas de `as Whatever` e altere-os para uma tipagem segura mais verdadeira, usando algumas das sugestões que forneci aqui. Acho que você vai agradecer a si mesmo mais tarde. Definitivamente vale a pena o esforço!

Se você quiser experimentar um exemplo disso, confira o demo [kentcdodds/full-typed-web-apps-demo](https://github.com/kentcdodds/fully-typed-web-apps-demo).

E antes que alguém pergunte, estarei 100% ensinando todos esses métodos no [EpicWeb.dev](https://www.epicweb.dev/).

Cadastre-se [nesse link para receber atualizações](https://epicweb.dev/fully-typed-web-apps#article)!

---

# Créditos

- Escrito originalmente por [Kent C. Dodds](https://twitter.com/kentcdodds), em [Fully Typed Web Apps](https://www.epicweb.dev/fully-typed-web-apps).
