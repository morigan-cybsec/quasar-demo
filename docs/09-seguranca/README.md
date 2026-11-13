# 9 · Segurança

[voltar ao início](../../README.md)

## Revisão antes de abrir ao público

Fiz a revisão com sondas ao vivo contra a demo:

| Área | Como foi testado | Resultado |
|---|---|---|
| SQL | consultas dinâmicas revistas; colunas e ordenação só de listas fixas | sem injeção |
| Autorização | consultora tentando ler e escrever leads de outra carteira, ações de gestão, rotas de admin | 403 em todas |
| Sessão | cookie `uid.expira.hmac` (HMAC-SHA256), `HttpOnly`, `Secure`, `SameSite=Lax`; troca de papel só entre os três nomes | sem escalada |
| Webhooks | assinatura e pagamento com HMAC; eventos do Flux com HMAC, janela de tempo e idempotência | recusados sem assinatura |
| Limites | nginx por IP, Algory por IP e por visitante, corpo máximo por rota | ok |
| Cabeçalhos | HSTS, CSP, `X-Frame-Options SAMEORIGIN`, `nosniff`, `Referrer-Policy`, `Permissions-Policy` | ok |
| Container | sem root, sem capabilities, disco só-leitura, rede só com o próprio banco | ok |
| Algory | 18 ataques de injeção de prompt | 18 de 18 segurados ([detalhes](../06-algory/#segurança)) |

Achei cinco problemas e corrigi todos antes de publicar:
1. **Documentação da API exposta** (`/docs`, `/redoc`, `/openapi.json`): agora responde 404.
2. **Administração editável pelo papel "dono"** (usuários, configurações, esteira, PIN do financeiro): essas rotas viraram só-leitura na demo.
3. **Falta de CSP:** adicionei a política no nginx.
4. **Corpo grande na rota do Algory:** limite de 64 KB.
5. **Cookie sobrescrito num caso de borda**, na primeira requisição de um visitante novo: o cookie só é gravado quando a rota não gravou o dela.

Teste de carga, com 9 rotas misturadas e de 10 a 300 clientes simultâneos: **0 erros**, teto de ~100 req/s por processo, 156 MB de memória.

## Publicação na VPS: do código ao ar com segurança

Este é o registro das decisões que tomei para publicar a demonstração do Quasar
(CRM em FastAPI + PostgreSQL) numa VPS que já hospeda outros projetos.
Cada passo traz o porquê e como eu medi.

---

### 0 · Levantamento da VPS (estado inicial)

VPS Debian 13, 1 vCPU, 4 GB RAM, 50 GB disco. Comecei só com comandos de leitura:
`ss -tlnp`, `sshd -T`, `nft list ruleset`, `systemctl list-units`, `psql -l`.

| Item | Encontrado | Avaliação |
|---|---|---|
| SSH | root bloqueado, sem senha (só chave), porta 22 | ok |
| fail2ban | ativo | ok |
| Atualizações de segurança | `unattended-upgrades` ativo | ok |
| Antimalware | agente do provedor ativo | ok |
| Firewall | UFW ativo (via iptables-nft): entrada só 22/80/443, política drop | ok |
| fail2ban → firewall | `banaction = nftables`, mas o binário `nft` não estava instalado | **banimentos falhavam em silêncio** |
| Postgres | só em `127.0.0.1` / `::1` | ok |
| Apps internos | só em `127.0.0.1` (8000, 8100, 8200), atrás do nginx | ok |
| Projetos parados | 2 apps e 4 bancos sem uso, ainda no ar | risco |

Achado no caminho: o `sshd_config` tem `PermitRootLogin yes`, mas o valor
efetivo é `no` porque o `Include` de `sshd_config.d/` vem antes e o sshd usa
o primeiro valor que encontra. Confirmado com `sshd -T`, não com `grep`.

### 1 · Decisão: mesma VPS, com isolamento

Opções avaliadas: VPS dedicada para a demo × isolar na VPS existente.

Escolhi isolar. Para sair da demo e chegar nos outros projetos, o atacante
precisaria de duas falhas encadeadas (execução de código no app **e** escape
de kernel). Com VPS dedicada sobraria só a primeira. O risco residual é uma
vulnerabilidade de kernel, coberto por atualizações automáticas.

As camadas que planejei:

1. Docker **rootless** num usuário `demo` sem sudo: escapar do container dá
   acesso só a esse usuário.
2. App **sem rede**: fala só com o próprio banco; o nginx chega nele por
   socket Unix. Sem saída para a internet, sem alcance ao Postgres ou aos
   apps do host.
3. Container sem root, sem capabilities, disco só-leitura, 0,5 CPU / 512 MB.
4. nginx: TLS, limite de requisições, cabeçalhos de segurança.
5. Banco da demo restaurado do seed a cada hora.

### 2 · Backup geral antes de qualquer mudança

`pg_dump -Fc` de cada banco + `pg_dumpall --globals-only` + `/etc`, `/srv`,
`/opt`, `/home`, `/var/www`, `/root` (sem venvs e caches).
Cada dump é lido de volta com `pg_restore --list` e o pacote com `tar -t`
antes de ser dado como pronto. Hash SHA-256 conferido na cópia externa.

Resultado: 5 bancos + arquivos, 566 MB, todos os dumps lidos de volta.

O teste de leitura pegou três falhas do meu próprio script antes de eu dar o
backup como pronto:
- o `pg_restore --list` rodava como `postgres` numa pasta só do root;
- o filtro `not datistemplate` pulava o banco-molde do multiempresa;
- esse molde tem `ALLOW_CONNECTIONS false`, então o `pg_dump` não entrava
  nele. Abri a trava só durante o dump e fechei logo depois.

Porquê: backup que não foi testado não é backup. E uma cópia só no mesmo
disco não sobrevive à perda do disco, por isso vai também para fora.

### 3 · Firewall: o erro de diagnóstico e o achado real

**Erro meu de leitura no levantamento.** Rodei `ufw status` e `nft list ruleset`
como usuário comum. No Debian esse usuário não tem `/usr/sbin`
no PATH, e eu li `ufw: command not found` como "sem firewall". Havia
um UFW ativo o tempo todo. Cheguei a aplicar uma tabela nftables extra
(com rollback automático, sem perda de acesso) antes de o UFW aparecer em
`nft list tables`. Lição: comando de auditoria roda com `sudo` e caminho
completo; "comando não encontrado" não é evidência de ausência.

Decidi manter **um** firewall só (o UFW existente). Dois em paralelo
somam regras que ninguém lembra de manter iguais. Removi a tabela extra.

**Achado real:** o fail2ban estava configurado com `banaction = nftables`,
mas o pacote `nftables` não estava instalado. Ele detectava os ataques
(2 banimentos registrados) e falhava ao bloquear. Corrigi instalando o
pacote e conferi com um banimento de teste.

Medição (contadores do UFW desde o último boot):

| Item | Pacotes |
|---|---|
| Descartados na entrada | 51.732 |
| Tentativas em SMB/NetBIOS (137–139, 445) | 523 |
| SSH aceitos | 7.786 |

### 4 · No ar: o que apareceu e o que eu decidi

**Scanners antes do primeiro visitante.** Minutos depois de o certificado ser
emitido, o log do app já tinha `GET /` de cinco IPs diferentes, nenhum deles
humano. Todo certificado novo é publicado num registro público (Certificate
Transparency), e robôs leem esse registro para varrer domínios recém-criados.
Consequência prática: o rate limit do nginx e a ausência de conta de admin
exposta tinham de estar valendo no minuto zero, não "depois do AppSec".

**Sem tela de login, e o login fechado.** Para o visitante, a tela de login só
atrapalhava. Mas as senhas das contas de exemplo estão no seed, que é público:
tirar só a tela deixaria `POST /api/login` aceitando `admin@… / demo1234`.
Por isso fiz o middleware da demo fazer as duas coisas: entrega uma sessão assinada
(HMAC, mesma função do app) de um dos três papéis de exemplo e responde 403 em
`/api/login`. Os papéis são supervisora, consultora e dono; o admin, que edita
usuários e integrações, fica fora de propósito. Trocar de papel é uma rota que
só aceita esses três nomes.

**A ficha num iframe.** A ficha nova tem CSS próprio, que colidia com o da
interface (`.it`, `.seg`, `.toast` existiam nos dois). Em vez de renomear
centenas de classes, ela abre num iframe da própria demo. Isso exigiu trocar
`X-Frame-Options: DENY` por `SAMEORIGIN`: continua proibido embutir a demo em
outro site (clickjacking), só a própria origem pode. As mensagens entre a ficha
e a página (`postMessage`) conferem a origem dos dois lados.

**Visitante escreve, outro visitante lê.** Numa demo pública, o que um visitante
digita (nome de cliente, nota, ticket) aparece para o próximo. Três defesas:
toda saída passa por escape de HTML (sem XSS armazenado); a exportação CSV
prefixa com apóstrofo células que começam com `= + - @` (sem injeção de
fórmula no Excel de quem baixa); e o banco volta ao estado de fábrica de hora
em hora.

**As travas de produção valem na demo.** Arrastar um card no kanban chama a
mesma rota de mudança de etapa do Quasar. Levar um lead de "Em contato" direto
para "Assinatura" é recusado com o motivo da regra ("preencha na ficha: banco
do contrato"). A demo não tem atalhos que a produção não tenha.

### Próximos passos

- Desativar (sem apagar) os projetos parados.
- Repositório privado + pipeline: gitleaks, Semgrep, Bandit, pip-audit, Trivy.
- Ataque controlado (OWASP ZAP + testes manuais) e relatório.
