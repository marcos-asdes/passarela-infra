<p align="center">
  <img src="docs/logo.png" alt="Passarela" width="360" />
</p>

Repositório de infraestrutura do **Passarela** (`backend/` e `frontend/` são repositórios git próprios, irmãos deste). Contém tudo que roda na VPS de produção: `docker-compose.prod.yml`, `Caddyfile` (proxy reverso + HTTPS automático) e o runbook de setup abaixo.

## Arquitetura

Uma única VPS sempre-on roda 4 containers via Docker Compose:

- **caddy** — proxy reverso, emite/renova certificado HTTPS (Let's Encrypt) automaticamente para os 2 domínios, repassa upgrade de WebSocket sem configuração extra.
- **web** — imagem `ghcr.io/<owner>/passarela-web` (nginx servindo o build estático do frontend).
- **api** — imagem `ghcr.io/<owner>/passarela-api` (NestJS). Cron (`@nestjs/schedule`) e WebSocket (Socket.IO) rodam dentro do próprio processo — por isso a VPS precisa ficar sempre ligada, sem scale-to-zero.
- **mongo** — MongoDB self-hosted, volume próprio.

`api`/`web` não publicam porta no host — só o `caddy` fala com eles, via rede Docker interna (`passarela`).

Cada push em `main` do `backend/` ou `frontend/` builda a imagem, publica no GHCR e faz deploy via SSH (`docker compose pull/up -d --no-deps <serviço>`) — deploy independente por serviço, sem acoplar os 2 pipelines. Push neste repo (infra) só dispara quando `docker-compose.prod.yml`/`Caddyfile` mudam, e sincroniza + recarrega o Caddy.
