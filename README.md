# CareDesk

**Simulador de central de teleasistencia para el módulo profesional de Teleasistencia (CFGM Atención a Personas en Situación de Dependencia).**

🌐 **[caredesk.me](https://caredesk.me)**

CareDesk recrea el puesto de trabajo de una central de teleasistencia para practicar en clase: recepción de alarmas, atención al usuario siguiendo el protocolo, clasificación de la incidencia, movilización de recursos, registro de la actuación y evaluación del alumnado — con **llamadas de voz y vídeo reales** entre profesorado y alumnado desde el navegador, sin instalar nada.

> ⚠️ **Uso educativo.** Proyecto independiente con fines docentes, inspirado en el tipo de funcionalidad de las plataformas de simulación de teleasistencia. No reutiliza código, textos, imágenes ni marcas de ningún producto comercial. Todos los datos de los casos son ficticios.

---

## ✨ Qué hace

### Para el profesor

- **Panel del centro** con sus alumnos, sus casos y su evaluación. Da de alta al alumnado por email (hasta 30 por licencia); cada alumno crea su propia contraseña la primera vez que entra.
- **Escenarios y usuarios ficticios** propios: ficha completa de la persona atendida (datos personales, contacto, dirección, dependencia, ayudas técnicas, contactos de emergencia, alergias, notas), guion, protocolo, recursos correctos y gravedad.
- **Lanzar un caso** a un alumno concreto, o una **simulación entre dos alumnos** (uno hace de teleoperador y el otro de persona dependiente).
- **Unirse a una llamada en curso** para escuchar en silencio y **intervenir** cuando quiera, sin cortar a nadie.
- **Evaluación práctica** por bloques sobre 100 puntos (No / Parcial / Sí), con nota, estrellas y observaciones. Puede editar y borrar actuaciones.
- **Exámenes** por módulos (MF1423 / MF1424 / MF1425): banco de preguntas, corrección automática del teórico y rúbrica para el práctico.
- **Estadísticas** por alumno y del grupo, y **encuestas** de fin de curso.
- **Configuración del centro** (datos de facturación y nombre visible) desde el propio simulador.

### Para el alumno

- **Puesto de la central**: le entra la alarma, atiende siguiendo el protocolo, clasifica y moviliza recursos, y al colgar queda registrada su actuación.
- **Practicar por su cuenta** un caso cuando quiera, sin esperar al profesor.
- **Practicar con un compañero**: elige a quién llamar de su clase, ve quién está conectado, y la práctica **no empieza hasta que los dos están dentro de la llamada**.
- **Mis actuaciones**: su historial con la nota y el comentario del profesor.
- **Exámenes** publicados por su profesor.

### Cuentas, licencias y clases

- **Login único** por email para profesorado y alumnado; el rol lo decide la cuenta, no el usuario.
- **Demo gratuita de 3 días** para probar, o **licencia de curso** para el centro.
- **Una sola sesión activa por cuenta**: si entras en otro dispositivo, el anterior se cierra solo. Cierre automático por inactividad (nunca durante una llamada).
- **Un alumno puede estar en varias clases** a la vez, con profesores distintos. Cada clase es independiente —sus casos, sus llamadas y sus notas— y el alumno elige en cuál entra, o cambia sin cerrar sesión.

### Grabaciones

Las llamadas se graban **solo en audio**, incluso cuando hay vídeo, y **se quedan en el ordenador del profesor** (IndexedDB del navegador). **Nunca se suben a internet**: es una decisión de diseño, no una limitación pendiente de resolver.

En Chrome y Edge, el profesor puede **elegir una carpeta de su disco** y cada grabación se copia sola ahí, con nombre legible (`Nombre_Alumno[-Segundo_Alumno]_AAAA-MM-DD_HH-MM_Escenario.webm`). En Firefox y Safari queda la descarga manual.

El plazo de conservación es el **curso escolar**. Al empezar uno nuevo, la app avisa de las grabaciones caducadas y ofrece borrarlas — pero **no las borra sola**, por si hay una reclamación de nota.

### Accesibilidad e idiomas

- **WCAG 2.1 AA / EN 301 549 / RD 1112/2018**, con declaración de accesibilidad dentro de la app. Auditoría automática con **axe-core: 0 incumplimientos** en escritorio y móvil, también con alto contraste y texto grande.
- Panel de **ajustes de accesibilidad**: tamaño de texto, alto contraste, reducir animaciones y cámara.
- Interfaz en **valencià, castellano e inglés** (los textos legales van siempre en castellano, con resumen en inglés).
- **Funciona en móvil** sin cambiar nada del diseño de escritorio: probado sin desbordamiento horizontal a 360, 390 y 820 px.

---

## 🏗️ Arquitectura

CareDesk es **un único archivo HTML** que se sirve tal cual, más dos servicios externos:

```
                    ┌────────────────────────────────────────────┐
  index.html  ──────┤ Supabase · cuentas, licencias y datos      │
  (navegador)       │ (Auth + Postgres con RLS + Edge Function)  │
        │           └────────────────────────────────────────────┘
        │
        │  (1) señalización: "presentaos"  ──►  PeerJS
        │  (2) voz y vídeo directos entre navegadores (P2P)
        └─ (3) si la red lo impide  ────────►  TURN (Metered) reenvía
```

- **Frontend**: HTML + CSS + JavaScript *vanilla*, sin framework ni proceso de *build*. PeerJS va **embebido** dentro del `index.html`.
- **Supabase**: autenticación, base de datos con **RLS** (cada profesor ve solo su clase; cada alumno, solo lo suyo) y una **Edge Function** para lo que el navegador no puede hacer con seguridad (cambiar el email de acceso de un alumno o eliminar su cuenta).
- **WebRTC** para el audio y el vídeo. La voz **no pasa** por el servidor de señalización.
- **TURN (Metered)** para cuando las dos redes no pueden conectar directamente — imprescindible si el centro aísla a los clientes de su wifi.

### Identidad en las llamadas

La sala se deriva de la cuenta del profesor, así que todos los que cuelgan de él calculan la misma sin escribir ningún código:

- Profesor → `SALA-PROF`
- Alumno → `SALA-AL-<slug de su email>`
- Simulación → `SALA-SIM-<caso>-<rol>` (teleoperador / persona dependiente / oyente)

> El identificador del alumno se deriva de su **email**, nunca de su nombre: un nombre lo puede cambiar el profesor, y si los dos extremos no calculan exactamente el mismo identificador, los mensajes se envían a una dirección que no existe y se pierden sin ningún error visible.

### Presencia

Alumnos y profesor se mandan **latidos** cada pocos segundos por canales de datos que se mantienen abiertos, así que la sala se sincroniza sola entre en el orden que entre cada uno. Los alumnos también se saludan entre ellos, de forma escalonada y acotada, para poder practicar en pareja **aunque el profesor no esté delante**.

El profesor **nunca abre conexiones en bloque**: con 30 alumnos eran decenas de negociaciones simultáneas y el servidor de señalización cerraba el socket. Son los alumnos los que saludan al entrar y laten; él contesta por ese mismo canal.

---

## 🚀 Publicación

Es una aplicación web de **un solo archivo**. Para publicarla:

1. Sube `index.html` a la raíz del repositorio.
2. **Settings → Pages** → *Source*: **Deploy from a branch**, rama `main`, carpeta `/ (root)`.
3. Con dominio propio, el archivo `CNAME` va en la raíz del repo y los registros DNS apuntan a GitHub Pages **sin proxy** (en Cloudflare, nube gris: con la naranja GitHub no emite el certificado).

> El micrófono, la cámara y las llamadas **solo funcionan en HTTPS** (GitHub Pages ya lo da) o en `http://localhost`. Abrir el archivo desde el disco con `file://` no funciona, y la app lo avisa.

### Puesta en marcha del backend

La app necesita un proyecto de **Supabase** con su esquema: `perfiles`, `profesores`, `licencias`, `invitaciones`, `alumnos`, `matriculas`, `actuaciones`, `escenarios`, `usuarios_ficticios`, `examenes_config`, `resultados_examen` y `sesiones`, con sus políticas RLS y sus funciones `SECURITY DEFINER`. Los scripts de migración se mantienen junto a la documentación del proyecto, fuera de este repositorio.

La `URL` y la **anon key** van embebidas en `index.html`: son públicas por diseño y la seguridad la dan las reglas RLS.

> 🔒 La **service_role** y la contraseña de la base de datos **no aparecen nunca** en el cliente. La `service_role` vive únicamente dentro de la Edge Function, en el servidor.

---

## 🧭 Cómo se usa en clase

1. **El profesor** entra en [caredesk.me](https://caredesk.me), registra su centro (o empieza la demo de 3 días) y da de alta a su alumnado por email.
2. **Cada alumno** entra con su email, crea su contraseña la primera vez y aparece en la sala.
3. El profesor **lanza un caso** a un alumno, o monta una **simulación entre dos**.
4. Si quiere, **se une a la llamada** para escuchar y corregir sobre la marcha.
5. Al colgar, la actuación queda registrada y el profesor la **evalúa** desde su panel; el alumno ve la nota en *Mis actuaciones*.

> En el móvil, si al conectar no se oye nada, basta con **tocar la pantalla** una vez: los navegadores bloquean el audio automático hasta que hay una interacción.

---

## 🔐 Protección de datos

El **centro educativo es el responsable** del tratamiento y CareDesk el **encargado**. Grabar para evaluar está amparado por la función docente y no requiere consentimiento, solo información transparente. Los datos se alojan en la **Unión Europea** (Irlanda), sin transferencias internacionales.

La app incluye aviso legal, política de privacidad y declaración de accesibilidad. Para un despliegue real existen además un contrato de encargo (art. 28 RGPD), una hoja informativa para alumnado y familias, y el registro de actividades (art. 30.2) — pendientes de revisión jurídica antes de firmar con ningún centro.

---

## 🗺️ Estado y hoja de ruta

**Hoy (Fase 2)**: producto con cuentas, licencias y persistencia real, en producción en caredesk.me.

Pendiente antes de vender la primera licencia:

- **Servidor de señalización propio** (PeerServer + coturn). El servicio público de PeerJS es un punto único de fallo y no es un apoyo aceptable para un producto de pago.
- **SMTP propio** para los correos de confirmación y recuperación de contraseña.
- **Pagos** (Stripe): hoy la licencia de curso se registra como pendiente y se activa a mano.
- **Revisión jurídica** de los documentos y paso del modo piloto a modo producto.
- **Contenido de los casos en inglés** (la interfaz ya lo está).

---

## 📁 Contenido del repositorio

```
index.html      # La aplicación completa (un único archivo)
CNAME           # Dominio propio para GitHub Pages
README.md       # Este archivo
```

---

## 📄 Licencia y aviso

Proyecto con fines **educativos** para la formación en Teleasistencia. Recreación independiente; los nombres, casos y personas atendidas son ficticios.
