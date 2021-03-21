# Um Guia sobre Injeção e Inversão de Dependências em Node.js e TypeScript

**Injeção e Inversão de dependência são dois termos relacionados, mas comumente usados ​​de maneira incorreta no desenvolvimento de software. Neste artigo, exploramos os dois tipos de DI (_Dependency Injection_ e _Dependency Inversion_) e como você pode usá-la para escrever código testável.**

![](https://khalilstemmler.com/img/blog/di-container/dependency-injection-inversion-explained.png)

> Este tópico foi retirado do livro [Solid Book - The Software Architecture & Design Handbook w / TypeScript + Node.js](https://solidbook.io/). Confira o livro se você gostou deste artigo.

Uma das primeiras coisas que aprendemos em programação é decompor grandes problemas em partes menores. Essa abordagem de dividir para conquistar pode nos ajudar a atribuir tarefas a outras pessoas, reduzir a ansiedade focando em uma coisa de cada vez e melhorar a modularidade de nossos projetos.

Mas chega um momento em que as coisas estão prontas para serem conectadas.

**É aí que a maioria dos desenvolvedores aborda as coisas da maneira errada.**

A maioria dos desenvolvedores que ainda não aprenderam sobre os [princípios SOLID](https://khalilstemmler.com/articles/solid-principles/solid-typescript/) ou a composição do software e continuam a escrever módulos e classes firmemente acoplados que não devem ser acoplados, resultando em um código difícil de mudar e testar .

Neste artigo, vamos aprender sobre:

1. Componentes e composição de software
2. Como NÃO conectar componentes
3. Como e por que injetar dependências usando injeção de dependência
4. Como aplicar Inversão de Dependência e escrever código testável
5. Considerações sobre containers de inversão de controle

## Terminologia

Vamos ter certeza de que entendemos a terminologia sobre como conectar dependências antes de continuar.

## Componentes

Vou usar muito o termo componente. Esse termo pode afetar o React.js ou desenvolvedores Angular, mas pode ser usado além do escopo da web, Angular ou React.

Um componente é simplesmente uma parte de um aplicativo. É qualquer grupo de software que se destina a fazer parte de um sistema maior.

A ideia é dividir um grande aplicativo em vários componentes modulares que podem ser desenvolvidos e montados independentemente.

Quanto mais você aprende sobre software, mais percebe que um bom design de software envolve composição de componentes.

A falha em acertar essa composição, leva a um código complicado que não pode ser testado.

## Injeção de dependência

Eventualmente, precisaremos conectar os componentes de alguma forma. Vejamos uma maneira trivial (e não ideal) de conectar dois componentes.

No exemplo a seguir, queremos conectar o `UserController` para que ele possa recuperar todos os `User[]` de um `UserRepo` (chamado de [repositório](https://khalilstemmler.com/articles/typescript-domain-driven-design/repository-dto-mapper/)) quando alguém fizer uma solicitação `HTTP GET` para `/api/users`.

```ts
// repos/userRepo.ts

/**
 * @class UserRepo
 * @desc Responsável por buscar usuários no banco de dados.
 **/
export class UserRepo {
  constructor() {}

  getUsers(): Promise<User[]> {
    // Usamos Sequelize ou TypeORM para recuperar
    // os usuários de do banco de dados
  }
}
```

E o controlador:

```ts
// controllers/userController.ts

import { UserRepo } from "../repos"; // #1 Prática Ruim

/**
 * @class UserController
 * @desc Responsável por lidar com solicitações de API para a rota /user
 **/

class UserController {
  private userRepo: UserRepo;

  constructor() {
    this.userRepo = new UserRepo(); // #2 Prática Ruim, continue lendo para ver o porquê
  }

  async handleGetUsers(req, res): Promise<void> {
    const users = await this.userRepo.getUsers();
    return res.status(200).json({ users });
  }
}
```

No exemplo, conecto um `UserRepo` diretamente a um `UserController` ao criar uma instância com a classe `UserRepo` dentro da classe `UserController`.

**Isso não é o ideal.** Quando fazemos isso, criamos uma **dependência do código-fonte.**

> **Dependência do código-fonte:** quando o componente atual (classe, módulo, etc) depende de pelo menos um outro componente para ser compilado. Dependências do código-fonte devem ser limitadas.

O problema é que toda vez que quisermos criar um `UserController`, precisamos ter certeza de que o `UserRepo` também está ao nosso alcance para que o código possa ser compilado.

![](https://khalilstemmler.com/img/blog/di-container/before-dependency-inversion.svg)
_A classe UserController depende diretamente da classe UserRepo._

E quando é que queremos criar um `UserController` isolado?

Durante os testes.

**É uma prática comum durante os testes simular ou falsificar dependências do módulo atual para isolar e testar diferentes comportamentos.**

Observe como estamos: 1) importando a [classe concreta](https://khalilstemmler.com/wiki/concrete-class/) `UserRepo` para o arquivo e; b) criando uma instância dela de dentro do construtor `UserController`?

Isso torna este código difícil de testar. Ou, pelo menos, se `UserRepo` estivesse conectado a um banco de dados real em execução, teríamos que trazer toda a conexão do banco de dados conosco para executar nossos testes, tornando-os muito lentos...

A injeção de dependência é uma técnica que pode melhorar a testabilidade de nosso código.

Ele funciona transmitindo (geralmente por meio do construtor) as dependências de que seu módulo precisa para operar.

Se mudarmos a forma como injetamos o `UserRepo` no `UserController`, podemos melhorá-lo ligeiramente.

```ts
// controllers/userController.ts

import { UserRepo } from "../repos"; // Ainda é uma prática ruim

/**
 * @class UserController
 * @desc Responsável por lidar com solicitações de API para a rota /user
 **/

class UserController {
  private userRepo: UserRepo;

  constructor(userRepo: UserRepo) {
    this.userRepo = userRepo; // Muito Melhor, injetamos a dependência através do construtor
  }

  async handleGetUsers(req, res): Promise<void> {
    const users = await this.userRepo.getUsers();
    return res.status(200).json({ users });
  }
}
```

Mesmo que estejamos usando injeção de dependência, ainda há um problema.

`UserController` ainda depende diretamente de `UserRepo`.

![](https://khalilstemmler.com/img/blog/di-container/before-dependency-inversion.svg)
_Essa relação de dependência ainda é verdadeira._

Mesmo assim, se quiséssemos simular nosso `UserRepo`, que no código fonte se conecta a um banco de dados SQL real, criando um mock do repositório em memória, atualmente não é possível.

`UserController` precisa de um `UserRepo`, especificamente.

```ts
// controllers/userRepo.spec.ts

let userController: UserController;

beforeEach(() => {
  userController = new UserController(
    new UserRepo() // Deixará os testes lentos porque ele conecta ao banco de dados
  );
});
```

Então, o que podemos fazer?

É aqui que entra o **princípio de inversão de dependência**!

## Inversão de Dependência

Inversão de dependência é uma técnica que nos permite desacoplar componentes uns dos outros. Veja isso.

Em que direção o fluxo de dependências vai agora?

![](https://khalilstemmler.com/img/blog/di-container/before-dependency-inversion.svg)

Da esquerda para a direita. O `UserController` depende do `UserRepo`.

OK. Preparado?

Veja o que acontece quando nós colocamos uma interface entre os dois componentes. Mostrando que o `UserRepo` implementa uma interface `IUserRepo` e, em seguida, dizemos ao `UserController` para se referir a ela ao invés da classe concreta `UserRepo`.

```ts
// repos/userRepo.ts

/**
 * @interface IUserRepo
 * @desc Responsável por buscar usuários no banco de dados.
 **/
export interface IUserRepo {          // Exportado
  getUsers (): Promise<User[]>
}

class UserRepo implements IUserRepo { // Não é exportado
  constructor () {}

  getUsers (): Promise<User[]> {
    ...
  }
}
```

E atualizamos nosso controlador para usar a interface `IUserRepo` ao invés da classe concreta `UserRepo`.

```ts
// controllers/userController.ts

import { IUserRepo } from "../repos"; // Muito Melhor!

/**
 * @class UserController
 * @desc Responsável por lidar com solicitações de API para a rota /user
 **/

class UserController {
  private userRepo: IUserRepo; // Mudados Aqui

  constructor(userRepo: IUserRepo) {
    this.userRepo = userRepo; // E Aqui Também
  }

  async handleGetUsers(req, res): Promise<void> {
    const users = await this.userRepo.getUsers();
    return res.status(200).json({ users });
  }
}
```

Agora observe a direção do fluxo de dependências.

![](https://khalilstemmler.com/img/blog/di-container/after-dependency-inversion.svg)

Você viu o que acabamos de fazer? Alterando todas as referências de classes concretas para interfaces, acabamos de inverter o gráfico de dependência e criar um limite arquitetônico entre os dois componentes.

**Princípio de Design:** Programar em interfaces, não em implementações.

Talvez você não esteja tão animado com isso quanto eu. Deixe-me mostrar por que isso é ótimo.

> E se você gostou deste artigo até agora, talvez goste do meu livro, [Solid Book - The Software Architecture & Design Handbook w / TypeScript + Node.js](https://solidbook.io/). Você aprenderá como escrever código testável, flexível e sustentável usando princípios (como este) que eu acho que todos os profissionais de software deveriam conhecer. Dê uma olhada!

Lembra quando eu disse que queríamos ser capazes de executar testes no `UserController` sem ter que passar um `UserRepo`, apenas porque isso tornaria os testes lentos (`UserRepo` precisa de uma conexão de banco de dados para operar)?

Bem, agora podemos escrever um `MockUserRepo` que implementa a interface `IUserRepo` e todos os seus métodos, ao invés de usar uma classe que depende de uma conexão de banco de dados. Usar uma classe que contém um array interno de `User[]` é muito mais rápido!

É isso que vamos passar para o `UserController`.

## Usando um `MockUserRepo` para fazer o mock no `UserController`

```ts
// repos/mocks/mockUserRepo.ts

import { IUserRepo } from "../repos";

class MockUserRepo implements IUserRepo {
  private users: User[] = [];

  constructor() {}

  async getUsers(): Promise<User[]> {
    return this.users;
  }
}
```

**Dica:** Adicionar `async` a um método irá transformá-lo em uma Promise, facilitando a simulação de atividades assíncronas.

Podemos escrever um teste usando um framework de testes como Jest.

```ts
// controllers/userRepo.spec.ts

import { MockUserRepo } from "../repos/mock/mockUserRepo";

let userController: UserController;

const mockResponse = () => {
  const res = {};
  res.status = jest.fn().mockReturnValue(res);
  res.json = jest.fn().mockReturnValue(res);
  return res;
};

beforeEach(() => {
  userController = new UserController(
    new MockUserRepo() // Super Rapído! E válido, já que implementa IUserRepo.
  );
});

test("Should 200 with an empty array of users", async () => {
  let res = mockResponse();
  await userController.handleGetUsers(null, res);
  expect(res.status).toHaveBeenCalledWith(200);
  expect(res.json).toHaveBeenCalledWith({ users: [] });
});
```

Parabéns. Você acabou de aprender como escrever código testável!

## As principais vantagens de DI

Essa separação não apenas torna seu código testável, mas também melhora as seguintes características de seu código:

1. **Testabilidade:** Podemos substituir componentes pesados de infraestrutura por componentes fictícios durante o teste.
2. **Substituibilidade:** Se programarmos em uma interface, habilitamos uma arquitetura de _plug-and-play_ que adere ao [Princípio de Substituição de Liskov](https://khalilstemmler.com/articles/solid-principles/solid-typescript/), o que torna incrivelmente fácil trocar componentes válidos e programar em código que ainda não existe. Como a interface define a forma da dependência, tudo o que precisamos fazer para substituir a dependência atual é criar uma nova que siga o contrato definido pela interface. [Veja este artigo para se aprofundar nisso](https://khalilstemmler.com/articles/enterprise-typescript-nodejs/clean-nodejs-architecture/).
3. **Flexibilidade:** Seguindo o [Princípio de Aberto e Fechado](https://khalilstemmler.com/articles/solid-principles/solid-typescript/), um sistema deve ser aberto para extensão, mas fechado para modificação. Isso significa que se quisermos estender o sistema, precisamos apenas criar um novo plugin para estender o comportamento atual.
4. **Delegação:** _Inversão de Controle_ é o fenômeno que observamos quando delegamos comportamento para ser implementado por outra pessoa, mas fornecemos os _hooks / plug-ins / callbacks_ para isso acontecer. Projetamos o componente atual para inverter o controle para outro. Muitos frameworks da web são construídos com base neste princípio.

## Inversão de Controle e Inversão de Controle com Containers

Os aplicativos ficam muito maiores do que apenas dois componentes.

Não apenas precisamos garantir que estamos nos referindo a interfaces e NÃO a implementações concretas, mas também precisamos lidar com o processo de injeção manual de instâncias de dependências em tempo de execução.

Se seu aplicativo for relativamente pequeno ou se você tiver um guia de estilo para conectar dependências em sua equipe, poderá fazer isso manualmente.

Se você tem um aplicativo enorme e não tem um plano de como realizará a injeção de dependência em seu aplicativo, ele pode sair do controle.

É por essa razão que existem os **Containers de Inversão de Controle (IoC)**.

Eles funcionam exigindo que você:

1. Crie um container (que manterá todas as dependências do seu aplicativo
2. Torne essa dependência conhecida pelo container (especifique que é injetável)
3. Resolva as dependências de que você precisa, pedindo ao container para injetá-las

Alguns dos mais populares para JavaScript / TypeScript são [Awilix](https://github.com/jeffijoe/awilix) e [InversifyJS](http://inversify.io/).

Pessoalmente, não sou um grande fã deles e da lógica de estrutura específica da infraestrutura adicional que eles espalham por toda a minha base de código.

Se você é como eu e não gosta da _vida em containers_, tenho meu próprio guia de estilo para injetar dependências, sobre o qual falo bastante em [solidbook.io](https://solidbook.io/). Também estou trabalhando em algum conteúdo de vídeo, fique ligado!

**Inversão de Controle:** O fluxo de controle tradicional de um programa ocorre quando o programa faz apenas o que nós mandamos (hoje). A inversão do fluxo de controle acontece quando desenvolvemos frameworks ou apenas nos referimos à arquitetura de plugins com áreas de código que podem ser conectadas. Nessas áreas, podemos não saber (hoje) como queremos que ele seja usado, ou desejamos permitir que os desenvolvedores adicionem funcionalidades adicionais. Isso significa que cada gancho de ciclo de vida, em React.js ou Angular, é um bom exemplo de Inversão de Controle na prática. IoC também é frequentemente explicado pelo "Princípio de Design de Hollywood": _Não ligue para nós, nós ligaremos para você_.

# Créditos

- [Dependency Injection & Inversion Explained | Node.js w/ TypeScript](https://khalilstemmler.com/articles/tutorials/dependency-injection-inversion-explained/), escrito originalmente por [Khalil Stemmler](https://twitter.com/stemmlerjs).
