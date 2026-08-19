# Solución de problemas — EntraIDRoles.pbit

Esta guía cubre los problemas más comunes que un usuario nuevo puede enfrentar al abrir y
configurar esta plantilla por primera vez, y cómo resolverlos.

---

## 1. Error: "Column 'UPN_o_Correo' ... contains a duplicate value"

**Mensaje completo (puede variar el valor duplicado):**
```
Column 'UPN_o_Correo' in Table 'Identidades' contains a duplicate value '...' and this is
not allowed for columns on the one side of a many-to-one relationship or for columns that
are used as the primary key of a table.
```

**Causa:** el `displayName` de un grupo de Microsoft Entra ID **no es único** — pueden existir
dos grupos distintos con el mismo nombre. Esto ya está corregido desde la versión 1.1.0 de la
plantilla (los grupos ahora usan su campo `mail` real, o un identificador único basado en su
`Id` si no tienen correo).

**Solución:** asegúrate de estar usando la **última versión** de `EntraIDRoles.pbit` publicada
en este repositorio (ver [Releases](../../releases)). Si el error persiste en la última
versión, es indicio de un caso de datos no contemplado — abre un
[Issue](../../issues) con el mensaje exacto.

---

## 2. Error en cascada: "OLE DB or ODBC error: Exception from HRESULT: 0x80040E4E"

Este mensaje genérico casi nunca es la causa raíz real — suele ser un **efecto secundario**
de que otra tabla (típicamente `Identidades`) falló antes en la misma transacción de refresco,
lo que cancela el resto del proceso. Revisa primero si hay otro error más específico reportado
en alguna tabla (clic en "Mostrar detalles del error" o revisa cada consulta individualmente
en el editor de Power Query) antes de asumir que es un problema de conexión.

---

## 3. Error: "Column 'ID_Entidad' in Table 'Elegibilidad' contains a duplicate value"

**Mensaje completo (el valor duplicado varía):**
```
Column 'ID_Entidad' in Table 'Elegibilidad' contains a duplicate value '...' and this is
not allowed for columns on the one side of a many-to-one relationship or for columns that
are used as the primary key of a table.
```

**Causa:** Power BI Desktop puede crear automáticamente una relación de modelo entre
`Reporte_Final.ID_Entidad` y `Elegibilidad.ID_Entidad` (función "Detectar nuevas relaciones
después de cargar los datos"), tratando a `Elegibilidad.ID_Entidad` como si debiera ser único.
Esto es incorrecto: `Elegibilidad` es una tabla donde es normal y esperado que la misma
persona aparezca en varias filas (una por cada rol para el que es elegible vía PIM). En
tenants con muchas asignaciones de roles múltiples por persona, esta relación espuria
eventualmente choca con datos reales y rompe el refresco completo.

Esto ya está corregido desde la versión 1.1.1 de la plantilla: se eliminó esa relación
redundante (`Reporte_Final` ya trae los datos de `Elegibilidad` aplanados vía Power Query, así
que no se pierde nada al quitarla del modelo de relaciones).

**Solución si aparece en una versión anterior o reaparece:**
1. Modelado → vista de Relaciones (Model view).
2. Busca la línea que conecta `Reporte_Final` con `Elegibilidad` por la columna `ID_Entidad`.
3. Clic derecho sobre la relación → Eliminar.
4. Ver la sección siguiente para evitar que Power BI la vuelva a crear sola.

---

## 4. Error de "Formula Firewall" al refrescar

**Mensaje típico:**
```
Consulta 'Elegibilidad' (etapa 'RawEligibility') hace referencia a otras consultas o pasos,
por lo tanto puede no tener acceso directo a un origen de datos. Recompile esta combinación
de datos.
```

**Causa:** Power Query bloquea por defecto la combinación de datos entre distintos orígenes o
consultas dependientes (los **Niveles de privacidad**), como medida de seguridad ante posible
fuga de datos entre fuentes. Esta plantilla combina varias llamadas a Microsoft Graph y
parámetros entre sí, lo cual dispara esta protección en la configuración por defecto de un
Power BI Desktop recién instalado.

**Esto no es un problema del archivo `.pbit`** — es una configuración local de cada instalación
de Power BI Desktop que no viaja con el archivo. Cada usuario nuevo debe configurarla una vez.

**Solución:**
1. Archivo → Opciones y configuración → Opciones.
2. En la sección **"ARCHIVO ACTUAL"** (no "Global") → **Privacidad**.
3. Selecciona **"Omitir siempre los niveles de privacidad (puede mejorar el rendimiento)"**.
4. Aceptar → Actualizar.

> ⚠️ Configurar el nivel de privacidad en la sección "Global" no es suficiente — debe hacerse
> específicamente en "Archivo actual" para este `.pbix`/`.pbit`.

---

## 5. Error de autenticación al refrescar: "Could not authenticate with the credentials provided"

**Contexto:** suele aparecer justo después de cambiar la configuración de privacidad del punto
3, cuando Power BI vuelve a evaluar los orígenes de datos y pide credenciales para
`login.microsoftonline.com` y/o `graph.microsoft.com`.

**Causa:** una credencial anterior quedó guardada de forma incompleta o incorrecta para ese
origen (por ejemplo, un intento fallido con otro método de autenticación), y Power BI la sigue
arrastrando en vez de pedir una nueva.

**Solución — limpiar credenciales desde cero:**
1. Cancela el diálogo de credenciales y el refresco en curso.
2. Archivo → Opciones y configuración → **Configuración de origen de datos**.
3. Cambia a **"Fuentes de datos en el archivo actual"**.
4. Selecciona cada entrada de `login.microsoftonline.com` y `graph.microsoft.com` y pulsa
   **"Quitar permisos"** (no solo editar) para eliminarlas por completo.
5. Cierra el diálogo y vuelve a pulsar **Actualizar**.
6. Cuando aparezca el diálogo **"Acceder a contenido web"**: selecciona la pestaña
   **"Anónimo"**. En el desplegable de nivel, elige la **raíz del dominio**
   (ej. `https://login.microsoftonline.com/`, no una subruta con el ID del tenant) y pulsa
   **Conectar**. Repite lo mismo si aparece para `graph.microsoft.com`.

> La autenticación real contra Microsoft Graph la maneja el propio código Power Query
> (`client_credentials` con el `ClientID`/`ClientSecret` del App Registration), no el
> mecanismo de credenciales nativo de Power BI — por eso el nivel correcto aquí siempre es
> **Anónimo**, nunca "Cuenta organizacional" ni "Básico".

---

## 6. Checklist de requisitos previos (recomendado revisar antes de abrir la plantilla)

- [ ] Tienes un **App Registration** en Microsoft Entra ID con permisos de **aplicación**
      (no delegados) y **consentimiento de administrador otorgado** para, como mínimo:
      `User.Read.All`, `Group.Read.All`, `Application.Read.All`, `Organization.Read.All`,
      `RoleManagement.Read.Directory`, `RoleEligibilitySchedule.Read.Directory`.
- [ ] Tienes a mano `Tenant ID`, `Client ID` y un `Client Secret` vigente (no vencido) de ese
      App Registration.
- [ ] En Power BI Desktop, configuraste **Archivo actual → Privacidad → Omitir siempre los
      niveles de privacidad** (ver punto 3 arriba) **antes** del primer refresco.
- [ ] Las credenciales de `login.microsoftonline.com` y `graph.microsoft.com` están en
      **Anónimo** en Configuración de origen de datos.
- [ ] Desactivaste "Detectar nuevas relaciones después de cargar los datos" en Archivo →
      Opciones y configuración → Opciones → Archivo actual → Carga de datos. Esto evita que
      Power BI cree automáticamente relaciones incorrectas entre tablas que comparten nombres
      de columna (como `ID_Entidad` o `ID_Rol`), que es la causa raíz de varios de los errores
      de "valor duplicado" descritos en este documento.

Seguir este checklist antes del primer refresco evita la mayoría de los problemas descritos
en este documento.
