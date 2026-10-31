# autopresupuesto

Banco de pruebas abierto para que un agente de IA mejore, en ciclos medibles, el camino que va de un modelo BIM a un presupuesto de obra: importar un IFC, sacar cantidades, conciliarlas con el cómputo métrico y verlas en 3D.

**Estado:** en arranque. No hay código todavía; este README fija la idea, el contrato, las métricas y los datos.

## La idea

Un presupuesto de obra se arma con cantidades. Hoy esas cantidades se miden a mano o se copian de una planilla. Un modelo BIM trae las cantidades de cada elemento (muro, losa, columna) con un identificador único, pero nadie las cruza de forma confiable con el cómputo del presupuesto. Este repo existe para medir cuánto mejora ese cruce y dejar que un agente lo mejore solo.

Tres reglas del enfoque:

1. **El código hace la matemática y la IA explica.** Las cantidades y las diferencias salen de funciones deterministas con decimales exactos. El agente no calcula.
2. **Se vincula por código, no por descripción.** Cada elemento del modelo lleva un parámetro (`Codigo_Rubro`) que coincide con el código del ítem del presupuesto. Los que no lo traen quedan aparte, a la vista.
3. **Todo cambio se mide.** Una mejora que no sube la métrica se descarta.

## El bucle (adaptado de autoresearch)

Tomamos la estructura de [karpathy/autoresearch](https://github.com/karpathy/autoresearch): un agente modifica un solo archivo, corre un experimento de duración fija, compara una métrica y conserva o descarta el cambio. Lo que sigue está escrito con nuestras palabras; no copiamos código ni texto de ese repo, que no declara licencia.

| En autoresearch | Acá |
| --- | --- |
| `prepare.py`: datos y evaluación fijos, que el agente no toca | `bench/`: pares modelo + cómputo, la verdad de cada par y la función que puntúa |
| `train.py`: el único archivo que edita el agente | `pipeline/`: IFC → cantidades → vínculo por código → conciliación |
| `program.md`: instrucciones que escribe y afina un humano | `program.md`: reglas, qué no tocar y cuándo parar |
| Presupuesto fijo de 5 minutos por experimento | Una corrida completa del benchmark en un tiempo fijo |
| Una métrica (`val_bpb`) | Una métrica escalar por pista, más condiciones mínimas |
| Registro `results.tsv` y una rama por corrida | Igual |

Reglas del bucle que conservamos:

- El agente edita un solo archivo por pista y nunca el benchmark ni la función de puntaje.
- No agrega dependencias.
- La primera corrida es siempre la línea base, sin modificar nada.
- **Criterio de simplicidad:** a igual resultado gana el código más simple. Borrar código y mantener la métrica es una victoria.
- Cada experimento queda en `results.tsv` (separado por tabulaciones) y cada corrida vive en su propia rama.

## Contrato de salida (v0, propuesto)

Todo pipeline entrega la misma estructura, para que el resultado se pueda copiar a cualquier aplicación sin acoplarla al banco de pruebas:

```json
{
  "modelo": { "archivo": "casa.ifc", "esquema": "IFC4", "hash": "sha256:…" },
  "elementos": [
    {
      "globalId": "2O2Fr$t4X7Zf8NOew3FLOH",
      "clase": "IfcWall",
      "codigo": "3.1.1",
      "unidad": "m2",
      "cantidad": "42.350"
    }
  ],
  "sinVincular": ["…globalId de los elementos sin código…"]
}
```

Las cantidades viajan como texto decimal, nunca como número de punto flotante. Los elementos sin código no se descartan: van a `sinVincular`.

## Métricas (propuestas)

| Pista | Métrica que decide | Condición mínima |
| --- | --- | --- |
| Importador | Error relativo mediano de la cantidad contra la verdad, por elemento | Todo elemento con cantidad en la verdad aparece en la salida |
| Conciliación | F1 de los cuatro estados: coincide, difiere, solo en el modelo, solo en el cómputo | Ninguna cantidad cambia de unidad en silencio |
| Visor | Seleccionar una fila resalta exactamente los elementos esperados | Cero violaciones de accesibilidad automáticas y todo por teclado |
| Asistente | La herramienta correcta con los argumentos correctos | Nunca inventa una cantidad |

La verdad tiene dos niveles. **Sintética:** modelos generados con cantidades exactas por construcción. **Real:** modelos de Revit cuya verdad sale de las tablas de cantidades que Revit exporta, no de un cómputo hecho a mano.

## Herramientas

Candidatas, **sin verificar** (licencia, precisión y peso se confirman antes de depender de ellas):

- [web-ifc](https://github.com/ThatOpen/engine_web-ifc) para leer IFC en TypeScript, y [That Open Components](https://github.com/ThatOpen/engine_components) para el visor.
- [IfcOpenShell](https://ifcopenshell.org/) en Python, como **oráculo**: una segunda implementación independiente para contrastar las cantidades, y para generar los modelos sintéticos.
- Decimales exactos (por ejemplo `decimal.js`) para toda cantidad.

Un archivo `.rvt` es propietario y estas herramientas leen IFC. El camino es exportar desde Revit a IFC (revisar que la exportación incluya las cantidades base) o pasar por la API de Autodesk Platform Services.

## Datos

No se redistribuye ningún modelo desde este repo. `datasets/` está ignorado por git y cada fuente se descarga según su licencia. La licencia de la tabla es **la que declara la fuente**; confirmala en origen antes de reutilizar.

| Fuente | Qué trae | Licencia declarada | Para qué sirve |
| --- | --- | --- | --- |
| [bongjae/ifc_sample_models](https://huggingface.co/datasets/bongjae/ifc_sample_models) (Hugging Face) | 59 archivos: varios IFC y al menos un `.rvt` (BasicHouse) con su IFC | MIT (ficha); procedencia de los modelos sin verificar | Par Revit + IFC para probar la exportación |
| [NNTS/BIM_IFC](https://huggingface.co/datasets/NNTS/BIM_IFC) (Hugging Face) | 6 IFC | Desconocida | Solo uso local |
| [ZijianWang-ZW/GNI-BIM-Dataset](https://github.com/ZijianWang-ZW/GNI-BIM-Dataset) | IFC de proyectos de estudiantes de TUM GNI | CC-BY-4.0 según la descripción; GitHub no la detecta | Modelos reales de tamaño variado |
| [buildingSMART/Certification-datasets](https://github.com/buildingSMART/Certification-datasets) (antes `Sample-Test-Files`) | Muestras por esquema y versión | "Other" | Casos mínimos de cada esquema |
| [STEP Tools, archivos IFC](https://downloads.steptools.com/docs/stpfiles/ifc/) | IFC2x3 e IFC4, incluido AC20-FZK-Haus | Por archivo, sin verificar | Casa completa de referencia |
| [Datos de investigación de TU Wien](https://researchdata.tuwien.ac.at/records/n0mcf-ghs19) | 5 IFC (IFC4 e IFC4X3) | Sin verificar | Variedad de esquemas |
| [Mendeley: BIM Spatial Models](https://data.mendeley.com/datasets/fm9myphk9g) | 3 proyectos IFC con CSV de datos extraídos | Sin verificar | CSV que sirven de verdad parcial |
| IFCNet (RWTH Aachen, [arXiv 2106.09712](https://arxiv.org/abs/2106.09712)) | IFC de una sola entidad | Sin verificar | Clasificación de clases; no sirve para cantidades |
| [Dietmar2020/ifc-bim-qa-improved](https://huggingface.co/datasets/Dietmar2020/ifc-bim-qa-improved) (Hugging Face) | Preguntas y respuestas de texto sobre IFC | Sin verificar | Pista del asistente; no tiene modelos |

Pendiente de ubicar: el Open IFC Model Repository de la Universidad de Auckland (modelos desde 54 entidades hasta cerca de dos millones, con algunos errores conocidos) y la versión 2 de IFC-Bench, que su autor pasó a Hugging Face.

**Kaggle:** no pude confirmar ningún dataset de IFC o BIM ahí. Si encontrás uno, abrí un issue con el enlace y la licencia.

## Datos privados

Los modelos y presupuestos reales de una empresa **nunca se suben a este repo ni se comparten con un agente en la nube**. Se usan solo en local, como conjunto reservado que el agente no ve, para medir si lo aprendido en los datos públicos se sostiene.

## Estructura prevista

```
bench/       datos y puntaje fijos (el agente no los toca)
pipeline/    el archivo que el agente modifica en cada pista
oracle/      segunda implementación, en Python, para contrastar
viewer/      visor 3D
program.md   instrucciones para el agente
results.tsv  registro de experimentos
datasets/    descargas locales (ignorado por git)
```

## Próximos pasos

1. Congelar el contrato de salida.
2. Reunir los primeros pares y escribir el puntaje del importador.
3. Medir la línea base con el pipeline más simple posible, antes de dejar correr al agente.

## Licencia

Pendiente de elegir. Hasta que se agregue un archivo `LICENSE`, el contenido tiene todos los derechos reservados.
