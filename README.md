# Usando Kafka com Spring Boot

Neste artigo, veremos como integrar um aplicativo Spring Boot ao Apache Kafka e começar a enviar e consumir mensagens de nosso aplicativo. Passaremos por cada seção com exemplos de código.

### Por que Kafka?
As filas de mensagens tradicionais, como ActiveMQ, RabbitMQ, podem lidar com alto rendimento geralmente usado para trabalhos de longa duração ou em segundo plano e comunicação entre serviços.

Kafka é uma plataforma de processamento de fluxo construída pelo LinkedIn e atualmente desenvolvida sob a égide da Apache Software Foundation. O Kafka visa fornecer ingestão de baixa latência de grandes quantidades de dados de eventos.

Podemos usar o Kafka quando temos que mover uma grande quantidade de dados e processá-los em tempo real. Um exemplo seria quando queremos processar o comportamento do usuário em nosso site para gerar sugestões de produtos ou monitorar eventos produzidos por nossos microsserviços.

Kafka é construído do zero com escala horizontal em mente. Podemos dimensionar adicionando mais agentes ao cluster Kafka existente.

### Vocabulário Kafka
Vejamos as principais terminologias de Kafka:

```
- Produtor: Um produtor é um cliente que envia mensagens ao servidor Kafka para o tópico especificado.

- Consumidor: Consumidores são os destinatários que recebem mensagens do servidor Kafka.

- Broker: Brokers podem criar um cluster Kafka compartilhando informações usando o Zookeeper. Um broker recebe mensagens de produtores e consumidores buscam mensagens do broker por tópico, partição e deslocamento.

- Cluster: Kafka é um sistema distribuído. Um cluster Kafka contém vários agentes que compartilham a carga de trabalho.

- Tópico: Um tópico é um nome de categoria na qual as mensagens são publicadas e da qual os consumidores podem receber mensagens.

- Partição: As mensagens publicadas em um tópico são espalhadas por um cluster Kafka em várias partições. Cada partição pode ser associada a um broker para permitir que os consumidores leiam de um tópico em paralelo.

- Offset: Offset é um ponteiro para a última mensagem que o Kafka já enviou para um consumidor.
```

### Configurando um cliente Kafka
Devemos ter um servidor Kafka rodando em nossa máquina. Se você não tiver a configuração do Kafka em seu sistema, dê uma olhada no guia de início rápido do Kafka. Assim que tivermos um servidor Kafka funcionando, um cliente Kafka pode ser facilmente configurado com a configuração do Spring em Java ou ainda mais rápido com o Spring Boot.

Vamos começar adicionando a dependência spring-kafka ao nosso pom.xml:

```
<dependency>
  <groupId>org.springframework.kafka</groupId>
  <artifactId>spring-kafka</artifactId>
  <version>2.5.2.RELEASE</version>
</dependency>
```

### Usando a configuração Java
Vamos agora ver como configurar um cliente Kafka usando a configuração Java do Spring. Para dividir as responsabilidades, separamos KafkaProducerConfig e KafkaConsumerConfig.

Vamos dar uma olhada na configuração do produtor primeiro:

```
@Configuration
class KafkaProducerConfig {

  @Value("${io.reflectoring.kafka.bootstrap-servers}")
  private String bootstrapServers;

  @Bean
  public Map<String, Object> producerConfigs() {
    Map<String, Object> props = new HashMap<>();
    props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG,
      bootstrapServers);
    props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG,
      StringSerializer.class);
    props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG,
      StringSerializer.class);
    return props;
  }

  @Bean
  public ProducerFactory<String, String> producerFactory() {
    return new DefaultKafkaProducerFactory<>(producerConfigs());
  }

  @Bean
  public KafkaTemplate<String, String> kafkaTemplate() {
    return new KafkaTemplate<>(producerFactory());
  }
}
```

O exemplo acima mostra como configurar o produtor Kafka para enviar mensagens. ProducerFactory é responsável por criar instâncias do Kafka Producer.

KafkaTemplate nos ajuda a enviar mensagens para seus respectivos tópicos. Veremos mais sobre o KafkaTemplate na seção de envio de mensagens.

Em produtorConfigs() estamos configurando algumas propriedades:

```
- BOOTSTRAP_SERVERS_CONFIG - Host e porta em que o Kafka está sendo executado.
- KEY_SERIALIZER_CLASS_CONFIG - Classe de serializador a ser usada para a chave.
- VALUE_SERIALIZER_CLASS_CONFIG - Classe de serializador a ser usada para o valor. Estamos usando StringSerializer para chaves e valores.
```

Agora que nossa configuração do produtor está pronta, vamos criar uma configuração para o consumidor:

```
@Configuration
class KafkaConsumerConfig {

  @Value("${io.reflectoring.kafka.bootstrap-servers}")
  private String bootstrapServers;

  @Bean
  public Map<String, Object> consumerConfigs() {
    Map<String, Object> props = new HashMap<>();
    props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG,
      bootstrapServers);
    props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG,
      StringDeserializer.class);
    return props;
  }

  @Bean
  public ConsumerFactory<String, String> consumerFactory() {
    return new DefaultKafkaConsumerFactory<>(consumerConfigs());
  }

  @Bean
  public KafkaListenerContainerFactory<ConcurrentMessageListenerContainer<String, String>> kafkaListenerContainerFactory() {
    ConcurrentKafkaListenerContainerFactory<String, String> factory =
      new ConcurrentKafkaListenerContainerFactory<>();
    factory.setConsumerFactory(consumerFactory());
    return factory;
  }
}
```

Usamos ConcurrentKafkaListenerContainerFactory para criar contêineres para métodos anotados com @KafkaListener. O KafkaListenerContainer recebe todas as mensagens de todos os tópicos ou partições em uma única thread. Veremos mais sobre os contêineres do ouvinte de mensagens na seção de mensagens de consumo.

### Usando a configuração automática do Spring Boot
O Spring Boot faz a maior parte da configuração automaticamente, para que possamos nos concentrar em construir os ouvintes e produzir as mensagens. Ele também oferece a opção de substituir a configuração padrão por meio de application.properties. A configuração do Kafka é controlada pelas propriedades de configuração com o prefixo spring.kafka.*:

```
spring.kafka.bootstrap-servers=localhost:9092
spring.kafka.consumer.group-id=myGroup
```

### Criando tópicos Kafka
Um tópico deve existir para começar a enviar mensagens para ele. Vamos agora dar uma olhada em como podemos criar tópicos Kafka:

```
@Configuration
class KafkaTopicConfig {

  @Bean
  public NewTopic topic1() {
    return TopicBuilder.name("reflectoring-1").build();
  }

  @Bean
  public NewTopic topic2() {
    return TopicBuilder.name("reflectoring-2").build();
  }  
}
```

Um bean Ka é responsável por criar novos tópicos Admin em nosso corretor. Com o Spring Boot, um bean KafkaAdmin é registrado automaticamente.

Para um aplicativo não Spring Boot, temos que registrar manualmente o bean KafkaAdmin:

```
@Bean
KafkaAdmin admin() {
  Map<String, Object> configs = new HashMap<>();
  configs.put(AdminClientConfig.BOOTSTRAP_SERVERS_CONFIG, ...);
  return new KafkaAdmin(configs);
}
```

Para criar um tópico, registramos um bean NewTopic para cada tópico no contexto da aplicação. Se o tópico já existir, o bean será ignorado. Podemos usar o TopicBuilder para criar esses beans. O KafkaAdmin também aumenta o número de partições se descobrir que um tópico existente tem menos partições do que NewTopic.numPartitions.

# Enviando mensagens
### Usando o KafkaTemplate
KafkaTemplate fornece métodos convenientes para enviar mensagens para tópicos:

```
@Component
class KafkaSenderExample {

  private KafkaTemplate<String, String> kafkaTemplate;
  ...

  @Autowired
  KafkaSenderExample(KafkaTemplate<String, String> kafkaTemplate, ...) {
    this.kafkaTemplate = kafkaTemplate;
    ...
  }

  void sendMessage(String message, String topicName) {
    kafkaTemplate.send(topicName, message);
  } 
}
```

Tudo o que precisamos fazer é chamar o método sendMessage() com a mensagem e o nome do tópico como parâmetros.

O Spring Kafka também nos permite configurar um retorno de chamada assíncrono:

```
@Component
class KafkaSenderExample {
  ...
  void sendMessageWithCallback(String message) {
    ListenableFuture<SendResult<String, String>> future = 
      kafkaTemplate.send(topic1, message);
  
    future.addCallback(new ListenableFutureCallback<SendResult<String, String>>() {
      @Override
      public void onSuccess(SendResult<String, String> result) {
        LOG.info("Message [{}] delivered with offset {}",
          message,
          result.getRecordMetadata().offset());
      }
  
      @Override
      public void onFailure(Throwable ex) {
        LOG.warn("Unable to deliver message [{}]. {}", 
          message,
          ex.getMessage());
      }
    });
  }
}
```

O método send() do KafkaTemplate retorna um ListenableFuture<SendResult>. Podemos registrar um ListenableFutureCallback com o ouvinte para receber o resultado do envio e fazer algum trabalho dentro de um contexto de execução.

Se não quisermos trabalhar com Futuros, podemos registrar um ProducerListener:

```
@Configuration
class KafkaProducerConfig {
  @Bean
  KafkaTemplate<String, String> kafkaTemplate() {
  KafkaTemplate<String, String> kafkaTemplate = 
    new KafkaTemplate<>(producerFactory());
  ...
  kafkaTemplate.setProducerListener(new ProducerListener<String, String>() {
    @Override
    public void onSuccess(
      ProducerRecord<String, String> producerRecord, 
      RecordMetadata recordMetadata) {
      
      LOG.info("ACK from ProducerListener message: {} offset:  {}",
        producerRecord.value(),
        recordMetadata.offset());
    }
  });
  return kafkaTemplate;
  }
}
```

Configuramos o KafkaTemplate com um ProducerListener que nos permite implementar os métodos onSuccess() e onError().

### Usando RoteamentoKafkaTemplate
Podemos usar RoutingKafkaTemplate quando temos vários produtores com configurações diferentes e queremos selecionar o produtor em tempo de execução com base no nome do tópico.

```
@Configuration
class KafkaProducerConfig {
  ...

  @Bean
  public RoutingKafkaTemplate routingTemplate(GenericApplicationContext context) {
    // ProducerFactory with Bytes serializer
    Map<String, Object> props = new HashMap<>();
    props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, 
      bootstrapServers);
    props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG,
      StringSerializer.class);
    props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG,
      ByteArraySerializer.class);
    DefaultKafkaProducerFactory<Object, Object> bytesPF = 
      new DefaultKafkaProducerFactory<>(props);
    context.registerBean(DefaultKafkaProducerFactory.class, "bytesPF", bytesPF);

    // ProducerFactory with String serializer
    props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, 
      StringSerializer.class);
    DefaultKafkaProducerFactory<Object, Object> stringPF = 
      new DefaultKafkaProducerFactory<>(props);

    Map<Pattern, ProducerFactory<Object, Object>> map = new LinkedHashMap<>();
    map.put(Pattern.compile(".*-bytes"), bytesPF);
    map.put(Pattern.compile("reflectoring-.*"), stringPF);
    return new RoutingKafkaTemplate(map);
  }
}
```

RoutingKafkaTemplate pega um mapa de instâncias java.util.regex.Pattern e ProducerFactory e encaminha mensagens para o primeiro ProducerFactory que corresponde a um determinado nome de tópico. Se tivermos dois padrões ref.* e reflectoring-.*, o padrão reflectoring-.* deve estar no início porque o padrão ref.* o substituiria, caso contrário.

No exemplo acima, criamos dois padrões .*-bytes e reflectoring-.*. Os nomes de tópicos que terminam com '-bytes' e começam com reflectoring-.* usarão ByteArraySerializer e StringSerializer respectivamente quando usarmos a instância RoutingKafkaTemplate.

# Consumindo mensagens
### Ouvinte de mensagens
Um KafkaMessageListenerContainer recebe todas as mensagens de todos os tópicos em um único thread.

Um ConcurrentMessageListenerContainer atribui essas mensagens a várias instâncias KafkaMessageListenerContainer para fornecer recursos multiencadeados.

### Usando @KafkaListener no nível do método
A anotação @KafkaListener nos permite criar ouvintes:

```
@Component
class KafkaListenersExample {

  Logger LOG = LoggerFactory.getLogger(KafkaListenersExample.class);

  @KafkaListener(topics = "reflectoring-1")
  void listener(String data) {
    LOG.info(data);
  }

  @KafkaListener(
    topics = "reflectoring-1, reflectoring-2", 
    groupId = "reflectoring-group-2")
  void commonListenerForMultipleTopics(String message) {
    LOG.info("MultipleTopicListener - {}", message);
  }
}
```

Para usar esta anotação devemos adicionar a anotação @EnableKafka em uma de nossas classes @Configuration. Além disso, requer uma fábrica de contêiner de ouvinte, que configuramos em KafkaConsumerConfig.java.

Usar @KafkaListener tornará esse método de bean um ouvinte e envolverá o bean em MessagingMessageListenerAdapter. Também podemos especificar vários tópicos para um único ouvinte usando o atributo topic conforme mostrado acima.

Usando @KafkaListener no nível da classe
Também podemos usar a anotação @KafkaListener no nível da classe. Se fizermos isso, precisamos especificar @KafkaHandler no nível do método:

```
@Component
@KafkaListener(id = "class-level", topics = "reflectoring-3")
class KafkaClassListener {
  ...

  @KafkaHandler
  void listen(String message) {
    LOG.info("KafkaHandler[String] {}", message);
  }

  @KafkaHandler(isDefault = true)
  void listenDefault(Object object) {
    LOG.info("KafkaHandler[Default] {}", object);
  }
}
```

Quando o ouvinte recebe mensagens, ele as converte nos tipos de destino e tenta corresponder esse tipo às assinaturas de método para descobrir qual método chamar.

No exemplo, mensagens do tipo String serão recebidas por listen() e do tipo Object serão recebidas por listenDefault(). Sempre que não houver correspondência, o manipulador padrão (definido por isDefault=true) será chamado.

Consumindo mensagens de uma partição específica com um deslocamento inicial
Podemos configurar ouvintes para ouvir vários tópicos, partições e um deslocamento inicial específico.

Por exemplo, se quisermos receber todas as mensagens enviadas para um tópico desde o momento de sua criação na inicialização do aplicativo, podemos definir o deslocamento inicial para zero:

```
@Component
class KafkaListenersExample {
  ...

  @KafkaListener(
    groupId = "reflectoring-group-3",
    topicPartitions = @TopicPartition(
      topic = "reflectoring-1",
      partitionOffsets = { @PartitionOffset(
        partition = "0", 
        initialOffset = "0") }))
  void listenToPartitionWithOffset(
    @Payload String message,
    @Header(KafkaHeaders.RECEIVED_PARTITION_ID) int partition,
    @Header(KafkaHeaders.OFFSET) int offset) {
      LOG.info("Received message [{}] from partition-{} with offset-{}", 
        message, 
        partition, 
        offset);
  }
}
```

Como especificamos initialOffset = "0", receberemos todas as mensagens a partir do offset 0 toda vez que reiniciarmos a aplicação.

Também podemos recuperar alguns metadados úteis sobre a mensagem consumida usando a anotação @Header().

### Filtrando mensagens
O Spring fornece uma estratégia para filtrar mensagens antes que elas cheguem aos nossos ouvintes:

```
class KafkaConsumerConfig {

  @Bean
  KafkaListenerContainerFactory<ConcurrentMessageListenerContainer<String, String>>
  kafkaListenerContainerFactory() {
    ConcurrentKafkaListenerContainerFactory<String, String> factory =
      new ConcurrentKafkaListenerContainerFactory<>();
    factory.setConsumerFactory(consumerFactory());
    factory.setRecordFilterStrategy(record -> 
      record.value().contains("ignored"));
    return factory;
  }
}
```

O Spring envolve o ouvinte com um FilteringMessageListenerAdapter. É preciso uma implementação de RecordFilterStrategy na qual implementamos o método de filtro. As mensagens que corresponderem ao filtro serão descartadas antes de chegarem ao ouvinte.

No exemplo acima, adicionamos um filtro para descartar as mensagens que contêm a palavra “ignorado”.

### Respondendo com @SendTo
Spring permite enviar o valor de retorno do método para o destino especificado com @SendTo:

```
@Component
class KafkaListenersExample {
  ...

  @KafkaListener(topics = "reflectoring-others")
  @SendTo("reflectoring-1")
  String listenAndReply(String message) {
    LOG.info("ListenAndReply [{}]", message);
    return "This is a reply sent after receiving message";
  }
}
```

A configuração padrão do Spring Boot nos fornece um modelo de resposta. Como estamos substituindo a configuração de fábrica acima, a fábrica de contêiner do ouvinte deve ser fornecida com um KafkaTemplate usando setReplyTemplate() que é usado para enviar a resposta.

No exemplo acima, estamos enviando a mensagem de resposta ao tópico “reflectoring-1”.

### Mensagens personalizadas
Vamos agora ver como enviar/receber um objeto Java. Estaremos enviando e recebendo objetos User em nosso exemplo.

```
class User {
  private String name;
}
```

### Configurando o serializador e desserializador JSON
Para conseguir isso, devemos configurar nosso produtor e consumidor para usar um serializador e desserializador JSON:

```
@Configuration
class KafkaProducerConfig {
  ...

  @Bean
  public ProducerFactory<String, User> userProducerFactory() {
    ...
    configProps.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, 
      JsonSerializer.class);
    return new DefaultKafkaProducerFactory<>(configProps);
  }

  @Bean
  public KafkaTemplate<String, User> userKafkaTemplate() {
    return new KafkaTemplate<>(userProducerFactory());
  }
}
```

```
@Configuration
class KafkaConsumerConfig {
  ...
  public ConsumerFactory<String, User> userConsumerFactory() {
    Map<String, Object> props = new HashMap<>();
    props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
    props.put(ConsumerConfig.GROUP_ID_CONFIG, "reflectoring-user");

    return new DefaultKafkaConsumerFactory<>(
      props,
      new StringDeserializer(),
      new JsonDeserializer<>(User.class));
  }

  @Bean
  public ConcurrentKafkaListenerContainerFactory<String, User> userKafkaListenerContainerFactory() {
    ConcurrentKafkaListenerContainerFactory<String, User> factory =
      new ConcurrentKafkaListenerContainerFactory<>();
    factory.setConsumerFactory(userConsumerFactory());
    return factory;
  }
  ...
}
```

O Spring Kafka fornece implementações JsonSerializer e JsonDeserializer baseadas no mapeador de objetos Jackson JSON. Ele nos permite converter qualquer objeto Java em bytes[].

No exemplo acima, estamos criando mais um ConcurrentKafkaListenerContainerFactory para serialização JSON. Neste, configuramos JsonSerializer.class como nosso serializador de valor na configuração do produtor e JsonDeserializer<>(User.class) como nosso desserializador de valor na configuração do consumidor.

Para isso, estamos criando um contêiner de ouvinte Kafka separado userKafkaListenerContainerFactory(). Se tivermos vários tipos de objetos Java a serem serializados/desserializados, teremos que criar um contêiner de ouvinte para cada tipo, conforme mostrado acima.

### Enviando objetos Java
Agora que configuramos nosso serializador e desserializador, podemos enviar um objeto User usando o KafkaTemplate:

```
@Component
class KafkaSenderExample {
  ...

  @Autowired
  private KafkaTemplate<String, User> userKafkaTemplate;

  void sendCustomMessage(User user, String topicName) {
    userKafkaTemplate.send(topicName, user);
  }
  ...
}
```

### Recebendo objetos Java
Podemos ouvir objetos User usando a anotação @KafkaListener:

```
@Component
class KafkaListenersExample {

  @KafkaListener(
    topics = "reflectoring-user",
    groupId="reflectoring-user",
    containerFactory="userKafkaListenerContainerFactory")
  void listener(User user) {
    LOG.info("CustomUserListener [{}]", user);
  }
}
```

Como temos vários contêineres de ouvinte, estamos especificando qual fábrica de contêineres usar.

Se não especificarmos o atributo containerFactory, o padrão será kafkaListenerContainerFactory que usa StringSerializer e StringDeserializer em nosso caso.

# Conclusão
Neste artigo, abordamos como podemos aproveitar o suporte do Spring para Kafka. Crie mensagens baseadas em Kafka com exemplos de código que podem ajudar a começar rapidamente.