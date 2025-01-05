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
    Person(Usuario, "Usuário", "Faz solicitações para o sistema")
    Container(API_Gateway, "API Gateway Ocelot", "Gateway", "Gerencia e balanceia as requisições para os microservices")    
}

System_Boundary(Componentes, "") {
    Container_Boundary(PedidosCluster, "Cluster de Microservices Pedidos") {
        Container(MicroservicePedidos1, "Microservice Pedidos", "Microservice", "Processa pedidos e publica eventos")
        
    }

    Container_Boundary(EstoqueCluster, "Cluster de Microservices Estoque") {
        Container(MicroserviceEstoque1, "Microservice Estoque", "Microservice", "Verifica, reserva ou rejeita pedidos de estoque")
        
    }

    Container_Boundary(MessageBroker, "Cluster de Microservices Estoque") {
        Container(MessageBroker1, "RabbitMQ", "Message Broker", "Gerencia eventos entre os microservices")
    }
    
}

BiRel(Usuario, API_Gateway, "Faz requisições para")
BiRel(API_Gateway, MicroservicePedidos1, "Balanceia requisições")
Rel(MicroservicePedidos1, MessageBroker1, "Publica evento")
BiRel(MessageBroker1, MicroserviceEstoque1, "Consome e publica eventos")

UpdateRelStyle(API_Gateway, MicroservicePedidos1, $offsetX="-50", $offsetY="-20")
UpdateRelStyle(MicroservicePedidos1, MessageBroker1, $offsetX="-80", $offsetY="-20")
UpdateRelStyle(MessageBroker1, MicroserviceEstoque1, $offsetX="-80", $offsetY="-20")
UpdateLayoutConfig($c4ShapeInRow="1", $c4BoundaryInRow="2")
```

