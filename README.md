# Cofre Seguro de Dados – Z4 Motors

Projeto da disciplina de **Cibersegurança** – Engenharia de Software – UniCEUB
Professor: Leonardo Rodrigo Ferreira · Parceiro: Z4 Motors

Sistema web de uso exclusivo da Z4 Motors para armazenar e proteger dados pessoais e
financeiros de clientes usados em financiamento de veículos, com um portal seguro para
o cliente enviar documentos sem passar pelo WhatsApp.

## Pilares
1. Criptografia (AES-256-GCM, Argon2id, HTTPS)
2. Controle de acesso (perfis, 2FA, mascaramento)
3. Auditoria (log com hash encadeado)
4. Documentos (upload criptografado e validado)
5. LGPD (consentimento, retenção, anonimização)
6. Portal de envio (link temporário de uso único)

## Stack
FastAPI · PostgreSQL (Neon) · React + Vite · Render · OWASP ZAP

## Como rodar
```bash
cp .env.example .env        # preencher as chaves
docker compose up --build
docker compose exec api alembic upgrade head
docker compose exec api python -m app.seed
```
- Sistema: http://localhost:8000
- E-mails de teste (Mailpit): http://localhost:8025

Produção: deploy automático no Render a cada push na `main`, com banco no Neon.

## Documentação
- `docs/Briefing_Escopo_MVP_Z4Motors.pdf`: escopo e MVP
- `docs/Apresentacao_Visual_Cofre_Z4.pdf`: telas de referência
- `CLAUDE.md`: regras de arquitetura e segurança do projeto

> Todos os dados usados são fictícios.

## Equipe
- João Victor Guimarães Alves – [@joaoguimaraesalves](https://github.com/joaoguimaraesalves)
- João G. R. Rocha – [@joaogabrieldev](https://github.com/joaogabrieldev)
