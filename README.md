# Práctica XML y DTD

Creación y validación de documentos XML con DTD internos y externos, con el proceso de construcción registrado en Git.

**Autora:** Erandeni Mendívil Morales

## Objetivo

Transformar información no estructurada en documentos XML bien formados, definir DTD internos y externos que expresen cardinalidades y restricciones de atributos, comprobar la validez de los documentos —incluidas pruebas negativas que deben fallar— y registrar cada avance como un commit que represente una unidad lógica de trabajo.

**Estructura del repositorio**

```text
xml-dtd-practica/
├── README.md
├── ejercicio1/
│   └── pedido.xml               documento bien formado, sin DTD
├── ejercicio2/
│   ├── nota.xml                 usa el DTD externo nota.dtd
│   ├── nota.dtd
│   └── nota-interno.xml         DTD dentro del propio documento
└── ejercicio3/
    ├── matricula.xml            usa el DTD externo matricula.dtd
    ├── matricula.dtd
    └── matricula-interno.xml    DTD dentro del propio documento
```

**Validación.** Las comprobaciones se hicieron con `xmllint` (libxml2 2.9.14). Sin `--valid` solo revisa la buena formación; con `--valid` además valida contra el DTD declarado en `<!DOCTYPE>`. Si no imprime nada y termina con código 0, el documento pasó.

```bash
xmllint --noout ejercicio1/pedido.xml          # buena formación
xmllint --noout --valid ejercicio2/nota.xml    # buena formación + validez
```

## Ejercicio 1: Pedido

Texto de partida: *"Pedido para el Juan Delgado Martínez. El pedido se compone de una Bicicleta Bianchi. A entregar calle Reforma 423, interior 201, día 19-09-2021"*. El documento debe permitir búsquedas por destinatario, artículo, dirección de entrega y fecha de entrega.

### Modelo propuesto

| Información  | Valor identificado              | Elemento XML propuesto |
|--------------|---------------------------------|------------------------|
| Destinatario | Juan Delgado Martínez           | `destinatario`, con `nombre`, `apellido_paterno` y `apellido_materno` |
| Artículo     | Bicicleta Bianchi (una unidad)  | `articulo`, con atributo `cantidad` y elementos `descripcion` y `marca` |
| Dirección    | calle Reforma 423, interior 201 | `direccion_entrega`, con `calle`, `numero_exterior` y `numero_interior` |
| Fecha        | 19-09-2021                      | `fecha_entrega`, en formato ISO 8601: `2021-09-19` |

Jerarquía:

```text
pedido
├── destinatario
│   ├── nombre
│   ├── apellido_paterno
│   └── apellido_materno
├── articulo  (@cantidad)
│   ├── descripcion
│   └── marca
├── direccion_entrega
│   ├── calle
│   ├── numero_exterior
│   └── numero_interior
└── fecha_entrega
```

`ejercicio1/pedido.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<pedido>
    <destinatario>
        <nombre>Juan</nombre>
        <apellido_paterno>Delgado</apellido_paterno>
        <apellido_materno>Martínez</apellido_materno>
    </destinatario>
    <articulo cantidad="1">
        <descripcion>Bicicleta</descripcion>
        <marca>Bianchi</marca>
    </articulo>
    <direccion_entrega>
        <calle>Reforma</calle>
        <numero_exterior>423</numero_exterior>
        <numero_interior>201</numero_interior>
    </direccion_entrega>
    <fecha_entrega>2021-09-19</fecha_entrega>
</pedido>
```

Buena formación: un único elemento raíz (`pedido`), todas las etiquetas cerradas, anidamiento correcto y el atributo entre comillas; `xmllint --noout ejercicio1/pedido.xml` termina sin errores.

Cada campo de búsqueda es un hijo directo de `pedido`, así que puede localizarse de forma independiente, igual que sus componentes. Consultas XPath ejecutadas sobre el archivo:

```text
$ xmllint --xpath 'normalize-space(/pedido/destinatario)' ejercicio1/pedido.xml
Juan Delgado Martínez
$ xmllint --xpath 'string(/pedido/articulo/marca)' ejercicio1/pedido.xml
Bianchi
$ xmllint --xpath 'string(/pedido/direccion_entrega/calle)' ejercicio1/pedido.xml
Reforma
$ xmllint --xpath 'string(/pedido/fecha_entrega)' ejercicio1/pedido.xml
2021-09-19
```

### Decisiones de diseño

**1. ¿Conviene almacenar la dirección como un único texto?** No. Una cadena como "calle Reforma 423, interior 201" solo admite búsquedas por coincidencia de texto, y cada persona la escribe distinto ("Int. 201", "interior 201", "423-201"). Para consultar o validar cualquiera de sus partes habría que volver a analizar la cadena.

**2. ¿Qué ventajas tendría separar sus componentes?** Cada parte se consulta, se ordena o se valida por separado (por ejemplo, los pedidos de la calle Reforma: `//direccion_entrega[calle='Reforma']`); las partes que no siempre existen, como el número interior, se omiten sin ambigüedad; y la dirección puede recomponerse en el formato que haga falta (etiqueta de envío, formulario, mapa). Sobre piso y letra: el texto dice "interior 201", que en México es el número interior, por lo que se conservó como `numero_interior`. No se descompuso en piso y letra porque el texto no indica si 201 significa piso 2, departamento 01; separarlo sería inventar información. Con el mismo criterio, el nombre se dividió en nombre y apellidos (para buscar u ordenar por apellido) y el artículo en descripción y marca (para buscar por marca).

**3. ¿Cómo debería almacenarse una fecha para facilitar su procesamiento?** En formato ISO 8601, `AAAA-MM-DD`: `2021-09-19`. Así no hay ambigüedad entre día y mes (19-09 frente a 09-19), el orden alfabético coincide con el cronológico, es el formato del tipo `xs:date` de XML Schema y la mayoría de los lenguajes de programación lo interpretan sin conversión.

**4. ¿Qué información podría ser atributo y cuál elemento?** Criterio aplicado: lo que se busca, puede repetirse o puede tener estructura interna va como elemento; los datos atómicos que califican a otro elemento (cantidades, identificadores, tipos, unidades) van como atributo. Por eso los cuatro campos de búsqueda son elementos —un atributo no puede contener elementos ni repetirse en el mismo elemento— y `cantidad="1"` es atributo de `articulo`: califica a la línea del pedido y no tiene partes internas.

## Ejercicio 2: Nota

Análisis del documento:

1. **¿Cuál es el elemento raíz?** `nota`.
2. **¿Cuántas veces aparece `para`?** Exactamente una.
3. **¿El orden de los elementos es significativo?** Sí. XML conserva el orden de los elementos y, en el DTD, la coma define una secuencia: `para`, `de`, `titulo` y `contenido` deben aparecer en ese orden. La prueba negativa 2 lo confirma.
4. **¿Los elementos contienen otros elementos o solamente texto?** `nota` contiene únicamente elementos; `para`, `de`, `titulo` y `contenido` contienen únicamente texto.

### DTD externo

| Elemento    | Contenido esperado | Declaración DTD |
|-------------|--------------------|-----------------|
| `nota`      | elementos          | `<!ELEMENT nota (para, de, titulo, contenido)>` |
| `para`      | texto              | `<!ELEMENT para (#PCDATA)>` |
| `de`        | texto              | `<!ELEMENT de (#PCDATA)>` |
| `titulo`    | texto              | `<!ELEMENT titulo (#PCDATA)>` |
| `contenido` | texto              | `<!ELEMENT contenido (#PCDATA)>` |

`ejercicio2/nota.dtd`:

```dtd
<!ELEMENT nota (para, de, titulo, contenido)>
<!ELEMENT para (#PCDATA)>
<!ELEMENT de (#PCDATA)>
<!ELEMENT titulo (#PCDATA)>
<!ELEMENT contenido (#PCDATA)>
```

`nota.xml` lo asocia con `<!DOCTYPE nota SYSTEM "nota.dtd">`. La ruta es relativa al XML, por eso ambos archivos están en la misma carpeta. `xmllint --noout --valid ejercicio2/nota.xml` termina sin errores.

### DTD interno

`ejercicio2/nota-interno.xml` lleva las mismas declaraciones dentro de la declaración de tipo de documento, entre corchetes, y valida sin errores:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE nota [
    <!ELEMENT nota (para, de, titulo, contenido)>
    <!ELEMENT para (#PCDATA)>
    <!ELEMENT de (#PCDATA)>
    <!ELEMENT titulo (#PCDATA)>
    <!ELEMENT contenido (#PCDATA)>
]>
<nota>
    <para>Pedro</para>
    <de>Laura</de>
    <titulo>Recordatorio</titulo>
    <contenido>A las 7:00 en la puerta del teatro</contenido>
</nota>
```

| Característica                      | DTD interno | DTD externo |
|-------------------------------------|-------------|-------------|
| Ubicación                           | Dentro del XML, entre corchetes en `<!DOCTYPE nota [ ... ]>` | En un archivo aparte (`nota.dtd`), referenciado con `SYSTEM` |
| Reutilizable entre XML              | No: cada documento lleva su propia copia | Sí: todos los documentos pueden apuntar al mismo `.dtd` |
| Archivo adicional                   | No | Sí, y debe estar accesible en la ruta indicada |
| Conveniente para un único documento | Sí: todo viaja en un solo archivo | Funciona, pero agrega un archivo que mantener |
| Conveniente para muchos documentos  | No: habría que mantener copias idénticas | Sí: un cambio en el `.dtd` aplica a todos |

### Pruebas realizadas

Cada modificación se aplicó por separado sobre `nota.xml` ya confirmado en Git; después se validó, se revisó con `git diff` y se deshizo con `git restore ejercicio2/nota.xml`.

| Modificación                                          | ¿Bien formado? | ¿Válido? | ¿Por qué? |
|-------------------------------------------------------|----------------|----------|-----------|
| Cambiar `para` por `destinatario` (apertura y cierre) | Sí | No | `destinatario` no está declarado y `nota` exige `para` como primer hijo |
| Cambiar el orden de `para` y `de`                     | Sí | No | La secuencia `(para, de, titulo, contenido)` obliga a ese orden |
| Agregar `telefono`                                    | Sí | No | `telefono` no está declarado y la secuencia no admite un quinto hijo |

Mensajes de `xmllint --noout --valid` (código de salida 4 en los tres casos):

```text
# 1. Cambiar para
ejercicio2/nota.xml:4: element destinatario: validity error : No declaration for element destinatario
ejercicio2/nota.xml:8: element nota: validity error : Element nota content does not follow the DTD, expecting (para , de , titulo , contenido), got (destinatario de titulo contenido )

# 2. Cambiar orden
ejercicio2/nota.xml:8: element nota: validity error : Element nota content does not follow the DTD, expecting (para , de , titulo , contenido), got (de para titulo contenido )

# 3. Agregar telefono
ejercicio2/nota.xml:8: element telefono: validity error : No declaration for element telefono
ejercicio2/nota.xml:9: element nota: validity error : Element nota content does not follow the DTD, expecting (para , de , titulo , contenido), got (para de titulo contenido telefono )
```

`git diff` de la prueba 2, antes de restaurar:

```diff
diff --git a/ejercicio2/nota.xml b/ejercicio2/nota.xml
index b21a76b..0148dff 100644
--- a/ejercicio2/nota.xml
+++ b/ejercicio2/nota.xml
@@ -1,8 +1,8 @@
 <?xml version="1.0" encoding="UTF-8"?>
 <!DOCTYPE nota SYSTEM "nota.dtd">
 <nota>
-    <para>Pedro</para>
     <de>Laura</de>
+    <para>Pedro</para>
     <titulo>Recordatorio</titulo>
     <contenido>A las 7:00 en la puerta del teatro</contenido>
 </nota>
```

**Prueba adicional: cambiar solo la etiqueta de apertura.** Si se cambia `<para>` pero se deja `</para>`, el documento deja de estar bien formado y `xmllint` se detiene (código 1) sin llegar a validar:

```text
ejercicio2/nota.xml:4: parser error : Opening and ending tag mismatch: destinatario line 4 and para
```

La buena formación es requisito previo de la validez: un documento mal formado ni siquiera llega a compararse con el DTD.

## Ejercicio 3: Matrícula

### Modelo

```text
matricula
├── personal
│   ├── dni
│   ├── nombre
│   ├── titulacion
│   ├── curso_academico
│   └── domicilios
│       └── domicilio+  (@tipo)
│           └── nombre
└── pago
    └── tipo_matricula
```

| Clasificación                              | Componentes |
|--------------------------------------------|-------------|
| Elementos simples (solo texto)             | `dni`, `nombre`, `titulacion`, `curso_academico`, `tipo_matricula` |
| Elementos compuestos (contienen elementos) | `matricula`, `personal`, `domicilios`, `domicilio`, `pago` |
| Elementos repetibles                       | `domicilio`, dentro de `domicilios` |
| Atributos                                  | `tipo`, en `domicilio` |
| Restricciones                              | al menos un `domicilio`; `tipo` obligatorio y limitado a `familiar` o `habitual` |

`nombre` aparece en dos contextos: el nombre del alumno (hijo de `personal`) y el del domicilio (hijo de `domicilio`). En DTD las declaraciones de elementos son globales, así que una sola línea `<!ELEMENT nombre (#PCDATA)>` rige ambos usos; no podrían tener contenidos distintos según el elemento padre (XML Schema sí permite declaraciones locales).

### Cardinalidad

"Al menos uno" corresponde a `+` (uno o más). `?` permitiría como máximo un domicilio y `*` permitiría ninguno:

```dtd
<!ELEMENT domicilios (domicilio+)>
```

### Restricción del atributo tipo

La enumeración se escribe entre paréntesis, con los valores separados por una barra vertical, y `#REQUIRED` hace obligatorio el atributo:

```dtd
<!ATTLIST domicilio tipo (familiar | habitual) #REQUIRED>
```

El historial muestra la evolución: el commit `adf0225` declaró `tipo` como `CDATA #IMPLIED` (cualquier texto, opcional) y el commit `cddc17f` lo sustituyó por la enumeración obligatoria. Con la versión intermedia, `tipo="temporal"` y un domicilio sin `tipo` todavía eran válidos; con la final, ambos casos se rechazan.

```diff
-<!ATTLIST domicilio tipo CDATA #IMPLIED>
+<!-- Requisito: tipo obligatorio, solo "familiar" o "habitual" -->
+<!ATTLIST domicilio tipo (familiar | habitual) #REQUIRED>
```

### DTD externo

`ejercicio3/matricula.dtd`, asociado en `matricula.xml` con `<!DOCTYPE matricula SYSTEM "matricula.dtd">`:

```dtd
<!ELEMENT matricula (personal, pago)>
<!ELEMENT personal (dni, nombre, titulacion, curso_academico, domicilios)>
<!ELEMENT dni (#PCDATA)>
<!ELEMENT nombre (#PCDATA)>
<!ELEMENT titulacion (#PCDATA)>
<!ELEMENT curso_academico (#PCDATA)>

<!-- Requisito: al menos un domicilio -->
<!ELEMENT domicilios (domicilio+)>
<!ELEMENT domicilio (nombre)>
<!-- Requisito: tipo obligatorio, solo "familiar" o "habitual" -->
<!ATTLIST domicilio tipo (familiar | habitual) #REQUIRED>

<!ELEMENT pago (tipo_matricula)>
<!ELEMENT tipo_matricula (#PCDATA)>
```

`xmllint --noout --valid ejercicio3/matricula.xml` termina sin errores.

### DTD interno

`ejercicio3/matricula-interno.xml` incluye las mismas declaraciones dentro de `<!DOCTYPE matricula [ ... ]>` y valida sin errores. Se desarrolló en la rama `dtd-interno-matricula` y se integró después en `main` (ver [Historial de Git](#historial-de-git)).

### Pruebas realizadas

Mismo procedimiento que en el ejercicio 2: modificar el archivo confirmado, validar, revisar con `git diff` y restaurar con `git restore`.

**Sin domicilios** (se eliminaron los dos `domicilio` y `domicilios` quedó vacío):

```text
¿XML bien formado? Sí
¿XML válido? No
¿Por qué? domicilios quedó vacío y su modelo (domicilio+) exige al menos un domicilio.
```

Como contraste, con `(domicilio*)` el mismo documento sin domicilios sí resulta válido; por eso el requisito se expresa con `+`.

**Atributo `tipo`:**

| Caso              | Predicción | Resultado | Explicación |
|-------------------|------------|-----------|-------------|
| `tipo="familiar"` | Válido     | Válido    | El valor pertenece a la enumeración |
| `tipo="habitual"` | Válido     | Válido    | El valor pertenece a la enumeración |
| `tipo="temporal"` | No válido  | No válido | El valor no pertenece a la enumeración |
| sin `tipo`        | No válido  | No válido | El atributo está declarado como `#REQUIRED` |

Los dos primeros casos corresponden al documento original, que contiene ambos valores. Los cuatro documentos están bien formados; solo cambia su validez.

Mensajes de `xmllint --noout --valid` (código de salida 4):

```text
# Sin domicilios
ejercicio3/matricula.xml:10: element domicilios: validity error : Element domicilios content does not follow the DTD, expecting (domicilio)+, got ()

# tipo="temporal"
ejercicio3/matricula.xml:10: element domicilio: validity error : Value "temporal" for attribute tipo of domicilio is not among the enumerated set

# Sin tipo
ejercicio3/matricula.xml:12: element domicilio: validity error : Element domicilio does not carry attribute tipo
```

Las tres pruebas negativas se repitieron sobre `matricula-interno.xml` con los mismos resultados: el DTD interno es equivalente al externo.

## Historial de Git

Salida de `git log --oneline --graph --all` antes del commit de este README:

```text
*   9a48eb6 Integrar la rama dtd-interno-matricula
|\
| * 0c7e2a7 Implementar DTD interno para matrícula
|/
* cddc17f Restringir el atributo tipo de domicilio a familiar o habitual
* adf0225 Agregar matrícula con DTD externo y cardinalidad de domicilios
* deb6136 Agregar versión con DTD interno para nota
* 18f5108 Agregar validación externa DTD para nota
* a811a94 Resolver ejercicio 1: documento XML de pedido
* 747c98e Inicializar estructura de la práctica XML
```

- Git no versiona carpetas vacías: el commit inicial solo contiene el README, y cada carpeta `ejercicioN/` entró al historial junto con su primer archivo.
- La integración se hizo con `git merge --no-ff dtd-interno-matricula`. Como `main` no avanzó mientras existía la rama, un `git merge` simple habría hecho *fast-forward* y la rama no habría dejado rastro; `--no-ff` crea el commit de merge y conserva la bifurcación en el historial.
- Cada commit deja el repositorio en un estado consistente: todos los XML presentes en cada commit son válidos (o, en el caso de `pedido.xml`, bien formados).

## Conclusiones

**1. ¿Cuál es la diferencia entre XML bien formado y XML válido?** Un XML bien formado cumple las reglas sintácticas del lenguaje: un solo elemento raíz, etiquetas cerradas y correctamente anidadas, atributos entre comillas, coincidencia exacta de mayúsculas y minúsculas entre etiquetas, y caracteres especiales escapados. Un XML válido, además, se ajusta a una gramática (DTD o XML Schema). Todo documento válido está bien formado, pero no al revés: las tres pruebas negativas de `nota` produjeron documentos bien formados e inválidos, mientras que la prueba adicional produjo uno mal formado que el analizador rechazó antes de validar.

**2. ¿Qué función cumple un DTD?** Define el vocabulario y la estructura de un tipo de documento: qué elementos existen, qué contienen, en qué orden y cuántas veces aparecen, y qué atributos admiten, con qué valores y si son obligatorios. Permite validar los documentos de forma automática y funciona como contrato entre quien los produce y quien los procesa.

**3. ¿Qué diferencia existe entre DTD interno y externo?** El interno se escribe dentro del documento, entre corchetes en `<!DOCTYPE>`; el externo vive en un archivo `.dtd` referenciado con `SYSTEM` (o `PUBLIC`). El interno conviene para un documento aislado; el externo, para muchos documentos que comparten estructura. También pueden combinarse (`<!DOCTYPE nota SYSTEM "nota.dtd" [ ... ]>`): el subconjunto interno se procesa primero, así que sus declaraciones de entidades y de atributos tienen prioridad.

**4. ¿Cómo se expresa cardinalidad en DTD?** Con un operador después del elemento o del grupo: sin operador, exactamente una vez; `?`, cero o una; `*`, cero o más; `+`, una o más. La coma indica secuencia y la barra vertical, alternativa. En esta práctica, `(domicilio+)` exige al menos un domicilio.

**5. ¿Cómo puede restringirse un atributo a determinados valores?** Con un tipo enumerado en `<!ATTLIST>`, como `<!ATTLIST domicilio tipo (familiar | habitual) #REQUIRED>`. Después de la lista va la regla de presencia: `#REQUIRED` (obligatorio), `#IMPLIED` (opcional), `#FIXED "valor"` (valor constante) o un valor por omisión entre comillas.

**6. ¿Qué ventaja proporcionó Git durante las pruebas?** Permitió romper los documentos a propósito sin riesgo: la versión válida ya estaba confirmada, así que cada prueba partía de un estado conocido y podía revertirse por completo. Además, el historial documenta el proceso, y cada commit corresponde a una unidad de trabajo verificada.

**7. ¿Qué utilidad tuvieron `git diff` y `git restore`?** `git diff` mostró exactamente qué líneas cambió cada prueba respecto al último commit, lo que confirma que el error de validación se debía a ese cambio y no a otro. `git restore` devolvió el archivo a la versión confirmada sin deshacer cambios a mano. Debe usarse con cuidado: descarta las modificaciones no confirmadas y no hay forma de recuperarlas.

**8. ¿Qué ventaja proporcionó una rama para desarrollar una solución alternativa?** La versión con DTD interno de la matrícula se construyó aislada en `dtd-interno-matricula` mientras `main` conservaba una versión validada. Si la variante hubiera fallado, bastaba con descartar la rama; como validó, se integró con un merge que deja constancia en el historial.

**Cierre.** El DTD controla la estructura —qué elementos y atributos existen, su orden, su cardinalidad y una lista cerrada de valores para los atributos—, pero no el contenido textual: no puede exigir que `dni` tenga ocho dígitos y una letra, que `curso_academico` siga el patrón `AAAA/AAAA` ni que `fecha_entrega` sea una fecha real. Tampoco permite declarar `nombre` de forma distinta según su contexto. Esas necesidades corresponden a XML Schema (XSD), que ofrece tipos de datos, patrones y declaraciones locales.
