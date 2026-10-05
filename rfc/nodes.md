# RFC: Nodos distribuidos con datos centralizados

Estado: borrador. Rama: `claude/fervent-cannon-bc7uz7`.

Este documento define el diseño, los requisitos de seguridad y los criterios de aceptación para unir varios servidores a una instancia de Appwrite. Los datos siguen centralizados, la carga se reparte y no hace falta Kubernetes.

## 1. Objetivos

1. **Un solo comando** para unir un servidor nuevo.
2. **Un solo endpoint y una sola API key** para la app cliente, sin importar cuántos nodos haya.
3. **Datos centralizados**: una única fuente de verdad (BD, Redis, archivos). Ningún nodo guarda estado propio.
4. **Principal ligero**: el nodo de control solo gestiona y enruta. Ninguna función de Appwrite se pierde.
5. **Sin puertos abiertos** en el nodo de datos (conexión solo saliente por túnel privado).
6. **Seguro por defecto**: un nodo comprometido no debe dar acceso a todo.
7. **Compatible hacia atrás**: sin nodos, Appwrite se comporta exactamente como hoy.

### No objetivos

- Replicar la BD entre regiones ni multi-escritor (fase posterior).
- Sustituir a un orquestador general. Solo se gestionan nodos de Appwrite.
- Autoescalado automático de servidores (el alta es manual con el comando).

## 2. Arquitectura

```
                 ┌──────────── Nodo de control (ligero) ───────────┐
Internet ─► LB ─►│ Traefik, consola, API de control, scheduler,     │
                 │ mantenimiento, registro de nodos                 │
                 └───────┬──────────────────────────────────────────┘
                         │ overlay WireGuard
        ┌────────────────┼──────────────────┐
   Nodo api/worker   Nodo api/worker     Nodo data
   API, realtime,    workers, executor   Postgres, Redis,
   workers, executor                     S3/MinIO, ClickHouse
```

Roles, controlados por `_APP_NODE_ROLE` (por defecto `all`, que es el comportamiento actual):

| Rol | Ejecuta |
|---|---|
| `control` | Traefik, consola, API de control, scheduler, mantenimiento, registro de nodos |
| `api` | API pública, realtime, mqtt |
| `worker` | Workers de cola y executor |
| `data` | Postgres, Redis, storage S3-compatible, ClickHouse |
| `all` | Todo (instalación actual, sin cambios) |

Todos los nodos usan **la misma imagen** de Appwrite. Solo cambia el perfil de compose.

## 3. Modelo de amenazas

| Amenaza | Actor | Impacto si no se mitiga |
|---|---|---|
| Nodo worker/api comprometido | Atacante con acceso al servidor | Lectura de todos los proyectos, secretos compartidos |
| Token de unión robado | Intercepción o fuga | Un servidor ajeno entra al overlay |
| Script de instalación suplantado | Man-in-the-middle, DNS | Ejecución de código como root en el servidor del usuario |
| Nodo falso o replay de heartbeat | Red | Nodo fantasma recibe tráfico |
| Pérdida del nodo `data` | Fallo de hardware | Pérdida total de datos |
| Deriva de reloj | Operativo | Fallos intermitentes de JWT y keys |

## 4. Requisitos de seguridad (obligatorios para lanzar)

Cada requisito tiene un identificador y una prueba que lo verifica.

### 4.1 Unión y confianza

- **S1.** El token de unión es de **un solo uso**, aleatorio (mínimo 32 bytes de CSPRNG), se guarda **hasheado** y caduca (por defecto 15 minutos).
- **S2.** El token queda ligado al **rol** pedido al crearlo. No se puede usarlo para obtener otro rol.
- **S3.** El endpoint de unión tiene **límite de velocidad** por IP y por token, y cada alta, rechazo y revocación se registra en **auditoría**.
- **S4.** El instalador solo se sirve por **HTTPS**. Se publica su hash SHA-256 y se documenta el modo manual (descargar, verificar y ejecutar), además del modo de un comando.
- **S5.** Cada nodo tiene su **propio par de claves WireGuard**, generado en el nodo. La clave privada nunca sale del nodo ni se almacena en el control.
- **S6.** El control puede **revocar** un nodo: elimina su peer del overlay, invalida su token y sus credenciales de BD, y lo marca `offline` de forma inmediata.

### 4.2 Aislamiento y mínimo privilegio

- **S7.** Cada nodo usa **credenciales de BD propias** con los permisos mínimos para su rol. Un nodo `worker` no necesita las mismas que el `control`.
- **S8.** Las credenciales de BD de un nodo son **revocables sin afectar a los demás**.
- **S9.** Postgres, Redis y MinIO escuchan **solo en la interfaz del overlay** (`wg0`). Ningún puerto de datos se publica a Internet.
- **S10.** El nodo `data` establece el túnel **saliente**. No acepta conexiones entrantes desde Internet.
- **S11.** El firewall del nodo solo permite el tráfico del overlay necesario para su rol (lista cerrada).

### 4.3 Secretos

- **S12.** Los secretos compartidos (`_APP_OPENSSL_KEY_V1` y similares) se entregan **cifrados con la clave pública del nodo**, nunca en claro por el endpoint ni en logs.
- **S13.** Los secretos no aparecen en `docker inspect` ni en variables visibles en logs. Se montan como ficheros con permisos `0400` cuando el servicio lo permite.
- **S14.** La **rotación** de claves distribuye la nueva a todos los nodos antes de activarla, y no retira la anterior hasta que todos confirmen tenerla.
- **S15.** Los secretos de nodo no se escriben en la BD en claro. Los tokens y claves se guardan hasheados o cifrados.

### 4.4 Integridad de la comunicación

- **S16.** Los heartbeats van **firmados** por el nodo y llevan marca de tiempo y nonce, con ventana corta. Se rechazan los repetidos (anti-replay).
- **S17.** Toda comunicación nodo a control va por el overlay y con TLS o autenticación mutua cuando atraviesa una red no confiable.
- **S18.** El control rechaza nodos con **versión distinta** de Appwrite o con **reloj desviado** más allá del umbral.

### 4.5 Enrutado

- **S19.** Solo nodos `online`, con heartbeat reciente y `/v1/health` correcto, entran al balanceador. Los `draining` y `offline` salen de inmediato.
- **S20.** Las cabeceras internas de enrutado (por ejemplo `X-Appwrite-Node`) no se aceptan desde clientes externos y se eliminan en el borde.

## 5. Una sola API key para varios servidores

- La key de un proyecto vive en la **BD central**. Todos los nodos `api` la validan contra los mismos datos.
- El endpoint público es **uno**. El balanceador reparte entre nodos `api`.
- Sesiones, rate limit y caché están en **Redis central**, así que no hay afinidad a un nodo.
- Requisitos: mismos secretos en todos los nodos (S12 a S14), mismo reloj (S18) y mismos dominios aceptados.
- Rotar o revocar una key aplica a todos los nodos a la vez.

**Pendiente de verificar en el código:** cómo se firman las keys dinámicas y qué secretos usan, para confirmar que S12 las cubre por completo.

## 6. Protocolo de unión (un comando)

```bash
curl -fsSL https://<dominio>/v1/nodes/install | sudo sh -s -- --token <token> [--role worker]
```

Pasos del script, en orden y deteniéndose ante cualquier fallo:

1. Comprueba sistema, Docker, reloj (NTP) y recursos mínimos.
2. Genera claves WireGuard locales (S5).
3. Presenta el token y su clave pública al control. Recibe la configuración del overlay y sus credenciales de BD por rol (S7, S12).
4. Levanta `wg0` con conexión saliente y aplica el firewall del rol (S10, S11).
5. Arranca el perfil de compose del rol.
6. Envía el primer heartbeat firmado. Si el control lo valida, pasa a `online`.

Comandos adicionales: `--status`, `--leave` (drena, espera a que terminen los trabajos y se retira) y `--rotate`.

Si cualquier paso falla, el script deja el servidor **sin cambios persistentes** (rollback) y muestra la causa.

## 7. Datos, almacenamiento y disponibilidad

- **Archivos:** storage S3-compatible (MinIO) en el nodo `data`. Los dispositivos locales no se usan con varios nodos.
- **Backups:** automáticos y programados desde el MVP, **cifrados** y almacenados fuera del nodo `data`. **Una restauración se prueba** antes de lanzar (criterio de aceptación).
- **Nodo `data` único:** es el punto crítico conocido. Réplica de Postgres y Redis queda planificada para una fase posterior.
- **Caída de un nodo worker:** los trabajos en curso se reintentan con la cola existente. El estado `draining` espera a que terminen antes de apagar.
- **Scheduler y mantenimiento:** un único ejecutor garantizado con lock en Redis con renovación (solo rol `control`).
- **Migraciones:** solo las ejecuta el control. Los demás nodos no aceptan tráfico hasta que terminen.

## 8. Rendimiento

- **Fase 0 (bloqueante):** medir la latencia de Postgres sobre WireGuard con una petición típica (de 20 a 50 queries). Si añade más de unos 5 ms por petición, la API pública se mantiene cerca de los datos y solo workers y executor se distribuyen.
- Conexiones persistentes y pool por nodo. Caché en Redis para lecturas frecuentes.
- Medir el tráfico del hub. Segundo hub o conexión directa entre nodos si es cuello de botella.

## 9. Observabilidad

- Cabecera `X-Appwrite-Node` en respuestas (no aceptada de entrada, S20) y `node.id` en spans con `Span::add`, según las convenciones del repo.
- Métricas por nodo: carga, heartbeat, versión, deriva de reloj, cola procesada.
- Eventos de auditoría: alta, baja, revocación, rotación, cambio de estado.

## 10. Compatibilidad

- `_APP_NODE_ROLE=all` por defecto: sin nodos, nada cambia.
- Variables nuevas en `app/config/variables.php`: `_APP_NODE_ROLE`, `_APP_NODE_ID`, `_APP_NODE_HUB`.
- No se cambian contratos existentes de la API ni de los SDK.

## 11. Diseño en el código (convenciones de AGENTS.md)

```
src/Appwrite/Platform/Modules/Nodes/
├── Module.php
├── Http/Nodes/{Create,Get,XList,Update,Delete}.php
├── Http/Nodes/Heartbeats/Create.php
├── Http/Nodes/Status/Update.php
├── Workers/Nodes.php
└── Services/{Http,Workers}.php
src/Appwrite/Network/Nodes/   (selector de nodo por carga y etiquetas)
bin/node-join                 (instalador)
docker-compose.node.yml       (perfiles por rol)
```

- Colección `nodes` en la config de colecciones de plataforma. Los atributos van en camelCase sin prefijo.
- Scopes `nodes.read` y `nodes.write`, solo administración.
- Sin regex (`preg_*`). Se usan funciones de cadena o validadores existentes.
- Sin clases `Helper` ni `Utils`. El selector de nodo es un concepto de dominio propio.
- Los guards de rol van en los puntos de arranque, no como funciones globales.

## 12. Fases y criterios de aceptación

| Fase | Entrega | Se acepta cuando |
|---|---|---|
| 0 | Prueba de latencia | Informe con cifras y decisión de qué se reparte |
| 1 | Colección `nodes`, tokens | S1, S2, S15 con pruebas |
| 2 | Módulo `Nodes` (API) | E2E de éxito y fallo para cada ruta |
| 3 | Instalador y overlay | Un comando deja un nodo `online`, con rollback ante error |
| 4 | Roles, enrutado, singleton | Reparto verificado y sin duplicados de scheduler |
| 5 | MinIO y realtime entre nodos | Archivo subido en un nodo visible desde otro |
| 6 | UI en la consola | Alta, drenado y revocación desde la UI |
| 7 | Endurecimiento | Todos los S1 a S20 verificados |

El MVP son las fases 0 a 4. No se considera lanzable sin las fases 5 y 7 para uso en producción.

## 13. Plan de pruebas

**Unit** (`tests/unit/`, solo librerías locales):
- Selector de nodos: carga, etiquetas, afinidad, `draining` y `offline`.
- Validación y caducidad del token, verificación de firma y anti-replay de heartbeat.

**E2E** (`tests/e2e/Services/Nodes/`):
- Éxito y fallo de cada ruta: scopes, permisos, token usado, caducado o de otro rol.
- Una key creada en el control funciona en N peticiones seguidas contra distintos nodos.
- Revocar un nodo corta su acceso a BD y al overlay.

**Manuales o de caos:**
- Apagar un nodo en pleno trabajo: sale del reparto y sus trabajos se reintentan.
- Restaurar un backup completo en un entorno limpio.
- Deriva de reloj simulada: el nodo es rechazado.
- Rotación de claves con un nodo desconectado.

## 14. Registro de riesgos abiertos

| Riesgo | Mitigación | Estado |
|---|---|---|
| Nodo `data` único | Backups probados ahora, réplica después | Abierto |
| Latencia sobre el túnel | Fase 0 decide el reparto | Abierto |
| Hub WireGuard como punto de fallo | Segundo hub en fase posterior | Abierto |
| Instalador con `curl \| sh` | S4 y modo manual | Mitigado en diseño |
| Claves dinámicas y secretos compartidos | Verificar en código | Pendiente de verificar |
| Estado local fuera de Redis/BD | Auditar antes de la Fase 4 | Pendiente de verificar |

## 15. Decisiones pendientes

1. Túnel: WireGuard con hub propio (recomendado) o un servicio de terceros (Cloudflare Tunnel, Tailscale).
2. Storage: MinIO/S3 central (recomendado) u otra opción.
3. Ubicación de ClickHouse y servicios auxiliares (assistant, browser, embedding, geo): nodo `data` o `worker`.
4. Repositorio de la consola para la UI de la Fase 6.

## 16. Lista de verificación para lanzar

- [ ] Fase 0 documentada con cifras reales.
- [ ] S1 a S20 con prueba asociada y en verde.
- [ ] Backup restaurado con éxito en un entorno limpio.
- [ ] Prueba de caída de nodo superada.
- [ ] `_APP_NODE_ROLE=all` verificado sin regresiones en la suite E2E existente.
- [ ] Documentación de usuario y de operación publicada.
- [ ] Revisión de seguridad independiente del instalador y del protocolo de unión.
