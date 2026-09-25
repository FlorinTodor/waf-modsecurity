# WAF con ModSecurity y OWASP CRS sobre Docker

Nginx con ModSecurity v3 y el OWASP Core Rule Set delante de
[OWASP Juice Shop](https://github.com/juice-shop/juice-shop), una tienda vulnerable a
propósito. Bloquea SQLi, XSS, path traversal y ejecución de comandos, y el coste en
rendimiento está medido por escenarios.

[![Demo: la misma inyección SQL sin WAF (500) y a través del WAF (403)](https://florintodor.dev/media/poster/waf-modsecurity.jpg)](https://florintodor.dev/proyectos/waf-modsecurity/)

**Vídeo de la demo y ficha del proyecto:** [florintodor.dev/proyectos/waf-modsecurity](https://florintodor.dev/proyectos/waf-modsecurity/)
**La medición completa:** [Cuánto cuesta un WAF con ModSecurity, medido](https://florintodor.dev/blog/cuanto-cuesta-un-waf/)

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

---

Trabajo de la asignatura Servidores Web de Altas Prestaciones (SWAP), Universidad de
Granada, 2025.
