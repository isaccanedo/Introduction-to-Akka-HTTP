## Akka HTTP

Introdução ao Akka HTTP

# 1. Visão Geral
Neste tutorial, com a ajuda dos modelos Actor & Stream da Akka, aprenderemos como configurar a Akka para criar uma API HTTP que fornece operações CRUD básicas.

# 2. Dependências Maven
Para começar, vamos dar uma olhada nas dependências necessárias para começar a trabalhar com Akka HTTP:

```
<dependency>
    <groupId>com.typesafe.akka</groupId>
    <artifactId>akka-http_2.12</artifactId>
    <version>10.0.11</version>
</dependency>
<dependency>
    <groupId>com.typesafe.akka</groupId>
    <artifactId>akka-stream_2.12</artifactId>
    <version>2.5.11</version>
</dependency>
<dependency>
    <groupId>com.typesafe.akka</groupId>
    <artifactId>akka-http-jackson_2.12</artifactId>
    <version>10.0.11</version>
</dependency>
<dependency>
    <groupId>com.typesafe.akka</groupId>
    <artifactId>akka-http-testkit_2.12</artifactId>
    <version>10.0.11</version>
    <scope>test</scope>
</dependency>
```

Podemos, é claro, encontrar a versão mais recente dessas bibliotecas Akka no Maven Central.

# 3. Criação de um ator
Como exemplo, construiremos uma API HTTP que nos permite gerenciar os recursos do usuário. A API suportará duas operações:

- criar um novo usuário
- carregar um usuário existente

Antes de fornecer uma API HTTP, precisaremos implementar um ator que forneça as operações de que precisamos:

```
class UserActor extends AbstractActor {

  private UserService userService = new UserService();

  static Props props() {
    return Props.create(UserActor.class);
  }

  @Override
  public Receive createReceive() {
    return receiveBuilder()
      .match(CreateUserMessage.class, handleCreateUser())
      .match(GetUserMessage.class, handleGetUser())
      .build();
  }

  private FI.UnitApply<CreateUserMessage> handleCreateUser() {
    return createUserMessage -> {
      userService.createUser(createUserMessage.getUser());
      sender()
        .tell(new ActionPerformed(
           String.format("User %s created.", createUserMessage.getUser().getName())), getSelf());
    };
  }

  private FI.UnitApply<GetUserMessage> handleGetUser() {
    return getUserMessage -> {
      sender().tell(userService.getUser(getUserMessage.getUserId()), getSelf());
    };
  }
}
```

Basicamente, estamos estendendo a classe AbstractActor e implementando seu método createReceive ().

Em createReceive (), estamos mapeando tipos de mensagens de entrada para métodos que manipulam mensagens do respectivo tipo.

Os tipos de mensagem são classes simples de contêiner serializáveis ​​com alguns campos que descrevem uma determinada operação. GetUserMessage e tem um único campo userId para identificar o usuário a ser carregado. CreateUserMessage contém um objeto User com os dados do usuário de que precisamos para criar um novo usuário.

Mais tarde, veremos como traduzir as solicitações HTTP de entrada nessas mensagens.

Por fim, delegamos todas as mensagens a uma instância de UserService, que fornece a lógica de negócios necessária para gerenciar objetos de usuário persistentes.

Além disso, observe o método props (). Embora o método props () não seja necessário para estender AbstractActor, ele será útil posteriormente ao criar o ActorSystem.

Para uma discussão mais aprofundada sobre atores, dê uma olhada em nossa introdução aos Atores Akka.

# 4. Definindo Rotas HTTP
Tendo um ator que faz o trabalho real para nós, tudo o que nos resta fazer é fornecer uma API HTTP que delega as solicitações HTTP de entrada para nosso ator.

Akka usa o conceito de rotas para descrever uma API HTTP. Para cada operação, precisamos de uma rota.

Para criar um servidor HTTP, estendemos a classe de estrutura HttpApp e implementamos o método de rotas:

```
class UserServer extends HttpApp {

  private final ActorRef userActor;

  Timeout timeout = new Timeout(Duration.create(5, TimeUnit.SECONDS));

  UserServer(ActorRef userActor) {
    this.userActor = userActor;
  }

  @Override
  public Route routes() {
    return path("users", this::postUser)
      .orElse(path(segment("users").slash(longSegment()), id -> route(getUser(id))));
  }

  private Route getUser(Long id) {
    return get(() -> {
      CompletionStage<Optional<User>> user = 
        PatternsCS.ask(userActor, new GetUserMessage(id), timeout)
          .thenApply(obj -> (Optional<User>) obj);

      return onSuccess(() -> user, performed -> {
        if (performed.isPresent())
          return complete(StatusCodes.OK, performed.get(), Jackson.marshaller());
        else
          return complete(StatusCodes.NOT_FOUND);
      });
    });
  }

  private Route postUser() {
    return route(post(() -> entity(Jackson.unmarshaller(User.class), user -> {
      CompletionStage<ActionPerformed> userCreated = 
        PatternsCS.ask(userActor, new CreateUserMessage(user), timeout)
          .thenApply(obj -> (ActionPerformed) obj);

      return onSuccess(() -> userCreated, performed -> {
        return complete(StatusCodes.CREATED, performed, Jackson.marshaller());
      });
    })));
  }
}
```

Agora, há uma boa quantidade de clichês aqui, mas observe que seguimos o mesmo padrão de antes das operações de mapeamento, desta vez como rotas. Vamos decompô-lo um pouco.

Em getUser (), simplesmente envolvemos o ID do usuário de entrada em uma mensagem do tipo GetUserMessage e encaminhamos essa mensagem para nosso userActor.

Depois que o ator processa a mensagem, o manipulador onSuccess é chamado, no qual concluímos a solicitação HTTP enviando uma resposta com um determinado status HTTP e um determinado corpo JSON. Usamos o Jackson marshaller para serializar a resposta dada pelo ator em uma string JSON.

Em postUser (), fazemos as coisas de maneira um pouco diferente, pois esperamos um corpo JSON na solicitação HTTP. Usamos o método entity () para mapear o corpo JSON de entrada em um objeto User antes de envolvê-lo em um CreateUserMessage e passá-lo para nosso ator. Novamente, usamos Jackson para mapear entre Java e JSON e vice-versa.

Como o HttpApp espera que forneçamos um único objeto de rota, combinamos ambas as rotas em uma única dentro do método de rotas. Aqui, usamos a diretiva de caminho para finalmente fornecer o caminho de URL no qual nossa API deve estar disponível.

Vinculamos a rota fornecida por postUser () ao caminho / usuários. Se a solicitação recebida não for uma solicitação POST, a Akka irá automaticamente para o branch orElse e esperará que o caminho seja / users / <id> e o método HTTP seja GET.

Se o método HTTP for GET, a solicitação será encaminhada para a rota getUser (). Se o usuário não existir, a Akka retornará o status HTTP 404 (não encontrado). Se o método não for POST nem GET, a Akka retornará o status HTTP 405 (Método não permitido).

Para obter mais informações sobre como definir rotas HTTP com a Akka, dê uma olhada nos documentos da Akka.

# 5. Iniciando o servidor
Depois de criar uma implementação HttpApp como acima, podemos iniciar nosso servidor HTTP com algumas linhas de código:

```
public static void main(String[] args) throws Exception {
  ActorSystem system = ActorSystem.create("userServer");
  ActorRef userActor = system.actorOf(UserActor.props(), "userActor");
  UserServer server = new UserServer(userActor);
  server.startServer("localhost", 8080, system);
}
```

Simplesmente criamos um ActorSystem com um único ator do tipo UserActor e iniciamos o servidor no localhost.

# 6. Conclusão
Neste artigo, aprendemos sobre os fundamentos do Akka HTTP com um exemplo que mostra como configurar um servidor HTTP e expor endpoints para criar e carregar recursos, semelhante a uma API REST.