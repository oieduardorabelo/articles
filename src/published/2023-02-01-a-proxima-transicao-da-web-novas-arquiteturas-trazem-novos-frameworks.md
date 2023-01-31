# A próxima transição da Web: Novas arquiteturas trazem novos frameworks

A web é composta por tecnologias que começaram há mais de 25 anos. HTTP, HTML, CSS e JS foram padronizados pela primeira vez em meados dos anos 90 (quando eu tinha 8 anos). Desde então, a web evoluiu para uma plataforma de aplicativos onipresente. À medida que a web evoluiu, também evoluiu a arquitetura para o desenvolvimento desses aplicativos. Atualmente, existem muitas arquiteturas principais para criar aplicativos para a Web. A arquitetura mais popular empregada por desenvolvedores da Web hoje é o Single Page App (SPA), mas estamos fazendo a transição para uma arquitetura nova e aprimorada para criar aplicativos da Web.

Os elementos <a> e <form> estão por aí desde o início. Links para um navegador obter coisas de um servidor e formulários para um navegador enviar coisas para um servidor (e obter coisas em troca). Com essa comunicação bidirecional estabelecida como parte da especificação desde o início, foi possível criar aplicações poderosas na web desde sempre.

Aqui estão as principais arquiteturas (em ordem cronológica de uso popular):

1. Aplicativos de várias páginas (MPAs)
2. Aplicativos de várias páginas progressivamente aprimorados (PEMPAs, também conhecidos como "JavaScript Sprinkles")
3. Aplicativos de página única (SPAs)
4. A próxima transição

Cada arquitetura de desenvolvimento web tem benefícios e pontos problemáticos. Eventualmente, os pontos problemáticos se tornaram um problema suficiente para motivar a mudança para a próxima arquitetura que veio com suas próprias compensações.

Não importa como construímos nossos aplicativos, quase sempre precisamos de código em execução em um servidor (exceções notáveis ​​incluem jogos como o Wordle, que _costumava_ armazenar o estado do jogo no _local storage_). Uma das coisas que distingue essas arquiteturas é onde o código reside. Vamos explorar cada um deles e observar como a localização do código mudou ao longo do tempo. Conforme abordamos cada arquitetura, consideraremos especificamente os seguintes casos de uso de código:

- Persistência: Salvar e ler dados de um banco de dados
- Roteamento: Direcionando o tráfego com base na URL
- Busca de dados: Recuperando dados da persistência de dados
- Mutação de dados: Alteração de dados na persistência de dados
- Lógica de renderização: Exibindo dados para o usuário
- Feedback da interface do usuário: Respondendo à interação do usuário

Há, naturalmente, mais partes de um aplicativo da web do que esses pedaços, mas esses são os pedaços que mais se movimentam e onde passamos a maior parte do nosso tempo como desenvolvedores da web. Dependendo da escala do projeto e da estrutura da equipe, podemos trabalhar em todas essas categorias de código ou em apenas uma parte.

# Aplicativos de Várias Páginas (MPAs)

No início, essa era a única arquitetura que funcionava na web com base nos recursos dos navegadores da época.

![Multi-Page App (MPA)](https://res.cloudinary.com/epic-web/image/upload/v1665491189/epicweb.dev/blog/The%20Web%E2%80%99s%20Next%20Transition/mpa.svg)

Com aplicativos de várias páginas, todo o código que escrevemos está no servidor. O código de feedback da interface do usuário no cliente é manipulado pelo navegador do usuário.

# Comportamentos de Arquitetura de MPAs

![Fluxo de requisição de documentos em MPAs](https://res.cloudinary.com/epic-web/image/upload/v1665491190/epicweb.dev/blog/The%20Web%E2%80%99s%20Next%20Transition/mpa-document-request.svg)

**Solicitação de documentos:** Quando o usuário insere uma URL na barra de endereços, o navegador envia uma solicitação ao nosso servidor. Nossa lógica de roteamento chamará uma função para buscar dados que se comunica com o código de persistência para recuperar os dados. Esses dados são usados ​​pela lógica de renderização para determinar o HTML que será enviado como resposta ao cliente. Enquanto isso, o navegador está dando feedback ao usuário com algum tipo de estado pendente (normalmente na posição do favicon).

![Mutação de redirecionamento em MPAs](https://res.cloudinary.com/epic-web/image/upload/v1665491191/epicweb.dev/blog/The%20Web%E2%80%99s%20Next%20Transition/mpa-redirect-mutation-request.svg)

**Solicitação de mutações:** Quando o usuário envia um formulário, o navegador serializa o formulário em uma solicitação enviada ao nosso servidor. Nossa lógica de roteamento chamará uma função que se comunicam com o código de persistência para alterar os dados e fazer atualizações no banco de dados. Em seguida, ele responderá com um redirecionamento para que o navegador acione uma solicitação GET para obter uma nova interface do usuário (que acionará a mesma coisa que aconteceu quando o usuário inseriu a URL para começar). Novamente, o navegador fornecerá feedback ao usuário com interface do usuário pendente.

Observação: é importante que as mutações bem-sucedidas enviem uma resposta de redirecionamento ao invés de apenas o novo HTML. Caso contrário, você terá a solicitação POST em sua pilha de histórico no navegador e pressionar o botão "Voltar" acionará a solicitação POST novamente (você já se perguntou por que os aplicativos às vezes dizem "NÃO APERTE O BOTÃO VOLTAR!!" Sim, é por isso. Eles deveriam ter respondido com um redirecionamento).

# Prós e Contras de MPAs

O modelo mental dos MPAs é simples. Nós não apreciamos isso naquela época. Embora houvesse algum estado e fluxos complicados manipulados principalmente por cookies nas solicitações, na maioria das vezes tudo acontecia dentro do tempo de um ciclo de solicitação/resposta.

Onde esta arquitetura falha:

1. Atualizações de página inteira: Tornando algumas coisas difíceis (gerenciamento de foco), outras impraticáveis ​​(imagine uma atualização de página inteira toda vez que curtimos um tweet) e algumas coisas simplesmente impossíveis (transições de página animadas).
2. Controle de feedback da interface do usuário: É bom que o favicon se transforme em um controle giratório, mas geralmente um UX melhor é o feedback visualmente mais próximo da interface do usuário com a qual o usuário interagiu. E certamente é algo que os designers gostam de personalizar para fins de branding. E a interface do usuário otimista?

É notável que a plataforma web esteja melhorando constantemente com a [proposta de API de transições de página](https://developer.chrome.com/blog/shared-element-transitions-for-spas/), que torna os MPAs uma opção mais viável para mais casos de uso. Mas para a maioria dos aplicativos da web, isso ainda não é suficiente. De qualquer forma, na época esse problema estava longe das mentes dos comitês de padrões e agora, nossos usuários querem mais!

# Aplicativos de várias páginas progressivamente aprimorados (PEMPAs)

_Progressive Enhancement_ é a ideia de que nossos aplicativos da Web devem ser funcionais e acessíveis a todos os navegadores da Web e, em seguida, aproveitar quaisquer recursos extras que o navegador tenha para aprimorar a experiência. O termo foi [cunhado em 2003 por Nick Finck e Steve Champeon](https://www.hesketh.com/publications/progressive_enhancement_and_the_future_of_web_design.html), falando dos recursos do navegador.

`XMLHttpRequest` foi inicialmente desenvolvido pela equipe do Outlook Web Access da Microsoft em 1998, mas não foi padronizado até 2016 (você acredita nisso!?). É claro que isso nunca impediu os fornecedores de navegadores e desenvolvedores da web antes. AJAX foi popularizado como um termo em 2005 e muitas pessoas começaram a fazer solicitações HTTP no navegador. As empresas foram construídas com base na ideia de que não precisamos voltar ao servidor para obter mais do que pequenos pedaços de dados para atualizar a interface do usuário na hora. Com isso, poderíamos criar aplicativos de várias páginas progressivamente aprimorados:

![Progressively Enhanced Multi-Page App (PEMPA) - Aplicativos de várias páginas progressivamente aprimorados](https://res.cloudinary.com/epic-web/image/upload/v1665491189/epicweb.dev/blog/The%20Web%E2%80%99s%20Next%20Transition/pempa.svg)

"Uau!", você pode estar pensando, "espere um minuto... de onde veio todo esse código?". Portanto, agora não apenas assumimos a responsabilidade pelo feedback da interface do usuário no navegador, mas também temos roteamento, busca de dados, mutação de dados e lógica de renderização no lado do cliente , além do que já tínhamos no servidor. "E daí?"

Bem, aqui está o negócio. A idéia por trás do aprimoramento progressivo é que nossa linha de base deve ser um aplicativo funcional. Especialmente no início dos anos 2000, não podíamos garantir que nosso usuário usaria um navegador capaz de executar nosso novo e sofisticado material AJAX ou que estaria em uma rede rápida o suficiente para baixar nosso JavaScript antes de interagir com nosso aplicativo. Portanto, precisávamos manter a arquitetura MPA existente e usar apenas JavaScript para aprimorar a experiência.

Dito isso, dependendo do nível de aprimoramento de que estamos falando, podemos realmente ter que escrever código em quase todas as nossas categorias, a exceção sendo a camada de persistência de dados (a menos que desejemos suporte ao modo offline, o que é realmente legal, mas não é um padrão da indústria na prática, por isso não está incluído no gráfico).

Além disso, tivemos que adicionar mais código ao back-end para dar suporte às solicitações AJAX que nosso cliente faria. Portanto, mais em ambos os lados da rede.

Esta é a era do jQuery, MooTools, etc.

# Comportamentos de Arquitetura de PEMPAs

![Fluxo de requisição de documentos em PEMPAs](https://res.cloudinary.com/epic-web/image/upload/v1665498312/epicweb.dev/blog/The%20Web%E2%80%99s%20Next%20Transition/pempa-document-request.svg)

**Solicitação de documentos:** Quando o usuário solicita o documento pela primeira vez, ocorre aqui a mesma coisa que no exemplo do MPA. No entanto, um PEMPA também carregará o JavaScript no lado do cliente incluindo tags `<script>` que serão usadas para os recursos de aprimoramento.

![Navegação no lado do cliente em PEMPAs](https://res.cloudinary.com/epic-web/image/upload/v1665491194/epicweb.dev/blog/The%20Web%E2%80%99s%20Next%20Transition/pempa-clientside-navigation.svg)

**Navegação no lado do cliente:** Quando o usuário clica em um elemento âncora com um `href` que está em nosso aplicativo, nosso código de busca de dados do lado do cliente evita o comportamento padrão de atualização de página inteira e usa JavaScript para atualizar a URL. Em seguida, a lógica de roteamento do cliente determina quais atualizações precisam acontecer na interface do usuário e executa manualmente essas atualizações, incluindo a exibição de quaisquer estados pendentes (feedback na interface do usuário) enquanto a biblioteca de busca de dados faz uma solicitação de rede para um endereço do servidor. A lógica de roteamento do servidor chama o código de busca de dados para recuperar dados do na camada de persistência de dados e envia isso como uma resposta (como XML ou JSON, podemos escolher 😂) que o cliente usa para executar as atualizações finais da interface do usuário com sua lógica de renderização.

![Mutações diretas por requisição em PEMPAs](https://res.cloudinary.com/epic-web/image/upload/v1665992270/epicweb.dev/blog/The%20Web%E2%80%99s%20Next%20Transition/pempa-inline-mutation-request.svg)

![Requisição de redirecionamento após requisições de mutação em PEMPAs](https://res.cloudinary.com/epic-web/image/upload/v1665491189/epicweb.dev/blog/The%20Web%E2%80%99s%20Next%20Transition/pempa-redirect-mutation-request.svg)

**Solicitação de mutações:** Quando o usuário envia um formulário, nossa lógica de mutação de dados do lado do cliente impede a atualização padrão de página inteira e o comportamento de `POST` e usa JavaScript para serializar o formulário e enviar os dados para um endereço do servidor. A lógica de roteamento do servidor então chama a função de mutação de dados, que interage com o código de persistência de dados para realizar a mutação e responde com os dados atualizados ao cliente. A lógica de renderização do cliente usará esses dados atualizados para atualizar a interface do usuário, conforme necessário; em alguns casos, a lógica de roteamento do lado do cliente enviará o usuário para outro local que aciona um fluxo semelhante ao fluxo de navegação do lado do cliente.

# Prós e Contras de PEMPAs

Definitivamente, resolvemos os problemas com os MPAs trazendo o código para o lado do cliente e assumindo a responsabilidade do feedback da interface do usuário. Temos muito mais controle e podemos dar aos usuários uma sensação de aplicativo mais personalizado.

Infelizmente, para dar aos usuários a melhor experiência que eles procuram, temos que ser responsáveis ​​pelo roteamento, busca de dados, mutações e lógica de renderização. Existem alguns problemas com isso:

1. Comportamento padrão: Não fazemos um trabalho tão bom quanto o navegador com roteamento e envio de formulários. Manter os dados da página atualizados nunca foi uma preocupação antes, mas agora é mais da metade do nosso código no lado do cliente. Além disso, _race conditions_, reenvios de formulários e tratamento de erros são ótimos lugares para os bugs se esconderem.

2. Código personalizado: Há muito mais código para gerenciar que não precisávamos escrever antes. Sei que correlação não implica causalidade, mas notei que, em geral, quanto mais código temos, mais bugs temos 🤷‍♂️

3. Duplicação de código: Há muita duplicação de código em relação à lógica de renderização. O código do cliente precisa atualizar a interface do usuário da mesma forma que o código de back-end renderizaria todos os estados possíveis após uma mutação ou transição do cliente. Portanto, a mesma UI do back-end também deve estar disponível no front-end. E, na maioria das vezes, eles estão em idiomas completamente diferentes, o que torna o compartilhamento de código algo inviável. E não são apenas os modelos, mas também a lógica. O desafio é: “faça uma interação do lado do cliente e verifique se a interface do usuário atualizada pelo código do cliente é a mesma que teria acontecido se tivesse sido uma atualização de página inteira”. Isso é surpreendentemente difícil de fazer (há um site que nós, desenvolvedores, usamos regularmente que é um PEMPA e com muita frequência não faz isso corretamente).

4. Organização do código: Com PEMPAs isso é muito difícil. Sem um local centralizado para armazenar dados ou renderizar a interface do usuário, as pessoas atualizavam manualmente o DOM em praticamente qualquer lugar e era muito difícil seguir o código, o que atrasava o desenvolvimento.

5. Falta de sincronia entre Servidor/Cliente: Há indireção entre as rotas da API e a busca de dados do lado do cliente e o código de mutação de dados que as utiliza. Uma alteração em um lado da rede exige uma alteração no outro lado, e essa falta de sincronia tornou difícil saber se não quebramos nada porque seguir caminhos de código envolvia percorrer uma série de arquivos. A rede tornou-se uma barreira que causou essa indireção da mesma forma que uma raposa usa um rio para se livrar de seu rastro para os caçadores.

Em uma nota pessoal, foi nessa época que entrei no mundo do desenvolvimento web. Lembro dessa época com um misto de saudade saudade e arrepio de medo 🍝.

# Aplicativos de página única (SPAs)

Não demorou muito para percebermos que poderíamos remover os problemas de duplicação se apenas excluíssemos o código da interface do usuário do back-end. Então foi isso que fizemos:

![Single Page App (SPA)](https://res.cloudinary.com/epic-web/image/upload/v1665491194/epicweb.dev/blog/The%20Web%E2%80%99s%20Next%20Transition/spa.svg)

Você notará que este gráfico é quase idêntico ao PEMPA. A única diferença é que a lógica de renderização desapareceu. Parte do código de roteamento também desapareceu porque não precisamos mais ter rotas para interface do usuário. Tudo o que nos resta são as rotas da API. Esta é a era do Backbone, Knockout, Angular, Ember, React, Vue, Svelte, etc. Esta é a estratégia usada pela maior parte da indústria hoje.

# Comportamentos de Arquitetura de SPAs

![Requisição de documentos em SPAs.](https://res.cloudinary.com/epic-web/image/upload/v1665495746/epicweb.dev/blog/The%20Web%E2%80%99s%20Next%20Transition/spa-document-request.svg)

Como o back-end não possui mais lógica de renderização, todas as solicitações de documentos (a primeira solicitação que um usuário faz ao inserir nossa URL) são atendidas por um servidor de arquivos estático (normalmente um CDN). Nos primórdios dos SPAs, esse documento HTML era quase sempre um arquivo HTML efetivamente vazio com um `<div id="root"></div>` no `<body>` que seria usado para “montar” o aplicativo. Hoje em dia, no entanto, os frameworks nos permitem pré-renderizar qualquer quantidade da página que tivermos no tempo de _build_ ou _deploy_ usando uma técnica conhecida como “Static Site Generation” (SSG).

![Navegação no lado do cliente em SPAs](https://res.cloudinary.com/epic-web/image/upload/v1665491193/epicweb.dev/blog/The%20Web%E2%80%99s%20Next%20Transition/spa-clientside-navigation.svg)

![Requisições de mutações em SPAs](https://res.cloudinary.com/epic-web/image/upload/v1665491192/epicweb.dev/blog/The%20Web%E2%80%99s%20Next%20Transition/spa-inline-mutation-request.svg)

![Requisições de redirecionamento após executar requisições de mutação em SPAs](https://res.cloudinary.com/epic-web/image/upload/v1665491193/epicweb.dev/blog/The%20Web%E2%80%99s%20Next%20Transition/spa-redirect-mutation-request.svg)

Os outros comportamentos nesta estratégia são os mesmos dos PEMPAs. Só que agora usamos principalmente `fetch` ao invés de `XMLHttpRequest`.

# Prós e Contras de SPAs

O que é interessante é que a única diferença entre SPAs e PEMPAs relacionados a suas arquiteturas é que a solicitação de documento é pior! Então, por que fizemos isso!?

De longe, o maior ganho aqui é a experiência do desenvolvedor. Essa foi a força original para a transição de PEMPAs para SPAs em primeiro lugar. Não ter duplicação de código foi um benefício enorme. Justificamos essa mudança por vários meios (afinal, o DX é uma entrada para o UX). Infelizmente, melhorar o DX é tudo o que os SPAs realmente fizeram por nós.

Lembro-me de estar pessoalmente convencido de que a arquitetura SPA ajudava no desempenho percebido porque um CDN poderia responder com um documento HTML mais rápido do que um servidor poderia gerar um, mas em cenários do mundo real isso nunca pareceu fazer diferença (e isso é ainda menos verdadeiro graças a infraestrutura moderna). A triste realidade é que os SPAs ainda têm os mesmos outros problemas que os PEMPAs, embora com ferramentas mais modernas tornam as coisas muito mais fáceis de lidar.

Para piorar a situação, os SPAs também introduziram vários novos problemas:

1. Tamanho dos pacotes/arquivos finais: Meio que explodiu. Leia mais sobre o [impacto do JavaScript no desempenho de uma página da Web](https://almanac.httparchive.org/en/2022/javascript) neste artigo completo no Web Almanac.

2. Chamadas de rede em efeito cachoeira: Como todo o código para buscar dados agora reside no JavaScript, temos que esperar que ele seja baixado e processado no cliente antes de podermos buscar os dados. Além disso, há a necessidade de implementar a divisão de código e o carregamento tardio (_lazy loading_) desses pacotes, e agora temos uma situação de dependência crítica como: `document → app.js → page.js → component.js → data.json → image.png`. Isso não é ótimo e, em última análise, resulta em uma experiência de usuário muito pior. Para conteúdo estático, podemos evitar muito disso, mas há uma série de problemas e limitações com os quais os fornecedores de estratégias SSG estão trabalhando e estão felizes em nos vender suas soluções específicas em cada plataforma.

3. Desempenho e performace em tempo de execução: Com tanto JavaScript do lado do cliente para executar, alguns dispositivos de menor potência lutam para manter uma boa performace (Leia: [The Cost of JavaScript](https://v8.dev/blog/cost-of-javascript-2019)). O que costumava ser executado em nossos robustos servidores agora deve ser executado no minicomputador das pessoas em suas mãos. Eu sei que mandamos pessoas para a lua com menos energia, mas isso ainda é um problema.

4. Gestão do estado: Isso se tornou um grande problema. Como prova disso, ofereço o número de bibliotecas disponíveis para resolver esse problema 😩 . Antes, o MPA renderizava nosso estado no DOM e apenas fazíamos referência/mutação disso. Agora estamos apenas obtendo o JSON e temos que não apenas informar o back-end quando os dados foram atualizados, mas também manter a representação em memória no lado do cliente desse estado atualizado. Isso tem todas as marcas dos desafios do cache (porque é isso que é), que é um dos problemas mais difíceis do software. Em um SPA típico, o gerenciamento de estado representa 30-50% do código em que as pessoas trabalham (essa estatística precisa de uma citação, mas você sabe que é verdade).

As bibliotecas foram criadas para ajudar a resolver esses problemas e reduzir seu impacto. Isso tem sido incrivelmente útil, mas alguns diriam que a [rotatividade é cansativa](https://medium.com/@ericclemmons/javascript-fatigue-48d4011b6fc4). Isso se tornou a maneira padrão de fato de criar aplicativos da web desde meados da década de 2010. Estamos bem na década de 2020 e há algumas novas ideias no horizonte.

# Aplicativos de página única progressivamente aprimorados (PESPAs)

MPAs têm um modelo mental simples. SPAs têm recursos mais poderosos. Pessoas que passaram pelo estágio MPA e estão trabalhando em SPAs realmente lamentam a simplicidade que perdemos na última década. Isso é particularmente interessante se você considerar o fato de que a motivação por trás da arquitetura SPA foi principalmente melhorar a experiência do desenvolvedor em relação aos PEMPAs. Se pudéssemos, de alguma forma, mesclar SPAs e MPAs em uma única arquitetura para obter o melhor de ambos, podemos ter algo que seja simples e mais capaz. Isso é o que os aplicativos de página única aprimorados progressivamente são.

Considere que, com o _Progressive Enhancement_, a linha de base é um aplicativo funcional, mesmo sem JavaScript do lado do cliente. Portanto, se nossa estrutura permite e incentiva o _Aprimoramento Progressivo_ como um **princípio fundamental**, a base de nosso aplicativo vem com a base sólida do modelo mental simples de MPAs. Especificamente, o modelo mental de pensar nas coisas no contexto de um ciclo de solicitação/resposta. Isso nos permite eliminar em grande parte os problemas dos SPAs.

Isso é importante: o principal benefício do Progressive Enhancement não é que "seu aplicativo funciona sem JavaScript" (embora esse seja um bom benefício colateral), mas sim que o modelo mental é drasticamente mais simples.

Continue lendo...

Para fazer isso, os PESPAs precisam “emular o navegador” quando impedem o comportamento padrão. Portanto, o código do servidor funciona da mesma maneira, independentemente de o navegador estar fazendo a solicitação ou seja uma solicitação de busca baseada em JavaScript. Embora esse código ainda exista, podemos manter o modelo mental simples no restante de nosso código. Uma parte importante disso é que os PESPAs emulam o comportamento do navegador de revalidar dados na página quando são feitas mutações para manter os dados na página atualizados. Com os MPAs, temos que recarregar a página inteira. Com PESPAs, essa revalidação acontece em cada solicitações de busca.

Lembre-se de que também tivemos um problema significativo com os PEMPAs: duplicação de código. Os PESPAs resolvem esse problema tornando o código de renderização da interface do back-end e o código de renderização de interface do front-end extamento o mesmo. Ao usar uma biblioteca de interface do usuário capaz de fazer ambos: renderizar seus dados no servidor e criar a interativa/manipulação de atualizações no cliente, não temos problemas de duplicação de código.

![Progressively Enhanced Single Page App (PESPA) - Aplicativos de página única progressivamente aprimorados.](https://res.cloudinary.com/epic-web/image/upload/v1665493076/epicweb.dev/blog/The%20Web%E2%80%99s%20Next%20Transition/pespa.svg)

Você notará que há pequenas caixas para busca, mutação e renderização de dados. Esses pedaços são para aprimoramento. Por exemplo, estados pendentes, interface do usuário otimista, etc. realmente não têm lugar no servidor, portanto, teremos algum código que será executado apenas no cliente. E usando as bibliotecas de interface do usuário modernas, a colocação que obtemos o torna sustentável.

# Comportamentos de Arquitetura de PESPAs

![Solicitação de documentos em PESPAs](https://res.cloudinary.com/epic-web/image/upload/v1665491191/epicweb.dev/blog/The%20Web%E2%80%99s%20Next%20Transition/pespa-document-request.svg)

**Solicitação de documentos**: Com PESPAs são efetivamente idênticas às PEMPAs. O HTML inicial necessário para o aplicativo é enviado diretamente do servidor e o JavaScript também é carregado para aprimorar a experiência de interação do usuário.

![Navegação no lado do cliente em PESPAs](https://res.cloudinary.com/epic-web/image/upload/v1665491191/epicweb.dev/blog/The%20Web%E2%80%99s%20Next%20Transition/pespa-clientside-navigation.svg)

**Navegação do lado do cliente:** quando o usuário clicar em um link, impediremos o comportamento padrão. Nosso roteador determinará os dados e a interface do usuário necessários para a nova rota e acionará a busca de dados para quaisquer dados que a próxima rota precise e renderizará a interface do usuário que é renderizada para essa rota.

![PESPA Inline Mutation Request](https://res.cloudinary.com/epic-web/image/upload/v1665491193/epicweb.dev/blog/The%20Web%E2%80%99s%20Next%20Transition/pespa-inline-mutation-request.svg)

![PESPA Redirect Mutation Request](https://res.cloudinary.com/epic-web/image/upload/v1665492615/epicweb.dev/blog/The%20Web%E2%80%99s%20Next%20Transition/pespa-redirect-mutation-request.svg)

**Solicitações de mutação:** Você notou que esses dois gráficos são iguais? Sim! Isso não é um acidente! As mutações com PESPAs são feitas por meio de envios de formulários. Chega de colocar isso em `onClick` + `fetch` (no entanto, mutações imperativas são boas para aprimoramento progressivo, como redirecionar para a tela de login quando a sessão do usuário expirar). Quando o usuário enviar um formulário, impediremos o comportamento padrão. Nosso código de mutação serializa o formulário e o envia como requisição para a rota associada ao `action` do formulário (que tem como padrão a URL atual). A lógica de roteamento no back-end chama a função _action_ que se comunica com o código de persistência para executar a atualização e envia de volta uma resposta bem-sucedida (por exemplo: uma curtição de tweet) ou redireciona (por exemplo: criando um novo repositório GitHub). Se for um redirecionamento, o roteador carrega código/dados/recursos para essa rota (em paralelo) e aciona a lógica de renderização. Se não for um redirecionamento, o roteador revalida os dados para a UI atual e aciona a lógica de renderização para atualizar a UI. Curiosamente, independentemente de ser uma mutação direta ou um redirecionamento, o roteador está envolvido, fornecendo o mesmo modelo mental para ambos os tipos de mutações.

# Prós e Contras de PESPAs

PESPAs eliminam muitos problemas de arquiteturas anteriores. Vamos vê-los um por um:

#### Questões em MPAs:

1. Atualizações de página inteira: PESPAs impedem o comportamento padrão e, em vez disso, usam JS do lado do cliente para emular o navegador. Da perspectiva do código que escrevemos, isso não parece diferente de um MPA, mas da perspectiva do usuário, é uma experiência muito melhor.

2. Controle de feedback da interface do usuário: PESPAs nos permitem controlar completamente as solicitações de rede porque estamos evitando o padrão e fazendo solicitações de busca e, portanto, podemos fornecer feedback aos usuários da maneira que fizer mais sentido para nossa interface do usuário.

#### Questões em PEMPAs:

1. Impedir o comportamento padrão: Um aspecto central dos PESPAs é que eles devem se comportar da mesma maneira que o navegador em relação ao roteamento e formulários. É assim que eles conseguem nos dar o modelo mental de uma MPA. Cancelar solicitações de reenvio de formulários, lidar com respostas fora de ordem adequadamente para evitar problemas de _race conditions_ e revelar erros para evitar spinners que nunca desaparecem fazem parte do que torna uma PESPA uma PESPA. É aqui que um framework realmente ajuda.

2. Código personalizado: Ao compartilhar código entre o cliente e o servidor e ter as abstrações certas que emulam o comportamento do navegador, acabamos reduzindo drasticamente a quantidade de código que temos que escrever.

3. Duplicação de código: Parte da idéia de um PESPA é que o servidor e o cliente usem exatamente o mesmo código para renderizar a lógica. Portanto, não há duplicação aqui. Não se esqueça do desafio: “faça uma interação do lado do cliente e, em seguida, verifique se a interface do usuário atualizada pelo cliente é a mesma que obtemos se atualizarmos a página”. Com um PESPA, essa atividade sempre passa sem esforço ou consideração por parte de nós desenvolvedores.

4. Organização do código: Devido ao modelo mental oferecido pela emulação do navegador em PESPAs, o gerenciamento do estado do aplicativo não é considerado. E a lógica de renderização é tratada da mesma forma em ambos os lados da rede, portanto, também não há mutações aleatórias do DOM.

5. Falta de sincronia entre Servidor/Cliente: O PESPA emulando o navegador significa que o código para o front-end e o código para o back-end são colocados, o que elimina a indireção e nos torna muito mais produtivos.

#### Problemas de SPA:

1. Tamanho dos pacotes/arquivos finais: Ir para um PESPA requer um servidor, o que significa que podemos mover uma tonelada de nosso código para o back-end. Tudo o que a interface do usuário precisa é de uma pequena biblioteca de interface do usuário que possa ser executada no servidor e no cliente, algum código para lidar com as interações e feedback da interface do usuário e o código para os componentes. E graças à divisão do código baseado em URLs/rotas, podemos finalmente dizer adeus às páginas da Web com centenas de KBs de JS. Além disso, devido ao aprimoramento progressivo, a maior parte do aplicativo deve funcionar antes que o carregamento do JS seja concluído. Além disso, há esforços em frameworks JS para reduzir ainda mais a quantidade de JS necessária no cliente.

2. Chamadas de rede em efeito cachoeira: Uma parte importante dos PESPAs é que eles podem estar cientes dos requisitos de código, dados e recursos para uma determinada URL sem precisar executar nenhum código. Isso significa que, além da divisão de código, os PESPAs podem acionar uma busca de código, dados e recursos de uma só vez, em vez de esperar um de cada vez em série. Isso também significa que os PESPAs podem pré-buscar essas coisas antes que o usuário acione uma navegação para que, quando necessário, o navegador possa renderizar as páginas imediatamente, fazendo com que toda a experiência de uso do aplicativo pareça instantânea.

3. Desempenho e performace em tempo de execução: PESPAs têm duas coisas a seu favor neste departamento, 1) eles movem muito código para o servidor, então há menos código para os dispositivos executarem em primeiro lugar e 2) graças ao Progressive Enhancement, a interface do usuário está pronta para uso antes que o JS termine de carregar e executar.

4. Gerenciamento de estado: Devido à emulação do navegador que nos fornece o modelo mental MPA, o gerenciamento de estado do aplicativo simplesmente não é uma preocupação em um contexto PESPA. A evidência disso é o fato de que o aplicativo deve funcionar principalmente sem JavaScript. Os PESPAs revalidam automaticamente os dados na página quando as mutações são concluídas (os MPAs obtiveram isso de graça, graças a um recarregamento de página inteira).

É importante ressaltar que PESPAs não funcionarão exatamente da mesma forma com e sem JavaScript do lado do cliente. De qualquer forma, esse nunca é o objetivo do aprimoramento progressivo. Só que a maior parte do aplicativo deve funcionar sem JavaScript. E não é apenas porque nos preocupamos com a experiência do usuário sem JavaScript. É porque, ao visar o aprimoramento progressivo, simplificamos drasticamente nosso código de interface do usuário. Você ficaria surpreso com o quão longe podemos chegar sem JS, mas para alguns aplicativos simplesmente não é necessário ou prático ter tudo funcionando sem JavaScript do lado do cliente. Mas ainda podemos colher os principais benefícios dos PESPAs, mesmo que alguns de nossos elementos de interface do usuário exijam algum JavaScript para operar.

O que distingue uma PESPA:

- Funcional é a linha de base, JS é usado para aprimorar, não habilitar
- Carregamento tardio (de código que não está em uso) + pré-busca inteligente (mais do que apenas código JS)
- Envia o código para o servidor
- Nenhuma duplicação manual de código de interface do usuário (como em PEMPAs)
- Emulação de navegador transparente (#useThePlatform)

Quanto aos contras. Ainda estamos descobrindo quais são. Mas aqui estão alguns pensamentos e reações iniciais:

**Muitos que estão acostumados com SPAs e SSG lamentarão que agora tenhamos código do lado do servidor executando nosso aplicativo.** No entanto, para qualquer aplicativo do mundo real, não podemos evitar o código do lado do servidor. Certamente, existem alguns casos de uso em que podemos criar o site inteiro uma vez e colocá-lo em um CDN, mas a maioria dos aplicativos nos quais trabalhamos diariamente não se encaixa nessa categoria.

Relacionado a isso, as pessoas estão preocupadas com o custo do servidor. A ideia é que o SSG nos permite construir nosso aplicativo uma vez e depois podemos colocar em uma CDN para um número quase infinito de usuários a um custo muito baixo. Há duas falhas nessa crítica. 1) Provavelmente estamos acessando APIs em nosso aplicativo, então esses usuários ainda estarão acionando muitos de nossos códigos de servidor mais caros em suas visitas de qualquer maneira. 2) CDNs oferecem suporte a mecanismos de cache HTTP, portanto, se formos realmente capazes de usar SSG, podemos definitivamente fazer uso disso para fornecer respostas rápidas e limitar a quantidade de trabalho com o qual nosso servidor de renderização está lidando.

Outro problema comum que as pessoas têm ao deixar SPAs é que agora temos que lidar com os desafios de renderização no servidor. Este é definitivamente um modelo diferente para pessoas acostumadas a executar seu código apenas no lado do cliente, mas se estivermos usando frameworks que cuidam disso e levam isso em consideração, dificilmente será um desafio. Se não, pode ser definitivamente um desafio grande, mas existem soluções alternativas razoáveis ​​para forçar determinado código a ser executado apenas no lado do cliente enquanto migramos.

Como eu disse, ainda estamos descobrindo os contras dos aplicativos de página única aprimorados progressivamente, mas acho que os benefícios valem muito a pena comparado aos contras.

Também devo mencionar que, embora tenhamos os recursos de uma arquitetura PESPA por algum tempo com as ferramentas existentes, o foco no Progressive Enhancement, ao mesmo tempo em que compartilhamos o código da lógica de renderização, é novo. Este post está interessado principalmente em demonstrar as arquiteturas padrões em aplicativo web, não apenas os recursos da plataforma.

# Uma implementação PESPA: Remix

Liderando a busca de PESPAs está o [Remix](https://remix.run/), um framework web com foco a laser nos fundamentos da web e na experiência moderna do usuário. Remix é o primeiro framework web a oferecer tudo o que descrevi como uma oferta PESPA. Outros frameworks podem e estão se adaptando para seguir o exemplo do Remix nisso. Estou especificamente ciente de que [SvelteKit](https://github.com/sveltejs/kit/discussions/5875) e [SolidStart](https://github.com/solidjs/solid-start/pull/74) trabalham nos princípios PESPA em suas implementações. Imagino que mais virão (novamente, os meta-frameworks são capazes de arquitetura PESPA há algum tempo, no entanto, o Remix colocou essa arquitetura em primeiro plano e outros estão seguindo a idéia). Veja como as coisas ficam quando temos um framework web para o nosso PESPA:

![Implementação da arquitetura PESPA pelo framework Remix](https://res.cloudinary.com/epic-web/image/upload/v1665491189/epicweb.dev/blog/The%20Web%E2%80%99s%20Next%20Transition/pespa-remix.svg)

Nesse caso, o Remix atua como uma ponte na rede. Sem o Remix, teríamos que implementar isso nós mesmos para ter um PESPA completo. O Remix também lida com nosso roteamento por meio de uma combinação de roteamento baseado em convenção e baseado em configuração. O Remix também ajudará com os pedaços de aprimoramento progressivo de nossas buscas e mutações de dados (como o botão de curtir do Twitter) e o feedback da interface do usuário para implementar coisas como estados pendentes e interface do usuário otimista.

Graças ao roteamento aninhado incorporado ao Remix, também obtemos uma melhor organização do código (algo que o Next.js também está buscando). Embora o roteamento aninhado não seja necessário para a arquitetura PESPA especificamente, a divisão de código baseada em rota é uma parte importante. Além disso, obtemos uma divisão de código muito mais granular com roteamento aninhado, por isso é um aspecto importante.

O Remix está demonstrando que podemos nos divertir mais criando melhores experiências mais rapidamente com a arquitetura PESPA. E acabamos com situações como esta:

[![Um tweet do @shamwhoah com uma captura de tela com pontuação de 100% no Lighthouse para um website usando Remix](https://www.epicweb.dev/_next/image?url=https%3A%2F%2Fres.cloudinary.com%2Fepic-web%2Fimage%2Fupload%2Fv1665167665%2Fepicweb.dev%2Fblog%2FThe%2520Web%25E2%2580%2599s%2520Next%2520Transition%2Ftweet_2x.png&w=1920&q=100)](https://twitter.com/shamwhoah/status/1575619809714503681)

Uma de desempenho perfeita no Lighthouse por padrão? Pode me chamar!

# Conclusão

Pessoalmente, estou pronto aqui para esta transição. Obter um UX e DX melhores ao mesmo tempo é uma vitória sólida. Acho que é importante e estou animado com o que o futuro nos reserva. Como recompensa por terminar esta postagem no blog, criei um repositório que demonstra todo esse código se movendo através dos tempos usando um aplicativo TodoMVC! Veja ele aqui: [kentcdodds/the-webs-next-transformation](https://github.com/kentcdodds/the-webs-next-transformation). Espero que ajude a tornar algumas das ideias mais concretas.

E é isso que estou animado para ensinar a você aqui no [EpicWeb.dev](https://www.epicweb.dev/). Se você gostaria de acompanhar meu progresso aqui, coloque [seu e-mail na caixa de formulário](https://www.epicweb.dev/). Vamos tornar a web melhor 🎉

Valeu!

---

Para uma visão mais detalhada da história da construção de aplicativos para a web, leia ["The new wave of Javascript web frameworks"](https://frontendmastery.com/posts/the-new-wave-of-javascript-web-frameworks/) por Frontend Mastery.

Para saber mais sobre aprimoramento progressivo, leia ["Progressively enhance for a more resilient web"](https://jjenzz.com/progressively-enhance-for-a-more-resilient-web), de [Jenna Smith](https://jjenzz.com/).

---

# Créditos

- Escrito originalmente por [Kent C. Dodds](https://twitter.com/kentcdodds), em [The Web’s Next Transition](https://www.epicweb.dev/the-webs-next-transition).
