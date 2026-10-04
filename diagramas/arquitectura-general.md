# Diagrama de arquitectura general

```mermaid
flowchart TB
    subgraph Cliente
        APP[App Android / KMP]
    end

    subgraph Backend propio
        CAT[catalogo-service<br/>Spring Boot - Hexagonal]
        TUR[turnos-service<br/>Spring Boot - Hexagonal]
        DBC[(Postgres<br/>catalogo_db)]
        DBT[(Postgres<br/>turnos_db)]
    end

    subgraph Catedra
        API[API REST<br/>servicio central]
        REDIS[(Redis<br/>catalogo publicado)]
        KAFKA[[Kafka<br/>eventos de sincronizacion]]
    end

    APP -->|REST + JWT| CAT
    APP -->|REST + JWT| TUR

    CAT --> DBC
    TUR --> DBT

    CAT -->|snapshot / sync incremental| API
    CAT -->|lectura de catalogo| REDIS
    CAT -->|consume eventos de nueva version| KAFKA

    TUR -->|holds, occupancies, appointments| API
    TUR -->|publica / consume eventos de reserva| KAFKA
```

## Qué representa cada parte

- **App KMP**: único punto de entrada para el usuario final. Habla con los dos servicios propios, nunca directo con la cátedra.
- **catalogo-service**: dueño del catálogo local (categorías, profesionales, horarios). Se mantiene sincronizado con la cátedra vía snapshot inicial + eventos Kafka, usando Redis como fuente de lectura del catálogo publicado.
- **turnos-service**: dueño del proceso de reserva (holds, confirmaciones). Orquesta el flujo REST + Kafka contra la cátedra para construir disponibilidad y confirmar turnos.
- **Catedra**: todo lo que no administramos nosotros — es la fuente de verdad externa con la que se sincronizan ambos servicios.

