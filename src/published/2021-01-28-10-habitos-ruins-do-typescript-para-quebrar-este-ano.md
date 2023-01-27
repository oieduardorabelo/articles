- https://dev.to/oieduardorabelo/10-habitos-ruins-do-typescript-para-quebrar-este-ano-5hda
- https://oieduardorabelo.medium.com/10-h%C3%A1bitos-ruins-do-typescript-para-quebrar-este-ano-a89dabc524a6

---

# 10 hábitos ruins do TypeScript para quebrar este ano

[_Créditos da Imagem_](https://unsplash.com/photos/oJ9HIznaSQY)

O TypeScript e o JavaScript têm evoluído constantemente nos últimos anos, e alguns dos hábitos que construímos nas últimas décadas se tornaram obsoletos. Alguns podem nunca ter sido significativos. Aqui está uma lista de 10 hábitos que todos devemos quebrar.

Se você estiver interessado em mais artigos e notícias sobre desenvolvimento de produtos para a web e empreendedorismo, fique à vontade para [me seguir no Twitter](https://startup-cto.net/10-bad-typescript-habits-to-break-this-year/) .

Vamos aos exemplos! Observe que cada caixa "Como deve ser" apenas corrige o problema discutido, mesmo se houver outros "odores no código" (_code smells_) que devam ser resolvidos.

## 1. Não usar o modo `strict`

### O que isso parece

Usando um `tsconfig.json` sem modo estrito:

```json
{
  "compilerOptions": {
    "target": "ES2015",
    "module": "commonjs"
  }
}
```

### Como deve ser

Basta ativar o modo `strict`:

```json
{
  "compilerOptions": {
    "target": "ES2015",
    "module": "commonjs",
    "strict": true
  }
}
```

### Porque fazemos isso

A introdução de regras mais rígidas em uma base de código existente leva tempo.

### Porque não deveríamos

Regras mais rígidas tornarão mais fácil alterar o código no futuro, de modo que o tempo investido para consertar o código em modo estrito será retornado e até um pouco mais quando se trabalhar no repositório no futuro.

## 2. Definindo valores padrão com `||`

### O que isso parece

Aplicando valores opcionais com `||`:

```ts
function createBlogPost (text: string, author: string, date?: Date) {
  return {
    text: text,
    author: author,
    date: date || new Date()
  }
}
```

### Como deve ser

Use o novo operador `??` ou, melhor ainda, defina o fallback bem no nível do parâmetro.

```ts
function createBlogPost (text: string, author: string, date: Date = new Date()
  return {
    text: text,
    author: author,
    date: date
  }
}
```

### Porque fazemos isso

O operador `??` acabou de ser introduzido no ano passado e, ao usar valores no meio de uma função longa, pode ser difícil defini-los já como padrões de parâmetro.

### Porque não deveríamos

O `??`, ao contrário `||`, recai apenas para `null` ou `undefined`, não para todos os valores falsos. Além disso, se suas funções são tão longas que você não pode definir padrões no início, dividi-las pode ser uma boa ideia.

## 3. Usando `any` como tipo

### O que isso parece

Use `any` para dados quando você não tiver certeza sobre a estrutura.

```ts
async function loadProducts(): Promise<Product[]> {
  const response = await fetch('https://api.mysite.com/products')
  const products: any = await response.json()
  return products
}
```

### Como deve ser

Em quase todas as situações em que você digita algo como `any`, na verdade você deve digitar `unknown`.

```ts
async function loadProducts(): Promise<Product[]> {
  const response = await fetch('https://api.mysite.com/products')
  const products: unknown = await response.json()
  return products as Product[]
}
```

### Porque fazemos isso

`any` é conveniente, pois basicamente desativa todas as verificações de tipo. Freqüentemente, `any` é usado mesmo em tipos oficiais como `response.json()` (por exemplo, no exemplo acima é digitado como `Promise<any>` pela equipe do TypeScript).

### Porque não deveríamos

Basicamente, `any` desativa todas as verificações de tipo. Qualquer coisa que vier através de `any` irá ignorar completamente qualquer verificação de tipo. Isso leva a bugs difíceis de detectar, pois o código falhará apenas quando nossas suposições sobre a estrutura do tipo forem relevantes para o código de tempo de execução.

## 4. Usando `val as SomeType`

### O que isso parece

Informar ao compilador sobre um tipo que ele não pode inferir.

```ts
async function loadProducts(): Promise<Product[]> {
  const response = await fetch('https://api.mysite.com/products')
  const products: unknown = await response.json()
  return products as Product[]
}
```

### Como deve ser

É para isso que servem os Guardas de Tipos (_Type Guard_):

```ts
function isArrayOfProducts (obj: unknown): obj is Product[] {
  return Array.isArray(obj) && obj.every(isProduct)
}

function isProduct (obj: unknown): obj is Product {
  return obj != null
    && typeof (obj as Product).id === 'string'
}

async function loadProducts(): Promise<Product[]> {
  const response = await fetch('https://api.mysite.com/products')
  const products: unknown = await response.json()
  if (!isArrayOfProducts(products)) {
    throw new TypeError('Received malformed products API response')
  }
  return products
}
```

### Porque fazemos isso

Ao converter de JavaScript para TypeScript, a base de código existente costuma fazer suposições sobre tipos que não podem ser deduzidos automaticamente pelo compilador TypeScript. Nesses casos, adicionar um rápido `as SomeOtherType` pode acelerar a conversão sem ter que afrouxar as configurações no `tsconfig`.

### Porque não deveríamos

Mesmo que a declaração possa ser salva agora, isso pode mudar quando alguém mover o código. Os guardas de tipo irão garantir que todas as verificações sejam explícitas.

## 5. Usando `as any` em testes

### O que isso parece

Criação de substitutos incompletos ao escrever testes.

```ts
interface User {
  id: string
  firstName: string
  lastName: string
  email: string
}

test('createEmailText returns text that greats the user by first name', () => {
  const user: User = {
    firstName: 'John'
  } as any

  expect(createEmailText(user)).toContain(user.firstName)
}
```

### Como deve ser

Se você precisar simular dados para seus testes, mova a lógica de simulação para perto do que você simula e torne-a reutilizável:

```ts
interface User {
  id: string
  firstName: string
  lastName: string
  email: string
}

class MockUser implements User {
  id = 'id'
  firstName = 'John'
  lastName = 'Doe'
  email = 'john@doe.com'
}

test('createEmailText returns text that greats the user by first name', () => {
  const user = new MockUser()

  expect(createEmailText(user)).toContain(user.firstName)
}
```

### Porque fazemos isso

Ao escrever testes em uma base de código que ainda não tem uma grande cobertura de teste, geralmente existem grandes estruturas de dados complicadas, mas apenas partes delas são necessárias para a funcionalidade específica em teste. Não ter que se preocupar com as outras propriedades fica mais fácil no curto prazo.

### Porque não deveríamos

Abandonar a criação de um mock vai nos incomodar mais tarde quando uma das propriedades mudar e precisarmos mudá-la em todos os testes, em vez de em um local central. Além disso, haverá situações em que o código em teste depende de propriedades que não consideramos importantes antes e, em seguida, todos os testes para essa funcionalidade precisam ser atualizados.

## 6. Propriedades Opcionais

### O que isso parece

Marcando propriedades como opcionais que às vezes existem e às vezes não.

```ts
interface Product {
  id: string
  type: 'digital' | 'physical'
  weightInKg?: number
  sizeInMb?: number
}
```

### Como deve ser

Modele explicitamente quais combinações existem e quais não.

```ts
interface Product {
  id: string
  type: 'digital' | 'physical'
}

interface DigitalProduct extends Product {
  type: 'digital'
  sizeInMb: number
}

interface PhysicalProduct extends Product {
  type: 'physical'
  weightInKg: number
}
```

### Porque fazemos isso

Marcar propriedades como opcionais em vez de separar os tipos é mais fácil e produz menos código. Também requer uma compreensão mais profunda do produto que está sendo construído e pode limitar o uso do código se as suposições sobre o produto mudarem.

### Porque não deveríamos

O grande benefício dos sistemas de tipo é que eles podem substituir as verificações de tempo de execução por verificações de tempo de compilação. Com uma digitação mais explícita, é possível obter verificações em tempo de compilação para bugs que, de outra forma, poderiam ter passado despercebidos, por exemplo, certificando-se de que todos `DigitalProduct` tenham um `sizeInMb`.

## 7. Tipos Genéricos de uma letra

### O que isso parece

Nomeando um genérico com uma letra:

```ts
function head<T> (arr: T[]): T | undefined {
  return arr[0]
}
```

### Como deve ser

Fornecendo um nome de tipo descritivo completo.

```ts
function head<Element> (arr: Element[]): Element | undefined {
  return arr[0]
}
```

### Porque fazemos isso

Acho que esse hábito cresceu porque [até mesmo os documentos oficiais usam nomes de uma letra](https://www.typescriptlang.org/docs/handbook/generics.html) . Também é mais rápido digitar e requer menos reflexão ao pressionar `T` ao invés de escrever um nome completo.

### Porque não deveríamos

Variáveis ​​de tipo genérico são variáveis, como qualquer outra. Abandonamos a ideia de descrever os detalhes técnicos das variáveis ​​em seus nomes quando as IDEs começaram a nos mostrar esses detalhes técnicos. Por exemplo, ao invés de `const strName = 'Daniel'` agora apenas escrevemos `const name = 'Daniel'`. Além disso, nomes de variáveis ​​de uma letra geralmente são malvistos porque pode ser difícil decifrar o que significam sem olhar para sua declaração.

## 8. Verificações booleanas e não booleanas

### O que isso parece

Verificar se um valor é definido passando o valor diretamente para uma instrução `if`.

```ts
function createNewMessagesResponse (countOfNewMessages?: number) {
  if (countOfNewMessages) {
    return `You have ${countOfNewMessages} new messages`
  }
  return 'Error: Could not retrieve number of new messages'
}
```

### Como deve ser

Verificando explicitamente a condição que nos interessa.

```ts
function createNewMessagesResponse (countOfNewMessages?: number) {
  if (countOfNewMessages !== undefined) {
    return `You have ${countOfNewMessages} new messages`
  }
  return 'Error: Could not retrieve number of new messages'
}
```

### Porque fazemos isso

Escrever o `if` em resumo parece mais sucinto e nos permite evitar pensar sobre o que realmente queremos verificar.

### Porque não deveríamos

Talvez devêssemos pensar sobre o que realmente queremos verificar. Os exemplos acima, por exemplo, tratam do caso de `countOfNewMessages` ser `0` diferente.

## 9. O operador BangBang

### O que isso parece

Converter um valor não booleano em booleano.

```ts
function createNewMessagesResponse (countOfNewMessages?: number) {
  if (!!countOfNewMessages) {
    return `You have ${countOfNewMessages} new messages`
  }
  return 'Error: Could not retrieve number of new messages'
}
```

### Como deve ser

Verificando explicitamente a condição que nos interessa.

```ts
function createNewMessagesResponse (countOfNewMessages?: number) {
  if (countOfNewMessages !== undefined) {
    return `You have ${countOfNewMessages} new messages`
  }
  return 'Error: Could not retrieve number of new messages'
}
```

### Porque fazemos isso

Para alguns, a compreensão `!!` é como um ritual de iniciação ao mundo do JavaScript. Parece curto e sucinto, e se você já está acostumado, então sabe do que se trata. É um atalho para converter qualquer valor em booleano. Especialmente se, em uma base de código, não há separação semântica clara entre valores falsos como `null`, `undefined` e `''`.

### Porque não deveríamos

Como muitos atalhos e rituais de iniciação, o uso `!!` ofusca o verdadeiro significado do código, promovendo o conhecimento interno. Isso torna a base de código menos acessível para novos desenvolvedores, seja ela nova no desenvolvimento em geral ou apenas nova no JavaScript. Também é muito fácil introduzir bugs sutis. O problema de `countOfNewMessages` ser `0` em "verificações booleanas não booleanas" persiste com `!!`.

## 10. Usando `!= null`

### O que isso parece

A irmã mais nova do operador BangBang, `!= null` permite verificar `null` e `undefined`ao mesmo tempo.

```ts
function createNewMessagesResponse (countOfNewMessages?: number) {
  if (countOfNewMessages != null) {
    return `You have ${countOfNewMessages} new messages`
  }
  return 'Error: Could not retrieve number of new messages'
}
```

### Como deve ser

Verificando explicitamente a condição que nos interessa.

```ts
function createNewMessagesResponse (countOfNewMessages?: number) {
  if (countOfNewMessages !== undefined) {
    return `You have ${countOfNewMessages} new messages`
  }
  return 'Error: Could not retrieve number of new messages'
}
```

### Porque fazemos isso

Se você chegou aqui, sua base de código e suas habilidades já estão em boa forma. Mesmo a maioria dos conjuntos de regras de linting que obrigam o uso de `!==` ao invés de `!=` oferecem uma isenção para `!= null`. Se não houver uma distinção clara na base de código entre `null` e `undefined`, o `!= null` ajudará a reduzir a verificação para ambas as possibilidades.

### Porque não deveríamos

Embora os valores `null` fossem um incômodo nos primeiros dias do JavaScript, com o TypeScript no modo `strict`, eles podem se tornar um membro valioso do cinturão de ferramentas da linguagem. Um padrão comum que tenho visto é definir valores `null` como coisas que não existem e `undefined` como coisas que não são desconhecidas, por exemplo, `user.firstName === null` pode significar que o usuário literalmente não tem um primeiro nome, enquanto `user.firstName === undefined` significa apenas que não perguntamos a esse usuário ainda (e `user.firstName === ''` significaria que o primeiro nome é literalmente `''` - você ficaria surpreso [com os tipos de nomes que realmente existem](https://www.kalzumeus.com/2010/06/17/falsehoods-programmers-believe-about-names/) ).

# Créditos

- [10 bad TypeScript habits to break this year](https://startup-cto.net/10-bad-typescript-habits-to-break-this-year/), escrito originalmente por [Daniel Bartholomae](https://twitter.com/the_startup_cto)
