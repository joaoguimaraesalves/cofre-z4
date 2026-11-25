# Divisão de tarefas – Cofre Seguro de Dados (Z4 Motors)

| | Dev | Foco |
|---|---|---|
| **Dev 1** | João Victor Guimarães Alves (@joaoguimaraesalves) | Núcleo de segurança e sistema interno da Z4 |
| **Dev 2** | João Gabriel R. Rocha (@joaogabrieldev) | Infraestrutura, portal do cliente e LGPD |

Regras gerais:
- Uma branch por tarefa (`feat/<nome>`), PR para a `main`, **o outro revisa** antes do merge.
- Antes de cada tarefa no Claude Code: "Leia o CLAUDE.md e os PDFs em docs/".
- Depois de cada tarefa: "Explique o que você fez e por quê" (para os dois saberem defender na apresentação).
- Só o Dev 1 altera `backend/app/core/`; só o Dev 2 altera arquivos de deploy/CI. Precisou mexer na área do outro? Combina antes.
- Migrações Alembic: sempre `git pull` antes de criar uma nova.

---

## Fase 0 – Fundação (Dev 1 primeiro, Dev 2 em paralelo)

### Dev 1
- [ ] **T1 – Estrutura do projeto** (`feat/estrutura`)
  > Crie a estrutura conforme o CLAUDE.md: backend FastAPI com /api/health, frontend React + Vite + TypeScript + Tailwind com a paleta do PDF, docker-compose para dev (api, web, mailpit), Dockerfile multi-stage que compila o front e serve junto com a API, Alembic configurado.
- [ ] **T2 – Segurança central** (`feat/core-security`)
  > Implemente app/core/security: AES-256-GCM para campos, blind index HMAC-SHA256 para CPF, Argon2id para senhas (parâmetros configuráveis por env), TOTP com pyotp. Testes unitários para tudo.

### Dev 2
- [ ] **T3 – Banco no Neon**
  - Criar projeto `cofre` no Neon, branch `dev` (desenvolvimento) e `main` (apresentação).
  - Passar ao Dev 1 a URL *pooled* do branch `dev` **por mensagem privada** (nunca no Git).
- [ ] **T4 – Hospedagem** (depois de decidida: Render / Koyeb / Fly.io / Heroku)
  - Criar a conta e conectar ao repositório GitHub.
- [ ] **T5 – E-mail**
  - Criar conta no Resend e gerar a API key.
- [ ] **T6 – CI** (`feat/ci`), após a T1
  > Crie um workflow do GitHub Actions que roda os testes do backend e o build do frontend a cada PR.

---

## Fase 1 – Funcionalidades

### Dev 1 – Sistema interno
- [ ] **T7 – Login, 2FA e perfis** (`feat/auth`)
  > Usuários com perfis seller/analyst/manager, login com Argon2id + TOTP obrigatório, sessão JWT em cookie HttpOnly/Secure/SameSite=Strict (15 min), bloqueio após 5 falhas, rate limiting. Telas de login e cadastro do 2FA (QR code). Testes de acesso negado.
- [ ] **T8 – Auditoria** (`feat/auditoria`)
  > Tabela append-only com hash encadeado, função de registro usada por todos os módulos, GET /api/audit/verify apontando o primeiro registro adulterado. Tela de auditoria para o gerente, igual ao PDF.
  - Entregar cedo: o Dev 2 usa a função de registro no portal.
- [ ] **T9 – Clientes** (`feat/clientes`)
  > CRUD de clientes com CPF, renda e dados bancários criptografados, busca por CPF via blind index, mascaramento por padrão, endpoint de revelar (com auditoria), autorização por perfil no servidor. Testes do 403 por perfil.
- [ ] **T10 – Telas internas** (`feat/telas-internas`)
  > Painel de clientes com cards de indicadores, ficha do cliente com visão diferente por perfil (vendedor x analista) e gestão de usuários, seguindo o PDF.

### Dev 2 – Cliente externo e LGPD
- [ ] **T11 – Documentos** (`feat/documentos`)
  > Upload de PDF/JPG/PNG até 10 MB, validando magic bytes, criptografado em AES-256-GCM e salvo em coluna bytea. Download só autenticado e autorizado (analyst/manager). Auditoria em upload e download.
- [ ] **T12 – Portal do cliente** (`feat/portal`)
  > Vendedor gera link (token de 32 bytes, só o hash no banco, 24h, uso único). Cliente recebe código de 6 dígitos por e-mail (Mailpit em dev, Resend em produção), 3 tentativas, aceita o termo LGPD (gravar data, hora, IP e versão), preenche os dados e envia os documentos. Link invalidado após o envio. As 4 telas mobile do PDF. O portal nunca exibe dados já armazenados.
- [ ] **T13 – LGPD** (`feat/lgpd`)
  > Prazo de retenção configurável, job que anonimiza clientes vencidos (com auditoria), endpoint protegido por CRON_SECRET, botão "rodar agora" para o gerente e GitHub Actions agendado 1x por dia.
- [ ] **T14 – Seed** (`feat/seed`)
  > Seed com Faker pt_BR: 3 usuários (um por perfil, com 2FA), ~30 clientes em vários status, alguns com prazo vencido para demonstrar a anonimização. Só dados fictícios.
- [ ] **T15 – Deploy** (`feat/deploy`)
  > Configure o deploy na plataforma escolhida usando o Dockerfile, migrações automáticas no start e variáveis de ambiente documentadas no .env.example. Endpoint /api/health para "acordar" o serviço.

---

## Fase 2 – Fechamento (os dois juntos)

- [ ] **T16 – Revisão de segurança cruzada:** cada um revisa o código do outro com o Claude Code
  > Revise o projeto inteiro procurando falhas contra o OWASP Top 10 e as regras do CLAUDE.md. Liste os problemas por gravidade antes de corrigir.
- [ ] **T17 – OWASP ZAP:** varredura no sistema publicado, corrigir os achados e guardar o relatório (vai para a apresentação).
- [ ] **T18 – Ensaio da demo:** rodar os 7 passos do roteiro do CLAUDE.md no ambiente publicado e gravar um vídeo de backup.
- [ ] **T19 – Apresentação:** slides (problema → solução → arquitetura → demo → antes x depois → protótipo x produção).
  - Dev 1 apresenta: criptografia, login/2FA, perfis, auditoria.
  - Dev 2 apresenta: arquitetura/infra, portal do cliente, LGPD, resultado do ZAP.

---

## Dependências (o que destrava o quê)

```
T1 Estrutura ──► T2 Core security ──► T7 Auth ──► T9 Clientes ──► T10 Telas internas
      │                │                │
      │                │                └──► T8 Auditoria ──► (usada por T11, T12, T13)
      │                └──► T11 Documentos ──► T12 Portal ──► T13 LGPD
      └──► T6 CI        T3 Neon + T4 Hospedagem + T5 E-mail ──► T15 Deploy
```

Enquanto o Dev 1 faz T1 e T2, o Dev 2 resolve T3 a T5 (contas e banco) e estuda o portal no PDF.
