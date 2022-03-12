### .env

Para essa primeira etapa, tudo que você precisa fazer é duplicar o arquivo `env.example` presente na raiz do seu projeto, renomeá-lo para `.env` e preencher o campo `CLIENT_ID` com o valor do `ID do Cliente` que você anotou ao registrar sua aplicação no Console da Twitch Developers.

<aside>
⚠️ É importante lembrar que as variáveis ambiente só são carregadas no momento que você executa a aplicação. Então se seu app já estiver rodando, pare a execução no terminal e inicie novamente.

</aside>

### src/hooks/useAuth.tsx

<aside>
ℹ️ Se ao longo do fluxo de autenticação OAuth você ficar confuso com as etapas que seguimos, sugerimos a leitura da doc oficial:

[Autenticação](https://dev.twitch.tv/docs/authentication)
[Referência da API](https://dev.twitch.tv/docs/api)

</aside>

Nesse arquivo, você tem algumas etapas a seguir. A primeira é buscar o `CLIENT_ID` das variáveis ambiente e adicioná-lo no cabeçalho das suas requisições para a API da Twitch (é obrigatório ter essas informações para as requisições funcionarem). Exemplo:

```tsx
const { NOME_DA_VARIAVEL } = process.env;

useEffect(() => {
  api.defaults.headers['Client-Id'] = NOME_DA_VARIAVEL;
}, [])
```

Em seguida, iremos criar toda a lógica da função `signIn`, começando por dentro do bloco `try`. Realize o `set` do estado `isLoggingIn` como `true`. Na sequência, precisamos montar a url de autenticação com as seguintes inormações:

1. client_id: valor do `ID do Cliente` do seu app registrado na Twitch. Você já buscou esse valor na etapa anterior;
2. redirect_uri: URL de redirecionamento, deve ser a mesma informada no registro do seu app no Console da Twitch Developers. Para gerar esse valor, utilize esse `helper` do Expo: `makeRedirectUri({ useProxy: true });`
3. response_type: especifica o tipo de resposta que você espera. Sete esse valor como `token`
4. scope: permissões que você deseja solicitar do usuário ao realizar o login. Para o desafio funcionar bem, você deve solicitar os scopes `openid`, `user:read:email` e `user:read:follows`. Lembre de utlizar o `encodeURI` para conseguir gerar a listagem desses scopes separados por espaço.
5. force_verify: sempre solicitar a autorização do usuário ao logar no app. Sete esse valor como `true` para permitir que os usuários consigam trocar de conta caso desejem.
6. state: string aleatória que você deve gerar para aumentar a segurança do seu app. Utilize o `generateRandom` informando o valor `30` como argumento para gerar uma string aleatória com 30 caracteres de tamanho.

Dessa forma, o que sugerimos para você é criar as constantes para cada um desses parâmetros (com excessão do `client_id` pois você já pegou na etapa anterior) e juntá-las ao montar a url de autenticação. Essa url precisa começar com o endpoint de autenticação da Twitch (disponível na variável `twitchEndpoints.authorization`) e informar esses valores acima como `Query Params`. Exemplo:

```tsx
const REDIRECT_URI ...
const RESPONSE_TYPE ...
const SCOPE ...
const FORCE_VERIFY ...
const STATE ...

const authUrl = twitchEndpoints.authorization + 
`?client_id=${CLIENT_ID}` + 
`&redirect_uri=${REDIRECT_URI}` + 
`&response_type=${RESPONSE_TYPE}` + 
`&scope=${SCOPE}` + 
`&force_verify=${FORCE_VERIFY}` +
`&state=${STATE}`;
```

Com a url de autenticação montada, é preciso executar o método `startAsync` informando essa url e capturar a resposta obtida. Faça uma verificação se a resposta possui o `type` igual a `success` e o `params.error` diferente de `access_denied`. Caso não possua, simplesmente ignore. Se atender esses requisitos faça o seguinte:

- Comece verificando se o `params.state` da resposta é diferente do `STATE` que você criou para enviar na requisição. Se for, lance um erro com a mensagem `Invalid state value`. Se os valores forem iguais, simplesmente ignore.
- Adicionar o `params.access_token` no cabeçalho das suas requisições. Para implementar isso, faça:
    
    ```tsx
    api.defaults.headers.authorization = `Bearer ${authResponse.params.access_token}`;
    ```
    
- Em seguida, faça uma requisição à API da Twitch na rota `/users` para buscar as informações do usuário logado. Para implementar isso, faça:
    
    ```tsx
    const userResponse = await api.get('/users');
    ```
    
    <aside>
    ⚠️ Se você não adicionou o `access_token` e o `client_id` no cabeçalho das suas requisições, essa etapa vai falhar.
    
    </aside>
    
- Atribua ao estado `user` os valores recebidos da requisição anterior. A resposta dessa requisição vem com diversas informações, para achar a que interessa para você, acesse o objeto em `userResponse.data.data[0]`. Esse objeto vai ter o seguinte formato:
    
    ```json
    {
      "id": "141981764",
      "login": "twitchdev",
      "display_name": "TwitchDev",
      "type": "",
      "broadcaster_type": "partner",
      "description": "Supporting third-party developers building Twitch integrations from chatbots to game integrations.",
      "profile_image_url": "https://static-cdn.jtvnw.net/jtv_user_pictures/8a6381c7-d0c0-4576-b179-38bd5ce1d6af-profile_image-300x300.png",
      "offline_image_url": "https://static-cdn.jtvnw.net/jtv_user_pictures/3f13ab61-ec78-4fe6-8481-8682cb3b0ac2-channel_offline_image-1920x1080.png",
      "view_count": 5980557,
      "email": "not-real@email.com",
      "created_at": "2016-12-14T20:32:28.894263Z"
    }
    ```
    
- Por fim, atribua ao estado `userToken` o `params.access_token` para poder revogá-lo quando o usuário deslogar da aplicação.

Com isso, termina a lógica do seu bloco `try`.  Para finalizar essa função, dentro do `catch`, simplesmente lance um erro vazio e dentro do `finally` atribua ao estado `isLoggingIn` o valor `false`.

Para finalizar esse arquivo, iremos criar toda a lógica da função `signOut`, começando por dentro do bloco `try` que é bem curto. Realize o `set` do estado `isLoggingOut` como `true`. Por fim, execute o método `revokeAsync` como parâmetros dois objetos:

1. Nesse objeto, informe na propriedade `token` o estado `userToken` e na propriedade `clientId` o valor do `CLIENT_ID` recuperado das variáveis ambiente.
2. Nesse objeto, informe na propriedade `revocationEndpoint` a url da API da Twitch para revogar os tokens: `twitchEndpoints.revocation`.

No block `catch`, não precisa fazer nada. Já no bloco `finally`, você deve:

1. Limpar o estado `user` atribuindo um objeto `User` vazio;
2. Limpar o estado `userToken` atribuindo uma string vazia;
3. Remover o `access_token` do cabeçalho das requisições. Para implementar isso, utilize:
    
    ```tsx
    delete api.defaults.headers.authorization;
    ```
    
4. Atribuir o valor `false` ao estado `isLoggingOut`

### src/pages/SignIn/index.tsx

Nessa página, comece implementando a função `handleSignIn`. Essa função vai ser bem simples: um bloco try/catch com 1 linha cada. Dentro do `try`, chame e aguarde a função `signIn` finalizar. Dentro do `catch`, lance um `Alert` com o título `Erro SignIn` e mensagem `Ocorreu um erro ao tentar logar no app`.

Para finalizar, no `JSX` altere você deve descomentar a parte referente ao `SignInButton` e fazer três coisas:

1. Informar a função `handleSignIn` no método `onPress` do botão;
2. Dentro do `SignInButtonIcon`, verificar se a variável `isLoggingIn` é verdadeira. Se for, renderizar um `ActivityIndicator` de tamanho `20` e cor `theme.colors.white`. Caso contrário, renderizar o ícone do pacote `Fontisto` de nome `twitch`, tamanho `20`, cor `theme.colors.white` e `1px` de `marginRight`
3. Dentro do `SignInButtonText`, verificar se a variável `isLoggingIn` é verdadeira. Se for, renderizar o texto `Entrando...`. Caso contrário, renderizar o texto `Entrar com Twitch`.

### src/pages/Home/index.tsx

Nessa página, comece implementando a função `handleSignOut`. Essa função vai ser bem simples: um bloco try/catch com 1 linha cada. Dentro do `try`, chame e aguarde a função `signOut` finalizar. Dentro do `catch`, lance um `Alert` com o título `Erro SignOut` e mensagem `Ocorreu um erro ao tentar se deslogar do app`.

Para finalizar, no `JSX` você deve descomentar a parte referente ao `SignOutButton` e fazer duas coisas:

1. Informar a função `handleSignOut` no método `onPress` do botão;
2. Dentro do `SignOutButton`, verificar se a variável `isLoggingOut` é verdadeira. Se for, renderizar um `ActivityIndicator` de tamanho `25` e cor `theme.colors.white`. Caso contrário, renderizar o ícone do pacote `Feather` de nome `power`, tamanho `24` e cor `theme.colors.white`.