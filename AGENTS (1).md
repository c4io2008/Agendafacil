# AGENTS.md

## 1. Visão Geral do Projeto

**AgendaFácil** é uma plataforma **SaaS multi-tenant** em **Node.js/Express + React/Next.js**. A **AgendaFácil** (a empresa) vende assinaturas a **Contratantes** (barbearias, salões, clínicas, personal trainers etc.). Cada Contratante recebe um link público onde seus **Clientes Finais** agendam sozinhos, com disponibilidade em tempo real e, conforme o plano, lembretes via WhatsApp.

Os três papéis que você vai encontrar no código:

| Papel | Termo no código | Superfície |
|---|---|---|
| AgendaFácil (equipe interna) | `UsuarioPlataforma` | Backoffice `/plataforma`, rotas `/api/v1/plataforma/*` |
| Contratante | `Estabelecimento` | Painel `/painel`, rotas autenticadas por JWT |
| Cliente Final | sem conta (identificado por `codigo` do agendamento) | Página `/agendar/[slug]`, rotas públicas |

A especificação completa (DDL, contratos de API, regras, critérios de aceite) está em `SPECIFICATION.md`. **Ela é a fonte da verdade.** Em caso de divergência, a especificação vence.

---

## 2. Stack Tecnológica

- **Linguagem:** TypeScript (Node.js 20+)
- **Backend:** Express.js (API REST) + camada de Use Cases/Services
- **Frontend:** React / Next.js (App Router)
- **Banco:** PostgreSQL 16+ via Prisma ORM
- **Validação:** Zod
- **Testes:** Jest + Supertest (backend), React Testing Library (frontend)
- **Containerização:** Docker e Docker Compose
- **Integrações:** WhatsApp Business Cloud API; gateway de pagamento atrás da porta `ProvedorPagamento` (provedor ainda não definido, usar `fake` em dev/teste)
- **Jobs:** `node-cron` no processo do backend (lembretes, reenvio, cobrança)

---

## 3. Arquitetura e Estrutura de Camadas

1. `Domain`: entidades puras, objetos de valor (`HorarioDisponivel`, `Plano`), enums, `POLITICA_PLANOS` e exceções de domínio.
2. `Use Cases / Services`: regras de negócio e orquestração (`CriarAgendamentoUseCase`, `CalcularHorariosDisponiveisUseCase`, `CancelarAgendamentoUseCase`, `CadastrarEstabelecimentoUseCase`, `MudarPlanoUseCase`, `ProcessarCobrancasUseCase`, `EnviarNotificacoesUseCase`).
3. `Adapters / Controllers / Views`: rotas Express, middlewares, schemas Zod, páginas e componentes Next.js.
4. `Infrastructure`: repositórios Prisma, migrations, clientes HTTP (WhatsApp, gateway), jobs.

Estrutura sugerida:

```
backend/src/
  domain/           # entidades, enums, errors, politica-planos
  application/      # use-cases, ports (repositórios, WhatsApp, ProvedorPagamento)
  adapters/http/    # routes, controllers, schemas zod, middlewares
  infrastructure/   # prisma, whatsapp, pagamentos, jobs
frontend/app/
  page.tsx (comercial) · cadastro · login
  agendar/[slug] · cancelar/[codigo]
  painel · painel/configuracoes · painel/assinatura
  plataforma
```

**Regras arquiteturais (inegociáveis):**
- **Proibido** regra de negócio em controllers Express ou componentes React. Conflito de horário, transição de status, **liberação de recurso por plano** e **bloqueio por assinatura** vivem em Use Cases/Domain.
- Plano e recursos vêm **somente** de `POLITICA_PLANOS`. Nunca escreva `if (plano === 'PREMIUM')` espalhado; use `POLITICA_PLANOS[plano].catalogoPublico` ou um método de domínio.
- Middlewares permitidos nas rotas: autenticação, resolução de tenant, rate limit, erro central. Regra de plano fica no Use Case.

---

## 4. Multi-tenancy (regra mais crítica do projeto)

- **Toda** query de dados de Contratante é filtrada por `estabelecimentoId`. Esse valor vem do **JWT** (rotas do painel) ou do **slug** (rotas públicas), **nunca** do body de requisições autenticadas.
- Repositórios recebem `estabelecimentoId` como parâmetro obrigatório. Método de repositório sem esse parâmetro em entidade de tenant é bug.
- Recurso de outro tenant responde **404**, nunca 403 (RN-09).
- O backoffice só vê dados agregados. **Nunca** exponha `nomeCliente` ou `whatsappCliente` na superfície `/plataforma`.
- Todo PR que toca em repositório ou rota precisa de teste de isolamento entre dois tenants.

---

## 5. Regras de Negócio e Casos de Borda (resumo; detalhes em SPECIFICATION.md §6)

### 5.1 Agendamento
- **RN-01** `nomeCliente` 3–150 caracteres; `whatsappCliente` E.164.
- **RN-02** `dataHora` futura.
- **RN-03** Enums: `status` (`CONFIRMADO`, `CANCELADO`), `plano` (`BASICO`, `PROFISSIONAL`, `PREMIUM`).
- **RN-04** `CONFIRMADO` ➔ `CANCELADO` permitido; `CANCELADO` **não** volta a `CONFIRMADO`.
- **RN-05** Serviço e profissional do mesmo Contratante, profissional vinculado ao serviço, ambos ativos.
- **RN-06** Sem sobreposição de `CONFIRMADO` para o mesmo profissional, considerando `duracaoMinutos`. Validar dentro de `prisma.$transaction` com bloqueio do profissional.

### 5.2 Plataforma
- **RN-07** Recursos por plano: `BASICO` só agenda; `PROFISSIONAL` + WhatsApp; `PREMIUM` + catálogo público. `catalogoPublico = true` só em `PREMIUM`.
- **RN-08** Só assinatura `ATIVA` ou `EM_ATRASO` (na tolerância) aceita novos agendamentos. `SUSPENSA`/`CANCELADA` bloqueia a página pública e deixa o painel somente leitura, preservando agendamentos existentes.
- **RN-09** Isolamento multi-tenant (seção 4).
- **RN-10/11** Slug único e válido; CNPJ/CPF válido e único; e-mail único.
- **RN-12/13** WhatsApp: até 3 tentativas; lembrete 24h antes; falha nunca afeta o agendamento.
- **RN-14/15** Upgrade imediato, downgrade no próximo ciclo; `EM_ATRASO` ➔ `SUSPENSA` após a tolerância.
- **RN-16** Não excluir profissional/serviço com agendamentos futuros; desativar (`ativo = false`).

### 5.3 Casos de Borda principais
- **CB-01** Profissional inexistente ➔ `404` (`"Profissional não encontrado"`).
- **CB-02** Data no passado ➔ Zod retorna `422` (`"A data do agendamento não pode ser no passado."`).
- **CB-03** Cancelar já cancelado ➔ `400` `INVALID_STATUS_TRANSITION`.
- **CB-04** Horário ocupado ➔ `409` `HORARIO_INDISPONIVEL` (`"Horário não está mais disponível"`).
- **CB-06** Recurso fora do plano ➔ `403` `RECURSO_INDISPONIVEL_NO_PLANO`.
- **CB-07** Assinatura inativa ➔ `403` `ASSINATURA_INATIVA`.
- **CB-08** Cadastro duplicado ➔ `409` `ESTABELECIMENTO_JA_CADASTRADO`.

Erro padrão: `{ "error": "CODIGO", "message": "texto" }`. Zod: `422` com `VALIDATION_ERROR` e `details`.

---

## 6. Modelo de Dados (resumo)

- **Estabelecimento (Contratante):** `id`, `nome`, `slug` (único), `cnpjCpf` (único), `email` (único), `telefone`, `senhaHash`, `catalogoPublico`, `plano`, `status` (`ATIVO`|`SUSPENSO`|`CANCELADO`), `fusoHorario`, `intervaloGradeMinutos`, `createdAt`.
- **HorarioFuncionamento** (do Contratante) e **HorarioAtendimento** (do Profissional): dia da semana, início, fim.
- **Profissional:** `id`, `estabelecimentoId`, `nome`, `ativo`. **Serviço:** `id`, `estabelecimentoId`, `nome`, `preco`, `duracaoMinutos`, `fotoUrl?`, `ativo`. **ProfissionalServico:** N:N.
- **Agendamento:** `id`, `codigo` (único, 8 chars), `estabelecimentoId`, `profissionalId`, `servicoId`, `nomeCliente`, `whatsappCliente`, `dataHora` (UTC), `status`, `createdAt`, `updatedAt`. FKs de profissional/serviço são `RESTRICT`.
- **Assinatura:** `estabelecimentoId` (1:1), `plano`, `valorMensal`, `status` (`ATIVA`|`EM_ATRASO`|`SUSPENSA`|`CANCELADA`), `planoAgendado?`, `inicio`, `proximaCobranca`.
- **Cobranca:** `assinaturaId`, `valor`, `vencimento`, `status` (`PENDENTE`|`PAGA`|`FALHA`), `pagoEm?`, `referenciaExterna?`.
- **NotificacaoWhatsapp:** `agendamentoId`, `tipo` (`CONFIRMACAO`|`LEMBRETE`|`CANCELAMENTO`), `status` (`PENDENTE`|`ENVIADA`|`FALHA`), `tentativas`, `ultimoErro?`, `agendadaPara`, `enviadaEm?`.
- **UsuarioPlataforma:** `nome`, `email`, `senhaHash`, `papel` (`ADMIN`|`SUPORTE`).

DDL completo em `SPECIFICATION.md` §3.

---

## 7. Diretrizes de Implementação e Código

Antes de modificar ou criar qualquer módulo:
1. Consulte `SPECIFICATION.md` para contratos, nomes de campos e códigos de erro.
2. Valide com **Zod** todo `body` e `query` de entrada.
3. **Sem `catch (e) {}` silencioso.** Propague exceções de domínio (`HorarioIndisponivelError`, `TransicaoDeStatusInvalidaError`, `RecursoIndisponivelNoPlanoError`, `AssinaturaInativaError`, `EstabelecimentoJaCadastradoError`) para o middleware central de erro, que mapeia para HTTP.
4. Escreva testes (Jest + Supertest no backend, React Testing Library no frontend) para **todos** os cenários felizes, casos de borda e os 14 critérios de aceite da especificação §8.
5. Cálculo de horários: interseção entre funcionamento do Contratante e atendimento do profissional, menos agendamentos `CONFIRMADO`, usando `duracaoMinutos`, no fuso do Contratante, descartando horários passados.
6. Envio de WhatsApp **nunca** é síncrono no fluxo de agendamento. Grave `NotificacaoWhatsapp` e deixe o job enviar.
7. Só crie notificação se `POLITICA_PLANOS[plano].whatsapp` for verdadeiro.
8. Datas em UTC no banco e na API; converta para o fuso do Contratante apenas na montagem da grade e na UI.
9. Gere `codigo` com alfabeto sem ambiguidade (sem `0/O`, `1/I`), tratando colisão com retry.

**Segurança e privacidade:**
- Senhas com argon2 ou bcrypt; JWT com expiração curta; segredos só por variável de ambiente.
- Rate limit em `POST /agendamentos`, `PATCH .../status` e `GET .../horarios`.
- Não logar `nomeCliente`, `whatsappCliente`, senhas, tokens ou `JWT_SECRET`.
- Login com mesma mensagem para e-mail inexistente e senha errada.
- Página pública de Contratante suspenso não revela motivo de cobrança.

**Fora de escopo da v1 (não implementar):** pagamento Pix pelo Cliente Final, lista de espera, Google Agenda, relatórios financeiros, fidelidade, IA de horários.

---

## 8. Variáveis de Ambiente

`DATABASE_URL`, `JWT_SECRET`, `JWT_EXPIRES_IN`, `WHATSAPP_API_TOKEN`, `WHATSAPP_PHONE_NUMBER_ID`, `WHATSAPP_RETRY_MINUTOS` (5), `LEMBRETE_ANTECEDENCIA_HORAS` (24), `TOLERANCIA_DIAS_ATRASO` (5), `PAGAMENTOS_PROVIDER` (`fake`), `PAGAMENTOS_API_KEY`, `APP_PUBLIC_URL`.

---

## 9. Comandos Recomendados

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
- Suíte de testes do backend:
  ```bash
  npm run test
  ```

### Frontend (Next.js)
- Desenvolvimento:
  ```bash
  npm run dev
  ```
- Build de produção:
  ```bash
  npm run build
  ```
- Testes de componentes:
  ```bash
  npm run test:frontend
  ```

### Docker
- Subir banco e aplicação:
  ```bash
  docker compose up -d
  ```

---

## 10. Definição de Pronto

Uma entrega só está pronta quando:
- [ ] Contrato e códigos de erro batem com `SPECIFICATION.md`.
- [ ] Zod cobre todas as entradas.
- [ ] Nenhuma regra de negócio em controller ou componente.
- [ ] Toda query de tenant filtra por `estabelecimentoId` e há teste de isolamento.
- [ ] Regras de plano e assinatura passam por `POLITICA_PLANOS` e Use Cases.
- [ ] Testes de felizes, bordas e aceite passam (`npm run test`).
- [ ] Nenhum dado pessoal de Cliente Final em log.
