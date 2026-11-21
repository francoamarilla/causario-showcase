# CI/CD y distribución de la aplicación de escritorio

[English version](en/ci-cd.md)

## Objetivo

Automatizar la validación del código y la distribución de nuevas versiones de Causario, reduciendo el trabajo manual necesario para mantener las instalaciones del estudio.

## Flujo general

- **Integración continua:** GitHub Actions ejecuta pruebas automatizadas.
- **Construcción y empaquetado:** se generan instaladores de la aplicación de escritorio Electron para Windows.
- **Distribución:** la aplicación instalada cuenta con un mecanismo de actualización automática para recibir nuevas versiones.

```mermaid
flowchart LR
    C[Cambios de código] --> T[GitHub Actions: pruebas]
    T --> B[Construcción y empaquetado]
    B --> I[Instalador Windows]
    I --> V[Distribución de versiones]
    V --> U[Actualización de la app instalada]
```

El diagrama representa el flujo conceptual de validación y distribución, no la configuración exacta de los workflows.

## Alcance

La documentación pública describe las capacidades de CI/CD, los instaladores y la actualización automática. Los workflows, disparadores, condiciones de publicación, gestión de artefactos y configuración del actualizador pertenecen al repositorio privado y no se detallan aquí.

Para conocer los componentes y el despliegue del sistema, consultar [Arquitectura](arquitectura.md).
