# Aplicando Padrões de Código com Pré-Commit Hook usando Husky

> Neste artigo, aprenderemos como configurar o Husky para evitar git commits ruins e impor padrões de código em seu projeto.

## Introdução

Na maioria dos projetos em que já trabalhei de forma colaborativa, alguém assume o papel de campeão de limpeza de código. Geralmente é o líder da equipe e, muitas vezes, sua função envolve revisar os [PRs](https://help.github.com/en/github/collaborating-with-issues-and-pull-requests/about-pull-requests) e garantir que o amor e o cuidado sejam colocados na qualidade do código.

A qualidade inclui as convenções e padrões de código escolhidas, além da formatação do código.

Hoje, é uma boa prática em projetos JavaScript utilizar [ESLint](https://khalilstemmler.com/blogs/typescript/eslint-for-typescript/) para definir as convenções de código do projeto. Por exemplo, como sua equipe se sente em relação ao uso de `for` loops? E o ponto-e-vírgula - são obrigatórios? Etc.

Essas são convenções.

A outra peça do quebra-cabeça é a formatação. Essa é a aparência visual do código. Quando há mais de um desenvolvedor trabalhando em um projeto, garantir que o código pareça consistente é algo a ser abordado.

[Prettier](https://khalilstemmler.com/blogs/tooling/prettier/) é a ferramenta correta para isso.

[No artigo anterior](https://khalilstemmler.com/blogs/tooling/prettier/), aprendemos como combinar ESLint e Prettier, mas não aprendemos como realmente aplicar as convenções e a formatação em um projeto da vida real com vários desenvolvedores.

Neste artigo, aprenderemos como usar o [Husky](https://github.com/typicode/husky) para fazer isso em um projeto baseado em Git.

## Husky

Husky é um pacote npm que "torna os hooks Git fáceis".

Quando você inicializa o [Git](https://git-scm.com/) (a ferramenta de controle de versão com a qual você provavelmente está familiarizado) em um projeto, ele vem automaticamente com um recurso chamado hooks.

Se você for até a raiz de um projeto inicializado com Git e digitar:

```bash
ls .git/hooks
```

Você verá uma lista de ganchos de exemplo, como `pre-push`, `pre-rebase`, `pre-commit`  e assim por diante. Esta é uma maneira de escrevermos algum código de plugin para executar alguma lógica antes de realizar a ação do Git.

Se quisermos garantir que seu código foi devidamente lintado e formatado, antes de alguém criar um commit usando o comando `git commit`, poderíamos escrever um hook Git de `pre-commit`.

Escrever isso manualmente provavelmente não seria divertido. Também seria um desafio distribuir e garantir que os hooks fossem instalados nas máquinas de outros desenvolvedores.

Esses são alguns dos desafios que o Husky pretende resolver.

Com o Husky, podemos garantir que, para um novo desenvolvedor trabalhando em nossa base de código (usando pelo menos o Node versão 10):

- Hooks são criados localmente
- Hooks são executados quando o comando Git é chamado
- Aplicar uma regra que define _como_ alguém pode contribuir para o projeto

Vamos configurá-lo.


## Instalando Husky

Para instalar o Husky, execute:

```bash
npm install husky --save-dev
```

## Configurando Husky

Para configurar o Husky, na raiz do nosso projeto no `package.json`, adicione a seguinte chave `husky`:

```json
{
  "husky": {
    "hooks": {
      "pre-commit": "", // seu comando vai aqui
      "pre-push": "", // seu comando vai aqui
      "...": "..."
    }
  }
}
```

Quando executamos o comando `git commit` ou `git push`, o respectivo hook executará o script que fornecemos em nosso `package.json`.

## Fluxo de trabalho de exemplo

Seguindo os exemplos dos artigos anteriores, se configurarmos o ESLint e Prettier, sugiro utilizar dois scripts:

```json

{
  "scripts": {
    "prettier-format": "prettier --config .prettierrc 'src/**/*.ts' --write",
    "lint": "eslint . --ext .ts",
    ...
  },
  "husky": {
    "hooks": {
      "pre-commit": "npm run prettier-format && npm run lint"
    }
  }
}
```

Inclua esses scripts no objeto `scripts` em seu `package.json`. E execute `prettier-format` e depois `lint` com um hook `pre-commit`.

Isso garantirá que você não possa completar um `commit` sem que seu código seja formatado de acordo com as convenções do seu time.

## Exemplo - Bloqueando um commit

Eu gosto de usar o pacote [no-loops](https://github.com/buildo/eslint-plugin-no-loops) como exemplo. Esta convenção não permite aos desenvolvedores usar `for` loops, ao invés disso, sugere que usamos funções de utilidade do Array como `forEach`, `map` e afins.

Adicionando o plug-in e sua regra ao `.eslintrc`:

```json
{
  "root": true,
  "parser": "@typescript-eslint/parser",
  "plugins": [
    "@typescript-eslint",
    "no-loops",
    "prettier"
  ],
  "extends": [
    "eslint:recommended",
    "plugin:@typescript-eslint/eslint-recommended",
    "plugin:@typescript-eslint/recommended",
    "prettier"
  ],
  "rules": {
    "no-loops/no-loops": 2, // 2 singifica "retornar um errro"
    "no-console": 1,
    "prettier/prettier": 2
  }
}
```

E vamos colocar um `for` loop no código-fonte:

```ts
console.log('Hello world!');

for (let i = 0; i < 12; i++) {
  console.log(i);
}
```

E ao tentar commitar, veremos um código de saída diferente de zero, o que, como sabemos, significa que ocorreu um erro:

```bash
simple-typescript-starter git:(prettier) ✗ git commit -m "Test commit"
husky > pre-commit (node v10.10.0)

> typescript-starter@1.0.0 prettier-format simple-typescript-starter
> prettier --config .prettierrc 'src/**/*.ts' --write

src/index.ts 191ms

> typescript-starter@1.0.0 lint /simple-typescript-starter
> eslint . --ext .ts


/simple-typescript-starter/src/index.ts
  1:1  warning  Unexpected console statement  no-console
  3:1  error    loops are not allowed         no-loops/no-loops
  4:3  warning  Unexpected console statement  no-console

✖ 3 problems (1 error, 2 warnings)
```

E aí está!

## Outras considerações

Se você perceber que o lint está demorando muito, verifique este pacote, [lint-staged](https://github.com/okonet/lint-staged). Ele executa o linter, mas apenas nos arquivos que estão em git stage (arquivos que você está pronto para commitar). Isso me foi sugerido por [@glambertmtl](https://twitter.com/glambertmtl). Obrigado!

# Créditos

- [Enforcing Coding Conventions with Husky Pre-commit Hooks](https://khalilstemmler.com/blogs/tooling/enforcing-husky-precommit-hooks/), escrito originalmente por [Khalil Stemmler](https://twitter.com/stemmlerjs).
