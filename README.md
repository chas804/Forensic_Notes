
# 🔍 DFIR Write-Up: USB Disk Forensics (Root-Me)

> **Reto:** Forensics - Memory / Disk / USB  
> **Dificultad:** Principiante / Intermedio  
> **Plataforma:** Root-Me.org  
> **Objetivo:** Identificar al propietario de la memoria USB recuperada  
> **Flag:** `javier_turcot`  
> **Autor:** [Christian Albeno]  

## 🧭 1. Roadmap de Aprendizaje: Metodología Forense Digital

Para afrontar este y cualquier reto de análisis forense en imágenes de disco (DFIR), se aplica la metodología estandarizada en 5 fases:
[1. Triaje e Identificación]
↓
[2. Exploración del Sistema de Archivos]
↓
[3. Recuperación de Evidencia (Carving/Inodes)]
↓
[4. Análisis de Capas Ocultas y Metadatos]
↓
[5. Correlación de Inteligencia y Respuesta]

1. **Triaje e Identificación del Medio:** Determinar la naturaleza real del contenedor binario mediante sus *magic bytes* (sin confiar en la extensión del archivo).
2. **Exploración de la Estructura de Disco:** Inspeccionar las tablas de asignación de archivos (FAT, NTFS, EXT) y el espacio no asignado (*unallocated space*).
3. **Recuperación de Evidencia (*Carving* / Inodes):** Extraer archivos borrados o fragmentados sin alterar la imagen original.
4. **Análisis de Capas Ocultas y Metadatos:** Inspeccionar metadatos (EXIF, XMP, ID3), flujos de compresión o archivos embebidos.
5. **Correlación de Inteligencia y Respuesta:** Mapear los hallazgos técnicos con el enunciado del problema para responder a la pregunta de investigación.

---

## 🛠️ 2. Misión Paso a Paso: Análisis Técnico y Justificación

### Paso 1: Triaje e Identificación del Medio
* **Comando:**
  ```bash
  gzip -d ch39.gz
  file usb.image

    ¿Por qué se usó esta acción?: Los artefactos forenses se entregan comprimidos. El comando file lee los números mágicos del archivo para confirmar que estábamos ante una imagen de disco con sistema de archivos FAT16.

Paso 2: Exploración de Estructuras y Recuperación

    Comando: fls -r -p usb.image

¿Por qué se usó esta acción?: Al borrar un archivo en FAT16, los datos no se destruyen inmediatamente; solo se desvincula su entrada. fls de The Sleuth Kit escaneo la tabla del sistema e identificó la entrada del archivo borrado:
r/r * 5: anonyme.png (donde * indica archivo eliminado y 5 es su inode/cluster).

Extracción de la evidencia: 
icat usb.image 5 > anonyme.png
file anonyme.png

    ¿Por qué se usó esta acción?: icat extrae los bloques crudos del disco asignados al nodo 5 directamente a un archivo limpio (anonyme.png).

Paso 3: Análisis de Metadatos

    Comando:
strings -a anonyme.png | grep -iE "creator|xmp|author"
# O bien usando exiftool:
exiftool anonyme.png

¿Por qué se usó esta acción?: Muchas imágenes contienen metadatos extendidos XMP/EXIF agregados por aplicaciones como Photoshop, Lightroom o ExifTool. Al analizar el texto limpio del archivo, encontramos la estructura XMP:

XML

<dc:creator>
  <rdf:Seq>
    <rdf:li>Javier Turcot</rdf:li>
  </rdf:Seq>
</dc:creator>

Paso 4: Correlación con el Enunciado

    Enunciado: "La bandera es la identidad del propietario en forma firstname_lastname".

    Resultado final: Formateando la identidad hallada (Javier Turcot) obtenemos la bandera: javier_turcot.

❌ 3. Matriz de Caminos sin Salida (Rabbit Holes) y Lecciones
Acción Intentada	Error / Resultado	¿Por qué falló?	Lección Aprendida
Búsqueda GUI en Autopsy	La sección File Analysis no mostraba datos.	Corrupción menor en las tablas FAT16 que la GUI no procesó adecuadamente.	Si la interfaz gráfica falla, la línea de comandos (fls, sleuthkit) es la fuente de verdad.
Filtro de strings simple	Mostró: "This is not a bootable disk..."	Coincidencia con cadenas por defecto del Boot Sector del disco.	El texto estándar del SO es ruido. Debemos filtrar los mensajes genéricos.
Extracción YAFFS vía binwalk	Errores con yaffshiv / unyaffs.	binwalk detectó falsos positivos de firmas YAFFS en datos comprimidos.	No todas las firmas que detecta binwalk son sistemas de archivos reales.
Descompresión Zlib básica	zlib.error: incorrect data check.	dd cortó el bloque Zlib incluyendo sumas de verificación que fallaban.	Los flujos raw deflate requieren descompresión sin validación estricta (-zlib.MAX_WBITS).

🧠 4. ¿Cómo Desarrollar el Pensamiento Analítico Forense?

    Separa el Contenedor del Contenido: Un archivo .png no es solo una imagen visual; es una estructura de datos binaria con cabeceras (IHDR), bloques comprimidos (IDAT), metadatos (XMP) y marcas de fin (IEND). La evidencia puede estar en cualquiera de ellos.

    Filtra el Ruido: Tu objetivo no es leer cada byte del disco, sino descartar lo estándar del SO mediante expresiones regulares (grep) y concentrarte en el espacio no asignado y archivos borrados.

    Lee los Errores de las Herramientas: Un error de descompresión o una falta de biblioteca no significa que el camino esté mal, sino que la herramienta necesita parámetros específicos (como descompresión en crudo o ajuste de offsets).

    Crea tu Matriz de Decisión Interna:

        ¿Imagen de disco? → file / fls / autopsy.

        ¿Archivo borrado? → icat / foremost / photorec.

        ¿Archivo multimedia extraído? → exiftool / strings / binwalk.
        EOF
