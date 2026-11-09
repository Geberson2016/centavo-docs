# Plano de evolução do Centavo — quebrado em commits

Plano incremental para sair do estado atual e chegar na arquitetura-alvo (**React → BFF → Spring → Postgres**, contract-first, i18n, SOLID). Cada item abaixo é **1 commit atômico**, em ordem de dependência. Conforme você finaliza um item, use a **mensagem de commit sugerida** (prefixo em inglês por Conventional Commits, texto em português — conforme `AGENTS.md §0`).

## Como usar
- Faça um item de cada vez, na ordem. Cada item é pensado para **manter o build verde** ao final do commit.
- `[backend]` = repo `centavo-backend`; `[frontend]` = repo `centavo-frontend`; `[bff]` = novo repo/pacote do BFF.
- ⚠️ **Par coordenado** = dois commits (repos diferentes) que devem ser **deployados juntos** porque mudam o contrato ao mesmo tempo.
- Marque `[x]` conforme concluir.

---

## 📌 Status atual (atualizado em 2026-11-08)

- **Concluídos:** C1–C17 (Fases 0 a 7). BFF: scaffold `e0481ef`, encaminhamento `fa9c8d7` e sessão `d865ee2`; frontend no BFF `fc0be69`. Fase 5: C9 `9ecb601` (backend) e C10 `402f34f` (frontend) (Fase 6, i18n e toasts de erro). Além do plano: seletor de idioma com persistência (`cd185c0`), fix do cadastro (`c3e3b8c`) e fix de segurança da categoria de outro usuário no lançamento (`731aac6`, backend).
- **Próximo passo:** **C18 — testes do backend** (Fase 8). C18 a C21 podem ser feitos em qualquer ordem; C20 (CI) e C21 (compose versionado) esbarram na decisão de deixar infra para depois.
- **BFF:** Fastify 5 + TS, em repositório próprio `centavo-bff` (decidido em 2026-11-08), porta 3000. Remoto: `Geberson2016/centavo-bff`. Rotas da UI em `/bff/*`, descritas no `openapi.yaml` do BFF (de onde o front vai gerar os tipos no C17).
- **Pendentes:** C18–C21 (testes, CI, infra).
- **Pendência conhecida:** o Spring ainda devolve 403 sem corpo para token ausente ou inválido (falta um `AuthenticationEntryPoint` → 401). Para a UI isso já está resolvido: o BFF traduz para `401 UNAUTHENTICATED` e apaga o cookie. Corrigir no Spring é opcional.
- **Rodar local:** Spring (`docker compose up -d` aqui), BFF (`npm run dev` no `centavo-bff`, `:3000`, precisa do `.env` com `SESSION_KEY`) e front (`npm run dev`, `:5173`, faz proxy de `/bff`).
- **Ambiente local:** `docker compose up -d` em `~/Projects/Centavo` sobe Postgres (`localhost:5433`) e a API (`localhost:8090`); o front roda com `npm run dev` (`localhost:5173`, `.env.local` aponta para a `:8090`). Fora dos repos, sem versionamento.

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

### [ ] C18 · [backend] Testes
JUnit 5 + Mockito para services e `@WebMvcTest` para controllers; Testcontainers (Postgres) para repositórios/integração.
- **Commit:** `test(backend): adiciona testes de service, controller e integração`

### [ ] C19 · [frontend] Testes
Vitest + React Testing Library para hooks e componentes; smoke E2E (Playwright) opcional (login → dashboard).
- **Commit:** `test(frontend): adiciona Vitest e testes de componente`

### [ ] C20 · [ci] Pipeline
GitHub Actions rodando lint + typecheck + build + test para backend, frontend e BFF nos PRs.
- **Commit:** `ci: adiciona pipeline de lint, build e testes`

### [ ] C21 · [infra] Docker Compose do stack
`docker-compose` com postgres + backend + bff + web para dev local com um comando; `Dockerfile` por serviço.
- **Commit:** `chore(infra): adiciona docker-compose com postgres, backend, bff e web`

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
