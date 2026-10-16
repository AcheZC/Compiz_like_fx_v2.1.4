# 🌧️ Rain Overlay for Windows

Un overlay visual para Windows inspirado en **Compiz Water**, que transforma el escritorio en diferentes ambientes dinámicos: lluvia, agua, nieve, estrellas, luciérnagas, Matrix, nubes y cascadas.

El proyecto incluye un **panel de control**, configuraciones persistentes y efectos interactivos con mouse y teclado.

## ✨ Características

### 🌊 Pantalla
Convierte el escritorio completo en una superficie de agua con ondas y distorsiones inspiradas en el plugin Water de Compiz.

### 🪟 Ventana
Simula estar detrás de un vidrio mojado:

- Gotas irregulares que se adhieren, crecen y se deslizan.
- Rastros y pequeñas gotas secundarias.
- Vidrio empañado o transparente.
- Truenos con relámpagos y sonido.
- Limpiaparabrisas.
- Interacción con el mouse para limpiar el vidrio.
- Lluvia cayendo detrás del vidrio.
- Diferentes direcciones e inclinaciones de lluvia.
- Modo dinámico según el estado de la ventana activa.

### ❄️ Nieve

- Copos con diferentes tamaños y profundidad.
- Escarcha sobre el vidrio.
- Neblina exterior.
- Viento configurable.
- Interacción con el mouse para mover los copos.
- La escarcha se limpia al interactuar y vuelve a formarse gradualmente.

### ⭐ Estrellas

Un cielo nocturno sobre el escritorio con:

- Estrellas fijas.
- Estrellas fugaces.
- Cometas.
- Auroras boreales.
- Nebulosa.
- Colores configurables.
- Diferentes direcciones de movimiento.

### ✨ Luciérnagas

Luces flotantes con movimiento suave y parpadeo:

- Cantidad configurable.
- Hasta 10 colores simultáneos.
- Diferentes tamaños y profundidad visual.

### 🟩 Matrix

Código cayendo sobre el escritorio con opciones para configurar:

- Color.
- Cantidad.
- Opacidad.
- Profundidad.
- Tamaño.
- Velocidad.
- Brillo.
- Difuminado.
- Hasta 5 mensajes ocultos de hasta 80 caracteres.

### ☁️ Nubes

Un cielo con nubes suaves tipo nebulosa:

- Cantidad y tamaño configurables.
- Opacidad individual.
- Nubes blancas o grises.
- Viento configurable.
- Diferentes direcciones del viento.
- Reflejos de diferentes colores.
- Arcoíris opcional.

### 🌈 Cascada

Una cascada que cae desde la parte superior de la pantalla con:

- Posición configurable.
- Grosor configurable.
- Cantidad de agua.
- Transparencia.
- Vapor.
- Salpicaduras.
- Arcoíris opcional.

## ⌨️ Atajos de teclado

| Atajo | Acción |
|---|---|
| `Ctrl + Ñ` | Limpiaparabrisas |
| `Ctrl + Mayús + Ñ` | Lanzar un trueno |
| `Alt + Ñ` | Lanzar una estrella fugaz o cometa |

En el modo **Matrix**, `Ctrl + Ñ` también puede hacer caer uno de los mensajes ocultos.

## ⚙️ Configuración

Las configuraciones realizadas desde el panel de control se guardan automáticamente y se recuperan al iniciar nuevamente el programa.

Archivo de configuración:

```text
%APPDATA%/RainOverlay/settings.json
```

Al minimizar el panel, la aplicación continúa ejecutándose en el **área de notificaciones de Windows**.

## 🛠️ Tecnologías

El proyecto está desarrollado en **Python** y utiliza:

- [PyQt5](https://pypi.org/project/PyQt5/) — interfaz gráfica.
- [NumPy](https://numpy.org/) — cálculos y simulaciones.
- [OpenCV](https://opencv.org/) — procesamiento y renderizado de imágenes.
- [MSS](https://pypi.org/project/mss/) — captura de pantalla.
- [pywin32](https://pypi.org/project/pywin32/) — integración con Windows.
- `winsound` — reproducción de sonido de truenos en Windows.

## 📋 Requisitos

- Windows
- Python 3.x
- Soporte para los paquetes utilizados por el proyecto.

## 📦 Instalación

Clona el repositorio:

```bash
git clone https://github.com/TU-USUARIO/TU-REPOSITORIO.git
cd TU-REPOSITORIO
```

Instala las dependencias:

```bash
pip install PyQt5 numpy opencv-python mss pywin32
```

Estas son las dependencias utilizadas por el proyecto.

## ▶️ Ejecución

Ejecuta:

```bash
python rain_overlay_windows_v214.py
```

El programa abrirá su panel de control desde el que podrás seleccionar y configurar los diferentes estilos.

## 🎨 Rendimiento

El proyecto utiliza diferentes técnicas para mantener los efectos fluidos, incluyendo procesamiento mediante NumPy y OpenCV y, cuando está disponible, aceleración mediante OpenCL.

Algunos efectos pueden consumir considerablemente más recursos dependiendo de la resolución de pantalla y de la cantidad de partículas, gotas, estrellas u otros elementos configurados.

## 📸 Efectos

El proyecto incluye actualmente:

**🌊 Agua · 🪟 Lluvia · ❄️ Nieve · ⭐ Estrellas · ✨ Luciérnagas · 🟩 Matrix · ☁️ Nubes · 🌈 Cascada**


### Rain Overlay for Windows

Un proyecto experimental de efectos visuales interactivos para transformar el escritorio de Windows en diferentes ambientes dinámicos.
