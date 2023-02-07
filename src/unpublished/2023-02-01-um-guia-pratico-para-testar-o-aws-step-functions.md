# Um guia prático para testar o AWS Step Functions

Testar Step Functions pode ser uma tarefa assustadora. No entanto, com um pouco de preparação e esforço, o processo de teste pode ser simplificado e otimizado. Neste artigo, forneceremos uma estratégia prática sobre como testar Step Functions. Então, vamos começar preparando o cenário e apresentando os jogadores.

# O que torna o Step Functions difícil de testar?

As Step Functions são complicadas por natureza porque consistem em muitas partes diferentes que precisam ser testadas individualmente.

Por exemplo, uma máquina de estado típica do Step Functions pode incluir ramificação condicional que pode direcionar a execução para diferentes caminhos. Todos esses caminhos precisam ser testados para que possamos ter certeza de que estão funcionando conforme o esperado.

![](https://theburningmonk.com/wp-content/uploads/2022/12/img_638bb6247e355.png)

Também podemos incorporar a lógica `try..catch` em nossa máquina de estado para nos ajudar a recuperar de erros que possam ocorrer em tempo de execução. Essa lógica também precisa ser testada minuciosamente para garantir que funcione corretamente.

![](https://theburningmonk.com/wp-content/uploads/2022/12/img_638bb65987708.png)

Outra coisa importante a observar é que o Step Functions funciona em um ambiente altamente distribuído. Podemos usar as funções do Lambda para executar a lógica de negócios personalizada, bem como integrar diretamente com mais de 200 serviços da AWS. Toda essa funcionalidade precisa ser testada minuciosamente para garantir que ela se comporte adequadamente quando lançarmos nosso aplicativo em produção.

Além do mais, você pode usar [padrões de retorno](https://docs.aws.amazon.com/step-functions/latest/dg/callback-task-sample-sqs.html) de chamada para suspender a execução enquanto esperamos que os processos externos sejam concluídos com êxito antes de prosseguir com nosso caminho de execução.

Esse padrão também pode ser muito útil para integrar etapas manuais em um processo automatizado. Por exemplo, para permitir que um operador humano aprove ou rejeite uma solicitação para implantar nosso aplicativo depois que o commit for enviado ao GitHub.

Finalmente, você tem os estados `Wait`. O que pode adicionar uma quantidade configurável de atraso à execução e dificultar o teste de ponta a ponta. Por exemplo, se houver um estado de espera que espere por uma hora, seu teste também precisaria dormir por tanto tempo? E você ficaria feliz em esperar uma hora para executar todos os casos de teste em sua suíte de testes? Provavelmente não!

Todos esses fatores tornam o Step Functions notoriamente difícil de testar. Mas existem maneiras de superar esses desafios (pelo menos até certo ponto) e produzir casos de teste de alta qualidade.

# Testando com Step Functions Local

[Step Functions Local](https://docs.aws.amazon.com/step-functions/latest/dg/sfn-local.html) é um simulador local para Step Functions e pode executar nossas máquinas de estado localmente. Eu geralmente evito simuladores locais (como [localstack](https://localstack.cloud/)) porque eles geralmente são mais problemáticos do que valem a pena. No entanto, abro uma exceção para Step Functions Local porque sua capacidade de simulação é quase uma necessidade se você deseja obter uma boa cobertura de teste para Step Functions.

Por padrão, o Step Functions Local executa os estados `Task` contra os serviços reais da AWS (por exemplo, S3 ou DynamoDB). Podemos substituir os endpoints dos serviços AWS na configuração do Step Functions Local.

![](https://theburningmonk.com/wp-content/uploads/2022/12/img_638bb68bc28da.png)

Usando esse recurso, podemos substituir as chamadas de serviços por simulações locais durante a execução de nossos testes. Embora isso possa parecer útil, apenas um pequeno número de serviços é suportado e isso significaria que teríamos que usar outras ferramentas para simular esses serviços da AWS.

O mais importante, o Step Functions Local nos permite mockar a saída dos estados `Task`. Essas respostas simuladas devem ser fornecidas quando você inicia o Step Functions Local. Mas podemos usá-las para ajudar a conduzir a execução da máquina de estado por um caminho específico que queremos testar. Por exemplo, lançando o erro certo em um estado `Tasks` para nos ajudar a testar o caminho de erro em nossa máquina de estado. Ou fornecendo a saída certa de um estado `Task` que alimenta um estado `Choice` para ter certeza de que seguiríamos o ramo certo.

![](https://theburningmonk.com/wp-content/uploads/2022/12/img_638bb6ae49b1d.png)

# Problemas com Step Functions Local

Embora seja uma ferramenta útil e que você deve ter em seu kit, ainda existem alguns problemas notáveis ​​com ela.

1. Não há garantia de que as chamadas simuladas funcionarão exatamente da mesma maneira que na produção. Este é um problema com todas as simulações e simuladores locais. Que eles não são uma réplica perfeita do ambiente de execução real e são propensos a apresentar atrasos e falsos negativos ou positivos.

2. Ele não simula o IAM e, portanto, não pode nos ajudar a detectar erros relacionados à permissão em nossa máquina de estado. Isso é importante, especialmente em uma máquina de estado complexa que interage com muitos serviços da AWS por meio de integrações diretas de serviço.

3. Não suporta a funcionalidade de avanço rápido através de um estado `Wait` ou simulando um tempo esgotado nos estados `Task`.

4. Por último e talvez o mais importante, ele não oferece suporte a referências do CloudFormation. Quando você cria uma máquina de estado no Step Functions Local, o ASL deve usar ARNs ao invés de referências do CloudFormation porque a máquina de estado não é implantada como parte de uma pilha do CloudFormation. Isso significa que você não pode usar referências a outros recursos na máquina de estado. Por exemplo, você não poderia fazer referência a um bucket do Amazon S3 se quisesse que sua máquina de estado gravasse dados nele.

Este último ponto cria atrito em seu fluxo de trabalho de desenvolvimento. Isso significa que "testar antes de deployar" não é realmente viável sem primeiro deployar o projeto e criar os recursos referenciados pela máquina de estado.

# Teste de ponta-a-ponta

Para executar testes de ponta-a-ponta, precisamos deployar o projeto na AWS e criamos uma máquina de estado e todos os recursos aos quais ela faz referência. Em seguida, executaríamos a máquina de estado com diferentes entradas para cobrir diferentes caminhos.

No entanto, muitas vezes é difícil ou impossível cobrir todos os caminhos de execução usando testes de ponta-a-ponta. Por exemplo, uma lógica de ramificação pode depender do resultado de uma chamada de API para uma API de terceiros, como Stripe ou Paypal. Ou talvez um caminho de erro dependa do DynamoDB lançar um erro. Estes são apenas alguns exemplos de cenários que não podemos cobrir facilmente usando testes de ponta-a-ponta.

Para alguns desses cenários, podemos usar APIs simuladas e retornar resultados fictícios para nossa lógica de ramificação.

Por exemplo, você pode usar o [Apidog](https://www.apidog.com/) para hospedar uma API Stripe simulada para testar o fluxo de pagamento de sua máquina de estado. Você também pode hospedar uma API simulada local e expô-la publicamente usando [ngrok](https://ngrok.com/).

Em ambos os casos, não estamos realmente interagindo com a API real de terceiros. Portanto, qualquer falha em nossa lógica, isso não afetará nossos clientes, pois eles não receberão um pagamento com falha em um cartão de crédito real. Mas a API simulada nos informará que nossa lógica está falhando quando não deveria.

No entanto, precisamos de uma maneira de "convencer" nossa máquina de estado a usar a API fictícia em vez da API real de terceiros. Uma maneira de fazer isso é adicionar a URL da API fictícia como variável em tempo de execução, e fazer com que nossas funções do Lambda a usem sempre que for especificada.

# Teste de componentes para as funções do Lambda

Se uma máquina de estado consistir principalmente de funções Lambda, podemos testar cada função separadamente usando técnicas de teste baseadas em componentes:

- Encapsule a lógica do domínio em seus próprios módulos e escreva testes de unidade para eles.

- Escreva testes de integração (também conhecidos como testes de usuário) que invoquem o código da função Lambda localmente, mas que se comuniquem com os serviços reais da AWS (sem simulações ou simuladores locais!). Esses testes permitem iterar rapidamente no código funcional sem precisar implantá-lo na AWS após cada alteração.

- Depois de ganhar confiança com os testes locais, implante tudo na AWS e teste as funções do Lambda como parte dos testes de ponta-a-ponta na máquina de estado.

Para saber mais sobre essa abordagem para testar as funções Lambda, confira meu artigo: [My strategy for testing serverless applications](https://theburningmonk.com/2022/05/my-testing-strategy-for-serverless-applications).

Ou, se você quiser um passo a passo mais aprofundado e ver isso em ação, confira meu novo curso ["Testing Serverless Architectures"](https://testserverlessapps.com/?utm_campaign=practical-guide-to-testing-sfn&utm_source=blog).

# Uma estratégia para testar Step Functions

Ok, até agora nós cobrimos os desafios com o teste de Step Functions e apresentamos três abordagens para testá-los:

- Usando Step Functions Local.
- Usando testes de ponta-a-ponta.
- Teste de componentes em funções individuais do Lambda.

Vamos combiná-los para chegar a uma estratégia que nos dê o melhor de todas as três abordagens.

Primeiro, use o teste de componentes para as funções individuais do Lambda.

Em seguida, tente cobrir o maior número possível de caminhos de execução usando testes de ponta-a-ponta. No entanto, lembre-se de que os testes de ponta-a-ponta não podem cobrir todos os caminhos de execução possíveis para uma máquina de estado. Ou, no mínimo, será muito desafiador atingir 100% de cobertura com testes de ponta-a-ponta. Portanto, é provável que haja algumas lacunas em nossa cobertura de teste. Por exemplo, alguns caminhos de difícil acesso com `Choice` e caminhos de erro.

Finalmente, use Step Functions Local e respostas simuladas para preencher as lacunas na cobertura dos testes de ponta-a-ponta. Onde não podemos direcionar os testes de ponta-a-ponta para um caso de teste que queremos executar, podemos usar as respostas simuladas no Step Functions Local para direcionar a máquina de estado para esses caminhos de execução.

Mas espere!

O objetivo do Step Functions Local não é nos permitir testar nossas máquinas de estado localmente sem fazer o deploy na AWS?

Na prática, isso é muito difícil de conseguir porque ela não oferece suporte a referências do CloudFormation. Em vez disso, acho que a melhor maneira de usar o Step Functions Local é preencher as lacunas em nossos testes de ponta-a-ponta.

Neste caso, eu faria:

1. Faça o deploy da máquina de estado e todos os outros recursos dos quais ela depende na AWS. A máquina de estado deployada conteria os ARNs totalmente qualificados em vez das referências do CloudFormation.
2. Inicie o Step Functions Local com as respostas fictícias necessárias para meus casos de teste.
3. Crie a máquina de estado no Step Functions Local, usando a definição da máquina de estado que foi deployada.
4. Execute casos de teste no Step Functions Local.

A combinação de testes de ponta-a-ponta com o Step Functions Local dessa forma forneceria quase 100% de cobertura de todos os caminhos de execução.

No entanto, ainda pode haver algumas lacunas deixadas em nossa cobertura de teste. Especificamente, quando estados `Wait` e timeouts `Task` estão sendo testados. Como não é viável escrever casos de teste que teriam que esperar indefinidamente, e o Step Functions Local não oferece suporte ao avanço rápido por meio desses estados `Wait`. A única solução viável que encontrei é usar o Step Functions Local e reescrever o relevante estados `Wait` para _esperar apenas um segundo_.

Podemos fazer isso na etapa 3 acima, quando criamos a máquina de estado no Step Functions Local. O mesmo pode ser feito para diminuir os tempos limite e, portanto, tornar viável o teste dos caminhos de erro.

# Finalizando

A estratégia que descrevi acima nos dá o melhor dos dois mundos — teste de ponta-a-ponta combinado com a execução local de alguns casos de teste que requerem atenção especial.

Espero que este post tenha sido útil para você. Se você quiser aprender mais sobre como testar arquiteturas serverless, incluindo Step Functions, verifique meu novo curso ["Testing Serverless Architectures"](https://testserverlessapps.com/?utm_campaign=widget&utm_source=blog).

Se você se inscrever antes de 1º de janeiro de 2023, também poderá obter 30% de desconto com nosso desconto de acesso antecipado! Espero vê-lo no curso :-)

![](https://theburningmonk.com/wp-content/uploads/2022/12/img_638bb6f03af8f.png)

---

# Créditos

- Escrito originalmente por [Yan Cui](https://twitter.com/theburningmonk), em [A practical guide to testing AWS Step Functions](https://theburningmonk.com/2022/12/a-practical-guide-to-testing-aws-step-functions/)
