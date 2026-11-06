# NɅCO

> Pronuncia-se **NACO**.

## Sobre o projeto

O NɅCO é um projeto de código aberto voltado para a área alimentícia. Inspirado em plataformas como o iFood, tem como objetivo oferecer praticidade: permitir que o usuário descubra restaurantes, consulte produtos e realize pedidos demonstrativos de forma simples e organizada.

O NɅCO nasceu com o objetivo de ser muito mais do que um simples aplicativo. Meu nome é Gabriel, sou estudante de Ciência da Computação e apaixonado por tecnologia. Criei este projeto para aplicar e aprofundar meus conhecimentos conforme avanço na minha formação e na minha carreira, além de contribuir com a comunidade por meio de uma solução aberta, que possa ser estudada, adaptada e utilizada como referência em outros projetos.

Mais do que entregar funcionalidades, o NɅCO funciona como um laboratório de aprendizado em engenharia de software, no qual pretendo evoluir gradualmente a arquitetura, a qualidade do código, a segurança, os testes e a experiência de uso.

## Objetivos

- Desenvolver uma aplicação de pedidos de comida com experiência mobile para clientes e painel web para gestão de restaurantes.
- Aplicar conhecimentos de desenvolvimento full stack e engenharia de software em um projeto evolutivo.
- Praticar arquitetura, modelagem de dados, construção de APIs, testes automatizados, segurança e observabilidade.
- Manter o projeto aberto para estudo, colaboração e possíveis adaptações pela comunidade.

## Visão geral

O NɅCO é composto por três partes principais:

| Componente | Responsabilidade |
|---|---|
| Aplicativo mobile | Experiência do cliente: descoberta de restaurantes, consulta de catálogo, carrinho, checkout demonstrativo e acompanhamento de pedidos. |
| Aplicação web | Área de gestão para responsáveis por restaurantes, incluindo catálogo, pedidos e indicadores. |
| Backend / API | Regras de negócio, autenticação, autorização, validações, persistência e comunicação entre as aplicações. |

As aplicações mobile e web compartilham o mesmo backend. A arquitetura inicial é um **monólito modular**, evitando a complexidade prematura de microsserviços e mantendo os módulos organizados para permitir evolução futura quando houver necessidade técnica comprovada.

## Funcionalidades previstas

### Cliente
- Cadastro, autenticação e gerenciamento dos próprios dados.
- Exploração e consulta de restaurantes.
- Consulta de categorias e produtos.
- Pesquisa e filtros de catálogo.
- Carrinho local, com itens de um único restaurante por vez.
- Checkout e criação de pedidos demonstrativos.
- Acompanhamento do estado do pedido e consulta ao histórico.
- Avaliação de produtos adquiridos em pedidos concluídos.

### Responsável pelo restaurante
- Cadastro e manutenção de restaurante, conforme permissões.
- Gerenciamento de produtos, preços, imagens, descrições, categorias e disponibilidade.
- Consulta e gerenciamento de pedidos recebidos.
- Atualização dos estados dos pedidos conforme o fluxo permitido.
- Consulta de indicadores do estabelecimento.

### Administração
- Gestão administrativa de contas, papéis e solicitações de acesso.
- Ações administrativas protegidas e registradas em auditoria.

> A implementação deve respeitar os requisitos e as regras de negócio documentados. A lista acima apresenta a visão funcional do produto, não uma declaração de que todas as funcionalidades já estejam implementadas.

## Escopo e limitações do MVP

Pedidos e entregas têm finalidade demonstrativa. O escopo atual não contempla:

- Processamento de pagamentos reais ou integração com gateway financeiro.
- Operação real de entregadores, despacho ou rastreamento por GPS.
- Promessas de prazo real de entrega.
- Emissão de documentos fiscais.
- Programa de fidelidade, cupons ou recomendações automatizadas.
- Autenticação multifator (MFA).

Esses itens podem ser reconsiderados em etapas futuras, mas não fazem parte do MVP documentado.

## Regras de negócio importantes

- Cada pedido pertence a um único restaurante.
- A criação do pedido não reserva nem reduz o estoque.
- O estoque é reduzido na confirmação do pedido pelo restaurante.
- O cancelamento de um pedido já confirmado restaura o estoque uma única vez.
- Um produto sem estoque permanece identificável como indisponível e não pode ser adicionado ao carrinho.
- O histórico do pedido preserva os dados relevantes do momento da compra, mesmo que o catálogo seja alterado posteriormente.
- Avaliações são vinculadas a itens de pedidos concluídos e pertencentes ao cliente autenticado.
- Operações de gestão exigem autenticação, papel adequado e autorização sobre o restaurante correspondente.

O fluxo principal de estados do pedido é:

```text
CRIADO
  → CONFIRMADO_RESTAURANTE
  → EM_PREPARO
  → SAIU_PARA_ENTREGA
  → CONCLUIDO
```

Estados de recusa e cancelamento seguem as regras de transição e permissões definidas na documentação técnica.

## Arquitetura e tecnologias

A stack registrada para o MVP é:

| Camada | Tecnologia |
|---|---|
| Aplicativo mobile | React Native + TypeScript |
| Navegação mobile | React Navigation |
| Aplicação web | React + TypeScript |
| Backend | Node.js 26 LTS + Express + TypeScript (modo estrito) |
| API | REST / HTTP |
| Banco de dados | PostgreSQL |
| ORM e migrations | Prisma |
| Cache e controles distribuídos | Redis |
| Validação de dados | Zod |
| Contrato e documentação da API | OpenAPI |
| Comunicação HTTP nas interfaces | Axios |
| Armazenamento de imagens | Cloudinary, por meio de uma abstração de storage |
| Testes de backend | Vitest + Supertest |
| Testes de interface | React Testing Library |
| Testes mobile ponta a ponta | Appium |
| Testes de carga | Grafana k6 |
| Ambiente local | Docker Compose |
| CI | GitHub Actions |

As versões exatas das dependências devem ser fixadas nos arquivos de configuração do projeto. Node.js 26 LTS é a linha de execução prevista para o backend.

## Qualidade e engenharia

A evolução do projeto considera práticas de engenharia de software desde o MVP:

- Monólito modular, com separação por domínio e responsabilidades.
- Validação de entradas nas fronteiras do sistema.
- Autorização aplicada no backend, sem confiar em papéis enviados pelo cliente.
- Transações para operações que alteram conjuntamente pedido, estoque e histórico.
- Idempotência na criação de pedidos para evitar duplicidade em retentativas.
- Tratamento consistente de erros HTTP.
- Testes unitários, integração, API, interface, segurança, concorrência e carga.
- Pipeline de integração contínua com lint, verificação de tipos, testes e build.
- Logs estruturados, métricas, rastreamento e rotinas de backup conforme a etapa de implantação.

## Documentação

A documentação técnica e funcional do NɅCO está organizada na pasta `docs/`, separada por categorias para facilitar a consulta e a manutenção.

Consulte o [Índice da documentação](docs/README.md) para acessar:

- Especificação de requisitos e decisões complementares.
- Arquitetura, modelo de dados e contratos da API.
- Registro de decisões técnicas.
- Planejamento e cronograma de desenvolvimento.
- Estratégia de testes e relatórios de auditoria.

A documentação é mantida separadamente do README principal para facilitar sua evolução sem sobrecarregar a apresentação do projeto.

## Estrutura do repositório

- `apps/`: aplicações mobile, web e API.
- `packages/`: contratos e componentes compartilhados.
- `docs/`: requisitos, arquitetura, decisões, planejamento, testes e contratos.
- `infra/`: infraestrutura local e observabilidade.
- `scripts/`: automações de desenvolvimento e manutenção.
- `.github/`: workflows de integração contínua.

Consulte [`docs/README.md`](docs/README.md) para o índice da documentação.

## Situação do projeto

O NɅCO está em fase de preparação e implementação evolutiva. A documentação registra as decisões de produto e arquitetura que orientam o MVP, mas isso não significa que todas as funcionalidades estejam concluídas, testadas ou prontas para produção.

A implementação deve avançar em incrementos pequenos e verificáveis. Antes de congelar contratos definitivos, eventuais conflitos e pendências identificados nos documentos devem ser resolvidos e refletidos na especificação e na OpenAPI.

## Como contribuir

O projeto é aberto a estudo, sugestões e contribuições. Antes de propor mudanças:

1. Consulte os documentos de requisitos e decisões.
2. Verifique se a alteração preserva as regras de negócio existentes.
3. Descreva o problema e a solução proposta.
4. Inclua ou atualize testes compatíveis com o comportamento alterado.
5. Atualize a documentação quando houver impacto em requisitos, contratos ou arquitetura.

## Licença

A licença de código aberto deve ser definida e incluída no repositório antes da publicação de uma versão que estabeleça formalmente os termos de uso e contribuição.
