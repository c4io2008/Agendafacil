# AGENTS.md

## 1. Visão Geral do Projeto

**AgendaFácil** é uma aplicação FullStack desenvolvida em **Node.js/Express + React/Next.js** para automatizar o agendamento de horários de pequenos negócios (barbearias, salões, clínicas, personal trainers etc.), permitindo que o cliente final agende sozinho, com cálculo de disponibilidade em tempo real e lembretes automáticos via WhatsApp.

A especificação técnica completa, esquemas SQL, contratos de API e casos de teste encontram-se em `SPECIFICATION.md`.

---

## 2. Stack Tecnológica

- **Linguagem:** TypeScript (Node.js 20+)
- **Framework Backend:** Express.js (API REST) + Camada de Use Cases/Services
- **Frontend:** React / Next.js (App Router)
- **Banco de Dados / Persistência:** PostgreSQL 16+ via Prisma ORM
- **Validação de Dados & Schemas:** Zod
- **Testes Automatizados:** Jest + Supertest (backend), React Testing Library (frontend)
- **Containerização:** Docker e Docker Compose
- **Integração Externa:** API do WhatsApp (WhatsApp Business Cloud API)

---

## 3. Arquitetura e Estrutura de Camadas

A aplicação deve seguir uma separação clara de responsabilidades:

1. `Domain`: Entidades puras, objetos de valor (ex: `HorarioDisponivel`), Enums e exceções de domínio (ex: `HorarioIndisponivelError`, `TransicaoDeStatusInvalidaError`).
2. `Use Cases / Services`: Regras de negócio da aplicação (ex: `CriarAgendamentoUseCase`, `CalcularHorariosDisponiveisUseCase`, `CancelarAgendamentoUseCase`), validação de fluxo e orquestração com a integração do WhatsApp.
3. `Adapters / Controllers / Views`: Rotas e controllers do Express, schemas Zod de entrada/saída, componentes e páginas React/Next.js.
4. `Infrastructure`: Repositórios via Prisma Client, migrations, configuração de conexão com o PostgreSQL e cliente HTTP da API do WhatsApp.

**Regra Arquitetural:** É proibido acoplar regras de negócio diretamente nas rotas do Express (`controllers`) ou nos componentes React. A lógica de cálculo de disponibilidade, conflito de horário e transição de status deve residir no domínio/use cases.

---

## 4. Regras de Negócio e Casos de Borda (SPECIFICATION.md)

### 4.1 Regras de Negócio (RN)
- **RN-01 (Validação de Dados do Cliente):** `nomeCliente` entre 3 e 150 caracteres; `whatsappCliente` no formato E.164.
- **RN-02 (Data/Hora Futura):** A `dataHora` de um novo agendamento deve ser obrigatoriamente posterior ao momento da requisição.
- **RN-03 (Valores Permitidos de Enums):**
  - `status`: `CONFIRMADO`, `CANCELADO`
  - `plano`: `BASICO`, `PROFISSIONAL`, `PREMIUM`
- **RN-04 (Ciclo de Vida do Status):**
  - Transição permitida: `CONFIRMADO` ➔ `CANCELADO`
  - **Transição proibida:** Um agendamento `CANCELADO` **não pode** retornar para `CONFIRMADO`.
- **RN-05 (Vínculo Profissional/Serviço):** O serviço escolhido deve pertencer ao mesmo estabelecimento do profissional, e o profissional deve estar vinculado a esse serviço.
- **RN-06 (Sem Conflito de Horário):** Não pode existir dois agendamentos `CONFIRMADO` para o mesmo profissional em horários sobrepostos, considerando a duração do serviço.

### 4.2 Casos de Borda (CB)
- **CB-01 (Profissional Inexistente):** Criar agendamento com `profissionalId` inexistente ➔ Retornar `404 Not Found` (`"Profissional não encontrado"`).
- **CB-02 (Data no Passado):** Criar agendamento com data retroativa ➔ O Zod intercepta e retorna `422 Unprocessable Entity` (`"A data do agendamento não pode ser no passado."`).
- **CB-03 (Cancelamento Já Realizado):** Cancelar um agendamento já `CANCELADO` ➔ Retornar `400 Bad Request` com código `"INVALID_STATUS_TRANSITION"`.
- **CB-04 (Horário Ocupado):** Confirmar agendamento em horário que acabou de ser ocupado ➔ Retornar `409 Conflict` (`"Horário não está mais disponível"`).

---

## 5. Modelo de Dados

### 5.1 Entidades Principais
- **Estabelecimento:** `id` (UUID), `nome` (VARCHAR 150), `slug` (VARCHAR UNIQUE), `cnpjCpf` (VARCHAR UNIQUE), `senhaHash`, `catalogoPublico` (BOOLEAN), `plano` (`BASICO` | `PROFISSIONAL` | `PREMIUM`), `createdAt`.
- **Profissional:** `id` (UUID), `estabelecimentoId` (FK ON DELETE CASCADE), `nome` (VARCHAR 150), `ativo` (BOOLEAN), `createdAt`.
- **Serviço:** `id` (UUID), `estabelecimentoId` (FK ON DELETE CASCADE), `nome` (VARCHAR 100), `preco` (NUMERIC 8,2), `duracaoMinutos` (INTEGER), `fotoUrl` (opcional), `createdAt`.
- **ProfissionalServico:** tabela de associação N:N entre `Profissional` e `Serviço`.
- **Agendamento:** `id` (UUID), `codigo` (VARCHAR UNIQUE), `estabelecimentoId`/`profissionalId`/`servicoId` (FKs), `nomeCliente`, `whatsappCliente`, `dataHora` (TIMESTAMP WITH TIME ZONE), `status` (`CONFIRMADO` | `CANCELADO`), `createdAt`, `updatedAt`.

---

## 6. Diretrizes de Implementação e Código

Antes de modificar ou criar qualquer módulo:
1. Verifique `SPECIFICATION.md` para garantir conformidade de contratos e nomes de campos.
2. Implemente validações Zod para todos os payloads de entrada da API (`request body` e `query params`).
3. Não use `catch (e) {}` silencioso. Trate erros de forma explícita, propagando exceções de domínio (`HorarioIndisponivelError`, `TransicaoDeStatusInvalidaError`) para um middleware central de erro no Express.
4. Escreva testes automatizados (Jest + Supertest no backend, React Testing Library no frontend) cobrindo todos os cenários felizes e casos de borda especificados.
5. O cálculo de horários disponíveis deve considerar `duracaoMinutos` do serviço e os agendamentos `CONFIRMADO` já existentes para o profissional na data consultada.

---

## 7. Comandos Recomendados

### Backend (Node/Express) & Prisma
- Gerar o Prisma Client:
  ```bash
  npx prisma generate
  ```
- Criar e aplicar migrações:
  ```bash
  npx prisma migrate dev --name init
  ```
- Aplicar migrações em produção:
  ```bash
  npx prisma migrate deploy
  ```
- Executar suíte de testes do backend:
  ```bash
  npm run test
  ```

### Frontend (Next.js)
- Rodar em modo desenvolvimento:
  ```bash
  npm run dev
  ```
- Build de produção:
  ```bash
  npm run build
  ```
- Executar testes de componentes:
  ```bash
  npm run test:frontend
  ```

### Docker
- Subir banco de dados e aplicação:
  ```bash
  docker compose up -d
  ```
