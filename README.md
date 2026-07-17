# Automatizacion-Pre-render-blender
Script de automatización de pre renderizado de acciones de un modelo 3D para una vista 2D en blender como la de Clash Royale.
Se necesitara el script de sprite_sheet (https://github.com/Daro25/FtsBlender-to-Sprite_sheet.git).

Este repositorio también incluye el escenario blender para una configuración de lucez y camara parecida a la de Clash Royale

## Importaciones
```Python
import math
import os
import winsound
import bpy
import re
from PIL import Image

```
## Spreet-Sheet
```Python
# sprite-sheet
def crear_spritesheet(img_name,directorio, frame_width, frame_height, columns, eliminar_originales=False):
    """
    Genera un sprite sheet a partir de imágenes numéricas en un directorio específico.
    
    :param directorio: Ruta de la carpeta donde están las imágenes (str).
    :param frame_width: Ancho en píxeles de cada fotograma (int).
    :param frame_height: Alto en píxeles de cada fotograma (int).
    :param columns: Cantidad de columnas del sprite sheet (int).
    :param eliminar_originales: Si es True, borra las imágenes numéricas originales (bool).
    """
    # Validar que el directorio exista
    if not os.path.exists(directorio):
        print(f"Error: El directorio '{directorio}' no existe.")
        return

    # Buscar archivos numéricos dentro de la carpeta especificada
    files = [f for f in os.listdir(directorio) if re.match(r'^\d+\.png$', f)]
    
    if not files:
        print(f"No se encontraron archivos numéricos en '{directorio}' para procesar.")
        return

    # Ordenar los archivos numéricamente basándose solo en el nombre del archivo
    files.sort(key=lambda f: int(os.path.splitext(f)[0]))
    
    total_images = len(files)
    rows = math.ceil(total_images / columns)
    
    # Crear lienzo transparente
    spritesheet_size = (columns * frame_width, rows * frame_height)
    spritesheet = Image.new("RGBA", spritesheet_size, (0, 0, 0, 0))

    # Pegar fotogramas en el lienzo
    for index, file in enumerate(files):
        # Obtener la ruta completa de cada imagen
        file_path = os.path.join(directorio, file)
        
        with Image.open(file_path) as img:
            if img.size != (frame_width, frame_height):
                img = img.resize((frame_width, frame_height), Image.Resampling.NEAREST)
            
            col = index % columns
            row = index // columns
            spritesheet.paste(img, (col * frame_width, row * frame_height))

    # Guardar el archivo final en el mismo directorio especificado
    output_filename = os.path.join(directorio, str(img_name)+'.png')
    spritesheet.save(output_filename)
    print(f"¡Éxito! Guardado como '{output_filename}' ({total_images} fotogramas procesados).")

    # Limpieza automática si se activa el parámetro
    if eliminar_originales:
        for file in files:
            file_path = os.path.join(directorio, file)
            try:
                os.remove(file_path)
            except Exception as e:
                print(f"No se pudo borrar {file_path}: {e}")
        print("Fotogramas originales eliminados.")
    else:
        print("Los fotogramas originales se han conservado.")

```
## Código de automatización
```Python
def procesar_accion_render(nombre_pieza, nombre_accion, angulos, 
ftStart, ftEnd,
tipo_render, folder_base,
objeto, resolucion=[256,256]):
    """
    Procesa, rota, renderiza y crea el sprite sheet para una acción específica.
    
    :param nombre_pieza: Nombre del personaje/pieza (ej: 'Alfil')
    :param nombre_accion: Nombre de la animación (ej: 'static', 'run', 'atack')
    :param angulos: Lista de rotaciones en grados (ej: [0, 45, -45])
    :param tipo_render: 'estatico' para fotogramas únicos, 'animacion' para clips de video/secuencias
    :param folder_base: Ruta base donde se guardará todo
    :param objeto: El objeto/Armature de Blender a rotar
    :param resolucion: Tamaño en píxeles (X e Y) para el renderizado
    """
    # Si la lista de ángulos está vacía, ignoramos la acción por completo
    if not angulos or len(angulos) == 0:
        print(f"No hay ángulos definidos para la acción '{nombre_accion}'. Saltando...")
        return

    scene = bpy.context.scene
    render = scene.render
    image_settings = render.image_settings

    # Configuración global del motor de renderizado
    render.engine = 'BLENDER_EEVEE_NEXT'
    render.resolution_x = resolucion[0]
    render.resolution_y = resolucion[1]
    render.use_overwrite = True
    render.film_transparent = True
    image_settings.file_format = 'PNG'
    image_settings.color_mode = 'RGBA'
    scene.frame_start = ftStart
    scene.frame_end = ftEnd

    # Definir la carpeta destino de esta acción específica (ej: .../ajedrezBImg/staticAlfil)
    folder_accion = os.path.join(folder_base, f"{nombre_accion}{nombre_pieza}")
    
    if not os.path.exists(folder_accion):
        os.makedirs(folder_accion)

    print(f"\nIniciando proceso para: {nombre_accion.upper()} ({tipo_render})")

    # --- CASO 1: SERIE DE IMÁGENES ESTÁTICAS ---
    if tipo_render == "estatico":
        render.filepath = folder_accion
        
        for contador, rotate in enumerate(angulos):
            # Rotar el objeto en el eje Z
            objeto.rotation_euler.z = math.radians(rotate)
            
            # Formatear el nombre del archivo (ej: 0000.png, 0001.png) para que coincida con tu re.match(r'^\d+\.png$')
            nombre_archivo = f"{contador:04d}.png" 
            render.filepath = os.path.join(folder_accion, nombre_archivo)
            
            print(f"Renderizando ángulo {rotate}° -> {nombre_archivo}")
            bpy.ops.render.render(write_still=True)
        
        # Una vez tomadas todas las fotos de los ángulos, creamos el Sprite Sheet global de la acción
        print("Creando Sprite Sheet estático...")
        crear_spritesheet('sprite_sheet',folder_accion, 
        resolucion[0], resolucion[1], len(angulos), eliminar_originales=True)

    # --- CASO 2: ANIMACIÓN COMPLETA DE BLENDER ---
    elif tipo_render == "animacion":
        for rotate in angulos:
            # Rotar el objeto en el eje Z
            objeto.rotation_euler.z = math.radians(rotate)
            
            # IMPORTANTE: Para animaciones, Blender genera múltiples imágenes (0001.png, 0002.png...).
            # Creamos una subcarpeta temporal por cada ángulo para que 
            #crear_spritesheet funcione correctamente sin mezclar archivos.
            folder_angulo_temporal = os.path.join(folder_accion, f"angulo_{rotate}")
            if not os.path.exists(folder_angulo_temporal):
                os.makedirs(folder_angulo_temporal)
                
            # La ruta de Blender debe terminar con una diagonal o prefijo para que guarde los frames dentro
            render.filepath = os.path.join(folder_angulo_temporal, "")
            
            print(f"Renderizando línea de tiempo para ángulo {rotate}°...")
            bpy.ops.render.render_animation()
            
            # Calculamos cuántos fotogramas se renderizaron en esa carpeta para definir las columnas
            total_frames = (scene.frame_end - scene.frame_start) + 1
            
            print(f"Creando Sprite Sheet para animación en ángulo {rotate}°...")
            crear_spritesheet(str(rotate),folder_angulo_temporal, 
            resolucion[0], resolucion[1], total_frames, eliminar_originales=True)
            
            # Opcional: Mover o renombrar el sprite sheet final de esa subcarpeta si lo deseas ordenado de otra forma
            # O eliminar la subcarpeta vacía si eliminar_originales=True la dejó limpia
            try:
                os.rmdir(folder_angulo_temporal)
            except OSError:
                pass

```

## Usage/Examples

Usalo así:

```Python
if __name__ == "__main__":
    OBJETO_TARGET = bpy.data.objects.get('Armature.001')
    FOLDER_BASE = r"C:/Users/windo/Downloads/ajedrezBImg/"
    
    # 1. CORRECCIÓN: Claves con comillas para un formato de diccionario válido
    Acciones = [
        {
            'nombre': 'Alfil',
            'accion': 'static',
            'mode': 'estatico', 'end': 0, 'start': 0,
            'angle': [0, 45, -45, 135, -135]
        },
        {
            'nombre': 'Alfil',
            'accion': 'run',
            'mode': 'animacion', 'end': 28, 'start': 1,
            'angle': [45, -45, 135, -135]
        },
        {
            'nombre': 'Alfil',
            'accion': 'atack',
            'mode': 'animacion', 'end': 17, 'start': 1,
            'angle': [45, -45, 135, -135]
        }
    ]

    if OBJETO_TARGET is not None:
        bpy.context.scene.render.fps = 12

        for accion in Acciones:
            procesar_accion_render(
                accion['nombre'], 
                accion['accion'], 
                accion['angle'],
                accion['start'], 
                accion['end'], 
                accion['mode'],
                FOLDER_BASE, 
                OBJETO_TARGET, 
                resolucion=[256, 256]
            )
        
        # 3. CORRECCIÓN: Requiere 'import winsound' arriba en el archivo
        winsound.Beep(1000, 1000)
        print("\n ¡Proceso completado con notificación auditiva!")
    else:
        print(" Error: No se encontró 'Armature.001' en la escena.")

```
