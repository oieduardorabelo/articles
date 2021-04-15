# Os Perigos dos ENUMS em TypeScript

O TypeScript apresenta muitos recursos novos que são comuns em linguagens de tipo estático, como [classes](https://www.typescriptlang.org/docs/handbook/classes.html) (que agora fazem parte da linguagem JavaScript), [interfaces](https://www.typescriptlang.org/docs/handbook/interfaces.html) , [genéricos](https://www.typescriptlang.org/docs/handbook/generics.html) e [tipos de união](https://www.typescriptlang.org/docs/handbook/advanced-types.html#union-types), para citar alguns.

Mas há um tipo especial que queremos discutir hoje e que é [enums](https://www.typescriptlang.org/docs/handbook/enums.html). Enum, abreviação de Enumerated Type, é um recurso de linguagem comum de muitas linguagens de tipos estáticos, como C, C #, Java, Swift e muitas outras, é um grupo de valores constantes nomeados que você pode usar em seu código.

Vamos criar um enum no TypeScript para representar os dias da semana:

```ts
enum DayOfWeek {
  Sunday,
  Monday,
  Tuesday,
  Wednesday,
  Thursday,
  Friday,
  Saturday,
}
```

O enum é denotado com a palavra-chave enum seguida pelo nome do enum (DayOfWeek) e, em seguida, definimos os valores constantes que queremos disponibilizar para o enum.

Poderíamos então criar uma função para determinar se é fim de semana e ter o argumento enum:

```ts
function isItTheWeekend(day: DayOfWeek) {
  switch (day) {
    case DayOfWeek.Sunday:
    case DayOfWeek.Saturday:
      return true;

    default:
      return false;
  }
}
```

E, finalmente, usá-lo assim:

```ts
console.log(isItTheWeekend(DayOfWeek.Monday)); // logs 'false'
```

Esta é uma boa maneira de remover o uso de valores mágicos dentro de uma base de código, uma vez que temos opções de representação de tipo seguro que estão todas relacionadas. Mas as coisas nem sempre são o que parecem. O que você acha que obterá se passar isso pelo compilador TypeScript?

```ts
console.log(isItTheWeekend(2)); // isso é válido?
```

Você pode ficar surpreso em saber que este é um TypeScript válido e que o compilador ficará feliz em aceitá-lo para você.

## Por quê isso aconteceu

Escrever este código pode fazer você pensar que descobriu um bug no sistema de tipos TypeScript, mas acontece que esse é o comportamento pretendido para esse tipo de enum. O que fizemos aqui foi criar um enum numérico e, se olharmos para o JavaScript gerado, pode ficar um pouco mais claro:

```ts
var DayOfWeek;
(function (DayOfWeek) {
  DayOfWeek[(DayOfWeek['Sunday'] = 0)] = 'Sunday';
  DayOfWeek[(DayOfWeek['Monday'] = 1)] = 'Monday';
  DayOfWeek[(DayOfWeek['Tuesday'] = 2)] = 'Tuesday';
  DayOfWeek[(DayOfWeek['Wednesday'] = 3)] = 'Wednesday';
  DayOfWeek[(DayOfWeek['Thursday'] = 4)] = 'Thursday';
  DayOfWeek[(DayOfWeek['Friday'] = 5)] = 'Friday';
  DayOfWeek[(DayOfWeek['Saturday'] = 6)] = 'Saturday';
})(DayOfWeek || (DayOfWeek = {}));
```

E se enviarmos para o console:

![](https://www.aaron-powell.com/images/typescript-enums/001.png)

Notaremos que o enum é na verdade apenas um objeto JavaScript com propriedades subjacentes, ele tem as propriedades nomeadas que definimos e são atribuídos a eles um número que representa a posição no enum que eles existem (domingo sendo 0, sábado sendo 6), mas o objeto também possui acesso numérico com um valor de string que representa a constante nomeada.

Portanto, podemos passar números para uma função que espera um enum, o próprio enum é um número e uma constante definida.

# Quando isso é útil

Você pode estar pensando que isso não parece particularmente útil, pois realmente quebra todo o aspecto de segurança de tipo do TypeScript se você puder passar um número arbitrário para uma função que espera um enum, então por que isso é útil?

Digamos que você tenha um serviço que retorna um JSON quando chamado e deseja modelar uma propriedade desse serviço como um valor enum. Em seu banco de dados, você pode ter esse valor armazenado como um número, mas definindo-o como um enum TypeScript, podemos convertê-lo corretamente:

```ts
const day: DayOfWeek = 3;
```

Esse cast explícito que está sendo feito durante a atribuição transformará a variável day de um número em nosso enum, o que significa que podemos obter um pouco mais de uma compreensão do que ele representa quando está sendo passado em nossa base de código.

## Controlando um Enums de Números

Como o número de um membro de enum é definido com base na ordem em que aparecem na definição de enum, pode ser um pouco opaco quanto ao valor até que você inspecione o código gerado, mas isso é algo que podemos controlar:

```ts
enum FileState {
  Read = 1,
  Write = 2,
}
```

Aqui está um novo enum que modela o estado em que um arquivo pode estar, pode estar no modo de leitura ou gravação e definimos explicitamente o valor que corresponde a esse modo (acabei de criar esses valores, mas pode ser algo vindo de nosso sistema de arquivos).

Agora está claro quais valores são válidos para este enum, já que fizemos isso explicitamente.

## Sinalizadores de Bits (_Bit Flags_)

Mas há outro motivo pelo qual isso pode ser útil: usar enums para sinalizadores de bits. Vamos pegar nosso `FileState` enum acima e adicionar um novo estado para o arquivo `ReadWrite`:

```ts
enum FileState {
  Read = 1,
  Write = 2,
  ReadWrite = 3,
}
```

Então, supondo que temos uma função que leva o enum, podemos escrever um código como este:

```ts
const file = await getFile('/path/to/file', FileState.Read | FileState.Write);
```

Observe como estamos usando o operador `|` no `FileState` enum e isso nos permite realizar uma operação bit a bit neles para criar um novo valor de enum; neste caso, ele criará 3, que é o valor do estado `ReadWrite`. Na verdade, podemos escrever isso de forma mais clara:

```ts
enum FileState {
  Read = 1,
  Write = 2,
  ReadWrite = Read | Write,
}
```

Agora que o membro ReadWrite não é uma constante codificada manualmente, está claro que é feito como uma operação bit a bit de outros membros do enum.

No entanto, temos que ter cuidado ao usar enums dessa forma, pegue o seguinte enum:

```ts
enum Foo {
  A = 1,
  B = 2,
  C = 3,
  D = 4,
  E = 5,
}
```

Se recebermos o valor enum `E` (ou `5`), é o resultado de uma operação bit a bit de `Foo.A | Foo.D` ou `Foo.B | Foo.C`? Então, se houver uma expectativa de que estamos usando enums bit a bit como este, queremos garantir que seja realmente óbvio como chegamos a esse valor.

## Controlando Índices

Vimos que um enum terá um valor numérico atribuído a ele por padrão ou podemos fazer isso explicitamente em todos eles, mas também podemos fazer em um subconjunto deles:

```ts
enum DayOfWeek {
  Sunday,
  Monday,
  Tuesday,
  Wednesday = 10,
  Thursday,
  Friday,
  Saturday,
}
```

Aqui, especificamos que o valor de 10 representará quarta-feira, mas todo o resto será deixado "como está", então o que isso gera no JavaScript?

```ts
var DayOfWeek;
(function (DayOfWeek) {
  DayOfWeek[(DayOfWeek['Sunday'] = 0)] = 'Sunday';
  DayOfWeek[(DayOfWeek['Monday'] = 1)] = 'Monday';
  DayOfWeek[(DayOfWeek['Tuesday'] = 2)] = 'Tuesday';
  DayOfWeek[(DayOfWeek['Wednesday'] = 10)] = 'Wednesday';
  DayOfWeek[(DayOfWeek['Thursday'] = 11)] = 'Thursday';
  DayOfWeek[(DayOfWeek['Friday'] = 12)] = 'Friday';
  DayOfWeek[(DayOfWeek['Saturday'] = 13)] = 'Saturday';
})(DayOfWeek || (DayOfWeek = {}));
```

Inicialmente, os valores são definidos usando sua posição no índice com domingo a terça sendo 0 a 2, então quando “zeramos” a ordem na quarta-feira, tudo depois disso é incrementado a partir da nova posição inicial.

Isso pode se tornar problemático se fizermos algo assim:

```ts
enum DayOfWeek {
  Sunday,
  Monday,
  Tuesday,
  Wednesday = 10,
  Thursday = 2,
  Friday,
  Saturday,
}
```

Fizemos a quinta-feira 2, então como é o nosso JavaScript gerado?

```ts
var DayOfWeek;
(function (DayOfWeek) {
  DayOfWeek[(DayOfWeek['Sunday'] = 0)] = 'Sunday';
  DayOfWeek[(DayOfWeek['Monday'] = 1)] = 'Monday';
  DayOfWeek[(DayOfWeek['Tuesday'] = 2)] = 'Tuesday';
  DayOfWeek[(DayOfWeek['Wednesday'] = 10)] = 'Wednesday';
  DayOfWeek[(DayOfWeek['Thursday'] = 2)] = 'Thursday';
  DayOfWeek[(DayOfWeek['Friday'] = 3)] = 'Friday';
  DayOfWeek[(DayOfWeek['Saturday'] = 4)] = 'Saturday';
})(DayOfWeek || (DayOfWeek = {}));
```

Ups, parece que pode haver um problema, 2 são terça e quinta! Se esse for um valor proveniente de uma fonte de dados de algum tipo, temos um ambigüidade em nosso aplicativo. Portanto, se vamos definir o valor, é melhor definir todos os valores para que seja óbvio o que são.

## Enums não numéricos

Até agora, discutimos apenas enums que são numéricos ou que atribuem números explicitamente a valores de enum, mas um enum não precisa ser um valor numérico, pode ser qualquer coisa constante ou valor calculado:

```ts
enum DayOfWeek {
  Sunday = 'Sun',
  Monday = 'Mon',
  Tuesday = 'Tues',
  Wednesday = 'Wed',
  Thursday = 'Thurs',
  Friday = 'Fri',
  Saturday = 'Sat',
}
```

Aqui, fizemos um enum de string e o código gerado é muito diferente:

```ts
var DayOfWeek;
(function (DayOfWeek) {
  DayOfWeek['Sunday'] = 'Sun';
  DayOfWeek['Monday'] = 'Mon';
  DayOfWeek['Tuesday'] = 'Tues';
  DayOfWeek['Wednesday'] = 'Wed';
  DayOfWeek['Thursday'] = 'Thurs';
  DayOfWeek['Friday'] = 'Fri';
  DayOfWeek['Saturday'] = 'Sat';
})(DayOfWeek || (DayOfWeek = {}));
```

Agora não poderemos mais passar um número para a função `isItTheWeekend`, já que o enum não é numérico, mas também não podemos passar uma string arbitrária, já que o enum sabe quais valores de string são válidos.

Isso introduz outro problema; não podemos mais fazer isso:

```ts
const day: DayOfWeek = 'Mon';
```

A string não pode ser atribuída diretamente ao tipo enum, em vez disso, temos que fazer uma conversão explícita:

```ts
const day = 'Mon' as DayOfWeek;
```

E isso pode ter um impacto sobre como consumimos valores que serão usados ​​como enum.

Mas por que parar nas strings? Na verdade, podemos misturar e combinar os valores de enums dentro de um próprio enum:

```ts
enum Confusing {
  A,
  B = 1,
  C = 1 << 8,
  D = 1 + 2,
  E = 'Hello World'.length,
}
```

Desde que todos os valores atribuíveis sejam do mesmo tipo (numérico, neste caso), podemos gerar esses números de várias maneiras diferentes, incluindo valores calculados, mas se forem todos constantes, podemos misturar tipos para fazer um enum heterogêneo:

```ts
enum MoreConfusion {
  A,
  B = 2,
  C = 'C',
}
```

Isso é muito confuso e pode dificultar o entendimento de como os dados funcionam por trás do enum, portanto, é recomendável não usar enums heterogêneos, a menos que você tenha certeza de que é o que você precisa.

## Conclusão

Enums no TypeScript são uma adição muito útil à linguagem JavaScript. Quando usados ​​corretamente, eles podem ajudar a esclarecer a intenção de normalmente “valores mágicos” (strings ou números) que podem existir em um aplicativo e fornecer uma visão segura de tipos deles. Mas, como qualquer ferramenta na caixa de ferramentas de alguém, se forem usadas incorretamente, pode não ficar claro o que representam e como devem ser usadas.

# Créditos

- [The Dangers of TypeScript Enums](https://www.aaron-powell.com/posts/2020-05-27-the-dangers-of-typescript-enums/), escrito originalmente por [Aaron Powell](https://twitter.com/slace).
