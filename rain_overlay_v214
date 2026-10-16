"""
Rain / Water overlay estilo Compiz Water — WINDOWS, con panel de control.  (v214)

Siete estilos:
  - "Pantalla": la pantalla completa se comporta como agua (ondas que
    distorsionan el escritorio, como el plugin Water de Compiz).
  - "Ventana":  la pantalla es un vidrio con lluvia "afuera":
        * gotas irregulares que se pegan, crecen al juntarse y resbalan
          a distintas velocidades dejando rastro y gotitas;
        * la ventana activa puede estar "dentro" (clara) o "fuera" (empañada);
        * truenos opcionales (relampagos + sonido) con frecuencia/intensidad;
        * el limpiaparabrisas quita gotas Y desempaña un poco;
        * clic / clic + arrastrar con el mouse desempaña el vidrio como un dedo;
        * lluvia cayendo "afuera" (detras del vidrio), delgada, activable/desactivable,
          con direccion elegible (derecha, izquierda, hacia la pantalla, en direccion
          opuesta o cambiante) e inclinacion ajustable;
        * ventana activa: "dentro" (clara), "fuera" (empañada) o "dinamica"
          (maximizada = fuera, no maximizada = dentro);
        * atajo Ctrl + Ñ: pasa el limpiaparabrisas UNA vez (otra vez Ctrl + Ñ lo detiene).
  - "Nieve":    sin lluvia: copos cayendo al fondo, escarcha (frost) realista en el
    vidrio y neblina (fog) afuera; tamaño y cantidad de copos, y viento (derecha,
    izquierda, hacia la ventana, alejandose de ella o "cambiante") con intensidad
    ajustable; con clic / arrastrar el mouse los copos se mueven como si soplaras aire
    y la escarcha se limpia como al pasar el dedo (y vuelve a formarse despacio).
  - "Estrellas": cielo nocturno sobre el escritorio: estrellas fijas (hasta 4 veces mas que
    antes, de uno o dos colores elegibles; el 1% titila),
    lluvia de estrellas fugaces (de ninguna a muchisimas), cometas, auroras
    boreales y nebulosa, con colores y direccion elegibles;
    atajo Alt + Ñ: lanza una estrella fugaz o un cometa.
  - "Luciernagas": luces que flotan y parpadean; cantidad y de 1 a 10 colores a la vez.
  - "Matrix":  codigo cayendo; color (verde, azulado, morado o mixto), cantidad,
    opacidad, profundidad (capas superpuestas), tamaño del codigo, velocidad de caida
    (fija o variable), partes del codigo que caen brillando y hasta 5 mensajes escondidos
    (de hasta 80 letras) que caen al azar entre el codigo; el brillo (del codigo y de los
    mensajes) se activa/desactiva y tiene intensidad y difuminado ajustables.
  - "Nubes":   cielo para sentir que flotas: nubes suaves tipo nebulosa; cantidad, rango
    de tamaños, opacidad (cada nube con la suya al azar), blancura (blancas o grises),
    viento (mas o menos fuerte: solo las mueve o ademas las deshace) con direccion
    elegible o cambiante, y color e intensidad del reflejo (azulado, naranja, rosa, gris,
    blanco o morado) y arcoiris opcional (posicion y opacidad elegibles).
  - "Cascada": una cascada de agua que cae desde arriba: desde que parte de la pantalla
    cae, que tan gruesa es (% del ancho), cuanta agua cae, que tan translucida es, vapor
    y salpicado al caer (activable y ajustable) y arcoiris opcional (posicion y opacidad).

Atajos globales: Ctrl + Ñ (limpiaparabrisas; en el estilo Matrix hace caer uno de los
mensajes escondidos), Ctrl + Mayus + Ñ (lanza un trueno en el
estilo Ventana) y Alt + Ñ (estrella fugaz / cometa en el estilo Estrellas).

Ajustes: todo lo que cambies en el panel se guarda solo (en
%APPDATA%/RainOverlay/settings.json) y se recupera al abrir el programa.
Bandeja: al minimizar el panel, la app desaparece de la barra de tareas y queda como un
icono junto al reloj (area de notificacion); clic en el icono para abrir el panel.

Instalar:
    pip install PyQt5 numpy opencv-python mss pywin32

Correr:
    python rain_overlay_windows_v214.py
"""

import sys
import os
import json
import time
import math
import atexit
import textwrap
import tempfile
import wave
import ctypes
from ctypes import wintypes

import numpy as np
import cv2
import mss
import win32gui
import win32con
import win32api
from PyQt5 import QtWidgets, QtCore, QtGui

try:
    import winsound
except ImportError:  # no-Windows / sin soporte de audio
    winsound = None

# ---------- Config base ----------
REFRACT_STRENGTH = 18.0
GRID_SCALE = 1          # 1 = simulacion a resolucion completa (maxima nitidez)
RENDER_DOWNSCALE = 1    # 1 = resolucion completa de pantalla (estilo Pantalla)

# Estilo Ventana: resolucion interna del vidrio. 2 = mitad (fluido), 1 = maxima
# nitidez pero ~4 veces mas pesado.
GLASS_SCALE = 2
FOG_DIV = 4             # el mapa de vaho se guarda a 1/4 de la resolucion del vidrio
DROP_UNIT = 1.0         # multiplicador del "tamaño de gota" en el estilo Ventana
FOG_REGROW = 0.0015     # que tan rapido vuelve a empañarse (por frame)
IRREGULAR_FRAC = 0.30   # fraccion de gotas con contorno MUY irregular (estilo Ventana)
FIREFLY_MAX = 400       # maximo de luciernagas (estilo Luciernagas)
STAR_MAX = 2800         # maximo de estrellas fijas (estilo Estrellas): slider 0..400 (100 = 700)
METEOR_MAX = 140        # maximo de estrellas fugaces a la vez
SNOW_MAX = 900          # maximo de copos de nieve (estilo Nieve)
RAIN_MAX = 700          # maximo de rayitas de lluvia "afuera" (detras del vidrio)

cv2.ocl.setUseOpenCL(True)  # usa GPU vía OpenCL si el driver lo soporta

with mss.mss() as sct:
    mon = sct.monitors[1]
W, H = mon["width"], mon["height"]
GW, GH = W // GRID_SCALE, H // GRID_SCALE
RW, RH = W // RENDER_DOWNSCALE, H // RENDER_DOWNSCALE
VW, VH = max(2, W // GLASS_SCALE), max(2, H // GLASS_SCALE)
FW, FH = max(2, VW // FOG_DIV), max(2, VH // FOG_DIV)
GS = 1.0 / GLASS_SCALE   # pixeles de pantalla -> pixeles de vidrio

SKIP_CLASSES = {
    "Progman", "WorkerW", "Shell_TrayWnd", "Shell_SecondaryTrayWnd",
    "XamlExplorerHostIslandWindow", "NotifyIconOverflowWindow",
}


# =====================================================================
# Utilidades
# =====================================================================
def make_drop_kernel(radius: int) -> np.ndarray:
    """Gaussiana 2D normalizada (pico=1.0) de radio dado — da gotas con bordes redondos."""
    radius = max(1, radius)
    k1d = cv2.getGaussianKernel(radius * 2 + 1, radius / 2.0)
    kernel = (k1d @ k1d.T).astype(np.float32)
    kernel /= kernel.max()
    return kernel


class WiperGeom:
    """Geometria del limpiaparabrisas (pivote abajo-centro) para una resolucion dada."""
    def __init__(self, w, h):
        self.px = w / 2.0
        self.py = float(h)
        dx_1d = np.arange(w, dtype=np.float32) - self.px
        dy_1d = np.arange(h, dtype=np.float32) - self.py
        DX, DY = np.meshgrid(dx_1d, dy_1d)
        self.radius = np.sqrt(DX ** 2 + DY ** 2)
        self.angle = np.arctan2(-DY, DX)  # 0..pi, como un reloj de izq a der
        self.arm = float(np.sqrt((w / 2.0) ** 2 + h ** 2)) * 1.02

    def mask(self, pos, half_w):
        return (self.radius <= self.arm) & (np.abs(self.angle - pos) <= half_w)


def wiper_hit_points(xs, ys, pos, half_w):
    """Que puntos (coordenadas de pantalla) toca la pala del limpiaparabrisas."""
    dx = xs - W / 2.0
    dy = ys - float(H)
    rad = np.sqrt(dx * dx + dy * dy)
    ang = np.arctan2(-dy, dx)
    arm = float(np.sqrt((W / 2.0) ** 2 + H ** 2)) * 1.02
    return (rad <= arm) & (np.abs(ang - pos) <= half_w)


# ---------- formas irregulares de gota ----------
N_PTS = 24
_THETA = np.linspace(0, 2 * np.pi, N_PTS, endpoint=False)


def make_shape(irregular: bool = False) -> np.ndarray:
    """Contorno irregular unitario (N,2): armonicos aleatorios + ruido + 'panza' abajo.
    Con irregular=True el contorno es mucho mas caprichoso (lobulos, muescas,
    alargamientos), como las gotas de agua reales que se deforman al pegarse."""
    rho = np.ones(N_PTS)
    if not irregular:
        for k in (2, 3, 4, 5, 7):
            amp = np.random.uniform(0.0, 0.26) / (k ** 0.55)
            rho += amp * np.cos(k * _THETA + np.random.uniform(0, 2 * np.pi))
        rho += np.random.normal(0, 0.05, N_PTS)
        rho = np.clip(rho, 0.4, 1.8)
        ex = np.random.uniform(0.75, 1.2)          # ancho
        sag = np.random.uniform(0.0, 0.45)         # la gota "se cae" hacia abajo
        shear = np.random.uniform(-0.18, 0.18)     # cizalla
    else:
        for k in (2, 3, 4, 5, 6, 7):
            amp = np.random.uniform(0.04, 0.40) / (k ** 0.45)
            rho += amp * np.cos(k * _THETA + np.random.uniform(0, 2 * np.pi))
        # 1-3 lobulos locales (salientes o muescas), como agua que se jalo hacia un lado
        for _ in range(int(np.random.randint(1, 4))):
            c = np.random.uniform(0, 2 * np.pi)
            w = np.random.uniform(0.35, 0.9)
            a = np.random.uniform(-0.30, 0.55)
            dth = np.angle(np.exp(1j * (_THETA - c)))
            rho += a * np.exp(-(dth / w) ** 2)
        rho += np.random.normal(0, 0.08, N_PTS)
        rho = np.clip(rho, 0.35, 2.0)
        ex = np.random.uniform(0.6, 1.5)
        sag = np.random.uniform(0.05, 0.7)
        shear = np.random.uniform(-0.32, 0.32)
    x = rho * np.cos(_THETA) * ex
    y = rho * np.sin(_THETA)
    y = np.where(y > 0, y * (1 + sag), y * (1 - 0.5 * sag))
    x = x + y * shear
    return np.stack([x, y], axis=1).astype(np.float32)


class Drop:
    """Gota sobre el vidrio. m = area (r^2). Puede estar quieta o resbalando."""
    __slots__ = ("x", "y", "m", "pin2", "grow", "slide", "v", "ang", "stall",
                 "tw", "wet", "dist_acc", "next_bead", "speedmul", "poly",
                 "ttl", "dead", "stretch")

    def __init__(self, x, y, r, pin, grow=0.0, ttl=0):
        self.x = float(x)
        self.y = float(y)
        self.m = float(r * r)
        self.pin2 = float(pin * pin)     # umbral (area) para empezar a resbalar
        self.grow = grow                 # condensacion: area que gana por frame
        self.slide = False
        self.v = 0.0
        self.ang = 0.0
        self.stall = 0
        self.tw = 0.3 * r                # semi-ancho del rastro
        self.wet = 1.0                   # "humedad" del rastro (varia)
        self.dist_acc = 0.0
        self.next_bead = np.random.uniform(10, 60)
        self.speedmul = float(np.exp(np.random.normal(0, 0.5)))  # rapidas y lentas
        self.poly = make_shape(np.random.random() < IRREGULAR_FRAC)
        self.ttl = ttl                   # >0 cuenta atras, -1 evaporandose, 0 permanente
        self.dead = False
        self.stretch = 0.0

    @property
    def r(self):
        return math.sqrt(self.m)


def start_slide(d: Drop, v0=0.4):
    d.slide = True
    d.v = v0
    d.ang = float(np.random.normal(0, 0.1))
    d.stall = 0
    d.tw = 0.22 * d.r
    d.wet = 1.0


# =====================================================================
# Sonido de trueno (sintetizado, sin archivos externos)
# =====================================================================
def synth_thunder(volume=0.8, sr=22050):
    rng = np.random.default_rng()
    dur = float(rng.uniform(3.5, 6.5))
    n = int(sr * dur)
    t = np.arange(n) / sr
    f = np.fft.rfftfreq(n, 1.0 / sr)

    def colored(f_lo, f_hi, slope):
        spec = np.fft.rfft(rng.standard_normal(n))
        g = 1.0 / np.maximum(f, 15.0) ** slope
        g = g / (1.0 + (f / f_hi) ** 6)
        g = g / (1.0 + (f_lo / np.maximum(f, 1e-3)) ** 4)
        y = np.fft.irfft(spec * g, n)
        return y / (np.max(np.abs(y)) + 1e-9)

    rumble = colored(25, 220, 0.9)
    mid = colored(120, 900, 0.5)
    mod = colored(0.2, 3.0, 0.0)
    mod = np.clip(0.5 + 0.5 * mod, 0, 1)
    env = (1 - np.exp(-t / 0.12)) * np.exp(-t / (dur * 0.35))
    crack = colored(300, 3500, 0.2) * np.exp(-t / 0.07) * 0.7 * (t < 0.5)
    audio = (rumble * 0.9 + mid * 0.35) * env * (0.4 + 0.6 * mod) + crack
    audio = audio / (np.max(np.abs(audio)) + 1e-9)
    fade = np.clip((dur - t) / 0.4, 0, 1)
    audio = audio * fade * float(np.clip(volume, 0, 1)) * 0.9
    return (audio * 32767).astype(np.int16), sr


class ThunderSound:
    def __init__(self):
        base = tempfile.gettempdir()
        self.files = [os.path.join(base, f"rain_overlay_thunder_{i}.wav") for i in range(3)]
        self.idx = 0
        atexit.register(self.cleanup)

    def play(self, volume):
        if winsound is None:
            return
        try:
            data, sr = synth_thunder(volume)
            path = self.files[self.idx % len(self.files)]
            self.idx += 1
            with wave.open(path, "wb") as wf:
                wf.setnchannels(1)
                wf.setsampwidth(2)
                wf.setframerate(sr)
                wf.writeframes(data.tobytes())
            winsound.PlaySound(path, winsound.SND_FILENAME | winsound.SND_ASYNC)
        except Exception as e:  # el sonido nunca debe tumbar el programa
            print("Sonido de trueno no disponible:", e)

    def cleanup(self):
        for p in self.files:
            try:
                os.remove(p)
            except OSError:
                pass


# =====================================================================
# Nieve (estilo Nieve)
# =====================================================================
class SnowSim:
    """Copos con profundidad z: lejos = chicos y nitidos, cerca = grandes y desenfocados.
    Todo en coordenadas del vidrio (VW x VH)."""
    def __init__(self):
        n = SNOW_MAX
        self.kx = VW / 960.0
        self.ky = VH / 540.0
        self.cx = VW / 2.0
        self.cy = VH / 2.0
        self.t = 0
        self.z = (np.random.random(n) ** 1.2).astype(np.float32)
        self.x = np.random.uniform(-0.1 * VW, 1.1 * VW, n).astype(np.float32)
        self.y = np.random.uniform(-0.1 * VH, 1.0 * VH, n).astype(np.float32)
        self.scl = np.ones(n, dtype=np.float32)
        self.vx = np.zeros(n, dtype=np.float32)
        self.vy = ((0.9 + 2.4 * self.z) * self.ky *
                   np.random.uniform(0.85, 1.15, n)).astype(np.float32)
        self.phase = np.random.uniform(0, 2 * np.pi, n).astype(np.float32)
        self.pw = np.random.uniform(0.03, 0.09, n).astype(np.float32)
        self.amp = (np.random.uniform(0.15, 0.6, n) * (0.4 + self.z) * self.kx).astype(np.float32)
        rj = np.random.uniform(0.75, 1.3, n)
        rj[np.random.random(n) < 0.05] *= 1.6                  # unos pocos copos grandes
        self.rj = rj.astype(np.float32)
        self.age = np.full(n, 99.0, dtype=np.float32)
        # viento "cambiante": racha actual (-1..1), objetivo y cuenta regresiva (frames)
        self.wv = 0.0
        self.wv_tgt = 0.0
        self.wv_timer = 0
        # velocidad vertical extra (soplo del mouse): se va apagando sola
        self.uy = np.zeros(n, dtype=np.float32)

    def variable_wind_tick(self):
        """Avanza el viento 'cambiante' (misma logica que en step) cuando no se estan
        moviendo copos pero la neblina necesita el viento."""
        self.wv_timer -= 1
        if self.wv_timer <= 0:
            self.wv_timer = int(np.random.randint(50, 240))
            if np.random.random() < 0.18:
                self.wv_tgt = float(np.random.uniform(-0.12, 0.12))
            else:
                self.wv_tgt = float(np.random.choice((-1.0, 1.0)) * np.random.uniform(0.3, 1.0))
        self.wv += (self.wv_tgt - self.wv) * 0.02

    def push(self, n, cx, cy, R, dvx, dvy, burst):
        """Soplo de aire del mouse: empuja los copos cercanos a (cx, cy) (coordenadas de vidrio)
        en la direccion del arrastre (dvx, dvy) y los abre hacia afuera (burst). Los copos
        cercanos a la ventana (z alto) vuelan mas. El empuje se apaga solo (inercia)."""
        sl = slice(0, n)
        dx = self.x[sl] - cx
        dy = self.y[sl] - cy
        d2 = dx * dx + dy * dy
        idx = np.nonzero(d2 < R * R)[0]
        if idx.size == 0:
            return
        d = np.sqrt(d2[idx]) + 1e-3
        f = ((1.0 - d / R) ** 2 * (0.55 + 0.9 * self.z[idx])).astype(np.float32)
        ux, uyy = dx[idx] / d, dy[idx] / d
        lim = 26.0 * self.kx
        self.vx[idx] = np.clip(self.vx[idx] + (dvx * 0.45 + burst * ux) * f, -lim, lim)
        self.uy[idx] = np.clip(self.uy[idx] + (dvy * 0.45 + burst * uyy) * f, -lim, lim)
        self.phase[idx] += 0.5 * f                    # y se arremolinan un poco

    def step(self, st, n):
        self.t += 1
        sl = slice(0, n)
        x, y, z = self.x[sl], self.y[sl], self.z[sl]
        scl, vx, age = self.scl[sl], self.vx[sl], self.age[sl]
        mode = st.snow_wind_dir if st.snow_wind_on else "none"
        inten = float(np.clip(st.snow_wind_strength, 1, 10)) / 10.0
        t = self.t
        # rachas: el viento sube y baja de forma natural
        gust = (0.70 + 0.25 * math.sin(t * 0.021) + 0.12 * math.sin(t * 0.057 + 1.3)
                + 0.08 * math.sin(t * 0.13))
        horizontal = mode in ("right", "left", "variable")
        wscale = 1.0      # cuanto del efecto lateral se aplica (en "cambiante" sube y baja)

        if mode == "variable":
            # viento CAMBIANTE: cada cierto tiempo (al azar) elige un lado y una fuerza
            # nuevos (a veces casi calma) y la racha se acerca suave a ese objetivo:
            # sopla a un lado, se alenta, se va al otro, se intensifica, etc.
            self.wv_timer -= 1
            if self.wv_timer <= 0:
                self.wv_timer = int(np.random.randint(50, 240))
                if np.random.random() < 0.18:
                    self.wv_tgt = float(np.random.uniform(-0.12, 0.12))        # casi calma
                else:
                    self.wv_tgt = float(np.random.choice((-1.0, 1.0))
                                        * np.random.uniform(0.3, 1.0))
            self.wv += (self.wv_tgt - self.wv) * 0.02
            micro = 0.88 + 0.12 * math.sin(t * 0.11) + 0.07 * math.sin(t * 0.29 + 0.7)
            wscale = min(1.0, abs(self.wv))
            tgt = self.wv * micro * inten * 11.0 * self.kx * (0.45 + 0.9 * z)
        elif horizontal:
            sign = 1.0 if mode == "right" else -1.0
            tgt = sign * inten * gust * 11.0 * self.kx * (0.45 + 0.9 * z)
        else:
            tgt = 0.0
        vx += (tgt - vx) * 0.07                       # inercia: no cambia de golpe
        self.phase[sl] += self.pw[sl]
        sway = np.cos(self.phase[sl]) * self.amp[sl]  # vaiven natural de cada copo
        x += vx + sway * (1.0 + (1.5 * inten * wscale if horizontal else 0.0))
        uy = self.uy[sl]
        y += self.vy[sl] + uy                         # caida + soplo del mouse
        uy *= 0.90
        if horizontal:
            y += (np.random.normal(0, 0.25 * inten * wscale, n) * self.ky).astype(np.float32)

        if mode in ("toward", "away"):
            k = (0.012 * inten * gust * (0.4 + 1.2 * z)).astype(np.float32)
            sc = (1.0 + k) if mode == "toward" else (1.0 - k)
            x[:] = self.cx + (x - self.cx) * sc
            y[:] = self.cy + (y - self.cy) * sc
            scl *= sc
        else:
            scl += (1.0 - scl) * 0.03
        age += 1.0

        # ---- reaparicion ----
        if mode == "toward":
            bad = ((scl > 3.4) | (x < -0.25 * VW) | (x > 1.25 * VW)
                   | (y > 1.25 * VH) | (y < -0.25 * VH))
            m = int(bad.sum())
            if m:
                x[bad] = np.random.uniform(0.02 * VW, 0.98 * VW, m)
                y[bad] = np.random.uniform(0.02 * VH, 0.98 * VH, m)
                scl[bad] = np.random.uniform(0.35, 0.9, m)
                age[bad] = 0.0
                vx[bad] = 0.0
        elif mode == "away":
            bad = ((scl < 0.3) | (y > 1.1 * VH) | (x < -0.25 * VW) | (x > 1.25 * VW))
            m = int(bad.sum())
            if m:
                x[bad] = np.random.uniform(0, VW, m)
                y[bad] = -np.random.uniform(0, 0.1 * VH, m)
                scl[bad] = np.random.uniform(1.0, 2.2, m)
                age[bad] = 0.0
                vx[bad] = 0.0
        else:
            rad = (1.0 + 3.0 * z) * self.rj[sl] * scl * self.kx * (st.snow_size / 4.0)
            bad = (y - rad) > VH
            m = int(bad.sum())
            if m:
                y[bad] = -np.random.uniform(0, 0.1 * VH, m) - rad[bad]
                x[bad] = np.random.uniform(-0.1 * VW, 1.1 * VW, m)
                age[bad] = 0.0
            left = x < -0.15 * VW
            right = x > 1.15 * VW
            x[left] += 1.3 * VW
            x[right] -= 1.3 * VW

    def draw(self, st, n):
        """Capa de copos (uint8, resolucion de vidrio). Cada capa de profundidad
        se desenfoca distinto (los cercanos fuera de foco)."""
        sl = slice(0, n)
        x, y, z, scl, age = self.x[sl], self.y[sl], self.z[sl], self.scl[sl], self.age[sl]
        r = (1.0 + 3.0 * z) * self.rj[sl] * scl * self.kx * (st.snow_size / 4.0)
        fade = (np.clip(age / 20.0, 0, 1) * np.clip((scl - 0.15) / 0.2, 0, 1)
                * np.clip((3.4 - scl) / 0.8, 0, 1))
        val = (255.0 * fade * (0.7 + 0.3 * z)).astype(np.int32)
        vis = ((x > -r * 2) & (x < VW + r * 2) & (y > -r * 2) & (y < VH + r * 2) & (val > 2))
        px = (x * 16).astype(np.int32)
        py = (y * 16).astype(np.int32)
        pr = np.maximum(r * 16, 6).astype(np.int32)
        layers = []
        for lo, hi, sigma in ((0.0, 0.4, 0.5), (0.4, 0.75, 0.9), (0.75, 1.01, 2.0)):
            sel = np.nonzero(vis & (z >= lo) & (z < hi))[0]
            img = np.zeros((VH, VW), dtype=np.uint8)
            xs, ys, rs, vs = px[sel].tolist(), py[sel].tolist(), pr[sel].tolist(), val[sel].tolist()
            for i in range(len(xs)):
                cv2.circle(img, (xs[i], ys[i]), rs[i], vs[i], -1, cv2.LINE_AA, 4)
            layers.append(cv2.GaussianBlur(img, (0, 0), sigma * self.kx))
        far, mid, near = layers
        return cv2.add(cv2.addWeighted(far, 0.85, mid, 1.0, 0),
                       cv2.convertScaleAbs(near, alpha=1.0))


# =====================================================================
# Estrellas (estilo Estrellas)
# =====================================================================
# colores en BGR
STAR_COLORS = {
    "verde": (110, 255, 130),
    "azul": (255, 175, 70),
    "morado": (235, 90, 180),
    "rosa": (200, 130, 255),
    "amarillo": (90, 225, 255),
    "blanco": (255, 248, 245),
}
STAR_COLOR_NAMES = ["verde", "azul", "morado", "rosa", "amarillo", "blanco"]


def pick_star_color(cfg):
    """Un color para un objeto (estrella fugaz / cometa) segun (color1, color2)."""
    c1, c2 = cfg
    if c1 == "aleatorio":
        return STAR_COLORS[STAR_COLOR_NAMES[int(np.random.randint(len(STAR_COLOR_NAMES)))]]
    if c2 not in (None, "ninguno") and c2 != c1:
        return STAR_COLORS[c1 if np.random.random() < 0.5 else c2]
    return STAR_COLORS[c1]


def color_pair(cfg):
    """Par de colores (base, punta) para auroras / nebulosa."""
    c1, c2 = cfg
    if c1 == "aleatorio":
        i, j = np.random.choice(len(STAR_COLOR_NAMES), 2, replace=False)
        return STAR_COLORS[STAR_COLOR_NAMES[int(i)]], STAR_COLORS[STAR_COLOR_NAMES[int(j)]]
    if c2 in (None, "ninguno") or c2 == c1:
        return STAR_COLORS[c1], STAR_COLORS[c1]
    return STAR_COLORS[c1], STAR_COLORS[c2]


def fbm_noise(rng, w, h, base, octs, persistence=0.5):
    """Ruido fractal 0..1 de tamaño (h, w)."""
    out = np.zeros((h, w), dtype=np.float32)
    amp, tot, cell = 1.0, 0.0, float(base)
    for _ in range(octs):
        gh, gw = max(2, int(h / cell) + 2), max(2, int(w / cell) + 2)
        r = rng.random((gh, gw)).astype(np.float32)
        out += amp * cv2.resize(r, (w, h), interpolation=cv2.INTER_CUBIC)
        tot += amp
        amp *= persistence
        cell = max(2.0, cell / 2.0)
    out /= tot
    lo, hi = float(out.min()), float(out.max())
    return (out - lo) / max(hi - lo, 1e-6)


class Meteor:
    __slots__ = ("x", "y", "vx", "vy", "acc", "age", "life", "tailk", "col", "mode")


class Comet:
    __slots__ = ("x", "y", "vx", "vy", "acc", "L", "r", "curv", "col", "wait", "mode")


class SkySim:
    """Cielo: nebulosa + aurora + estrellas fijas + estrellas fugaces + cometas.
    Todo se acumula en una imagen 'premultiplicada' a resolucion de vidrio."""
    def __init__(self):
        self.kx = VW / 960.0
        self.ky = VH / 540.0
        self.cx = VW / 2.0
        self.cy = VH / 2.0
        self.diag = math.hypot(VW, VH)
        self.t = 0
        self.P = np.zeros((VH, VW, 3), dtype=np.float32)
        rng = np.random

        # ---- estrellas fijas ----
        n = STAR_MAX
        self.sz = (rng.random(n) ** 1.5).astype(np.float32)
        self.sx = rng.uniform(0, VW, n).astype(np.float32)
        self.sy = rng.uniform(0, VH, n).astype(np.float32)
        self.sb = (rng.uniform(0.35, 1.0, n) * (0.55 + 0.45 * self.sz)).astype(np.float32)
        self.sph = rng.uniform(0, 2 * np.pi, n).astype(np.float32)
        self.spw = rng.uniform(0.04, 0.16, n).astype(np.float32)
        self.sage = np.full(n, 99.0, dtype=np.float32)
        tint = rng.random(n).astype(np.float32)[:, None]
        cool = np.array([255, 240, 225], dtype=np.float32)
        warm = np.array([190, 225, 255], dtype=np.float32)
        self.scol = (cool * (1 - tint) + warm * tint).astype(np.float32)
        # solo el 1% titila de verdad (una de cada 100); el resto brilla fijo
        self.stw = (np.arange(n) % 100) == 37
        self.sbright = (rng.random(n) < 0.12) | self.stw
        # colores elegibles de las estrellas fijas
        self.scol_nat = self.scol.copy()
        self.sci = rng.randint(0, len(STAR_COLOR_NAMES), n)
        self.sr2 = rng.random(n).astype(np.float32)
        self.sjit = rng.uniform(0.85, 1.0, n).astype(np.float32)
        self._star_key = None

        self.meteors = []
        self.comets = []
        self.extra_comets = []          # cometas lanzados con el atajo (pasan una sola vez)

        # ---- aurora ----
        self.AW, self.AH = max(8, VW // 3), max(8, VH // 3)
        self.au = (np.arange(self.AW, dtype=np.float32) / (self.AW - 1))[None, :]
        self.av = (np.arange(self.AH, dtype=np.float32) / (self.AH - 1))[:, None]
        self.auroras = []
        self._aur_key = None

        # ---- nebulosa (texturas a 1/2 de la resolucion de vidrio) ----
        self.NW, self.NH = max(8, VW // 2), max(8, VH // 2)
        kn = self.NW / 480.0
        g = np.random.default_rng()
        self.neb_tex = []
        for _ in range(2):
            nz = fbm_noise(g, self.NW, self.NH, 110 * kn, 5, 0.55)
            tex = np.clip((nz - 0.30) / 0.55, 0, 1) ** 1.5
            det = 0.7 + 0.3 * fbm_noise(g, self.NW, self.NH, 24 * kn, 3)
            self.neb_tex.append((tex * det).astype(np.float32))
        self.nph = 0.0
        self.neb_cols = None
        self._neb_key = None

    # ------------------------------------------------------------------
    # nebulosa: flujo infinito con dos copias en desfase (crossfade)
    # ------------------------------------------------------------------
    def _flow_matrix(self, q, mode, shift, cx, cy, w):
        if mode == "right":
            return np.float32([[1, 0, q * shift * w], [0, 1, 0]])
        if mode == "left":
            return np.float32([[1, 0, -q * shift * w], [0, 1, 0]])
        sc = 1.0 + 0.7 * q if mode == "toward" else 1.7 - 0.7 * q
        return np.float32([[sc, 0, cx * (1 - sc)], [0, sc, cy * (1 - sc)]])

    def _nebula(self, st, mode, dt):
        key = st.sky_colors["nebula"]
        if key != self._neb_key:
            self._neb_key = key
            a, b = color_pair(key)
            if a == b:                       # un solo color: capa 2 mas clara
                b = tuple(int(c + (255 - c) * 0.35) for c in a)
            self.neb_cols = (np.array(a, dtype=np.float32), np.array(b, dtype=np.float32))
        self.nph = (self.nph + dt / 70.0) % 1.0
        acc = np.zeros((self.NH, self.NW, 3), dtype=np.float32)
        for li in (0, 1):
            tex = self.neb_tex[li]
            col = self.neb_cols[li]
            shift = 0.30 if li == 0 else 0.45
            for off in (0.0, 0.5):
                q = (self.nph + off + 0.25 * li) % 1.0
                wgt = math.sin(math.pi * q) ** 2
                M = self._flow_matrix(q, mode, shift, self.NW / 2.0, self.NH / 2.0, self.NW)
                w = cv2.warpAffine(tex, M, (self.NW, self.NH), flags=cv2.INTER_LINEAR,
                                   borderMode=cv2.BORDER_REFLECT)
                acc += (w * (wgt * 0.34))[..., None] * col
        return cv2.resize(acc, (VW, VH), interpolation=cv2.INTER_LINEAR)

    # ------------------------------------------------------------------
    # aurora boreal: cortinas con rayos verticales que ondulan
    # ------------------------------------------------------------------
    def _make_auroras(self, st):
        self.auroras = []
        for _ in range(int(np.clip(st.sky_aurora_n, 1, 4))):
            base, top = color_pair(st.sky_colors["aurora"])
            U = np.random.uniform
            self.auroras.append(dict(
                y0=U(0.14, 0.42), a1=U(0.04, 0.10), f1=U(0.7, 1.8), p1=U(0, 1), s1=U(0.0006, 0.0016),
                a2=U(0.015, 0.04), f2=U(2.5, 4.5), p2=U(0, 1), s2=U(0.001, 0.003),
                H=U(0.16, 0.30), fr=U(9, 18), p3=U(0, 1), p4=U(0, 1), s3=U(0.001, 0.003),
                p5=U(0, 6.28),
                base=np.array(base, dtype=np.float32), top=np.array(top, dtype=np.float32)))

    def _aurora(self, st, mode, dt):
        key = (st.sky_colors["aurora"], int(st.sky_aurora_n))
        if key != self._aur_key:
            self._aur_key = key
            self._make_auroras(st)
        sgn = -1.0 if mode == "left" else 1.0
        t = self.t
        u, v = self.au, self.av
        img = np.zeros((self.AH, self.AW, 3), dtype=np.float32)
        tp = 2 * np.pi
        for a in self.auroras:
            yb = (a["y0"] + a["a1"] * np.sin(tp * (a["f1"] * u + a["p1"] + sgn * a["s1"] * t))
                  + a["a2"] * np.sin(tp * (a["f2"] * u + a["p2"] - sgn * a["s2"] * t)))
            h = yb - v                                           # >0: por encima de la base
            prof = np.exp(-np.maximum(h, 0) / a["H"] + np.minimum(h, 0) / 0.02)
            r1 = np.sin(tp * (a["fr"] * u + a["p3"] + sgn * a["s3"] * t))
            r2 = np.sin(tp * (a["fr"] * 0.37 * u + a["p4"] - a["s3"] * 0.6 * t))
            rays = np.clip(0.55 + 0.45 * r1 * (0.6 + 0.4 * r2), 0.08, 1.0)
            shimmer = 0.78 + 0.22 * math.sin(t * 0.025 + a["p5"])
            inten = prof * (0.30 + 0.70 * rays) * shimmer
            mix = np.clip(h / (a["H"] * 1.6), 0, 1)[..., None]
            col = a["base"] * (1 - mix) + a["top"] * mix
            img += inten[..., None] * col * 0.5
        if mode in ("toward", "away"):
            out = np.zeros_like(img)
            for off in (0.0, 0.5):
                q = (self.nph + off) % 1.0
                sc = 1.0 + 0.6 * q if mode == "toward" else 1.6 - 0.6 * q
                px, py = self.AW / 2.0, 0.1 * self.AH
                M = np.float32([[sc, 0, px * (1 - sc)], [0, sc, py * (1 - sc)]])
                w = cv2.warpAffine(img, M, (self.AW, self.AH), flags=cv2.INTER_LINEAR,
                                   borderMode=cv2.BORDER_REFLECT)
                out += w * (math.sin(math.pi * q) ** 2)
            img = out
        return cv2.resize(img, (VW, VH), interpolation=cv2.INTER_LINEAR)

    # ------------------------------------------------------------------
    # estrellas fijas (titilan y se desplazan despacio segun la direccion)
    # ------------------------------------------------------------------
    def _apply_star_colors(self, st):
        """Color de las estrellas fijas: natural, uno, dos colores o aleatorio."""
        cfg = tuple(st.sky_colors.get("stars", ("natural", "ninguno")))
        if cfg == self._star_key:
            return
        self._star_key = cfg
        c1, c2 = cfg
        if c1 == "natural" or (c1 != "aleatorio" and c1 not in STAR_COLORS):
            self.scol = self.scol_nat.copy()
        elif c1 == "aleatorio":
            pal = np.array([STAR_COLORS[nm] for nm in STAR_COLOR_NAMES], dtype=np.float32)
            self.scol = (pal[self.sci] * self.sjit[:, None]).astype(np.float32)
        else:
            b1 = np.array(STAR_COLORS[c1], dtype=np.float32)
            if c2 in (None, "ninguno") or c2 == c1 or c2 not in STAR_COLORS:
                cols = np.tile(b1, (len(self.sr2), 1))
            else:
                b2 = np.array(STAR_COLORS[c2], dtype=np.float32)
                cols = np.where((self.sr2 < 0.5)[:, None], b1, b2)
            self.scol = (cols * self.sjit[:, None]).astype(np.float32)

    def _stars(self, n, mode, P):
        sl = slice(0, n)
        x, y, z, age = self.sx[sl], self.sy[sl], self.sz[sl], self.sage[sl]
        age += 1.0
        self.sph[sl] += self.spw[sl]
        if mode in ("right", "left"):
            sgn = 1.0 if mode == "right" else -1.0
            x += sgn * (0.04 + 0.22 * z) * self.kx
            x[x > VW + 2] -= VW + 4
            x[x < -2] += VW + 4
        else:
            k = (0.0012 * (0.3 + z)).astype(np.float32)
            sc = (1.0 + k) if mode == "toward" else (1.0 - k)
            x[:] = self.cx + (x - self.cx) * sc
            y[:] = self.cy + (y - self.cy) * sc
            if mode == "toward":
                bad = (x < 0) | (x >= VW) | (y < 0) | (y >= VH)
            else:
                bad = (np.hypot(x - self.cx, y - self.cy) < 0.03 * VW)
            m = int(bad.sum())
            if m:
                x[bad] = np.random.uniform(0, VW - 1, m)
                y[bad] = np.random.uniform(0, VH - 1, m)
                age[bad] = 0.0
        # fijas: brillo estable; el 1% titila (de casi apagada a toda su luz)
        tw = np.where(self.stw[sl], 0.08 + 0.92 * (0.5 + 0.5 * np.sin(self.sph[sl] * 1.6)), 0.78)
        b = self.sb[sl] * tw * np.clip(age / 25.0, 0, 1)
        xi = x.astype(np.int32)
        yi = y.astype(np.int32)
        ok = (xi >= 1) & (xi < VW - 1) & (yi >= 1) & (yi < VH - 1)
        xi, yi, b = xi[ok], yi[ok], b[ok]
        col = self.scol[sl][ok]
        P[yi, xi] += col * b[:, None]
        br = self.sbright[sl][ok]
        if br.any():                                   # destello en cruz de las mas brillantes
            xb, yb, cb = xi[br], yi[br], (col[br] * (b[br] * 0.45)[:, None])
            for dx, dy in ((1, 0), (-1, 0), (0, 1), (0, -1)):
                P[yb + dy, xb + dx] += cb

    # ------------------------------------------------------------------
    # estrellas fugaces y cometas
    # ------------------------------------------------------------------
    def _new_meteor(self, st, mode):
        m = Meteor()
        m.col = pick_star_color(st.sky_colors["meteor"])
        m.mode = mode
        m.age = 0
        U = np.random.uniform
        if mode in ("right", "left"):
            sgn = 1.0 if mode == "right" else -1.0
            th = math.radians(U(12, 48))
            v = U(16, 32) * self.kx
            m.vx, m.vy = sgn * v * math.cos(th), v * math.sin(th)
            if np.random.random() < 0.75:
                m.x = U(-0.15 * VW, 0.85 * VW) if sgn > 0 else U(0.15 * VW, 1.15 * VW)
                m.y = -U(0, 0.08 * VH)
            else:
                m.x = -0.05 * VW if sgn > 0 else 1.05 * VW
                m.y = U(0, 0.5 * VH)
            m.life = int(U(18, 48))
            m.acc = 1.0
            m.tailk = U(3.5, 7.0)
        elif mode == "toward":
            ang = U(0, 2 * np.pi)
            r0 = U(0.03, 0.3) * VW * 0.5
            m.x, m.y = self.cx + r0 * math.cos(ang), self.cy + r0 * math.sin(ang)
            v = U(1.5, 4.0) * self.kx
            m.vx, m.vy = v * math.cos(ang), v * math.sin(ang)
            m.acc = 1.09
            m.life = 400
            m.tailk = U(3.0, 6.0)
        else:                                          # away
            ang = U(0, 2 * np.pi)
            R = U(0.45, 0.8) * self.diag / 2.0
            m.x, m.y = self.cx + R * math.cos(ang), self.cy + R * math.sin(ang)
            v = U(18, 32) * self.kx
            m.vx, m.vy = -v * math.cos(ang), -v * math.sin(ang)
            m.acc = 0.93
            m.life = 400
            m.tailk = U(3.0, 6.0)
        return m

    def _step_meteor(self, m):
        m.x += m.vx
        m.y += m.vy
        m.vx *= m.acc
        m.vy *= m.acc
        m.age += 1
        if m.age >= m.life:
            return False
        sp = math.hypot(m.vx, m.vy)
        if m.mode == "away" and sp < 1.5 * self.kx:
            return False
        if (m.x < -0.35 * VW or m.x > 1.35 * VW or m.y < -0.35 * VH or m.y > 1.35 * VH):
            return m.age < 3
        return True

    def _draw_meteor(self, M, m):
        sp = math.hypot(m.vx, m.vy)
        if sp < 1e-3:
            return
        ux, uy = m.vx / sp, m.vy / sp
        if m.mode in ("right", "left"):
            b = math.sin(math.pi * min(1.0, m.age / m.life)) ** 0.5
        else:
            b = min(1.0, m.age / 5.0)
            if m.mode == "away":
                b *= min(1.0, sp / (5.0 * self.kx))
        L = sp * m.tailk
        col = m.col
        nseg = 7
        for i in range(nseg):
            f0, f1 = i / nseg, (i + 1) / nseg
            br = b * (1.0 - f0) ** 1.6
            c = (int(col[0] * br), int(col[1] * br), int(col[2] * br))
            p0 = (int((m.x - ux * L * f0) * 16), int((m.y - uy * L * f0) * 16))
            p1 = (int((m.x - ux * L * f1) * 16), int((m.y - uy * L * f1) * 16))
            th = max(1, int(round(1.6 * self.kx))) if i < 3 else 1
            cv2.line(M, p0, p1, c, th, cv2.LINE_AA, 4)
        hc = (int(col[0] * 0.4 + 255 * 0.6 * b), int(col[1] * 0.4 + 255 * 0.6 * b),
              int(col[2] * 0.4 + 255 * 0.6 * b))
        cv2.circle(M, (int(m.x * 16), int(m.y * 16)), int(1.5 * self.kx * 16), hc, -1,
                   cv2.LINE_AA, 4)

    def _new_comet(self, st, mode, initial=False):
        c = Comet()
        U = np.random.uniform
        c.col = pick_star_color(st.sky_colors["comet"])
        c.mode = mode
        c.wait = int(U(0, 120)) if initial else int(U(60, 500))
        c.L = U(150, 280) * self.kx
        c.r = U(3.0, 5.0) * self.kx
        c.curv = U(-0.25, 0.25)
        if mode in ("right", "left"):
            sgn = 1.0 if mode == "right" else -1.0
            th = math.radians(U(6, 30))
            v = U(0.9, 2.2) * self.kx
            c.vx, c.vy = sgn * v * math.cos(th), v * math.sin(th)
            c.acc = 1.0
            c.y = U(0.05, 0.7) * VH
            if initial:
                c.x = U(0, VW)
            else:
                c.x = -0.12 * VW if sgn > 0 else 1.12 * VW
        elif mode == "toward":
            ang = U(0, 2 * np.pi)
            r0 = U(0.03, 0.25) * VW * 0.5
            c.x, c.y = self.cx + r0 * math.cos(ang), self.cy + r0 * math.sin(ang)
            v = U(0.5, 1.2) * self.kx
            c.vx, c.vy = v * math.cos(ang), v * math.sin(ang)
            c.acc = 1.006
        else:
            ang = U(0, 2 * np.pi)
            R = U(0.4, 0.7) * self.diag / 2.0
            c.x, c.y = self.cx + R * math.cos(ang), self.cy + R * math.sin(ang)
            v = U(1.5, 3.0) * self.kx
            c.vx, c.vy = -v * math.cos(ang), -v * math.sin(ang)
            c.acc = 0.994
        return c

    def _step_comet(self, c, st, mode):
        if c.wait > 0:
            c.wait -= 1
            return True
        c.x += c.vx
        c.y += c.vy
        c.vx *= c.acc
        c.vy *= c.acc
        if c.mode in ("right", "left"):
            if c.vx > 0 and c.x > VW + c.L + 20:
                return False
            if c.vx < 0 and c.x < -c.L - 20:
                return False
            return c.y < VH + c.L
        r = math.hypot(c.x - self.cx, c.y - self.cy)
        if c.mode == "toward":
            return (-0.1 * VW < c.x < 1.1 * VW) and (-0.1 * VH < c.y < 1.1 * VH)
        return r > 0.04 * VW

    def _draw_comet(self, M, c):
        if c.wait > 0:
            return
        sp = math.hypot(c.vx, c.vy)
        if sp < 1e-3:
            return
        ux, uy = c.vx / sp, c.vy / sp
        nx, ny = -uy, ux
        L, rh = c.L, c.r
        if c.mode in ("toward", "away"):
            rr = math.hypot(c.x - self.cx, c.y - self.cy) / (0.5 * VW)
            sc = 0.45 + 0.9 * min(rr, 1.4)
            L *= sc
            rh *= (0.6 + 0.6 * min(rr, 1.4))
        col = c.col
        N = 24
        for kind, (curv, dim) in enumerate(((c.curv, 1.0), (-c.curv * 0.6, 0.55))):
            pts = []
            for i in range(N + 1):
                f = i / N
                lat = curv * L * f * f
                pts.append((c.x - ux * L * f + nx * lat, c.y - uy * L * f + ny * lat))
            for i in range(N):
                f = i / N
                br = dim * (1.0 - f) ** 1.7
                cc = (int(col[0] * br), int(col[1] * br), int(col[2] * br))
                th = max(1, int(round(((1 - f) * rh * (1.0 if kind == 0 else 0.6)) + 0.5)))
                a = (int(pts[i][0] * 16), int(pts[i][1] * 16))
                b = (int(pts[i + 1][0] * 16), int(pts[i + 1][1] * 16))
                cv2.line(M, a, b, cc, th, cv2.LINE_AA, 4)
        hx, hy = int(c.x * 16), int(c.y * 16)
        halo = (int(col[0] * 0.35), int(col[1] * 0.35), int(col[2] * 0.35))
        cv2.circle(M, (hx, hy), int(rh * 2.2 * 16), halo, -1, cv2.LINE_AA, 4)
        core = (int(col[0] * 0.3 + 255 * 0.7), int(col[1] * 0.3 + 255 * 0.7),
                int(col[2] * 0.3 + 255 * 0.7))
        cv2.circle(M, (hx, hy), int(rh * 16), core, -1, cv2.LINE_AA, 4)

    def launch_one(self, st):
        """Atajo Alt + Ñ: aparece UNA estrella fugaz o un cometa (al azar)."""
        mode = st.sky_dir
        if np.random.random() < 0.6:
            if len(self.meteors) < METEOR_MAX:
                self.meteors.append(self._new_meteor(st, mode))
        elif len(self.extra_comets) < 6:
            c = self._new_comet(st, mode)
            c.wait = 0
            self.extra_comets.append(c)

    def _shooters(self, st, mode, dt):
        inten = int(st.sky_meteors)
        if inten > 0:
            rate = 0.05 * (500.0 ** ((inten - 1) / 99.0))      # estrellas fugaces por segundo
            k = int(np.random.poisson(rate * dt))
            for _ in range(min(k, 8)):
                if len(self.meteors) < METEOR_MAX:
                    self.meteors.append(self._new_meteor(st, mode))
        nc = int(st.sky_comets_n) if st.sky_comets_on else 0
        while len(self.comets) < nc:
            self.comets.append(self._new_comet(st, mode, initial=True))
        del self.comets[nc:]
        self.meteors = [m for m in self.meteors if self._step_meteor(m)]
        alive = []
        for c in self.comets:
            if c.mode != mode:                         # cambio de direccion: reaparece
                c = self._new_comet(st, mode, initial=True)
            if not self._step_comet(c, st, mode):
                c = self._new_comet(st, mode)
            alive.append(c)
        self.comets = alive
        self.extra_comets = [c for c in self.extra_comets if self._step_comet(c, st, mode)]
        if (not self.meteors and not self.extra_comets
                and not any(c.wait <= 0 for c in self.comets)):
            return None
        M = np.zeros((VH, VW, 3), dtype=np.uint8)
        for m in self.meteors:
            self._draw_meteor(M, m)
        for c in self.comets:
            self._draw_comet(M, c)
        for c in self.extra_comets:
            self._draw_comet(M, c)
        glow = cv2.GaussianBlur(M, (0, 0), 3.0 * self.kx)
        return cv2.add(M, cv2.convertScaleAbs(glow, alpha=1.6))

    # ------------------------------------------------------------------
    def render(self, st):
        """Devuelve la imagen BGRA (uint8, resolucion de vidrio) del cielo."""
        self.t += 1
        dt = max(10, st.interval_ms) / 1000.0
        mode = st.sky_dir
        P = self.P
        P.fill(0.0)
        if st.sky_nebula_on:
            P += self._nebula(st, mode, dt)
        else:
            self.nph = (self.nph + dt / 70.0) % 1.0
        if st.sky_aurora_on:
            P += self._aurora(st, mode, dt)
        n = int(np.clip(st.sky_stars, 0, 400) * STAR_MAX / 400)
        if n > 0:
            self._apply_star_colors(st)
            self._stars(n, mode, P)
        M = self._shooters(st, mode, dt)
        if M is not None:
            P += M.astype(np.float32)

        alpha = np.clip(P.max(axis=2) * (1.0 / 255.0), 0.0, 1.0)
        col = P / np.maximum(alpha, 0.004)[..., None]
        out = np.empty((VH, VW, 4), dtype=np.uint8)
        np.clip(col, 0, 255, out=col)
        out[:, :, :3] = col.astype(np.uint8)
        out[:, :, 3] = (alpha * 255.0).astype(np.uint8)
        return out


# =====================================================================
# Luciernagas / Matrix / Nubes (estilos nuevos)
# =====================================================================
def premult_to_bgra(P):
    """Imagen 'premultiplicada' (VH,VW,3 float, 0..255) -> BGRA uint8 (alpha = brillo)."""
    a = np.maximum(np.maximum(P[:, :, 0], P[:, :, 1]), P[:, :, 2])
    inv = 255.0 / np.clip(a, 1.0, 255.0)
    out = np.empty((VH, VW, 4), dtype=np.uint8)
    out[:, :, :3] = cv2.convertScaleAbs(P * inv[..., None])
    out[:, :, 3] = cv2.convertScaleAbs(np.clip(a, 0.0, 255.0))
    return out


def premult8_to_bgra(P8, opacity=1.0):
    """Igual que premult_to_bgra pero para imagenes uint8 (mas rapido) y con opacidad global."""
    b, g, r = cv2.split(P8)
    a = cv2.max(cv2.max(b, g), r)
    out = np.empty((VH, VW, 4), dtype=np.uint8)
    out[:, :, :3] = cv2.divide(P8, cv2.merge([a, a, a]), scale=255.0)
    out[:, :, 3] = cv2.convertScaleAbs(a, alpha=float(opacity))
    return out


# ---------------------------------------------------------------------
# Luciernagas
# ---------------------------------------------------------------------
FIREFLY_COLORS = {          # BGR
    "amarillo": (70, 225, 255),
    "blanco": (255, 250, 245),
    "azul": (255, 170, 70),
    "verde": (110, 255, 130),
    "naranja": (40, 150, 255),
    "morado": (235, 90, 180),
    "rosa": (200, 130, 255),
    "rojo": (60, 60, 255),
    "dorado": (40, 190, 235),
    "plateado": (225, 215, 205),
}
FIREFLY_COLOR_NAMES = ["amarillo", "blanco", "azul", "verde", "naranja",
                       "morado", "rosa", "rojo", "dorado", "plateado"]


class FireflySim:
    """Luces que flotan con trayectorias suaves y parpadean. Cada una tiene profundidad:
    las lejanas son chicas y nitidas, las cercanas grandes y desenfocadas (bokeh)."""
    def __init__(self):
        n = FIREFLY_MAX
        self.kx = VW / 960.0
        rng = np.random
        self.x = rng.uniform(0, VW, n).astype(np.float32)
        self.y = rng.uniform(0, VH, n).astype(np.float32)
        self.z = (rng.random(n) ** 1.1).astype(np.float32)
        self.ang = rng.uniform(0, 2 * np.pi, n).astype(np.float32)
        self.spd = ((0.25 + 0.9 * self.z) * self.kx * rng.uniform(0.6, 1.3, n)).astype(np.float32)
        self.turn = rng.uniform(0.03, 0.10, n).astype(np.float32)
        self.ph = rng.uniform(0, 2 * np.pi, n).astype(np.float32)       # parpadeo
        self.pw = rng.uniform(0.025, 0.09, n).astype(np.float32)
        self.bob = rng.uniform(0, 2 * np.pi, n).astype(np.float32)      # cabeceo
        self.fcol = np.zeros((n, 3), dtype=np.float32)
        self._names = None
        self.P = np.zeros((VH, VW, 3), dtype=np.float32)

    def _assign_colors(self, names):
        idx = np.random.randint(0, len(names), FIREFLY_MAX)
        self.fcol = np.array([FIREFLY_COLORS[names[i]] for i in idx], dtype=np.float32)

    def render(self, st):
        names = tuple(c for c in FIREFLY_COLOR_NAMES if c in st.fly_colors) or ("amarillo",)
        if names != self._names:
            self._names = names
            self._assign_colors(names)
        n = int(np.clip(st.fly_amount, 1, FIREFLY_MAX))
        sl = slice(0, n)
        x, y, z = self.x[sl], self.y[sl], self.z[sl]
        kx = self.kx

        # --- movimiento: vuelo suave con giros lentos y cabeceo ---
        self.ang[sl] += np.random.normal(0, 1.0, n).astype(np.float32) * self.turn[sl]
        self.bob[sl] += 0.05
        sp = self.spd[sl] * (0.55 + 0.45 * np.sin(self.bob[sl]))
        x += np.cos(self.ang[sl]) * sp
        y += np.sin(self.ang[sl]) * sp * 0.8 + 0.22 * np.sin(self.bob[sl] * 0.7) * kx * (0.4 + z)
        m = 60.0 * kx
        x[x < -m] += VW + 2 * m
        x[x > VW + m] -= VW + 2 * m
        y[y < -m] += VH + 2 * m
        y[y > VH + m] -= VH + 2 * m

        # --- parpadeo ---
        self.ph[sl] += self.pw[sl]
        b = 0.18 + 0.82 * (0.5 + 0.5 * np.sin(self.ph[sl])) ** 2.5

        sizef = (st.fly_size / 4.0) * kx
        r = (0.9 + 3.4 * z) * sizef * (1.0 + 0.8 * np.clip((z - 0.6) / 0.4, 0, 1))
        px = (x * 16).astype(np.int32)
        py = (y * 16).astype(np.int32)
        pr = np.maximum(r * 16, 8).astype(np.int32)
        ph_r = (pr * 3).astype(np.int32)
        col = self.fcol[sl]
        core_c = ((col * 0.8 + 255 * 0.2) * b[:, None]).astype(np.int32)
        halo_c = (col * (0.30 * b)[:, None]).astype(np.int32)
        vis = (x > -4 * r) & (x < VW + 4 * r) & (y > -4 * r) & (y < VH + 4 * r)

        sel_all = np.nonzero(vis)[0]
        halo_img = np.zeros((VH, VW, 3), dtype=np.uint8)           # halos: una sola capa suave
        xs, ys, hs, hc = (px[sel_all].tolist(), py[sel_all].tolist(),
                          ph_r[sel_all].tolist(), halo_c[sel_all].tolist())
        for i in range(len(xs)):
            cv2.circle(halo_img, (xs[i], ys[i]), hs[i], hc[i], -1, cv2.LINE_AA, 4)
        halo_img = cv2.GaussianBlur(halo_img, (0, 0), 5.0 * kx)

        layers = []
        for lo, hi, sigma in ((0.0, 0.4, 0.6), (0.4, 0.75, 1.2), (0.75, 1.01, 3.0)):
            sel = np.nonzero(vis & (z >= lo) & (z < hi))[0]
            img = np.zeros((VH, VW, 3), dtype=np.uint8)
            xs, ys = px[sel].tolist(), py[sel].tolist()
            rs, cc = pr[sel].tolist(), core_c[sel].tolist()
            for i in range(len(xs)):                       # nucleos (bokeh en las cercanas)
                cv2.circle(img, (xs[i], ys[i]), rs[i], cc[i], -1, cv2.LINE_AA, 4)
            layers.append(cv2.GaussianBlur(img, (0, 0), sigma * kx))
        tot = cv2.add(cv2.add(layers[0], layers[1]), cv2.add(layers[2], halo_img))
        self.P[:] = tot
        return premult_to_bgra(self.P)


# ---------------------------------------------------------------------
# Matrix (codigo cayendo)
# ---------------------------------------------------------------------
MATRIX_CHARSETS = {
    "katakana": "ｱｲｳｴｵｶｷｸｹｺｻｼｽｾｿﾀﾁﾂﾃﾄﾅﾆﾇﾈﾉﾊﾋﾌﾍﾎﾏﾐﾑﾒﾓﾔﾕﾖﾗﾘﾙﾚﾛﾜﾝ0123456789:.=*+-<>",
    "numeros": "0123456789",
    "binario": "01",
}
# (color de la estela, color de la cabeza) en BGR
MATRIX_PALETTES = [
    ((60, 235, 45), (205, 255, 205)),      # verde
    ((255, 190, 40), (255, 240, 205)),     # azulado
    ((235, 70, 170), (255, 205, 240)),     # morado
]
MATRIX_COLOR_IDX = {"verde": 0, "azulado": 1, "morado": 2}
_ATLAS_BASE = 32
MX_MSG_MAX = 80         # maximo de letras de cada mensaje escondido (estilo Matrix)


def build_glyph_atlas(chars, cell=_ATLAS_BASE):
    """Mascaras (n, cell, cell) uint8 de cada caracter. Usa Qt (MS Gothic en Windows, que
    tiene katakana); si algo falla, cae a las letras simples de OpenCV."""
    n = len(chars)
    try:
        font = QtGui.QFont("MS Gothic")
        font.setPixelSize(max(6, int(cell * 0.85)))
        font.setStyleStrategy(QtGui.QFont.PreferAntialias)
        img = QtGui.QImage(cell * n, cell, QtGui.QImage.Format_ARGB32_Premultiplied)
        img.fill(0)
        pt = QtGui.QPainter(img)
        pt.setRenderHint(QtGui.QPainter.TextAntialiasing, True)
        pt.setFont(font)
        pt.setPen(QtGui.QColor(255, 255, 255))
        for i, ch in enumerate(chars):
            pt.drawText(QtCore.QRect(i * cell, 0, cell, cell), QtCore.Qt.AlignCenter, ch)
        pt.end()
        ptr = img.constBits()
        nbytes = img.sizeInBytes() if hasattr(img, "sizeInBytes") else img.byteCount()
        ptr.setsize(nbytes)
        arr = np.frombuffer(ptr, dtype=np.uint8).reshape(cell, img.bytesPerLine() // 4, 4)
        strip = np.ascontiguousarray(arr[:, :cell * n, 3])
        atlas = strip.reshape(cell, n, cell).transpose(1, 0, 2).copy()
        if int(atlas.max()) == 0:
            raise ValueError("atlas vacio")
        return atlas
    except Exception:
        return _fallback_atlas(chars, cell)


def _fallback_atlas(chars, cell):
    """Respaldo sin Qt: caracteres ASCII con cv2.putText."""
    pool = "ABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789"
    atlas = np.zeros((len(chars), cell, cell), dtype=np.uint8)
    for i, ch in enumerate(chars):
        c = ch if ord(ch) < 128 else pool[int(np.random.randint(len(pool)))]
        cv2.putText(atlas[i], c, (int(cell * 0.12), int(cell * 0.82)),
                    cv2.FONT_HERSHEY_SIMPLEX, cell / 34.0, 255, 1, cv2.LINE_AA)
    return atlas


class MatrixSim:
    """Columnas de caracteres que caen, en varias capas de profundidad."""
    def __init__(self):
        self.kx = VW / 960.0
        self.P8 = np.zeros((VH, VW, 3), dtype=np.uint8)
        self.base_atlas = {}
        self.layers = []
        self._key = None
        self.t = 0
        # velocidad variable (a veces rapido, a veces despacio)
        self.spd_cur = 1.0
        self.spd_tgt = 1.0
        self.spd_timer = 0
        # mensajes escondidos que caen dentro del codigo
        self.msgs = []
        self.next_msg = 60
        self.msg_base = {}
        self.msg_glyphs = {}

    def _atlas(self, name):
        if name not in self.base_atlas:
            self.base_atlas[name] = build_glyph_atlas(MATRIX_CHARSETS[name])
        return self.base_atlas[name]

    @staticmethod
    def _size_factor(st):
        """Tamaño del codigo: 5 = como antes (x1.0); 1 = x0.4; 10 = x3.2."""
        return 2.0 ** ((float(np.clip(st.mx_size, 1, 10)) - 5.0) / 3.0)

    @staticmethod
    def _new_stream(L, idx):
        L["head"][idx] = -np.random.uniform(0, L["rows"] * 0.8, len(idx))
        L["speed"][idx] = np.random.uniform(0.7, 1.4, len(idx)) * L["base_speed"]
        L["trail"][idx] = np.random.uniform(7, 24, len(idx))
        L["cidx"][idx] = np.random.randint(0, 3, len(idx))
        L["gl"][idx] = np.random.random(len(idx)) < 0.35      # algunas columnas caen brillando

    def _build(self, st):
        """(Re)crea las capas cuando cambian la profundidad, el tamaño o los caracteres."""
        name = st.mx_chars if st.mx_chars in MATRIX_CHARSETS else "katakana"
        base = self._atlas(name)
        nch = base.shape[0]
        strip = base.transpose(1, 0, 2).reshape(_ATLAS_BASE, nch * _ATLAS_BASE)
        depth = int(np.clip(st.mx_depth, 1, 4))
        fs = self._size_factor(st)
        self.layers = []
        self.msgs = []
        for i in range(depth):
            t = 1.0 if depth == 1 else i / (depth - 1.0)            # 0 lejos .. 1 cerca
            cell = max(6, int(round((10 + 12 * t) * fs * self.kx)))
            rows, cols = int(math.ceil(VH / cell)) + 1, int(math.ceil(VW / cell))
            gs = cv2.resize(strip, (nch * cell, cell), interpolation=cv2.INTER_AREA)
            atlas = np.ascontiguousarray(gs.reshape(cell, nch, cell).transpose(1, 0, 2))
            # la velocidad se compensa con el tamaño: en pantalla cae igual de rapido
            L = dict(cell=cell, rows=rows, cols=cols, atlas=atlas, nch=nch,
                     gain=0.35 + 0.65 * t, base_speed=(0.25 + 0.35 * t) / fs,
                     head=np.zeros(cols), speed=np.zeros(cols), trail=np.zeros(cols),
                     cidx=np.zeros(cols, dtype=np.int32),
                     rnd=np.random.random(cols),
                     chars=np.random.randint(0, nch, (rows, cols)),
                     gimg=None, gl=np.zeros(cols, dtype=bool))
            self._new_stream(L, np.arange(cols))
            L["head"] = np.random.uniform(-rows * 0.8, rows * 1.2, cols)   # ya en marcha
            self.layers.append(L)
        self._key = (name, depth, int(np.clip(st.mx_size, 1, 10)))

    def _glyph_image(self, L):
        cell, rows, cols = L["cell"], L["rows"], L["cols"]
        G = L["atlas"][L["chars"]]                                    # (rows, cols, cell, cell)
        L["gimg"] = np.ascontiguousarray(G.transpose(0, 2, 1, 3).reshape(rows * cell, cols * cell))

    # ----- mensajes escondidos -----
    def _msg_mask(self, ch, cell):
        """Mascara (cell x cell) de una letra cualquiera del mensaje (cualquier caracter)."""
        key = (ch, cell)
        m = self.msg_glyphs.get(key)
        if m is None:
            base = self.msg_base.get(ch)
            if base is None:
                if ch.strip():
                    base = build_glyph_atlas(ch)[0]
                else:
                    base = np.zeros((_ATLAS_BASE, _ATLAS_BASE), dtype=np.uint8)
                self.msg_base[ch] = base
            m = cv2.resize(base, (cell, cell), interpolation=cv2.INTER_AREA)
            self.msg_glyphs[key] = m
        return m

    def _start_message(self, msgs):
        """Hace empezar a caer un mensaje (al azar entre los escritos)."""
        text = msgs[int(np.random.randint(len(msgs)))][:MX_MSG_MAX]
        li = int(np.random.randint(len(self.layers)))
        L = self.layers[li]
        # un mensaje largo no cabe en una sola columna: se reparte en columnas contiguas
        # (se lee de arriba a abajo y de izquierda a derecha); uno corto sigue en una sola
        per = max(5, L["rows"] - 2)
        chunks = textwrap.wrap(text, width=per, break_long_words=True,
                               break_on_hyphens=False) or [text]
        chunks = chunks[:max(1, L["cols"])]
        col = int(np.random.randint(max(1, L["cols"] - len(chunks) + 1)))
        self.msgs.append(dict(layer=li, col=col, chunks=chunks, head=-1.0))

    def launch_message(self, st):
        """Atajo Ctrl + Ñ: hace caer YA uno de los mensajes escondidos (si hay alguno escrito).
        Se puede repetir: pueden caer varios a la vez (maximo 4)."""
        msgs = [m.strip() for m in (st.mx_messages or []) if isinstance(m, str) and m.strip()]
        if not msgs or not self.layers:
            return False
        self.msgs = self.msgs[-3:]
        self._start_message(msgs)
        return True

    def _update_message(self, msgs):
        """De vez en cuando (al azar) un mensaje empieza a caer, como una columna mas."""
        if self.msgs or not self.layers:
            return
        self.next_msg -= 1
        if self.next_msg > 0 or not msgs:
            return
        self._start_message(msgs)
        self.next_msg = int(np.random.randint(150, 520))              # cada ~5 a 17 s

    def render(self, st):
        name = st.mx_chars if st.mx_chars in MATRIX_CHARSETS else "katakana"
        key = (name, int(np.clip(st.mx_depth, 1, 4)), int(np.clip(st.mx_size, 1, 10)))
        if key != self._key:
            self._build(st)
        self.t += 1
        P8 = self.P8
        P8.fill(0)
        frac = 0.12 + 0.88 * float(np.clip(st.mx_amount, 0, 100)) / 100.0
        tmul = 0.5 + float(np.clip(st.mx_amount, 0, 100)) / 100.0
        fixed = MATRIX_COLOR_IDX.get(st.mx_color, -1)               # -1 = mixto

        # velocidad de caida: slider (5 = como antes) y, si se pide, variable al azar
        spd = 2.0 ** ((float(np.clip(st.mx_speed, 1, 10)) - 5.0) / 2.5)
        if st.mx_speed_var:
            self.spd_timer -= 1
            if self.spd_timer <= 0:
                self.spd_timer = int(np.random.randint(40, 220))
                self.spd_tgt = float(math.exp(np.random.uniform(math.log(0.2), math.log(2.4))))
            self.spd_cur += (self.spd_tgt - self.spd_cur) * 0.03
            spd *= self.spd_cur
        else:
            self.spd_cur = self.spd_tgt = 1.0

        msgs = [m.strip() for m in (st.mx_messages or []) if isinstance(m, str) and m.strip()]
        self._update_message(msgs)

        # capa de brillo (a 1/4 de resolucion): destellos del codigo y letras de los mensajes
        GL = np.zeros((max(2, VH // 4), max(2, VW // 4), 3), dtype=np.uint8)
        gfx, gfy = GL.shape[1] / float(VW), GL.shape[0] / float(VH)
        gl_used = False

        for li, L in enumerate(self.layers):
            rows, cols, cell = L["rows"], L["cols"], L["cell"]
            L["head"] += L["speed"] * spd
            done = np.nonzero(L["head"] - 3.0 * L["trail"] * tmul > rows)[0]
            if done.size:
                self._new_stream(L, done)
            if self.t % 6 == 0 or L["gimg"] is None:               # los caracteres cambian
                flip = np.random.random((rows, cols)) < 0.04
                L["chars"][flip] = np.random.randint(0, L["nch"], int(flip.sum()))
                self._glyph_image(L)
            R = np.arange(rows, dtype=np.float32)[:, None]
            d = (L["head"][None, :] - R).astype(np.float32)
            B = np.where(d >= 0, np.exp(-np.maximum(d, 0) / (L["trail"][None, :] * tmul)), 0.0)
            B = B * (L["rnd"][None, :] < frac) * (L["gain"] * 255.0 * 1.3)
            B8 = np.clip(B, 0, 255).astype(np.uint8)
            # caracter de la cabeza (si va a mas de 1 fila por frame, abarca lo que avanza)
            T = ((d >= 0) & (d < np.maximum(1.0, L["speed"] * spd)[None, :])
                 ).astype(np.float32)[..., None]
            pal = np.full(cols, fixed, dtype=np.int32) if fixed >= 0 else L["cidx"]
            trail_c = np.array([MATRIX_PALETTES[k][0] for k in pal], dtype=np.float32)
            head_c = np.array([MATRIX_PALETTES[k][1] for k in pal], dtype=np.float32)
            cg8 = np.clip(trail_c[None, :, :] + (head_c - trail_c)[None, :, :] * T,
                          0, 255).astype(np.uint8)

            # --- mensajes escondidos: columnas que caen con las letras del mensaje ---
            live = []
            for mg in [m for m in self.msgs if m["layer"] == li]:
                mg["head"] += L["base_speed"] * 0.7 * spd           # un poco mas lento: se alcanza a leer
                mc = max(len(c) for c in mg["chunks"])
                if int(math.floor(mg["head"])) - (mc - 1) >= rows:  # ya salio por abajo
                    self.msgs.remove(mg)
                    self._glyph_image(L)                            # borra las letras de las columnas
                else:
                    live.append(mg)
            for mg in live:
                chunks = mg["chunks"]
                mc = max(len(c) for c in chunks)
                lead = int(math.floor(mg["head"]))                  # fila de la ultima letra
                glow_on = bool(st.mx_msg_glow)
                base_b = 255.0 * 1.3 * max(L["gain"], 0.7)
                for ci, text in enumerate(chunks):
                    col = mg["col"] + ci
                    if col >= cols:
                        break
                    B8[:, col] = 0                                  # la columna normal se apaga
                    tc, hc = MATRIX_PALETTES[int(pal[col])]
                    for j, ch in enumerate(text):
                        r = lead - (mc - 1) + j
                        if 0 <= r < rows and ch != " ":
                            if glow_on:                             # brillo estable
                                lvl = 1.0
                                w = 1.0 if j == mc - 1 else 0.75
                            else:
                                lvl = 0.55 + 0.25 * (j / float(max(1, mc - 1)))
                                w = 1.0 if j == mc - 1 else 0.35    # un poco mas claras que el codigo
                            B8[r, col] = int(min(255.0, base_b * lvl))
                            cg8[r, col] = [int(tc[c] + (hc[c] - tc[c]) * w) for c in range(3)]
                            L["gimg"][r * cell:(r + 1) * cell, col * cell:(col + 1) * cell] = \
                                self._msg_mask(ch, cell)
                            if glow_on:                             # halo alrededor de la letra
                                cx = int((col * cell + cell * 0.5) * gfx)
                                cy = int((r * cell + cell * 0.5) * gfy)
                                rad = max(2, int(cell * gfx * 0.9))
                                cv2.circle(GL, (cx, cy), rad,
                                           tuple(int(hc[c] * 0.6) for c in range(3)),
                                           -1, cv2.LINE_AA)
                                gl_used = True

            # --- brillo continuo: algunas columnas caen brillando (sigue a la cabeza y a la estela) ---
            if st.mx_glow_on and L["gl"].any():
                emit = cg8.astype(np.float32) * ((B8.astype(np.float32) * (1.0 / 255.0)) ** 2)[..., None]
                emit *= L["gl"][None, :, None]
                ew, eh = max(1, int(cols * cell * gfx)), max(1, int(rows * cell * gfy))
                es = cv2.resize(np.clip(emit, 0, 255).astype(np.uint8), (ew, eh),
                                interpolation=cv2.INTER_LINEAR)
                h2, w2 = min(eh, GL.shape[0]), min(ew, GL.shape[1])
                GL[:h2, :w2] = cv2.add(GL[:h2, :w2], es[:h2, :w2])
                gl_used = True

            h, w = rows * cell, cols * cell
            Bpix = cv2.resize(B8, (w, h), interpolation=cv2.INTER_NEAREST)
            Cpix = cv2.resize(cg8, (w, h), interpolation=cv2.INTER_NEAREST)
            M = cv2.multiply(L["gimg"], Bpix, scale=1.0 / 255.0)
            contrib = cv2.multiply(cv2.merge([M, M, M]), Cpix, scale=1.0 / 255.0)
            hh, ww = min(h, VH), min(w, VW)
            P8[:hh, :ww] = cv2.add(P8[:hh, :ww], contrib[:hh, :ww])
        if gl_used:                                                 # brillo del codigo y de los mensajes
            sig = 0.8 + 0.5 * float(np.clip(st.mx_glow_blur, 1, 10))      # difuminado
            GL = cv2.GaussianBlur(GL, (0, 0), sig)
            wgt = 0.09 * float(np.clip(st.mx_glow_int, 1, 10))            # intensidad
            P8[:] = cv2.addWeighted(P8, 1.0, cv2.resize(GL, (VW, VH), interpolation=cv2.INTER_LINEAR),
                                    wgt, 0)
        # brillo (glow) barato: se desenfoca una version reducida y se suma
        small = cv2.resize(P8, (max(2, VW // 4), max(2, VH // 4)), interpolation=cv2.INTER_AREA)
        small = cv2.GaussianBlur(small, (0, 0), 2.0)
        glow = cv2.resize(small, (VW, VH), interpolation=cv2.INTER_LINEAR)
        P8[:] = cv2.addWeighted(P8, 1.0, glow, 0.9, 0)
        return premult8_to_bgra(P8, float(np.clip(st.mx_opacity, 0, 100)) / 100.0)


# ---------------------------------------------------------------------
# Nubes
# ---------------------------------------------------------------------
# color del "reflejo" de la luz sobre las nubes y el cielo (BGR)
CLOUD_REFL = {
    "azulado": (255, 180, 110),
    "naranja": (60, 150, 255),
    "rosa": (190, 140, 255),
    "gris": (175, 175, 175),
    "blanco": (255, 255, 255),
    "morado": (235, 110, 170),
}
CLOUD_REFL_NAMES = tuple(CLOUD_REFL)
CLOUD_DIRS = ("right", "left", "toward", "away", "variable")


def tile_fbm(rng, w, h, base, octs, persistence=0.55):
    """Ruido fractal 0..1 que se repite SIN costuras (asi puede deslizarse sin fin)."""
    out = np.zeros((h, w), dtype=np.float32)
    amp, tot, cell = 1.0, 0.0, float(base)
    for _ in range(octs):
        gh, gw = max(2, int(round(h / cell))), max(2, int(round(w / cell)))
        r = rng.random((gh, gw)).astype(np.float32)
        big = cv2.resize(np.tile(r, (3, 3)), (3 * w, 3 * h), interpolation=cv2.INTER_CUBIC)
        out += amp * big[h:2 * h, w:2 * w]
        tot += amp
        amp *= persistence
        cell = max(2.0, cell / 2.0)
    out /= tot
    lo, hi = float(out.min()), float(out.max())
    return (out - lo) / max(hi - lo, 1e-6)


def tile_fbm2(rng, w, h, bx, by, octs, persistence=0.55):
    """Como tile_fbm pero con celdas distintas en x (bx) y en y (by): da hebras alargadas."""
    out = np.zeros((h, w), dtype=np.float32)
    amp, tot, cx, cy = 1.0, 0.0, float(bx), float(by)
    for _ in range(octs):
        gh, gw = max(2, int(round(h / cy))), max(2, int(round(w / cx)))
        r = rng.random((gh, gw)).astype(np.float32)
        big = cv2.resize(np.tile(r, (3, 3)), (3 * w, 3 * h), interpolation=cv2.INTER_CUBIC)
        out += amp * big[h:2 * h, w:2 * w]
        tot += amp
        amp *= persistence
        cx, cy = max(2.0, cx / 2.0), max(2.0, cy / 2.0)
    out /= tot
    lo, hi = float(out.min()), float(out.max())
    return (out - lo) / max(hi - lo, 1e-6)


def periodic_noise_1d(rng, n, cell, octs=3):
    """Ruido suave que se repite, de largo n, con valores de -1 a 1."""
    out = np.zeros(n, dtype=np.float32)
    amp, tot, c = 1.0, 0.0, float(cell)
    for _ in range(octs):
        g = max(2, int(round(n / c)))
        r = rng.random(g).astype(np.float32)
        big = cv2.resize(np.tile(r, 3).reshape(1, -1), (3 * n, 1),
                         interpolation=cv2.INTER_CUBIC).reshape(-1)
        out += amp * big[n:2 * n]
        tot += amp
        amp *= 0.5
        c = max(2.0, c / 2.0)
    out /= tot
    lo, hi = float(out.min()), float(out.max())
    return ((out - lo) / max(hi - lo, 1e-6) * 2.0 - 1.0).astype(np.float32)


def wrap_shift(tex, dx, dy):
    """Desplaza una textura que se repite (dx, dy en px, con decimales)."""
    h, w = tex.shape
    M = np.float32([[1, 0, dx], [0, 1, dy]])
    try:
        return cv2.warpAffine(tex, M, (w, h), flags=cv2.INTER_LINEAR, borderMode=cv2.BORDER_WRAP)
    except cv2.error:
        return np.roll(np.roll(tex, int(round(dy)), axis=0), int(round(dx)), axis=1)


class RainbowLayer:
    """Arcoiris suave (premultiplicado, a resolucion del lienzo). Se calcula solo cuando
    cambia la posicion; la opacidad la aplica quien lo usa."""
    STOPS_S = np.array([0.0, 0.17, 0.33, 0.50, 0.67, 0.83, 1.0], dtype=np.float32)
    STOPS_C = np.array([[70, 70, 255], [60, 140, 255], [70, 225, 255], [100, 220, 110],
                        [255, 200, 90], [255, 110, 70], [230, 90, 170]], dtype=np.float32)

    def __init__(self, cw, ch):
        self.cw, self.ch = cw, ch
        self._key = None
        self._P = None
        self._A = None

    def get(self, px, py):
        """px: 0 izquierda .. 100 derecha (centro del arco); py: 0 arriba .. 100 abajo (la cima)."""
        key = (int(px), int(py))
        if key != self._key:
            self._key = key
            self._build(key[0] / 100.0, key[1] / 100.0)
        return self._P, self._A

    def _build(self, fx, fy):
        cw, ch = self.cw, self.ch
        k = cw / 480.0
        R = 0.40 * cw                       # radio del arco
        T = 0.085 * cw                      # grosor de la franja de colores
        cx, cy = fx * cw, fy * ch + R       # la cima del arco queda en fy
        xx, yy = np.meshgrid(np.arange(cw, dtype=np.float32), np.arange(ch, dtype=np.float32))
        r = np.hypot(xx - cx, yy - cy)
        s = (R - r) / T                     # 0 = borde rojo (afuera) .. 1 = violeta (adentro)
        a = np.clip(s / 0.18, 0, 1) * np.clip((1.0 - s) / 0.18, 0, 1)
        a = a * a * (3.0 - 2.0 * a)
        a *= np.clip((cy - yy) / (0.28 * R), 0, 1)       # se desvanece hacia las patas
        a = cv2.GaussianBlur(a, (0, 0), 1.4 * k)
        sc = np.clip(s, 0, 1)
        col = np.stack([np.interp(sc, self.STOPS_S, self.STOPS_C[:, c]) for c in range(3)],
                       axis=-1).astype(np.float32)
        A = (a * 0.85).astype(np.float32)
        self._A = A
        self._P = col * A[..., None]


class Cloud:
    __slots__ = ("x", "y", "u", "a0", "gj", "blobs", "vk", "p", "life", "sc")


class CloudSim:
    """Nubes suaves tipo NEBULOSA (sin bordes duros): cada nube es una envolvente de manchas
    suaves; su interior es ruido fractal que el viento arrastra (y estira) y la luz las
    modela desde arriba, con un reflejo de color. Con viento fuerte se deshacen."""

    K_E, K_N1, K_N2, K_THR, K_RAMP, K_GATE = 0.85, 1.25, 0.60, 0.40, 0.85, 1.8   # forma de la nube
    K_WARP = 40.0         # cuanto deforma el ruido el contorno (px del lienzo a 480 de ancho)
    K_LIGHT = (0.95, 0.30, 0.28)     # luz: base, relieve (arriba claro), sombra del grosor

    def __init__(self):
        self.CW, self.CH = max(16, VW // 2), max(9, VH // 2)             # lienzo
        self.EW, self.EH = max(8, self.CW // 2), max(6, self.CH // 2)    # envolventes
        self.k = self.CW / 480.0
        g = np.random.default_rng()
        self.T1 = tile_fbm(g, self.CW, self.CH, 90 * self.k, 5, 0.55)    # masas grandes
        self.T2 = tile_fbm(g, self.CW, self.CH, 26 * self.k, 4, 0.55)    # jirones finos
        self.xs = np.arange(self.EW, dtype=np.float32)
        self.ys = np.arange(self.EH, dtype=np.float32)
        self.env = np.zeros((self.EH, self.EW), dtype=np.float32)
        self.envg = np.zeros((self.EH, self.EW), dtype=np.float32)
        self.gx, self.gy = np.meshgrid(np.arange(self.CW, dtype=np.float32),
                                       np.arange(self.CH, dtype=np.float32))
        self.yrel = (np.arange(self.CH, dtype=np.float32) / (self.CH - 1))[:, None] * \
            np.ones((1, self.CW), dtype=np.float32)
        self.clouds = []
        self.nx = 0.0                 # desplazamiento del ruido (px del lienzo)
        self.zph = 0.0                # fase del zoom (acercarse / alejarse)
        self.wd = 1.0                 # direccion actual del viento cambiante (-1..1)
        self.wd_tgt = 1.0
        self.wd_timer = 0
        self._veil_key = None
        self._veil = None
        self.rainbow = RainbowLayer(self.CW, self.CH)

    # ----- utilidades -----
    def _warp(self, tex, M):
        try:
            return cv2.warpAffine(tex, M, (self.CW, self.CH), flags=cv2.INTER_LINEAR,
                                  borderMode=cv2.BORDER_WRAP)
        except cv2.error:
            return np.roll(np.roll(tex, int(round(M[1, 2])), axis=0), int(round(M[0, 2])), axis=1)

    @staticmethod
    def _smooth(x):
        x = min(max(x, 0.0), 1.0)
        return x * x * (3.0 - 2.0 * x)

    def _sizes(self, st):
        lo = float(np.clip(min(st.cl_size_min, st.cl_size_max), 3, 100)) / 100.0
        hi = float(np.clip(max(st.cl_size_min, st.cl_size_max), 3, 100)) / 100.0
        return lo, hi

    def _spawn(self, st, c, initial, mode, shred):
        rng = np.random
        c.u = float(rng.random())                    # 0 pequeña y lejana .. 1 grande y cercana
        c.a0 = float(rng.uniform(0.35, 1.0))         # opacidad propia de esta nube
        c.gj = float(rng.uniform(-0.15, 0.15))       # esta nube un poco mas gris o mas blanca
        c.vk = (0.45 + 0.9 * c.u) * float(rng.uniform(0.85, 1.15))
        c.sc = 1.0
        # forma: base ancha y plana + algunos "bultos" arriba (mas grandes al centro)
        blobs = [(0.0, 0.03, 0.34, 0.075, 1.0)]
        for _ in range(int(rng.randint(3, 6))):
            ox = float(rng.uniform(-0.26, 0.26))
            sx = float(rng.uniform(0.10, 0.20)) * (1.15 - abs(ox))
            blobs.append((ox, -float(rng.uniform(0.02, 0.10)), sx,
                          sx * float(rng.uniform(0.55, 0.85)), float(rng.uniform(0.7, 1.0))))
        c.blobs = blobs
        shr = min(max(shred, 0.0), 1.0)
        c.life = (3600.0 + (600.0 - 3600.0) * shr) * float(rng.uniform(0.6, 1.4))
        c.p = float(rng.uniform(0.15, 0.85)) if initial else float(rng.uniform(0.0, 0.08))
        lo, hi = self._sizes(st)
        w = (lo + (hi - lo) * c.u) * self.CW
        if mode in ("toward", "away"):
            ang = float(rng.uniform(0, 2 * np.pi))
            if mode == "toward":
                c.sc = float(rng.uniform(0.35, 0.8)) if not initial else float(rng.uniform(0.4, 1.8))
                rad = float(rng.uniform(0.02, 0.45 if initial else 0.2)) * math.hypot(self.CW, self.CH) * 0.5
            else:
                c.sc = float(rng.uniform(1.4, 2.1)) if not initial else float(rng.uniform(0.5, 2.0))
                rad = float(rng.uniform(0.35 if not initial else 0.05, 0.7)) * math.hypot(self.CW, self.CH) * 0.5
            c.x = self.CW * 0.5 + math.cos(ang) * rad
            c.y = self.CH * 0.5 + math.sin(ang) * rad * 0.7
        else:
            c.y = float(rng.uniform(0.08, 0.92)) * self.CH
            if initial or mode == "variable":
                c.x = float(rng.uniform(-0.1 * self.CW, 1.1 * self.CW))
            else:
                c.x = -w * 0.9 if mode == "right" else self.CW + w * 0.9

    def _veil_field(self):
        """Resplandor del cielo (sol arriba + horizonte abajo), se calcula una vez."""
        if self._veil is None:
            yy = self.yrel
            xx = (np.arange(self.CW, dtype=np.float32) / (self.CW - 1))[None, :] * \
                np.ones((self.CH, 1), dtype=np.float32)
            sun = np.exp(-(((xx - 0.60) / 0.45) ** 2 + ((yy - 0.18) / 0.40) ** 2))
            hor = yy ** 2.2
            v = 0.60 * sun + 0.45 * hor
            self._veil = (v / float(v.max())).astype(np.float32)
        return self._veil

    # ----- un frame -----
    def render(self, st):
        CW, CH = self.CW, self.CH
        n = int(np.clip(st.cl_amount, 1, 30))
        mode = st.cl_dir if st.cl_dir in CLOUD_DIRS else "right"
        w10 = float(np.clip(st.cl_wind, 0, 10))
        base = 0.02 + 0.9 * (w10 / 10.0) ** 1.6            # px del lienzo por frame
        shred = (max(0.0, (w10 - 3.0) / 7.0)) ** 1.2        # 0 = solo se mueven .. 1 = se deshacen
        lo, hi = self._sizes(st)

        # ---- direccion del viento ----
        if mode == "variable":
            self.wd_timer -= 1
            if self.wd_timer <= 0:
                self.wd_timer = int(np.random.randint(120, 520))
                if np.random.random() < 0.15:
                    self.wd_tgt = float(np.random.uniform(-0.12, 0.12))        # casi calma
                else:
                    self.wd_tgt = float(np.random.choice((-1.0, 1.0)) * np.random.uniform(0.35, 1.0))
            self.wd += (self.wd_tgt - self.wd) * 0.012
            dirv = self.wd
        elif mode == "left":
            dirv = -1.0
        elif mode == "right":
            dirv = 1.0
        else:
            dirv = 0.0
        zoom_sgn = 1.0 if mode == "toward" else (-1.0 if mode == "away" else 0.0)
        zr = 0.0006 + 0.0035 * (w10 / 10.0)                 # crecimiento por frame al acercarse

        # ---- nubes: alta / baja y movimiento ----
        while len(self.clouds) < n:
            c = Cloud()
            self._spawn(st, c, True, mode, shred)
            self.clouds.append(c)
        del self.clouds[n:]
        cx0, cy0 = CW * 0.5, CH * 0.5
        env, envg = self.env, self.envg
        env.fill(0.0)
        envg.fill(0.0)
        for c in self.clouds:
            c.p += 1.0 / c.life
            wcl = (lo + (hi - lo) * c.u) * CW * c.sc
            if zoom_sgn != 0.0:
                f = 1.0 + zoom_sgn * zr * c.vk
                c.x = cx0 + (c.x - cx0) * f
                c.y = cy0 + (c.y - cy0) * f
                c.sc *= f
                zf = self._smooth((2.4 - c.sc) / 0.6) * self._smooth((c.sc - 0.30) / 0.30)
                gone = c.p >= 1.0 or c.sc > 2.5 or c.sc < 0.28
            else:
                c.x += dirv * base * c.vk
                zf = 1.0
                gone = c.p >= 1.0 or c.x - wcl > CW * 1.05 or c.x + wcl < -CW * 0.05
                if mode == "variable":
                    gone = c.p >= 1.0 or c.x - wcl > CW * 1.25 or c.x + wcl < -CW * 0.25
            if gone:
                self._spawn(st, c, False, mode, shred)
                continue
            fi = self._smooth(c.p / 0.15)
            fo = 1.0 - self._smooth((c.p - (0.55 + 0.30 * (1.0 - shred))) / (1.0 - (0.55 + 0.30 * (1.0 - shred))))
            amp = c.a0 * fi * fo * zf
            if amp < 0.01:
                continue
            stretch = 1.0 + 0.9 * shred * c.p
            squash = 1.0 - 0.25 * shred * c.p
            for ox, oy, sx, sy, a in c.blobs:
                cx = (c.x + ox * wcl) * 0.5
                cy = (c.y + oy * wcl) * 0.5
                rx = max(2.0, sx * wcl * stretch * 0.5)
                ry = max(2.0, sy * wcl * squash * 0.5)
                xa, xb = int(max(0, cx - 2.3 * rx)), int(min(self.EW, cx + 2.3 * rx) + 1)
                ya, yb = int(max(0, cy - 2.3 * ry)), int(min(self.EH, cy + 2.3 * ry) + 1)
                if xb <= xa or yb <= ya:
                    continue
                gx = np.exp(-((self.xs[xa:xb] - cx) / rx) ** 2)
                gy = np.exp(-((self.ys[ya:yb] - cy) / ry) ** 2)
                blob = np.outer(gy, gx) * (amp * a)
                env[ya:yb, xa:xb] += blob
                envg[ya:yb, xa:xb] += blob * c.gj
        E = cv2.resize(env, (CW, CH), interpolation=cv2.INTER_LINEAR)
        EG = cv2.resize(envg, (CW, CH), interpolation=cv2.INTER_LINEAR) / np.maximum(E, 1e-3)

        # ---- ruido: lo arrastra el viento (y se estira al deshacerse) ----
        if zoom_sgn == 0.0:
            self.nx += dirv * base * 1.1
            sxs1 = 1.0 + 1.8 * shred
            sxs2 = 1.0 + 2.8 * shred
            M1 = np.float32([[sxs1, 0, CW * 0.5 * (1 - sxs1) + self.nx], [0, 1, 0]])
            M2 = np.float32([[sxs2, 0, CW * 0.5 * (1 - sxs2) + self.nx * 1.7], [0, 1, 0]])
            n1 = self._warp(self.T1, M1)
            n2 = self._warp(self.T2, M2)
        else:
            self.zph = (self.zph + 0.0004 + 0.0016 * (w10 / 10.0)) % 1.0
            n1 = np.zeros((CH, CW), dtype=np.float32)
            n2 = np.zeros((CH, CW), dtype=np.float32)
            for off in (0.0, 0.5):
                q = (self.zph + off) % 1.0
                wgt = math.sin(math.pi * q) ** 2
                sc = 1.0 + 0.7 * q if zoom_sgn > 0 else 1.7 - 0.7 * q
                M = np.float32([[sc, 0, cx0 * (1 - sc)], [0, sc, cy0 * (1 - sc)]])
                n1 += wgt * self._warp(self.T1, M)
                n2 += wgt * self._warp(self.T2, M)

        # ---- el ruido deforma el contorno (cumulos esponjosos, no elipses) y evoluciona ----
        aw = self.K_WARP * self.k
        E = cv2.remap(E, self.gx + (n1 - 0.5) * 2.0 * aw, self.gy + (n2 - 0.5) * 1.2 * aw,
                      interpolation=cv2.INTER_LINEAR, borderMode=cv2.BORDER_REPLICATE)

        # ---- densidad: envolvente + ruido, con erosion segun el viento ----
        gate = np.clip(E * self.K_GATE, 0.0, 1.0)
        thr = self.K_THR + 0.07 * shred
        D = (E * self.K_E + gate * ((n1 - 0.5) * self.K_N1 + (n2 - 0.5) * (self.K_N2 + 0.35 * shred))
             - thr) / self.K_RAMP
        np.clip(D, 0.0, 1.0, out=D)
        D = D * D * (3.0 - 2.0 * D)

        # ---- luz desde arriba + grosor (los bordes finos brillan) ----
        Db = cv2.GaussianBlur(D, (0, 0), 2.2 * self.k)
        gy_ = cv2.Sobel(Db, cv2.CV_32F, 0, 1, ksize=3)
        gy_ = np.clip(gy_ / (float(np.percentile(np.abs(gy_), 98)) + 1e-6), -1.0, 1.0)
        kb, kr, ks = self.K_LIGHT
        Lm = np.clip(kb + kr * gy_ - ks * Db * (0.4 + 0.6 * self.yrel), 0.12, 1.0)

        # ---- color: blancas o grises (con variacion por nube) + reflejo ----
        grey = np.clip((1.0 - float(np.clip(st.cl_white, 0, 100)) / 100.0) + EG, 0.0, 1.0)[..., None]
        S_w = np.array([215, 196, 184], np.float32)
        S_g = np.array([98, 86, 74], np.float32)
        L_w = np.array([255, 254, 252], np.float32)
        L_g = np.array([178, 168, 158], np.float32)
        S = S_w + (S_g - S_w) * grey
        Lc = L_w + (L_g - L_w) * grey
        rgb = S + (Lc - S) * Lm[..., None]
        ri = float(np.clip(st.cl_refl_int, 0, 100)) / 100.0
        Rc = np.array(CLOUD_REFL.get(st.cl_refl_color, CLOUD_REFL["blanco"]), np.float32)
        rk = (ri * (0.10 + 0.90 * Lm ** 2.2) * (0.55 + 0.45 * (1.0 - Db)) * 0.85)[..., None]
        rgb = rgb * (1.0 - rk) + Rc * rk
        A = np.clip(D * (0.80 + 0.40 * grey[..., 0]), 0.0, 1.0)

        # ---- resplandor del cielo con el color del reflejo (se ve aun sin nubes) ----
        Av = ri * 0.30 * self._veil_field()
        Atot = A + Av * (1.0 - A)
        P = rgb * A[..., None] + Rc * (Av * (1.0 - A))[..., None]          # premultiplicado

        op = float(np.clip(st.cl_opacity, 0, 100)) / 100.0
        if st.cl_rb_on:                                # arcoiris (detras de las nubes)
            Pr, Ar = self.rainbow.get(st.cl_rb_x, st.cl_rb_y)
            rop = float(np.clip(st.cl_rb_op, 0, 100)) / 100.0
            Ac = Atot * op
            P = P * op + (Pr * rop) * (1.0 - Ac)[..., None]
            Atot = Ac + (Ar * rop) * (1.0 - Ac)
            op = 1.0
        up = cv2.resize(P, (VW, VH), interpolation=cv2.INTER_LINEAR)
        aup = cv2.resize(Atot, (VW, VH), interpolation=cv2.INTER_LINEAR)
        out = np.empty((VH, VW, 4), dtype=np.uint8)
        col = up / np.maximum(aup, 1e-4)[..., None]
        np.clip(col, 0, 255, out=col)
        out[:, :, :3] = col.astype(np.uint8)
        out[:, :, 3] = (np.clip(aup * op, 0, 1) * 255.0).astype(np.uint8)
        return out


# ---------------------------------------------------------------------
# Cascada
# ---------------------------------------------------------------------
class WaterfallSim:
    """Cascada de agua que cae desde arriba de la pantalla: hebras que bajan, bordes que
    ondulan, vapor que sube y gotitas que salpican al caer (abajo). Se dibuja a resolucion
    de lienzo (mitad del vidrio) y se escala."""
    SP_MAX = 360

    def __init__(self):
        self.CW, self.CH = max(16, VW // 2), max(9, VH // 2)
        CW, CH = self.CW, self.CH
        self.k = CW / 480.0
        k = self.k
        g = np.random.default_rng()
        self.S1 = tile_fbm2(g, CW, CH, 5 * k, 70 * k, 4, 0.55)       # hebras
        self.S2 = tile_fbm2(g, CW, CH, 2.5 * k, 26 * k, 3, 0.60)     # hebras finas y rapidas
        self.M1 = tile_fbm(g, CW, CH, 70 * k, 5, 0.55)               # vapor (grande)
        self.M2 = tile_fbm(g, CW, CH, 22 * k, 4, 0.55)               # vapor (fino)
        self.eL = periodic_noise_1d(g, CH, 60 * k, 3)                # bordes ondulantes
        self.eR = periodic_noise_1d(g, CH, 60 * k, 3)
        self.eC = periodic_noise_1d(g, CH, 140 * k, 2)               # vaiven del centro
        self.xs = np.arange(CW, dtype=np.float32)[None, :]
        self.rows = np.arange(CH)
        self.yrel = np.linspace(0.0, 1.0, CH, dtype=np.float32)
        self.sy1 = self.sy2 = self.sy_e = 0.0
        self.mx = self.my = 0.0
        self.t = 0
        n = self.SP_MAX                                              # gotitas del salpicado
        self.px = np.zeros(n, dtype=np.float32)
        self.py = np.zeros(n, dtype=np.float32)
        self.pvx = np.zeros(n, dtype=np.float32)
        self.pvy = np.zeros(n, dtype=np.float32)
        self.plife = np.ones(n, dtype=np.float32)
        self.page = np.full(n, 999.0, dtype=np.float32)              # todas apagadas
        self.rainbow = RainbowLayer(CW, CH)

    def _spray(self, x0, wfull, flow, vapor):
        CW, CH, k = self.CW, self.CH, self.k
        al = self.page < self.plife
        self.pvy[al] += 0.30 * k
        self.px[al] += self.pvx[al]
        self.py[al] += self.pvy[al]
        self.page[al] += 1.0
        rate = 3.0 + 14.0 * vapor * (0.4 + 0.6 * flow)               # gotitas nuevas por frame
        nnew = int(rate) + (1 if np.random.random() < rate - int(rate) else 0)
        free = np.nonzero(~(self.page < self.plife))[0][:nnew]
        m = free.size
        if m:
            hw = 0.5 * wfull * 1.3
            self.px[free] = x0 + np.random.uniform(-hw, hw, m)
            self.py[free] = CH - np.random.uniform(0.0, 0.05 * CH, m)
            side = np.sign(self.px[free] - x0)
            self.pvx[free] = (np.random.uniform(-1.6, 1.6, m)
                              + side * np.random.uniform(0.0, 1.8, m)) * k
            self.pvy[free] = -np.random.uniform(3.0, 8.5, m) * (0.6 + 0.8 * flow) * k
            self.plife[free] = np.random.uniform(22, 55, m)
            self.page[free] = 0.0

    def _spray_alpha(self, vapor):
        CW, CH = self.CW, self.CH
        A = np.zeros((CH, CW), dtype=np.float32)
        al = self.page < self.plife
        if al.any():
            ix = self.px[al].astype(np.int32)
            iy = self.py[al].astype(np.int32)
            a = (1.0 - self.page[al] / self.plife[al]) * (0.55 + 0.40 * vapor)
            ok = (ix >= 1) & (ix < CW - 1) & (iy >= 0) & (iy < CH - 1)
            ix, iy, a = ix[ok], iy[ok], a[ok]
            A[iy, ix] = np.maximum(A[iy, ix], a)
            A[iy, ix + 1] = np.maximum(A[iy, ix + 1], a * 0.6)
            A[iy + 1, ix] = np.maximum(A[iy + 1, ix], a * 0.6)
            A[iy + 1, ix + 1] = np.maximum(A[iy + 1, ix + 1], a * 0.4)
        return A

    def render(self, st):
        CW, CH, k = self.CW, self.CH, self.k
        self.t += 1
        flow = float(np.clip(st.wf_amount, 1, 100)) / 100.0
        x0 = float(np.clip(st.wf_x, 0, 100)) / 100.0 * CW
        wfull = max(4.0 * k, float(np.clip(st.wf_width, 3, 100)) / 100.0 * CW)
        op = float(np.clip(st.wf_opacity, 5, 100)) / 100.0

        # ---- la cascada: las hebras bajan y los bordes ondulan ----
        v = (5.0 + 5.0 * flow) * k
        self.sy1 += v
        self.sy2 += v * 1.7
        self.sy_e += v * 0.8
        t1 = wrap_shift(self.S1, 0.0, self.sy1 % CH)
        t2 = wrap_shift(self.S2, 0.0, self.sy2 % CH)
        stk = 0.6 * t1 + 0.4 * t2
        idx = (self.rows - int(self.sy_e)) % CH
        yr = self.yrel
        hw = 0.5 * wfull * (1.0 + 0.30 * yr)                         # se ensancha un poco al caer
        amp_e = 0.06 * wfull + 2.5 * k
        cen = x0 + self.eC[idx] * (0.025 * CW + 2 * k) * (0.3 + yr)
        left = cen - hw + self.eL[idx] * amp_e
        right = cen + hw + self.eR[idx] * amp_e
        soft = ((2.0 + 7.0 * yr) * k)[:, None]
        dl = self.xs - left[:, None]
        dr = right[:, None] - self.xs
        m = np.clip(dl / soft, 0, 1) * np.clip(dr / soft, 0, 1)
        m = m * m * (3.0 - 2.0 * m)
        ed = np.clip(np.minimum(dl, dr) / (0.22 * hw[:, None] + 1.0), 0, 1)
        thr = (1.0 - flow) * 0.72                                    # poco caudal: solo hebras fuertes
        D = np.clip((stk - thr) / (0.42 + 0.40 * flow), 0.0, 1.0)
        dens = m * (0.30 + 0.70 * D) * (0.30 + 0.70 * flow ** 0.6) * (0.65 + 0.35 * ed)
        Aw = np.clip(dens, 0, 1) * op * 0.97
        aer = np.clip(0.30 + 0.55 * yr[:, None] + 0.35 * (stk - 0.5), 0, 1)   # mas espuma abajo
        blue = np.array([235, 190, 130], dtype=np.float32)
        white = np.array([255, 252, 248], dtype=np.float32)
        Cw = blue + (white - blue) * aer[..., None] + 30.0 * np.clip(t2 - 0.72, 0, 1)[..., None]

        P = np.zeros((CH, CW, 3), dtype=np.float32)
        A = np.zeros((CH, CW), dtype=np.float32)
        if st.wf_rb_on:                                              # arcoiris (al fondo)
            Pr, Ar = self.rainbow.get(st.wf_rb_x, st.wf_rb_y)
            rop = float(np.clip(st.wf_rb_op, 0, 100)) / 100.0
            P = Pr * rop
            A = Ar * rop
        P = Cw * Aw[..., None] + P * (1.0 - Aw)[..., None]
        A = Aw + A * (1.0 - Aw)

        # ---- vapor y salpicado al caer ----
        vapor = float(np.clip(st.wf_mist, 0, 100)) / 100.0
        if st.wf_mist_on and vapor > 0:
            self.mx += (0.5 + 0.6 * math.sin(self.t * 0.013)) * k
            self.my += (1.2 + 1.8 * flow) * k
            n1 = wrap_shift(self.M1, self.mx, (-self.my) % CH)               # el vapor sube
            n2 = wrap_shift(self.M2, self.mx * 1.8, (-self.my * 1.7) % CH)
            xm = x0 + 0.10 * CW * math.sin(self.t * 0.007)                   # y se mece
            rx = 0.5 * wfull * 1.25 + (0.08 + 0.22 * vapor) * CW
            ry = (0.08 + 0.30 * vapor) * CH * (0.6 + 0.4 * flow)
            ex = np.exp(-((self.xs - xm) / rx) ** 2)
            ey = np.exp(-(((CH - 1) - np.arange(CH, dtype=np.float32)) / ry) ** 2)[:, None]
            E = ex * ey + cv2.GaussianBlur(m, (0, 0), 6 * k) * (0.35 * vapor)  # + neblina pegada
            gate = np.clip(E * 2.2, 0, 1)
            Dm = np.clip((E * 1.30 + gate * ((n1 - 0.5) * 1.10 + (n2 - 0.5) * 0.50) - 0.28) / 0.70,
                         0.0, 1.0)
            Dm = Dm * Dm * (3.0 - 2.0 * Dm)
            Am = np.clip(Dm * (0.25 + 0.55 * vapor) * (0.5 + 0.5 * flow) * 1.25, 0.0, 0.95)
            Cm = np.array([252, 246, 238], dtype=np.float32) * (0.90 + 0.10 * n2)[..., None]
            P = Cm * Am[..., None] + P * (1.0 - Am)[..., None]
            A = Am + A * (1.0 - Am)
            self._spray(x0, wfull, flow, vapor)
            Asp = self._spray_alpha(vapor)
            P = 252.0 * Asp[..., None] + P * (1.0 - Asp)[..., None]
            A = Asp + A * (1.0 - Asp)

        up = cv2.resize(P, (VW, VH), interpolation=cv2.INTER_LINEAR)
        aup = cv2.resize(A, (VW, VH), interpolation=cv2.INTER_LINEAR)
        out = np.empty((VH, VW, 4), dtype=np.uint8)
        col = up / np.maximum(aup, 1e-4)[..., None]
        np.clip(col, 0, 255, out=col)
        out[:, :, :3] = col.astype(np.uint8)
        out[:, :, 3] = (np.clip(aup, 0, 1) * 255.0).astype(np.uint8)
        return out


# =====================================================================
# Estado compartido
# =====================================================================
class RainState:
    """Parametros compartidos entre el panel de control y el overlay."""
    def __init__(self):
        self.style = "pantalla"    # "pantalla" o "ventana"
        self.interval_ms = 33      # ~30fps al minimo, sube = mas lento
        self.intensity = 6         # gotas/droplets nuevos por tick
        self.strength = 5.0        # fuerza de cada gota (pantalla) / gravedad (ventana)
        self.drop_size = 4         # radio base de la gota en px (varia por gota)
        self.wave_persistence = 0.985  # pantalla: cuanto viajan las ondas
                                        # ventana: cuanto duran los rastros
        self.glass_blur = 14       # desenfoque del "vidrio" en modo ventana
        self.wiper_active = False
        self.wiper_pos = 0.0
        self.wiper_dir = 1
        self.wiper_speed = 0.06    # rad por tick
        self.wiper_half_width = 0.12   # rad, ancho de la "pala"
        self.wiper_single = False  # True = una sola pasada (atajo Ctrl + Ñ), luego se apaga
        self.recapture_every = 30  # frames entre re-capturas de pantalla
        self.recapture_every_glass = 15

        # ---- estilo Ventana ----
        self.fog = 0.75            # 0..1 nivel de empañado (opacidad del vidrio)
        self.fg_inside = True      # ventana activa "dentro" (clara) o "fuera" (empañada)
        self.fg_dynamic = False    # "dinamica": maximizada = fuera, no maximizada = dentro
        self.stickiness = 40       # 0..100 que tan pegajosas son las gotas
        self.wiper_defog = 0.45    # fraccion de vaho que quita cada pasada
        self.thunder_on = False
        self.thunder_freq = 4      # 1..10
        self.thunder_intensity = 6 # 1..10
        self.thunder_sound = True
        self.thunder_test = False  # el boton "probar" lo enciende un frame
        self.rain_outside = True   # ver la lluvia cayendo "afuera" (detras del vidrio)
        self.rain_amount = 40      # 5..100 cantidad de rayitas de lluvia afuera
        self.rain_dir = "right"    # right | left | toward | away | variable (cambiante)
        self.rain_incl = 20        # 0..100 inclinacion de la lluvia (20 = como antes)
        self.finger_defog = True   # clic / arrastrar desempaña el vidrio
        self.finger_size = 24      # radio del "dedo" en px de pantalla
        self.finger_regrow = 45    # segundos que tarda en volver a empañarse

        # ---- estilo Nieve ----
        self.snow_on = True        # ver nevar
        self.snow_amount = 50      # 0..100 cantidad de copos
        self.snow_size = 4         # 1..12 tamaño de los copos
        self.snow_frost_on = True  # escarcha en el vidrio
        self.snow_frost = 50       # 0..100 cantidad de escarcha
        self.snow_wind_on = False  # viento
        self.snow_wind_strength = 5    # 1..10
        self.snow_wind_dir = "right"   # right | left | toward | away | variable (cambiante)
        self.snow_fog_on = False   # neblina (fog) afuera
        self.snow_fog = 40         # 0..100 cantidad de neblina
        self.snow_fog_move = True  # la neblina se mueve (y responde al viento y su direccion)
        self.snow_touch = True     # clic / arrastrar: sopla los copos y limpia la escarcha
        self.snow_touch_size = 26  # 6..80 radio del "dedo" (px de pantalla); el soplo es mayor
        self.snow_touch_regrow = 40    # segundos que tarda en volver a formarse la escarcha

        # ---- estilo Luciernagas ----
        self.fly_amount = 60       # 1..400
        self.fly_size = 4          # 1..12
        self.fly_colors = ["amarillo"]   # de 1 a 10 colores a la vez

        # ---- estilo Matrix ----
        self.mx_color = "verde"    # verde | azulado | morado | mixto
        self.mx_chars = "katakana" # katakana | numeros | binario
        self.mx_amount = 60        # 5..100 cantidad de codigo
        self.mx_opacity = 80       # 5..100
        self.mx_depth = 3          # 1..4 capas superpuestas
        self.mx_size = 5           # 1..10 tamaño del codigo (5 = como antes)
        self.mx_speed = 5          # 1..10 velocidad de caida (5 = como antes)
        self.mx_speed_var = False  # velocidad variable: a veces rapido, a veces despacio
        self.mx_messages = ["", "", "", "", ""]   # mensajes escondidos que caen entre el codigo
        self.mx_msg_glow = True    # brillo en las letras de los mensajes escondidos
        self.mx_glow_on = True     # algunas columnas del codigo caen brillando
        self.mx_glow_int = 6       # 1..10 intensidad del brillo (codigo y mensajes)
        self.mx_glow_blur = 6      # 1..10 difuminado del brillo (codigo y mensajes)

        # ---- estilo Nubes ----
        self.cl_amount = 8         # 1..30
        self.cl_opacity = 80       # 5..100
        self.cl_distance = 35      # 0 cerca y grandes .. 100 lejos y pequeñas
        self.cl_white = 80         # 0 grises .. 100 blancas
        self.cl_dir = "right"      # right | left | toward | away | variable (cambiante)
        self.cl_size_min = 25      # 3..100 tamaño minimo de las nubes (% del ancho de la pantalla)
        self.cl_size_max = 60      # 3..100 tamaño maximo
        self.cl_wind = 4           # 0..10 viento: mas rapido y, si es fuerte, las deshace
        self.cl_refl_color = "blanco"   # azulado | naranja | rosa | gris | blanco | morado
        self.cl_refl_int = 40      # 0..100 intensidad del reflejo
        self.cl_rb_on = False      # arcoiris en el cielo
        self.cl_rb_x = 50          # 0..100 posicion horizontal (centro del arco)
        self.cl_rb_y = 30          # 0..100 posicion vertical (cima del arco)
        self.cl_rb_op = 55         # 0..100 opacidad del arcoiris

        # ---- estilo Cascada ----
        self.wf_x = 50             # 0..100 desde que parte de arriba cae (0 izquierda .. 100 derecha)
        self.wf_width = 22         # 3..100 grosor: % del ancho de la pantalla
        self.wf_amount = 70        # 1..100 cuanta agua cae
        self.wf_opacity = 75       # 5..100 opacidad del agua (menos = mas translucida)
        self.wf_mist_on = True     # vapor y salpicado al caer
        self.wf_mist = 60          # 0..100 cantidad de vapor
        self.wf_rb_on = False      # arcoiris
        self.wf_rb_x = 50
        self.wf_rb_y = 28
        self.wf_rb_op = 55

        # ---- estilo Estrellas ----
        self.sky_dir = "right"          # right | left | toward | away
        self.sky_meteors = 25           # 0..100 estrellas fugaces (0 = ninguna)
        self.sky_stars = 45             # 0..400 estrellas fijas (100 = el maximo de antes)
        self.sky_comets_on = False
        self.sky_comets_n = 2           # 1..5
        self.sky_aurora_on = False
        self.sky_aurora_n = 2           # 1..4
        self.sky_nebula_on = False
        # (color1, color2): color1 puede ser "aleatorio"; color2 "ninguno" o un color
        self.sky_colors = {
            "meteor": ("blanco", "ninguno"),
            "comet": ("azul", "ninguno"),
            "aurora": ("verde", "morado"),
            "nebula": ("morado", "azul"),
            "stars": ("natural", "ninguno"),     # estrellas fijas (natural = blanco azulado/calido)
        }
        self.own_hwnds = set()     # ventanas propias (panel/overlay) para ignorarlas


# =====================================================================
# Guardado de ajustes (se recuperan al abrir el programa)
# =====================================================================
# ajustes que NO se guardan (estado del momento, no preferencias)
_SETTINGS_SKIP = {"own_hwnds", "wiper_active", "wiper_pos", "wiper_dir", "wiper_single",
                  "thunder_test"}
SETTING_CHOICES = {
    "style": ("pantalla", "ventana", "nieve", "estrellas", "luciernagas", "matrix", "nubes",
              "cascada"),
    "rain_dir": ("right", "left", "toward", "away", "variable"),
    "snow_wind_dir": ("right", "left", "toward", "away", "variable"),
    "sky_dir": ("right", "left", "toward", "away"),
    "cl_dir": ("right", "left", "toward", "away", "variable"),
    "cl_refl_color": ("azulado", "naranja", "rosa", "gris", "blanco", "morado"),
    "mx_color": ("verde", "azulado", "morado", "mixto"),
    "mx_chars": ("katakana", "numeros", "binario"),
}
SETTING_RANGES = {            # (minimo, maximo) de cada ajuste numerico
    "interval_ms": (10, 100), "intensity": (0, 30), "strength": (1, 20), "drop_size": (1, 30),
    "wave_persistence": (0.9, 0.999), "glass_blur": (0, 40), "wiper_speed": (0.01, 0.30),
    "fog": (0.0, 1.0), "stickiness": (0, 100), "wiper_defog": (0.0, 1.0),
    "thunder_freq": (1, 10), "thunder_intensity": (1, 10), "rain_amount": (5, 100),
    "rain_incl": (0, 100), "finger_size": (6, 80), "finger_regrow": (5, 180),
    "snow_amount": (0, 100), "snow_size": (1, 12), "snow_frost": (0, 100),
    "snow_wind_strength": (1, 10), "snow_fog": (0, 100), "snow_touch_size": (6, 80),
    "snow_touch_regrow": (5, 180), "fly_amount": (1, FIREFLY_MAX), "fly_size": (1, 12),
    "mx_amount": (5, 100), "mx_opacity": (5, 100), "mx_depth": (1, 4),
    "mx_size": (1, 10), "mx_speed": (1, 10), "mx_glow_int": (1, 10), "mx_glow_blur": (1, 10),
    "cl_amount": (1, 30), "cl_opacity": (5, 100), "cl_distance": (0, 100), "cl_white": (0, 100),
    "cl_size_min": (3, 100), "cl_size_max": (3, 100), "cl_wind": (0, 10), "cl_refl_int": (0, 100),
    "cl_rb_x": (0, 100), "cl_rb_y": (0, 100), "cl_rb_op": (0, 100),
    "wf_x": (0, 100), "wf_width": (3, 100), "wf_amount": (1, 100), "wf_opacity": (5, 100),
    "wf_mist": (0, 100), "wf_rb_x": (0, 100), "wf_rb_y": (0, 100), "wf_rb_op": (0, 100),
    "sky_meteors": (0, 100), "sky_stars": (0, 400), "sky_comets_n": (1, 5),
    "sky_aurora_n": (1, 4),
}


def settings_path():
    base = os.environ.get("APPDATA") or os.path.join(os.path.expanduser("~"), ".config")
    return os.path.join(base, "RainOverlay", "settings.json")


def state_to_dict(state):
    d = {}
    for k, v in vars(state).items():
        if k in _SETTINGS_SKIP or k.startswith("_"):
            continue
        if isinstance(v, (bool, int, float, str, list, dict)):
            d[k] = v
    return d


def save_settings(state):
    try:
        path = settings_path()
        os.makedirs(os.path.dirname(path), exist_ok=True)
        data = {"version": 1}
        data.update(state_to_dict(state))
        tmp = path + ".tmp"
        with open(tmp, "w", encoding="utf-8") as fh:
            json.dump(data, fh, ensure_ascii=False, indent=2)
        os.replace(tmp, path)
        return True
    except Exception:
        return False


def load_settings(state):
    """Lee los ajustes guardados y los aplica a `state` (ignora lo que no sea valido)."""
    try:
        with open(settings_path(), "r", encoding="utf-8") as fh:
            data = json.load(fh)
    except Exception:
        return False
    if not isinstance(data, dict):
        return False
    for k, new in data.items():
        if k in _SETTINGS_SKIP or k.startswith("_") or not hasattr(state, k):
            continue
        cur = getattr(state, k)
        try:
            num = isinstance(new, (int, float)) and not isinstance(new, bool)
            if isinstance(cur, bool):
                if isinstance(new, bool):
                    setattr(state, k, new)
            elif isinstance(cur, (int, float)):
                if num:
                    val = float(new)
                    if k in SETTING_RANGES:
                        lo, hi = SETTING_RANGES[k]
                        val = min(max(val, lo), hi)
                    setattr(state, k, int(round(val)) if isinstance(cur, int) else val)
            elif isinstance(cur, str):
                if isinstance(new, str) and (k not in SETTING_CHOICES or new in SETTING_CHOICES[k]):
                    setattr(state, k, new)
            elif k == "fly_colors":
                cols = [c for c in FIREFLY_COLOR_NAMES if isinstance(new, list) and c in new]
                if cols:
                    setattr(state, k, cols)
            elif k == "mx_messages":
                if isinstance(new, list):
                    msgs = [m[:MX_MSG_MAX] if isinstance(m, str) else "" for m in new[:5]]
                    msgs += [""] * (5 - len(msgs))
                    setattr(state, k, msgs)
            elif k == "sky_colors" and isinstance(new, dict):
                for key, pair in new.items():
                    if key in cur and isinstance(pair, (list, tuple)) and len(pair) == 2:
                        c1, c2 = pair
                        if (c1 in STAR_COLOR_NAMES or c1 == "aleatorio"
                                or (key == "stars" and c1 == "natural")) and \
                                (c2 in STAR_COLOR_NAMES or c2 == "ninguno"):
                            cur[key] = (c1, c2)
        except Exception:
            pass
    return True


# =====================================================================
# Overlay
# =====================================================================
class OverlayWindow(QtWidgets.QWidget):
    def __init__(self, state: RainState):
        super().__init__()
        self.state = state
        self.setWindowFlags(
            QtCore.Qt.FramelessWindowHint
            | QtCore.Qt.WindowStaysOnTopHint
            | QtCore.Qt.Tool
        )
        self.setAttribute(QtCore.Qt.WA_TranslucentBackground)
        self.setGeometry(mon["left"], mon["top"], W, H)

        # ---- estado modo "pantalla" (ondas) ----
        self.h_prev = np.zeros((GH, GW), dtype=np.float32)
        self.h_cur = np.zeros((GH, GW), dtype=np.float32)

        # ---- estado modo "ventana" (vidrio) ----
        self.drops = []                                   # lista de Drop
        self.trail = np.zeros((VH, VW), dtype=np.uint8)   # rastros humedos (persistente)
        self.trail_acc = 0.0
        self.fog = np.full((FH, FW), state.fog, dtype=np.float32)  # vaho (0..1)
        self.fg_hwnd = 0
        self.fg_rect = None
        self.capture_excluded = False
        self.glass_dirty = True
        self._last_blur = state.glass_blur

        # truenos
        self.thunder_sound = ThunderSound()
        self.next_strike = None
        self.strike_t0 = 0.0
        self.pulses = []
        self.flash = 0.0
        self.flash_amp = 1.0
        self.bolt = None
        self.bolt_halo = None
        self.bolt_vis = 0.0
        self.bolt_gain = 1.0
        self.flash_map = None

        # desempañado con el dedo (mouse)
        self.clear = np.zeros((VH, VW), dtype=np.uint8)   # 255 = vidrio limpio
        self.clear_acc = 0.0
        self.clear_any = False
        self.f_down = False
        self.f_active = False
        self.f_last = None
        self.finger_water = 0.0

        # lluvia "afuera" (detras del vidrio)
        self.rain_img = None
        self.init_outside_rain()

        # estilo Nieve: copos + escarcha
        self.snow = SnowSim()
        self.snow_img = None
        self._frost_key = None
        self.frost_alpha = np.zeros((VH, VW), dtype=np.float32)
        self.init_frost()
        self.init_mist()
        self.hotkey_poll = False        # True si no se pudo registrar Ctrl + Ñ (se sondea)
        self._hk_prev = False
        self.hk_poll_extra = set()      # atajos extra que no se pudieron registrar (se sondean)
        self.hk_prev = {}

        # estilo Nieve: mouse (soplo de aire + limpiar escarcha con el dedo)
        self.frost_clear = np.zeros((VH, VW), dtype=np.uint8)   # 255 = escarcha limpiada
        self.fclear_acc = 0.0
        self.fclear_any = False
        self.t_down = False
        self.t_active = False
        self.t_last = None

        # lluvia de afuera: direccion "cambiante"
        self.rwv = 0.0
        self.rwv_tgt = 0.0
        self.rwv_timer = 0

        # estilo Estrellas
        self.sky = SkySim()
        self.sky_half = None

        # estilos Luciernagas / Matrix / Nubes
        self.fly = FireflySim()
        self.matrix = MatrixSim()
        self.clouds = CloudSim()
        self.waterfall = WaterfallSim()
        self.fx_half = None

        self.frame = 0
        self.bg_full = self.capture_screen()
        self.bg = cv2.resize(self.bg_full, (RW, RH), interpolation=cv2.INTER_AREA)
        self.bg_s = cv2.resize(self.bg_full, (VW, VH), interpolation=cv2.INTER_AREA)
        self.fog_img = np.zeros((VH, VW, 3), dtype=np.float32)

        # precalculados una sola vez (evita recrearlos cada frame)
        xs, ys = np.meshgrid(np.arange(RW, dtype=np.float32), np.arange(RH, dtype=np.float32))
        self.xs, self.ys = xs, ys
        gxs, gys = np.meshgrid(np.arange(VW, dtype=np.float32), np.arange(VH, dtype=np.float32))
        self.gxs, self.gys = gxs, gys

        # geometria del limpiaparabrisas
        self.wgeom_grid = WiperGeom(GW, GH)
        self.wgeom_glass = WiperGeom(VW, VH)

        self.mouse_down = False
        self.last_mouse_pos = None

        self.timer = QtCore.QTimer()
        self.timer.timeout.connect(self.tick)
        self.timer.start(self.state.interval_ms)

    # ------------------------------------------------------------------
    def make_click_through(self):
        hwnd = int(self.winId())
        self.state.own_hwnds.add(hwnd)
        styles = win32gui.GetWindowLong(hwnd, win32con.GWL_EXSTYLE)
        win32gui.SetWindowLong(
            hwnd, win32con.GWL_EXSTYLE,
            styles | win32con.WS_EX_LAYERED | win32con.WS_EX_TRANSPARENT,
        )
        # Excluye el overlay de las capturas de pantalla: asi el programa ve el
        # escritorio REAL (sin su propio efecto) y no se acumula el desenfoque.
        # Requiere Windows 10 v2004 o superior.
        try:
            WDA_EXCLUDEFROMCAPTURE = 0x11
            ok = ctypes.windll.user32.SetWindowDisplayAffinity(
                wintypes.HWND(hwnd), WDA_EXCLUDEFROMCAPTURE)
            self.capture_excluded = bool(ok)
        except Exception:
            self.capture_excluded = False
        if not self.capture_excluded:
            print("Aviso: no se pudo excluir el overlay de la captura "
                  "(Windows < 10 v2004). El estilo Ventana puede verse mas borroso.")

    def showEvent(self, event):
        super().showEvent(event)
        self.make_click_through()

    def capture_screen(self):
        with mss.mss() as sct:
            img = np.array(sct.grab(mon))[:, :, :3]
        return img

    # ---------------------------------------------------------------
    # Limpiaparabrisas (compartido)
    # ---------------------------------------------------------------
    def advance_wiper(self):
        st = self.state
        theta_min, theta_max = 0.01 * np.pi, 0.99 * np.pi
        st.wiper_pos += st.wiper_dir * st.wiper_speed
        if st.wiper_pos >= theta_max:
            st.wiper_pos = theta_max
            st.wiper_dir = -1
        elif st.wiper_pos <= theta_min:
            st.wiper_pos = theta_min
            st.wiper_dir = 1
            if st.wiper_single:          # pasada unica (atajo): al volver al inicio se apaga
                st.wiper_single = False
                st.wiper_active = False
        return st.wiper_pos

    def trigger_wiper_pass(self):
        """Atajo Ctrl + Ñ: pasa el limpiaparabrisas UNA vez (ida y vuelta) y se apaga.
        Si ya esta en marcha (una pasada o el modo continuo), lo detiene.
        En el estilo Matrix hace caer uno de los mensajes escondidos."""
        st = self.state
        if st.style == "matrix":
            self.matrix.launch_message(st)
            return
        if st.wiper_active:
            st.wiper_active = False
            st.wiper_single = False
        else:
            st.wiper_pos = 0.01 * np.pi
            st.wiper_dir = 1
            st.wiper_single = True
            st.wiper_active = True

    def poll_wiper_hotkey(self):
        """Respaldo si Windows no dejo registrar el atajo: se sondea Ctrl + Ñ."""
        try:
            ctrl = bool(win32api.GetAsyncKeyState(win32con.VK_CONTROL) & 0x8000)
            alt = bool(win32api.GetAsyncKeyState(win32con.VK_MENU) & 0x8000)
            shift = bool(win32api.GetAsyncKeyState(win32con.VK_SHIFT) & 0x8000)
            key = bool(win32api.GetAsyncKeyState(ntilde_vk()) & 0x8000)
        except Exception:
            return
        down = ctrl and key and not alt and not shift
        if down and not self._hk_prev:
            self.trigger_wiper_pass()
        self._hk_prev = down

    def hotkey_thunder(self):
        """Atajo Ctrl + Mayus + Ñ: lanza un trueno (estilo Ventana)."""
        if self.state.style == "ventana":
            self.state.thunder_test = True

    def hotkey_meteor(self):
        """Atajo Alt + Ñ: aparece una estrella fugaz o un cometa (estilo Estrellas)."""
        if self.state.style == "estrellas":
            self.sky.launch_one(self.state)

    def poll_extra_hotkeys(self):
        """Respaldo si Windows no dejo registrar los atajos extra: se sondean."""
        try:
            ctrl = bool(win32api.GetAsyncKeyState(win32con.VK_CONTROL) & 0x8000)
            alt = bool(win32api.GetAsyncKeyState(win32con.VK_MENU) & 0x8000)
            shift = bool(win32api.GetAsyncKeyState(win32con.VK_SHIFT) & 0x8000)
            key = bool(win32api.GetAsyncKeyState(ntilde_vk()) & 0x8000)
        except Exception:
            return
        combos = {"thunder": (ctrl and shift and key and not alt, self.hotkey_thunder),
                  "meteor": (alt and key and not ctrl and not shift, self.hotkey_meteor)}
        for name in self.hk_poll_extra:
            down, cb = combos[name]
            if down and not self.hk_prev.get(name, False):
                cb()
            self.hk_prev[name] = down

    def wiper_mask(self):
        st = self.state
        pos = self.advance_wiper()
        return self.wgeom_grid.mask(pos, st.wiper_half_width)

    # ---------------------------------------------------------------
    # MODO "PANTALLA" — simulacion de ondas (height-field)
    # ---------------------------------------------------------------
    def add_drop_at(self, gx, gy, r, strength):
        """Aplica un impacto gaussiano en (gx,gy), recortado con seguridad si toca el borde."""
        r = max(1, int(r))
        x0, x1 = gx - r, gx + r + 1
        y0, y1 = gy - r, gy + r + 1
        kx0, ky0 = 0, 0
        kx1 = ky1 = 2 * r + 1
        if x0 < 0:
            kx0, x0 = -x0, 0
        if y0 < 0:
            ky0, y0 = -y0, 0
        if x1 > GW:
            kx1 -= (x1 - GW)
            x1 = GW
        if y1 > GH:
            ky1 -= (y1 - GH)
            y1 = GH
        if x1 <= x0 or y1 <= y0:
            return
        kernel = make_drop_kernel(r)
        self.h_cur[y0:y1, x0:x1] -= kernel[ky0:ky1, kx0:kx1] * strength

    def simulate(self):
        h, hp = self.h_cur, self.h_prev
        new = np.zeros_like(h)
        new[1:-1, 1:-1] = (
            h[:-2, 1:-1] + h[2:, 1:-1] + h[1:-1, :-2] + h[1:-1, 2:]
        ) / 2 - hp[1:-1, 1:-1]
        new *= self.state.wave_persistence
        # suavizado ligero: evita el patron "diamante" del stencil de 4 vecinos
        new = cv2.GaussianBlur(new, (3, 3), 0)
        self.h_prev = h
        self.h_cur = new

    def tick_pantalla(self, st):
        # lluvia distribuida en toda la pantalla, con gotas de tamaño variable
        n_drops = st.intensity
        if n_drops > 0:
            base = st.drop_size
            for _ in range(n_drops):
                r = max(1, int(round(base * np.random.uniform(0.25, 2.8))))
                gx = np.random.randint(0, GW)
                gy = np.random.randint(0, GH)
                self.add_drop_at(gx, gy, r, st.strength)

        # click del mouse mueve el agua (la ventana es click-through, asi que
        # se sondea el estado global del boton izquierdo)
        left_down = bool(win32api.GetAsyncKeyState(win32con.VK_LBUTTON) & 0x8000)
        if left_down:
            cx, cy = win32api.GetCursorPos()
            gx, gy = cx - mon["left"], cy - mon["top"]
            if 0 <= gx < GW and 0 <= gy < GH:
                if not self.mouse_down or self.last_mouse_pos is None:
                    self.add_drop_at(gx, gy, st.drop_size * 2.5, st.strength * 3.0)
                else:
                    lx, ly = self.last_mouse_pos
                    dist = max(abs(gx - lx), abs(gy - ly))
                    step_px = max(2, st.drop_size // 2)
                    steps = int(min(60, max(1, dist / step_px)))
                    for i in range(1, steps + 1):
                        t = i / steps
                        ix = int(round(lx + (gx - lx) * t))
                        iy = int(round(ly + (gy - ly) * t))
                        self.add_drop_at(ix, iy, st.drop_size * 1.1, st.strength * 0.7)
                self.last_mouse_pos = (gx, gy)
            else:
                self.last_mouse_pos = None
        else:
            self.last_mouse_pos = None
        self.mouse_down = left_down

        if st.wiper_active:
            mask = self.wiper_mask()
            self.h_cur[mask] = 0
            self.h_prev[mask] = 0

        self.simulate()

    def render_pantalla(self):
        h_r = cv2.resize(self.h_cur, (RW, RH), interpolation=cv2.INTER_CUBIC)

        gx = np.zeros_like(h_r)
        gy = np.zeros_like(h_r)
        gx[:, 1:-1] = (h_r[:, 2:] - h_r[:, :-2]) * 0.5
        gy[1:-1, :] = (h_r[2:, :] - h_r[:-2, :]) * 0.5

        map_x = self.xs + gx * REFRACT_STRENGTH
        map_y = self.ys + gy * REFRACT_STRENGTH

        distorted = cv2.remap(
            self.bg, map_x, map_y,
            interpolation=cv2.INTER_LINEAR, borderMode=cv2.BORDER_REPLICATE
        )

        alpha = np.clip(np.abs(h_r) * 40, 0, 255).astype(np.uint8)
        shade = np.clip(128 + gx * 300, 0, 255).astype(np.uint8)
        distorted = cv2.addWeighted(
            distorted, 0.85, cv2.merge([shade, shade, shade]), 0.15, 0
        )

        bgra = cv2.cvtColor(distorted, cv2.COLOR_BGR2BGRA)
        bgra[:, :, 3] = alpha
        return np.ascontiguousarray(bgra)

    # ---------------------------------------------------------------
    # MODO "VENTANA" — vidrio con vaho, gotas, rastros y truenos
    # ---------------------------------------------------------------
    def rebuild_glass_bg(self):
        """Fondo del vidrio: escritorio desenfocado, aclarado y desaturado (vaho)."""
        st = self.state
        self.bg_s = cv2.resize(self.bg_full, (VW, VH), interpolation=cv2.INTER_AREA)
        sw, sh = max(2, VW // 4), max(2, VH // 4)
        small = cv2.resize(self.bg_s, (sw, sh), interpolation=cv2.INTER_AREA)
        sigma = max(0.5, st.glass_blur / (GLASS_SCALE * 4.0))
        small = cv2.GaussianBlur(small, (0, 0), sigma)
        blur = cv2.resize(small, (VW, VH), interpolation=cv2.INTER_LINEAR).astype(np.float32)
        gray = blur.mean(axis=2, keepdims=True)
        self.fog_img = blur * 0.80 + gray * 0.10 + 255.0 * 0.10
        self.glass_dirty = False
        self._last_blur = st.glass_blur

    # ----- ventana activa -----
    @staticmethod
    def _window_rect(hwnd):
        rect = wintypes.RECT()
        try:
            res = ctypes.windll.dwmapi.DwmGetWindowAttribute(
                wintypes.HWND(hwnd), 9, ctypes.byref(rect), ctypes.sizeof(rect))
            if res == 0:
                return rect.left, rect.top, rect.right, rect.bottom
        except Exception:
            pass
        return win32gui.GetWindowRect(hwnd)

    @staticmethod
    def _is_cloaked(hwnd):
        v = wintypes.DWORD(0)
        try:
            if ctypes.windll.dwmapi.DwmGetWindowAttribute(
                    wintypes.HWND(hwnd), 14, ctypes.byref(v), ctypes.sizeof(v)) == 0:
                return v.value != 0
        except Exception:
            pass
        return False

    @staticmethod
    def _is_maximized(hwnd, rect):
        """True si la ventana esta maximizada (o a pantalla completa sin bordes)."""
        try:
            if win32gui.GetWindowPlacement(hwnd)[1] == win32con.SW_SHOWMAXIMIZED:
                return True
        except Exception:
            pass
        x0, y0, x1, y1 = rect
        return (x1 - x0) >= W - 2 and (y1 - y0) >= H - 2

    def _valid_foreground(self, hwnd):
        try:
            if not win32gui.IsWindowVisible(hwnd) or win32gui.IsIconic(hwnd):
                return False
            if win32gui.GetClassName(hwnd) in SKIP_CLASSES:
                return False
            if self._is_cloaked(hwnd):
                return False
        except Exception:
            return False
        return True

    def update_foreground(self):
        """Calcula el rectangulo de la ventana activa (para dejarla 'dentro', clara)."""
        st = self.state
        if not st.fg_inside:
            self.fg_rect = None
            return
        try:
            hwnd = win32gui.GetForegroundWindow()
        except Exception:
            hwnd = 0
        if hwnd and hwnd in st.own_hwnds:
            hwnd = self.fg_hwnd          # el panel tiene el foco: conserva la anterior
        elif hwnd and not self._valid_foreground(hwnd):
            hwnd = 0                     # escritorio / barra de tareas
        self.fg_hwnd = hwnd
        self.fg_rect = None
        if hwnd and self._valid_foreground(hwnd):
            try:
                l, t, r, b = self._window_rect(hwnd)
            except Exception:
                return
            l -= mon["left"]; r -= mon["left"]
            t -= mon["top"]; b -= mon["top"]
            x0, x1 = max(0, int(l)), min(W, int(math.ceil(r)))
            y0, y1 = max(0, int(t)), min(H, int(math.ceil(b)))
            if x1 > x0 and y1 > y0:
                self.fg_rect = (x0, y0, x1, y1)
                # modo dinamico: maximizada -> "fuera" (empañada); si no -> "dentro" (clara)
                if st.fg_dynamic and self._is_maximized(hwnd, self.fg_rect):
                    self.fg_rect = None

    # ----- gotas -----
    def spawn_drop(self, st, X=None, Y=None, R=None):
        u = st.drop_size * DROP_UNIT
        x = np.random.uniform(0, W)
        y = np.random.uniform(0, H * 0.98)
        if np.random.random() < 0.05:
            r = u * np.random.uniform(1.5, 2.5)               # gotas grandes, muy pocas
        else:
            r = u * (0.25 + 1.1 * np.random.random() ** 2.6)   # mayoria diminutas
        sticky = 0.7 + st.stickiness / 100.0
        pin = u * np.random.uniform(1.25, 2.2) * sticky
        if np.random.random() < 0.2:
            pin *= 1.9                                        # algunas casi nunca resbalan
        grow = np.random.uniform(0, 0.01) * u * u             # condensacion lenta
        # si cae encima de otra gota, se une a ella (la hace crecer)
        if X is not None and len(X):
            dx, dy = X - x, Y - y
            lim = (R + r) * 0.7
            near = np.nonzero(dx * dx + dy * dy < lim * lim)[0]
            if len(near):
                o = self.drops[int(near[0])]
                if not o.dead:
                    self.merge(o, Drop(x, y, r, pin))
                    return
        d = Drop(x, y, r, pin, grow)
        if r * r > d.pin2:
            start_slide(d, 0.5)
        self.drops.append(d)

    def spawn_bead(self, x, y, u):
        """Gotita residual que deja una gota al resbalar."""
        r = np.random.uniform(0.35, 1.0) * u * 0.4 + 0.6
        d = Drop(x, y, r, u * 6.0, 0.0, ttl=int(np.random.uniform(500, 3500)))
        self.drops.append(d)
        return r * r

    def merge(self, a: Drop, b: Drop):
        """Une dos gotas conservando masa y movimiento. Devuelve la que sobrevive."""
        ma, mb = a.m, b.m
        tot = ma + mb
        big, small = (a, b) if ma >= mb else (b, a)
        x = (a.x * ma + b.x * mb) / tot
        y = (a.y * ma + b.y * mb) / tot
        pin2 = (a.pin2 * ma + b.pin2 * mb) / tot
        va = a.v if a.slide else 0.0
        vb = b.v if b.slide else 0.0
        vm = (va * ma + vb * mb) / tot
        big.x, big.y, big.m, big.pin2 = x, y, tot, pin2
        big.grow = min(a.grow, b.grow)
        small.dead = True
        if tot > pin2 or (vm > 1.0 and tot > 0.7 * pin2):
            # el golpe/peso extra la hace resbalar
            if not big.slide:
                start_slide(big, 0.4)
            big.v = max(vm, 0.6) + 0.4
            big.stall = 0
            big.speedmul = max(big.speedmul, small.speedmul)
        else:
            big.slide = False
            big.v = 0.0
        # forma: hereda algo del contorno de la otra gota y, si queda pegada, a
        # veces forma un lobulo hacia donde llego la otra (agua que se une)
        share = small.m / tot
        wgt = 0.5 * share
        big.poly = (big.poly * (1.0 - wgt) + small.poly * wgt).astype(np.float32)
        if (not big.slide) and share > 0.12 and np.random.random() < 0.5:
            ang = math.atan2(small.y - big.y, small.x - big.x)
            pa = np.arctan2(big.poly[:, 1], big.poly[:, 0])
            dth = np.angle(np.exp(1j * (pa - ang)))
            bump = 1.0 + (0.10 + 0.30 * share) * np.exp(-(dth / 0.7) ** 2)
            big.poly = np.clip(big.poly * bump[:, None], -2.4, 2.4).astype(np.float32)
        return big

    def update_drops(self, st):
        u = st.drop_size * DROP_UNIT
        sc = st.strength / 5.0
        drops = self.drops

        # ---- gotas nuevas (condensacion) ----
        rate = st.intensity * 0.25
        k = int(rate) + (1 if np.random.random() < rate - int(rate) else 0)
        if k and drops:
            X = np.array([d.x for d in drops], dtype=np.float32)
            Y = np.array([d.y for d in drops], dtype=np.float32)
            R = np.array([d.r for d in drops], dtype=np.float32)
        else:
            X = Y = R = None
        for _ in range(k):
            self.spawn_drop(st, X, Y, R)
        drops = self.drops

        # (el clic del mouse ahora desempaña el vidrio: ver update_finger)

        # ---- fisica por gota ----
        for d in drops:
            if d.dead:
                continue
            if d.slide:
                self._update_slider(d, u, sc)
                continue
            if d.ttl > 0:                       # gotitas: se evaporan con el tiempo
                d.ttl -= 1
                if d.ttl == 0:
                    d.ttl = -1
            elif d.ttl < 0:
                d.m *= 0.97
                if d.m < 0.35:
                    d.dead = True
                continue
            d.m += d.grow                        # va ganando agua
            if d.m > d.pin2:                     # supera su friccion: empieza a resbalar
                start_slide(d, 0.5)

        # ---- choques: una gota que resbala se une a lo que toque ----
        if any(d.slide and not d.dead for d in drops):
            X = np.array([d.x for d in drops], dtype=np.float32)
            Y = np.array([d.y for d in drops], dtype=np.float32)
            R = np.array([d.r for d in drops], dtype=np.float32)
            for i, d in enumerate(drops):
                if d.dead or not d.slide:
                    continue
                dx, dy = X - d.x, Y - d.y
                lim = (R + d.r) * 0.72
                hit = np.nonzero(dx * dx + dy * dy < lim * lim)[0]
                for j in hit:
                    if j == i:
                        continue
                    o = drops[int(j)]
                    if o.dead or o is d or d.dead:
                        continue
                    self.merge(d, o)

        # ---- limpiar / limite ----
        drops = [d for d in self.drops if not d.dead]
        cap = int(250 + st.intensity * 40)
        if len(drops) > cap:
            drops.sort(key=lambda d: d.m)        # las mas pequeñas se evaporan primero
            for d in drops[:len(drops) - cap]:
                if d.ttl >= 0 and not d.slide:
                    d.ttl = -1
        self.drops = drops

    def _update_slider(self, d: Drop, u: float, sc: float):
        r = d.r
        # --- velocidad: irregular (se atora, arranca, acelera) ---
        if d.stall > 0:
            d.stall -= 1
            target = 0.0
        else:
            target = float(np.clip((0.6 + 0.45 * r) * d.speedmul * sc, 0.3, 14.0))
            # mas probable atorarse cuando ya casi no pesa
            p_stall = 0.010 + 0.03 * max(0.0, (math.sqrt(d.pin2) / max(r, 0.1)) - 0.9)
            if np.random.random() < min(p_stall, 0.2):
                d.stall = int(np.random.randint(6, 70))
        d.v += (target - d.v) * (0.12 if target > d.v else 0.30)
        d.v = max(0.0, d.v + np.random.normal(0, 0.04))
        # --- trayectoria que serpentea ---
        d.ang += np.random.normal(0, 0.045) - 0.08 * d.ang
        d.ang = float(np.clip(d.ang, -0.55, 0.55))
        dx = d.v * math.sin(d.ang)
        dy = d.v * math.cos(d.ang)
        x0, y0 = d.x, d.y
        d.x += dx
        d.y += dy
        dist = math.hypot(dx, dy)

        # --- rastro: ancho y humedad irregulares ---
        d.tw = float(np.clip(d.tw + np.random.normal(0, 0.04) * r, 0.08 * r, 0.4 * r))
        if np.random.random() < 0.02:
            d.tw *= 0.5                          # estrechamiento
        d.wet = float(np.clip(d.wet + np.random.normal(0, 0.08), 0.55, 1.0))
        if dist > 0.05:
            w = d.tw / (1.0 + 0.08 * d.v)
            thick = max(1, int(round(2 * w * GS)))
            col = int(255 * d.wet)
            p0 = (int(round(x0 * GS * 8)), int(round(y0 * GS * 8)))
            p1 = (int(round(d.x * GS * 8)), int(round(d.y * GS * 8)))
            cv2.line(self.trail, p0, p1, col, thick, cv2.LINE_AA, 3)
            if thick > 2:
                cv2.circle(self.trail, (int(d.x * GS), int(d.y * GS)),
                           max(1, thick // 2), col, -1, cv2.LINE_AA)

        # --- pierde agua: se va haciendo mas chica y deja gotitas ---
        d.m -= 0.025 * r * dist
        d.dist_acc += dist
        if d.dist_acc > d.next_bead and d.m > 4.0:
            d.dist_acc = 0.0
            d.next_bead = float(np.random.uniform(10, 60))
            bx = d.x + np.random.normal(0, d.tw * 0.7)
            by = d.y - np.random.uniform(0, max(dist, 1.0))
            d.m = max(0.5, d.m - self.spawn_bead(bx, by, u))

        # --- se detiene (queda pegada) si ya pesa poco ---
        if d.m < 0.64 * d.pin2:
            d.slide = False
            d.v = 0.0
        if d.y - r > H or d.x < -r * 3 or d.x > W + r * 3 or d.m < 0.3:
            d.dead = True

    # ----- vaho / rastros / limpiaparabrisas -----
    def update_fog_and_trail(self, st):
        # rastros: se secan lentamente (duracion = slider de persistencia)
        u = float(np.clip((st.wave_persistence - 0.9) / 0.099, 0.0, 1.0))
        life = 40.0 + 2000.0 * u ** 1.5
        self.trail_acc += 255.0 / life
        n = int(self.trail_acc)
        if n:
            self.trail_acc -= n
            self.trail = cv2.subtract(self.trail, n)

        # desempañado con el dedo: el vidrio vuelve a empañarse poco a poco
        if self.clear_any:
            ticks = max(30.0, st.finger_regrow * 1000.0 / max(10, st.interval_ms))
            self.clear_acc += 255.0 / ticks
            n = int(self.clear_acc)
            if n:
                self.clear_acc -= n
                self.clear = cv2.subtract(self.clear, n)
            if self.frame % 15 == 0 and not self.clear.any():
                self.clear_any = False

        # vaho: no puede superar el nivel pedido y vuelve a empañarse poco a poco
        np.minimum(self.fog, st.fog, out=self.fog)
        self.fog += (st.fog - self.fog) * FOG_REGROW

        if st.wiper_active:
            pos = self.advance_wiper()
            # 1) quita las gotas que toca la pala
            if self.drops:
                xs = np.array([d.x for d in self.drops], dtype=np.float32)
                ys = np.array([d.y for d in self.drops], dtype=np.float32)
                hit = wiper_hit_points(xs, ys, pos, st.wiper_half_width)
                if hit.any():
                    self.drops = [d for d, hh in zip(self.drops, hit) if not hh]
            # 2) seca el rastro y 3) desempaña un poco
            gm = self.wgeom_glass.mask(pos, st.wiper_half_width)
            self.trail[gm] = 0
            cov = cv2.resize(gm.astype(np.uint8) * 255, (FW, FH),
                             interpolation=cv2.INTER_AREA).astype(np.float32) / 255.0
            ticks_under = max(1.0, 2.0 * st.wiper_half_width / max(st.wiper_speed, 1e-3))
            k = 1.0 - (1.0 - float(np.clip(st.wiper_defog, 0, 0.99))) ** (1.0 / ticks_under)
            self.fog *= (1.0 - k * cov)

    # ----- desempañar con el mouse (como pasar el dedo por un vidrio empañado) -----
    def _cursor_over_panel(self, cx, cy):
        """True si el clic empezo sobre el panel de control (no se desempaña)."""
        try:
            hw = win32gui.WindowFromPoint((int(cx), int(cy)))
            if not hw:
                return False
            root = win32gui.GetAncestor(hw, win32con.GA_ROOT) or hw
            return (root in self.state.own_hwnds) and root != int(self.winId())
        except Exception:
            return False

    @staticmethod
    def _seg_dist(xs, ys, p0, p1):
        ax, ay = p0
        bx, by = p1
        abx, aby = bx - ax, by - ay
        l2 = abx * abx + aby * aby + 1e-6
        t = np.clip(((xs - ax) * abx + (ys - ay) * aby) / l2, 0.0, 1.0)
        return np.hypot(xs - (ax + t * abx), ys - (ay + t * aby))

    def update_finger(self, st):
        left_down = bool(win32api.GetAsyncKeyState(win32con.VK_LBUTTON) & 0x8000)
        u = st.drop_size * DROP_UNIT
        if not st.finger_defog:
            self.f_down, self.f_active, self.f_last = left_down, False, None
            return
        if left_down:
            cx, cy = win32api.GetCursorPos()
            gx, gy = cx - mon["left"], cy - mon["top"]
            if not self.f_down:                      # recien presionado
                self.f_active = not self._cursor_over_panel(cx, cy)
                self.f_last = None
                self.finger_water = 0.0
            if self.f_active and 0 <= gx < W and 0 <= gy < H:
                self._finger_wipe(st, self.f_last, (gx, gy), u)
                self.f_last = (gx, gy)
            else:
                self.f_last = None
        else:
            if self.f_down and self.f_active and self.f_last is not None \
                    and self.finger_water > 2.0 * u * u:
                self._spawn_finger_drop(self.f_last, None, u)
            self.f_last, self.f_active, self.finger_water = None, False, 0.0
        self.f_down = left_down

    def _finger_wipe(self, st, p0, p1, u):
        R = float(st.finger_size)
        if p0 is None:
            p0 = p1
        s16 = GS * 16.0
        a = (int(round(p0[0] * s16)), int(round(p0[1] * s16)))
        b = (int(round(p1[0] * s16)), int(round(p1[1] * s16)))
        th_core = max(1, int(round(2 * R * GS)))
        th_soft = max(2, int(round(2 * R * GS * 1.35)))
        r_core = int(round(R * s16))
        r_soft = int(round(R * 1.35 * s16))

        # vidrio limpio: nucleo total + borde suave (se acumula con maximo)
        tmp = np.zeros((VH, VW), dtype=np.uint8)
        cv2.line(tmp, a, b, 140, th_soft, cv2.LINE_AA, 4)
        cv2.circle(tmp, b, r_soft, 140, -1, cv2.LINE_AA, 4)
        cv2.line(tmp, a, b, 255, th_core, cv2.LINE_AA, 4)
        cv2.circle(tmp, b, r_core, 255, -1, cv2.LINE_AA, 4)
        np.maximum(self.clear, tmp, out=self.clear)
        self.clear_any = True

        # el dedo tambien seca los rastros humedos
        cv2.line(self.trail, a, b, 0, th_core, cv2.LINE_8, 4)
        cv2.circle(self.trail, b, r_core, 0, -1, cv2.LINE_8, 4)

        # barre las gotas que toca y junta su agua
        if self.drops:
            xs = np.array([d.x for d in self.drops], dtype=np.float32)
            ys = np.array([d.y for d in self.drops], dtype=np.float32)
            rad = np.array([d.r for d in self.drops], dtype=np.float32)
            hit = self._seg_dist(xs, ys, p0, p1) < (R + rad * 0.4)
            if hit.any():
                for d, h in zip(self.drops, hit):
                    if h and not d.dead:
                        self.finger_water += d.m
                        d.dead = True
        if self.finger_water > (u * 3.2) ** 2:
            self._spawn_finger_drop(p1, p0, u)

    def _spawn_finger_drop(self, p, prev, u):
        """El agua barrida se junta en una gota al borde del dedo y escurre."""
        mass = self.finger_water * 0.6
        r = min(math.sqrt(max(mass, 1.0)), u * 3.2)
        if prev is not None and (prev[0] != p[0] or prev[1] != p[1]):
            dx, dy = p[0] - prev[0], p[1] - prev[1]
            ln = math.hypot(dx, dy)
            nx, ny = -dy / ln, dx / ln
        else:
            ang = np.random.uniform(0, 2 * np.pi)
            nx, ny = math.cos(ang), math.sin(ang)
        side = 1.0 if np.random.random() < 0.5 else -1.0
        off = float(self.state.finger_size) * np.random.uniform(0.85, 1.1)
        x = p[0] + nx * side * off
        y = p[1] + ny * side * off
        sticky = 0.7 + self.state.stickiness / 100.0
        pin = u * np.random.uniform(1.3, 2.0) * sticky
        d = Drop(x, y, r, pin, 0.0)
        if r * r > d.pin2:
            start_slide(d, 0.5)
        self.drops.append(d)
        self.finger_water *= 0.35

    # ----- lluvia cayendo "afuera" (detras del vidrio) -----
    def init_outside_rain(self):
        n = RAIN_MAX
        ky = VH / 540.0
        self.rz = (np.random.random(n) ** 1.3).astype(np.float32)        # 0 lejos .. 1 cerca
        self.rv = ((12.0 + 34.0 * self.rz) * ky *
                   np.random.uniform(0.9, 1.1, n)).astype(np.float32)     # px/tick
        self.rlen = (self.rv * np.random.uniform(0.9, 1.4, n)).astype(np.float32)
        self.rslope = np.random.normal(0.13, 0.015, n).astype(np.float32)  # viento
        self.rx = np.random.uniform(-0.2 * VW, 1.0 * VW, n).astype(np.float32)
        self.ry = np.random.uniform(-0.2 * VH, VH, n).astype(np.float32)
        self.rval = (55 + 175 * self.rz).astype(np.int32).tolist()
        self.rth = np.where(self.rz > 0.78, 2, 1).astype(np.int32).tolist()

    def update_outside_rain(self, st):
        if not st.rain_outside:
            self.rain_img = None
            return
        n = int(np.clip(st.rain_amount, 1, 100) * RAIN_MAX / 100)
        s_ = float(GLASS_SCALE)
        mode = st.rain_dir
        if mode in ("toward", "away"):
            x0, y0, x1, y1, vals = self._rain_radial(n, mode)
        else:
            x, y, v = self.rx[:n], self.ry[:n], self.rv[:n]
            L = self.rlen[:n]
            slope0 = float(np.clip(st.rain_incl, 0, 100)) * 0.0065   # 20 -> 0.13 (como antes)
            f = self.rslope[:n] / 0.13                               # variacion por gota
            if mode == "variable":
                # direccion CAMBIANTE: cada cierto tiempo elige un lado y una inclinacion
                # nuevos (a veces casi vertical) y se acerca suave a ellos
                self.rwv_timer -= 1
                if self.rwv_timer <= 0:
                    self.rwv_timer = int(np.random.randint(60, 260))
                    if np.random.random() < 0.15:
                        self.rwv_tgt = float(np.random.uniform(-0.15, 0.15))
                    else:
                        self.rwv_tgt = float(np.random.choice((-1.0, 1.0))
                                             * np.random.uniform(0.35, 1.0))
                self.rwv += (self.rwv_tgt - self.rwv) * 0.02
                amp = max(0.30, 3.0 * slope0)
                micro = 0.92 + 0.08 * math.sin(self.frame * 0.13)
                sl = self.rwv * micro * amp * f
                ext = amp * VH * 1.05
                xlo, xhi = -ext, VW + ext
            else:
                sgn = 1.0 if mode == "right" else -1.0
                sl = sgn * slope0 * f
                ext = max(0.2 * VW, slope0 * 1.2 * VH * 1.05)
                xlo, xhi = (-ext, VW) if sgn > 0 else (0.0, VW + ext)
            x += v * sl
            y += v
            gone = ((y - L) > VH) | (x < xlo - 0.1 * VW) | (x > xhi + 0.1 * VW)
            k = int(gone.sum())
            if k:
                y[gone] = -np.random.uniform(0.0, 0.3 * VH, k)
                x[gone] = np.random.uniform(xlo, xhi, k)
            x0 = ((x - sl * L) * s_).astype(np.int32).tolist()
            y0 = ((y - L) * s_).astype(np.int32).tolist()
            x1 = (x * s_).astype(np.int32).tolist()
            y1 = (y * s_).astype(np.int32).tolist()
            vals = self.rval
        # se dibuja a RESOLUCION COMPLETA con lineas de 1 px: hilos finos y nitidos
        # (antes se dibujaba a mitad de resolucion y salian gruesos)
        img = np.zeros((H, W), dtype=np.uint8)
        for i in range(n):
            cv2.line(img, (x0[i], y0[i]), (x1[i], y1[i]), vals[i], 1, cv2.LINE_AA)
        self.rain_img = cv2.GaussianBlur(img, (0, 0), 0.6)    # apenas suave: esta afuera

    def _rain_radial(self, n, mode):
        """Lluvia que viene HACIA la pantalla (rayas que salen del centro) o se ALEJA de ella
        (rayas que van hacia el centro): mas largas y rapidas cuanto mas cerca de ti."""
        cx, cy = VW / 2.0, VH / 2.0
        Rm = math.hypot(cx, cy)
        x, y, v, z = self.rx[:n], self.ry[:n], self.rv[:n], self.rz[:n]
        dx, dy = x - cx, y - cy
        r = np.hypot(dx, dy)
        ang = np.arctan2(dy, dx)
        tiny = r < 1.0
        if tiny.any():
            ang[tiny] = np.random.uniform(0, 2 * np.pi, int(tiny.sum()))
        r = np.maximum(r, 1.0)
        if mode == "toward":
            step = r * (0.035 + 0.05 * z) + v * 0.12
            rn = r + step
            gone = rn > Rm * 1.12
            k = int(gone.sum())
            if k:                                    # reaparecen cerca del centro (lejos)
                rn[gone] = np.random.uniform(0.01, 0.30, k) * Rm
                ang[gone] = np.random.uniform(0, 2 * np.pi, k)
                step = np.where(gone, v * 0.12, step)
            head = rn
            tail = np.maximum(rn - step * 1.8, 0.0)
        else:
            step = r * (0.03 + 0.045 * z) + v * 0.05
            rn = r - step
            gone = rn < 0.04 * Rm
            k = int(gone.sum())
            if k:                                    # reaparecen en la orilla (cerca)
                rn[gone] = np.random.uniform(0.55, 1.10, k) * Rm
                ang[gone] = np.random.uniform(0, 2 * np.pi, k)
                step = np.where(gone, rn * 0.04, step)
            head = rn
            tail = rn + step * 1.8
        ca, sa = np.cos(ang), np.sin(ang)
        x[:] = cx + ca * rn
        y[:] = cy + sa * rn
        fade = np.clip(head / (0.28 * Rm), 0.0, 1.0)         # lejos / al centro: mas tenue
        vals = (np.array(self.rval[:n], dtype=np.float32) * fade).astype(np.int32).tolist()
        s_ = float(GLASS_SCALE)
        x0 = ((cx + ca * tail) * s_).astype(np.int32).tolist()
        y0 = ((cy + sa * tail) * s_).astype(np.int32).tolist()
        x1 = ((cx + ca * head) * s_).astype(np.int32).tolist()
        y1 = ((cy + sa * head) * s_).astype(np.int32).tolist()
        return x0, y0, x1, y1, vals

    # ----- truenos -----
    def schedule_next(self, first=False):
        st = self.state
        freq = max(1, st.thunder_freq)
        mean = 90.0 / (freq ** 1.1)
        if first:
            delay = np.random.uniform(2.0, 5.0)
        else:
            delay = float(np.clip(np.random.exponential(mean), 2.5, 4 * mean))
        self.next_strike = time.monotonic() + delay

    def _bolt_path(self, p0, p1, rough, levels):
        """Rayo por desplazamiento de punto medio (fractal): zigzag natural."""
        pts = [p0, p1]
        r = rough
        for _ in range(levels):
            new = [pts[0]]
            for a, b in zip(pts[:-1], pts[1:]):
                dx, dy = b[0] - a[0], b[1] - a[1]
                ln = math.hypot(dx, dy)
                if ln > 1e-6:
                    nx, ny = -dy / ln, dx / ln
                else:
                    nx = ny = 0.0
                off = np.random.uniform(-1, 1) * ln * r
                new.append(((a[0] + b[0]) / 2 + nx * off, (a[1] + b[1]) / 2 + ny * off))
                new.append(b)
            pts = new
            r *= 0.8
        return pts

    def make_bolt(self):
        """Genera el rayo (tronco fractal + ramas) y su resplandor. Devuelve x de origen."""
        kx = VW / 960.0
        ky = VH / 540.0
        img = np.zeros((VH, VW), dtype=np.uint8)

        def draw(pts, val, th):
            arr = (np.array(pts, dtype=np.float32) * 16).astype(np.int32).reshape(-1, 1, 2)
            cv2.polylines(img, [arr], False, val, th, cv2.LINE_AA, 4)

        x0 = float(np.random.uniform(0.15, 0.85) * VW)
        y_end = float(VH * np.random.uniform(0.5, 0.9))
        x_end = x0 + float(np.random.normal(0, 0.10) * VW)
        main = self._bolt_path((x0, -4.0), (x_end, y_end), 0.28, 7)
        draw(main, 255, max(2, int(round(2.5 * kx))))
        n = len(main)
        for _ in range(int(np.random.randint(3, 8))):
            i = int(np.random.randint(int(n * 0.1), int(n * 0.85)))
            bx, by = main[i]
            j, k = min(i + 4, n - 1), max(i - 4, 0)
            ang0 = math.atan2(main[j][0] - main[k][0], main[j][1] - main[k][1])
            ang = ang0 + float(np.random.choice([-1, 1])) * float(np.random.uniform(0.35, 0.95))
            L = float(np.random.uniform(0.10, 0.32) * VH)
            ex = bx + math.sin(ang) * L
            ey = by + abs(math.cos(ang)) * L
            br = self._bolt_path((bx, by), (ex, ey), 0.30, 5)
            draw(br, 170, max(1, int(round(1.2 * kx))))
            if np.random.random() < 0.4:                      # sub-rama
                mx, my = br[len(br) // 2]
                a2 = ang + float(np.random.choice([-1, 1])) * float(np.random.uniform(0.4, 0.9))
                L2 = L * float(np.random.uniform(0.3, 0.55))
                sub = self._bolt_path((mx, my),
                                      (mx + math.sin(a2) * L2, my + abs(math.cos(a2)) * L2),
                                      0.32, 4)
                draw(sub, 110, 1)

        f = img.astype(np.float32) * (1.0 / 255.0)
        core = cv2.GaussianBlur(f, (0, 0), 1.0)
        h1 = cv2.GaussianBlur(f, (0, 0), max(2.0, 3.0 * kx))
        sm = cv2.resize(f, (max(2, VW // 4), max(2, VH // 4)), interpolation=cv2.INTER_AREA)
        h2 = cv2.GaussianBlur(sm, (0, 0), max(2.0, 3.5 * kx))
        h2 = cv2.resize(h2, (VW, VH), interpolation=cv2.INTER_LINEAR)
        h1 = h1 / (float(h1.max()) + 1e-6)
        h2 = h2 / (float(h2.max()) + 1e-6)
        self.bolt = np.clip(core * 1.5, 0, 1).astype(np.float32)
        self.bolt_halo = np.clip(h1 * 0.55 + h2 * 0.45, 0, 1).astype(np.float32)
        return x0

    def make_flash_map(self, cx, cy, sx_frac, amb):
        """Iluminacion del relampago: mas fuerte cerca del rayo, suave en el resto."""
        sx = VW * sx_frac
        sy = VH * 0.55
        g = np.exp(-0.5 * (((self.gxs - cx) / sx) ** 2 + ((self.gys - cy) / sy) ** 2))
        return (amb + (1.0 - amb) * g).astype(np.float32)

    def trigger_strike(self):
        st = self.state
        inten = st.thunder_intensity / 10.0
        dist = float(np.random.uniform(0.15, 1.0))     # que tan lejos cae
        pulses, t = [], 0.0
        for i in range(int(np.random.randint(1, 5))):  # trazo principal + retornos
            amp = 1.0 if i == 0 else float(np.random.uniform(0.45, 1.0))
            pulses.append((t, amp, float(np.random.uniform(0.05, 0.13))))
            t += float(np.random.uniform(0.05, 0.20))
        self.pulses = pulses
        self.strike_t0 = time.monotonic()
        self.flash_amp = (0.3 + 0.7 * inten) * (1.1 - 0.5 * dist)
        self.bolt = None
        self.bolt_halo = None
        if dist < 0.8 and np.random.random() < 0.8:
            bx = self.make_bolt()
            self.bolt_gain = float(np.clip(1.15 - 0.6 * dist + 0.2 * inten, 0.35, 1.2))
            self.flash_map = self.make_flash_map(bx, VH * 0.25, 0.30, 0.30 + 0.35 * dist)
        else:                                           # relampago lejano dentro de las nubes
            self.bolt_gain = 0.0
            self.flash_map = self.make_flash_map(
                float(np.random.uniform(0.1, 0.9) * VW),
                float(np.random.uniform(0.05, 0.35) * VH), 0.45, 0.55)
        if st.thunder_sound:
            delay_ms = int((0.2 + dist * 3.0) * 1000)
            vol = (0.35 + 0.65 * inten) * (1.15 - 0.6 * dist)
            QtCore.QTimer.singleShot(delay_ms, lambda v=vol: self.thunder_sound.play(v))

    def update_thunder(self, st):
        now = time.monotonic()
        if st.thunder_test:
            st.thunder_test = False
            self.trigger_strike()
        if st.thunder_on:
            if self.next_strike is None:
                self.schedule_next(first=True)
            elif now >= self.next_strike:
                self.trigger_strike()
                self.schedule_next()
        else:
            self.next_strike = None

        f = 0.0
        bv = 0.0
        if self.pulses:
            t = now - self.strike_t0
            for t_i, a, dc in self.pulses:
                if t >= t_i:
                    f = max(f, a * math.exp(-(t - t_i) / dc))
                    bv = max(bv, a * math.exp(-(t - t_i) / (dc * 0.55)))
            f = max(f, 0.22 * math.exp(-t / 0.55))      # resplandor residual en las nubes
            if t > self.pulses[-1][0] + 1.2:
                self.pulses = []
                self.bolt = None
                self.bolt_halo = None
                self.flash_map = None
        self.flash = f * self.flash_amp
        self.bolt_vis = bv * self.bolt_gain * float(np.random.uniform(0.85, 1.0))

    def tick_ventana(self, st):
        if self.glass_dirty or st.glass_blur != self._last_blur:
            self.rebuild_glass_bg()
        self.update_foreground()
        self.update_finger(st)
        self.update_drops(st)
        self.update_fog_and_trail(st)
        self.update_thunder(st)
        self.update_outside_rain(st)

    # ----- dibujo -----
    def draw_drops_fullres(self, out, fl):
        """Gotas a resolucion completa: cada una es un lente (imagen invertida y
        reducida del fondo nitido) con borde oscuro, brillo y creciente luminoso."""
        drops = self.drops
        if not drops:
            return
        n = len(drops)
        P = np.stack([d.poly for d in drops])                       # (n,N,2)
        r = np.array([d.r for d in drops], dtype=np.float32)
        cx = np.array([d.x for d in drops], dtype=np.float32)
        cy = np.array([d.y for d in drops], dtype=np.float32)
        # estiramiento (gota que resbala: cabeza redonda, cola angosta hacia arriba)
        for d in drops:
            tgt = min(1.0, d.v / 4.0) if d.slide else 0.0
            d.stretch += (tgt - d.stretch) * 0.2
        st_ = np.array([d.stretch for d in drops], dtype=np.float32)[:, None]
        X, Y = P[..., 0], P[..., 1]
        Y2 = np.where(Y < 0, Y * (1 + st_ * 1.8), Y * (1 - 0.12 * st_))
        X2 = X * (1 - 0.35 * st_ * np.clip(-Y, 0, 1))
        rr = np.maximum(r, 0.8)
        px = (X2 * rr[:, None] + cx[:, None]) * 16.0
        py = (Y2 * rr[:, None] + cy[:, None]) * 16.0
        pts = np.stack([px, py], axis=2).astype(np.int32)

        M = np.zeros((H, W), dtype=np.uint8)                        # cobertura suave
        cv2.fillPoly(M, list(pts), 255, lineType=cv2.LINE_AA, shift=4)
        ids = np.zeros((H, W), dtype=np.uint16)                     # dueña de cada pixel (+1)
        for i in range(n):
            cv2.fillPoly(ids, [pts[i]], i + 1, lineType=cv2.LINE_8, shift=4)
        flat = np.flatnonzero(ids)
        if len(flat) == 0:
            return
        oid = ids.ravel()[flat].astype(np.int32) - 1
        ys, xs = np.divmod(flat, W)
        m = M.ravel()[flat].astype(np.float32) * (1 / 255.0)
        ccx, ccy, rad = cx[oid], cy[oid], rr[oid]
        dx = xs.astype(np.float32) - ccx
        dy = ys.astype(np.float32) - ccy
        rx, ry = dx / rad, dy / rad
        rho = np.sqrt(rx * rx + ry * ry)

        # lente: se ve el fondo INVERTIDO y reducido (como una gota real)
        k = 1.7
        sx = np.clip(ccx - k * dx, 0, W - 1).astype(np.int32)
        sy = np.clip(ccy - k * dy, 0, H - 1).astype(np.int32)
        col = self.bg_full[sy, sx].astype(np.float32)

        rc = np.minimum(rho, 1.3) / 1.3
        rc2 = rc * rc
        nd = (rx * 0.55 + ry * 0.83) / np.maximum(rho, 1e-3)
        caus = np.clip(nd, 0, 1)
        caus = caus * caus * caus * rc2 * 150.0                      # creciente luminoso abajo-der
        dark = np.clip(-nd, 0, 1) * rc2 * 75.0 + rc2 * rc2 * rc2 * 35.0   # borde oscuro arriba-izq
        spec = np.exp(-((rx + 0.42) ** 2 + (ry + 0.5) ** 2) / 0.05) * 200.0  # puntito de brillo
        light = caus + spec - dark + fl * 90.0
        col = col * 0.92 + light[:, None]

        o4 = out.reshape(-1, 4)
        px4 = o4[flat].astype(np.float32)
        mm = m[:, None]
        px4[:, :3] += (col - px4[:, :3]) * mm
        px4[:, 3] += (255.0 - px4[:, 3]) * m
        np.clip(px4, 0, 255, out=px4)
        o4[flat] = px4.astype(np.uint8)

    def render_ventana(self):
        st = self.state
        fl = float(self.flash)

        # --- vaho -> opacidad del vidrio (resolucion de vidrio) ---
        Fup = cv2.resize(self.fog, (VW, VH), interpolation=cv2.INTER_LINEAR)
        light_t = None
        edge_a = None

        # --- desempañado con el dedo: vidrio limpio + borde de agua ---
        if self.clear_any:
            Cf = cv2.GaussianBlur(self.clear.astype(np.float32) * (1 / 255.0), (0, 0), 1.5)
            Fup = Fup * (1.0 - np.clip(Cf * 1.15, 0, 1))
            cgx = cv2.Sobel(Cf, cv2.CV_32F, 1, 0, ksize=3, scale=0.125)
            cgy = cv2.Sobel(Cf, cv2.CV_32F, 0, 1, ksize=3, scale=0.125)
            ce = (cgx * 0.6 + cgy * 0.8) * 3.0
            chi = np.clip(ce, 0, 0.6)
            cdk = np.clip(-ce, 0, 0.6)
            edge_a = (chi + cdk) * 0.8
            light_t = chi * 140.0 - cdk * 65.0

        # --- rastros (pelicula de agua) ---
        if self.trail.any():
            Tf = cv2.GaussianBlur(self.trail.astype(np.float32) * (1 / 255.0), (0, 0), 1.0)
            tgx = cv2.Sobel(Tf, cv2.CV_32F, 1, 0, ksize=3, scale=0.125)
            tgy = cv2.Sobel(Tf, cv2.CV_32F, 0, 1, ksize=3, scale=0.125)
            tf = np.clip(Tf * 1.3, 0, 1)
            te = (tgx * 0.6 + tgy * 0.8) * 3.0
            hi = np.clip(te, 0, 0.6)
            dk = np.clip(-te, 0, 0.6)
            a_f = Fup * (1.0 - tf * 0.92) * 0.96
            a_f = np.maximum(a_f, (hi + dk) * 0.8)       # bordes del rastro siempre se notan
            lt = hi * 150.0 - dk * 70.0
            light_t = lt if light_t is None else light_t + lt
        else:
            a_f = Fup * 0.96
        if edge_a is not None:
            a_f = np.maximum(a_f, edge_a)

        C = self.fog_img.copy()
        if light_t is not None:
            C += light_t[..., None]

        # --- destello de relampago: mas intenso cerca del rayo ---
        if fl > 0.02:
            fm = self.flash_map if self.flash_map is not None else 1.0
            fk = np.clip(fl * 0.75 * fm, 0, 1)
            fk3 = np.asarray(fk, dtype=np.float32)[..., None]
            fcol = np.array([255.0, 240.0, 225.0], dtype=np.float32)
            C = C * (1.0 - fk3) + fcol * fk3
            a_f = a_f + (0.97 - a_f) * np.clip(fl * 0.55 * fm, 0, 1)
        alpha = a_f

        # --- rayo: nucleo blanco + halo azulado ---
        if self.bolt is not None and self.bolt_halo is not None and self.bolt_vis > 0.03:
            bv = min(1.0, self.bolt_vis)
            core = self.bolt * bv
            halo = self.bolt_halo * bv
            C += (core[..., None] * np.array([255.0, 248.0, 240.0], dtype=np.float32) * 0.95
                  + halo[..., None] * np.array([255.0, 205.0, 170.0], dtype=np.float32) * 0.55)
            alpha = np.maximum(alpha, np.maximum(core * 0.95, halo * 0.6))

        np.clip(C, 0, 255, out=C)
        half = np.empty((VH, VW, 4), dtype=np.uint8)
        half[:, :, :3] = C.astype(np.uint8)
        half[:, :, 3] = (np.clip(alpha, 0, 1) * 255).astype(np.uint8)

        # --- a resolucion completa: vidrio + gotas nitidas ---
        out = cv2.resize(half, (W, H), interpolation=cv2.INTER_LINEAR)

        # --- lluvia cayendo afuera (detras del vidrio): hilos finos a resolucion completa ---
        if st.rain_outside and self.rain_img is not None:
            ry, rx = np.nonzero(self.rain_img)
            if ry.size:
                R = self.rain_img[ry, rx].astype(np.float32) * (1 / 255.0)
                rk = np.clip(R * (1.05 + 0.5 * fl), 0, 0.9)[:, None]
                rcol = np.array([250.0, 222.0, 190.0], dtype=np.float32) * (1.0 + 0.25 * fl)
                px = out[ry, rx]
                col = px[:, :3].astype(np.float32) * (1.0 - rk) + rcol * rk
                a_r = np.maximum(px[:, 3].astype(np.float32), R * 0.75 * 255.0)
                px[:, :3] = np.clip(col, 0, 255).astype(np.uint8)
                px[:, 3] = np.clip(a_r, 0, 255).astype(np.uint8)
                out[ry, rx] = px

        self.draw_drops_fullres(out, fl)

        # --- ventana activa "dentro" del vidrio: se ve clara ---
        if st.fg_inside and self.fg_rect is not None:
            x0, y0, x1, y1 = self.fg_rect
            out[y0:y1, x0:x1, 3] = 0
        return out

    # ---------------------------------------------------------------
    # ESTILO "NIEVE" — copos al fondo + escarcha en el vidrio
    # ---------------------------------------------------------------
    @staticmethod
    def _fbm(rng, base, octs, persistence=0.5):
        """Ruido fractal 0..1 a resolucion de vidrio."""
        out = np.zeros((VH, VW), dtype=np.float32)
        amp, tot, cell = 1.0, 0.0, float(base)
        for _ in range(octs):
            gh, gw = max(2, int(VH / cell) + 2), max(2, int(VW / cell) + 2)
            r = rng.random((gh, gw)).astype(np.float32)
            out += amp * cv2.resize(r, (VW, VH), interpolation=cv2.INTER_CUBIC)
            tot += amp
            amp *= persistence
            cell = max(2.0, cell / 2.0)
        out /= tot
        lo, hi = float(out.min()), float(out.max())
        return (out - lo) / max(hi - lo, 1e-6)

    @staticmethod
    def _fbm_aniso(rng, bx, by, octs, persistence=0.5):
        """Ruido fractal 0..1 ANISOTROPO (celdas de bx x by px): da vetas alargadas."""
        out = np.zeros((VH, VW), dtype=np.float32)
        amp, tot, cx, cy = 1.0, 0.0, float(bx), float(by)
        for _ in range(octs):
            gh, gw = max(2, int(VH / cy) + 2), max(2, int(VW / cx) + 2)
            r = rng.random((gh, gw)).astype(np.float32)
            out += amp * cv2.resize(r, (VW, VH), interpolation=cv2.INTER_CUBIC)
            tot += amp
            amp *= persistence
            cx, cy = max(2.0, cx / 2.0), max(2.0, cy / 2.0)
        out /= tot
        lo, hi = float(out.min()), float(out.max())
        return (out - lo) / max(hi - lo, 1e-6)

    @staticmethod
    def _fern_branch(canvas, x, y, ang, length, step, curve, rng, col, th):
        """Dibuja una rama curva (polilinea) y devuelve sus puntos y angulos."""
        n = max(2, int(length / step))
        angs = ang + np.cumsum(rng.normal(curve, 0.05, n))
        xs = x + np.concatenate([[0.0], np.cumsum(np.cos(angs) * step)])
        ys = y + np.concatenate([[0.0], np.cumsum(np.sin(angs) * step)])
        pts = np.round(np.stack([xs, ys], axis=1)).astype(np.int32).reshape(-1, 1, 2)
        cv2.polylines(canvas, [pts], False, col, th)
        return xs, ys, angs

    def _draw_ferns(self, rng, k):
        """Helechos de hielo (dendritas): tallo curvo + ramas inclinadas hacia la punta
        + sub-ramas, naciendo en los bordes (sobre todo abajo) como en una ventana real."""
        canvas = np.zeros((VH, VW), dtype=np.uint8)
        step = max(1.5, 3.0 * k)
        for f in range(22):
            r = rng.random()
            if r < 0.40:        # borde de abajo
                x, y, a = rng.uniform(0, VW), VH + 4.0 * k, -np.pi / 2 + rng.normal(0, 0.35)
                L = rng.uniform(80, 320) * k
            elif r < 0.65:      # laterales
                if rng.random() < 0.5:
                    x, y, a = -4.0 * k, rng.uniform(0.2 * VH, VH), rng.normal(0, 0.5)
                else:
                    x, y, a = VW + 4.0 * k, rng.uniform(0.2 * VH, VH), np.pi + rng.normal(0, 0.5)
                L = rng.uniform(70, 280) * k
            elif r < 0.80:      # borde de arriba
                x, y, a = rng.uniform(0, VW), -4.0 * k, np.pi / 2 + rng.normal(0, 0.35)
                L = rng.uniform(60, 220) * k
            else:               # algunos naciendo dentro del vidrio
                x, y, a = rng.uniform(0, VW), rng.uniform(0, VH), rng.uniform(0, 2 * np.pi)
                L = rng.uniform(40, 150) * k
            sx, sy, sa = self._fern_branch(canvas, x, y, a, L, step, rng.normal(0, 0.012),
                                           rng, 255, 2 if L > 150 * k else 1)
            n = len(sa)
            spread = rng.uniform(0.85, 1.15)
            for i in range(3, n, 5):
                frac = i / float(n)
                bl = L * 0.40 * (1.0 - frac) ** 0.75 * rng.uniform(0.75, 1.15) + 5.0 * k
                for sgn in (-1.0, 1.0):
                    bx, by, ba = self._fern_branch(canvas, sx[i], sy[i], sa[i - 1] + sgn * spread,
                                                   bl, step, -sgn * 0.02, rng, 215, 1)
                    m = len(ba)
                    for j in range(2, m, 3):
                        sl = bl * 0.30 * (1.0 - j / float(m)) + 2.0 * k
                        for sg2 in (-1.0, 1.0):
                            a2 = ba[j - 1] + sg2 * spread
                            cv2.line(canvas, (int(bx[j]), int(by[j])),
                                     (int(bx[j] + np.cos(a2) * sl), int(by[j] + np.sin(a2) * sl)),
                                     165, 1)
        fern = cv2.GaussianBlur(canvas.astype(np.float32) * (1.0 / 255.0), (0, 0), 0.8 * max(k, 0.7))
        return np.clip(fern * 1.35, 0, 1).astype(np.float32)

    def init_frost(self):
        """Escarcha realista (se genera una sola vez):
          * helechos de hielo dibujados (tallo + ramas + sub-ramas) que nacen en bordes y
            esquinas, y entre ellos escarcha fina granulada, TRANSLUCIDA (se ve el fondo);
          * relieve con luz y destellos en los cristales;
          * 'orden de aparicion' U (uniforme 0..1): primero se congelan los helechos de los
            bordes (mas abajo) y luego el resto; asi el slider = % real de vidrio cubierto y
            se puede poner muy poquita escarcha (antes saltaba de golpe)."""
        rng = np.random.default_rng()
        k = VW / 960.0
        gx, gy = np.meshgrid(np.arange(VW, dtype=np.float32), np.arange(VH, dtype=np.float32))

        def ridge(n, p):
            return np.clip(1.0 - np.abs(2.0 * n - 1.0), 0, 1) ** p

        fern = self._draw_ferns(rng, k)

        # escarcha fina entre los helechos: agujas muy finas + granito (no fibras onduladas)
        wx = self._fbm(rng, 30 * k, 3)
        wy = self._fbm(rng, 30 * k, 3)
        n1 = self._fbm(rng, 7 * k, 3, 0.55)
        n1 = cv2.remap(n1, gx + (wx - 0.5) * 6.0 * k, gy + (wy - 0.5) * 6.0 * k,
                       interpolation=cv2.INTER_LINEAR, borderMode=cv2.BORDER_REFLECT)
        needles = ridge(n1, 4.0).astype(np.float32)
        grain = cv2.GaussianBlur(rng.random((VH, VW)).astype(np.float32), (0, 0), 0.8)
        grain = (grain - grain.min()) / max(float(grain.max() - grain.min()), 1e-6)

        cry = np.clip(np.maximum(fern, 0.55 * needles), 0, 1).astype(np.float32)

        # --- textura de opacidad (translucida) ---
        tex = (0.13 + 0.62 * fern + 0.24 * needles) * (0.82 + 0.36 * grain)
        self.frost_tex = np.clip(tex, 0.05, 0.93).astype(np.float32)

        # --- brillo: relieve de los cristales + destellos ---
        h = cv2.GaussianBlur(cry, (0, 0), 0.8)
        lx = cv2.Sobel(h, cv2.CV_32F, 1, 0, ksize=3, scale=0.125)
        ly = cv2.Sobel(h, cv2.CV_32F, 0, 1, ksize=3, scale=0.125)
        light = (lx * 0.6 + ly * 0.8) * 1.2
        spark = (rng.random((VH, VW)) > 0.9978).astype(np.float32)
        spark = cv2.GaussianBlur(spark, (0, 0), 0.7) * 5.0
        self.frost_mod = np.clip(0.82 + 0.28 * cry + light + 0.55 * spark * cry,
                                 0.55, 1.45).astype(np.float32)

        # --- campo de avance -> orden de aparicion U (0 = primero en congelarse) ---
        ex = np.clip(np.minimum(gx, VW - 1 - gx) / (0.5 * VW), 0, 1)
        ey = np.clip(np.minimum(gy * 1.15, (VH - 1 - gy) * 0.65) / (0.5 * VH), 0, 1)
        d = np.sqrt(ex * ey)                       # 0 en bordes/esquinas, mas abajo se congela antes
        patch = self._fbm(rng, 95 * k, 5, 0.58)    # parches irregulares
        fern_s = cv2.GaussianBlur(fern, (0, 0), 1.6 * max(k, 0.7))
        field = 0.30 * d + 0.15 * patch + 0.55 * (1.0 - np.clip(fern_s * 2.0, 0, 1))
        flat = field.ravel()
        order = np.argsort(flat)
        U = np.empty(flat.size, dtype=np.float32)
        U[order] = np.linspace(0.0, 1.0, flat.size, dtype=np.float32)
        self.frost_U = U.reshape(VH, VW)

    def update_frost_cache(self, st):
        """Recalcula la opacidad de la escarcha solo cuando cambia el slider."""
        on = bool(st.snow_frost_on) and st.snow_frost > 0
        key = (on, int(st.snow_frost))
        if key == self._frost_key:
            return
        self._frost_key = key
        if not on:
            self.frost_alpha = np.zeros((VH, VW), dtype=np.float32)
            return
        a = st.snow_frost / 100.0
        w = 0.012 + 0.05 * a                       # suavidad del frente de hielo
        th = a * (1.0 + w)                         # a=1 -> todo cubierto
        c = np.clip((th - self.frost_U) / w, 0, 1)
        c = c * c * (3.0 - 2.0 * c)
        dens = 0.55 + 0.45 * a ** 0.7              # poca escarcha = mas fina / transparente
        self.frost_alpha = (c * self.frost_tex * dens).astype(np.float32)

    # ----- neblina (fog) afuera: manchas suaves que derivan despacio -----
    def init_mist(self):
        rng = np.random.default_rng()
        self.mist_w = max(16, VW // 8)
        self.mist_h = max(9, VH // 8)
        n = 8
        cyc_x = rng.uniform(1.2, 5.0, n)                       # ciclos a lo ancho
        cyc_y = rng.uniform(0.4, 2.2, n) * rng.choice([-1.0, 1.0], n)
        kxy = np.stack([cyc_x, cyc_y], axis=1) * 2.0 * np.pi
        amp = (1.0 / (0.6 + cyc_x * 0.35)).astype(np.float32)  # lo grande pesa mas
        rate = cyc_x * 2.0 * np.pi * 0.0007 * rng.uniform(0.6, 1.4, n) + rng.uniform(-0.004, 0.004, n)
        mx = np.linspace(0.0, 1.0, self.mist_w, dtype=np.float32)
        my = np.linspace(0.0, VH / float(VW), self.mist_h, dtype=np.float32)
        MX, MY = np.meshgrid(mx, my)
        phi = rng.uniform(0, 2 * np.pi, n)
        self.mist_ph0 = (MX[..., None] * kxy[:, 0] + MY[..., None] * kxy[:, 1]
                         + phi).astype(np.float32)
        self.mist_rate = rate.astype(np.float32)
        self.mist_amp = amp
        self.mist_amp_sum = float(amp.sum())
        # para mover la neblina con el viento (desplazamiento / acercamiento)
        self.mist_kx = kxy[:, 0].astype(np.float32)
        self.mist_ky = kxy[:, 1].astype(np.float32)
        self.mist_kx64 = kxy[:, 0].astype(np.float64)
        self.mist_ky64 = kxy[:, 1].astype(np.float64)
        self.mist_MX = MX.astype(np.float32)
        self.mist_MY = MY.astype(np.float32)
        self.mist_phi = phi.astype(np.float32)
        self.mist_t = 0.0          # tiempo propio de la neblina (se congela si no se mueve)
        self.mist_ox = 0.0         # desplazamiento acumulado por el viento
        self.mist_oy = 0.0
        self.mist_q = 0.0          # fase del acercamiento / alejamiento

    def mist_alpha(self, st):
        """Opacidad (VH x VW) de la neblina en este frame."""
        t = float(self.mist_t)
        shift = np.mod(self.mist_kx64 * self.mist_ox + self.mist_ky64 * self.mist_oy,
                       2.0 * np.pi).astype(np.float32)
        mode = st.snow_wind_dir if (st.snow_fog_move and st.snow_wind_on) else "none"
        if mode in ("toward", "away"):
            # el viento sopla hacia / desde la ventana: la neblina se acerca o se aleja
            m = 0.0
            cym = 0.5 * VH / float(VW)
            for off in (0.0, 0.5):
                q = (self.mist_q + off) % 1.0
                zoom = 1.0 + 0.7 * q if mode == "toward" else 1.7 - 0.7 * q
                MXz = 0.5 + (self.mist_MX - 0.5) / zoom
                MYz = cym + (self.mist_MY - cym) / zoom
                ph = (MXz[..., None] * self.mist_kx + MYz[..., None] * self.mist_ky
                      + self.mist_phi - self.mist_rate * t - shift)
                m = m + (np.sin(ph) * self.mist_amp).sum(axis=2) * (math.sin(math.pi * q) ** 2)
        else:
            m = (np.sin(self.mist_ph0 - self.mist_rate * t - shift) * self.mist_amp).sum(axis=2)
        m /= self.mist_amp_sum                                  # -1..1
        d = np.clip(m * 0.5 + 0.55, 0, 1)
        d = d * d * (3.0 - 2.0 * d)
        amt = float(np.clip(st.snow_fog, 0, 100)) / 100.0
        a = (amt * (0.30 + 0.70 * d) * 0.80).astype(np.float32)
        a = cv2.resize(a, (VW, VH), interpolation=cv2.INTER_CUBIC)
        return np.clip(a, 0, 0.92)

    def update_mist(self, st):
        """Mueve la neblina: deriva despacio y, si hay viento, responde a su direccion
        e intensidad (derecha / izquierda / cambiante = desplazamiento; hacia la ventana /
        alejandose = acercamiento)."""
        if not (st.snow_fog_on and st.snow_fog_move):
            return                                   # sin movimiento: se queda como esta
        self.mist_t += 1.0
        if not st.snow_wind_on:
            return
        mode = st.snow_wind_dir
        inten = float(np.clip(st.snow_wind_strength, 1, 10)) / 10.0
        t = float(self.frame)
        gust = (0.70 + 0.25 * math.sin(t * 0.021) + 0.12 * math.sin(t * 0.057 + 1.3)
                + 0.08 * math.sin(t * 0.13))
        if mode in ("right", "left"):
            self.mist_ox += (1.0 if mode == "right" else -1.0) * inten * gust * 0.0035
        elif mode == "variable":
            if not (st.snow_on and st.snow_amount > 0):
                self.snow.variable_wind_tick()       # el viento cambiante sigue avanzando
            self.mist_ox += self.snow.wv * inten * 0.0035
        else:                                        # toward / away
            self.mist_q = (self.mist_q + 0.004 * inten * gust) % 1.0

    # ----- Nieve: clic / arrastrar = soplar los copos y limpiar la escarcha con el dedo -----
    def update_touch_snow(self, st):
        left_down = bool(win32api.GetAsyncKeyState(win32con.VK_LBUTTON) & 0x8000)
        if not st.snow_touch:
            self.t_down, self.t_active, self.t_last = left_down, False, None
        else:
            if left_down:
                cx, cy = win32api.GetCursorPos()
                gx, gy = cx - mon["left"], cy - mon["top"]
                fresh = not self.t_down                       # recien presionado
                if fresh:
                    self.t_active = not self._cursor_over_panel(cx, cy)
                    self.t_last = None
                if self.t_active and 0 <= gx < W and 0 <= gy < H:
                    self._touch_apply(st, self.t_last, (gx, gy), fresh)
                    self.t_last = (gx, gy)
                else:
                    self.t_last = None
            else:
                self.t_last, self.t_active = None, False
            self.t_down = left_down

        # la escarcha limpiada vuelve a formarse poco a poco
        if self.fclear_any:
            ticks = max(30.0, st.snow_touch_regrow * 1000.0 / max(10, st.interval_ms))
            self.fclear_acc += 255.0 / ticks
            n = int(self.fclear_acc)
            if n:
                self.fclear_acc -= n
                self.frost_clear = cv2.subtract(self.frost_clear, n)
            if self.frame % 15 == 0 and not self.frost_clear.any():
                self.fclear_any = False

    def _touch_apply(self, st, p0, p1, fresh):
        R = float(st.snow_touch_size)
        if p0 is None:
            p0 = p1
        # 1) escarcha: se limpia como al pasar el dedo (nucleo total + borde suave)
        if st.snow_frost_on and st.snow_frost > 0:
            s16 = GS * 16.0
            a = (int(round(p0[0] * s16)), int(round(p0[1] * s16)))
            b = (int(round(p1[0] * s16)), int(round(p1[1] * s16)))
            th_core = max(1, int(round(2 * R * GS)))
            th_soft = max(2, int(round(2 * R * GS * 1.35)))
            r_core = int(round(R * s16))
            r_soft = int(round(R * 1.35 * s16))
            tmp = np.zeros((VH, VW), dtype=np.uint8)
            cv2.line(tmp, a, b, 140, th_soft, cv2.LINE_AA, 4)
            cv2.circle(tmp, b, r_soft, 140, -1, cv2.LINE_AA, 4)
            cv2.line(tmp, a, b, 255, th_core, cv2.LINE_AA, 4)
            cv2.circle(tmp, b, r_core, 255, -1, cv2.LINE_AA, 4)
            np.maximum(self.frost_clear, tmp, out=self.frost_clear)
            self.fclear_any = True
        # 2) copos: soplo de aire en esa zona (un golpe fuerte al hacer clic, luego sigue al arrastre)
        if st.snow_on and st.snow_amount > 0:
            n = int(np.clip(st.snow_amount, 0, 100) * SNOW_MAX / 100)
            if n > 0:
                kx = self.snow.kx
                self.snow.push(n, p1[0] * GS, p1[1] * GS, R * 3.2 * GS,
                               (p1[0] - p0[0]) * GS, (p1[1] - p0[1]) * GS,
                               (16.0 if fresh else 1.4) * kx)

    def update_snow(self, st):
        if not st.snow_on:
            self.snow_img = None
            return
        n = int(np.clip(st.snow_amount, 0, 100) * SNOW_MAX / 100)
        if n <= 0:
            self.snow_img = None
            return
        self.snow.step(st, n)
        self.snow_img = self.snow.draw(st, n)

    def tick_nieve(self, st):
        if self.glass_dirty or st.glass_blur != self._last_blur:
            self.rebuild_glass_bg()
        self.update_foreground()
        self.update_frost_cache(st)
        self.update_mist(st)
        self.update_touch_snow(st)
        self.update_snow(st)

    def render_nieve(self):
        st = self.state
        frost_on = bool(st.snow_frost_on) and st.snow_frost > 0
        a_f = self.frost_alpha
        if frost_on and self.fclear_any:
            a_f = a_f * (1.0 - self.frost_clear.astype(np.float32) * (1.0 / 255.0))
        C = None
        if frost_on:
            # escarcha: escritorio desenfocado + blanco azulado, con brillo en las venas
            C = (self.fog_img * 0.5 + np.array([255.0, 245.0, 235.0], dtype=np.float32) * 0.5)
            C *= self.frost_mod[..., None]
            np.clip(C, 0, 255, out=C)

        al = None
        if st.snow_on and self.snow_img is not None:
            al = self.snow_img.astype(np.float32) * (1.0 / 255.0)
            if frost_on:
                al *= (1.0 - 0.5 * a_f)          # detras de la escarcha se ven mas tenues

        FL = np.array([250.0, 236.0, 222.0], dtype=np.float32)   # copos blanco azulado (BGR)
        if al is None:
            a_out = a_f
            col = C if C is not None else np.broadcast_to(FL, (VH, VW, 3))
        else:
            a_out = a_f + al * (1.0 - a_f)
            if C is None:
                col = np.broadcast_to(FL, (VH, VW, 3))
            else:
                wf = (a_f / np.maximum(a_out, 1e-4))[..., None]
                col = C * wf + FL * (1.0 - wf)

        # neblina (fog) afuera: capa de fondo, debajo de los copos y de la escarcha
        if st.snow_fog_on and st.snow_fog > 0:
            a_g = self.mist_alpha(st)
            FC = np.array([236.0, 230.0, 222.0], dtype=np.float32)   # blanco grisaceo (BGR)
            a_new = a_out + a_g * (1.0 - a_out)
            wt = (a_out / np.maximum(a_new, 1e-4))[..., None]
            col = col * wt + FC * (1.0 - wt)
            a_out = a_new

        half = np.empty((VH, VW, 4), dtype=np.uint8)
        half[:, :, :3] = np.clip(col, 0, 255).astype(np.uint8)
        half[:, :, 3] = (np.clip(a_out, 0, 1) * 255).astype(np.uint8)
        out = cv2.resize(half, (W, H), interpolation=cv2.INTER_LINEAR)

        # ventana activa "dentro": se ve clara (igual que en el estilo Ventana)
        if st.fg_inside and self.fg_rect is not None:
            x0, y0, x1, y1 = self.fg_rect
            out[y0:y1, x0:x1, 3] = 0
        return out

    # ---------------------------------------------------------------
    # ESTILO "ESTRELLAS" — cielo nocturno sobre el escritorio
    # ---------------------------------------------------------------
    def tick_estrellas(self, st):
        self.update_foreground()
        self.sky_half = self.sky.render(st)

    def tick_luciernagas(self, st):
        self.update_foreground()
        self.fx_half = self.fly.render(st)

    def tick_matrix(self, st):
        self.update_foreground()
        self.fx_half = self.matrix.render(st)

    def tick_nubes(self, st):
        self.update_foreground()
        self.fx_half = self.clouds.render(st)

    def tick_cascada(self, st):
        self.update_foreground()
        self.fx_half = self.waterfall.render(st)

    def render_fx(self):
        """Luciernagas / Matrix / Nubes / Cascada: imagen BGRA a resolucion de vidrio -> pantalla."""
        st = self.state
        if self.fx_half is None:
            return np.zeros((H, W, 4), dtype=np.uint8)
        out = cv2.resize(self.fx_half, (W, H), interpolation=cv2.INTER_LINEAR)
        if st.fg_inside and self.fg_rect is not None:     # ventana activa "dentro": clara
            x0, y0, x1, y1 = self.fg_rect
            out[y0:y1, x0:x1, 3] = 0
        return out

    def render_estrellas(self):
        st = self.state
        if self.sky_half is None:
            return np.zeros((H, W, 4), dtype=np.uint8)
        out = cv2.resize(self.sky_half, (W, H), interpolation=cv2.INTER_LINEAR)
        # ventana activa "dentro": se ve clara (igual que en Ventana y Nieve)
        if st.fg_inside and self.fg_rect is not None:
            x0, y0, x1, y1 = self.fg_rect
            out[y0:y1, x0:x1, 3] = 0
        return out

    # ---------------------------------------------------------------
    def tick(self):
        self.frame += 1
        st = self.state

        if self.timer.interval() != st.interval_ms:
            self.timer.setInterval(st.interval_ms)

        if st.style == "pantalla":
            self.tick_pantalla(st)
        elif st.style == "nieve":
            self.tick_nieve(st)
        elif st.style == "estrellas":
            self.tick_estrellas(st)
        elif st.style == "luciernagas":
            self.tick_luciernagas(st)
        elif st.style == "matrix":
            self.tick_matrix(st)
        elif st.style == "nubes":
            self.tick_nubes(st)
        elif st.style == "cascada":
            self.tick_cascada(st)
        else:
            self.tick_ventana(st)

        if st.wiper_single and st.wiper_active and st.style in (
                "nieve", "estrellas", "luciernagas", "matrix", "nubes", "cascada"):
            self.advance_wiper()       # sin efecto visible aqui, pero termina la pasada y se apaga
        if self.hotkey_poll:
            self.poll_wiper_hotkey()
        if self.hk_poll_extra:
            self.poll_extra_hotkeys()

        every = st.recapture_every if st.style == "pantalla" else st.recapture_every_glass
        if self.frame % every == 0 and st.style not in (
                "estrellas", "luciernagas", "matrix", "nubes", "cascada"):
            self.bg_full = self.capture_screen()
            self.bg = cv2.resize(self.bg_full, (RW, RH), interpolation=cv2.INTER_AREA)
            if st.style in ("ventana", "nieve"):
                self.rebuild_glass_bg()
            else:
                self.glass_dirty = True

        self.update()

    def paintEvent(self, event):
        if self.state.style == "pantalla":
            bgra = self.render_pantalla()
            w, h = RW, RH
        elif self.state.style == "nieve":
            bgra = self.render_nieve()
            w, h = W, H
        elif self.state.style == "estrellas":
            bgra = self.render_estrellas()
            w, h = W, H
        elif self.state.style in ("luciernagas", "matrix", "nubes", "cascada"):
            bgra = self.render_fx()
            w, h = W, H
        else:
            bgra = self.render_ventana()
            w, h = W, H
        qimg = QtGui.QImage(bgra.data, w, h, w * 4, QtGui.QImage.Format_ARGB32)
        painter = QtGui.QPainter(self)
        painter.drawImage(self.rect(), qimg)
        painter.end()


# =====================================================================
# Panel de control
# =====================================================================
# =====================================================================
# Atajo global Ctrl + Ñ (funciona aunque otra ventana tenga el foco)
# =====================================================================
def ntilde_vk():
    """Tecla virtual de la 'ñ' en el teclado activo (VK_OEM_3 = 0xC0 en es-LA / es-ES)."""
    try:
        r = win32api.VkKeyScan("ñ")
        if r != -1 and (r & 0xFF) != 0xFF:
            return r & 0xFF
    except Exception:
        pass
    return 0xC0


class HotkeyFilter(QtCore.QAbstractNativeEventFilter):
    WM_HOTKEY = 0x0312

    def __init__(self, callback, hk_id):
        super().__init__()
        self.callback = callback
        self.hk_id = hk_id

    def nativeEventFilter(self, event_type, message):
        try:
            if bytes(event_type) in (b"windows_generic_MSG", b"windows_dispatcher_MSG"):
                msg = wintypes.MSG.from_address(int(message))
                if msg.message == self.WM_HOTKEY and msg.wParam == self.hk_id:
                    self.callback()
                    return True, 0
        except Exception:
            pass
        return False, 0


def register_wiper_hotkey(app, overlay):
    """Registra Ctrl + Ñ. Si Windows no lo permite (otro programa lo usa), el overlay
    lo sondea en cada frame como respaldo."""
    MOD_CONTROL, MOD_NOREPEAT, HK_ID = 0x0002, 0x4000, 0x5A11
    try:
        ok = ctypes.windll.user32.RegisterHotKey(None, HK_ID, MOD_CONTROL | MOD_NOREPEAT,
                                                 ntilde_vk())
    except Exception:
        ok = 0
    if not ok:
        overlay.hotkey_poll = True
        return None
    filt = HotkeyFilter(overlay.trigger_wiper_pass, HK_ID)
    app.installNativeEventFilter(filt)
    atexit.register(lambda: ctypes.windll.user32.UnregisterHotKey(None, HK_ID))
    return filt


class ControlPanel(QtWidgets.QWidget):
    def __init__(self, state: RainState):
        super().__init__()
        self.state = state
        self.setWindowTitle("Lluvia - Panel de control")
        self.setWindowFlag(QtCore.Qt.WindowStaysOnTopHint, True)  # siempre nitido sobre la lluvia
        self.setMinimumWidth(460)

        root = QtWidgets.QVBoxLayout(self)
        tabs = QtWidgets.QTabWidget()
        root.addWidget(tabs)

        # ============ pestaña General ============
        general = QtWidgets.QWidget()
        g = QtWidgets.QVBoxLayout(general)

        g.addWidget(QtWidgets.QLabel("Estilo de lluvia"))
        self.style_combo = QtWidgets.QComboBox()
        self.style_combo.addItems(["Pantalla (agua)", "Ventana (vidrio)", "Nieve (copos y escarcha)",
                                    "Estrellas (lluvia de estrellas)", "Luciérnagas",
                                    "Matrix (código cayendo)", "Nubes", "Cascada (agua cayendo)"])
        self.style_combo.setCurrentIndex(self._idx(SETTING_CHOICES["style"], state.style))
        self.style_combo.currentIndexChanged.connect(self.on_style)
        g.addWidget(self.style_combo)

        self.speed_slider = self._slider(g, "Velocidad (más a la derecha = más rápido)",
                                         10, 100, 110 - state.interval_ms, self.on_speed)
        self.intensity_slider = self._slider(g, "Intensidad de lluvia",
                                             0, 30, state.intensity, self.on_intensity)
        self.strength_slider = self._slider(g, "Fuerza / gravedad",
                                            1, 20, int(state.strength), self.on_strength)
        self.size_slider = self._slider(g, "Tamaño de las gotas",
                                        1, 30, state.drop_size, self.on_drop_size)
        self.persistence_slider = self._slider(g, "Expansión de ondas / duración del rastro",
                                               900, 999, int(state.wave_persistence * 1000),
                                               self.on_persistence)

        self.wiper_btn = QtWidgets.QPushButton("Activar limpiaparabrisas")
        self.wiper_btn.setCheckable(True)
        self.wiper_btn.toggled.connect(self.on_wiper)
        g.addWidget(self.wiper_btn)
        g.addWidget(QtWidgets.QLabel("Atajo: Ctrl + Ñ → una pasada del limpiaparabrisas "
                                     "(otra vez Ctrl + Ñ la detiene)"))

        self.wiper_speed_slider = self._slider(g, "Velocidad del limpiaparabrisas",
                                               1, 30, int(state.wiper_speed * 100),
                                               self.on_wiper_speed)
        g.addStretch(1)
        tabs.addTab(general, "General")

        # ============ pestaña Ventana ============
        glass = QtWidgets.QWidget()
        v = QtWidgets.QVBoxLayout(glass)

        v.addWidget(QtWidgets.QLabel("Ventana activa (en primer plano)"))
        self.fg_combo = QtWidgets.QComboBox()
        self.fg_combo.addItems(["Dentro (se ve clara)", "Fuera (empañada, bajo la lluvia)",
                                "Dinámica (maximizada = fuera · no maximizada = dentro)"])
        self.fg_combo.setCurrentIndex(2 if state.fg_dynamic else (0 if state.fg_inside else 1))
        self.fg_combo.currentIndexChanged.connect(self.on_fg)
        v.addWidget(self.fg_combo)

        self.blur_slider = self._slider(v, "Desenfoque del vidrio",
                                        0, 40, state.glass_blur, self.on_blur)
        self.fog_slider = self._slider(v, "Empañado del vidrio",
                                       0, 100, int(state.fog * 100), self.on_fog)
        self.sticky_slider = self._slider(v, "Gotas pegajosas (más alto = menos resbalan)",
                                          0, 100, state.stickiness, self.on_sticky)
        self.defog_slider = self._slider(v, "Desempañado del limpiaparabrisas (por pasada)",
                                         0, 100, int(state.wiper_defog * 100), self.on_defog)

        rain_box = QtWidgets.QGroupBox("Lluvia cayendo afuera (detrás del vidrio)")
        rb = QtWidgets.QVBoxLayout(rain_box)
        self.rain_chk = QtWidgets.QCheckBox("Ver la lluvia cayendo afuera de la ventana")
        self.rain_chk.setChecked(state.rain_outside)
        self.rain_chk.toggled.connect(self.on_rain_outside)
        rb.addWidget(self.rain_chk)
        self.rain_amount_slider = self._slider(rb, "Cantidad de lluvia afuera",
                                               5, 100, state.rain_amount,
                                               self.on_rain_amount)
        rb.addWidget(QtWidgets.QLabel("Dirección de la lluvia"))
        self.rain_dir_combo = QtWidgets.QComboBox()
        self.rain_dir_combo.addItems([
            "Cae hacia la derecha",
            "Cae hacia la izquierda",
            "Hacia la pantalla (viene hacia ti)",
            "En dirección opuesta (se aleja de la pantalla)",
            "Cambiante (a veces a un lado, a veces al otro)",
        ])
        self.rain_dir_combo.setCurrentIndex(self._idx(SETTING_CHOICES["rain_dir"], state.rain_dir))
        self.rain_dir_combo.currentIndexChanged.connect(self.on_rain_dir)
        rb.addWidget(self.rain_dir_combo)
        self.rain_incl_slider = self._slider(
            rb, "Inclinación de la lluvia (derecha / izquierda / cambiante)",
            0, 100, state.rain_incl, self.on_rain_incl)
        v.addWidget(rain_box)

        finger_box = QtWidgets.QGroupBox("Desempañar con el mouse (clic / arrastrar)")
        fb = QtWidgets.QVBoxLayout(finger_box)
        self.finger_chk = QtWidgets.QCheckBox("Desempañar al hacer clic o arrastrar")
        self.finger_chk.setChecked(state.finger_defog)
        self.finger_chk.toggled.connect(self.on_finger)
        fb.addWidget(self.finger_chk)
        self.finger_size_slider = self._slider(fb, "Tamaño del dedo",
                                               6, 80, state.finger_size,
                                               self.on_finger_size)
        self.finger_regrow_slider = self._slider(fb, "Tiempo en volver a empañarse (segundos)",
                                                 5, 180, state.finger_regrow,
                                                 self.on_finger_regrow)
        v.addWidget(finger_box)

        box = QtWidgets.QGroupBox("Truenos (solo estilo Ventana)")
        tb = QtWidgets.QVBoxLayout(box)
        self.thunder_chk = QtWidgets.QCheckBox("Activar truenos")
        self.thunder_chk.setChecked(state.thunder_on)
        self.thunder_chk.toggled.connect(self.on_thunder)
        tb.addWidget(self.thunder_chk)
        self.thunder_freq_slider = self._slider(tb, "Cuántos truenos (frecuencia)",
                                                1, 10, state.thunder_freq, self.on_thunder_freq)
        self.thunder_int_slider = self._slider(tb, "Intensidad (brillo y volumen)",
                                               1, 10, state.thunder_intensity,
                                               self.on_thunder_int)
        self.sound_chk = QtWidgets.QCheckBox("Con sonido")
        self.sound_chk.setChecked(state.thunder_sound)
        self.sound_chk.toggled.connect(self.on_sound)
        tb.addWidget(self.sound_chk)
        test_btn = QtWidgets.QPushButton("Probar un trueno")
        test_btn.clicked.connect(self.on_test_thunder)
        tb.addWidget(test_btn)
        hint = QtWidgets.QLabel("Atajo: Ctrl + Mayús + Ñ → lanza un trueno")
        hint.setWordWrap(True)
        tb.addWidget(hint)
        v.addWidget(box)
        v.addStretch(1)
        scroll = QtWidgets.QScrollArea()
        scroll.setWidgetResizable(True)
        scroll.setFrameShape(QtWidgets.QFrame.NoFrame)
        scroll.setWidget(glass)
        tabs.addTab(scroll, "Ventana")

        # ============ pestaña Nieve ============
        snow = QtWidgets.QWidget()
        n = QtWidgets.QVBoxLayout(snow)

        self.snow_chk = QtWidgets.QCheckBox("Ver nevar (copos al fondo)")
        self.snow_chk.setChecked(state.snow_on)
        self.snow_chk.toggled.connect(self.on_snow_on)
        n.addWidget(self.snow_chk)
        self.snow_amount_slider = self._slider(n, "Cantidad de copos",
                                               0, 100, state.snow_amount,
                                               self.on_snow_amount)
        self.snow_size_slider = self._slider(n, "Tamaño de los copos",
                                             1, 12, state.snow_size, self.on_snow_size)

        frost_box = QtWidgets.QGroupBox("Escarcha (frost) en la ventana")
        fx = QtWidgets.QVBoxLayout(frost_box)
        self.frost_chk = QtWidgets.QCheckBox("Activar escarcha")
        self.frost_chk.setChecked(state.snow_frost_on)
        self.frost_chk.toggled.connect(self.on_frost_on)
        fx.addWidget(self.frost_chk)
        self.frost_slider = self._slider(fx, "Cantidad de escarcha",
                                         0, 100, state.snow_frost, self.on_frost)
        n.addWidget(frost_box)

        fog_box = QtWidgets.QGroupBox("Neblina (fog) afuera")
        fgx = QtWidgets.QVBoxLayout(fog_box)
        self.snow_fog_chk = QtWidgets.QCheckBox("Activar neblina")
        self.snow_fog_chk.setChecked(state.snow_fog_on)
        self.snow_fog_chk.toggled.connect(self.on_snow_fog_on)
        fgx.addWidget(self.snow_fog_chk)
        self.snow_fog_slider = self._slider(fgx, "Cantidad de neblina",
                                            0, 100, state.snow_fog, self.on_snow_fog)
        self.snow_fog_move_chk = QtWidgets.QCheckBox(
            "Mover la neblina (responde al viento y a su dirección)")
        self.snow_fog_move_chk.setChecked(state.snow_fog_move)
        self.snow_fog_move_chk.toggled.connect(self.on_snow_fog_move)
        fgx.addWidget(self.snow_fog_move_chk)
        n.addWidget(fog_box)

        wind_box = QtWidgets.QGroupBox("Viento")
        wx = QtWidgets.QVBoxLayout(wind_box)
        self.wind_chk = QtWidgets.QCheckBox("Activar viento")
        self.wind_chk.setChecked(state.snow_wind_on)
        self.wind_chk.toggled.connect(self.on_wind_on)
        wx.addWidget(self.wind_chk)
        self.wind_slider = self._slider(wx, "Intensidad del viento",
                                        1, 10, state.snow_wind_strength, self.on_wind_strength)
        wx.addWidget(QtWidgets.QLabel("Dirección del viento"))
        self.wind_combo = QtWidgets.QComboBox()
        self.wind_combo.addItems([
            "Hacia la derecha",
            "Hacia la izquierda",
            "Hacia la ventana (los copos vienen hacia ti)",
            "En dirección contraria (los copos se alejan de la ventana)",
            "Cambiante (a veces a un lado, a veces al otro, con intensidad variable)",
        ])
        self.wind_combo.setCurrentIndex(
            self._idx(SETTING_CHOICES["snow_wind_dir"], state.snow_wind_dir))
        self.wind_combo.currentIndexChanged.connect(self.on_wind_dir)
        wx.addWidget(self.wind_combo)
        n.addWidget(wind_box)

        touch_box = QtWidgets.QGroupBox("Mouse: soplar los copos y limpiar la escarcha (clic / arrastrar)")
        tbx = QtWidgets.QVBoxLayout(touch_box)
        self.snow_touch_chk = QtWidgets.QCheckBox("Activar al hacer clic o arrastrar")
        self.snow_touch_chk.setChecked(state.snow_touch)
        self.snow_touch_chk.toggled.connect(self.on_snow_touch)
        tbx.addWidget(self.snow_touch_chk)
        self.snow_touch_size_slider = self._slider(tbx, "Tamaño del dedo / soplo",
                                                   6, 80, state.snow_touch_size,
                                                   self.on_snow_touch_size)
        self.snow_touch_regrow_slider = self._slider(
            tbx, "Tiempo en volver a formarse la escarcha (segundos)",
            5, 180, state.snow_touch_regrow, self.on_snow_touch_regrow)
        n.addWidget(touch_box)
        n.addStretch(1)
        snow_scroll = QtWidgets.QScrollArea()
        snow_scroll.setWidgetResizable(True)
        snow_scroll.setFrameShape(QtWidgets.QFrame.NoFrame)
        snow_scroll.setWidget(snow)
        tabs.addTab(snow_scroll, "Nieve")

        # ============ pestaña Estrellas ============
        sky = QtWidgets.QWidget()
        k = QtWidgets.QVBoxLayout(sky)
        hint2 = QtWidgets.QLabel("Atajo: Alt + Ñ → aparece una estrella fugaz o un cometa")
        hint2.setWordWrap(True)
        k.addWidget(hint2)

        meteor_box = QtWidgets.QGroupBox("Estrellas fugaces (lluvia de estrellas)")
        mb = QtWidgets.QVBoxLayout(meteor_box)
        self.meteor_slider = self._slider(
            mb, "Intensidad (0 = ninguna · bajo = unas pocas por minuto · alto = lluvia)",
            0, 100, state.sky_meteors, self.on_sky_meteors)
        self._color_pair(mb, "meteor")
        k.addWidget(meteor_box)

        star_box = QtWidgets.QGroupBox("Estrellas fijas")
        stb = QtWidgets.QVBoxLayout(star_box)
        self.sky_stars_slider = self._slider(
            stb, "Cantidad (100 = el máximo de antes · hasta 400 = 4 veces más) · "
                 "el 1% de ellas titila", 0, 400, state.sky_stars, self.on_sky_stars)
        self._color_pair(stb, "stars", natural=True)
        k.addWidget(star_box)

        comet_box = QtWidgets.QGroupBox("Cometas")
        cb = QtWidgets.QVBoxLayout(comet_box)
        self.comet_chk = QtWidgets.QCheckBox("Activar cometas")
        self.comet_chk.setChecked(state.sky_comets_on)
        self.comet_chk.toggled.connect(self.on_sky_comets_on)
        cb.addWidget(self.comet_chk)
        self.comet_slider = self._slider(cb, "Cantidad de cometas", 1, 5,
                                         state.sky_comets_n, self.on_sky_comets_n)
        self._color_pair(cb, "comet")
        k.addWidget(comet_box)

        aur_box = QtWidgets.QGroupBox("Auroras boreales")
        ab = QtWidgets.QVBoxLayout(aur_box)
        self.aurora_chk = QtWidgets.QCheckBox("Activar auroras")
        self.aurora_chk.setChecked(state.sky_aurora_on)
        self.aurora_chk.toggled.connect(self.on_sky_aurora_on)
        ab.addWidget(self.aurora_chk)
        self.aurora_slider = self._slider(ab, "Cantidad de auroras", 1, 4,
                                          state.sky_aurora_n, self.on_sky_aurora_n)
        self._color_pair(ab, "aurora")
        k.addWidget(aur_box)

        neb_box = QtWidgets.QGroupBox("Nebulosa")
        nb = QtWidgets.QVBoxLayout(neb_box)
        self.nebula_chk = QtWidgets.QCheckBox("Activar nebulosa")
        self.nebula_chk.setChecked(state.sky_nebula_on)
        self.nebula_chk.toggled.connect(self.on_sky_nebula_on)
        nb.addWidget(self.nebula_chk)
        self._color_pair(nb, "nebula")
        k.addWidget(neb_box)

        dir_box = QtWidgets.QGroupBox("Dirección")
        db = QtWidgets.QVBoxLayout(dir_box)
        self.sky_dir_combo = QtWidgets.QComboBox()
        self.sky_dir_combo.addItems([
            "Hacia la derecha",
            "Hacia la izquierda",
            "Hacia la ventana (vienen hacia ti)",
            "En dirección contraria (se alejan de la ventana)",
        ])
        self.sky_dir_combo.setCurrentIndex(self._idx(SETTING_CHOICES["sky_dir"], state.sky_dir))
        self.sky_dir_combo.currentIndexChanged.connect(self.on_sky_dir)
        db.addWidget(self.sky_dir_combo)
        k.addWidget(dir_box)
        k.addStretch(1)
        sky_scroll = QtWidgets.QScrollArea()
        sky_scroll.setWidgetResizable(True)
        sky_scroll.setFrameShape(QtWidgets.QFrame.NoFrame)
        sky_scroll.setWidget(sky)
        tabs.addTab(sky_scroll, "Estrellas")

        # ============ pestaña Luciérnagas ============
        fw = QtWidgets.QWidget()
        f = QtWidgets.QVBoxLayout(fw)
        self.fly_amount_slider = self._slider(f, "Cantidad de luciérnagas", 1, FIREFLY_MAX,
                                              state.fly_amount, self.on_fly_amount)
        self.fly_size_slider = self._slider(f, "Tamaño de las luces", 1, 12,
                                            state.fly_size, self.on_fly_size)
        fc_box = QtWidgets.QGroupBox("Colores (elige uno o varios, hasta los 10 a la vez)")
        fcg = QtWidgets.QGridLayout(fc_box)
        self.fly_chks = {}
        for i, nm in enumerate(FIREFLY_COLOR_NAMES):
            chk = QtWidgets.QCheckBox(nm.capitalize())
            chk.setChecked(nm in state.fly_colors)
            chk.toggled.connect(lambda checked, name=nm: self.on_fly_color(name, checked))
            fcg.addWidget(chk, i // 2, i % 2)
            self.fly_chks[nm] = chk
        f.addWidget(fc_box)
        f.addStretch(1)
        fly_scroll = QtWidgets.QScrollArea()
        fly_scroll.setWidgetResizable(True)
        fly_scroll.setFrameShape(QtWidgets.QFrame.NoFrame)
        fly_scroll.setWidget(fw)
        tabs.addTab(fly_scroll, "Luciérnagas")

        # ============ pestaña Matrix ============
        mw = QtWidgets.QWidget()
        mxl = QtWidgets.QVBoxLayout(mw)
        mxl.addWidget(QtWidgets.QLabel("Color del código"))
        self.mx_color_combo = QtWidgets.QComboBox()
        self.mx_color_combo.addItems(["Verde", "Azulado", "Morado", "Mixto (verde + azulado + morado)"])
        self.mx_color_combo.setCurrentIndex(("verde", "azulado", "morado", "mixto").index(state.mx_color))
        self.mx_color_combo.currentIndexChanged.connect(self.on_mx_color)
        mxl.addWidget(self.mx_color_combo)
        mxl.addWidget(QtWidgets.QLabel("Caracteres"))
        self.mx_chars_combo = QtWidgets.QComboBox()
        self.mx_chars_combo.addItems(["Katakana y números", "Solo números", "Binario (0 y 1)"])
        self.mx_chars_combo.setCurrentIndex(("katakana", "numeros", "binario").index(state.mx_chars))
        self.mx_chars_combo.currentIndexChanged.connect(self.on_mx_chars)
        mxl.addWidget(self.mx_chars_combo)
        self.mx_amount_slider = self._slider(mxl, "Cantidad de código", 5, 100,
                                             state.mx_amount, self.on_mx_amount)
        self.mx_opacity_slider = self._slider(mxl, "Opacidad", 5, 100,
                                              state.mx_opacity, self.on_mx_opacity)
        self.mx_depth_slider = self._slider(
            mxl, "Profundidad (capas de código una sobre otra)", 1, 4,
            state.mx_depth, self.on_mx_depth)
        self.mx_size_slider = self._slider(
            mxl, "Tamaño del código (más a la derecha = más grande)", 1, 10,
            state.mx_size, self.on_mx_size)
        self.mx_speed_slider = self._slider(
            mxl, "Velocidad de caída (más a la derecha = más rápido)", 1, 10,
            state.mx_speed, self.on_mx_speed)
        self.mx_speed_var_chk = QtWidgets.QCheckBox(
            "Velocidad variable (a veces cae rápido y a veces despacio)")
        self.mx_speed_var_chk.setChecked(state.mx_speed_var)
        self.mx_speed_var_chk.toggled.connect(self.on_mx_speed_var)
        mxl.addWidget(self.mx_speed_var_chk)

        msg_box = QtWidgets.QGroupBox("Mensajes escondidos (caen al azar entre el código)")
        mgx = QtWidgets.QVBoxLayout(msg_box)
        mlab = QtWidgets.QLabel("Escribe hasta 5 mensajes (máx. %d letras cada uno). "
                                "Los campos vacíos no se usan." % MX_MSG_MAX)
        mlab.setWordWrap(True)
        mgx.addWidget(mlab)
        mhk = QtWidgets.QLabel("Atajo: Ctrl + Ñ hace caer uno de estos mensajes ahora mismo "
                               "(se puede repetir).")
        mhk.setWordWrap(True)
        mgx.addWidget(mhk)
        self.mx_msg_edits = []
        for i in range(5):
            ed = QtWidgets.QLineEdit()
            ed.setMaxLength(MX_MSG_MAX)
            ed.setPlaceholderText("Mensaje %d" % (i + 1))
            ed.setText(state.mx_messages[i] if i < len(state.mx_messages) else "")
            ed.textChanged.connect(lambda text, idx=i: self.on_mx_message(idx, text))
            mgx.addWidget(ed)
            self.mx_msg_edits.append(ed)
        self.mx_msg_glow_chk = QtWidgets.QCheckBox("Brillo en los mensajes escondidos")
        self.mx_msg_glow_chk.setChecked(state.mx_msg_glow)
        self.mx_msg_glow_chk.toggled.connect(self.on_mx_msg_glow)
        mgx.addWidget(self.mx_msg_glow_chk)

        glow_box = QtWidgets.QGroupBox("Brillo (código y mensajes)")
        glx = QtWidgets.QVBoxLayout(glow_box)
        self.mx_glow_chk = QtWidgets.QCheckBox("Algunas columnas del código caen brillando")
        self.mx_glow_chk.setChecked(state.mx_glow_on)
        self.mx_glow_chk.toggled.connect(self.on_mx_glow_on)
        glx.addWidget(self.mx_glow_chk)
        self.mx_glow_int_slider = self._slider(
            glx, "Intensidad del brillo (código y mensajes)", 1, 10,
            state.mx_glow_int, self.on_mx_glow_int)
        self.mx_glow_blur_slider = self._slider(
            glx, "Difuminado del brillo (más a la derecha = más difuso)", 1, 10,
            state.mx_glow_blur, self.on_mx_glow_blur)
        mxl.addWidget(glow_box)
        mxl.addWidget(msg_box)
        mxl.addStretch(1)
        mx_scroll = QtWidgets.QScrollArea()
        mx_scroll.setWidgetResizable(True)
        mx_scroll.setFrameShape(QtWidgets.QFrame.NoFrame)
        mx_scroll.setWidget(mw)
        tabs.addTab(mx_scroll, "Matrix")

        # ============ pestaña Nubes ============
        cw = QtWidgets.QWidget()
        cl = QtWidgets.QVBoxLayout(cw)
        self.cl_amount_slider = self._slider(cl, "Cantidad de nubes", 1, 30,
                                             state.cl_amount, self.on_cl_amount)
        self.cl_opacity_slider = self._slider(cl, "Opacidad", 5, 100,
                                              state.cl_opacity, self.on_cl_opacity)
        self.cl_size_min_slider = self._slider(
            cl, "Tamaño mínimo de las nubes (% del ancho de la pantalla)", 3, 100,
            state.cl_size_min, self.on_cl_size_min)
        self.cl_size_max_slider = self._slider(
            cl, "Tamaño máximo de las nubes (% del ancho de la pantalla)", 3, 100,
            state.cl_size_max, self.on_cl_size_max)
        self.cl_white_slider = self._slider(
            cl, "Blancura (0 = grises de tormenta · 100 = blancas)", 0, 100,
            state.cl_white, self.on_cl_white)
        self.cl_wind_slider = self._slider(
            cl, "Viento (0 = quietas · más fuerte = se mueven más rápido y, si es fuerte, "
                "se deshacen)", 0, 10, state.cl_wind, self.on_cl_wind)
        cl.addWidget(QtWidgets.QLabel("Dirección del viento"))
        self.cl_dir_combo = QtWidgets.QComboBox()
        self.cl_dir_combo.addItems([
            "Hacia la derecha",
            "Hacia la izquierda",
            "Hacia ti (las nubes se acercan)",
            "Alejándose (las nubes se alejan)",
            "Cambiante (a veces a un lado, a veces al otro)",
        ])
        self.cl_dir_combo.setCurrentIndex(self._idx(SETTING_CHOICES["cl_dir"], state.cl_dir))
        self.cl_dir_combo.currentIndexChanged.connect(self.on_cl_dir)
        cl.addWidget(self.cl_dir_combo)
        cl.addWidget(QtWidgets.QLabel("Color del reflejo"))
        self.cl_refl_combo = QtWidgets.QComboBox()
        self.cl_refl_combo.addItems(["Azulado", "Naranja", "Rosa", "Gris", "Blanco", "Morado"])
        self.cl_refl_combo.setCurrentIndex(
            self._idx(SETTING_CHOICES["cl_refl_color"], state.cl_refl_color))
        self.cl_refl_combo.currentIndexChanged.connect(self.on_cl_refl_color)
        cl.addWidget(self.cl_refl_combo)
        self.cl_refl_slider = self._slider(cl, "Intensidad del reflejo", 0, 100,
                                           state.cl_refl_int, self.on_cl_refl_int)
        self._rainbow_box(cl, "cl")
        cl.addStretch(1)
        cl_scroll = QtWidgets.QScrollArea()
        cl_scroll.setWidgetResizable(True)
        cl_scroll.setFrameShape(QtWidgets.QFrame.NoFrame)
        cl_scroll.setWidget(cw)
        tabs.addTab(cl_scroll, "Nubes")

        # ============ pestaña Cascada ============
        wfw = QtWidgets.QWidget()
        wf = QtWidgets.QVBoxLayout(wfw)
        self.wf_x_slider = self._slider(
            wf, "Desde qué parte de arriba cae (0 = izquierda · 50 = centro · 100 = derecha)",
            0, 100, state.wf_x, lambda val: setattr(self.state, "wf_x", val))
        self.wf_width_slider = self._slider(
            wf, "Grosor de la cascada (% del ancho de la pantalla)", 3, 100,
            state.wf_width, lambda val: setattr(self.state, "wf_width", val))
        self.wf_amount_slider = self._slider(
            wf, "Cantidad de agua que cae", 1, 100,
            state.wf_amount, lambda val: setattr(self.state, "wf_amount", val))
        self.wf_opacity_slider = self._slider(
            wf, "Opacidad del agua (menos = más translúcida)", 5, 100,
            state.wf_opacity, lambda val: setattr(self.state, "wf_opacity", val))
        mist_box = QtWidgets.QGroupBox("Vapor y salpicado al caer")
        mtb = QtWidgets.QVBoxLayout(mist_box)
        self.wf_mist_chk = QtWidgets.QCheckBox("Activar vapor")
        self.wf_mist_chk.setChecked(state.wf_mist_on)
        self.wf_mist_chk.toggled.connect(lambda c: setattr(self.state, "wf_mist_on", c))
        mtb.addWidget(self.wf_mist_chk)
        self.wf_mist_slider = self._slider(
            mtb, "Cantidad de vapor", 0, 100,
            state.wf_mist, lambda val: setattr(self.state, "wf_mist", val))
        wf.addWidget(mist_box)
        self._rainbow_box(wf, "wf")
        wf.addStretch(1)
        wf_scroll = QtWidgets.QScrollArea()
        wf_scroll.setWidgetResizable(True)
        wf_scroll.setFrameShape(QtWidgets.QFrame.NoFrame)
        wf_scroll.setWidget(wfw)
        tabs.addTab(wf_scroll, "Cascada")

        # ventana de ajustes mas grande (sin pasarse del tamaño de la pantalla)
        try:
            ag = QtWidgets.QApplication.primaryScreen().availableGeometry()
            self.resize(min(640, ag.width() - 40), min(900, ag.height() - 80))
        except Exception:
            self.resize(640, 860)

        # mantiene el boton del limpiaparabrisas sincronizado (el atajo / la pasada unica
        # lo enciende y lo apaga por su cuenta)
        self._wiper_sync = QtCore.QTimer(self)
        self._wiper_sync.timeout.connect(self._sync_wiper_btn)
        self._wiper_sync.start(150)

        self._setup_tray()

    @staticmethod
    def _idx(seq, val, default=0):
        return seq.index(val) if val in seq else default

    # ---- bandeja del sistema: al minimizar, la app queda junto al reloj (no en la barra de tareas) ----
    def _setup_tray(self):
        self.tray = None
        self._tray_told = False
        try:
            if not QtWidgets.QSystemTrayIcon.isSystemTrayAvailable():
                return
        except Exception:
            return
        pm = QtGui.QPixmap(64, 64)
        pm.fill(QtCore.Qt.transparent)
        p = QtGui.QPainter(pm)
        p.setRenderHint(QtGui.QPainter.Antialiasing, True)
        p.setPen(QtCore.Qt.NoPen)
        p.setBrush(QtGui.QColor(70, 160, 240))
        p.drawEllipse(QtCore.QRectF(10, 24, 44, 36))                      # gota de agua
        p.drawPolygon(QtGui.QPolygonF([QtCore.QPointF(32, 3), QtCore.QPointF(14, 38),
                                       QtCore.QPointF(50, 38)]))
        p.setBrush(QtGui.QColor(255, 255, 255, 190))
        p.drawEllipse(QtCore.QRectF(20, 36, 9, 13))                       # brillo
        p.end()
        self.tray = QtWidgets.QSystemTrayIcon(QtGui.QIcon(pm), self)
        self.tray.setToolTip("Lluvia — clic para abrir el panel de control")
        menu = QtWidgets.QMenu()
        act_show = menu.addAction("Mostrar panel de control")
        act_show.triggered.connect(self.restore_from_tray)
        menu.addSeparator()
        act_quit = menu.addAction("Salir")
        act_quit.triggered.connect(QtWidgets.QApplication.quit)
        self.tray.setContextMenu(menu)
        self._tray_menu = menu
        self.tray.activated.connect(self._on_tray_activated)
        self.tray.show()
        QtWidgets.QApplication.instance().aboutToQuit.connect(self.tray.hide)

    def _on_tray_activated(self, reason):
        if reason in (QtWidgets.QSystemTrayIcon.Trigger, QtWidgets.QSystemTrayIcon.DoubleClick):
            self.restore_from_tray()

    def restore_from_tray(self):
        self.showNormal()
        self.raise_()
        self.activateWindow()

    def changeEvent(self, ev):
        super().changeEvent(ev)
        if (ev.type() == QtCore.QEvent.WindowStateChange and self.isMinimized()
                and getattr(self, "tray", None) is not None):
            QtCore.QTimer.singleShot(0, self._minimize_to_tray)

    def _minimize_to_tray(self):
        self.hide()                                                    # fuera de la barra de tareas
        self.setWindowState(self.windowState() & ~QtCore.Qt.WindowMinimized)
        if not self._tray_told:
            self._tray_told = True
            self.tray.showMessage(
                "Lluvia", "Sigue funcionando aquí, en la bandeja del sistema. "
                "Clic en el icono para abrir el panel.",
                QtWidgets.QSystemTrayIcon.Information, 4000)

    def _slider(self, layout, label, lo, hi, value, cb):
        lab = QtWidgets.QLabel(label)
        lab.setWordWrap(True)
        layout.addWidget(lab)
        s = QtWidgets.QSlider(QtCore.Qt.Horizontal)
        s.setRange(lo, hi)
        s.setValue(value)
        s.valueChanged.connect(cb)
        layout.addWidget(s)
        return s

    def _color_pair(self, layout, key, natural=False):
        """Dos combos de color: color 1 (o aleatorio) y color 2 opcional para combinar."""
        names1 = ["Verde", "Azul", "Morado", "Rosa", "Amarillo", "Blanco", "Aleatorio"]
        names2 = ["Ninguno (un solo color)"] + names1[:6]
        keys1 = STAR_COLOR_NAMES + ["aleatorio"]
        keys2 = ["ninguno"] + STAR_COLOR_NAMES
        if natural:                                   # estrellas fijas: opcion "natural" (como antes)
            names1 = ["Natural (blanco azulado y cálido)"] + names1
            keys1 = ["natural"] + keys1
        c1, c2 = self.state.sky_colors[key]
        layout.addWidget(QtWidgets.QLabel("Color"))
        cb1 = QtWidgets.QComboBox()
        cb1.addItems(names1)
        cb1.setCurrentIndex(keys1.index(c1))
        cb1.currentIndexChanged.connect(lambda i, kk=key, ks=keys1: self._set_color(kk, 0, ks[i]))
        layout.addWidget(cb1)
        layout.addWidget(QtWidgets.QLabel("Combinar con un segundo color (opcional)"))
        cb2 = QtWidgets.QComboBox()
        cb2.addItems(names2)
        cb2.setCurrentIndex(keys2.index(c2))
        cb2.currentIndexChanged.connect(lambda i, kk=key, ks=keys2: self._set_color(kk, 1, ks[i]))
        layout.addWidget(cb2)

    def _rainbow_box(self, layout, prefix):
        """Arcoiris (Nubes y Cascada): activar, posicion en la pantalla y opacidad."""
        box = QtWidgets.QGroupBox("Arcoíris")
        v = QtWidgets.QVBoxLayout(box)
        chk = QtWidgets.QCheckBox("Mostrar arcoíris")
        chk.setChecked(bool(getattr(self.state, prefix + "_rb_on")))
        chk.toggled.connect(lambda c, p=prefix: setattr(self.state, p + "_rb_on", c))
        v.addWidget(chk)
        sx = self._slider(v, "Posición horizontal (0 = izquierda · 100 = derecha)", 0, 100,
                          getattr(self.state, prefix + "_rb_x"),
                          lambda val, p=prefix: setattr(self.state, p + "_rb_x", val))
        sy = self._slider(v, "Posición vertical (0 = arriba · 100 = abajo)", 0, 100,
                          getattr(self.state, prefix + "_rb_y"),
                          lambda val, p=prefix: setattr(self.state, p + "_rb_y", val))
        so = self._slider(v, "Opacidad del arcoíris", 0, 100,
                          getattr(self.state, prefix + "_rb_op"),
                          lambda val, p=prefix: setattr(self.state, p + "_rb_op", val))
        layout.addWidget(box)
        setattr(self, prefix + "_rb_chk", chk)
        setattr(self, prefix + "_rb_x_slider", sx)
        setattr(self, prefix + "_rb_y_slider", sy)
        setattr(self, prefix + "_rb_op_slider", so)

    def _sync_wiper_btn(self):
        on = bool(self.state.wiper_active)
        if self.wiper_btn.isChecked() != on:
            self.wiper_btn.blockSignals(True)
            self.wiper_btn.setChecked(on)
            self.wiper_btn.setText(
                "Desactivar limpiaparabrisas" if on else "Activar limpiaparabrisas")
            self.wiper_btn.blockSignals(False)

    def _set_color(self, key, which, name):
        cfg = list(self.state.sky_colors[key])
        cfg[which] = name
        self.state.sky_colors[key] = tuple(cfg)

    def showEvent(self, event):
        super().showEvent(event)
        self.state.own_hwnds.add(int(self.winId()))

    # ---- callbacks ----
    def on_style(self, idx):
        self.state.style = ("pantalla", "ventana", "nieve", "estrellas",
                            "luciernagas", "matrix", "nubes", "cascada")[idx]

    def on_speed(self, val):
        self.state.interval_ms = max(10, 110 - val)

    def on_intensity(self, val):
        self.state.intensity = val

    def on_strength(self, val):
        self.state.strength = float(val)

    def on_drop_size(self, val):
        self.state.drop_size = val

    def on_persistence(self, val):
        self.state.wave_persistence = val / 1000.0

    def on_blur(self, val):
        self.state.glass_blur = max(1, val)

    def on_fog(self, val):
        self.state.fog = val / 100.0

    def on_sticky(self, val):
        self.state.stickiness = val

    def on_defog(self, val):
        self.state.wiper_defog = val / 100.0

    def on_fg(self, idx):
        # 0 dentro (clara) · 1 fuera (empañada) · 2 dinamica (maximizada = fuera)
        self.state.fg_inside = (idx != 1)
        self.state.fg_dynamic = (idx == 2)

    def on_fly_amount(self, val):
        self.state.fly_amount = val

    def on_fly_size(self, val):
        self.state.fly_size = val

    def on_fly_color(self, name, checked):
        cols = [c for c in FIREFLY_COLOR_NAMES
                if (c != name and c in self.state.fly_colors) or (c == name and checked)]
        if not cols:                       # siempre debe quedar al menos un color
            chk = self.fly_chks[name]
            chk.blockSignals(True)
            chk.setChecked(True)
            chk.blockSignals(False)
            cols = [name]
        self.state.fly_colors = cols

    def on_mx_color(self, idx):
        self.state.mx_color = ("verde", "azulado", "morado", "mixto")[idx]

    def on_mx_chars(self, idx):
        self.state.mx_chars = ("katakana", "numeros", "binario")[idx]

    def on_mx_size(self, val):
        self.state.mx_size = val

    def on_mx_speed(self, val):
        self.state.mx_speed = val

    def on_mx_speed_var(self, checked):
        self.state.mx_speed_var = checked

    def on_mx_message(self, idx, text):
        msgs = list(self.state.mx_messages) + [""] * 5
        msgs = msgs[:5]
        msgs[idx] = text
        self.state.mx_messages = msgs

    def on_mx_msg_glow(self, checked):
        self.state.mx_msg_glow = checked

    def on_mx_glow_on(self, checked):
        self.state.mx_glow_on = checked

    def on_mx_glow_int(self, val):
        self.state.mx_glow_int = val

    def on_mx_glow_blur(self, val):
        self.state.mx_glow_blur = val

    def on_mx_amount(self, val):
        self.state.mx_amount = val

    def on_mx_opacity(self, val):
        self.state.mx_opacity = val

    def on_mx_depth(self, val):
        self.state.mx_depth = val

    def on_cl_amount(self, val):
        self.state.cl_amount = val

    def on_cl_opacity(self, val):
        self.state.cl_opacity = val

    def on_cl_distance(self, val):
        self.state.cl_distance = val

    def on_cl_white(self, val):
        self.state.cl_white = val

    def on_cl_dir(self, idx):
        self.state.cl_dir = CLOUD_DIRS[idx]

    def on_cl_size_min(self, val):
        self.state.cl_size_min = val
        if val > self.state.cl_size_max:                 # el minimo no pasa del maximo
            self.cl_size_max_slider.setValue(val)

    def on_cl_size_max(self, val):
        self.state.cl_size_max = val
        if val < self.state.cl_size_min:
            self.cl_size_min_slider.setValue(val)

    def on_cl_wind(self, val):
        self.state.cl_wind = val

    def on_cl_refl_color(self, idx):
        self.state.cl_refl_color = CLOUD_REFL_NAMES[idx]

    def on_cl_refl_int(self, val):
        self.state.cl_refl_int = val

    def on_sky_meteors(self, val):
        self.state.sky_meteors = val

    def on_sky_stars(self, val):
        self.state.sky_stars = val

    def on_sky_comets_on(self, checked):
        self.state.sky_comets_on = checked

    def on_sky_comets_n(self, val):
        self.state.sky_comets_n = val

    def on_sky_aurora_on(self, checked):
        self.state.sky_aurora_on = checked

    def on_sky_aurora_n(self, val):
        self.state.sky_aurora_n = val

    def on_sky_nebula_on(self, checked):
        self.state.sky_nebula_on = checked

    def on_sky_dir(self, idx):
        self.state.sky_dir = ("right", "left", "toward", "away")[idx]

    def on_snow_on(self, checked):
        self.state.snow_on = checked

    def on_snow_amount(self, val):
        self.state.snow_amount = val

    def on_snow_size(self, val):
        self.state.snow_size = val

    def on_frost_on(self, checked):
        self.state.snow_frost_on = checked

    def on_frost(self, val):
        self.state.snow_frost = val

    def on_wind_on(self, checked):
        self.state.snow_wind_on = checked

    def on_wind_strength(self, val):
        self.state.snow_wind_strength = val

    def on_wind_dir(self, idx):
        self.state.snow_wind_dir = ("right", "left", "toward", "away", "variable")[idx]

    def on_snow_fog_move(self, checked):
        self.state.snow_fog_move = checked

    def on_snow_fog_on(self, checked):
        self.state.snow_fog_on = checked

    def on_snow_fog(self, val):
        self.state.snow_fog = val

    def on_rain_outside(self, checked):
        self.state.rain_outside = checked

    def on_rain_amount(self, val):
        self.state.rain_amount = val

    def on_rain_dir(self, idx):
        self.state.rain_dir = ("right", "left", "toward", "away", "variable")[idx]

    def on_rain_incl(self, val):
        self.state.rain_incl = val

    def on_snow_touch(self, checked):
        self.state.snow_touch = checked

    def on_snow_touch_size(self, val):
        self.state.snow_touch_size = val

    def on_snow_touch_regrow(self, val):
        self.state.snow_touch_regrow = val

    def on_finger(self, checked):
        self.state.finger_defog = checked

    def on_finger_size(self, val):
        self.state.finger_size = val

    def on_finger_regrow(self, val):
        self.state.finger_regrow = val

    def on_thunder(self, checked):
        self.state.thunder_on = checked

    def on_thunder_freq(self, val):
        self.state.thunder_freq = val

    def on_thunder_int(self, val):
        self.state.thunder_intensity = val

    def on_sound(self, checked):
        self.state.thunder_sound = checked

    def on_test_thunder(self):
        self.state.thunder_test = True

    def on_wiper(self, checked):
        self.state.wiper_active = checked
        self.state.wiper_single = False          # el boton = modo continuo
        self.wiper_btn.setText(
            "Desactivar limpiaparabrisas" if checked else "Activar limpiaparabrisas"
        )

    def on_wiper_speed(self, val):
        self.state.wiper_speed = val / 100.0


def register_extra_hotkeys(app, overlay):
    """Registra Ctrl + Mayus + Ñ (trueno) y Alt + Ñ (estrella fugaz / cometa). Si Windows no
    deja registrar alguno, el overlay lo sondea en cada frame como respaldo."""
    MOD_ALT, MOD_CONTROL, MOD_SHIFT, MOD_NOREPEAT = 0x0001, 0x0002, 0x0004, 0x4000
    specs = [("thunder", 0x5A12, MOD_CONTROL | MOD_SHIFT, overlay.hotkey_thunder),
             ("meteor", 0x5A13, MOD_ALT, overlay.hotkey_meteor)]
    filters = []
    for name, hk_id, mods, cb in specs:
        try:
            ok = ctypes.windll.user32.RegisterHotKey(None, hk_id, mods | MOD_NOREPEAT, ntilde_vk())
        except Exception:
            ok = 0
        if not ok:
            overlay.hk_poll_extra.add(name)
            continue
        filt = HotkeyFilter(cb, hk_id)
        app.installNativeEventFilter(filt)
        atexit.register(lambda i=hk_id: ctypes.windll.user32.UnregisterHotKey(None, i))
        filters.append(filt)
    return filters


if __name__ == "__main__":
    app = QtWidgets.QApplication(sys.argv)
    state = RainState()
    load_settings(state)                                   # recupera los ajustes guardados
    overlay = OverlayWindow(state)
    panel = ControlPanel(state)
    hotkey_filter = register_wiper_hotkey(app, overlay)   # Ctrl + Ñ (mantener referencia viva)
    extra_filters = register_extra_hotkeys(app, overlay)  # Ctrl+Mayus+Ñ y Alt+Ñ
    overlay.show()
    panel.show()

    # guarda solo los ajustes cuando cambian (y al cerrar)
    _last_saved = [json.dumps(state_to_dict(state), sort_keys=True)]

    def autosave():
        snap = json.dumps(state_to_dict(state), sort_keys=True)
        if snap != _last_saved[0] and save_settings(state):
            _last_saved[0] = snap

    autosave_timer = QtCore.QTimer()
    autosave_timer.timeout.connect(autosave)
    autosave_timer.start(1500)
    app.aboutToQuit.connect(autosave)
    sys.exit(app.exec_())
