## CU-08: Calcular carga y avance

```mermaid
sequenceDiagram
    autonumber
    participant Sync as Sincronizacion / IndicadorCarga
    participant DB as PostgreSQL 17 (AlmacenPostgres)

    Note over Sync,DB: Incluido en CU-05; sin priorización algorítmica

    Sync->>+DB: consultarPeticionesAbiertasYEtapas(desarrollador, umbral)
    DB-->>-Sync: Peticiones abiertas, etapa actual y Umbral.maxPeticiones (default 6)
    Sync->>Sync: Estimar avance por etapa (ANALISIS 10%, DESARROLLO 40%, CERTIFICACION 70%, DESPLIEGUE 90%, ESTABILIZACION 100%)
    Sync->>Sync: Calcular carga (conteo peticiones abiertas; sin algoritmo de priorización)
    Sync->>+DB: guardarIndicadorCarga(carga, avanceEstimado)
    DB-->>-Sync: IndicadorCarga actualizado

    alt peticionesAbiertas > Umbral.maxPeticiones (default 6)
        Sync->>Sync: Disparar condición de alerta [CU-09]
        Note over Sync,DB: ext: CU-09 Enviar alerta
    end
```
