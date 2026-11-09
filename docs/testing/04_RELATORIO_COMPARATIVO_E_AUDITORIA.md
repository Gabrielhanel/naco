# NɅCO — Auditoria documental pós-consolidação

**Versão:** 1.3  
**Data-base:** 09/11/2026  
**Escopo:** consistência entre requisitos, decisões complementares, arquitetura, modelo/API, armazenamento de imagens, cronograma e registro RNF-01. Não é auditoria de código.

## 1. Escopo e critério de avaliação

Esta revisão compara os documentos do ZIP recebido em 09/11/2026 e consolida as decisões já aprovadas até a versão 1.2 do adendo complementar. O objetivo é que todos os documentos-base reflitam a mesma regra, sem duplicar ou reabrir decisões fechadas.

**Fora do escopo desta auditoria:** `docs/api/openapi.yaml`, schema Prisma/PostgreSQL, migrations, backend, frontend, integração executável, CI, infraestrutura e resultados de testes. Esses artefatos ainda serão desenvolvidos e, portanto, não reduzem a nota da documentação de planejamento/especificação nesta etapa.

## 2. Consolidações propagadas

| Tema | Estado vigente refletido nos documentos |
|---|---|
| Conta, papéis e ADMIN | Conta única; CLIENTE automático; RESPONSAVEL mediante solicitação/aprovação; ADMIN único, exclusivo e criado manualmente; trilha `RoleAuditEvent`. Uma conta pode ter CLIENTE + RESPONSAVEL; mobile/web são superfícies por papel, sem seletor de identidade no login. |
| Solicitação RESPONSAVEL | `RoleRequest` com estados PENDING/APPROVED/REJECTED, no máximo uma pendente por usuário, cooldown de 7 dias após rejeição e aprovação/concessão na mesma transação. |
| Transferência | Restaurantes não são apagados ao remover RESPONSAVEL; transferência para outro responsável existente é pré-condição. |
| Sessão | Access JWT de 15 min; refresh opaco rotativo até 30 dias; hash persistido, revogação por logout/reuso e armazenamento de acordo com plataforma. |
| Horários | Timezone IANA, grade semanal ISO 1–7, intervalos `HH:mm`, regras de cruzamento de meia-noite; feriados/exceções fora do MVP. |
| Dinheiro | BRL, `DECIMAL(12,2)` no banco, strings decimais na API; sem `float` binário. |
| Produto/oferta | `display_name` e `normalized_name` calculado no servidor; unicidade `(restaurant_id, normalized_name)`, inclusive ofertas desativadas. |
| Carrinho | Local no cliente, um restaurante por carrinho; checkout revalida preço/estoque; sem entidade/API de carrinho no MVP. |
| Painel | Produtos vendidos = soma de quantidades em pedidos CONCLUIDO; avaliações contadas por data de criação; default 30 dias, máximo 365, timezone do restaurante; sem receita/faturamento. |
| Pedido recebido | GET de lista/detalhes é consulta pura; inbox consolidada disponível; POST de visualização registra primeira leitura; PATCH altera status. |
| Avaliações | PRODUCT_ONLY; POST por OrderItem; GET público/paginado por produto sem login; autor “Cliente verificado”, sem dados pessoais. |
| Idempotência/QU-60 | Criação atômica de IdempotencyRecord + Order + itens sem estoque; confirmação/cancelamento atômicos com estoque + status + evento; baixa só na confirmação. |
| Stack | Monólito modular; Node.js 26 LTS, Express, TypeScript strict, PostgreSQL/Prisma, Redis, React Native, React, Zod, Cloudinary/StoragePort, testes e CI conforme adendo. |
| Imagens | Cloudinary Free via StoragePort; sem cartão, upgrade ou cobrança sem autorização explícita; upload backend-only, processamento/EXIF/WebP, operações persistentes, `destroy` + `invalidate=true` best effort. |

## 3. Correções realizadas nesta consolidação

1. Substituídas as listas de “pendências” obsoletas na especificação, arquitetura e auditoria por itens que realmente ainda exigem contrato, implementação ou operação.
2. Sincronizadas as entidades lógicas `RoleRequest`, `RoleAuditEvent`, `OrderView` e `IdempotencyRecord` no modelo de dados; separada a etapa de schema físico/migrations.
3. Corrigidas as rotas de pedidos recebidos: GET para consulta, `POST /api/orders/{orderId}/views` para visualização idempotente e `PATCH /api/orders/{orderId}/status` para transições.
4. Corrigido o contrato do dashboard para exigir `restaurantId` e validar ownership.
5. Sincronizadas as rotas de avaliações PRODUCT_ONLY e a leitura pública de avaliações por produto.
6. Registrada a resolução posterior de QU-60 sem alterar a regra de estoque.
7. Atualizado o cronograma para incluir leitura pública de avaliações e testes associados.
8. Corrigida a referência à licença: o repositório contém `LICENSE` MIT.
9. Clarificada a margem preventiva de 90% do uso Cloudinary como salvaguarda técnica recomendada/configurável, não como decisão de produto; o bloqueio de novos uploads ao limite gratuito permanece a decisão aprovada.
10. Corrigida a distinção entre contrato lógico decidido e artefatos executáveis ainda futuros (OpenAPI, schema físico, migrations, código e testes).
11. Corrigida a matriz de rastreabilidade RF-08, que ainda dizia incorretamente que a rota de transição de pedidos não estava formalizada; alinhada ao `PATCH /api/orders/{orderId}/status`.
12. Removidas dependências órfãs P-01…P-12 do cronograma, substituídas pelos contratos residuais e políticas operacionais identificados nos documentos; corrigida a dependência de OrderView, já aprovada, e removida uma linha D10 duplicada.

## 4. Itens que continuam a exigir detalhamento

### 4.1. Contratos para a futura OpenAPI

- Rotas/DTOs de consulta e gestão de categorias.
- Path, request, response e erros do upload de imagem pelo responsável.
- Mecanismo operacional/administrativo de reprocessamento de `StorageDeletionOperation`.
- Schema de resposta/erros do POST de visualização e filtros/DTO completo do dashboard.
- Nomes e tipos dos filtros de consulta administrativa de operações de upload.

### 4.2. Operação e segurança

- Política de moderação, denúncia e retenção de avaliações públicas, sem dados pessoais na visualização pública.
- Comportamento fail-safe/degraded quando Redis estiver indisponível, principalmente no rate limiting.
- Monitoramento Cloudinary com cache/frequência conservadora, idade da última leitura, tratamento de erro/dados atrasados e alertas. O patamar preventivo de 90% é recomendação técnica configurável; bloquear novos uploads no limite gratuito é a decisão de produto.
- Evidência/procedimento externo de autorização do proprietário para ações ADMIN sensíveis.
- Guards de navegação por papel e seletor explícito de restaurante na tela de dashboard para responsáveis com vários estabelecimentos.
- Ajustar SLO/RPO/RTO e retenção após staging, preservando os baselines aprovados até então.
- Fixar limites concretos de CPU/memória/timeouts e qualidade WebP na configuração do sistema.

### 4.3. Entregas futuras esperadas

- Criar `docs/api/openapi.yaml` antes do freeze da API executável.
- Criar schema Prisma/PostgreSQL, constraints, índices, seeds e migrations.
- Implementar módulos backend, clientes mobile/web, integração Cloudinary, Redis, observabilidade e CI/CD.
- Executar testes unitários, integração, segurança, carga (130 execuções RNF-01), concorrência, recuperação e homologação; registrar resultados reais.

A ausência desses artefatos é compatível com o estágio de planejamento e não é tratada como falha documental nem como evidência de conclusão.

## 5. Resultado da auditoria

**Nota da documentação: 8,8/10.** A avaliação cobre exclusivamente a qualidade da especificação, arquitetura, rastreabilidade, contratos lógicos, segurança planejada, estratégia de testes e planejamento. Não avalia implementação.

| Dimensão | Nota | Fundamentação |
|---|---:|---|
| Requisitos e regras de negócio | 9,2 | Regras relevantes e critérios de aceite amplos, com regras de estoque, pedidos, papéis e reviews. |
| Rastreabilidade e decisões | 9,0 | Adendo central, decisões e histórico RNF-01; decisões tardias propagadas. |
| Arquitetura e segurança | 9,0 | Monólito modular, limites de módulos, autorização backend, sessão, privacidade e retries. |
| Modelo lógico de dados | 8,7 | Entidades, relações e constraints conceituais claras; schema físico ainda será materializado. |
| Contratos lógicos da API | 8,5 | Principais rotas, métodos, atores e erros definidos; categorias e upload ainda demandam contrato detalhado. |
| Armazenamento de imagens | 9,1 | Fluxo Cloudinary bem definido, operações persistentes, retries, invalidação best effort e preservação da imagem anterior. |
| Estratégia de testes | 8,9 | Critérios quantitativos RNF-01, cenários de integridade, idempotência e concorrência; execução futura. |
| Cronograma | 8,4 | Fases, dependências e DoD definidos; estimativas precisam ser recalibradas durante a execução. |
| Consistência entre documentos | 8,8 | Conflitos principais eliminados; itens futuros explicitamente separados de decisões fechadas. |

## 6. Limites e checklist de validação

- A auditoria é documental: não foram executados código, schema físico, migrations, CI, segurança dinâmica ou campanhas de carga.
- Os arquivos `openapi.yaml`, schema físico, migrations e testes de execução não foram criados nesta atualização.
- PDFs da arquitetura mestre (69 páginas), armazenamento de imagens (14 páginas) e cronograma (10 páginas, regenerado após as correções) foram renderizados e inspecionados visualmente. Não foram identificados cortes ou sobreposições graves; os três usam papel Letter.
- Links locais, destinos dos PDFs, estrutura do ZIP e ausência intencional de `openapi.yaml` foram validados na entrega. O OpenAPI, schema físico, migrations, código e testes executáveis permanecem planejados para desenvolvimento.
