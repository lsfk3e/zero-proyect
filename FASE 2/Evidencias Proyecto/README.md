# Project Z.E.R.O

Sistema de protección remota de información sensible en equipos corporativos Windows 10/11, construido con **ASP.NET Core 8**, **PostgreSQL (Supabase)**, **MinIO S3 (Docker)**, un **panel web administrativo** (HTML/CSS/JS modular) y un **agente cliente Windows (.NET 8)**.

La protección preventiva cifra los archivos de la carpeta controlada `TestData` en un contenedor autenticado **AES-256-GCM**, genera respaldos cifrados independientes en MinIO S3 (`ZERO1`), permite revocar claves de forma irreversible (automáticamente al reportar robo o manualmente), mantiene telemetría de seguimiento y ejecuta órdenes de eliminación remota sobre `TestData` con trazabilidad completa en auditoría. No implementa bloqueo de Windows/BIOS ni rastreo GPS satelital.

---

## Estructura del repositorio

| Ruta | Descripción |
|---|---|
| `zero/` | API REST en ASP.NET Core 8 y panel web estático (`zero/wwwroot/`). |
| `AgenteClienteDemo/` | Agente cliente Windows (.NET 8): vinculación, cifrado local, respaldos, telemetría y monitor de órdenes. |
| `scripts-sql/` | Scripts base y migraciones incrementales (`002` a `007`) para PostgreSQL/Supabase. |
| `scripts-agente/` | Scripts PowerShell para publicar e instalar el inicio automático del agente en el Programador de tareas de Windows. |
| `postman/` | Colección funcional QA `ProjectZERO.postman_collection.json` (35 peticiones / 37 aserciones). |
| `tests/` | Suites automatizadas `.NET` (`RespaldoCifrado.Tests`, `AgenteMonitor.Tests`, `GestionClaves.Tests`), E2E web con Playwright (`PanelClaves.Tests/pruebas.cjs`) y plan de carga JMeter (`rendimiento-zero.jmx`). |
| `docker-compose.yml` | Servicio local de almacenamiento de objetos **MinIO (S3)** en `127.0.0.1:9000/9001`. |

---

## 1. Base de datos y migraciones SQL

Aplicar el esquema base (`scripts-sql/script_supabase.sql`) y luego las migraciones incrementales en orden (la API no ejecuta migraciones automáticamente):

1. `scripts-sql/002_telemetria_dispositivo.sql` — columnas de telemetría (`ultima_conexion`, `ip_publica`, `ubicacion_aproximada`).
2. `scripts-sql/003_gestion_claves.sql` — protección de claves criptográficas por dispositivo y auditoría directa por dispositivo.
3. `scripts-sql/004_identidad_agente.sql` — hash SHA-256 de credencial individual del agente (`agente_cliente.credencial_hash`) y administrador emisor.
4. `scripts-sql/005_sensibilidad_dispositivo.sql` — nivel de sensibilidad por dispositivo (`sin_clasificar`, `baja`, `media`, `alta`).
5. `scripts-sql/006_rol_supervisor.sql` — rol `Supervisor` de solo lectura.
6. `scripts-sql/007_fecha_reporte_robo.sql` — marca temporal `dispositivo.fecha_reporte_robo` para confirmar sincronización tras reporte de robo.

*(Para respaldos externos en MinIO, la tabla `respaldo` incluye `ubicacion_remota` y `tamano_bytes`).*

---

## 2. Configuración local y secretos

La API (`zero`) usa el User Secrets ID `project-zero-luis-dev` y el agente guarda sus credenciales individuales y URL en `project-zero-agente-demo`. No guardar contraseñas, JWT keys ni cadenas de conexión en el repositorio.

### Secretos requeridos en `zero` (`--project zero`)
- `ConnectionStrings:DefaultConnection` — cadena de conexión PostgreSQL/Supabase.
- `Jwt:Key` — clave simétrica para firmar tokens JWT.
- `KeyManagement:MasterKey` — clave maestra Base64 de **32 bytes** para proteger las claves de dispositivo en BD (generar una sola vez; no reemplazarla si ya existen sobres creados).
- `Agent:BackupEncryptionKey` — clave independiente Base64 de **32 bytes** para cifrar/descifrar los respaldos `ZERO1` en servidor y en recuperación local.
- `Storage:Endpoint`, `Storage:Bucket`, `Storage:AccessKey`, `Storage:SecretKey` — configuración S3 de MinIO.

Generación de claves aleatorias de 32 bytes en PowerShell:

```powershell
$claveMaestra = [Convert]::ToBase64String([System.Security.Cryptography.RandomNumberGenerator]::GetBytes(32))
dotnet user-secrets set "KeyManagement:MasterKey" "$claveMaestra" --project zero
Remove-Variable claveMaestra

$claveRespaldo = [Convert]::ToBase64String([System.Security.Cryptography.RandomNumberGenerator]::GetBytes(32))
dotnet user-secrets set "Agent:BackupEncryptionKey" "$claveRespaldo" --project zero
Remove-Variable claveRespaldo
```

En Development, los servicios criptográficos releen User Secrets para evitar copias obsoletas en el host; en Production usan exclusivamente la configuración del entorno sin fallback ni generación silenciosa.

---

## 3. Almacenamiento de respaldos con MinIO (Docker)

Abre **Docker Desktop** y levanta MinIO con tu archivo local de credenciales fuera de Git (referencia en `.env.example`, por ejemplo `%LOCALAPPDATA%\ProjectZERO\minio.env`):

```powershell
$minioEnv = Join-Path $env:LOCALAPPDATA 'ProjectZERO\minio.env'
docker compose --env-file $minioEnv up -d
docker compose --env-file $minioEnv ps
```

- Los puertos `9000` (API S3) y `9001` (consola web) se publican solo en `127.0.0.1`.
- El volumen `minio-data` conserva los objetos `.zip.enc`.
- En los secretos de `zero`, configurar `Storage:Endpoint=http://127.0.0.1:9000`, `Storage:Bucket=respaldos`, `Storage:AccessKey` y `Storage:SecretKey`, y reiniciar la API.
- Los respaldos cifrados subidos por el agente se pueden listar (`GET /api/Respaldo/dispositivo/{id}`), descargar cifrados (`GET /api/Respaldo/{id}/descargar`) o descargar descifrados en `.zip` desde el panel web (**⋮ → Ver respaldos**, solo Administrador, mediante `GET /api/Respaldo/{id}/descargar-zip`, con registro en auditoría y limpieza de memoria).

---

## 4. Ejecutar la API y el Panel Web

```powershell
dotnet run --project zero --launch-profile http
```

- **Panel web:** `http://localhost:5137/`
- **Swagger / OpenAPI (solo Development):** `http://localhost:5137/swagger`
- **Acceso externo para pruebas en 2 PCs:** la API incluye `UseForwardedHeaders` (`X-Forwarded-For`, `X-Forwarded-Proto`), compatible con **Visual Studio Dev Tunnels** (público y persistente) o proxy inverso TLS.

### Seguridad HTTP y Control de Acceso (RBAC)
- **Cabeceras de seguridad (`zero/Program.cs`):** oculta `Server: Kestrel` (`AddServerHeader = false`) e inyecta `Content-Security-Policy` estricto (`default-src 'self'`, sin `'unsafe-inline'` en el panel), `X-Frame-Options: DENY`, `X-Content-Type-Options: nosniff`, `Referrer-Policy: no-referrer`, `Permissions-Policy`, `Cross-Origin-Opener-Policy: same-origin`, `Cross-Origin-Resource-Policy: same-origin`, `Strict-Transport-Security` en HTTPS y `Cache-Control: no-store` en `/api/*`.
- **Sesión en el panel:** JWT guardado únicamente en memoria (recargar la página solicita iniciar sesión nuevamente).
- **Roles:**
  - **Administrador:** registra dispositivos, clasifica sensibilidad, administra claves, marca robos, emite órdenes de eliminación, consulta/descarga respaldos `.zip`, audita y genera reportes.
  - **Supervisor:** acceso de solo lectura a dispositivos, telemetría, órdenes, auditoría y reportes. El panel oculta acciones administrativas y la API responde `403 Forbidden` en cualquier endpoint restringido.

### Flujo operativo en el Panel Web
1. **Registrar dispositivo:** exige nombre (1–150 caracteres) y nivel de sensibilidad obligatorio (`baja`, `media` o `alta`; rechaza `sin_clasificar`). Se asigna automáticamente al Administrador autenticado y **crea automáticamente una clave criptográfica activa** en la misma transacción.
2. **Clasificar sensibilidad (`⋮ → Clasificar información`):** permite reclasificar entre `baja`, `media`, `alta` o `sin_clasificar`, auditando cada cambio real.
3. **Administrar claves (`⋮ → Administrar claves`):** consulta historial sin exponer material secreto, permite crear clave en equipos activos sin clave activa y revocar manualmente con motivo obligatorio (5–80 caracteres) y confirmación irreversible.
4. **Ver respaldos (`⋮ → Ver respaldos`, solo Administrador):** lista copias cifradas en MinIO con fecha y tamaño y permite descargar el `.zip` descifrado dejando registro en bitácora.
5. **Marcar robado (`Marcar robado`):** cambia el estado a `robado`, registra `fecha_reporte_robo` y **revoca automáticamente la clave activa** (`material_protegido = NULL`) dentro de una transacción con bloqueo `FOR UPDATE`.
   - En la tabla se muestra **`Robado · Pendiente`** hasta que el equipo robado vuelva a enviar telemetría posterior a `fecha_reporte_robo`, momento en que pasa a **`Robado · Confirmado`** y se registra un evento en auditoría.
6. **Enviar eliminación (`Enviar eliminación`):** solo disponible para equipos en estado `robado` sin orden pendiente. Al confirmar ejecución el agente, el equipo pasa a `eliminado` y se conservan los respaldos en MinIO.
7. **Ver incidente y Reporte HTML:** consolida estado, `fecha_reporte_robo`, sensibilidad, telemetría, respaldos, órdenes y bitácora, con descarga en JSON y HTML.

---

## 5. Agente Cliente Windows (`AgenteClienteDemo`)

Todos los endpoints del agente (`AgenteClienteController` y `AgenteClaveController`) autentican exclusivamente mediante la credencial individual por dispositivo **`X-Device-Credential`** (hash SHA-256 en BD) sobre HTTPS (HTTP permitido solo en loopback de desarrollo). La antigua clave compartida `X-Agent-Key` fue retirada y devuelve `401`.

### 5.1 Configurar URL de la API (si corre en otro PC)
```powershell
dotnet user-secrets set "Agent:ApiBaseUrl" "https://TU-TUNEL-O-SERVIDOR" --id project-zero-agente-demo
```

### 5.2 Vincular un dispositivo activo (ejemplo ID `8`)
```powershell
dotnet run --project AgenteClienteDemo -- 8 vincular
```
Solicita credenciales de Administrador (ocultas), genera una credencial aleatoria de 32 bytes, envía únicamente su hash a `POST /api/AgenteCredencial/dispositivo/8` y guarda el secreto localmente en `project-zero-agente-demo` (`Agent:DeviceCredentials:8`). Repetir `vincular` rota la credencial e invalida la anterior.

### 5.3 Carpetas aisladas por dispositivo
Por defecto (`AppContext.BaseDirectory`, o `Agent:DataInstallationDirectory`), el agente crea un directorio independiente por ID:

```text
AgenteClienteDemo/bin/Debug/net8.0/Dispositivos/8/
  TestData/          # Archivos monitoreados (cifrado inicial y borrado remoto)
  DatosProtegidos/   # Contenedor cifrado AES-256-GCM (dispositivo-8.zero.enc)
  Backups/           # Copias locales .zip.enc (formato ZERO1)
  Restored/          # Destino de restauración local
  Ordenes/           # Diario persistente anti-duplicados ({idOrden}.json)
```

Un archivo `.agente.lock` impide ejecutar dos instancias simultáneas sobre el mismo ID.

### 5.4 Modos de ejecución
```powershell
dotnet run --project AgenteClienteDemo -- 8 monitor          # Modo continuo recomendado
dotnet run --project AgenteClienteDemo -- 8 proteger         # Protección inicial manual de TestData
dotnet run --project AgenteClienteDemo -- 8 verificar-datos  # Verifica integridad del contenedor en memoria
dotnet run --project AgenteClienteDemo -- 8 respaldo         # Fuerza respaldo inmediato
dotnet run --project AgenteClienteDemo -- 8 ordenes          # Consulta y ejecuta órdenes una sola vez
dotnet run --project AgenteClienteDemo -- 8 restaurar "RUTA_DEL_ARCHIVO.zip.enc"
```

Comportamiento del modo **`monitor`**:
- **Protección inicial automática:** si aparecen archivos en `TestData` y el dispositivo tiene credencial vinculada, cifra los archivos en `DatosProtegidos`, sube el respaldo cifrado a la API/MinIO y retira los originales.
- **Respaldos y telemetría desacoplados:** los respaldos solo corren mientras el equipo está `activo` (`Agent:BackupIntervalSeconds`, default `60 s`); la telemetría se envía en `activo`, inmediatamente al detectar cambio a `robado` (confirmando el robo en el panel), tras confirmar una eliminación y periódicamente incluso en estado `eliminado` sin repetir borrados ni recrear archivos.
- **Resiliencia:** reintentos con espera progresiva (`5–60 s`) ante cortes/timeouts, diario persistente en `Ordenes/{idOrden}.json` para no duplicar eliminaciones y conciliación ante `HTTP 409`.
- **Intervalos configurables:** `Agent:PollIntervalSeconds` (`10 s`), `Agent:BackupIntervalSeconds` (`60 s`), `Agent:RequestTimeoutSeconds` (`15 s`).

### 5.5 Inicio automático al iniciar sesión en Windows
`scripts-agente/Instalar-InicioAutomatico.ps1` publica el agente en `artifacts/agente-automatico/{id}/{version}/app` y registra la tarea `ZERO-Agente-{id}` en el Programador de tareas para el usuario actual (con logs rotativos en `logs/agente.log`, sin ventana y sin guardar contraseñas):

```powershell
.\scripts-agente\Instalar-InicioAutomatico.ps1 -IdDispositivo 8
Start-ScheduledTask -TaskName 'ZERO-Agente-8'
Get-ScheduledTask -TaskName 'ZERO-Agente-8' | Select-Object TaskName, State
```

Para deshabilitar o quitar la tarea conservando los datos:
```powershell
Disable-ScheduledTask -TaskName 'ZERO-Agente-8'
Stop-ScheduledTask -TaskName 'ZERO-Agente-8'
Unregister-ScheduledTask -TaskName 'ZERO-Agente-8' -Confirm:$false
```

---

## 6. Pruebas automatizadas y suite QA

### 6.1 Compilación y suites .NET (unitarias + integración PostgreSQL y MinIO)
```powershell
dotnet build zero.slnx -c Release -m:1 -p:UseSharedCompilation=false
dotnet run --project tests/RespaldoCifrado.Tests
dotnet run --project tests/AgenteMonitor.Tests -c Release
dotnet run --project tests/GestionClaves.Tests -c Release -- --http --integracion --minio
```
- `RespaldoCifrado.Tests` y `AgenteMonitor.Tests` no requieren base de datos ni secretos.
- `GestionClaves.Tests -- --http --integracion --minio` crea un esquema temporal `zero_test_claves_*` en PostgreSQL y un bucket temporal `zero-prueba-*` en MinIO, probando AES-GCM, validación HTTP, migraciones repetibles, RBAC, revocación automática al marcar robado, confirmación por telemetría, descarga `.zip` byte a byte y flujo completo de agente.

### 6.2 Pruebas E2E del Panel Web (Node.js + Playwright)
```powershell
$pw = Join-Path $env:TEMP 'zero-playwright'
New-Item -ItemType Directory -Path $pw -Force | Out-Null
npm.cmd --prefix $pw install playwright --ignore-scripts
$env:NODE_PATH = Join-Path $pw 'node_modules'
node tests/PanelClaves.Tests/pruebas.cjs
```
*(Usa Edge/Chromium en modo headless con API simulada; valida clasificación, registro con sensibilidad, robo pendiente/confirmado, descarga ZIP, gestión de claves, respuesta perdida y vista móvil).*

### 6.3 Pruebas funcionales de API (Postman)
Importar `postman/ProjectZERO.postman_collection.json` en Postman, completar `adminEmail`, `adminPassword` (y opcionalmente `supervisorEmail`, `supervisorPassword`) en la pestaña **Variables** y ejecutar **Collection Runner** (35 peticiones / 37 aserciones en 6 carpetas).

### 6.4 Pruebas de carga y rendimiento (Apache JMeter vía Docker)
Con la API en ejecución (`http://localhost:5137`):
```powershell
$out = Join-Path $env:TEMP 'zero-jmeter-out'
if (Test-Path $out) { Remove-Item $out -Recurse -Force }
New-Item -ItemType Directory -Path $out | Out-Null

docker run --rm `
  -v "${PWD}\tests:/tests:ro" `
  -v "${out}:/out" `
  justb4/jmeter:5.5 `
  -n -t /tests/rendimiento-zero.jmx `
  "-Jhost=host.docker.internal" `
  "-Jport=5137" `
  "-JadminEmail=TU_ADMIN" `
  "-JadminPassword=TU_PASSWORD" `
  -l /out/resultados.jtl `
  -e -o /out/reporte-html

Start-Process "$env:TEMP\zero-jmeter-out\reporte-html\index.html"
```

### 6.5 Pruebas de seguridad DAST (OWASP ZAP vía Docker)
```powershell
$zapOut = Join-Path $env:TEMP 'zero-zap'
New-Item -ItemType Directory -Path $zapOut -Force | Out-Null

# Escaneo Baseline del panel web
docker run --rm -v "${zapOut}:/zap/wrk/:rw" `
  ghcr.io/zaproxy/zaproxy:stable `
  zap-baseline.py -t http://host.docker.internal:5137 -r zap-baseline.html

# Escaneo OpenAPI de los endpoints de la API
docker run --rm -v "${zapOut}:/zap/wrk/:rw" `
  ghcr.io/zaproxy/zaproxy:stable `
  zap-api-scan.py -t http://host.docker.internal:5137/swagger/v1/swagger.json -f openapi -r zap-api.html
```
