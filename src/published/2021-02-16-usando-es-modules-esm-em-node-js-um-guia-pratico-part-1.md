- https://dev.to/oieduardorabelo/usando-es-modules-esm-em-node-js-um-guia-pratico-part-1-3bjp
- https://oieduardorabelo.medium.com/usando-es-modules-esm-em-node-js-um-guia-pr%C3%A1tico-parte-1-5577d9adc580

---

# Usando ES Modules (ESM) em Node.js: Um guia Prático - Parte 1

_(Ei, se você quiser vir trabalhar comigo na Roundforest e experimentar ESM em Node.js, sinta-se à vontade para entrar em contato no [LinkedIn](https://www.linkedin.com/in/giltayar/) ou no [Twitter](https://twitter.com/giltayar))_

Módulos ES são o futuro dos módulos em JavaScript. Eles já são a regra no frontend, mas até agora não eram usados ​​no Node.js. Agora nós podemos! Além disso, a comunidade Node.js está trabalhando rapidamente para adicionar suporte para ESM no Node.js. Isso inclui ferramentas como Mocha, Ava e até Jest (embora no Jest o suporte seja incremental). Além disso, ESlint e TypeScript funcionam bem com ESM, apesar de precisarmos de alguns truques.

Este guia mostra como usar o ESM no Node.js, detalhando os fundamentos e também as pegadinhas com as quais você precisa ter cuidado. Você pode encontrar todo o código no [repositório do GitHub](https://github.com/giltayar/jsm-in-nodejs-guide). É um monorepo onde cada pacote exibe uma certa estrutura do suporte ESM do Node.js. Este post passa por cada um dos pacotes, explicando o que foi feito lá e quais são as pegadinhas.

Este guia acabou sendo bem longo, então eu o dividi em três partes:

1.  Parte 1 - O Básico (este artigo que você está lendo)
2.  Parte 2 - "exports" e seus usos (incluindo bibliotecas de duplo-módulos)
3.  Parte 3 - Ferraments e TypeScript

**Importante:** Este guia abrange **Node.js ESM** e _não_ cobre ESM em navegadores.

## O que quero dizer com ESM em Node.js? Já não temos isso?

ESM é o sistema de módulo JavaScript padrão (ESM é um abreviamento para Módulos JavaScript que também é chamado de ESM, ou Módulos EcmaScript, em que “EcmaScript” é o nome oficial da especificação da linguagem JavaScript). ESM é o sistema de módulo “mais novo” e deve ser um substituto para o sistema de módulo Node.js atual, que é CommonJS (CJS para abreviar), embora o CommonJS provavelmente ainda estará conosco por muito, muito tempo. A sintaxe do módulo é esta:

```js
// add.js
export function add(a, b) {
  return a + b;
}

// main.js
import { add } from "./add.js";
```

_(Uma introdução ao ESM está fora do escopo deste guia, mas você pode encontrá-la hoje em qualquer lugar na Internet)_

O ESM foi padronizado em 2015, mas demorou um pouco para os navegadores suportarem isso, e demorou ainda mais para o Node.js suportá-lo (a versão estável final no Node.js foi finalizada apenas em 2020!). Se você quiser mais informações, pode ver minha [palestra no Node.TLV](https://www.youtube.com/watch?v=kK_3OP0uJ0Y). Na palestra, no final, discuto se o ESM está pronto para funcionar, e digo que ainda não está lá e as pessoas devem começar a migrar para ele em um ou dois anos. Bem, esse ano é AGORA e está PRONTO, e este guia irá prepará-lo para isso.

Alguns de vocês podem estar balançando a cabeça e se perguntando: já não estamos usando isso? Bem, se estiver, então você está transpilando seu código usando Babel ou TypeScript, que oferecem suporte a ESM pronto para uso e o transpilando para CJS. O ESM sobre o qual esta postagem está falando é o ESM _nativo_ compatível com Node.js sem transpilar. Embora sintaticamente seja o mesmo, existem pequenas diferenças entre ele e o Babel / TypeScript ESM, diferenças que são discutidas em minha palestra no Node.TLV acima. Mais importante ainda, o ESM nativo em Node.js não precisa de transpilação e, portanto, não vem com a bagagem de problemas que a transpilação traz.

## Sem enrolação, posso começar a usar o ESM no Node.js?

Sim. Praticamente, sim. Todas as ferramentas que eu uso suportam isso, mas há duas pegadinhas que são provavelmente difíceis de engolir para algumas pessoas, pegadinhas que são difíceis de contornar:

- O suporte do Jest para ESM em Node.js é [experimental](https://jestjs.io/docs/en/ecmascript-modules)
- O suporte experimental do Jest ainda não suporta módulos de simulação (_mocking modules_) mas funções regulares e simulação de objeto são compatíveis.
- `proxyquire` e outros mockers de módulo populares ainda não suportam ESM (embora `testdouble` seja totalmente compatível)

O maior problema é a falta de suporte para mockers de módulo. Temos _uma_ biblioteca de mock que suporta ESM, a [`testdouble`](https://www.npmjs.com/package/testdouble), e usamos ela neste guia.

Então você pode viver com isso? Se você puder, ir _all-in_ com ESM no Node.js agora é totalmente possível. Estou usando há quatro meses, sem problemas. Na verdade, parece que o suporte VSCode para ESM é muito melhor do que para CJS, então de repente recebo importações automáticas de módulos e outras vantagens, que não recebia antes no mundo CJS.

# O guia para Node.js ESM

1.  Parte 1 - O Básico (este artigo que você está lendo)
    1.1. Um pacote Node.js ESM simples
    1.2. Usando a extensão `.js` no ESM
2.  Parte 2 - "exports" e seus usos (incluindo bibliotecas de duplo-módulos)
    2.1. O campo "exports"
    2.2. Exportações múltiplas
    2.3. Bibliotecas de duplo-módulos
3.  Parte 3 - Ferraments e TypeScript
    3.1. Ferramentas
    3.2. TypeScript

Este guia vem com um monorepo que possui 7 diretórios, cada diretório sendo um pacote que demonstra as seções acima do suporte Node.js para ESM. Você pode encontrar o monorepo [nesse link](https://github.com/giltayar/jsm-in-nodejs-guide).

# Um pacote Node.js ESM simples

Código complementar: [https://github.com/giltayar/jsm-in-nodejs-guide/tree/main/01-simplest-mjs](https://github.com/giltayar/jsm-in-nodejs-guide/tree/main/01-simplest-mjs)

Esse é o exemplo mais simples e demonstra os fundamentos básico. Vamos começar explorando `package.json` e o novo campo `exports`.

### `main` e `.mjs`

Código: [https://github.com/giltayar/jsm-in-nodejs-guide/blob/main/01-simplest-mjs/package.json](https://github.com/giltayar/jsm-in-nodejs-guide/blob/main/01-simplest-mjs/package.json)

```json
{
  "name": "01-simplest-mjs",
  "version": "1.0.0",
  "description": "",
  "main": "src/main.mjs"
}
```

O principal ponto de entrada é `src/main.mjs`. Por que o arquivo usa a extensão `.mjs`? Porque no Node.js ESM, a extensão `.js` é reservada para CJS e `.mjs` significa que este é um Módulo JS (na próxima seção, veremos como mudar isso). Falaremos um pouco mais sobre isso na próxima parte.

Vamos continuar explorando `main.mjs`.

### "imports" usando extensões

Código: [https://github.com/giltayar/jsm-in-nodejs-guide/blob/main/01-simplest-mjs/src/main.mjs](https://github.com/giltayar/jsm-in-nodejs-guide/blob/main/01-simplest-mjs/src/main.mjs)

```js
// src/main.mjs
import { bannerInColor } from "./banner-in-color.mjs";

export function banner() {
  return bannerInColor("white");
}
```

Observe a instrução de importação que importa `banner-in-color`: Node.js ESM _força_ você a especificar o caminho relativo completo para o arquivo, _incluindo a extensão_ . A razão pela qual eles fizeram isso é para serem compatíveis com o ESM do navegador (ao usar o ESM em navegadores, você sempre especifica o nome completo do arquivo, incluindo a extensão). Portanto, não se esqueça dessa extensão! (Você pode entender mais sobre isso em minha [palestra na Node.TLV](https://www.youtube.com/watch?v=kK_3OP0uJ0Y)).

Infelizmente, o VSCode não gosta da extensão `.mjs` e, portanto, Ctrl / Cmd + click nele não funcionará, e seu intellisense embutido não funciona nele.

**Pegadinha** : VSCode não gosta da extensão `.mjs` e ignora essa extensão. Na próxima seção, veremos como lidar com isso, então não é um problema _real_.

O `main.mjs` exporta a função `banner`, que será testada em `test/tryout.mjs`. Mas primeiro, vamos explorar `banner-in-color.mjs`, que contém a maior parte da implementação da função `banner()`.

### Importando pacotes ESM e CJS

Código: [https://github.com/giltayar/jsm-in-nodejs-guide/blob/main/01-simplest-mjs/src/banner-in-color.mjs](https://github.com/giltayar/jsm-in-nodejs-guide/blob/main/01-simplest-mjs/src/banner-in-color.mjs)

Vimos como podemos importar módulos ESM. Vamos ver como importar outros pacotes:

```js
// src/banner-in-color.mjs
import { join } from "path";
import chalk from "chalk";
const { underline } = chalk;
```

Podemos importar pacotes internos do Node.js como `path` facilmente, porque Node.js os expõe como módulos ES.

E se tivéssemos um pacote ESM no NPM, o mesmo poderia ter sido usado para importar esse pacote ESM. Mas a maioria dos pacotes que o NPM possui ainda são pacotes CJS. Como você pode ver na segunda linha, onde importamos `chalk`, os pacotes CJS também podem ser importados usando `import`. Mas, na maior parte, ao importar módulos CJS, você só pode usar a importação “padrão” (_default_) e não as importações “nomeadas”. Portanto, embora você possa importar importações nomeadas em um arquivo CJS:

```js
// -a-cjs-file.cjs
const { underline } = require("chalk");
```

Você _não pode_ fazer isso em um arquivo ESM:

```js
// -a-jsm-file.mjs
import { underline } from "chalk";
```

Você só pode importar a importação padrão (não noemada) e usar a desestruturação mais tarde:

```js
import chalk from "chalk";
const { underline } = chalk;
```

Por que disso? É complicado, mas o ponto principal é que, ao carregar módulos, o ESM não permite a _execução de_ um módulo para determinar o que são as exportações e, portanto, as exportações precisam ser determinadas estaticamente. Infelizmente, no CJS, executar um módulo é a única maneira confiável de determinar quais são as exportações. Na verdade, o Node.js tenta muito descobrir quais são as exportações nomeadas (parseando e analisando o módulo usando um parser muito rápido), mas minha experiência é que esse método não funciona para a maioria dos pacotes com os quais tentei, e eu precisa voltar à importação padrão.

**Pegadinha**: Importar um módulo CJS é fácil, mas geralmente, você não pode usar importações nomeadas e precisa adicionar uma segunda linha para desestruturar as importações nomeadas.

Eu acredito que em 2021, mais e mais pacotes terão pontos de entrada ESM que se exportam como ESM com as exportações nomeadas corretas. Mas, por enquanto, você pode usar a desestruturação adicional para usar as importações nomeadas de pacotes CJS.

### "await" de nível superior

Código: [https://github.com/giltayar/jsm-in-nodejs-guide/blob/main/01-simplest-mjs/src/banner-in-color.mjs](https://github.com/giltayar/jsm-in-nodejs-guide/blob/main/01-simplest-mjs/src/banner-in-color.mjs)

Continuando nossa exploração de `banner-in-color.mjs`, encontramos esta linha extraordinária que lê um arquivo do disco:

```js
// src/banner-in-color.mjs
const text = await fs.readFile(join(__dirname, "text.txt"), "utf8");
```

Por que tão extraordinário? Por causa do `await`. Isso é um `await` fora de uma função `async` e está no nível superior do código. Esse `await`é chamado de "await de nível superior" (_top-level await_) e é compatível desde o Node.js v14. É extraordinário porque é o único recurso em Node.js que está disponível _apenas_ em módulos ESM (ou seja, não disponível em CJS). Por que? Como o ESM é um sistema de módulo assíncrono e, portanto, oferece suporte a operações assíncronas ao carregar o módulo, enquanto o CJS é carregado de forma síncrona e, portanto, não tem suporte `await`.

Excelente recurso, e apenas no ESM! 🎉🎉🎉🎉

Mas observe o uso de `__dirname` na linha acima. Vamos discutir isso.

### `__dirname`

Código: [https://github.com/giltayar/jsm-in-nodejs-guide/blob/main/01-simplest-mjs/src/banner-in-color.mjs](https://github.com/giltayar/jsm-in-nodejs-guide/blob/main/01-simplest-mjs/src/banner-in-color.mjs)

Se você tentar usar `__dirname` no ESM, verá que ele não está disponível (assim como `__filename`). Mas se você precisar, pode trazê-lo rapidamente usando estas linhas:

```js
// src/banner-in-color.mjs
import url from "url";

const __dirname = url.fileURLToPath(new URL(".", import.meta.url));
```

Complexo? Sim. Portanto, vamos desconstruir esse código para entendê-lo.

Em primeiro lugar, a expressão `import.meta.url` faz parte da especificação do ESM e seu propósito é o mesmo do CJS `__filename`, exceto que é uma _URL_ e não um caminho de arquivo. Por que URLs? Porque o ESM é definido em termos de URLs e não de caminhos de arquivo (para ser compatível com o navegador). Aliás, o URL que obtemos não é um URL HTTP. É um “ `file://...`” URL.

Agora que temos a URL do arquivo atual, precisamos da URL pai para chegar ao diretório e usaremos `new URL('.', import.meta.url)`para chegar até ela (o motivo pelo qual isso funciona está fora do escopo deste guia). Finalmente, para obter o caminho do arquivo e não a URL, precisamos de uma função que converta entre os dois e o módulo `url` do Node.js nos fornece isso por meio da função `url.fileURLToPath`.

Finalmente, colocamos o caminho do diretório em uma variável chamada `__dirname`, assim chamada pelas tradições do Node.js 😀.

### Testando este módulo

Código: [https://github.com/giltayar/jsm-in-nodejs-guide/blob/main/01-simplest-mjs/test/tryout.mjs](https://github.com/giltayar/jsm-in-nodejs-guide/blob/main/01-simplest-mjs/test/tryout.mjs)

```js
// test/tryout.mjs
import assert from "assert";
import { banner } from "../src/main.mjs";

assert.strict.match(banner(), /The answer is.*42/);

console.log(banner());
```

O teste executará `test/tryout.mjs`, no qual fará o `import` do módulo `src/main.mjs`, que utilizará (como vimos acima) várias importações CJS e ESM, para exportar uma função do banner colorido retornando a resposta (para a vida, o universo e tudo) de valor `42`. Ele afirmará que a resposta é tal, e com `console.log` podemos vê-la com toda a sua glória.

Para executar o teste, faça cd para `01-simplest-js`e execute:

```shell
npm install
npm test
```

Sim! Escrevemos nosso primeiro pacote ESM! Agora vamos fazer o mesmo, mas com uma extensão `.js`!

# Usando a extensão `.js` para ESM

Código complementar: [https://github.com/giltayar/jsm-in-nodejs-guide/blob/main/02-simplest-js](https://github.com/giltayar/jsm-in-nodejs-guide/blob/main/02-simplest-js)

Como vimos na seção anterior, a extensão `.mjs` é problemática, porque as ferramentas ainda não a suportam totalmente. Queremos nossa extensão `.js` de volta, e é isso que faremos nesta seção, com uma mudança muito simples no `package.json`.

### `type: module`

Código: [https://github.com/giltayar/jsm-in-nodejs-guide/blob/main/02-simplest-js/package.json](https://github.com/giltayar/jsm-in-nodejs-guide/blob/main/02-simplest-js/package.json)

```json
{
  "name": "02-simplest-js",
  "version": "1.0.0",
  "description": "",
  "type": "module",
  "main": "src/main.js"
}
```

Existe uma maneira muito simples de fazer com que todos os seus arquivos `.js` sejam interpretados como ESM e não como CJS: basta adicionar `"type": "module"` ao seu `package.json`, como acima. É isso aí. A partir desse ponto, todos os arquivos `.js` serão interpretados como ESM, portanto, todo o seu código pode agora usar a extensão `.js`.

Você ainda pode usar `.mjs` que sempre será ESM. Além disso, se você precisar de um módulo CJS em seu código, pode usar a nova extensão `.cjs` (veremos como usamos isso na seção “Bibliotecas de duplo-módulos”).

É isso aí. O restante do código neste diretório usa `.js`, e ao importar, também usaremos a extensão `.js`:

Código: [https://github.com/giltayar/jsm-in-nodejs-guide/blob/main/02-simplest-js/src/main.js](https://github.com/giltayar/jsm-in-nodejs-guide/blob/main/02-simplest-js/src/main.js)

```js
// src/main.js
import { bannerInColor } from "./banner-in-color.js";
```

É isso para o básico. Para a próxima parte deste guia, onde aprendemos sobre uma característica importante do ESM: `exports`.

# Créditos

- [Using ES Modules (ESM) in Node.js: A Practical Guide (Part 1)](https://gils-blog.tayar.org/posts/using-jsm-esm-in-nodejs-a-practical-guide-part-1/), escrito originalmente por [Gil Tayar](https://twitter.com/giltayar).
