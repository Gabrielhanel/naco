# NɅCO
#### _Pronuncia-se "NACO"_
### _OBJETIVO_
* O NɅCO é um projeto de codigo aberto voltado para a área alimenticia. Assim como plataformas como o Ifood, o objetivo principal do NɅCO é a praticidade. Com poucos cliques, o usuário consegue realizar pedidos de comida de forma rápida e eficiente.
    * O NɅCO nasceu com o objetivo de ser muito mais do que um simples aplicativo. Para isso, é importante conhecer seu criador; meu nome é Gabriel, sou estudante de Ciências da Computação e apaixonado por tecnologia. Criei esse aplicativo com o objetivo de aplicar meus conhecimentos conforme minha progressão na carreira, além de contribuir com a comunidade, oferecendo uma solução que pode ser adaptada para vários projetos.

### _FUNCIONALIDADES_
_O sistema será capaz de realizar:_

| ID | Funcionalidade
| --- | --- |
| RF-01 | Cadastro e login de usuários |
| RF-02 | Avaliações de estabelecimentos |
| RF-03 | Simulações de pedidos / entregas |
| RF-04 | Visualização de dados por parte dos estabelecimentos |
| RF-05 | Cadastro, edição e exclusão de produtos |
| RF-06 | Cadastro e login de responsáveis por restaurantes |
| RF-07 | Cadastro e manutenção de dados do restaurante |
| RF-08 | Controle de estoque e preços dos produtos |
| RF-09 | Alteração de preços, imagens, nomes dos produtos, descrições e categorias |


_O sistema não terá:_
* Integração com gateway de pagamento
* Autenticação de dois fatores
* Integração operacional com entregadores, rastreamento por GPS ou despacho real.
* Promessa de entrega em prazo real.
* Emissão de documentos fiscais.
* Programa de fidelidade, cupons ou recomendações automatizadas.

### _Permissões_
* Usuários
    * Cadastro e login
    * Avaliação de estabelecimentos
    * Simulações de pedidos / entregas
    * Alteração de seus proprios dados pessoais
* Restaurante
    * Cadastro e login
    * Cadastro e exclusão de produtos
    * Alteração de preços, imagens, nomes dos produtos, descrições e categorias
    * Visualização de dados como quantidade de produtos vendidos, avaliações recebidas

### _Restrições e premissas_
* Pedidos e entregas serão apenas demonstrativos.
* Não terá integração com pagamento real.
* O aplicativo será desenvolvido apenas para dispositivos móveis, e o sistema web será  para visualização de dados, indicadores e gestão de estoque para os restaurantes.
* O código será publicado como código aberto.

### _Regras de negócio_
| ID | Regra de negócio
| --- | --- |
| RN-01 | Cada usuário só poderá realizar pedidos em um restaurante por vez |
| RN-02 | Um restaurante pode ter apenas um produto com o mesmo nome |
| RN-03 | Um responsável só pode gerenciar produtos associados a um restaurante para o qual tenha permissão |
| RN-04 | Cada conta só pode atualizar os próprios dados pessoais |
| RN-05 | Uma avaliação precisa estar associada a um cliente identificado |
| RN-06 | Não deve ser possivel adicionar um produto sem estoque ao carrinho |
| RN-07 |  Um produto excluído não deve aparecer como opção nova em pedidos |
| RN-08 | Um usuário poderá adicionar produtos ao carrinho apenas se o restaurante estiver aberto e se o usuário estiver logado no sistema |
| RN-09 | Ao remover um produto do carrinho, o sistema deve atualizar o valor total do pedido |

### _Estados do pedido_
* Fluxo:
    * CRIADO → CONFIRMADO_RESTAURANTE → EM_PREPARO → SAIU_PARA_ENTREGA → CONCLUÍDO
    * O status "CANCELADO" poderá ser aplicado em qualquer etapa do fluxo, desde que o pedido ainda não tenha sido concluído.

### _Requisitos Não Funcionais_
| ID | Requisito Não Funcional
| --- | --- |
| RNF-01 | As telas mais utilizadas devem responder em tempo aceitável sob uma carga de 1000 usuários simultâneos |
| RNF-02 | Falhas de comunicação exibem mensagem compreensível e não simulam sucesso |

### _Permissões_
| Ação | Visitante | Cliente autenticado | Responsável do restaurante | Administrador da plataforma |
|---|---:|---:|---:|---:|
| Consultar restaurantes e produtos públicos | Sim | Sim | Sim | Sim |
| Criar conta / iniciar sessão | Sim | Sim | Sim | Sim |
| Atualizar dados pessoais próprios | Não | Sim | Sim | Sim |
| Criar avaliação | Não | Sim | Não | Não |
| Criar pedido simulado | Não | Sim | Não | Apenas em ambiente de teste |
| Criar ou editar produtos | Não | Não | Apenas no restaurante autorizado | Sim, mediante autorização prévia |
| Excluir/desativar produtos | Não | Não | Apenas no restaurante autorizado | Sim, mediante autorização prévia |
| Consultar indicadores do restaurante | Não | Não | Apenas no restaurante autorizado | Sim, mediante autorização prévia |
| Moderar contas ou conteúdo | Não | Não | Não | Sim |

###  _Arquitetura proposta_
* Com base em pesquisas, decidimos começar o desenvolvimento iniciando uma arquitetura monolitica e escalar para uma arquitetura de microsserviços conforme surgir a necessidade. Outro motivo seria evitar uma complexidade desnecessária.

### _Comunicação entre os serviços_
```mermaid
flowchart LR
    Cliente[Cliente] --> App[Aplicação mobile NɅCO]
    Admin[Administrador] --> App
    Admin --> Web
    Restaurante[Responsável pelo restaurante] --> App
    Restaurante[Responsável pelo restaurante] --> Web[App web para gestão de dados]
    App --> API[API do NɅCO]
    Web --> API
    API -.-> Cache[(Redis)]
    API --> DB[(Banco de dados)]
```

### _Fluxo de dados de exemplo — criar produto_

```mermaid
sequenceDiagram
    actor R as Responsável
    participant UI as Interface
    participant API as API
    participant Auth as Autorização (Middleware)
    participant DB as Banco de dados
    R->>UI: Preenche os dados do produto
    UI->>API: Envia requisição autenticada
    API->>Auth: Verifica sessão e acesso ao restaurante
    Auth-->>API: Permissão confirmada ou negada
    API->>API: Valida campos e regras de negócio
    API->>DB: Persiste o produto válido
    DB-->>API: Resultado da operação
    API-->>UI: Resposta de sucesso ou erro
    UI-->>R: Atualiza a interface
```

### _Tecnologias e ferramentas_

As escolhas abaixo servem como exemplo de preenchimento, não como uma recomendação definitiva. Compare as opções e registre a razão da decisão antes de instalar serviços ou bibliotecas por hábito.

| Área |  Opção | Critério para decidir |
|---|---|---|
| Interface mobile | React Native com TypeScript | Familiaridade e facilidade de manutenção |
| Interface web | React  com TypeScript | Familiaridade |
| Backend | Node.js + Express.js | Organização da API, simplicidade e manutenção |
| API | REST sobre HTTP (com possibilidade de mudança para Server-Sent Events ou websockets) | Simplicidade dos fluxos e facilidade para documentar/testar |
| Banco de dados | PostgreSQL + Prisma | Familiaridade, maior robustez e integridade de dados (Também cheguei a considerar MySQL mas nesses pontos o PostgreSQL é melhor)  |
| Cache | Redis   | Simplicidade, performance e redução de consultas ao banco de dados |
| Acesso a dados | Prisma   | Simplicidade, tipagem forte e familiaridade |
| Autenticação | Autenticação JWT | Robustez e segurança |
| Imagens | Cloudinary  | Facilidade, otimização |
| Testes de API | Postman | Familiaridade|
| Testes de Carga | Grafana K6 | Estabilidade, escalabilidade e performance |
| Testes de interface | Appium  | Necessidade de validar fluxos reais de navegação |
| Versionamento | Git/Github  | Histórico, revisão e familiaridade |
| CI | GitHub Actions | Reduzir regressões sem criar manutenção excessiva |

### _Mapa inicial de telas_

Tela | Acesso | Conteúdo principal |
---|---|---|
Explorar restaurantes | Visitante/cliente | Lista de estabelecimentos e filtros iniciais |
Detalhes do restaurante | Visitante/cliente | Perfil, categorias, produtos e avaliações |
Detalhe do produto | Visitante/cliente | Descrição, preço e informações relevantes |
Cadastro e login | Visitante | Formulários e mensagens de validação |
Perfil | Cliente | Dados pessoais e edição permitida |
Carrinho/resumo | Cliente | Itens, quantidades, totais e aviso de simulação |
Pedido simulado | Cliente | Confirmação e estado demonstrativo |
Visão geral | Restaurante | Dados sobre vendas e estoque |
Produtos | Restaurante | Listagem, cadastro, edição e exclusão/desativação de produtos |
Avaliações | Restaurante | Notas e comentários recebidos |

## _Organização do frontend_

* A navegação será implementada usando a biblioteca React Navigation, além da implementação de paginação para a listagem de produtos e restaurantes. Entre as paginas, dependendo do contexto, serão compartilhados dados como o token e o ID do usuário, o ID do restaurante e do produto.

* A comunicação com a api será feita usando a biblioteca Axios, que retornará codigos de status HTTP conforme cada caso. Em caso de falha, o usuário será redirecionado para uma tela de erro com as opções de voltar ou tentar novamente (com rate limiting). Com o tempo, a ideia é implementar mensageria em tempo real, tanto para casos de 2 pedidos simultâneos quanto para atualizações de status de pedidos. O cache será responsável por armazenar dados de restaurantes e produtos, atualizando sempre que houver alterações de alguma informação.

* Os formulários validarão os campos obrigatorios e os dados informados tanto via frontend quanto via backend, retornando mensagens de erro ou sucesso. Será avaliado o uso de bibliotecas de validação de formulários, como Formik ou React Hook Form.

* No dispositivo, serão armazenados dados como o token de autenticação, permitindo que o usuário permaneça logado após a saida do aplicativo. Os dados ficarão persistidos no armazenamento local do dispositivo por um periodo indeterminado.

### _Backend e contrato da API_

* O backend será responsável por autenticar usuários, aplicar permissões, validar as regras de negócio, calcular dados confiáveis, persistir as operações e responder com erros consistentes.

#### _Rotas iniciais_

| Método | Rota ilustrativa | Objetivo | Acesso esperado |
|---|---|---|---|
| POST | `/api/auth/register` | Criar conta | Público, com rate limiting e proteção contra SQL injection |
| POST | `/api/auth/login` | Iniciar sessão | Público, com rate limiting e proteção contra SQL Injection |
| POST | `/api/auth/logout` | Encerrar sessão | Autenticado |
| GET | `/api/restaurants` | Listar estabelecimentos | Público |
| GET | `/api/restaurants/:id` | Consultar um estabelecimento | Público |
| GET | `/api/restaurants/:id/products` | Consultar catálogo | Público |
| POST | `/api/restaurants/:id/products` | Criar produto | Responsável pelo estabelecimento |
| PATCH | `/api/products/:id` | Atualizar produto | Responsável pelo estabelecimento |
| DELETE | `/api/products/:id` | Excluir ou desativar produto | Responsável pelo estabelecimento |
| POST | `/api/restaurants/:id/reviews` | Criar avaliação | Cliente autenticado |
| POST | `/api/orders/create` | Criar pedido  | Cliente autenticado |
| GET | `/api/orders/` | Consultar pedidos do cliente  | Cliente autenticado |
| GET | `/api/restaurants/:restaurantId/orders` | Receber e aceitar pedido  | Cliente proprietário |
| GET | `/api/restaurant/dashboard` | Consultar resumo do painel | Responsável autorizado |

### _Formato de resposta_

#### _Erros:_
* Exemplo conceitual de erro de validação para cadastro de produtos:

```json
{
  "error": {
    "code": "422", // HTTP 422 Unprocessable Entity
    "message": "Revise os campos informados.",
    "fields": {
      "price": "O valor precisa ser maior que zero."
    }
  }
}
```

* Exemplo conceitual de erro de validação para login:

```json
{
    "error": {
        "code": "401", // HTTP 401 Unauthorized
        "message": "Usuário ou senha inválidos."
    }
}
```
* Exemplo conceitual de acesso ao aplicativo sem conexão com internet:

```json
{
    "error": {
        "code": "503", // HTTP 503 Service Unavailable
        "message": "Não foi possível estabelecer conexão com o servidor. Verifique sua conexão com a internet e tente novamente."
    }
}
```
#### _Sucesso:_

* Exemplo conceitual de sucesso de login:

```json
{
    "data": {
            "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6IjEyMyIsIm5hbWUiOiJKb2huIERvZSIsImlhdCI6MTY4NzQyMDAwMCwiZXhwIjoxNjg3NDIzNjAwfQ.abc123def456ghi789jkl012mno345pqr678stu901vwx234yz567890",
            "user": {
                "id": "123",
                "name": "John Doe",
                "email": "0Mv1o@example.com"
            }
        }
}
```

* Exemplo conceitual de sucesso de cadastro de produto:

```json
{
    "data": {
            "id": "456",
            "name": "Pizza Margherita",
            "description": "Deliciosa pizza com molho de tomate, mussarela e manjericão.",
            "price": 19.99,
            "situation": "active",
            "restaurant": {
                "id": "789",
                "name": "Pizzaria do João"
            }
            }
}
```

* Exemplo conceitual de sucesso de pedido:

```json
{
    "data": {
            "id": "789",
            "status": "CRIADO",
            "total": 39.98,
            "items": [
                {
                    "productId": "456",
                    "name": "Pizza Margherita",
                    "quantity": 2,
                    "price": 19.99
                }
            ],
            "restaurant": {
                "id": "789",
                "name": "Pizzaria do João"
            }
        }
}
```