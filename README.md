# 🎬 CineCatálogo v10

App de un solo archivo para explorar películas con **TMDb**, filtrar, guardar una
**watchlist local** y sincronizar tus **vistas y notas de Trakt**. Sin backend, sin
proxies y sin subir tus claves a ningún servidor.

- **Archivo:** `index.html` (octubre 2026). Se abre con doble clic; no requiere instalar nada.
- **Referencia:** `original-v9.html` es la versión que salió del chat de DeepSeek, sin tocar.
- **Fragmento suelto:** el `deepseek_javascript_*.js` era un trozo de esta app que usaba
  `corsproxy.io`; se descartó (ver *Seguridad*).

---

## Cómo empezar

1. Abre `index.html` en Chrome, Edge o Firefox.
2. Pega tu **API Key de TMDb** (obligatoria) y guarda.
   - v3: clave de 32 caracteres → viaja como `api_key` en la URL de la API.
   - v4: *Read Access Token* (`eyJ…`) → viaja en la cabecera `Authorization` (mejor).
3. Opcional: añade OMDb, MDBList y Trakt desde el mismo panel.

| Servicio | Para qué | Dónde se obtiene |
|---|---|---|
| **TMDb** | Catálogo, búsqueda, detalles, reparto, plataformas | https://www.themoviedb.org/settings/api |
| **OMDb** | Nota IMDb en las tarjetas y puntuaciones extra | https://www.omdbapi.com/apikey.aspx |
| **MDBList** | RT, Metacritic, Letterboxd, Trakt, MAL… | https://mdblist.com/preferences/ |
| **Trakt** | Vistas, notas personales, «solo no vistas» | https://trakt.tv/oauth/applications |

### Configurar Trakt

1. Crea una aplicación en Trakt con *Redirect URI* `urn:ietf:wg:oauth:2.0:oob`.
2. Copia **solo el Client ID** (nunca el Client Secret) en Ajustes.
3. Escribe tu usuario de Trakt. El perfil y el historial de películas deben ser **públicos**.
4. La app pagina automáticamente (`limit=100` + `X-Pagination-Page-Count`) hasta traer todo.

---

## Qué se corrigió respecto a la v9

### Fallos que impedían que funcionara

1. **Trakt nunca podía cargar.** La v9 llamaba a `api.trakt.tv` sin la cabecera
   `trakt-api-key`, así que la API respondía 401/403 siempre. Ahora se envía el Client ID
   y hay un campo para configurarlo (antes ni existía).
2. **Proxy innecesario y peligroso.** El fragmento `.js` metía las peticiones por
   `corsproxy.io`, lo que habría enviado tu Client ID a un tercero. Se eliminó: Trakt,
   TMDb, OMDb y MDBList permiten CORS directo (verificado en navegador real).
3. **Las notas IMDb no se cargaban nunca.** La v9 esperaba `imdb_id` en los listados de
   TMDb, que **no lo devuelven**. Ahora la nota de IMDb es la **puntuación principal** de
   cada tarjeta y se resuelve en cascada: título original + año → título traducido + año →
   `imdb_id` real pedido a TMDb (`/external_ids`) → título sin año. Si no se encuentra,
   la tarjeta muestra la nota de TMDb en lugar de dejar el hueco vacío.
4. **Búsqueda con debounce roto.** Se creaba un temporizador nuevo en cada tecla, así que
   el debounce no cancelaba nada.
5. **Spinner eterno.** El estado se pintaba con spinner y nunca se limpiaba al terminar.
6. **`setImdbStatus(msg, true)`** recibía un parámetro que la función ignoraba (los errores
   no se marcaban como error).
7. **Contadores mal en singular** («1 filtradas», «1 vistas», «1 películas»).

### Seguridad

- **CSP** (`meta Content-Security-Policy`): solo se permite conectarse a
  `api.themoviedb.org`, `omdbapi.com`, `api.mdblist.com` y `api.trakt.tv`; las imágenes solo
  desde `image.tmdb.org`. Un script inyectado no puede exfiltrar datos a otro dominio.
- **Sin claves en la URL ni en enlaces compartidos.** Los filtros van en la URL; las claves no.
- **Token v4 de TMDb en cabecera** (`Authorization: Bearer`), no en cada *query string*.
- **Modo «no guardar»**: guarda las claves en `sessionStorage` (se borran al cerrar la pestaña),
  útil en equipos compartidos.
- **Borrar todo** elimina claves, caché, watchlist y datos de Trakt de un clic.
- **`referrerpolicy="no-referrer"`** en imágenes y `rel="noopener noreferrer"` en enlaces.
- **Escapado de todo el HTML dinámico** y validación de rutas de imágenes/backdrops
  (`safePath`), incluida la URL usada en `background-image`.
- **Nunca se pide el Client Secret de Trakt** ni se guarda en disco.
- Botones de mostrar/ocultar claves y campos `password` por defecto.

> Las claves siguen viviendo en el navegador: cualquier extensión con permisos sobre esta
> página podría leerlas. Para uso serio, la alternativa es un proxy propio (Cloudflare
> Worker) que guarde las claves en el servidor; esta app no lo necesita para funcionar.

### Velocidad y caché

- **Caché HTTP en IndexedDB** (base `cinecatalogo`, almacén `http`), con memoria como primer
  nivel y `sessionStorage` como red de seguridad si el navegador bloquea IndexedDB (algunos
  contextos `file://`). Tolera muchas más entradas que `localStorage` y **sobrevive a recargas
  y reinicios**: al reabrir la app, catálogo, detalles, similares y filmografías salen de caché.
  TTL por tipo de dato: géneros 7 días, listados 10 min, detalles 24 h, puntuaciones 12 h,
  Trakt 30 min, IMDb 6 h (y **30 min** para notas con menos de 500 votos, que cambian rápido).
- **Purga automática**: al pasar de 400 entradas se eliminan las expiradas y la mitad más
  antigua. La caché antigua de `sessionStorage` se migra sola a IndexedDB.
- **Imágenes adaptativas**: `srcset` con `w185`, `w342` y `w500` más `sizes`, de modo que un
  móvil descarga la versión pequeña y un monitor grande la nítida.
- **Prioridad de carga**: las 4 primeras tarjetas usan `loading="eager"` +
  `fetchpriority="high"`; el resto, `loading="lazy"`. `preconnect` a `image.tmdb.org` y
  `api.themoviedb.org` para ahorrar el handshake inicial.
- **`content-visibility: auto`** en las tarjetas: las que están fuera de pantalla no se pintan
  hasta que hacen falta (`contain-intrinsic-size` evita saltos de scroll).
- **Deduplicación de peticiones en vuelo**: dos vistas que piden lo mismo comparten la respuesta.
- **Reintento con espera** en 429/5xx respetando `Retry-After`, y timeout de 15 s.
- **Trakt sin `extended=full`**: la respuesta pesa mucho menos al pedir solo los IDs.
- **Trakt en paralelo con el catálogo**: la primera pantalla no espera a Trakt; cuando llega,
  se repinta con los badges.
- **Paginación real de Trakt** en lugar de confiar en `limit=1000`.
- **Trakt al día sin pulsar nada**: al volver a esta pestaña se refresca solo si los datos
  tienen más de 2 minutos (silencioso y como máximo una vez por minuto). Así, cuando ves una
  película en Nuvio y Trakt la registra, el catálogo la refleja al volver.
- **Cancelación lógica por `requestId`**: al cambiar filtros rápido, las respuestas viejas
  se descartan.
- **Skeletons** solo en la primera carga; en cambios de filtro se atenúa la rejilla actual.
- `decoding="async"` + dimensiones fijas para evitar saltos de layout.

### Interfaz

- Tema claro/oscuro persistente (sin destello al cargar) y respeto a `prefers-reduced-motion`.
- Barra de filtros **sticky** con desenfoque.
- Barra de estado de Trakt con «Actualizar» y hora de la última sincronización.
- Tarjetas con **IMDb como nota principal** (amarillo) y TMDb como dato secundario; si OMDb
  aún no ha respondido o no encuentra la película, se muestra TMDb y se sustituye solo.
  Solo se pinta como IMDb la nota **verificada por el ID de TMDb**: si OMDb solo coincide por
  título (puede ser otra película con el mismo nombre), la tarjeta mantiene TMDb y se reintenta
  por ID. El tooltip del badge indica los votos de IMDb.
- **Una sola nota por película**: al abrir una ficha se consulta OMDb por ID aunque hubiera una
  nota guardada, se actualiza la caché y **la tarjeta cambia al momento**. Así la tarjeta y la
  ficha no pueden mostrar 8,7 y 7,1 a la vez. Las notas con pocos votos se refrescan cada 30
  minutos porque son las que más se mueven (estrenos).
- «✓ Vista», «★ tu nota» y corazón de watchlist en la tarjeta.
- **Orden por nota IMDb** (`Nota IMDb mejor → peor` / `peor → mejor`): ranking local sobre las
  **60 películas más populares** que cumplen los filtros (3 páginas de TMDb), con paginación de
  20 en 20. Si no hay clave de OMDb, las opciones aparecen deshabilitadas.
- **Solo notas fiables**: solo entran en el ranking las notas que OMDb devuelve **por el
  `imdb_id` de TMDb** y con **al menos 500 votos** en IMDb. Las demás (estrenos recién
  publicados, títulos que solo coinciden por nombre o con nota inflada por pocos votos) no
  aparecen en la lista y se cuentan como «sin nota fiable». Es lo que evita que un estreno de
  2026 con 120 votos se cuele como «mejor película». Si **ninguna** candidata supera ese
  criterio, la app no disfraza el orden de TMDb: deja la rejilla con un aviso y el botón para
  añadir candidatas, en lugar de mostrar una lista que parezca un ranking.
- **«Solo no vistas»**: oculta las películas que ya figuran en tu historial de Trakt. Se aplica
  al pintar, sin volver a pedir datos, así que se puede activar y desactivar al instante; el
  contador indica cuántas se han ocultado.
- **Enlace directo a una ficha**: `?peli=ID` (por ejemplo `?peli=27205`) abre esa película al
  cargar la página. Al abrir o cerrar el detalle la URL se actualiza con `replaceState`, sin
  ensuciar el historial y sin perder los filtros activos; atrás/adelante abre o cierra la ficha.
- **Nota IMDb mínima** (6,0 / 7,0 / 7,5 / 8,0 / 8,5): aparece solo en modo ranking y filtra
  **en el cliente**, sin hacer ni una petición. El contador y la paginación se recalculan.
- **Ampliar el ranking (botón «＋ Añadir 60 candidatas»)**: TMDb no sabe ordenar por IMDb, así
  que el ranking se calcula sobre las **60 películas más populares** que cumplen los filtros.
  Si la que buscas es menos popular, no está en ese bloque. Este botón pide las **60
  siguientes** por popularidad (páginas 4-6 de TMDb) y sus notas a OMDb, y luego otras 60
  (hasta 180). No cambia los filtros ni el orden: solo amplía la lista de candidatas para que
  entren películas menos populares. Antes de gastar cuota avisa con un `confirm()` indicando
  cuántas consultas hará; al llegar a 180 queda como «Máximo alcanzado».
  El bloque se guarda en `sessionStorage` (`cine.rank.v1`), así que **cambiar de página, de
  dirección, de nota mínima o activar «Solo no vistas» no vuelve a pedir nada**, ni siquiera
  tras recargar la pestaña.
- **Vista de watchlist propia**: funciona incluso sin API key, con búsqueda y orden local.
- Modal de detalle con sinopsis, puntuaciones (IMDb/RT/MC/Letterboxd/Trakt/MAL/TMDb),
  tráiler, reparto, plataformas por región y enlaces a TMDb/IMDb/Trakt. MDBList a veces manda
  las notas en escala 0-100 (TMDb 69, Trakt 70): se convierten a la escala de cada fuente
  (6,9 y 7,0). IMDb y TMDb directos tienen prioridad sobre los agregados.
- **Películas similares** al final del detalle: 8 recomendaciones de TMDb con póster, año y
  nota; un clic abre su ficha sin cerrar el modal.
- **Reparto y dirección clicables**: al pulsar una persona se abre su ficha con foto,
  departamento y filmografía (24 títulos por popularidad, también clicables), con botón
  «← Volver a la película» que restaura el detalle anterior.
- **Región configurable** para «dónde verla» (23 países) y aviso de otras regiones con datos.
- Toasts, estados vacíos con acción, mensajes de error específicos por código HTTP.
- Teclado: `/` busca, `Esc` cierra, `←`/`→` cambia de página, `Tab` atrapado en modales.
- Accesibilidad: **Lighthouse 100** en accesibilidad y buenas prácticas, `aria-live` en
  estados, `aria-pressed` en el corazón, foco gestionado y restaurado.
- Exportar/importar watchlist en JSON para moverla entre navegadores.

### Datos y compatibilidad

- Migración automática de la v9: `tmdb_api_key`, `omdb_api_key`, `mdblist_api_key`,
  `trakt_username` y la watchlist antigua (array de IDs) se convierten al formato nuevo.
- Los datos de watchlist incompletos se completan solos la primera vez que abres la vista
  (título, año, póster y nota), con 5 peticiones en paralelo.
- La caché de OMDb se recorta a 1.200 entradas para no llenar `localStorage`.

---

## Límites conocidos

- **Solo lectura de Trakt**: muestra tus vistas y notas, no las modifica (eso requiere OAuth
  con tokens y no se hace desde un archivo local).
- **El ranking IMDb no cubre todo el catálogo**: TMDb solo ordena por sus propios campos (nota
  media, popularidad, fecha…). El orden por IMDb se calcula en local sobre un bloque de
  **60 candidatos ampliable a 120 y 180** (3, 6 y 9 páginas de 20) que cumplen los filtros y
  tienen al menos **100 votos** en TMDb. Para entrar en el ranking, la nota debe venir del
  `imdb_id` de TMDb y tener **≥ 500 votos en IMDb**; el resto se descarta y se cuenta aparte.
  No es un ranking global de IMDb, sino de esas 60–180 películas.
- **Coste del ranking en OMDb**: se pide primero el `imdb_id` a TMDb y luego **una sola consulta
  a OMDb por película** (las 20 de la página visible ya suelen estar en caché). La primera vez
  son ~60 consultas; cada ampliación añade hasta 60 más; después, 0 mientras la caché siga
  vigente. Con 1.000 consultas diarias gratuitas da para bastantes ampliaciones.
- **Filtro de nota mínima**: se aplica sobre las candidatas ya reunidas, no sobre todo IMDb. Si
  con 8,0 salen pocas, la app sugiere bajar la nota o ampliar el bloque.
- TMDb no devuelve más de **500 páginas** (10.000 resultados) por búsqueda.
- OMDb gratis: **1.000 consultas/día**. Si se agota, la app lo detecta y deja de insistir.
- La búsqueda de TMDb no acepta `with_genres`, así que género, año y votos se filtran en el
  cliente para los resultados de búsqueda (los contadores lo indican).
- Las imágenes vienen de `image.tmdb.org`; si TMDb cambia el CDN hay que ajustar `API.IMG`.
- El botón de exportar descarga un `.json`; si el navegador bloquea descargas locales,
  prueba a servir la carpeta con `python -m http.server`.

---

## Estructura

```
cinecatalogo/
├── index.html          # la app completa (HTML + CSS + JS, 1 archivo)
├── original-v9.html    # versión de referencia, sin modificar
└── README.md
```

## Verificación hecha

- `node --check` sobre los dos bloques de script: sin errores de sintaxis.
- Pruebas funcionales en navegador con la API simulada: guardado de claves, catálogo,
  búsqueda, exclusiones, «ocultar vistas», paginación, detalle, puntuaciones, tráiler,
  plataformas, watchlist, migración, `popstate`, modo sesión y errores 401.
- «Borrar todo» probado con confirmación cancelada y aceptada: deja el almacenamiento vacío,
  reinicia tema, filtros y Trakt, y permite volver a configurar la app.
- Puntuación IMDb probada en los tres caminos: título encontrado, título no encontrado con
  rescate por `imdb_id` de TMDb, y película sin nota (la tarjeta conserva la de TMDb).
- Ranking IMDb probado con 45 películas simuladas: orden descendente y ascendente correctos,
  «Página 2» y el cambio de dirección **sin peticiones nuevas** (0 a OMDb y a TMDb), 45
  entradas en `cine.imdb.v3` y 1 consulta a OMDb por película.
- Ranking ampliado con 120 y 180 candidatas: `imdbMin=8` dejó 20 tarjetas **con 0 peticiones**;
  «Añadir 60 candidatas» hizo exactamente 3 `discover` + 60 consultas OMDb por bloque; el botón
  pasa por «Ampliando…» y termina en «Máximo alcanzado (180)» deshabilitado, con un clic
  posterior sin efecto.
- **Persistencia probada en frío**: pool de 60 guardado, recarga completa sin la API simulada
  (clave inválida) y el ranking se pintó con **20 tarjetas y 20 notas IMDb, 0 peticiones**.
- La nota mínima y el bloque de ampliación se ocultan al entrar en la watchlist y reaparecen al
  volver al catálogo; cambiar la nota solo repinta (0 peticiones) y la URL conserva `imdbMin`.
- Similares y personas: 8 recomendaciones en el detalle, 3 elementos clicables de reparto y
  dirección, ficha con 6 películas y vuelta correcta al detalle original.
- IndexedDB verificada en `file://` con 252 entradas reales y en el arranque en frío con caché
  previa; el fallback a `sessionStorage` existe para navegadores que la bloqueen.
- `srcset`, `sizes`, `loading` y `fetchpriority` comprobados en las tarjetas (4 primeras
  `eager` + `high`, resto `lazy`). 0 errores de JavaScript en la consola.
- Sin clave de OMDb: las dos opciones de IMDb quedan deshabilitadas, el selector vuelve a
  «Mejor puntuación (TMDb)» y no se hace ninguna consulta a OMDb. La nota mínima y el botón
  de ampliar permanecen ocultos.
- Lighthouse: accesibilidad **1.0**, buenas prácticas **1.0**, SEO 0.8 (solo por
  `robots.txt`, irrelevante en `file://`).
- **Caso 2026 probado** con 60 estrenos simulados: el estreno con nota alta pero sin `imdb_id`
  y solo 120 votos («Primetime») queda fuera; la primera tarjeta es la de mayor nota verificada
  (8,9), el estado avisa de «1 sin nota fiable» y las 20 primeras tarjetas muestran nota IMDb.
- **Misma nota en tarjeta y ficha**: con una nota vieja de 8,7 en caché y 7,1 real en OMDb, al
  abrir la ficha la cabecera mostró 7,1, **la tarjeta cambió a 7,1** y la caché quedó con 250.000
  votos. Tras recargar, 0 peticiones y **0 tarjetas sin badge**.
- **Coincidencia por título descartada**: una entrada simulada de 8,7 «por título» no se pintó
  como IMDb; la tarjeta mostró TMDb y se volvió a consultar por ID hasta obtener 8,8 verificado.
- **Notas 0-100 de MDBList**: TMDb 69 y Trakt 70 se mostraron como 6,9 y 7,0; el 87 de IMDb de
  MDBList no pisó el 7,1 de OMDb.
- **Sin notas fiables**: con todas las candidatas por debajo de 500 votos, la rejilla quedó
  vacía (0 tarjetas) con el aviso y el botón «＋ Añadir 60 candidatas» disponible; no se mostró
  el orden de TMDb.
- **«Solo no vistas» dentro del ranking**: activarlo ocultó 2 películas vistas y repintó con
  **0 peticiones**; el estado pasó a «57 películas … · 2 ocultas por vistas».
- **Auto-sincronización de Trakt**: con la caché de 5 minutos, al volver a la pestaña se
  hicieron exactamente 2 peticiones (vistas + notas), **0 a TMDb y 0 a OMDb**, y el segundo
  disparo seguido no repitió nada (límite de 1/minuto).
- **Enlace directo**: `?peli=900010` abrió la ficha al cargar; abrir/cerrar el detalle actualiza
  la URL (`&peli=…`) sin recargar y sin perder `year`, `sort` ni `peli` en el historial. Tras
  recargar con `?peli`, la ficha vuelve a abrirse con **0 peticiones de red** (todo en caché).
