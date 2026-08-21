# CareDesk

**Simulador de central de teleasistencia para el módulo profesional de Teleasistencia (CFGM Atención a Personas en Situación de Dependencia · TAPSD).**

CareDesk recrea el puesto de trabajo de una central de teleasistencia para practicar en clase: recepción de alarmas, atención al usuario siguiendo el protocolo, gestión de incidencias, agenda de seguimiento, evaluación del alumnado y **llamadas de voz reales** entre el profesorado y el alumnado desde el navegador.

> ⚠️ **Uso educativo.** Proyecto independiente con fines docentes, inspirado en el tipo de funcionalidad de las plataformas de simulación de teleasistencia. No reutiliza código, textos, imágenes ni marcas de ningún producto comercial. Todos los datos son ficticios.

---

## ✨ Características

- **Roles Profesor / Alumno**, con un panel de gestión para el profesor y un puesto de central para el alumno.
- **Usuarios ficticios** con ficha completa (datos personales, contacto, dirección, dependencia, ayudas técnicas, contactos de emergencia, alergias, notas) y observaciones con valoración por estrellas.
- **Escenarios y central simulada**: alarma entrante, ficha del usuario, protocolo, clasificación de la incidencia, movilización de recursos y registro de la actuación.
- **Evaluación práctica** por bloques sobre 100 puntos (No / Parcial / Sí) con nota y estrellas.
- **Agenda avanzada**: calendario mes/semana/día, 17 tipos de evento, eventos con llamada, citas repetidas (series), pares aleatorios y estados (programado / en curso / completado / cancelado).
- **Exámenes** por módulos (MF1423 / MF1424 / MF1425): banco de preguntas, exámenes teóricos con corrección automática, rúbrica práctica y resultados.
- **Alertas y recordatorios**, **Estadísticas** (por alumno y globales) y **Encuestas** de fin de curso.
- **Llamadas de voz reales (WebRTC)**:
  - Profesor → alumno (llamada directa).
  - Alumno → compañero.
  - Emergencias simuladas **112 / 061** (entran al profesor).
  - **Simulación en grupo a 3**: persona dependiente + teleoperador + profesor que se une a escuchar/supervisar (puede salir sin cortar la llamada).
- Extras: perfil, cambiar contraseña, ayuda/FAQ y campana de notificaciones.

---

## 🚀 Publicación (GitHub Pages)

Es una aplicación web de **un solo archivo** (`index.html`), con la librería de llamadas y la configuración del relay ya incluidas.

1. Sube `index.html` a la raíz de este repositorio.
2. Ve a **Settings → Pages**.
3. En **Source** elige **Deploy from a branch**, rama **`main`**, carpeta **`/ (root)`** y **Save**.
4. En ~1 minuto tendrás la URL: `https://TU-USUARIO.github.io/TU-REPO/`.

> El micrófono del navegador **solo funciona en HTTPS** (GitHub Pages ya lo proporciona) o en `http://localhost`.

---

## 🧭 Cómo se usa

1. Todos abren la **misma URL**.
2. Escriben el **mismo código de sala/centro** (por defecto `CENTRO-1`). El **profesor** define ese código y lo comparte; los **alumnos** lo escriben para unirse.
3. Aceptan el **permiso de micrófono**.
4. Uno entra como **Profesor** y el resto como **Alumno** (nombres precargados: Lucía, Marcos, Aixa).

**Prueba de llamada a 3 (la más completa):**
- Alumno 1 → Central → *Simulación en grupo* → **Persona dependiente**.
- Alumno 2 → Central → *Simulación en grupo* → **Teleoperador**.
- Profesor → Inicio → *Unirse a simulación* → **Profesor / Oyente**.

> En móvil, si al conectar no se oye, basta con **tocar la pantalla** una vez (los navegadores móviles bloquean el audio automático hasta que hay una interacción).

---

## 🏗️ Arquitectura (resumen)

CareDesk es **todo frontend**: se ejecuta en el navegador, sin backend propio.

- **Navegador (frontend)** — la app y la lógica; usa **WebRTC** (nativo del navegador) para el audio.
- **Señalización** — servicio **público y gratuito de PeerJS** que "presenta" a los navegadores antes de la llamada. La voz **no** pasa por él.
- **TURN (relay)** — **Metered**, reenvía la voz cuando las dos redes no pueden conectar directamente (imprescindible para llamadas entre redes/antenas distintas).

```
Navegador A ─┐        (1) señalización: "presentaos" (PeerJS)
             ├──────────────────────────────────────────────
Navegador B ─┘        (2) voz directa entre navegadores (P2P)
                      (3) si la red bloquea lo directo → TURN (Metered) reenvía la voz
```

### Configuración del TURN (Metered)
Las credenciales del TURN van embebidas en `index.html` (bloque `TURN_SERVERS_SIM`). Son credenciales **de TURN** (pensadas para el front-end); si necesitas cambiarlas, genera unas nuevas en tu panel de [Metered](https://dashboard.metered.ca) → *TURN Server* y sustituye ese bloque. El plan gratuito incluye una cuota mensual de relay.

---

## 🔧 Descripción técnica

**Stack:** HTML + CSS + JavaScript *vanilla* (sin framework ni proceso de *build*). La única dependencia, **PeerJS**, va **embebida** dentro del `index.html`, por lo que la app es un **único archivo autoservido**: no requiere instalación, ni Node, ni servidor de aplicaciones.

- **Frontend puro**: todo se ejecuta en el navegador del usuario.
- **WebRTC** (nativo del navegador) para el audio de las llamadas.
- **PeerJS** para la señalización (descubrimiento y *handshake* entre pares).
- **STUN + TURN de Metered** para la travesía de NAT (que la voz cruce entre redes distintas).
- **Sin base de datos**: el estado vive en memoria (objeto `state`) y se reinicia al recargar. Es una decisión consciente de la Fase 1.

## ⚙️ Cómo funciona por dentro

**Interfaz por vistas.** Hay dos roles (Profesor / Alumno). Según el rol, se pinta una barra de pestañas y cada pestaña renderiza su vista con funciones `v*()` (por ejemplo `vAgenda`, `vExamenes`, `vFicticios`). Todo el estado (alumnos, usuarios ficticios, escenarios, agenda, exámenes, alertas, etc.) se guarda en un objeto `state` en memoria y las vistas se regeneran al vuelo.

**Identidad en las llamadas.** Como no hay backend, la coordinación de las llamadas se hace con un **código de sala** compartido + la identidad de cada participante, formando un identificador PeerJS determinista:

- Profesor → `SALA-PROF`
- Alumno → `SALA-AL-<nombre-normalizado>`
- Simulación en grupo → `SALA-SIM-PD` / `SALA-SIM-OP` / `SALA-SIM-OB` (persona dependiente / teleoperador / oyente)

Así, cuando el profesor pulsa *Llamar* sobre "Lucía Fernández", el navegador llama al identificador `SALA-AL-lucia-fernandez`, que es el que ha registrado el navegador de esa alumna.

**Flujo de una llamada (paso a paso):**
1. Cada navegador se registra en la señalización de PeerJS con su identificador (sala + rol/nombre).
2. Al iniciar una llamada, el emisor solicita al receptor mediante la señalización (widget "Llamando…" / "Llamada entrante").
3. El receptor **acepta** y ambos negocian la conexión WebRTC (con los servidores STUN/TURN configurados).
4. La **voz viaja directa** entre los dos navegadores (P2P). Si sus redes lo impiden, el **TURN (Metered)** reenvía el audio.
5. La simulación en grupo forma una **malla**: cada participante se conecta con los otros dos; el oyente entra silenciado y puede salir sin cortar la llamada de los alumnos.

## 🗂️ Modelo de datos (entidades principales)

En memoria, dentro de `state`:

- **alumnos / profesores** — participantes del centro.
- **usuarios (ficticios)** — personas simuladas con ficha extensa + observaciones.
- **escenarios** — casos con ficha, guion, protocolo, recursos correctos y gravedad.
- **agenda** — eventos (tipo, fecha/hora, alumno, con-llamada, serie, estado…).
- **registros** — actuaciones del alumno en cada llamada simulada + su evaluación por bloques.
- **examenes** — módulos, banco de preguntas, exámenes publicados, rúbricas y resultados.
- **alertas**, **encuestas** — recordatorios y satisfacción de fin de curso.

## 🧩 Estructura del código

Todo en `index.html`, en secciones claramente separadas por comentarios:

- Bloque `<style>` con el sistema visual (tema oscuro, tarjetas, tablas, calendario, widgets de llamada).
- **PeerJS embebido**.
- `state` (datos y catálogos) y utilidades (`uid`, `esc`, `norm`, fechas…).
- Navegación (`TABS`, `renderTabs`, `render`) y vistas por módulo (`v*`).
- Motor de **llamadas reales** (`rtc*`) y de **simulación en grupo** (`sim*`).

## 🗺️ Estado y hoja de ruta

- **Fase 1 (actual):** aplicación de un solo archivo, datos en memoria (se reinician al recargar), señalización y TURN mediante servicios externos.
- **Fase 2 (futuro):** backend (Node) con cuentas/login, centros multi-tenant, licencias y **persistencia real** de los datos; monitor de "llamadas en curso" y consola de teleasistencia a pantalla completa.

---

## 📁 Contenido del repositorio

```
index.html      # La aplicación completa (un único archivo)
README.md       # Este archivo
```

---

## 📄 Licencia y aviso

Proyecto con fines **educativos** para la formación en Teleasistencia (TAPSD). Recreación independiente; los nombres, casos y usuarios son ficticios. Añade aquí la licencia que prefieras (por ejemplo, MIT) si vas a compartirlo públicamente.
