# Resumo Completo das Tabelas e Coleções do Banco de Dados
## EduTok & EduTok Para Escolas

> **Modo Executado**: Leitura e Análise (Read-Only). Nenhum comando de alteração ou execução foi executado no servidor ou no banco de dados.

---

## 1. Arquitetura do Banco de Dados

O ecossistema **EduTok** utiliza uma arquitetura híbrida de armazenamento dividida em:

1. **PostgreSQL Relacional (`DATABASE_URL`)**:
   - **Tabela `store` (Key-Value Protegido)**: Armazena coleções e sub-coleções de dados criptografados em tempo de execução via **AES-256-GCM** (usando a chave `DATA_ENCRYPTION_KEY`). Possui sincronização em tempo real entre instâncias via `LISTEN / NOTIFY` no canal `store_updated`.
   - **Tabela `school_archive_settings`**: Armazena as configurações do módulo de arquivamento por escola.
   - **Tabela `student_archive_snapshots`**: Armazena histórico criptografado em formato snapshot dos alunos por ano letivo.

2. **Firebase Realtime Database (Legado / Sincronização Secundária)**:
   - Utilizado para sincronização leve em tempo real (notificações push, presença e anúncios voláteis).

---

## 2. Divisão de Estrutura: EduTok "Normal" vs. EduTok "Para Escolas"

| Módulo / Camada | EduTok (Normal / Alunos & Feed) | EduTok Para Escolas (B2B Multitenant) |
| :--- | :--- | :--- |
| **Escopo de Dados** | Global (`users`, `efeed`, `videos`, `edugen`) | Isolado por Escola (`${schoolId}`) |
| **Gestão Acadêmica** | Simples (Feed de Vídeos e Exercícios) | Completa (Turmas, Notas, Frequência, Remanejamento) |
| **Redes Escolares** | N/A | Gestão de Redes de Ensino (`networks`, `networkMembers`) |
| **Módulo Financeiro** | N/A | Mensalidades, Matrículas, Cobranças Pix/Boleto e Caixa |
| **Matrícula Digital** | N/A | Links tokenizados e formulários digitais de matrícula |
| **Portal dos Pais** | N/A | Associação Responsável-Aluno e acompanhamento |

---

## 3. Catálogo Completo das Tabelas e Coleções

### 3.1 Tabelas Relacionais Nativas no PostgreSQL

#### 1. `store`
- **Descrição**: Tabela principal Key-Value do PostgreSQL para persistência criptografada de coleções.
- **Campos**:
  - `key` (`TEXT PRIMARY KEY`): Identificador da coleção ou sub-coleção (ex: `users`, `schools`, `finance:school123`).
  - `value` (`TEXT NOT NULL`): Conteúdo JSON criptografado com AES-256-GCM.

#### 2. `school_archive_settings`
- **Descrição**: Configurações de retenção e arquivo de dados históricos das escolas.
- **Campos**:
  - `school_id` (`TEXT PRIMARY KEY`): ID da escola.
  - `provider` (`TEXT CHECK (provider = 'edutok')`): Provedor do arquivo.
  - `plan_code` (`TEXT`): Plano de arquivamento contratado.
  - `record_limit` (`INTEGER`): Limite de registros em arquivo.
  - `connected_at` (`TIMESTAMPTZ`): Data de conexão.
  - `updated_at` (`TIMESTAMPTZ`): Última atualização.

#### 3. `student_archive_snapshots`
- **Descrição**: Snapshots anuais arquivados de históricos de alunos.
- **Campos**:
  - `id` (`UUID PRIMARY KEY`): Identificador único do snapshot.
  - `school_id` (`TEXT NOT NULL`): ID da escola.
  - `student_id` (`TEXT NOT NULL`): ID do aluno.
  - `archived_year` (`INTEGER NOT NULL`): Ano arquivado.
  - `academic_year` (`INTEGER`): Ano acadêmico.
  - `provider` (`TEXT`): Provedor.
  - `payload` (`TEXT`): Dados criptografados do snapshot do aluno.
  - `checksum` (`TEXT`): Hash SHA-256 para validação de integridade.
  - `created_at` / `updated_at` (`TIMESTAMPTZ`).
  - **Índices**: Único `(school_id, student_id)` e índice composto `(school_id, archived_year DESC)`.

---

### 3.2 Coleções Globais do Sistema (Key-Value em `store`)

#### 4. `users`
- **Chave no `store`**: `"users"`
- **Descrição**: Perfil global de usuários (Alunos, Professores, Diretores, Admins, Pais).
- **Campos Principais**:
  - `uid`: ID único do usuário.
  - `email`, `displayName`, `photoURL`, `phone`, `cpf`, `birthdate`.
  - `schoolId`, `networkId`, `role` (`student`, `teacher`, `school_admin`, `parent`, `superadmin`).
  - `classes`: Lista de IDs das turmas vinculadas.
  - `permissions`: Array de permissões administrativas granulares.
  - `lgpdConsent`: Objeto com consentimentos LGPD (analytics, marketing, functional, ipAddress, timestamp).

#### 5. `schools`
- **Chave no `store`**: `"schools"`
- **Descrição**: Cadastro de instituições de ensino (Escolas e Redes).
- **Campos Principais**:
  - `id`, `name`, `slug`, `cnpj`, `email`, `phone`, `address`, `city`, `state`, `logoURL`.
  - `accountType`: `'school'` ou `'network'`.
  - `networkId`: ID da rede a qual pertence (se aplicável).
  - `subjects`: Lista de disciplinas oferecidas.
  - `scheduleConfig` & `shiftConfigs`: Configuração de turnos (Manhã/Tarde/Noite), horários e intervalos.
  - `assessmentPeriod`: Sistema de avaliação (`bimestre`, `trimestre`, `semestre`, `anual`).
  - `assessmentMaxGrade`: Nota máxima do período.
  - `customDomain`, `domainVerificationToken`, `dnsStatus`: Domínio personalizado da escola.

#### 6. `authIndex`
- **Chave no `store`**: `"authIndex"`
- **Descrição**: Índice rápido de busca para autenticação por CPF, E-mail ou Nome de usuário mapeado para o `uid`.

#### 7. Redes de Ensino (`networks`, `networkMembers`, `networkDomains`, `networkSubscriptions`, `networkMailboxes`, `networkAuditLogs`)
- **Chaves no `store`**: `"networks"`, `"networkMembers"`, `"networkDomains"`, `"networkSubscriptions"`, `"networkMailboxes"`, `"networkAuditLogs"`.
- **Descrição**: Gestão de grupos multiescolas/redes privadas de ensino, logs de auditoria e caixas postais corporativas.

#### 8. `pushTokens`
- **Chave no `store`**: `"pushTokens"`
- **Descrição**: Tokens de notificações push mobile (Expo & FCM) cadastrados por dispositivo e usuário.

#### 9. `apiKeys_collection`
- **Chave no `store`**: `"apiKeys_collection"`
- **Descrição**: Chaves de API de desenvolvedor para integração de terceiros.
- **Campos**: `id`, `uid`, `keyHash`, `keyPrefix`, `name`, `tokensUsed`, `tokensLimit`, `requestsToday`, `lastResetDate`, `active`.

#### 10. `sessions`
- **Chave no `store`**: `"sessions"`
- **Descrição**: Sessões ativas de login via cookie express-session.

#### 11. `googleLinks`
- **Chave no `store`**: `"googleLinks"`
- **Descrição**: Vínculo entre contas de login social Google (`googleUid`, `googleEmail`) e perfis internos (`userUid`, `cpf`).

#### 12. `teacher_invitation_tokens`
- **Chave no `store`**: `"teacher_invitation_tokens"`
- **Descrição**: Convites e tokens de cadastro tokenizados enviados por e-mail para novos professores.

#### 13. `qrSessions`
- **Chave no `store`**: `"qrSessions"`
- **Descrição**: Sessões de autenticação por QR Code para login rápido entre dispositivos.

#### 14. `grade_reports`
- **Chave no `store`**: `"grade_reports"`
- **Descrição**: Links temporários e públicos para consulta de boletins escolares.

#### 15. EduGen AI (`edugen_sessions`, `edugen_progress`, `edugen_errors`)
- **Chaves no `store`**: `"edugen_sessions"`, `"edugen_progress"`, `"edugen_errors"`.
- **Descrição**: Histórico de simulados gerados por IA, estatísticas de desempenho por disciplina/tópico e log de tópicos fracos/erros do estudante.

---

### 3.3 Coleções Multitenant Isoladas por Escola (`${schoolId}`)

#### 16. Alunos da Escola (`students` / `students:${schoolId}`)
- **Chave**: `students:${schoolId}`
- **Descrição**: Cadastro completo e ficha cadastral do estudante.
- **Campos Principais**:
  - `id`, `name`, `schoolId`, `grade`, `class`, `cpf`, `cpfHash`, `cpfMasked`, `birthdate`, `rg`, `gender`, `bloodType`, `colorRace`, `nationality`.
  - `filiacao1`, `filiacao2`, `responsavel`: Dados dos pais e responsáveis legais.
  - `address`, `addressCEP`, `addressBairro`, `addressCity`, `addressState`.
  - `specialNeeds`, `specialNeedsDescription`: Necessidades educacionais especiais.
  - `enrollmentDate`, `enrollmentHistory`: Histórico de matrículas e aprovações por ano letivo.
  - `averageGrade`, `totalAbsences`, `subjectAbsences`.

#### 17. Professores da Escola (`teachers` / `teachers:${schoolId}`)
- **Chave**: `teachers:${schoolId}`
- **Descrição**: Cadastro de docentes alocados na instituição.
- **Campos**: `id`, `name`, `cpf`, `email`, `phone`, `photoURL`, `schoolIds`, `subjects`, `academicDegree`, `hireDate`, `active`, `accessPaused`.

#### 18. Responsáveis/Pais (`parents` / `parents:${schoolId}`)
- **Chave**: `parents:${schoolId}`
- **Descrição**: Cadastro de pais/responsáveis vinculados aos alunos.
- **Campos**: `id`, `name`, `cpf`, `birthdate`, `email`, `phone`, `schoolId`, `childrenIds` (array de IDs dos filhos).

#### 19. Turmas (`classes` / `classes:${schoolId}`)
- **Chave**: `classes:${schoolId}`
- **Descrição**: Turmas e salas de aula da escola.
- **Campos**: `id`, `name`, `grade` (ex: "6º Ano A"), `year`, `shift` (Manhã/Tarde), `school`, `teachers` (atribuição docente por matéria), `students` (array de IDs), `schedule` (grade horária).

#### 20. Frequência / Chamada (`attendance` & `finalized_attendance`)
- **Chaves**: `"attendance"`, `"finalized_attendance"`
- **Descrição**: Registros diários de presença/falta por disciplina e encerramento/assinatura de chamada pelo professor.

#### 21. Notas e Boletins (`grades:${schoolId}`)
- **Chave**: `grades:${schoolId}`
- **Descrição**: Lançamento de notas por avaliação, disciplina e trimestre.
- **Campos**: `id`, `uid` (ID do aluno), `subject`, `grade`, `trimestre`, `maxGrade`, `gradeFormat`, `type` (provisória/final/trabalho), `teacherUid`, `timestamp`.

#### 22. Tarefas e Trabalhos (`assignments:${schoolId}`)
- **Chave**: `assignments:${schoolId}`
- **Descrição**: Atividades postadas por professores para turmas.
- **Campos**: `id`, `title`, `description`, `subject`, `dueDate`, `classId`, `teacherId`, `attachments`, `submissions` (entregas dos alunos com notas e feedbacks), `completions`.

#### 23. Módulo Financeiro (`finance:${schoolId}`, `finance_plans:${schoolId}`, `finance_transactions:${schoolId}`)
- **Chaves**: `finance:${schoolId}`, `finance_plans:${schoolId}`, `finance_transactions:${schoolId}`
- **Descrição**:
  - `finance:${schoolId}`: Faturas e mensalidades (`amountCents`, `dueDate`, `status`: open/paid/overdue, `pixCopyPaste`, `boletoUrl`).
  - `finance_plans:${schoolId}`: Planos de pagamento e categorias de mensalidade.
  - `finance_transactions:${schoolId}`: Lançamentos de caixa (entradas e saídas manuais).

#### 24. Sistema de Chat (`chat:${schoolId}`)
- **Chave**: `chat:${schoolId}`
- **Descrição**: Salas de bate-papo de turma, grupos e mensagens diretas Professor-Aluno/Pai.
- **Campos**: `rooms` (salas) e `messages` (mensagens com suporte a reações emoji, resposta/swipe a mensagens, anexos de áudio, imagem e documentos).

#### 25. Matrícula Digital (`digital_enrollments:${schoolId}`, `digital_tokens:${schoolId}`)
- **Chaves**: `digital_enrollments:${schoolId}`, `digital_tokens:${schoolId}`
- **Descrição**: Processo de pré-matrícula e rematrícula online por formulário público com token de segurança.

#### 26. Feed Escolar / E-feed (`efeed:${schoolId}`)
- **Chave**: `efeed:${schoolId}`
- **Descrição**: Rede social interna da escola (Posts, Stories de 24h, Comentários, Enquetes, Badges e Pontuação de Engajamento).

#### 27. Biblioteca Digital (`library_collection:${schoolId}`)
- **Chave**: `library_collection:${schoolId}`
- **Descrição**: Acervo de materiais didáticos, apostilas, PDFs, links e mídias compartilhadas.

#### 28. Calendário e Eventos (`calendar:${schoolId}`)
- **Chave**: `calendar:${schoolId}`
- **Descrição**: Eventos acadêmicos, provas, feriados e suspensão de aulas por turma ou escola.

#### 29. EduTok Vídeos (`videos_collection:${schoolId}`)
- **Chave**: `videos_collection:${schoolId}`
- **Descrição**: Feed de vídeos educativos curtos no estilo TikTok com comentários e curtidas.

#### 30. Notificações Internas (`notifications_v2:${schoolId}`)
- **Chave**: `notifications_v2:${schoolId}`
- **Descrição**: Central de notificações por usuário (novas notas, tarefas, posts do efeed, avisos).

#### 31. Remanejamento / Transição de Ano (`yearTransitionDrafts`)
- **Chave**: `"yearTransitionDrafts"` (indexado por `${schoolId}:${fromYear}`)
- **Descrição**: Rascunho de decisões de encerramento de ano letivo (Aprovado, Retido, Transferido, Formado) antes da efetivação nas fichas dos alunos.

---

## 4. Como Criar e Inicializar o Banco de Dados (Schema SQL)

Caso deseje recriar o esquema nativo no PostgreSQL via DDL SQL:

```sql
-- 1. Tabela Principal de Key-Value Criptografado
CREATE TABLE IF NOT EXISTS store (
  key TEXT PRIMARY KEY,
  value TEXT NOT NULL DEFAULT ''
);

-- 2. Tabela de Configuração de Arquivo Histórico por Escola
CREATE TABLE IF NOT EXISTS school_archive_settings (
  school_id TEXT PRIMARY KEY,
  provider TEXT NOT NULL CHECK (provider = 'edutok'),
  plan_code TEXT,
  record_limit INTEGER,
  connected_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- 3. Tabela de Snapshots Históricos dos Alunos
CREATE TABLE IF NOT EXISTS student_archive_snapshots (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  school_id TEXT NOT NULL,
  student_id TEXT NOT NULL,
  archived_year INTEGER NOT NULL,
  academic_year INTEGER,
  provider TEXT NOT NULL CHECK (provider = 'edutok'),
  payload TEXT,
  checksum TEXT NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  CONSTRAINT uk_school_student UNIQUE (school_id, student_id)
);

CREATE INDEX IF NOT EXISTS student_archive_school_year_idx
ON student_archive_snapshots (school_id, archived_year DESC);
```

---

## 5. Resumo da Estrutura

- **Total de Tabelas SQL Diretas**: 3 (`store`, `school_archive_settings`, `student_archive_snapshots`).
- **Total de Coleções Armazenadas em `store`**: 28 coleções ativas (divididas entre escopo Global e isolamento Multitenant por `${schoolId}`).
- **Segurança e Proteção**: Todos os dados gravados no PostgreSQL na tabela `store` e nos snapshots são encriptados com algoritmo **AES-256-GCM**, garantindo conformidade com a **LGPD** e proteção total dos dados de alunos e escolas.
