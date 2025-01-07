# Reserva de Estoque Demo
Essa implementação busca exemplificar a Arquitetura de Microsserviços como solução para cenários de alta disponibilidade e alta volumetria. 

Nessa demonstração vamos construir uma aplicação simplificada para controlar a reserva de estoque durante o processamento de pedidos de compra. 

Componentes:
- API Gateway Ocelot: A funcionalidade principal de um Gateway de API Ocelot é obter solicitações HTTP de entrada e encaminhá-las para um serviço downstream, simultaneamente à outra solicitação HTTP 
- Microsserviço Pedido: Responsável por gerenciar os pedidos de compras. Nesse exemplo simplificado faremos apenas as camadas responsáveis por criar pedidos, publicar eventos de novos pedidos, consumir eventos de atualização de status e permitir a consulta de status.
-  Microsserviço Estoque: Responsável por gerenciar o estoque de produtos. Nesse exemplo simplificado faremos apenas as camadas responsáveis por consultar o estoque atual de cada produto fazendo a reserva quando possível e publicando o evento informando o status do pedido.

Requisitos
- Todos os microsserviços precisam ser escaláveis horizontalmente
- A reserva de estoque precisa garantir que, mesmo em um cenário de concorrência e paralelismo, o estoque seja controlado corretamente
- Um pedido só será aceito quando houver saldo em estoque para dodos os itens
- As reservas terão um prazo de 30 minutos, após esse tempo será necessário uma nova consulta ao estoque antes de fechar o pedido.


# Diagrama de Sequência
```mermaid
sequenceDiagram
    participant Usuário
    participant MicroservicePedidos as Microservice "Pedidos"
    participant RabbitMQ
    participant MicroserviceEstoque as Microservice "Estoque"

    Usuário->>MicroservicePedidos: HTTP POST /solicitação de compras
    MicroservicePedidos->>RabbitMQ: Publica evento "PedidoCriado"
    RabbitMQ-->>MicroserviceEstoque: Evento "PedidoCriado"
    MicroserviceEstoque->>MicroserviceEstoque: Verifica estoque
    alt Estoque disponível
        MicroserviceEstoque->>MicroserviceEstoque: Atualiza status para "EstoqueReservado"
        MicroserviceEstoque->>RabbitMQ: Publica evento "AtualizarStatus"
    else Estoque indisponível
        MicroserviceEstoque->>MicroserviceEstoque: Atualiza status para "EstoqueNaoDisponivel"
        MicroserviceEstoque->>RabbitMQ: Publica evento "AtualizarStatus"
    end
    RabbitMQ-->>MicroservicePedidos: Evento "AtualizarStatus"
    Usuário->>MicroservicePedidos: GET /statusPedido
    MicroservicePedidos->>Usuário: Retorna status do pedido
```

# Arquitetura de Componentes
```mermaid
C4Context
title Arquitetura do Sistema de Reserva de Estoque

Container_Boundary(Gateway, "API Gateway Ocelot") {    
    Person(Usuario, "Usuário", "Faz Pedidos")
    Container(API_Gateway, "API Gateway", "Ocelot", "Gerencia requisições para os microservices")
    Container(Consul, "Service Discovery", "Consul", "Gerencia service discovery dos microservices")    
}

System_Boundary(Componentes, "") {
    Container_Boundary(PedidosCluster, "Cluster de Microservices Pedidos") {
        ContainerDb(MicroservicePedidosDB, "Banco de Dados Pedidos", "PostgreSQL")
        Container(MicroservicePedidos1, "Microservice Pedidos", "Microservice", "Processa pedidos")
    }

    Container_Boundary(EstoqueCluster, "Cluster de Microservices Estoque") {
        ContainerDb(MicroserviceEstoqueDB, "Banco de Dados Estoque", "PostgreSQL")
        Container(MicroserviceEstoque1, "Microservice Estoque", "Microservice", "Processa estoque")        
    }

    Container_Boundary(MessageBroker, "Cluster de Microservices Estoque") {
        Container(MessageBroker1, "RabbitMQ", "Message Broker", "Gerencia eventos")
    }    
}

BiRel(Usuario, API_Gateway, "Faz requisições para")
BiRel(API_Gateway, Consul, "Service Discovery Microservices")
BiRel(API_Gateway, MicroservicePedidos1, "Balanceia requisições")
Rel(MicroservicePedidos1, MessageBroker1, "Publica evento")
BiRel(MessageBroker1, MicroserviceEstoque1, "Consome e publica eventos")
Rel(MicroservicePedidos1, MicroservicePedidosDB, "Armazena Dados de Pedidos")
Rel(MicroserviceEstoque1, MicroserviceEstoqueDB, "Armazena Dados de Estoque")

UpdateRelStyle(API_Gateway, MicroservicePedidos1, $offsetX="-50", $offsetY="-20")
UpdateRelStyle(MicroservicePedidos1, MessageBroker1, $offsetX="-44", $offsetY="-20")
UpdateRelStyle(MessageBroker1, MicroserviceEstoque1, $offsetX="-80", $offsetY="-20")
UpdateRelStyle(MicroservicePedidos1, MicroservicePedidosDB, $offsetX="-80", $offsetY="0")
UpdateRelStyle(MicroserviceEstoque1, MicroserviceEstoqueDB, $offsetX="-80", $offsetY="0")
UpdateRelStyle(Usuario, API_Gateway, $offsetX="-60", $offsetY="0")
UpdateRelStyle(API_Gateway, Consul, $offsetX="-90", $offsetY="0")

UpdateLayoutConfig($c4ShapeInRow="1", $c4BoundaryInRow="2")
```

