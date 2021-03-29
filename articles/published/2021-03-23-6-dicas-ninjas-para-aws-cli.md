# 6 Dicas Ninjas para AWS CLI

A [Linha de Comando da AWS (CLI)](https://aws.amazon.com/cli) permite que você gerencie os serviços da AWS. Usar a CLI de seu terminal, permite interativamente que você automatize metade das tarefas e libera você de fazer login no Console da AWS. Além disso, a integração da CLI em scripts shell permite automatizar sua infraestrutura e a configuração de instâncias EC2 durante o processo de inicialização.

Este artigo cobre os obstáculos típicos ao usar a AWS CLI.

## Autocompletar de Comandos

Ao usar a CLI no seu terminal de comando, o autocompletar de um comando é um recurso indispensável que você não deve perder. Quando habilitado, permite que você use a tecla TAB para completar os comandos. Isso irá acelerar significativamente o seu uso da CLI.

As etapas a seguir são necessárias para ativar o autocompletar de comandos para o bash no macOS:

```bash
echo "complete -C aws_completer aws" >> ~/.bash_profile
source ~/.bash_profile
```

A [documentação oficial](http://docs.aws.amazon.com/cli/latest/userguide/cli-command-completion.html) também contém instruções gerais para outros shells.

## Filtrando Resultados no Lado do Servidor

Por padrão, a CLI usa um tamanho de página de 1000 e recupera todos os itens disponíveis. Se você precisar solicitar itens de uma lista de mais de 1000 itens ou se quiser acelerar a execução de seus comandos, uma boa ideia é filtrar os resultados de sua solicitação no lado do servidor.

Muitos comandos `describe-*` e ` list-*` suportam a filtragem do lado do servidor: `--filter`. Por exemplo, é possível filtrar instâncias EC2 por tipo de instância:

```bash
$ aws ec2 describe-instances --filter Name=instance-type,Values=t2.nano
```

## Filtrando Resultados no Lado do Cliente

Outra característica útil da CLI é a filtragem de resultados de qualquer comando no lado do cliente com: `--query`. A linguagem de consulta [JMESPath](http://jmespath.org/) é usada para filtragem.

O exemplo a seguir lista todos os VPCs em uma região e filtra os resultados usando um `--query`.

```bash
$ aws ec2 describe-vpcs --query "Vpcs[?VpcId == 'vpc-aaa22bbb'].CidrBlock"
[
    "94.194.0.0/16"
]
```

Você pode precisar do CIDR de um VPC como uma variável em seu script de shell. O exemplo a seguir mostra como fazer isso. Formatar a saída como texto adicionando o parâmetro `--output text` remove o caractere `"` do resultado JSON.

```bash
#!/bin/bash
CIDR=$(aws ec2 describe-vpcs --query "Vpcs[?VpcId == 'vpc-aaa22bbb'].CidrBlock" --output text)
echo $CIDR
```

## Esperar por...

Ao escrever scripts de shell usando a CLI, é necessário aguardar uma condição específica de vez em quando. Por exemplo, após iniciar um _snapshot_ EBS, seu script pode precisar esperar até que o _snapshot_ seja concluído. A espera pode ser realizada com um loop de pesquisa e um comando `describe-*`. Mas há uma solução mais simples embutida na CLI: `aws <service> wait <condition>`.

O exemplo a seguir contém um comando de espera que bloqueará o script até que o _snapshot_ seja concluído.

```bash
#!/bin/bash
echo "Esperando o EBS snapshot"
aws ec2 wait snapshot-completed --snapshot-ids snap-aabbccdd
echo "EBS snapshot completado"
```

## Assumindo uma função IAM

A CLI oferece suporte para assumir uma função IAM. Isso é muito útil se você precisar alternar entre várias contas da AWS com a ajuda de IAM Role entre contas (_cross-account_).

Tudo que você precisa fazer é configurar dois perfis em `~/.aws/config`: um IAM User e um perfil IAM Role.

```ini
[profile iam-user]
output = json
region = eu-west-1

[profile iam-role]
role_arn = arn:aws:iam::<ACCOUNT_ID>:role/<IAM_ROLE>
source_profile = iam-user
output = json
region = eu-west-1
```

Apenas o IAM User precisa de credenciais de segurança armazenadas em `~/.aws/credentials`.

```ini
[iam-user]
aws_access_key_id = ***
aws_secret_access_key = ***
```

Depois disso, você pode assumir a IAM Role adicionando `--profile iam-role` aos seus comandos CLI.

## Ajustando configurações do S3

A AWS CLI inclui comandos de transferência para S3: `cp`, `sync`, `mv`, e `rm`. Você pode ajustar esses comandos com configurações especiais.

Por exemplo, se você precisar sincronizar um grande número de arquivos pequenos com o S3, você pode adicionar valores de configuração no seu `~/.aws/config` para acelerar o processo de sincronização.

```ini
[profile default]
...
s3 =
  max_concurrent_requests = 100
  max_queue_size = 10000
  use_accelerate_endpoint = true
```

A [documentação oficial](http://docs.aws.amazon.com/cli/latest/topic/s3-config.html) contém informações detalhadas sobre valores de configuração da sessão `s3`.

# Créditos

- [6 tips and tricks for AWS command-line ninjas](https://cloudonaut.io/6-tips-and-tricks-for-aws-command-line-ninjas/), escrito originalmente por [Andreas Wittig](https://twitter.com/andreaswittig).
