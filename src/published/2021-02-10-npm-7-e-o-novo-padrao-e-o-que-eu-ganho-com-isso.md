- https://dev.to/oieduardorabelo/npm-7-e-o-novo-padrao-e-o-que-eu-ganho-com-isso-c7
- https://oieduardorabelo.medium.com/npm-7-%C3%A9-o-novo-padr%C3%A3o-e-o-que-eu-ganho-com-isso-c1c387163da0

---

# npm 7 é o novo padrão e o que eu ganho com isso?

[_Créditos da Imagem_](https://morioh.com/p/7ffcf5a9cdf3)

Finalmente, o npm 7 está geralmente disponível e publicado como o mais recente no registro do npm. Leia sobre as diferenças, novos recursos e melhorias de desempenho em comparação com o npm 6.

Com a versão 7 do npm, eles reduziram suas dependências em cerca de 54%, enquanto aumentaram a cobertura de testes em cerca de 17%. Também deve incluir um aumento de desempenho em várias áreas de acordo com [seus próprios benchmarks](https://github.com/npm/benchmarks) .

npm 7 agora é a versão `latest` no registro npm e, portanto, é o padrão. Para instalar a nova versão do npm, você pode executar o seguinte comando em seu terminal de linha de comando:

```bash
npm install --global npm@latest
```

A nova versão principal vem com alguns novos recursos e melhorias excelentes, incluindo espaços de trabalho (_Workspaces_), dependências de pares (_peer dependencies_) e um novo arquivo de bloqueio (_lockfile_). Ele também vem com algumas alterações importantes. Vamos ver quais são!

## Novas Funcionalidades

### 1) Versão 2 do arquivo _package-lock_

Com o novo arquivo `package-lock.json`, teremos a capacidade de fazer compilações reproduzíveis de forma determinística. Agora ele deve incluir tudo que o npm precisa para instalar os pacotes necessários. Antes do npm 7, o `yarn.lock` era ignorado pelo npm, mas não é mais o caso. Agora ele pode usá-lo para se manter atualizado com a árvore de pacotes.

O novo _lockfile_ deve ser compatível com os usuários do npm 6. No entanto, quando você executa `npm install` em um projeto com um _lockfile_ da versão 1, ele substituirá esse arquivo pela nova estrutura. Isso pode ser evitado executando `npm install --no-save` durante a instalação.

### 2) Espaços de Trabalho (_Workspaces_)

Este é um dos novos recursos com o qual estou mais animado. Inclui um conjunto de funcionalidades que irão tornar muito melhor a gestão de múltiplos pacotes. Ele permite que você manipule pacotes de um único arquivo na raiz do seu projeto. Isso já foi possível fazer com, por exemplo, _yarn_, _Lerna_ ou _pnpm_.

Para tornar o npm ciente de que o projeto atual é um espaço de trabalho, você deve adicionar a chave `workspaces` ao seu `package.json`. Isso pode ser feito adicionando cada subpasta ou usando um glob, como no exemplo abaixo:

```json
{
  "name": "example",
  "version": "1.33.7",
  "workspaces": [
    "packages/*"
  ]
}
```

Leia mais sobre os espaços de trabalho no [rfc](https://github.com/npm/rfcs/blob/latest/implemented/0026-workspaces.md) e nos [documentos](https://docs.npmjs.com/cli/v7/using-npm/workspaces) do [npm](https://docs.npmjs.com/cli/v7/using-npm/workspaces) .

### 3) Instalação automática de dependências de pares (_peer dependencies_)

Em versões anteriores ao npm 7, os desenvolvedores tinham que instalar as dependências de pares (_peer dependencies_). Agora o npm usará um novo algoritmo para garantir que as dependências de pares sejam instaladas corretamente. Se uma dependência de par, que não é compatível com a especificada, for instalada, o npm 7 irá bloquear a instalação.

## Mudanças e Quebras

Como a nova versão é considerada uma versão principal, ela virá com algumas alterações importantes. Aqui estão alguns:

-   Você não pode mais usar `require()` nos módulos internos do npm. npm agora usa o campo `package.exports`.
-   A equipe reescreveu totalmente o `npx` para usar internamente o `npm exec`, o `npx CLI` ainda estará disponível. Algumas mudanças de funcionalidade são esperadas. Uma é que agora você será solicitado se tentar executar um módulo que ainda não está instalado.
-   As mudanças mencionadas acima com relação às dependências dos pares podem atrapalhar alguns fluxos de trabalho.
-   `npm audit` tem uma nova saída.
-   O npm 6 mostrou todos os pacotes por padrão durante a execução do `npm ls`. Com o npm 7, ele mostrará apenas os pacotes de nível superior. Execute `npm ls --all` para imitar o comportamento do npm 6.

# Créditos

- [Npm 7 is now the standard, here is what you'll get](https://justfrontendthings.com/post/npm-7-now-standard), escrito originalmente por [Just Frontend Things](https://twitter.com/frontend_things).
