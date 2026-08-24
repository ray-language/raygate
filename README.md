# raygate

API gateway / reverse proxy escrito en [raylang](https://github.com/roberto-ayala/raylang): rutas declarativas en TOML, rate limiting, circuit breaker por upstream, validación JWT, reintentos bajo deadline, **proxy en streaming** (cuerpo-a-cuerpo, con contrapresión), métricas Prometheus, trace-context W3C propagado y logs JSON con `trace_id`. Es la app que estruja el eje servidor completo del lenguaje: `net/webserver` y el cliente `net/http` a la vez, `std/resilience`, `net/{jwt,trace,metrics,log}` y fibras bajo carga.

```text
$ raygate check --config raygate.toml
listen 127.0.0.1:8080
  /api/    -> http://127.0.0.1:9001 (timeout 5000 ms, breaker 5/10000 ms +rate:50/s +retries:2)
  /events/ -> http://127.0.0.1:9002 (timeout 30000 ms, breaker 5/10000 ms +stream)
  /admin/  -> http://127.0.0.1:9003 (timeout 10000 ms, breaker 5/10000 ms +jwt)

$ raygate --config raygate.toml
{"ts":"…","level":"INFO","service":"raygate","msg":"gateway listening","port":8080}
{"ts":"…","level":"INFO","service":"raygate","trace_id":"cf36e2…","msg":"proxied","route":"api","method":"GET","status":200,"ms":2}
```

## Configuración

Las rutas son tablas con nombre (`[route.api]`) — `std/toml` aún no soporta
arrays de tablas `[[route]]` (hallazgo anotado); el nombre es la etiqueta de
métricas. Prefijo más largo gana.

```toml
[server]
bind = "0.0.0.0"
port = 8080
drain_ms = 3000            # drenado del apagado graceful (SIGTERM/SIGINT)
metrics_path = "/metrics"  # "" lo desactiva
health_path = "/health"

[route.api]
prefix = "/api/"
upstream = "http://127.0.0.1:9001"
strip_prefix = true        # /api/users -> /users
timeout_ms = 5000          # presupuesto TOTAL (los reintentos viven dentro de él)
retries = 2                # solo métodos idempotentes (GET/HEAD/OPTIONS)
breaker_threshold = 5      # fallos seguidos que abren el circuito
breaker_cooldown_ms = 10000
rate_limit = 50            # req/s por ruta (token bucket); 0 = sin límite
jwt_secret = ""            # no vacío = exige Bearer HS256 (y valida exp)
stream = false             # true = respuesta en streaming (sin reintentos)
add_request_headers = ["X-Gateway: raygate"]
```

## Qué hace por petición

1. Ruta por prefijo más largo; `/metrics` y `/health` los sirve el propio gateway.
2. **Admisión** en el actor de control: breaker abierto → 503 al instante;
   token bucket agotado → 429.
3. **JWT** si la ruta lo exige: firma HS256 (`net/jwt`) + expiración (`exp`) →
   401 con `WWW-Authenticate`.
4. **Upstream**: cabeceras reenviadas menos hop-by-hop (RFC 9110 §7.6.1),
   extras de la ruta añadidas, `traceparent` HIJO propagado (W3C).
   - *Buffered*: reintentos con backoff+jitter (`resilience.retry`) bajo un
     `deadline` único — cada intento recibe el presupuesto restante como
     timeout, los reintentos jamás alargan el total.
   - *Streaming*: status y cabeceras en cuanto llegan; el cuerpo se bombea
     trozo a trozo por un canal ACOTADO (un cliente lento frena la lectura del
     upstream). Verificado: primer byte en 2 ms con un upstream que tarda
     500 ms en terminar.
5. **Report** al actor: breaker, contadores y histograma de latencia; log JSON
   con `trace_id`, ruta, método, status y ms.

Todo el estado mutable (breakers, buckets, métricas) vive en **un actor**;
los handlers son fibras que le hablan por canal (mismo patrón que rayrelay).

## Rendimiento (sanity check)

1000 peticiones (20 fibras × 50) atravesando el hop completo
cliente→gateway→upstream en la misma máquina: **~4.9k req/s en la VM, ~5.5k
req/s en nativo** (generador de carga co-alojado y en VM: cota inferior).

## Estado actual

| Capacidad | Estado |
|-----------|--------|
| Rutas TOML declarativas + prefijo más largo + `check` | ✅ |
| Rate limit por ruta (token bucket) → 429 | ✅ |
| Circuit breaker por ruta (`std/resilience`) → 503 fail-fast | ✅ |
| Reintentos idempotentes con backoff bajo deadline único | ✅ |
| JWT HS256 + expiración → 401 | ✅ |
| Proxy streaming con contrapresión (canal acotado) | ✅ |
| Higiene de cabeceras hop-by-hop + extras por ruta | ✅ |
| `traceparent` W3C propagado + logs JSON con `trace_id` | ✅ |
| Métricas Prometheus (`/metrics`): contadores + histograma | ✅ |
| Apagado graceful (`serve_graceful`, SIGTERM/SIGINT + drenado) | ✅ |
| Binario nativo (E2E y streaming verificados) | ✅ |
| Tests (config + E2E completo con upstreams reales) | ✅ 4 |
| Rate limit por IP de cliente / X-Forwarded-For | ❌ bloqueado (ver hallazgos) |
| Passthrough WebSocket/SSE de larga vida, TLS de entrada | 📋 v2 |

## Hallazgos de dogfood (necesidades confirmadas del lenguaje)

Anotados en `raylang/IDEAS.md` §65:

1. **`webserver.Request` no expone la dirección del cliente** → imposible el
   rate limit por IP y el `X-Forwarded-For` que cualquier proxy real necesita.
2. **`ray test` deja listeners medio muertos entre tests**: las fibras de un
   `@test` anterior se descartan pero sus sockets de escucha del SO sobreviven
   (aceptan y nadie atiende) → un boot compartido entre tests se envenena; el
   E2E vive en UN solo `@test`. Sin `var` top-level tampoco hay "boot once".
3. **`std/toml` sin arrays de tablas `[[route]]`** (diferido documentado, aquí
   confirmado como necesidad real: es LA forma natural de configurar un proxy).
4. **`resilience.guard` no sirve cuando la llamada protegida no puede correr
   en la fibra dueña del estado** (el gateway reimplementa sus transiciones
   sobre los campos del `Breaker` — que además expone `abierto_hasta` en
   español). Un par `admit/report` de primera clase encajaría mejor.
5. `jwt_verify` valida solo la firma (documentado): cada gateway reescribe la
   política de `exp` — candidato a un helper con claims.
6. **Positivo y citable**: la nota "VM only" de `webserver.serve` está
   desactualizada — el gateway completo (accept, fibras, streaming chunked,
   señales) funciona compilado a nativo; `stream_response` + `http.stream_with`
   componen un proxy streaming real con contrapresión sin tocar el runtime.

## Desarrollo

```sh
ray test                          # 4 tests (config + E2E con upstreams reales)
ray run src/main.ray check
ray run src/main.ray --config raygate.toml
ray build --native src/main.ray -o raygate --release
```

Estructura: `src/main.ray` (CLI) · `config.ray` (TOML → rutas) · `control.ray`
(actor: breakers, buckets, métricas) · `proxy.ray` (cabeceras, JWT, buffered/
streaming) · `gateway.ray` (handler + serve graceful) · `debug/` (upstreams y
generador de carga usados en las medidas).
