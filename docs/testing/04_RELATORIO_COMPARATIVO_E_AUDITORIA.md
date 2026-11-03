# NɅCO — Relatório comparativo e auditoria documental

## 1. Escopo e fontes comparadas

- Documento funcional e técnico original: `README(2).MD`.
- Lista de achados: `writing-block.md`.
- Modelo de dados previamente revisado: `NACO_modelo_dados_revisado.md`.
- Revisão documental anterior: `NACO_relatorio_revisao_completo.md` e `NACO_README_revisado.md`.
- Decisões anteriores do formulário e histórico de conversa, incluindo decisões de conta/papéis, restaurantes, catálogo, pedidos, avaliações e as QU-01 a QU-73 do RNF-01.

O resultado desta revisão consolida conteúdo documental. Não houve alteração de aplicação, schema, migração, endpoint em código ou integração.

## 2. Comparativo — versão anterior versus consolidação

| Tema | Antes | Agora | Natureza da mudança |
|---|---|---|---|
| RF-01 a RF-09 | Lista funcional ampla; algumas funcionalidades sem critérios claros. | IDs originais preservados e descrições/aceitação detalhadas a partir das decisões. | Detalhamento documental. |
| RN-01 | Pedido de um restaurante por vez. | Mantida e associada ao modelo `Order` de um único restaurante. | Preservação. |
| RN-02 | Um produto com mesmo nome por restaurante, sem normalização. | Regra preservada; normalização e constraint física continuam pendentes. | Lacuna explicitada, não resolvida por suposição. |
| RN-03/RN-04 | Autorização de responsável e dados pessoais próprios. | Consolidadas com conta única, múltiplos papéis e autorização por restaurante. | Detalhamento documental. |
| RN-05/RF-02 | RF-02 menciona estabelecimento; entidade `Review` descreve produto. | Decisão aprovada `PRODUCT_ONLY`; uma avaliação por `order_item_id`; pedido próprio concluído; comentário opcional; nota 1–5. | Conflito funcional resolvido. Contrato da rota ainda pendente. |
| RN-06/RN-07 | Produto sem estoque não vai ao carrinho; exclusão não deve permitir nova compra. | Mantido produto visível/indisponível, snapshots e preservação histórica; estoque baixa somente na confirmação. | Detalhamento e consolidação. |
| RN-08 | Carrinho depende de usuário logado e restaurante aberto. | Mantida; criação do pedido também exige restaurante publicado. Estrutura de horário ainda pendente. | Detalhamento documental. |
| RN-09 | Remover produto do carrinho atualiza total. | Mantida; contrato/persistência/endpoint não definidos. | Lacuna preservada. |
| Cancelamento | Regra geral permitia cancelar em qualquer estado pré-conclusão; tabela detalhada limitava por ator/estado. | Cliente: CRIADO, CONFIRMADO_RESTAURANTE, EM_PREPARO. Responsável: CONFIRMADO_RESTAURANTE. Estados terminais e restauração de estoque documentados. | Contradição resolvida por decisão aprovada. |
| Conta e papéis | Matriz e modelo não explicitavam conta única e processo de atribuição de papéis. | `SINGLE`; e-mail único; CLIENTE automático; RESPONSAVEL via request/approval e cooldown 7 dias; ADMIN exclusivo, único, manual, hash e sem cadastro público; auditoria para concessões/remoções/aprovações/rejeições. | Decisões de formulário incorporadas. |
| Restaurante | Modelo já descrevia responsável único, vários restaurantes, publicação e funcionamento separados. | Preservado e ligado aos endpoints de gestão aprovados; restaurante começa UNPUBLISHED; acesso ao painel permanece mesmo despublicado. | Detalhamento documental. |
| Produto/categoria | `ProductBase`, `RestaurantProduct`, `Category` documentados. | Relações e regras de oferta/categoria no mesmo restaurante consolidadas; snapshots preservados. | Detalhamento documental. |
| RNF-01 | 1.000 usuários simultâneos e “tempo aceitável”, sem limites. | Especificação QU-01 a QU-73: operações, P95/P99, taxa de erro, ramp-up, duração, dados, cache, RPS, 30 cargas, 60 overhead, 40 concorrência, idempotência, falhas, recuperação e evidências. | Critérios mensuráveis adicionados com aprovação. |
| RNF-02 | Mensagem compreensível e sem falso sucesso. | Texto preservado; lacunas de retry, classes de falha e rate limiting identificadas. | Sem alteração de requisito; pendências destacadas. |
| API de restaurantes | Endpoints de gestão estavam em documentação de modelo/revisão. | Os cinco endpoints de gestão permanecem; autenticação identifica responsável e HTTPs documentados permanecem. | Consolidação. |
| Idempotência | Havia apenas risco de duplo envio/reenvio. | `Idempotency-Key`, escopo por usuário, hash canônico, 24h, tombstone permanente, HTTP 409 para payload diferente, 202 após 5s e endpoint de consulta aprovado. | Complementações técnicas aprovadas; QU-60 requer esclarecimento de fronteira transacional. |
| Matriz de rastreabilidade | Relações RF/RN/entidades/endpoints/testes parciais. | Matriz consolidada com relações explícitas e “pendente/não especificado” onde não há base. | Organização e detalhamento. |
| Testes | Cenários negativos e concorrência dispersos; critérios formais insuficientes. | Casos de aceitação/teste agrupados por RF, RN, idempotência, carga, concorrência e recuperação. | Detalhamento derivado das decisões. |

## 3. Resoluções efetivas de conflitos

1. **Cancelamento:** a matriz aprovada por papel/estado prevalece sobre a frase geral original que permitia cancelamento em qualquer etapa anterior a CONCLUIDO.
2. **Avaliações:** RF-02/RN-05 passam a tratar somente avaliações de produtos adquiridos, uma por `OrderItem`, depois de pedido `CONCLUIDO`. O caminho exato da rota ainda não foi alterado.
3. **Estado de conclusão:** `CONCLUIDO` é a forma canônica no domínio/API/persistência; “Concluído” pode ser texto exibido na interface.
4. **Conta e papéis:** a regra SINGLE, papéis compostos, ADMIN exclusivo e fluxos de concessão foram definidos; detalhes de persistência de solicitações/auditoria ainda faltam.
5. **RNF-01:** os limites e a metodologia mensurável substituem a expressão vaga “tempo aceitável”, mantendo o requisito original de 1.000 usuários simultâneos.

## 4. Impactos documentados que não foram aplicados unilateralmente

| Decisão | Impacto | Por que ainda exige aprovação/esclarecimento |
|---|---|---|
| RF-02 `PRODUCT_ONLY` | Endpoint original `/api/restaurants/:id/reviews` sugere avaliação do restaurante. | É necessário aprovar rota/contrato compatível com avaliação de produto; não foi inventado novo caminho. |
| QU-52 | Endpoint específico `GET /api/orders/idempotency/{key}`. | Endpoint foi explicitamente escolhido; contrato de resposta detalhado continua pendente. |
| QU-60 | Atomicidade da chave, criação de pedido e operações relacionadas ao estoque. | A regra original/aprovada diz que estoque é baixado apenas na confirmação; não se pode deslocar a baixa para a criação sem nova aprovação. |
| RESPONSAVEL request/approval | Exige registrar solicitações e auditoria em algum meio persistente. | Nome da entidade, campos, estados e endpoints não foram aprovados. |
| ADMIN | Escopo `FULL` limitado pela matriz e auditável; matriz original exige autorização prévia para certas ações. | O procedimento e autoridade da autorização prévia não foram definidos. |
| `GET /api/restaurants/:restaurantId/orders` | Finalidade de receber/aceitar, acesso descrito como Cliente proprietário. | Ator, método/finalidade e rota de mudança de status precisam de decisão; caminho original fica preservado. |

## 5. Pendências remanescentes detalhadas

### 5.1. Funcionais e de domínio

- **Rota pedidos recebidos:** confirmar ator correto (responsável ou cliente), finalidade precisa e método HTTP; definir em que rota/contrato o responsável confirma/recusa/transiciona o pedido.
- **Avaliações:** decidir contrato/rota de avaliação de produto que corresponda à decisão `PRODUCT_ONLY` sem inferir alteração do endpoint legado.
- **Produto com nome duplicado:** definir normalização (caixa, espaços, acentos, Unicode) e entidade/campo em que se aplica RN-02.
- **Horários:** definir estrutura de `opening_hours`, timezone, intervalos que cruzam meia-noite, exceções/feriados e como derivar `operating_status`.
- **Carrinho:** definir se é mantido no cliente ou persistido no servidor, seus campos/vida útil e operações de remoção/atualização; não criar rota até aprovação.
- **Perfil:** definir campos editáveis e contrato/endereço de atualização.
- **Painel:** definir catálogo de métricas, fórmulas, janela temporal, filtros, paginação e tratamento de dados simulados.
- **Conta e perda de papel:** definir como impedir múltiplas solicitações pendentes; se criação de solicitação gera auditoria; como administrar restaurantes quando responsável perde papel sem apagá-los automaticamente.
- **ADMIN:** esclarecer mecanismo de autorização prévia para ações da matriz, mantendo ADMIN exclusivo, único e auditável.

### 5.2. Financeiro, sessão e segurança

- Precisão/escala monetária, moeda e arredondamento de `price`, `unit_price`, `subtotal` e `total`.
- JWT: validade, refresh, expiração, logout/revogação e compatibilidade com persistência local descrita no original como período indeterminado.
- Política de senhas, algoritmo/parâmetros de hash, rate limiting concreto, bloqueio/limite de tentativas e regras de retenção/privacidade.

### 5.3. API e idempotência

- Padronizar HTTP 400 versus 422, 403 versus 404, estrutura e significado de `error.code`.
- Confirmar mapeamento DTO/modelo: `situation`/`status`, `productId`/`restaurant_product_id`, `price`/`unit_price`.
- Definir body/campos do 202, formato da referência, resposta do endpoint de consulta, status para chave ausente/expirada e resposta externa quando chave pertence a outro usuário sem vazar existência.
- Definir classificação do HTTP 409 por payload diferente no cálculo da taxa de erro técnico (QU-03 explicitamente exclui 409 de estoque; o tratamento geral de 409 de idempotência não foi separado nessa decisão).
- Definir campos exatos incluídos no payload canônico/hash.
- **QU-60:** confirmar atomicidade idempotência/pedido separada da baixa de estoque em confirmação e restauração em cancelamento; não há aprovação para baixar estoque ao criar pedido.
- Definir schema físico/índices/tombstone permanente do registro idempotente.

### 5.4. Operacionais e qualidade

- RNF-02: classes de falha, condições de retry, backoff, número máximo, rate limiting e quais operações podem ser repetidas com segurança.
- Disponibilidade/SLA e janela de manutenção.
- Política de backup, retenção, restauração, RPO/RTO.
- Observabilidade operacional geral (logs, métricas, tracing, alertas e retenção) além das métricas exigidas na campanha RNF-01.
- Estratégia de testes: critérios de entrada/saída, cobertura, regressão e gates de liberação.
- Acessibilidade: escopo e critérios.
- Decisões finais sobre stack por componente; tabela original apresenta opções, não stack final.
- Governança e prioridade individual, se o projeto quiser formalizar versão/status/aprovação e prioridade.

### 5.5. Modelagem física

- Constraints/índices físicos para categoria/oferta no mesmo restaurante.
- Garantia de ADMIN globalmente único e exclusivo, além das regras de papel.
- Persistência para solicitação de RESPONSAVEL e auditoria de papéis.
- Persistência idempotente e retenção dos tombstones.
- Índices adicionais não devem ser inventados como requisitos físicos antes do schema; documentar no schema definitivo.

## 6. Auditoria de preservação

| Item de auditoria | Resultado | Evidência documental |
|---|---|---|
| IDs RF-01 a RF-09 preservados | Passou | Especificação seção 3 e matriz de rastreabilidade. |
| IDs RN-01 a RN-09 preservados | Passou | Especificação seção 4. |
| RNF-01 e RNF-02 preservados | Passou | Especificação seção 5; RNF-01 detalhado; RNF-02 sem expansão não aprovada. |
| 73 decisões QU registradas | Passou | `03_REGISTRO_DECISOES_RNF01.md`. |
| Decisões de conta/papéis preservadas | Passou com pendências explícitas | Especificação seção 4.1 e modelo de dados seção 1. |
| Regras de restaurante/publicação preservadas | Passou | Especificação seção 4.2 e modelo seção 1.5. |
| Separação `ProductBase`/`RestaurantProduct` e categorias preservada | Passou | Modelo seções 1.6–1.8. |
| Estados de pedido e cancelamento aprovados refletidos | Passou | Especificação seção 4.4. |
| Estoque baixado somente na confirmação | Passou, com conflito QU-60 destacado | Especificação seção 4.4/5.1; modelo seção 4. |
| Review PRODUCT_ONLY/ONE e campos sem redundância | Passou | Especificação seção 4.5 e modelo seção 1.11. |
| `OrderEvent` e rastreabilidade preservados | Passou | Modelo seção 1.12. |
| Endpoints originais preservados | Passou; conflitos mantidos | Modelo seção 5.1. |
| Endpoints de gestão de restaurante preservados | Passou | Modelo seção 5.2. |
| Único novo endpoint adicional (idempotência) tem decisão explícita | Passou | QU-52 e modelo seção 5.3. |
| Tecnologias/integrações não substituídas | Passou | README atualizado mantém caráter de opções da tabela original. |
| Nenhum código foi implementado/alterado | Passou | Escopo desta entrega é exclusivamente documental. |

## 7. Limites da auditoria

- A auditoria valida a consistência da documentação produzida contra os arquivos-base e o histórico de decisões disponível nesta conversa.
- Ela não substitui revisão jurídica, inspeção de schema/código, execução de testes ou validação em ambiente real.
- Como não existe schema físico/código a revisar nesta tarefa, as constraints são descritas como regras conceituais quando a implementação física ainda não foi definida.
- Não é possível declarar o RNF-01 aprovado em execução: os 130 testes planejados ainda precisam ser realizados e seus resultados preservados.

## 8. Resultado

**Consolidação documental concluída com ressalvas identificadas.** Os requisitos e decisões explicitamente aprovados foram incorporados. As lacunas não decididas foram mantidas como pendências; os impactos de contrato/modelagem que exigem nova aprovação estão destacados. Nenhuma implementação de código foi iniciada.
