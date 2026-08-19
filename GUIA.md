# Lab de nginx — solo ejercicios

No hay soluciones aquí. Ni una línea de config. Todo lo escribes tú,
de memoria, y lo verificas tú. Esta guía solo dice **qué** hacer y
**cómo** comprobar que funcionó. Si un ejercicio te lleva más de un intento,
perfecto: ese es el punto.

---

## Entorno (única parte que no practicas)

Un container `nginx:alpine` en el puerto 8081. Tu carpeta `conf.d/` se
monta como config de servidores y tu carpeta `www/` como contenido web.
Es nginx puro: mismos archivos y mismas directivas que en un servidor real.

```bash
# levantar con tu config y tu contenido (desde la carpeta del ejercicio)
docker run -d --name lab -p 8081:80 \
  -v "$PWD/conf.d:/etc/nginx/conf.d:ro" \
  -v "$PWD/www:/usr/share/nginx/html:ro" \
  nginx:alpine

docker exec lab nginx -t            # validar config (siempre antes de recargar)
docker exec lab nginx -s reload     # aplicar cambios sin reiniciar
docker logs -f lab                  # logs en vivo
docker rm -f lab                    # borrar el ejercicio entero
```

Dentro del container: config principal en `/etc/nginx/nginx.conf`,
servidores en `/etc/nginx/conf.d/`, logs en `/var/log/nginx/`.

Cada ejercicio vive en su propia carpeta con `conf.d/` y `www/`. Al
terminar, `docker rm -f lab` y sigues en la siguiente.

---

## Ejercicios

### 00 — Hola mundo
- **Objetivo:** un server que escuche en 80 y sirva un `index.html` tuyo desde `www/`.
- **Verificación:** `curl http://localhost:8081/` devuelve tu HTML. Los headers (`curl -sI`) dicen `Server: nginx` y `200 OK`.
- **Ojo:** pide una ruta que no exista. ¿Qué código de estado devuelve? ¿Por qué?

### 01 — Server a mano
- **Objetivo:** desde cero (sin copiar el default), un `server {}` con `listen`, `server_name` (pruébalo con `curl -H "Host: ..."`), `root` e `index`. Crea el index tú.
- **Verificación:** la ruta `/` resuelve al index; una ruta con archivo real (`www/hola.txt`) se sirve; una ruta sin archivo da 404.
- **Ojo:** quita `index` y pide `/` de nuevo. ¿Qué cambia y por qué?

### 02 — location
- **Objetivo:** un server con 4 `location` distintos (coincidencia exacta, prefijo con prioridad, regex, y el genérico). Que cada uno responda un texto distinto.
- **Verificación:** pide rutas que coincidan con cada uno, y también variantes tramposas (con barra final, mayúsculas, dos segmentos) hasta que sepas predecir cuál gana **antes** de ejecutar el curl.
- **Ojo:** la pregunta del millón: ¿qué coincide con `/exacto/`? ¿y la regex con mayúsculas? El orden de precedencia tiene que salirte sin pensar.

### 03 — try_files
- **Objetivo:** un `try_files` que pruebe ruta, ruta como carpeta, y caiga en un fallback.
- **Verificación:** el archivo real se sirve, y una ruta inexistente cae en el fallback en vez de 404.
- **Ojo:** cambia el fallback a `=404` y a un archivo tuyo. Cuando domines esto ya sabes el patrón de las apps SPA.

### 04 — Redirecciones
- **Objetivo:** redirigir una ruta vieja a una nueva con 301, y una ruta con segmento variable a otra con ese segmento (regex + captura). La nueva ruta responde 200 con tu texto.
- **Verificación:** `curl -sI` muestra el 3xx y `Location:`. `curl -sL` termina en la nueva ruta.
- **Ojo:** prueba 301 vs 302 y piensa cuándo cada uno. ¿Qué pasa con un cliente que cachea?

### 05 — Virtual hosts
- **Objetivo:** dos servidores en el mismo puerto que responden distinto según `Host`. Cada uno con su `www/` y su `server_name`.
- **Verificación:** `curl -H "Host: a.tu-dominio"` y `-H "Host: b.tu-dominio"` devuelven sitios distintos. Un Host cualquiera cae en el primero (el default).
- **Ojo:** ¿cuál es el default y cómo elegirías tú cuál debe ser?

### 06 — Reverse proxy
- **Objetivo:** nginx recibe en 8081 y delega a un backend que corras aparte (un segundo container sirviendo algo, o el puerto de cualquier servicio local que tengas). A la vez, nginx sirve estáticos propios en otra ruta.
- **Verificación:** la ruta principal devuelve el contenido del backend, y el `Server:` header sigue siendo el de nginx. La ruta estática se sirve sin pasar por el backend.
- **Ojo:** ¿qué header dice quién es el cliente real? ¿Cómo ve el backend la IP del cliente? Pruébalo mirando el log del backend.

### 07 — Load balancing
- **Objetivo:** un `upstream` con 3 backends distintos (puedes correr 3 containers con contenidos diferentes, o un mismo servicio en 3 puertos). `proxy_pass` hacia el grupo.
- **Verificación:** pide 6 veces seguidas y observa cómo se reparte. Después mata un backend (¿cómo?) y vuelve a pedir.
- **Ojo:** ¿las peticiones caen siempre en el mismo backend? ¿Qué implica eso para una app con sesiones? ¿Qué opción cambia esa distribución?

### 08 — Logs
- **Objetivo:** un `log_format` propio y un `access_log` con ese formato. Genera peticiones de distintos tipos (200, 404, con query string) y lee el log.
- **Verificación:** cada petición aparece con los campos que definiste. El `error.log` registra los 404 (¿con qué nivel?).
- **Ojo:** ¿qué variable te dice el navegador del cliente? ¿Y la ruta pedida? Son las que van a usar tus `log_format`.

### 09 — gzip
- **Objetivo:** activar compresión para texto y JSON. Crea un archivo de texto grande (>1KB) tú.
- **Verificación:** `curl -H "Accept-Encoding: gzip" -w "%{size_download}"` da menos bytes que sin el header, y `curl -sI -H "Accept-Encoding: gzip"` muestra `Content-Encoding: gzip`.
- **Ojo:** ¿qué pasa si el cliente no pide gzip? ¿Y si el archivo es chico? No comprimas todo sin pensar.

### 10 — Errores
- **Objetivo:** una página 404 propia tuya, servida por un location interno (el cliente no puede pedirla directo).
- **Verificación:** una ruta inexistente devuelve 404 con TU página. El código de estado sigue siendo 404 (no 200). Pedir la página de error directo da 404 también.
- **Ojo:** haz que el 502 (backend caído) también muestre tu página. Vas a necesitar un proxy al que se pueda matar.

---

## Reglas del lab

1. **Nada de copiar.** Si necesitas mirar una solución, te la tienes que recordar de la doc oficial (`cat /etc/nginx/nginx.conf` y `nginx -T` son las únicas fuentes permitidas). Verificar ≠ copiar.
2. **Un ejercicio, dos veces.** La segunda vez, de memoria, con otros nombres de rutas y hosts. Si dudaste en un `;` o en un nivel de bloque, no lo sabes todavía.
3. **Antes de recargar, `nginx -t`.** Siempre.
4. **Los errores son el progreso.** Un 404 inesperado te enseña más que diez aciertos.
5. **Anti-objetivos:** sin TLS, sin caching, sin rate limiting, sin tuning, sin compilar módulos. Solo las bases, hasta que duela.