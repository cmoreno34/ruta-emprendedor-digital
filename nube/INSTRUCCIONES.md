# Guardado en la nube de la Ruta: puesta en marcha (solo el profesor, una vez, ≈ 10 minutos)

Con esto, cada equipo guarda su trabajo **automáticamente en tu Google Drive** y lo recupera en cualquier ordenador con un **código de equipo** y un **PIN**. Lo que guardan en el navegador y la copia .json siguen funcionando como respaldo.

## 1. Crear el script
1. Entra en https://script.google.com con la cuenta de Google donde quieras guardar los datos y pulsa **Nuevo proyecto**.
2. Renómbralo, por ejemplo, «Ruta ED · nube».
3. Borra lo que haya en `Código.gs` y pega todo el contenido de [`Code.gs`](Code.gs) (está en esta misma carpeta).
4. Guarda (icono del disquete).

## 2. Autorizar y crear la carpeta y el panel
1. En la barra superior, elige la función **`instalar`** y pulsa **Ejecutar**.
2. Google pedirá permisos: **Revisar permisos** → elige tu cuenta → si aparece «Google no ha verificado esta aplicación», pulsa **Configuración avanzada** → **Ir a Ruta ED · nube (no seguro)** → **Permitir**. (Es tu propio script; solo accede a la carpeta y la hoja que crea.)
3. En el registro de ejecución aparecen dos enlaces:
   - la carpeta **«Ruta Emprendedor Digital · datos»** (un archivo .json por equipo y la subcarpeta *historial* con las últimas 40 versiones de cada equipo);
   - la hoja **«Ruta ED · panel del profesor»** (una fila por equipo: proyecto, miembros, versión, última actualización, progreso y última estación).

## 3. Publicarlo como aplicación web
1. **Implementar** → **Nueva implementación** → icono del engranaje → **Aplicación web**.
2. Descripción: «Ruta ED v1». **Ejecutar como: Yo**. **Quién tiene acceso: Cualquier usuario**.
3. **Implementar** y copia la **URL de la aplicación web** (termina en `/exec`).
4. Comprueba que responde: abre en el navegador la URL añadiendo `?a=ping` al final. Debe mostrar `{"ok":true,"service":"ruta-ed",…}`.

## 4. Conectar la web
- **Opción A:** pásale la URL a Claude y la pondrá en `nube/config.js` y publicará la web.
- **Opción B:** en GitHub, abre `nube/config.js` del repositorio `ruta-emprendedor-digital` → icono del lápiz → pega la URL entre las comillas de `window.RUTA_CLOUD_URL = "";` → **Commit changes**. En 1–2 minutos la web mostrará la barra «Guardado automático en la nube».

## 5. En clase
- Da a cada equipo un **código**: `ED-01`, `ED-02`, … (letras, números y guiones).
- Cada equipo elige un **PIN** de 4 a 12 caracteres la primera vez que se conecta y lo usa siempre igual en todos sus ordenadores.
- **PIN olvidado:** en la hoja del panel, borra la celda `pin_hash` de ese equipo. La siguiente vez que se conecten, el PIN que pongan será el nuevo.
- **Recuperar una versión anterior:** en la carpeta *historial* están las últimas 40 versiones de cada equipo (`ED-03_v12_20261020-1130.json`). Descárgala y el equipo la abre con «Abrir copia».
- **Dos personas a la vez:** si dos ordenadores del mismo equipo guardan cambios distintos, la web avisa y deja elegir qué versión conservar; la otra queda en el historial.

## 6. Medidor de los prototipos (analítica casera)
El mismo script cuenta las **visitas, clics en el CTA, registros y clics en Premium** de los prototipos que publican los equipos en GitHub Pages. No hace falta Tally, Clarity ni ninguna cuenta externa.
- Cada prototipo lleva un pequeño script (lo genera la estación 9 · Prototipo con el código del equipo) que envía cada evento a esta URL. No usa cookies ni guarda nada en el navegador del visitante.
- Los eventos se apuntan en la pestaña **«eventos»** de la hoja del panel (fecha, equipo, evento, versión A/B, utm_source, utm_campaign y, solo en los registros, el email).
- Los equipos ven **solo los recuentos** en la estación 11 · Validación cuantitativa («Traer datos del prototipo»), con su código y PIN. **Los emails solo los ves tú**: bórralos al terminar el curso.
- Solo cuentan eventos de equipos que ya existen en el panel (se han conectado al menos una vez).
- **Si ya tenías el script desplegado:** pega el nuevo `Code.gs`, guarda y ve a **Implementar → Gestionar implementaciones → lápiz → Versión: Nueva versión → Implementar**. La URL no cambia.

## 7. Encuesta cruzada en clase
En la estación 11 cada equipo publica su encuesta (con su prototipo dentro) y el resto de la clase la contesta desde el móvil. Las respuestas llegan directamente a la Ruta del equipo.
- Pestaña **«encuestas»**: una fila por equipo con su cuestionario. Pestaña **«respuestas»**: una fila por respuesta (fecha, equipo y respuestas). Son anónimas: no se pide nombre ni email.
- **Página de la clase:** proyecta el QR de `https://cmoreno34.github.io/ruta-emprendedor-digital/?movil=encuestas`: lista todas las encuestas publicadas y cada alumno marca las que ya ha contestado (pide que contesten al menos 5 de otros equipos).
- Para empezar de cero en otra edición del curso, borra las filas de las pestañas «encuestas», «respuestas» y «eventos».

## Privacidad y límites
- Los datos se guardan en **tu** Drive; no pasan por servidores de terceros. Los alumnos no deben introducir datos personales de entrevistados.
- La URL del script es pública (está en la web), pero sin código de equipo y PIN no se puede leer ni escribir nada.
- Cuotas gratuitas de Google Apps Script: de sobra para varias clases (cada guardado es una petición pequeña).
- Si cambias el código del script, vuelve a **Implementar → Gestionar implementaciones → Editar → Nueva versión** para que la URL siga siendo la misma.
