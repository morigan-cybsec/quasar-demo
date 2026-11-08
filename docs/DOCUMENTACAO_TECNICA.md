# Quasar demo — documentação técnica

Versão de referência: pacote `qo038` (08/11/2026). Endereço: https://demoquasar.osmio.tech

## 1. Visão geral

```
navegador ──HTTPS──▶ nginx (TLS, limites, cabeçalhos) ──127.0.0.1:8300──▶ app (FastAPI, uvicorn)
                                                                            │
                                                                            ├── db (PostgreSQL 16, rede interna)
                                                                            ├── Groq / Gemini (só o Algory, saída HTTPS)
                                                                            └── provedores simulados (no próprio app)
cron (usuário demo, de hora em hora) ──▶ seed (reconstrói o banco)
```

Tudo roda em **Docker rootless** sob o usuário `demo`, em `/home/demo/quasar-demo`. O app escuta só em localhost; o banco não publica porta.

## 2. Estrutura do repositório

```
backend/
  app.py            produto (rotas de produção: leads, fila, encaminhar, documentos, assinatura, pagamento, relatórios)
  demo_v2.py        a demo: papéis, home, contatos, listas, Algory (ferramentas, regras, propostas, prévias, árvores)
  algory_ia.py      laço do agente, provedores (Groq/Gemini), filtro de injeção, limpeza da saída
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
  v2/               interface nova: index.html, app.js, telas.js, contatos.js, dist.js, rel.js, ficha.html/js, algory.css, ...
  patch_front.py    copia web/v2 para backend/static (sem build)
deploy/
  nginx-demoquasar.conf, dq-proxy.conf     proxy
  demo-reset.sh                            seed por hora (cron)
  testa_algory.py                          suíte do Algory (51 casos)
  gera_correcao.py, correcao_modelo.py     pacotes de correção
docs/
  CASE.md           publicação segura na VPS (levantamento, isolamento, backup)
docker-compose.yml
```

## 3. Serviços e configuração

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
| `ALGORY_LIMITE_*`, `ALGORY_GUARD_LIMIAR` | limites e filtro de injeção |
| `DEMO_CODIGO` (opcional) | código de acesso; vazio = demo aberta |
| `SENHA_DEMO` | senha dos usuários do seed (login fica desativado na demo) |

### nginx
TLS 1.2/1.3 (Let's Encrypt), HTTP→HTTPS, `limit_req` 10 r/s por IP (burst 40) e 5 r/min no login, `client_max_body_size 6m` (64k no Algory), `server_tokens off`, cabeçalhos: HSTS, `X-Content-Type-Options`, `X-Frame-Options SAMEORIGIN`, `Referrer-Policy`, `Permissions-Policy`, `Content-Security-Policy`.

## 4. Sessão, papéis e autorização

- **Login desativado.** O middleware da demo dá a todo visitante sem cookie a sessão do papel padrão e permite trocar entre **renata** (supervisora), **carla** (consultora) e **davi** (dono) por `POST /api/demo/v2/papel`. Qualquer outro valor: 422. O usuário `admin` existe no seed, mas nenhuma sessão da demo o alcança.
- **Cookie** `sessao` = `uid.expira.hmac`, `HttpOnly`, `Secure`, `SameSite=Lax`, 12 h.
- **Autorização** no servidor, em duas camadas: `clausula_dono(u)` restringe as consultas à carteira (consultor vê só os próprios leads e os compartilhados) e `checar_acesso_lead(cur, u, id, escrita)` barra leitura/escrita fora dela (403). Ações de gestão exigem `eh_gestor(u)`; configurações da empresa, `eh_admin(u)`.
- **Só-leitura na demo** (qo038): `/api/usuarios*` (exceto `peso_distribuicao` e `na_fila`), `/api/configs/*`, `/api/config/*`, `/api/esteira/config`, `/api/termo/revalidar`, `/api/financeiro/pin`. `/docs`, `/redoc`, `/openapi.json`: 404.

## 5. Modelo de dados (o essencial)

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
| `metas_mensais` | meta da empresa e por consultor |

Travas de etapa (`HARDLOCK_STATUS`, `_c225_*`): para entrar em certas etapas, a ficha precisa de CPF, banco e tipo de contrato; com a trava de fatos ligada, também do fato (documento gerado, assinatura, pagamento). O Algory usa **as mesmas funções** para dizer "o que falta".

## 6. Fluxos principais

1. **Chegada do lead** (`/api/demo/v2/aquecimento/chegada`): entra no aquecimento sem dono → `c1571_fila` calcula a fila por peso e crédito → `bulk_pretendidos` entrega → cai em *Novo lead* na carteira.
2. **Funil**: mover é `PATCH` da etapa com trava; o gatilho de resposta anda o card sozinho quando o cliente responde no WhatsApp.
3. **Ficha**: OCR (simulado; não processa arquivo do visitante) → campos → `documento` gera o PDF → `assinatura` cria o envelope → simulador assina → webhook HMAC muda a etapa → `cobranca` gera link/Pix/boleto → simulador paga ou recusa → webhook → baixa.
4. **WhatsApp**: fora da janela de 24 h só modelo aprovado; dentro, texto livre. O envio mostra o payload da API da Meta (não envia de verdade).
5. **Listas**: `POST /api/demo/v2/listas` com objetivo (`tocar`, `responder`, `avancar`…); o progresso é calculado pelo estado dos clientes, não por marcação manual.

## 7. O Algory

### Rotas
| Rota | O que faz |
|---|---|
| `GET /api/demo/v2/algory/contexto?tela=` | cartões da tela, `estado` (verdade para testes), assuntos e perguntas |
| `POST /api/demo/v2/algory` | a conversa: `texto`, `tela`, `hist`, `confirmar` (token de proposta), `ajuste`, `cliente`, `op`, `obj` |
| `GET /api/demo/v2/algory/previa?c=` | prévia de uma proposta (lista, fila, meta, mensagem, mover, encaminhar) |
| `GET /api/demo/v2/algory/cliente?id=` | dossiê do cliente (etapa, toque, conversa, documentos, assinatura, cobrança, o que falta) |
| `POST /api/demo/v2/algory/relatorio` | o agente dos Relatórios (mesmos filtros e período da tela) |
| `POST /api/demo/v2/algory/listas` | o agente das listas |

### Ordem de decisão em `POST /api/demo/v2/algory`
1. `obj` → objetivo do fim do galho: monta a proposta sem modelo.
2. Pergunta da árvore que o banco responde direto (`_arvore_resposta`): sem modelo, `modo: dados`.
3. `cliente` + `op` → operação sobre um cliente escolhido na busca (resumo, falta, mensagem, mover, encaminhar).
4. Caso geral: intenção por regex (`INTENCOES`) + agente com ferramentas (`conversas`, `propor_lista_conversas`, `propor_automacao`, `propor_meta`, …) + redes de segurança do servidor (`DESISTE`, `EXPLICA`, `ACAO_NOME`, `TEXTO_CHAMADA`, notas por cargo).
5. Depois de qualquer resposta: `_arvore_pos` acrescenta as próximas perguntas e a sugestão (`objetivo`).

### Propostas
Tokens em memória (`_PROPOSTAS`, 900 s): `distribuir`, `lista:tok`, `listaconv:tok`, `meta:tok`, `modelo:idc:k`, `mover:idc:ids`, `encaminhar:idc:uid`, `reatribuir:tok`, `auto:tok`, `auto_off:id`, `gatilho:idc`. A confirmação executa pelas rotas de produção; `ajuste` só restringe (subconjunto de ids, nome, prazo, valor dentro de `META_MIN..META_MAX`, etapa/pessoa entre as válidas).

### Limites e defesa
- 60 pedidos por IP por hora (429 com o tempo restante); texto cortado em 400 caracteres; histórico de 6 mensagens.
- Prompt Guard (limiar configurável) **e** defesas estruturais: cargo da sessão, números do banco, `_nome_seguro` para nomes que parecem instrução, ações só por token de proposta.
- Suíte: `deploy/testa_algory.py` (51 casos; os 18 ataques repetidos sem o filtro).

### Front (`web/v2/app.js`, `algory.css`)
Modos `alg-grande`, `alg-min`, `alg-card`, `alg-guia`, `alg-icone` com uma animação de abrir e uma de fechar; `algGuia(passos)` aponta elementos (inclusive na ficha, mesmo domínio); `algVigia` liga erros da interface ao "!"; cartões de ação `algAcHtml/algAcMonta`; árvore por assunto (`ASSUNTOS`, `ARV`), busca de cliente na conversa, dossiê, árvores locais (conhecer o Quasar, missões) e o tour dos modos.

## 8. Testes

| Teste | Como | O que cobre |
|---|---|---|
| Ponta a ponta | Playwright/Chromium, roteiro das 7 missões | kanban, OCR, documentos, assinatura, pagamento, recusa, Algory, relatórios |
| Algory | `python3 deploy/testa_algory.py` (na VPS, com o modelo real: ~18 min) | 33 casos de uso + 18 ataques, com e sem filtro |
| Carga | 9 rotas misturadas, 10 a 300 clientes | 0 erros; teto ~100 req/s por processo |
| Segurança | sondas ao vivo (IDOR, cargo, webhooks, traversal, corpo grande, JSON inválido, limites) | ver [CASE.md](../CASE.md#segurança) |

## 9. Operação (na VPS)

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
