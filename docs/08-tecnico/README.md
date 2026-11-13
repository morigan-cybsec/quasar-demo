# 8 · Documentação técnica

[voltar ao início](../../README.md)

Versão de referência: pacote `qo048` (13/11/2026). Endereço: https://demoquasar.osmio.tech

Escrevi esta parte para quem vai avaliar o sistema por dentro. O código não é público, mas a estrutura, as decisões e os testes estão aqui.

## 1. Visão geral

```
navegador ──HTTPS──▶ nginx (TLS, limites, cabeçalhos) ──127.0.0.1:8300──▶ app (FastAPI, uvicorn)
                                                                            │
                                                                            ├── db (PostgreSQL 16, rede interna)
                                                                            ├── Claude Haiku · Groq · Gemini (só o Algory, saída HTTPS)
                                                                            ├── Flux (eventos assinados, HTTPS)
                                                                            └── provedores simulados (no próprio app)
cron (usuário demo, de hora em hora) ──▶ seed (reconstrói o banco)
```

Tudo roda em **Docker rootless** sob o usuário `demo`, em `/home/demo/quasar-demo`. O app escuta só em localhost; o banco não publica porta.

## 2. Por que essas ferramentas

| Ferramenta | Por que eu escolhi |
|---|---|
| **Python + FastAPI** | O Quasar de produção já é Python. Com o FastAPI, cada rota tem o corpo validado por modelo (pydantic) e as rotas ficam explícitas e tipadas. A demo reaproveita as rotas de produção sem camada nova. |
| **PostgreSQL 16** | Os dados são relacionais e as regras dependem de integridade (etapa, dono, contrato, cobrança). Uso gatilhos do próprio banco onde a consistência importa: a fila de eventos para o Flux é gravada na mesma transação da mudança de etapa, então nenhum evento se perde nem sai sem a mudança. |
| **HTML/CSS/JS sem framework** | É o mesmo front da produção, sem etapa de build. A VPS é pequena e o deploy é copiar arquivos. A página carrega rápido e não depende de pacote de terceiros no navegador. |
| **WeasyPrint** | O contrato e o termo saem de HTML/CSS, com o mesmo modelo que a tela usa. O PDF é gerado no servidor, então o documento é sempre o que os dados da ficha dizem. |
| **Docker rootless** | A demo divide a VPS com outros projetos. Num usuário sem sudo, quem escapar do container chega só nesse usuário. Uso também disco só-leitura, sem capabilities e com rede só para o banco. |
| **nginx** | TLS, limite por IP, tamanho de corpo por rota e cabeçalhos de segurança, antes de a requisição chegar no app. |
| **Claude Haiku 5.5** | Modelo principal do Algory. Chama ferramentas de forma confiável e custa ~US$ 0,0006 por pergunta, o que cabe numa verba de US$ 5 por mês. Entra pela camada compatível com OpenAI, então o código é o mesmo dos outros provedores. |
| **Groq (gpt-oss) e Gemini** | Reserva gratuita. Quando o limite do Haiku acaba, o visitante continua atendido sem custo. |
| **Prompt Guard** | Filtro de injeção antes do modelo, separado do modelo que responde. |
| **Playwright** | Testa as missões como um visitante: clica, arrasta, abre a ficha, paga no simulador. Pega erro que teste de rota não pega. |
| **Pacotes de correção** em vez de git no servidor | O servidor não tem acesso ao repositório. Cada pacote leva os arquivos inteiros, com hash. Antes de ir, ele é aplicado duas vezes num ambiente de teste e comparado arquivo a arquivo; no servidor, faz backup antes de trocar. |

## 3. Estrutura do código

```
backend/
  app.py            produto (rotas de produção: leads, fila, encaminhar, documentos, assinatura, pagamento, relatórios)
  demo_v2.py        a demo: papéis, home, contatos, listas, Algory (ferramentas, regras, propostas, prévias, árvores)
  algory_ia.py      laço do agente, provedores (Claude/Groq/Gemini), custo por chamada, filtro de injeção, limpeza da saída
  integracao_flux.py  eventos com o Flux: fila de saída, envio assinado, recebimento, base de assinantes
  dados/seed_democorp.json  seed compartilhado da Demo Corporation (557 assinantes)
  algory_kb.md      base de conhecimento do Algory (o que ele explica sem consultar o banco)
  ficha_demo.py     a ficha: OCR simulado, documentos PDF, assinatura, tickets, cobrança, WhatsApp, agenda
  demo.py           simuladores de pagamento e assinatura (páginas e webhooks)
  simulado.py, payzen.py, cora.py, integracao.py, fase_pagamento.py, pagamento_baixa.py   provedores e baixa
  static/           interface servida (gerada a partir de web/)
  Dockerfile, requirements.txt
db/
  schema.sql        140 tabelas
  seed.py           dados fictícios, determinísticos
web/
  v2/               interface nova: index.html, app.js, telas.js, contatos.js, dist.js, rel.js, ficha.html/js, algory.css, apresentacao.html, ...
  patch_front.py    copia web/v2 para backend/static (sem build)
deploy/
  nginx-demoquasar.conf, dq-proxy.conf     proxy
  demo-reset.sh                            seed por hora (cron)
  testa_algory.py                          suíte do Algory (55 casos)
  gera_correcao.py, correcao_modelo.py     pacotes de correção
docker-compose.yml
```

## 4. Serviços e configuração

### docker-compose
| Serviço | Imagem | Limites | Observações |
|---|---|---|---|
| `db` | postgres:16-alpine | 0,5 CPU, 384 MB | schema no `initdb`; só rede interna |
| `seed` | build `backend/` | — | roda uma vez e sai; `db/seed.py` |
| `app` | build `backend/` | 1 CPU, 512 MB | FS só-leitura, `cap_drop: ALL`, `no-new-privileges`, usuário 10001, `127.0.0.1:8300` |

### `.env` (só na VPS, nunca no repositório)
| Variável | Para quê |
|---|---|
| `APP_SECRET` (≥ 32 bytes) | assina os cookies de sessão (HMAC-SHA256) |
| `DB_PASSWORD` | Postgres |
| `SIM_WEBHOOK_TOKEN` | token do webhook do provedor simulado |
| `CLICKSIGN_WEBHOOK_SECRET` | HMAC do webhook de assinatura |
| `GROQ_API_KEY`, `ALGORY_MODELO` | agente (padrão `openai/gpt-oss-120b`, reserva 20b) |
| `GEMINI_API_KEY`, `GEMINI_MODELO` | reserva quando a Groq estoura a cota |
| `ANTHROPIC_API_KEY`, `ALGORY_HAIKU_*` | o Haiku e os limites da demo pública (perguntas por hora, dias, verba do mês, pico) |
| `FLUX_EVENTOS_SEGREDO`, `FLUX_EVENTOS_URL` | integração com o Flux (HMAC dos eventos) |
| `ALGORY_LIMITE_*`, `ALGORY_GUARD_LIMIAR` | limites e filtro de injeção |
| `DEMO_CODIGO` (opcional) | código de acesso; vazio = demo aberta (é como está) |
| `SENHA_DEMO` | senha dos usuários do seed (login fica desativado na demo) |

### nginx
TLS 1.2/1.3 (Let's Encrypt), HTTP→HTTPS, `limit_req` 10 r/s por IP (burst 40) e 5 r/min no login, `client_max_body_size 6m` (64k no Algory), `server_tokens off`, cabeçalhos: HSTS, `X-Content-Type-Options`, `X-Frame-Options SAMEORIGIN`, `Referrer-Policy`, `Permissions-Policy`, `Content-Security-Policy`.

## 5. Sessão, papéis e autorização

- **Login desativado.** O middleware da demo dá a todo visitante sem cookie a sessão do papel padrão e permite trocar entre **renata** (supervisora), **carla** (consultora) e **davi** (dono) por `POST /api/demo/v2/papel`. Qualquer outro valor: 422. O usuário `admin` existe no seed, mas nenhuma sessão da demo o alcança.
- **Cookie** `sessao` = `uid.expira.hmac`, `HttpOnly`, `Secure`, `SameSite=Lax`, 12 h.
- **Autorização** no servidor, em duas camadas: `clausula_dono(u)` restringe as consultas à carteira (consultor vê só os próprios leads e os compartilhados) e `checar_acesso_lead(cur, u, id, escrita)` barra leitura/escrita fora dela (403). Ações de gestão exigem `eh_gestor(u)`; configurações da empresa, `eh_admin(u)`.
- **Só-leitura na demo** (qo038): `/api/usuarios*` (exceto `peso_distribuicao` e `na_fila`), `/api/configs/*`, `/api/config/*`, `/api/esteira/config`, `/api/termo/revalidar`, `/api/financeiro/pin`. `/docs`, `/redoc`, `/openapi.json`: 404.

## 6. Modelo de dados (o essencial)

| Tabela | Papel |
|---|---|
| `clientes_leads` | o lead/cliente: dados, etapa (`id_status`), dono (`id_consultor_atual`), pretendido pela fila, toques, origem de marketing |
| `status_catalogo` | as 11 etapas em 4 fases (`FUNIL`), com códigos como `PROSPECT`, `CONTATO`, `NEGOCIACAO`, `ASSINAR_CONTRATO`, `REALIZAR_PAGAMENTO`, `POS_VENDA` |
| `usuarios` | cargo (`consultor`, `supervisor`, `dono`, `admin`, `hunter`), peso e presença na fila |
| `contratos`, `titulares`, `enderecos` | o que entra no documento |
| `documentos_gerados` | PDFs por versão (WeasyPrint), páginas em PNG |
| `assinaturas`, eventos | envelopes do FluxSign: `enviado → em_assinatura → assinado | vencido`, webhook idempotente |
| `pagamentos_link`, eventos, `financeiro_*` | cobranças do FluxPay: `aberta → paga | recusada | vencida → liquidada`, parcelas, taxa, líquido |
| `agendamentos`, `tickets`, `historico` | agenda, tickets com valor, trilha de auditoria |
| `demo_listas`, `demo_automacoes`, chats do Algory (`DDL_CHAT`) | listas de trabalho com objetivo e progresso; automações que montam lista e avisam |
| `demo_flux_saida`, `demo_flux_entrada`, `demo_flux_base` | fila de eventos para o Flux, eventos recebidos, base de assinantes |
| `demo_alg_haiku`, `demo_alg_haiku_hora`, `demo_alg_visitante` | uso do Haiku por visitante (hash do IP) por dia e por hora; ficam fora do reset |
| `metas_mensais` | meta da empresa e por consultor |

Travas de etapa (`HARDLOCK_STATUS`, `_c225_*`): para entrar em certas etapas, a ficha precisa de CPF, banco e tipo de contrato; com a trava de fatos ligada, também do fato (documento gerado, assinatura, pagamento). O Algory usa **as mesmas funções** para dizer "o que falta".

## 7. Fluxos principais

1. **Chegada do lead** (`/api/demo/v2/aquecimento/chegada`): entra no aquecimento sem dono → `c1571_fila` calcula a fila por peso e crédito → `bulk_pretendidos` entrega → cai em *Novo lead* na carteira.
2. **Funil**: mover é `PATCH` da etapa com trava; o gatilho de resposta anda o card sozinho quando o cliente responde no WhatsApp.
3. **Ficha**: OCR (simulado; não processa arquivo do visitante) → campos → `documento` gera o PDF → `assinatura` cria o envelope → simulador assina → webhook HMAC muda a etapa → `cobranca` gera link/Pix/boleto → simulador paga ou recusa → webhook → baixa.
4. **WhatsApp**: fora da janela de 24 h só modelo aprovado; dentro, texto livre. O envio mostra o payload da API da Meta (não envia de verdade).
5. **Listas**: `POST /api/demo/v2/listas` com objetivo (`tocar`, `responder`, `avancar`…); o progresso é calculado pelo estado dos clientes, não por marcação manual.

## 8. O Algory

Tem pasta própria: [docs/06-algory](../06-algory/). Lá estão a ordem de decisão, como ele aciona a tela, a gestão dos tokens, a segurança, os testes e as rotas.

## 9. Testes

| Teste | Como | O que cobre |
|---|---|---|
| Ponta a ponta | Playwright/Chromium, roteiro das 7 missões | kanban, OCR, documentos, assinatura, pagamento, recusa, Algory, relatórios |
| Algory | `python3 deploy/testa_algory.py` (na VPS, com o modelo real: ~19 min) | 37 casos de uso + 18 ataques; última rodada: 31/31 e 18/18 |
| Painel do Algory | cliques aleatórios (16 sequências de 150 ações) | o painel nunca fica preso nem vazio |
| Integração Flux | servidor falso do Flux com eventos assinados | 200/401/400/404/409 como o contrato |
| Carga | 9 rotas misturadas, 10 a 300 clientes | 0 erros; teto ~100 req/s por processo |
| Segurança | sondas ao vivo (IDOR, cargo, webhooks, traversal, corpo grande, JSON inválido, limites) | ver [segurança](../09-seguranca/) |

## 10. Operação (na VPS)

```
sudo dq ps                      # serviços
sudo dq logs -f app             # log
sudo dq run --rm seed           # reset manual
sudo dq build --quiet && sudo dq up -d
sudo python3 ~/correcao_qoNNN.py   # aplica um pacote (cumulativo; backup em /home/demo/backup-qoNNN)
```
Pacotes de correção: um arquivo Python com os arquivos completos (gzip+base64+sha256); rodar duas vezes não estraga; voltar atrás é copiar do backup e reconstruir.

---
© 2026 Morigan Ventrici e Matheus Ventrici. Todos os direitos reservados.
