---
name: hermes-windows-setup
description: "Configure, debug, and troubleshoot Hermes Agent on Windows (git-bash/MSYS). Covers model/provider setup, .env pitfalls, path resolution quirks, and diagnostic workflows."
version: 1.1.0
author: Hermes Agent
license: MIT
platforms: [windows, wsl]
metadata:
  hermes:
    tags: [hermes, windows, setup, configuration, debugging, providers, env]
    related_skills: [hermes-agent]
---

# Hermes Agent — Windows Setup & Debugging

Guía para configurar, diagnosticar y resolver problemas de Hermes Agent en Windows usando git-bash (MSYS). Cubre los escollos específicos de Windows que no están en la skill `hermes-agent` (bundled) o que necesitan más detalle.

## When to Use

Cargar esta skill cuando:
- El usuario pide configurar un provider/modelo en Hermes en Windows
- `hermes doctor` reporta problemas de API key o conectividad
- Los comandos de shell para manipular `.env`/config fallan silenciosamente
- Hay confusión entre rutas MSYS, Python, y las reales de Hermes

## Windows Path Resolution

Tres "verdades" distintas sobre `~` y `$HOME`:

| Contexto | `~` / `$HOME` resuelve a |
|----------|--------------------------|
| **git-bash (MSYS)** | `/c/Users/<user>/` |
| **Python `os.path.expanduser('~')`** | `C:\Users\<user>\` |
| **Hermes config real** | `%LOCALAPPDATA%\hermes\` → `C:\Users\<user>\AppData\Local\hermes\` |

**Regla de oro**: al leer/escribir archivos de configuración de Hermes desde Python (`execute_code`), usar siempre rutas absolutas de Windows (`C:\Users\...`), nunca `~` ni `os.path.expanduser('~')`. La skill `hermes-agent` usa `get_hermes_home()` internamente, pero desde `execute_code` no tenemos acceso a eso.

```python
# ✅ Correcto
env_path = r"C:\Users\wruizz\AppData\Local\hermes\.env"

# ❌ No confiar en esto en Windows
env_path = os.path.expanduser("~/.hermes/.env")
```

Para descubrir la ruta real: `hermes config path` y `hermes config env-path`.

### WSL (Windows Subsystem for Linux)

Cuando Hermes se ejecuta desde WSL, hay **tres ubicaciones posibles** donde puede residir la configuración. Identificar cuál usa tu instancia es el primer paso de cualquier diagnóstico.

#### Las Tres Ubicaciones de Hermes en Windows+WSL

| # | Ruta Windows | Ruta WSL | Quién la usa |
|---|-------------|----------|-------------|
| 1 | `C:\Users\<user>\.hermes\` | `/mnt/c/Users/<user>/.hermes/` | Hermes en Windows nativo (default `~`) |
| 2 | `C:\Users\<user>\AppData\Local\hermes\` | `/mnt/c/Users/<user>/AppData/Local/hermes/` | `HERMES_HOME` (instalación oficial) |
| 3 | `\\wsl$\<distro>\home\<user>\.hermes\` | `/home/<user>/.hermes/` | **Hermes en WSL nativo** (Linux `~`) |

**⚠️ Trampa crítica**: en WSL, `~` (tilde) resuelve a `/home/<user>/` (Linux home), **NO** a `C:\Users\<user>\` (Windows home). Son directorios completamente distintos en sistemas de archivos diferentes. Cuando un usuario en WSL dice "edité `~/.hermes/.env`", está editando la ubicación #3, no la #1.

**⚠️ Segunda trampa**: `HERMES_HOME` se establece con rutas estilo Windows (`C:\Users\<user>\AppData\Local\hermes`), pero Hermes ejecutándose dentro de WSL puede ignorar `HERMES_HOME` y usar `~/.hermes/` del Linux home. **Siempre verificar con `hermes config path` desde dentro de WSL** para confirmar qué ubicación está usando realmente.

**Diagnóstico de múltiples instalaciones**:
```bash
# Desde WSL — ¿dónde está leyendo Hermes realmente?
hermes config path
hermes config env-path

# Verificar las 3 ubicaciones posibles
ls -la /mnt/c/Users/$USER/.hermes/.env 2>/dev/null && echo "→ Ubicación #1 existe"
ls -la /mnt/c/Users/$USER/AppData/Local/hermes/.env 2>/dev/null && echo "→ Ubicación #2 existe"
ls -la ~/.hermes/.env 2>/dev/null && echo "→ Ubicación #3 existe (WSL nativo)"
```

**⚠️ Error común**: editar el `.env` en una ubicación mientras Hermes lee de otra. Si `hermes config env-path` dice `C:\Users\...\AppData\Local\hermes\.env` pero el error muestra una key diferente a la que acabas de guardar, es porque Hermes está leyendo de otra ubicación (#1 o #3).

**Arreglo rápido**: copiar los archivos de config al directorio que reporta `hermes config path`, no a `~/.hermes/`:
```bash
# ❌ No hacer esto en WSL — no funciona
cp config.yaml ~/.hermes/config.yaml

# ✅ Hacer esto — usa la ruta real
cp config.yaml "$(hermes config path | sed 's|\\|/|g')"
```

## .env Files and CRLF Pitfalls

Los archivos `.env` en Windows suelen tener terminaciones CRLF (`\r\n`). Esto rompe los pipelines de shell en git-bash:

```bash
# ❌ ROTO en Windows — CRLF rompe cut/xargs/export
grep '^OPENAI_API_KEY=' ~/.hermes/.env | cut -d'=' -f2-
export $(grep -v '^#' ~/.hermes/.env | xargs)

# ✅ Usar Python para extraer valores del .env
```

**Diagnóstico**: si `export $(grep ... | xargs)` falla silenciosamente (variable vacía), sospechar CRLF. Verificar con:

```bash
file ~/.hermes/.env          # "ASCII text, with CRLF line terminators"
cat -A ~/.hermes/.env        # Las líneas terminan en ^M$
```

**Solución**: usar `execute_code` (Python) para leer y parsear el `.env`, o usar `tr -d '\r'`:

```bash
# Alternativa shell si es necesario
KEY=$(grep '^OPENAI_API_KEY=' ~/.hermes/.env | tr -d '\r' | cut -d'=' -f2-)
```

## Configuring Custom Providers

### DeepSeek

DeepSeek usa un endpoint compatible con OpenAI. La API key DEBE empezar con `sk-`:

```yaml
# config.yaml
model:
  default: "deepseek-v4-pro"
  provider: "custom"
  base_url: "https://api.deepseek.com/v1"
```

```bash
# .env — IMPORTANTE: la variable es OPENAI_API_KEY, NO DEEPSEEK_API_KEY
OPENAI_API_KEY=sk-xxxxxxxxxxxxxxxx
```

**Verificación de conectividad** (desde Python en `execute_code`):

```python
import urllib.request, json

# Usar ruta absoluta de Windows
with open(r"C:\Users\<user>\AppData\Local\hermes\.env", "r") as f:
    for line in f:
        if line.strip().startswith("OPENAI_API_KEY="):
            key = line.strip().split("=", 1)[1]

req = urllib.request.Request(
    "https://api.deepseek.com/v1/models",
    headers={"Authorization": f"Bearer {key}"}
)
resp = urllib.request.urlopen(req, timeout=15)
data = json.loads(resp.read())
for m in data["data"]:
    print(f"  - {m['id']}")
```

Modelos disponibles (mayo 2026): `deepseek-v4-pro`, `deepseek-v4-flash`.

### Otros providers custom

El patrón es el mismo para cualquier endpoint compatible con OpenAI:
- `model.provider: "custom"`
- `model.base_url: "<endpoint>/v1"`
- `OPENAI_API_KEY` en `.env` (o la variable que espere el endpoint)

## Diagnostic Workflow

Cuando Hermes muestra "No inference provider configured" o errores de autenticación (HTTP 401):

0. **(WSL) Verificar múltiples ubicaciones**: antes de tocar cualquier archivo, identificar en cuál de las 3 ubicaciones está leyendo Hermes realmente. Si el error persiste tras editar un `.env`, sospechar que se editó el equivocado.
1. **Confirmar rutas**: `hermes config path` y `hermes config env-path`
2. **Verificar config**: `hermes config` — revisar `model.provider`, `model.base_url`, `model.default`
3. **Verificar .env**: ¿existe en la ruta de `hermes config env-path`? ¿tiene la key correcta? Verificar que `OPENAI_API_KEY` existe (`grep -c "OPENAI_API_KEY"`) y que no hay un `DEEPSEEK_API_KEY` corrupto.
4. **Verificar formato de key**: para DeepSeek, debe empezar con `sk-`
5. **Probar conectividad**: usar el script Python de arriba (no curl desde shell — CRLF)
6. **Diagnóstico completo**: `hermes doctor`
7. **Test de inferencia real**: `hermes chat -q -m <modelo> --provider custom -Q` — si responde correctamente, la config funciona sin importar lo que diga el footer de la TUI

## TUI Paste / Keyboard Issues in Git Bash (MSYS2)

En la TUI interactiva de Hermes (prompt `❯`), los atajos de teclado estándar de Windows **no siempre funcionan** porque Git Bash (MINGW64) y prompt_toolkit procesan las teclas de forma diferente:

| Atajo | ¿Funciona? | Notas |
|-------|-----------|-------|
| `Shift+Insert` | ✅ Sí | El más fiable para pegar |
| `Ctrl+Shift+V` | ✅ Suele funcionar | Depende del terminal |
| `Ctrl+V` | ❌ No | Interpretado literalmente por prompt_toolkit |
| Click derecho | ❌ No | Abre menú contextual de Git Bash |
| `Ctrl+Shift+C/V` | ❌ No | Son atajos de Windows Terminal, no de Git Bash |

**Mejor práctica**: para pegar valores largos (API keys, tokens), NO uses la TUI. Edita el archivo directamente con nano o un editor externo:

```bash
# Editar .env directamente — evita el problema de paste en la TUI
nano "$(hermes config env-path)"

# O abrir con VS Code desde Windows
code "$(hermes config env-path | sed 's|\\|/|g')"
```

Esto aplica a `hermes setup`, `hermes model`, y cualquier wizard interactivo que pida pegar texto.

## MSYS2/Git Bash ≠ WSL — Cómo distinguirlos

Los usuarios a menudo confunden Git Bash con WSL. La diferencia importa porque las rutas y el comportamiento de `$HOME` son distintos:

| Característica | Git Bash (MSYS2/MINGW64) | WSL |
|---------------|--------------------------|-----|
| `uname` | `MINGW64_NT-...` | `Linux` |
| `$HOME` | `/c/Users/<user>/` | `/home/<user>/` |
| `/mnt/` | No existe | `/mnt/c/` → `C:\` |
| Acceso a Windows | Nativo (`/c/...`) | Via `/mnt/c/...` |

**Diagnóstico rápido**: `echo $HOME` — si devuelve `/c/Users/...`, estás en Git Bash, no en WSL.

## Common Pitfalls

1. **Creer que `echo $VAR` en git-bash refleja el `.env`**. git-bash no carga automáticamente `.env` — solo Hermes lo hace al iniciar.
2. **Crear `.env` en `~/.hermes/` de MSYS en lugar de `%LOCALAPPDATA%\hermes\`**. Usar `hermes config env-path` para saber la ruta exacta.
3. **En WSL, copiar config a `~/.hermes/` (MSYS) esperando que funcione**. `HERMES_HOME` apunta al directorio de instalación con rutas Windows. Usar `hermes config path` como fuente de verdad.
4. **Usar `DEEPSEEK_API_KEY` en lugar de `OPENAI_API_KEY`**. El provider `custom` siempre busca `OPENAI_API_KEY`.
5. **Shell piping con CRLF**. `grep`/`cut`/`xargs` fallan silenciosamente. Preferir Python.
6. **`os.path.expanduser('~')` en Windows**. Resuelve a `C:\Users\<user>`, no a donde Hermes guarda su config (`AppData\Local\hermes`).
7. **Confiar en el footer de la TUI (`⚕ claude-opus-4.6`)**. Es un placeholder decorativo — NO refleja el modelo real en uso. Para verificar qué modelo se está usando realmente, ejecutar `hermes chat -q "¿qué modelo eres?" -m <modelo> --provider custom -Q`.
8. **Archivo `.env` corrupto con keys concatenadas**. Si el `.env` tiene líneas como `DEEPSEEK_API_KEY=sk-xxxsk-xxxsk-xxx...` (keys repetidas sin separador, >100 caracteres) o falta `OPENAI_API_KEY` por completo (`grep -c "OPENAI_API_KEY"` devuelve 0), el archivo está corrupto. Solución: `sed -i '/DEEPSEEK_API_KEY=/d'` para eliminar la línea corrupta, y restaurar `OPENAI_API_KEY=sk-<key>` en la sección DeepSeek (~línea 17).
9. **Editar el `.env` equivocado mientras Hermes lee de otro**. Si tras cambiar la key el error HTTP 401 sigue mostrando la key antigua, Hermes está leyendo de otra de las 3 ubicaciones posibles. Verificar las 3 ubicaciones (ver sección WSL → Las Tres Ubicaciones) y actualizar la que `hermes config env-path` reporta desde dentro de WSL.
10. **`provider: deepseek` (nativo) ≠ `provider: custom` — variables de entorno distintas**. Son providers diferentes que esperan variables diferentes en `.env`:
    - `provider: deepseek` (nativo) → busca `DEEPSEEK_API_KEY`
    - `provider: custom` → busca `OPENAI_API_KEY`
    **Síntoma**: si `config.yaml` tiene `provider: deepseek` pero el `.env` solo tiene `OPENAI_API_KEY`, el error será HTTP 401 (`Authentication Fails, Your api key: ****ired is invalid`) y el log mostrará `Provider: deepseek`. La key que aparece censurada (***ired) NO es la key correcta — es basura o una key vacía que el provider nativo intenta usar.
    **Solución**: decidir uno de los dos esquemas y ser consistente. Recomendado: `provider: custom` + `OPENAI_API_KEY=sk-...` (mismo esquema que Windows nativo). Alternativa: `provider: deepseek` + `DEEPSEEK_API_KEY=sk-...`. Nunca mezclar: `provider: deepseek` con `OPENAI_API_KEY` no funciona.
    **Detección rápida**: `grep "provider:" $(hermes config path)` — si dice `deepseek` y `grep -c "DEEPSEEK_API_KEY" $(hermes config env-path)` devuelve 0, hay mismatch.

## Bundled Scripts

- `scripts/test-deepseek-connectivity.py` — test de conectividad standalone para DeepSeek. Lee la key del `.env` y lista los modelos disponibles.

## Verification Checklist

- [ ] `hermes config env-path` → el archivo existe y tiene `OPENAI_API_KEY=sk-...`
- [ ] `hermes config` → `model.provider: custom`, `model.base_url` correcto
- [ ] `hermes doctor` → "✓ API key or custom endpoint configured"
- [ ] Test de conectividad con Python exitoso (lista de modelos)
- [ ] **Test de inferencia**: `hermes chat -q "di qué modelo eres" -m <modelo> --provider custom -Q` → confirma el modelo correcto
- [ ] **En WSL**: `hermes config path` coincide con `$HERMES_HOME`, no con `~/.hermes/`
- [ ] Reiniciar Hermes después de cualquier cambio en `.env` o `config.yaml`
