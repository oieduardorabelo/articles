# 10 melhores práticas para nomes de arquivos e buckets no Amazon S3

## O Amazon S3 é um serviço de armazenamento simples que oferece escalabilidade, disponibilidade de dados, segurança e desempenho que são líderes do setor. Ao nomear seus buckets do S3, siga estas práticas recomendadas para garantir um gerenciamento de dados tranquilo e eficiente.

Amazon S3 é um popular serviço de armazenamento em nuvem fornecido pela Amazon Web Services (AWS). O S3 é um armazenamento de valor-chave simples com o benefício adicional de oferecer baixa latência e alta durabilidade.

Ao trabalhar com o S3, é importante seguir as práticas recomendadas para nomear objetos armazenados nos buckets. Isso ajudará você a manter seus dados organizados e facilitará a localização do que você está procurando.

Neste artigo, discutiremos as 10 melhores práticas para nomes de arquivos e buckets no Amazon S3.

# 1. Use uma convenção de nomenclatura consistente

Quando você tem uma convenção de nomenclatura consistente, é muito mais fácil encontrar os arquivos que está procurando. Por exemplo, se todos os seus arquivos de imagem forem nomeados com a data primeiro, seguida por uma descrição da imagem, será muito mais fácil encontrar uma imagem específica do que se todos fossem nomeados aleatoriamente.

Também é importante usar uma convenção de nomenclatura consistente porque facilita a automatização de tarefas. Por exemplo, se você sabe que todos os seus arquivos de imagem são nomeados da mesma maneira, você pode facilmente escrever um script que faça backup deles automaticamente em outro local.

Por fim, usar uma convenção de nomenclatura consistente facilita o compartilhamento de arquivos com outras pessoas. Se você sabe que todos os seus arquivos são nomeados da mesma maneira, basta dizer a alguém para ir ao seu site e procurar o nome do arquivo que está procurando. Isso é muito mais fácil do que tentar explicar onde encontrar um arquivo quando os nomes estão por toda parte.

# 2. Evite caracteres especiais em nomes de buckets

Quando você cria um bucket, a Amazon atribui automaticamente a ele um endpoint de site. Este é o URL que você usa para acessar o conteúdo do seu bucket quando está usando o recurso de hospedagem de site estático do S3.

No entanto, se o nome do seu bucket contiver caracteres especiais, como pontos ou hífens, o endpoint do site não funcionará. Isso significa que qualquer pessoa que tentar visitar seu site receberá uma mensagem de erro.

Para evitar esse problema, certifique-se de que seus nomes de bucket não contenham nenhum caractere especial.

# 3. Não use pontos nos nomes dos buckets

Quando você cria um bucket com um ponto no nome, como `"meu.projeto"`, o S3 irá tratá-lo como dois buckets diferentes: `"meu"` e `"projeto"`. Isso pode causar problemas porque, ao tentar acessar o bucket `"meu.projeto"`, você pode ser redirecionado para o bucket `"projeto"`.

Para evitar esse problema, simplesmente não use pontos em seus nomes de bucket.

# 4. Os nomes dos buckets são globalmente únicos

Quando você cria um novo bucket, o Amazon S3 verifica se o nome escolhido está disponível como um subdomínio DNS (Domain Name System). Se o nome estiver disponível, o Amazon S3 atribuirá esse nome ao seu bucket. No entanto, se o nome escolhido não estiver disponível, o Amazon S3 retornará uma mensagem de erro.

Isso é importante porque significa que você poderia criar inadvertidamente dois buckets com o mesmo nome e esses buckets seriam acessíveis a partir de URLs diferentes. Por exemplo, digamos que você crie um bucket chamado `"example-bucket"` na região Leste dos EUA (N. Virgínia, `us-east-1`). Em seguida, outra pessoa cria um bucket com o mesmo nome na região da UE (Irlanda, `eu-west-1`). Embora sejam dois buckets diferentes, ambos podem ser acessados ​​usando a URL `"example-bucket.s3.amazonaws.com"`.

Para evitar esse problema, escolha um nome único para seu bucket e use o identificador de região ao criar buckets em diferentes regiões. Por exemplo, você pode nomear seu bucket "example-bucket-us-east-1" na região US East (N. Virginia) e "example-bucket-eu-west-1" na região EU (Irlanda).

# 5. Mantenha seus buckets organizados

Quando você tem muitos dados no S3, pode ser difícil acompanhar tudo sem uma estrutura organizacional clara. Ao usar uma convenção de nomenclatura consistente para seus buckets, você pode garantir que todos os seus dados sejam fáceis de localizar e gerenciar.

Uma boa maneira de organizar seus buckets é usar um prefixo ou sufixo que indique o tipo de dados armazenados em cada bucket. Por exemplo, você pode usar o prefixo `"dados-brutos"` (_raw-data_) para buckets contendo arquivos não tratados, `"dados-processados"` para buckets contendo arquivos de dados processados ​​e assim por diante.

Ao usar uma convenção de nomenclatura consistente, você pode tornar mais fácil para você e para outras pessoas encontrar e usar os dados armazenados em seus buckets do S3.

# 6. Crie e aplique políticas para o gerenciamento do ciclo de vida de objetos no S3

Os objetos no S3, por padrão, são colocados em uma classe de armazenamento _Standard_, o que significa que eles são armazenados indefinidamente, a menos que você especifique uma classe de armazenamento diferente ou exclua o objeto. No entanto, você pode não querer manter todos os objetos do S3 para sempre. Por exemplo, você pode precisar acessar determinados objetos apenas por um tempo limitado, após o qual poderá excluí-los.

A criação e aplicação de políticas para o gerenciamento do ciclo de vida de objetos no S3 permite que você exclua objetos automaticamente após um determinado período de tempo, o que pode ajudar a economizar nos custos de armazenamento. Ele também pode ajudar a melhorar a segurança, garantindo que objetos antigos e não utilizados não sejam deixados espalhados onde possam ser acessados ​​por usuários não autorizados.

Para criar e aplicar políticas para o gerenciamento do ciclo de vida do objeto S3, você pode usar o AWS Management Console, os SDKs da AWS ou a AWS Command Line Interface (AWS CLI).

# 7. Habilite o versionamento em todos os buckets

O controle de versão é um recurso crítico do S3 que permite manter várias versões de um objeto no mesmo bucket. Isso é útil por vários motivos, como poder reverter para versões anteriores de um objeto ou recuperar-se de exclusões acidentais.

Habilitar o controle de versão em um bucket é fácil e leva apenas alguns cliques no console AWS. Depois de ativados, todos os objetos armazenados no bucket terão seu próprio ID de versão única. Esses IDs podem ser usados ​​para recuperar versões específicas de um objeto quando necessário.

# 8. Criptografar dados em repouso (_Encrypt data at rest_)

Quando os dados são armazenados no S3, eles são armazenados fisicamente em servidores pertencentes e operados pela Amazon. Embora a Amazon faça um ótimo trabalho protegendo seus servidores, eles não podem garantir que os servidores nunca sejam comprometidos.

Se os dados forem criptografados em repouso, mesmo que os servidores sejam comprometidos, os dados ainda estarão seguros porque serão ilegíveis sem as chaves de criptografia.

Há duas maneiras principais de criptografar dados em repouso no S3: criptografia do lado do servidor e criptografia do lado do cliente.

A criptografia do lado do servidor é quando a Amazon criptografa os dados antes de serem gravados no servidor. Os dados são descriptografados automaticamente quando são lidos do servidor.

A criptografia do lado do cliente é quando os dados são criptografados pelo cliente antes de serem enviados para a Amazon. Os dados permanecem criptografados enquanto são armazenados no servidor e só são descriptografados quando o cliente os recupera.

Ambos os métodos são igualmente seguros, mas a criptografia do lado do cliente requer mais trabalho para configurar.

# 9. Monitore e audite o acesso aos seus buckets

Se você não estiver monitorando e auditando o acesso aos seus buckets, não terá como saber quem está acessando seus dados ou o que eles estão fazendo com eles. Isso pode levar a violações de dados, vazamentos de dados ou até mesmo perda de dados.

Para evitar esses riscos, você deve sempre monitorar e auditar o acesso aos seus buckets do S3. A AWS fornece um serviço chamado CloudTrail que facilita isso. Com o CloudTrail, você pode ver quem acessou seus buckets, o que eles fizeram e quando o fizeram.

Você também pode usar o CloudTrail para configurar alertas para ser notificado imediatamente se alguém tentar acessar seus buckets sem permissão. Dessa forma, você pode agir rapidamente para evitar qualquer dano.

# 10. Limite o acesso público aos seus buckets

Quando você cria um novo bucket do S3, ele é automaticamente privado. No entanto, existem várias maneiras de alguém tornar seu bucket público sem que você perceba. Por exemplo, se você usar um console do Amazon S3 para fazer upload de arquivos para um bucket e esquecer de definir as permissões, qualquer pessoa poderá acessar esses arquivos.

Para evitar isso, é importante limitar o acesso público aos seus buckets. Você pode fazer isso configurando uma política de bucket que nega todo o acesso público ou usando Amazon S3 Block Public Access.

Ambos os métodos ajudarão a garantir que apenas usuários autorizados possam acessar seus buckets S3 e que seus dados estejam seguros.

---

# Créditos

- Escrito originalmente por [Janet Maples](https://climbtheladder.com/users/1710/profile/janet-maples/), em [10 S3 Naming Convention Best Practices](https://climbtheladder.com/10-s3-naming-convention-best-practices/).
