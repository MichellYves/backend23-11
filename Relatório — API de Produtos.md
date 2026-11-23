 Relatório — API de Produtos
 1. Integrantes

- Michel Yves Pereira Costa

 2. Escopo e limitação do material recebido

O link fornecido aponta para uma conversa compartilhada cujo resumo público mostra uma API chamada `api-produtos-concluido`, alterações em testes, `Dockerfile`, `maven-settings.xml` e scripts de execução. O enunciado integral e o código original não ficaram acessíveis porque a opção “Continue in ChatGPT” exige autenticação. Portanto, este relatório documenta uma implementação executável de referência, alinhada aos entregáveis explicitamente solicitados: API REST, testes, JAR, Docker e Actuator. As perguntas abaixo são respondidas por tema, sem atribuí-las ao enunciado original que não estava disponível.

 3. Implementação proposta

A solução usa Spring Boot 3.3.5, Java 21, Spring Web, Bean Validation, Spring Data JPA e H2 em memória. O domínio `Produto` possui `id`, `nome`, `preco` e `descricao`. O controlador disponibiliza CRUD em `/produtos`; o serviço concentra as regras de busca, criação, atualização e exclusão; e o repositório usa `JpaRepository`.

O caso de estudo é resolvido separando responsabilidades: entrada HTTP e validação ficam no controller, regras ficam no service, persistência fica no repository e observabilidade fica no Actuator. Produtos inexistentes retornam HTTP 404 e dados inválidos retornam HTTP 400. A resposta de criação é HTTP 201 com `Location` apontando para o recurso criado.

 4. Parte I — testes

 Pergunta 1 — Por que testar em camadas?

Testes unitários isolam uma classe e fornecem feedback rápido. Nesta entrega, `ProdutoServiceTest` usa Mockito para validar as regras do serviço sem subir o Spring. Testes de slice, representados por `ProdutoControllerTest`, carregam apenas o contexto MVC e verificam status HTTP, serialização e validação. O teste de integração, `ApiProdutosIntegrationTest`, sobe a aplicação em porta aleatória, usa H2 e verifica o fluxo completo via HTTP.

Essa combinação reduz falsos positivos: a unidade verifica lógica, o slice verifica o contrato do controller e a integração confirma que wiring, JPA, banco, servidor e Actuator funcionam juntos. O starter oficial `spring-boot-starter-test` reúne os módulos e bibliotecas de teste mais usados pelo Spring Boot [1].

 Pergunta 2 — O que caracteriza um teste adequado?

Um teste adequado é determinístico, isolado, legível e verifica comportamento observável. Os testes desta entrega têm nomes descritivos, dados pequenos e asserts diretamente relacionados ao contrato. O teste de integração limpa o repositório antes de cada execução para não depender da ordem dos testes.

 Exemplo comentado de teste

```java
@Test
void deveRejeitarProdutoSemNome() throws Exception {
    // Arrange: monta uma entrada inválida; nome vazio viola @NotBlank.
    Produto p = new Produto("", new BigDecimal("10.00"), "desc");

    // Act + Assert: envia a requisição e verifica o contrato HTTP.
    mvc.perform(post("/produtos")
            .contentType(MediaType.APPLICATION_JSON)
            .content(mapper.writeValueAsString(p)))
        .andExpect(status().isBadRequest());
}
```

O teste não chama o banco nem a implementação real do serviço. Assim, ele prova que a camada HTTP rejeita a entrada antes de delegar a operação.

5. Parte II — Docker e Actuator

 Pergunta 3 — Por que usar build em múltiplos estágios?

O primeiro estágio contém Maven e o compilador; o estágio final contém somente o JRE e o JAR. Isso reduz o conteúdo da imagem final e separa ferramentas de build do runtime. A documentação do Docker recomenda multi-stage builds para separar compilação e saída executável [2].

 Pergunta 4 — Por que executar como usuário não-root?

A aplicação não precisa de privilégios administrativos. O `Dockerfile` cria o usuário com UID explícito e aplica `USER 1001`, reduzindo o impacto de uma eventual exploração da aplicação. A recomendação oficial do Docker é usar `USER` quando o serviço puder rodar sem privilégios [2].

 Exemplo comentado de Dockerfile

```
 Estágio 1: compila e empacota, sem levar Maven para produção.
FROM maven:3.9.9-eclipse-temurin-21 AS build
WORKDIR /workspace
COPY pom.xml .
RUN mvn -q -DskipTests dependency:go-offline
COPY src src
RUN mvn -q -DskipTests package

 Estágio 2: runtime mínimo.
FROM eclipse-temurin:21-jre
WORKDIR /app
RUN useradd --system --uid 1001 spring
COPY --from=build /workspace/target/api-produtos-1.0.0.jar app.jar
USER 1001
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "/app/app.jar"]
```

O `COPY --from=build` transfere apenas o artefato gerado. O `.dockerignore` evita enviar `target`, `.git`, IDEs e logs para o contexto do build.

 Pergunta 5 — Qual é a finalidade do Actuator?

O Spring Boot Actuator disponibiliza recursos de monitoramento e gerenciamento, incluindo health e métricas, por HTTP ou JMX [3]. Nesta API, são expostos explicitamente `/actuator/health`, `/actuator/info` e `/actuator/metrics`. O healthcheck do Compose consulta `/actuator/health`, permitindo que o runtime saiba se o processo e o banco H2 estão disponíveis.

6. Evidências da execução

 Testes e JAR

O comando executado foi:

```bash
mvn -B clean test package
```

Resultado registrado em `evidencias/maven-test-package.log`: `BUILD SUCCESS`. Foram gerados relatórios XML para `ApiProdutosIntegrationTest`, `ProdutoControllerTest` e `ProdutoServiceTest`. O artefato foi gerado em `target/api-produtos-1.0.0.jar`, aproximadamente 50 MB, como JAR executável repackaged pelo Spring Boot.

 Aplicação empacotada e Actuator

O JAR foi iniciado com `java -jar target/api-produtos-1.0.0.jar`. A evidência em `evidencias/actuator-api.txt` registra:

```
GET /actuator/health -> {"status":"UP", ... "db":{"status":"UP", ... "database":"H2"}}
GET /produtos -> []
POST /produtos -> status 201
```

Isso comprova que o processo iniciou, o banco H2 foi conectado, o endpoint de saúde respondeu e o CRUD aceitou uma criação.

 Containers

O ambiente desta execução não possuía o daemon Docker (`docker` não estava instalado/disponível); por isso, não é correto afirmar que a imagem ou o container foram executados aqui. O projeto, porém, contém `Dockerfile` e `docker-compose.yml` prontos para a evidência final no ambiente com Docker:

```bash
docker build -t api-produtos:1.0.0 .
docker run --rm -p 8080:8080 api-produtos:1.0.0
docker ps
docker inspect --format='{{.State.Health.Status}}' <container>
curl http://localhost:8080/actuator/health
docker compose up --build
```

A comprovação esperada é a imagem `api-produtos:1.0.0`, o container em estado `running`/`healthy` e a resposta `{"status":"UP"}` do Actuator.

 7. Status da solução

A implementação local está OK para o escopo de referência: testes unitários, MVC e integração passaram; o JAR foi criado; a aplicação empacotada respondeu à API e ao Actuator. A validação de container permanece pendente somente por ausência do daemon Docker neste ambiente, não por erro observado no `Dockerfile`.

 9. Estrutura de entrega

- `pom.xml`: dependências e build.

- `src/main`: aplicação, controller, service, model e repository.

- `src/test`: testes unitários, MVC e integração.

- `Dockerfile` e `.dockerignore`: imagem em múltiplos estágios.

- `docker-compose.yml`: execução com healthcheck.

- `README.md`: instruções de execução.

- `evidencias/maven-test-package.log`: build, testes e pacote.

- `evidencias/actuator-api.txt`: evidência do JAR, endpoints e Actuator.

 Referências

[1]: https://docs.spring.io/spring-boot/reference/testing/index.html "Spring Boot Testing"

[2]: https://docs.docker.com/build/building/best-practices/ "Docker Build best practices"

[3]: https://docs.spring.io/spring-boot/reference/actuator/index.html "Spring Boot Production-ready Features"

