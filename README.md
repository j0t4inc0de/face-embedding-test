# face-embedding-test

Este proyecto es una demostración de reconocimiento facial utilizando "face embeddings" en una aplicación Vue.js. Utiliza la librería `face-api.js` para realizar la detección y el reconocimiento de rostros directamente en el navegador.

## ¿Cómo funciona el proyecto?

El proyecto se divide en dos secciones principales:

1.  **Registro de Personas**:
    *   Activa la cámara web del usuario.
    *   Permite al usuario introducir un nombre.
    *   Al hacer clic en "Capturar y Registrar", la aplicación detecta un rostro en el video de la cámara.
    *   Si se detecta un rostro, `face-api.js` calcula un "face embedding" (un vector numérico de 128 dimensiones que representa las características únicas del rostro).
    *   **Importante**: La imagen del rostro capturado *no se almacena*. Solo se guarda el nombre de la persona y su correspondiente vector numérico (el embedding).
    *   Los datos registrados (nombre y embedding) se guardan en el `localStorage` del navegador para persistir entre sesiones.

2.  **Reconocimiento de Personas**:
    *   Al hacer clic en "Iniciar Reconocimiento", se activa una segunda instancia de la cámara web.
    *   La aplicación detecta continuamente rostros en el flujo de video.
    *   Por cada rostro detectado, calcula su embedding y lo compara con todos los embeddings previamente registrados.
    *   Si encuentra una coincidencia con una "distancia" suficientemente pequeña (indicando alta similitud), se dibuja un cuadro alrededor del rostro con el nombre de la persona.
    *   Si una persona registrada es reconocida por primera vez en la sesión de reconocimiento, se muestra una alerta de "Bienvenido [Nombre]".
    *   Si la persona sale del cuadro y vuelve a entrar, se le dará la bienvenida de nuevo.

## ¿Dónde se guardan los datos biométricos (vectores numéricos)?

Actualmente, los "face embeddings" (los vectores numéricos que representan los rostros) se almacenan en el **`localStorage` del navegador**.

*   **Ventajas**: Es simple de implementar, no requiere un servidor backend y los datos permanecen en el dispositivo del usuario.
*   **Desventajas**:
    *   **Persistencia limitada**: Los datos solo existen en el navegador específico donde se registraron. Si el usuario cambia de navegador, borra el caché o usa otro dispositivo, los datos no estarán disponibles.
    *   **Seguridad**: Aunque los embeddings no son reversibles a una imagen, no están cifrados en `localStorage`. Para aplicaciones más serias, se necesitaría un almacenamiento más seguro.
    *   **Escalabilidad**: No es adecuado para compartir datos entre múltiples usuarios o dispositivos.

## ¿Qué tan exacto es el reconocimiento?

La librería `face-api.js` utiliza modelos de aprendizaje profundo pre-entrenados que son bastante precisos para la detección y el reconocimiento facial en entornos controlados.

*   **Face Embeddings**: Son representaciones de alta dimensión de los rostros, diseñadas para que rostros similares tengan embeddings "cercanos" en el espacio vectorial.
*   **Umbral de Distancia (Distance Threshold)**: En el código, se utiliza un umbral de `0.5` en el `faceapi.FaceMatcher`. Este valor determina qué tan "cercanos" deben estar dos embeddings para considerarse la misma persona.
    *   Un valor más bajo (ej. `0.4`) hará el reconocimiento más estricto (menos falsos positivos, pero más falsos negativos).
    *   Un valor más alto (ej. `0.6`) hará el reconocimiento más laxo (más falsos positivos, pero menos falsos negativos).
*   **Factores que afectan la precisión**:
    *   **Iluminación**: Cambios drásticos en la luz pueden afectar la detección y la calidad del embedding.
    *   **Ángulo y Pose**: Rostros de perfil o con ángulos extremos pueden ser más difíciles de reconocer.
    *   **Expresiones faciales**: Expresiones muy diferentes a la del registro pueden reducir la precisión.
    *   **Calidad de la cámara**: Una cámara de baja resolución o con ruido puede generar embeddings menos fiables.

## Posibles mejoras

1.  **Almacenamiento Persistente y Centralizado**: Mover el almacenamiento de los embeddings a un backend (como se describe a continuación) para que los datos persistan y sean accesibles desde cualquier dispositivo.
2.  **Interfaz de Usuario Mejorada**:
    *   Indicar visualmente si un rostro es "conocido" (ej. cuadro verde) o "desconocido" (ej. cuadro rojo).
    *   Mostrar un indicador de confianza junto al nombre reconocido.
    *   Mejorar la retroalimentación visual durante el registro (ej. un círculo que se llena cuando el rostro está bien posicionado).
3.  **Detección de Múltiples Rostros**: Mejorar la gestión cuando hay varias personas en el encuadre, mostrando todos los nombres reconocidos.
4.  **Detección de "Liveness" (Anti-Spoofing)**: Implementar técnicas para asegurar que el rostro detectado es de una persona viva y no una foto o un video.
5.  **Gestión de Usuarios**: Una interfaz para ver, editar o eliminar usuarios registrados.
6.  **Optimización de Rendimiento**: Ajustar la frecuencia de detección para equilibrar la fluidez y el uso de recursos, especialmente en dispositivos móviles o con menos potencia.
7.  **Seguridad**: Cifrar los embeddings si se almacenan en el cliente o asegurar la comunicación con el backend.


## Project setup
```
npm install
```

### Compiles and hot-reloads for development
```
npm run serve
```

### Compiles and minifies for production
```
npm run build
```

### Lints and fixes files
```
npm run lint
```

### Customize configuration
See [Configuration Reference](https://cli.vuejs.org/config/).
