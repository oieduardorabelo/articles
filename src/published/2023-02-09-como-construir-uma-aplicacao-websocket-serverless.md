# Como construir uma aplicação WebSocket Serverless

Ao criar aplicativos da web modernos, é cada vez mais importante ser capaz de lidar com dados em tempo real com uma arquitetura orientada a eventos para propagar mensagens para todos os clientes conectados instantaneamente.

Vários protocolos estão disponíveis, mas o WebSocket é indiscutivelmente o mais usado, pois é otimizado para sobrecarga mínima e baixa latência. O protocolo WebSocket oferece suporte à comunicação bidirecional full-duplex entre cliente e servidor em uma conexão persistente de soquete único. Com uma conexão WebSocket, você pode eliminar o polling e enviar atualizações para um cliente assim que um evento ocorrer.

## Como integrar WebSocket em sua aplicação

Você pode fornecer uma arquitetura orientada a eventos configurando um servidor WebSocket dedicado para os clientes se conectarem e receberem atualizações. No entanto, essa arquitetura tem várias desvantagens, incluindo a necessidade de gerenciar e [escalar o servidor](https://ably.com/topic/the-challenge-of-scaling-websockets) e a latência inerente no envio de atualizações desse servidor para clientes, que podem ser distribuídos mundialmente.

Para expandir seu hardware em todo o mundo, você pode delegar a responsabilidade e o custo de hardware ao alugar recursos de um provedor de serviços em nuvem. Por exemplo, você pode usar um serviço de nuvem gerenciado, como o Amazon ECS, ou pode optar por uma solução WebSocket Serverless.

> _Um aplicativo serverless é aquele que não custa nada para você executar quando ninguém o está usando, excluindo os custos de armazenamento de dados._ - [Cloud 2.0: Code is no longer King — Serverless has dethroned it](https://pauldjohnston.medium.com/cloud-2-0-code-is-no-longer-king-serverless-has-dethroned-it-c6dc955db9d5)

Economizar nos custos operacionais é um excelente motivo para considerar uma estrutura serverless: você colhe os benefícios de escala e elasticidade, mas não precisa provisionar ou gerenciar os servidores.

Este artigo explica como criar uma aplicação básica de WebSocket Serverless, ideal para aplicativos simples, como bate-papo, usando o AWS API Gateway para criar um endpoint WebSocket e o AWS Lambda para gerenciamento de conexão e lógica de negócios de back-end.

## A arquitetura básica de uma aplicação WebSocket Serverless

Construir uma aplicação simples de WebSocket Serverless é relativamente direto, usando o AWS API Gateway para criar APIs WebSocket e funções Lambda para processamento do back-end, armazenando metadados de mensagem no Amazon DynamoDB.

### API Gateway

O Amazon API Gateway é um serviço para criar e gerenciar APIs. Você pode usá-lo como um proxy com estado para encerrar conexões WebSocket persistentes e transferir dados entre seus clientes e back-ends baseados em HTTP, como AWS Lambda, Amazon Kinesis ou qualquer outro endpoint HTTP.

### AWS Lambda

O AWS Lambda é um serviço de computação serverless que permite executar código sem provisionar ou gerenciar servidores.

As APIs WebSocket podem invocar funções do Lambda, por exemplo, para entregar mensagens para processamento adicional e, em seguida, retornar o resultado ao cliente. As funções do Lambda são executadas apenas pelo tempo necessário para lidar com qualquer tarefa individual. Para autorização de conexões de clientes, a solicitação de conexão deve incluir credenciais a serem verificadas por uma [função de autorizador do Lambda](https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-use-lambda-authorizer.html) que retorna a política AWS IAM apropriada para direitos de acesso.

## Exemplo: aplicativo de bate-papo com WebSocket Serverless

Como é tradicional, vamos ver um exemplo de aplicativo de bate-papo para ilustrar como a solução WebSocket Serverless alimenta um aplicativo simples de tempo real. Primeiro, considere a sequência pela qual uma conexão WebSocket do cliente é feita e armazenada.

1.  Um dispositivo cliente estabelece uma única conexão WebSocket com o API Gateway. Cada conexão WebSocket ativa tem um URL de retorno individual no API Gateway que é usado para enviar mensagens de volta ao cliente correspondente.
2.  A mensagem de conexão é enviada para uma função de autoriazação no AWS Lambda do API Gateway para confirmar se o dispositivo possui as credenciais corretas para se conectar ao serviço.
3.  A mensagem de conexão e os metadados são enviados para um AWS Lambda separado quando ele é autenticado.
4.  O manipulador Lambda inspeciona os metadados e armazena as informações apropriadas no Amazon DynamoDB, um banco de dados de documentos NoSQL.

![](https://ik.imagekit.io/ably/ghost/prod/2022/09/basic-serverless-connection-with-aws@2x.png?tr=w-1520,q-50)

Em um aplicativo de bate-papo, os usuários podem se envolver em várias conversas envolvendo um ou mais participantes. Este é um caso de uso típico para o padrão de publicação e assinatura (_pub/sub_). Quando um usuário envia uma mensagem para uma conversa ou canal específico, o aplicativo de bate-papo _publica_ a mensagem para todos os outros usuários que _se inscrevem_ nesse canal. Cada assinante desse canal recebe a mensagem em seu aplicativo.

Quando o aplicativo de bate-papo inicia uma conexão WebSocket com o API Gateway, deve haver uma maneira de determinar os canais para os quais essa conexão está autorizada a publicar e se inscrever. O código do cliente deve enviar essas informações dentro dos metadados da conexão.

Agora vamos considerar o ponto em que o usuário envia uma mensagem e outro usuário a recebe:

1.  Um cliente envia uma mensagem de bate-papo, incluindo os metadados para identificar o canal específico no qual publicar.
2.  No recebimento, o API Gateway o envia para a função Lambda apropriada.
3.  Após a chamada, o Lambda verifica as conexões inscritas nesse canal e envia ao API Gateway o ID de cada assinante e a mensagem enviada.
4.  O API Gateway usa as URLs de retorno para cada conexão de assinante para enviar a mensagem ao cliente da conexão WebSocket estabelecida.
5.  Após a desconexão, uma função do Lambda é invocada para limpar o ID de conexão do banco de dados.

![](https://ik.imagekit.io/ably/ghost/prod/2022/09/basic-serverless-message-send-with-aws@2x.png?tr=w-1520,q-50)

## As limitações dessa implementação simples de WebSocket Serverless

O exemplo acima é um exemplo simplista de um aplicativo orientado a eventos em tempo real usando WebSocket Serverless. Algumas das limitações incluem:

**O rastreamento do estado da conexão não escala:** O API Gateway não rastreia metadados de conexão, e é por isso que ele precisa ser armazenado em um banco de dados como o [Amazon DynamoDB](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Introduction.html) usando uma função Lambda para atualizar o banco para cada conexão/desconexão. Para um grande número de clientes conectados, você atingiria os limites de escala do Lambda. Ao invés disso, você pode usar a [integração nativa do API Gateway para chamar o DynamoDB diretamente](https://docs.aws.amazon.com/apigateway/latest/developerguide/how-to-integration-settings.html), mas você precisará ficar atento ao alto nível de uso do banco de dados.

Outro aspecto que não abordamos é o da desconexão abrupta. Nos casos em que uma conexão do cliente cai sem aviso, a conexão ativa do WebSocket não é devidamente limpa. O banco de dados provavelmente armazenará identificadores de conexão que não estão mais presentes, o que pode causar ineficiências.

Uma maneira de evitar conexões zumbis é verificar periodicamente se uma conexão está "ativa" usando um mecanismo de pulsação, como TCP keepalives ou [controle de frames em ping/pong](https://tools.ietf.org/html/rfc6455#page-36). O envio de pulsações afeta negativamente a escalabilidade e a confiabilidade do sistema, especialmente ao lidar com milhões de conexões WebSocket simultâneas, pois cria uma carga extra no sistema.

**Você não pode transmitir mensagens para todos os clientes conectados:** Neste design simples, não há como enviar uma mensagem simultaneamente para várias conexões, como seria de esperar de um canal ou tópico pub/sub padrão. Distribuir uma atualização publicando-a para milhares de clientes exigiria que você buscasse os IDs de conexão do banco de dados e, em seguida, fizesse uma chamada de API para cada um, o que não é uma abordagem escalável. Como alternativa, você pode considerar o Amazon SNS para adicionar pub/sub ao seu aplicativo, mas ainda há limitações e a desvantagem da complexidade adicional. Para um exemplo de aplicativo de bate-papo, você pode supor que os canais típicos terão um número baixo de inscritos, a menos que sejam usados em sessões interativas de transmissão ao vivo com milhares de usuários. Outros cenários que podem tolerar a falta de transmissão incluem notificações de atualização individuais e recursos de interatividade que funcionam item por item. Os casos de uso de _fan-out_ de dados ao vivo não podem suportar essa limitação se houver um grande número de assinantes em um canal para receber o placar esportivo ou atualização de notícias mais recente.

**Há um limite para o número de conexões WebSocket permitidas por segundo:** o AWS API Gateway define um limite por região de [500 novas conexões por segundo por conta](https://docs.aws.amazon.com/apigateway/latest/developerguide/limits.html) e 10.000 conexões simultâneas. Isso pode ser insuficiente se você planeja escalar seu aplicativo.

**A aplicação está vinculada a uma única região da AWS:** As conexões WebSocket oferecidas pelo API Gateway estão vinculadas a uma única região, resultando em baixo desempenho devido à latência para clientes distantes geograficamente dessa região. É possível usar o [Amazon EventBridge](https://ably.zoom.us/j/87643486944?pwd=ZEh1MzgxQUFOKytQSEFrbUNyS2UxUT09), um _event bus_ serverless, para roteamento de eventos entre regiões para replicar eventos.

## Desafios da construção de uma aplicação WebSocket Serverless pronta para produção

A arquitetura acima tem potencial se usada para criar uma aplicação simples de mensagens em tempo real, embora já tenhamos visto que ela tem sérias limitações em escala. Se você precisar de um sistema pronto para produção, precisará escala para milhões de conexões, mas também precisará considerar a confiabilidade de sua solução em termos de desempenho, integridade da entrega de mensagens, tolerância a falhas e suporte de fallback, todos os quais podem ser complexos e demorados. consumindo tempo para resolver.

### Desempenho

Em um sistema distribuído, a latência se deteriora à medida que avança, portanto, os dados devem ser mantidos o mais próximo possível dos usuários por meio de datacenters gerenciados e [edge acceleration points](https://ably.com/blog/what-is-edge-messaging) . No entanto, não é suficiente para minimizar a latência. A experiência do usuário precisa que a variação de latência seja mínima para garantir a previsibilidade.

### Integridade da mensagem

Quando a conexão de um usuário cai e se reconecta, o aplicativo precisa continuar com o mínimo de atrito do ponto antes de ser desconectado. Quaisquer mensagens perdidas precisam ser entregues sem duplicar as já processadas. Toda a experiência precisa ser [totalmente integrada](https://ably.com/blog/achieving-exactly-once-message-processing-with-ably).

### Tolerância a falhas e escalabilidade

A demanda por um serviço pode ser imprevisível e é um desafio planejar e fornecer capacidade suficiente. Uma solução em tempo real deve estar altamente disponível em todos os momentos para suportar picos de dados. O desafio é escalar horizontalmente e absorver rapidamente milhões de conexões sem a necessidade de pré-provisionamento.

Sua solução também deve ser [tolerante](https://ably.com/blog/engineering-dependability-and-fault-tolerance-in-a-distributed-system) a falhas para continuar operando mesmo se um componente falhar. É necessário haver vários componentes capazes de manter o sistema se alguns forem perdidos. Não deve haver nenhum ponto único de congestionamento e nenhum ponto único de falha.

### Conexões alternativas

Apesar do amplo suporte, o protocolo WebSocket não é suportado por todos os proxies ou navegadores, e alguns firewalls corporativos até bloqueiam portas específicas, o que pode afetar o acesso ao WebSocket. Você pode considerar oferecer suporte a conexões alternativas, como streaming XHR, XHR polling ou long-polling.

### Desvio de recursos

Outro desafio para o uso de WebSocket Serverless puro em um sistema pronto para produção pode se apresentar apenas depois de usá-lo com sucesso na primeira iteração do produto. Embora você tenha adicionado o recurso essencial para receber atualizações em tempo real, o aumento de recursos geralmente significa que os recursos geram requisitos adicionais para experiências ao vivo compartilhadas e recursos colaborativos.

Construir e manter uma solução proprietária de WebSocket para suportar as futuras necessidades em tempo real de um produto pode ser um desafio. A infraestrutura que sustenta o sistema deve ser estável e confiável e requer engenheiros experientes para construí-la e mantê-la. Uma equipe de desenvolvimento pode descobrir que tem um compromisso de longo prazo para oferecer suporte à solução em tempo real, em vez de se concentrar nos recursos que aumentam o produto principal.

## Ably: um WebSocket PaaS sem servidor criado para escala

Adotar uma solução WebSocket Serverless de um fornecedor terceirizado faz mais sentido financeiro para muitas organizações. Caso contrário, você pode gastar vários meses e muito dinheiro. Você acabaria sobrecarregado com o alto custo de propriedade da infraestrutura, dívida técnica e demandas contínuas de investimento em engenharia.

[O Ably](https://www.ably.com) possui infraestrutura global elástica tolerante a falhas e altamente disponível para escalabilidade sem esforço e baixa complexidade. Nossa plataforma WebSocket Serverless é modelada matematicamente em torno [dos Quatro Pilares de Confiabilidade](https://ably.com/four-pillars-of-dependability). Podemos garantir que as mensagens sejam entregues com baixa latência em uma rede de ponta segura, confiável e global. Não há sobrecarga de DevOps e não há infraestrutura para gerenciar.

A plataforma Ably abstrai a preocupação de construir uma solução em tempo real para que você possa priorizar o roadmap do seu produto e desenvolver recursos para se manter competitivo.

Embora o protocolo nativo de Ably seja o WebSocket, raramente existe um protocolo de tamanho único: protocolos diferentes atendem a propósitos diferentes melhor do que outros. O Ably oferece vários [protocolos](https://ably.com/protocols) , como WebSocket, MQTT, SSE e HTTP puro.

Para oferecer suporte a experiências ao vivo, Ably adiciona recursos como [presença de dispositivo](https://ably.com/docs/core-features/presence) , [histórico de transmissão](https://ably.com/docs/realtime/history) , [rebobinar](https://ably.com/docs/realtime/channels/channel-parameters/rewind) canais e tratamento de [desconexões abruptas](https://ably.com/docs/realtime/connection#connection-state-recovery), facilitando a criação de aplicativos avançados de tempo real. Há uma variedade de integrações de webhook para acionar a lógica de negócios em tempo real. Oferecemos um gateway para funções serverless de seus provedores de serviços de nuvem preferidos. Você pode implantar as melhores ferramentas da categoria em toda a sua pilha e criar aplicativos orientados a eventos usando os ecossistemas nos quais já investiu.

![](https://ik.imagekit.io/ably/ghost/prod/2022/09/how-ably-augments-basic-serverless-websockets@2x.png?tr=w-1520,q-50)

Ably permite que você ofereça uma experiência ao vivo sem atrasos, custos descontrolados ou usuários insatisfeitos. O uso do Ably permite que suas equipes de engenharia se concentrem na inovação do produto principal sem precisar provisionar e manter a infraestrutura em tempo real. [Nossos clientes existentes](https://ably.com/case-studies) já criam aplicativos em tempo real massivamente escaláveis, com entrada rápida no mercado e economia média de US$ 500.000 no custo de construção e manutenção.

Faça o download do nosso relatório ["O estado da infraestrutura WebSocket Serverless"](https://ably.com/resources/reports/the-state-of-serverless-websocket-infrastructure).

## Conclusão

Neste artigo, revisamos a arquitetura de uma plataforma simples de WebSocket Serverless usando o Amazon API Gateway e o AWS Lambda. Também consideramos algumas das limitações desse design, como falta de suporte para transmissão de mensagens em larga escala, deploy de região única e gerenciamento de conexão.

Apresentamos a plataforma WebSocket Serverless da Ably, que lida de forma confiável com a distribuição de dados em tempo real em alta escala para aplicativos móveis e da web.

Você [pode entrar em contato](https://www.ably.io/contact?__hstc=12655464.d4d5789854ba0058155a69ee37650d08.1632475645006.1661951951785.1661956486878.594&__hssc=12655464.13.1661956486878&__hsfp=2252184059) para saber como podemos trabalhar com você!

---

# Créditos

- Escrito originalmente por [Jo Stichbury](https://ably.com/blog/author/jo-stichbury), em [How to build a serverless WebSockets platform](https://ably.com/blog/how-to-build-a-serverless-websocket-platform).
