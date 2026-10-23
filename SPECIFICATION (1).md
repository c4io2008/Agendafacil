# Especificação Técnica: Aplicação FullStack de Agendamento Online (AgendaFácil)

---

## 1. Visão Geral e Stack

### 1.1 Objetivo
O **AgendaFácil** é uma aplicação FullStack responsável por automatizar o agendamento de horários de pequenos negócios que trabalham com atendimento marcado (barbearias, salões, clínicas, personal trainers, etc.). O sistema permite que o estabelecimento cadastre profissionais e serviços, e que o cliente final agende sozinho — sem intervenção humana — através de um link público, com cálculo automático de disponibilidade e envio de confirmações e lembretes via WhatsApp.

### 1.2 Stack Tecnológica
* **Linguagem:** TypeScript (Node.js 20+)
* **Framework Backend:** Express.js, expondo uma API REST consumida pelo frontend
* **Persistência / ORM:** PostgreSQL 16+ via Prisma ORM
* **Frontend:** React / Next.js (App Router)
* **Validação de Dados:** Zod (nas fronteiras da API — request bodies e query params)
* **Testes Automatizados:** Jest + Supertest (backend), React Testing Library (frontend)
* **Containerização:** Docker e Docker Compose
* **Integração Externa:** API do WhatsApp (WhatsApp Business Cloud API) para confirmações e lembretes

### 1.3 Arquitetura e Restrições
* **Padrão Arquitetural:** Clean Architecture / Camadas, aplicada dentro do backend Express:
  1. `Domain`: Entidades puras, objetos de valor (ex: `HorarioDisponivel`) e exceções de domínio (ex: `HorarioIndisponivelError`, `TransicaoDeStatusInvalidaError`).
  2. `Use Cases / Services`: Regras de negócio da aplicação (ex: `CriarAgendamentoUseCase`, `CalcularHorariosDisponiveisUseCase`) e orquestração entre repositórios e integrações externas (WhatsApp).
  3. `Adapters / Controllers`: Rotas e controllers do Express, schemas Zod de entrada/saída, componentes e páginas React/Next.js.
  4. `Infrastructure`: Repositórios via Prisma Client, migrations, configuração de banco e do cliente da API do WhatsApp.
* **Restrição de Camadas:** É proibido colocar regra de negócio diretamente nas rotas do Express ou nos componentes React. Toda validação de fluxo (ex: conflito de horário, transição de status) deve residir na camada de `Use Cases`.

---

## 2. Entidades e Dados (DDL SQL)

### 2.1 Modelo de Dados

```sql
-- Tabela de Estabelecimentos
CREATE TABLE estabelecimentos (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    nome VARCHAR(150) NOT NULL,
    slug VARCHAR(160) UNIQUE NOT NULL,
    cnpj_cpf VARCHAR(18) UNIQUE NOT NULL,
    senha_hash VARCHAR(255) NOT NULL,
    catalogo_publico BOOLEAN NOT NULL DEFAULT FALSE,
    plano VARCHAR(20) NOT NULL DEFAULT 'BASICO', -- 'BASICO', 'PROFISSIONAL', 'PREMIUM'
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Tabela de Profissionais
CREATE TABLE profissionais (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    estabelecimento_id UUID NOT NULL REFERENCES estabelecimentos(id) ON DELETE CASCADE,
    nome VARCHAR(150) NOT NULL,
    ativo BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Tabela de Serviços
CREATE TABLE servicos (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    estabelecimento_id UUID NOT NULL REFERENCES estabelecimentos(id) ON DELETE CASCADE,
    nome VARCHAR(100) NOT NULL,
    preco NUMERIC(8,2) NOT NULL,
    duracao_minutos INTEGER NOT NULL,
    foto_url VARCHAR(255),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Tabela de associação N:N entre Profissionais e Serviços
CREATE TABLE profissionais_servicos (
    profissional_id UUID NOT NULL REFERENCES profissionais(id) ON DELETE CASCADE,
    servico_id UUID NOT NULL REFERENCES servicos(id) ON DELETE CASCADE,
    PRIMARY KEY (profissional_id, servico_id)
);

-- Tabela de Agendamentos
CREATE TABLE agendamentos (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    codigo VARCHAR(12) UNIQUE NOT NULL,
    estabelecimento_id UUID NOT NULL REFERENCES estabelecimentos(id) ON DELETE CASCADE,
    profissional_id UUID NOT NULL REFERENCES profissionais(id) ON DELETE CASCADE,
    servico_id UUID NOT NULL REFERENCES servicos(id) ON DELETE CASCADE,
    nome_cliente VARCHAR(150) NOT NULL,
    whatsapp_cliente VARCHAR(20) NOT NULL,
    data_hora TIMESTAMP WITH TIME ZONE NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'CONFIRMADO', -- 'CONFIRMADO', 'CANCELADO'
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_agendamentos_profissional_data ON agendamentos (profissional_id, data_hora);
```

### 2.2 Relacionamentos
* **Estabelecimento -> Profissional:** 1 Estabelecimento possui N Profissionais (1:N). Ao excluir o estabelecimento, os profissionais são excluídos em cascata (`CASCADE`).
* **Estabelecimento -> Serviço:** 1 Estabelecimento possui N Serviços (1:N), também em cascata.
* **Profissional <-> Serviço:** Relação N:N — um profissional pode realizar vários serviços, e um serviço pode ser realizado por vários profissionais (tabela `profissionais_servicos`).
* **Agendamento:** Referencia obrigatoriamente um Estabelecimento, um Profissional e um Serviço. Não pode existir mais de um agendamento com `status = 'CONFIRMADO'` para o mesmo `profissional_id` na mesma `data_hora` — restrição validada na camada de Use Case, dentro de uma transação Prisma (`prisma.$transaction`), e não como constraint de banco.

---

## 3. Interface Gráfica WEB (React / Next.js)

A aplicação disponibiliza uma interface visual completa em Next.js (App Router), consumindo a API REST do Express:

### 3.1 Página Pública de Agendamento (`GET /agendar/[slug]`)
* **Rota:** `/agendar/[slug]` (ex: `/agendar/barbearia-souza`)
* **Pré-requisito de inicialização:** Antes de acessar a rota, executar `npx prisma migrate deploy` (ou `dev` em ambiente local). A migration inicial cria as tabelas `estabelecimentos`, `profissionais`, `servicos`, `profissionais_servicos` e `agendamentos` consultadas por essa página.
* **Recursos visuais:**
  * **Lista de Serviços:** Cartões com nome, preço e foto (exibidos apenas se `catalogo_publico = true`).
  * **Seleção de Profissional:** Lista filtrada pelos profissionais aptos a realizar o serviço escolhido.
  * **Grade de Horários:** Botões clicáveis com os horários livres do profissional, calculados em tempo real a partir da `duracao_minutos` do serviço e dos agendamentos já confirmados.
  * **Formulário de Confirmação:** Campos `nome_cliente` e `whatsapp_cliente`.
  * **Tela de Confirmação:** Exibe o `codigo` do agendamento gerado e o link de cancelamento.

### 3.2 Painel Administrativo (`GET /painel`)
* **Rota:** `/painel` (autenticada via sessão/JWT do estabelecimento)
* **Recursos visuais:**
  * **Barra de Métricas:** Contadores de agendamentos do dia, confirmados e cancelados.
  * **Filtro por Data/Profissional:** Navegação rápida para consultar a agenda de um período específico.
  * **Lista de Agenda:** Cartões de agendamento com nome do cliente, serviço, horário e status, incluindo indicador de falha no envio da confirmação via WhatsApp.
  * **Modais Interativos:**
    * Modal de Cadastro de Serviço (nome, preço, duração, foto).
    * Modal de Cadastro de Profissional (nome, serviços vinculados).
    * Modal de Cancelamento de Agendamento pelo estabelecimento.

### 3.3 Ações e Formulários Web
* **Agendar Horário:** `POST /api/v1/agendamentos` (Campos: `estabelecimentoId`, `profissionalId`, `servicoId`, `nomeCliente`, `whatsappCliente`, `dataHora`).
* **Cancelar Agendamento:** `PATCH /api/v1/agendamentos/{codigo}/status` (Campo: `status = "CANCELADO"`).
* **Cadastrar Serviço:** `POST /api/v1/servicos` (Campos: `estabelecimentoId`, `nome`, `preco`, `duracaoMinutos`, `fotoUrl`).
* **Cadastrar Profissional:** `POST /api/v1/profissionais` (Campos: `estabelecimentoId`, `nome`, `servicoIds`).

---

## 4. Contratos de API REST (v1)

### 4.1 Criar Agendamento
* **Endpoint:** `POST /api/v1/agendamentos`
* **Descrição:** Cria um novo agendamento para um cliente, vinculado a um estabelecimento, profissional e serviço existentes.
* **Headers:** `Content-Type: application/json`
* **Request Body:**
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
* **Respostas:**
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
  * `404 Not Found`: Profissional ou serviço não encontrado.
  * `409 Conflict`: Horário não está mais disponível para o profissional.
  * `422 Unprocessable Entity`: Dados de entrada inválidos (ex: `whatsappCliente` ausente).

---

### 4.2 Listar Horários Disponíveis
* **Endpoint:** `GET /api/v1/estabelecimentos/{slug}/horarios`
* **Descrição:** Calcula e retorna os horários livres de um profissional em uma data específica, com base na duração do serviço escolhido.
* **Query Parameters:** `profissionalId` (UUID), `servicoId` (UUID), `data` (YYYY-MM-DD)
* **Respostas:**
  * `200 OK`:
```json
{
  "data": "2026-10-15",
  "horarios": ["09:00", "09:30", "10:00", "14:00", "14:30"]
}
```
  * `404 Not Found`: Profissional não encontrado ou não vinculado ao estabelecimento.

---

### 4.3 Atualizar Status do Agendamento
* **Endpoint:** `PATCH /api/v1/agendamentos/{codigo}/status`
* **Descrição:** Altera o status de um agendamento (usado para cancelamento, pelo cliente ou pelo estabelecimento).
* **Path Parameter:** `codigo` (string)
* **Request Body:**
```json
{
  "status": "CANCELADO"
}
```
* **Respostas:**
  * `200 OK`: Retorna o objeto completo do agendamento com o status atualizado.
  * `400 Bad Request`: Transição de status inválida.
  * `404 Not Found`: Agendamento não encontrado.

---

## 5. Regras de Negócio e Casos de Borda

### 5.1 Regras de Negócio (RN)
* **RN-01 (Validação de Dados do Cliente):** `nomeCliente` deve conter entre 3 e 150 caracteres; `whatsappCliente` deve seguir o formato E.164 (ex: `+5582999990000`).
* **RN-02 (Data/Hora Futura):** A `dataHora` de um novo agendamento deve obrigatoriamente ser posterior ao momento da requisição (`dataHora > agora`).
* **RN-03 (Valores Permitidos de Enums):**
  * `status`: Apenas `CONFIRMADO`, `CANCELADO`.
  * `plano`: Apenas `BASICO`, `PROFISSIONAL`, `PREMIUM`.
* **RN-04 (Ciclo de Vida do Status):**
  * Transição permitida: `CONFIRMADO` ➔ `CANCELADO`.
  * Transição proibida: Um agendamento `CANCELADO` não pode retornar para `CONFIRMADO`.
* **RN-05 (Vínculo Profissional/Serviço):** O `servicoId` informado deve pertencer ao mesmo `estabelecimentoId` do `profissionalId`, e o profissional deve estar vinculado a esse serviço (tabela `profissionais_servicos`).
* **RN-06 (Sem Conflito de Horário):** Não pode existir dois agendamentos com `status = CONFIRMADO` para o mesmo profissional em horários que se sobrepõem, considerando a `duracao_minutos` do serviço.

### 5.2 Casos de Borda (CB)
* **CB-01 (Profissional Inexistente):** Criar agendamento com `profissionalId` inexistente ➔ Retornar `404 Not Found` (`"Profissional não encontrado"`).
* **CB-02 (Data no Passado):** Criar agendamento com `dataHora` retroativa ➔ O Zod intercepta e retorna `422 Unprocessable Entity` (`"A data do agendamento não pode ser no passado."`).
* **CB-03 (Cancelamento Já Realizado):** Tentar cancelar um agendamento já `CANCELADO` ➔ Retornar `400 Bad Request` com código de erro de domínio `"INVALID_STATUS_TRANSITION"`.
* **CB-04 (Horário Ocupado):** Confirmar um agendamento em um horário que acabou de ser ocupado por outro cliente ➔ Retornar `409 Conflict` (`"Horário não está mais disponível"`).

---

## 6. Critérios de Aceite (Exemplos para Testes Automatizados)

Estes cenários devem orientar a geração de testes automatizados com `jest` e `supertest`.

### Cenário 1: Sucesso na Criação de Agendamento
* **Dado** que existem um estabelecimento, um profissional e um serviço cadastrados e vinculados entre si
* **Quando** for enviada uma requisição `POST` para `/api/v1/agendamentos` com um horário livre
* **Então** o código de status HTTP retornado deve ser `201 Created`
* **E** o corpo da resposta deve conter o campo `codigo` gerado automaticamente
* **E** o campo `status` deve ser inicializado como `"CONFIRMADO"`.

### Cenário 2: Falha por Horário Já Ocupado
* **Dado** que já existe um agendamento `CONFIRMADO` para o profissional `X` às `14:00`
* **Quando** for enviada uma segunda requisição `POST` para o mesmo `profissionalId` e a mesma `dataHora`
* **Então** o código de status HTTP retornado deve ser `409 Conflict`
* **E** o corpo da resposta deve conter:
  ```json
  {
    "error": "HORARIO_INDISPONIVEL",
    "message": "Horário não está mais disponível"
  }
  ```

### Cenário 3: Transição Inválida de Status
* **Dado** um agendamento salvo no banco com status `CANCELADO`
* **Quando** for enviada uma requisição `PATCH` para `/api/v1/agendamentos/{codigo}/status` com `{"status": "CONFIRMADO"}`
* **Então** o status HTTP deve ser `400 Bad Request`
* **E** o corpo da resposta deve conter:
  ```json
  {
    "error": "INVALID_STATUS_TRANSITION",
    "message": "Agendamentos cancelados não podem ser reativados."
  }
  ```
