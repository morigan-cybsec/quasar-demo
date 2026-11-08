# Case · Quasar

## Como a ideia surgiu

O Quasar nasceu dentro de uma operação comercial de verdade. Antes de escrever código, o trabalho foi **remodelar o fluxo de processos** da empresa: onde o lead entrava, por quantas mãos passava, onde parava e por que se perdia. O desenho que saiu dessa análise virou o sistema: cada etapa do funil, cada trava e cada regra de distribuição do Quasar corresponde a uma decisão tomada com o time comercial, não a uma feature genérica de CRM.

O que havia antes era o de sempre: planilha de leads, WhatsApp pessoal, contrato no Word, boleto em outro sistema e o supervisor descobrindo o problema no fim do mês. O Quasar juntou isso numa esteira só, com as regras no servidor.

## O que está em produção

Hoje o Quasar atende **duas empresas** que contrataram o serviço. O que roda nelas:

| Área | Em produção |
|---|---|
| Captação | entrada de leads de anúncios (click-to-WhatsApp e planilha), deduplicação, origem de marketing por lead |
| Distribuição | fila por peso e crédito por consultor, aquecimento, encaminhamento e reencaminhamento de parados |
| Funil | kanban em fases e etapas com travas (dados, documento, assinatura, pagamento), gatilhos automáticos, histórico auditável |
| Ficha | OCR de documento, titulares e endereços, contrato e termo em PDF gerados no servidor, versões |
| Assinatura | envelopes com estados e eventos, webhook assinado (Clicksign) |
| Pagamento | links de pagamento, Pix, boleto, cartão parcelado, recusa e troca de método, baixa por webhook (PayZen, Cora) |
| Atendimento | WhatsApp Business com janela de 24 h e modelos aprovados; registro de ligações |
| Gestão | metas mensais, ranking, carteira por consultor, relatórios financeiros e de funil, ROI por canal de marketing |
| Governança | cargos (consultor, supervisor, dono, admin, hunter), termo de uso versionado, trilha de auditoria |

A demo pública usa **o mesmo código de produção** com os dados trocados por um cenário fictício (a "Demo Corporation", um provedor de internet) e os provedores externos trocados por simuladores que respondem pelos mesmos webhooks.

## O que a demo acrescenta

Dois diferenciais que ainda não estão em produção e que a demo serve para provar:

- **Algory**, o assistente: lê o sistema com o acesso de quem pergunta, responde com os números de agora e **leva até a ação** (lista, mensagem, mover, encaminhar, fila, meta), sempre como proposta revisável. Suíte de 51 casos; os 18 ataques de injeção de prompt barrados em 18 de 18.
- **Missões guiadas**: 10 missões levam o visitante pelo ciclo inteiro, da leitura do RG ao dinheiro na conta, com o Algory apontando cada clique.

## Números

| | |
|---|---|
| Código | ~70 mil linhas (backend, schema de 140 tabelas, interface) |
| Rotas | 464 no produto, 61 da demo |
| Testes | 7 missões de ponta a ponta (Playwright), 51 casos do Algory, teste de carga (0 erros até 300 clientes simultâneos) |
| Publicação | 38 pacotes de correção cumulativos, cada um verificado e com backup |

## Segurança

Revisão feita antes de abrir a demo ao público, com sondas ao vivo: SQL (consultas dinâmicas com colunas de listas fixas), autorização por carteira e por cargo (403 em todas as tentativas), sessão assinada por HMAC, webhooks com assinatura, limites por IP, cabeçalhos de segurança e CSP, container sem root e só-leitura. Os cinco achados da revisão (documentação da API exposta, administração editável pelo papel "dono", falta de CSP, corpo grande numa rota, cookie sobrescrito num caso de borda) foram corrigidos antes da publicação. A publicação da VPS (isolamento, Docker rootless, firewall, backup) está em [docs/CASE.md](docs/CASE.md).

## Autores

**Morigan Ventrici** e **Matheus Ventrici**.
