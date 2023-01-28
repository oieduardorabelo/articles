# É por isso que você deve manter os recursos AWS com e sem estado juntos

Acoplamento fraco (_loose coupling_) e alta coesão (_high cohesion_) são dois dos princípios de engenharia de software mais essenciais. Coisas não relacionadas devem ficar separadas, enquanto elementos relacionados devem ser mantidos juntos.

Esses princípios se aplicam a todos os níveis de nosso aplicativo — desde a arquitetura no nível do sistema até os módulos ou funções individuais.

Com esse princípio em mente, vamos falar sobre um tópico polêmico: se você deve manter recursos com e sem estado na mesma pilha do CloudFormation.

Eu estou no time que escolhe a pilha monolítica. Prefiro manter recursos com estado (bancos de dados, filas, etc) e sem estado (funções do Lambda, API Gateway, etc.) juntos.

# Argumentos a favor para pilha monolítica

Supondo que a pilha do CloudFormation encapsula um serviço inteiro, que inclui recursos com e sem estado, faz sentido definir todos os recursos em uma única pilha do CloudFormation. Isso facilita o gerenciamento e a implantação de serviços de várias maneiras:

1. Referência de recursos é fácil. Você pode usar `!Ref` e `!GetAtt` contra quaisquer recursos definidos na pilha. Por exemplo, quando você precisa passar o nome de uma tabela do DynamoDB para uma função do Lambda como uma variável de ambiente.

2. Você pode atualizar os recursos com e sem estado em um único commit e deploy. Por exemplo, quando você precisa adicionar uma nova tabela do DynamoDB e adicionar uma nova função do Lambda para usá-la.

3. A configuração de CI/CD é mais simples. Um serviço, uma pilha, um repositório, um pipeline.

4. É fácil criar ambientes efêmeros. Por exemplo, quando você precisa trabalhar em um novo recurso, pode criar um ambiente temporário com uma única implantação. Usando o Serverless Framework, isso é tão simples quanto executar `npx sls deploy --stage <stage-name>`. Quando terminar de usar o recurso, simplesmente exclua o ambiente temporário.

# Argumentos a favor para pilhas separadas

Os contra-argumentos dessa abordagem geralmente evoluem a partir desses três pontos:

1. É menos arriscado se você separar os recursos com estado em sua própria pilha. Se alguém excluir acidentalmente a pilha sem estado, pelo menos você não perderá dados.

2. Os recursos com estado mudam com menos frequência do que os recursos sem estado. Portanto, os deploys serão mais rápidos se você precisar implantar apenas os recursos sem estado na maioria das vezes.

3. O CloudFormation tem um limite rígido de 500 recursos por pilha. Mover os recursos com estado para fora permite que você coloque mais recursos sem estado na pilha.

Todos esses contra-argumentos parecem razoáveis, mas o quanto eles importam na prática e justificam a complexidade extra de ter pilhas separadas?

# Exclusão acidental

Mover os recursos com estado para sua própria pilha não elimina o risco de exclusão acidental. Apenas move o alvo. Alguém aperta o botão delete na pilha com estado e o jogo se acaba do mesmo jeito.

A maneira certa de se proteger contra exclusão acidental é habilitar `TerminationProtection` na pilha e/ou adicionar `DeletionPolicy` para `Retain` nos recursos com estado.

Existem outras maneiras pelas quais os recursos podem ser excluídos acidentalmente. Por exemplo, quando você altera o nome de uma tabela do DynamoDB, o CloudFormation substitui a tabela durante a implantação. Como você pode ver na documentação oficial abaixo.

![](https://theburningmonk.com/wp-content/uploads/2023/01/img_63bb627f17dd1.png)

Portanto, você também deve considerar definir o `UpdateReplacePolicy` para `Retain` para **proteger contra perda de dados de alterações acidentais**. Esse risco específico está presente independentemente da pilha, sejam recursos com estado ou sem estado.

# Deploy mais rápido

Por padrão, o CloudFormation ignora os recursos que não foram alterados. Portanto, se os recursos com estado não tiverem sido atualizados, eles terão um impacto insignificante no tempo necessário para o deploy da pilha.

Na maioria dos casos, o tempo que leva para atualizar uma pilha existente é baseado no número de recursos sem estado que você está atualizando. Por exemplo, funções do Lambda, funções do IAM, recursos do API Gateway, etc.

Para ilustrar isso, coletei dados de três pilhas do CloudFormation.

- Pilha 1: 5 funções Lambda.

- Pilha 2: 5 funções Lambda e 5 tabelas DynamoDB.

- Pilha 3: 20 funções Lambda.

Deixando de lado o tempo inicial de deploy com CloudFormation, este é o tempo médio que levou para atualizar essas pilhas (sem alterações nas tabelas do DynamoDB):

- Pilha 1: 46,4 seg

- Pilha 2: 46,4 seg

- Pilha 3: 55 seg

Não houve diferença no tempo médio de implantação entre a Pilha 1 e a Pilha 2. Isso considerando que a Pilha 2 tem 5 tabelas adicionais do DynamoDB.

A pilha 3 tem muito mais funções do Lambda, portanto, leva em média 10 segundos a mais para atualizar, cada vez.

# Limites de recursos do CloudFormation

Você pode contornar o limite de 500 recursos usando pilhas aninhadas.

Como você normalmente tem muito menos recursos com estado do que sem estado, mover esses recursos para uma outra pilha não comprará muito espaço e tempo na maioria das situações.

Embora este seja um argumento válido, na prática, não importa, a menos que você tenha muitos recursos com estado em sua pilha. E mesmo assim, usar pilhas aninhadas é a melhor maneira de lidar com o limite de 500 recursos.

# Conclusão

Os recursos com e sem estado precisam trabalhar juntos para que nosso sistema funcione. Para alcançar alta coesão, devemos mantê-los juntos. Separá-los em pilhas separadas viola um dos princípios mais importantes da engenharia de software e complica as coisas desnecessariamente.

No entanto, esta não é uma regra rígida e rápida. Sempre haverá casos extremos para os quais faz algum sentido dividi-los.

Mas na maioria dos casos, mantê-los juntos é a melhor escolha.

---

# Créditos

- Escrito originalmente por [Yan Cui](https://twitter.com/theburningmonk), em [This is why you should keep stateful and stateless resources together](https://theburningmonk.com/2023/01/this-is-why-you-should-keep-stateful-and-stateless-resources-together/).
