# Divisão de tarefas – Cofre Seguro de Dados (Z4 Motors)

| | Dev | Plano Claude | Foco |
|---|---|---|---|
| **Dev 1** | João Victor Guimarães Alves (@joaoguimaraesalves) | Pro (limite semanal menor) | Contas e serviços, módulos menores, testes, revisão, ZAP e apresentação |
| **Dev 2** | João Gabriel R. Rocha (@joaogabrieldev) | Max | Geração do código pesado: estrutura, segurança, login, clientes, documentos, portal, telas |

Divisão pensada pelo consumo de tokens: o Dev 2 faz as tarefas que geram muito código; o Dev 1
faz tarefas curtas, configuração, validação manual e a apresentação (que não precisa de Claude Code).
Quando o limite semanal do Dev 1 renovar, ele pode puxar tarefas da lista "Pode passar para o Dev 1".

Regras gerais:
- Uma branch por tarefa (`feat/<nome>`), PR para a `main`, **o outro revisa** antes do merge
  (revisar lendo o diff no GitHub não gasta tokens).
- Depois de cada tarefa: "Explique em poucas linhas o que você fez e por quê"
  (para os dois saberem defender na apresentação).
- Migrações Alembic: sempre `git pull` antes de criar uma nova.

## Economia de tokens (principalmente para o Dev 1)

- **Não pedir para ler os PDFs a cada tarefa.** O CLAUDE.md já resume escopo, regras e paleta.
  O PDF visual só é necessário em tarefas de **tela**, e de preferência só a página da tela em questão.
- **Uma tarefa por sessão** e `/clear` entre tarefas: o contexto acumulado é o que mais consome.
- Usar modelo mais leve (Sonnet) para tarefas simples: configuração, testes, ajustes de texto, CI.
- Prompts objetivos, citando o arquivo/pasta exato ("em backend/app/api/audit.py…") para o Claude
  não sair lendo o projeto inteiro.
- Evitar "rode tudo e me mostre a saída completa": pedir só o resumo dos erros.

---

## Fase 0 – Fundação

### Dev 2 (Max)
- [ ] **T1 – Estrutura do projeto** (`feat/estrutura`)
  > Leia o CLAUDE.md. Crie a estrutura: backend FastAPI com /api/health, frontend React + Vite + TypeScript + Tailwind com a paleta do CLAUDE.md, docker-compose para dev (api, web, mailpit), Dockerfile multi-stage que compila o front e serve junto com a API, Alembic configurado.
- [ ] **T2 – Segurança central** (`feat/core-security`)
  > Implemente app/core/security: AES-256-GCM para campos, blind index HMAC-SHA256 para CPF, Argon2id (parâmetros configuráveis por env), TOTP com pyotp. Testes unitários.
- [ ] **T3 – Banco no Neon**
  - Criar projeto `cofre`, branches `dev` e `main`. Passar a URL *pooled* do `dev` ao Dev 1 **por mensagem privada** (nunca no Git).

### Dev 1 (Pro) – sem ou com pouco Claude Code
- [ ] **T4 – Hospedagem:** criar a conta na plataforma escolhida (Render / Koyeb / Fly.io / Heroku) e conectar ao repositório.
- [ ] **T5 – E-mail:** criar conta no Resend e gerar a API key (passar ao Dev 2 em privado).
- [ ] **T6 – Segurança do GitHub:** ativar Dependabot e secret scanning; proteger a branch `main` (exigir PR com 1 aprovação).
- [ ] **T7 – CI** (`feat/ci`), após a T1 – tarefa curta, usar Sonnet
  > Crie um workflow do GitHub Actions que roda pytest no backend e o build do frontend a cada PR.

---

## Fase 1 – Funcionalidades

### Dev 2 (Max) – código pesado
- [ ] **T8 – Login, 2FA e perfis** (`feat/auth`)
  > Usuários com perfis seller/analyst/manager, login com Argon2id + TOTP obrigatório, sessão JWT em cookie HttpOnly/Secure/SameSite=Strict (15 min), bloqueio após 5 falhas, rate limiting. Telas de login e cadastro do 2FA (QR code). Testes de acesso negado.
- [ ] **T9 – Clientes** (`feat/clientes`)
  > CRUD de clientes com CPF, renda e dados bancários criptografados, busca por CPF via blind index, mascaramento por padrão, endpoint de revelar (com auditoria), autorização por perfil no servidor. Testes do 403 por perfil.
- [ ] **T10 – Documentos** (`feat/documentos`)
  > Upload de PDF/JPG/PNG até 10 MB validando magic bytes, criptografado em AES-256-GCM em coluna bytea. Download só para analyst/manager. Auditoria em upload e download.
- [ ] **T11 – Portal do cliente** (`feat/portal`)
  > Vendedor gera link (token de 32 bytes, só o hash no banco, 24h, uso único). Código de 6 dígitos por e-mail (Mailpit em dev, Resend em produção), 3 tentativas, aceite do termo LGPD (data, hora, IP, versão), preenchimento e envio de documentos. Link invalidado após o envio. As 4 telas mobile da página 4 do PDF visual. O portal nunca exibe dados já armazenados.
- [ ] **T12 – Telas internas** (`feat/telas-internas`)
  > Painel de clientes com cards de indicadores, ficha do cliente por perfil e gestão de usuários, seguindo as páginas 5 a 7 do PDF visual.

### Dev 1 (Pro) – módulos menores e independentes
- [ ] **T13 – Auditoria** (`feat/auditoria`) – depois da T2
  > Em backend/app: tabela de auditoria append-only com hash encadeado SHA-256, função audit.log(usuario, acao, cliente_id) para os outros módulos usarem, e GET /api/audit/verify que aponta o primeiro registro adulterado. Testes.
  - Entregar cedo: o Dev 2 usa a função nas T9–T11. A **tela** de auditoria fica para o Dev 2 junto com a T12.
- [ ] **T14 – LGPD** (`feat/lgpd`) – usar Sonnet
  > Job que anonimiza clientes com prazo de retenção vencido (com auditoria), endpoint POST /api/admin/jobs/anonymize protegido por CRON_SECRET e GitHub Actions agendado 1x por dia.
- [ ] **T15 – Seed** (`feat/seed`) – usar Sonnet
  > Seed com Faker pt_BR: 3 usuários (um por perfil, com 2FA), ~30 clientes em vários status, alguns com prazo vencido. Só dados fictícios.
- [ ] **T16 – Testes manuais:** a cada PR do Dev 2, testar no navegador o caso permitido e o proibido (sem gastar tokens) e comentar no PR.

### Pode passar para o Dev 1 quando o limite renovar
- Deploy (T17), tela de auditoria, botão "rodar anonimização agora" para o gerente.

---

## Fase 2 – Fechamento

- [ ] **T17 – Deploy** (Dev 2, ou Dev 1 se tiver tokens)
  > Configure o deploy na plataforma escolhida usando o Dockerfile, migrações automáticas no start, variáveis documentadas no .env.example e /api/health para acordar o serviço.
- [ ] **T18 – Revisão de segurança com IA** (Dev 2, com o Max)
  > Revise o projeto inteiro contra o OWASP Top 10 e as regras do CLAUDE.md. Liste os problemas por gravidade antes de corrigir.
- [ ] **T19 – OWASP ZAP** (Dev 1, ferramenta própria, não gasta tokens): varrer o sistema publicado, guardar o relatório e abrir issues para o Dev 2 com os achados.
- [ ] **T20 – Ensaio e vídeo de backup** (os dois): rodar os 7 passos do roteiro do CLAUDE.md no ambiente publicado.
- [ ] **T21 – Slides** (Dev 1): problema → solução → arquitetura → demo → antes x depois → protótipo x produção → relatório do ZAP.
  - Dev 1 apresenta: problema, arquitetura, auditoria, LGPD e ZAP.
  - Dev 2 apresenta: criptografia, login/2FA, perfis e portal do cliente.

---

## Dependências

```
T1 Estrutura (D2) ──► T2 Core security (D2) ──► T8 Auth (D2) ──► T9 Clientes (D2) ──► T12 Telas (D2)
      │                       │
      │                       └──► T13 Auditoria (D1) ──► usada por T9, T10, T11
      │                                                   T10 Documentos (D2) ──► T11 Portal (D2)
      │                                                   T14 LGPD (D1)   T15 Seed (D1)
      └──► T7 CI (D1)        T3 Neon (D2) + T4 Hospedagem (D1) + T5 E-mail (D1) ──► T17 Deploy
```

Enquanto o Dev 2 faz T1 e T2, o Dev 1 resolve T4 a T6 sem gastar tokens.
