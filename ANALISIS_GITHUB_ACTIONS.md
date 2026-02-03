# Análisis de GitHub Actions en el Proyecto

## Descripción General del Proyecto

Este es un repositorio educativo diseñado para enseñar CodeQL, una herramienta de análisis estático que ayuda a identificar vulnerabilidades de seguridad en el código. El proyecto utiliza GitHub Actions para crear una experiencia de aprendizaje interactiva y progresiva.

## Estructura del Proyecto

```
skills-introduction-to-codeql/
├── .github/
│   ├── workflows/           # Workflows de GitHub Actions
│   │   ├── 0-start-exercise.yml
│   │   ├── 1-step.yml
│   │   ├── 2-step.yml
│   │   ├── 3-step.yml
│   │   └── 4-step.yml
│   └── steps/              # Contenido de cada paso
│       ├── 1-step.md
│       ├── 2-step.md
│       ├── 3-step.md
│       ├── 4-step.md
│       └── x-review.md
├── server/                 # Aplicación Flask con vulnerabilidad intencional
│   ├── routes.py
│   ├── webapp.py
│   └── models/
└── templates/             # Templates HTML
```

## Arquitectura de GitHub Actions

### 1. Workflow: Step 0 - Iniciar Ejercicio (`0-start-exercise.yml`)

**Propósito**: Inicializar el ejercicio creando un issue de seguimiento y publicando las instrucciones del primer paso.

**Trigger**: 
```yaml
on:
  push:
    branches:
      - main
```
Se activa cuando se hace push a la rama `main`.

**Permisos Requeridos**:
- `contents: write` - Para modificar el repositorio
- `actions: write` - Para habilitar/deshabilitar workflows
- `issues: write` - Para crear y comentar en issues

**Flujo de Trabajo**:

1. **Job: `start_exercise`**
   - Utiliza un workflow reutilizable de `skills/exercise-toolkit`
   - Crea un issue de seguimiento para el ejercicio
   - Título: "Introduction to CodeQL"
   - Mensaje: "Learn to use CodeQL to find security vulnerabilities in your code."

2. **Job: `post_next_step_content`**
   - Depende de: `start_exercise`
   - Pasos:
     - Hace checkout del repositorio
     - Clona el toolkit de ejercicios
     - Publica el contenido del Paso 1 (`.github/steps/1-step.md`) como comentario en el issue
     - Publica mensaje de "observando progreso"
     - **Habilita el workflow "Step 1"** usando GitHub CLI

**Variables de Entorno**:
- `STEP_1_FILE`: Ruta al archivo markdown del paso 1
- `ISSUE_NUMBER`: Número del issue creado
- `ISSUE_REPOSITORY`: Nombre del repositorio

---

### 2. Workflow: Step 1 - Habilitar Code Scanning (`1-step.yml`)

**Propósito**: Detectar cuando el usuario habilita CodeQL y avanzar al siguiente paso.

**Trigger**:
```yaml
on:
  workflow_run:
    workflows: [CodeQL]
    types:
      - in_progress
```
Se activa cuando el workflow de CodeQL entra en estado `in_progress`. Esto ocurre cuando el usuario habilita CodeQL en la configuración de seguridad del repositorio.

**Permisos Requeridos**:
- `contents: read` - Para leer el repositorio
- `actions: write` - Para gestionar workflows
- `issues: write` - Para comentar en issues

**Flujo de Trabajo**:

1. **Job: `find_exercise`**
   - Busca el issue del ejercicio usando el toolkit
   - Retorna el número y URL del issue

2. **Job: `post_next_step_content`**
   - Depende de: `find_exercise`
   - Pasos:
     - Publica mensaje de "paso completado"
     - Publica el contenido del Paso 2 (`.github/steps/2-step.md`)
     - Publica mensaje de "observando progreso"
     - **Deshabilita el workflow actual (Step 1)**
     - **Habilita el workflow "Step 2"**

**Instrucciones del Paso 1**:
El usuario debe habilitar Code Scanning con CodeQL en la configuración de seguridad del repositorio, lo que activa automáticamente este workflow.

---

### 3. Workflow: Step 2 - Detectar Vulnerabilidades (`2-step.yml`)

**Propósito**: Guiar al usuario en la creación de una vulnerabilidad intencional y observar cómo CodeQL la detecta en un Pull Request.

**Trigger**:
```yaml
on:
  workflow_run:
    workflows: [CodeQL]
    types:
      - in_progress
```
Idéntico al Step 1, se activa cuando CodeQL se ejecuta.

**Flujo de Trabajo**: Idéntico a Step 1, pero publica el contenido del Paso 3.

**Instrucciones del Paso 2**:
El usuario debe:
1. Modificar `server/routes.py` línea 16 para introducir una vulnerabilidad SQL injection:
   ```python
   "SELECT * FROM books WHERE name LIKE '%" + name + "%'"
   ```
2. Crear un Pull Request en una nueva rama `learning-codeql`
3. Observar cómo CodeQL detecta la vulnerabilidad en el PR
4. Revisar los logs del workflow CodeQL en la pestaña Actions

---

### 4. Workflow: Step 3 - Revisar y Gestionar Alertas (`3-step.yml`)

**Propósito**: Validar que el usuario ha interactuado con las alertas de CodeQL mediante comentarios específicos.

**Trigger**:
```yaml
on:
  issue_comment:
    types: [created, edited]
```
Se activa cuando se crea o edita un comentario en cualquier issue.

**Permisos Requeridos**:
- `contents: read`
- `actions: write`
- `issues: write`

**Flujo de Trabajo**:

1. **Job: `check_keywords`**
   - Busca palabras clave específicas en el comentario:
     - `"professortocat"` (case-insensitive)
     - `"alert"` (case-insensitive)
   - Utiliza la acción `skills/action-keyphrase-checker@v1`
   - Combina ambas verificaciones: **ambas** deben tener éxito
   - Output: `result: success` o `result: fail`

2. **Job: `find_exercise`**
   - Depende de: `check_keywords`
   - Condición: Solo se ejecuta si `result == 'success'`
   - Busca el issue del ejercicio

3. **Job: `post_next_step_content`**
   - Similar a pasos anteriores
   - Publica el contenido del Paso 4
   - Deshabilita Step 3 y habilita Step 4

**Instrucciones del Paso 3**:
El usuario debe:
1. Hacer merge del PR con la vulnerabilidad
2. Ver la alerta en la pestaña Security > Code scanning
3. Revisar los detalles de la alerta
4. Cerrar (dismiss) la alerta con razón "Used in tests"
5. Reabrir la alerta
6. Comentar en el issue con las palabras clave: `"Hey @professortocat, I've closed and reopened an alert. What is the next step?"`

---

### 5. Workflow: Step 4 - Corregir Vulnerabilidades (`4-step.yml`)

**Propósito**: Detectar que el usuario ha corregido la vulnerabilidad y finalizar el ejercicio.

**Trigger**:
```yaml
on:
  workflow_run:
    workflows: [CodeQL]
    types:
      - completed
```
Se activa cuando el workflow de CodeQL se completa exitosamente.

**Permisos Requeridos**:
- `contents: write`
- `actions: write`
- `issues: write`

**Flujo de Trabajo**:

1. **Job: `find_exercise`**
   - Busca el issue del ejercicio

2. **Job: `post_review_content`**
   - Depende de: `find_exercise`
   - Pasos:
     - Publica mensaje de "revisión final"
     - Publica el contenido de revisión (`.github/steps/x-review.md`)
     - Deshabilita el workflow Step 4

3. **Job: `finish_exercise`**
   - Depende de: `find_exercise`, `post_review_content`
   - Utiliza workflow reutilizable para finalizar el ejercicio
   - Cierra el issue de seguimiento

**Instrucciones del Paso 4**:
El usuario debe corregir la vulnerabilidad en `routes.py` línea 16:
```python
"SELECT * FROM books WHERE name LIKE %s", name
```
CodeQL detectará que la vulnerabilidad fue corregida y cerrará automáticamente la alerta.

---

## Componentes Clave de GitHub Actions Utilizados

### 1. Workflows Reutilizables
```yaml
uses: skills/exercise-toolkit/.github/workflows/start-exercise.yml@v0.7.1
```
- Permite reutilizar lógica compleja
- Facilita el mantenimiento
- Proporciona funcionalidad estandarizada

### 2. Outputs entre Jobs
```yaml
needs: [start_exercise]
env:
  ISSUE_NUMBER: ${{ needs.start_exercise.outputs.issue-number }}
```
- Permite pasar datos entre jobs
- Crea dependencias y orden de ejecución

### 3. Conditional Execution
```yaml
if: |
  !github.event.repository.is_template
```
```yaml
if: needs.check_keywords.outputs.result == 'success'
```
- Controla qué jobs se ejecutan según condiciones
- Valida el progreso del usuario

### 4. GitHub CLI en Actions
```yaml
- name: Enable next step workflow
  run: |
    gh workflow enable "Step 1"
  env:
    GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```
- Permite automatizar operaciones de GitHub
- Gestión dinámica de workflows

### 5. Acciones de Terceros
- `actions/checkout@v4`: Clona el repositorio
- `GrantBirki/comment@v2.1.1`: Crea comentarios en issues
- `skills/action-keyphrase-checker@v1`: Valida palabras clave en texto

### 6. Triggers Diversos
- `push`: Eventos de push a ramas
- `workflow_run`: Reacciona a otros workflows
- `issue_comment`: Reacciona a comentarios en issues

---

## Flujo Completo del Ejercicio

```
1. Usuario hace push a main
   └─> Activa "Step 0"
       └─> Crea issue con instrucciones
           └─> Habilita "Step 1"

2. Usuario habilita CodeQL en Settings
   └─> CodeQL workflow comienza (in_progress)
       └─> Activa "Step 1"
           └─> Publica siguiente paso
               └─> Habilita "Step 2"

3. Usuario crea PR con vulnerabilidad
   └─> CodeQL analiza el PR (in_progress)
       └─> Activa "Step 2"
           └─> Publica siguiente paso
               └─> Habilita "Step 3"

4. Usuario hace merge y comenta en issue
   └─> issue_comment event
       └─> "Step 3" verifica palabras clave
           └─> Si correctas: publica siguiente paso
               └─> Habilita "Step 4"

5. Usuario corrige la vulnerabilidad
   └─> CodeQL completa análisis (completed)
       └─> Activa "Step 4"
           └─> Publica revisión final
               └─> Cierra el ejercicio
```

---

## Ventajas de esta Arquitectura

1. **Progresión Automática**: Los workflows se habilitan/deshabilitan automáticamente según el progreso
2. **Feedback Inmediato**: Los comentarios en issues guían al usuario en tiempo real
3. **Validación Robusta**: Verifica que el usuario complete cada paso correctamente
4. **Experiencia Interactiva**: Combina acciones del usuario con automatización
5. **Modular y Mantenible**: Cada paso es independiente y fácil de modificar
6. **Reutilización**: Usa workflows compartidos del toolkit de ejercicios

---

## Permisos y Seguridad

Todos los workflows requieren permisos específicos para:
- **contents**: Leer y/o escribir en el repositorio
- **actions**: Habilitar/deshabilitar workflows
- **issues**: Crear y comentar en issues

Estos permisos son necesarios para la automatización pero están limitados al contexto del repositorio, siguiendo el principio de mínimo privilegio.

---

## Integración con CodeQL

El proyecto depende del workflow de CodeQL (configurado automáticamente por GitHub cuando se habilita Code Scanning) que:

1. Se ejecuta en eventos de push y pull_request
2. Analiza el código Python en busca de vulnerabilidades
3. Publica resultados en la pestaña Security
4. Comenta en Pull Requests con alertas encontradas
5. Actualiza automáticamente el estado de las alertas

Los workflows de pasos reaccionan a los estados del workflow CodeQL:
- **in_progress**: Steps 1 y 2
- **completed**: Step 4

---

## Conclusión

Este proyecto demuestra un uso avanzado de GitHub Actions para crear una experiencia de aprendizaje guiada. La arquitectura utiliza:

- **Workflows encadenados** que se activan secuencialmente
- **Validación de progreso** mediante análisis de eventos y contenido
- **Automatización inteligente** que habilita/deshabilita pasos
- **Feedback contextual** mediante comentarios automatizados
- **Integración profunda** con las características de seguridad de GitHub

El resultado es una experiencia educativa interactiva que enseña CodeQL de manera práctica, guiando al usuario paso a paso mientras valida automáticamente su progreso.
