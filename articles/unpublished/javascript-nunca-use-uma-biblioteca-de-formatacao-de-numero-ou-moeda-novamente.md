# JavaScript - Nunca use uma biblioteca de formatação de número ou moeda novamente

## Conteúdo

1. Introdução
2. Formato Numérico
3. Formato de Moeda
4. Formato de Unidades
5. Resumo

## Introdução

Reduzir as dependências que você envia no seu front-end é sempre uma coisa boa!
Se você estiver usando uma biblioteca de formatação de número ou moeda, verifique no [Bundlephobia](https://bundlephobia.com/) e veja quanto tempo e bytes ela adiciona ao seu aplicativo.

Tudo isso pode ser feito com uma nova API para vários navegadores! [Intl.NumberFormat](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl/NumberFormat).

## Formato Numérico

Formatar números é difícil! Adicionando separadores de milhares, casas decimais e assim por diante. Vale lembrar a internacionalização também! Alguns idiomas usam separadores de vírgula, alguns separadores de ponto e isso é apenas o começo!

Entra o [Intl.NumberFormat](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl/NumberFormat).

A API [Intl](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl) tem alguns métodos realmente úteis, mas vamos nos concentrar na formatação de números neste blog.

Vamos pular direto com um exemplo:

```js
const numberFormat = new Intl.NumberFormat('ru-RU');
console.log(numberFormat.format(654321.987));
// → "654 321,987"
```

Aqui, especificamos o local como Russo; no entanto, se você usar o construtor sem passar um local, ele será detectado automaticamente com base no navegador do usuário.

O que significa que mudará dependendo da preferência do usuário, localizando para seus usuários:

```js
const numberFormat = new Intl.NumberFormat();
console.log(numberFormat.format(654321.987));
```

Isso é [compatível com todos os navegadores](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl/NumberFormat#browser_compatibility) hoje em dia, incluindo Safari!

Mas podemos ir ainda mais longe...

## Formato de Moeda

Não apenas podemos formatar números dessa forma, mas também podemos oferecer suporte a moedas. Este é um suporte relativamente novo entre navegadores, então depende de quais versões do Safari seus usuários estão usando.

Isso funciona muito bem para formatar números:

```js
const number = 123456.789;

console.log(new Intl.NumberFormat('de-DE', { style: 'currency', currency: 'EUR' }).format(number));
// saída esperada: "123.456,79 €"
```

Existe [suporte para todas as moedas](http://www.currency-iso.org/en/home/tables/table-a1.html) que eu poderia imaginar.

Lembre-se de que isso não fará nenhuma conversão de valores de moeda entre elas, apenas o formato de exebição.

## Formato de Unidades

Eu não sabia disso até fazer a pesquisa para esse artigo. Mas você pode até formatar unidades! Isso ainda não é compatível com o Safari, portanto, verifique novamente a [compatibilidade do navegador](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl/NumberFormat/NumberFormat#browser_compatibility).

```js
new Intl.NumberFormat('en-US', {
    style: 'unit',
    unit: 'liter',
    unitDisplay: 'long'
}).format(amount);
// → '3,500 liters'
```

Há uma lista enorme de unidades com suporte, incluindo velocidade e muito mais. Ele até permite que você formate porcentagens, o que sempre achei uma dor de cabeça!

```js
new Intl.NumberFormat("en-US", {
    style: "percent",
    signDisplay: "exceptZero"
}).format(0.55);
// → '+55%'
```

## Resumo

O [Intl.NumberFormat](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl/NumberFormat) é uma ferramenta realmente poderosa no arsenal dos desenvolvedores web!

Não há necessidade de adicionar dependências adicionais ao seu aplicativo. Aumente a velocidade e o suporte internacional com a API Intl!

Bom código!

# Créditos

- [Never use a number or currency formatting library again!](https://dev.to/jordanfinners/never-use-a-number-or-currency-formatting-library-again-mhb), escrito originalmente por [Jordan Finneran](https://twitter.com/JordanFinners).
