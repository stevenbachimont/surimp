<script>
  import { onMount } from 'svelte';

  let videoElement;
  let canvasElement;
  let context;
  let mediaStream;
  let isInverted = false;
  let videoDevices = [];
  let selectedDeviceId = '';

  // Ajout d'un canvas visible pour le flux vidéo
  let displayCanvas;
  let displayContext;
  let animationFrameId;

  let contrast = 100; // valeur par défaut (100 = pas de changement)
  let exposure = 0; // valeur par défaut (0 = pas de changement)

  let capturedPhotos = [];
  let selectedPhoto = null;
  let isCapturing = false;  // Pour éviter les captures multiples

  let exposures = []; // Tableau pour stocker les différentes expositions
  let exposureCount = 1; // Valeur par défaut à 1
  let currentExposureIndex = 0;

  // Ajouter une nouvelle variable pour stocker le canvas de prévisualisation
  let previewCanvas;
  let previewContext;

  let updateDisplay = () => {
    if (!videoElement || !displayCanvas) return;
    
    displayCanvas.width = videoElement.videoWidth;
    displayCanvas.height = videoElement.videoHeight;
    
    displayContext.drawImage(videoElement, 0, 0);
    
    // Si en mode multi-exposition et qu'il y a des expositions, superposer la prévisualisation
    if (exposureCount > 1 && exposures.length > 0 && previewCanvas) {
      displayContext.globalCompositeOperation = 'lighter';
      displayContext.drawImage(previewCanvas, 0, 0);
      displayContext.globalCompositeOperation = 'source-over';
    }
    
    const imageData = displayContext.getImageData(0, 0, displayCanvas.width, displayCanvas.height);
    const data = imageData.data;
    
    const contrastFactor = (contrast / 100);
    const exposureFactor = Math.pow(2, exposure); // Conversion exponentielle pour l'exposition

    for (let i = 0; i < data.length; i += 4) {
      // Appliquer l'exposition et le contraste
      for (let j = 0; j < 3; j++) {
        let color = data[i + j];
        
        // Appliquer l'exposition
        color *= exposureFactor;
        
        // Appliquer le contraste
        color = contrastFactor * (color - 128) + 128;
        
        // Limiter les valeurs entre 0 et 255
        data[i + j] = Math.min(255, Math.max(0, color));
      }
      
      // Appliquer l'inversion si activée
      if (isInverted) {
        data[i] = 255 - data[i];
        data[i + 1] = 255 - data[i + 1];
        data[i + 2] = 255 - data[i + 2];
      }
    }
    
    displayContext.putImageData(imageData, 0, 0);
    animationFrameId = requestAnimationFrame(updateDisplay);
  };

  async function getVideoDevices() {
    try {
      if (!navigator.mediaDevices) {
        console.warn('MediaDevices API non disponible. Vérifiez que vous utilisez HTTPS ou localhost.');
        return [];
      }
      const devices = await navigator.mediaDevices.enumerateDevices();
      return devices.filter(device => device.kind === 'videoinput');
    } catch (error) {
      console.error('Erreur lors de la récupération des périphériques vidéo : ', error);
      return [];
    }
  }

  async function updateVideoDevicesList() {
    videoDevices = await getVideoDevices();
    if (videoDevices && videoDevices.length > 0 && !selectedDeviceId) {
      selectedDeviceId = videoDevices[0].deviceId;
      await startCapture();
    }
  }

  async function startCapture() {
    try {
      if (mediaStream) {
        mediaStream.getTracks().forEach(track => track.stop());
      }
      if (animationFrameId) {
        cancelAnimationFrame(animationFrameId);
      }

      const constraints = {
        video: {
          deviceId: selectedDeviceId ? { exact: selectedDeviceId } : undefined,
          facingMode: selectedDeviceId ? undefined : "environment", // Utilise la caméra arrière par défaut
          width: { ideal: 1920 },
          height: { ideal: 1080 },
          // Ajout des capacités avancées pour les smartphones
          advanced: [{
            zoom: true,
            autoFocus: true,
            whiteBalance: true,
            exposureMode: "auto"
          }]
        }
      };
      
      mediaStream = await navigator.mediaDevices.getUserMedia(constraints);
      videoElement.srcObject = mediaStream;
      
      // Attendre que la vidéo soit prête
      await videoElement.play();
      updateDisplay();
    } catch (error) {
      console.error('Erreur lors de l\'accès à la webcam : ', error);
      // En cas d'erreur, essayer avec des contraintes plus basiques
      try {
        const basicConstraints = {
          video: {
            facingMode: "environment"
          }
        };
        mediaStream = await navigator.mediaDevices.getUserMedia(basicConstraints);
        videoElement.srcObject = mediaStream;
        await videoElement.play();
        updateDisplay();
      } catch (fallbackError) {
        console.error('Erreur même avec les contraintes basiques : ', fallbackError);
      }
    }
  }

  // Ajouter une fonction pour basculer entre les caméras
  async function switchCamera() {
    const currentTrack = mediaStream?.getVideoTracks()[0];
    const currentFacingMode = currentTrack?.getSettings()?.facingMode;
    
    try {
      const newFacingMode = currentFacingMode === "environment" ? "user" : "environment";
      const constraints = {
        video: {
          facingMode: newFacingMode,
          width: { ideal: 1920 },
          height: { ideal: 1080 }
        }
      };
      
      await startCapture();
    } catch (error) {
      console.error('Erreur lors du changement de caméra : ', error);
    }
  }

  function capturePhoto() {
    if (isCapturing) return;
    isCapturing = true;

    canvasElement.width = displayCanvas.width;
    canvasElement.height = displayCanvas.height;
    context.drawImage(displayCanvas, 0, 0);
    
    if (exposureCount > 1) {
      // Ajouter l'exposition au tableau
      exposures.push(canvasElement.toDataURL('image/jpeg'));
      currentExposureIndex++;

      // Mettre à jour la prévisualisation
      updatePreview();

      if (currentExposureIndex >= exposureCount) {
        // Fusionner toutes les expositions
        mergeExposures();
        // Réinitialiser
        currentExposureIndex = 0;
        exposures = [];
        // Réinitialiser la prévisualisation
        clearPreview();
      }
    } else {
      // Comportement normal pour une seule photo
    const imageBase64 = canvasElement.toDataURL('image/jpeg');
    capturedPhotos = [{ src: imageBase64, timestamp: Date.now() }, ...capturedPhotos];
    }

    setTimeout(() => {
      isCapturing = false;
    }, 500);
  }

  function mergeExposures() {
    const tempCanvas = document.createElement('canvas');
    tempCanvas.width = displayCanvas.width;
    tempCanvas.height = displayCanvas.height;
    const tempContext = tempCanvas.getContext('2d', { willReadFrequently: true });

    // Charger et fusionner toutes les expositions
    let loadedImages = 0;
    const images = [];

    // Créer une promesse pour chaque image
    const imagePromises = exposures.map(exposureData => {
      return new Promise((resolve) => {
        const img = new Image();
        img.onload = () => {
          images.push(img);
          resolve();
        };
        img.src = exposureData;
      });
    });

    // Attendre que toutes les images soient chargées
    Promise.all(imagePromises).then(() => {
      // Une fois toutes les images chargées, les fusionner dans l'ordre
      images.forEach((image, index) => {
        tempContext.globalCompositeOperation = 'lighter';
        tempContext.globalAlpha = 0.5;
        tempContext.drawImage(image, 0, 0);
      });

      // Sauvegarder le résultat
      const finalImage = tempCanvas.toDataURL('image/jpeg');
      capturedPhotos = [{ src: finalImage, timestamp: Date.now() }, ...capturedPhotos];
    });
  }

  function handleKeydown(event) {
    if (event.code === 'Space') {
      capturePhoto();
    }
  }

  // Ajouter un attribut willReadFrequently au canvas
  function initCanvas(canvas) {
    return canvas.getContext('2d', { willReadFrequently: true });
  }

  function openModal(photo) {
    selectedPhoto = photo;
  }

  function closeModal() {
    selectedPhoto = null;
  }

  function updatePreview() {
    if (!previewCanvas) {
      previewCanvas = document.createElement('canvas');
      previewCanvas.width = displayCanvas.width;
      previewCanvas.height = displayCanvas.height;
      previewContext = previewCanvas.getContext('2d', { willReadFrequently: true });
    }
    
    previewContext.clearRect(0, 0, previewCanvas.width, previewCanvas.height);
    
    exposures.forEach((exposureData, index) => {
      const img = new Image();
      img.onload = () => {
        previewContext.globalCompositeOperation = 'lighter';
        previewContext.globalAlpha = 0.5;
        previewContext.drawImage(img, 0, 0);
      };
      img.src = exposureData;
    });
  }

  function clearPreview() {
    if (previewCanvas) {
      previewContext.clearRect(0, 0, previewCanvas.width, previewCanvas.height);
    }
  }

  onMount(() => {
    // Initialiser les contextes avec willReadFrequently
    context = initCanvas(canvasElement);
    displayContext = initCanvas(displayCanvas);
    updateVideoDevicesList();
    
    return () => {
      if (animationFrameId) {
        cancelAnimationFrame(animationFrameId);
      }
      if (mediaStream) {
        mediaStream.getTracks().forEach(track => track.stop());
      }
    };
  });
</script>

<svelte:window on:keydown={handleKeydown}/>

<div class="container">
  <div class="video-container">
      <video
        bind:this={videoElement}
        autoplay
        playsinline
        class="hidden"
        muted
      >
        <track kind="captions" />
      </video>
    
    <canvas
      bind:this={displayCanvas}
      class="display-canvas"
    ></canvas>
    
    <canvas
      bind:this={canvasElement}
      class="hidden"
    ></canvas>
  </div>

  <div class="controls">
    <div class="controls-left">
      <div class="slider-control">
            <input 
          type="range" 
          id="exposure" 
          min="-2" 
          max="2" 
          step="0.1"
          bind:value={exposure}
        >
        <label for="exposure">EXP {exposure > 0 ? '+' : ''}{exposure}</label>
        </div>

      <div class="exposure-control">
        <input 
          type="number" 
          id="exposureCount" 
          min="1" 
          max="10" 
          bind:value={exposureCount}
        >
        {#if exposureCount > 1 && currentExposureIndex > 0}
          <p>Vue {currentExposureIndex + 1} / {exposureCount}</p>
    {/if}
      </div>
    </div>

    <div class="camera-controls">
      <button 
        class="switch-camera" 
        on:click={switchCamera}
        aria-label="Changer de caméra"
      >
        📷
      </button>
      
      <div class="capture-button-container">
        <button 
          class="capture-button" 
          on:click={capturePhoto}
          aria-label="Prendre une photo"
        >
          {#if exposureCount > 1 && currentExposureIndex > 0}
            <div class="exposure-counter">{currentExposureIndex + 1}/{exposureCount}</div>
          {/if}
        </button>
      </div>
    </div>

    <div class="controls-right">
      <div class="slider-control">
        <input 
          type="range" 
          id="contrast" 
          min="0" 
          max="200" 
          bind:value={contrast}
        >
        <label for="contrast">CON {contrast}%</label>
      </div>

      <button 
        class={isInverted ? 'active' : ''} 
        on:click={() => isInverted = !isInverted}
      >
        {isInverted ? "-" : "+"}
      </button>
    </div>
  </div>

  {#if capturedPhotos.length > 0}
    <div class="last-photo">
      <div 
        class="photo-item" 
        on:click={() => openModal(capturedPhotos[0])}
        on:keydown={(e) => e.key === 'Enter' && openModal(capturedPhotos[0])}
        tabindex="0"
        role="button"
      >
        <img src={capturedPhotos[0].src} alt="Dernière prise" />
      </div>
    </div>
  {/if}

  {#if selectedPhoto}
    <div 
      class="modal-overlay" 
      on:click={closeModal} 
      on:keydown={(e) => e.key === 'Escape' && closeModal()}
      role="button"
      tabindex="0"
    >
      <div class="modal-content">
        <img src={selectedPhoto.src} alt="Vue agrandie" />
      </div>
    </div>
  {/if}
</div>

<style>
  :global(body) {
    margin: 0;
    padding: 0;
    overflow-y: auto;
    min-height: 100vh;
    background: #000000;
  }

  .container {
    display: flex;
    flex-direction: column;
    align-items: center;
    padding: 0.5rem;
    height: 100vh;
    width: 100%;
    position: fixed;
    top: 0;
    left: 0;
    padding-bottom: 160px; /* Pour accommoder le bouton de capture */
  }

  .video-container {
    position: relative;
    width: 100vw;
      height: 100vh;
  }

  video {
    width: 100%;
    height: auto;
  }

  .controls {
    position: fixed;
    bottom: 20px;
    left: 0;
    right: 0;
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 1rem;
    z-index: 89;
  }

  .controls-left, .controls-right {
    position: fixed;
    right: 1rem;
    display: flex;
    flex-direction: row;
    gap: 0.5rem;
    background: rgba(0, 0, 0, 0.5);
    padding: 0.5rem;
    border-radius: 12px;
    height: 40px;
    align-items: center;
    width: 200px;
    justify-content: center;
  }

  .controls-left {
    bottom: 20px;
  }

  .controls-right {
    bottom: calc(20px + 50px);
  }

  .exposure-control {
    display: flex;
    flex-direction: row;
    align-items: center;
    gap: 0.5rem;
    width: 100%;
    justify-content: space-between;
  }

  .exposure-control input[type="number"] {
    width: 2.5rem;
    text-align: center;
    background: rgba(255, 255, 255, 0.2);
    border: 1px solid rgba(255, 255, 255, 0.3);
    color: white;
    border-radius: 8px;
    padding: 0.2rem;
    height: 25px;
  }

  .display-canvas {
    width: 100%;
    height: 100%;
    object-fit: cover;
  }

  .hidden {
    display: none;
  }

  button {
    padding: 0.4rem 0.8rem;
    background-color: #404040;
    color: white;
    border: none;
    border-radius: 8px;
    cursor: pointer;
    font-weight: 500;
    transition: all 0.2s ease;
    height: 25px;
    display: flex;
    align-items: center;
    justify-content: center;
  }

  button:hover {
    background-color: #505050;
  }

  @media (orientation: portrait) {
    .video-container {
      height: 100vh;
      width: 100vw;
    }

    .display-canvas {
      width: 100%;
      height: 100%;
      object-fit: cover;
    }

    .controls {
      padding: 0 1rem;
    }

    button {
      padding: 0.4rem 0.8rem;
      font-size: 0.9rem;
      min-height: 36px;
    }

    .slider-control {
      padding: 0.25rem;
      font-size: 0.9rem;
    }

    .slider-control label {
      font-size: 0.85rem;
    }

    .capture-button {
      width: 80px;
      height: 80px;
    }

    .capture-button::after {
      width: 64px;
      height: 64px;
    }

    .capture-button:active::after {
      width: 60px;
      height: 60px;
    }
  }

  input[type="range"] {
    accent-color: #8b98b9;
  }

  .last-photo {
    position: fixed;
    bottom: 20px;
    left: 1rem;
    width: auto; /* Modifier pour ne pas prendre toute la largeur */
    z-index: 100;
    display: flex;
  }

  .photo-item {
    position: relative;
    width: 60px;
    height: 60px;
    overflow: hidden;
    border-radius: 4px;
    cursor: pointer;
    transition: transform 0.2s ease;
    border: 2px solid rgba(255, 255, 255, 0.3);
  }

  .photo-item:active {
    transform: scale(0.95);
  }

  .photo-item img {
    width: 100%;
    height: 100%;
    object-fit: cover;
  }

  .camera-controls {
    position: fixed;
    bottom: 20px;
    left: 0;
    right: 0;
    display: flex;
    align-items: center;
    justify-content: center;
    z-index: 91;
  }

  .capture-button-container {
    position: relative;
    margin: 0 auto;
  }

  .capture-button {
    width: 70px;
    height: 70px;
    border-radius: 50%;
    background-color: rgba(255, 255, 255, 0.2);
    border: 4px solid #fff;
    padding: 0;
    cursor: pointer;
    position: relative;
    transition: all 0.3s ease;
  }

  .capture-button::after {
    content: '';
    position: absolute;
    top: 50%;
    left: 50%;
    transform: translate(-50%, -50%);
    width: 54px;
    height: 54px;
    background-color: #ff3b30;
    border-radius: 50%;
    transition: all 0.2s ease;
  }

  .capture-button:active::after {
    width: 50px;
    height: 50px;
  }

  .exposure-counter {
    position: absolute;
    top: -30px;
    left: 50%;
    transform: translateX(-50%);
    background-color: rgba(0, 0, 0, 0.6);
    color: white;
    padding: 4px 8px;
    border-radius: 12px;
    font-size: 12px;
    z-index: 2;
  }

  .slider-control {
    display: flex;
    flex-direction: row;
    align-items: center;
    gap: 0.3rem;
    width: 100%;
    justify-content: space-between;
  }
  
  .slider-control input[type="range"] {
    width: 120px;
    appearance: none;
    -webkit-appearance: none;
    background: rgba(255, 255, 255, 0.2);
    border-radius: 8px;
    height: 4px;
  }
  
  .slider-control input[type="range"]::-webkit-slider-thumb {
    -webkit-appearance: none;
    width: 16px;
    height: 16px;
    background: white;
    border-radius: 50%;
    cursor: pointer;
  }
  
  .slider-control label {
    color: white;
    font-size: 0.8rem;
    text-transform: uppercase;
    text-align: center;
  }

  .modal-overlay {
    position: fixed;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    background: rgba(0, 0, 0, 0.9);
    display: flex;
    align-items: center;
    justify-content: center;
    z-index: 1000;
    cursor: pointer;
  }

  .modal-content {
    position: relative;
    max-width: 90vw;
    max-height: 90vh;
  }

  .modal-content img {
    max-width: 100%;
    max-height: 90vh;
    object-fit: contain;
  }

  .switch-camera {
    position: absolute;
    left: calc(50% - 100px); /* Ajustez cette valeur selon vos besoins */
    bottom: 10px;
    background-color: rgba(255, 255, 255, 0.2);
    border: none;
    border-radius: 50%;
    width: 50px;
    height: 50px;
    display: flex;
    align-items: center;
    justify-content: center;
    font-size: 1.2rem;
    cursor: pointer;
  }

  .switch-camera:active {
    background-color: rgba(255, 255, 255, 0.3);
    transform: scale(0.95);
  }
</style> 