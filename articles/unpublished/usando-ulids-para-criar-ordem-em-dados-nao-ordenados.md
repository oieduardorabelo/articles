# Usando ULIDs para criar ordem em dados não ordenados

_Identificadores exclusivos classificáveis ​​lexicograficamente podem ser aproveitados para consultar objetos no Amazon S3 por tempo, sem precisar armazenar metadados, veja como!_

O aumento dos armazenamentos de dados distribuídos e a decomposição geral dos sistemas em partes menores significa que a coordenação entre cada servidor, serviço ou função está menos disponível. Em meus primeiros aplicativos, a geração de ID exclusivo significava configuração `auto_increment=True` em uma coluna no banco de dados SQL. Fácil, pronto, sem problemas. Hoje, cada microsserviço tem suas próprias fontes de dados e armazenamentos NoSQL são comuns. Cada banco de dados NoSQL é "NoSQL" em sua própria maneira, mas eles geralmente evitam soluções coordenadas e de gravação única em nome da confiabilidade / desempenho / ambos. Você não pode ter uma coluna de incremento automático sem implementar a coordenação do lado do cliente.

Usar números como identificadores também cria problemas. O incremento automático pode levar a ataques baseados em enumeração. Os campos podem ter tamanhos fixos. Esses problemas podem não ser percebidos até que você transborde o campo `uint32` e agora seus registros são uma pilha de erros de conflito de ID. Em vez de inteiros, podemos usar um tipo diferente de campo de comprimento fixo e torná-lo não sequencial para que diferentes hosts possam gerar IDs sem um ponto de coordenação central.

UUIDs são uma melhoria e evitam colisões em configurações distribuídas, mas sendo estritamente aleatório, você não tem uma maneira de classificá-los facilmente ou determinar a ordem aproximada. [Segment postou um artigo há algum tempo](https://segment.com/blog/a-brief-history-of-the-uuid) sobre uma substituição de UUIDs pelo KSUID (K-Sortable Universal ID), mas tem limitações e usa um deslocamento estranho de `14e8` para evitar esgotar o tempo de época nos próximos 100 anos.

Entra o [Identificador Único Lexicograficamente Classificável (ULID)](https://github.com/ulid/spec). Esses são identificadores classificáveis ​​de alta entropia que podemos gerar em qualquer lugar em nosso pipeline sem coordenação e ter a confiança de que não haverá colisões. Um ULID se parece `01E5TZRCM5WZYPB2BH7KMYR5HT`, e os primeiros 10 caracteres são um carimbo de data / hora e os próximos 16 caracteres são aleatórios.

## E quanto ao UUID?

Eu descobri a necessidade de ULID / KSUID ao trabalhar com objetos S3 que precisavam ser nomeados, mas também queria poder consultar objetos recentes. Normalmente, quando preciso de um identificador aleatório, procuro `UUID-v4`. Por que v4?

- UUID v1 e v2 contêm endereços MAC com base no host que os gera. Este não é realmente um problema de segurança, já que um endereço L2 não ajudará muito na Internet pública. No entanto, isso significa que se meus UUIDs forem gerados em Lambdas, os endereços MAC não terão valor semântico. Não consigo usar SSH em meu Lambda e procurar o endereço MAC ou usar essas informações de outra forma.
- UUID v3 requer uma entrada, e eu usaria apenas `random.randint()` ou o equivalente para escolher meu valor de entrada. Qualquer sistema que requeira uma entrada significa que tenho que pensar sobre o que usar como entrada, como isso afeta a aleatoriedade e como isso pode afetar a segurança ou as colisões.
- O UUID v4 é aleatório, mas, como é totalmente aleatório, não fornece sobrecarga semântica.

Por que eu iria querer sobrecarregar semanticamente o UUID em meu sistema? Peguei uma dica do próprio Mago da Sobrecarga Semântica, [Rick Houlihan](https://twitter.com/houlihan_rick). Dediquei um tempo aos designs de tabela única do DynamoDB e essa forma de pensar se espalhou pelo design do meu sistema de armazenamento no Amazon S3.

## ULIDs para habilitar consultas de tempo no Amazon S3

O pensamento baseado em índices pode ser esclarecedor, especialmente porque a TI está repleta de sistemas de armazenamento classificados intrinsecamente. O Amazon S3 classifica as chaves e prefixos de seus objetos ao retorná-los, independentemente da ordem em que foram adicionados.

> As chaves são selecionadas para listagem por intervalo e prefixo. Por exemplo, considere um intervalo denominado "dicionário" que contém uma chave para cada palavra em inglês. Você pode fazer uma chamada para listar todas as chaves desse balde que começam com a letra "q". Os resultados da lista são sempre retornados na ordem binária UTF-8. - [Documentação AWS S3](https://docs.aws.amazon.com/AmazonS3/latest/dev/ListingKeysUsingAPIs.html)

O que isso significa para nosso aplicativo? Isso significa que se fornecermos chaves classificáveis ​​para S3 e as classificarmos na ordem em que realmente queremos receber os itens, poderemos colocar nossos objetos em ordem sem ter que fazer nenhuma classificação do lado do cliente. Usar um ULID em um nome de objeto (ou melhor, dividir um ULID com um prefixo) nos permite evitar colisões e também prevenir ataques relacionados a enumeração em nossos objetos.

Usar [ULIDs em Python](https://github.com/ahawker/ulid) é simples. Primeiro, você precisa instalar a biblioteca `ulid-py`, então você pode `import ulid` e começar a gerar identificadores:

Isso carregaria um objeto com apenas um ULID como nome, com o conteúdo `abc`. Então, quando listamos objetos na CLI ou em qualquer outro aplicativo, eles são classificados pelo tempo em que foram criados, mesmo que houvesse vários novos objetos em um único milissegundo.

```bash
$ aws --profile personal s3 ls s3://t10-blog-ulids
2020-04-13 21:17:53          3 01E5V474WE4DE0N63ZWT7P6YWH
2020-04-13 21:17:54          3 01E5V475QFRCEHXKJAS3BRS6BV
2020-04-13 21:24:51          3 01E5V4KXFTP52C9M5DVPQ2XR8T
2020-04-13 21:48:33          3 01E5V5Z9J0GX72VFSENBCKMHF0
```

A classificação automática é útil e, claro, os ULIDs podem ser formatados de diferentes maneiras, dependendo de suas necessidades.

```bash
>>> import ulid
>>> u = ulid.new()
>>> u.str
'01E5V7GWA9CHP337PB8SR18ZP4'
>>> u.bytes
b'\x01qvxqIdl1\x9e\xcbFp\x14~\xc4'
>>> u.int
1918360407572615930874316424782053060
>>> u.uuid
UUID('01717a42-cde2-b5be-eed8-55222c867b58')
>>> u.float
1.918360407572616e+36
>>> bin(u.int)
'0b1011100010111011001111000011100010100100101100100011011000011000110011110110010110100011001110000000101000111111011000100'
```

Especialmente útil é o tipo `u.uuid` que permite substituir UUIDs existentes em seu sistema por ULIDs sem alterar o formato do valor. Isso significa que você pode começar a se beneficiar das propriedades de ordem dos ULIDs em sistemas existentes.

## Geração Descentralizada

Porque o formato ULID de timestamp de 48 bits + aleatoriedade de 100 bits significa que obtemos 100 bits por milissegundo, o que quase elimina a chance de colisões*. Compare isso com nossa coluna numérica de incremento automático. O incremento faz com que tenhamos que centralizar o gerenciamento desse número no banco de dados para evitar conflitos de ID. Com ULIDs, podemos gerar IDs em qualquer um de nossos Lambdas, Containers ou instâncias EC2.

Como os IDs têm carimbo de data / hora nativamente, podemos tolerar partições e atrasos. Inserir dados atrasados ​​não causa problemas de ordenação porque os itens são marcados com data e hora quando o ID é gerado, e sempre podemos adicionar outro campo de data e hora na ingestão, se necessário. Os IDs nos permitem manter a ordem e inserir dados com atraso, sem precisar adicionar um processo de ingestão separado.

A geração distribuída significa que não existe um "relógio verdadeiro" que nos permite ordenar perfeitamente os itens em que colocamos ULIDs. Essa compensação entre um ponto de sincronização central (para pedidos) e maior confiabilidade / resiliência é comum em sistemas de qualquer tamanho e torna-se quase necessária em escala.

Além disso, você pode escolher sair das especificações e usar os 2 bits mais significativos do ULID que nossa codificação nos fornece. Isso é possível porque há 150 bits disponíveis na representação de texto, menos 148 usados ​​pelo carimbo de data / hora e aleatoriedade na especificação. Você pode obter 4 subtipos de ULID no mesmo espírito de IDs descritivos como `i-0123456789` e `AKIAXNMVN` fazendo com que o próprio ID contenha um tipo codificado.

*Se você for o Amazon Retail, não siga este conselho, uma em um milhão de coisas acontecem algumas vezes por hora em escala suficiente.

## ULIDs no DynamoDB

A nova tendência no DynamoDB são os designs de [tabela única](https://www.alexdebrie.com/posts/dynamodb-single-table/). Usando uma única tabela com um design que permite que diferentes GSIs atendam a várias consultas. Rick tuitou este exemplo do mundo real do serviço [Kindle Collection Rights](https://twitter.com/houlihan_rick/status/1249506999433334785) que atende a 9 consultas com 4 GSIs.

![](https://www.trek10.com/assets/kindle-table.png?mtime=20200414084529&focal=none#asset:100371:url)

Esses designs de tabela única contam com o uso de propriedades classificáveis ​​para permitir consultas, normalmente combinando as chaves `Hash` e `Range` de novas maneiras para cada tipo de objeto. Por exemplo, você pode criar uma chave como a `Hash=Org#Trek10 Range=Post#2020-04-03#ca21477c-5693-4f2d-92e5-068102b24be9` que é composta por um tipo, nome da organização, hora de criação e UUIDv4. Ao invés disso, com um ULID, você seria capaz de evitar a combinação de carimbo de data / hora e ID e usar uma chave de intervalo de `Range=Post#01E5WF8AERWH9F8PDTQ5K4GW7R`. Esta é uma representação mais eficiente que também permite usar o mesmo ID como uma chave estrangeira.

ULIDs também podem ser usados ​​para associar itens semelhantes que são criados ao mesmo tempo, manipulando os valores de aleatoriedade para serem monotônicos.

Veja este exemplo em NodeJS que cria um ULID e usa a aleatoriedade desse ULID para criar uma série de itens relacionados que serão classificados lexicamente:

```js
const monotonicFactory = require('ulid').monotonicFactory;
const ulid = monotonicFactory()

ulid(1586872590191)
'01E5WFM7VFPWCNF4DM76ADV80W'

ulid(1586872590191)
'01E5WFM7VFPWCNF4DM76ADV80X'

ulid(1586872590191)
'01E5WFM7VFPWCNF4DM76ADV80Y'

ulid(1586872590191)
'01E5WFM7VFPWCNF4DM76ADV80Z'

ulid(1586872590191)
'01E5WFM7VFPWCNF4DM76ADV810'
```

Esses ULIDs podem ser usados ​​para associar ações e eventos ou para agrupar atividades de uma tarefa ou host específico.

## Jogando Xadrez com Amazon S3

Vamos voltar ao nosso exemplo anterior S3 por um momento. Ao procurar dados em um intervalo de tempo específico, você pode reduzir significativamente o número de objetos retornados por `ListObjects`. O argumento `Delimiter` permite estreitar o intervalo de sua pesquisa em incrementos de 5-bits. Um ULID possui 10 caracteres iniciais que representam um carimbo de data / hora de 48-bits com precisão de milissegundos, com cada caractere codificando 5-bits do número.

Os timestamps de época de milissegundos de 48-bits ficarão sem espaço em 10889 DC, marque no seu calendário. O leitor astuto também notará que um valor de carimbo de data / hora de 48-bits não codifica uniformemente para 50-bits, disponíveis em uma string Crockford Base32, portanto, o carimbo de data / hora mais alto que poderá ser representado, na verdade é  `7ZZZZZZZZZ` e não `ZZZZZZZZZZ`.

```bash
t = time character
r = randomness character
ttttttttttrrrrrrrrrrrrrrrr
```

Qual é o alcance por personagem? Bem, aqui estão algumas ordens de magnitude do bit menos significativo representável em cada um.

- 1º personagem: 407226 dias
- 2º personagem: 12.725 dias
- 3º personagem: 397 dias
- 4º personagem: 12 dias, 10 horas
- 5º personagem: 9 horas, 19 minutos
- 6º personagem: 17 minutos, 28 segundos
- 7º caractere: 32 segundos
- 8º caractere: 1 segundo
- 9º caractere: 30 milissegundos
- 10º caractere: 1 milissegundo

Isso significa que, com a API `ListObjectsV2` do Amazon S3 e o parâmetro `Delimiter`, você pode obter intervalos de 17 minutos de seus dados usando o 6º caractere do ULID como seu `Delimiter`. Pegue estes objetos:

```bash
2020-04-13 21:17:54          3 01E5V475QFRCEHXKJAS3BRS6BV
2020-04-13 21:24:51          3 01E5V4KXFTP52C9M5DVPQ2XR8T
2020-04-13 21:48:33          3 01E5V5Z9J0GX72VFSENBCKMHF0
```

Podemos dividir o intervalo `01E5V5Z...` com o seguinte código:

```bash
>>> [k['Key'] for k in s3.list_objects_v2(
    Bucket='t10-blog-ulids',
    Delimiter='4',
    Prefix='01E5V4'
)['Contents']]
['01E5V475QFRCEHXKJAS3BRS6BV', '01E5V4KXFTP52C9M5DVPQ2XR8T']
​
​
>>> [k['Key'] for k in s3.list_objects_v2(
    Bucket='t10-blog-ulids',
    Delimiter='5',
    Prefix='01E5V5'
)['Contents']]
['01E5V5Z9J0GX72VFSENBCKMHF0']
```

Como esperado, as chaves são ordenadas quando são retornadas e podemos usar operadores bit a bit (também conhecidos como mágica) para mudar qualquer carimbo de data / hora ou intervalo que quisermos em uma consulta com prefixo no Amazon S3. Isso nos permite fazer filtros baseados em intervalo de tempo sem listar todos os objetos no intervalo ou usando um trabalho externo como o S3 Inventory para listar todos os nomes de objetos e carimbos de data / hora.

## Finalizando

Nesse artigo, abordamos algumas maneiras pelas quais identificadores carregados semanticamente podem ser úteis em sua camada de armazenamento. No geral, ULIDs e especificações semelhantes para identificadores classificáveis ​​são um aprimoramento do totalmente aleatório padrão do UUID. Eles podem tornar seu aplicativo mais rápido e, ao mesmo tempo, evitar colisões e ataques de enumeração, e também podem ser armazenados com mais eficiência (26 caracteres contra 36).

# Créditos

- [Leveraging ULIDs to create order in unordered datastores](https://www.trek10.com/blog/leveraging-ulids-to-create-order-in-unordered-datastores), escrito originalmente por [Ryan Scott Brown](https://twitter.com/ryan_sb).
