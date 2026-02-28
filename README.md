# dup_finder

Herramienta de línea de comandos escrita en Rust para detectar archivos duplicados dentro de un árbol de directorios. Escanea recursivamente desde el directorio donde se ejecuta, identifica duplicados por contenido (no por nombre), genera un reporte en texto plano y opcionalmente permite eliminar las copias de forma interactiva.

## Cómo funciona

El proceso tiene dos pasadas para minimizar I/O innecesario:

1. **Agrupación por tamaño**: se descarta cualquier archivo con tamaño único. Un archivo con tamaño distinto a todos los demás no puede tener duplicado.
2. **Hash SHA-256**: solo los grupos con el mismo tamaño se hashean. Dos archivos son duplicados si y solo si su hash coincide.

Los resultados se ordenan por tamaño descendente (los grupos que más espacio desperdician aparecen primero) y se guardan en un archivo `.txt` en el directorio raíz del escaneo.

Los symlinks se ignoran para evitar ciclos. Los archivos vacíos también se ignoran (no aportan información útil).

## Requisitos

- Rust 1.70 o superior (edición 2021)
- Cargo

## Compilación

```bash
cargo build --release
```

El binario queda en `target/release/dup_finder`.

## Instalación global

```bash
cargo install --path .
```

Instala el binario en `~/.cargo/bin/`, que debe estar en el `$PATH` si Rust fue instalado con `rustup`. Alternativamente:

```bash
sudo cp target/release/dup_finder /usr/local/bin/
```

## Uso

```bash
# Escanear desde el directorio actual (sin filtro de tamaño)
dup_finder

# Ignorar archivos menores a N megabytes
dup_finder 10
```

El argumento numérico es el umbral en **megabytes**. Archivos con tamaño menor o igual a ese valor son excluidos del análisis. Útil para ignorar archivos pequeños y enfocarse en los que realmente consumen espacio.

## Salida

Durante la ejecución se imprime en consola un resumen del proceso:

```
Directorio base: /home/usuario/documentos
Escaneando...

Archivos encontrados: 3842
Candidatos (mismo tamano): 120 archivos en 48 grupos

=== DUPLICADOS (12 grupos) ===

[3a8f1c0e9b21] tamano: 45.30 MB | 3 copias | desperdicio: 90.60 MB
    /home/usuario/documentos/backup/video.mp4
    /home/usuario/documentos/proyectos/video.mp4
    /home/usuario/documentos/archivo/video.mp4

...

Espacio desperdiciado total: 312.40 MB
Tiempo: 4.21s
```

Al finalizar, el resultado completo se guarda en:

```
duplicados_bajo_raiz_<nombre_carpeta>.txt
```

Por ejemplo, si se ejecuta dentro de `/home/usuario/documentos`, el archivo generado es `duplicados_bajo_raiz_documentos.txt`, en el mismo directorio.

## Limpieza interactiva

Al terminar el escaneo, el programa pregunta si se desea revisar los duplicados de forma interactiva:

```
Deseas revisar y eliminar duplicados ahora? (s/n):
```

Si se responde `s`, para cada grupo se muestra:

```
[3a8f1c0e9b21] tamano: 45.30 MB | 3 copias | desperdicio: 90.60 MB
  [1] /home/usuario/documentos/backup/video.mp4
  [2] /home/usuario/documentos/proyectos/video.mp4
  [3] /home/usuario/documentos/archivo/video.mp4
  [0] No hacer nada con este grupo
Conservar cual? (0-3):
```

Se elige el número del archivo a conservar. Los demás son eliminados con `fs::remove_file`. Con `0` se salta el grupo sin modificar nada. La eliminación es **irreversible**; no hay papelera.

## Dependencias

| Crate | Versión | Uso |
|-------|---------|-----|
| `sha2` | 0.10 | Hash SHA-256 para comparación de contenido |

## Limitaciones conocidas

- La recursión en `walk_dir` usa el call stack. En sistemas de archivos con anidamiento patológico (miles de niveles) podría causar stack overflow. En uso normal no es un problema.
- El modo interactivo lee el `.txt` generado en la misma sesión; si ese archivo fue editado manualmente entre el escaneo y la limpieza, el parseo puede fallar silenciosamente.
- No hay paralelismo: el hashing es secuencial. En discos SSD con muchos archivos grandes, paralelizarlo con `rayon` reduciría el tiempo considerablemente.
- SHA-256 es correcto pero no el más rápido. `blake3` es una alternativa de mayor rendimiento con API casi idéntica.
