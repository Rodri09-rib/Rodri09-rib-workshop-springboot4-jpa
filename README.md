<div align="center">

# Projeto Web Services

**API REST de comércio eletrônico em Java, com Spring Boot 4, JPA e Hibernate**

[![Licença MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://github.com/Rodri09-rib/Rodri09-rib-workshop-springboot4-jpa/blob/main/LICENSE)
[![Java 25](https://img.shields.io/badge/Java-25-orange.svg)](https://openjdk.org/)
[![Spring Boot 4](https://img.shields.io/badge/Spring%20Boot-4.0-brightgreen.svg)](https://spring.io/projects/spring-boot)
[![Maven](https://img.shields.io/badge/Maven-C71A36.svg)](https://maven.apache.org/)
[![H2](https://img.shields.io/badge/H2-2.x-0D87DB.svg)](https://www.h2database.com/)

</div>

## Sobre

Este é um projeto de desenvolvimento de *web services* em Java utilizando o ecossistema
Spring Boot, JPA e Hibernate, com o objetivo de construir o *backend* para um sistema de
comércio eletrónico e gerenciamento de pedidos.

O foco não é o produto: é a **cadeia completa de um serviço REST** do mapeamento de
entidades e associações, à passagem pela camada de serviço, até à tradução de erros em
respostas HTTP. As rotas expõem clientes, produtos, categorias e pedidos; o trabalho real
está no modelo de domínio e no tratamento de exceções.

O projeto está organizado em **camadas** (`resource`, `service`, `repository`, `entities`):
o `resource` conhece o `service`, o `service` conhece o `repository` e as entidades.

## Funcionalidades

**Clientes** recurso com o CRUD completo.

- `GET /users` e `GET /users/{id}` para consulta.
- `POST /users` devolve `201 Created` e o URI do recurso criado, montado com
  `ServletUriComponentsBuilder`.
- `PUT /users/{id}` atualiza nome, e-mail e telefone a partir do corpo do pedido; o `id` do
  corpo é ignorado e vale o do *path*.
- `DELETE /users/{id}` devolve `204 No Content`.

**Produtos, categorias e pedidos** — leitura.

- `GET /products`, `GET /categories` e `GET /orders`, com detalhe por `/{id}`.
- O total de um pedido (`getTotal()`) é calculado a partir dos itens, e o subtotal de cada
  item a partir de `price * quantity`. Nenhum dos dois é guardado na base: são derivados.

**Tratamento de exceções** erros de negócio viram respostas HTTP com corpo padronizado.

- `ResourceNotFoundException` → `404 Not Found`, com a mensagem `Resource not found. iD {id}`.
- `DatabaseException` → `400 Bad Request`, o que acontece quando um cliente com pedidos é
  apagado e a base recusa por violação de integridade referencial.
- Todas as respostas de erro seguem o mesmo formato (`StandardError`): `timestamp`,
  `status`, `error`, `message` e `path`.

## Associações na JPA

O modelo é o exercício principal, por isso vale apontar **onde** cada associação aparece.

**`@OneToOne` com `@MapsId` entre `Order` e `Payment`** — o `Payment` não tem chave
própria: herda o `id` do pedido, e a coluna `order_id` é simultaneamente chave estrangeira
e chave primária. Uma única linha para as duas coisas, sem duplicar o identificador.

```java
// Payment.java — o id do pagamento É o id do pedido
@Id @GeneratedValue(strategy = GenerationType.IDENTITY)
private Long id;

@OneToOne @MapsId
private Order order;
```

**`@ManyToOne` de `Order` para `User`** o dono da associação é o pedido
(`@JoinColumn(name = "client_id")`), porque é ele que perde se o cliente for apagado. No
lado inverso, `User.orders` é `@JsonIgnore`: sem ele, serializar um cliente tentava percorrer
os pedidos, que voltavam ao cliente, e a resposta não terminava.

**`@OneToMany` de `Order` para `OrderItem`** o lado inverso é `mappedBy = "id.order"`,
porque o nome da propriedade que contém o pai é `id.order` (o `id` é um
`@EmbeddedId`, não um campo simples).

**`@ManyToMany` entre `Product` e `Category`** o produto é o dono, com a tabela de
ligação `tb_product_category`; a categoria só tem `mappedBy = "categories"`.

**`@Embeddable` como chave composta** — `OrderItem` não tem `@Id` próprio: tem um
`@EmbeddedId` (`OrderItemPK`) com as duas referências, o que garante que o mesmo produto não
aparece duas vezes no mesmo pedido. O `equals` e o `hashCode` do `OrderItemPK` comparam
`order` **e** `product`, e o do `OrderItem` compara o `id` composto.

**O estado do pedido é gravado como código, não como texto** `OrderStatus` é um `enum`
com `WAITING_PAYMENT(1)`, `PAID(2)`, `SHIPPED(3)`, `DELIVERED(4)` e `CANCELED(5)`, e a
entidade guarda o `int` do código. Um `valueOf(int)` próprio traduz o código de volta, e
lança `IllegalArgumentException` se não corresponder a nenhum estado — o `valueOf` de
`Enum` só funcionaria por nome.

## Decisões de projeto

**As entidades são serializadas diretamente.** Não há DTOs: o que sai da API é a entidade.
É aceitável num projeto didático e simplifica a leitura das respostas, mas obriga a
cuidado no *what* é exposto daí o `@JsonIgnore` nas coleções de retorno e no `password`
de `User`.

**`@Profile("test")` isola o povoamento.** O `TestConfig` é um `CommandLineRunner` que só
corre no perfil `test`, e `application.properties` ativa esse perfil por omissão. Arrancar a
aplicação já devolve dados: 3 categorias, 5 produtos, 2 clientes, 3 pedidos, 4 itens e 1
pagamento. O mesmo `CommandLineRunner` noutro perfil deixaria a base vazia, e foi esse
registo explícito que evitou um `@Component` a correr sempre.

**O `CommandLineRunner` salva duas vezes.** Os produtos são guardados antes das categorias
para que tenham `id` (gerado pela base), e outra vez depois de `add(cat)` a primeira
gravação cria a linha, a segunda persiste a coleção de categorias. É a sequência que
resolve a dependência entre as duas associações sem cascade.

**O `delete` apanha a violação de integridade e traduz-a.** `UserService.delete` envolve o
`deleteById` em `try/catch`: `EmptyResultDataAccessException` passa a
`ResourceNotFoundException` (o `deleteById` do Spring Data é silencioso quando o `id` não
existe) e `DataIntegrityViolationException` passa a `DatabaseException`. Sem o `catch`, um
cliente com pedidos devolveria um `500` com *stack trace* em vez de um `400` a dizer o que
se passou.

**`findById` usa `Optional.orElseThrow`.** O serviço de clientes devolve
`repository.findById(id).orElseThrow(() -> new ResourceNotFoundException(id))` em vez de
`get()`: o `get()` do `Optional` lançaria `NoSuchElementException`, que ninguém traduz, e o
cliente receberia um `500` em vez de um `404`.

**O `delete` responde `204` mesmo sem corpo.** Não há `ResponseEntity` com `User`, só
`noContent().build()`: o recurso já não existe, portanto devolvê-lo seria redundante.

**`User.getId()` devolve `long`.** Desempacota o `Long`, e um cliente novo sem `id` dá
`NullPointerException` ao ser serializado.

**`open-in-view` fica ligado.** `spring.jpa.open-in-view=true` mantém a sessão do Hibernate
aberta durante a serialização, o que evita `LazyInitializationException` ao percorrer
coleções preguiçosas. Em produção desligava-se: o custo passa a ser uma `N+1` por pedido
em vez de uma falha.

**As entidades implementam `Serializable`** e definem `hashCode`/`equals` pelo `id`, para
que um objeto da lista seja o mesmo objeto do `Set` quando é seria com a mesma chave. Sem
isso, um `Set` não deduplicaria o mesmo cliente duas vezes.

## Stack

| Camada | Tecnologias |
| --- | --- |
| Linguagem | Java 25 |
| Framework | Spring Boot 4.0.0 |
| Web | Spring Web MVC (`spring-boot-starter-webmvc`) |
| Persistência | Spring Data JPA + Hibernate, com `@Embeddable` para a chave composta |
| Base de dados | H2 em memória (`jdbc:h2:mem:testdb`) com `CommandLineRunner` de povoamento |
| Exceções | `@ControllerAdvice` com `StandardError` como corpo único de erro |
| Testes | Spring Boot Test (JUnit 5), arranque do contexto |
| Build | Maven, com *wrapper* (`mvnw`) incluído |

O `postgresql` também está declarado em âmbito `runtime`, mas não há perfil que o use: o
`application.properties` aponta para H2.


## API

| Método | Rota | Descrição |
| --- | --- | --- |
| `GET` | `/users` | Lista todos os clientes |
| `GET` | `/users/{id}` | Consulta um cliente |
| `POST` | `/users` | Cria um cliente (`201` com o `Location` do novo recurso) |
| `PUT` | `/users/{id}` | Atualiza um cliente |
| `DELETE` | `/users/{id}` | Apaga um cliente (`204`) |
| `GET` | `/products` | Lista todos os produtos |
| `GET` | `/products/{id}` | Consulta um produto |
| `GET` | `/categories` | Lista todas as categorias |
| `GET` | `/categories/{id}` | Consulta uma categoria |
| `GET` | `/orders` | Lista todos os pedidos |
| `GET` | `/orders/{id}` | Consulta um pedido, com itens e total |

Não há autenticação em nenhuma rota: qualquer pedido HTTP chega ao serviço.

## Estrutura

```
src/
├── main/java/com/educandoweb/course/
│   ├── CourseApplication.java      classe principal (@SpringBootApplication)
│   ├── config/
│   │   └── TestConfig.java         CommandLineRunner que popula a base (@Profile("test"))
│   ├── entities/
│   │   ├── User.java               tb_user     — 1:N para Order
│   │   ├── Product.java            tb_product  — N:M com Category, 1:N com OrderItem
│   │   ├── Category.java           tb_category — lado inverso do N:M
│   │   ├── Order.java              tb_order    — 1:1 com Payment, 1:N com OrderItem
│   │   ├── OrderItem.java          tb_order_item — chave composta
│   │   ├── Payment.java            tb_payment  — @MapsId, herda o id do pedido
│   │   ├── enums/OrderStatus.java  estados do pedido gravados por código
│   │   └── pk/OrderItemPK.java     @Embeddable (order, product)
│   ├── repositories/               JpaRepository de cada entidade, sem métodos próprios
│   ├── resources/                  @RestController de cada recurso
│   │   └── exceptions/
│   │       ├── ResourceExceptionHandler.java  @ControllerAdvice
│   │       └── StandardError.java             corpo único de erro
│   └── services/                   regras de negócio e tradução de exceções
│       └── exceptions/             ResourceNotFoundException, DatabaseException
├── main/resources/
│   ├── application.properties     perfil ativo e open-in-view
│   └── application-test.properties  H2, consola e SHOW SQL
└── test/java/                      Spring Boot Test
```

O sentido das dependências é de cima para baixo: `resources` conhece `services`, que
conhece `repositories` e `entities`. Uma entidade não conhece o serviço que a usa.

## Mapa de responsabilidades

| Camada | Responde por | Não faz |
| --- | --- | --- |
| `resource` | Rotas, códigos HTTP, serialização | Regras de negócio |
| `service` | Validação, tradução de exceções, orquestração | Conhecer `HttpServletRequest` |
| `repository` | Persistência, via `JpaRepository` | Decidir o que é erro de negócio |
| `entities` | Mapeamento relacional e invariantes do modelo | Dependências de Spring Web |

## Modelo conceitual

<img width="783" height="325" alt="Modelo conceitual do projeto" src="https://github.com/user-attachments/assets/d6ea9bb0-3c56-4653-a4a2-7aaeb2ab041b" />

## Limitações conhecidas

São limitações de um projeto em construção, e não esquecimentos:

- **Só os clientes têm escrita.** Produtos, categorias e pedidos aceitam apenas `GET`. O
  CRUD completo está feito no `UserService`; os restantes só chegam a `findAll` e
  `findById`.
- **Os outros `findById` usam `Optional.get()`.** Um `id` inexistente em `/products/{id}`
  lança `NoSuchElementException`, que o `@ControllerAdvice` não traduz, e o cliente recebe
  um `500` em vez de um `404`.
- **Não há `@ExceptionHandler` genérico.** Uma exceção não mapeada — violação de restrições
  que não seja o `delete`, por exemplo — escapa como `500` e devolve a mensagem interna do
  Hibernate.
- **A palavra-passe é devolvida na resposta.** `User.password` não tem `@JsonIgnore`, e
  `UserService.update` também não a atualiza. Num projeto a sério, `password` levaria
  `@JsonIgnore`, `BCryptPasswordEncoder` e um DTO de entrada.
- **Preços e totais são `double`.** `price * quantity` acumula erro de arredondamento em
  operações repetidas; `BigDecimal` é o tipo correto para valores monetários.
- **`User.getId()` devolve `long`.** Desempacota o `Long`, e um cliente novo sem `id` dá
  `NullPointerException` ao ser serializado.
- **`OrderItem.getProuct()` tem o nome trocado.** Como não há `@JsonIgnore`, o campo sai no
  JSON como `prouct`, e não `product`.
- **Os dados não sobrevivem ao arranque.** O H2 é `mem`, pelo que cada reinício volta aos
  dados do `TestConfig`.
- **Não há autenticação nem autorização.** Todas as rotas estão abertas, incluindo a
  criação e o apagamento de clientes.
- **Não há validação de entrada.** Não há Bean Validation nos `@RequestBody`, e `email` e
  `phone` são guardados como vierem.

## Autor

**Rodrigo Ribeiro Ferreira** — [LinkedIn](https://www.linkedin.com/in/rodrigo-ribeiro-abbb713aa?utm_source=share_via&utm_content=profile&utm_medium=member_ios)

## Licença

[MIT](LICENSE)
