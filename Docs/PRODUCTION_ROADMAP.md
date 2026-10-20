# SWUI — Auditoría y plan hacia producción

Fork: `dannybend/SWUI` (upstream `ZenaMel/SWUI`, base `021a118`).
Fecha de la auditoría: 2026-10-20.

Este documento tiene dos partes:

1. **Auditoría**: las grietas encontradas en el código, ordenadas por gravedad, con su ubicación.
2. **Plan de acción**: fases con criterios de salida, más un proceso continuo para seguir buscando grietas.

Gravedad:

- **P0**: bloquea producción. Crash, vulnerabilidad o "no se puede empaquetar".
- **P1**: bug real que un usuario va a encontrar.
- **P2**: deuda que frena la evolución o la portabilidad.
- **P3**: limpieza.

---

## Parte 1 — Auditoría

### A. Seguridad (superficie CEF y puente JS → UE)

| # | Grav. | Hallazgo | Ubicación |
|---|---|---|---|
| A1 | P0 | `remote_debugging_port = 7777` fijo, también en Shipping. Cualquier proceso local puede conectarse a DevTools y ejecutar JS dentro de la UI, que a su vez puede invocar comandos UE. | `SwuiRuntime/Private/SwuiRuntime.cpp` (StartupModule) |
| A2 | P0 | `disable-web-security` siempre que no está el modo CPU, que es siempre (`CPURenderSettings = false` fijo). Desactiva la same-origin policy: una página `file://` puede leer cualquier archivo local y hacer fetch a cualquier origen. | `SwuiManager.cpp` OnBeforeCommandLineProcessing |
| A3 | P0 | `OnBeforeDownload` acepta **cualquier descarga sin preguntar** y usa `SuggestedName` sin sanear (`..\..\`), así que escribe dentro del directorio del plugin. Es path traversal más drive-by download. | `RenderHandler.cpp` OnBeforeDownload |
| A4 | P0 | El bus JS→UE (`cefQuery` y `swui://bus`) no comprueba el **origen** del frame. Si la vista navega a una URL http(s) (permitido por `LoadURL`) o un iframe carga contenido remoto, ese contenido puede disparar cualquier `SwuiCommand` / navigation tag. | `RenderHandler.cpp` OnQuery / OnBeforeBrowse; `SwuiView.cpp` HandleIncomingMessage |
| A5 | P1 | No hay restricción de navegación: `OnBeforeBrowse` deja ir a cualquier URL, no hay `OnBeforePopup` (un `window.open` puede crear ventanas) ni allowlist de esquemas. | `RenderHandler.cpp` |
| A6 | P1 | `LoadURL` concatena `ProjectContentDir() + Relative` sin normalizar, así que `..` sale de Content. Además acepta `"localhost..."` sin esquema. | `SwuiView.cpp` LoadURL |
| A7 | P1 | `no_sandbox = 1` fijo. Aceptable en OSR, pero tiene que ser una decisión documentada y configurable. | `SwuiRuntime.cpp` |
| A8 | P2 | `enable-media-stream`, `autoplay-policy=no-user-gesture-required` y `HTMLImports` (feature eliminada de Chromium) se activan sin opción. | `SwuiManager.cpp` |
| A9 | P2 | Cada mensaje JS→UE se registra en log a nivel `Log` con el payload completo, varias veces. Es fuga de datos (chat, nombres) a los logs del jugador y además cuesta rendimiento. | `RenderHandler.cpp`, `SwuiView.cpp`, `SwuiNavigation.cpp` |

### B. Ciclo de vida y memoria (crashes)

| # | Grav. | Hallazgo | Ubicación |
|---|---|---|---|
| B1 | P0 | `RenderHandler` guarda `ISwuiRenderTarget* = this` (la `USwuiView`) como **puntero crudo**. `BeginDestroy` llama `CloseBrowser(true)`, que es asíncrono, y CEF puede seguir llamando `OnPaint` sobre una vista ya destruida. Es un use-after-free. | `SwuiView.cpp` Init/BeginDestroy; `RenderHandler.cpp` OnPaint |
| B2 | P0 | `BrowserClient::OwningView` es `USwuiView*` crudo. `OnBeforeBrowse` construye un `TWeakObjectPtr` **a partir de ese puntero ya posiblemente liberado**, y el weak pointer no protege en ese caso. | `RenderHandler.cpp` |
| B3 | P0 | `CefShutdown()` está comentado. Los subprocesos CEF quedan huérfanos al cerrar, y `NOTES` lo confirma: el autor mata `BluBrowserProcess` a mano. | `SwuiRuntime.cpp` ShutdownModule |
| B4 | P0 | El dispatch de comandos usa `Malloc + Memzero` para los parámetros, sin `InitializeValue` (solo para out params) y **sin `DestroyValue`**. Los `FText` y structs con constructor no trivial quedan inválidos, lo que puede causar crash, y cada `FString`/`TArray` deserializado tiene fuga. Hay que usar `FStructOnScope` o `InitializeStruct`/`DestroyStruct`. | `SwuiNavigation.cpp` ReceiveNavigationEventFromJs |
| B5 | P1 | `IsTickable()` devuelve false si no hay `View`, así que CEF deja de bombearse (`CefDoMessageLoopWork`). Tras `ShutdownRenderer` el cierre del browser nunca se completa (`OnBeforeClose` no llega). | `SwuiSubsystem.h` |
| B6 | P1 | `IsTickableWhenPaused` no está sobrescrito. Con el juego en pausa, la UI (justo el menú de pausa del tutorial) **se congela**. | `SwuiSubsystem.h` |
| B7 | P1 | `Unobserve` quita la entrada pero **no desvincula el delegate** ni destruye el `USwuiDelegateBridge`, así que los eventos siguen llegando y los bridges se acumulan. `ObserveDelegate` tampoco evita duplicados (dos llamadas producen eventos dobles). | `SwuiSubsystem.cpp` |
| B8 | P1 | `FSwuiObservedProperty::CachedProp` es un `FProperty*` crudo. Tras recompilar un Blueprint o con Live Coding puede apuntar a memoria liberada. | `SwuiSubsystem.cpp` |
| B9 | P1 | La caché CEF se crea en `%TEMP%/SWUI_BundledCEF_<pid>` en cada arranque y no se borra nunca, así que el disco crece sin límite. | `SwuiRuntime.cpp` |
| B10 | P2 | El render command ROI captura `FTextureResource*` crudo. Si la textura se recrea por un resize entre el enqueue y la ejecución, la ventana es pequeña pero existe. | `SwuiView.cpp` TickDeferredUpload |
| B11 | P2 | El `dlopen` del framework en Mac no comprueba el handle. Si `CefInitialize` falla, solo se escribe en el log y se continúa. | `SwuiLoader.cpp`, `SwuiRuntime.cpp` |

### C. Correctitud funcional

| # | Grav. | Hallazgo | Ubicación |
|---|---|---|---|
| C1 | P1 | El README promete "updating UI state only when values change", pero `FlushHudStateToJs` reenvía **todas** las propiedades observadas en cada tick, aunque no hayan cambiado. | `SwuiSubsystem.cpp` ~L841 |
| C2 | P1 | Hay lógica **de un juego concreto** dentro del plugin: `ammo`, `reload`, `compass`, `crosshair` y `window.__SWUI_HUD__.setAmmo/...`, además de un `if` vacío. | `SwuiSubsystem.cpp` FlushHudStateToJs |
| C3 | P1 | Floats `NaN`/`Inf` se serializan como `nan`/`inf`, que no es JS válido. El `ReferenceError` resultante rompe **todo** el script batch del tick. | `Swui_SerializePropertyValue` |
| C4 | P1 | `NamespacedKey` se inserta entre comillas simples sin escapar (`s.state['%s']`), y el mensaje de error de comando igual. | `SwuiSubsystem.cpp`, `SwuiNavigation.cpp` |
| C5 | P1 | La detección de ambigüedad en `TryResolveActiveBindingTarget` falla. Con 2 instancias distintas `MatchCount` vale 1, así que no la marca como ambigua y despacha a la primera. | `SwuiSubsystem.cpp` |
| C6 | P1 | **Una sola vista por GameInstance** ("view already exists — skipping"). No se pueden tener HUD y menú separados, ni UI en el mundo 3D, ni split-screen. | `SwuiSubsystem.cpp` InitRenderer |
| C7 | P1 | El widget se crea contra el `World` del GameInstance. Tras un viaje de mapa (`OpenLevel`/`ServerTravel`) el widget sale del viewport y la HUD desaparece, aunque la `View` siga viva. | `SwuiSubsystem.cpp` InitRenderer |
| C8 | P1 | El script de foco (`focusin`/`focusout`) se inyecta en `about:blank` y una sola vez. Tras navegar a otra página se pierde. Hay que inyectarlo en `OnLoadEnd`/`OnContextCreated`. | `SwuiSubsystem.cpp` |
| C9 | P1 | `SetBindingSources` solo observa objetos que **ya existen** en BeginPlay. Los actores spawneados después no se enlazan solos. Además itera `TObjectIterator<UObject>` completo por cada fuente, lo que es lento. | `SwuiSubsystem.cpp` |
| C10 | P1 | Hay dos buses de eventos inconsistentes: los delegates y `swui:tick` van a `document`, la navegación a `window`. `Swui.onEvent()` escucha en `window`, así que no recibe los eventos de delegates (solo el TS generado los recibe). | `SwuiSubsystem.cpp` vs `SwuiNavigation.cpp` vs `swui.ts` |
| C11 | P1 | Problemas de URL local:<br>- El README documenta `local://`, pero el código solo entiende `swui://`.<br>- El README dice `Content/html`, `LoadURL` resuelve contra la raíz de `Content/`, el generador escribe en `Content/UI/generated` y el dev server corre en `Content/UI`. | README, `SwuiView.cpp`, `SwuiTSGenerator.cpp`, `SwuiEditorModule.cpp` |
| C12 | P2 | En los Map solo se serializan claves string/name (las int se descartan sin aviso). En los arrays, los elementos no serializables se descartan y los índices se desplazan. | `Swui_SerializePropertyValue` |
| C13 | P2 | `ResolveNamespace` quita la primera letra si es `A` o `U`, incluso en clases Blueprint. Por ejemplo, `AudioManager_C` queda como `udiomanager_c`. | `SwuiSubsystem.cpp` |
| C14 | P2 | Los tags nativos se registran con `AddNativeGameplayTag` desde el constructor del componente, recorriendo miembros por aritmética de punteros (`&Confirm .. &PointerWheel+1`). Es frágil y depende del layout. Hay que pasar a `UE_DEFINE_GAMEPLAY_TAG`. | `SwuiNavigation.cpp` FSwuiNavTags |
| C15 | P2 | La resolución depende de `GameUserSettings` y tiene un caso especial para 1280×720. No hay soporte de DPI / `device_scale_factor`, y el texto queda borroso en 4K con escala. | `SwuiSubsystem.cpp`, `RenderHandler.cpp` |

### D. Input

| # | Grav. | Hallazgo | Ubicación |
|---|---|---|---|
| D1 | P0 | `GetKeyboardState`/`ToUnicode`/`MapVirtualKeyW` (Win32) se usan sin `#if PLATFORM_WINDOWS`, así que **no compila en Mac/Linux**. El README marca macOS como "Supported", lo cual es falso. | `SwuiInputPreprocessor.cpp` |
| D2 | P1 | `ToUnicode(..., flags=0)` altera el estado de dead keys del sistema. **Rompe los acentos** en teclados en español (´ + a = á) y no hay IME (chino, japonés, coreano). | `SwuiInputPreprocessor.cpp` |
| D3 | P1 | Los eventos de ratón se envían con `modifiers = 0` siempre, sin `EVENTFLAG_LEFT_MOUSE_BUTTON` durante el movimiento. **El arrastre no funciona**: sliders, selección de texto, drag & drop. Tampoco se envía `mouseLeave`. | `SwuiView.cpp` Forward* |
| D4 | P1 | Con la UI activa se consume **todo** el teclado y todo el ratón dentro del viewport, sin hit-test por transparencia. Teclas globales (consola `~`, Escape del juego) y clics sobre zonas vacías nunca llegan al juego. | `SwuiInputPreprocessor.cpp` |
| D5 | P1 | No hay soporte de gamepad (`HandleAnalogInputEvent`, D-pad, botones), así que la navegación por mando depende de Blueprint manual. | `SwuiInputPreprocessor.cpp` |
| D6 | P2 | `native_key_code = 0` e `is_system_key = false` siempre: los atajos con Alt y la entrada en Mac/Linux fallan. El doble clic envía `down(2)` y luego `up(1)`. | `SwuiView.cpp` |
| D7 | P2 | `ScreenToBrowserPixel` usa solo `GetGameViewport()` y no cubre PIE multi-ventana ni split-screen. | `SwuiView.cpp` |

### E. Empaquetado, build y distribución

| # | Grav. | Hallazgo | Ubicación |
|---|---|---|---|
| E1 | P0 | **Los binarios CEF no están** en el repo (solo `.gitkeep`) y tampoco hay script para obtenerlos. El `builder/` archivado es de CEF 2623 (año 2016) y VS2015. El repo no compila tal cual. | `ThirdParty/cef/`, `builder/` |
| E2 | P0 | **El subproceso** (`SwuiBrowserProcess.exe` / `swui_ue_process`) no está, ni sus fuentes. Sin su `CefMessageRouterRendererSide`, `window.cefQuery` no existe. | `SwuiRuntime.cpp` |
| E3 | P0 | En un juego empaquetado `Content/` va dentro del `.pak` y `file:///ProjectContentDir/...` no existe en disco. Faltan `DirectoriesToAlwaysStageAsNonUFS` o un `CefResourceHandler`/esquema propio que lea del pak. **Hoy la UI no carga en Shipping.** | `SwuiView.cpp` LoadURL |
| E4 | P1 | `FilterPlugin.ini` está vacío, así que el plugin empaquetado (Fab/Marketplace o `BuildPlugin`) no incluye `ThirdParty/cef`. | `Config/FilterPlugin.ini` |
| E5 | P1 | `Build.cs` de Linux no hace `RuntimeDependencies` (no se copian los `.so` ni `locales`). En Mac se copian `shipping` y `lib` completos. `ffmpegsumo.dll` es una DLL que CEF dejó de distribuir hace años. | `SwuiRuntime.Build.cs` |
| E6 | P1 | El `.uplugin` dice "Unreal Engine 4", `VersionName 5.0.0`, `EngineVersion` ausente y `EnabledByDefault: true`. Depende de `GameplayTagsEditor` (un plugin de editor) desde módulos runtime. | `SimpleWebUI.uplugin` |
| E7 | P2 | `RenderCore` está duplicado en las dependencias. El módulo `UncookedOnly` depende de `UnrealEd`. La detección de AngelScript se hace con `Directory.Exists`. | `*.Build.cs` |
| E8 | P2 | Hay archivos personales en el repo: rutas `F:\GameDev\...` en `NOTES` y `build-loop.ps1`, y carpetas `.kilo/`, `BRIEF/`, `.github/copilot-instructions.md`. | raíz |

### F. Editor y herramientas

| # | Grav. | Hallazgo | Ubicación |
|---|---|---|---|
| F1 | P1 | El dev server mata **cualquier** proceso que use el puerto 5173 (puede ser de otra app) mediante `pwsh.exe`, que no viene instalado por defecto en Windows. Nunca se detiene al cerrar el editor y es solo Windows sin guard. | `SwuiEditorModule.cpp` MaybeLaunchDevServer |
| F2 | P1 | La auto-regeneración de bindings TS está **desactivada porque crasheaba** el editor, así que los tipos quedan desincronizados sin aviso. | `SwuiEditorModule.cpp` (comentario) |
| F3 | P2 | `.uassetignore` no es un mecanismo real de UE. `InterfaceName` se usa en rutas de archivo sin sanear. | `SwuiEditorModule.cpp`, `SwuiTSGenerator.cpp` |
| F4 | P2 | No hay validación de los TS generados (ni `tsc --noEmit` en CI). | — |

### G. Cliente JS (`SwuiClientLib`)

| # | Grav. | Hallazgo |
|---|---|---|
| G1 | P1 | En el puente por iframe (`swui://bus`), varios envíos seguidos sobrescriben `iframe.src` y **se pierden mensajes**. |
| G2 | P1 | `_whenReady` crea un `setInterval` por cada `on()` que nunca se limpia si el runtime no aparece. |
| G3 | P2 | No hay request/response: `cefQuery` responde `"ok"` antes de procesar el mensaje, así que JS no puede saber si el comando falló salvo con `swui:navigation-error`. |
| G4 | P2 | El paquete no está publicado, no tiene tests, el export `./src/*` filtra internos y no hay versión de protocolo entre cliente y runtime. |

### H. Calidad y proceso

- No hay **ningún test** (ni Automation Tests de UE ni tests del cliente).
- No hay CI.
- No se declara qué versión de UE se soporta (el autor usa un fork de UE con AngelScript).
- Hay 55 usos de `LogTemp` en lugar de la categoría `LogSwuiRuntime`.
- `IMPLEMENTATION_STATUS.md` tiene todas las historias en `[ ]` y contradice al README.
- Hay logs a nivel `Log` en caminos calientes: cada clic, cada wheel, cada mensaje de consola CEF.

---

## Parte 2 — Plan de acción

Cada fase termina con un **criterio de salida** verificable. No se pasa a la siguiente sin cumplirlo.

### Fase 0 — Base reproducible (1–2 semanas)

Objetivo: cualquiera clona, ejecuta un script y compila. Sin esto no se puede verificar nada más.

1. Fijar **una** versión de UE (recomendado: la última 5.x estable) y **una** versión CEF compatible. Hay que documentar la versión exacta (`cef_binary_X.Y.Z+chromium-N`).
2. Crear `Scripts/fetch-cef.ps1` / `.sh`, que descargue el binario oficial de CEF (Spotify CDN), verifique su SHA256 y lo coloque en `ThirdParty/cef/<Plat>/`. Esto cierra E1.
3. Traer al repo las **fuentes del subproceso** (`Source/SwuiBrowserProcess/` como programa UBT o proyecto CMake), con `CefMessageRouterRendererSide`. Esto cierra E2.
4. Limpiar el repo (E8): borrar `NOTES`, `build-loop.ps1`, `.kilo/`, `BRIEF/` y `builder/` archivado. Corregir el `.uplugin` (E6) y los `Build.cs` (E5, E7).
5. Añadir guards de plataforma a todo el código Win32 (D1) para que compile, aunque Mac/Linux queden "no soportados" por ahora.
6. Montar CI (GitHub Actions con runner self-hosted con UE): `BuildPlugin` en Win64, más `pnpm build` y `tsc --noEmit` del cliente.
7. Crear un **proyecto de ejemplo mínimo** (`Samples/SwuiDemo/`) con HUD y menú. Sirve como banco de pruebas permanente.

**Criterio de salida:** CI en verde compila el plugin, el subproceso y el cliente desde cero, y el sample abre una HUD en PIE.

### Fase 1 — Seguridad y crashes P0 (2–3 semanas)

1. **A1**: `remote_debugging_port` vale 0 en Shipping y se configura por CVar/ini en Development.
2. **A2**: eliminar `disable-web-security`. Servir la UI desde un **esquema propio** (`swui://app/...`) con `CefSchemeHandlerFactory`, así el origen es estable y no hace falta `file://`. Esto cierra E3 a la vez (ver Fase 2).
3. **A3**: `OnBeforeDownload` rechaza por defecto y, si se habilita, sanea el nombre y usa un directorio en `Saved/`.
4. **A4/A5**: allowlist de orígenes en `OnQuery`/`OnBeforeBrowse` (solo `swui://app` y, en Development, `http://localhost:<puerto>`). Añadir `OnBeforePopup` que devuelva `true`, y cancelar navegaciones fuera de la allowlist.
5. **A6**: normalizar la ruta y rechazar cualquier ruta que salga del root de UI.
6. **B1/B2**: sustituir los punteros crudos por un `TSharedRef<FSwuiViewProxy>` thread-safe con flag `bAlive`. `BeginDestroy` marca el proxy como muerto y CEF consulta el flag antes de tocar la vista. Usar `IsReadyForFinishDestroy` para esperar a `OnBeforeClose`.
7. **B3**: `CefShutdown()` en `ShutdownModule`, después de cerrar todos los browsers, con el bombeo final del loop. Comprobarlo: después de cerrar el editor no queda ningún subproceso vivo.
8. **B4**: reescribir el dispatch con `FStructOnScope(Function)` o `InitializeStruct`/`DestroyStruct`, y validar el resultado de `JsonObjectToUStruct` (rechazar si falla).
9. **B5/B6**: CEF se bombea desde un `FTSTicker` del módulo, independiente de la vista y de la pausa.

**Criterio de salida:**

- Test de seguridad automatizado: una página hostil en el sample no puede leer archivos, invocar comandos, abrir popups ni descargar.
- Un soak test de 1 h con crear y destruir vistas en bucle, más PIE start/stop ×100, termina sin crash ni fuga de procesos o memoria.

### Fase 2 — Funciona en un juego empaquetado (2 semanas)

1. Esquema `swui://app/` que lea los archivos de la UI vía `IPlatformFile`, para que funcione con pak, IoStore y loose. Unificar **una** carpeta de UI (`Content/SwuiUI/` o configurable en `USwuiSettings`) y alinear docs, generador y dev server (C11).
2. Staging: `+DirectoriesToAlwaysStageAsUFS` (o NonUFS) automático desde la configuración del plugin y `FilterPlugin.ini` completo (E4).
3. Copia correcta de dependencias en Win, Linux y Mac en `Build.cs`, y verificar el `.app` bundle en Mac.
4. Política de caché CEF (B9): un directorio fijo por proyecto en `Saved/SwuiCache`, o caché en memoria.
5. Probar en CI un empaquetado Shipping del sample y un smoke test (arrancar el juego, esperar el primer paint, salir con código 0).

**Criterio de salida:** el build Shipping del sample muestra la HUD y el menú en una máquina limpia, sin UE instalado.

### Fase 3 — Correctitud del puente (2–3 semanas)

1. Quitar toda la lógica específica del juego (C2) y enviar **solo los cambios** (C1), con un diff por clave y un `swui:state` batch.
2. Reemplazar la generación de JS por concatenación de strings con **un solo serializador JSON** (`TJsonWriter`) y un único punto de entrada en JS: `window.__SWUI__.__receive(json)`. Así se cierran C3, C4 y C12 de raíz, porque no se ejecuta código construido con strings.
3. Unificar el bus de eventos (C10): un solo `EventTarget` propio en el cliente.
4. Versionar el protocolo (`{v:1, type, ...}`) y añadir request/response real con id (G3).
5. Arreglar el ciclo de vida del Observe (B7, B8, C9): handles de suscripción, `Unobserve` que desvincula, auto-observe de actores spawneados vía `OnActorSpawned`, e invalidar la caché de `FProperty` en recompilaciones.
6. Arreglar la ambigüedad de la resolución de targets (C5) y añadir resolución explícita (target por id/namespace).
7. Inyectar el script de foco en `OnLoadEnd` (C8). Mejor aún: ponerlo en el cliente JS y no inyectarlo desde C++.
8. Cliente: cola de mensajes para el iframe bridge, o eliminar ese bridge (G1). Un solo `whenReady` compartido (G2).

**Criterio de salida:** suite de Automation Tests de UE (ver "Búsqueda de grietas") en verde, cubriendo serialización de todos los tipos soportados, Observe/Unobserve y dispatch de comandos.

### Fase 4 — Input de calidad (2–3 semanas)

1. Un `SWidget` propio (`SSwuiSurface`) que reciba foco y gestione `OnKeyChar`, `OnKeyDown`, `OnMouse*` y la IME (`ITextInputMethodSystem`). Sustituye el `IInputProcessor` con `ToUnicode` (D1, D2).
2. Modificadores y botones pulsados en todos los eventos de ratón, `mouseLeave`, y clics múltiples correctos (D3, D6).
3. Hit-test por alfa: leer el alfa del último frame en la posición del cursor y no consumir el evento si es transparente (D4). Hacer configurable qué teclas siempre pasan al juego.
4. Gamepad: mapear el analógico y los botones a navegación espacial (D5).
5. DPI: `GetScreenInfo` con `device_scale_factor` derivado del DPI de Slate (C15).

**Criterio de salida:** página de pruebas de input en el sample (inputs, textarea, slider arrastrable, contenteditable, selección) verificada con teclado ES, US y JP (IME), ratón y mando.

### Fase 5 — Arquitectura multi-vista y rendimiento (3–4 semanas)

1. Desacoplar `USwuiView` del Subsystem: N vistas, cada una con su widget y su capa (C6). Que sobrevivan al viaje de mapa (C7) o se recreen de forma declarativa.
2. Vista en el mundo 3D (`UWidgetComponent` o material) para pantallas in-game.
3. Ruta GPU: `shared_texture_enabled` y `OnAcceleratedPaint` en D3D11/D3D12 (el esqueleto ya existe en `RenderHandler`). Mantener CPU como fallback.
4. Presupuesto de rendimiento con `stat` y Insights (`TRACE_CPUPROFILER_EVENT_SCOPE`) en Tick, OnPaint y Upload. Bajar los logs del camino caliente a `Verbose` (A9).
5. Benchmarks en CI: ms por frame en el sample con una HUD de referencia, alertando si empeora más de un 10 %.

**Criterio de salida:** el sample corre con HUD, menú y una pantalla 3D a la vez, cumpliendo el presupuesto (orientativo: <1 ms de game thread y <2 ms de upload a 1080p en la HUD de referencia).

### Fase 6 — Experiencia de desarrollo y lanzamiento (continuo)

1. Generación de bindings TS fiable (F2): ejecutarla en `OnBlueprintCompiled` con debounce, protegida contra GC (`FGCScopeGuard`, sin iterar objetos en medio de una compilación), y añadir un commandlet para CI.
2. Dev server opcional y seguro (F1): puerto configurable, sin matar procesos ajenos, detenido al cerrar y multiplataforma.
3. Publicar `@swui/client` en npm con semver ligado a la versión del protocolo.
4. Documentación coherente con el código: guía de inicio, matriz de plataformas **verificada por CI** y guía de seguridad (qué puede y qué no puede hacer el JS).
5. Mac y Linux pasan a "soportado" solo cuando el CI empaquete y haga smoke test en ellas.
6. Política de releases: changelog, tags, binarios precompilados por versión de UE y actualización periódica de CEF (Chromium recibe parches de seguridad cada pocas semanas).

---

## Búsqueda intensiva de grietas (proceso permanente)

Esto no es una fase: arranca en la Fase 0 y no se detiene.

### 1. Tests automatizados (UE Automation + Vitest)

- **Serialización:** tabla de casos por cada tipo (`float` NaN/Inf/-0, `int64` > 2^53, strings con `'`, `"`, `\u2028`, `</script>`, emoji, structs anidados, maps con clave int, arrays con elementos no soportados). Se comprueba que el JSON resultante parsea en V8 y coincide con lo esperado.
- **Dispatch de comandos:** cada tipo de parámetro (FText, FString, TArray, struct, enum, object), payload inválido, payload gigante, tag inexistente, target ambiguo y target destruido.
- **Ciclo de vida:** crear/destruir vista ×1000, PIE start/stop ×100, viaje de mapa ×50, Observe/Unobserve sobre actores destruidos y recompilación de un BP observado.
- **Cliente:** Vitest más happy-dom para la cola de mensajes, suscripciones y el protocolo.

### 2. Fuzzing del bus JS → UE

- Harness que llame `USwuiView::HandleIncomingMessage` y el dispatch con entradas generadas por libFuzzer (o un fuzzer en JS dentro del sample que envíe millones de mensajes aleatorios por `cefQuery`).
- Corpus inicial a partir de los mensajes reales del sample. Objetivo: cero crashes y cero asserts en 24 h de fuzzing.

### 3. Sanitizers y herramientas de memoria

- Build del sample con **ASan** (UE lo soporta en Win64/Linux con `-EnableASan`) en el job nocturno, ejecutando la suite de ciclo de vida.
- Revisar con **Unreal Insights** (Memory) que no haya crecimiento tras el soak test.
- En Windows, Application Verifier sobre el subproceso CEF.

### 4. Soak tests y caos

- Job nocturno: el sample corre 8 h con la HUD actualizándose a 60 Hz, abriendo y cerrando menús cada pocos segundos, mientras se mide RAM, handles y número de procesos.
- Caos: matar el subproceso renderer a mitad de partida (`OnRenderProcessTerminated` tiene que recuperar la vista), red desconectada y disco lleno en la caché.

### 5. Revisión de seguridad

- Página "hostil" en el sample que intenta: leer `file:///C:/Windows/win.ini`, hacer fetch a un origen externo, `window.open`, descargar, navegar fuera, invocar todos los comandos y conectarse a 7777. Es un test de regresión que tiene que fallar en todos los intentos.
- Revisión manual en cada actualización de CEF y cada vez que se toque `RenderHandler.cpp`/`SwuiManager.cpp`.
- Seguir los boletines de seguridad de Chromium/CEF y tener una política de actualización con fecha.

### 6. Matriz de compatibilidad

- CI con matriz: {UE 5.x actual, 5.x−1} × {Win64, Linux, Mac} × {Editor, Development, Shipping}.
- Una celda solo cuenta como "soportada" en el README si está en verde.

### 7. Dogfooding

- Un juego pequeño real construido solo con SWUI (HUD, inventario, ajustes, chat), mantenido junto al plugin. Cada bug que salga ahí se convierte en un test antes de arreglarlo.

---

## Orden recomendado de trabajo

```text
Fase 0 ──► Fase 1 ──► Fase 2 ──► Fase 3 ──► Fase 4 ──► Fase 5 ──► Fase 6
  │           └─ (bloqueante: sin esto no hay "producción")
  └─ Búsqueda de grietas arranca aquí y corre en paralelo a todo
```

Estimación total hasta un 1.0 utilizable en Win64: **3–4 meses** para una persona a tiempo completo (Fases 0–4). Multi-vista, GPU y multiplataforma (Fases 5–6) suman otros 2–3 meses.

### Decisiones que hay que tomar antes de empezar

1. **Versión de UE objetivo**, y si se mantiene la dependencia opcional de AngelScript.
2. **¿Seguir con CEF propio o usar el `WebBrowser`/CEF que trae UE?** El de UE reduce mucho el mantenimiento de binarios, a cambio de menos control sobre la versión y los flags. Conviene evaluarlo en la Fase 0 antes de invertir en `fetch-cef`.
3. **Relación con upstream:** fork independiente o PRs a `ZenaMel/SWUI`. Muchos arreglos de la Fase 1 serían valiosos upstream.
4. **Licencia:** confirmar que `LICENSE` y `CREDITS.md` (linaje BLUI, MIT) permiten el uso previsto, y añadir la licencia de CEF (BSD) a la distribución.
