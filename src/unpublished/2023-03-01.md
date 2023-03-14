# Criando autenticação sem senha com Amazon Cognito

> **Atualmente patrocinado por [Momento](https://www.gomomento.com/)**: Com Momento Serverless Cache você tem benefícios de velocidade, durabilidade, disponibilidade e otimização de custos ao aplicar cache em banco de dados com zero configuração. Comece rápido (e de graça): escolha uma linguagem, escreva 5 linhas de código, selecione sua nuvem/região e vá embora.

A autenticação baseada em senha tem sido a norma para proteger contas de usuário. No entanto, está ficando cada vez mais claro que a autenticação baseada em senha tem várias desvantagens. Como o risco de roubo de senha, a necessidade de os usuários lembrarem senhas complexas e o tempo e esforço necessários para redefinir senhas esquecidas.

Felizmente, mais e mais sites começaram a adotar autenticação sem senha. Como o nome sugere, é um meio de verificar a identidade de um usuário sem usar senhas.

Neste artigo, exploraremos como implementar a autenticação sem senha com o Amazon Cognito.

O Amazon Cognito é um serviço totalmente gerenciado que fornece criação, login e controle de acesso de usuário. Sua integração direta com outros serviços da AWS, como API Gateway, AppSync e Lambda, é uma das maneiras mais fáceis de adicionar autenticação e autorização aos aplicativos em execução na AWS. E também é um dos produtos mais econômicos do mercado, comparado aos da Auth0 e Okta.

Se você quiser ver uma análise completa do caso a favor e contra Cognito, [confira meu artigo sobre o tema](https://theburningmonk.com/2021/03/the-case-for-and-against-amazon-cognito/).

# Autenticação sem senha com Amazon Cognito

A autenticação sem senha pode ser implementada de várias maneiras, como:

- **Biometria:** Pense em IDs ou impressões digitais.
- **Fatores de posse:** Algo que o usuário possui, como endereço de email ou número de telefone. Se um usuário puder abrir uma conta com você usando o email, poderá autenticar o usuário enviando uma senha única (_OTP - One-time passwords_) para o email do usuário.
- **Links mágicos:** O usuário digita o email e você envia um email com um link especial (também conhecido como "link mágico"). Quando o usuário clica no link, ele volta ao aplicativo e autentica o acesso.

O Cognito não suporta autenticação sem senha por padrão. Mas você pode implementar fluxos de autenticação personalizados usando [Lambda Triggers](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-identity-pools-working-with-aws-lambda-triggers.html).

Neste artigo, mostrarei como implementar a autenticação sem senha usando senhas únicas (_OTP - One-time passwords_).

# Como funciona

Para esta solução, eu usaria esses três ganchos Lambda para implementar o fluxo de autenticação personalizado:

![](https://theburningmonk.com/wp-content/uploads/2023/03/img_640f00fb5ef4d.png)

- *Define Auth Challenge*: https://docs.aws.amazon.com/cognito/latest/developerguide/user-pool-lambda-define-auth-challenge.html
- *Create Auth Challenge*: https://docs.aws.amazon.com/cognito/latest/developerguide/user-pool-lambda-create-auth-challenge.html
- *Verify Auth Challenge*: https://docs.aws.amazon.com/cognito/latest/developerguide/user-pool-lambda-verify-auth-challenge-response.html

Usarei o Amazon Simple Email Service (SES) para enviar e-mails ao usuário. Se você for experimentar por si mesmo, precisará criar e verificar uma identidade de domínio no SES. Por favor, [consulte a página oficial da documentação](https://docs.aws.amazon.com/ses/latest/dg/creating-identities.html) para saber mais detalhes sobre como fazer isso.

Aqui está o nosso fluxo de autenticação:

![](https://theburningmonk.com/wp-content/uploads/2023/03/img_640f011131323.png)

1. O usuário inicia o fluxo de autenticação com seu endereço de email.

2. O pool de usuários chama a função lambda `DefineAuthChallenge` para decidir o que deve fazer. A função recebe o corpo da requisição e invocação, como na imagem abaixo:

![](https://theburningmonk.com/wp-content/uploads/2023/03/img_640f0140b9b40.png)

Aqui podemos ver um usuário com o email especificado encontrado no pool de usuários porque `userNotFound` é `false`. E podemos inferir que este é o começo de um novo fluxo de autenticação porque o array `session` está vazia.

Portanto, a função instrui o pool de usuários a emitir um `CUSTOM_CHALLENGE` para o usuário como o próximo passo. Como você pode ver no valor de retorno desta invocação da função lambda:

![](https://theburningmonk.com/wp-content/uploads/2023/03/img_640f015a96c91.png)

3. Para criar o desafio personalizado, o pool de usuários chama a função lambda `CreateAuthChallenge`.

![](https://theburningmonk.com/wp-content/uploads/2023/03/img_640f0171114fe.png)

A função gera uma senha única e a envia por e-mail ao usuário, usando o Serviço de E-mail Simples (SES). 

![](https://theburningmonk.com/wp-content/uploads/2023/03/img_640f018946b36.png)

É importante que essa função salve a senha única em algum lugar para que possamos verificar a resposta do usuário posteriormente. Você pode fazer isso salvando dados privados no objeto `response.privateChallengeParameters`.

![](https://theburningmonk.com/wp-content/uploads/2023/03/img_640f019f6d935.png)

O que você colocar aqui não será enviado ao usuário, mas será passado para a função lambda `VerifyAuthChallengeResponse` quando o usuário responder ao nosso desafio.

Qualquer informação que você deseja transmitir de volta ao front-end pode ser adicionada ao objeto `response.publicChallengeParameters`. Aqui, eu incluo o email do usuário e informações sobre quantas tentativas o usuário ainda tem para responder com o código certo.

Na captura de tela, você pode ver que o [Lumigo](https://lumigo.io/) removou a senha única (em `response.privateChallengeParameters.secretLoginCode`) do segmento/trace. Esse é um comportamento interno em que ele elimina todos os dados que se parecem com segredos ou dados confidenciais. Mas posso dizer que a senha única é `XQezeO` neste caso. Porque também é capturado em `response.challengeMetadata`, para o qual vamos usar mais tarde.

4. O usuário digita a senha única na tela de login.

5. O pool de usuários chama a função lambda `VerifyAuthChallengeResponse` para verificar se a resposta do usuário é válida. Como você pode ver no evento de invocação abaixo, podemos ver os dois:

- a resposta do usuário (`request.challengeAnswer`) e
- a senha única que a função lambda `CreateAuthChallenge` gerou e salvou em `request.privateChallengeParameters`

![](https://theburningmonk.com/wp-content/uploads/2023/03/img_640f01be2740c.png)

Podemos comparar os dois e informar ao pool de usuários se o usuário respondeu corretamente, definindo `response.answerCorrect` para `true` ou `false`.

6. O pool de usuários chama a função lambda `DefineAuthChallenge` novamente para decidir o que acontece a seguir. No evento de invocação abaixo, você pode ver que o array `session` agora tem um elemento. O `challengeResult` é o valor retornado pelo `VerifyAuthChallengeResponse` em `response.answerCorrect`.

![](https://theburningmonk.com/wp-content/uploads/2023/03/img_640f01d3785fb.png)

Neste ponto, temos algumas opções:

- Falha na autenticação porque o usuário respondeu incorretamente muitas vezes.
- Sucesso no fluxo de autenticação e podemos emitir os tokens JWT para o usuário.
- Dê ao usuário outra chance de responder corretamente.

Os dois primeiros casos são bastante diretos. A função lambda `DefineAuthChallenge` precisaria definir `response.failAuthentication` ou `response.issueTokens` para `true` respectivamente.

Onde fica mais interessante é se queremos dar ao usuário outra chance. Nesse caso, definimos os dois `response.issueTokens` e `response.failAuthentication` para `false` e `response.challengeName` para `CUSTOM_CHALLENGE`.

![](https://theburningmonk.com/wp-content/uploads/2023/03/img_640f01eaeb404.png)

O fluxo retornaria a função lambda `CreateAuthChallenge`. Mas como você pode ver abaixo, o `privateChallengeParameters`, que usamos anteriormente, não está incluído nesse evento de invocação!

![](https://theburningmonk.com/wp-content/uploads/2023/03/img_640f0203720ac.png)

É por isso que incluímos a senha única no `response.challengeMetadata` na etapa 3!

Desse modo, a função lambda `CreateAuthChallenge` pode reutilizar a mesma senha única de antes. E a julgar pelo número de itens no array `request.session`, ele sabe quantas tentativas com falha o usuário fez. Portanto, também podemos informar ao front-end quantas tentativas o usuário tem sobrando antes que ele deva reiniciar o fluxo de autenticação e obter uma nova senha única.

![](https://theburningmonk.com/wp-content/uploads/2023/03/img_640f021c0cad3.png)

Espero que isso lhe dê uma sólida estrutura conceitual de como o fluxo de autenticação funciona.

Agora vamos falar sobre a implementação.

# Como implementá-lo

### 1. Configure um pool de usuários Cognito

Primeiro, precisamos configurar um pool de usuários Cognito.

```yaml
PasswordlessOtpUserPool:
  Type: AWS::Cognito::UserPool
  Properties:
    UsernameConfiguration:
      CaseSensitive: false
    UsernameAttributes:
      - email
    Policies:
      # isso é apenas para satisfazer os requisitos de Amazon Cognito
      # não usaremos senhas, mas também não queremos senhas fracas no sistema ;-)
      PasswordPolicy:
        MinimumLength: 16
        RequireLowercase: true
        RequireNumbers: true
        RequireUppercase: true
        RequireSymbols: true
    Schema:
      - AttributeDataType: String
        Mutable: false
        Required: true
        Name: email
        StringAttributeConstraints: 
          MinLength: '8'
    LambdaConfig:
      PreSignUp: !GetAtt PreSignUpLambdaFunction.Arn
      DefineAuthChallenge: !GetAtt DefineAuthChallengeLambdaFunction.Arn
      CreateAuthChallenge: !GetAtt CreateAuthChallengeLambdaFunction.Arn
      VerifyAuthChallengeResponse: !GetAtt VerifyAuthChallengeResponseLambdaFunction.Arn
```

É importante observar que as senhas ainda são necessárias, mesmo que você não pretenda usá-las. Eu defini um requisito de senha bastante forte aqui, mas as senhas seriam geradas pelo front end e nunca serão expostas ao usuário.

Nosso pool de usuários não verificará o email do usuário quando ele inscrever. Como toda vez que o usuário tenta entrar, iremos enviar uma senha única para o email. Esse processo já verifica a propriedade do endereço de email pelo usuário naquele momento.

### 2. Configure o cliente de pool de usuários para o front-end

O aplicativo front-end precisa de um ID do cliente para conversar com o pool de usuários. Como não queremos que os usuários efetuem login com senhas, suportaremos apenas o fluxo de autenticação personalizado com `ALLOW_CUSTOM_AUTH`.

```yaml
WebUserPoolClient:
  Type: AWS::Cognito::UserPoolClient
  Properties:
    ClientName: web
    UserPoolId: !Ref PasswordlessOtpUserPool
    ExplicitAuthFlows:
      - ALLOW_CUSTOM_AUTH
      - ALLOW_REFRESH_TOKEN_AUTH
    PreventUserExistenceErrors: ENABLED
```

### 3. O hook `PreSignUp`

Normalmente, um usuário precisa confirmar seu registro com um código de verificação (para provar que possui o endereço de email usado). Mas, como mencionado acima, vamos pular essa etapa de verificação porque iremos verificar a propriedade do usuário do email toda vez que eles tentassem entrar.

Então na função lambda `PreSignUp`, precisamos informar ao Cognito para confirmar o usuário, definindo `event.response.autoConfirmUser` para `true`.

```js
module.exports.handler = async (event) => {
  event.response.autoConfirmUser = true
  return event
}
```

### 4. (Front-end) Criando usuário

![](https://theburningmonk.com/wp-content/uploads/2023/03/img_640f024ee52db.png)

Quando o usuário se inscrever no nosso aplicativo, o front-end gera uma senha aleatória de 16 dígitos nos bastidores. Essa senha nunca é mostrada ao usuário e é essencialmente descartada após esse ponto.

O `aws-amplify` possui um módulo `Auth` bem útil, que podemos usar para interagir com o pool de usuários.

```js
import { Amplify, Auth } from 'aws-amplify'

Amplify.configure({
  Auth: {
    region: ...,
    userPoolId: ...,
    userPoolWebClientId: ...,
    mandatorySignIn: true
  }
})

async function signUp() {
  const chance = new Chance()
  const password = chance.string({ length: 16 })
  await Auth.signUp({
    username: email.value,
    password
  })
}
```

Novamente, isso é necessário porque o Cognito exige que você configure senhas, mesmo que não pretenda usá-las.

### 5. (Front-end) Realizando login

![](https://theburningmonk.com/wp-content/uploads/2023/03/img_640f026acbe5e.png)

Depois de registrado, o usuário pode entrar fornecendo apenas o endereço de e-mail.

```js
async function signIn() {
  cognitoUser = await Auth.signIn(email.value)
}
```

Isso inicia o fluxo de autenticação personalizado.

### 6. A função `DefineAuthChallenge`

O fluxo de autenticação personalizado do Cognito se comporta como uma máquina de estado. A função `DefineAuthChallenge` é o tomador de decisão e instrui o pool de usuários sobre o que fazer a seguir toda vez que algo importante acontecer.

Como você pode ver na visão geral da solução, esta função é ativada várias vezes durante uma sessão de autenticação:

- quando o usuário inicia a autenticação e
- toda vez que o usuário responde a um desafio de autenticação

Esta é a máquina de estado que queremos implementar:

![](https://theburningmonk.com/wp-content/uploads/2023/03/img_640f0280e314b.png)

E o minha função `DefineAuthChallenge` se parece com o seguinte:

```js
const _ = require('lodash')
const { MAX_ATTEMPTS } = require('../lib/constants')

module.exports.handler = async (event) => {
  const attempts = _.size(event.request.session)
  const lastAttempt = _.last(event.request.session)

  if (event.request.session &&
      event.request.session.find(attempt => attempt.challengeName !== 'CUSTOM_CHALLENGE')) {
      // Nunca deve acontecer, mas caso tenhamos algo diferente
      // do que um desafio personalizado, então algo está errado e devemos abortar
      event.response.issueTokens = false
      event.response.failAuthentication = true
  } else if (attempts >= MAX_ATTEMPTS && lastAttempt.challengeResult === false) {
      // O usuário deu muitas respostas erradas seguidas
      event.response.issueTokens = false
      event.response.failAuthentication = true
  } else if (attempts >= 1 &&
      lastAttempt.challengeName === 'CUSTOM_CHALLENGE' &&
      lastAttempt.challengeResult === true) {
      // Resposta certa
      event.response.issueTokens = true
      event.response.failAuthentication = false
  } else {
      // Resposta errada, tente novamente
      event.response.issueTokens = false
      event.response.failAuthentication = false
      event.response.challengeName = 'CUSTOM_CHALLENGE'
  }

  return event
}
```

Observe que toda vez que o usuário tenta responder ao desafio, o resultado é registrado em `event.request.session`.

_**Nota**: Se você está se perguntando como o Cognito é capaz de agrupar essas tentativas em um só lugar, é porque a string `Session` é passada para frente e para trás à medida que o cliente interage com o pool de usuários. Você pode ver [isso na resposta da API `InitiateAuth`](https://docs.aws.amazon.com/cognito-user-identity-pools/latest/APIReference/API_InitiateAuth.html#CognitoUserPools-InitiateAuth-response-Session) e na [requisição da  API `RespondToAuthChallenge`](https://docs.aws.amazon.com/cognito-user-identity-pools/latest/APIReference/API_RespondToAuthChallenge.html#CognitoUserPools-RespondToAuthChallenge-request-Session)._

### 7. A função `CreateAuthChallenge`

A função `CreateAuthChallenge` é responsável por gerar a senha única e enviá-la por e-mail ao usuário.

Essa função também pode ser invocada várias vezes em uma sessão de autenticação se o usuário não fornecer a resposta certa no início. Mais uma vez, podemos usar o `request.session` para descobrir se estamos lidando com uma sessão de autenticação existente.

```js
const _ = require('lodash')
const Chance = require('chance')
const chance = new Chance()
const { MAX_ATTEMPTS } = require('../lib/constants')

module.exports.handler = async (event) => {
  let otpCode
  if (!event.request.session || !event.request.session.length) {
    // nova sessão de autenticação
    otpCode = chance.string({ length: 6, alpha: false, symbols: false })
    await sendEmail(event.request.userAttributes.email, otpCode)
  } else {
    // sessão existente, o usuário forneceu uma resposta errada, por isso precisamos dar uma outra chance
    const previousChallenge = _.last(event.request.session)
    const challengeMetadata = previousChallenge?.challengeMetadata

    if (challengeMetadata) {
      // challengeMetadata deve começar com "CODE-", daí o índice de 5
      otpCode = challengeMetadata.substring(5)
    }
  }

  const attempts = _.size(event.request.session)
  const attemptsLeft = MAX_ATTEMPTS - attempts
  event.response.publicChallengeParameters = {
    email: event.request.userAttributes.email,
    maxAttempts: MAX_ATTEMPTS,
    attempts,
    attemptsLeft
  }

  // NOTA: os parâmetros de desafio privado são passados para a
  // etapa de verificação, mas não são exposto ao chamador original
  // precisamos passar o código secreto para realizar a verificação da resposta do usuário
  event.response.privateChallengeParameters = { 
    secretLoginCode: otpCode
  }

  event.response.challengeMetadata = `CODE-${otpCode}`

  return event
}
```

A função `sendEmail` foi omitida aqui por uma questão de brevidade. Ele faz o que você esperaria e envia a senha única ao usuário por email.

### 8. (Front-end) Respondendo ao desafio

![](https://theburningmonk.com/wp-content/uploads/2023/03/img_640f029e111a3.png)

No front-end, você deve capturar o objeto `CognitoUser` retornado por `Auth.signIn`. Você precisa responder ao desafio de autenticação personalizada, pois ele contém os dados da `Session` que Cognito exige.

```js
async function answerCustomChallenge() {
  // Isso causará um erro se for a terceira resposta errada
  try {
    const challengeResult = await Auth.sendCustomChallengeAnswer(cognitoUser, secretCode.value)

    if (challengeResult.challengeName) {
      secretCode.value = ''
      attemptsLeft.value = parseInt(challengeResult.challengeParam.attemptsLeft)

      alert(`O código digitado está incorreto. ${attemptsLeft.value} tentativas restantes.`)
    }
  } catch (error) {
    alert('Muitas tentativas erradas. Por favor, tente novamente.')
  }  
}
```

Observe que o `publicChallengeParameters` devolvido pelo pela função `CreateAuthChallenge` está acessível aqui. Para descobrir quantas tentativas o usuário tem sobrando na sessão atual.

Se a função `DefineAuthChallenge` dizer ao pool de usuários para falhar a autenticação, então, `Auth.sendCustomChallengeAnswer` jogaria uma exceção `NotAuthorizedException` com a mensagem `Incorrect username or password`.

### 9. A função `VerifyAuthChallengeResponse`

A função `VerifyAuthChallengeResponse` é responsável por verificar a resposta do usuário. Para fazer isso, ele precisa acessar a senha única que a função `CreateAuthChallenge` geraou e escondeu no `privateChallengeParameters`.

```js
module.exports.handler = async (event) => {
  const expectedAnswer = event.request?.privateChallengeParameters?.secretLoginCode
  if (event.request.challengeAnswer === expectedAnswer) {
    event.response.answerCorrect = true
  } else {
    event.response.answerCorrect = false
  }
  
  return event
}
```

E é isso! Estes são os ingredientes necessários para implementar a autenticação sem senha com Amazon Cognito. 🥳

# Experimente você mesmo

Para ter uma idéia de como esse mecanismo de autenticação sem senha funciona, sinta-se à vontade para experimentar com o aplicativo de demonstração: https://passwordless-cognito.theburningmonk.com/otp

![](https://theburningmonk.com/wp-content/uploads/2023/03/img_640f02b8ad507.png)

E você pode encontrar o código fonte dessa demonstração no GitHub:

- Back-end e Infrasestrutura (usando Serverless Framework): https://github.com/theburningmonk/passwordless-otp-cognito-demo
- Front-end (usando Vue.js versão 3 com Vite): https://github.com/theburningmonk/passwordless-cognito-ui-demo

# Finalizando

Espero que você tenha achado este artigo útil e que ele o ajude a tirar mais proveito do Cognito, um serviço um tanto mal amado.

Se você quiser saber mais sobre como criar arquitetura serverless, confira meu próximo workshop onde eu vou cobrir tópicos como testes, segurança, observabilidade e muito mais.

https://productionreadyserverless.com/

Espero te ver por lá! 🤓

---

# Créditos

- Escrito originalmente por [Yan Cui](https://twitter.com/theburningmonk), em [Passwordless Authentication made easy with Cognito: a step-by-step guide](https://theburningmonk.com/2023/03/passwordless-authentication-made-easy-with-cognito-a-step-by-step-guide).