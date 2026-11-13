# Case · Quasar

Por Morigan Ventrici.

## Como a ideia surgiu

O Quasar nasceu dentro de uma operação comercial de verdade. Antes de escrever código, eu remodelei o fluxo de processos da empresa: onde o lead entrava, por quantas mãos passava, onde parava e por que se perdia. O sistema saiu desse desenho. Cada etapa do funil, cada trava e cada regra de distribuição corresponde a uma decisão tomada com o time comercial, e não a uma feature genérica de CRM.

O que havia antes era o de sempre: planilha de leads, WhatsApp pessoal, contrato no Word, boleto em outro sistema e o supervisor descobrindo o problema no fim do mês. Juntei isso numa esteira só, com as regras no servidor.

## Uso comercial

O Quasar está em uso comercial em empresas que contrataram o serviço. Mantenho as empresas em sigilo. É um serviço prestado por mim e pelo Matheus Ventrici: implantação, adaptação do fluxo e manutenção.

O que roda em produção:

| Área | Em produção |
|---|---|
| Captação | leads de anúncios (click-to-WhatsApp e planilha), deduplicação, origem de marketing por lead |
| Distribuição | fila por peso e crédito por consultor, aquecimento, encaminhamento e reencaminhamento de parados |
| Funil | kanban em fases e etapas com travas (dados, documento, assinatura, pagamento), gatilhos automáticos, histórico auditável |
| Ficha | ficha dinâmica, OCR de documento, titulares e endereços |
| Contrato | contrato e termo gerados no servidor a partir da ficha, em PDF, com versões |
| Assinatura | por API: envelopes com estados e eventos, webhook assinado (Clicksign) |
| Pagamento | por API: link, Pix, boleto, cartão parcelado, recusa e troca de método, baixa por webhook (PayZen, Cora) |
| Atendimento | WhatsApp Business pela API da Meta, com histórico das conversas na ficha, janela de 24 h e modelos aprovados; registro de ligações |
| Gestão | metas mensais, ranking, carteira por consultor, relatórios financeiros e de funil, ROI por canal |
| Governança | cargos (consultor, supervisor, dono, admin, hunter), termo de uso versionado, trilha de auditoria |

A demo usa o mesmo código. Troquei os dados por um cenário fictício, a Demo Corporation (um provedor de internet), e os provedores externos por provedores internos que respondem pelos mesmos webhooks. É a mesma estrutura de provedor plugável que permite trocar a API de assinatura ou de pagamento sem mexer no resto.

## O que a demo acrescenta

O que ainda não está em produção, e que a demo serve para provar:

- **Algory como agente interno.** O Algory nasceu de um projeto de faculdade do Matheus, e começamos juntos pelo agente de relatórios. Eu tirei do papel em uma semana e coloquei para rodar. No Quasar acrescentei os gatilhos internos e o uso como agente do próprio sistema. Ele lê o sistema com o acesso de quem pergunta, responde com os números de agora e leva até a ação: lista, mensagem, mover, encaminhar, fila, meta. Toda ação sai como proposta. Ele também guia a pessoa, clique a clique, até a página certa. Origem, planos e detalhes em [docs/06-algory](docs/06-algory/).
- **Missões guiadas.** São 10 missões que levam o visitante pelo ciclo inteiro, do RG lido pelo OCR ao dinheiro na conta.
- **Integração com o Flux.** O comercial (Quasar) e a operação (Flux) conversam por eventos assinados sobre a mesma empresa fictícia. Uma fidelidade vencendo no Flux vira um lead de renovação no Quasar. Detalhes em [docs/07-integracao-flux](docs/07-integracao-flux/).
- **Apresentação** em https://demoquasar.osmio.tech/apresentacao, que já entra na demo com o papel escolhido.

## Números

| | |
|---|---|
| Código | ~76 mil linhas (backend, schema de 140 tabelas, interface) |
| Rotas | 462 no produto, 65 da demo |
| Algory | 55 casos de teste (37 de uso, 18 ataques); com o modelo real, 31 de 31 casos de uso e 18 de 18 ataques |
| Testes | 7 missões de ponta a ponta (Playwright), cliques aleatórios no painel do Algory, carga sem erro até 300 clientes simultâneos |
| Publicação | 48 pacotes de correção cumulativos; cada um é aplicado duas vezes num ambiente de teste, comparado arquivo a arquivo e vai para o servidor com backup |

## Custo da IA

A demo é pública e quem paga a IA sou eu. O uso de tokens é tratado como gestão financeira:
- verba fixa de US$ 5 por mês;
- limite de perguntas por visitante;
- modelos gratuitos de reserva;
- o banco responde sozinho tudo o que dá para responder sem modelo.

Explico a conta em [Algory › Gestão dos tokens](docs/06-algory/#gestão-dos-tokens).

## Segurança

Antes de abrir ao público, fiz uma revisão com sondas ao vivo e corrigi os cinco achados. A revisão e a publicação na VPS (isolamento, Docker rootless, firewall, backup) estão em [docs/09-seguranca](docs/09-seguranca/).

## Autores

Morigan Ventrici, autor, com Matheus Ventrici · mail@osmio.tech
