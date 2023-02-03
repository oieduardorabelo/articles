# Como é o fluxo de dados no Remix

Quando o React apareceu pela primeira vez, um de seus recursos mais atraentes era seu "fluxo de dados unidirecional". Isso ainda está descrito nos documentos do React na página ["Thinking in React"](https://reactjs.org/docs/thinking-in-react.html).

> O componente no topo da hierarquia receberá seus dados como props. Se você fizer uma alteração em seus dados e chamar `root.render()` novamente, a UI será atualizada. Você percebe como sua interface do usuário é atualizada e onde fazer alterações são feitas. O fluxo de dados unidirecional (também chamado de ligação unidirecional) do React mantém tudo modular e rápido.

A ideia é que os dados só possam fluir em um sentido através de seu aplicativo, o que, portanto, torna seu aplicativo muito mais fácil de intuir, entender e raciocinar.

![Ilustração da ideia de fluxo de dados unidirecional representando linhas desenhando um fluxo circular da exibição para a ação e para o estado.](https://remix.run/blog-images/posts/remix-data-flow/view-action-state.png)

Isso veio a ser resumido na frase: "UI é uma função do estado", ou `ui = fn(state)`. Sempre que algum estado muda, devido a uma ação, a interface é renderizada novamente. Até o momento, várias soluções sofisticadas de "gerenciamento de estado" foram criadas para facilitar a construção de aplicativos com esse modelo mental.

Um problema raramente reconhecido aqui, no entanto, é que esse "fluxo de dados unidirecional" é um pouco impróprio. É realmente um fluxo de dados unidirecional no cliente. Mas ter dados exclusivamente no cliente raramente é prático. Na maioria das vezes, você precisa manter os dados - para sincronizá-los - o que significa que você precisa que os dados fluam de duas maneiras: entre o cliente e o servidor.

![Ilustração do fluxo "Interface -> Ação -> Estado" enquadrado em um navegador à esquerda. À direita está uma ilustração de um servidor com um banco de dados. Duas setas conectam essas duas ilustrações denotando transferência de rede.](https://remix.run/blog-images/posts/remix-data-flow/view-action-state-server-client.png)

Muitas ferramentas de gerenciamento de estado ajudam você a gerenciar o estado apenas no cliente, mas não o ajudam a cruzar efetivamente [o abismo da rede](https://kentcdodds.com/blog/remix-the-yang-to-react-s-yin): a lacuna entre o estado no cliente e o estado no servidor.

![Ilustração do fluxo “Interface -> Ação -> Estado” enquadrado em um navegador à esquerda. À direita está uma ilustração de um servidor com um banco de dados. Duas setas conectam essas duas ilustrações denotando transferência de rede. A ênfase visual está na parte de rede do gráfico com “?”s ao seu redor.](https://remix.run/blog-images/posts/remix-data-flow/view-action-state-network.png)

[Entra o Remix](https://remix.run/docs/en/v1/guides/data-loading): "Um dos principais recursos do Remix é simplificar as interações com o servidor para obter dados em componentes." O Remix estende o fluxo de dados pela rede, tornando-o verdadeiramente unidirecional e cíclico: do servidor (estado), para o cliente (interface) e de volta ao servidor (ação).

![Ilustração do fluxo “Interface -> Action -> State” atravessando o navegador e o servidor.](https://remix.run/blog-images/posts/remix-data-flow/view-action-state-server-client-network.png)

Quando é dizemos que sua “UI é uma função do estado”, uma maneira de desembaraçar as suposições dessa afirmação seria: A UI é uma função do seu estado remoto e do seu estado local. Em um aplicativo React tradicional, todos os estados residem no cliente e as partes que você deseja persistir devem pular para fora do "fluxo de dados unidirecional" e ser sincronizadas na rede para um servidor. Como você pode imaginar, esta é uma área propensa a bugs.

No Remix, no entanto, a ideia de "UI como uma função do estado" é transformada porque o estado remoto pode ser mais facilmente separado do estado local. "Qual é a diferença", você pergunta? Pense desta maneira.

O "estado remoto" é qualquer dado que precisa persistir, como dados do usuário. Esse estado (por exemplo, quantas notificações não lidas o usuário tem?) é armazenado no cliente e reconciliado em seu aplicativo a partir de mecanismos Remix, como [`loaders`](https://remix.run/docs/en/v1/guides/data-loading) e [actions](https://remix.run/docs/en/v1/guides/data-writes).

(Observação: o Remix ajuda você a cruzar o abismo da rede, fornecendo informações sobre o estado da transmissão de seus dados usando [`transitions`](https://remix.run/docs/en/v1.5.1/api/remix#usetransition) e [`fetchers`](https://remix.run/docs/en/v1/api/remix#usefetcher) - não há necessidade de rastrear o status de cada solicitação de rede por meio de booleanos como `isLoading` ou enums como `initial | loading | success | failed`).

Por outro lado, o estado local são dados efêmeros que podem ser perdidos (por exemplo, por meio de uma atualização) sem afetar negativamente a experiência do usuário. Esse estado (por exemplo, o menu suspenso aberto que revela as notificações do usuário?) é armazenado no cliente por meio de mecanismos como `useState` em React ou _local storage_ nos navegadores. É importante ressaltar que ele não precisa persistir e sincronizar pela rede (para o servidor), reduzindo assim a complexidade e o potencial de bugs.

![Ilustração representando um fluxo unidirecional de “dados remotos” entre o cliente/servidor facilitado pelo Remix. Um fluxo unidirecional de “dados locais” está no cliente facilitado exclusivamente pelo React e/ou localStorage.](https://remix.run/blog-images/posts/remix-data-flow/view-action-state-local-vs-remote.png)

Formulários, _fetchers_, _loaders_, _actions_, todos são soluções de “gerenciamento de estado” no Remix (embora não os chamemos assim). Eles fornecem as ferramentas para manter o estado persistente em sincronia entre o cliente e o servidor, garantindo que os dados fluam ciclicamente em um sentido único por meio de seu aplicativo e pela rede: de _loaders_ a um componente para uma _action_ e vice-versa.

![As palavras “Loader” -> “Action” -> “Component” mostradas em um diagrama circular.](https://remix.run/blog-images/posts/remix-data-flow/loader-action-component.png)

Com o Remix, sua UI se torna uma função de estado em toda a rede, não apenas localmente. [Uma analogia interessante](https://discord.com/channels/770287896669978684/770287896669978687/980184501726642186) com as abstrações de dados que o Remix fornece, é a abstração do DOM Virtual do React.

No React, você não se preocupa em atualizar o DOM sozinho. Você define o estado e o DOM Virtual faz toda a diferença para descobrir como fazer atualizações eficientes no DOM. O Remix estende essa idéia para a camada de API para persistir dados.

No Remix, você não se preocupa em manter o estado do lado do cliente sincronizado com o servidor. Você "define o estado" com uma mutação e os _loaders_ assumem o controle para buscar novamente os dados mais atualizados e fazer atualizações em seus componentes de interface.

![Captura de tela do exemplo de código no Remix ilustrando o fluxo cíclico unidirecional de dados por meio de um aplicativo. Há uma função loader cujo código flui para o componente de rota cujo código, por meio de um <Form>, flui para a função action, cujo código flui de volta para uma função loader novamente.](https://remix.run/blog-images/posts/remix-data-flow/loader-action-component-code.png)

Espero que isso ajude a ilustrar como o Remix ajuda a reduzir drasticamente a quantidade de complexidade necessária para criar sites melhores. Como Kent [disse em sua palestra no RenderATL](https://youtu.be/zED9ePuht4g?t=24852), o Remix funciona antes do JavaScript, isso é uma vitória para seus usuários porque eles obtêm uma experiência de aprimoramento progressivo. Mas também é uma vitória para você como desenvolvedor, porque não precisa criar toda a complexidade tradicionalmente associada às soluções de gerenciamento de estado.

> Você não precisa se preocupar com o gerenciamento de estados ao usar o Remix. Redux, Apollo, por mais legais que essas ferramentas sejam, você não precisa delas quando estiver usando o Remix porque nem precisamos de JavaScript do lado do cliente para que tudo funcione... pense no aplicativo que você está criando... e vamos fingir que você pode jogar fora todo o código que tem a ver com o gerenciamento de estado do aplicativo... é assim que funciona quando você trabalha com o Remix. Se funcionar sem JavaScript no navegador, isso significa que você não precisa de nada no navegador que exija gerenciamento de estado.

---

# Crédios

- Escrito originalmente por [Jim Nielsen](https://twitter.com/jimniels), em [Data Flow in Remix](https://remix.run/blog/remix-data-flow).
