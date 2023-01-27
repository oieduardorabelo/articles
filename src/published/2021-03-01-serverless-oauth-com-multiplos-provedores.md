- https://dev.to/oieduardorabelo/serverless-oauth-com-multiplos-provedores-49k6
- https://oieduardorabelo.medium.com/serverless-oauth-com-m%C3%BAltiplos-provedores-a8eb5a7c2921

---

# Serverless OAuth com Múltiplos Provedores

Este exemplo mostra como usar múltiplos provedores OAuth (Google e Github, neste caso) para autenticar em um aplicativo serverless hospedado em [Begin.com](https://begin.com/) ou [Architect](https://arc.codes/) . Autenticação é quem é o usuário, e autorização (também chamada de permissões) é o que o usuário vê ou faz. Ambos são importantes, mas apenas a autenticação é abordada aqui. Nenhuma biblioteca de autenticação, serviço ou SDK do provedor é usado. Com Lambda, menos dependências proporcionam inícios mais rápidos. [Begin.com](https://begin.com/) limita especificamente as dependências a 5 MB para encorajar esta prática recomendada.

Para se concentrar no código OAuth, este aplicativo tem o mínimo possível de código. O arquivo `app.arc` de manifesto abaixo mostra as cinco rotas. A primeira rota `/` está acessível para convidados e usuários autenticados. A rota `/admin` só é visível para usuários autenticados e redireciona para `/login` outros usuários.

```
# ./app.arc
@app
oauth-example

@http
get /
get /admin
get /auth
get /login
post /logout
```

Para ver todo o aplicativo, fique à vontade para clonar o [repositório oauth-example](https://github.com/rbethel/oauth-example). Para experimentar por si mesmo, você pode implantá-lo diretamente no Begin.

[![Implantar para começar](https://res.cloudinary.com/practicaldev/image/fetch/s--yYw27_-V--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_880/https://static.begin.com/deploy-to-begin.svg)](https://begin.com/apps/create?template=https://github.com/rbethel/oauth-example)

## Visão geral do OAuth

O fluxo básico do OAuth é mostrado abaixo. Um usuário solicita um login para o aplicativo e é apresentado com opções para autenticar em qualquer um dos provedores disponíveis. Os links para esses provedores enviam uma solicitação do usuário diretamente para o provedor. Depois de entrar no provedor, uma resposta é enviada ao servidor com um token. O servidor usa esse token para enviar uma solicitação ao provedor para o perfil do usuário usando esse token. Com a resposta, o usuário é autenticado no aplicativo.

[
![diagrama de fluxo básico oauth](https://res.cloudinary.com/practicaldev/image/fetch/s--n5MtLAZn--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_880/https://dev-to-uploads.s3.amazonaws.com/i/22dedmk0e4wcqdvxw45r.png)](https://res.cloudinary.com/practicaldev/image/fetch/s--n5MtLAZn--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_880/https://dev-to-uploads.s3.amazonaws.com/i/22dedmk0e4wcqdvxw45r.png)

## Pedido de Login

Quando um usuário solicita `/login` (ou é redirecionado para lá), a rota gera URLs para cada um dos provedores. Se eles foram redirecionados, um parâmetro "próximo" (isto é `/login?next=admin`) aponta de volta para a página original. O parâmetro `next` é verificado em relação a opções válidas (apenas admin aqui) para proteger um usuário de ser direcionado a um site malicioso após a autenticação.

```js
// ./src/http/get-login/index.js
const arc = require("@architect/functions");
const githubOAuthUrl = require("./githubOAuthUrl");
const googleOAuthUrl = require("./googleOAuthUrl");

async function login(req) {
  let finalRedirect = "/";
  if (req.query.next === "admin") {
    finalRedirect = "/admin";
  }
  const googleUrl = await googleOAuthUrl({ finalRedirect });
  const githubUrl = githubOAuthUrl({ finalRedirect });
  return {
    status: 200,
    html: `<!doctype html>
            <html lang="en">
                <head>
                    <meta charset="utf-8">
                    <meta name="viewport" content="width=device-width, initial-scale=1">
                    <title>login page</title>
                </head>
                <body>
                    <h1>Login</h1></br>
                    <a href="${githubUrl}">Login with Github</a></br>
                    <a href="${googleUrl}">Login with Google</a>
                </body>
            </html>`,
  };
}

exports.handler = arc.http.async(login);
```

## Parâmetro de Estado

O parâmetro de consulta `state` ajuda a garantir que a solicitação de retorno foi iniciada pelo servidor. Uma opção é gerar um número aleatório seguro armazenado no servidor e, em seguida, verificar com a solicitação de retorno. Neste exemplo, um JSON Web Token (JWT) é usado. O JWT é um objeto assinado criptograficamente que contém o provedor e o local de redirecionamento final (que vem do parâmetro `next`). Este JWT tem um conjunto de expiração de uma hora. Ele só precisa permanecer válido por tempo suficiente para concluir a autenticação.

```js
// ./src/http/get-login/githubOAuthUrl.js
const jwt = require("jsonwebtoken");
module.exports = function githubOAuthUrl({ finalRedirect }) {
  let client_id = process.env.GITHUB_CLIENT_ID;
  let redirect_uri = encodeURIComponent(process.env.AUTH_REDIRECT);
  let state = jwt.sign(
    {
      provider: "github",
      finalRedirect,
    },
    process.env.APP_SECRET,
    { expiresIn: "1 hour" }
  );
  let url = `https://github.com/login/oauth/authorize?client_id=${client_id}&redirect_uri=${redirect_uri}&state=${state}`;
  return url;
};
```

## URL do Google

A URL do Google é semelhante a função Github, exceto pela solicitação `GET` na parte superior. A documentação OAuth do Google recomenda verificar o endpoint de autenticação por uma solicitação ao documento "openid-configuration". Se o google alterar o caminho real do endpoint, ele será atualizado neste documento. Este documento é armazenado em cache agressivamente e a solicitação geralmente será retornada de lá.

```js
// ./src/http/get-login/googleOAuthUrl.js
const tiny = require("tiny-json-http");
const jwt = require("jsonwebtoken");

module.exports = async function googleOAuthUrl({ finalRedirect }) {
  const googleDiscoveryDoc = await tiny.get({
    url: "https://accounts.google.com/.well-known/openid-configuration",
    headers: { Accept: "application/json" },
  });
  const authorization_endpoint = googleDiscoveryDoc.body.authorization_endpoint;
  const state = await jwt.sign(
    {
      provider: "google",
      finalRedirect,
    },
    process.env.APP_SECRET,
    { expiresIn: "1 hour" }
  );
  const options = {
    access_type: "online",
    scope: ["profile", "email"],
    redirect_uri: process.env.AUTH_REDIRECT,
    response_type: "code",
    client_id: process.env.GOOGLE_CLIENT_ID,
  };
  const url = `${authorization_endpoint}?access_type=${
    options.access_type
  }&scope=${encodeURIComponent(
    options.scope.join(" ")
  )}&redirect_uri=${encodeURIComponent(options.redirect_uri)}&response_type=${
    options.response_type
  }&client_id=${options.client_id}&state=${state}`;

  return url;
};
```

## Auth Redirect do Provedor

Depois que o usuário fazer login com o provedor escolhido (Google ou Github), ele será redirecionado para a rota `/auth` com um conjunto de parâmetros `code` e `state`. O estado deve ser exatamente o mesmo estado que foi enviado ao provedor. O parâmetro `code` é um token usado para acessar as informações do perfil do usuário. O manipulador da rota `/auth` é mostrado abaixo. Ele decodifica o estado do JWT para verificar se a solicitação foi iniciada pelo aplicativo e para determinar qual provedor enviou o código de autorização.

```js
// ./src/http/get-auth/index.js
const arc = require("@architect/functions");
const githubAuth = require("./githubAuth");
const googleAuth = require("./googleAuth");
const jwt = require("jsonwebtoken");

async function auth(req) {
  let account = {};
  let state;
  if (req.query.code && req.query.state) {
    try {
      state = jwt.verify(req.query.state, process.env.APP_SECRET);
      if (state.provider === "google") {
        account.google = await googleAuth(req);
        if (!account.google.email) {
          throw new Error();
        }
      } else if (state.provider === "github") {
        account.github = await githubAuth(req);
        if (!account.github.login) {
          throw new Error();
        }
      } else {
        throw new Error();
      }
    } catch (err) {
      return {
        status: 401,
        body: "not authorized",
      };
    }
    return {
      session: { account },
      status: 302,
      location: state.finalRedirect,
    };
  } else {
    return {
      status: 401,
      body: "not authorized",
    };
  }
}

exports.handler = arc.http.async(auth);
```

## Solicitar perfil de usuário

A etapa final na sequência OAuth é obter o perfil do usuário. Para o Github, uma solicitação `POST` é enviada usando o `code` junto com o `client_id` e `client_secret`. O Github responde com um token de acesso. Esse token é então usado para fazer uma solicitação `GET` para o perfil do usuário. Com isso, o perfil do usuário é finalmente retornado e armazenado na sessão do Architect / Begin. O usuário agora está autenticado para qualquer solicitações adicionais do aplicativo.

```js
// ./src/http/get-auth/githubAuth.js
const tiny = require("tiny-json-http");

module.exports = async function githubAuth(req) {
  try {
    let result = await tiny.post({
      url: "https://github.com/login/oauth/access_token",
      headers: { Accept: "application/json" },
      data: {
        code: req.query.code,
        client_id: process.env.GITHUB_CLIENT_ID,
        client_secret: process.env.GITHUB_CLIENT_SECRET,
        redirect_uri: process.env.AUTH_REDIRECT,
      },
    });
    let token = result.body.access_token;
    let user = await tiny.get({
      url: `https://api.github.com/user`,
      headers: {
        Authorization: `token ${token}`,
        Accept: "application/json",
      },
    });
    return {
      name: user.body.name,
      login: user.body.login,
      id: user.body.id,
      url: user.body.url,
      avatar: user.body.avatar_url,
    };
  } catch (err) {
    return {
      error: err.message,
    };
  }
};
```

O Google também exige a verificação do endpoint do token antes da autenticação final (semelhante à etapa inicial). Novamente, essa resposta é armazenada em cache agressivamente para minimizar solicitações desnecessárias. Para obter o perfil do usuário com o Github usamos uma chamada `POST` para o token de acesso e um `GET` com esse token para o perfil do usuário. O Google combina esses dois. Com uma solicitação `POST`, recebemos um `id_token` que é um JWT com o perfil do usuário. O JWT é então decodificado para obter as informações do perfil do usuário.

```js
// ./src/http/get-auth/googleAuth.js
const tiny = require("tiny-json-http");
const jwt = require("jsonwebtoken");

module.exports = async function googleAuth(req) {
  let googleDiscoveryDoc = await tiny.get({
    url: "https://accounts.google.com/.well-known/openid-configuration",
    headers: { Accept: "application/json" },
  });
  let token_endpoint = googleDiscoveryDoc.body.token_endpoint;

  let result = await tiny.post({
    url: token_endpoint,
    headers: { Accept: "application/json" },
    data: {
      code: req.query.code,
      client_id: process.env.GOOGLE_CLIENT_ID,
      client_secret: process.env.GOOGLE_CLIENT_SECRET,
      redirect_uri: process.env.AUTH_REDIRECT,
      grant_type: "authorization_code",
    },
  });
  return jwt.decode(result.body.id_token);
};
```

## Configurando OAuth nos provedores

Para se autenticar no Google e no Github, você precisa configurar isso com os dois provedores.

### Configuração do Github

Para [Github.com](https://github.com/), navegue até:

Configurações -> Configurações do Desenvolvedor -> Aplicativos OAuth -> Novo Aplicativo OAuth.

A partir daí, preencha o formulário conforme necessário. Certifique-se de que o url de retorno de chamada corresponda ao domínio completo e ao caminho do seu aplicativo. O domínio para teste e produção pode ser encontrado nas configurações no Begin. Mais detalhes podem ser encontrados no [Github Docs](https://developer.github.com/apps/building-oauth-apps/authorizing-oauth-apps/) .

### Configuração do Google

O console do Google é mais complicado de navegar. Comece inscrevendo-se em uma conta de desenvolvedor em [https://console.cloud.google.com/](https://console.cloud.google.com/) . Configure um novo projeto e vá para o painel "API e Serviços". Configure a "Tela de consentimento OAuth" para usuários externos. Em seguida, escolha Credenciais -> Criar Credenciais -> OAuth Client ID. Siga a configuração do "aplicativo da web" e insira o URL de redirecionamento e outras informações para seu aplicativo.

## Fluxo OAuth completo

O diagrama de fluxo completo do OAuth é mostrado abaixo. As etapas do Google são mostradas em vermelho e o Github em azul.

[![diagrama de fluxo oauth](https://res.cloudinary.com/practicaldev/image/fetch/s--fGhYT3Hr--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_880/https://dev-to-uploads.s3.amazonaws.com/i/45mff76o5qswqzrz6i82.png)](https://res.cloudinary.com/practicaldev/image/fetch/s--fGhYT3Hr--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_880/https://dev-to-uploads.s3.amazonaws.com/i/45mff76o5qswqzrz6i82.png)

# Créditos

- [Serverless OAuth with Multiple Providers](https://dev.to/rbethel/serverless-oauth-with-multiple-providers-1094), escrito originalmente por [Ryan Bethel](https://dev.to/rbethel).
