# Qual tipo de índice secundário do DynamoDB você deve escolher?

# Use o poder dos índices secundários locais e globais.

---

# Créditos

- Escrito originalmente por [Pete Naylor](https://twitter.com/pj_naylor), em [Which flavor of DynamoDB secondary index should you pick?](https://www.gomomento.com/blog/which-flavor-of-dynamodb-secondary-index-should-you-pick).

---

Modelagem de dados em Amazon DynamoDB, séries de artigos:

1. [What really matters in DynamoDB data modeling?](https://www.gomomento.com/blog/what-really-matters-in-dynamodb-data-modeling)
2. Qual tipo de índice secundário do DynamoDB você deve escolher? **(VOCÊ ESTÁ AQUI)**
3. Custos de projeção de índice e perigos de GSIs sobrecarregados (em breve)

Este é o segundo artigo de uma série curta sobre modelagem de dados no Amazon DynamoDB (sem os equívocos de "design de tabela única"). Se você não leu o primeiro artigo, eu o encorajo [a dar uma olhada](https://www.gomomento.com/blog/what-really-matters-in-dynamodb-data-modeling). Nele, discuto os conceitos mais importantes da modelagem do DynamoDB: **flexibilidade de esquema** e **coleções de itens**. Vou us essa base para discutir índices secundários, um dos meus recursos favoritos do DynamoDB. Observe que este artigo assume alguma familiaridade com os principais conceitos do DynamoDB, como partições, itens, tipos de chave primária e tipos de dados. Se você precisa se atualizar sobre eles, aqui está uma [lista de vídeos que eu recomendo](https://www.youtube.com/playlist?list=PLJo-rJlep0EDNtcDeHDMqsXJcuKMcrC5F).

Índices secundários são poderosos! Você pode usá-los para fornecer automaticamente diferentes perspectivas para leituras dos dados em sua tabela. Eles ajudam você a definir e manter relacionamentos adicionais (coleções de itens) dentro dos mesmos dados, permitem que você classifique os dados relacionados com uma dimensão diferente e podem criar filtros incrivelmente eficazes.

Mas, como qualquer outra ferramenta, é possível utilizar os índices secundários do DynamoDB de um modo ruim.

Com grandes poderes vêm grandes responsabilidades - e uma grande necessidade de entender suas opções! Neste blog, vou me concentrar em comparar os diferentes tipos de índices secundários e oferecer algumas dicas para escolher entre eles. Deixarei de falar sobre escolhas de projeção de índice secundário e as consequências de custo e escalabilidade para o próximo artigo, além de algumas observações sobre uma tendência recente no uso de índices secundários globais do DynamoDB chamada "sobrecarga" (_overloading_) e por que ela deve ser evitada no design das suas tabelas na maioria dos casos.

Então fique ligado! E segurem seus chapéus - será uma turnê alucinante.

## Onde vivem os índices secundários? E como eles chegam lá?

Os dados de sua tabela (o índice primário) são projetados em cseus índies secundários pelo DynamoDB com base na definição de índice que você fornece. Você não pode escrever diretamente em um índice secundário, mas ao escrever em um item em sua tabela base, o DynamoDB projetará alterações relevantes em seus índices secundários para você. Existem dois tipos de índice secundário: índices secundários locais (LSIs - Local Secondary Indexes) e índices secundários globais (GSIs - Global Secondary Indexes).

## Índices secundários locais

Os LSIs residem nas mesmas partições do DynamoDB que a tabela base — eles compartilham o mesmo atributo de chave de partição (mas têm um atributo de chave de classificação diferente) e compartilham a taxa de transferência com a tabela base. Os LSIs são locais porque oferecem uma ordem de classificação diferente para uma coleção de itens dentro da mesma partição. Os LSIs oferecem suporte a **leituras fortemente consistentes após escrever na tabela** base, se o parâmetro for especificado na hora da solicitação de dados (caso contrário, a consistência eventual padrão é usada).

Na verdade, os LSIs e as tabelas base compartilham as mesmas coleções de itens — e isso restringe cada coleção de itens a residir em uma única partição. Quando uma tabela tem um ou mais LSIs, cada coleção de itens nunca pode crescer além de aproximadamente 10 GB (todos os dados para o mesmo valor da chave de partição na tabela base e todos os LSIs combinados). A taxa de transferência de leitura e escrita para qualquer coleção de itens é limitada a 3.000 unidades de leitura por segundo e 1.000 unidades de escrita por segundo na tabela e todos os LSIs associados.

Os LSIs **devem ser definidos quando a tabela é criada** e não podem ser excluídos sem excluir a tabela base associada. **Pense bem antes de usar LSIs em seu modelo de dados** - você deve ter um bom motivo (como um requisito válido para leituras fortemente consistentes) e deve saber que nunca terá uma coleção de itens que possa crescer e exigir mais de 10 GB, 3.000 unidades de leitura por segundo ou 1.000 unidades de escrita por segundo.

> Se você decidir posteriormente que não quer ser limitado pelas propriedades dos LSIs ou não precisa mais de um LSI específico, pode ser necessária uma migração complexa de sua existente tabela para uma tabela de substituição/nova tabela.

## Índices secundários globais

Os GSIs são como uma tabela separada — eles podem ter um atributo de chave de partição diferente e têm suas próprias partições e capacidade de taxa de transferência. Eles podem ser criados (com preenchimento/_with backfill_) conforme necessário e removidos (sem custo) quando não forem mais necessários. Eles são globais porque permitem que novos relacionamentos (coleções de itens) sejam definidos entre itens em todas as partições da tabela base. As coleções de itens em um GSI podem abranger partições para armazenar mais dados e fornecer maior taxa de transferência (também verdadeiro para a tabela base, desde que não haja LSIs).

Uma das maiores diferenças entre LSIs e GSIs está em seu comportamento durante a escrita na tabela base. Como a tabela base e quaisquer LSIs compartilham as mesmas partições, quaisquer atualizações nos LSIs são manipuladas atomicamente com a alteração no item da tabela base. Para GSIs, a **alteração é propagada de forma assíncrona** para uma partição diferente. Isso tem implicações de consistência de leitura após escrita. As leituras de um LSI podem ser solicitadas para serem consistentes, se desejado, mas **uma leitura de um GSI é sempre eventualmente consistente**.

Uma consideração adicional para leituras de um GSI é a [monotonicidade](https://blog.palantir.com/on-monotonicity-in-relational-databases-and-service-oriented-architecture-90b0a848dd3d). As leituras GSI não são monotônicas. Se você atualizar um item em sua tabela base para incrementar o valor de um atributo de 7 para 8, poderá fazer três leituras sucessivas dos dados projetados em seu GSI e ver o valor como 8 primeiro, depois 7 e, finalmente, de volta para 8. Uma série de leituras para os mesmos dados em um GSI pode retornar resultados que avançam e retrocedem ao longo do tempo. As leituras fortemente consistentes da tabela ou de um LSI são monotônicas.

Os GSIs são mais flexíveis que os LSIs, e qualquer LSI pode ser facilmente modelado como um GSI. Use LSIs apenas se tiver certeza de que o índice precisará suportar leituras consistentes/monotônicas ou se quiser se beneficiar do recurso de "leitura completa" (_read through_) (detalhes sobre isso em um artigo futuro).

## Propriedades interessantes de chaves de índice secundárias

Em primeiro lugar: o valor da chave de índice não é garantido como único, como a chave primária em sua tabela base. O índice pode ter várias entradas projetadas com os mesmos valores para a chave de partição do índice e a chave de ordenação! A API GetItem não tem suporte para índices secundários porque GetItem implica a leitura de no máximo um item para um determinado valor da chave. Mas em um índice secundário, mesmo quando um valor específico da chave de índice é fornecido, você pode ver muitos itens retornados! Você deve usar Query ou Scan para ler de um LSI ou GSI. LSIs fornecem uma ordenação alternativa (_sort key_), GSIs fornecem coleções alternativas e ordenação (opcional).

Conforme mencionado anteriormente, um LSI deve ter uma chave composta. A tabela base à qual o LSI está anexado também deve ter uma chave composta. A chave de partição do LSI deve ser a mesma da tabela base e o atributo da chave de ordenação deve ser diferente daquele da tabela. Um exemplo simples pode ser uma interface do usuário que lista um conjunto de entradas em formato tabular — as entradas são agrupadas (coleções de itens relacionadas pelo mesmo valor do atributo chave de partição) e ordenadas por um dos valores de seus atributos (a chave de ordenação da tabela base). E se o usuário precisar ordenar o mesmo grupo de entradas por um atributo diferente? É aqui que um LSI pode ajudá-lo.

![Um modelo de dados de carrinho de compras usando LSIs para ordernar a lista com um atributo diferente.](https://assets.website-files.com/628fadb065a50abf13a11485/63d2b98a06650352608c9d57_Shopping-Carts-Sort-Table-2s-v2.gif)

GSIs são mais flexíveis: uma chave de partição é necessária, mas pode ser diferente da tabela base - você pode relacionar/agrupar os dados da sua tabela em um atributo diferente! Uma GSI pode ter uma chave simples ou uma chave composta — e o mesmo vale para a tabela base à qual está anexada. Se você não precisa recuperar coleções de itens de seu GSI ordenados (talvez o padrão seja apenas consultar todos os itens que têm um valor comum de chave de partição no índice), não defina uma chave de ordenação - a GSI ainda pode construir coleções de itens para uma busca de dados eficiente. **Definir uma chave de ordenação quando não é necessário pode limitar sua escalabilidade para as coleções de itens.**

# Detalhando distinções entre tipos de índices secundários

Há muitas nuâncias aqui, mas na verdade se resumem a algumas diferenças simples, então fiz uma folha de dicas que você pode usar na próxima vez que estiver considerando suas opções de índice secundário.

![Tabela mostrando as nuâncias dos tipos de índices secundários do DynamoDB](https://assets.website-files.com/628fadb065a50abf13a11485/63d2b9fbcfc70e60ff8707f3_DynamoDB%20Secondary%20Index%20Type%20Cheat%20Sheet%20V2.png)

---

Fique de olho no próximo artigo desta série de modelagem de dados do DynamoDB (que visa detalhar o "design de tabela única").

Em artigos futuros: mais sobre indexação secundária e uma discussão resumida sobre onde o "design de tabela única" deu terrivelmente errado.

Se você quiser discutir esse tópico comigo, obter minha opinião sobre uma pergunta de modelagem de dados do DynamoDB que você tem ou sugerir tópicos para eu escrever em artigos futuros, entre em contato comigo no Twitter ([@pj_naylor](https://twitter.com/pj_naylor)) ou envie-me um [e-mail diretamente]mailto:petenaylor@momentohq.com)!
