# WAF with ModSecurity and the OWASP CRS on Docker

**English** · [Español](#waf-con-modsecurity-y-owasp-crs-sobre-docker)

Nginx with ModSecurity v3 and the OWASP Core Rule Set in front of
[OWASP Juice Shop](https://github.com/juice-shop/juice-shop), a deliberately vulnerable
shop. It blocks SQLi, XSS, path traversal and command injection, and its performance cost
is measured scenario by scenario.

[![Demo: the same SQL injection without the WAF (500) and through it (403)](https://florintodor.dev/media/poster/waf-modsecurity.jpg)](https://florintodor.dev/en/proyectos/waf-modsecurity/)

**Demo video and project page:** [florintodor.dev/en/proyectos/waf-modsecurity](https://florintodor.dev/en/proyectos/waf-modsecurity/)
**The full measurement:** [What it costs to put ModSecurity in front of an application](https://florintodor.dev/en/blog/cuanto-cuesta-un-waf/)

## Results

What each request returns when sent straight to the application and through the WAF:

| Request | Direct | WAF | CRS rules (paranoia 1) |
|---|---|---|---|
| `q=apple'))--` (SQLi) | 200 | 403 | 942100 |
| `q=' OR 1=1--` (SQLi) | 500 | 403 | 942100 |
| `q=<script>alert(1)</script>` | 200 | 403 | 941100, 941110, 941160 |
| `q=../../../etc/passwd` | 200 | 403 | 930100, 930110, 930120, 932160 |
| `q=a;cat /etc/passwd` | 200 | 403 | 930120, 932100, 932160 |

Performance cost with `ab` (5,000 requests, concurrency 10, median of three rounds) against
a static file, so the backend doesn't hide the difference:

| Scenario | req/s | ms per request |
|---|---|---|
| Juice Shop directly | 3,366 | 2.97 |
| Nginx as a proxy, no ModSecurity | 3,388 | 2.95 |
| WAF at paranoia 4, auditing everything (the repo's configuration) | 2,480 | 4.03 |
| WAF at paranoia 1, auditing only suspicious requests | 2,791 | 3.58 |

The proxy costs nothing measurable; the WAF adds between 0.6 and 1.1 ms per request. At
paranoia 4, searching for "apple juice" also returns 403 (rule 920273: the space is outside
the allowed character set), so for this application the sensible setting is paranoia 1.

## Architecture

```text
client  →  nginx-waf :8080  (Nginx 1.25.5 + ModSecurity 3.0.12 + CRS 3.3.5)  →  juice-shop :3000
           juice-shop :3001  (direct access, for comparison)
```

- `nginx_modsec/Dockerfile` builds libModSecurity and the Nginx connector as a dynamic
  module on top of `nginx:1.25-alpine`, and downloads the CRS at a pinned version.
- `nginx_modsec/modsecurity.d/crs-setup.conf` sets the paranoia level (it is at 4).
- `nginx_modsec/modsecurity.d/modsecurity.conf` has `SecAuditEngine On`, which writes every
  request to `audit.log` (around 900 MB per million requests). Outside of testing, use
  `RelevantOnly`.
- `my-socketio-exclusion.conf` excludes the rules that broke Juice Shop's WebSocket.

## Running it

```bash
docker compose up -d --build
curl -s -o /dev/null -w '%{http_code}\n' "http://localhost:8080/rest/products/search?q=%27%20OR%201=1--"   # 403
```

Nginx and ModSecurity logs are mounted in `nginx_modsec/logs/` and are not versioned.

Coursework for High Performance Web Servers (SWAP), University of Granada, 2025.

---

# WAF con ModSecurity y OWASP CRS sobre Docker

[English](#waf-with-modsecurity-and-the-owasp-crs-on-docker) · **Español**

Nginx con ModSecurity v3 y el OWASP Core Rule Set delante de
[OWASP Juice Shop](https://github.com/juice-shop/juice-shop), una tienda vulnerable a
propósito. Bloquea SQLi, XSS, path traversal y ejecución de comandos, y el coste en
rendimiento está medido por escenarios.

**Vídeo de la demo y ficha del proyecto:** [florintodor.dev/proyectos/waf-modsecurity](https://florintodor.dev/proyectos/waf-modsecurity/)
**La medición completa:** [Cuánto cuesta poner ModSecurity delante de una aplicación](https://florintodor.dev/blog/cuanto-cuesta-un-waf/)

## Resultados

Qué devuelve cada petición directa a la aplicación y a través del WAF:

| Petición | Directa | WAF | Reglas del CRS (paranoia 1) |
|---|---|---|---|
| `q=apple'))--` (SQLi) | 200 | 403 | 942100 |
| `q=' OR 1=1--` (SQLi) | 500 | 403 | 942100 |
| `q=<script>alert(1)</script>` | 200 | 403 | 941100, 941110, 941160 |
| `q=../../../etc/passwd` | 200 | 403 | 930100, 930110, 930120, 932160 |
| `q=a;cat /etc/passwd` | 200 | 403 | 930120, 932100, 932160 |

Coste en rendimiento con `ab` (5.000 peticiones, concurrencia 10, mediana de tres rondas)
contra un fichero estático, para que el backend no tape la diferencia:

| Escenario | req/s | ms por petición |
|---|---|---|
| Juice Shop directo | 3.366 | 2,97 |
| Nginx como proxy, sin ModSecurity | 3.388 | 2,95 |
| WAF en paranoia 4, auditando todo (la configuración del repo) | 2.480 | 4,03 |
| WAF en paranoia 1, auditando sólo lo sospechoso | 2.791 | 3,58 |

El proxy no cuesta nada medible; el WAF, entre 0,6 y 1,1 ms por petición. En paranoia 4,
además, buscar «apple juice» devuelve 403 (regla 920273: el espacio queda fuera del juego
de caracteres permitido), así que para esta aplicación la configuración razonable es
paranoia 1.

## Arquitectura

```text
cliente  →  nginx-waf :8080  (Nginx 1.25.5 + ModSecurity 3.0.12 + CRS 3.3.5)  →  juice-shop :3000
            juice-shop :3001  (acceso directo, para comparar)
```

- `nginx_modsec/Dockerfile` compila libModSecurity y el conector de Nginx como módulo
  dinámico sobre `nginx:1.25-alpine`, y descarga el CRS en su versión fijada.
- `nginx_modsec/modsecurity.d/crs-setup.conf` fija el nivel de paranoia (está en 4).
- `nginx_modsec/modsecurity.d/modsecurity.conf` tiene `SecAuditEngine On`, que escribe cada
  petición en `audit.log` (unos 900 MB por millón de peticiones). Para algo que no sea una
  prueba, `RelevantOnly`.
- `my-socketio-exclusion.conf` excluye las reglas que rompían el WebSocket de Juice Shop.

## Arrancarlo

```bash
docker compose up -d --build
curl -s -o /dev/null -w '%{http_code}\n' "http://localhost:8080/rest/products/search?q=%27%20OR%201=1--"   # 403
```

Los logs de Nginx y ModSecurity se montan en `nginx_modsec/logs/` y no se versionan.

Trabajo de la asignatura Servidores Web de Altas Prestaciones (SWAP), Universidad de
Granada, 2025.
