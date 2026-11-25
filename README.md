# Back-end do e-commerce da oficina "Auto Center Silva"

API REST em **Java 17 + Spring Boot 3.2 + Spring Security (JWT) + PostgreSQL + Flyway**.
O código fica em `demo/`.

## Variáveis de ambiente

Nenhum segredo fica no código. A aplicação **não sobe** sem `DB_PASSWORD` e `JWT_SECRET`.
Modelo em `demo/.env.example`.

| Variável | Obrigatória | Descrição |
|---|---|---|
| `DB_PASSWORD` | sim | Senha do PostgreSQL |
| `JWT_SECRET` | sim | Segredo do JWT, mínimo 32 bytes (`openssl rand -base64 48`) |
| `DB_URL` | não | Padrão `jdbc:postgresql://localhost:5432/bdAutoCenterSilva` |
| `DB_USER` | não | Padrão `postgres` |
| `JWT_EXPIRATION_MS` | não | Validade do token em ms (padrão 86400000 = 24 h) |
| `CORS_ALLOWED_ORIGINS` | não | Origens separadas por vírgula (padrão `http://localhost:4200`). Em produção o Nginx serve front e `/api` na mesma origem |
| `WHATSAPP_NUMERO` | não | Número usado no link de confirmação do pedido |
| `SWAGGER_ENABLED` | não | `true` habilita Swagger (somente desenvolvimento) |

## Executar em desenvolvimento

```bash
cd demo
export DB_PASSWORD=... JWT_SECRET=$(openssl rand -base64 48)
./mvnw spring-boot:run          # http://localhost:8080
```

No IntelliJ, defina as variáveis na *Run Configuration*. O Flyway cria/atualiza o schema na inicialização.

## Build para deploy

```bash
cd demo
./mvnw -B -Dmaven.test.skip=true package
java -jar target/demo-0.0.1-SNAPSHOT.jar
```

> **Atenção:** a suíte em `src/test` ainda não foi atualizada para a autenticação JWT, a posse de
> pedidos e a reserva de estoque. Ela não compila contra o código atual, por isso o build usa
> `-Dmaven.test.skip=true` (pula compilação **e** execução dos testes).

Os logs saem no console. No servidor (systemd): `journalctl -u <servico> -f`.

## Permissões

| Rota | Acesso |
|---|---|
| `POST /api/clientes/criar`, `POST /api/clientes/login` | público |
| `GET /api/produtos/listar`, `POST /api/produtos/pesquisar`, `GET /cotacao/produto/{id}/converter-para-reais` | público |
| `POST /api/pedidos/criar`, `GET /api/pedidos/meus`, `PUT /api/pedidos/{id}/cancelar` | autenticado (somente os próprios pedidos; equipe cancela qualquer um) |
| `PATCH /api/clientes/atualizar/{id}`, `GET /api/clientes/busca/{id}`, `DELETE /api/clientes/desativar/{id}` | autenticado (somente a própria conta; ADM qualquer uma) |
| `GET /api/pedidos/admin`, `GET /api/pedidos/listar`, `PUT /api/pedidos/{id}/confirmar-pagamento` | ATENDENTE ou ADM |
| `POST/DELETE /api/carro/**`, `/api/produtos/criar`, `/variacao/**`, `/variacoes/**`, `/admin/estoque/**`, `/api/clientes/usuarios/**` | ADM |

Erros seguem o formato `{ "status", "error", "message" }`: 400 validação/regra de negócio, 401 não
autenticado ou credenciais inválidas, 403 sem permissão, 404 não encontrado, 410 reserva expirada,
503 API de cotação indisponível.
