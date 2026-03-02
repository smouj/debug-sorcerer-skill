---
name: debug-sorcerer
version: 1.0.0
description: Asistente de depuración impulsado por IA que rastrea errores y genera correcciones para JavaScript, TypeScript, Python y Go.
author: FlickClaw Engineering
homepage: https://kilocode.ai/docs/debug-sorcerer
tags:
  - debugging
  - ai
  - code-analysis
  - developer-tools
license: MIT
dependencies:
  - kilocode-cli>=1.0.0
  - python>=3.8
  - openai>=4.0.0 | anthropic>=0.5.0
  - git
---

# Debug Sorcerer

## Purpose

Debug Sorcerer es una habilidad CLI avanzada que aprovecha modelos de lenguaje grandes para depurar código de forma autónoma. Cuando ocurre un error, la habilidad captura el contexto completo (stack trace, logs, entorno), realiza un análisis semántico profundo de los archivos fuente relevantes, identifica la probable causa raíz y genera un parche mínimo para resolver el problema. Está diseñado para desarrolladores que desean acelerar la resolución de bugs en proyectos de JavaScript, TypeScript, Python y Go. La habilidad se integra con flujos de trabajo git existentes y suites de pruebas, asegurando que las correcciones sean seguras, reversibles y verificadas antes de aplicarse.

Casos de uso específicos:

- **Depuración de crashes**: Diagnosticar rápidamente errores en tiempo de ejecución como `TypeError`, `ReferenceError`, `undefined is not a function`, etc.
- **Análisis de pruebas inestables**: Cuando las pruebas fallan de forma intermitente, la habilidad captura el contexto de la falla y hipotetiza sobre race conditions o problemas de timing.
- **Rastreo de cuellos de botella de rendimiento**: Identificar patrones de código que causan lentitud (por ejemplo, re-renderizaciones innecesarias, llamadas bloqueantes).
- **Escaneo de vulnerabilidades de seguridad**: Detectar anti-patrones de seguridad comunes como entradas no validadas, secretos hardcodeados y uso inseguro de API.
- **Asistencia en code review**: Durante la revisión de PR, se puede solicitar a la habilidad que analice archivos cambiados específicamente para detectar posibles bugs.

## Alcance

La habilidad proporciona los siguientes comandos CLI:

1. `debug-sorcerer trace <error_message> [OPTIONS]`
   - Requerido: `error_message` (cadena o ruta de archivo con prefijo `@`, ej. `@error.txt`)
   - Opciones:
     - `--file <path>`: Ruta al archivo fuente donde ocurrió el error (si no es inferible)
     - `--line <number>`: Número de línea del error
     - `--context N`: Incluir N líneas antes y después de la línea de error (por defecto: 10)
     - `--project-root <dir>`: Anular la detección automática de raíz del proyecto (por defecto: directorio actual)
     - `--ai-model <model>`: Especificar modelo de IA (ej. `gpt-4`, `claude-3-opus-20240229`). Por defecto: configurado en ajustes globales.
   - Ejemplo: `debug-sorcerer trace "UnhandledPromiseRejectionWarning: TypeError: Cannot read property 'id' of undefined" --file src/controllers/user.js --line 45`

2. `debug-sorcerer analyze <file_path> [OPTIONS]`
   - Analiza un archivo para detectar posibles bugs, anti-patrones y vulnerabilidades.
   - Opciones:
     - `--checks <comma-separated>`: Limitar análisis a verificaciones específicas (ej. `null-safety,async,security`). Por defecto: todas activadas.
     - `--output <format>`: Formato de salida (`json`, `text`, `sarif`). Por defecto: `text`.
     - `--severity-threshold <level>`: Severidad mínima a reportar (`low`, `medium`, `high`, `critical`). Por defecto: `low`.
   - Ejemplo: `debug-sorcerer analyze src/utils/api.js --checks security --output json > api-issues.json`

3. `debug-sorcerer fix <issue_identifier> [OPTIONS]`
   - Genera un parche para un problema identificado previamente (de `trace` o `analyze`).
   - Opciones:
     - `--dry-run`: Mostrar diff sin aplicar cambios.
     - `--apply`: Aplicar corrección inmediatamente (usar con precaución).
     - `--auto-commit`: Después de verificación exitosa, commitear cambios automáticamente.
     - `--branch <name>`: Crear una nueva rama para la corrección. Por defecto: `debug-sorcerer/fix-<timestamp>`.
   - Ejemplo: `debug-sorcerer fix 7f3a2b --dry-run`

4. `debug-sorcerer test [OPTIONS]`
   - Ejecuta la suite de pruebas del proyecto con monitoreo de IA para capturar y diagnosticar fallas.
   - Opciones:
     - `--watch`: Monitorear continuamente cambios en archivos y volver a ejecutar pruebas.
     - `--auto-fix`: Intentar automáticamente corregir pruebas fallidas después de cada ejecución.
     - `--only-failing`: Solo ejecutar pruebas que fallaron en ejecución anterior.
     - `--reporter <name>`: Reportero de pruebas a usar (ej. `spec`, `dot`, `json`). Por defecto: predeterminado del proyecto.
   - Ejemplo: `debug-sorcerer test --watch --auto-fix`

5. `debug-sorcerer verify <fix_identifier>`
   - Verifica que una corrección aplicada previamente resuelva el problema original.
   - Opciones:
     - `--re-run-tests`: Forzar re-ejecución de todas las pruebas.
     - `--manual`: Pausar para pasos de validación manual.
   - Ejemplo: `debug-sorcerer verify 7f3a2b --re-run-tests`

6. `debug-sorcerer report <format> [OPTIONS]`
   - Genera un reporte comprehensivo de la sesión de depuración.
   - Formatos: `html`, `json`, `markdown`, `sarif`.
   - Opciones:
     - `--output <file>`: Escribir reporte a archivo en lugar de stdout.
     - `--include-diffs`: Incluir diffs de código en el reporte.
     - `--session <id>`: Generar reporte para una sesión específica (por defecto: la más reciente).
   - Ejemplo: `debug-sorcerer report html --include-diffs --output debug-session-2024-01-15.html`

7. `debug-sorcerer config [OPTIONS]`
   - Gestionar configuración de la habilidad.
   - Subcomandos: `get <key>`, `set <key> <value>`, `list`, `reset`.
   - Ejemplo: `debug-sorcerer config set ai-model gpt-4-turbo`

## Proceso de Trabajo

Cuando un usuario invoca `debug-sorcerer trace`:

1. **Recopilación de Contexto**:
   - La habilidad lee el mensaje de error. Si el error proviene de un archivo, carga el contenido del archivo (y líneas de contexto circundantes).
   - También escanea el proyecto en busca de archivos relacionados (imports, dependencias del módulo) para construir un grafo de llamadas.
   - Variables de entorno del proceso en ejecución son capturadas si están disponibles (ej. `NODE_ENV`, `DATABASE_URL`).

2. **Análisis Inicial con IA**:
   - Construye un prompt para el modelo de IA que contiene:
     - El mensaje de error y stack trace.
     - Los fragmentos de código fuente relevantes (incluyendo imports y funciones adyacentes).
     - Una solicitud para identificar la causa raíz y proponer 2-3 correcciones candidatas ordenadas por confianza.
   - El prompt incluye instrucciones para evitar sugerir cambios a bibliotecas o frameworks de terceros (solo código del usuario).
   - Ejemplo de prompt interno: `Dado el siguiente error en una app Node.js Express: ... Identifica el bug y propone correcciones.`

3. **Generación de Hipótesis**:
   - La IA retorna una respuesta estructurada (JSON) con campos: `root_cause`, `confidence`, `affected_lines`, `suggested_fix`, `side_effects`.
   - La habilidad parsea la respuesta y presenta la hipótesis principal al usuario en formato legible.
   - Si la confianza está por debajo de 0.7, también muestra hipótesis alternativas.

4. **Validación de la Corrección**:
   - Antes de presentar la corrección, la habilidad realiza una simulación usando parseo de Árbol de Sintaxis Abstracta (AST) para asegurar que el código sugerido es sintácticamente válido y que el código modificado sigue conformándose a las reglas ESLint/Prettier del proyecto.
   - También verifica que la corrección no introduzca nuevas variables undefined o rompa statements de import.

5. **Interacción con el Usuario**:
   - La habilidad pregunta al usuario: `¿Corrección propuesta para [problema]? [S]í, [N]o, [A]lternativa, [E]dición manual`. Si se establece `--auto`, auto-acepta si confianza > 0.85.
   - Si el usuario selecciona Alternativa, la habilidad obtiene la siguiente hipótesis y repite.
   - Si el usuario selecciona Edición manual, la habilidad abre el archivo en el editor configurado del usuario (ej. `$EDITOR`) con la línea problemática resaltada.

6. **Aplicación del Parche**:
   - Al aceptar, la habilidad crea una rama git (si no está ya en una rama) llamada `debug-sorcerer/fix-<timestamp>`.
   - Aplica el parche usando `git apply` o edición directa de archivo, preservando permisos y codificación del archivo.
   - Registra el metadata de la corrección en `.debug-sorcerer/fixes.json` incluyendo problema original, parche aplicado, modelo de IA usado y timestamp.

7. **Verificación**:
   - La habilidad ejecuta automáticamente la suite de pruebas usando el runner de pruebas del proyecto (detectado desde `package.json`, `pytest.ini`, `go.mod`, etc.).
   - Captura la salida de las pruebas. Si las pruebas pasan, marca la corrección como `verified`.
   - Si las pruebas fallan, intenta analizar el trace de falla para determinar si la corrección causó una regresión. Si es así, sugiere rollback o corrección alternativa.
   - En modo `--auto-commit`, hace commit con mensaje: `fix: <descripción auto-generada> [debug-sorcerer]` y abre un PR si está configurado.

8. **Reporte**:
   - Genera un log de sesión resumiendo los pasos, interacciones con IA y resultado final.
   - Opcionalmente produce un reporte detallado en el formato elegido.

## Reglas de Oro

1. **No Perder Datos**: Todas las modificaciones son precedidas por un git commit o stash. Si el proyecto no está bajo control de versiones, la habilidad falla con un error solicitando inicializar un repo git.
2. **Consentimiento del Usuario para Acciones Destructivas**: Commitear cambios, crear ramas o eliminar archivos requiere `--auto` explícito o confirmación manual.
3. **Respetar Convenciones del Proyecto**: La habilidad lee las configuraciones ESLint, Prettier y de formato del proyecto y asegura que el código generado coincida con el estilo existente.
4. **Principio de Cambio Mínimo**: Los parches solo deben cambiar líneas necesarias para corregir el bug. No refactorizar a menos que se solicite explícitamente via `--refactor`.
5. **Sin Secretos Externos**: La habilidad nunca registra o almacena API keys, passwords o connection strings. Enmascara dichos valores en mensajes de error antes de enviar a la IA.
6. **Reproducibilidad**: Cada corrección incluye un `debug-sorcerer-id` en el mensaje de commit y en un comentario cerca del cambio (si el lenguaje soporta comentarios) para trazabilidad.
7. **Escalada Gradual**: La habilidad comienza con análisis no invasivo. Solo después de aprobación del usuario escribe en archivos.
8. **Seguridad de Lenguaje**: Para lenguajes compilados (Go, Rust), la habilidad asegura que el código compila antes de aplicar la corrección.
9. **Conciencia de Rendimiento**: Evita analizar archivos enormes (>10K líneas) por defecto; requiere `--force` para anular.
10. **No Sobrescribir Correcciones Manuales**: Si un desarrollador corrige manualmente un bug después de que la IA haya sugerido algo, la habilidad lo detecta y evita esfuerzos duplicados.

## Ejemplos

**Ejemplo 1: Tracear un error de runtime desde un archivo de log**

```bash
$ debug-sorcerer trace "Error: Cannot find module 'axios' in src/services/api.js:12" --file src/services/api.js --line 12
```
Salida:
```
[Debug Sorcerer] Recopilando contexto...
[Debug Sorcerer] El error indica que Node.js no puede resolver el módulo 'axios'. Esto típicamente es causado por:
  1. El módulo no está instalado (dependencia faltante)
  2. El statement de import tiene un typo
  3. La ruta del archivo es incorrecta

Analizando src/services/api.js alrededor de la línea 12...
  10: import fetch from 'node-fetch';
  11: // import axios from 'axios';  // <- esta línea está comentada
  12: const apiClient = axios.create({ baseURL: process.env.API_URL });

Causa raíz: El módulo 'axios' no está instalado y la línea de import está comentada.
Confianza: 0.95

Corrección sugerida:
  Opción A: Instalar axios: `npm install axios` y descomentar la import.
  Opción B: Reemplazar uso de axios con node-fetch (si es apropiado).

¿Qué opción? [A/B] (por defecto A):
```
Si el usuario selecciona A, la habilidad ejecuta `npm install axios` y descomenta la línea 11, luego verifica ejecutando pruebas.

**Ejemplo 2: Analizar un archivo para problemas de seguridad**

```bash
$ debug-sorcerer analyze src/auth/password.js --checks security --output json > password-audit.json
```

Salida (password-audit.json):
```json
{
  "file": "src/auth/password.js",
  "issues": [
    {
      "severity": "high",
      "line": 23,
      "message": "Secreto hardcodeado detectado: 'super-secret-password'",
      "suggestion": "Usar variable de entorno: process.env.ADMIN_PASSWORD"
    },
    {
      "severity": "medium",
      "line": 45,
      "message": "Algoritmo de hashing débil: MD5",
      "suggestion": "Usar bcrypt o Argon2 en su lugar"
    }
  ]
}
```

**Ejemplo 3: Auto-corrección de pruebas fallidas en modo watch**

```bash
$ debug-sorcerer test --watch --auto-fix
```
La habilidad ejecuta `npm test`. Supongamos que una prueba falla con "Expected 200 but got 500". La habilidad:
- Identifica el archivo de prueba fallido.
- Analiza la prueba y la implementación.
- Sugiere agregar un null check.
- Pregunta: "Corrección detectada: agregar guard en src/controllers/user.js:27. ¿Aplicar? [S/n]" (auto-aceptado si `--auto`).
- Aplica corrección y re-ejecuta pruebas. Si pasan, continúa vigilando.

## Comandos de Rollback

Si una corrección introduce una regresión o es insatisfactoria, el usuario puede hacer rollback:

1. **Rollback de una corrección específica**:
   ```bash
   $ debug-sorcerer rollback <fix_id>
   ```
   donde `<fix_id>` es el identificador mostrado después de aplicar una corrección (ej. `20240115-143022`). Esto revierte los cambios hechos por esa corrección, usando git si está disponible o restaurando desde backup.

2. **Rollback de sesión completa**:
   ```bash
   $ debug-sorcerer rollback --session <session_id>
   ```
   Esto deshace todas las correcciones aplicadas durante una sesión de depuración particular.

3. **Rollback manual con git** (si prefieres):
   ```bash
   $ git log --oneline --grep="\[debug-sorcerer\]"  # encontrar commit(s)
   $ git revert <commit_sha>   # revertir una corrección
   $ git reset --hard HEAD~N   # revertir últimos N commits (si no se han pusheado)
   ```

4. **Limpieza**:
   La habilidad también ofrece `debug-sorcerer cleanup` para remover archivos temporales y ramas más antiguas que un umbral. Por defecto mantiene 30 días.

## Pasos de Verificación

Después de aplicar una corrección, verifica que sea correcta:

1. **Re-ejecutar el escenario exacto** que disparó el error. Si el error ya no aparece, procede al paso 2.
2. **Ejecutar la suite completa de pruebas**:
   ```bash
   $ npm test   # o pytest, go test, etc.
   ```
   Asegurar que todas las pruebas pasan, especialmente pruebas de regresión.
3. **Realizar testing funcional manual** de la característica afectada para confirmar que el comportamiento es el esperado.
4. **Ejecutar `debug-sorcerer verify <fix_id>`** para que la habilidad automáticamente re-ejecute pruebas y verifique errores residuales.
5. **Revisar métricas** (si aplica) como rendimiento, uso de memoria, etc. para asegurar que no hay degradación.

Si alguna verificación falla, inmediatamente hacer rollback y reportar el problema a los mantenedores de la habilidad (por defecto abre un issue en GitHub).

## Solución de Problemas

| Problema | Causa Probable | Solución |
|----------|----------------|----------|
| `Error: No AI API key configured` | Falta `OPENAI_API_KEY` o `ANTHROPIC_API_KEY`. | Establecer la variable de entorno o ejecutar `debug-sorcerer config set api-key <your_key>`. |
| `Error: Project not a git repository` | La habilidad necesita git para seguridad pero no se encontró `.git`. | Inicializar git: `git init` y commitear estado actual. O usar `--no-git` (no recomendado). |
| `AI model quota exceeded` | Límites de API alcanzados. | Esperar o actualizar tu plan. Cambiar modelo con `debug-sorcerer config set ai-model gpt-3.5-turbo` (más barato). |
| `Analysis too slow on large file` | Archivo excede 5000 líneas por defecto. | Usar `--force` para analizar de todos modos, o dividir archivo. Considerar escribir un unit test para aislar la parte problemática. |
| `Fix conflicts with existing changes` | El archivo fue editado después de capturar el trace. | Stash de cambios antes de ejecutar fix, o resolver conflictos de merge manualmente. La habilidad lo detectará y abortará. |
| `Unsupported language` | La extensión del archivo no es reconocida. | Asegurar que el archivo sea uno de los lenguajes soportados (js, ts, py, go, rs). Contribuir una extensión si falta. |
| `Test runner not detected` | La habilidad no puede encontrar el script de prueba. | Especificar con `--test-command "pytest -v"` o setear `debug-sorcerer.config.testCommand`. |
| `False positive root cause` | IA malinterpretó el código. | Usar `--manual` para proporcionar pistas, o ajustar el prompt via `config set prompt-override`. Reportar el problema con el contexto. |

## Dependencias

- **Kilocode CLI**: Requerido para ejecutar la habilidad. Instalar con `npm install -g @kilocode/cli` o vía Homebrew.
- **Python**: El runtime principal de la habilidad está en Python 3.8+ (requerido para algo de manipulación AST). Asegurar que `python3` esté en PATH.
- **AI Provider**: Ya sea OpenAI API (`openai>=4.0.0`) o Anthropic API (`anthropic>=0.5.0`). Se debe setear una API key en `OPENAI_API_KEY` o `ANTHROPIC_API_KEY`.
- **Git**: Recomendado para seguridad de versionado. Si no está presente, la habilidad opera en modo solo-lectura (no se aplican correcciones).
- **Node.js / npm / yarn / pnpm**: Para proyectos JavaScript/TypeScript, el gestor de paquetes apropiado debe estar disponible para instalar dependencias faltantes sugeridas por la habilidad.
- **Herramientas específicas de lenguaje**: Para Python, `pytest` o `tox`; para Go, `go test`; para Rust, `cargo test`. Estas deben estar instaladas y en PATH.

## Variables de Entorno

- `DEBUG_SORCERER_API_KEY`: Anular la API key de IA. Si está seteada, tiene prioridad sobre config global.
- `DEBUG_SORCERER_LOG_LEVEL`: Controla verbosidad de logging. Opciones: `DEBUG`, `INFO`, `WARNING`, `ERROR`. Por defecto: `INFO`.
- `DEBUG_SORCERER_WORKDIR`: Directorio raíz del proyecto. Por defecto: directorio de trabajo actual.
- `DEBUG_SORCERER_AI_MODEL`: Modelo de IA por defecto a usar. Sobreescribe config. Ejemplo: `gpt-4-turbo-preview`.
- `DEBUG_SORCERER_NO_GIT`: Si se setea a `true`, deshabilita checks de seguridad git (usar con precaución).
- `DEBUG_SORCERER_DRY_RUN`: Si se setea a `true`, nunca aplicar cambios automáticamente; siempre preguntar.
- `DEBUG_SORCERER_TIMEOUT`: Timeout en segundos para requests a IA. Por defecto: `60`.

Opcional: `DEBUG_SORCERER_DISABLE_TELEMETRY` para opt-out de estadísticas de uso.
```