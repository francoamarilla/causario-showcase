# Arquitectura

## Componentes

Tres procesos separados. Si una PC está apagada o el portal cambia, la aplicación sigue funcionando; solo faltan novedades nuevas.

```mermaid
flowchart TB
    subgraph Cliente
        N[Navegador<br/>Angular + Material]
        E[App de escritorio<br/>Electron]
        AG[Agente Java<br/>login + cliente HTTP del portal]
        E --- AG
    end
    subgraph Nube
        WEB[nginx<br/>estáticos + proxy /api]
        API[API<br/>Spring Boot]
        WK[Worker<br/>mails, Google Calendar,<br/>días inhábiles]
        DB[(PostgreSQL 16<br/>RLS + cola de trabajos)]
        WEB --> API
        API --> DB
        WK --> DB
    end
    PORTAL[Portal judicial<br/>solo lectura]
    N -->|HTTPS + JWT| WEB
    E -->|ventana con la misma web| WEB
    AG -->|HTTPS + token de PC| API
    AG -->|IP del estudio| PORTAL
```

- **API:** autenticación, ABM, permisos y la API que usan las PCs. Es el único proceso que corre las migraciones.
- **Worker:** encola sincronizaciones, manda mails, sincroniza Google Calendar y el calendario de días inhábiles. No toca el portal.
- **Agente:** corre en las PCs del estudio. Hace el login, mantiene la sesión en memoria y consulta el portal. No tiene acceso a la base: todo pasa por la API con un token revocable.
- **Cola:** es una tabla de Postgres, sin broker de mensajes. Es suficiente para el volumen de estudios chicos.

## Capas de aislamiento entre estudios

1. Filtro por estudio en la capa de persistencia (discriminador de Hibernate).
2. Claves foráneas compuestas (`id`, `estudio_id`), para que un dato no pueda vincularse a otro estudio.
3. Row Level Security en PostgreSQL: la aplicación se conecta con un rol que no es dueño de las tablas, así que la base filtra aunque una consulta se olvide del filtro.

El estudio sale siempre del JWT validado en el servidor, nunca de un parámetro del request.

## Sincronización de causas

```mermaid
sequenceDiagram
    participant W as Worker
    participant DB as PostgreSQL
    participant A as Agente (PC del estudio)
    participant API as API
    participant P as Portal judicial

    W->>DB: encolar trabajos por cuenta (cada 2 h)
    loop cada ~2 s
        A->>API: pedir trabajo (token de PC)
        API->>DB: tomar un trabajo, un solo trabajo por cuenta a la vez
        API-->>A: trabajo (o nada)
    end
    A->>P: ¿sesión viva?
    alt sesión vencida
        A->>P: login (un solo intento)
    end
    A->>P: novedades, operaciones, cédulas (pausa de 1 a 2 s entre llamadas)
    A->>API: guardar expedientes, movimientos, cédulas
    API->>DB: insertar sin duplicar
    A->>API: cerrar trabajo (ok o error)
    API->>DB: actualizar estado de la cuenta
```

Si el trabajo no avisa que sigue vivo (por ejemplo, la PC se suspendió), la nube lo devuelve a la cola y lo toma otra PC.

## Despliegue

- **Nube:** `docker-compose` con cuatro servicios (base, API, worker, nginx) y un servicio de backup. Solo nginx recibe tráfico de afuera, detrás del proxy que termina TLS. La base y la API quedan en la red interna.
- **Estudio:** cada PC instala la app de escritorio (instalador de Windows con actualización automática) y la vincula con su cuenta mediante un token que se muestra una sola vez.

```mermaid
flowchart LR
    subgraph Estudio
        PC1[PC 1<br/>app de escritorio]
        PC2[PC 2<br/>navegador]
        PC3[PC 3<br/>navegador]
    end
    PROXY[Proxy con TLS]
    subgraph Servidor
        WEB[nginx] --> API[API]
        API --> DB[(PostgreSQL)]
        WK[Worker] --> DB
        BK[Backup diario<br/>cifrado] --> DB
    end
    PC1 & PC2 & PC3 --> PROXY --> WEB
    BK -->|copia fuera del servidor| S3[(Almacenamiento externo)]
```
