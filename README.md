# Seguimiento departamento técnico · Nissei Asunción – Servicio Posventa

Plataforma con cuatro perfiles: Supervisor, Recepción, Técnico y Atención al cliente.
Publicada en https://asutecnicos.netlify.app · Base de datos en el proyecto de Supabase "Plataforma Tecnicos SPV ASU".

## Archivos

| Archivo | Para qué sirve |
|---|---|
| `index.html` | La aplicación completa: inicio de sesión y una vista por perfil. |
| `supabase/001_plataforma_tecnicos.sql` | Versión 1: tablas, permisos, acciones y tiempo real. |
| `supabase/002_mejoras.sql` | Versión 2: Atención al cliente, urgencias con motivo, No reparable / Presupuesto rechazado, confirmación SATI, notificaciones, alertas por demora, reingresos y KPIs. Solo agrega; no borra datos. |
| `supabase/003_productos_marcas_nuevas.sql` | Productos y marcas fuera de la lista: catálogo de marcas, aviso al supervisor y panel "Por verificar" en Resumen. Solo agrega; no borra datos. |
| `supabase/004_asignacion_automatica.sql` | Asignación automática al cargar una orden: entre los técnicos que reparan ese producto (KPIs → Tabla de valores), al que tenga menos órdenes en curso. Solo agrega; no borra datos. |
| `supabase/005_procesos_nuevos.sql` | Procesos fuera de la lista: al marcar Reparados, el técnico puede escribir el proceso que hizo. Se registra en la orden y el supervisor lo verifica en "Por verificar". Requiere el 003. Solo agrega; no borra datos. |
| `supabase/006_avisos_cierre.sql` | Aviso de cierre según el estado: Reparados, No reparable y Presup. rechazado llegan cada uno con su texto (y el motivo) a supervisores y Atención al cliente. Requiere el 003. |
| `supabase/007_presupuestos_y_regreso.sql` | Estado "Presupuestado" con falla y monto USD, confirmación del cliente por Atención al cliente, y regreso del producto a la oficina (Ubicado para envío → Recibido en oficina). Requiere 003, 005 y 006. Solo agrega; no borra datos. |
| `supabase/008_remisiones.sql` | Remisiones internas numeradas (Oficina → Taller y Taller → Oficina). La llegada al taller y "Listo para envío" pasan a Recepción y supervisor. Requiere el 007. Solo agrega; no borra datos. |
| `supabase/009_pantalla_tv.sql` | Perfil "Pantalla" (solo lectura) para el panel de TV. Solo agrega; no borra datos. |
| `netlify/functions/usuarios.mjs` | Crea el usuario del técnico al guardar su ficha y cambia contraseñas. |
| `netlify.toml` | Configuración del sitio. |

## Cómo publicar un cambio

1. Si hay un script SQL nuevo: **Supabase → SQL Editor → +** → pegarlo → **Run**.
2. **GitHub → repositorio plataforma-tecnicos → Add file → Upload files** → arrastrar los archivos que cambiaron → **Commit changes**.
3. Netlify publica solo en menos de un minuto. Recargar la página con **Ctrl + F5**.

Si el cambio incluye SQL, subí el `index.html` nuevo enseguida después de ejecutarlo: la página vieja no funciona con la base nueva.

## Usuarios

- **Técnicos:** desde la plataforma, con **Crear ficha técnico**.
- **Recepción, Supervisor y Atención al cliente:** crear la cuenta en **Authentication → Users → Add user** (Auto Confirm User) y ejecutar:
  ```sql
  insert into public.tec_usuarios (id, nombre, rol)
  select id, 'Nombre Apellido', 'recepcion' from auth.users where email = 'correo@nissei.com';
  ```
  Perfiles: `supervisor`, `recepcion`, `sac` (Atención al cliente), `pantalla` (panel de TV, solo lectura).
- **Dar de baja:** *Editar ficha → Acceso → Deshabilitado*. Se conserva su historial.

## Circuito de una orden

1. **Recepción o el supervisor cargan la orden**: ID SATI, marca, tipo de producto, técnico. Si es **urgente**, con motivo; si es **reingreso**, con el ID SATI de la orden original.
2. La orden queda en **Tránsito** en la pantalla del técnico (las urgentes parpadean) y le llega una notificación.
3. **Remisión Oficina → Taller:** Recepción o el supervisor tocan **Remisiones → Nueva remisión**, tildan las órdenes en tránsito (las del día arriba; también aparecen las de días anteriores sin remitir) y generan la remisión. Recibe un número (ej. N° 0012) y se imprime en ticket de 80 mm con espacio para firmas.
   En el taller, Recepción o el supervisor tocan **Recibir remisión**, tipean el número y tildan cada orden que llegó → pasa a **Asignado** y el técnico recibe el aviso. Si una orden llega sin remisión (por ejemplo, después de reasignarla), se confirma desde su detalle con **Confirmar que llegó al taller**.
4. El técnico (o el supervisor) la mueve de estado. En cada cambio tiene que confirmar **"Actualicé el caso en SATI"**.
   - **Pend. repuesto:** indica la pieza solicitada. Avisa a los supervisores.
   - **Reparados:** elige los procesos realizados (para KPIs). Si hizo uno que no está en la lista, lo escribe: se registra igual y el supervisor recibe un aviso para verificarlo. Avisa a supervisores y Atención al cliente.
   - **No reparable / Presupuesto rechazado:** indica el motivo. Cierran la orden y avisan a supervisores y Atención al cliente con el motivo.
   - **Presupuestado:** indica la falla y el monto en USD. Atención al cliente recibe un aviso emergente, habla con el cliente y registra la respuesta: si **acepta**, la orden vuelve a En revisión y el técnico recibe el aviso para continuar; si **rechaza**, pasa a Presup. rechazado con el motivo. Mientras espera, el técnico no puede moverla.
5. **Regreso a la oficina** (el taller está en otro edificio): al cerrar la orden, el técnico entrega el producto a la persona que gestiona el envío. Recepción o el supervisor la marcan **Listo para envío** (panel "Regreso a la oficina"), generan la **remisión Taller → Oficina** con esas órdenes y, en la oficina, tipean el número y tildan cada orden que llegó (**Recibido en oficina**). Si la orden se reabre, estos pasos se borran y sale de la remisión pendiente.
6. **Reasignar** la devuelve a Tránsito en la pantalla del nuevo técnico.

## Alertas y notificaciones

- La **campanita** muestra las notificaciones de cada usuario y, a los supervisores, las **órdenes demoradas**.
- **Avisos en el escritorio:** cada usuario los activa una vez desde la campanita (**Activar**). Mientras la plataforma esté abierta en alguna pestaña —aunque esté minimizada—, cada notificación aparece también como aviso de Windows; al hacer clic abre la orden. Se pueden apagar o probar desde la misma campanita.
- Los días de demora por estado se configuran en **Resumen → Órdenes demoradas → Configurar alertas**.

## Panel de TV (Taller SPV Barrail)

- En el televisor se inicia sesión con el usuario de perfil **Pantalla** y el panel aparece solo, a pantalla completa. Queda con la sesión iniciada.
- El supervisor también lo puede abrir con el botón **Panel TV** (o agregando `?tv` a la dirección).
- Muestra: órdenes abiertas, tránsito sin remitir, remisiones en camino, urgentes, demoradas, cerradas hoy, remisiones en tránsito en ambos sentidos, llegadas del día, carga de trabajo por técnico, órdenes por estado y las que más tiempo llevan esperando.
- Se actualiza solo en tiempo real. Una remisión con más de 12 horas en camino se marca en rojo.
- Para cerrar la sesión del panel: doble clic en el título "Taller SPV Barrail".

## Paneles del supervisor: Resumen de casos y Desempeño de técnicos

- Solo el supervisor. Se abren en una pestaña nueva con los botones **Resumen de casos** y **Desempeño** (o con `?panel=casos` / `?panel=desempeno` en la dirección). Mismo diseño y colores que el Panel TV.
- Miden las **órdenes cerradas en el mes**. Con **‹ ›** se cambia de mes, y con el botón de arriba se pasa de un panel al otro. El mes en curso se actualiza solo cada 5 minutos.
- **Resumen de casos:** demora promedio por estado (con los días de alerta), motivos de la demora, demora por técnico, en qué parte del proceso se demora (del traslado a la oficina hasta el regreso) y las órdenes más demoradas.
  - Los tiempos salen del historial de movimientos: cada cambio de estado cierra el tramo anterior.
  - Motivos: solo los que ya se registran. Pend. repuesto → la pieza solicitada (se agrupa sin importar mayúsculas). Presupuestado → si el cliente aceptó o rechazó. Tránsito → espera de la remisión y viaje. Asignado, En revisión y Control calidad no tienen motivo.
- **Desempeño de técnicos:** reparadas, procesos, días promedio por orden (de la recepción al cierre), % dentro del plazo y reingresos por técnico; reparadas por semana contra el promedio semanal del mes anterior; cómo se cerró cada orden; procesos más realizados. **No muestra montos.**
- "Dentro del plazo": la orden no pasó los días de alerta en ningún estado. El reingreso cuenta para quien reparó la orden original (igual que en KPIs).
- No necesitan scripts SQL nuevos.

## KPIs (supervisor)

- Pestaña **KPIs**: por técnico, órdenes reparadas, procesos realizados, no reparables, presupuestos rechazados, reingresos y % de reingreso, días promedio y valor en USD.
- **Reingresos y comisión:** el reingreso se atribuye al técnico de la orden original, en el período en que se cargó el reingreso, y le **descuenta** el valor de los procesos de esa orden original. Neto USD = valor de lo reparado − descuento por reingresos. Si la orden original no está en la plataforma, se cuenta como reingreso pero no se atribuye ni descuenta.
- **Tabla de valores**: por tipo de producto, sus procesos con el valor en USD. Cada trabajo guarda el valor vigente el día que se marcó como reparado; si en ese momento no había valor, se usa el de la tabla actual.

## Qué puede hacer cada perfil

| Acción | Supervisor | Recepción | Técnico | Atención al cliente |
|---|:-:|:-:|:-:|:-:|
| Cargar y reasignar órdenes | ✓ | ✓ | — | — |
| Órdenes asignadas (seguimiento) | — | ✓ | — | — |
| Consultar casos | ✓ | ✓ | Solo los suyos | ✓ (solo lectura) |
| Mover estados | ✓ | — | Solo sus órdenes | — |
| Notas | ✓ | ✓ | Solo sus órdenes | — |
| Responder presupuestos (cliente aceptó / rechazó) | ✓ | — | — | ✓ |
| Generar, imprimir y recibir remisiones | ✓ | ✓ | — | — |
| Confirmar llegada al taller | ✓ | ✓ | — | — |
| Listo para envío / Recibido en oficina | ✓ | ✓ | — | — |
| Corregir datos | ✓ | ✓ | — | — |
| Fichas de técnicos, alertas, KPIs, valores, Excel, eliminar | ✓ | — | — | — |

Los permisos se aplican en la base de datos, no solo en la pantalla.

## Excel para Power BI

**Exportar a Excel** (supervisor) descarga todo el historial en seis hojas:

- **Ordenes**: una fila por orden: fechas de carga, recepción y cierre, urgencia, reingreso, pieza solicitada, motivo de cierre.
- **Movimientos**: cada alta, recepción, cambio de estado (con confirmación SATI), reasignación, nota o corrección.
- **Procesos**: cada proceso realizado, con técnico, producto y valor en Gs. y USD.
- **Valores**: la tabla de valores vigente.
- **Usuarios**: todos los perfiles.
- **Remisiones**: una fila por orden remitida, con sentido, fecha de la remisión, llegada y horas en camino.

Relacioná las hojas por `Orden ID` y `Técnico ID` / `Usuario ID`.
