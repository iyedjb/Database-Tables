# Mapa de bancos de dados — Ecossistema EduTok

Documentação consolidada dos **cinco diretórios de código**, que formam quatro produtos do ecossistema. O levantamento foi feito a partir dos schemas, migrações, clientes HTTP e rotinas de criação de tabelas presentes no código em **17/09/2026**.

## Visão geral

| Diretório | Papel | Banco próprio | Tabelas físicas |
|---|---|---:|---:|
| `EduTok/` | Aplicação web, API Express e módulo Eduna | PostgreSQL | 26 |
| `Edutok Para Escolas/` | Aplicação escolar e API Express | PostgreSQL | 4 |
| `EduTok Mail/` | Cliente web e API de e-mail | SQLite | 13 |
| `KeepUp/` | Aplicativo Expo/React Native | Não | 0 |
| `KeepUp Backend/` | API Fastify do aplicativo KeepUp | MySQL 8.4 | 8 |

> **Importante:** EduTok e EduTok Para Escolas são produtos relacionados, mas as configurações locais atuais apontam para bancos PostgreSQL diferentes. KeepUp e KeepUp Backend formam um único produto: o aplicativo móvel não possui banco servidor próprio e consome o banco MySQL administrado pelo backend.

## Como a arquitetura funciona

### Fluxo geral

```text
Navegador EduTok ───────────────► API Express EduTok ─────────► PostgreSQL EduTok
                                        │                           ▲
                                        └─ proxy /eduna ─► Eduna ──┘

Navegador Escolas ──────────────► API Express Escolas ────────► PostgreSQL Escolas
Aplicativo KeepUp ─► API KeepUp ─┬► MySQL KeepUp
                                 ├► API Escolas, por proxy
                                 └► API EduTok Mail, por proxy

Cliente EduTok Mail ─────────────► API Hono/tRPC ──────────────► SQLite
                                                        └──────► IMAP/SMTP/Stalwart
```

Nenhum frontend acessa diretamente um banco servidor. Os clientes chamam APIs HTTP/WebSocket; somente os backends possuem as credenciais dos bancos.

### `EduTok/`: aplicação, API e banco

- O processo inicia o PostgreSQL antes de carregar as rotas. A API principal usa Express, normalmente na porta `3300`, e registra as rotas sob `/api`.
- O `dbStore` cria um pool `pg` de até **10 conexões**, com timeout de conexão de 10 segundos e ociosidade de 30 segundos.
- Ao iniciar, o backend lê a tabela `store`, descriptografa as coleções e mantém uma cópia em memória. Alterações são agrupadas, gravadas em transações e sincronizadas entre instâncias com `LISTEN/NOTIFY`. Há um flush imediato serializado e uma verificação de segurança a cada 5 segundos.
- Arquivos e sessões persistidas também usam linhas da tabela `store`, com chaves como `file:*`, `upload:*` e `session:*`; não são novas tabelas.
- Sessões de aplicação usam Redis quando disponível, mantendo uma cópia de compatibilidade no PostgreSQL. Se Redis falhar, o armazenamento baseado no `store` assume o atendimento.
- O banco de questões oficiais usa o mesmo pool principal através de `queryDatabase` e `withDatabaseClient`.
- O Eduna roda como aplicação Next.js integrada e é exposto pelo proxy do EduTok. Ele cria **outro pool PostgreSQL** via Drizzle: usa `POSTGRES_URL` quando definida e, caso contrário, a mesma `DATABASE_URL` do EduTok. O pool não define `max` no código; o driver `pg` usa seu padrão. Os timeouts configurados são 3 segundos para conexão e 10 segundos para consulta/comando.
- Consequência operacional: mesmo usando o mesmo banco, EduTok e Eduna não compartilham o objeto de pool; cada processo administra suas próprias conexões.

### `Edutok Para Escolas/`: aplicação, API e banco

- A API também usa Express. Ela atende o portal web, integrações, aplicativo escolar e as rotas consumidas pelo KeepUp sob `/api`.
- O armazenamento principal usa um pool `pg` de até **10 conexões**, com a mesma estratégia de cache em memória, criptografia, gravação serializada e `LISTEN/NOTIFY` da tabela `store`.
- O arquivo histórico de estudantes cria um pool separado, de até **5 conexões**, para `school_archive_settings` e `student_archive_snapshots`.
- O núcleo bancário cria outro pool separado, de até **4 conexões**, para `banking_records`.
- Se os três módulos forem usados simultaneamente, o processo pode manter até **19 conexões PostgreSQL**. Todos usam a mesma `DATABASE_URL`, embora sejam pools distintos.
- Sessões web ficam no Redis quando ele está disponível. Sem Redis, o `JsonSessionStore` usa a persistência da própria aplicação.
- O aplicativo KeepUp pode autenticar com token bearer `esa_*`; o middleware resolve esse token para usuário e escola sem criar uma sessão web adicional.

### `EduTok Mail/`: API, SQLite e servidor de e-mail

- O servidor roda em Bun com Hono, normalmente na porta `3030`.
- SQLite é local ao processo e síncrono; portanto, **não existe pool de conexões**. Na inicialização, o serviço abre `SQLITE_PATH`, ativa WAL e chaves estrangeiras e executa o bootstrap idempotente das 13 tabelas.
- A API principal é tRPC, disponível em `/trpc/*` e `/api/trpc/*`. Há também superfícies REST para `/auth`, `/owner`, `/mail`, `/admin`, `/provisioning`, `/v1` e `/mobile`.
- O SQLite guarda sessões, preferências, administração, domínios e logs transacionais. O conteúdo das caixas postais continua no servidor de e-mail e é lido/escrito por IMAP e SMTP.
- Stalwart é usado para provisionamento administrativo de domínios e caixas postais; não é uma tabela nem um segundo banco controlado pelo schema deste projeto.

### `KeepUp/`: aplicativo frontend

- É um aplicativo Expo/React Native e não abre conexão com PostgreSQL, MySQL ou SQLite servidor.
- Um cliente Axios usa `EXPO_PUBLIC_API_URL`, por padrão `https://keepup.edutok.online`, envia cookies com `withCredentials` e acrescenta o bearer token salvo como `auth_token`.
- No dispositivo, dados sensíveis e caches passam pelo wrapper de armazenamento: SecureStore no ambiente nativo e `localStorage` no navegador, com cache de memória como apoio. Esses itens locais não são tabelas do banco do backend.
- O aplicativo acessa tarefas, conversas de IA, e-mail e recursos escolares pela API KeepUp. Atualizações de e-mail em tempo real usam WebSocket.

### `KeepUp Backend/`: API, pool MySQL e proxies

- A API usa Fastify, normalmente na porta `3007`, com CORS, JWT, limite global de 240 requisições por minuto e limite de corpo de 50 MB.
- O `mysql2` cria um pool de até **12 conexões**, com keep-alive, `utf8mb4` e múltiplos comandos habilitados para o bootstrap.
- Antes de aceitar tráfego, o processo tenta alcançar o MySQL por até 30 tentativas e executa `CREATE TABLE IF NOT EXISTS` para as 8 tabelas. As consultas da aplicação usam SQL parametrizado diretamente, sem ORM.
- Rotas próprias incluem OTP (`/v1/auth/otp/*`), dados sincronizados (`/v1/data/*`), tarefas, histórico de IA, agente Eduna, contas de e-mail e WebSocket em `/ws/mail`.
- Rotas `/api/*` que não pertencem ao KeepUp são encaminhadas para o EduTok Para Escolas. Rotas legadas de e-mail (`/mobile`, `/auth`, `/oauth`, `/google` e `/microsoft`) são encaminhadas para o EduTok Mail.
- Respostas JSON elegíveis do proxy são registradas de forma sanitizada em `app_records`; cada chamada também gera auditoria em `audit_events`.
- O backend aceita JWT próprio e consegue validar sessões legadas consultando `/api/auth/session` do EduTok Para Escolas. Tokens de contas de e-mail são protegidos com AES-256-GCM antes da gravação.

### Pools e conexões, resumidos

| Codebase | Mecanismo | Limite/configuração |
|---|---|---|
| EduTok principal | `pg.Pool` | máximo 10 |
| Eduna dentro do EduTok | `pg.Pool` + Drizzle | máximo não definido no código; padrão do driver |
| EduTok Para Escolas — principal | `pg.Pool` | máximo 10 |
| EduTok Para Escolas — arquivo | `pg.Pool` | máximo 5 |
| EduTok Para Escolas — bancário | `pg.Pool` | máximo 4 |
| EduTok Mail | `bun:sqlite` | conexão local única; sem pool |
| KeepUp aplicativo | Axios + armazenamento local | sem conexão direta a banco |
| KeepUp Backend | `mysql2/promise` | máximo 12 |

## 1. EduTok

O banco PostgreSQL do EduTok reúne três áreas: armazenamento geral da plataforma, banco de questões oficiais e o módulo Eduna. Na configuração atual, o Eduna usa a mesma `DATABASE_URL` do produto principal quando não existe uma `POSTGRES_URL` exclusiva.

### Núcleo da plataforma e questões oficiais

| Tabela | Área | Finalidade |
|---|---|---|
| `store` | Plataforma | Armazenamento chave–valor criptografado que concentra os dados gerais da aplicação. |
| `official_exams` | Avaliações | Catálogo de provas oficiais, como ENEM e SAEB. |
| `official_exam_documents` | Avaliações | Documentos e cadernos vinculados às provas oficiais. |
| `official_questions` | Avaliações | Questões oficiais estruturadas e seus metadados. |
| `official_question_options` | Avaliações | Alternativas e resposta correta de cada questão. |
| `official_question_assets` | Avaliações | Imagens e demais recursos associados às questões. |
| `official_exam_import_runs` | Avaliações | Histórico e resultado das importações de provas. |

### Eduna

| Tabela | Área | Finalidade |
|---|---|---|
| `user` | Identidade | Usuários, perfil, preferências e situação da conta. |
| `session` | Identidade | Sessões autenticadas dos usuários. |
| `account` | Identidade | Contas e provedores de autenticação vinculados. |
| `verification` | Identidade | Tokens temporários de verificação. |
| `chat_thread` | Conversas | Conversas criadas pelos usuários. |
| `chat_message` | Conversas | Mensagens pertencentes a uma conversa. |
| `agent` | IA | Agentes personalizados e suas instruções. |
| `bookmark` | Organização | Itens salvos pelo usuário. |
| `archive` | Organização | Arquivos ou agrupamentos de conteúdo. |
| `archive_item` | Organização | Itens associados a cada arquivo. |
| `mcp_server` | Integrações | Servidores MCP cadastrados. |
| `mcp_server_tool_custom_instructions` | Integrações | Instruções personalizadas por ferramenta MCP e usuário. |
| `mcp_server_custom_instructions` | Integrações | Instruções gerais por servidor MCP e usuário. |
| `mcp_oauth_session` | Integrações | Estado e tokens das autorizações OAuth de servidores MCP. |
| `workflow` | Automação | Definição de workflows do usuário. |
| `workflow_node` | Automação | Nós e configurações de cada workflow. |
| `workflow_edge` | Automação | Ligações entre os nós de um workflow. |
| `chat_export` | Compartilhamento | Exportações compartilháveis de conversas. |
| `chat_export_comment` | Compartilhamento | Comentários feitos em uma exportação de conversa. |

### Dados lógicos dentro de `store`

`store` é uma tabela física genérica (`key`, `value`). Portanto, entidades como usuários, escolas e turmas não aparecem como tabelas PostgreSQL separadas. Elas são coleções criptografadas dentro dessa tabela. Os principais domínios armazenados ali são:

- usuários, escolas, professores, alunos, responsáveis, turmas e índices de autenticação;
- presença, notas, boletins, calendários, tarefas, notificações e eventos;
- chats, feed, vídeos, biblioteca, relatórios e dados dos estudantes;
- sessões do EduGen, progresso, erros, simulados e resultados OMR;
- chaves de API, integrações Google/Canva/Microsoft, push tokens e sessões por QR Code.

## 2. EduTok Para Escolas

O produto usa outro banco PostgreSQL. Assim como no EduTok, a maior parte dos módulos fica consolidada na tabela `store`.

| Tabela | Área | Finalidade |
|---|---|---|
| `store` | Plataforma | Armazenamento chave–valor criptografado dos módulos escolares. |
| `school_archive_settings` | Arquivo escolar | Configuração do serviço de arquivamento por escola e seus limites. |
| `student_archive_snapshots` | Arquivo escolar | Cópias criptografadas dos registros históricos de estudantes. |
| `banking_records` | Financeiro | Registros bancários criptografados, separados por instituição e tipo. |

### Dados lógicos dentro de `store`

Os principais domínios são:

- escolas, redes, usuários, professores, alunos, responsáveis, turmas e convites;
- matrículas digitais, frequência, notas, boletins, tarefas e transição de ano letivo;
- calendário, eventos, comunicados, chat, notificações, feed e vídeos;
- financeiro escolar, biblioteca, relatórios e auditoria;
- EduGen, OMR, integrações externas, tokens do aplicativo e configurações da plataforma.

Sessões web podem ser mantidas no Redis quando disponível; no modo alternativo, são persistidas no armazenamento JSON da aplicação. Elas não criam outra tabela PostgreSQL.

## 3. EduTok Mail

O EduTok Mail usa SQLite para estado da aplicação. As mensagens de e-mail não são armazenadas nessas tabelas: elas permanecem no servidor de e-mail e são acessadas por IMAP/SMTP.

| Tabela | Área | Finalidade |
|---|---|---|
| `session` | Acesso ao e-mail | Sessões do usuário e credenciais IMAP criptografadas. |
| `mobile_auth_flow` | Autenticação | Fluxos temporários de autenticação do aplicativo móvel. |
| `user_settings` | Preferências | Configurações pessoais da interface. |
| `user_hotkeys` | Preferências | Atalhos personalizados por usuário. |
| `email_template` | Composição | Modelos de mensagens salvos. |
| `cookie_consent` | Privacidade | Preferências de consentimento de cookies. |
| `owner_account` | Administração | Contas proprietárias do painel administrativo. |
| `organization` | Administração | Organizações pertencentes a uma conta proprietária. |
| `owner_session` | Administração | Sessões do painel vinculadas a proprietário e organização. |
| `managed_domain` | Domínios | Domínios de e-mail gerenciados e seu estado de verificação. |
| `managed_mailbox` | Caixas postais | Caixas postais criadas dentro dos domínios gerenciados. |
| `transactional_api_key` | API transacional | Chaves de API para envio de e-mails transacionais. |
| `transactional_message_log` | API transacional | Histórico e estado dos envios transacionais. |

## 4. KeepUp

O aplicativo em `KeepUp/` é o cliente móvel. O diretório `KeepUp Backend/` contém a API e é o único responsável pelo banco MySQL do produto. O schema padrão do Docker usa o banco `keepup`.

| Tabela | Área | Finalidade |
|---|---|---|
| `users` | Identidade | Usuários do KeepUp e vínculo opcional com escola/identidade externa. |
| `otp_challenges` | Autenticação | Desafios OTP para login, cadastro e recuperação. |
| `mail_accounts` | E-mail | Contas de e-mail vinculadas, com tokens criptografados. |
| `app_records` | Sincronização | Registros genéricos do aplicativo, organizados por namespace. |
| `account_tasks` | Tarefas | Tarefas sincronizadas por conta. |
| `ai_conversations` | IA | Conversas de IA e respectivos títulos. |
| `ai_messages` | IA | Mensagens e modelos usados nas conversas de IA. |
| `audit_events` | Auditoria | Eventos de API, rotas, status e metadados operacionais. |

O backend também atua como ponte para APIs legadas do EduTok Para Escolas e do EduTok Mail. Esse acesso não torna os bancos desses produtos parte do banco KeepUp.

## Relações principais

```text
EduTok ─────────────── PostgreSQL próprio
  └─ Eduna ────────── mesmo PostgreSQL por padrão

EduTok Para Escolas ─ PostgreSQL separado

EduTok Mail ───────── SQLite de estado + servidor IMAP/SMTP

KeepUp (aplicativo) ── API ── KeepUp Backend ── MySQL
                               ├─ integração com EduTok Para Escolas
                               └─ integração com EduTok Mail
```

## Fontes de verdade

- EduTok: `server/index.ts`, `server/dbStore.ts`, `server/sessionStore.ts`, `server/officialExamRepository.ts`, `apps/eduna/src/lib/db/pg/db.pg.ts` e `apps/eduna/src/lib/db/pg/schema.pg.ts`.
- EduTok Para Escolas: `server/index.ts`, `server/dbStore.ts`, `server/archiveStore.ts` e `server/bankingCore.ts`.
- EduTok Mail: `server/src/main.ts`, `server/src/db/index.ts`, `server/src/db/schema.ts` e `server/src/db/bootstrap.ts`.
- KeepUp aplicativo: `services/api.ts`, `services/storage.ts` e os serviços de domínio em `services/`.
- KeepUp Backend: `src/server.ts`, `src/db.ts`, `src/proxy.ts`, `src/security.ts` e `docker-compose.yml`.

### Cobertura da conferência

- Foram pesquisadas todas as ocorrências de `CREATE TABLE`, `pgTable` e `sqliteTable` fora de dependências e artefatos de build.
- Foram comparados schemas atuais, migrações e bootstraps; tabelas antigas `project` e `mcp_server_binding` do Eduna não entram na contagem porque migrações posteriores as removem.
- As **51 tabelas declaradas** estão listadas neste documento: 26 + 4 + 13 + 8. Como `store` e `session` existem em mais de um banco, são 49 nomes únicos distribuídos em 51 tabelas físicas.
- Também foram conferidos os cinco pontos de entrada de aplicação/API, seus clientes de banco, pools, fallback de sessão e integrações entre serviços.

> **Limite da garantia:** o catálogo dos dois PostgreSQL remotos não estava acessível a partir deste ambiente durante a revisão — um destino expirou por timeout e o outro usa um hostname interno que não resolveu fora da rede de implantação. Assim, o documento garante integralmente o que o código atual cria e utiliza. Tabelas criadas manualmente em produção, fora do versionamento, só podem ser garantidas com acesso de leitura ao `information_schema` de cada servidor.
