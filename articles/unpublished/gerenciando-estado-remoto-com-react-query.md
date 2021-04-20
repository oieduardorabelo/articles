# Gerenciando estado remoto com React Query

React é uma das bibliotecas de front-end mais apreciadas pela comunidade de desenvolvedores. Junto com o React, os termos como DOM Virtual, Componentes Funcionais, Gestão de Estado e Componentes de Ordem-Superior (_Higher-Order Components_). Entre esses termos, a Gestão de Estado desempenha um papel vital.

O gerenciamento de estado é um dos principais fatores que precisam ser considerados antes de iniciar um projeto React. Os desenvolvedores usam padrões e bibliotecas famosas como Flux, Redux e Mobx para gerenciar o estado em React. No entanto, eles adicionam complexidade e código boilerplate ao seus aplicativos.

Neste artigo, vamos discutir como o React Query aborda o problema mencionado acima, criando um pequeno aplicativo pokemon e mergulhando em seus conceitos-chave.

---

Dica: Compartilhe seus componentes reutilizáveis entre projetos usando [Bit](https://bit.dev/) (veja no [GitHub](https://github.com/teambit/bit)). O Bit simplifica o compartilhamento, a documentação e a organização de componentes independentes de qualquer projeto.

Podemos usá-lo para maximizar a reutilização de código, colaboração em componentes independentes e criar aplicativos escaláveis.

O [Bit](https://bit.dev/) suporta Node, TypeScript, React, Vue, Angular e muito mais.

![Exemplo: Explorando componentes reutilizáveis em React ​​compartilhados no Bit.dev](https://miro.medium.com/max/700/0*NWClWl8qitQnAgRu.gif)

---

## O que é React Query?

React Query é uma das ferramentas de gerenciamento de estado que tem uma abordagem diferente do Flux, Redux e Mobx. Ele apresenta os principais conceitos de Estado-do-Cliente e Estado-do-Servidor. Isso torna o React Query uma das melhores bibliotecas para gerenciar estado, já que todos os outros padrões de gerenciamento de estado tratam apenas do estado do cliente e acham difícil lidar com o estado do servidor que precisa ser buscado, ouvido ou inscrito.

Além de lidar com o estado do servidor, ele funciona incrivelmente bem, sem precisar de configurações customizadas, e pode ser personalizado de acordo com o seu gosto conforme o crescimento do seu aplicativo.

Vamos ver isso na prática usando alguns exemplos.

## Instalando o React Query

Em primeiro lugar, vamos instalar a React QUery dentro de um projeto React:

```bash
npm install react-query react-query-devtools axios --save
```

Ou:

```bash
yarn add react-query react-query-devtools axios
```

## Configurando Ferramentas de Desenvolvimento

O React Query também tem suas próprias ferramentas de desenvolvimento, que nos ajudam a visualizar o funcionamento interno do React Query. Vamos configurar as ferramentas de desenvolvimento do React Query no arquivo App.js:

```jsx
import { ReactQueryDevtools } from "react-query-devtools";
function App() {
  return (
    <>
      {/* Os outros componentes da nossa aplicação */}
      <ReactQueryDevtools initialIsOpen={false} />
    </>
  );
}
```

Quando configuramos as ferramentas de desenvolvimento do React Query, você pode ver o logotipo do React Query na parte inferior esquerda do seu aplicativo, assim:

![Ícone React Query Devtools na parte inferior esquerda](https://miro.medium.com/max/193/1*MbGgcUqm7DAMtkurWZhDRg.png)

![React Query Devtools](https://miro.medium.com/max/700/1*_VUwsO_bC-OfPpNV9Svwiw.png)

O devtools nos ajuda a ver como o fluxo de dados acontece dentro do aplicativo, assim como Redux Devtools. Isso realmente ajuda a reduzir o tempo de depuração do aplicativo.

Assim como o GraphQL, o React Query também se baseia em conceitos básicos semelhantes, como

- Query
- Mutações
- Invalidação de Query

## Buscando Pokémons usando Query

Neste exemplo, vamos usar a [PokéApi](https://pokeapi.co/). Começaremos com **useQuery**, que recebe uma chave única e uma função responsável por buscar dados:

```jsx
import React from "react";
import axios from "axios";
import { useQuery } from "react-query";
import Card from "./Card";
const fetchPokemons = async () => {
 const { data } = await axios.get("https://pokeapi.co/api/v2/pokemon/?limit=50");
 return data;
};
function Main() {
const { data, status } = useQuery("pokemons", fetchPokemons);
const PokemonCard = (pokemons) => {
 return pokemons.results.map((pokemon) => {
  return <Card key={pokemon.name} name={pokemon.name}></Card>;
 });
};
return (
  <div>
  {status === "loading" && <div>Loading...</div>}
  {status === "error" && <div>Error fetching pokemons</div>}
  {status === "success" && <div>{PokemonCard(data)}</div>}
 </div>
);
}
export default Main;
```

O código acima irá renderizar uma UI como abaixo:

![Dados renderizados ao buscar pokemons da pokeapi por meio do react query](https://miro.medium.com/max/700/1*0GvFFLP03ShTN6QAUT42fQ.png)

## Cache no React Query

Como você pode ver, useQuery retorna os dados e o status que podem ser usados ​​para exibir componentes de "Carregando...", mensagens de erro e os dados reais. Por padrão, React Query só solicitará dados quando eles estiverem desatualizados ou antigos.

O React Query armazena os dados em cache para não renderizar os componentes, a menos que haja uma alteração. Também podemos usar alguma configuração especial com useQuery para atualizar os dados em segundo plano.

```
const {data, status} = useQuery ("pokemons", fetchPokemons, {staleTime: 5000, cacheTime: 10});
```

A configuração acima fará com que o React Query busque dados a cada 5 segundos em segundo plano. Também podemos definir um `cacheTime` e um `retryTime` que define o tempo que o navegador deve manter o cache e o número de tentativas em que ele deve buscar dados.

## Redefinindo o Cache com Invalidação de Query

O React Query buscará dados assim que os dados / cache estiverem desatualizados. Isso acontece quando o `staleTime` padrão é passado. Você também pode invalidar o cache de maneira programática para que o React Query atualize os dados.

Para fazer isso, use `queryCache`. É uma instância utilitária que contém muitas funções que podem ser usadas para manipular ainda mais as Query e invalidar o cache.

```js
queryCache.invalidateQueries("pokemons");
```

## Variáveis no React Query

Também podemos passar variáveis ​​para a query. Para isso, precisamos transmiti-los como um array.

```js
const { data, status } = useQuery(["pokemons",75], fetchPokemons);
```

O primeiro elemento será a chave e o resto dos elementos são variáveis. Para usar a variável, vamos fazer algumas modificações em nossa função `fetchPokemons`.

```js
const fetchPokemons = async (key,limit) => {
 const { data } = await axios.get(`https://pokeapi.co/api/v2/pokemon/?limit=${limit}`);
 return data;
};
```

## Brincando com Mutações

As mutações são normalmente usadas para criar / atualizar / excluir dados ou executar efeitos colaterais do lado do servidor. O React Query fornece o hook `useMutation` para realizar mutações. Vamos criar uma mutação para criar um pokémon:

```jsx
import React from "react";
import { useQuery } from "react-query";

function Pokemon() {
  const [name, setName] = useState("");
  const [mutateCreate, { error, reset }] = useMutation(
    (text) => axios.post("/api/data", { text }),
    {
      onSuccess: () => {
        setName("");
      },
    }
  );
  return (
    <div>
      <form
        onSubmit={(e) => {
          e.preventDefault();
          mutateCreate(name);
        }}
      >
        {error && <h5 onClick={() => reset()}>{error}</h5>}
        <input
          type="text"
          value={name}
          onChange={(e) => setName(e.target.value)}
        />
        <br />
        <button type="submit">Create Pokemon</button>
      </form>
    </div>
  );
}

export default Pokemon;
```

Neste exemplo, quando adicionamos um novo nome de Pokémon e clicamos no botão Criar Pokémon , ele fará a mutação e irá buscar os dados. Se a mutação falhar, o erro será exibido.

O erro e o estado dos dados podem ser eliminados usando a função `reset`, que reinicializará a mutação. A função `onSuccess` pode ser usada para limpar o estado da entrada ou do nome.

Uma mutação tem mais propriedades como `onSuccess`, `isIdle` , `isLoading` , `isError` , `isSuccess`. Eles podem ser usados ​​para lidar com erros e exibir informações relevantes para diferentes estados da mutação.

## Conclusão

O React Query é uma das melhores maneiras de buscar, armazenar em cache e atualizar dados remotos. Precisamos apenas dizer à biblioteca onde você precisa buscar os dados, e ela tratará do cache, das atualizações em segundo plano e da atualização dos dados sem nenhum código ou configuração extra.

Ele também fornece alguns hooks e eventos para mutação e query para lidar com erros e outros estados dos efeitos colaterais, o que remove a necessidade de usar hooks como `useState` e `useEffect` e os substitui por algumas linhas com React Query.

# Créditos

- [React Query — An Underrated State Management Tool](https://blog.bitsrc.io/react-query-an-underrated-state-management-tool-5618b7b8cb36), escrito originalmente por [Tharaka Romesh](https://medium.com/@TRomesh).
