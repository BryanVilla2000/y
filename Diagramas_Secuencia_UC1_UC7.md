# Diagramas de Secuencia (UC1 - UC7)

Proyecto Final - Casos Padre Julián Lorente

---

## UC1: Iniciar sesión con cuenta de la cooperativa

```mermaid
sequenceDiagram
    autonumber
    actor User as Usuario
    participant UI as Frontend Web (UI)
    participant Auth as Módulo de Autenticación (Usuario)
    database BD as Base de Datos Central
    participant SSO as SSO Cooperativa

    User->>+UI: Ingresar credenciales institucionales
    UI->>+Auth: validarAcceso(usuario, clave)
    Auth->>+SSO: autenticarInstitucional(credenciales)

    alt Credenciales válidas
        SSO-->>-Auth: Token SSO válido + datos básicos
        Auth->>+BD: obtenerRolYPermisos(idUsuario)
        BD-->>-Auth: Rol (Desarrollador / LiderTecnologia / Administrador)
        Auth-->>UI: Sesión iniciada + perfil y rol
        UI-->>User: Redirigir a panel según rol
    else Credenciales inválidas
        SSO-->>Auth: Error de autenticación
        Auth-->>-UI: Acceso denegado
        UI-->>-User: Mostrar mensaje de error
    end
```

---

## UC2: Visualizar carga individual de trabajo

```mermaid
sequenceDiagram
    autonumber
    actor Dev as Desarrollador
    participant UI as Frontend Web (UI)
    participant Motor as Motor de Carga (CargaDesarrollador)
    database BD as Base de Datos Central

    rect rgba(2, 132, 199, 0.08)
        Note over Dev,UI: ref: UC1 - Iniciar sesión con cuenta de la cooperativa
    end

    Dev->>+UI: Solicitar vista "Mi Carga Individual"

    rect rgba(2, 132, 199, 0.08)
        Note over UI,BD: ref: UC6 - Sincronizar datos desde herramientas externas
    end

    UI->>+Motor: obtenerCargaIndividual(idDesarrollador)
    Motor->>+BD: consultarItemsAsignados(idDesarrollador)
    BD-->>-Motor: Lista de Peticion, IncidenteGLPI y MergeRequest
    Motor->>Motor: calcular() [totalItems, horasEstimadas, nivelCarga]
    Motor-->>-UI: Objeto CargaDesarrollador
    UI-->>-Dev: Renderizar panel individual con métricas y tareas

    opt Si el desarrollador aplica filtros
        Note over Dev,Motor: ref: UC5 - Filtrar información en el panel
    end
```

---

## UC3: Visualizar carga consolidada del equipo

```mermaid
sequenceDiagram
    autonumber
    actor Lider as Líder de tecnología
    participant UI as Frontend Web (UI)
    participant Motor as Motor de Carga (PanelCarga)
    database BD as Base de Datos Central

    rect rgba(2, 132, 199, 0.08)
        Note over Lider,UI: ref: UC1 - Iniciar sesión con cuenta de la cooperativa
    end

    Lider->>+UI: Solicitar "Panel Consolidado del Equipo"

    rect rgba(2, 132, 199, 0.08)
        Note over UI,BD: ref: UC6 - Sincronizar datos desde herramientas externas
    end

    UI->>+Motor: generar()
    Motor->>+BD: obtenerDesarrolladoresYItemsActivos()
    BD-->>-Motor: Desarrolladores, Peticion, IncidenteGLPI, MergeRequest

    loop Por cada desarrollador del equipo
        Motor->>Motor: calcular() [CargaDesarrollador]
    end

    Motor-->>-UI: PanelCarga (comparativa y alertas de sobrecarga)
    UI-->>-Lider: Mostrar dashboard consolidado del equipo

    opt Si el líder aplica filtros
        Note over Lider,Motor: ref: UC5 - Filtrar información en el panel
    end
```

---

## UC4: Consultar detalle y trazabilidad de ítems

```mermaid
sequenceDiagram
    autonumber
    actor User as Usuario (Desarrollador / Líder)
    participant UI as Frontend Web (UI)
    participant Motor as Motor de Carga
    database BD as Base de Datos Central

    rect rgba(2, 132, 199, 0.08)
        Note over User,UI: ref: UC1 - Iniciar sesión con cuenta de la cooperativa
    end

    User->>+UI: Seleccionar ítem (Petición, Incidente o MR)

    rect rgba(2, 132, 199, 0.08)
        Note over UI,BD: ref: UC6 - Sincronizar datos desde herramientas externas
    end

    UI->>+Motor: obtenerDetalleItem(idItem, tipo)
    Motor->>+BD: consultarDetalleCompleto(idItem, tipo)
    BD-->>-Motor: Atributos del ítem + Pipeline asociado (si es MR)
    Motor-->>-UI: Ficha de trazabilidad + urlExterna
    UI-->>-User: Mostrar detalle completo y enlace a herramienta origen
```

---

## UC5: Filtrar información en el panel

```mermaid
sequenceDiagram
    autonumber
    actor User as Usuario (Desarrollador / Líder)
    participant UI as Frontend Web (UI)
    participant Motor as Motor de Carga (PanelCarga)
    database BD as Base de Datos Central

    User->>+UI: Seleccionar criterios (proyecto, estado, fuente, fechas)
    UI->>+Motor: aplicarFiltros(filtrosAplicados)
    Motor->>+BD: consultarItemsFiltrados(filtrosAplicados)
    BD-->>-Motor: Colección filtrada de ítems
    Motor->>Motor: calcular() [recalcular métricas visibles]
    Motor-->>-UI: Datos y métricas actualizadas
    UI-->>-User: Actualizar tablas y gráficos en pantalla
```

---

## UC6: Sincronizar datos desde herramientas externas

```mermaid
sequenceDiagram
    autonumber
    participant Trigger as Motor de Carga / Planificador
    participant Sync as Servicio de Integraciones (Integracion / Notificacion)
    database BD as Base de Datos Central
    participant Externos as Sistemas Externos (RedMine / GitLab / GLPI)
    participant Chat as Rocket Chat

    Trigger->>+Sync: sincronizar()
    Sync->>+BD: obtenerIntegracionesActivasYCredenciales()
    BD-->>-Sync: Lista de Integracion + Credencial (urlBase, token)

    loop Por cada integración activa
        Sync->>+Externos: GET datos vía API REST (urlBase, token)

        alt Respuesta exitosa (200 OK)
            Externos-->>-Sync: JSON (Peticiones, Incidentes, MRs, Pipelines)
            Sync->>+BD: guardarOActualizarItems() + estado="OK", ultimaSync
            BD-->>-Sync: Persistencia confirmada
        else Error de conexión o timeout
            Externos-->>Sync: Error HTTP / Timeout
            Sync->>BD: actualizarEstado(estado="ERROR") [mantiene datos previos]
            Sync->>+Chat: enviar() [Notificacion de fallo al Administrador]
            Chat-->>-Sync: Alerta enviada
        end
    end

    Sync-->>-Trigger: Proceso de sincronización finalizado
```

---

## UC7: Configurar integraciones y credenciales

```mermaid
sequenceDiagram
    autonumber
    actor Admin as Administrador
    participant UI as Frontend Web (UI)
    participant Sync as Servicio de Integraciones (Integracion / Credencial)
    participant Externos as Sistemas Externos (RedMine / GitLab / GLPI / Rocket Chat)
    database BD as Base de Datos Central

    rect rgba(2, 132, 199, 0.08)
        Note over Admin,UI: ref: UC1 - Iniciar sesión con cuenta de la cooperativa
    end

    Admin->>+UI: Ingresar herramienta, urlBase, tipoAuth y token
    UI->>+Sync: guardarConfiguracion(datosIntegracion, credencial)
    Sync->>+Externos: probarConexion(urlBase, token)

    alt Conexión y credenciales válidas
        Externos-->>-Sync: 200 OK (Conexión verificada)
        Sync->>Sync: encriptarToken(credencial)
        Sync->>+BD: guardarIntegracionYCredencial(activa=true)
        BD-->>-Sync: Registro guardado
        Sync-->>UI: Integración activada correctamente
        UI-->>Admin: Mostrar confirmación en pantalla
    else URL o token inválidos
        Externos-->>Sync: 401 Unauthorized / Error de red
        Sync-->>-UI: Error de validación de credenciales
        UI-->>-Admin: Mostrar alerta de error sin activar
    end
```

---

