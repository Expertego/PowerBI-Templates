# Changelog

Todos los cambios notables de esta plantilla se documentan en este archivo.
Formato basado en [Keep a Changelog](https://keepachangelog.com/es-ES/1.0.0/).

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
