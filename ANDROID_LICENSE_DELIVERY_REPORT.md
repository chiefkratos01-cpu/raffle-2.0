# Informe de entrega — Raffle 2.0 Android y licencias

**Fecha:** 30 de septiembre de 2026  
**Empresa:** Codexia Systems  
**Desarrollador:** Carlos Alejandro Arcia Miranda  
**Versión del proyecto:** `2.0.0-rc.1`  
**Estado:** código integrado y compilado para pruebas; **no es una publicación de producción**.

> **Importante:** se compilaron un APK de depuración y un AAB como pruebas de compilación usando URLs `.invalid` y una clave pública ficticia. No son clientes conectables ni se incluyen en la entrega. El AAB no está firmado con una clave de publicación del propietario. No se probó la app en un teléfono o emulador conectado.

## A. Arquitectura utilizada

Se añadió un proyecto Android **híbrido con Capacitor 8**, reutilizando el frontend React existente y empaquetando sus recursos localmente dentro de Android; no abre una web remota de Chrome. FastAPI y la base de datos siguen siendo las autoridades para autenticación, roles, rifas, pagos, tickets, sorteo y límites.

La interfaz React se ejecuta en el WebView que forma parte de la arquitectura híbrida; no se reescribió en Jetpack Compose. Se añadieron puentes nativos de Capacitor para almacenamiento seguro, Back, red, compartir, portapapeles, vibración/háptica, teclado, barra de estado, enlaces externos y splash.

- **Application ID:** `com.codexiasystems.raffle2`
- **Android mínimo:** API 24
- **Target/compile SDK:** API 36
- **Nombre/versión:** Raffle 2.0 · `2.0.0-rc.1`

## B. Funcionalidades convertidas

Se conserva el frontend existente y su comunicación con la API/Socket.IO. Esto mantiene, conforme al producto fuente, los flujos de cliente, inicio de sesión/registro, exploración y detalle de rifas, selección y reserva de boletos, envío de información de pago para revisión, consulta de tickets/resultados, notificaciones dentro de la app, perfil, solicitud de acceso de organizador y paneles administrativos sujetos a rol.

No se sustituyeron pagos manuales por Stripe, PayPal u otra pasarela, ni se usaron datos simulados para reemplazar el backend real.

## C. Funcionalidades modificadas para móvil

- Se configuraron safe areas, viewport, barra de estado, teclado, splash e iconos/adaptive icons derivados del logo existente.
- El Back de Android utiliza la integración de Capacitor y permite cerrar primero los overlays/modal de pago y edición administrativa registrados; la navegación normal conserva las rutas existentes.
- La selección de tickets puede dar una háptica ligera; hay banner de conectividad y se usan los diálogos nativos de compartir y portapapeles cuando existen.
- El enlace de WhatsApp se entrega al resolutor de intents de Android. Si WhatsApp no puede atenderlo, la URL HTTPS `wa.me` queda disponible para el navegador.
- Los QR existentes siguen mostrando enlaces de verificación. Los enlaces de rifa/verificación también aceptan el esquema `raffle://` para abrir la pantalla correspondiente dentro de la app; al configurar la web pública se generan enlaces HTTPS compartibles.
- La fuente genera/muestra QR, pero no tenía un escáner QR integrado. No se inventó un flujo de escaneo ni se pidió permiso de cámara; la lectura puede hacerse con la cámara/lector del dispositivo.
- Se añadió la atribución `© Codexia Systems` y `Developed by Carlos Alejandro Arcia Miranda`.

## D. Problemas encontrados en el proyecto original

1. El ZIP era un producto React/FastAPI, sin proyecto Android nativo ni configuración de build para Android.
2. La persistencia web usaba `localStorage`; no proporcionaba almacenamiento nativo protegido por Android Keystore.
3. No existían manejadores nativos para Back, intents externos, háptica, red ni caché cifrada offline.
4. La licencia existente no representaba los claims v2 solicitados ni aplicaba límites de licencia server-side a la creación/publicación de rifas, boletos, cuentas de equipo y organizadores.
5. Una migración ya existente (`f1c2d3e4b5a6`) fallaba al retirar un default temporal en SQLite, impidiendo probar la cadena completa en una base SQLite limpia.
6. El proyecto fuente no proporciona un backend de producción, una clave pública Ed25519 del emisor ni una keystore de publicación. Tampoco incorpora proveedor de push ni escáner de cámara.

## E. Problemas corregidos

- Se creó el proyecto Capacitor/Android y las adaptaciones nativas indicadas arriba.
- Se movieron los tokens de acceso/refresh y las respuestas GET privadas almacenables a un puente cifrado AES-GCM respaldado por Android Keystore; al cerrar sesión o cambiar de usuario se limpian las instantáneas privadas.
- Se agregó el contrato de licencia Ed25519 schema v2 y su aplicación en backend; se distingue vencimiento natural de revocación y se mantienen intactos los datos al cambiar el estado.
- Se ajustó la migración heredada para soportar SQLite y se añadió la migración de claims de licencia reversible.
- La prueba de humo del Keygen comprobó los valores Trial exactos, duración inicial vacía y defaults Commercial inicialmente vacíos. Se eliminaron semánticas comerciales no especificadas como “sin límite”.
- Se incorporó la prueba Android local para rechazar claims de cuota ausentes, nulos, negativos, fraccionarios o booleanos.

## F. Cambios realizados en backend

- Se acepta y verifica firma Ed25519 para claims v2: `license_id`, `license_type` (`TRIAL`/`COMMERCIAL`), `organization_id`, fechas, estado firmado, versión de esquema, cuotas, `limits`, `features` e instalación cuando se vincula.
- El backend es autoridad para activar/verificar licencia y para permitir o rechazar operaciones licenciadas; modificar el APK, almacenamiento o request no eleva los límites.
- Las cuotas firmadas se aplican a rifas activas/publicación, máximo de boletos por rifa, cuentas de equipo y aprobación de organizadores. Los puntos de mutación bloquean/refrescan la fila de licencia para serializar las comprobaciones concurrentes.
- Trial inicial firmado: **3 rifas activas simultáneas, 100 boletos por rifa, 2 cuentas de equipo/administración y 1 organizador asociado**. Su vencimiento se elige desde el Keygen (días o fecha); Android no fija duración.
- Commercial no recibe cifras predeterminadas. El Keygen exige un entero no negativo explícito para cada uno de los cuatro máximos estándar; el emisor puede cambiar esos valores en configuración Keygen por usuario sin modificar Android. El objeto `limits`/`features` permite extender claims futuros; los límites futuros no se activaron.
- Se distinguen `ACTIVE`, `EXPIRED`, `REVOKED` y `SUSPENDED`; la expiración se deriva de fechas y la revocación/suspensión activa se conserva en la fila del servidor. Los cambios administrativos requieren `SUPER_ADMIN` y se auditan.
- `LICENSE_CHECK_INTERVAL_HOURS` es configurable (por defecto actual: 24 horas); `LICENSE_OFFLINE_GRACE_HOURS` exige decisión explícita para producción; `LICENSE_SUSPENDED_POLICY` permite `READ_ONLY` o `ALLOW_EXISTING_COMPLETION`.
- Offline conserva consultas/instantáneas disponibles dentro del período de gracia; no permite eludir mutaciones/cuotas server-side. No se cierran, borran ni destruyen rifas/tickets por pérdida de conexión o revocación.

## G. Cambios realizados en base de datos

La migración `a6d4f20c9b31_versioned_license_claims` añade identificación externa de licencia, asociación de organización y versión de esquema, sin borrar filas. Es reversible.

Se corrigió la operación de eliminación del default temporal en `f1c2d3e4b5a6` para compatibilidad con SQLite. La cadena Alembic completa se aplicó sobre SQLite limpia; se probó downgrade de la migración nueva y re-upgrade. **No se aplicó esta migración a una base de producción ni se validó contra la base desplegada.**

## H. Cambios de seguridad

- Acceso/refresh tokens, licencia firmada y caché GET privada se guardan cifrados dentro del almacenamiento app-private; el puente Android usa AES-GCM y Android Keystore.
- Se deshabilitó backup Android para evitar restaurar fuera de su contexto los datos cifrados locales.
- El cliente sólo contiene la clave pública para verificación; **no se generó ni incluyó clave privada, frase de desbloqueo, token SUPER_ADMIN, keystore ni licencia de cliente**.
- El Keygen cifra claves privadas nuevas con derivación Scrypt y Fernet autenticado; solicita la frase de desbloqueo y permite firmar/verificar localmente. El backend continúa validando claims y autorizando las operaciones.
- Los tokens/cache privados se limpian al logout y los snapshots de otro usuario no se reutilizan.

## I. Permisos Android utilizados

El APK de prueba declara:

- `android.permission.INTERNET`
- `android.permission.ACCESS_NETWORK_STATE`
- `android.permission.VIBRATE`
- `com.codexiasystems.raffle2.DYNAMIC_RECEIVER_NOT_EXPORTED_PERMISSION` (infraestructura del stack/plugin Capacitor)

No declara cámara, ubicación, acceso amplio a almacenamiento ni permiso de notificaciones push. Los inputs de imagen existentes pueden usar el selector de archivos/fotos del sistema. No se añadió escaneo de QR con cámara ni infraestructura push inexistente en el proyecto fuente.

## J. Dependencias añadidas

Al frontend se añadieron Capacitor Android 8 y sus paquetes `@capacitor/core`, `@capacitor/cli`, `@capacitor/android`, `@capacitor/app`, `@capacitor/app-launcher`, `@capacitor/clipboard`, `@capacitor/haptics`, `@capacitor/keyboard`, `@capacitor/network`, `@capacitor/share`, `@capacitor/splash-screen` y `@capacitor/status-bar`; para validar firmas desde el cliente se añadió `@noble/ed25519` 3.2.0. Se actualizó `pnpm-lock.yaml`.

Para el Keygen Windows se documentaron las dependencias de empaquetado en `Backend/tools/requirements-keygen-windows.txt` (Cryptography y PyInstaller); Tkinter forma parte de Python en Windows.

## K. Dependencias eliminadas

**Ninguna** de las dependencias declaradas originalmente se eliminó. El build usa pnpm 10.15.1 conforme al manifiesto.

## L. Pruebas realizadas

- Backend: **86 pruebas pasaron**; `compileall` y `py_compile` también pasaron.
- Frontend/Jest: **38 pruebas pasaron**.
- `react-scripts build`: compilación de producción pasó con endpoints de prueba `.invalid` y una clave pública ficticia.
- Alembic: upgrade completo SQLite, downgrade de la nueva migración y re-upgrade pasaron.
- Keygen GUI: inicialización y configuración bajo Xvfb; Trial 3/100/2/1 y duración vacía; Commercial inicialmente vacío y valores configurados guardados/releídos.
- Capacitor sincronizó 9 plugins. `assembleDebug` y `bundleRelease` terminaron con `BUILD SUCCESSFUL`.
- Se inspeccionó el APK: paquete/versión correctos, `minSdk 24`, `targetSdk/compileSdk 36`, permisos enumerados y firma debug Android válida. El AAB no tiene firma de distribución.
- Tras borrar dependencias/caches/builds generados, `python3 Backend/tools/release_check.py`: **PASS**.

## M. Errores encontrados durante las pruebas

- La primera instalación/build de herramientas Android falló por una descarga de Build Tools dañada y por disponer inicialmente de JRE sin compilador Java.
- Las primeras pruebas del helper de licencia fallaron por una ruta de mock incorrecta y por incompatibilidad del entry point ESM de noble-ed25519 con el runner Jest de esta app.
- El gate estático rechaza intencionalmente `node_modules` y caches dentro del árbol fuente.

## N. Errores corregidos

- Se instaló JDK 21 y Android Build Tools 36.1.0 para compilar con SDK/API 36.
- La comprobación pura de cuotas se aisló en un módulo sin dependencia ESM directa; sus pruebas pasan sin sustituir la verificación criptográfica del producto.
- Se eliminaron dependencias, builds y caches locales antes de repetir el release gate, que pasó.
- La migración SQLite y los claims v2 incompletos/nulos se corrigieron y se volvieron a probar.

## O. Errores pendientes

No hay fallos de compilación o de las suites automatizadas reportadas arriba. Sí quedan **bloqueos previos a producción**:

1. El propietario debe proporcionar/configurar URLs HTTPS reales de API, Socket.IO y web pública, además de una clave pública Ed25519 que corresponda al emisor autorizado.
2. Se deben elegir `LICENSE_OFFLINE_GRACE_HOURS`, política de suspensión y valores Commercial reales; aplicar la migración en una base respaldada y revisar compatibilidad con las licencias v1 existentes.
3. No hay teléfono ni emulador conectado: navegación, pantalla, rotación, teclado, Back, intents, haptics, offline y permisos no se validaron manualmente en dispositivo.
4. No se generó el ejecutable Windows ni se ejecutó en Windows; el empaquetado de PyInstaller debe hacerse allí.
5. No se incluyó keystore de release ni se firmó el AAB de producción.
6. La fuente no tenía escáner QR ni proveedor push; no se añadieron ni se validaron esas capacidades.

## P. Cómo ejecutar el proyecto

Requisitos usados/esperados: Node.js 22, pnpm 10.15.1, Python con dependencias de `Backend/requirements.txt`, JDK 21, Android SDK API 36 y Build Tools 36.1.0.

```bash
cd Backend
python3 -m pip install -r requirements.txt
python3 -m pytest -q

cd ../Frontend
pnpm install --frozen-lockfile
export REACT_APP_API_URL=https://<api-real>/api/v1
export REACT_APP_SOCKET_URL=https://<api-real>
export REACT_APP_PUBLIC_URL=https://<web-publica-real>
export REACT_APP_LICENSE_PUBLIC_KEY='<clave-publica-ed25519-real>'
pnpm android:debug
```

En backend, configurar `LICENSE_PUBLIC_KEY`, los valores Trial de entorno si corresponden, `LICENSE_OFFLINE_GRACE_HOURS`, `LICENSE_CHECK_INTERVAL_HOURS` y `LICENSE_SUSPENDED_POLICY`; revisar `.env.example` y respaldar la base antes de `python3 -m alembic upgrade head`.

## Q. Cómo generar APK

- APK debug: `cd Frontend && pnpm android:debug`.
- Salida: `Frontend/android/app/build/outputs/apk/debug/app-debug.apk`.
- Para un APK release, primero configura la firma de release del propietario y ejecuta desde `Frontend/android`: `./gradlew assembleRelease`.

El APK generado durante esta tarea se firmó con la clave Android debug del entorno y **no** con la clave de publicación. La URL/key ficticias hacen que no sea utilizable; no se distribuye.

## R. Cómo generar AAB

- Configura primero la firma release privada del propietario, sin subir keystores/secretos al repositorio.
- Ejecuta `cd Frontend && pnpm android:release`.
- Salida esperada: `Frontend/android/app/build/outputs/bundle/release/app-release.aab`.

El AAB de esta verificación sólo prueba compilación; quedó **sin firma de publicación** y con configuración ficticia, así que no se adjunta ni publica.

## S. Qué debe probar el desarrollador manualmente

1. Con backend real y una base respaldada, migración, login/refresh/logout, permisos por rol y compatibilidad de licencias v1/v2.
2. Alta, publicación, límites de Trial/Commercial, concurrencia de rifas/cuentas, creación/edición de tickets, reserva, revisión manual de pago, sorteo y consulta de resultados.
3. Instalar en Android API 24 y una versión moderna/API 36; verificar teléfonos pequeños/grandes, safe areas, portrait, contraste/dark mode, teclado, overlays, formularios y navegación Back/gestos.
4. Hacer taps en WhatsApp tanto con WhatsApp instalado como sin él; verificar fallback HTTPS en navegador y compartir/abrir deep links QR de rifa/ticket/resultado.
5. Verificar estados de red y caché durante desconexión, caducidad de gracia, expiración, suspensión y revocación; confirmar que las consultas existentes siguen disponibles y las mutaciones protegidas no eluden al backend.
6. Revisar lector de pantalla/tamaños táctiles, selector de imágenes del sistema y lista final de permisos en el dispositivo.
7. En Windows, crear/guardar claves de emisor protegidas fuera del proyecto, probar emisión Trial y Commercial con números elegidos, validar firma y estado `SUPER_ADMIN`, y generar `RaffleKeygen.exe` con `Backend/tools/build_windows_keygen.bat`.
8. Con la keystore del propietario, firmar el release AAB, verificarlo con las credenciales oficiales y sólo entonces preparar distribución.

---

**Conclusión:** el árbol fuente Android/licencias queda implementado y validado por suites automatizadas y compilación de humo. Se entrega el código fuente y este informe; no se declara una release de producción porque faltan endpoints/clave reales, firma release y validación manual en dispositivos.