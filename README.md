# Quasar · demo pública

**Quasar** é um ERP comercial para captação e conversão de clientes: do lead que chega pelo anúncio até o contrato assinado e o primeiro pagamento. Este repositório contém a **demonstração pública** do sistema, com dados fictícios e provedores simulados.

🔗 **Demo ao vivo:** https://demoquasar.osmio.tech
📄 [Case](CASE.md) · [Guia da demo e missões](docs/DEMO.md) · [Documentação técnica](docs/DOCUMENTACAO_TECNICA.md) · [Publicação segura na VPS](docs/CASE.md)

![Pipeline](docs/prints/03_pipeline_kanban.png)

## O que tem aqui

| | |
|---|---|
| **Pipeline** | kanban em 4 fases e 11 etapas, travas por etapa, parados por dias sem toque, listas de trabalho com objetivo e progresso |
| **Ficha do cliente** | OCR, contrato e termo em PDF gerados no servidor, assinatura eletrônica (FluxSign), cobrança Pix/boleto/cartão parcelado (FluxPay), tickets, histórico |
| **Distribuição** | fila por peso e crédito por consultor, com prévia antes de entregar |
| **Contatos** | WhatsApp com janela de 24 h e modelos aprovados; mostra o payload exato da API da Meta |
| **Gestão e Relatórios** | ranking, metas, financeiro, funil e time, ao vivo |
| **Algory** | assistente que lê o sistema com o seu acesso, responde com os números de agora e leva até a ação, sempre com confirmação da pessoa |

## Como usar a demo

1. Abra https://demoquasar.osmio.tech. Não há login: você entra como **Renata** (supervisora) e pode trocar para **Carla** (consultora) ou **Davi** (dono) no perfil.
2. Siga as **10 missões** da tela de boas-vindas, ou peça ao Algory: *"Por onde eu começo?"*.
3. Tudo o que você fizer é real dentro da demo e **volta ao estado de fábrica a cada hora**.

O que é real: pipeline, regras de etapa, distribuição, documentos, relatórios, auditoria (é o código de produção).
O que é simulado: assinatura e pagamento (provedores simulados com os mesmos webhooks da produção).
O que fica de fora: telefonia e WhatsApp de verdade.

## Onde a demo roda

Só em **https://demoquasar.osmio.tech**. Este repositório não contém o código executável nem instruções para subir o sistema: ele documenta o projeto (case, guia da demo, documentação técnica, revisão de segurança e prints). Para ver o Quasar funcionando, use o site.

## Políticas do projeto

- **Licença:** todos os direitos reservados. O conteúdo pode ser lido para avaliação; **qualquer uso, cópia ou reprodução exige autorização por escrito** dos participantes do projeto. Veja [LICENSE](LICENSE).
- **Dados:** tudo na demo é fictício. Não envie dados reais; o sistema é apagado de hora em hora.
- **Segurança:** achou algo? Abra uma *issue* privada ou fale com o autor. A revisão de segurança feita antes da publicação está no [case](CASE.md#5-segurança). Não teste a demo com ferramentas de carga: há limites por IP.
- **Contribuições:** o repositório é de portfólio e não aceita *pull requests*. Sugestões por *issue* são bem-vindas.
- **Código-fonte:** não é publicado. Avaliações técnicas do código podem ser combinadas no particular.

## Stack

Python 3.12 · FastAPI · PostgreSQL 16 · WeasyPrint · HTML/CSS/JS sem framework · Docker rootless · nginx · Groq (gpt-oss) e Gemini para o Algory · Playwright nos testes.

## Autores

**Morigan Ventrici** — concepção, produto, backend, interface, Algory, infraestrutura e segurança.
**Matheus Ventrici** — colaboração no projeto.

© 2026 Morigan Ventrici e Matheus Ventrici. Todos os direitos reservados.
