# Especificação Técnica: AgendaFácil — Plataforma SaaS de Agendamento Online

> **Ponto de vista deste documento:** a **AgendaFácil** (empresa) entregando a plataforma aos seus **Contratantes** (estabelecimentos que pagam a assinatura), que por sua vez atendem os **Clientes Finais**.

---

## 0. Atores e Glossário

| Ator | Quem é | O que faz na plataforma |
|---|---|---|
| **AgendaFácil (Plataforma)** | A empresa dona do SaaS (nós). | Opera a plataforma, vende planos, cobra mensalidade, suspende/reativa contas, monitora saúde do serviço (backoffice `/plataforma`). |
| **Contratante** | Cliente da AgendaFácil. Barbearia, salão, clínica, nutricionista, estúdio de tatuagem, personal trainer etc. No código é a entidade `Estabelecimento`. | Assina um plano, cadastra serviços, profissionais e horários, acompanha a agenda no `/painel`. |
| **Cliente Final** | Cliente do Contratante (ex: Marina Costa). | Acessa o link público `/agendar/[slug]`, agenda e cancela sozinho. Não tem conta. |
| **Profissional** | Pessoa que executa o serviço, cadastrada pelo Contratante. | Recebe agendamentos. Na v1 **não faz login**. |

**Regra de nomenclatura:** "Contratante" é o termo de negócio; `Estabelecimento` (`estabelecimentos`, `estabelecimentoId`) é o nome técnico no código e no banco.

---

## 1. Visão Geral e Stack

### 1.1 Objetivo
A AgendaFácil é uma plataforma **SaaS multi-tenant** que automatiza o agendamento de horários de pequenos negócios. O Contratante contrata um plano, configura seu negócio e recebe um **link público**. O Cliente Final agenda sozinho, sem intervenção humana, com cálculo automático de disponibilidade em tempo real e (conforme o plano) confirmação e lembretes via WhatsApp.

Do ponto de vista da AgendaFácil, a plataforma também precisa: cadastrar Contratantes, controlar plano e assinatura, liberar/bloquear recursos por plano, cobrar mensalidade e permitir operação e suporte via backoffice.

### 1.2 Planos e Modelo de Negócio

| Plano (`plano`) | Preço/mês | Agenda online | Confirmação e lembretes WhatsApp | Catálogo público (preços e fotos) |
|---|---|---|---|---|
| `BASICO` | R$ 49,00 | ✅ | ❌ | ❌ |
| `PROFISSIONAL` | R$ 79,00 | ✅ | ✅ | ❌ |
| `PREMIUM` | R$ 129,00 | ✅ | ✅ | ✅ |

**Referência de negócio (não é regra de código):**
* Custo operacional estimado da plataforma: ≈ R$ 235/mês (VPS R$ 80, API WhatsApp R$ 100, domínio R$ 5, outros R$ 50). Com 3 Contratantes no plano `PROFISSIONAL` o custo fixo já é coberto.
* Argumento de venda ao Contratante: com ticket médio de R$ 40, evitar ~2 faltas por mês paga o plano `PROFISSIONAL`.
* O backoffice deve expor MRR e custo de notificações para acompanhar o break-even (ver seção 4.3).

### 1.3 Stack Tecnológica
* **Linguagem:** TypeScript (Node.js 20+)
* **Backend:** Express.js (API REST)
* **Persistência / ORM:** PostgreSQL 16+ via Prisma ORM
* **Frontend:** React / Next.js (App Router)
* **Validação:** Zod (request bodies e query params)
* **Testes:** Jest + Supertest (backend), React Testing Library (frontend)
* **Containerização:** Docker e Docker Compose
* **Integração externa:** WhatsApp Business Cloud API (confirmações e lembretes)
* **Agendador de tarefas:** `node-cron` no processo do backend na v1 (lembretes, reenvio e cobrança)
* **Cobrança da assinatura:** porta `ProvedorPagamento` (interface no domínio). O provedor concreto (gateway) é decisão em aberto; ver seção 6.3.

### 1.4 Arquitetura e Restrições
* **Padrão:** Clean Architecture / Camadas dentro do backend Express:
  1. `Domain`: entidades puras, objetos de valor (`HorarioDisponivel`, `Plano`), enums e exceções de domínio.
  2. `Use Cases / Services`: regras de negócio e orquestração (repositórios, WhatsApp, pagamentos).
  3. `Adapters / Controllers`: rotas Express, middlewares, schemas Zod, páginas e componentes Next.js.
  4. `Infrastructure`: repositórios Prisma, migrations, clientes HTTP (WhatsApp, gateway), jobs.
* **Restrição de camadas:** é proibido colocar regra de negócio em rotas Express ou componentes React. Conflito de horário, transição de status, **liberação de recurso por plano** e **bloqueio por assinatura** vivem em `Use Cases`/`Domain`.
* **Multi-tenancy:** banco único, schema único, isolamento por `estabelecimento_id` em toda consulta (RN-09).
* **Fuso horário:** datas persistidas em UTC; cada Contratante tem `fuso_horario` (padrão `America/Maceio`) usado para montar a grade de horários.

### 1.5 Fora de Escopo da v1 (Roadmap)
Pagamento do serviço via Pix pelo Cliente Final, lista de espera, integração Google Agenda (v2); relatórios financeiros, programa de fidelidade, catálogos expandidos (v3); IA para sugestão de horários e remarcação automática (v4). Nada disso deve ser implementado agora.

---

## 2. Requisitos

### 2.1 Requisitos Funcionais
| ID | Descrição | Ator |
|---|---|---|
| RF01 | Cadastro, alteração e exclusão (cancelamento de conta) do Contratante | Contratante / Plataforma |
| RF02 | Cadastro, alteração e exclusão de profissionais | Contratante |
| RF03 | Cadastro, alteração e exclusão de serviços (catálogo) | Contratante |
| RF04 | Login do Contratante no painel | Contratante |
| RF05 | Agendamento online feito pelo próprio Cliente Final | Cliente Final |
| RF06 | Cálculo automático de horários disponíveis | Sistema |
| RF07 | Confirmação e lembretes automáticos via WhatsApp (planos `PROFISSIONAL` e `PREMIUM`) | Sistema |
| RF08 | Cancelamento de agendamento pelo Cliente Final | Cliente Final |
| RF09 | Consulta da agenda pelo Contratante | Contratante |
| RF10 | Exibição opcional do catálogo com preços e fotos (plano `PREMIUM`) | Cliente Final |
| RF11 | **Contratação, upgrade, downgrade e cancelamento de plano** | Contratante |
| RF12 | **Cobrança recorrente e controle do status da assinatura** | Sistema |
| RF13 | **Configuração de horários de funcionamento e de atendimento** | Contratante |
| RF14 | **Backoffice da AgendaFácil: listar Contratantes, suspender/reativar, ver métricas** | Plataforma |

### 2.2 Requisitos Não Funcionais
| ID | Descrição |
|---|---|
| RNF01 | Cada agendamento tem um `codigo` único (8 caracteres alfanuméricos maiúsculos, ex: `AF7K92XQ`). |
| RNF02 | Disponibilidade de horários calculada em tempo real, sem cache que ignore agendamentos recém-criados. |
| RNF03 | Interface responsiva, usável em qualquer dispositivo móvel. |
| RNF04 | Isolamento total de dados entre Contratantes (ver RN-09). |
| RNF05 | LGPD: nome e WhatsApp do Cliente Final são dados pessoais. O Contratante é o controlador e a AgendaFácil é operadora. Não logar esses campos em texto puro e não expô-los fora do tenant dono. |
| RNF06 | Senhas com hash forte (argon2 ou bcrypt). Autenticação por JWT com expiração curta. |
| RNF07 | Rate limiting nos endpoints públicos (`POST /agendamentos`, `PATCH .../status`, `GET .../horarios`) para impedir enumeração de `codigo` e abuso. |
| RNF08 | Falha do WhatsApp nunca pode impedir ou desfazer um agendamento. |

---

## 3. Entidades e Dados (DDL SQL)

### 3.1 Modelo de Dados

```sql
-- Contratantes (Estabelecimentos)
CREATE TABLE estabelecimentos (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    nome VARCHAR(150) NOT NULL,
    slug VARCHAR(160) UNIQUE NOT NULL,
    cnpj_cpf VARCHAR(18) UNIQUE NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,           -- login do painel
    telefone VARCHAR(20),
    senha_hash VARCHAR(255) NOT NULL,
    catalogo_publico BOOLEAN NOT NULL DEFAULT FALSE,
    plano VARCHAR(20) NOT NULL DEFAULT 'BASICO',  -- 'BASICO','PROFISSIONAL','PREMIUM'
    status VARCHAR(20) NOT NULL DEFAULT 'ATIVO',  -- 'ATIVO','SUSPENSO','CANCELADO'
    fuso_horario VARCHAR(50) NOT NULL DEFAULT 'America/Maceio',
    intervalo_grade_minutos INTEGER NOT NULL DEFAULT 30,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    CHECK (plano IN ('BASICO','PROFISSIONAL','PREMIUM')),
    CHECK (status IN ('ATIVO','SUSPENSO','CANCELADO'))
);

-- Horário de funcionamento do Contratante (RF13)
CREATE TABLE horarios_funcionamento (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    estabelecimento_id UUID NOT NULL REFERENCES estabelecimentos(id) ON DELETE CASCADE,
    dia_semana SMALLINT NOT NULL CHECK (dia_semana BETWEEN 0 AND 6), -- 0 = domingo
    abre TIME NOT NULL,
    fecha TIME NOT NULL,
    CHECK (fecha > abre)
);

CREATE TABLE profissionais (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    estabelecimento_id UUID NOT NULL REFERENCES estabelecimentos(id) ON DELETE CASCADE,
    nome VARCHAR(150) NOT NULL,
    ativo BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Horário de atendimento de cada profissional (RF13)
CREATE TABLE horarios_atendimento (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    profissional_id UUID NOT NULL REFERENCES profissionais(id) ON DELETE CASCADE,
    dia_semana SMALLINT NOT NULL CHECK (dia_semana BETWEEN 0 AND 6),
    inicio TIME NOT NULL,
    fim TIME NOT NULL,
    CHECK (fim > inicio)
);

CREATE TABLE servicos (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    estabelecimento_id UUID NOT NULL REFERENCES estabelecimentos(id) ON DELETE CASCADE,
    nome VARCHAR(100) NOT NULL,
    preco NUMERIC(8,2) NOT NULL,
    duracao_minutos INTEGER NOT NULL CHECK (duracao_minutos > 0),
    foto_url VARCHAR(255),
    ativo BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE profissionais_servicos (
    profissional_id UUID NOT NULL REFERENCES profissionais(id) ON DELETE CASCADE,
    servico_id UUID NOT NULL REFERENCES servicos(id) ON DELETE CASCADE,
    PRIMARY KEY (profissional_id, servico_id)
);

-- Agendamentos: FKs de profissional/serviço são RESTRICT para nunca perder histórico (RN-16)
CREATE TABLE agendamentos (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    codigo VARCHAR(12) UNIQUE NOT NULL,
    estabelecimento_id UUID NOT NULL REFERENCES estabelecimentos(id) ON DELETE CASCADE,
    profissional_id UUID NOT NULL REFERENCES profissionais(id) ON DELETE RESTRICT,
    servico_id UUID NOT NULL REFERENCES servicos(id) ON DELETE RESTRICT,
    nome_cliente VARCHAR(150) NOT NULL,
    whatsapp_cliente VARCHAR(20) NOT NULL,
    data_hora TIMESTAMP WITH TIME ZONE NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'CONFIRMADO', -- 'CONFIRMADO','CANCELADO'
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    CHECK (status IN ('CONFIRMADO','CANCELADO'))
);
CREATE INDEX idx_agendamentos_profissional_data ON agendamentos (profissional_id, data_hora);
CREATE INDEX idx_agendamentos_estab_data ON agendamentos (estabelecimento_id, data_hora);

-- Assinatura do Contratante (RF11, RF12)
CREATE TABLE assinaturas (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    estabelecimento_id UUID NOT NULL UNIQUE REFERENCES estabelecimentos(id) ON DELETE CASCADE,
    plano VARCHAR(20) NOT NULL,
    valor_mensal NUMERIC(8,2) NOT NULL,           -- 49.00 | 79.00 | 129.00 no momento da contratação
    status VARCHAR(20) NOT NULL DEFAULT 'ATIVA',  -- 'ATIVA','EM_ATRASO','SUSPENSA','CANCELADA'
    plano_agendado VARCHAR(20),                   -- downgrade que passa a valer no próximo ciclo
    inicio DATE NOT NULL,
    proxima_cobranca DATE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    CHECK (status IN ('ATIVA','EM_ATRASO','SUSPENSA','CANCELADA'))
);

CREATE TABLE cobrancas (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    assinatura_id UUID NOT NULL REFERENCES assinaturas(id) ON DELETE CASCADE,
    valor NUMERIC(8,2) NOT NULL,
    vencimento DATE NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'PENDENTE', -- 'PENDENTE','PAGA','FALHA'
    pago_em TIMESTAMP WITH TIME ZONE,
    referencia_externa VARCHAR(100),                -- id no gateway de pagamento
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    CHECK (status IN ('PENDENTE','PAGA','FALHA'))
);

-- Fila/histórico de mensagens WhatsApp (RF07, UC008)
CREATE TABLE notificacoes_whatsapp (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    agendamento_id UUID NOT NULL REFERENCES agendamentos(id) ON DELETE CASCADE,
    tipo VARCHAR(20) NOT NULL,                      -- 'CONFIRMACAO','LEMBRETE','CANCELAMENTO'
    status VARCHAR(20) NOT NULL DEFAULT 'PENDENTE', -- 'PENDENTE','ENVIADA','FALHA'
    tentativas INTEGER NOT NULL DEFAULT 0,
    ultimo_erro VARCHAR(255),
    agendada_para TIMESTAMP WITH TIME ZONE NOT NULL,
    enviada_em TIMESTAMP WITH TIME ZONE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    CHECK (tipo IN ('CONFIRMACAO','LEMBRETE','CANCELAMENTO')),
    CHECK (status IN ('PENDENTE','ENVIADA','FALHA'))
);

-- Equipe interna da AgendaFácil (backoffice, RF14)
CREATE TABLE usuarios_plataforma (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    nome VARCHAR(150) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    senha_hash VARCHAR(255) NOT NULL,
    papel VARCHAR(20) NOT NULL DEFAULT 'SUPORTE',   -- 'ADMIN','SUPORTE'
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    CHECK (papel IN ('ADMIN','SUPORTE'))
);
```

### 3.2 Relacionamentos
* **Estabelecimento (Contratante)** 1:N **Profissionais**, **Serviços**, **Horários de funcionamento**, **Agendamentos**; 1:1 **Assinatura**.
* **Assinatura** 1:N **Cobranças**.
* **Profissional** N:N **Serviço** (`profissionais_servicos`); 1:N **Horários de atendimento**.
* **Agendamento** referencia obrigatoriamente Estabelecimento, Profissional e Serviço; 1:N **Notificações WhatsApp**.
* **Conflito de horário:** não existe constraint de banco. A validação roda no Use Case dentro de `prisma.$transaction`, com o profissional bloqueado para escrita (`SELECT ... FOR UPDATE` via `$queryRaw` ou equivalente) para evitar corrida entre dois Clientes Finais.

### 3.3 Política de Planos (fonte única da verdade)
O domínio expõe um mapa imutável, usado por Use Cases e middlewares. Nunca repetir esses valores em controllers ou componentes.

```ts
export const POLITICA_PLANOS = {
  BASICO:       { preco: 49,  whatsapp: false, catalogoPublico: false },
  PROFISSIONAL: { preco: 79,  whatsapp: true,  catalogoPublico: false },
  PREMIUM:      { preco: 129, whatsapp: true,  catalogoPublico: true  },
} as const;
```

---

## 4. Contratos de API REST (v1)

**Convenções**
* Prefixo `/api/v1`. JSON em `camelCase`. Datas em ISO 8601 UTC.
* Erro padrão: `{ "error": "CODIGO_MAIUSCULO", "message": "texto legível" }`.
* Erro de validação Zod: `422` com `{ "error": "VALIDATION_ERROR", "message": "...", "details": [{ "campo": "...", "mensagem": "..." }] }`.
* Três superfícies: **Pública** (Cliente Final, sem auth), **Contratante** (`Authorization: Bearer <JWT>` com claim `estabelecimentoId`), **Plataforma** (JWT de `usuarios_plataforma`).

### 4.A Superfície Pública (Cliente Final)

#### 4.A.1 Dados da página pública
* `GET /api/v1/estabelecimentos/{slug}`
* `200 OK`:
```json
{
  "nome": "Barbearia Souza & Cia",
  "slug": "barbearia-souza",
  "catalogoPublico": true,
  "servicos": [
    { "id": "c9bf9e57-1685-4c89-bafb-ff5af830be8a", "nome": "Corte masculino", "preco": 40.00, "duracaoMinutos": 30, "fotoUrl": "https://..." }
  ]
}
```
* Se `catalogoPublico = false` (ou plano ≠ `PREMIUM`), `preco` e `fotoUrl` **não são enviados**; só `id`, `nome` e `duracaoMinutos` (UC010.FS001).
* `404 Not Found`: slug inexistente ou Contratante `CANCELADO`.
* `403 Forbidden` (`ASSINATURA_INATIVA`): Contratante `SUSPENSO`.

#### 4.A.2 Listar Horários Disponíveis
* `GET /api/v1/estabelecimentos/{slug}/horarios?profissionalId=<uuid>&servicoId=<uuid>&data=YYYY-MM-DD`
* `200 OK`:
```json
{ "data": "2026-10-15", "horarios": ["09:00", "09:30", "10:00", "14:00", "14:30"] }
```
* O cálculo considera: horário de funcionamento do Contratante ∩ horário de atendimento do profissional, `duracao_minutos` do serviço, agendamentos `CONFIRMADO` existentes e descarta horários já passados.
* `404`: profissional/serviço não encontrado ou não pertence ao Contratante do `slug`.
* `403`: `ASSINATURA_INATIVA`.

#### 4.A.3 Criar Agendamento
* `POST /api/v1/agendamentos`
* Request:
```json
{
  "estabelecimentoId": "a0eebc99-9c0b-4ef8-bb6d-6bb9bd380a11",
  "profissionalId": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
  "servicoId": "c9bf9e57-1685-4c89-bafb-ff5af830be8a",
  "nomeCliente": "Marina Costa",
  "whatsappCliente": "+5582999990000",
  "dataHora": "2026-10-15T14:00:00Z"
}
```
* `201 Created`:
```json
{
  "id": "d3b07384-d113-4a4a-9a2e-1d5c9c9e3d1a",
  "codigo": "AF7K92XQ",
  "profissionalId": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
  "servicoId": "c9bf9e57-1685-4c89-bafb-ff5af830be8a",
  "nomeCliente": "Marina Costa",
  "whatsappCliente": "+5582999990000",
  "dataHora": "2026-10-15T14:00:00Z",
  "status": "CONFIRMADO",
  "createdAt": "2026-09-18T20:00:00Z"
}
```
* `403`: `ASSINATURA_INATIVA`.
* `404`: profissional ou serviço não encontrado (`"Profissional não encontrado"`).
* `409`: `HORARIO_INDISPONIVEL`.
* `422`: validação (RN-01, RN-02, RN-05).
* Efeito colateral: se o plano permite WhatsApp, cria `notificacoes_whatsapp` de `CONFIRMACAO` (imediata) e `LEMBRETE` (agendada). A resposta `201` **não espera** o envio.

#### 4.A.4 Consultar Agendamento (tela de cancelamento)
* `GET /api/v1/agendamentos/{codigo}`
* `200 OK`: dados do agendamento + nome do serviço, do profissional e do Contratante.
* `404`: código inexistente.

#### 4.A.5 Atualizar Status (cancelamento)
* `PATCH /api/v1/agendamentos/{codigo}/status`
* Request: `{ "status": "CANCELADO" }`
* Autorização: o `codigo` basta para o Cliente Final; se houver JWT de Contratante, o agendamento precisa pertencer ao tenant dele.
* `200 OK`: objeto completo do agendamento.
* `400`: `INVALID_STATUS_TRANSITION`.
* `404`: agendamento não encontrado (ou de outro tenant).
* Efeito colateral: libera o horário e, se o plano permite WhatsApp, cria notificação `CANCELAMENTO`.

### 4.B Superfície do Contratante

| Método e rota | Descrição | Auth |
|---|---|---|
| `POST /estabelecimentos` | Cadastro self-service (UC001). | Pública |
| `POST /auth/login` | Login do Contratante (UC005). | Pública |
| `GET /estabelecimentos/me` | Dados do próprio Contratante. | JWT |
| `PATCH /estabelecimentos/me` | Altera nome, contato, fuso, intervalo da grade e `catalogoPublico` (UC002). | JWT |
| `PUT /estabelecimentos/me/horarios-funcionamento` | Substitui os horários de funcionamento. | JWT |
| `POST/GET/PATCH/DELETE /servicos` | CRUD de serviços (UC004). | JWT |
| `POST/GET/PATCH/DELETE /profissionais` | CRUD de profissionais, com `servicoIds` e `horariosAtendimento` (UC003). | JWT |
| `GET /agenda?inicio=&fim=&profissionalId=` | Agenda do período (UC009). | JWT |
| `GET /agenda/metricas?data=` | Contadores do dia: total, confirmados, cancelados. | JWT |
| `GET /assinatura` | Plano, status, próxima cobrança, recursos liberados. | JWT |
| `POST /assinatura/mudar-plano` | Upgrade imediato ou downgrade agendado (RN-14). | JWT |
| `POST /assinatura/cancelar` | Cancela a assinatura ao fim do ciclo. | JWT |
| `GET /cobrancas` | Histórico de cobranças. | JWT |

#### 4.B.1 Cadastro do Contratante
* `POST /api/v1/estabelecimentos`
```json
{
  "nome": "Barbearia Souza & Cia",
  "slug": "barbearia-souza",
  "cnpjCpf": "12.345.678/0001-90",
  "email": "rafael@barbeariasouza.com",
  "telefone": "+5582988887777",
  "senha": "********",
  "plano": "PROFISSIONAL"
}
```
* `201 Created`: `{ "id": "...", "slug": "barbearia-souza", "plano": "PROFISSIONAL", "status": "ATIVO", "assinatura": { "status": "ATIVA", "valorMensal": 79.00, "proximaCobranca": "2026-11-23" } }`
* `409`: `ESTABELECIMENTO_JA_CADASTRADO` (CNPJ/CPF, e-mail ou slug já em uso).
* `422`: campos obrigatórios ausentes ou CNPJ/CPF inválido.

#### 4.B.2 Login
* `POST /api/v1/auth/login` com `{ "email": "...", "senha": "..." }`
* `200 OK`: `{ "token": "<jwt>", "expiraEm": "2026-10-23T22:00:00Z" }`
* `401`: `CREDENCIAIS_INVALIDAS` (mesma mensagem para e-mail inexistente e senha errada).

#### 4.B.3 Agenda
* `GET /api/v1/agenda?inicio=2026-10-15&fim=2026-10-15&profissionalId=<uuid>`
* `200 OK`:
```json
{
  "itens": [
    {
      "codigo": "AF7K92XQ",
      "nomeCliente": "Marina Costa",
      "servico": "Corte masculino",
      "profissional": "Rafael",
      "dataHora": "2026-10-15T14:00:00Z",
      "status": "CONFIRMADO",
      "whatsapp": { "status": "FALHA", "ultimoErro": "Número inválido" }
    }
  ]
}
```
* `whatsapp` é `null` em plano `BASICO`. Lista vazia é `200` com `itens: []` (UC009.FS001).

#### 4.B.4 Mudar de Plano
* `POST /api/v1/assinatura/mudar-plano` com `{ "plano": "PREMIUM" }`
* `200 OK`: `{ "plano": "PREMIUM", "efetivoEm": "2026-10-23T20:00:00Z", "valorMensal": 129.00, "tipo": "UPGRADE" }`
* Downgrade devolve `"tipo": "DOWNGRADE"` e `efetivoEm` no fim do ciclo atual.
* `409`: `MUDANCA_DE_PLANO_INVALIDA` (mesmo plano ou assinatura `CANCELADA`).

#### 4.B.5 Recurso bloqueado por plano
* Ex: `PATCH /estabelecimentos/me` com `{ "catalogoPublico": true }` em plano ≠ `PREMIUM`:
* `403`: `{ "error": "RECURSO_INDISPONIVEL_NO_PLANO", "message": "O catálogo público está disponível apenas no plano PREMIUM." }`

### 4.C Superfície da Plataforma (backoffice AgendaFácil)

| Método e rota | Descrição | Papel |
|---|---|---|
| `POST /plataforma/auth/login` | Login da equipe interna. | Pública |
| `GET /plataforma/estabelecimentos?status=&plano=&busca=` | Lista Contratantes com plano e status da assinatura. | ADMIN, SUPORTE |
| `GET /plataforma/estabelecimentos/{id}` | Detalhe, cobranças e falhas de WhatsApp recentes. | ADMIN, SUPORTE |
| `PATCH /plataforma/estabelecimentos/{id}/status` | Suspende ou reativa (`ATIVO` / `SUSPENSO`). | ADMIN |
| `GET /plataforma/metricas` | Métricas do negócio. | ADMIN |

* `GET /plataforma/metricas` → `200 OK`:
```json
{
  "contratantesAtivos": 42,
  "contratantesPorPlano": { "BASICO": 10, "PROFISSIONAL": 25, "PREMIUM": 7 },
  "mrr": 3368.00,
  "assinaturasEmAtraso": 3,
  "notificacoesEnviadas30d": 5120,
  "notificacoesComFalha30d": 41
}
```
* Toda ação do backoffice sobre um Contratante é registrada (quem, quando, o quê) em log de auditoria.
* O backoffice **não** lê nome ou WhatsApp de Clientes Finais; só contadores agregados (RNF05).

---

## 5. Interface Gráfica WEB (React / Next.js, App Router)

Pré-requisito de inicialização: `npx prisma migrate deploy` (ou `dev` em ambiente local).

| Rota | Quem usa | Conteúdo |
|---|---|---|
| `/` | Visitante | Página comercial com os três planos, preços e botão "Começar". |
| `/cadastro` | Futuro Contratante | Formulário de cadastro + escolha de plano (UC001). |
| `/login` | Contratante | E-mail e senha (UC005). |
| `/agendar/[slug]` | Cliente Final | Fluxo de agendamento (abaixo). |
| `/cancelar/[codigo]` | Cliente Final | Dados do agendamento, botões "Cancelar agendamento" e "Voltar" (UC007). |
| `/painel` | Contratante | Agenda, métricas, filtros e modais (abaixo). |
| `/painel/configuracoes` | Contratante | Dados do negócio, slug, horários de funcionamento, toggle de catálogo público (desabilitado com aviso de upgrade fora do `PREMIUM`). |
| `/painel/assinatura` | Contratante | Plano atual, comparativo de planos, botões de upgrade/downgrade/cancelar, histórico de cobranças. |
| `/plataforma` | Equipe AgendaFácil | Lista de Contratantes, métricas, suspender/reativar. |

### 5.1 Página Pública `/agendar/[slug]`
* **Lista de serviços:** com nome, preço e foto apenas se catálogo público habilitado (`PREMIUM`); caso contrário, só os nomes.
* **Seleção de profissional:** apenas profissionais ativos e vinculados ao serviço escolhido.
* **Grade de horários:** botões com os horários livres em tempo real.
* **Formulário:** `nomeCliente` e `whatsappCliente`.
* **Confirmação:** mostra `codigo` e link de cancelamento. Se o horário foi tomado, mostra o aviso e recarrega a grade (UC006.FS001).
* **Contratante suspenso:** página de indisponibilidade neutra ("Agendamento temporariamente indisponível"), sem expor motivo de cobrança.

### 5.2 Painel `/painel`
* **Barra de métricas:** agendamentos do dia, confirmados, cancelados.
* **Filtro:** por data/período (dia, semana, mês) e por profissional.
* **Lista de agenda:** cartões com cliente, serviço, profissional, horário e status. Indicador de **falha de WhatsApp** (somente planos com WhatsApp).
* **Banners de assinatura:** `EM_ATRASO` mostra aviso com prazo; `SUSPENSA` coloca o painel em modo somente leitura com botão de regularização.
* **Modais:** cadastro de serviço (nome, preço, duração, foto), cadastro de profissional (nome, serviços, horários de atendimento), cancelamento de agendamento pelo Contratante.

### 5.3 Ações e Formulários
* Agendar: `POST /api/v1/agendamentos`
* Cancelar: `PATCH /api/v1/agendamentos/{codigo}/status`
* Cadastrar serviço: `POST /api/v1/servicos` (`nome`, `preco`, `duracaoMinutos`, `fotoUrl`; o `estabelecimentoId` vem do JWT, nunca do body)
* Cadastrar profissional: `POST /api/v1/profissionais` (`nome`, `servicoIds`, `horariosAtendimento`)

---

## 6. Regras de Negócio e Casos de Borda

### 6.1 Regras de Negócio (RN)

**Agendamento**
* **RN-01 (Dados do Cliente Final):** `nomeCliente` de 3 a 150 caracteres; `whatsappCliente` em E.164 (ex: `+5582999990000`).
* **RN-02 (Data/Hora Futura):** `dataHora > agora`.
* **RN-03 (Enums):** `status`: `CONFIRMADO`, `CANCELADO`. `plano`: `BASICO`, `PROFISSIONAL`, `PREMIUM`. Status do Contratante: `ATIVO`, `SUSPENSO`, `CANCELADO`. Status da assinatura: `ATIVA`, `EM_ATRASO`, `SUSPENSA`, `CANCELADA`.
* **RN-04 (Ciclo de vida do agendamento):** `CONFIRMADO` ➔ `CANCELADO` é permitido. `CANCELADO` **nunca** volta a `CONFIRMADO`.
* **RN-05 (Vínculo):** o serviço pertence ao mesmo Contratante do profissional e o profissional está vinculado ao serviço (`profissionais_servicos`). Profissional e serviço devem estar `ativo`.
* **RN-06 (Sem conflito):** dois agendamentos `CONFIRMADO` do mesmo profissional não podem se sobrepor, considerando `duracao_minutos`.

**Plataforma e Contratantes**
* **RN-07 (Recursos por plano):** segue `POLITICA_PLANOS` (seção 3.3). `BASICO` não envia WhatsApp e não exibe catálogo. `PROFISSIONAL` adiciona WhatsApp. `PREMIUM` adiciona catálogo público. `catalogoPublico = true` só é aceito em `PREMIUM`; ao sair do `PREMIUM`, é forçado a `false`.
* **RN-08 (Assinatura ativa):** só Contratante `ATIVO` com assinatura `ATIVA` ou `EM_ATRASO` (dentro da tolerância) aceita **novos** agendamentos e consultas de horários. `SUSPENSA` ou `CANCELADA` bloqueia a página pública e põe o painel em somente leitura. Agendamentos já confirmados são preservados e **continuam visíveis** ao Contratante.
* **RN-09 (Isolamento multi-tenant):** toda leitura e escrita é filtrada pelo `estabelecimento_id` do JWT ou do `slug`. Acesso a recurso de outro tenant responde `404`, nunca `403`, para não revelar existência.
* **RN-10 (Slug):** minúsculas, números e hífen, 3 a 60 caracteres, único na plataforma. Mudar o slug invalida o link público antigo, e o painel deve avisar.
* **RN-11 (Cadastro único):** `cnpjCpf` válido (dígitos verificadores) e único; `email` único.
* **RN-12 (Reenvio de WhatsApp):** falha no envio gera até 3 tentativas com intervalo de alguns minutos (`WHATSAPP_RETRY_MINUTOS`). Esgotadas, a notificação fica `FALHA` e é sinalizada no painel. O agendamento nunca é afetado.
* **RN-13 (Lembrete):** enviado `LEMBRETE_ANTECEDENCIA_HORAS` antes do horário (padrão 24h). Agendamento criado dentro dessa janela recebe só a confirmação. Agendamento `CANCELADO` tem o lembrete descartado.
* **RN-14 (Mudança de plano):** upgrade vale na hora, com cobrança proporcional ou na próxima fatura conforme o provedor. Downgrade vale no próximo ciclo (`plano_agendado`). Recursos perdidos no downgrade são desativados quando ele passa a valer.
* **RN-15 (Cobrança e inadimplência):** cobrança gerada em `proxima_cobranca`. Sem pagamento no vencimento ➔ `EM_ATRASO`. Passados `TOLERANCIA_DIAS_ATRASO` ➔ `SUSPENSA` e Contratante `SUSPENSO`. Pagamento reativa automaticamente.
* **RN-16 (Exclusão protegida):** profissional ou serviço com agendamentos `CONFIRMADO` futuros não pode ser excluído (`409`). Para exclusão com histórico, aplica-se desativação (`ativo = false`): some da página pública, permanece nos relatórios e agendas passadas.
* **RN-17 (Cancelamento de conta):** cancelar a assinatura mantém o serviço até o fim do ciclo pago. Depois, Contratante `CANCELADO`, página pública fora do ar e dados retidos conforme política da AgendaFácil antes da exclusão definitiva.

### 6.2 Casos de Borda (CB)

| ID | Situação | Resposta |
|---|---|---|
| CB-01 | Agendar com `profissionalId` inexistente | `404` — `"Profissional não encontrado"` |
| CB-02 | `dataHora` no passado | `422` — `"A data do agendamento não pode ser no passado."` |
| CB-03 | Cancelar agendamento já `CANCELADO` | `400` — `INVALID_STATUS_TRANSITION` |
| CB-04 | Horário acabou de ser ocupado | `409` — `HORARIO_INDISPONIVEL` / `"Horário não está mais disponível"` |
| CB-05 | Slug inexistente ou Contratante `CANCELADO` | `404` |
| CB-06 | Recurso fora do plano (ex: catálogo no `BASICO`) | `403` — `RECURSO_INDISPONIVEL_NO_PLANO` |
| CB-07 | Contratante com assinatura `SUSPENSA` recebe tentativa de agendamento | `403` — `ASSINATURA_INATIVA` |
| CB-08 | Cadastro com CNPJ/CPF, e-mail ou slug já usado | `409` — `ESTABELECIMENTO_JA_CADASTRADO` |
| CB-09 | WhatsApp fora do ar ao confirmar | `201` normal; notificação `FALHA` após tentativas; sinal no painel |
| CB-10 | Contratante A tenta acessar recurso do Contratante B | `404` |
| CB-11 | Login com e-mail ou senha errados | `401` — `CREDENCIAIS_INVALIDAS` |
| CB-12 | Profissional não realiza o serviço informado | `422` — `PROFISSIONAL_NAO_REALIZA_SERVICO` |
| CB-13 | Excluir profissional/serviço com agendamentos futuros | `409` — `POSSUI_AGENDAMENTOS_FUTUROS` |
| CB-14 | Duas requisições simultâneas para o mesmo horário | Uma `201`, outra `409` (transação com bloqueio) |
| CB-15 | Contratante em plano `BASICO` com `whatsappCliente` | Campo continua obrigatório (é o contato do cliente), mas nenhuma notificação é criada |

### 6.3 Fluxos Assíncronos

**WhatsApp (UC008)**
1. Agendamento criado ➔ Use Case grava `notificacoes_whatsapp` (`CONFIRMACAO` agora, `LEMBRETE` em `dataHora - 24h`), apenas se `POLITICA_PLANOS[plano].whatsapp`.
2. Job a cada minuto busca `PENDENTE` com `agendada_para <= agora`, envia pela API, marca `ENVIADA`.
3. Erro ou timeout ➔ incrementa `tentativas`, grava `ultimo_erro`, reagenda; ao passar de 3 ➔ `FALHA`.
4. Mensagem de confirmação contém: data, horário, serviço, profissional, `codigo` e link de cancelamento.

**Cobrança (RF12)**
1. Job diário gera `cobrancas` para assinaturas com `proxima_cobranca = hoje`.
2. Webhook ou consulta do provedor atualiza a cobrança para `PAGA` ou `FALHA`.
3. Job diário aplica RN-15 (`EM_ATRASO` ➔ `SUSPENSA`) e aplica downgrades agendados.
4. **Decisão em aberto:** qual gateway (cartão, boleto, Pix) será usado para cobrar os Contratantes. Implementar apenas a porta `ProvedorPagamento` e um *fake* para testes até a definição.

---

## 7. Variáveis de Ambiente

| Variável | Descrição |
|---|---|
| `DATABASE_URL` | Conexão PostgreSQL |
| `JWT_SECRET`, `JWT_EXPIRES_IN` | Autenticação |
| `WHATSAPP_API_TOKEN`, `WHATSAPP_PHONE_NUMBER_ID` | WhatsApp Business Cloud API |
| `WHATSAPP_RETRY_MINUTOS` | Intervalo de reenvio (padrão 5) |
| `LEMBRETE_ANTECEDENCIA_HORAS` | Antecedência do lembrete (padrão 24) |
| `TOLERANCIA_DIAS_ATRASO` | Dias de `EM_ATRASO` antes de suspender (padrão 5) |
| `PAGAMENTOS_PROVIDER`, `PAGAMENTOS_API_KEY` | Gateway de cobrança (`fake` em dev/teste) |
| `APP_PUBLIC_URL` | Base dos links de cancelamento |

---

## 8. Critérios de Aceite (Testes Automatizados com Jest + Supertest)

### Cenário 1: Sucesso na criação de agendamento
* **Dado** Contratante `ATIVO` com assinatura `ATIVA`, profissional e serviço vinculados
* **Quando** `POST /api/v1/agendamentos` com horário livre
* **Então** `201 Created`, corpo com `codigo` gerado e `status` `"CONFIRMADO"`.

### Cenário 2: Horário já ocupado
* **Dado** agendamento `CONFIRMADO` do profissional `X` às `14:00`
* **Quando** segundo `POST` com mesmo `profissionalId` e `dataHora`
* **Então** `409` e corpo `{ "error": "HORARIO_INDISPONIVEL", "message": "Horário não está mais disponível" }`.

### Cenário 3: Transição inválida de status
* **Dado** agendamento `CANCELADO`
* **Quando** `PATCH /api/v1/agendamentos/{codigo}/status` com `{"status": "CONFIRMADO"}`
* **Então** `400` e `{ "error": "INVALID_STATUS_TRANSITION", "message": "Agendamentos cancelados não podem ser reativados." }`.

### Cenário 4: Plano `BASICO` não dispara WhatsApp
* **Dado** Contratante no plano `BASICO`
* **Quando** um agendamento é criado com sucesso
* **Então** `201` e **nenhuma** linha em `notificacoes_whatsapp` é criada.

### Cenário 5: Plano `PROFISSIONAL` agenda confirmação e lembrete
* **Dado** Contratante `PROFISSIONAL` e agendamento para daqui a 3 dias
* **Quando** o agendamento é criado
* **Então** existem 2 notificações `PENDENTE`: `CONFIRMACAO` (agora) e `LEMBRETE` (24h antes do horário).

### Cenário 6: Falha de WhatsApp não afeta o agendamento
* **Dado** cliente WhatsApp configurado para falhar sempre
* **Quando** o job de envio roda 3 vezes
* **Então** a notificação fica `FALHA` com `tentativas = 3`, o agendamento segue `CONFIRMADO` e `GET /agenda` retorna `whatsapp.status = "FALHA"`.

### Cenário 7: Catálogo bloqueado fora do `PREMIUM`
* **Dado** Contratante `PROFISSIONAL`
* **Quando** `PATCH /api/v1/estabelecimentos/me` com `{"catalogoPublico": true}`
* **Então** `403` com `RECURSO_INDISPONIVEL_NO_PLANO`.

### Cenário 8: Página pública esconde preços sem catálogo
* **Dado** Contratante `BASICO` com serviços de preço definido
* **Quando** `GET /api/v1/estabelecimentos/{slug}`
* **Então** resposta lista serviços **sem** `preco` e **sem** `fotoUrl`.

### Cenário 9: Assinatura suspensa bloqueia novos agendamentos
* **Dado** Contratante com assinatura `SUSPENSA`
* **Quando** `POST /api/v1/agendamentos`
* **Então** `403` com `ASSINATURA_INATIVA`, e agendamentos já existentes continuam acessíveis em `GET /agenda`.

### Cenário 10: Inadimplência progressiva
* **Dado** cobrança `PENDENTE` vencida
* **Quando** o job de cobrança roda no dia seguinte ao vencimento, e de novo após `TOLERANCIA_DIAS_ATRASO`
* **Então** assinatura vai a `EM_ATRASO` e depois a `SUSPENSA`; ao registrar pagamento volta a `ATIVA` e o Contratante a `ATIVO`.

### Cenário 11: Isolamento entre Contratantes
* **Dado** Contratante A e Contratante B autenticados
* **Quando** A chama `GET /api/v1/profissionais/{id-do-profissional-de-B}`
* **Então** `404`.

### Cenário 12: Cadastro duplicado
* **Dado** Contratante já cadastrado com um CNPJ
* **Quando** `POST /api/v1/estabelecimentos` com o mesmo CNPJ
* **Então** `409` com `ESTABELECIMENTO_JA_CADASTRADO`.

### Cenário 13: Corrida pelo mesmo horário
* **Dado** dois Clientes Finais
* **Quando** ambos enviam `POST` ao mesmo tempo para o mesmo profissional e horário
* **Então** exatamente um recebe `201` e o outro `409`.

### Cenário 14: Downgrade agendado
* **Dado** Contratante `PREMIUM` com `catalogoPublico = true`
* **Quando** solicita downgrade para `PROFISSIONAL`
* **Então** nada muda até o fim do ciclo; ao aplicar, `plano = PROFISSIONAL` e `catalogoPublico = false`.

---

## 9. Rastreabilidade

| Requisito | Seções |
|---|---|
| RF01, RF04 | 4.B.1, 4.B.2, 5 (`/cadastro`, `/login`), RN-10, RN-11, CB-08, CB-11 |
| RF02, RF03, RF13 | 4.B, 5.2, RN-05, RN-16, CB-13 |
| RF05, RF06 | 4.A.2, 4.A.3, RN-01, RN-02, RN-05, RN-06, CB-01 a CB-04, CB-14 |
| RF07 | 6.3, RN-07, RN-12, RN-13, CB-09, CB-15 |
| RF08 | 4.A.4, 4.A.5, RN-04, CB-03 |
| RF09 | 4.B.3, 5.2 |
| RF10 | 4.A.1, RN-07, CB-06 |
| RF11, RF12 | 4.B.4, 6.3, RN-14, RN-15, RN-17, CB-07 |
| RF14 | 4.C, 5 (`/plataforma`) |
