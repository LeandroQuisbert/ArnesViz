## VERSIÓN 1.1 – DOCUMENTACIÓN DE ARQUITECTURA DEL VISUALIZADOR DE ARNÉS

---

### 1. RESUMEN Y PROPÓSITO DEL SISTEMA

1.1. Este visualizador modela el arnés de una moto eléctrica como un grafo de **componentes físicos** (contenedores, conectores) unidos por **relaciones** (cables fijos, acoples enchufables) y organizados mediante **señales lógicas** (nets).  

1.2. Permite mover elementos respetando la jerarquía de contención, visualizar cables flexibles y validar automáticamente la coherencia de géneros, pines y señales.

**Memoria de diseño – Sección 1**  
Se mantiene el propósito original del MVP: ofrecer una vista interactiva del arnés con restricciones físicas realistas. La separación entre componentes, relaciones y señales sigue el patrón modelo-vista, donde los componentes tienen presencia gráfica, las relaciones definen vínculos y los nets permiten validación eléctrica. La decisión de mantener estos tres pilares se tomó para no mezclar responsabilidades y facilitar futuras ampliaciones como simulación de continuidad.

---

### 2. SISTEMA DE IDENTIFICADORES (ÍNDICE DE ENTIDADES)

2.1. Cada entidad del sistema recibe un **ID único** compuesto por una letra de tipo más un número de tres dígitos. La siguiente tabla actúa como índice general de todas las entidades modeladas.

**Tabla 1 – Índice de entidades y prefijos de identificación**

| Prefijo | Entidad                  | Rango numérico       | Ejemplo |
|---------|--------------------------|----------------------|---------|
| T       | Contenedor (system, enclosure, pcb) | 100 – 399 (subdividido) | T100, T200, T300 |
| C       | Conector (connector)     | 001 – 999            | C001    |
| W       | Cable fijo (wired)       | 001 – 999            | W001    |
| M       | Acople enchufable (mated)| 001 – 999            | M001    |
| N       | Señal lógica (net)       | 001 – 999            | N001    |

2.2. La letra **`T`** agrupa todos los contenedores (system, enclosure, pcb) bajo un mismo prefijo. La subdivisión del rango numérico indica el nivel jerárquico:  
 2.2.1. `1xx` → sistema raíz (system).  
 2.2.2. `2xx` → caja (enclosure).  
 2.2.3. `3xx` → placa de circuito (pcb).  

2.3. Esta organización permite hasta 99 contenedores de cada tipo (ej. T101, T201, T301…) y mantiene la coherencia conceptual: todos los contenedores comparten prefijo, pero su posición jerárquica es deducible del número.

2.4. El campo `type` (p. ej. `"system"`, `"enclosure"`, `"pcb"`) sigue presente en los datos para definir explícitamente el subtipo; el ID solo actúa como identificador único.

**Memoria de diseño – Sección 2**  
La tabla única de entidades facilita la consulta rápida y evita la redundancia de tener varias filas para el mismo prefijo. La subdivisión de `T` se explica a continuación, manteniendo la tabla limpia. Se conserva el campo `type` porque la letra `T` por sí sola no distingue entre sistema, caja o placa; el rango numérico da una pista, pero el tipo explícito es necesario para el procesamiento. La decisión de usar centenas como niveles jerárquicos se tomó para permitir un crecimiento muy holgado (en la práctica rara vez se superan 10 elementos por tipo). Conectores, cables, acoples y nets tienen cada uno su propio prefijo porque representan conceptos fundamentalmente diferentes.

---

### 3. CONTENEDORES (SYSTEM, ENCLOSURE, PCB)

Son los **contenedores** que definen la estructura física del arnés. No tienen género ni pines; su función es agrupar conectores y limitar su movimiento.

#### 3.1 Atributos comunes

**Tabla 2 – Atributos de contenedores**

| Campo       | Tipo         | Descripción |
|-------------|--------------|-------------|
| id          | string       | T100, T200, T300… |
| type        | string       | `"system"`, `"enclosure"`, `"pcb"` |
| name        | string       | Nombre descriptivo (ej. "Caja 1") |
| parent_id   | string / null| ID del contenedor padre (`null` solo para el sistema raíz) |
| designator  | string       | Etiqueta técnica (ej. "BOX1", "PCB1") |
| position    | object       | `{ x, y, width, height }` en píxeles |

#### 3.2 Catálogo de contenedores

**Tabla 3 – Lista de contenedores del ejemplo**

| ID   | Type       | Nombre         | Padre | Designator | Posición (x, y, width, height) |
|------|------------|----------------|-------|------------|--------------------------------|
| T100 | system     | Moto Eléctrica | null  | –          | 50, 50, 2600, 1200 |
| T200 | enclosure  | Caja 1         | T100  | BOX1       | 140, 160, 950, 1000 |
| T201 | enclosure  | Caja 2         | T100  | BOX2       | 1643, 172, 950, 1000 |
| T300 | pcb        | PCB 1          | T200  | PCB1       | 220, 260, 368, 837 |
| T301 | pcb        | PCB 2          | T201  | PCB2       | 2070, 289, 472, 817 |

**Memoria de diseño – Sección 3**  
El término "contenedores" es más claro y directo que "componentes no conectores". Agrupa system, enclosure y pcb bajo un mismo concepto: elementos que contienen otros. La jerarquía de `parent_id` nulo solo en el sistema raíz evita componentes huérfanos y simplifica las comprobaciones de contención. Las posiciones son absolutas respecto al lienzo para facilitar el renderizado.

#### 3.3 Movimiento y redimensionamiento

##### 3.3.1 Movimiento de contenedores

3.3.1.1. Al arrastrar un contenedor (`system`, `enclosure` o `pcb`), todos sus descendientes (hijos directos e indirectos) se desplazan exactamente el mismo vector **(dx, dy) en tiempo real**, sin retardo.  
3.3.1.2. El contenedor arrastrado **no modifica su tamaño** ni ninguna otra propiedad; solo cambian sus coordenadas **X** e **Y**.  
3.3.1.3. Los conectores anclados (`edgeSide`) se reposicionan automáticamente sobre el borde correspondiente del padre **durante el mismo fotograma**, sin necesidad de ajustes posteriores.

**Memoria de diseño – 3.3.1**  
El movimiento solidario en tiempo real evita el parpadeo y la sensación de "arrastre por partes". Se decidió no cambiar el tamaño durante el arrastre porque esa operación tiene su propio modo (redimensionamiento con handles). La restricción de borde se aplica a cada frame para mantener la coherencia visual.

##### 3.3.2 Redimensionamiento

3.3.2.1. Los contenedores pueden redimensionarse arrastrando **cualquier esquina o borde** (handles en todo el perímetro).  
3.3.2.2. Durante el redimensionamiento:  
 a. Los **conectores anclados** ajustan su posición según la regla definida en **3.3.2.1**.  
 b. Los **componentes internos no anclados** (por ejemplo, conectores con `edgeSide: null` dentro de ese contenedor) conservan su distancia relativa al **centro geométrico** del contenedor.

###### 3.3.2.1 Comportamiento de conectores anclados durante el redimensionamiento

3.3.2.1.1. Un conector anclado **conserva su distancia respecto a la esquina más cercana del borde que no se está estirando**.  
3.3.2.1.2. **Ejemplo concreto** (borde derecho, estiramiento vertical hacia abajo):  
 a. Sea un conector en el borde derecho (`edgeSide: "right"`) de un PCB.  
 b. Si el PCB se estira **solo hacia abajo** (aumenta su altura, manteniendo fija la esquina superior izquierda), el borde derecho se alarga.  
 c. La **esquina superior derecha** no se ha movido, solo la inferior.  
 d. La distancia desde el **centro del conector hasta la esquina superior derecha** permanece constante.  
 e. En consecuencia, el conector se desplaza verticalmente **la misma cantidad** que el estiramiento, manteniéndose a la misma altura relativa respecto a la esquina superior.  
3.3.2.1.3. En general, para cualquier borde:  
 a. Si se estira un eje paralelo al borde (por ejemplo, estirar horizontalmente un contenedor con un conector en el borde superior), la distancia al extremo izquierdo (o derecho) que no se haya movido se mantiene.  
 b. Si el estiramiento es perpendicular al borde (cambia la profundidad del contenedor), el conector **no se mueve** porque el borde en sí no se desplaza, solo se ensancha.  
3.3.2.1.4. Esta regla reemplaza el antiguo concepto de “posición porcentual” y se traduce en un comportamiento más intuitivo para el usuario.

**Memoria de diseño – 3.3.2**  
El redimensionamiento desde cualquier borde se adoptó para dar flexibilidad total al usuario, a diferencia de la versión inicial que solo permitía la esquina inferior derecha. La regla de mantener distancia a la esquina fija se introdujo tras observar que el redimensionamiento porcentual provocaba desplazamientos inesperados cuando se estiraba un solo eje. Con esta regla, el comportamiento es predecible y coincide con la intuición física: un conector atornillado a una pared no cambia de posición respecto a la esquina más cercana si solo se alarga la pared opuesta.

---

### 4. CONECTORES

Los conectores son los puntos de conexión eléctrica. Cada uno tiene un género y pertenece a un contenedor físico.

#### 4.1 Atributos

**Tabla 4 – Atributos de conectores**

| Campo         | Tipo         | Descripción |
|---------------|--------------|-------------|
| id            | string       | C001, C002… |
| type          | string       | `"connector"` |
| name          | string       | Nombre descriptivo (ej. "Molex 2P") |
| parent_id     | string       | ID del contenedor donde está montado (T100, T200, T300…) |
| designator    | string       | Etiqueta técnica (ej. "J1") |
| pins          | number       | Cantidad de pines (2, 4, 5…) |
| gender        | string       | `"male"` o `"female"` (obligatorio) |
| edgeSide      | string / null| `"left"`, `"right"`, `"top"`, `"bottom"` o `null` |
| position      | object       | `{ x, y, width, height }` en píxeles |
| expectedPair  | string / null| ID del conector con el que está diseñado para acoplarse, o `null` si admite cualquier compatible |

4.1.1. El campo `edgeSide` define el tipo de fijación del conector:  
 a. **Anclado (anchored)**: cuando `edgeSide` toma uno de los valores `"left"`, `"right"`, `"top"` o `"bottom"`. El conector está fijo a ese borde del contenedor padre y solo puede deslizarse a lo largo del mismo o cambiarse a otro borde.  
 b. **Libre (free)**: cuando `edgeSide` es `null`. El conector se mueve libremente dentro de los límites de su contenedor, sin estar sujeto a ningún borde.

**Memoria de diseño – 4.1**  
El campo `expectedPair` sustituye al antiguo `lockedWith` persistente. La razón del cambio es que `lockedWith` representaba un estado (con quién está bloqueado ahora), mientras que `expectedPair` expresa una intención de diseño inmutable: este conector está pensado para enchufarse a ese otro, y solo a ese. Esto permite validar en tiempo de diseño que las conexiones son correctas y evita ambigüedades. El bloqueo real (`lockedWith`) se calcula dinámicamente a partir de los M activos, reflejando el estado actual del arnés.

#### 4.2 Género y validación

4.2.1. Todo conector tiene un género obligatorio (`male` o `female`).  
4.2.2. Los acoples **M** requieren que los dos conectores tengan géneros opuestos (uno macho y otro hembra). No importa cuál se asigna como `from` o `to`; lo único relevante es que sean de distinto género.  
4.2.3. El sistema verificará automáticamente la coherencia de géneros al cargar los datos o al editar, y notificará cualquier inconsistencia (por ejemplo, dos machos o dos hembras en un mismo `M`).

**Memoria de diseño – 4.2**  
Se eliminó la restricción de que `from` deba ser macho y `to` hembra porque en muchos conectores reales el género no determina la dirección de la señal. Lo importante es que el acople físico sea posible, y eso solo exige géneros complementarios.

#### 4.3 Catálogo completo de conectores

**Tabla 5 – Conectores del ejemplo**

| ID   | Nombre       | Padre | Designator | Pines | Género | EdgeSide | Posición (x, y, w, h) | expectedPair |
|------|--------------|-------|------------|-------|--------|----------|------------------------|--------------|
| C001 | Molex 2P     | T300  | J1         | 2     | male   | right    | 548, 360, 180, 115    | C002         |
| C002 | Molex 2P     | T200  | J2         | 2     | female | null     | 588, 360, 180, 115    | C001         |
| C003 | GX12         | T200  | J3         | 2     | female | right    | 1060, 900, 180, 115   | C004         |
| C004 | GX12         | T100  | J4         | 2     | male   | null     | 1240, 900, 180, 115   | C003         |
| C005 | GX12 4P      | T100  | J5         | 4     | male   | null     | 1463, 900, 180, 115   | C006         |
| C006 | GX12 4P      | T201  | J6         | 4     | female | left     | 1643, 900, 180, 115   | C005         |
| C007 | Molex 5P     | T201  | J7         | 5     | male   | null     | 1890, 360, 180, 115   | C008         |
| C008 | Molex 5P     | T301  | J8         | 5     | female | left     | 1890, 360, 180, 115   | C007         |
| C009 | Molex 2P     | T301  | J9         | 2     | female | left     | 1890, 540, 180, 115   | null         |

**Memoria de diseño – 4.3**  
La tabla ahora incluye `expectedPair` en lugar de `lockedWith`. Los conectores C001 a C008 tienen pareja esperada explícita, mientras que C009 está libre (null). Esto refleja la intención de diseño: C009 es un conector de reserva sin uso planificado, por lo que no tiene pareja asignada.

#### 4.4 Posicionamiento de conectores anclados

4.4.1. Si `edgeSide` está definido (conector anclado), el conector se sitúa **completamente dentro** de su contenedor padre.  
4.4.2. La **cara de los pines** (el borde donde se dibujan los círculos de pines) **coincide exactamente con el borde del contenedor** indicado por `edgeSide`.  
 4.4.2.1. Ejemplo: `edgeSide: "right"` → el borde derecho del conector toca el borde derecho del contenedor; los pines apuntan hacia la derecha.

#### 4.5 Movimiento de conectores

##### 4.5.1 Conectores anclados

4.5.1.1. Pueden **deslizarse a lo largo del borde actual** arrastrándolos con el ratón.  
4.5.1.2. Pueden **cambiarse a otro borde** del mismo contenedor:  
 a. Durante el arrastre, si el cursor se aleja más de **30 píxeles** del borde actual (medidos perpendicularmente) y está más cerca de otro borde, el sistema reasigna `edgeSide` inmediatamente.  
 b. El cambio se aplica **en tiempo real**, sin esperar a soltar el botón.

##### 4.5.2 Conectores libres

4.5.2.1. Se comportan como elementos libres **dentro de los límites de su contenedor**.  
4.5.2.2. Si su padre es el sistema raíz (`T100`), pueden moverse por todo el lienzo.  
4.5.2.3. Respeta la restricción de contención (margen de 30 px).

##### 4.5.3 Propagación rígida del movimiento

4.5.3.1. **Solo los conectores unidos mediante un `M` activo** (con `status: "connected"`) se mueven solidariamente.  
4.5.3.2. Al arrastrar un conector, el sistema consulta el array `lockedWith` (generado dinámicamente a partir de los M activos) y desplaza todos los IDs listados exactamente el mismo **dx, dy** **en cada fotograma**.  
4.5.3.3. Los conectores unidos solo por **W (wired)** no se arrastran entre sí; el cable se redibuja flexiblemente entre la nueva posición del conector arrastrado y la posición inalterada del otro extremo.

---

### 5. CABLES FIJOS (WIRED) — ENTIDAD TIPO W

Representan un conductor físico real (cable soldado o crimpado). **No transmiten movimiento mecánico**, solo se redibujan como una conexión flexible.

#### 5.1 Atributos

**Tabla 6 – Atributos de un wire**

| Campo          | Tipo   | Descripción |
|----------------|--------|-------------|
| id             | string | W001, W002… |
| type           | string | `"wired"` |
| from           | object | `{ connector: "C002", pin: 1 }` |
| to             | object | `{ connector: "C003", pin: 1 }` |
| net            | string | ID del net que transporta (obligatorio) |
| length         | number | Longitud real del cable en milímetros (informativo, sin efecto mecánico) |
| gauge          | number | Sección o calibre (AWG o mm², opcional) |
| color          | string | Color de la línea en pantalla (por defecto azul) |
| thickness      | number | Grosor visual en píxeles (opcional) |

5.1.1. El wire **no tiene `parent_id`**. Su ubicación física se deduce automáticamente de los contenedores de sus extremos:  
 a. Si ambos conectores pertenecen al mismo contenedor (o a subcontenedores dentro de él), el cable se considera que está en dicho contenedor.  
 b. Si los conectores están en contenedores diferentes, el cable se sitúa en el **ancestro común más bajo** de ambos. Por ejemplo, un cable entre C004 (en T100) y C005 (en T100) está en T100; un cable entre C002 (en T200) y C005 (en T100) tendría como ancestro común T100.  
5.1.2. Esta regla hace innecesario almacenar una propiedad explícita y evita inconsistencias.

**Memoria de diseño – 5.1**  
Se eliminó `parent_id` de los wires porque generaba confusión en cables que cruzaban entre contenedores. La solución de calcular el ancestro común es más robusta y refleja mejor la realidad física: un cable existe en el espacio que une sus extremos, no está "dentro" de una sola caja.

#### 5.2 Tabla de wires del ejemplo

**Tabla 7 – Wires del arnés**

| ID   | From (conector, pin) | To (conector, pin) | Net    | Longitud (mm) | Descripción |
|------|----------------------|---------------------|--------|---------------|-------------|
| W001 | C002, 1              | C003, 1             | N001   | 150           | Cable interno Caja 1: J2 pin1 → J3 pin1 |
| W002 | C004, 1              | C005, 1             | N001   | 400           | Latiguillo externo: J4 pin1 → J5 pin1 |
| W003 | C007, 2              | C006, 2             | N001   | 180           | Cable interno Caja 2: J7 pin2 → J6 pin2 |

#### 5.3 Visualización y comportamiento

5.3.1. Se dibuja como una **línea azul continua y flexible** entre los puntos de conexión de los dos conectores.  
5.3.2. Siempre se representa **por encima** de cajas y conectores para garantizar su visibilidad.  
5.3.3. Durante el arrastre de un conector, el wire se redibuja automáticamente **como una curva adaptable** (por ejemplo, utilizando una curva Bézier o un camino ortogonal) entre la nueva posición y la del otro extremo.  
5.3.4. La línea del cable **se acorta o se extiende libremente** según la distancia entre sus extremos, sin ninguna restricción. El campo `length` es meramente informativo (no afecta al comportamiento ni genera advertencias).

---

### 6. ACOPLES ENCHUFABLES (MATED) — ENTIDAD TIPO M

Define una unión **macho‑hembra** entre dos conectores. Es una unión **rígida**: los conectores se mueven juntos y no se dibuja ninguna línea adicional.

#### 6.1 Atributos

**Tabla 8 – Atributos de un mated**

| Campo   | Tipo   | Descripción |
|---------|--------|-------------|
| id      | string | M001, M002… |
| type    | string | `"mated"` |
| from    | object | `{ connector: "C001", pin: 1 }` |
| to      | object | `{ connector: "C002", pin: 1 }` |
| net     | string | ID del net que transporta (obligatorio) |
| status  | string | `"connected"`, `"planned"`, `"disconnected"`, `"obsolete"` |

#### 6.2 Significado de `status`

6.2.1. **`"connected"`** – El acople está físicamente realizado y los conectores se comportan como un solo bloque.  
6.2.2. **`"planned"`** – Acople previsto en el diseño pero aún no instalado. No genera `lockedWith` ni movimiento solidario, pero se muestra como una relación planificada.  
6.2.3. **`"disconnected"`** – Acople que existía pero ha sido retirado temporalmente (por ejemplo, para mantenimiento). No propaga movimiento.  
6.2.4. **`"obsolete"`** – Relación histórica que ya no se utiliza; se conserva para trazabilidad, pero no tiene efecto mecánico ni eléctrico.

6.2.5. En la fase actual del MVP, todos los **M** listados tienen `status: "connected"`.

#### 6.3 Validación de género

6.3.1. Un acople `M` es válido únicamente si los dos conectores implicados tienen **géneros opuestos** (uno `"male"` y otro `"female"`).  
6.3.2. La asignación de `from` y `to` no impone una dirección obligatoria de género; el sistema solo comprueba que los géneros no sean iguales.  
6.3.3. Cualquier combinación de dos machos o dos hembras en un mismo `M` será detectada y señalada como inconsistencia.

#### 6.4 Validación de exclusividad por `expectedPair`

6.4.1. Si un conector tiene definido el campo `expectedPair` (distinto de `null`), **solo puede participar en un `M` con ese conector específico**.  
6.4.2. Cualquier intento de crear o cargar un `M` que involucre a ese conector con un compañero distinto será rechazado por el sistema.  
6.4.3. Esta restricción garantiza que el diseño del arnés se respete estrictamente y evita conexiones incorrectas.

**Memoria de diseño – 6.4**  
La exclusividad mediante `expectedPair` es una mejora clave respecto al modelo anterior. Antes se podía acoplar cualquier conector compatible, lo que podía llevar a errores de cableado no detectados. Ahora, si un diseñador declara que J1 solo debe enchufarse con J2, el sistema lo hace cumplir. Esto es especialmente útil en arneses complejos con muchos conectores similares.

#### 6.5 Tabla de mated del ejemplo

**Tabla 9 – Acoples mated del arnés**

| ID   | From (conector, pin) | To (conector, pin) | Net    | Status    | Descripción |
|------|----------------------|---------------------|--------|-----------|-------------|
| M001 | C001, 1              | C002, 1             | N001   | connected | J1 enchufa a J2 (dentro de PCB1) |
| M002 | C004, 1              | C003, 1             | N001   | connected | J4 enchufa a J3 (panel Caja1) |
| M003 | C005, 1              | C006, 1             | N001   | connected | J5 enchufa a J6 (panel Caja2) |
| M004 | C007, 2              | C008, 2             | N001   | connected | J7 enchufa a J8 (dentro de PCB2) |

#### 6.6 Visualización

6.6.1. **No se dibuja ninguna línea** para los acoples `M`.  
6.6.2. Los dos conectores se sitúan **enfrentados borde con borde**, con sus pines en contacto directo.  
6.6.3. La unión es evidente por la disposición espacial y porque al mover uno, el otro le sigue rígidamente (gracias a `lockedWith` calculado en runtime).

---

### 7. BLOQUEO DINÁMICO (`lockedWith`)

#### 7.1 Generación en tiempo de ejecución

7.1.1. `lockedWith` **no se almacena en el archivo de proyecto**. Es un array que el sistema calcula cada vez que se cargan los datos o se modifica un `M`.  
7.1.2. Se construye examinando todos los acoples `M` con `status: "connected"`. Para cada uno, ambos conectores se añaden mutuamente a sus respectivos arrays `lockedWith`.  
7.1.3. Los **wires (`W`) no contribuyen a `lockedWith`** porque representan uniones flexibles, no rígidas.

#### 7.2 Cadenas rígidas resultantes en el ejemplo

7.2.1. Con los M activos de la tabla 9, se forman las siguientes cadenas independientes:  
 a. **Cadena 1:** C001 ↔ C002 (por M001)  
 b. **Cadena 2:** C003 ↔ C004 (por M002)  
 c. **Cadena 3:** C005 ↔ C006 (por M003)  
 d. **Cadena 4:** C007 ↔ C008 (por M004)  
7.2.2. Los wires que interconectan estas cadenas (W001 entre C002 y C003, etc.) **no las hacen solidarias mecánicamente**. Arrastrar C002 mueve también C001, pero no arrastra C003; el cable W001 se redibujará entre la nueva posición de C002 y la posición fija de C003.

**Memoria de diseño – Sección 7**  
La eliminación de `lockedWith` del almacenamiento resuelve un problema de redundancia: antes se guardaba un estado derivado que podía desincronizarse de los M. Ahora, `lockedWith` es siempre consistente porque se recalcula. Además, al no persistirse, el archivo de proyecto es más limpio y solo contiene la fuente de verdad (los M con su status). La introducción de `expectedPair` como propiedad de diseño complementa este enfoque: el diseño establece qué parejas son válidas; el runtime calcula cuáles están actualmente bloqueadas.

---

### 8. SEÑALES LÓGICAS (NET) — ENTIDAD TIPO N

Un **net** es una señal eléctrica declarada una única vez. Cada **wire** y cada **mated** referencian al net que transportan. La topología de la señal (conexiones punto a punto, bifurcaciones, buses) **emerge automáticamente** de los puntos compartidos entre wires y mated que tengan el mismo `net`.

#### 8.1 Atributos de un net

**Tabla 10 – Atributos de un net**

| Campo      | Tipo   | Descripción |
|------------|--------|-------------|
| id         | string | N001, N002… |
| name       | string | Nombre descriptivo (ej. "+12V", "Señal control") |
| type       | string | `"power"`, `"ground"`, `"data"`, `"analog"`, `"differential"`, etc. |
| voltage    | string | Tensión nominal (opcional, ej. "12V", "5V") |
| pair       | string | ID del net complementario en señales diferenciales (opcional) |

#### 8.2 Tabla de nets del ejemplo

**Tabla 11 – Nets definidos**

| ID   | Nombre               | Type  | Voltage |
|------|----------------------|-------|---------|
| N001 | Señal PCB1 → PCB2    | data  | –       |

#### 8.3 Construcción automática del grafo de señal

8.3.1. El sistema examina todos los wires y mated que comparten el mismo `net` y cuyos extremos coinciden en conector y pin.  
8.3.2. A partir de esas coincidencias, se construye internamente un grafo no dirigido de nodos `(conector, pin)` unidos por aristas `W` o `M`.  
8.3.3. Ese grafo representa la ruta eléctrica completa de la señal. Cualquier bifurcación o estrella aparece de forma natural si varios wires/mated comparten un mismo punto.  
8.3.4. En el ejemplo actual, con un único net `N001`, el grafo resultante es lineal y recorre todos los conectores de la cadena:  
 C001.pin1 – C002.pin1 – C003.pin1 – C004.pin1 – C005.pin1 – C006.pin1 – C007.pin2 – C008.pin2

#### 8.4 Uso de nets

8.4.1. Permiten **verificar la continuidad eléctrica** comprobando que todos los puntos del grafo están conectados.  
8.4.2. Facilitan **validaciones de coherencia**: por ejemplo, detectar si un cable etiquetado como `+12V` llega a un pin que se espera que sea de `5V`.  
8.4.3. Posibilitan el **filtrado visual**: al seleccionar un net, se resaltan todos los wires y mated que lo transportan.  
8.4.4. En el futuro, podrán declararse **puntos de chasis** unidos implícitamente para modelar la toma de tierra sin cables adicionales.

**Memoria de diseño – Sección 8**  
El modelo de nets como etiquetas compartidas se adoptó tras descartar el enfoque de secuencia ordenada. La razón principal es que una señal puede bifurcarse (un cable de +12V que alimenta dos dispositivos) y modelar eso con una lista plana requeriría entidades adicionales. Con el enfoque de grafo, la topología emerge de las conexiones físicas, que es exactamente como funcionan las herramientas profesionales de diseño de arneses.

---

### 9. ÁRBOL DE CONTENCIÓN (JERARQUÍA FÍSICA)

#### 9.1 Diagrama jerárquico

```
T100 (Moto)
├── T200 (Caja 1)
│   ├── T300 (PCB 1)
│   │   ├── C001 (J1, male, anclado right)
│   │   └── C002 (J2, female, libre)
│   └── C003 (J3, female, anclado right)
├── T201 (Caja 2)
│   ├── T301 (PCB 2)
│   │   ├── C008 (J8, female, anclado left)
│   │   └── C009 (J9, female, anclado left, sin conexiones)
│   ├── C006 (J6, female, anclado left)
│   └── C007 (J7, male, libre)
├── C004 (J4, male, libre)
└── C005 (J5, male, libre)
```

#### 9.2 Ubicación efectiva de los wires

9.2.1. **W001**: ambos extremos (C002 y C003) están dentro de T200 (Caja 1); por tanto, el cable se considera dentro de la Caja 1 y se mueve con ella.  
9.2.2. **W002**: los extremos C004 y C005 están directamente en T100 (sistema raíz); el cable se ubica en el sistema raíz.  
9.2.3. **W003**: ambos extremos (C007 y C006) pertenecen a T201 (Caja 2); el cable está dentro de la Caja 2.  
9.2.4. La determinación es automática y no requiere un campo `parent_id` explícito.

---

### 10. REGLAS DE CONSISTENCIA Y VALIDACIONES

10.1. **Género en M:** Los dos conectores de un acople `M` deben ser de géneros opuestos (un macho y una hembra). La asignación `from`/`to` no modifica esta exigencia. Cualquier combinación del mismo género genera un error.  
10.2. **Exclusividad de pareja:** Si un conector tiene `expectedPair` definido, solo puede aparecer en un `M` junto a ese conector. Cualquier otro emparejamiento es inválido.  
10.3. **Pines válidos:** Todos los pines referenciados en `W` y `M` deben existir en los conectores asociados (ej. si un conector tiene 2 pines, no se puede usar el pin 5).  
10.4. **Longitud de wire:** El campo `length` es meramente informativo; no se utiliza para validar la disposición física ni para restringir el movimiento. Los cables se adaptan visualmente sin limitaciones.  
10.5. **Compatibilidad jerárquica:** Dos conectores pueden estar conectados aunque estén en distintos niveles del árbol, siempre que compartan un ancestro contenedor común que los contenga físicamente.  
10.6. **Continuidad de nets:** El sistema verificará que el grafo del net sea conexo (todos los nodos alcanzables) y que no existan desconexiones inesperadas. También puede detectar si un mismo punto pertenece a dos nets distintos (conflicto de señales).

**Memoria de diseño – 10.2**  
La validación de exclusividad es nueva en esta versión y responde a la necesidad de detectar errores de cableado en tiempo de diseño. Al declarar `expectedPair`, el diseñador expresa su intención; el sistema la hace cumplir, evitando que un operario enchufe un conector donde no corresponde.

---

### 11. VISUALIZACIÓN GENERAL

11.1. **Cables fijos (W):** líneas azules curvas y flexibles, siempre en primer plano.  
11.2. **Acoples enchufables (M):** sin línea; los conectores aparecen enfrentados y alineados por sus pines.  
11.3. **Conectores anclados:** se muestran dentro del contenedor, con los círculos de pines exactamente sobre el borde indicado.  
11.4. **Interacción:** clic en cualquier entidad → panel de propiedades con todos los atributos. El modo edición está permanentemente activo en esta fase MVP.  
11.5. **Redimensionamiento:** tiradores en todos los bordes y esquinas de los contenedores. Los conectores anclados mantienen distancia a la esquina fija más cercana, según lo descrito en **3.3.2.1**.  
11.6. **Resaltado de nets:** al seleccionar un net en un panel lateral, todos los wires que lo transportan cambian temporalmente de color o aumentan su grosor.

---

### 12. NOTAS PARA FUTURAS VERSIONES

12.1. Incorporar un **modo “no edición”** con bloqueo del lienzo mediante un botón.  
12.2. Implementar **enrutamiento automático de wires** que evite obstáculos y permita agrupar cables en mazos.  
12.3. Adaptar el **tamaño visual de los conectores** al número de pines o a la longitud del `designator` para mejorar la legibilidad.  
12.4. Añadir un **panel de validación global** que recoja todas las inconsistencias encontradas (género, pines, conflictos de nets, violaciones de `expectedPair`).  
12.5. Soportar **pares trenzados** mediante el campo `pair` en nets diferenciales, para validar que ambos nets sigan rutas coherentes.  
12.6. Implementar **puntos de chasis** como nodos comunes de tierra modelados sin cables explícitos.

---

### 13. HISTORIAL DE VERSIONES

**Versión 1.0** – Documentación inicial completa del MVP con sistema de IDs, jerarquía de contenedores, conectores con expectedPair, cálculo dinámico de lockedWith, nets como etiquetas compartidas, y validaciones.

**Versión 1.1** – Mejoras de estructura y claridad:
- La tabla de identificación de entidades se consolida en un índice único (Tabla 1) con una sola fila para el prefijo `T`, detallando la subdivisión de rangos en el texto.
- La sección 3 se renombra de "Componentes no conectores" a "Contenedores (system, enclosure, pcb)" para reflejar mejor su función.
- Sin cambios en la lógica del sistema, solo en la presentación de la documentación.
