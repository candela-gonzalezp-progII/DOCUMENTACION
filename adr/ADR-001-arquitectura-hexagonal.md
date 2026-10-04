# ADR-001: Arquitectura hexagonal y estructura de paquetes


## Contexto

Ambos backends (`catalogo-service` y `turnos-service`) necesitan integrarse con múltiples sistemas externos: la API REST de la cátedra, Redis, Kafka y una base de datos propia. El enunciado del proyecto exige aplicar arquitectura hexagonal, principios SOLID y patrones de diseño, y que el código sea defendible en una presentación final.

## Decisión

Se adopta **arquitectura hexagonal (ports & adapters)** en los dos servicios Spring Boot, con la siguiente estructura de paquetes:

```
com.integrador.catalogo
├── domain/model          -> entidades de dominio, Java puro, sin anotaciones de framework
├── application/port/out  -> interfaces que el dominio necesita (ej: repositorios)
└── infrastructure
    └── persistence
        ├── entity        -> entidades JPA (anotadas)
        ├── jpa            -> interfaces Spring Data
        ├── mapper         -> traducción dominio <-> entidad
        └── adapter        -> implementación de los puertos usando JPA
```

- El paquete `domain` no depende de Spring, JPA ni de ningún framework. Valida sus propias reglas en los constructores (ej: `WeeklySchedule` rechaza un rango horario inválido).
- El paquete `application/port/out` define **qué necesita** la aplicación de la infraestructura, sin saber **cómo** se implementa (Dependency Inversion Principle).
- El paquete `infrastructure` contiene el **cómo**: JPA hoy, pero reemplazable por otra tecnología de persistencia sin tocar el dominio.

A medida que se agreguen adaptadores de entrada (controllers REST, consumers Kafka), se va a sumar `application/port/in` e `infrastructure/web` / `infrastructure/messaging` siguiendo el mismo criterio.

## Alternativas consideradas

- **Arquitectura en capas tradicional (controller → service → repository)**: descartada porque mezcla reglas de negocio con detalles de framework, dificultando el testeo del dominio de forma aislada y no es lo que exige el enunciado.
- **Un solo paquete plano por servicio**: descartado por falta de separación de responsabilidades (viola SRP) y porque no permite mostrar con claridad los puertos y adaptadores en la defensa final.

## Consecuencias

- Los tests de dominio (`ProfessionalTest`, `WeeklyScheduleTest`) no necesitan levantar Spring ni una base de datos — corren en milisegundos.
- Los tests de los adaptadores de persistencia se hacen con Testcontainers contra Postgres real, nunca H2, respetando la consigna del enunciado.
- Agregar un patrón como Strategy o State machine (para el ciclo de vida de la reserva en `turnos-service`) va a vivir naturalmente en `domain`, sin ensuciar los adaptadores.