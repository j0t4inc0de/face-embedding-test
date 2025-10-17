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
        <p class="instructions">Coloca tu cara frente a la cámara y pulsa el botón.</p>
        <div class="camera-container">
          <video ref="videoRegister" width="640" height="480" autoplay muted></video>
          <canvas ref="canvasRegister" width="640" height="480"></canvas>
        </div>
        <div class="controls">
          <input type="text" v-model="newUserName" placeholder="Nombre de la persona" />
          <button @click="captureAndRegister" :disabled="!newUserName">Capturar y Registrar</button>
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
  },
  methods: {
    async loadModels() {
      const MODEL_URL = '/models';
      try {
        await Promise.all([
          faceapi.nets.ssdMobilenetv1.loadFromUri(MODEL_URL),
          faceapi.nets.faceLandmark68Net.loadFromUri(MODEL_URL),
          faceapi.nets.faceRecognitionNet.loadFromUri(MODEL_URL),
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
    },

    stopCamera() {
      this.stopSingleCamera('videoRegister');
      this.stopSingleCamera('videoRecognize');
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
            const drawBox = new faceapi.draw.DrawBox(box, { label: result.toString() });
            drawBox.draw(canvas);

            const label = result.label;
            if (label !== 'unknown') {
              currentFacesInFrame.add(label);
              // Si es la primera vez que vemos a este usuario en la sesión, le damos la bienvenida.
              if (!this.welcomedUsers.has(label)) {
                alert(`Bienvenido ${label}`);
                this.welcomedUsers.add(label);
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
</style>
