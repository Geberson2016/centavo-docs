# Plano de evolução do Centavo — quebrado em commits

Plano incremental para sair do estado atual e chegar na arquitetura-alvo (**React → BFF → Spring → Postgres**, contract-first, i18n, SOLID). Cada item abaixo é **1 commit atômico**, em ordem de dependência. Conforme você finaliza um item, use a **mensagem de commit sugerida** (prefixo em inglês por Conventional Commits, texto em português — conforme `AGENTS.md §0`).

## Como usar
- Faça um item de cada vez, na ordem. Cada item é pensado para **manter o build verde** ao final do commit.
- `[backend]` = repo `centavo-backend`; `[frontend]` = repo `centavo-frontend`; `[bff]` = novo repo/pacote do BFF.
- ⚠️ **Par coordenado** = dois commits (repos diferentes) que devem ser **deployados juntos** porque mudam o contrato ao mesmo tempo.
- Marque `[x]` conforme concluir.

---

## 📌 Status atual (atualizado em 2026-11-21)

- **Plano concluído:** C1–C21 (Fases 0 a 8). Fase 8: testes do backend `aa5e929` (com o fix do filtro JWT `f71c37d`), testes do frontend `998c75a`, CI nos três repositórios e o stack em Docker no novo repositório `centavo-infra`. Antes, BFF: scaffold `e0481ef`, encaminhamento `fa9c8d7` e sessão `d865ee2`; frontend no BFF `fc0be69`.
- **Repositórios:** `centavo-backend`, `centavo-bff`, `centavo-frontend`, `centavo-infra` (compose do stack) e `centavo-docs` (este), todos lado a lado em `~/Projects/Centavo`.
- **Rodar local:** em `centavo-infra`, `cp .env.example .env` (preencher) e `docker compose up -d --build --wait`. UI em `http://localhost:4173` (nginx com proxy de `/bff`), BFF `:3000`, API `:8090`, Postgres no host em `POSTGRES_PORT` (padrão `5433`). Para hot reload: `docker compose up -d postgres backend` e `npm run dev` no BFF e no front (`:5173`).
- **Fluxo de trabalho:** desde 2026-11-21 nada entra direto na `main`: issue → branch → PR, com merge feito pelo Geberson. Ver [`processo/fluxo-de-trabalho.md`](../processo/fluxo-de-trabalho.md).
- **CI:** GitHub Actions em cada repositório, no push para `main` e em PR. Front e BFF têm também um job `contract`, que baixa o repositório vizinho e falha se os tipos gerados ou a cópia do contrato ficaram para trás.
- **Pendências conhecidas (fora do plano):**
  - O Spring devolve 403 sem corpo para token ausente ou inválido. O BFF já traduz para `401 UNAUTHENTICATED`; mudar no Spring exige mudar o BFF junto.
  - Logout sem estado: apagar o cookie não revoga uma cópia dele até o JWT expirar (1 dia).
  - Ideias do roadmap ainda não feitas: smoke E2E com Playwright, Dependabot, actuator e logs estruturados, Spotless/Checkstyle no backend.

---

## Fase 0 — Higiene de configuração e limpezas (backend, baixo risco, sem dependências)

### [x] C1 · [backend] Externalizar segredos
Tirar do `application.properties` o segredo do JWT (`api.security.token.secret=chave-super-secreta-12345`) e as credenciais do datasource (`admin/admin`); ler de variáveis de ambiente (`${JWT_SECRET}`, `${DB_USER}`, `${DB_PASSWORD}`). Adicionar `application-example.properties` versionado com placeholders e garantir `.env`/valores reais no `.gitignore`.
- **Commit:** `chore(config): externaliza segredo do JWT e credenciais do banco via env`

### [x] C2 · [backend] Remover starter de persistência duplicado
Remover do `pom.xml` o `spring-boot-starter-data-jdbc` (mantendo apenas `data-jpa`, que é o em uso).
- **Commit:** `chore(deps): remove spring-boot-starter-data-jdbc não utilizado`

### [x] C3 · [backend] Centralizar CORS
Remover os `@CrossOrigin` por controller (ex.: `TransactionController`) e deixar o CORS só no `SecurityConfig`, com as origens vindo de configuração (`${CORS_ALLOWED_ORIGINS}`) em vez de literal.
- **Commit:** `refactor(security): centraliza CORS no SecurityConfig e remove @CrossOrigin`

---

## Fase 1 — Banco versionado

### [x] C4 · [backend] Adotar Flyway
Adicionar a dependência do Flyway, criar a migração baseline (`V1__baseline.sql`) refletindo as entidades **atuais** (users com email/phone/password, accounts, categories, transactions), trocar `spring.jpa.hibernate.ddl-auto=update` por `validate` e aposentar o `schema.sql` divergente.
- **Commit:** `chore(db): adiciona Flyway com baseline do schema e ddl-auto=validate`
- **Depende de:** —

---

## Fase 2 — Modelo de erro

### [x] C5 · [backend] Exceptions tipadas + handler global
Criar exceptions de domínio com **código estável** (ex.: `NotFoundException(code)`, `ConflictException(code)`), um `@RestControllerAdvice` que devolve **RFC 7807 `application/problem+json`** (`{ code, status, detail? }`) e substituir todos os `throw new RuntimeException("...")` dos services pelos tipos novos, usando os códigos da tabela de `AGENTS.md §4` (`ACCOUNT_NOT_FOUND`, `CATEGORY_NOT_FOUND`, `TRANSACTION_NOT_FOUND`, `EMAIL_ALREADY_EXISTS`, `TRANSACTION_CATEGORY_TYPE_MISMATCH`).
- **Commit:** `feat(error): adiciona exceptions de domínio tipadas e handler global (RFC 7807)`
- **Depende de:** —
- **Nota:** o front ainda mostra `alert()` genérico até a Fase 6; o mapeamento código→mensagem fecha lá.

---

## Fase 3 — Validação de entrada

### [x] C6 · [backend] Bean validation nos DTOs de request
Anotar os `record`s `*Request` com `jakarta.validation` (`@NotBlank`, `@Email`, `@Positive`, `@NotNull`, etc.) e colocar `@Valid` nos parâmetros dos controllers. O handler da Fase 2 traduz `MethodArgumentNotValidException` para o formato RFC 7807 com código `VALIDATION_ERROR`.
- **Commit:** `feat(validation): valida DTOs de request com jakarta validation`
- **Depende de:** C5

---

## Fase 4 — Migração PT→EN do domínio ⚠️ par coordenado (subir juntos)

### [x] C7 · [backend] Migrar enums para inglês
Renomear os valores dos enums conforme `AGENTS.md §4` (`RECEITA→INCOME`, `DESPESA→EXPENSE`, `CORRENTE→CHECKING`, `POUPANCA→SAVINGS`, `INVESTIMENTO→INVESTMENT`, `DINHEIRO→CASH`, `CARTAO_CREDITO→CREDIT_CARD`, `FIXO→FIXED`, `VARIAVEL→VARIABLE`) e criar migração Flyway `V2__rename_domain_enums_to_english.sql` com `UPDATE` sobre `transactions.type`, `accounts.type`, `categories.type`, `categories.budget_type`.
- **Commit:** `refactor(domain): migra enums de domínio para inglês (INCOME/EXPENSE, CHECKING…)`
- **Depende de:** C4
- ⚠️ **Atenção (constatado no C4):** o banco atual foi criado pelo `ddl-auto=update`, então o Hibernate criou **check constraints** nos enums (ex.: `type IN ('RECEITA','DESPESA')`). A migração Flyway do C7 precisa **dropar essas check constraints antes do `UPDATE`** dos valores e **recriá-las** com os valores em inglês (ou removê-las de vez), senão o `UPDATE` falha. Isso vale para `accounts.type`, `categories.type`, `categories.budget_type` e `transactions.type`.

### [x] C8 · [frontend] Alinhar tipos ao contrato em inglês
Trocar os union types `'RECEITA' | 'DESPESA'` por `'INCOME' | 'EXPENSE'` (ex.: `hooks/useCreateTransaction.ts`) e qualquer valor de conta correspondente. Ajustar labels exibidos (que viram chaves de i18n na Fase 6).
- **Commit:** `refactor(domain): alinha os tipos de transação e conta ao contrato em inglês`
- **Depende de:** C7 (deployar junto — a API passa a devolver os valores novos)

---

## Fase 5 — Contract-first OpenAPI ⚠️ par coordenado

### [x] C9 · [backend] Contrato OpenAPI como fonte da verdade
Criar `openapi.yaml` versionado descrevendo todos os endpoints atuais (`/api/v1/**`), com paths/schemas/`operationId` em inglês e descrições/summaries em português (`AGENTS.md §0`). Configurar o openapi-generator (`spring`, `interfaceOnly=true`) para gerar as interfaces e fazer os controllers implementá-las. (springdoc opcional para servir o Swagger UI a partir do mesmo contrato.)
- **Feito assim:** os schemas foram mapeados (`schemaMappings`) para os records/enums existentes, sem gerar models; o `Problem` foi mapeado para o `ProblemDetail` do Spring. O springdoc ficou de fora.
- **Commit:** `feat(api): adiciona contrato OpenAPI e gera interfaces do servidor`
- **Depende de:** C7 (enums já em inglês no spec)

### [x] C10 · [frontend] Gerar client TypeScript do spec
Gerar tipos/client a partir do `openapi.yaml` (`openapi-typescript` ou `openapi-generator typescript-fetch`), adicionar o script de geração ao `package.json` e substituir as interfaces escritas à mão nos hooks pelos tipos gerados.
- **Commit:** `feat(api): gera client TypeScript a partir do openapi.yaml`
- **Depende de:** C9
- **Feito assim:** `openapi-typescript` → `src/api/schema.ts`, versionado, com aliases em `src/api/types.ts`; scripts `api:generate` e `api:check`. O axios ficou, porque os interceptors de auth saem só no C17. Usa `overrides` porque o gerador declara peer TS 5 e o projeto está no TS 6.

---

## Fase 6 — i18n e UX de erro (frontend)

### [x] C11 · [frontend] Configurar react-i18next
Instalar e configurar `react-i18next` com locale padrão `pt-BR` e `en`, provider na raiz e estrutura de bundles (`src/i18n/pt-BR.json`, `src/i18n/en.json`).
- **Commit:** `feat(i18n): configura react-i18next com bundles pt-BR e en`
- **Depende de:** —

### [x] C12 · [frontend] Extrair strings hardcoded
Mover todos os textos de UI hoje cravados no JSX (Login, Dashboard, Sidebar, etc.) para chaves de i18n nos dois bundles.
- **Commit:** `refactor(i18n): move as strings da UI para os bundles de tradução`
- **Depende de:** C11

### [x] C13 · [frontend] Toasts + mapa de códigos de erro
Substituir os `alert()` por um sistema de toast/erro inline, mapeando os **códigos** de erro do backend (RFC 7807 da Fase 2) para mensagens localizadas, com fallback genérico.
- **Commit:** `feat(ui): substitui alert() por toasts e mapeia códigos de erro do backend`
- **Depende de:** C5, C11

---

## Fase 7 — BFF (Node/TS)

### [x] C14 · [bff] Scaffold do serviço
Inicializar o serviço BFF (Fastify/NestJS/Hono, TS), com estrutura base, `healthcheck`, config por env e `.env.example`.
- **Commit:** `chore(bff): inicializa o serviço BFF com healthcheck e configuração por env`

### [x] C15 · [bff] Encaminhar/agregar o contrato do Spring
Consumir o `openapi.yaml` (client gerado) e expor os endpoints/view-models que a UI precisa, sem regra de negócio (só agregação/adaptação).
- **Commit:** `feat(bff): encaminha e agrega o contrato do Spring para a UI`
- **Depende de:** C9
- **Feito assim:** cópia do spec do Spring em `contracts/` + openapi-typescript/openapi-fetch; `openapi.yaml` próprio com só o que a UI usa; `GET /bff/dashboard` agrega 3 chamadas; erros do Spring passam intactos e Spring fora do ar vira `502 UPSTREAM_UNAVAILABLE` (código novo, que o front precisa traduzir no C17). O `Authorization` ainda é repassado até o C16.

### [x] C16 · [bff] Sessão em cookie httpOnly
Implementar login no BFF que troca credenciais pelo JWT do Spring e devolve uma **sessão em cookie httpOnly, Secure**; preparar fluxo de refresh.
- **Commit:** `feat(bff): adiciona login e sessão em cookie httpOnly`
- **Depende de:** C15
- **Feito assim:** `@fastify/secure-session`, ou seja, cookie `centavo_session` cifrado com o JWT do Spring dentro, sem estado no servidor. `POST/GET/DELETE /bff/session`. A sessão expira junto com o JWT; 403 do Spring vira 401. CSRF: SameSite=Lax + só `application/json`. Refresh não existe, porque o Spring não tem endpoint para isso.

### [x] C17 · [frontend] Consumir o BFF e sair do localStorage
Apontar o client do front para o BFF (não mais o Spring direto), remover o token do `localStorage` e ajustar `utils/auth.ts` / interceptors para a sessão por cookie.
- **Commit:** `refactor(auth): consome o BFF e move a sessão para cookie httpOnly`
- **Depende de:** C16 (par coordenado com o BFF)
- **Feito assim:** proxy do Vite para `/bff`, tipos gerados do `openapi.yaml` do BFF, `useSession`/`useLogin`/`useLogout`, `useDashboard` agregado. Validado num navegador headless: login, dashboard, reload, logout; o token não fica acessível ao JS.

---

## Fase 8 — Testes, CI e infra

### [x] C18 · [backend] Testes
JUnit 5 + Mockito para services e `@WebMvcTest` para controllers; Testcontainers (Postgres) para repositórios/integração.
- **Commit:** `test(backend): adiciona testes de service, controller e integração`
- **Feito assim:** 68 testes. Services com Mockito; controllers com `@WebMvcTest` e o `SecurityConfig` real; repositories com `@DataJpaTest` contra Postgres do Testcontainers (schema das migrações validado pelo Hibernate); `ApiIntegrationTest` ponta a ponta, incluindo o isolamento entre usuários. Os testes revelaram que token inválido fazia a exceção escapar do filtro JWT (`f71c37d`).

### [x] C19 · [frontend] Testes
Vitest + React Testing Library para hooks e componentes; smoke E2E (Playwright) opcional (login → dashboard).
- **Commit:** `test(frontend): adiciona Vitest e testes de componente`
- **Feito assim:** 41 testes com Vitest + React Testing Library e o BFF simulado pelo MSW (chamada sem handler falha o teste). Cobre o interceptor do axios, `useSession`, `PrivateRoute`, Login, Dashboard, a tabela de lançamentos e a paridade dos bundles pt-BR/en. O E2E com Playwright ficou de fora (era opcional).

### [x] C20 · [ci] Pipeline
GitHub Actions rodando lint + typecheck + build + test para backend, frontend e BFF nos PRs.
- **Commit:** `ci: adiciona pipeline de lint, build e testes`
- **Feito assim:** um workflow `ci.yml` por repositório. Backend: `./mvnw verify` (com Testcontainers). BFF: typecheck, lint, test, build. Front: lint, test, build. Front e BFF ganharam o job `contract` (`npm run api:check` com o repositório vizinho ao lado).

### [x] C21 · [infra] Docker Compose do stack
`docker-compose` com postgres + backend + bff + web para dev local com um comando; `Dockerfile` por serviço.
- **Commit:** `chore(infra): adiciona docker-compose com postgres, backend, bff e web`
- **Feito assim:** `Dockerfile` multi-stage em cada repositório (JRE só com o jar; Node só com o `dist/` e as dependências de produção; nginx servindo o build do Vite com proxy de `/bff`). O compose fica no repositório novo `centavo-infra`, com healthchecks em cadeia e portas presas a `127.0.0.1`. Para a imagem funcionar a partir de um clone limpo, o `application.properties` do backend passou a ser versionado (só tem `${VAR}`).

---

## Resumo da ordem
```
C1 → C2 → C3        (backend: config/limpeza)
C4                  (backend: Flyway)
C5 → C6             (backend: erro + validação)
C7 ⇄ C8             (PT→EN, par coordenado)
C9 ⇄ C10            (OpenAPI, par coordenado)
C11 → C12 → C13     (frontend: i18n + erro)
C14 → C15 → C16 ⇄ C17   (BFF + auth cookie)
C18 · C19 · C20 · C21   (testes/CI/infra, paralelizáveis)
```

> Observação: os itens de fases diferentes são independentes salvo o "Depende de". Dá pra reordenar as Fases 6 (i18n) e 4/5 se preferir ver UX antes; só respeite os pares coordenados e as dependências marcadas.
