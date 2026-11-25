# Cofre Seguro de Dados – Z4 Motors

Projeto acadêmico da disciplina de Cibersegurança (Engenharia de Software – UniCEUB),
em parceria com a concessionária Z4 Motors. Professor: Leonardo Rodrigo Ferreira.

Sistema web, de uso exclusivo da Z4, para armazenar e proteger dados pessoais e
financeiros de clientes usados em financiamento de veículos, mais um portal seguro
para o cliente enviar documentos (substituindo o WhatsApp).

## Documentos de referência (em `docs/`)

- `docs/Apresentacao_Visual_Cofre_Z4.pdf`: **referência visual das telas**. O front-end deve
  seguir esses mockups: login com 2FA, painel de clientes com CPF mascarado e cards de
  indicadores, ficha do cliente por perfil (vendedor x analista), tela de auditoria com
  alerta de integridade e as 4 telas mobile do portal (código → termo LGPD → documentos →
  confirmação). Paleta: azul-marinho #0B2A5B, azul #1F5FA8, azul-claro #E8F0FA,
  destaque amarelo #F2A900, verde #1E9E6A, vermelho #D64545.
- `docs/Briefing_Escopo_MVP_Z4Motors.pdf`: escopo, MVP e critérios de aceite de cada
  funcionalidade. Em caso de dúvida sobre o que entra ou não, seguir esse documento.

Leia esses documentos antes de criar ou alterar telas e funcionalidades.

## Stack

- Back-end: Python 3.12 + FastAPI + SQLAlchemy 2 + Alembic
- Banco: PostgreSQL 16
- Front-end: React + Vite + TypeScript (painel interno e portal do cliente, responsivo)
- Infra: Docker Compose (api, db, web, mailpit para e-mails de teste)
- Testes: pytest (back), OWASP ZAP (varredura de segurança)
- Idioma: código em inglês; textos de interface e mensagens em português do Brasil

## Estrutura

```
backend/
  app/
    core/        config, security (crypto, hashing, totp), deps
    models/      SQLAlchemy
    schemas/     Pydantic
    repositories/  ÚNICA camada que acessa o banco
    services/    regras de negócio
    api/         rotas (auth, clients, documents, audit, portal, admin)
    jobs/        anonimização LGPD agendada
  tests/
frontend/
  src/pages/     Login, Dashboard, ClientDetail, Audit, Users, Portal/*
docker-compose.yml
.env.example
```

## Regras de segurança (obrigatórias)

1. **Criptografia de campos**: CPF, renda, dados bancários e segredo 2FA são gravados
   com AES-256-GCM (biblioteca `cryptography`), nonce aleatório por valor.
   A chave vem de variável de ambiente (`DATA_ENCRYPTION_KEY`), nunca do banco nem do código.
   Para busca por CPF, guardar também um HMAC-SHA256 do CPF (blind index).
2. **Senhas**: Argon2id (`argon2-cffi`). Nunca logar senha, token ou dado sensível.
3. **2FA**: TOTP obrigatório para todos os usuários internos (`pyotp`).
4. **Sessão**: JWT de curta duração (15 min) em cookie HttpOnly, Secure, SameSite=Strict.
   Bloqueio temporário após 5 tentativas de login falhas.
5. **Autorização no servidor**: toda rota verifica o perfil (seller, analyst, manager).
   Nunca confiar apenas em esconder algo na tela.
   - seller: nome, contato, veículo, status. Sem CPF completo, renda, banco ou documentos.
   - analyst: dados financeiros e documentos (CPF mascarado até "revelar").
   - manager: tudo + usuários + auditoria.
6. **Mascaramento**: a API devolve dados sensíveis mascarados por padrão
   (`***.482.913-**`). Endpoint específico de "revelar" gera registro de auditoria.
7. **Auditoria**: tabela append-only. Cada registro guarda o hash SHA-256 do registro
   anterior (hash encadeado). Endpoint `GET /audit/verify` percorre a corrente e aponta
   o primeiro registro adulterado. Registrar: login, falha de login, consulta, revelar,
   criação/edição/exclusão, upload, download, geração e uso de link, anonimização.
8. **Documentos**: aceitar só PDF, JPG e PNG, até 10 MB, validando a assinatura real do
   arquivo (magic bytes), não a extensão. Armazenar criptografado fora da pasta pública.
   Download apenas por rota autenticada e autorizada.
9. **Portal do cliente**: link com token aleatório de 32 bytes; no banco só o hash do token.
   Validade de 24h, uso único. Cliente confirma código de 6 dígitos enviado por e-mail
   (hash no banco, 3 tentativas). Após o envio, o link é invalidado.
   O portal só RECEBE dados; nunca exibe dados já armazenados.
10. **LGPD**: registrar aceite do termo (data, hora, IP, versão). Job diário anonimiza
    clientes com prazo de retenção vencido. Coletar apenas os campos necessários.
11. **Geral**: HTTPS (em dev via túnel), CORS restrito, cabeçalhos de segurança,
    validação de entrada com Pydantic, consultas só via ORM (sem SQL concatenado),
    rate limiting em login e portal, mensagens de erro sem detalhes internos.
12. **Dados**: usar APENAS dados fictícios (seed com Faker pt_BR). Nunca dados reais.

## Execução local + túnel (forma de apresentação)

O sistema roda só no notebook e é exposto no dia da apresentação por Cloudflare Tunnel.

- Expor apenas o front (porta 5173). O Vite faz proxy de `/api` para o back-end, então
  front e API ficam no mesmo domínio (sem CORS entre domínios, cookie de sessão funcionando).
- `vite.config.ts`: `server.host = true` e `server.allowedHosts = ['.trycloudflare.com', 'localhost']`.
- O endereço do túnel muda a cada execução: nenhuma URL fixa no código. Os links do
  portal enviados por e-mail usam a variável `PUBLIC_URL` do `.env`.
- Cookies com `Secure` (o túnel entrega HTTPS). Em `localhost`, permitir via configuração de dev.
- E-mails (código do portal) vão para o Mailpit (`http://localhost:8025`); não usar SMTP real.

## Convenções

- Tudo que acessa o banco passa por `repositories/` (permite trocar o banco depois).
- Toda nova rota precisa de teste cobrindo o caso permitido e o caso proibido por perfil.
- Segredos só em `.env` (que está no `.gitignore`); manter `.env.example` atualizado.
- Commits pequenos, em português, descrevendo o que mudou.

## Comandos

- Subir tudo: `docker compose up --build`
- Migrações: `docker compose exec api alembic upgrade head`
- Seed: `docker compose exec api python -m app.seed`
- Testes: `docker compose exec api pytest`
- Portal no celular (demo): `cloudflared tunnel --url http://localhost:5173`

## Roteiro da demonstração (o sistema precisa suportar)

1. Mostrar no banco que os campos sensíveis estão ilegíveis.
2. Mesmo cliente visto como vendedor e como analista.
3. Chamada direta à API com perfil errado → 403.
4. Link de portal usado/expirado → bloqueado.
5. Upload de .exe renomeado para .pdf → rejeitado.
6. Alterar um registro de auditoria no banco → `/audit/verify` acusa.
7. Rodar o job de anonimização → cliente vencido anonimizado.
