# Escalando DynamoDB: como partições, chaves de partição e divisão de partições quentes impactam performance (Parte 1: Carregando dados)

A regra geral com o [Amazon DynamoDB](https://aws.amazon.com/dynamodb/) é escolher uma chave de partição de alta cardinalidade. Mas por que; e o que acontece se você não fizer isso? Inspirados por um caso de uso de cliente, nos aprofundamos nessa questão e exploramos o desempenho de carregamento e consulta do DynamoDB usando diferentes designs de chave de partição e configurações de tabela.

Após cada experimento, examinamos o gráfico de desempenho gerado, explicamos os padrões que vemos e, por meio de repetidas iterações de aprimoramento, mostramos a você os fundamentos internos do DynamoDB e as práticas recomendadas para escrever aplicativos de alto desempenho. Cobrimos partições de tabela, valores de chave de partição, partições quentes, divisão de partições quentes, capacidade de estouro (_burst capacity_) e limites de taxa de transferência em nível de tabela.

Esta série de três partes começa apresentando o problema que iremos explorar e analisando as estratégias de carregamento de dados e o comportamento do DynamoDB durante execuções de curta duração. A [Parte 2](https://aws.amazon.com/blogs/database/part-2-scaling-dynamodb-how-partitions-hot-keys-and-split-for-heat-impact-performance/) aborda o desempenho da consulta e o comportamento adaptativo do DynamoDB durante a atividade sustentada. A série termina com a [Parte 3](https://aws.amazon.com/blogs/database/part-3-scaling-dynamodb-how-partitions-hot-keys-and-split-for-heat-impact-performance/), que é um resumo dos aprendizados e das melhores práticas.

A AWS estimou que a solução que estamos prestes a explorar seria não apenas massivamente escalável, mas custaria cerca de US$ 0,18 para carregar os itens, US$ 0,05 por mês em armazenamento contínuo e, com o modo sob demanda totalmente flexível, eles poderiam fazer oito milhões de pesquisas por $1. Começaremos com uma carga lenta e, no final, estaremos processando milhões de solicitações por segundo com uma latência média inferior a 2 milissegundos.

# Caso de uso do cliente: pesquisa de endereço IP

Esta postagem foi inspirada por uma pergunta de cliente sobre a importância das chaves de partição de alta cardinalidade. Muitos casos de uso do DynamoDB têm uma chave de partição de alta cardinalidade natural e óbvia (como um ID de cliente ou ID de ativo). Não neste caso. Accenture Federal Services entrou em contato conosco e disse que queria projetar um serviço de pesquisa de metadados de endereço IP. Seu conjunto de dados consistia em centenas de milhares de intervalos de endereços IP, cada intervalo tendo um endereço inicial (como 192.168.0.0), um endereço final (como 192.168.10.255) e metadados associados (como proprietário, país, regras de segurança para aplicar, e assim por diante). Sua consulta precisava aceitar um endereço IP, encontrar o intervalo que o continha e retornar os metadados.

Accenture Federal Services queria saber:

- Quão bem o DynamoDB funcionaria para este serviço de pesquisa.
- Qual design de tabela funcionaria melhor.
- Qual projeto seria mais eficiente em termos de tempo de execução, custo e escalabilidade.
- Qual seria o máximo teórico de pesquisas por segundo que o DynamoDB poderia atingir.
- Eles também estavam preocupados com o fato de seu caso de uso não parecer um caso de uso clássico do DynamoDB, porque não havia uma chave de partição óbvia. Eles queriam saber se isso limitaria o desempenho.

Nesta postagem, trabalharemos com o problema e responderemos a essas perguntas.

# O algoritmo de pesquisa

Antes de fazer qualquer design de tabela, é importante ter o algoritmo básico. Ele deve aceitar um endereço IP e localizar com eficiência o intervalo que contém o endereço.

Um esquema de tabela do DynamoDB tem uma chave de partição e uma chave de ordenação opcional. Atributos adicionais podem estar presentes, mas não são indexados (a menos que sejam colocados em um índice secundário).

Se você ainda não estiver familiarizado com chaves de partição e chaves de ordenação, recomendo [aprender primeiro os fundamentos do DynamoDB](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/GettingStartedDynamoDB.html). Se você preferir vídeo, sugiro [DynamoDB: sua finalidade, principais recursos e conceitos-chave](https://www.youtube.com/watch?v=ummOosOW4lE) e [DynamoDB: sob o capô, gerenciando a taxa de transferência, padrões de design avançados](https://www.youtube.com/watch?v=0iGR8GnIItQ).

Uma maneira de extrair dados do DynamoDB é executar uma operação `Query`. UMA `Query` pode fazer muitas coisas, uma delas é recuperar itens onde a chave de partição é especificada exatamente e a chave de ordenação é especificada com uma desigualdade (onde a chave de partição é igual a X e a chave de ordenação não é igual a Y). Podemos usar uma `Query` para procurar um IP em um intervalo de IP, desde que duas coisas sejam verdadeiras para os dados:

- Os intervalos de IP no conjunto de dados nunca se sobrepõem. Isso é inerentemente verdadeiro, caso contrário, o conjunto de dados seria ambíguo (porque o mesmo IP teria várias entradas de metadados).
- Os intervalos de IP descrevem totalmente todos os endereços IP, sem lacunas. Isso nem sempre é verdade, porque alguns intervalos de IP são especialmente reservados e não possuem metadados. No entanto, intervalos sintéticos podem preencher as lacunas com um valor indicando que não há metadados disponíveis, para tornar essa suposição verdadeira.

Cada intervalo de IP pode ser armazenado como um item (linha) no DynamoDB. Por enquanto, vamos supor um único valor de chave de partição fixo (um que seja o mesmo para todos os itens) e um valor de chave de ordenação igual ao endereço IP inicial do intervalo. A figura 1 a seguir mostra o modelo de dados. (Melhoraremos esse modelo de dados mais tarde.)

![Figura 1: modelo de dados usando uma chave de partição fixa ](https://d2908q01vomqb2.cloudfront.net/887309d048beef83ad3eabf2a79a64a389ab1c9f/2023/01/13/dbblog-2539-image001.png)

A consulta pode localizar itens em que a chave de partição é igual a 0 e a chave de ordenação é menor ou igual ao endereço IP de pesquisa, escaneando o valor para trás e [limitando a um resultado](https://docs.aws.amazon.com/amazondynamodb/latest/APIReference/API_Query.html#API_Query_RequestSyntax). O primeiro item correspondente são os metadados a serem retornados. O valor do intervalo final não precisa ser considerado, porque sabemos que não há sobreposições ou lacunas. A seguir é o código Python para uma função do [AWS Lambda](http://aws.amazon.com/lambda) mostrando uma execução de teste:

```python
import json
import boto3
from boto3.dynamodb.conditions import Key

client = boto3.resource('dynamodb')
table = client.Table('IPAddressRanges')

def lambda_handler(event, context):
    pk = '0'
    ip = event.get('ip')
    response = table.query(
        KeyConditionExpression=Key('PK').eq(pk) & Key('SK').lte(ip),
        ScanIndexForward = False,
        Limit = 1
    )
    print(response['Items'][0])
```

Se você está tentando imaginar como esse algoritmo funciona, pode ser útil visualizá-lo. Imagine uma cerca no campo com muitos postes colocados em intervalos aleatórios. Você deseja descobrir qual segmento da cerca corresponde a qualquer distância específica que você pode caminhar ao longo da cerca.

Para resolver isso, você começa percorrendo essa distância e, em seguida, anda para trás até encontrar um poste. Nesse poste da cerca estão todos os metadados sobre esse segmento. Cada poste da cerca descreve o segmento que o segue. O poste da cerca do outro lado contém os metadados sobre o segmento que o segue. Se houver uma lacuna na cerca, o poste da cerca logo antes da lacuna diz "tem uma lacuna aqui".

Mas espera... apenas um valor de chave de partição torna a consulta simples, mas vai diretamente contra as [práticas recomendadas de ter alta cardinalidade entre os valores de chave de partição](https://aws.amazon.com/blogs/database/choosing-the-right-dynamodb-partition-key/) no DynamoDB. Vamos testar as implicações disso.

# O formato de dados

Para executar um teste, devemos primeiro considerar o formato dos dados reais. Os dados de origem estão em um arquivo CSV, um intervalo de IP por linha, sem lacunas, sem sobreposições. Cada linha tem valores de IP inicial e final, bem como metadados. O seguinte é um exemplo simulado com espaço em branco adicionado para facilitar a leitura:

```text
Netblock     , start     , end         , metadata
1.0.0.0/24   , 1.0.0.0   , 1.0.0.255   , Range 1.0.0.0/24 metadata
1.0.1.0/24   , 1.0.1.0   , 1.0.1.255   , Range 1.0.1.0/24 metadata
1.0.2.0/23   , 1.0.2.0   , 1.0.3.255   , Range 1.0.2.0/23 metadata
1.0.4.0/22   , 1.0.4.0   , 1.0.7.255   , Range 1.0.4.0/22 metadata
1.0.8.0/21   , 1.0.8.0   , 1.0.15.255  , Range 1.0.8.0/21 metadata
1.0.16.0/20  , 1.0.16.0  , 1.0.31.255  , Range 1.0.16.0/20 metadata
1.0.32.0/19  , 1.0.32.0  , 1.0.63.255  , Range 1.0.32.0/19 metadata
1.0.64.0/18  , 1.0.64.0  , 1.0.127.255 , Range 1.0.64.0/18 metadata
1.0.128.0/17 , 1.0.128.0 , 1.0.255.255 , Range 1.0.128.0/17 metadata
```

É tentador usar a string IP inicial como a chave de ordenação, mas é perigoso classificar as strings como se fossem números. Você verá que 0,100 está entre 0,1 e 0,2. Uma solução é zerar os valores para que 1.0.32.0 se torne 001.000.032.000. Se cada número tiver sempre três dígitos, strings e números serão classificados de forma idêntica.

Existe uma maneira melhor: converter cada endereço IP em seu formato numérico natural. Os endereços IP quase sempre são escritos em um formato quad com pontos, como 1.0.32.0, mas essa é apenas uma serialização amigável. Fundamentalmente, um endereço IP (pelo menos IPv4) é um valor único de 32 bits.

O endereço IP 1.0.32.0 quando serializado para binário (mas mantendo os pontos para legibilidade) é 00000001.00000000.00100000.00000000, que se transmitido como um único decimal é 16.785.408. Esse é o modelo de numeração que usamos para a chave de ordenação porque é simples e eficiente para a operação de desigualdade, além de ser [compacto no armazenamento](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/CapacityUnitCalculations.html).

A figura 2 a seguir mostra um fragmento de como a tabela ficará depois de converter os endereços IP em seus valores numéricos.

![Figura 2: Tabela mostrando os endereços IP convertidos em seus valores numéricos](https://d2908q01vomqb2.cloudfront.net/887309d048beef83ad3eabf2a79a64a389ab1c9f/2023/01/13/dbblog-2539-image002.png)

# Carregando dados

Agora estamos prontos para começar o teste, começando com o desempenho de carregamento de dados. Carregamos o arquivo CSV usando um script Python simples de loop único. Observe que todos os resultados dos testes foram coletados na região `us-east-1`.

> Teste de carregamento: tabela sob demanda, CSV sequencial, usando uma única chave de partição.

Como primeiro teste, vamos realizar um carregamento em massa em uma tabela sob demanda recém-criada. Carregamos do arquivo CSV e fazemos com que nosso código Python atribua a mesma chave de partição para todos os itens e converta strings de intervalo de IP em valores numéricos. Observe que os dados do arquivo CSV são classificados por intervalo de IP, com IPs mais baixos no início. Esse tipo de ordenação é comum com arquivos CSV, o que será importante posteriormente no experimento. A Figura 3 a seguir mostra os resultados.

![Figura 3: Resultados do primeiro teste de carga: tabela sob demanda, CSV sequencial, chave de partição única](https://d2908q01vomqb2.cloudfront.net/887309d048beef83ad3eabf2a79a64a389ab1c9f/2023/01/13/dbblog-2539-image003.png)

Conforme mostrado na Figura 3, o primeiro teste de carga é executado a uma taxa constante de 1.000 unidades de escrita por segundo. O restante das solicitações de escrita estão sendo _throttled_.

Aqui é o que está acontecendo nos bastidores: Cada tabela do DynamoDB é distribuída em algumas partições físicas. Cada partição física pode suportar 3.000 unidades de leitura por segundo e 1.000 unidades de escrita por segundo. Ao usar apenas um valor de chave de partição, todas as escritas são enviadas para uma partição e isso cria um gargalo.

Isso não significa que novas tabelas sob demanda alocam apenas uma partição. As tabelas sob demanda se adaptam continuamente ao tráfego ao vivo e as tabelas sob demanda recém-criadas são documentadas como capazes de atender até 4.000 unidades de solicitação de escrita e 12.000 unidades de solicitação de leitura. Para obter mais informações, consulte a documentação em [modo de capacidade de leitura/escrita](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/HowItWorks.ReadWriteCapacityMode.html).

Esses recursos de taxa de transferência se alinham exatamente com o que uma tabela com quatro partições faria. Portanto, mesmo com quatro partições disponíveis, como estamos usando um único valor de chave de partição, todo o tráfego está sendo alocado para uma dessas partições, deixando as outras três inativas.

Uma atualização rápida sobre como os dados são atribuídos às partições: Cada partição é responsável por um subconjunto do espaço de chaves da tabela, semelhante a intervalos de números em uma linha numérica muito grande. O valor da chave de partição _hasheado_, que o converte em um número, e a partição cujo intervalo inclui esse número obtém a leitura ou escrita desse valor de chave de partição. Se os valores da chave de partição forem sempre os mesmos, a mesma partição geralmente obterá todas as leituras e escritas. Observe que diferentes chaves de partição podem fazer hash para a mesma área geral do intervalo numérico e, nesse caso, serão colocadas na partição que lida com esse intervalo. Uma partição pode ser dividida, movendo seus itens para duas novas partições e introduzindo um novo ponto de divisão no intervalo numérico. As partições também podem ser divididas em uma coleção de itens (entre itens com o mesmo valor de chave de partição), que no caso, a chave de ordenação é considerada no cálculo para o ponto de divisão.

Veremos isso se aplicar em breve, mas a conclusão é que o uso de um único valor de chave de partição colocará inicialmente todos os itens na mesma partição, o que pode criar um gargalo grande nas escritas causando _throttling_ nas operações.

# Teste de carregamento: tabela sob demanda, CSV sequencial e várias chaves de partição

Podemos acelerar o carregamento de dados se espalharmos os dados por mais valores de chave de partição. Talvez podemos usar o valor da chave de partição que é a primeira parte do endereço IP. Por exemplo, 192.168.0.0 obtém uma chave de partição de 192. Isso distribui as gravações em mais de 200 valores de chave de partição e deve distribuir o trabalho de maneira mais uniforme entre as partições.

Temos que ajustar nossa consulta para especificar a chave de partição correta com base no intervalo de IP que está sendo pesquisado e nosso código Python garantir que nenhum intervalo cruze os limites da chave de partição, porque, se isso acontecer, a lógica principal falhará. O novo design da tabela se parece com a Figura 4 a seguir.

![Figura 4: design da tabela mostrando chaves de partição distintas para cada intervalo de endereços /8 ](https://d2908q01vomqb2.cloudfront.net/887309d048beef83ad3eabf2a79a64a389ab1c9f/2023/01/13/dbblog-2539-image004.png)

A Figura 5 a seguir mostra o desempenho durante o carregamento.

![Figura 5: Resultados do segundo teste de carga: agora com muitas chaves de partição](https://d2908q01vomqb2.cloudfront.net/887309d048beef83ad3eabf2a79a64a389ab1c9f/2023/01/13/dbblog-2539-image005.png)

O carregamento de dados melhorou para cerca de 1.250 gravações por segundo. Por que não melhorou mais e por que o trabalho não é melhor distribuído pelas partições? É porque o arquivo CSV é sequencial. Todos os milhares de intervalos para cada valor de chave de partição vão um após o outro. Uma partição leva todo o tráfego por um tempo, depois outra, depois outra. Não está bem espalhado. A única melhoria ocorre quando o valor da chave de partição muda e essa nova partição faz _throttling_ nas operações de escrita só dessa partição.

A conclusão aqui é que CSV sequenciais não espalham bem o tráfego.

# Teste de carregamento: tabela sob demanda, CSV aleatório e várias chaves de partição

Se randomizarmos a ordem das entradas do arquivo CSV (ajustando o arquivo ou fazendo um embaralhamento como uma tarefa interna no código do Python), podemos distribuir a carga de maneira mais uniforme e obter um aumento de desempenho. A Figura 6 a seguir mostra o resultado do nosso teste.

![Figura 6: Resultados do terceiro teste de carga: agora com um arquivo CSV de linha aleatória](https://d2908q01vomqb2.cloudfront.net/887309d048beef83ad3eabf2a79a64a389ab1c9f/2023/01/13/dbblog-2539-image006.png)

Este teste terminou em menos de 2 minutos. A primeira e a última marcação de tempo são minutos parciais sobre os quais não podemos inferir uma taxa por segundo. Durante o minuto central, a medição tota atingiu cerca de 3.600 gravações por segundo. Isso combina bem com quatro partições sendo bem utilizadas. O simples ato de randomizar a ordem de escrita de dados melhorou muito a taxa de transferência de escrita.

Por esses motivos, você deve randomizar carregamentos de CSV sempre que o arquivo CSV tiver linhas classificadas por (e, portanto, agrupadas por) valor de chave de partição.

Também é útil randomizar os dados de origem antes de carregar se os dados de origem vierem de um `Scan` de outra tabela do DynamoDB, porque os itens retornados pelo `Scan` são naturalmente agrupados por valores de hash de chave de partição e, portanto, criariam uma linha constante de calor nessa chave durante o carregamento.

# Teste de carregamento: tabela provisionada, CSV aleatório e várias chaves de partição

Todos os testes até agora foram em tabelas sob demanda recém-criadas. Vamos testar uma tabela provisionada em 10.000 unidades de capacidade de escrita (WCUs) (para simplificar, não ativaremos o dimensionamento automático). Continuaremos usando o arquivo CSV aleatório. A Figura 7 a seguir mostra o que observamos.

![Figura 7: Resultados do quarto teste de carga: agora usando uma tabela provisionada em 10.000 WCUs](https://d2908q01vomqb2.cloudfront.net/887309d048beef83ad3eabf2a79a64a389ab1c9f/2023/01/13/dbblog-2539-image007.png)

O carregamento foi concluído em menos de um minuto. Toda a atividade foi reunida em um ponto de dados de subminuto com uma média de 6.000 escritas por segundo, o que significa que o pico durante aquele minuto foi bem acima disso.

Uma tabela recém-provisionada com 10.000 WCUs tem mais partições do que uma nova tabela sob demanda (ela precisaria de pelo menos 10 partições para lidar com essa carga de escrita), e por ter muitos valores de chave de partição espalhados por essas partições extras, conseguimos melhorar o desempenho.

Isso significa que provisionado é melhor do que sob demanda? Não, porque o on-demand se [ajusta rapidamente e aumenta a capacidade e as partições ao longo do tempo](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/HowItWorks.ReadWriteCapacityMode.html) com base no tráfego que ele recebe. Acontece que o tamanho padrão de uma tabela sob demanda é inferior a 10.000 WCUs. Se você espera que uma tabela receba uma alta carga de tráfego desde o início, é uma boa ideia pré-aquecer uma nova tabela sob demanda com uma capacidade inicial especificada criando primeiro a tabela como provisionada e, em seguida, alternando-a para sob demanda.

# Teste de carregamento: resumo

A taxa máxima de carregamento depende do número de partições físicas, do número de valores de chave de partição e da capacidade do carregamento de paralelizar o trabalho nas partições. Ter mais partições tende a aumentar a taxa de carregamento, mas mais partições é mais benéfico quando há valores de chave de partição suficientes para distribuir o trabalho entre as partições e a lógica de carregamento também distribui o trabalho entre os valores de chave de partição.

Na Parte 2, exploraremos o desempenho da consulta e o comportamento adaptativo que o DynamoDB exibe durante um período de tráfego de operações.

# Conclusão

Nesta série de três partes, você aprendeu sobre os componentes internos do DynamoDB revisando os resultados do teste de desempenho de carregamento e consulta de dados usando diferentes designs de chave de partição. Discutimos partições de tabela, valores de chave de partição, partições quentes, divisão de partições quentes, capacidade de estouro (_burst capacity_) e limites de taxa de transferência em nível de tabela.

Como sempre, você está convidado a deixar perguntas ou feedback nos comentários.

---

# Créditos

- Escrito originalmente por [Jason Hunter e Vivek Natarajan](https://aws.amazon.com/blogs/database/part-1-scaling-dynamodb-how-partitions-hot-keys-and-split-for-heat-impact-performance/), em [Scaling DynamoDB: How partitions, hot keys, and split for heat impact performance (Part 1: Loading)](https://aws.amazon.com/blogs/database/part-1-scaling-dynamodb-how-partitions-hot-keys-and-split-for-heat-impact-performance/)
