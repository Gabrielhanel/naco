# NɅCO — Decisões complementares para início do desenvolvimento

**Versão:** 1.2  
**Data:** 09/11/2026  
**Finalidade:** complementar `01_ESPECIFICACAO_CONSOLIDADA.md`, `02_MODELO_DADOS_E_API.md`, `03_REGISTRO_DECISOES_RNF01.md` e `04_RELATORIO_COMPARATIVO_E_AUDITORIA.md`, fechando padrões de implementação necessários ao MVP.

> Este documento é um adendo de decisões. Não substitui os documentos-base e não altera os RF-01 a RF-09, RN-01 a RN-09, RNF-01 ou RNF-02. As decisões abaixo foram adotadas com base na autorização do usuário para aceitar as opções tecnicamente recomendadas. Onde há impacto em contrato legado, a recomendação é explícita e deve ser incorporada aos documentos antes de congelar a API.

## 1. Estado e regra de precedência

- Requisitos e regras de negócio já aprovados nos documentos-base continuam prevalecendo.
- Este adendo fecha detalhes técnicos e escolhas de MVP que estavam em aberto, sem mudar o comportamento aprovado.
- A regra de estoque permanece inequívoca: **criar pedido não baixa nem reserva estoque; confirmar pedido baixa; cancelar pedido já confirmado restaura uma única vez**.
- As operações de pedidos recebidos deixam de tratar `GET` como mutação. `GET` consulta; `PATCH` altera estado.
- A avaliação é `PRODUCT_ONLY`, conforme decisão já aprovada. A rota antiga de avaliação de restaurante não deve ser implementada como está.
- Decisão posterior aprovada: avaliações e comentários são públicos e qualquer visitante pode consultá-los sem login. O autor deve ser apresentado como “Cliente verificado”, sem exposição de dados pessoais.
- Quando houver divergência entre este adendo e algum contrato legado, prevalece a regra de domínio aprovada; o contrato legado deve ser ajustado antes da implementação daquela operação.

## 2. Decisões pelas 14 etapas

### Etapa 1 — Pedidos recebidos, detalhes e acompanhamento

**Listagem por restaurante**

- `GET /api/restaurants/{restaurantId}/orders` é exclusivo para responsável autenticado e autorizado naquele restaurante.
- A resposta contém resumo: ID, data/hora, cliente, total e estado atual. Os itens e o histórico completo ficam na consulta de detalhes.
- Todos os estados são consultáveis: `CRIADO`, `CONFIRMADO_RESTAURANTE`, `EM_PREPARO`, `SAIU_PARA_ENTREGA`, `CONCLUIDO`, `RECUSADO` e `CANCELADO`.
- Ordenação: estados ativos primeiro (`CRIADO`, `CONFIRMADO_RESTAURANTE`, `EM_PREPARO`, `SAIU_PARA_ENTREGA`), depois estados encerrados; dentro dos grupos, mais recentes primeiro. Desempate determinístico por ID descendente.
- Filtros: grupo ativo/encerrado, um ou mais estados e pesquisa. Dimensões diferentes combinam com AND; múltiplos valores dentro de uma dimensão combinam com OR.
- Pesquisa por ID, nome do cliente e nome do restaurante na visão consolidada. Nomes têm correspondência parcial, sem distinção de caixa/acentos e com tokens em qualquer ordem.
- Paginação tradicional: `page=1`, `limit=20` por padrão; máximo 100. A interface usa 20 por página.

**Visão consolidada**

- `GET /api/me/orders/inbox` reúne pedidos de todos os restaurantes gerenciados pelo responsável.
- A resposta identifica o restaurante de cada pedido e inclui contagem global e contagens por restaurante.
- A visão consolidada usa a mesma ordenação, pesquisa, filtros e paginação da visão individual, acrescida do filtro por um ou mais restaurantes.
- A seleção de um pedido abre `GET /api/orders/{orderId}` em tela separada; ao voltar, a interface preserva filtro, pesquisa e página.

**Visualização por responsável**

- Nova entidade lógica/física `OrderView`, com unicidade `(order_id, responsible_user_id)` e `first_viewed_at`.
- Carregar a lista não marca o pedido como visto.
- Abrir os detalhes registra a primeira visualização por meio de `POST /api/orders/{orderId}/views`; o `GET /api/orders/{orderId}` permanece puro e somente consulta.
- O registro é idempotente: a primeira data/hora é preservada; chamadas posteriores não a substituem.
- Cada responsável possui seu próprio estado de visualização. A visualização de um não altera o contador dos demais.
- O registro é preservado se o usuário perder autorização sobre o restaurante.
- Contador de novos pedidos considera pedidos não visualizados e ainda ativos. `CONCLUIDO`, `RECUSADO` e `CANCELADO` saem do contador, sem apagar o histórico de visualização.
- A contagem ignora os filtros da lista.

**Atualização e transições**

- Polling automático a cada 15 segundos, botão de atualizar e pull-to-refresh em telas compatíveis.
- Pausar polling quando a tela não estiver visível; atualizar imediatamente ao retornar.
- Atualizações externas refletem estado e histórico automaticamente; alterações locais ainda não enviadas são preservadas.
- Após conflito, erro ou falha de comunicação em uma ação, consultar o estado atual antes de permitir nova tentativa.
- Botão de ação fica desabilitado após o primeiro clique até haver resposta ou reconciliação.
- `GET /api/orders/{orderId}` retorna detalhes, itens, snapshots, cliente, endereço de entrega e eventos cronológicos, sem alterar estado ou gravar visualização.
- `POST /api/orders/{orderId}/views` registra a primeira visualização do pedido para o responsável autorizado, sem alterar o pedido; repetir a chamada mantém `first_viewed_at` original.
- `PATCH /api/orders/{orderId}/status` recebe o estado de destino e é o único contrato de transição de estado do pedido. O backend valida papel, propriedade, estado atual e transição permitida.
- Aceitar: `CRIADO` → `CONFIRMADO_RESTAURANTE`; recusar: `CRIADO` → `RECUSADO`; avançar manualmente: `CONFIRMADO_RESTAURANTE` → `EM_PREPARO` → `SAIU_PARA_ENTREGA` → `CONCLUIDO`; cancelamentos obedecem estritamente à matriz aprovada por papel/estado.
- Atualizações concorrentes ou transições inválidas retornam `409 ORDER_STATE_CONFLICT`; nenhuma alteração parcial é persistida.

**Snapshots do pedido**

- O checkout envia os dados de entrega. O backend grava endereço completo e imutável no pedido.
- Também são gravados snapshots do nome e e-mail do cliente para que o histórico não mude se o perfil for atualizado.
- Alterar dados do cliente depois da compra não altera pedido já criado.

### Etapa 2 — Avaliações de produto

- Escopo definitivo: `PRODUCT_ONLY`.
- Rota recomendada e adotada para o MVP: `POST /api/order-items/{orderItemId}/reviews`.
- Cliente autenticado, dono do pedido, pode criar uma avaliação somente quando o pedido estiver `CONCLUIDO`.
- Uma avaliação por `order_item_id`; nota inteira de 1 a 5; comentário opcional.
- A entidade `Review` mantém apenas referência a `OrderItem`, nota, comentário e timestamps; não duplicar cliente, pedido ou restaurante.
- Não implementar `/api/restaurants/{id}/reviews` como avaliação de restaurante. A referência legada deve ser removida ou marcada como substituída na documentação da API.
- MVP permite criar avaliações vinculadas ao produto e consultar publicamente avaliações individuais pela rota `GET /api/products/{productId}/reviews`. Edição/exclusão pelo cliente não entram no primeiro corte.

### Etapa 3 — Idempotência e transações de estoque

**Criação de pedido**

- `POST /api/orders/create` exige `Idempotency-Key` UUID v4 gerada pelo cliente para cada intenção lógica.
- Escopo de unicidade: `(user_id, idempotency_key)`.
- Payload canônico considera `restaurantId`, endereço de entrega normalizado e itens ordenados por `restaurantProductId`, com quantidade e preço esperado de cada item. IDs/quantidades/endereço são dados relevantes; ordem das linhas e espaços irrelevantes não alteram o hash.
- O preço enviado pelo cliente nunca é autoridade. O servidor lê preço atual. Se diferir de `expectedUnitPrice`, retorna `409 PRICE_CHANGED` com os preços atuais, sem criar pedido; a interface pede revisão e uma nova intenção usa nova chave.
- Produto indisponível, restaurante fechado/despublicado ou estoque insuficiente no momento da criação resultam em recusa de negócio, sem criação do pedido. Ainda assim, a confirmação revalida o estoque porque nenhum estoque é reservado.
- Registro idempotente, `Order` e `OrderItem` são persistidos na mesma transação. Falha antes do commit faz rollback integral. Estoque não é alterado nessa transação.
- Persistir estado, hash, status HTTP final, referência ao pedido quando houver e timestamps. Evitar duplicar dados pessoais no registro idempotente; usar referência ao pedido.
- Resultado completo retido por 24 horas; depois, manter tombstone mínimo permanentemente enquanto o sistema existir. Chave expirada nunca pode ser reutilizada.
- Mesma chave + mesmo hash: retornar o resultado original. Mesma chave + payload diferente: `409 IDEMPOTENCY_KEY_REUSED`, sem nova operação.
- Requisições simultâneas: apenas uma processa. A segunda aguarda até 5 segundos sem manter transação ou lock de banco aberto. Se a primeira não terminar, retornar `202` com referência de consulta; a primeira continua.
- `GET /api/orders/idempotency/{key}` exige autenticação e titularidade. Chave desconhecida retorna 404; tombstone próprio expirado retorna 410; chave de outro usuário também retorna 404 para não revelar existência; em processamento retorna 200 com estado `PROCESSING`; concluída retorna 200 com resultado/pedido.
- O `202` contém `status: "PROCESSING"`, referência/URL de consulta e sugestão de intervalo de consulta. Não cria uma segunda operação.

**Confirmação/cancelamento e estoque**

- A confirmação do pedido bloqueia as ofertas envolvidas em ordem determinística de ID, valida todas as quantidades e atualiza todo o estoque, status e `OrderEvent` em uma única transação.
- Se qualquer item não tiver estoque suficiente, toda a confirmação faz rollback e o pedido permanece `CRIADO`; retorna `409 INSUFFICIENT_STOCK`.
- Cancelamento de pedido já confirmado altera estado, cria evento e restaura estoque na mesma transação.
- A transição condicional de estado impede que duas confirmações ou dois cancelamentos processem o mesmo pedido.
- Restauração de estoque é permitida somente na primeira transição elegível para `CANCELADO`; nunca duplicar.
- Assim, QU-60 fica atendida pela atomicidade entre idempotência/criação de pedido/itens na criação e pela atomicidade entre estoque/transição/evento na confirmação/cancelamento, sem antecipar a baixa de estoque.

### Etapa 4 — ADMIN e permissões

- Continua existindo uma única conta `ADMIN`, criada manualmente por script/seed protegido ou intervenção controlada no banco; nunca por cadastro público.
- `ADMIN` é exclusivo e não pode combinar com `CLIENTE` ou `RESPONSAVEL`.
- Garantir unicidade global do papel ADMIN com índice único parcial e exclusividade de papéis com trigger/regra transacional no PostgreSQL, além da validação na camada de serviço.
- As ações administrativas permanecem limitadas à matriz de permissões aprovada. Para ação sensível sujeita a autorização prévia, considerar autorização do proprietário do projeto fora do sistema; registrar ator, alvo, ação, justificativa e data/hora. Não criar um segundo fluxo de aprovação dentro do app.
- Operações administrativas que alteram papéis, responsáveis ou conteúdo são auditáveis e não podem ser executadas sem justificativa.
- A API deve conferir papel atual no servidor; não confiar em role enviado pelo cliente ou apenas em claim antigo do JWT.

### Etapa 5 — Solicitações RESPONSAVEL e auditoria

- Criar estrutura `RoleRequest` com `id`, `user_id`, `status`, `created_at`, `decided_at`, `decided_by` e `decision_reason` opcional.
- Estados: `PENDING`, `APPROVED`, `REJECTED`.
- No máximo uma solicitação `PENDING` por usuário, garantida por índice único parcial.
- Criação da solicitação também gera auditoria (`REQUEST_CREATED`). Aprovação, rejeição, concessão e remoção geram eventos auditáveis.
- Após rejeição, usuário aguarda 7 dias a partir de `decided_at` para solicitar novamente.
- Aprovação e atribuição do papel `RESPONSAVEL` ocorrem na mesma transação.
- Remoção do papel não apaga restaurantes. Se houver restaurantes, ADMIN deve transferi-los previamente para outro usuário que já possua `RESPONSAVEL`; sem transferência, a remoção é bloqueada, preservando a regra de um responsável por restaurante.
- Entidade de auditoria `RoleAuditEvent`: `id`, `actor_user_id`, `target_user_id`, `action`, `reason`, `metadata` JSONB e `created_at`. Registros são append-only; não editar nem apagar pelo fluxo normal.
- Endpoints recomendados: `POST /api/me/role-requests`, `GET /api/me/role-requests`, `GET /api/admin/role-requests?status=PENDING`, `PATCH /api/admin/role-requests/{id}` para aprovar/rejeitar, `PATCH /api/admin/restaurants/{id}/responsible` para transferência e `DELETE /api/admin/users/{userId}/roles/RESPONSAVEL` para remoção controlada.

### Etapa 6 — Horários de funcionamento

- `Restaurant` recebe `timezone` IANA; valor inicial `America/Sao_Paulo`, ajustável para o fuso real da localização do restaurante.
- `openingHours` é uma grade semanal: dias ISO 1–7 (segunda a domingo), cada dia contém zero ou mais intervalos `opensAt`/`closesAt` no formato `HH:mm`.
- Dia sem intervalos é fechado. Restaurante sem grade válida é considerado fechado.
- Mais de um intervalo por dia é permitido. Intervalos não podem se sobrepor.
- `closesAt` menor que `opensAt` representa intervalo que atravessa a meia-noite; a avaliação considera o intervalo iniciado no dia anterior.
- `closesAt` igual a `opensAt` não é permitido; usar dia sem intervalos para fechado.
- Horários são avaliados no fuso do restaurante, inclusive para criação de pedido.
- Feriados, exceções por data e fechamento temporário manual ficam fora do MVP.

### Etapa 7 — Valores monetários

- Moeda única do MVP: BRL.
- Banco: `DECIMAL(12,2)` para preço, preço unitário de snapshot, subtotal e total.
- API transmite valores monetários como strings decimais, por exemplo `"19.90"`, evitando perda de precisão em JavaScript.
- Cálculos usam Decimal ou centavos inteiros; nunca `float`/`number` binário para somas monetárias.
- Entradas com mais de duas casas decimais são rejeitadas com `422`; não fazer arredondamento silencioso.
- Subtotal = preço unitário × quantidade; total = soma dos subtotais. Sem taxa de entrega ou pagamento no MVP, salvo requisito aprovado posterior.

### Etapa 8 — Unicidade e normalização de nomes

- `ProductBase` continua representando identidade compartilhável.
- `RestaurantProduct.display_name` representa o nome efetivamente exibido por restaurante, preenchido inicialmente a partir de `ProductBase.name`.
- `RestaurantProduct.normalized_name` é derivado no servidor: Unicode NFKD, remoção de marcas diacríticas, trim, espaços consecutivos reduzidos a um e comparação em minúsculas de forma independente de locale.
- Restrição única `(restaurant_id, normalized_name)` vale inclusive para ofertas desativadas, evitando duplicidade ao reativar.
- Alterar o nome de uma oferta atualiza `display_name` e `normalized_name` na mesma transação. A identidade global `ProductBase` não é renomeada por uma alteração isolada de restaurante.
- Não tentar equiparar caracteres visualmente semelhantes de alfabetos diferentes; a regra é de caixa/acentos/espaços, não detecção antifraude de homógrafos.

### Etapa 9 — JWT, sessão e segurança

- Senha: mínimo de 12 caracteres; máximo de 128 caracteres sem truncamento; aceitar Unicode e espaços; não exigir combinação artificial de maiúsculas/números/símbolos.
- Hash: Argon2id com pelo menos `m=19456 KiB`, `t=2`, `p=1`; calibrar em ambiente de execução e nunca reduzir abaixo da referência adotada sem revisão. Salt gerado pela biblioteca.
- Access token JWT com validade de 15 minutos; algoritmo HS256 com segredo aleatório de pelo menos 256 bits, armazenado em secret manager/variável protegida, nunca no repositório.
- Refresh token opaco, aleatório e rotativo, validade máxima de 30 dias; banco guarda somente hash do token, sessão, usuário, expiração e revogação.
- Rotação de refresh em cada uso; reutilização de refresh antigo revoga a família de sessão.
- Logout revoga refresh/session no servidor e limpa os dados locais. A API verifica sessão ativa; remoção de papel/status de usuário tem efeito imediato em autorização.
- Mobile: access token em memória e refresh token em Keychain/Keystore por biblioteca segura. Web: refresh token em cookie `HttpOnly; Secure; SameSite=Lax`; access token somente em memória. Não guardar token em `localStorage`/`sessionStorage`.
- Web deve ter proteção CSRF compatível com cookie em operações mutáveis, além de prevenção XSS, CSP e HTTPS.
- JWT contém apenas identificador de usuário/sessão e claims técnicas mínimas; papéis e autorização por restaurante são verificados no servidor.
- Uma conta pode manter CLIENTE e RESPONSAVEL simultaneamente. No MVP não há seletor de identidade/papel no login: o app mobile expõe fluxos de cliente (exige CLIENTE) e o web expõe gestão (exige RESPONSAVEL); o backend verifica o papel e ownership em cada operação. A existência dos dois papéis não cria contas ou sessões de identidade separadas.
- Login: limite separado por identificador de conta e IP; referência inicial de 5 falhas por conta em 15 minutos e 30 tentativas por IP em 15 minutos, com atraso progressivo e resposta genérica. Cadastro: 5 tentativas por IP em 15 minutos. Limite geral inicial: 120 requisições/minuto por usuário autenticado; endpoints sensíveis têm limites próprios.
- Rate limiting usa janela deslizante/token bucket e Redis; resposta `429` genérica, com `Retry-After` quando aplicável.
- Validar e normalizar entradas no backend, usar queries parametrizadas/Prisma, limitar CORS às origens conhecidas, TLS, Helmet/CSP, upload validado por tipo/tamanho quando habilitado, segredos fora do Git e dependências verificadas no CI.
- Não registrar senha, token, refresh token, chave idempotente completa, endereço ou payload sensível em logs.
- MFA continua fora do escopo inicial, conforme requisito original.

### Etapa 10 — DTOs, HTTP e erros

**Convenções gerais**

- JSON externo em `camelCase`; banco pode usar `snake_case`.
- `status` é o campo canônico de estado; não usar `situation`.
- IDs externos de oferta: `restaurantProductId`; preço de catálogo: `price`; snapshot em pedido: `unitPrice`; subtotal: `subtotal`.
- Datas em ISO 8601 UTC (`Z`) nas respostas; horários de funcionamento usam hora local + timezone do restaurante.
- Paginação por `page` e `limit`, padrão 20 e máximo 100; respostas de lista incluem `items`, `page`, `limit`, `totalItems`, `totalPages`.

**Criação de pedido — request de referência**

```json
{
  "restaurantId": "...",
  "deliveryAddress": {
    "recipientName": "...",
    "postalCode": "...",
    "street": "...",
    "number": "...",
    "complement": null,
    "neighborhood": "...",
    "city": "...",
    "state": "RS"
  },
  "items": [
    {
      "restaurantProductId": "...",
      "quantity": 2,
      "expectedUnitPrice": "19.90"
    }
  ]
}
```

O identificador do cliente vem da sessão, nunca do body. Preços definitivos, nome e subtotal são calculados e persistidos pelo backend.

**Envelope de erro**

```json
{
  "error": {
    "code": "INSUFFICIENT_STOCK",
    "message": "Não há estoque suficiente para confirmar este pedido.",
    "details": [],
    "requestId": "..."
  }
}
```

- `error.code` é código simbólico estável, não string numérica do HTTP.
- HTTP 400: JSON malformado, cabeçalho obrigatório ausente ou sintaxe inválida.
- HTTP 401: autenticação ausente, inválida ou sessão expirada sem renovação possível.
- HTTP 403: usuário autenticado sem papel geral necessário.
- HTTP 404: recurso inexistente ou recurso fora do escopo de propriedade/autorização, evitando revelar existência.
- HTTP 409: conflito de negócio, estoque insuficiente, transição concorrente/inválida, duplicidade ou reutilização de chave idempotente com payload diferente.
- HTTP 410: tombstone de chave idempotente expirada do próprio usuário.
- HTTP 422: JSON válido com campos/valores semanticamente inválidos.
- HTTP 429: rate limit.
- HTTP 500: erro interno inesperado; sem stack trace ou detalhe interno para o cliente.
- HTTP 503: indisponibilidade temporária de dependência/serviço.
- Sucessos: 200 consulta/alteração, 201 criação, 202 processamento idempotente aceito.

### Etapa 11 — Carrinho, perfil e painel

**Carrinho**

- Carrinho local no aplicativo mobile, persistido no armazenamento não sensível do dispositivo e associado ao usuário autenticado.
- Um carrinho ativo por vez e um único restaurante por carrinho (RN-01).
- Produto sem estoque não pode ser adicionado; estoque zero continua visível no catálogo como indisponível.
- Remoção/quantidade atualiza imediatamente subtotal e total no app.
- Carrinho não é entidade nem API no MVP; no checkout o servidor revalida todos os dados e é autoridade de preço/estoque/horário/publicação.
- No logout, limpar o carrinho local e dados temporários do usuário.

**Perfil**

- `GET /api/me` retorna dados da conta autenticada.
- `PATCH /api/me` permite alterar `name` no MVP; e-mail permanece identificador de login não editável neste primeiro corte.
- Senha é tratada no fluxo de autenticação/sessão, nunca devolvida em DTO.
- Endereço de entrega é informado no checkout e congelado no pedido; não criar catálogo de endereços nesta primeira versão.

**Painel**

- Preservar `GET /api/restaurant/dashboard`, exigindo `restaurantId` como parâmetro de consulta para identificar explicitamente qual restaurante gerenciado será agregado. O backend valida a autorização do responsável sobre esse restaurante.
- Indicador “produtos vendidos” = soma de `OrderItem.quantity` em pedidos `CONCLUIDO` no período.
- Indicador “avaliações recebidas” = quantidade de `Review` dos itens daquele restaurante no período de criação da avaliação.
- Período padrão: últimos 30 dias; filtros de início/fim aceitos até no máximo 365 dias, interpretados no timezone do restaurante.
- Não apresentar faturamento como métrica oficial, pois o sistema não processa pagamento real.
- Dashboard retorna agregações; não precisa de paginação de registros agregados.

### Etapa 12 — Stack e arquitetura

- Arquitetura: monólito modular, organizado por domínio; sem microsserviços no MVP.
- Backend: Node.js 26 LTS, Express.js e TypeScript em modo strict.
- Banco: PostgreSQL + Prisma ORM/migrations.
- Cache/rate limit: Redis. Cache-aside somente para consultas públicas de restaurantes e catálogo; TTL inicial 60 s; invalidar após alterações de restaurante/produto/categoria. Não cachear estoque/autorização/pedidos como fonte de verdade.
- Mobile: React Native + TypeScript + React Navigation + Axios.
- Web: React + TypeScript + Axios.
- Validação backend: Zod; contratos documentados em OpenAPI.
- Imagens: Cloudinary escolhido para o MVP no plano Free sem cartão. Upload, substituição e exclusão somente pelo backend, abstraídos por `StoragePort`; backend mantém validação/sanitização/conversão para WebP. Não há upgrade, cobrança ou cartão sem autorização explícita. Ao atingir o limite interno de uso, bloquear novos uploads e preservar as imagens existentes. Ver especificação Q63–Q103 adaptada ao Cloudinary.
- Testes: Vitest para unidade; Supertest para API; React Testing Library para componentes; Appium para fluxos mobile; Grafana k6 para carga; Postman para exploração/manual; GitHub Actions para CI.
- Desenvolvimento local: Docker Compose para PostgreSQL e Redis; `.env.example` sem segredos; migrações versionadas e seed de dados de desenvolvimento/admin controlado.
- REST/HTTP; polling de 15 s para pedidos. WebSocket, SSE, filas/mensageria e microsserviços ficam fora do MVP.
- Node.js 26 tornou-se Active LTS em 28/10/2026; fixar versão exata suportada no ambiente e CI. Não usar linha Alpha/Current em produção.

### Etapa 13 — RNF-02, disponibilidade, backup e observabilidade

**Retry e falhas**

- GET idempotentes: até 3 tentativas totais para falha transitória de rede, timeout, 429, 502, 503 ou 504; backoff exponencial com jitter e respeito a `Retry-After`.
- 401 permite uma renovação de sessão e repetição única da chamada original.
- Não repetir automaticamente erros 400, 403, 404, 409 ou 422.
- `POST /api/orders/create` pode ser repetido no máximo 3 vezes somente com a mesma `Idempotency-Key`; em timeout, consultar primeiro o estado da chave. Nunca gerar nova chave para retry da mesma intenção.
- Operações mutáveis de estado não são repetidas às cegas; consultar pedido atualizado e reapresentar ações disponíveis.
- Toda falha deve exibir mensagem compreensível, preservar contexto útil e nunca exibir sucesso sem confirmação da API.

**Operação**

- Não publicar SLA contratual no MVP. Objetivo interno inicial após implantação: disponibilidade mensal de 99,5%, excluindo manutenção planejada, apenas como indicador operacional.
- Backup automático diário do PostgreSQL, criptografado, retenção de 7 backups diários; snapshot extra antes de migração relevante.
- RPO inicial de até 24 h e RTO inicial de até 8 h; executar teste de restauração mensal e registrar evidência.
- Logs estruturados, requestId/correlationId, métricas de taxa/erros/latência, recursos de aplicação, pool/queries do PostgreSQL, Redis e hit/miss do cache.
- OpenTelemetry para tracing de requisições e chamadas ao banco; logs sem credenciais ou PII sensível.
- Alertas iniciais: erro técnico acima de 1% por 5 min, P95 acima do limite por 5 min, indisponibilidade de banco/Redis e falha de backup.
- Retenção técnica inicial: logs de aplicação 90 dias; eventos de auditoria 24 meses, com revisão de privacidade/necessidade antes de produção. Dados pessoais devem seguir política de retenção e eliminação documentada; este adendo não declara conformidade jurídica automática.
- Endpoint operacional de health/readiness sem expor segredos nem detalhes internos.

**Acessibilidade e experiência**

- Web buscar WCAG 2.2 AA como referência de implementação; mobile deve oferecer rótulos para leitor de tela, foco/ordem coerentes, contraste adequado e suporte a ampliação de texto.
- Acessibilidade é critério de qualidade de interface; não substitui os testes funcionais.

### Etapa 14 — Testes, homologação e liberação

**Testes contínuos**

- Unitários para regras de domínio/serviços, validações, máquina de estados, cálculo monetário, normalização e autorização.
- Integração com PostgreSQL real/test container para transações, constraints, migrations, idempotência e estoque. Não simular transações críticas apenas com mocks.
- API: autenticação, ownership/escopo, erros, paginação, DTOs e todos os status definidos.
- Mobile/web: testes de componentes e fluxos principais.
- E2E prioritários: cadastro/login; explorar catálogo; carrinho; checkout; responsável recebe/aceita/recusa; avançar pedido; cancelamentos permitidos; estoque; avaliações após conclusão; gestão de restaurante/produtos.
- Segurança: tentativa de acesso cruzado entre clientes/restaurantes, alteração de role no cliente, uso de token revogado, refresh reutilizado, rate limit, injeção, XSS/CSRF no web e vazamento de segredo nos logs.
- Concorrência: chaves idempotentes sequenciais/simultâneas, payload diferente, confirmação concorrente, cancelamento concorrente, confirmação × cancelamento e falhas pré/pós-commit.

**Qualidade e gates**

- CI obrigatório: lint, typecheck, testes unitários, testes de integração, build e validação de migrations.
- Cobertura de 80% como meta mínima para serviços/regras de domínio; não usar cobertura global isolada como prova de qualidade.
- Nenhum bug crítico/alto conhecido em fluxo de autenticação, autorização, criação/transição de pedidos, estoque ou idempotência no release.
- Smoke test em staging antes de qualquer release.
- RNF-01 só é declarado aprovado após executar todas as campanhas planejadas e preservar resultados: 30 execuções de carga, 60 de overhead e 40 de concorrência, total 130, além de recuperação/idempotência conforme QU-29 a QU-70.
- Redis foi selecionado; portanto a campanha de carga mantém as duas condições QU-17: cache frio e aquecido (3 volumes × 2 condições × 5 execuções = 30).
- Cache frio: limpar cache antes da execução e registrar misses/aquecimento. Cache aquecido: pré-carregar os dados da amostra e confirmar hit rate antes de medir.
- Estabilização antes de cada rodada: nenhuma requisição/execução pendente; filas de infraestrutura vazias; CPU, RAM e conexões do banco sem tendência e dentro de ±5% da linha de base por 5 minutos. Restauração de dataset é etapa separada.
- Instrumentação completa: métricas + logs estruturados + tracing detalhado. Instrumentação mínima: métricas agregadas, requestId e logs essenciais, sem spans detalhados. Alternar a ordem completa/mínima conforme QU-47; não descontar overhead dos resultados oficiais.
- Percentis de concorrência são calculados sobre latências individuais de todas as respostas de cada nível, agregadas nas 10 repetições daquele nível; informar tamanho de amostra e limitação do P99 nos níveis menores. Se a amostra não sustentar conclusão, ampliar repetições antes de declarar aprovação, sem alterar silenciosamente os limites.
- Os 1.000 VUs usam sessões persistentes e pool de contas de teste com papéis adequados; não autenticar a cada request. Concorrência de contas deve ser documentada para não confundir carga do produto com gargalo artificial de login.
- Ambientes/datasets de carga são isolados; pedidos, usuários e endereços são sintéticos, sem dados pessoais reais.

### Consolidação posterior — imagens e armazenamento (Q63–Q103)

As decisões de upload, processamento e remoção estão detalhadas em `../architecture/05_ARMAZENAMENTO_E_PROCESSAMENTO_DE_IMAGENS.md`. O resumo vinculante inclui: Cloudinary no MVP via `StoragePort`; `public_id` único e imutável, `asset_id` e versão persistidos; JPEG/PNG/WebP estáticos, 5 MB e 40 MP; saída WebP até 1600×1600, EXIF removido e sRGB; operações persistentes de upload/eventos/exclusão; `destroy` com `invalidate=true` e invalidação best effort; retries transitórios, reconciliação, controle de concorrência e métricas. Permanecem a fechar antes da implementação: schema físico final, DTOs, limites concretos de recursos/timeouts, qualidade numérica WebP e integração da Admin API Usage para bloqueio preventivo.

## 3. Rotas consolidadas para orientar a API

| Método | Rota | Uso |
|---|---|---|
| POST | `/api/auth/register` | Cadastro público; atribui CLIENTE. |
| POST | `/api/auth/login` | Início de sessão. |
| POST | `/api/auth/refresh` | Renovação/rotação de refresh token. |
| POST | `/api/auth/logout` | Revoga sessão/refresh. |
| GET/PATCH | `/api/me` | Consultar/editar nome da conta. |
| GET | `/api/restaurants` | Explorar restaurantes. |
| GET | `/api/restaurants/{id}` | Detalhes públicos. |
| GET | `/api/restaurants/{id}/products` | Catálogo. |
| POST | `/api/restaurants/{id}/products` | Criar oferta/produto no restaurante. |
| PATCH | `/api/products/{id}` | Atualizar oferta. |
| DELETE | `/api/products/{id}` | Desativar/excluir respeitando histórico. |
| POST | `/api/restaurants` | Criar restaurante. |
| GET | `/api/me/restaurants` | Listar restaurantes gerenciados. |
| GET | `/api/restaurants/{id}/management` | Consultar gestão. |
| PATCH | `/api/restaurants/{id}` | Atualizar cadastro/horários/fuso. |
| PATCH | `/api/restaurants/{id}/publication` | Publicar/despublicar. |
| POST | `/api/orders/create` | Criar pedido; exige Idempotency-Key. |
| GET | `/api/orders` | Histórico do cliente autenticado. |
| GET | `/api/restaurants/{restaurantId}/orders` | Pedidos do restaurante autorizado. |
| GET | `/api/me/orders/inbox` | Pedidos consolidados de todos os restaurantes do responsável. |
| GET | `/api/orders/{orderId}` | Detalhes e histórico do pedido; leitura sem efeito colateral. |
| POST | `/api/orders/{orderId}/views` | Registrar primeira visualização pelo responsável; operação idempotente. |
| PATCH | `/api/orders/{orderId}/status` | Transição validada de estado. |
| GET | `/api/orders/idempotency/{key}` | Consulta idempotência. |
| POST | `/api/order-items/{orderItemId}/reviews` | Cliente autenticado cria avaliação do produto adquirido. |
| GET | `/api/products/{productId}/reviews` | Consultar avaliações públicas do produto, sem autenticação. |
| GET | `/api/restaurant/dashboard?restaurantId={restaurantId}` | Indicadores do restaurante indicado e autorizado. |
| POST/GET | `/api/me/role-requests` | Criar/consultar solicitação de RESPONSAVEL. |
| GET | `/api/admin/role-requests` | Listar solicitações para ADMIN. |
| PATCH | `/api/admin/role-requests/{id}` | Aprovar/rejeitar solicitação. |
| PATCH | `/api/admin/restaurants/{id}/responsible` | Transferir responsável, com auditoria. |
| DELETE | `/api/admin/users/{userId}/roles/RESPONSAVEL` | Remover papel somente após transferências necessárias. |

As rotas novas deste quadro são decisões de contrato para o MVP e devem ser refletidas no documento de API/OpenAPI. O endpoint de visualização é uma operação de escrita separada para preservar a semântica de `GET`. Não implementar endpoints adicionais além destes e dos já preservados sem registrar a necessidade.

## 4. Ajustes de modelo necessários

Além das entidades existentes, implementar no schema físico as seguintes estruturas/campos derivados das decisões acima:

- `OrderView(order_id, responsible_user_id, first_viewed_at)` com unique `(order_id, responsible_user_id)`.
- `RoleRequest` e `RoleAuditEvent` conforme etapa 5.
- `IdempotencyRecord` com unicidade `(user_id, key)`, hash, estado, status HTTP, referência ao pedido, corpo mínimo de resposta/erro, expiração e tombstone.
- `Restaurant.timezone` e estrutura validada de `opening_hours`.
- `RestaurantProduct.display_name` e `normalized_name`; unique `(restaurant_id, normalized_name)`.
- `Order.delivery_address_snapshot` JSONB imutável; `customer_name_snapshot` e `customer_email_snapshot`.
- Restrições compostas para garantir que `RestaurantProduct.category_id` pertence ao mesmo `restaurant_id`.
- Índices para consulta de pedidos por restaurante/estado/data, pedidos do cliente/data, eventos por pedido/data e contagens de visualização.
- Todos os índices adicionais devem ser verificados por plano de consulta e volume, sem substituir constraints de domínio.

`Cart` permanece fora do banco no MVP.

## 5. Ordem recomendada de desenvolvimento

1. Repositório, ambientes, Docker Compose, CI, convenções, validação e OpenAPI.
2. Schema/migrations PostgreSQL/Prisma, constraints e seeds controlados.
3. Cadastro, login, refresh/logout, sessão, papéis e middleware de autorização.
4. Restaurantes, horários, publicação e gestão.
5. Categorias, ProductBase/RestaurantProduct, preço, estoque, imagem e unicidade.
6. Pedidos, snapshots, idempotência, eventos, visualização e transações de estoque.
7. Avaliações e painel.
8. Aplicativo mobile (catálogo, carrinho local, checkout, acompanhamento e avaliações).
9. Web de gestão (catálogo, pedidos, indicadores, contadores e polling).
10. Testes de regressão, segurança, concorrência, homologação e campanha RNF-01.

Desenvolver como fatias verticais: contrato → migração → serviço → controller → testes → interface integrada. Não construir telas contra DTOs provisórios.

## 6. O que não bloqueia o início do desenvolvimento

- Execução dos 130 testes do RNF-01: é gate de homologação, não pré-requisito para criar o projeto e implementar módulos.
- RPO/RTO/SLO e retenções operacionais: adotar os valores acima como baseline e revisar antes de produção.
- Feriados/exceções de horário, multi-endereços salvos, carrinho sincronizado entre dispositivos, WebSocket/SSE, microsserviços, pagamento real e entrega real: fora do MVP.
- Detalhes cosméticos da interface podem ser refinados durante implementação sem alterar contratos de domínio.

## 7. Critério de prontidão para desenvolvimento

A especificação funcional e técnica do MVP está suficientemente fechada para iniciar a implementação com as decisões deste adendo. Antes de congelar a primeira versão da API, incorporar estas rotas/DTOs ao OpenAPI e conferir a compatibilidade com o repositório real. Antes de produção, executar os testes e revisões de segurança/privacidade, testar restauração e validar os indicadores operacionais.

**Este documento não declara o sistema implementado, testado, homologado ou em produção.** Declara que as decisões de produto e os padrões técnicos necessários para iniciar o desenvolvimento foram definidos em nível de MVP.
