# DemoQuasar · guia do visitante

🔗 **https://demoquasar.osmio.tech**

Sem login. Você entra como **Renata Dias** (supervisora). No seu perfil, no canto inferior esquerdo, dá para trocar para **Carla Mendes** (consultora: vê só a própria carteira) ou **Davi Nunes** (dono: vê dinheiro, ROI e o funil da empresa). Tudo é fictício e **volta ao começo a cada hora**; o seu progresso nas missões fica salvo no navegador.

Se a demo pedir um **código de acesso**, peça a quem compartilhou o endereço.

## A empresa da demo

**Demo Corporation**, provedora de internet por fibra com cerca de 520 assinantes. Vende planos mensais, add-ons e serviços de instalação. O Quasar é o comercial dela: capta o lead, conduz a venda, assina o contrato e recebe o primeiro pagamento.

## O que tem em cada tela

| Tela | O que você encontra |
|---|---|
| **Boas-vindas** | as 10 missões, seu nível e conquistas, os três papéis, o que é real e o que é simulado, o contador do próximo reset |
| **Home** | agenda dos próximos dias, carteira ativa, leads esfriando, meta do mês, recado da gestão |
| **Pipeline › Kanban** | 4 fases (Novo lead, Atendimento, Negociação, Onboarding) e 11 etapas; arraste um card para mover; as travas dizem o que falta |
| **Pipeline › Listas** | listas de trabalho com objetivo, prazo e progresso automático; o Algory monta a lista a partir de um pedido em texto |
| **Pipeline › Distribuição** | aquecimento (leads sem dono), fila por peso de cada consultor, prévia e entrega |
| **Ficha do cliente** | Visão geral · Atendimento · Cliente (OCR) · Endereços · Documentos (PDF) · Assinatura (FluxSign) · Pagamento (FluxPay) · Tickets · Histórico |
| **Contatos** | WhatsApp: conversas em aberto, janela de 24 h, modelos aprovados, o payload da API da Meta que seria enviado |
| **Gestão** | ranking, metas, carteira e esfriando por pessoa |
| **Relatórios** | financeiro, funil e time, com período e filtros; o Algory em tela cheia faz contas com os mesmos filtros |
| **FluxPay** | cobranças por estado (aberta, paga, recusada, vencida), líquido depois da taxa |
| **FluxSign** | envelopes por estado, tempo até assinar, vencidos para reenviar |
| **Algory** | botão no topo. Lateral, tela cheia, cartão, guia ou ícone; perguntas por assunto da tela; cada caminho termina numa ação que você revisa antes de confirmar |

## As 10 missões

| # | Missão | Onde | O que você faz | Pontos |
|---|---|---|---|---|
| 1 | Conheça o pipeline | Pipeline | abra o Pipeline, recolha uma fase, abra a ficha de um cliente | 10 |
| 2 | Ficha sem digitar | Ficha › Cliente | leia o RG pelo OCR, simule uma falha na leitura, corrija e salve | 15 |
| 3 | Contrato em um clique | Ficha › Documentos | gere o contrato e o termo, abra uma versão | 15 |
| 4 | Assinado pelo cliente | Ficha › Assinatura | veja o convite, envie, assine como se fosse o cliente (o retorno vem por webhook) | 20 |
| 5 | Dinheiro na conta | Ficha › Pagamento | escolha os itens, gere em 2x ou mais no cartão, pague como cliente | 20 |
| 6 | Quando dá errado | Ficha › Pagamento | provoque uma recusa, cancele e troque o método, receba no Pix | 15 |
| 7 | Peça ao Algory | Algory | pergunte quantos leads estão na fila e confirme a distribuição (é da supervisão) | 20 |
| 8 | Distribua o aquecimento | Pipeline › Distribuição | monte a fila pelo peso, confira lead a lead e entregue | 20 |
| 9 | Lista com objetivo | Pipeline › Lista | abra uma lista do time, crie a sua com objetivo e prazo | 15 |
| 10 | Leia os números | Relatórios | abra o funil, veja os valores, compare o time | 10 |

As missões 3 a 6 dependem da anterior (não dá para assinar sem contrato nem cobrar sem assinatura). Se travar, pergunte ao Algory *"Como faço 'Dinheiro na conta'?"*: ele leva até a primeira missão livre e aponta cada passo.

## Perguntas que valem a pena fazer ao Algory

- *"Por onde eu começo?"* · *"O que é o Quasar?"* · *"Mostre seus modos"*
- *"Quem tem mais lead parado?"* → termina em **Reencaminhar os parados de …**
- *"Quantas conversas estão em aberto?"* (em Contatos) → termina em **Montar a lista** para responder
- *"O que tenho na agenda?"* (na Home) → **Abrir a ficha** do próximo compromisso
- *"Um cliente"* → busca por nome, telefone ou #id e mostra o dossiê: etapa, conversa, documentos, contrato, pagamento e o que falta para avançar
- *"Monte uma lista dos frios há 5 dias para o time trabalhar até sexta"*
- *"Mude a meta da empresa para 80 mil"* (como Renata) → vem com o botão **Voltar para R$ 60 mil**

Limites: 60 pedidos por IP por hora; quando a cota do modelo acaba, o Algory responde pelas regras do servidor.

## O que é real e o que é simulado

- **Real** (código de produção): pipeline, travas, distribuição, documentos em PDF, relatórios, auditoria, regras de cargo.
- **Simulado**: assinatura e pagamento. Os simuladores respondem pelos mesmos webhooks assinados da produção; "pagar como cliente" abre a página do simulador.
- **Fora**: telefonia e envio real de WhatsApp. O envio mostra o payload exato que iria para a API da Meta.

## Avisos

- Não coloque dados reais: tudo é apagado de hora em hora e a demo é pública.
- Limites por IP estão ligados. Teste de carga, não.
- Achou um problema? Abra uma *issue* ou fale com o autor.

© 2026 Morigan Ventrici e Matheus Ventrici. Todos os direitos reservados.
