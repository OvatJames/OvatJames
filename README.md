<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>Video Call with Background Replacement</title>
  <style>
    @import url('https://fonts.googleapis.com/css2?family=Inter:wght@400;700&display=swap');
    body {
      margin: 0;
      font-family: 'Inter', sans-serif;
      background: #121212;
      color: #eee;
      display: flex;
      flex-direction: column;
      align-items: center;
      min-height: 100vh;
      padding: 1rem;
      box-sizing: border-box;
    }
    h1 {
      margin-bottom: 0.5rem;
      font-weight: 700;
    }
    #container {
      display: flex;
      gap: 1rem;
      margin-top: 1rem;
      flex-wrap: wrap;
      justify-content: center;
      max-width: 1000px;
      width: 100%;
    }
    .video-wrapper {
      position: relative;
      width: 320px;
      height: 240px;
      background: #222;
      border-radius: 12px;
      overflow: hidden;
      box-shadow: 0 4px 12px rgba(0,0,0,0.6);
    }
    video, canvas {
      position: absolute;
      width: 100%;
      height: 100%;
      object-fit: cover;
      top: 0;
      left: 0;
    }
    #background-selection {
      margin-top: 1rem;
      display: flex;
      gap: 1rem;
      flex-wrap: wrap;
      justify-content: center;
      max-width: 1000px;
    }
    .bg-option {
      width: 120px;
      height: 90px;
      border-radius: 8px;
      overflow: hidden;
      border: 3px solid transparent;
      cursor: pointer;
      box-shadow: 0 2px 8px rgba(0,0,0,0.5);
      transition: border-color 0.3s ease;
      background-size: cover;
      background-position: center;
    }
    .bg-option.selected {
      border-color: #2ee59d;
    }
    #call-controls {
      margin-top: 1rem;
      display: flex;
      gap: 1rem;
      justify-content: center;
      flex-wrap: wrap;
    }
    button {
      background-color: #2ee59d;
      border: none;
      color: #121212;
      font-size: 16px;
      font-weight: 600;
      padding: 0.5rem 1rem;
      border-radius: 9999px;
      cursor: pointer;
      box-shadow: 0 4px 12px rgba(46,229,157,0.4);
      transition: background-color 0.3s ease;
    }
    button:hover:not(:disabled) {
      background-color: #27bf88;
    }
    button:disabled {
      background-color: #555;
      cursor: default;
      box-shadow: none;
    }
    #linkContainer {
      margin-top: 1rem;
      max-width: 600px;
      word-break: break-all;
      text-align: center;
    }
    #info {
      margin-top: 1rem;
      font-size: 0.9rem;
      color: #bbb;
      max-width: 600px;
      text-align: center;
    }
    input[type="file"] {
      display: none;
    }
    label.upload-btn {
      background-color: #0bb;
      color: white;
      padding: 0.4rem 1rem;
      border-radius: 9999px;
      font-weight: 600;
      cursor: pointer;
      box-shadow: 0 4px 12px rgba(11,187,187,0.4);
      transition: background-color 0.3s ease;
      user-select: none;
    }
    label.upload-btn:hover {
      background-color: #099;
    }
  </style>
</head>
<body>
  <h1>Video Call with Background Replacement</h1>
  <p>Open this page in two tabs or devices. Share the call link from one tab to the other to start the video call.</p>
  
  <div id="container">
    <div class="video-wrapper" id="localWrapper">
      <canvas id="localCanvas"></canvas>
      <video id="localVideo" autoplay muted playsinline style="display:none;"></video>
    </div>
    <div class="video-wrapper" id="remoteWrapper">
      <canvas id="remoteCanvas"></canvas>
      <video id="remoteVideo" autoplay playsinline style="display:none;"></video>
    </div>
  </div>
  
  <div id="background-selection" aria-label="Background selection">
    <!-- Background options will be added dynamically -->
  </div>
  
  <div id="call-controls">
    <button id="startCallBtn">Start Call</button>
    <button id="endCallBtn" disabled>End Call</button>
    <label for="bgUpload" class="upload-btn" title="Upload custom background image">Upload Background</label>
    <input type="file" id="bgUpload" accept="image/*" />
  </div>

  <div id="linkContainer"></div>
  <div id="info"></div>

  <!-- Libraries -->
  <script src="https://cdn.jsdelivr.net/npm/@tensorflow/tfjs@4.8.0/dist/tf.min.js"></script>
  <script src="https://cdn.jsdelivr.net/npm/@tensorflow-models/body-pix@2.2.0/dist/body-pix.min.js"></script>
  <script src="https://unpkg.com/peerjs@1.4.7/dist/peerjs.min.js"></script>

  <script>
    (async () => {
      const localVideo = document.getElementById('localVideo');
      const remoteVideo = document.getElementById('remoteVideo');
      const localCanvas = document.getElementById('localCanvas');
      const remoteCanvas = document.getElementById('remoteCanvas');
      const startCallBtn = document.getElementById('startCallBtn');
      const endCallBtn = document.getElementById('endCallBtn');
      const linkContainer = document.getElementById('linkContainer');
      const info = document.getElementById('info');
      const bgSelectionDiv = document.getElementById('background-selection');
      const bgUploadInput = document.getElementById('bgUpload');
      
      const localCtx = localCanvas.getContext('2d');
      const remoteCtx = remoteCanvas.getContext('2d');
      
      let localStream = null;
      let peer = null;
      let call = null;
      let bodyPixNet = null;
      let localBackgroundImage = null;
      let remoteBackgroundImage = null;
      let segmentLocal = null;  // seg for local user
      let segmentRemote = null; // seg for remote video
      
      // Background images choices - URLs from unsplash or solid colors
      const backgrounds = [
        {name: 'Beach', url: 'https://images.unsplash.com/photo-1506744038136-46273834b3fb?auto=format&fit=crop&w=320&q=80'},
        {name: 'City', url: 'https://images.unsplash.com/photo-1494526585095-c41746248156?auto=format&fit=crop&w=320&q=80'},
        {name: 'Mountain', url: 'https://images.unsplash.com/photo-1501785888041-af3ef285b470?auto=format&fit=crop&w=320&q=80'},
        {name: 'Space', url: 'https://images.unsplash.com/photo-1446776811953-b23d57bd21aa?auto=format&fit=crop&w=320&q=80'},
        {name: 'Solid Teal', url: 'data:image/svg+xml;base64,' + btoa('<svg xmlns="http://www.w3.org/2000/svg" width="320" height="240"><rect width="320" height="240" fill="#1abc9c"/></svg>')}
      ];
      
      // Create background thumbnails
      backgrounds.forEach((bg, i) => {
        const div = document.createElement('div');
        div.className = 'bg-option';
        div.title = bg.name;
        if (bg.url.startsWith('data:image/svg+xml;base64,')) {
          div.style.background = bg.url;
        } else {
          div.style.backgroundImage = `url(${bg.url})`;
        }
        if (i === 0) {
          div.classList.add('selected');
          loadImage(bg.url).then(img => localBackgroundImage = img);
        }
        div.addEventListener('click', async () => {
          document.querySelectorAll('.bg-option').forEach(el => el.classList.remove('selected'));
          div.classList.add('selected');
          localBackgroundImage = await loadImage(bg.url);
          info.textContent = '';
        });
        bgSelectionDiv.appendChild(div);
      });
      
      // Load image utility
      function loadImage(src) {
        return new Promise((resolve, reject) => {
          const img = new Image();
          img.crossOrigin = "anonymous";
          img.onload = () => resolve(img);
          img.onerror = reject;
          img.src = src;
        });
      }
      
      // Handle file upload for background
      bgUploadInput.addEventListener('change', async e => {
        const file = e.target.files[0];
        if (!file) return;
        if (!file.type.startsWith('image/')) {
          info.textContent = 'Please upload a valid image file.';
          return;
        }
        const url = URL.createObjectURL(file);
        try {
          const img = await loadImage(url);
          localBackgroundImage = img;
          document.querySelectorAll('.bg-option').forEach(el => el.classList.remove('selected'));
          info.textContent = 'Custom background set.';
        } catch {
          info.textContent = 'Failed to load image.';
        }
      });
      
      // Load BodyPix model
      info.textContent = 'Loading BodyPix model...';
      bodyPixNet = await bodyPix.load({
        architecture: 'MobileNetV1',
        outputStride: 16,
        multiplier: 0.75,
        quantBytes: 2,
      });
      info.textContent = 'Model loaded. Start your call.';
      
      // Get user media
      async function getLocalStream() {
        try {
          const stream = await navigator.mediaDevices.getUserMedia({ video: { width: 640, height: 480 }, audio: true });
          localVideo.srcObject = stream;
          localStream = stream;
          // Set canvas size
          localCanvas.width = 640;
          localCanvas.height = 480;
          remoteCanvas.width = 640;
          remoteCanvas.height = 480;
          return stream;
        } catch (err) {
          alert('Error accessing camera/microphone: ' + err.message);
          throw err;
        }
      }
      
      // Segment and draw function for background replacement
      async function segmentAndDraw(video, canvasCtx, backgroundImage, width, height) {
        if (!bodyPixNet) return;
        try {
          // Segment person in the video frame
          const segmentation = await bodyPixNet.segmentPerson(video, {
            flipHorizontal: true,
            internalResolution: 'medium',
            segmentationThreshold: 0.7,
          });
          // Create mask
          const maskBackground = bodyPix.toMask(segmentation, {r:0, g:0, b:0, a:0}, {r:0,g:0,b:0,a:255});
          canvasCtx.clearRect(0, 0, width, height);
          // Draw background first
          if (backgroundImage) {
            canvasCtx.drawImage(backgroundImage, 0, 0, width, height);
          } else {
            // Fill with dark if no background
            canvasCtx.fillStyle = '#222';
            canvasCtx.fillRect(0, 0, width, height);
          }
          // Draw person from video using mask
          canvasCtx.save();
          canvasCtx.globalCompositeOperation = 'destination-in';
          canvasCtx.drawImage(video, 0, 0, width, height);
          canvasCtx.restore();

          // Draw person alpha mask with original video
          canvasCtx.globalCompositeOperation = 'source-in';
          canvasCtx.drawImage(video, 0, 0, width, height);
          canvasCtx.globalCompositeOperation = 'source-over';

          // Alternatively, draw mask to exclude background
          // But to simplify, just draw video onto background with mask composed.

        } catch (err) {
          console.warn('Segmentation error', err);
        }
      }
      
      // Improved segmentation and compositing based on examples: we can do temporal smoothing and better blending:
      async function drawFrame(video, ctx, bgImage, width, height) {
        if (!bodyPixNet) return;
        const segmentation = await bodyPixNet.segmentPerson(video, {
          flipHorizontal: true,
          internalResolution: 'medium',
          segmentationThreshold: 0.7,
          maxDetections: 1,
        });

        // Create person mask for alpha blending
        const foregroundColor = {r:0, g:0, b:0, a:0};
        const backgroundColor = {r:0, g:0, b:0, a:255};
        const mask = bodyPix.toMask(segmentation, foregroundColor, backgroundColor);

        ctx.clearRect(0, 0, width, height);

        // Draw background
        if (bgImage) {
          ctx.drawImage(bgImage, 0, 0, width, height);
        } else {
          ctx.fillStyle = '#222';
          ctx.fillRect(0, 0, width, height);
        }

        // Draw video frame
        ctx.drawImage(video, 0, 0, width, height);

        // Apply mask to hide background
        ctx.globalCompositeOperation = 'destination-in';
        ctx.putImageData(mask, 0, 0);
        ctx.globalCompositeOperation = 'source-over';
        // This blends to show only the person on top of background

      }
      
      // Local loop to draw segmented video with background
      async function localRenderLoop() {
        if (!localVideo.paused && !localVideo.ended) {
          await drawFrame(localVideo, localCtx, localBackgroundImage, localCanvas.width, localCanvas.height);
        }
        requestAnimationFrame(localRenderLoop);
      }

      // Remote stream processing: We can't directly access remoteVideo pixels, but we can draw the remote video over canvas as is (or implement segmentation on remote side - but here we just draw the remote as is)
      function remoteRenderLoop() {
        if (!remoteVideo.paused && !remoteVideo.ended) {
          remoteCtx.drawImage(remoteVideo, 0, 0, remoteCanvas.width, remoteCanvas.height);
        }
        requestAnimationFrame(remoteRenderLoop);
      }
      
      // Establish PeerJS connection and call
      startCallBtn.onclick = async () => {
        startCallBtn.disabled = true;
        info.textContent = 'Accessing camera and connecting...';
        await getLocalStream();
        await localVideo.play();
        localRenderLoop();
        remoteRenderLoop();

        // Create Peer with random id, let user share id to connect
        peer = new Peer(undefined, {
          debug: 2
        });

        peer.on('open', id => {
          info.textContent = `Share this ID with your friend to call: ${id}`;
          const inputId = prompt('Enter remote peer ID to call (or cancel to wait for calls):');
          if (inputId) {
            const outgoingCall = peer.call(inputId, localStream);
            setupCall(outgoingCall);
          } else {
            info.textContent = `Waiting for incoming calls. Your ID: ${id}`;
          }
        });

        peer.on('call', incomingCall => {
          if (confirm('Incoming call. Accept?')) {
            incomingCall.answer(localStream);
            setupCall(incomingCall);
          } else {
            incomingCall.close();
            info.textContent = 'Call rejected.';
          }
        });

        peer.on('error', err => {
          info.textContent = 'Peer error: ' + err;
          startCallBtn.disabled = false;
        });

        endCallBtn.disabled = false;
      };

      function setupCall(c) {
        call = c;
        call.on('stream', remoteStream => {
          remoteVideo.srcObject = remoteStream;
          remoteVideo.play();
          info.textContent = 'Call established.';
        });
        call.on('close', () => {
          info.textContent = 'Call ended.';
          remoteVideo.srcObject = null;
          endCallBtn.disabled = true;
          startCallBtn.disabled = false;
        });
        call.on('error', err => {
          info.textContent = 'Call error: ' + err;
          call.close();
          endCallBtn.disabled = true;
          startCallBtn.disabled = false;
        });
      }

      endCallBtn.onclick = () => {
        if (call) call.close();
        if (peer) peer.destroy();
        remoteVideo.srcObject = null;
        startCallBtn.disabled = false;
        endCallBtn.disabled = true;
        info.textContent = 'Call ended.';
      };
    })();
  </script>
</body>
</html>

