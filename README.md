<div align="center">

# .GRAMO — Ponto Eletrônico Corporativo (REP-P)

**SaaS multiempresa de controle de jornada em conformidade com a Portaria MTP 671/2021**

![TypeScript](https://img.shields.io/badge/TypeScript-3178C6?logo=typescript&logoColor=white)
![NestJS](https://img.shields.io/badge/NestJS-E0234E?logo=nestjs&logoColor=white)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL_+_RLS-4169E1?logo=postgresql&logoColor=white)
![Prisma](https://img.shields.io/badge/Prisma-2D3748?logo=prisma&logoColor=white)
![React](https://img.shields.io/badge/React_PWA-20232A?logo=react&logoColor=61DAFB)
![Docker](https://img.shields.io/badge/Docker-2496ED?logo=docker&logoColor=white)

</div>

---

Sistema de ponto eletrônico desenvolvido para a **GRAMO Engenharia**, pensado para equipes de campo
distribuídas em várias obras. O colaborador bate o ponto pelo celular com **reconhecimento facial e
geolocalização**, mesmo sem internet; o RH trata exceções, colhe assinaturas e gera os arquivos exigidos
pela fiscalização do trabalho. Software em processo de registro no **INPI**.

## Destaques

| | |
|---|---|
| **Conformidade legal** | Enquadrado como REP-P (Portaria 671/2021): NSR sequencial sem furos, geração de **AFD e AEJ**, comprovante **PAdES**, guarda de 5 anos |
| **Registro imutável** | Marcação é append-only com hash SHA-256; correção vira novo registro de ajuste com autor e motivo — nunca sobrescreve |
| **Multi-tenant no banco** | Isolamento por **Row Level Security** do PostgreSQL, API conectando com role sem superusuário, JWT amarrado à empresa |
| **Operador sem acesso a dados** | Portal da plataforma (planos, faturas, empresas) usa uma role Postgres **sem privilégio** nas tabelas operacionais |
| **Offline-first** | Fila local em IndexedDB e sincronização idempotente por UUID; hora do servidor quando online |
| **Assinatura digital** | Espelho de ponto assinado com SHA-256 do documento + manifesto + **assinatura Ed25519** do servidor, verificável depois |
| **LGPD** | Biometria como dado sensível: consentimento separado e **AES-256 em repouso** para fotos e documentos |
| **Nunca bloqueia o ponto** | Fora da área ou do horário, o registro é aceito e vai para a fila de exceções — bloquear poderia configurar fraude de jornada |

## Arquitetura

```
┌─ apps/web · React + Vite (PWA) ──────────────────────────────────────────────────────────┐
│ App do colaborador        Painel RH / Gestor          Quiosque     Portal /super         │
│ (face + GPS, offline)     (2FA, exceções, AFD/AEJ)                                       │
└──────────────────────────────────────────────────────────────────────────────────────────┘
                                            │ REST /api/v1  (subdomínio → empresa)
                                            ▼
┌─ apps/api · NestJS ──────────────────────────────────────────────────────────────────────┐
│ 16 módulos de domínio: pontos, assinaturas, compliance, auditoria, férias, plataforma... │
└──────────────────────────────────────────────────────────────────────────────────────────┘
        │ role repp_app  (RLS aplicado)              │ role repp_super  (sem dados operacionais)
        ▼                                            ▼
┌─ PostgreSQL · Prisma ────────────────────────────────────────────────────────────────────┐
│ 35 modelos · Row Level Security por empresa_id · registros de ponto append-only          │
└──────────────────────────────────────────────────────────────────────────────────────────┘
packages/shared — validadores, enums e contratos de sincronização usados pelo front e pelo back
```

## Módulos

**Colaborador:** bater ponto (face + REGAP + offline) · espelho de ponto · assinatura de documentos ·
documentos pessoais · férias e afastamentos · contestação de marcação · comunicados

**RH / Gestão:** dashboard · fila de exceções · funcionários e importação CSV · jornadas e feriados ·
banco de horas · assinaturas em lote · exportação AFD/AEJ · pacote de fiscalização · auditoria de acessos
e aprovações · integrações (folha / eSocial) · escopo por filial para gestores

**Plataforma:** empresas, planos, faturas e uso — com 2FA próprio e isolamento de dados

## Stack

| Camada | Tecnologia |
|---|---|
| Backend | Node.js · TypeScript · NestJS · REST versionada · OpenAPI/Swagger |
| Banco | PostgreSQL · Prisma · Row Level Security · roles separadas por função |
| Frontend | React · TypeScript · Vite · PWA · Dexie (IndexedDB) · reconhecimento facial no navegador |
| Segurança | JWT + refresh rotativo · Argon2id · 2FA TOTP obrigatório para admin · helmet · rate limit · Ed25519 · AES-256 |
| Qualidade | Vitest · testes e2e contra PostgreSQL real embarcado · ESLint · Prettier · Husky · CodeQL · Dependabot |
| Infra | Docker Compose · Caddy · scripts de backup |

## Testes

Suíte com testes de unidade e **e2e contra um PostgreSQL real** (`embedded-postgres`), cobrindo o que não
pode falhar: isolamento entre empresas via RLS, NSR sem furos sob concorrência, imutabilidade dos registros
e a fronteira entre o portal da plataforma e os dados dos clientes.

```bash
npm test                       # unidade (todos os workspaces)
npm run test:e2e -w @repp/api  # e2e: RLS + NSR + imutabilidade + boundary super
```

## Rodando localmente

Pré-requisitos: Node ≥ 20. Docker (para o Postgres) opcional mas recomendado.

```bash
# 1. Configurar ambiente
cp .env.example .env      # e ajuste os segredos

# 2. Instalar dependências (workspaces)
npm install

# 3. Subir banco + API (requer Docker)
docker compose up -d

# --- OU, sem Docker, apontando para um Postgres já existente: ---
npm run prisma:generate -w @repp/api
npm run db:setup -w @repp/api     # migrations + policies de RLS
npm run start:dev -w @repp/api    # API em http://localhost:3000/api/v1

# 4. Frontend
npm run dev -w @repp/web          # http://localhost:5173
```

### Modelo de duas roles no Postgres (por quê)

Para o RLS ser realmente aplicado, a API **não** pode conectar como superusuário
(superuser ignora RLS). Por isso há duas roles:

- `repp_owner` — dona do banco, roda migrations e o DDL de RLS (`directUrl` do Prisma).
- `repp_app` — role `NOSUPERUSER` que a API usa em runtime (`DATABASE_URL`).


> **Homologação antes do go-live:** validar os leiautes AFD/AEJ no verificador oficial gov.br e instalar um
> certificado **ICP-Brasil** (`.p12`). Em desenvolvimento, um certificado autoassinado é gerado automaticamente.

<details>
<summary><b>Referência da API</b></summary>

### Autenticação (Fase 1)

Toda requisição identifica a empresa pelo subdomínio; em dev use o header
`X-Tenant-Subdominio: <subdominio>`. Endpoints principais (prefixo `/api/v1`):

| Método | Rota                                                     | Descrição                                                 |
| ------ | -------------------------------------------------------- | --------------------------------------------------------- |
| POST   | `/auth/admin/login`                                      | E-mail + senha → devolve `desafioToken` (2FA obrigatório) |
| POST   | `/auth/admin/2fa/setup`                                  | 1º acesso: gera segredo TOTP (otpauth p/ QR)              |
| POST   | `/auth/admin/2fa/verify`                                 | Valida código TOTP → emite `accessToken`+`refreshToken`   |
| POST   | `/auth/admin/refresh` · `/logout`                        | Rotaciona / revoga sessão                                 |
| POST   | `/admins` · `/admins/:id/revogar`                        | Gestão de admins (RH Master)                              |
| POST   | `/convites`                                              | RH gera convite de 1º acesso (RH Master / Gestor)         |
| POST   | `/auth/funcionario/primeiro-acesso`                      | Convite + CPF + senha + aceite LGPD                       |
| POST   | `/auth/funcionario/login`                                | CPF + senha (bloqueia 5 falhas / 15 min)                  |
| POST   | `/auth/funcionario/recuperar-senha` · `/redefinir-senha` | Recuperação                                               |

Regras aplicadas: 2FA TOTP obrigatório para admin, senhas Argon2id, lockout
5/15min, refresh rotacionado (hash no banco), consentimento LGPD no 1º acesso,
e vínculo do JWT ao tenant (token de outra empresa é rejeitado). Documentação
interativa em `/api/docs` (Swagger).

### Ponto (Fase 2)

Endpoints do funcionário (autenticados, `/api/v1`):

| Método | Rota                                      | Descrição                                                          |
| ------ | ----------------------------------------- | ------------------------------------------------------------------ |
| POST   | `/pontos`                                 | Registra a marcação (online). **Nunca bloqueia**; hora do servidor |
| POST   | `/pontos/sync`                            | Sincroniza a fila offline (idempotente por UUID)                   |
| GET    | `/pontos/regap-status?latitude&longitude` | Status REGAP em tempo real (anel)                                  |
| GET    | `/pontos/hoje`                            | Espelho de ponto do dia                                            |

Endpoints do ADM 4 (RH Master / Gestor; Auditoria só leitura):

| Método         | Rota                                             | Descrição                                      |
| -------------- | ------------------------------------------------ | ---------------------------------------------- |
| GET            | `/admin/pontos/dashboard`                        | Indicadores do dia                             |
| GET            | `/admin/pontos/excecoes`                         | Fila de exceções pendentes                     |
| POST           | `/admin/pontos/excecoes/:id/decidir`             | Aprova/recusa (motivo obrigatório)             |
| POST           | `/admin/pontos/:id/ajuste`                       | Ajuste manual (cria registro, não sobrescreve) |
| GET            | `/admin/pontos/espelho?funcionarioId&inicio&fim` | Espelho por período                            |
| GET·POST·PATCH | `/admin/regaps`                                  | CRUD de áreas (REGAP)                          |

Garantias: botão que nunca bloqueia, **NSR atômico por estabelecimento**, **hash
SHA-256** por registro, ponto **imutável** (correção = ajuste; exceção em tabela
à parte), foto **cifrada em repouso** (AES-256), e **offline-first** (fila Dexie +
sync idempotente).

### Funcionário e documentos (Fase 3)

Tela 4 (funcionário): `GET/POST /documentos` (status + upload cifrado até 10MB).

ADM 2 (RH Master / Gestor; Auditoria só leitura):

| Método   | Rota                                     | Descrição                                              |
| -------- | ---------------------------------------- | ------------------------------------------------------ |
| GET·POST | `/admin/funcionarios`                    | Lista (busca/status) · cadastra (valida CPF/unicidade) |
| PATCH    | `/admin/funcionarios/:id`                | Atualiza (cargo, salário, jornada, filial, status)     |
| POST     | `/admin/funcionarios/:id/foto/aprovar`   | Aprova foto de referência                              |
| POST     | `/admin/funcionarios/:id/desligar`       | Soft delete (guarda 5 anos)                            |
| GET      | `/admin/funcionarios/:id/documentos`     | Documentos do funcionário                              |
| POST     | `/admin/documentos/:id/decidir`          | Aprova/rejeita documento (motivo)                      |
| POST     | `/admin/funcionarios/importar`           | Importação CSV tudo-ou-nada                            |
| GET      | `/admin/funcionarios/alertas/vencimento` | Documentos vencendo em 30 dias                         |

### Folha e assinatura virtual (Fase 4)

Tela 2 (funcionário): `GET /assinaturas`, `GET /assinaturas/:id/visualizar`,
`POST /assinaturas/:id/assinar` (re-autentica), `POST /assinaturas/:id/recusar`,
`GET /assinaturas/:id/comprovante`, `GET /assinaturas/chave-publica`.

ADM 3 (RH Master / Gestor / Financeiro): `POST /admin/assinaturas` (+ `/lote`),
`GET /admin/assinaturas` (fila), `GET /admin/assinaturas/:id` (detalhe),
`GET /admin/assinaturas/:id/comprovante`.

Cada assinatura gera registro **imutável** com **SHA-256 do documento** +
**SHA-256 do manifesto** + **assinatura Ed25519 do servidor** + timestamp +
IP/dispositivo. O comprovante reconstrói o manifesto e confere **assinatura** e
**integridade** (re-hash do arquivo). PAdES/ICP-Brasil embarcado no PDF é a Fase 5.

### Compliance Portaria 671 (Fase 5)

ADM 6 (RH Master / Auditoria): `POST /admin/exportacoes/afd` e `/aej` (gera do
período: hash SHA-256 + assinatura Ed25519 + registro imutável), `GET
/admin/exportacoes` (lista), `GET /admin/exportacoes/:id/download` (decifra +
reconfere integridade/assinatura), `GET /admin/comprovantes/ponto/:id`
(comprovante PDF **PAdES-B**), `POST /admin/relatorios/fiscalizacao` (pacote
AFD+AEJ+trilha).

> **Homologação obrigatória antes do go-live:** validar os leiautes AFD/AEJ no
> verificador oficial gov.br e instalar um certificado **ICP-Brasil** (`.p12`)
> em `ASSINATURA_P12_BASE64`/`ASSINATURA_P12_SENHA` (em dev, um cert autoassinado
> é gerado automaticamente — sem validade jurídica).

### Plataforma / Super Admin (Fase 6)

Portal `/super` (opera **acima** dos tenants; isento de subdomínio):
`POST /super/auth/login` · `/2fa/setup` · `/2fa/verify` · `/refresh`; e (com token
Super) `GET·POST /super/empresas`, `/super/empresas/:id/{suspender,ativar,uso}`,
`GET·POST /super/planos`, `GET·POST /super/faturas`, `/super/faturas/:id/pagar`.

> **Isolamento no banco:** o Super Admin usa a role `repp_super` (`SUPER_DATABASE_URL`),
> **sem** privilégio nas tabelas operacionais; uso vem de função `SECURITY DEFINER`.
> Bootstrap do 1º super admin via `SUPER_ADMIN_EMAIL`/`SUPER_ADMIN_SENHA`.
> Hardening: `helmet` + rate limiting global.

### Auditoria (ADM 6)

RH Master / Auditoria (somente leitura): `GET /admin/auditoria` (trilha de ações,
filtrável/paginada), `GET /admin/auditoria/acessos` (login/2FA/logout),
`GET /admin/auditoria/aprovacoes` (exceções decididas com justificativas).
Autorização administrativa do **Gestor de Filial** é restrita às suas filiais
(`AdminFilialAcesso`).

### Configurações (ADM 7)

`/admin/configuracoes/jornadas` (CRUD) e `/admin/configuracoes/feriados` (CRUD).
A jornada (horário, tolerância, dias de escala) é atribuída ao funcionário e
avaliada no **registro de ponto**: fora do horário/escala (e não sendo feriado)
→ `PENDENTE_HORARIO`, no **fuso da filial**. Nunca bloqueia — apenas sinaliza.

### Férias/afastamentos (ADM 10) e contestação (ADM 11)

Funcionário: `POST/GET /ferias` (solicitar/consultar), `POST/GET /contestacoes`
(contestar marcação própria). RH: `/admin/ferias` (listar/`:id/decidir`/`calendario`)
e `/admin/contestacoes` (listar/`:id/responder`). **Férias/afastamento aprovado
abona o dia** (não gera `PENDENTE_HORARIO`). Contestações e respostas ficam na
trilha de auditoria. Escopo por filial aplicado ao Gestor.

### Dashboard, banco de horas, comunicados, integrações (ADM 5/8/9)

- **ADM 5** `GET /admin/dashboard` (KPIs, alertas >48h, presença 7 dias).
- **Banco de horas** `GET /admin/pontos/banco-horas?funcionarioId&inicio&fim`
  (trabalhado/dia + saldo vs. carga da jornada, no fuso da filial).
- **ADM 8** `/admin/comunicados` (criar/histórico + taxa de leitura) e
  `/comunicados` (funcionário lê + marca lido); público-alvo todos/filial/cargo/func.
- **ADM 9** `/admin/integracoes/chaves` (gerar/revogar; token exibido 1x) e
  `/admin/integracoes/config/:tipo` (folha/eSocial). Transmissão externa é infra.


</details>

---

<sub>Desenvolvido por <a href="https://github.com/luizfernandoantonio345-webs">Luiz Fernando</a> para a GRAMO Engenharia.</sub>
