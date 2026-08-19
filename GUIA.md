# Lab de nginx — solo ejercicios

No hay soluciones acá. Ni una línea de config. Todo lo escribís vos,
de memoria, y lo verificás vos. Esta guía solo dice **qué** hacer y
**cómo** comprobar que funcionó. Si un ejercicio te lleva más de un intento,
perfecto: ese es el punto.

---

## Entorno (única parte que no practicás)

nginx instalado nativo en la máquina (`sudo apt install nginx`). Servicio
gestionado por `service`. Todo vive en `/etc/nginx/`.

- `nginx.conf` — la config principal (bloque `http`)
- `sites-available/` — tus `server {}` sin activar, un archivo por sitio
- `sites-enabled/` — links a los sitios activos
- `conf.d/` — alternativa: cualquier `.conf` acá se carga solo
- Contenido web por defecto: `/var/www/html`
- Logs: `/var/log/nginx/access.log` y `error.log`

```bash
nginx -t                    # validar config (siempre antes de recargar)
sudo service nginx reload   # aplicar cambios sin cortar conexiones
sudo service nginx restart  # reiniciar
sudo service nginx stop     # apagar
```

Cada ejercicio vive en su propia carpeta de config. Al terminar, la borrás
o seguís en la siguiente.

---

## Ejercicios

### 00 — Hola mundo
- **Objetivo:** un server que escuche en 80 y sirva un `index.html` tuyo desde `www/`.
- **Verificación:** `curl http://localhost:8081/` devuelve tu HTML. Los headers (`curl -sI`) dicen `Server: nginx` y `200 OK`.
- **Ojo:** pedí una ruta que no exista. ¿Qué status devuelve? ¿Por qué?

### 01 — Server a mano
- **Objetivo:** desde cero (sin copiar el default), un `server {}` con `listen`, `server_name` (probalo con `curl -H "Host: ..."`), `root` e `index`. Creá el index vos.
- **Verificación:** la ruta `/` resuelve al index; una ruta con archivo real (`www/hola.txt`) se sirve; una ruta sin archivo da 404.
- **Ojo:** quitá `index` y pedí `/` de nuevo. ¿Qué cambia y por qué?

### 02 — location
- **Objetivo:** un server con 4 `location` distintos (match exacto, prefijo con prioridad, regex, y el genérico). Que cada uno responda un texto distinto.
- **Verificación:** pedí rutas que matcheen cada uno, y también variantes tramposas (con barra final, mayúsculas, dos segmentos) hasta que sepas predecir cuál gana **antes** de ejecutar el curl.
- **Ojo:** la pregunta del millón: ¿qué matchea `/exacto/`? ¿y la regex con mayúsculas? El orden de precedencia tiene que salirte sin pensar.

### 03 — try_files
- **Objetivo:** un `try_files` que pruebe ruta, ruta como carpeta, y caiga en un fallback.
- **Verificación:** el archivo real se sirve, y una ruta inexistente cae en el fallback en vez de 404.
- **Ojo:** cambiá el fallback a `=404` y a un archivo tuyo. Cuando domines esto ya sabés el patrón de las apps SPA.

### 04 — Redirecciones
- **Objetivo:** redirigir una ruta vieja a una nueva con 301, y una ruta con segmento variable a otra con ese segmento (regex + captura). La nueva ruta responde 200 con tu texto.
- **Verificación:** `curl -sI` muestra el 3xx y `Location:`. `curl -sL` termina en la nueva ruta.
- **Ojo:** probá 301 vs 302 y pensá cuándo cada uno. ¿Qué pasa con un cliente que cachea?

### 05 — Virtual hosts
- **Objetivo:** dos servidores en el mismo puerto que responden distinto según `Host`. Cada uno con su `www/` y su `server_name`.
- **Verificación:** `curl -H "Host: a.tu-dominio"` y `-H "Host: b.tu-dominio"` devuelven sitios distintos. Un Host cualquiera cae en el primero (el default).
- **Ojo:** ¿cuál es el default y cómo elegirías vos cuál debe ser?

### 06 — Reverse proxy
- **Objetivo:** nginx recibe en 8081 y delega a un backend que corras aparte (un segundo container sirviendo algo, o el puerto de cualquier servicio local que tengas). A la vez, nginx sirve estáticos propios en otra ruta.
- **Verificación:** la ruta principal devuelve el contenido del backend, y el `Server:` header sigue siendo el de nginx. La ruta estática se sirve sin pasar por el backend.
- **Ojo:** ¿qué header dice quién es el cliente real? ¿Cómo ve el backend la IP del cliente? Probalo mirando el log del backend.

### 07 — Load balancing
- **Objetivo:** un `upstream` con 3 backends distintos (podés correr 3 containers con contenidos diferentes, o un mismo servicio en 3 puertos). `proxy_pass` hacia el grupo.
- **Verificación:** pedí 6 veces seguidas y observá cómo se reparte. Después matá un backend (¿cómo?) y volvé a pedir.
- **Ojo:** ¿los requests caen siempre en el mismo backend? ¿Qué implica eso para una app con sesiones? ¿Qué opción cambia esa distribución?

### 08 — Logs
- **Objetivo:** un `log_format` propio y un `access_log` con ese formato. Generá requests de distintos tipos (200, 404, con query string) y leé el log.
- **Verificación:** cada request aparece con los campos que definiste. El `error.log` registra los 404 (¿con qué nivel?).
- **Ojo:** ¿qué variable te dice el navegador del cliente? ¿Y la ruta pedida? Son las que van a usar tus `log_format`.

### 09 — gzip
- **Objetivo:** activar compresión para texto y JSON. Creá un archivo de texto grande (>1KB) vos.
- **Verificación:** `curl -H "Accept-Encoding: gzip" -w "%{size_download}"` da menos bytes que sin el header, y `curl -sI -H "Accept-Encoding: gzip"` muestra `Content-Encoding: gzip`.
- **Ojo:** ¿qué pasa si el cliente no pide gzip? ¿Y si el archivo es chico? No comprimas todo sin pensar.

### 10 — Errores
- **Objetivo:** una página 404 propia tuya, servida por un location interno (el cliente no puede pedirla directo).
- **Verificación:** una ruta inexistente devuelve 404 con TU página. El status code sigue siendo 404 (no 200). Pedir la página de error directo da 404 también.
- **Ojo:** hacé que el 502 (backend caído) también muestre tu página. Vas a necesitar un proxy al que se pueda matar.

---

## Reglas del lab

1. **Nada de copiar.** Si necesitás mirar una solución, te la tenés que acordar de la doc oficial (`cat /etc/nginx/nginx.conf` y `nginx -T` son las únicas fuentes permitidas). Verificar ≠ copiar.
2. **Un ejercicio, dos veces.** La segunda vez, de memoria, con otros nombres de rutas y hosts. Si dudaste en un `;` o en un nivel de bloque, no lo sabés todavía.
3. **Antes de recargar, `nginx -t`.** Siempre.
4. **Los errores son el progreso.** Un 404 inesperado te enseña más que diez aciertos.
5. **Anti-objetivos:** sin TLS, sin caching, sin rate limiting, sin tuning, sin compilar módulos. Solo las bases, hasta que duela.
