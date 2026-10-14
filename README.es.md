# analisevideo

**🇧🇷 [Português](README.md) · 🇺🇸 [English](README.en.md) · 🇪🇸 [Español](README.es.md)**

Análisis **visual y cinematográfico** de video con Gemini, archivado en una base
de datos local consultable. No es una transcripción del habla: lo que sale de
aquí es lo que un director de fotografía, un editor y un productor musical ven
en el material.

Funciona como skill de [OpenPCBot](https://github.com/inematds/openpcbotv2) con
el comando `/analisevideo` — deliberadamente **no se activa por vibra** (un
enlace suelto o «analiza este video» siguen yendo a inemaVOX).

## 📖 Guía de uso

Guía completa (landing + paso a paso): **https://inematds.github.io/analisevideo/guia/es/**

## Uso

```bash
S=~/projetos/apps/analisevideo/analisevideo.sh

bash $S analisa <url|arquivo> [slug] [--keep-src] [--prompt "..."]
bash $S ver     <slug>       # informe markdown
bash $S json    <slug>       # análisis sin procesar
bash $S prompts <slug>       # 1 prompt por escena (5–10), listo para Kling/Veo/Seedance
bash $S list    [N]          # últimos análisis
bash $S search  "<termo>"    # búsqueda en la base de datos
bash $S stats                # lo que más aparece en la base de datos
bash $S reindex              # reconstruye el índice a partir de las carpetas
```

Necesita `yt-dlp`, `ffmpeg`/`ffprobe`, `jq`, `python3` y una `GOOGLE_API_KEY`
(leída de `~/projetos/openpcbotv2/.env` o `~/projetos/wifi/.env`).

## Qué cubre el análisis

- **Prompts por escena** — de 5 a 10 prompts en inglés (+ negative prompt), uno
  por escena con código de tiempo, que cubren todo el video.
- **Fotografía** — paleta en hex, esquema de iluminación, temperatura, contraste,
  look, lente aparente, profundidad de campo, textura/grain.
- **Cámara bloque a bloque** con código de tiempo — plano, ángulo, movimiento
  (pan, dolly, gimbal, whip-pan, drone, orbital...), velocidad y nota.
- **Composición** — encuadre, reglas, capas de profundidad.
- **Montaje** — cortes estimados, cortes por minuto, ritmo, transiciones, match
  cut, jump cut, corte al beat, slow-mo/speed ramp.
- **Banda sonora y sonido** — género, BPM aproximado, instrumentación, mood, SFX,
  mezcla y sincronía con la imagen.
- **Texto y gráficos** — tipografía, animación, subtítulos, motion graphics.
- **Posproducción** — efectos, VFX, LUT sugerida, sound design.
- **Narrativa** — gancho de los primeros 3 segundos, estructura, arco, CTA.
- **Cómo recrearlo** — equipo mínimo, pasos prácticos y un prompt listo para
  Kling/Veo/Seedance que reproduce el look y el movimiento del fragmento más
  potente.

## Base de datos

Cada análisis se convierte en una carpeta en `~/projetos/output/analisevideo/<slug>/` con
`meta.json`, `analise.json` y `analise.md`. Se añade una línea por video a
`index.jsonl` — es lo que leen `list`, `search` y `stats`. El video descargado se
borra al final (guárdalo con `--keep-src`).

## Detalles

- Descarga hasta 480p deliberadamente: el color, la luz y el movimiento se leen
  bien y la carga es rápida. Por encima de 18 MB, el archivo se vuelve a
  codificar a 360p/12fps solo para el análisis.
- Si el enlace no se puede descargar con yt-dlp (sitio con sesión iniciada),
  descárgalo por otro medio y pasa la ruta.
- Los videos largos (>15 min) son costosos y vagos — es mejor analizar un
  fragmento.
- Los errores 429/500/503 de Gemini se reprocesan automáticamente (4 intentos).

## Claves de Gemini — tres, y cambia automáticamente

`analisa.py` intenta las claves en este orden y se detiene en la primera que
funciona:

| variable | dónde |
|---|---|
| `GOOGLE_API_KEY` | la de siempre |
| `GEMINI_API_KEY` | idem (se deduplica si tiene el mismo valor) |
| `GEMINI_API_KEY_INEMACCBOT_TIME` | `projects/1056030032122` |
| `GEMINI_API_KEY_INEMACCBOT_PROMPTS` | `projects/1000152753819` |

Todas se obtienen de `~/projetos/wifi/.env` (o del `.env` local).

**Qué hace que cambie y qué hace que espere** — esa es la distinción clave:

- **429 (cuota) y 403 (bloqueada)** → pasa a la siguiente clave **de inmediato**.
  Está en otro proyecto, con cuota propia; esperar no resolvería nada.
- **500 / 502 / 503** → Gemini está congestionado. La clave no tiene la culpa y
  cambiarla no ayuda: espera (20s, 40s, 60s, 90s, 120s) y vuelve a intentarlo
  con la misma.

La carga del archivo grande también entra en este proceso: si devuelve un 429,
cambia de clave en vez de convertirse en un error definitivo — justo el caso en
que el video ya se descargó y comprimió, y perder todo ese trabajo sería costoso.

Cuando usa una clave que no es la primera, indica cuál en stderr. Si todas
fallan por cuota, el error enumera **cuáles** fallaron, en vez de limitarse a
decir «hubo un error».


## Sitios que exigen sesión (Facebook, Instagram, TikTok, YouTube)

Facebook rechaza el `yt-dlp` sin más con `Cannot parse data` — no es una versión
antigua ni falta `--impersonate` (ambos se probaron el 2026-08-24 y fallaron
igual). Es un problema de **sesión**: con las cookies de Firefox, el mismo
enlace muestra los formatos y se descarga en 7 segundos.

El script encuentra el perfil automáticamente (el primero que tenga
`cookies.sqlite`, buscando primero en `snap`) y solo añade las opciones en esos
dominios. Para forzar otro perfil:

    export ANALISEVIDEO_FIREFOX_PROFILE=~/snap/firefox/common/.mozilla/firefox/xxxx.default

**Sin Firefox en la máquina** (VPS), la descarga continúa sin cookies y muestra
un aviso en stderr — sigue funcionando para los sitios públicos.


## Orden de los motores (2026-08-26: Gemini directo va primero)

    GOOGLE_API_KEY → GEMINI_API_KEY → …_TIME → …_PROMPTS  →  z-ai/glm-5.3-flash

El 2026-08-25 la alternativa había pasado al frente porque `stealth/ox-alpha`
era **gratis** y las tres claves de Gemini agotaban la cuota diaria con ~9
análisis. Ese modelo **desapareció de OpenRouter** — era el riesgo señalado
desde el principio. La alternativa actual es de pago, así que vuelve al final:
no tiene sentido gastar crédito cuando todavía queda cuota de Gemini. Entra
cuando se agota la cuota del día.

Para invertir el orden (poner la alternativa primero), **sin tocar el código**:

    export ANALISEVIDEO_MOTOR=reserva

## Alternativa: cuando el motor principal rechaza la solicitud

El paso costoso de `analisa` no es la llamada a la API — es la **descarga y la
compresión** que ocurren antes. Un clip de 41 MB de Facebook tarda minutos en
llegar hasta ese punto. El 2026-08-24, tres claves distintas de Gemini
respondieron 429 en el mismo minuto y todo ese trabajo se perdió.

Por eso existe la alternativa. El orden es:

    GOOGLE_API_KEY → GEMINI_API_KEY → …_TIME → …_PROMPTS → z-ai/glm-5.3-flash

`z-ai/glm-5.3-flash` (OpenRouter, `OPENROUTER_API_KEY` en `wifi/.env`) acepta
videos y devuelve el MISMO JSON — el prompt no cambia en absoluto. Se incorporó
el 2026-09-08 en lugar de `google/gemini-3.7-flash`: cuesta ~10x menos (US$ 0,075/M
de entrada frente a 0,75) y, al ser otro proveedor, no comparte la cuota ni la
congestión de Gemini, que es el motor principal.

**El uso de video en OpenRouter requiere US$ 1,00 disponibles EN LA CLAVE** —
no en la cuenta. Una clave con un límite (`limit` en openrouter.ai/settings/keys)
inferior a ese monto responde `402 … requires at least $1.00 in balance for video`
para cualquier video, incluso un clip de 20 s, mientras que el texto funciona.
Por eso la alternativa quedó inactiva desde 2026-08 hasta 2026-09-08. `analisa`
ahora imprime el cuerpo del error.

Está AL FINAL deliberadamente: **es de pago**. Cada vez que responde, `analisa`
lo indica en stderr (`ATENCAO: ... reserva paga`) — un motor de pago no se
ejecuta en silencio. Para cambiar el modelo alternativo, `OPENROUTER_VIDEO_MODEL`.

`_modelo` en el JSON registra QUIÉN hizo el análisis: sin eso, la base de datos
diría que fue Gemini un análisis que Gemini no hizo.

Sin `OPENROUTER_API_KEY`, todo funciona como antes — la alternativa es opcional.


## Volver a procesar sin descargar otra vez

El slug se obtiene del TÍTULO, y Facebook devuelve «Facebook» para todos los
enlaces — así que cada intento creaba `facebook-2`, `-3`, `-4`… y volvía a
descargar todo. El 2026-08-24 se descargó **cuatro veces** el MISMO clip (141 MB
de red y disco) debido a un error de cuota que no tenía nada que ver con la
descarga.

Ahora la **URL es la identidad**: si ya existe una carpeta con esa URL y el
archivo sigue en el disco, se reutiliza — se omiten la descarga y la compresión.
También se reutiliza la versión comprimida (`analise-src.mp4`), si existe.

Dos consecuencias prácticas:

- **Repetir `analisa <url>` después de un error es barato** — va directamente al
  análisis. Es la forma de volver a procesar lo que falló por la cuota.
- **`--keep-src` ahora resulta útil** para videos que quizá quieras volver a
  analizar (por ejemplo, con otro `--prompt`). Sin esa opción, el archivo se
  borra al final y la próxima vez se vuelve a descargar.

Pasar un `slug` explícito desactiva la reutilización: ahí estás indicando qué
carpeta quieres, y adivinar otra sería peor.
