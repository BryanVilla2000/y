# Diagramas de Secuencia

## CU-01: Iniciar sesión con SSO

```mermaid
sequenceDiagram
    autonumber
    actor User as Usuario PMO / Técnico / Admin TI
    participant UI as Componente React
    participant Api as api.js
    participant Nginx as nginx
    participant Express as Express rutas/auth.js
    participant Negocio as ServicioSesiones / Auth
    participant DB as PostgreSQL 17
    participant SSO as SSO Cooperativa mock / pendiente

    User->>+UI: Ingresar en vista de acceso sin guardar contraseña
    UI->>+Api: postAuthLogin(datosAcceso)
    Api->>+Nginx: POST /api/auth/login JSON
    Nginx->>+Express: Proxy /api/auth/login
    Express->>+Negocio: autenticarYCrearSesion(datosAcceso)
    Negocio->>+SSO: validarCredencialesSSO(mock / pendiente)

    alt Autenticación SSO válida 200 OK
        SSO-->>-Negocio: Identidad institucional y rol
        Negocio->>+DB: registrarOActualizarSesion(expiracion=480min)
        DB-->>-Negocio: Sesión persistida y permisos RBAC
        Negocio-->>-Express: Bearer token 480 min y perfil
        Express-->>-Nginx: 200 OK JSON y Bearer token
        Nginx-->>-Api: 200 OK
        Api-->>-UI: Estado autenticado y rol con soloPropios si aplica
        UI-->>-User: Mostrar dashboard según rol
    else Credenciales inválidas o fallo SSO
        SSO-->>Negocio: Error de autenticación
        Negocio-->>Express: Rechazo de acceso
        Express-->>Nginx: 401 Unauthorized
        Nginx-->>Api: 401 Unauthorized
        Api-->>UI: Error de autenticación
        UI-->>User: Mostrar mensaje de error 401
    end
```

---

## CU-02: Filtrar información

```mermaid
sequenceDiagram
    autonumber
    actor User as Usuario PMO / Técnico / Admin TI
    participant UI as Componente React
    participant Api as api.js
    participant Nginx as nginx
    participant Express as Express rutas/carga.js
    participant Negocio as Filtro / MotorCarga
    participant Cache as AlmacenMemoria

    Note over User,Express: Requiere CU-01: Bearer token válido, si es Técnico aplica soloPropios

    User->>+UI: Seleccionar filtros: desarrollador, tipo, etapa, proyecto, fuente, fechas
    UI->>+Api: getCargaFiltrada(parametrosFiltro, token)
    Api->>+Nginx: GET /api/carga con Bearer token
    Nginx->>+Express: Proxy /api/carga y validar token/rol
    Express->>+Negocio: aplicarFiltros(criterios, soloPropios)
    Negocio->>+Cache: leerSnapshotEnMemoria()
    Cache-->>-Negocio: Ítems e indicadores en memoria
    Negocio-->>-Express: Colección filtrada y métricas recalculadas
    Express-->>-Nginx: 200 OK JSON filtrado
    Nginx-->>-Api: 200 OK
    Api-->>-UI: Datos filtrados para gráficos Chart.js y tablas
    UI-->>-User: Actualizar vista filtrada
```

---

## CU-03: Consultar dashboard de carga y avance

```mermaid
sequenceDiagram
    autonumber
    actor User as Usuario PMO / Técnico / Admin TI
    participant UI as Componente React
    participant Api as api.js
    participant Nginx as nginx
    participant Express as Express rutas/dashboard.js
    participant Negocio as MotorCarga / IndicadorCarga
    participant Cache as AlmacenMemoria

    Note over User,UI: Incluye CU-01 Iniciar sesión con SSO y permite filtrar vía CU-02

    User->>+UI: Abrir Dashboard de carga y avance
    UI->>+Api: getDashboardCarga(filtros, token)
    Api->>+Nginx: GET /api/dashboard con Bearer token
    Nginx->>+Express: Proxy /api/dashboard valida RBAC y soloPropios
    Express->>+Negocio: consultarCargaYAvance(filtros, rolUsuario)
    Negocio->>+Cache: obtenerIndicadoresYAvance()
    Cache-->>-Negocio: Peticiones, avance por etapa y alertas activas
    Negocio-->>-Express: Resumen de carga y avance 10, 40, 70, 90 y 100 por ciento
    Express-->>-Nginx: 200 OK JSON
    Nginx-->>-Api: 200 OK
    Api-->>-UI: Datos de carga, avance y alertas
    UI-->>-User: Renderizar dashboard Chart.js y lista de peticiones

    opt Exportar reporte CU-04 pendiente
        Note over User,UI: ext: CU-04 Exportar reporte pendiente
    end
```

---

## CU-04: Exportar reporte pendiente

```mermaid
sequenceDiagram
    autonumber
    actor PMO as PMO
    participant UI as Componente React
    participant Api as api.js
    participant Nginx as nginx
    participant Express as Express rutas/reportes.js
    participant Negocio as Reporte pendiente

    Note over PMO,Express: Extiende CU-03 y requiere CU-01: exportación Excel/PDF pendiente

    PMO->>+UI: Solicitar exportar reporte Excel o PDF
    UI->>+Api: postExportarReporte(filtrosActuales, formato, token)
    Api->>+Nginx: POST /api/reportes/exportar con Bearer token
    Nginx->>+Express: Proxy /api/reportes/exportar
    Express->>+Negocio: generarReporte()
    Negocio-->>-Express: Funcionalidad marcada como pendiente
    Express-->>-Nginx: 501 o 400 estado pendiente
    Nginx-->>-Api: Respuesta de estado pendiente
    Api-->>-UI: Notificar estado pendiente
    UI-->>-PMO: Mostrar aviso de exportación pendiente
```

---

## CU-05: Sincronizar fuentes cron 15 min

```mermaid
sequenceDiagram
    autonumber
    actor Cron as Programador de tareas cron cada 15 min
    participant Express as Express rutas/sincronizacion.js
    participant Sync as Sincronizacion / Conector
    participant DB as PostgreSQL 17 AlmacenPostgres / RepoSync
    participant Cache as AlmacenMemoria
    participant Externos as RedMine / GitLab / GLPI / Rocket Chat

    Cron->>+Express: Ejecutar tarea cada 15 min
    Express->>+Sync: ejecutarCicloSincronizacion(cursorConSolape5Min)
    Sync->>+DB: leerCursorYCredencialesFuentes()
    DB-->>-Sync: Cursores con solape de 5 min y configuración de conectores

    loop Por cada conector
        Sync->>+Externos: leerDatos(desde) hasta 3 reintentos

        alt Lectura exitosa 200 OK
            Externos-->>-Sync: Registros desde cursor
            Sync->>Sync: cruzarYNormalizarPorCodigo() CU-07
            Sync->>Sync: calcularCargaYAvance() CU-08
            Sync->>+DB: persistirTransaccion(items, cursor, estado=OK)
            DB-->>-Sync: Commit transaccional en PostgreSQL 17
            Sync->>+Cache: actualizarSnapshotEnMemoria()
            Cache-->>-Sync: Caché actualizada
        else Falla tras 3 reintentos CU-06
            Externos-->>Sync: Error de conexión o timeout tras 3 intentos
            Sync->>DB: registrarFalloYConservarSnapshotAnterior() CU-06
            DB-->>Sync: RegistroFallo guardado y último snapshot válido
            Sync->>Cache: mantenerSnapshotAnterior()
        end
    end

    Sync-->>-Express: Resumen de sincronización
    Express-->>-Cron: Ciclo completado
```

---

## CU-06: Registrar fallo de fuente

```mermaid
sequenceDiagram
    autonumber
    participant Sync as Sincronizacion / Conector
    participant DB as PostgreSQL 17 RepositorioSincronizacionPostgres
    participant Cache as AlmacenMemoria

    Note over Sync,Cache: Extiende CU-05 cuando una fuente falla tras 3 reintentos

    Sync->>Sync: Agotar 3 reintentos en leerDatos(desde) para fuente
    Sync->>+DB: insertarRegistroFallo(fuente, error, timestamp)
    DB-->>-Sync: RegistroFallo persistido
    Sync->>+DB: conservarUltimoSnapshotFuenteValido(fuente)
    DB-->>-Sync: Snapshot anterior intacto
    Sync->>+Cache: mantenerDatosPreviosEnMemoria(fuente)
    Cache-->>-Sync: Dashboard sigue mostrando último dato válido
    Sync-->>Cache: Marcar estado de sincronización con fallo registrado
```

---

## CU-07: Cruzar datos por código

```mermaid
sequenceDiagram
    autonumber
    participant Sync as Sincronizacion / Normalizador
    participant DB as PostgreSQL 17 AlmacenPostgres

    Note over Sync,DB: Incluido en CU-05 durante la sincronización cada 15 min

    Sync->>Sync: Recibir ítems de RedMine, GitLab, GLPI y Rocket Chat
    Sync->>Sync: Extraer y normalizar código único POA-AAAA-NNN, REC, iniciativa.ti.001, INCIDENTE
    Sync->>+DB: buscarCoincidenciaPorCodigoYDeduplicar(codigo)
    DB-->>-Sync: Resultado de cruce: coincidencia o sin cruce

    alt Código reconocido y cruzado sin duplicar
        Sync->>Sync: Consolidar atributos en Peticion y ActividadCodigo
        Sync->>+DB: guardarItemConsolidado()
        DB-->>-Sync: Registro actualizado sin duplicados
    else Código no reconocido o sin cruce válido
        Sync->>+DB: guardarRegistroNoReconocido(codigo, payloadOrigen)
        DB-->>-Sync: Guardado en RegistroNoReconocido para revisión CU-11
    end
```

---

## CU-08: Calcular carga y avance

```mermaid
sequenceDiagram
    autonumber
    participant Sync as Sincronizacion / IndicadorCarga
    participant DB as PostgreSQL 17 AlmacenPostgres

    Note over Sync,DB: Incluido en CU-05, sin priorización algorítmica

    Sync->>+DB: consultarPeticionesAbiertasYEtapas(desarrollador, umbral)
    DB-->>-Sync: Peticiones abiertas, etapa actual y Umbral.maxPeticiones default 6
    Sync->>Sync: Estimar avance por etapa: 10, 40, 70, 90 y 100 por ciento
    Sync->>Sync: Calcular carga por peticiones abiertas sin algoritmo de priorización
    Sync->>+DB: guardarIndicadorCarga(carga, avanceEstimado)
    DB-->>-Sync: IndicadorCarga actualizado

    alt peticionesAbiertas mayor que Umbral.maxPeticiones default 6
        Sync->>Sync: Disparar condición de alerta CU-09
        Note over Sync,DB: ext: CU-09 Enviar alerta
    end
```

---

## CU-09: Enviar alerta

```mermaid
sequenceDiagram
    autonumber
    participant Sync as Sincronizacion / Alerta
    participant DB as PostgreSQL 17 AlmacenPostgres
    participant Chat as ConectorRocketChat / Rocket Chat

    Note over Sync,Chat: Extiende CU-08 cuando peticionesAbiertas supera Umbral.maxPeticiones default 6

    Sync->>+DB: verificarAlertaActivaSinDuplicar(desarrollador, umbral)
    DB-->>-Sync: Estado de alerta previa

    alt Alerta aún no enviada para este estado o umbral
        Sync->>+DB: crearAlerta(desarrollador, peticionesAbiertas, umbral)
        DB-->>-Sync: Alerta persistida sin duplicados
        Sync->>+Chat: enviarAlertaCanal(canal="#35;ti-alertas", mensaje)
        Chat-->>-Sync: 200 OK Mensaje publicado en #35;ti-alertas
        Sync-->>DB: Marcar alerta como notificada
    else Alerta ya notificada para evitar duplicados
        Sync-->>DB: Omitir envío duplicado
    end
```

---

## CU-10: Configurar umbral

```mermaid
sequenceDiagram
    autonumber
    actor Admin as Administrador TI
    participant UI as Componente React
    participant Api as api.js
    participant Nginx as nginx
    participant Express as Express rutas/umbral.js
    participant Negocio as Umbral / Config
    participant DB as PostgreSQL 17
    participant Cache as AlmacenMemoria

    Note over Admin,Express: Requiere CU-01 y rol Administrador TI

    Admin->>+UI: Ingresar nuevo valor de umbral maxPeticiones
    UI->>+Api: putUmbralConfig(nuevoUmbral, token)
    Api->>+Nginx: PUT /api/config/umbral JSON y Bearer token
    Nginx->>+Express: Proxy /api/config/umbral valida rol Admin TI
    Express->>+Negocio: actualizarUmbral(nuevoUmbral)

    alt Valor válido entero mayor o igual a 1
        Negocio->>+DB: guardarUmbral(maxPeticiones)
        DB-->>-Negocio: Umbral persistido
        Negocio->>+Cache: actualizarUmbralEnMemoria(maxPeticiones)
        Cache-->>-Negocio: Umbral en memoria actualizado
        Negocio-->>-Express: Configuración actualizada
        Express-->>-Nginx: 200 OK JSON
        Nginx-->>-Api: 200 OK
        Api-->>-UI: Confirmación de umbral guardado
        UI-->>-Admin: Mostrar confirmación y recalcular alertas en panel
    else Valor inválido 400 Bad Request
        Negocio-->>Express: Error de validación de umbral
        Express-->>Nginx: 400 Bad Request
        Nginx-->>Api: 400 Bad Request
        Api-->>UI: Mensaje de validación inválido
        UI-->>Admin: Mostrar error de validación 400
    end
```

---

## CU-11: Revisar registros no reconocidos

```mermaid
sequenceDiagram
    autonumber
    actor Admin as Administrador TI
    participant UI as Componente React
    participant Api as api.js
    participant Nginx as nginx
    participant Express as Express rutas/no-reconocidos.js
    participant Negocio as RegistroNoReconocido / Sincronizacion
    participant DB as PostgreSQL 17
    participant Cache as AlmacenMemoria

    Note over Admin,Express: Requiere CU-01 y rol Administrador TI

    Admin->>+UI: Abrir bandeja de registros no reconocidos
    UI->>+Api: getRegistrosNoReconocidos(token)
    Api->>+Nginx: GET /api/registros-no-reconocidos y Bearer token
    Nginx->>+Express: Proxy /api/registros-no-reconocidos
    Express->>+Negocio: listarRegistrosNoReconocidos()
    Negocio->>+DB: consultarRegistroNoReconocido()
    DB-->>-Negocio: Registros pendientes de vinculación
    Negocio-->>-Express: Lista de RegistroNoReconocido
    Express-->>-Nginx: 200 OK JSON
    Nginx-->>-Api: 200 OK
    Api-->>-UI: Mostrar registros no reconocidos
    UI-->>-Admin: Listar ítems sin cruce por código

    Admin->>+UI: Vincular código válido o descartar registro
    UI->>+Api: postVincularRegistroNoReconocido(idRegistro, codigo, token)
    Api->>+Nginx: POST /api/registros-no-reconocidos/:id/vincular y Bearer token
    Nginx->>+Express: Proxy de vinculación
    Express->>+Negocio: vincularYConsolidar(idRegistro, codigo)
    Negocio->>+DB: actualizarItemYMarcarResuelto(idRegistro, codigo)
    DB-->>-Negocio: Persistencia confirmada en PostgreSQL 17
    Negocio->>+Cache: refrescarSnapshotEnMemoria()
    Cache-->>-Negocio: Caché actualizada
    Negocio-->>-Express: Vinculación completada
    Express-->>-Nginx: 200 OK o 201 Created
    Nginx-->>-Api: 200 OK
    Api-->>-UI: Confirmar vinculación
    UI-->>-Admin: Actualizar lista y métricas del dashboard
```