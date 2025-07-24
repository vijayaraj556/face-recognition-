<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Face Detection App</title>
    <!-- Tailwind CSS CDN for basic styling -->
    <script src="https://cdn.tailwindcss.com"></script>
    <style>
        /* Custom CSS for better control and aesthetics */
        body {
            font-family: 'Inter', sans-serif;
            background-color: #1a202c; /* Dark background */
            display: flex;
            justify-content: center;
            align-items: center;
            min-height: 100vh;
            margin: 0;
            overflow: hidden; /* Prevent scrollbars */
        }

        .container {
            position: relative;
            display: flex;
            flex-direction: column;
            align-items: center;
            background-color: #2d3748; /* Slightly lighter dark background for container */
            padding: 2rem;
            border-radius: 1rem; /* Rounded corners */
            box-shadow: 0 10px 15px rgba(0, 0, 0, 0.3); /* Subtle shadow */
            max-width: 90vw; /* Responsive width */
            width: 800px; /* Max width for larger screens */
        }

        h1 {
            color: #e2e8f0; /* Light text color */
            margin-bottom: 1.5rem;
            font-size: 2.25rem; /* Larger heading */
            font-weight: 700; /* Bold */
            text-align: center;
        }

        #video-container {
            position: relative;
            width: 100%;
            max-width: 720px; /* Max width for video */
            border-radius: 0.75rem; /* Rounded corners for video container */
            overflow: hidden; /* Ensure video and canvas stay within bounds */
            box-shadow: 0 4px 6px rgba(0, 0, 0, 0.2);
        }

        video {
            width: 100%;
            height: auto;
            display: block;
            border-radius: 0.75rem; /* Rounded corners for video */
        }

        canvas {
            position: absolute;
            top: 0;
            left: 0;
            width: 100%;
            height: 100%;
            border-radius: 0.75rem; /* Rounded corners for canvas */
        }

        #loading-message {
            color: #a0aec0; /* Gray text for loading */
            margin-top: 1.5rem;
            font-size: 1.125rem;
            text-align: center;
        }

        #error-message {
            color: #fc8181; /* Red text for errors */
            margin-top: 1rem;
            font-size: 1.125rem;
            text-align: center;
        }

        /* Responsive adjustments */
        @media (max-width: 768px) {
            .container {
                padding: 1.5rem;
            }
            h1 {
                font-size: 1.75rem;
            }
        }
    </style>
</head>
<body>
    <div class="container">
        <h1>Real-time Face Detection</h1>
        <div id="video-container">
            <video id="videoInput" autoplay muted></video>
            <canvas id="overlayCanvas"></canvas>
        </div>
        <div id="loading-message" class="mt-4 text-gray-400">Loading face detection models...</div>
        <div id="error-message" class="mt-4 text-red-400 hidden"></div>
    </div>

    <!-- Face-API.js library -->
    <script src="https://cdn.jsdelivr.net/npm/face-api.js@0.22.2/dist/face-api.min.js"></script>
    <script defer>
        // Get references to the video and canvas elements
        const video = document.getElementById('videoInput');
        const canvas = document.getElementById('overlayCanvas');
        const loadingMessage = document.getElementById('loading-message');
        const errorMessage = document.getElementById('error-message');

        // Initialize face-api.js models path
        // Changed MODEL_URL to point to a reliable CDN path for face-api.js models
        const MODEL_URL = 'https://cdn.jsdelivr.net/gh/justadudewhohacks/face-api.js@0.22.2/weights';

        // Function to load all required face-api.js models
        async function loadModels() {
            try {
                // Load Tiny Face Detector model (fastest for real-time)
                await faceapi.nets.tinyFaceDetector.load(MODEL_URL);
                // Load Face Landmark model
                await faceapi.nets.faceLandmark68Net.load(MODEL_URL);
                // Load Face Expression model (optional, but good for more features)
                await faceapi.nets.faceExpressionNet.load(MODEL_URL);
                console.log('Models loaded successfully!');
                loadingMessage.textContent = 'Models loaded. Starting webcam...';
            } catch (error) {
                console.error('Failed to load models:', error);
                loadingMessage.style.display = 'none';
                errorMessage.textContent = 'Error loading models. Please check your internet connection or try again later.';
                errorMessage.classList.remove('hidden');
            }
        }

        // Function to start the webcam stream
        async function startWebcam() {
            try {
                // Request access to the user's webcam
                const stream = await navigator.mediaDevices.getUserMedia({ video: true });
                video.srcObject = stream;
                loadingMessage.textContent = 'Webcam started. Waiting for video to play...';
            } catch (error) {
                console.error('Error accessing webcam:', error);
                loadingMessage.style.display = 'none';
                if (error.name === 'NotAllowedError' || error.name === 'PermissionDeniedError' || error.name === 'NotReadableError') {
                    errorMessage.innerHTML = 'Webcam access denied or permission dismissed. <br>Please allow camera permissions in your browser settings and refresh the page.';
                } else if (error.name === 'NotFoundError') {
                    errorMessage.textContent = 'No webcam found. Please ensure a camera is connected.';
                } else {
                    errorMessage.textContent = 'Error starting webcam: ' + error.message;
                }
                errorMessage.classList.remove('hidden');
            }
        }

        // Event listener for when the video starts playing
        video.addEventListener('play', () => {
            // Hide loading message once video starts
            loadingMessage.style.display = 'none';
            errorMessage.classList.add('hidden'); // Hide any previous error messages

            // Set canvas dimensions to match the video dimensions
            const displaySize = { width: video.videoWidth, height: video.videoHeight };
            faceapi.matchDimensions(canvas, displaySize);

            // Start the detection loop
            setInterval(async () => {
                // Detect all faces in the video stream with landmarks and expressions
                const detections = await faceapi.detectAllFaces(
                    video,
                    new faceapi.TinyFaceDetectorOptions() // Use TinyFaceDetector for performance
                ).withFaceLandmarks().withFaceExpressions();

                // Resize the detections to fit the display size (canvas)
                const resizedDetections = faceapi.resizeResults(detections, displaySize);

                // Clear the canvas before drawing new detections
                canvas.getContext('2d').clearRect(0, 0, canvas.width, canvas.height);

                // Draw the bounding boxes around detected faces
                faceapi.draw.drawDetections(canvas, resizedDetections);

                // Draw the facial landmarks
                faceapi.draw.drawFaceLandmarks(canvas, resizedDetections);

                // Draw face expressions (e.g., happy, sad, neutral)
                faceapi.draw.drawFaceExpressions(canvas, resizedDetections);

            }, 100); // Run detection every 100 milliseconds
        });

        // Initialize the application
        async function init() {
            await loadModels(); // Load models first
            await startWebcam(); // Then start the webcam
        }

        // Call the initialization function when the window loads
        window.onload = init;

    </script>
</body>
</html>
