# NɅCO — Especificação consolidada de requisitos

**Base documental:** `README(2).MD`, `writing-block.md`, modelo de dados revisado e decisões registradas no histórico do formulário.  
**Escopo desta revisão:** consolidação documental. Nenhuma implementação de código foi iniciada ou modificada. Decisões complementares do MVP (versão 1.2) e decisões Q63–Q103 de imagens foram propagadas aos documentos-base; OpenAPI, schema físico, migrations e implementação permanecem como etapas futuras.

## 1. Convenções de leitura e origem

- **Original:** requisito, regra, comportamento, endpoint ou restrição presente na documentação original.
- **Decisão aprovada:** resposta confirmada no formulário/histórico e incorporada à especificação.
- **Complementação técnica aprovada:** decisão identificada por QU-01 a QU-73, que detalha como validar ou proteger comportamento já existente. Não é um RF/RN adicional.
- **Pendente:** dado ou comportamento que não foi decidido. Não deve ser presumido na implementação.
- **Decisões Q63–Q103 de imagens:** trilha independente das decisões `QU-01` a `QU-73` do RNF-01; não misturar os identificadores.
- **Impacto sujeito a aprovação:** decisão aprovada que exige alteração ou esclarecimento em outro contrato/regra. O impacto é documentado e propagado nos documentos diretamente afetados.

As sugestões marcadas como exemplo, hipótese ou opção na documentação de arquitetura não foram promovidas a decisões finais, salvo quando posteriormente consolidadas no adendo complementar versão 1.2.

## 2. Visão e escopo do produto

O NɅCO permite que clientes explorem restaurantes e produtos, simulem pedidos e acompanhem estados demonstrativos. Restaurantes podem administrar seus dados, catálogo, preços e estoque e consultar indicadores disponibilizados pela plataforma.

A experiência descrita é mobile para clientes e web para gestão/indicadores/estoque dos restaurantes. Os pedidos e entregas são demonstrativos.

### 2.1. Fora do escopo inicial preservado

- Gateway ou processamento de pagamento real.
- Autenticação de dois fatores.
- Integração operacional com entregadores, despacho real ou rastreamento por GPS.
- Promessa de prazo de entrega real.
- Emissão de documentos fiscais.
- Programa de fidelidade, cupons ou recomendações automatizadas.

## 3. Requisitos funcionais consolidados

Os identificadores originais foram preservados. Os detalhes abaixo consolidam decisões aprovadas, sem acrescentar funcionalidades não previstas.

| ID | Requisito consolidado | Critério verificável principal | Dependências |
|---|---|---|---|
| RF-01 | Permitir cadastro e autenticação de conta única por pessoa, com e-mail globalmente único e papel `CLIENTE` automático. A sessão usa access JWT de 15 minutos e refresh token opaco/rotativo com validade máxima de 30 dias. | Cadastro válido cria conta e CLIENTE; e-mail duplicado é recusado; login/refresh/logout respeitam expiração, rotação, revogação e proteção contra reutilização de refresh. | `User`, `UserRole`, sessão; `/api/auth/register`, `/api/auth/login`, `/api/auth/refresh`, `/api/auth/logout`. |
| RF-02 | Permitir avaliação **somente de produtos** adquiridos pelo cliente, associada a um `OrderItem` de pedido próprio concluído. É permitida uma avaliação por item do pedido; comentário é opcional e nota varia de 1 a 5. As avaliações e comentários publicados são consultáveis por qualquer visitante, sem autenticação. | Cliente autenticado, dono do pedido, consegue avaliar item após `CONCLUIDO`; segunda avaliação para o mesmo item é bloqueada; visitantes conseguem consultar avaliações públicas sem dados pessoais do autor. | `Review`, `OrderItem`, `Order`, `User`; criação autenticada e leitura pública. |
| RF-03 | Permitir simulações de pedido e de entrega, com um pedido por restaurante, status inicial `CRIADO` e transições previamente definidas. Não há pagamento/entrega real. | Pedido válido contém itens de um restaurante, fica em `CRIADO`, é exibido no histórico/painel e percorre apenas transições permitidas; estados finais permanecem no histórico. | `Order`, `OrderItem`, `OrderEvent`, `Restaurant`, `RestaurantProduct`; regras de estoque, horário, publicação, autenticação e idempotência. |
| RF-04 | Permitir que responsáveis autorizados consultem indicadores de produtos vendidos e avaliações recebidas por restaurante. Produtos vendidos são a soma de quantidades em pedidos `CONCLUIDO`; avaliações são contadas pela data de criação. Período padrão de 30 dias, filtros de até 365 dias no timezone do restaurante; sem faturamento oficial. | Responsável autorizado consulta apenas o restaurante indicado; fórmulas, período default e limite máximo são consistentes e auditáveis. | `Restaurant`, `Order`, `OrderItem`, `Review`, `RestaurantProduct`; `GET /api/restaurant/dashboard?restaurantId={restaurantId}`. |
| RF-05 | Permitir cadastro, edição e desativação/exclusão lógica de ofertas no restaurante autorizado, preservando referências históricas. | Operações autorizadas alteram o catálogo; oferta já referenciada não perde histórico; categoria pertence ao mesmo restaurante. | `ProductBase`, `RestaurantProduct`, `Category`, `OrderItem`; rotas de produto existentes e futura consolidação de gestão de categorias. |
| RF-06 | Permitir que uma conta única solicite e receba o papel RESPONSAVEL após aprovação administrativa. Uma solicitação `PENDING` por usuário; após rejeição, cooldown de 7 dias. `ADMIN` é único, exclusivo e criado manualmente, sem cadastro público. | Solicitação, aprovação/rejeição, concessão/remoção e transferência de restaurantes são auditadas; remoção de RESPONSAVEL é bloqueada até transferência de todos os seus restaurantes. | `User`, `UserRole`, `RoleRequest`, `RoleAuditEvent`; rotas `/api/me/role-requests` e `/api/admin/...`. Schema físico ainda será implementado. |
| RF-07 | Permitir cadastro/manutenção do restaurante pelo responsável autenticado. Um responsável pode gerenciar vários restaurantes; cada restaurante tem exatamente um responsável. Novo restaurante começa `UNPUBLISHED`; `timezone` IANA e grade semanal de horários definem funcionamento separadamente da publicação. | Criar restaurante usa responsável da sessão; horários são avaliados no fuso do restaurante, com virada de meia-noite; despublicação é bloqueada com pedidos ativos; gestão permanece acessível quando despublicado. | `User`, `UserRole`, `Restaurant`, `Order`; `opening_hours` e `timezone` definidos no adendo; rotas de gestão existentes. |
| RF-08 | Controlar preço e estoque de cada oferta de produto por restaurante. Preço deve ser maior que zero e estoque não pode ser negativo. Produto com estoque zero permanece visível, mas indisponível para compra. O estoque é reduzido ao confirmar pedido e restaurado ao cancelar pedido já confirmado, em operações atômicas. | Preço zero/negativo e estoque negativo são rejeitados; estoque não baixa na criação do pedido; confirmação baixa a quantidade correta; cancelamento elegível após confirmação restaura uma única vez. | `RestaurantProduct`, `Order`, `OrderItem`, `OrderEvent`; concorrência e transações do banco. |
| RF-09 | Permitir alterar preço, imagem, nome, descrição e categoria da oferta. `ProductBase` é identidade compartilhável; `RestaurantProduct` guarda `display_name` próprio e `normalized_name` calculado no servidor. | Alteração atualiza catálogo corrente, mas não snapshots; não há duplicidade no mesmo restaurante após normalização NFKD, sem marcas diacríticas, trim, espaços consecutivos e case-insensitive. | `ProductBase`, `RestaurantProduct`, `Category`, `OrderItem`; constraint única `(restaurant_id, normalized_name)` será materializada no schema físico. |

## 4. Regras de negócio consolidadas

| ID | Regra consolidada | Estado |
|---|---|---|
| RN-01 | Cada pedido/carrinho operacional deve conter produtos de um único restaurante. | Aprovada; pedido de vários restaurantes não permitido. |
| RN-02 | Um restaurante pode ter apenas uma oferta com o mesmo nome normalizado. | Decidida: `RestaurantProduct.display_name` + `normalized_name` calculado no servidor; unicidade `(restaurant_id, normalized_name)`, inclusive ofertas desativadas. |
| RN-03 | Responsável só gerencia produtos/recursos associados a restaurante para o qual possui autorização. | Aprovada e detalhada por associação do usuário ao restaurante. |
| RN-04 | Cada conta só pode atualizar os próprios dados pessoais, salvo ações administrativas previstas na matriz. | No MVP, `GET/PATCH /api/me`; somente `name` é editável. E-mail é identificador de login e não pode ser alterado neste corte. |
| RN-05 | Avaliação `PRODUCT_ONLY`: item de pedido próprio concluído, uma por `order_item_id`, nota 1–5 e comentário opcional; leitura pública sem login com autor “Cliente verificado”. | Aprovada; `POST /api/order-items/{orderItemId}/reviews` e `GET /api/products/{productId}/reviews`; rota antiga de review de restaurante substituída. |
| RN-06 | Não é permitido adicionar produto sem estoque ao carrinho. Produto com estoque zero continua visível no catálogo, com estado indisponível. | Aprovada. Estoque não é reservado/baixado na criação do pedido; baixa ocorre na confirmação. |
| RN-07 | Produto excluído não deve aparecer como opção para novos pedidos. Produtos já referenciados em pedidos devem ser desativados ou ter exclusão física restringida, preservando o histórico. | Aprovada. |
| RN-08 | Adicionar produto ao carrinho exige autenticação e restaurante aberto; criar pedido exige restaurante aberto e publicado. | Aprovada; timezone IANA por restaurante e grade semanal por dia ISO com intervalos `HH:mm`; sem feriados/exceções no MVP. |
| RN-09 | Ao remover/alterar item do carrinho, atualizar subtotal e total. | Aprovada; carrinho permanece local no app mobile, um restaurante por vez; sem entidade ou API de carrinho no MVP. |

### 4.1. Conta, papéis e matriz de permissões

| Ação | Visitante | Cliente autenticado | Responsável autorizado | ADMIN |
|---|---:|---:|---:|---:|
| Consultar restaurantes e produtos públicos | Sim | Sim | Sim | Sim |
| Criar conta / iniciar sessão | Sim | Sim | Sim | Sim, conforme manutenção manual da conta |
| Atualizar dados pessoais próprios | Não | Sim | Sim | Sim |
| Criar avaliação | Não | Sim | Não | Não |
| Criar pedido simulado | Não | Sim | Não | Apenas em ambiente de teste |
| Criar/editar produtos | Não | Não | Apenas no restaurante autorizado | Sim, mediante autorização prévia |
| Excluir/desativar produtos | Não | Não | Apenas no restaurante autorizado | Sim, mediante autorização prévia |
| Consultar indicadores do restaurante | Não | Não | Apenas no restaurante autorizado | Sim, mediante autorização prévia |
| Moderar contas ou conteúdo | Não | Não | Não | Sim |

O escopo de ADMIN é `FULL` dentro da matriz, não uma autorização irrestrita para qualquer ação ou recurso. As ações sujeitas a autorização prévia continuam condicionadas a procedimento ainda não especificado. Operações administrativas relevantes são auditáveis conforme o fluxo aprovado.

- Uma pessoa possui uma única conta `User`, com e-mail globalmente único e senha pertencente à conta.
- Papéis são separados em `UserRole`, com chave composta (`user_id`, `role`).
- Uma conta pode combinar `CLIENTE` e `RESPONSAVEL`.
- `ADMIN` é exclusivo: não combina com `CLIENTE` ou `RESPONSAVEL`.
- O NɅCO terá uma única conta `ADMIN`, criada/manutenida manualmente no banco pelo desenvolvedor; não existe cadastro público de ADMIN. A senha deve ser armazenada como hash.
- `CLIENTE` é atribuído automaticamente no cadastro público.
- `RESPONSAVEL` depende de solicitação (`REQUEST`) e aprovação administrativa.
- Após rejeição, o usuário aguarda 7 dias antes de nova solicitação.
- Remoção do papel `RESPONSAVEL` é realizada por ADMIN.
- Concessões/remoções/aprovações/rejeições devem ser auditadas.
- Há no máximo uma `RoleRequest` PENDING por usuário; a criação também gera evento `REQUEST_CREATED`. Ao remover RESPONSAVEL, ADMIN deve transferir os restaurantes para outro usuário RESPONSAVEL; a remoção sem transferência é bloqueada. Os dados não são apagados automaticamente.
- ADMIN tem escopo administrativo `FULL` limitado à matriz. Para ações sujeitas à autorização prévia do proprietário do projeto, a autorização ocorre fora do produto; a operação deve registrar ator, alvo, ação, justificativa e data/hora. O procedimento operacional/evidência deve ser documentado antes de executar a ação sensível.

### 4.2. Restaurante

- Um responsável pode gerir vários restaurantes; cada restaurante possui exatamente um responsável.
- Responsável pode existir sem restaurante.
- Cada restaurante tem CNPJ único.
- Cadastro não depende de aprovação administrativa de restaurante; o próprio responsável decide publicar/despublicar.
- Novo restaurante começa `UNPUBLISHED`.
- `publication_status` não equivale a `operating_status`.
- Restaurante publicado e fechado pode aparecer em consultas públicas, mas não aceita novos pedidos.
- Despublicação é bloqueada quando há pedidos em `CRIADO`, `CONFIRMADO_RESTAURANTE`, `EM_PREPARO` ou `SAIU_PARA_ENTREGA`.
- Despublicar não apaga produtos, pedidos ou histórico; responsável mantém acesso ao painel e gerenciamento.

### 4.3. Produto, categoria e oferta

- `ProductBase` representa um produto compartilhável entre restaurantes.
- `RestaurantProduct` representa a oferta específica e contém, entre outros, restaurante, produto-base, categoria, preço, estoque, referência `object_key` da imagem e status. O PostgreSQL não armazena URL pública.
- Imagens são armazenadas e distribuídas pelo Cloudinary via `StoragePort`; upload/substituição/exclusão passam pelo backend, sem credenciais no cliente.
- Entradas JPEG/PNG/WebP estáticas, até 5 MB e 40 MP; validar extensão, MIME, assinatura binária e decodificação; remover EXIF, converter para sRGB/WebP, preservar transparência e redimensionar até 1600×1600 sem upscale.
- Cache público de um ano com `immutable`; cada substituição usa nova chave imutável. Falhas não substituem a imagem anterior.
- O detalhamento operacional de upload, reconciliação, invalidação CDN e exclusão está em `05_ARMAZENAMENTO_E_PROCESSAMENTO_DE_IMAGENS.md`.
- Cada restaurante possui suas próprias categorias.
- `RestaurantProduct` só pode referenciar `Category` do mesmo `restaurant_id`.
- Preço > 0; estoque >= 0.
- Estoque zero não remove o produto do catálogo, mas impede compra.
- Exclusão não pode invalidar `OrderItem` existente.

### 4.4. Pedido, estados e estoque

Estados canônicos de domínio/persistência:

`CRIADO`, `CONFIRMADO_RESTAURANTE`, `EM_PREPARO`, `SAIU_PARA_ENTREGA`, `CONCLUIDO`, `RECUSADO`, `CANCELADO`.

A grafia canônica aprovada é `CONCLUIDO` (sem acento) no enum/API/persistência; a interface pode apresentar “Concluído”.

Transições principais:

- `CRIADO` → `CONFIRMADO_RESTAURANTE` (responsável autorizado aceita; valida estoque e reduz em transação atômica).
- `CRIADO` → `RECUSADO` (responsável autorizado recusa; não baixa estoque).
- `CONFIRMADO_RESTAURANTE` → `EM_PREPARO`.
- `EM_PREPARO` → `SAIU_PARA_ENTREGA`.
- `SAIU_PARA_ENTREGA` → `CONCLUIDO`.
- Cancelamentos conforme a matriz abaixo.

| Estado atual | Cliente pode cancelar? | Responsável pode cancelar? |
|---|---:|---:|
| `CRIADO` | Sim | Não; pode recusar |
| `CONFIRMADO_RESTAURANTE` | Sim | Sim |
| `EM_PREPARO` | Sim | Não |
| `SAIU_PARA_ENTREGA` | Não | Não |
| `CONCLUIDO` | Não | Não |
| `RECUSADO` | Não | Não |
| `CANCELADO` | Não | Não |

A matriz foi aprovada e resolve a regra geral anterior que permitia cancelamento em qualquer etapa anterior à conclusão. Quando o cancelamento ocorrer após a baixa de estoque (a partir de `CONFIRMADO_RESTAURANTE`), status, evento e restauração do estoque devem ser consistentes e atômicos, sem restauração duplicada.

Cada pedido:
- pertence a um único restaurante;
- é criado por usuário autenticado com papel `CLIENTE`;
- inicia em `CRIADO`;
- tem pelo menos um item;
- tem total igual à soma dos subtotais;
- valida preço e disponibilidade no backend;
- aparece no painel do restaurante;
- recusado permanece no histórico do cliente.

Cada `OrderItem`:
- referencia produto ofertado do mesmo restaurante do pedido;
- registra snapshot de nome (`product_name`) e preço (`unit_price`);
- aceita quantidade de 1 a 20;
- não pode se repetir em outra linha com o mesmo `restaurant_product_id` no pedido (`UNIQUE(order_id, restaurant_product_id)`);
- subtotal é `unit_price × quantity`.

Cada alteração de status gera `OrderEvent` com pedido, status anterior/novo, usuário que realizou a alteração e timestamp.

### 4.5. Avaliação de produto

- Somente cliente autenticado e dono do pedido pode criar avaliação.
- Qualquer visitante, autenticado ou não, pode consultar avaliações públicas de um produto.
- A avaliação se refere a produto presente em `OrderItem` do próprio pedido.
- Somente após o pedido atingir `CONCLUIDO`.
- Uma avaliação por `order_item_id` (índice/restrição única lógica).
- `rating` de 1 a 5; `comment` opcional.
- Leitura pública sem autenticação; autor exibido como “Cliente verificado”, sem expor nome, e-mail, ID ou outros dados pessoais.
- Somente o cliente dono do pedido concluído pode criar a avaliação; unicidade por `order_item_id`.
- `Review` relaciona-se a `OrderItem`; não repetir `customer_id`, `order_id` ou `restaurant_id` na entidade, pois esses dados são recuperáveis pelo relacionamento.

## 5. Arquitetura, organização do frontend e estrutura de projeto preservadas

- A direção arquitetural registrada é iniciar com arquitetura monolítica e evoluir para microsserviços apenas quando a necessidade surgir, evitando complexidade desnecessária.
- A experiência prevista no escopo original é mobile para clientes e aplicação web para visualização de dados, indicadores e gestão de estoque pelos restaurantes.
- A comunicação prevista é por API REST sobre HTTP. A documentação menciona possibilidade futura de Server-Sent Events/websockets e mensageria em tempo real, mas não aprova sua inclusão no MVP.
- A organização original usa React Navigation para navegação mobile e Axios para comunicação com a API; dados como token, ID do usuário, restaurante e produto podem ser compartilhados entre telas conforme contexto.
- Formulários validam no frontend para experiência e novamente no backend para segurança/integridade. O documento original avalia Formik ou React Hook Form, sem escolha definitiva.
- Sessão aprovada: access JWT de 15 minutos; refresh token opaco, rotativo, validade máxima de 30 dias, somente hash persistido no backend; rotação a cada uso e revogação da família se houver reutilização; logout revoga sessão/refresh. Mobile usa access em memória + refresh em Keychain/Keystore; web usa refresh em cookie HttpOnly/Secure/SameSite=Lax e access em memória; tokens não ficam em localStorage/sessionStorage.
- Redis foi selecionado para cache/rate limiting. Cache-aside com TTL inicial de 60 s aplica-se a consultas públicas de restaurantes/catálogo; não cachear estoque, autorização ou pedidos como fonte de verdade; invalidar ao alterar restaurante/produto/categoria.
- Estrutura de backend proposta originalmente: `config`; `modules` para `users`, `restaurants`, `products`, `categories`, `orders` e `reviews`; `middlewares`; `lib`; `routes`; `tests/unit`, `tests/integration`, `tests/fixtures`; `prisma` e arquivos de configuração. O desenho de camadas separa `controller`, `service`, `repository`, `schema` e `routes`. Trata-se de organização proposta, não de código ou estrutura física já implementada.
- A stack do MVP foi consolidada no adendo complementar: React Native/TypeScript/React Navigation, React/TypeScript, Node.js 26 LTS + Express + TypeScript strict, PostgreSQL + Prisma, Redis, Zod, REST/OpenAPI, Axios, Cloudinary via `StoragePort`, Vitest/Supertest, React Testing Library, Appium, k6, Docker Compose e GitHub Actions. Esta escolha orienta a implementação; versões exatas serão fixadas nos manifests/CI.

### 5.1. Tecnologias e responsabilidades citadas na documentação original

| Área | Stack decidida para o MVP | Observação de implementação |
|---|---|---|
| Mobile | React Native + TypeScript + React Navigation + Axios | Cliente; carrinho local. |
| Web | React + TypeScript + Axios | Gestão do responsável e administração. |
| Backend | Node.js 26 LTS + Express + TypeScript strict | Monólito modular. |
| API/validação | REST/HTTP + OpenAPI + Zod | OpenAPI será criada antes do freeze do contrato executável. |
| Banco/ORM | PostgreSQL + Prisma | Schema e migrations ainda serão desenvolvidos. |
| Cache/rate limiting | Redis | Cache-aside para consultas públicas; rate limiting distribuído. |
| Autenticação | JWT access + refresh opaco/rotativo | Ciclo de vida definido na seção de segurança/adendo. |
| Imagens | Cloudinary via `StoragePort` | Plano Free sem cartão; sem upgrade/cobrança sem autorização explícita. |
| Testes | Vitest, Supertest, React Testing Library, Appium, Grafana k6 | CI com GitHub Actions. |
| Ambiente local | Docker Compose | PostgreSQL e Redis; `.env.example` sem segredos. |

---

## 6. Requisitos não funcionais

### 6.1. RNF-01 — Desempenho sob carga e validação de integridade

Requisito original: telas mais utilizadas devem responder em tempo aceitável sob carga de 1.000 usuários simultâneos. As decisões QU-01 a QU-73 tornam essa declaração mensurável e verificável.

#### Operações incluídas (QU-01)

Somente estas nove operações serão incluídas na distribuição oficial:
1. Listar/explorar restaurantes.
2. Consultar detalhes de restaurante.
3. Listar produtos.
4. Consultar detalhes de produto.
5. Login/autenticação.
6. Criar pedido.
7. Consultar pedidos do cliente.
8. Consultar pedidos recebidos pelo restaurante.
9. Consultar painel de gerenciamento.

O contrato foi consolidado: a operação 8 mede a listagem dos pedidos recebidos pelo responsável, por `GET /api/restaurants/{restaurantId}/orders` ou pela inbox consolidada `GET /api/me/orders/inbox`; visualização e transição são registradas por POST e PATCH separados.

#### Limites de latência e erros (QU-02, QU-03, QU-05)

| Classe | P95 máximo | P99 máximo |
|---|---:|---:|
| Operações comuns (consultas/listagens/autenticação/painel) | 2 s | 4 s |
| Criação de pedido | 3 s | 6 s |

Taxa máxima de erros técnicos: **1%**. Contabilizar HTTP 5xx, timeouts e falhas inesperadas. Recusas de negócio esperadas, como HTTP 409 por estoque insuficiente, não contam como erro técnico. HTTP 401 inesperado em fluxo autenticado deve ser investigado.

#### Carga e duração (QU-04, QU-06)

- Ramp-up progressivo até 1.000 usuários virtuais simultâneos: 10 minutos.
- Após atingir 1.000, manter carga sustentada por 60 minutos adicionais.
- Duração mínima do trecho oficial por execução: 70 minutos, sem contar preparação, aquecimento e encerramento.

#### Distribuição de requisições (QU-07, QU-08)

| Operação | Percentual de requisições |
|---|---:|
| Listar restaurantes | 20% |
| Detalhes de restaurante | 10% |
| Listar produtos | 20% |
| Detalhes de produto | 10% |
| Login/autenticação | 5% |
| Criar pedido | 5% |
| Pedidos do cliente | 10% |
| Pedidos recebidos pelo restaurante | 10% |
| Painel | 10% |
| **Total** | **100%** |

Os percentuais referem-se a requisições, não a usuários. O login não deve ser repetido a cada ação como se cada chamada fosse novo login; o gerador deve modelar sessão e ciclo de vida dos usuários virtuais.

#### Ambiente, recursos e dados (QU-09 a QU-13)

- Executar em staging/homologação isolado e representativo de produção.
- Registrar, em cada execução: CPU, RAM, instâncias da aplicação, banco/versão/recursos/conexões, sistema operacional, runtime/dependências, rede, gerador de carga, data, versão da aplicação e configuração do teste.
- Documentar recursos efetivamente usados e avaliar a representatividade do ambiente, não apenas preencher inventário.
- Preparar três volumes:

| Entidade | Pequeno | Médio | Grande |
|---|---:|---:|---:|
| Usuários | 500 | 5.000 | 20.000 |
| Restaurantes | 20 | 100 | 500 |
| Produtos/ofertas | 500 | 5.000 | 25.000 |
| Pedidos | 2.000 | 20.000 | 100.000 |
| Avaliações | 500 | 5.000 | 25.000 |

Os números são registros cadastrados, não usuários simultâneos. Preservar integridade relacional e de domínio.
- Distribuição dos estados de pedidos no conjunto de teste: `CRIADO` 10%; `CONFIRMADO_RESTAURANTE` 10%; `EM_PREPARO` 10%; `SAIU_PARA_ENTREGA` 10%; `CONCLUIDO` 45%; `RECUSADO` 5%; `CANCELADO` 10%. Criar os estados por transições válidas e temporalmente coerentes, nunca atribuindo estados aleatoriamente sem histórico.

#### Paginação, cache e padrão de interação (QU-14 a QU-16)

- Paginação padrão: 20 itens; máximo: 100 por página para listagens de restaurantes, produtos e pedidos.
- Testar primeira página e páginas profundas; o limite máximo deve ser rejeitado ou limitado de forma consistente. Ordenação estável com desempate único.
- Paginação não substitui índices adequados.
- Se a arquitetura tiver cache, executar/registrar cenários de cache frio e aquecido separadamente. Cache frio exige limpeza prévia; cache aquecido após preenchimento; registrar tempo de aquecimento e hit/miss. Não há obrigação de implementar cache nem escolher tecnologia.
- Entre ações de um mesmo VU, usar intervalo aleatório de 1 a 5 segundos. Isso não define RPS-alvo; registrar RPS realmente alcançado.

#### Repetições e aprovação da carga (QU-17 a QU-19, QU-20)

- Cinco execuções oficiais por combinação de volume e condição de cache: 3 volumes × 2 condições × 5 repetições = **30 execuções oficiais**.
- Armazenar resultados individualmente e comparar média, mediana, dispersão e percentis.
- Limites P95/P99 e taxa de erro valem igualmente em todos os volumes e condições de cache, inclusive cache frio.
- Cada uma das cinco execuções de cada combinação deve cumprir individualmente os critérios. Percentis devem ser calculados por operação dentro de cada execução; resultado agregado não pode mascarar operação lenta. Uma execução fora dos limites reprova a combinação.
- RPS efetivo deve ser medido/documentado: total e por operação, média/variação, concluídas, falhas, timeouts e relação com latência/VUs. Não foi definido RPS mínimo.

#### Disputa concorrente por estoque (QU-21 a QU-28)

Além da carga geral, executar testes específicos para disputa pela mesma oferta:

| Nível | Tentativas simultâneas | Estoque inicial (50% da demanda, arredondado para baixo) | Repetições |
|---|---:|---:|---:|
| 1 | 10 | 5 | 10 |
| 2 | 25 | 12 | 10 |
| 3 | 50 | 25 | 10 |
| 4 | 100 | 50 | 10 |

Cada tentativa representa uma unidade; estoque é restaurado antes de cada execução. Total: **40 execuções**. Barreira de sincronização libera requisições simultaneamente; registrar horário de liberação e início efetivo, pois sincronizar o gerador não garante início simultâneo no banco.

- Latência concorrente: P95 até 3 s e P99 até 6 s, medida da liberação da barreira até a resposta HTTP.
- Tentativas sem estoque devem receber HTTP 409; não podem consumir unidades. Recusas legítimas 409 não são erros técnicos.
- Após cada execução, reconciliar pedidos, respostas HTTP e estoque, verificando pedidos/unidades, sem duplicidade, estoque negativo ou alteração parcial.
- A disputa deve exercitar o momento real de baixa de estoque, isto é, a confirmação do pedido, e não alterar a regra aprovada de que a criação em `CRIADO` não baixa estoque.

#### Idempotência, falhas e recuperação (QU-29 a QU-39; QU-51 a QU-70)

- Reenviar a mesma solicitação com a mesma chave não pode duplicar pedido ou consumo; testar reenvios sequenciais e simultâneos.
- O cliente gera `Idempotency-Key` por intenção lógica. Nova intenção exige nova chave.
- Escopo de unicidade: combinação `user_id + idempotency_key`, garantida no banco e validada na aplicação.
- Payload é normalizado/canônico, com ordenação determinística de itens, e comparado por hash dos campos relevantes. Alterar payload com a mesma chave retorna HTTP 409 e não executa nova operação.
- Persistir a chave, usuário, hash do payload, estado do processamento e referência/resultado do pedido. Nome físico da tabela e detalhes de colunas ainda dependem do schema definitivo.
- Retenção dos dados completos da operação: 24 horas. Após expiração, manter registro mínimo/tombstone permanentemente enquanto o sistema existir; reuso da chave expirada é rejeitado. Nova intenção exige nova chave.
- Se uma segunda requisição simultânea usar a mesma chave, somente uma execução processa. A segunda aguarda até 5 segundos sem manter transação/bloqueio de banco aberto enquanto espera. Se a primeira concluir dentro do prazo, a segunda recebe o mesmo resultado; se não, recebe HTTP 202 Accepted e referência para consulta. A primeira continua processando.
- Endpoint de consulta aprovado: `GET /api/orders/idempotency/{key}`. Exige autenticação e valida titularidade da chave; não cria pedido nem consome estoque.
- Falhas devem ser injetadas antes, durante e depois de gravação do pedido/alterações pertinentes. Distinguir falha antes do commit de perda de resposta HTTP após commit.
- Rollback integral antes do commit; nenhuma alteração parcial. Se commit ocorreu e a resposta se perdeu, recuperar o resultado pela idempotência.
- Após reinicialização, validar consistência do banco, preservar transações confirmadas, descartar incompletas e recuperar operações pela chave.
- Evidência por execução: cenário/ponto de falha, horário, chave (sem expor segredo), respostas HTTP, logs correlacionados, estado pedido/estoque antes e depois e reconciliação, sem dados sensíveis.
- Qualquer pedido duplicado, estoque negativo, consumo indevido ou divergência de integridade reprova a execução. Erros técnicos podem ocorrer sem reprovação automática se integridade e demais critérios permanecerem satisfeitos; a taxa de erro segue QU-03.

#### Preparação, monitoramento e instrumento (QU-40 a QU-50)

- Antes de cada execução, restaurar estado inicial controlado, validar dados e isolamento entre cenários e aplicar as condições de volume, estados e cache.
- Monitorar CPU, memória, rede, conexões/consultas ao banco, pool de conexões, disco e métricas da aplicação. Métricas de recursos são diagnósticas, sem limites arbitrários adicionais.
- Validar que o gerador suporta 1.000 VUs sem se tornar gargalo. Geração distribuída é permitida quando necessária; documentar topologia e validar perfil, sincronização e carga real.
- Medir overhead da instrumentação, manter configuração consistente e documentar impacto; não subtrair overhead automaticamente dos resultados oficiais.
- Comparar instrumentação completa e mínima em condições equivalentes: mesma infraestrutura, aplicação, dados, carga, cache, duração e repetições. Configuração mínima preserva identificação e rastreabilidade básicas.
- Cinco pares por combinação: 3 volumes × 2 cache × 5 pares = **30 pares**, totalizando **60 execuções adicionais**.
- Alternar a ordem entre os pares (completa/mínima, depois mínima/completa, repetindo alternância).
- Aguardar estabilização do ambiente antes de cada execução, verificando ausência de requisições pendentes, filas residuais e recuperação dos recursos. Restaurar dados é etapa separada e também obrigatória.
- Calcular overhead percentual por par, com resultados individuais, média, mediana e dispersão. Avaliar cada métrica separadamente, sem agregar grandezas diferentes em um único percentual.
- Overhead é indicador diagnóstico: sem limite percentual fixo e sem reprovação automática. Investigar e documentar impactos relevantes.

#### Consolidação e rastreabilidade (QU-71 a QU-73)

- Relatório consolidado deve apresentar resultados, percentis, erros, RPS, recursos e overhead.
- Preservar configuração, versão, dados, logs e evidências por execução.
- RNF-01 só pode ser marcado como validado/concluído após execução e avaliação de todos os cenários e critérios aprovados. A aprovação documental não equivale à execução dos testes.

#### Resumo quantitativo da campanha RNF-01

| Campanha | Quantidade prevista |
|---|---:|
| Carga oficial (3 volumes × 2 condições de cache × 5 execuções) | 30 execuções |
| Comparação de overhead (30 pares × 2 configurações) | 60 execuções adicionais |
| Disputa de estoque (4 níveis × 10 repetições) | 40 execuções |
| **Total planejado dessas três campanhas** | **130 execuções** |

O total acima não inclui testes unitários/de integração, testes exploratórios, nem eventuais repetições de correção. Não significa que as execuções já foram realizadas.

### 6.2. RNF-02 — Tratamento de falhas de comunicação

Texto original preservado: falhas de comunicação devem exibir mensagem compreensível e não simular sucesso.

O frontend utiliza Axios segundo a documentação; em falha, prevê tela de erro com opções de voltar ou tentar novamente, mencionando rate limiting. A política aprovada prevê retry limitado apenas para falhas transitórias e operações seguras, com backoff exponencial/jitter conforme adendo. Criação de pedido usa `Idempotency-Key`; não gerar nova chave em retry da mesma intenção. O backend aplica limites distribuídos via Redis e devolve o envelope de erro simbólico padronizado. Detalhes numéricos por dependência serão configurados e testados na implementação.

## 7. Critérios de aceitação consolidados

### 7.1. RF-01 — Cadastro e autenticação

- Dado um e-mail ainda não cadastrado e dados válidos, quando a conta for criada, então deve existir exatamente uma entidade `User` e um papel `CLIENTE` associado.
- Dado e-mail já cadastrado, quando houver nova tentativa de cadastro, então não deve ser criada outra conta.
- Dadas credenciais válidas, o login autentica e retorna a sessão conforme contrato vigente.
- Dadas credenciais inválidas, a autenticação é recusada (exemplo original HTTP 401).
- Não é possível obter papel `ADMIN` por cadastro público.
- Fluxo de solicitação de `RESPONSAVEL` respeita aprovação e cooldown de 7 dias após rejeição.

### 7.2. RF-02 — Avaliação de produto

- Cliente só avalia `OrderItem` pertencente a pedido seu.
- Pedido deve estar em `CONCLUIDO`.
- Nota deve estar entre 1 e 5; comentário pode estar ausente.
- Uma segunda avaliação para o mesmo `order_item_id` deve ser rejeitada.
- Visitante sem autenticação pode consultar a lista paginada de avaliações do produto; o autor deve aparecer como “Cliente verificado”, sem dados pessoais.
- Avaliação não armazena cópia redundante de customer/order/restaurant; vínculo é obtido por relações.

### 7.3. RF-03 — Pedido demonstrativo

- Cliente deve estar autenticado e ter `CLIENTE`.
- Restaurante deve estar publicado e aberto.
- Pedido contém itens de um único restaurante, com quantidade de 1 a 20 por produto e sem duas linhas para o mesmo `restaurant_product_id`.
- Backend revalida preço/disponibilidade e calcula total.
- Pedido inicia `CRIADO` e mantém snapshot de nome/preço.
- Transições inválidas são recusadas; cada alteração de status cria `OrderEvent`.
- Cancelamento respeita matriz de ator/estado; estoque é restaurado apenas se havia sido baixado na confirmação.
- Idempotência impede duplicação conforme seção 5.1.

### 7.4. RF-04 — Painel

- Apenas responsável autorizado consulta dados do restaurante.
- Tentativa de consultar restaurante alheio é negada.
- “Produtos vendidos” corresponde à soma de quantidades em pedidos `CONCLUIDO`; avaliações recebidas são contadas pela data de criação da avaliação. Período padrão de 30 dias, com filtros de até 365 dias no timezone do restaurante. Não há faturamento oficial sem integração de pagamento.

### 7.5. RF-05/RF-09 — Catálogo

- Responsável só cria/edita/remove produto/oferta dentro de restaurante autorizado.
- Preço > 0; estoque >= 0.
- Produto sem estoque permanece visível e indisponível.
- Produto referenciado por pedido não pode ter histórico invalidado.
- Alterar nome/preço não modifica snapshots antigos.
- Oferta só usa categoria do mesmo restaurante.
- Duplicidade de nome é validada por `normalized_name` calculado no servidor; unicidade `(restaurant_id, normalized_name)`, inclusive para ofertas desativadas.
- Upload aceita apenas JPEG/PNG/WebP estáticos dentro dos limites de 5 MB e 40 MP; conteúdo é validado e convertido para WebP.
- Processamento aplica remoção EXIF, sRGB, preservação de transparência e redimensionamento máximo de 1600×1600 sem upscale.
- A associação da nova imagem só ocorre após upload confirmado e atualização condicional; falhas preservam a imagem anterior.
- Substituição/remoção usa operação persistente no Cloudinary: `destroy` com `invalidate=true`; não há confirmação consultável de purge global e caches externos podem manter cópias temporariamente.

### 7.6. RF-06/RF-07 — Papéis e restaurantes

- Conta é única e e-mail globalmente único.
- Conta pode ser CLIENTE e RESPONSAVEL; ADMIN é exclusivo.
- ADMIN é único, criado manualmente, sem cadastro público e com senha hash.
- Responsável pode gerenciar vários restaurantes; cada restaurante tem um único responsável.
- Novo restaurante começa `UNPUBLISHED`.
- `GET /api/me/restaurants` lista somente restaurantes do usuário autenticado.
- Despublicação é bloqueada quando há pedidos ativos; painel permanece acessível quando despublicado.

### 7.7. RNF-01

- Cada uma das 30 execuções oficiais atende P95/P99 e erro por operação; cada combinação tem 5 execuções individuais aprovadas.
- Execução com qualquer operação acima do limiar ou taxa de erro acima de 1% reprova a combinação.
- Campanha de concorrência não apresenta estoque negativo, duplicidade de consumo/pedido ou reconciliação divergente.
- Campanha de overhead preserva pares equivalentes e evidências; overhead não reprova por percentual próprio.
- Os cenários de falha e recuperação comprovam rollback pré-commit e recuperação pós-commit via chave.

### 7.8. RNF-02

- Diante de falha de comunicação, a interface exibe mensagem compreensível.
- A interface não mostra sucesso quando a operação não foi confirmada.
- Retry automático é limitado a falhas transitórias e operações idempotentes/seguras; criação de pedido repete a mesma intenção com a mesma `Idempotency-Key`, sem gerar nova chave.

## 8. Cenários de teste relacionados

### 8.1. Conta, papéis e autorização

1. Cadastro válido cria User e papel CLIENTE.
2. E-mail duplicado não cria uma segunda conta.
3. CLIENTE e RESPONSAVEL podem coexistir na mesma conta.
4. Tentativa de associar ADMIN junto a CLIENTE/RESPONSAVEL é bloqueada.
5. ADMIN não pode ser criado via rota pública e deve existir uma única conta ADMIN.
6. Pedido de papel RESPONSAVEL segue aprovação administrativa.
7. Rejeição inicia cooldown de 7 dias.
8. Remoção de RESPONSAVEL exige ação administrativa e gera auditoria.
9. Testar concessão, remoção, aprovação e rejeição com registro auditável.
10. Acesso a dados pessoais de outra conta é negado.

### 8.2. Restaurante, catálogo e painel

1. Responsável sem papel tenta cadastrar/gerir restaurante: negar.
2. Responsável cadastra dois restaurantes: ambos ficam associados ao mesmo usuário.
3. Restaurante novo inicia UNPUBLISHED.
4. Usuário não autorizado tenta listar/editar/gerir restaurante alheio: negar.
5. Despublicação com pedidos em estado ativo: HTTP 409 conforme contrato de gestão documentado.
6. Despublicação sem pedido ativo: atualiza publication_status sem excluir histórico.
7. Gerenciar restaurante despublicado continua permitido.
8. Restaurante publicado porém fechado permanece consultável, sem aceitar novo pedido.
9. Criar oferta com preço <= 0 ou estoque < 0: rejeitar.
10. Produto com estoque zero permanece visível, mas indisponível.
11. Categoria de outro restaurante não pode ser usada na oferta.
12. Atualizar oferta não altera snapshots de pedidos antigos.
13. Produto já usado em pedido não perde referências históricas.
14. Responsável consulta somente indicadores do seu escopo.
15. Upload rejeita arquivo acima de 5 MB, acima de 40 MP, animado, malformado ou com extensão/MIME/assinatura incompatíveis.
16. JPEG, PNG e WebP estáticos válidos são processados para WebP com EXIF removido, sRGB, transparência preservada e limite de 1600×1600 sem ampliação.
17. Falha de validação/processamento/upload preserva a imagem anterior.
18. Duas substituições simultâneas não permitem que operação desatualizada sobrescreva a mais recente.
19. Substituição/remoção gera operação de exclusão idempotente para a chave antiga; no Cloudinary, `destroy` com `invalidate=true` solicita exclusão e invalidação best effort, sem confirmação global consultável.
20. Temporários são limpos; eventos e métricas de upload/exclusão são registrados sem segredos.

### 8.3. Pedido e avaliação

1. Visitante tenta criar pedido: negar.
2. Usuário sem CLIENTE tenta criar pedido: negar.
3. Pedido com produtos de restaurantes diferentes: rejeitar.
4. Pedido com quantidade fora de 1..20: rejeitar.
5. Duplicar `restaurant_product_id` em linhas separadas do mesmo pedido: rejeitar.
6. Criar pedido válido gera estado CRIADO e OrderItems com snapshots.
7. Falha ao criar pedido não deixa cabeçalho sem itens nem registro parcial.
8. Confirmação baixa estoque atomicamente.
9. Recusa não baixa estoque.
10. Cancelar em cada combinação papel/estado valida matriz.
11. Cancelar após baixa restaura estoque uma única vez.
12. Mudança inválida de status não persiste.
13. Cada mudança válida produz OrderEvent com antes/depois, ator e horário.
14. Cliente avalia item próprio após CONCLUIDO.
15. Avaliar item de outro cliente ou antes de CONCLUIDO: rejeitar.
16. Avaliação fora de 1..5: rejeitar.
17. Segunda avaliação para mesmo item: rejeitar.
18. Visitante anônimo consulta avaliações paginadas do produto; resposta não inclui nome, e-mail ou identificador pessoal do autor.

### 8.4. Idempotência e falhas

1. Mesma key + mesmo payload em sequência: não duplica pedido e retorna o resultado original.
2. Mesma key + payload diferente: HTTP 409, sem mutação.
3. Mesma key em requisições simultâneas: apenas uma execução.
4. Segunda chamada aguarda até 5 segundos; se primeira terminar, devolve resultado; caso contrário 202 com referência.
5. Consultar `GET /api/orders/idempotency/{key}` com usuário titular: retorna status/resultado conforme contrato a detalhar.
6. Usuário diferente tenta consultar chave alheia: não vaza existência/conteúdo.
7. Chave expirada após 24h é recusada por causa do tombstone permanente.
8. Nova intenção gera nova key.
9. Payload equivalente com itens em ordem diferente gera o mesmo hash canônico.
10. Payload com campo relevante alterado gera hash diferente e 409.
11. Falha antes do commit gera rollback integral.
12. Resposta HTTP perdida após commit é recuperada pela idempotência.
13. Reiniciar processo com operação pendente e verificar estado persistido/recuperável.

### 8.5. RNF-01: carga, concorrência e instrumentação

Executar os 30 testes oficiais, 60 execuções adicionais de overhead e 40 execuções de disputa de estoque conforme seção 5.1. Guardar logs/evidências individuais e reconciliação. Não substituir os limites aprovados por médias agregadas.

### 8.6. RNF-02

1. API indisponível: mostrar mensagem compreensível, sem falsa confirmação.
2. Timeout: informar estado incerto sem disparar automaticamente uma nova intenção de compra com nova chave.
3. Erro de servidor: não apresentar sucesso.
4. Retry manual: respeitar futuras regras de repetição e idempotência quando forem especificadas.

## 9. Dependências entre requisitos

| Origem | Depende de | Impacto/dependência |
|---|---|---|
| RF-01 | `User`, `UserRole` | Conta única e autenticação são pré-condição para fluxos protegidos. |
| RF-02 | RF-01, RF-03; `Order`, `OrderItem`, `Review` | Avaliação exige cliente autenticado, pedido próprio concluído e item comprado. |
| RF-03 | RF-01, RF-07, RF-08; catálogo, horários, publicação, `Order`, `OrderItem`, `OrderEvent` | Pedido depende de cliente, restaurante válido, ofertas e disponibilidade. |
| RF-04 | RF-07, RF-08; `Restaurant`, `Order`, `OrderItem`, `Review` | Painel depende de dados de restaurante, pedidos e avaliações; fórmulas e janela temporal definidas no adendo complementar. |
| RF-05 | RF-06, RF-07; `ProductBase`, `RestaurantProduct`, `Category` | Gestão de produtos depende de papel RESPONSAVEL e ownership do restaurante. |
| RF-06 | RF-01; `UserRole` e fluxo de solicitação/auditoria | Atribuição de RESPONSAVEL depende de aprovação administrativa; `RoleRequest` limita a uma solicitação PENDING por usuário e registra auditoria em `RoleAuditEvent`. |
| RF-07 | RF-06; `User`, `UserRole`, `Restaurant`, `Order` | Gestão de restaurante depende de papel e associação do responsável. |
| RF-08 | RF-03; `RestaurantProduct`, `Order`, `OrderItem`, `OrderEvent` | Estoque/preço participam da validação do pedido e das transições de confirmação/cancelamento. |
| RF-09 | RF-05, RF-08; `ProductBase`, `RestaurantProduct`, `Category`, `OrderItem` | Alterações de catálogo não podem reescrever snapshots. |
| RNF-01 | RF-01, RF-03, RF-04, RF-05, RF-07, RF-08; staging, banco, gerador, dados | Testa nove operações, concorrência, idempotência e integridade sob carga. |
| RNF-02 | Todos os RF via API/cliente; Axios e contratos de erro | Falhas não podem ser mostradas como sucesso; retry e contratos precisam ser detalhados. |

## 10. Matriz de rastreabilidade resumida

| Requisito | Regras principais | Entidades | Endpoints documentados/relacionados | Testes principais |
|---|---|---|---|---|
| RF-01 | RN-04 e regras de conta/papéis | `User`, `UserRole` | `POST /api/auth/register`; `POST /api/auth/login`; `POST /api/auth/logout` | Cadastro, e-mail único, login, logout, papéis e ciclo de sessão. |
| RF-02 | RN-05 | `Review`, `OrderItem`, `Order`, `User` | `POST /api/order-items/{orderItemId}/reviews`; `GET /api/products/{productId}/reviews` | Criação exige ownership e pedido concluído; leitura pública sem login e sem dados pessoais do autor. |
| RF-03 | RN-01, RN-06, RN-07, RN-08, RN-09; cancelamento/idempotência | `Order`, `OrderItem`, `OrderEvent`, `Restaurant`, `RestaurantProduct` | `POST /api/orders/create`; `GET /api/orders`; `GET /api/restaurants/{restaurantId}/orders`; `GET /api/me/orders/inbox`; `PATCH /api/orders/{orderId}/status` | Criação, snapshots, transições, cancelamento, estoque, idempotência e falhas. |
| RF-04 | Permissões de responsável/admin | `Restaurant`, `Order`, `OrderItem`, `Review`, `RestaurantProduct` | `GET /api/restaurant/dashboard?restaurantId={restaurantId}` | Ownership e fórmulas aprovadas de produtos vendidos/avaliações; filtro temporal no timezone. |
| RF-05 | RN-02, RN-03, RN-06, RN-07 | `ProductBase`, `RestaurantProduct`, `Category`, `OrderItem` | `POST /api/restaurants/:id/products`; `PATCH /api/products/:id`; `DELETE /api/products/:id` | CRUD, ownership, duplicidade, estoque zero, histórico. |
| RF-06 | Conta SINGLE, CLIENTE automático, RESPONSAVEL request/approval, ADMIN exclusivo | `User`, `UserRole`, `RoleRequest`, `RoleAuditEvent` | `/api/me/role-requests`; `/api/admin/role-requests`; rotas admin de transferência/remoção | Cadastro de papel, aprovação, rejeição, cooldown, exclusividade e auditoria. |
| RF-07 | Responsável único por restaurante, vários restaurantes por responsável; publicação | `User`, `UserRole`, `Restaurant`, `Order` | Endpoints de gerenciamento listados na seção API | Criação, listagem, gestão, publicação, despublicação, pedidos ativos. |
| RF-08 | RN-06, RN-08; estoque na confirmação/estorno no cancelamento | `RestaurantProduct`, `Order`, `OrderItem`, `OrderEvent` | `POST /api/orders/create`; `PATCH /api/orders/{orderId}/status`; consulta idempotente em `GET /api/orders/idempotency/{key}` | Estoque não negativo, disputa, baixa/restauração atômicas; criação sem baixa de estoque. |
| RF-09 | RN-02, RN-03, RN-07 | `ProductBase`, `RestaurantProduct`, `Category`, `OrderItem` | `PATCH /api/products/:id`; rotas de produto | Atualização, categoria no mesmo restaurante, snapshots. |
| RNF-01 | QU-01 a QU-73 | Todas as entidades envolvidas nos cenários | Nove operações da QU-01; endpoint idempotência aprovado | 30 carga + 60 overhead + 40 concorrência, recuperação e reconciliação. |
| RNF-02 | Tratamento de falhas | API, frontend, sessão | Endpoints via Axios; política de retry e envelope do adendo | Mensagem compreensível, sem falso sucesso; retry limitado e idempotência respeitada. |

## 11. Estado atual: decisões fechadas e itens ainda a materializar

As decisões de produto/arquitetura abaixo já estão fechadas e não devem continuar aparecendo como pendências: ciclo de sessão JWT/refresh, `RoleRequest` e `RoleAuditEvent`, limite de uma solicitação pendente, transferência antes da remoção de RESPONSAVEL, timezone/grade semanal, BRL/DECIMAL(12,2), normalização de nomes, carrinho local, métricas do dashboard, stack, convenções HTTP, rotas de pedidos recebidos e avaliações públicas, além das duas fronteiras transacionais da QU-60.

### 11.1. Contratos que ainda exigem detalhamento antes do freeze da OpenAPI

1. **Categorias:** definir e documentar as rotas de consulta/gestão de categorias, mantendo ownership por restaurante. A entidade e suas regras de domínio já estão definidas.
2. **Upload de imagem do responsável:** o fornecedor Cloudinary e o fluxo técnico estão escolhidos, mas o path/request/response público de upload não foi aprovado; especificar antes de implementar o endpoint.
3. **Reprocessamento de exclusão Cloudinary:** definir o mecanismo operacional/administrativo de reprocessamento de `StorageDeletionOperation`; não supor endpoint público.
4. **Marcação de visualização:** contrato decidido como `POST /api/orders/{orderId}/views`; incluir schema de resposta e código de erro na OpenAPI.
5. **Dashboard multi-restaurante:** `restaurantId` é obrigatório em `GET /api/restaurant/dashboard`; registrar filtros/períodos e esquema de resposta agregada na OpenAPI.
6. **Filtros de administração de mídia:** nomes e tipos dos parâmetros de lista de operações ainda precisam ser registrados.

### 11.2. Detalhes operacionais a validar antes de produção

1. Política fail-safe/degraded quando Redis estiver indisponível para cache/rate limiting, sem bypass silencioso de controles de segurança.
2. Procedimento externo e evidência de autorização do proprietário para ações ADMIN sujeitas a autorização prévia.
3. Política de moderação/denúncia e retenção de avaliações públicas; o acesso de leitura pública já está aprovado, mas as ferramentas de moderação não fazem parte dos contratos atuais.
4. Monitoramento Cloudinary: frequência/cache da consulta Admin API Usage, tratamento de dados periódicos/atrasados/indisponíveis e alarme de quota. O bloqueio preventivo interno de novos uploads é uma salvaguarda operacional; não garante interrupção do consumo de entrega de imagens existentes.
5. Ajustar SLO/RPO/RTO, retenção e alertas após primeiro ambiente de staging, mantendo os baselines do adendo até então.

### 11.3. Entregas futuras de implementação — não são decisões pendentes

- Criar `docs/api/openapi.yaml` a partir dos contratos já definidos; o arquivo ainda não faz parte deste ZIP por estar planejado para a etapa de desenvolvimento.
- Materializar o schema físico Prisma/PostgreSQL, constraints, índices, seeds e migrations.
- Implementar backend, mobile, web, integrações Cloudinary e infraestrutura.
- Executar testes, campanhas RNF-01 (130 execuções), segurança, recuperação e homologação; documentar resultados reais, sem antecipar aprovação.
- Implementar guards de navegação por papel nos clientes e seletor de restaurante no dashboard de responsáveis com múltiplos estabelecimentos.
- Fixar versões de dependências e pipelines de CI conforme a stack aprovada.

A ausência atual de OpenAPI, código, schema físico ou resultados de execução é compatível com o estágio de preparação do projeto e não deve ser confundida com decisão de produto em aberto. Não se declara que software ou infraestrutura estejam implementados.


