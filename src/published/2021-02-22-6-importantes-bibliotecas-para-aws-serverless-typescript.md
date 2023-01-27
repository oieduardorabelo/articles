- https://dev.to/oieduardorabelo/6-importantes-bibliotecas-para-aws-serverless-typescript-554o
- https://oieduardorabelo.medium.com/6-importantes-bibliotecas-para-aws-serverless-typescript-bd93ec1c3ec6

---

# 6 importantes bibliotecas para AWS Serverless TypeScript

Se você é um **desenvolvedor AWS Serverless TypeScript,** aqui está uma lista de bibliotecas que você deve conhecer e usar para melhorar sua experiência de desenvolvedor. A maioria das bibliotecas listadas neste artigo são executadas em tempo de execução ao invés de serem ferramentas de desenvolvedor.

## Middy.js

**Fonte:** [https://github.com/middyjs/middy](https://github.com/middyjs/middy)

![](https://miro.medium.com/max/450/1*Su8PKNEL01_hNGgAFGLxkQ.png)

Middy é uma biblioteca para organizar seu código em middlewares. Usando o middy, você pode padronizar as operações a serem executadas antes, depois e sempre que um erro é lançado no código do manipulador lambda, sem duplicar o código em cada arquivo. Middy vem com vários middlewares dedicados a tarefas específicas, como análise da requisição HTTP do API Gateway, validação de eventos com esquema JSON, captura de erros e retorno correto de respostas HTTP formatadas. Você também pode escrever seu próprio middleware.

Em combinação com [os middleware do AWS JavaScript SDK](https://aws.amazon.com/fr/blogs/developer/middleware-stack-modular-aws-sdk-js/), que será introduzida na v3, a base de código de todos os manipuladores será reduzida ainda mais, mantendo apenas linhas específicas de negócios e movendo todas as tarefas técnicas para esses middlewares.

## DynamoDB-Toolbox

**Fonte:** [https://github.com/jeremydaly/dynamodb-toolbox](https://github.com/jeremydaly/dynamodb-toolbox)

![](https://miro.medium.com/max/382/1*4lS-5aZ2E4YT4o5E7-X3Mg.png)

DynamoDB-Toolbox é uma abstração para o DynamoDB AWS SDK. Ele fornece uma API muito mais amigável ao desenvolvedor. Para configurar esse pacote, você precisa definir classes JavaScript que representam cada uma das várias entidades armazenadas no DynamoDB. Essas classes fornecem a equipe de desenvolvimento funções dedicadas de leitura e gravação. Acho isso particularmente útil ao lidar com entidades que contêm chaves compostas, pois a construção e a desconstrução da chave são processadas internamente. Também é muito conveniente ao trabalhar com um design de tabela única, pois cada atributo pode ser renomeado para a nomenclatura específica da entidade. Você nunca mais terá um `PK`ou `SK` em seu código.

O elemento que falta no DynamoDB-Toolbox são as definições do TypeScript. A biblioteca está escrita em JavaScript no momento. Ela está sendo reescrita em TypeScript no momento deste artigo, graças ao esforço de [Jeremy Daly](https://medium.com/u/611b6bf539d2?source=post_page-----c50e859a0ef0--------------------------------) . Enquanto isso, você pode usar [esse GitHub Gist](https://gist.github.com/ThomasAribart/618a3b097a29cbfb26094ed064c39041) e criar uma versão em seu projeto para desfrutar de APIs tipadas.

## Esquema JSON para TS

**Fonte:** [https://github.com/ThomasAribart/json-schema-to-ts](https://github.com/ThomasAribart/json-schema-to-ts)

![](https://miro.medium.com/max/500/1*MAVtgO0-W21StlOVkNDIrA.png)

Esquema JSON para TS é um ótimo pacote desenvolvido por [Thomas Aribart](https://medium.com/u/b708b8c5a024?source=post_page-----c50e859a0ef0--------------------------------) , não necessariamente dedicado a projetos serverless, mas muito útil em algumas situações específicas. Ele gera estaticamente tipos de TypeScript a partir de definições de esquema JSON. O esquema JSON é amplamente usado no mundo serverless: o API Gateway v1 o usa para realizar a validação de entrada e o EventBridge também o usa no registro do esquema. Ao escrever manipuladores TypeScript, você só precisa definir um esquema de evento uma **vez** usando a sintaxe do esquema JSON. A base de código de seus manipuladores pode se beneficiar dos tipos TypeScript sem criar uma definição real do TypeScript. Isso garante que você esteja sempre atualizado e nunca duplique suas definições entre várias sintaxes.

## Definições de arquivo de serviço em Typescript com Serverless Framework

Este é um projeto que comecei recentemente com o objetivo de facilitar a adoção de arquivos de serviço em TypeScript e suas vantagens. Usar um arquivo `serverless.ts` ao invés do tradicional `serverless.yml` ou `serverless.json`, capacita sua equipe de desenvolvimento com dicas do IDE sobre os **tipos de** configuração esperados ao modificar este arquivo. Também permite **importações** mais fáceis de outros arquivos, bem como **referências** semanticamente válidas . Você pode aprender mais lendo [o artigo que escrevi detalhando essas vantagens](https://medium.com/serverless-transformation/infrastructure-as-code-only-works-as-code-a8f0072b29cf). As definições do arquivo de serviço do Serverless Framework TypeScript é um pequeno pacote, atualizado automaticamente pela versão do Serverless Framework, contendo tipos validando a lógica interna do framework. Isso garante que você sempre use a configuração que o Serverless Framework espera! Se você quiser saber mais sobre este pacote, dê uma olhada na [explicação detalhada que publiquei](https://medium.com/serverless-transformation/serverless-service-file-typescript-definitions-will-never-be-outdated-again-f5c0d2a80e95) .

## TypeBridge

**Fonte:** [https://github.com/fredericbarthelet/typebridge](https://github.com/fredericbarthelet/typebridge)

TypeBridge é uma pequena biblioteca que comecei, atualmente em beta, muito semelhante ao DynamoDB-Toolbox. Tem como escopo os eventos e _event bus_ do EventBridge e ajuda você em:

- Definir programaticamente os eventos personalizados do seu aplicativo com o esquema JSON
- Fornece a você uma publicação de eventos orientado a negócios e APIs tipadas pelo consumidor, validando suas mensagens e eventos consumidos com o esquema JSON
- Chamadas `putEvents` em lote automaticamente ao publicar mais de 10 eventos por vez
- Verifique o tamanho do evento antes de publicar

Ele aproveita as definições do esquema JSON de seus eventos e calcula estaticamente as definições do TypeScript a serem usadas nas APIs de consumo.

## AWS Lambda Power Tuning

**Fonte:** [https://github.com/alexcasalboni/aws-lambda-power-tuning](https://github.com/alexcasalboni/aws-lambda-power-tuning)

![](https://miro.medium.com/max/500/1*c8HqmLGpeW55iFyp-HZTZw.png)

O AWS Lambda Power Tuning é o estranho nesta lista, mas é incrível! Esta ferramenta, desenvolvida por [Alex Casalboni,](https://medium.com/u/56b578970060?source=post_page-----c50e859a0ef0--------------------------------) visa otimizar a configuração de memória da sua lambda de forma a reduzir os custos de execução. Não está necessariamente vinculado aos tempos de execução do Node.js. Não faz parte do seu código de produção, mas é usado como uma ferramenta de ambiente de trabalho que pode ser usada em um determinado momento ou ser parte de um processo de CI. A cobrança do Lambda é calculado usando o tamanho da memória e a duração. Embora um lambda de 1024 MB custe o dobro do preço de um de 512 MB em execução pelo mesmo período de tempo, se o código do manipulador for executado 3 vezes mais rápido no lambda de 1024 MB, sua carteira ficará feliz com esta configuração. Essa otimização se torna ainda mais interessante com a recente introdução do [cobrança em granularidade de 1ms para lambda](https://aws.amazon.com/fr/blogs/aws/new-for-aws-lambda-1ms-billing-granularity-adds-cost-savings/), onde cada ms conta (ao invés da granularidade de 100 ms anterior, tornando a otimização da duração da lambda menos preocupante).

Se você gostaria de saber mais sobre outras bibliotecas, você pode encontrar uma lista muito mais ampla de ferramentas serverless no [incrível repositório Github Serverless do Anibal](https://github.com/anaibol/awesome-serverless).

---

# Créditos

- [The 6 Top Libraries for AWS Serverless TypeScript Developers](https://medium.com/serverless-transformation/the-awesome-libraries-of-aws-typescript-serverless-developers-c50e859a0ef0) escrito originalmente por [Frédéric Barthelet](https://twitter.com/bartheletf).
