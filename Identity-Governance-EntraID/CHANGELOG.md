# Changelog

Todos los cambios notables de esta plantilla se documentan en este archivo.
Formato basado en [Keep a Changelog](https://keepachangelog.com/es-ES/1.0.0/).

## [1.2.0] - 2026-10-04

### Fixed
- **"Vía PIM" contaba de menos**: solo incluía elegibilidades sin fecha de fin; las elegibilidades con expiración quedaban fuera y la tarjeta no coincidía con la dona "Método de asignación". Ahora cuenta todas las asignaciones PIM.
- **"Permanentes" incluía activaciones PIM temporales**: `roleAssignments` devuelve también los roles activados temporalmente. Las asignaciones activas se leen ahora de `roleAssignmentScheduleInstances` (permanente / temporal / activada) y las activaciones no se duplican con su elegibilidad. Si el endpoint no está permitido, se usa `roleAssignments` como respaldo.
- **Datos incompletos en tenants grandes**: las consultas a Microsoft Graph solo leían la primera página de resultados. Nueva función común `GraphPaginas` que sigue `@odata.nextLink` en todas las consultas (compatible con el refresco programado del servicio) y reintenta ante respuestas 429/503 respetando `Retry-After`. Usuarios, service principals y grupos se piden en páginas de 999.

### Changed
- **Nivel de riesgo por rol**: antes solo 3 roles se clasificaban por nombre (el resto quedaba como "Bajo"). Ahora: Crítico = roles Tier 0 identificados por ID de plantilla; Alto = roles con `isPrivileged` de Microsoft Graph; Medio = otros roles de administración; Bajo = resto.
- **Score de Riesgo 0–100**: reemplaza la suma absoluta (que crecía con el tamaño del tenant) por un puntaje comparable: 60% asignaciones privilegiadas permanentes de usuarios/grupos + 40% exceso de Global Admins. Niveles: Crítico ≥ 60, Alto ≥ 40, Medio ≥ 20.
- **Service principals**: excluidos de "% Cobertura PIM" y del Score (no pueden ser elegibles en PIM), con indicadores propios ("Apps con rol", "Apps privileg.") y la etiqueta **Permanente (app)** en las tablas.
- **Rediseño del reporte**: las dos páginas casi idénticas se unifican en **Asignaciones privilegiadas** (encabezado con estado, tarjetas con íconos, gráficos y tabla alineados) y la segunda pasa a ser **Detalle de identidad** (página de drill-through con botón de regreso).
- **Tema propio** (`EntraIDSeguridad.json`): paleta semántica, fondo de página y estilo de tablas, gráficos y segmentadores para que los visuales nuevos hereden el diseño.
- `Asignaciones` y `Elegibilidad` pasan a ser consultas intermedias (no se cargan al modelo); se desactiva *Fecha/hora automática* y se eliminan las tablas de fecha automáticas.
- Límite de Global Admins centralizado en la medida `Limite Global Admins` (por defecto 4).

### Added
- Columna **n.º de asignaciones** en las tablas de detalle (filas idénticas se agrupan; el conteo permite cuadrar con las tarjetas).
- Columnas de fecha legibles (`Inicio`, `Fin`) para elegibilidades.

## [1.1.1] - 2026-08-19

### Fixed
- **Error de duplicados en `Elegibilidad`** (`Column 'ID_Entidad' ... contains a duplicate value ...`): causado por una relación de modelo auto-detectada por Power BI entre `Reporte_Final.ID_Entidad` y `Elegibilidad.ID_Entidad`, que forzaba unicidad en una tabla donde es normal tener múltiples filas por persona (una por cada rol elegible). Se eliminó la relación redundante — `Reporte_Final` ya obtiene esos datos aplanados vía Power Query.

## [1.1.0] - 2026-08-12

### Fixed
- **Error de refresco en `Identidades`**: `Column 'UPN_o_Correo' in Table 'Identidades' contains a duplicate value 'All Company'...`.
  Causa raíz: el `displayName` de un grupo de Entra ID no es único, y la consulta usaba ese nombre como sustituto de correo para los grupos, generando valores duplicados en una columna usada como lado "uno" de una relación.
  Solución: los grupos ahora usan su campo `mail` real cuando existe; si no, un identificador único construido a partir de su `Id` de objeto.
- **Error en cascada en `Reporte_Final`** (`OLE DB or ODBC error: Exception from HRESULT: 0x80040E4E`): era un efecto secundario del error anterior — al fallar `Identidades`, se cancelaba el resto de la transacción de refresco. Se resuelve al corregir la causa raíz.

### Added
- **Manejo de errores HTTP** en las consultas `Asignaciones`, `Roles`, `_Config` e `Identidades`: ahora usan `ManualStatusHandling` para capturar respuestas 400/401/403/404/429/500/503 de Microsoft Graph y devolver un resultado controlado (tabla vacía con el esquema correcto) en vez de romper todo el refresco. Antes solo `Elegibilidad` tenía este manejo.
- **Resiliencia por tipo de objeto en `Identidades`**: las llamadas a `/users`, `/servicePrincipals` y `/groups` ahora se evalúan de forma independiente — si un tipo de objeto falla (p. ej. por falta de permiso `Group.Read.All`), los demás igual se cargan en vez de fallar la tabla completa.
- **Columna `Estado_Conexion` en `_Config`**: indica "OK" o el motivo del error HTTP del último refresco, útil como indicador visual en el reporte.

## [1.0.0] - 2026-04-XX

### Added
- Versión inicial de la plantilla de reporte Power BI para gobernanza de identidades en Entra ID (roles, asignaciones, elegibilidad PIM, identidades).
