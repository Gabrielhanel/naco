# NɅCO — Registro de decisões e complementações do RNF-01

**Nota de origem:** QU-01 a QU-19 detalham critérios centrais do RNF-01; QU-20 a QU-73 são complementações técnicas propostas durante a análise e aceitas pelo usuário. Elas não são questões originalmente numeradas no documento-base. QU-57 a QU-73 foram aceitas em conjunto por autorização expressa para prosseguir com todas as recomendações apresentadas.

> **Leitura do registro:** as linhas QU preservam a formulação aprovada no momento da decisão. Quando uma questão foi posteriormente esclarecida no adendo complementar v1.2, a nota histórica na própria linha indica que a interpretação vigente é a do adendo. Não tratar ressalvas históricas como pendências atuais.

## QU-01 a QU-19 — Critérios centrais de desempenho

| ID | Decisão aprovada |
|---|---|
| QU-01 | Operações medidas: listar restaurantes; detalhes de restaurante; listar produtos; detalhes de produto; login/autenticação; criar pedido; pedidos do cliente; pedidos recebidos pelo restaurante; painel de gerenciamento. Não adicionar outra operação sem revisão. **Nota histórica:** ator/finalidade/método da listagem de pedidos recebidos foram corrigidos posteriormente pelo adendo complementar v1.2. |
| QU-02 | P95: até 2 s para operações comuns; até 3 s para criação de pedido. |
| QU-03 | Taxa máxima de erros técnicos de 1%; contar HTTP 5xx, timeouts e falhas inesperadas; excluir recusas legítimas como 409 de estoque insuficiente. 401 inesperado em fluxo autenticado deve ser investigado. |
| QU-04 | Carga sustentada por 60 minutos. |
| QU-05 | P99: até 4 s para operações comuns; até 6 s para criação de pedido. |
| QU-06 | Ramp-up de 10 minutos até 1.000 usuários simultâneos; depois 60 minutos sustentados. Duração mínima oficial de 70 minutos, fora preparação/aquecimento/encerramento. |
| QU-07 | Distribuição de carga qualitativa realista, com mais consultas/listagens e menos pedidos. |
| QU-08 | Distribuição de requisições: listar restaurantes 20%; detalhes restaurante 10%; listar produtos 20%; detalhes produto 10%; login 5%; criar pedido 5%; pedidos cliente 10%; pedidos recebidos restaurante 10%; painel 10%. Total 100%. Modelar sessão/VUs sem novo login a cada chamada. |
| QU-09 | Ambiente de staging/homologação isolado e representativo de produção. |
| QU-10 | Documentar recursos efetivos por execução: CPU, RAM, instâncias, banco/versão/recursos/conexões, OS, runtime/dependências, rede, gerador, data, versão da aplicação e configuração; verificar representatividade. |
| QU-11 | Usar dados progressivos em volumes pequeno, médio e grande. |
| QU-12 | Volumes: usuários 500/5.000/20.000; restaurantes 20/100/500; produtos/ofertas 500/5.000/25.000; pedidos 2.000/20.000/100.000; avaliações 500/5.000/25.000. São dados cadastrados, não simultaneidade. |
| QU-13 | Estados de pedidos no dataset: CRIADO 10%; CONFIRMADO_RESTAURANTE 10%; EM_PREPARO 10%; SAIU_PARA_ENTREGA 10%; CONCLUIDO 45%; RECUSADO 5%; CANCELADO 10%. Gerar por transições válidas e temporalmente coerentes. |
| QU-14 | Paginação: 20 padrão, 100 máximo; testar primeira e páginas profundas; acima do máximo rejeitar ou limitar consistentemente; ordenação estável com desempate único; paginação não substitui índices. |
| QU-15 | Se houver cache, separar cache frio/aquecido, limpar previamente o frio, medir aquecimento e hit/miss. Não exige cache nem tecnologia específica. |
| QU-16 | Intervalo aleatório de 1–5 s entre ações do mesmo VU; não determina RPS. Registrar RPS efetivo. |
| QU-17 | Cinco execuções por combinação volume/cache: 3 × 2 × 5 = 30 execuções oficiais; preservar resultados individuais e analisar média, mediana, dispersão e percentis. |
| QU-18 | Mesmos critérios de latência e erros em todos os cenários, inclusive cache frio e todos os volumes. |
| QU-19 | Todas as cinco execuções de cada combinação devem cumprir individualmente. Percentis por operação dentro de cada execução; não mascarar operação lenta por média agregada. Uma execução reprovada reprova a combinação. |

## QU-20 a QU-28 — RPS e concorrência de estoque

| ID | Decisão aprovada |
|---|---|
| QU-20 | Medir/documentar RPS efetivo, total e por operação, incluindo variação, concluídas, falhas, timeouts e relação com latência/VUs. Não há RPS mínimo. |
| QU-21 | Criar cenários específicos de disputa simultânea por estoque, além da carga geral. |
| QU-22 | Níveis de 10, 25, 50 e 100 tentativas simultâneas pela mesma oferta. |
| QU-23 | Estoque inicial igual a 50% da demanda, arredondado para baixo, uma unidade por tentativa: 5, 12, 25 e 50 unidades, respectivamente. |
| QU-24 | Dez repetições por nível; total de 40 execuções. Restaurar estoque antes de cada execução. |
| QU-25 | Usar barreira para liberação simultânea; registrar liberação e início efetivo. Sincronização do gerador não garante simultaneidade no banco. |
| QU-26 | Latência de concorrência: P95 até 3 s e P99 até 6 s desde a liberação da barreira até resposta HTTP. |
| QU-27 | Tentativas sem estoque recebem HTTP 409 e não consomem unidades. 409 de recusa legítima não é erro técnico. |
| QU-28 | Após cada execução, reconciliar pedidos, respostas e estoque; sem duplicação, estoque negativo, consumo indevido ou alteração parcial. |

## QU-29 a QU-39 — Idempotência, atomicidade e falhas

| ID | Decisão aprovada |
|---|---|
| QU-29 | Reenvio da mesma solicitação não pode duplicar pedido nem consumo; testar reenvios sequenciais e simultâneos. |
| QU-30 | Cliente gera `Idempotency-Key` para cada intenção lógica; nova intenção usa nova chave; chave associada ao usuário e ao resultado; payload diferente com mesma chave deve ser rejeitado (409 recomendado e depois aprovado na QU-56); persistência/atomicidade com pedido e estoque deve ser garantida, observada a ressalva QU-60. |
| QU-31 | Dados completos idempotentes retidos por 24 horas. Após expiração, requisição antiga não pode gerar compra inesperada. |
| QU-32 | Para mesma chave simultânea, segunda chamada aguarda a primeira. Se exceder limite, comunicar processamento sem segunda operação. Não manter transação/bloqueio do banco aberto enquanto aguarda. |
| QU-33 | Espera de até 5 s; se primeira concluir, devolver o mesmo resultado; senão responder como processamento. A espera não cancela primeira operação; resultado continua recuperável. |
| QU-34 | Injetar falhas em todos os pontos críticos antes/durante/depois da gravação do pedido e atualização de estoque. |
| QU-35 | Rollback integral se falha antes do commit, sem alteração parcial; se commit ocorreu e resposta HTTP se perdeu, recuperação pela idempotência. |
| QU-36 | Injeção controlada de falhas de banco, interrupção do processo e perda de resposta HTTP. Distinguir falha pré-commit de resposta perdida pós-commit. |
| QU-37 | Após reinicialização, verificar consistência; preservar transações confirmadas, descartar incompletas e recuperar operações por chave. |
| QU-38 | Evidência por execução: cenário/ponto de falha, horário, chave, respostas HTTP, logs correlacionados, estado de pedido/estoque antes/depois e reconciliação, sem dados sensíveis. |
| QU-39 | Zero inconsistências: divergência, pedido duplicado, estoque negativo ou consumo indevido reprova a execução. Erros técnicos não reprovam automaticamente se integridade e demais critérios passam; taxa de erro segue QU-03. |

## QU-40 a QU-50 — Preparação, observabilidade e overhead

| ID | Decisão aprovada |
|---|---|
| QU-40 | Restaurar estado inicial controlado antes de cada execução; validar dados/isolamento e preservar volume, distribuição de status e cache definidos. |
| QU-41 | Monitorar CPU, memória, rede, conexões/consultas DB, pool, disco e métricas da aplicação. |
| QU-42 | Métricas de recurso são diagnósticas, sem limites arbitrários; reprovação segue critérios funcionais/desempenho já definidos. |
| QU-43 | Validar gerador para 1.000 VUs sem gargalo; geração distribuída permitida se necessário; documentar topologia e validar perfil/sincronização/carga. |
| QU-44 | Medir overhead da instrumentação, manter configuração consistente e documentar impacto; não subtrair overhead automaticamente dos resultados oficiais. |
| QU-45 | Comparar instrumentação completa e mínima em condições equivalentes: mesma infraestrutura, app, dados, carga, cache, duração e repetições. Mínima preserva rastreabilidade básica. |
| QU-46 | Cinco pares por combinação: 3 volumes × 2 cache × 5 pares = 30 pares/60 execuções adicionais. |
| QU-47 | Alternar ordem entre pares: completa/mínima, depois mínima/completa, repetindo o padrão. |
| QU-48 | Antes de cada execução, aguardar estabilização: sem requisições pendentes, filas residuais e com recuperação dos recursos. Separar de restauração de dados. |
| QU-49 | Calcular percentual de overhead por par; manter resultados individuais, média, mediana e dispersão. Analisar cada métrica separadamente. |
| QU-50 | Overhead como indicador diagnóstico, sem limite fixo ou reprovação automática; investigar e documentar impactos relevantes. |

## QU-51 a QU-56 — Fluxo de consulta e escopo da chave

| ID | Decisão aprovada |
|---|---|
| QU-51 | Se a segunda requisição com mesma chave não obtiver resultado dentro de 5 s, responder HTTP 202 Accepted, indicar processamento e fornecer referência de consulta. Primeira operação continua. |
| QU-52 | Criar endpoint específico `GET /api/orders/idempotency/{key}` para consultar estado e resultado; exige autenticação e valida titularidade. |
| QU-53 | Após expiração, manter registro mínimo da chave e rejeitar novas requisições com ela; nova intenção exige nova chave. |
| QU-54 | Registro mínimo/tombstone permanece armazenado permanentemente enquanto o sistema existir. |
| QU-55 | Unicidade por usuário: combinação `user_id + idempotency_key`. |
| QU-56 | Mesma chave com payload diferente retorna HTTP 409 Conflict e não executa nova operação. |

## QU-57 a QU-62 — Payload e persistência idempotente

| ID | Decisão aprovada |
|---|---|
| QU-57 | Normalizar payload, ordenar itens deterministicamente e calcular hash dos campos relevantes. |
| QU-58 | Considerar restaurante, produtos, quantidades e demais campos que efetivamente alterem o pedido; ignorar formatação irrelevante. Lista exata de campos permanece para contrato/schema. |
| QU-59 | Persistir chave, usuário, hash do payload, estado do processamento e referência/resultado do pedido. |
| QU-60 | Garantir atomicidade do registro idempotente, criação do pedido e operações relacionadas ao estoque. **Nota histórica:** a ressalva foi resolvida posteriormente pelo adendo complementar v1.2 em duas fronteiras transacionais, mantendo estoque sem baixa na criação e baixa somente na confirmação. |
| QU-61 | Diferenciar internamente chave inexistente, expirada, pertencente a outro usuário e operação em processamento, sem expor informações indevidas. |
| QU-62 | Para mesma chave concorrente, garantir processamento único; demais requisições recuperam o mesmo resultado ou estado. |

## QU-63 a QU-67 — Concorrência entre ações de pedido

| ID | Decisão aprovada |
|---|---|
| QU-63 | Testar confirmações simultâneas de pedidos disputando a mesma oferta, respeitando baixa no momento da confirmação. |
| QU-64 | Testar concorrência entre confirmação e cancelamento, evitando baixa/restauração duplicada. |
| QU-65 | Testar cancelamentos simultâneos do mesmo pedido; estoque não pode ser restaurado duas vezes. |
| QU-66 | Testar transições simultâneas de status; preservar máquina de estados e permissões. |
| QU-67 | Após operações concorrentes, reconciliar pedidos, eventos, estoque e histórico. |

## QU-68 a QU-70 — Recuperação

| ID | Decisão aprovada |
|---|---|
| QU-68 | Injetar falhas antes, durante e após confirmação, baixa de estoque e gravação de eventos. |
| QU-69 | Validar recuperação e consistência após reinicialização do processo. |
| QU-70 | Recuperar pela chave a operação cujo commit ocorreu, mas cuja resposta HTTP foi perdida. |

## QU-71 a QU-73 — Relatório e encerramento

| ID | Decisão aprovada |
|---|---|
| QU-71 | Relatório consolidado com resultados, percentis, erros, RPS, recursos e overhead. |
| QU-72 | Preservar rastreabilidade por execução: configuração, versão, dados, logs e evidências. |
| QU-73 | Considerar RNF-01 validado somente após executar e avaliar todos os cenários e critérios aprovados. |

## Resumo quantitativo

| Campanha | Quantidade |
|---|---:|
| Carga oficial | 30 |
| Comparação de overhead | 60 adicionais |
| Disputa concorrente de estoque | 40 |
| **Total planejado** | **130** |

As quantidades são planejamento de testes, não evidência de execução.

## Decisões complementares aprovadas — produto e mídia (09/11/2026)

| ID | Decisão aprovada |
|---|---|
| DEC-PROD-REV-01 | Avaliações continuam `PRODUCT_ONLY`, uma por `OrderItem` de pedido próprio `CONCLUIDO`, nota 1–5 e comentário opcional; leitura de avaliações e comentários é pública, inclusive para visitantes sem login. O autor é exibido como “Cliente verificado”, sem dados pessoais. |
| DEC-MEDIA-01 | Cloudinary é o fornecedor de mídia do MVP, priorizando o plano Free sem cartão de crédito. O backend mantém validação, sanitização, conversão WebP e controle do ciclo de vida; integração fica atrás de `StoragePort`. |
| DEC-MEDIA-02 | Ao atingir o limite interno de uso gratuito, bloquear novos uploads, alertar ADMIN e preservar imagens existentes. Não usar fallback automático para outro fornecedor. |
| DEC-BILLING-01 | Não cadastrar cartão, contratar plano pago, executar upgrade ou gerar cobrança sem autorização explícita do proprietário do projeto. |
| DEC-MEDIA-03 | Q77–Q86 são adaptadas ao Cloudinary: usar `destroy` com `invalidate=true`; persistir operação e retries transitórios; não alegar confirmação global de purge nem polling por ID/status que o contrato padrão não oferece. Aceita-se invalidação best effort e possível permanência temporária em caches externos. |
