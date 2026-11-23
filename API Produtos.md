 API Produtos

Projeto de referência para uma API REST de produtos em Spring Boot 3.3.5, Java 21, JPA/H2, testes automatizados, Docker e Spring Boot Actuator.

 Integrantes

- Michel Yves Pereira Costa

 Executar localmente

```bash
mvn clean test
mvn spring-boot:run
curl http://localhost:8080/produtos
curl http://localhost:8080/actuator/health
```

 Gerar JAR

```bash
mvn clean package
java -jar target/api-produtos-1.0.0.jar
```

 Docker

```bash
docker build -t api-produtos:1.0.0 .
docker run --rm -p 8080:8080 api-produtos:1.0.0
# ou
docker compose up --build
```

 Endpoints

`GET /produtos`, `GET /produtos/{id}`, `POST /produtos`, `PUT /produtos/{id}` e `DELETE /produtos/{id}`. O Actuator está exposto em `/actuator/health`, `/actuator/info` e `/actuator/metrics`.

