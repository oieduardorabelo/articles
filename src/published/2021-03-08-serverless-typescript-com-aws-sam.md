- https://dev.to/oieduardorabelo/serverless-typescript-com-aws-sam-2i48
- https://oieduardorabelo.medium.com/serverless-typescript-com-aws-sam-117805a2c5a4

---

# Serverless TypeScript com AWS SAM

### Aprenda a escrever Lambdas para AWS Serverless Application Model (SAM) em puro TypeScript sem a necessidade de comprometer seu fluxo de trabalho de desenvolvimento. Veja como confiar nas [camadas compartilhadas](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/building-layers.html) do SAM para empacotar suas dependências. Consulte nosso [repositório exemplo no GitHub](https://github.com/Envek/aws-sam-typescript-layers-example) para que você possa construir e implantar AWS Lambdas com TypeScript em um ambiente de produção completo.

Funções Serverless (ou [AWS Lambdas](https://aws.amazon.com/lambda/) na linguagem AWS) são uma ótima escolha quando a carga do seu aplicativo tende a ser altamente irregular e você deseja evitar o provisionamento de servidores e configuração de todo o ambiente para realizar algumas operações por alguns dias ou semanas por ano.

Existem muitas ferramentas para simplificar o desenvolvimento de funções serverless: [Serverless Framework](https://www.serverless.com/) suporta várias plataformas, o [AWS Serverless Application Model (SAM)](https://aws.amazon.com/serverless/sam/) que vem direto da AWS, entre outros.

O AWS SAM é uma ótima escolha se você já confia no ecossistema AWS. O SAM facilita a implantação de Lambdas junto com toda a infraestrutura em nuvem necessária (API Gateways, bancos de dados, filas, logs, etc.) com a ajuda de [templates do AWS CloudFormation](https://aws.amazon.com/cloudformation/resources/templates/), que são amplamente usados no ecosistema.

Embora o AWS Lambda suporte [muitas linguagens de programação](https://dashbird.io/blog/most-effictient-lambda-language/). Node.js continua sendo minha escolha número um pela riqueza de seu ecossistema npm, a abundância de documentação online e exemplos e tempos de inicialização estelares. No entanto, _escrever_ JavaScript puro é indiscutivelmente menos agradável do que _executar_ JavaScript puro. Felizmente, estamos em 2021 e podemos contar com o TypeScript para trazer a alegria de escrever JS.

**Há apenas um problema:** o AWS SAM não oferece suporte ao TypeScript automáticamente.

É uma grande desvantagem, mas não um motivo para desistir e jogar fora seus tipos e verificação de tempo de compilação pela janela. Vamos ver como podemos construir nós mesmos uma experiência agradável de SAM-TypeScript: do desenvolvimento local à implantação na nuvem. E não vamos recorrer a hacks amplamente recomendados. Vamos lá!

### O Bingo do TypeScript

Em meu mundo imaginário perfeito, é assim que _o_ suporte _adequado do_ TypeScript deve ser:

- Manter a experiência de desenvolvimento local praticamente inalterada: sem mover `package.json` para outros lugares ou alterar a estrutura do diretório antes do deploy.
- Sem executar `sam build` em cada mudança no código do manipulador da função
- Manter o código JS gerado o mais próximo possível da fonte TS, preservando o layout do arquivo (não empacote tudo em um único arquivo como o _webpack_ faz).
- Manter dependências em uma camada separada compartilhada entre Lambdas relacionadas. Tornando o deploy mais rápido, pois você só precisa atualizar o código da função e não suas dependências. Além disso, as funções Lambda têm um limite de tamanho que pode ser facilmente atingido com dependências pesadas; camadas compartilhadas nos permitem manter a coloração entre as linhas.
- Mantendo o deploy o mais simples possível: `sam build`e `sam deploy`, sem nenhuma mágica CLI extra.

Resumindo, uma AWS Lambda com TypeScript e camadas compartilhadas deve se comportar da mesma maneira que um AWS Lambda recém-gerado em Node.js.

Vamos ver como podemos conseguir isso!

## 1. Mova dependências para camadas compartilhadas

Eu revi alguns manuais sobre "como mover dependências Node.js para camadas Lambda" ( [1](https://aws.amazon.com/blogs/compute/working-with-aws-lambda-and-lambda-layers-in-aws-sam/) , [2](https://medium.com/@anjanava.biswas/nodejs-runtime-environment-with-aws-lambda-layers-f3914613e20e) ), mas não segui nenhum deles, pois eles propõem mover `package.json` da raiz do projeto para uma nova pasta, obrigatória, chamada `dependencies` e, assim, interromper o desenvolvimento e os testes locais.

Então me deparei com o documento oficial de [Construíndo Camadas](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/building-layers.html) da AWS e decidi substituir o processo de construção do AWS SAM com Node.js ( `sam build` copia o código automagicamente, instala pacotes e faz a limpeza, mas você não pode interferir em suas decisões) por um personalizado um, com base em um `Makefile`.

Primeiro, declaramos nossas próprias camadas e quaisquer camadas de terceiros dentro de `template.yml`:

```yaml
Globals:
  Function:
    Layers:
      # Nossa camada que iremos criar
      - !Ref RuntimeDependenciesLayer
      # Ao mesmo tempo, podemos referenciar camadas de terceiros
      - !Sub "arn:${AWS::Partition}:lambda:${AWS::Region}:464622532012:layer:Datadog-Node14-x:48"

  RuntimeDependenciesLayer:
    Type: AWS::Serverless::LayerVersion
    Metadata:
      BuildMethod: makefile # Aqui está o truque!
    Properties:
      Description: Runtime dependencies for Lambdas
      ContentUri: ./
      CompatibleRuntimes:
        - nodejs14.x
      RetentionPolicy: Retain
```

Esta seção `Metadata` é um ponto chave aqui. Nós o adicionamos não apenas à nossa camada, mas também aos nossos Lambdas:

```yaml
Metadata:
  BuildMethod: makefile
```

Podemos então escrever um simples `Makefile` que mantém apenas o código executável dentro de um Lamda e coloca todas as dependências e `node_modules` dentro de uma _camada_ separada que pode ser compartilhada com outras lambdas.

```makefile
.PHONY: build-ExampleLambda build-RuntimeDependenciesLayer

build-ExampleLambda:
	cp -r src "$(ARTIFACTS_DIR)/"

build-RuntimeDependenciesLayer:
	mkdir -p "$(ARTIFACTS_DIR)/nodejs"
	cp package.json package-lock.json "$(ARTIFACTS_DIR)/nodejs/"
	npm install --production --prefix "$(ARTIFACTS_DIR)/nodejs/"
	# Para evitar ter que fazer o build quando as mudanças não se relacionam com dependências
	rm "$(ARTIFACTS_DIR)/nodejs/package.json"
```

_Dica: se você estiver usando o Yarn, precisará alternar `--cwd` ao invés do npm`--prefix`_

Agora há muito mais observabilidade no processo de construção do Lambda, sem mágica!

No entanto, uma compilação baseada em Makefile também tem suas desvantagens: é difícil depurar seu processo de compilação. Consulte [aws / aws-sam-cli # 2006](https://github.com/aws/aws-sam-cli/issues/2006) para obter detalhes.

Veja [este commit](https://github.com/Envek/aws-sam-typescript-layers-example/commit/06598650aa8e7628b147ca8e770665f8d4f75786) em nosso [repositório examplo](https://github.com/Envek/aws-sam-typescript-layers-example) para mais detalhes.

## 2. Migrar para TypeScript

Agora, como temos um pipeline de construção personalizável, podemos finalmente adicionar a etapa de compilação do TypeScript.

Primeiro, vamos instalar o próprio TypeScript. Coloque os seguintes pacotes em seu `package.json` e defina alguns scripts para compilar seu TypeScript para desenvolvimento e produção:

```json
"dependencies": {
  "source-map-support": "^0.5.19"
},
"devDependencies": {
  "@tsconfig/node14": "^1.0.0",
  "@types/aws-lambda": "^8.10.72",
  "@types/node": "^14.14.26",
  "typescript": "^4.1.5"
},
"scripts": {
    "build": "node_modules/typescript/bin/tsc",
    "watch": "node_modules/typescript/bin/tsc -w --preserveWatchOutput"
}
```

Em segundo lugar, crie e configure o seu `tsconfig.json`:

```json
{
  "extends": "@tsconfig/node14/tsconfig.json",
  "compilerOptions": {
    "sourceMap": true,
    "moduleResolution": "node",
    "outDir": "./dist",
    "rootDir": "./src"
  },
  "include": ["src/**/*.ts", "src/**/*.js"]
}
```

Como o `nodejs14.x` no AWS Lambda funciona (obviamente) na última versão LTS do Node.js, podemos usar `"target": "es2020"` e `"lib": ["es2020"]` para construir o código JS que será muito, muito semelhante ao código TypeScript de origem, mantendo todos os `async`s e `await`s.

Agora você pode substituir seu `build-ExampleLamda` no Makefile pela definição que inclui as etapas de instalação e compilação:

```makefile
# Makefile
build-ExampleLambda:
	npm install
	npm run build
	cp -r dist "$(ARTIFACTS_DIR)/"
```

## 3. Usando "sam build" no desenvolvimento local

Os comandos como `sam local invoke` ou `sam local start-api` procuram primeiro a pasta `.aws-sam/` e, se não houver nenhuma, eles procuram pelo manipulador da lambda na pasta atual.

É importante remover o diretório gerado automaticamente `.aws-sam` após cada deploy, para que o SAM possa ver suas alterações locais sem ser executar `sam build` constantemente.

Precisamos apenas garantir que o código TypeScript compilado esteja localizado no mesmo caminho de uma lambda deployada _e_ localmente.

Por padrão, o código Node.js para uma lamda está localizado na pasta `src/`, mas agora contém nosso código TypeScript, portanto, precisamos colocar nosso código compilado em outro lugar. Vamos pegar emprestado uma convenção popular do pessoal do front-end e apresentar a pasta `dist` para o JavaScript final. Mude o seu `template.yml` para:

```git
--- a/template.yml
+++ b/template.yml
@@ -29,7 +29,7 @@ Resources:
     Metadata:
       BuildMethod: makefile
     Properties:
-      Handler: src/handlers/get-all-items.getAllItemsHandler
+      Handler: dist/handlers/get-all-items.getAllItemsHandler
```

Tudo que você precisa agora é iniciar seu compilador TypeScript no modo de observação, para que seu código compile magicamente em cada alteração (ou não, mas então você saberá imediatamente o porquê). Felizmente, já cuidamos disso em nosso `package.json`. Apenas não se esqueça de executar isso em seu terminal antes de começar a codificar (ou iniciar uma tarefa de observação em seu IDE favorito).

```bash
$ npm run watch
```

Você não precisa mais executar localmente `sam build`, e comandos como `sam local start-api` poderão ver suas alterações imediatamente (porque eles apontam para o código transpilado, não para o código-fonte).

E se você tiver um inicializador `Procfile` como o [Overmind](https://evilmartians.com/chronicles/introducing-overmind-and-hivemind) (qualidade marciana, altamente recomendado!), Você pode configurar a inicialização do compilador SAM e TS em paralelo em `Procfile`:

```makefile
# Procfile
sam: sam local start-api
tsc: npm run watch
```

Em seguida, use-o para iniciar o compilador TypeScript no modo de observação _e_ um API Gateway local como dois processos simultâneos:

```
$ overmind start
```

E é isso!

Veja um resumo das mudanças que fizemos nesta etapa deste [commit](https://github.com/Envek/aws-sam-typescript-layers-example/commit/6876e72b1605b6864edcc9ff2b7e28f8ae3e47d4) em nosso repositório de exemplo.

## 4. Escrevendo seu código

Também é uma boa ideia ativar o suporte a mapas de origem, portanto, podemos rastrear de pilha do TypeScript em caso de qualquer erro:

```js
import "source-map-support/register";
```

Também precisamos incluir os tipos específicos da AWS em nossos manipuladores:

```js
import { APIGatewayProxyEvent, APIGatewayProxyResult } from "aws-lambda";
```

E declare os manipuladores que os usam:

```js
export const getAllItemsHandler = async (
  event: APIGatewayProxyEvent
): Promise<APIGatewayProxyResult> => {
  // ...mais código
};
```

Veja o exemplo completo de um manipulador digitado corretamente [neste](https://github.com/Envek/aws-sam-typescript-layers-example/commit/81b283071f4769135cfbf6033977e061f3b9383b) exemplo.

## 5. Configurando testes

1.  Adicione Jest com suporte TypeScript ao seu `package.json`:

```json
"devDependencies": {
  "@types/jest": "^26.0.20",
  "jest": "^26.6.3",
  "ts-jest": "^26.5.1",
},
```

2.  Adicione a configuração relacionada ao TypeScript ao seu `jest.config.js`

```js
module.exports = {
  preset: "ts-jest",
  modulePathIgnorePatterns: ["<rootDir>/.aws-sam"],
};
```

E é isso! Reescreva seus testes em TS e execute-os com `npm t`, como você fez antes.

## 6. Depuração (Debug)

Com essa abordagem, a depuração não só é possível, mas também funciona imediatamente!

Você pode usar um depurador externo seguindo este manual da AWS: [Depuração passo a passo de funções Node.js localmente](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/serverless-sam-cli-using-debugging-nodejs.html) .

1.  Execute `sam local invoke` com a opção `--debug-port`.

```bash
$ sam local invoke getAllItemsFunction --event events/event-get-all-items.json --debug-port 5858
```

Isso aguardará a conexão de um depurador antes de iniciar a execução da função.

1.  Coloque um ponto de interrupção onde necessário (sim, direto no seu código TypeScript!)
2.  Inicie o depurador externo (no Visual Studio Code, você pode simplesmente pressionar F5).

E aqui está um exemplo de código VSCode `.vscode/launch.json`:

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Attach to SAM CLI",
      "type": "node",
      "request": "attach",
      "address": "localhost",
      "port": 5858,
      "localRoot": "${workspaceRoot}/",
      "remoteRoot": "/var/task",
      "protocol": "inspector",
      "stopOnEntry": false
    }
  ]
}
```

## Bônus: Coloque apenas o código relevante em suas funções Lambdas

Embora seja conveniente ter muitas funções Lambda relacionadas juntas em um projeto SAM com configuração comum, dependências e código utilitário comum (reutilizado por algumas, mas não todas as funções), parece redundante fazer o deploy de todas as funções de uma vez quando você muda um pouco de código que é usado apenas por uma delas.

Quero que cada função use apenas os arquivos que são realmente usados ​​por esta função, sem importações estranhas. Esse é o meu maximalismo estético puro, já que não faz mal importar todos os arquivos em todas as funções: os arquivos de código-fonte são muito leves. Mas o TypeScript nos permite alcançar a _pureza_ ! Quando o compilador TypeScript recebe um caminho para um único arquivo, ele apenas compila esse arquivo e suas dependências, nada mais.

Dado que cada função do Lambda tem um _manipulador_ (uma única função de ponto de entrada em um único arquivo), podemos compilar apenas esse manipulador, e isso nos fornecerá apenas os arquivos necessários para executar um determinado Lambda.

Isso nos permite economizar nas linhas do código-fonte e (mais importante) fazer o deploy apenas das funções que usam o código alterado.

Porém, para fazer o TSC honrar nossa configuração `tsconfig.json`, precisamos de um pequeno hack (veja [microsoft / TypeScript # 27379 (comentário)](https://github.com/microsoft/TypeScript/issues/27379#issuecomment-743393449) para mais detalhes).

Aí vem:

```
build-lambda-common:
	npm install
	rm -rf dist
	echo "{\"extends\": \"./tsconfig.json\", \"include\": [\"${HANDLER}\"] }" > tsconfig-only-handler.json
	npm run build -- --build tsconfig-only-handler.json
	cp -r dist "$(ARTIFACTS_DIR)/"

build-getAllItemsFunction:
	$(MAKE) HANDLER=src/handlers/get-all-items.ts build-lambda-common
build-getByIdFunction:
	$(MAKE) HANDLER=src/handlers/get-by-id.ts build-lambda-common
build-putItemFunction:
	$(MAKE) HANDLER=src/handlers/put-item.ts build-lambda-common

```

E agora para a seguinte estrutura de arquivo do nosso projeto:

```
.
└── src
    ├── handlers
    │   ├── a.ts
    │   ├── b.ts
    │   └── c.ts
    └── utils
        ├── ab.ts
        └── bc.ts

```

Obteremos as três funções lambda a seguir:

```
.aws-sam/build/FunctionA
└── dist
    ├── handlers
    │   ├── a.js
    │   └── a.js.map
    └── utils
        ├── ac.js
        └── ac.js.map

.aws-sam/build/FunctionB
└── dist
    ├── handlers
    │   ├── b.js
    │   └── b.js.map
    └── utils
        ├── ab.js
        ├── ab.js.map
        ├── bc.js
        └── bc.js.map

.aws-sam/build/FunctionC
└── dist
    ├── handlers
    │   ├── c.js
    │   └── c.js.map
    └── utils
        ├── bc.js
        └── bc.js.map

```

Se mudarmos apenas o `src/handlers/a.ts`, apenas o recurso `FunctionA`será reimplantado. E se mudarmos `src/utils/bc.ts` (importado nos manipuladores `b` e `c`), apenas `FunctionB`e `FunctionC` serão reimplantados. Maneiro né?

Veja o commit completo [aqui](https://github.com/Envek/aws-sam-typescript-layers-example/commit/15fc870eefaca51b646544f0df6c19cf3524af81) .

## Resumindo

- Precisamos executar `tsc -w` quando codificamos, mas, bem, é inevitável e torna a vida mais divertida.
- Nossos testes estão funcionando bem.
- Nossos Lambdas são os menores possíveis: a pasta `node_modules` está dentro da camada externa compartilhada e cada Lambda inclui apenas o código real de que precisa para _funcionar_ (entendeu o trocadilho?).
- Podemos colocar pontos de depuração diretamente em um código TypeScript.
- Não estamos reinventando o SAM, mas configurando-o para atender às nossas necessidades.
- O procedimento de deploy não mudou em nada!

Essa configuração não é a ideal, mas se adapta muito bem às nossas necessidades. Se você tiver algo a acrescentar, mencione [@evilmartians](https://twitter.com/evilmartians) em um tweet (ou apenas [abra um PR](https://github.com/Envek/aws-sam-typescript-layers-example/compare) no repositório de exemplo).

## Mostre-me seu código!

Se você é novo em lamdas, oferecemos o modelo de aplicativo SAM totalmente configurado com API Gateway, banco de dados, filas, tudo junto com algumas funções serverless CRUD de demonstração. Tudo que você precisa para experimentar é uma conta da AWS:

```bash
gh repo clone Envek/aws-sam-typescript-layers-example
sam build
sam deploy --guided
```

Confira os [nossos commits](https://github.com/Envek/aws-sam-typescript-layers-example/commits/master) individuais para entender melhor as coisas.

Se você deseja apenas começar a desenvolver seus Lambdas com isso - aqui está o modelo para começar:

```
sam init --location gh:Envek/cookiecutter-aws-sam-typescript-layers
```

E você está pronto para implantar AWS Lambda com TypeScript?

# Créditos

- [Serverless TypeScript: A complete setup for AWS SAM Lambdas](https://evilmartians.com/chronicles/serverless-typescript-a-complete-setup-for-aws-sam-lambda), escrito originalmente por [Andrey Novikov](https://twitter.com/Envek) e [Sergey Alexandrovich](https://twitter.com/darth_sim).
