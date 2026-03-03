# Práctica 1

## Script 1: `log_reporter_grep.sh`
### Descripción del Funcionamiento

El script opera mediante un flujo lógico dividido en cuatro funciones principales:

#### 1. Interfaz y Selección (`greeting` y `logSelector`)
El script despliega un menú numérico que permite al usuario elegir el nivel deseado (`INFO`, `WARN`, `ERROR`, `DEBUG`). Utiliza un ciclo `while` para validar que la entrada sea correcta y una estructura `case` para asignar el nivel a una variable global.

#### 2. Filtrado con Expresiones Regulares (`read_log`)
Utiliza el motor de **Regex Extendida** de `grep` para validar la estructura de cada línea. Así como la bandera `-E` para poder usar cuantificadores como `{n}` y `+`.
- **Patrón**: `\[$LOG_LEVEL\] [0-9]{4}-[0-9]{2}-[0-9]{2} [0-9]{2}:[0-9]{2}:[0-9]{2} .+`
- **Acción**: Solo las líneas que inician exactamente con el nivel seleccionado, seguido de una fecha (AAAA-MM-DD) y una hora (HH:MM:SS) válida, son extraídas y guardadas en un archivo llamado `${LOG_LEVEL}_validos.txt`.
- **Descripción de la Expresión Regular**
  - `\[$LOG_LEVEL\]`: Busca el nivel seleccionado rodeado de corchetes literales. El uso de `\` es vital para que `[` no se interprete como un set de caracteres-
  - `[0-9]{4}-[0-9]{2}-[0-9]{2}`: Valida el formato de fecha `AAAA-MM-DD`. El `{n}` indica la cantidad exacta de dígitos.
  - `[0-9]{2}:[0-9]{2}:[0-9]{2}`: Valida el formato de tiempo `HH:MM:SS`.
  - `.+`: Asegura que después de la estampa de tiempo exista, al menos, un carácter de mensaje (evita logs vacíos).

#### 3. Análisis de Métricas (`generate_metrics`)
El script calcula tres datos clave mediante el conteo de líneas (`grep -c`):
* **Líneas no vacías**: Filtra todas las líneas que contienen al menos un carácter.
  * `-v`: Invierte la selección (selecciona lo que NO coincida).
  * `^$`: Representa una línea totalmente vacía (inicio de línea seguido inmediatamente por fin de línea).
* **Líneas válidas**: Cuenta los registros que se hay en el archivo [LOG_LEVEL]_validos.txt.
* **Líneas sospechosas**: Utiliza una **tubería (pipe)** para encontrar líneas que mencionan el nivel seleccionado pero que **no** cumplen con el formato de fecha/hora. Esto ayuda a detectar errores de escritura en el log.
  * El primer grep extrae todas las líneas que mencionen el nivel selecionado por el usuario.
  * Se usa un pipe `|` para usar la salida del primer grep como entrada del segundo.
  * El segundo grep usa `-v` para descartar las líneas que sigues el formato deseado.

#### 4. Generación de Reporte JSON (`create_json`)
Para finalizar, el script utiliza el comando `printf` con una plantilla estructurada para generar un archivo `reporte_log.json`. Este método garantiza que el archivo de salida sea legible y respete los tipos de datos (strings para texto y números sin comillas para los contadores).

## Script 2: `log_reporte_re.py`
### Descripción del Funcionamiento

Este script traslada la lógica de filtrado de logs a Python, utilizando el módulo `re` para el manejo de expresiones regulares y proporcionando una interfaz interactiva.

#### 1. Interfaz y Selección (`greeting` y `logSelector`)
El script implementa un bucle `while` que presenta un menú interactivo para el usuario. 
- Utiliza el método `.isdigit()` para validar que la entrada sea numérica antes de procesarla.
- Emplea la estructura `match` (disponible en Python 3.10+) para mapear la opción del usuario al nivel de log correspondiente: `INFO`, `WARN`, `ERROR` o `DEBUG`.

#### 2. Filtrado con el módulo `re` (`readLog`)
A diferencia de las herramientas de shell, el script utiliza la potencia de las f-strings de Python para construir patrones.
- **Patrón**: `rf"\[{log_level}\] [0-9]{{4}}-[0-9]{{2}}-[0-9]{{2}} [0-9]{{2}}:[0-9]{{2}}:[0-9]{{2}} .+"`.
- **Acción**: Se utiliza `re.search()` para identificar las líneas que cumplen con el formato de estampa de tiempo y nivel de log, guardándolas en un archivo de texto dentro del directorio `out`.



#### 3. Análisis de Métricas (`generateMetrics`)
El script realiza un análisis profundo del archivo original para calcular tres estadísticas clave:
* **Líneas no vacías**: Cuenta cada registro que contiene caracteres después de aplicar `.strip()`.
* **Líneas válidas**: Suma los registros que fueron filtrados exitosamente por la expresión regular.
* **Líneas sospechosas**: Identifica líneas que mencionan el nivel de log (ej. `[INFO]`) pero que **no** cumplen con el formato de fecha/hora esperado, detectando posibles corrupciones de datos.

#### 4. Generación de Reporte JSON (`createJson`)
Para finalizar, el script utiliza la librería `json` para exportar los resultados en un formato estándar de intercambio de datos.
- El reporte incluye metadatos como la fecha actual del sistema (vía `datetime.now()`) y el desglose de las métricas en un objeto anidado.
- Se utiliza `indent=4` para garantizar que el archivo `reporte_log.json` sea legible.

## Script 3: `password_validator_grep.sh`

Antes de iniciar el procesamiento, el script limpia los archivos de salida (`validas.txt` e `invalidas.txt`) utilizando el redireccionamiento `>`. Esto asegura que los reportes de ejecuciones anteriores no se mezclen con los nuevos resultados.

#### 1. Procesamiento y Reglas de Validación
El script lee el archivo de entrada línea por línea mediante un ciclo `while` y aplica cuatro filtros de seguridad utilizando el operador de comparación de cadenas y expresiones regulares de Bash (`[[ $var =~ regex ]]`). El operador `=~` compara la cadena de la izquierda con el patrón regex de la derecha, devolviendo si hay coincidencias:

* **Caracteres Permitidos**: Verifica que la contraseña contenga **únicamente** caracteres alfanuméricos. 
    * **Regex**: `[^a-zA-Z0-9]` (Busca cualquier carácter que NO sea una letra o número).
* **Longitud Mínima**: Utiliza la expansión de parámetros `${#pass}` para comprobar que la cadena tenga al menos 8 caracteres.
* **Presencia de Mayúsculas**: Valida que exista al menos una letra mayúscula.
    * **Regex**: `[A-Z]`
* **Presencia de Dígitos**: Valida que exista al menos un número.
    * **Regex**: `[0-9]`

#### 2. Clasificación y Reporte
Dependiendo del cumplimiento de las reglas, el script realiza lo siguiente:
* **Si es válida**: La contraseña se añade a `validas.txt` y se incrementa el contador `val_count`.
* **Si es inválida**: La contraseña se añade a `invalidas.txt` junto con una cadena de texto (`reasons`) que especifica exactamente qué reglas fallaron (ej. "longitud insuficiente", "no tiene mayúscula"), y se incrementa el contador `inv_count`.

#### 3. Resumen de Ejecución
Al finalizar el análisis, el script imprime en la terminal un resumen visual que indica la cantidad total de contraseñas procesadas con éxito y la ubicación exacta de los archivos generados.

## Script 4: `password_validator_re.py`
### Descripción del Funcionamiento

Este script automatiza la auditoría de seguridad de contraseñas mediante el uso de expresiones regulares y lógica condicional acumulativa.

#### 1. Preparación y Manejo de Archivos
El script define constantes para las rutas de entrada y salida, asegurando la creación del directorio `out` mediante `os.makedirs()` para evitar errores de ejecución. Utiliza el manejo de contextos (`with open`) para gestionar de forma segura múltiples archivos simultáneamente.

#### 2. Reglas de Validación con Regex
Cada contraseña es evaluada bajo cuatro criterios de seguridad independientes. Si una regla falla, se concatena el motivo en la variable `reason`:

* **Caracteres Válidos**: `re.search(r'[^a-zA-Z0-9]', password)` busca cualquier carácter que NO sea una letra o número para marcarlo como inválido.
* **Longitud Mínima**: Se verifica que la cadena tenga al menos 8 caracteres con `len(password) < 8`.
* **Presencia de Mayúsculas**: Se utiliza el patrón `[A-Z]` para confirmar la existencia de al menos una letra mayúscula.
* **Presencia de Dígitos**: Se utiliza el patrón `[0-9]` para asegurar que la contraseña incluya al menos un número.



#### 3. Clasificación y Reporte Detallado
Dependiendo del resultado de la validación, la contraseña se clasifica en uno de dos reportes:
* **Válidas**: Si la variable `reason` permanece vacía, la contraseña se escribe en `validas.txt`.
* **Inválidas**: Si se detectó algún fallo, la contraseña se escribe en `invalidas.txt` acompañada de los motivos específicos encontrados (ej. "longitud insuficiente no tiene mayúscula").

#### 4. Resumen Final
Al terminar el ciclo, el script imprime en la terminal el conteo total de contraseñas procesadas (`val_count` e `inv_count`) y confirma las rutas donde se guardaron los resultados finales.
