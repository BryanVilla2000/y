## CU-02: Filtrar información

```mermaid
sequenceDiagram
    autonumber
    actor User as Usuario (PMO / Técnico / Admin TI)
    participant UI as Componente React (*.jsx)
    participant Api as api.js
    participant Nginx as nginx (/api)
    participant Express as Express (rutas/carga.js)
    participant Negocio as Filtro / MotorCarga
    participant Cache as AlmacenMemoria

    Note over User,Express: Requiere CU-01 (Bearer token válido; si es Técnico aplica restricción soloPropios)

    User->>+UI: Seleccionar filtros (desarrollador, tipo, etapa, proyecto, fuente, fechas)
    UI->>+Api: getCargaFiltrada(parametrosFiltro, token)
    Api->>+Nginx: GET /api/carga?... + Bearer token
    Nginx->>+Express: Proxy /api/carga + validar token/rol
    Express->>+Negocio: aplicarFiltros(criterios, soloPropios)
    Negocio->>+Cache: leerSnapshotEnMemoria()
    Cache-->>-Negocio: Ítems e indicadores en memoria
    Negocio-->>-Express: Colección filtrada + métricas recalculadas
    Express-->>-Nginx: 200 OK (JSON filtrado)
    Nginx-->>-Api: 200 OK
    Api-->>-UI: Datos filtrados para gráficos (Chart.js) y tablas
    UI-->>-User: Actualizar vista filtrada
```
