<template>
  <div id="app-container">
    <h1>Reconocimiento Facial con Face Embeddings en Vue.js</h1>

    <div v-if="loading" class="loading-overlay">
      <div class="spinner"></div>
      <p>Cargando modelos de IA...</p>
    </div>

    <div class="main-content" v-else>
      <!-- Sección de Registro -->
      <div class="section">
        <h2>1. Registrar Persona</h2>
        <p class="instructions">Coloca tu rostro dentro del óvalo y mira a la cámara. El registro será automático.</p>
        <div class="camera-container">
          <video ref="videoRegister" width="640" height="480" autoplay muted class="camera-video"></video>
          <canvas ref="canvasRegister" width="640" height="480" class="feedback-canvas"></canvas>
        </div>
        <div class="controls">
          <input type="text" v-model="newUserName" placeholder="Nombre de la persona" />
        </div>
        <p v-if="feedback" class="feedback">{{ feedback }}</p>
      </div>

      <!-- Sección de Reconocimiento -->
      <div class="section">
        <h2>2. Reconocer Persona</h2>
        <div class="camera-container">
          <video ref="videoRecognize" width="640" height="480" autoplay muted></video>
          <canvas ref="canvasRecognize" width="640" height="480"></canvas>
        </div>
        <button @click="startRecognition">Iniciar Reconocimiento</button>
        <button @click="stopRecognition">Detener Reconocimiento</button>
      </div>

    </div>

    <!-- Sección de Personas Registradas -->
    <div class="registered-users" v-if="!loading && registeredUsers.length > 0">
      <h3>Personas Registradas</h3>
      <ul>
        <li v-for="user in registeredUsers" :key="user.name">
          {{ user.name }}
        </li>
      </ul>
      <button @click="clearRegisteredUsers" class="clear-button">Limpiar Registros</button>
    </div>
  </div>
</template>

<script>
import * as faceapi from 'face-api.js';

export default {
  name: 'App',
  data() {
    return {
      loading: true,
      newUserName: '',
      registeredUsers: [],
      feedback: '',
      recognitionInterval: null,
      faceMatcher: null,
      welcomedUsers: new Set(), // Para rastrear a quién ya se le dio la bienvenida
      registrationLoopInterval: null,
      registrationCountdown: 3,
      isCountingDown: false,
      registrationCountdownInterval: null,
      facePositionFeedback: {
        detected: false,
        centered: false,
        sizeOk: false,
        neutralExpression: false,
        allOk: false,
        landmarks: null,
      },
    };
  },
  async mounted() {
    await this.loadModels();
    this.startCamera('videoRegister');
    this.loadRegisteredUsers();
  },
  beforeUnmount() {
    this.stopRecognition();
    this.stopCamera();
    this.stopRegistrationFeedbackLoop();
  },
  methods: {
    async loadModels() {
      const MODEL_URL = '/models';
      try {
        await Promise.all([
          faceapi.nets.ssdMobilenetv1.loadFromUri(MODEL_URL),
          faceapi.nets.faceLandmark68Net.loadFromUri(MODEL_URL),
          faceapi.nets.faceRecognitionNet.loadFromUri(MODEL_URL),
          faceapi.nets.faceExpressionNet.loadFromUri(MODEL_URL),
        ]);
        this.loading = false;
        console.log('Modelos cargados correctamente.');
      } catch (error) {
        console.error('Error al cargar los modelos de face-api.js:', error);
        this.feedback = 'Error al cargar los modelos de IA. Por favor, recarga la página.';
      }
    },

    async startCamera(videoRefName) {
      try {
        const stream = await navigator.mediaDevices.getUserMedia({ video: {} });
        if (videoRefName === 'videoRegister') {
          this.$refs[videoRefName].onloadedmetadata = () => this.startRegistrationFeedbackLoop();
        }
        this.$refs[videoRefName].srcObject = stream;
      } catch (error) {
        console.error('Error al acceder a la cámara:', error);
        this.feedback = 'No se pudo acceder a la cámara. Asegúrate de dar los permisos necesarios.';
      }
    },

    stopSingleCamera(videoRefName) {
      const video = this.$refs[videoRefName];
      if (video && video.srcObject) {
        const tracks = video.srcObject.getTracks();
        tracks.forEach(track => track.stop());
        video.srcObject = null;
      }
      if (videoRefName === 'videoRegister') {
        this.stopRegistrationFeedbackLoop();
      }
    },

    stopCamera() {
      this.stopSingleCamera('videoRegister');
      this.stopSingleCamera('videoRecognize');
    },
    
    // Nuevo: Iniciar el bucle de feedback para el registro
    startRegistrationFeedbackLoop() {
      if (this.registrationLoopInterval) {
        clearInterval(this.registrationLoopInterval);
      }
      const video = this.$refs.videoRegister;
      const canvas = this.$refs.canvasRegister;
      if (!video || !canvas) return;

      // Asegurarse de que los modelos estén cargados antes de empezar
      if (this.loading) {
        console.warn('Esperando a que los modelos se carguen para iniciar el feedback de registro.');
        return;
      }

      const displaySize = { width: video.width, height: video.height };
      faceapi.matchDimensions(canvas, displaySize);

      this.registrationLoopInterval = setInterval(async () => {
        if (!video.srcObject) { // Asegurarse de que el video esté activo
          this.resetFacePositionFeedback();
          this.drawRegistrationFeedback(canvas, this.facePositionFeedback);
          return;
        }

        const detections = await faceapi.detectSingleFace(video).withFaceLandmarks().withFaceExpressions();
        let detected = !!detections;
        let centered = false;
        let sizeOk = false;
        let neutralExpression = false;
        let landmarks = null;

        if (detected) {
          const { detection: detectionBox, expressions } = detections;
          landmarks = detections.landmarks;
          const box = detectionBox.box;
          const videoCenter = { x: video.width / 2, y: video.height / 2 };
          const faceCenter = { x: box.x + box.width / 2, y: box.y + box.height / 2 };

          // Comprobar si el rostro está centrado (dentro de una tolerancia)
          const centerXTolerance = video.width * 0.15; // 15% del ancho del video
          const centerYTolerance = video.height * 0.15; // 15% del alto del video
          centered = Math.abs(faceCenter.x - videoCenter.x) < centerXTolerance && Math.abs(faceCenter.y - videoCenter.y) < centerYTolerance;

          // Comprobar si el tamaño del rostro es apropiado (ej. 30-70% del alto del video)
          const minFaceHeight = video.height * 0.3;
          const maxFaceHeight = video.height * 0.7;
          sizeOk = box.height > minFaceHeight && box.height < maxFaceHeight;

          // Comprobar si la expresión es neutra
          if (expressions) {
            const dominantExpression = Object.keys(expressions).reduce((a, b) => expressions[a] > expressions[b] ? a : b);
            neutralExpression = dominantExpression === 'neutral' && expressions.neutral > 0.8;
          }
        }

        this.facePositionFeedback = {
          detected,
          centered,
          sizeOk,
          neutralExpression,
          allOk: detected && centered && sizeOk && neutralExpression,
          // Pasamos los landmarks para dibujarlos
          landmarks,
        };
        
        if (this.facePositionFeedback.allOk && this.newUserName && !this.isCountingDown) {
          this.startRegistrationCountdown();
        } else if ((!this.facePositionFeedback.allOk || !this.newUserName) && this.isCountingDown) {
          this.stopRegistrationCountdown();
        }
        
        this.drawRegistrationFeedback(canvas, this.facePositionFeedback);
      }, 100); // Ejecutar cada 100ms
    },

    // Nuevo: Detener el bucle de feedback para el registro
    stopRegistrationFeedbackLoop() {
      this.stopRegistrationCountdown();

      if (this.registrationLoopInterval) {
        clearInterval(this.registrationLoopInterval);
        this.registrationLoopInterval = null;
      }
      // Limpiar el canvas al detener el bucle
      const canvas = this.$refs.canvasRegister;
      if (canvas) {
        const ctx = canvas.getContext('2d');
        ctx.clearRect(0, 0, canvas.width, canvas.height);
      }
      this.resetFacePositionFeedback();
    },

    resetFacePositionFeedback() {
      this.facePositionFeedback = { detected: false, centered: false, sizeOk: false, neutralExpression: false, allOk: false, landmarks: null };
    },

    // Nuevo: Iniciar cuenta atrás para el registro automático
    startRegistrationCountdown() {
      this.isCountingDown = true;
      this.registrationCountdownInterval = setInterval(() => {
        this.registrationCountdown--;
        if (this.registrationCountdown === 0) {
          this.captureAndRegister();
          this.stopRegistrationCountdown();
        }
      }, 1000);
    },

    // Nuevo: Detener la cuenta atrás
    stopRegistrationCountdown() {
      if (this.registrationCountdownInterval) {
        clearInterval(this.registrationCountdownInterval);
        this.registrationCountdownInterval = null;
      }
      this.isCountingDown = false;
      this.registrationCountdown = 3;
    },

    async captureAndRegister() {
      if (!this.newUserName) {
        this.feedback = 'Por favor, introduce un nombre.';
        return;
      }

      const video = this.$refs.videoRegister;
      const detections = await faceapi.detectSingleFace(video).withFaceLandmarks().withFaceDescriptor();

      if (detections) {
        // Guardamos el nombre y el "embedding" (descriptor)
        this.registeredUsers.push({
          name: this.newUserName,
          descriptor: detections.descriptor,
        });

        // Actualizamos el FaceMatcher para el reconocimiento
        this.updateFaceMatcher();

        // Guardamos en localStorage
        this.saveRegisteredUsers();

        this.feedback = `¡${this.newUserName} registrado correctamente!`;
        this.newUserName = ''; // Limpiamos el input

        // Borramos la imagen (en este caso, no la guardamos, solo el vector)
        console.log('Rostro registrado. La imagen no se almacena, solo su embedding.');

      } else {
        this.feedback = 'No se detectó ningún rostro. Inténtalo de nuevo.';
      }
    },

    // Nuevo: Dibujar feedback en el canvas de registro
    drawRegistrationFeedback(canvas, feedbackState) {
      const ctx = canvas.getContext('2d');
      ctx.clearRect(0, 0, canvas.width, canvas.height);

      const centerX = canvas.width / 2;
      const centerY = canvas.height / 2;
      const radiusX = canvas.width * 0.2;
      const radiusY = canvas.height * 0.25;

      // Dibuja el óvalo guía
      ctx.beginPath();
      ctx.ellipse(centerX, centerY, radiusX, radiusY, 0, 0, 2 * Math.PI);
      ctx.lineWidth = 5;
      ctx.setLineDash([10, 5]); // Línea discontinua para el óvalo

      // Dibuja guías de ojos, nariz y barbilla dentro del óvalo
      const guideColor = 'rgba(255, 255, 255, 0.5)';
      // Línea de ojos
      ctx.beginPath();
      ctx.moveTo(centerX - radiusX * 0.6, centerY - radiusY * 0.2);
      ctx.lineTo(centerX + radiusX * 0.6, centerY - radiusY * 0.2);
      ctx.strokeStyle = guideColor;
      ctx.lineWidth = 1;
      ctx.stroke();
      // Punto de nariz
      ctx.beginPath();
      ctx.arc(centerX, centerY + radiusY * 0.1, 3, 0, 2 * Math.PI);
      ctx.fillStyle = guideColor;
      ctx.fill();
      // Línea de barbilla
      ctx.beginPath();
      ctx.moveTo(centerX - radiusX * 0.3, centerY + radiusY * 0.7);
      ctx.lineTo(centerX + radiusX * 0.3, centerY + radiusY * 0.7);
      ctx.strokeStyle = guideColor;
      ctx.stroke();

      // Determina el color y el texto de feedback
      let strokeColor = '#e74c3c'; // Rojo
      let text = 'No se detecta rostro';

      if (feedbackState.detected) {
        strokeColor = '#f39c12'; // Naranja
        text = 'Centra tu rostro en el óvalo';
        if (!this.newUserName) {
          text = 'Introduce un nombre para empezar';
        }
        if (feedbackState.centered) {
          strokeColor = '#f1c40f'; // Amarillo
          text = 'Acerca o aleja el rostro';
          if (feedbackState.sizeOk) {
            strokeColor = '#2ecc71'; // Verde
            text = 'Mantén una expresión neutra';
            if (feedbackState.neutralExpression) {
              text = this.isCountingDown ? `Mantén la pose... ${this.registrationCountdown}` : '¡Perfecto! Mantén la posición';
            }
          }
        }
      }

      // Dibuja el óvalo con el color de feedback
      ctx.beginPath();
      ctx.ellipse(centerX, centerY, radiusX, radiusY, 0, 0, 2 * Math.PI);
      ctx.lineWidth = 5;
      ctx.strokeStyle = strokeColor;
      ctx.stroke();

      // Dibuja los landmarks si están disponibles
      if (feedbackState.landmarks) {
        const drawOptions = {
          drawLines: true,
          color: strokeColor,
          lineWidth: 2,
        };
        faceapi.draw.drawFaceLandmarks(canvas, feedbackState.landmarks, drawOptions);
      }

      // Dibuja el texto de feedback
      ctx.font = '20px Arial';
      ctx.fillStyle = strokeColor;
      ctx.textAlign = 'center';
      ctx.fillText(text, centerX, centerY + radiusY + 40);
    },

    saveRegisteredUsers() {
      localStorage.setItem('faceio_registered_users', JSON.stringify(this.registeredUsers));
      console.log('Usuarios guardados en localStorage.');
    },

    loadRegisteredUsers() {
      const usersJson = localStorage.getItem('faceio_registered_users');
      if (usersJson) {
        const users = JSON.parse(usersJson);
        // Los descriptores se guardan como objetos, hay que convertirlos de nuevo a Float32Array
        this.registeredUsers = users.map(user => ({
          name: user.name,
          descriptor: new Float32Array(Object.values(user.descriptor))
        }));
        this.updateFaceMatcher();
        console.log('Usuarios cargados desde localStorage.');
      }
    },

    clearRegisteredUsers() {
      this.registeredUsers = [];
      this.faceMatcher = null;
      this.welcomedUsers.clear();
      localStorage.removeItem('faceio_registered_users');
      this.feedback = 'Todos los registros han sido eliminados.';
    },

    updateFaceMatcher() {
      if (this.registeredUsers.length === 0) return;

      const labeledFaceDescriptors = this.registeredUsers.map(
        user => new faceapi.LabeledFaceDescriptors(user.name, [user.descriptor])
      );

      this.faceMatcher = new faceapi.FaceMatcher(labeledFaceDescriptors, 0.5); // 0.5 es el umbral de distancia
    },

    async startRecognition() {
      if (this.registeredUsers.length === 0) {
        this.feedback = 'No hay nadie registrado. Registra a alguien primero.';
        return;
      }

      this.stopSingleCamera('videoRegister'); // Apagamos la cámara de registro
      this.stopRegistrationFeedbackLoop(); // Asegurar que el bucle de feedback se detenga

      await this.startCamera('videoRecognize');
      const video = this.$refs.videoRecognize;
      const canvas = this.$refs.canvasRecognize;
      const displaySize = { width: video.width, height: video.height };
      faceapi.matchDimensions(canvas, displaySize);

      this.recognitionInterval = setInterval(async () => {
        const detections = await faceapi.detectAllFaces(video).withFaceLandmarks().withFaceDescriptors();
        const resizedDetections = faceapi.resizeResults(detections, displaySize);

        canvas.getContext('2d').clearRect(0, 0, canvas.width, canvas.height);

        if (this.faceMatcher) {
          const currentFacesInFrame = new Set();
          const results = resizedDetections.map(d => this.faceMatcher.findBestMatch(d.descriptor));

          results.forEach((result, i) => {
            const box = resizedDetections[i].detection.box;
            const isKnown = result.label !== 'unknown';
            
            // Convertir la distancia (0 a 1, donde 0 es mejor) a un porcentaje de confianza
            const confidence = Math.round((1 - result.distance) * 100);
            const drawLabel = isKnown ? `${result.label} (${confidence}%)` : 'Desconocido';
            const boxColor = isKnown ? '#42b983' : '#e74c3c'; // Verde para conocido, rojo para desconocido

            const drawBox = new faceapi.draw.DrawBox(box, { label: drawLabel, boxColor });
            drawBox.draw(canvas);

            const personName = result.label;
            if (personName !== 'unknown') {
              currentFacesInFrame.add(personName);
              // Si es la primera vez que vemos a este usuario en la sesión, le damos la bienvenida.
              if (!this.welcomedUsers.has(personName)) {
                alert(`Bienvenido ${personName}`);
                this.welcomedUsers.add(personName);
              }
            }
          });

          // Comprueba si algún usuario bienvenido ha salido del cuadro para resetear su estado.
          this.welcomedUsers.forEach(welcomedUser => {
            if (!currentFacesInFrame.has(welcomedUser)) {
              this.welcomedUsers.delete(welcomedUser);
              console.log(`${welcomedUser} ha salido del cuadro. Se le puede dar la bienvenida de nuevo.`);
            }
          });
        }
      }, 100); // Cada 100ms
    },

    stopRecognition() {
      if (this.recognitionInterval) {
        clearInterval(this.recognitionInterval);
        this.recognitionInterval = null;
      }
      this.stopSingleCamera('videoRecognize'); // Apagamos la cámara de reconocimiento
      this.startCamera('videoRegister'); // Volvemos a encender la de registro
      if (!this.loading) {
        this.startRegistrationFeedbackLoop(); // Reiniciar el bucle de feedback al volver al registro
      }

      this.welcomedUsers.clear(); // Limpiamos el registro de bienvenidas al detener.
      const canvas = this.$refs.canvasRecognize;
      if (canvas) {
        canvas.getContext('2d').clearRect(0, 0, canvas.width, canvas.height);
      }
    },
  }
}
</script>

<style>
#app {
  display: flex;
  flex-direction: column;
  align-items: center;
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 20px;
}

.main-content {
  display: flex;
  gap: 2rem;
  justify-content: center;
  flex-wrap: wrap;
}

.section {
  border: 1px solid #ccc;
  padding: 20px;
  border-radius: 8px;
  min-width: 680px;
}

.instructions {
  font-size: 0.9rem;
  color: #666;
}

.camera-container {
  position: relative;
  width: 640px;
  height: 480px;
  margin: 0 auto;
}

.camera-video {
  margin: 0 auto;
}

video, canvas {
  position: absolute;
  top: 0;
  left: 0;
}

.controls {
  margin-top: 10px;
}

.controls input {
  padding: 8px;
  margin-right: 10px;
}

.feedback {
  margin-top: 10px;
  color: #42b983;
  font-weight: bold;
}

.loading-overlay {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: rgba(255, 255, 255, 0.8);
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  z-index: 1000;
}

.spinner {
  border: 8px solid #f3f3f3;
  border-top: 8px solid #3498db;
  border-radius: 50%;
  width: 60px;
  height: 60px;
  animation: spin 2s linear infinite;
}

.registered-users {
  margin-top: 20px;
}

.clear-button {
  margin-top: 10px;
  background-color: #e74c3c;
  color: white;
  border: none;
  padding: 8px 15px;
}

@keyframes spin {
  0% { transform: rotate(0deg); }
  100% { transform: rotate(360deg); }
}

.feedback-canvas {
  position: absolute;
  top: 0;
  left: 0;
  pointer-events: none; /* Permite que los clics pasen a través del video si es necesario */
}
</style>
