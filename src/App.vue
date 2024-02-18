<template>
  <div
    class="max-w-7xl grid md:grid-cols-2 grid-cols-1 gap-6 py-10 mx-auto px-4"
  >
    <div class="space-y-6">
      <h1 class="text-2xl font-semibold col-span-2">
        Driver's License Scanner
      </h1>
      <div class="flex flex-col items-start gap-2">
        <button
          class="bg-gray-100 border border-slate-700 px-5 py-2 rounded-lg text-slate-600"
          @click="openWebcam"
          :disabled="showWebcam"
        >
          Open Webcam
        </button>
        <button
          class="bg-gray-100 border border-slate-700 px-5 py-2 rounded-lg text-slate-600"
          @click="captureScreen"
          :disabled="!showWebcam"
        >
          Capture Screen
        </button>
        <button
          class="bg-gray-100 border border-slate-700 px-5 py-2 rounded-lg text-slate-600"
          @click="startScan"
          :disabled="!showImage"
        >
          Start Scan
        </button>
      </div>
      <div class="w-full relative">
        <video
          v-if="!showImage"
          ref="video"
          id="video"
          v-show="showWebcam"
          autoplay
        ></video>
        <img v-show="showImage" :src="capturedImage" alt="Captured Screen" />
        <button
          class="bg-gray-100 absolute bottom-2 left-2 border border-slate-700 px-5 py-2 rounded-lg text-slate-600"
          @click="toggleCameraMode"
        >
          Switch to back {{ toggleCamera }}
        </button>
      </div>
    </div>
    <div>
      <div id="dataDisplay" class="space-y-2">
        <h2 class="text-xl font-semibold">Extracted Data:</h2>
        <p class="font-semibold text-slate-800" v-if="extractedData.fullName">
          Full Name:
          <span class="font-normal">{{ extractedData.fullName }}</span>
        </p>
        <p class="font-semibold text-slate-800" v-if="extractedData.address">
          Address: <span class="font-normal"> {{ extractedData.address }}</span>
        </p>
        <p class="font-semibold text-slate-800" v-if="extractedData.issueDate">
          Issue Date:
          <span class="font-normal">{{ extractedData.issueDate }}</span>
        </p>
        <p
          class="font-semibold text-slate-800"
          v-if="extractedData.expirationDate"
        >
          Expiration Date:
          <span class="font-normal">{{ extractedData.expirationDate }}</span>
        </p>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref } from "vue";
import Tesseract from "tesseract.js";
import Swal from "sweetalert2";

const showWebcam = ref(false);
const showImage = ref(false);
const extractedData = ref({});
const capturedImage = ref("");

let stream;

const toggleCamera = ref(true);

const openWebcam = () => {
  const video = document.getElementById("video");
  if (!video) {
    console.error("Video element not found.");
    return;
  }

  navigator.mediaDevices
    .getUserMedia({
      video: {
        facingMode: {
          exact: toggleCamera.value ? "user" : "environment",
        },
      },
    })
    .then((str) => {
      video.srcObject = str;
      video.play();
      showWebcam.value = true;
      stream = str;
    })
    .catch((error) => {
      console.error("Error accessing webcam:", error);
      alert(
        "Error accessing webcam: Your device is not supported Another Camera"
      );
    });
};

const toggleCameraMode = () => {
  stopStreaming();
  toggleCamera.value = !toggleCamera.value;
  setTimeout(() => {
    openWebcam(); // Reopen the webcam with the updated camera mode
  }, 2000);
};

const captureScreen = () => {
  const video = document.getElementById("video");
  if (!video) {
    console.error("Video element not found.");
    return;
  }

  const canvas = document.createElement("canvas");
  const context = canvas.getContext("2d");
  canvas.width = video.videoWidth;
  canvas.height = video.videoHeight;
  context.drawImage(video, 0, 0, canvas.width, canvas.height);

  capturedImage.value = canvas.toDataURL();
  showImage.value = true;

  stopStreaming();
};

const sampleImage = "/did.PNG";

const stopStreaming = () => {
  if (stream) {
    const tracks = stream.getTracks();
    tracks.forEach((track) => {
      track.stop();
    });
  }
};

const parseData = (text) => {
  const data = {};

  // Define regular expressions for each field to match the corresponding data
  const fullNameRegex = /FN\s+(.*?)(?=\nLN|$)/s;
  const addressRegex = /LN\s+(.*?)(?=\nDL|$)/s;
  const issueDateRegex = /DL\s+([^\n]+)/s;
  const expirationDateRegex = /EXP\s+([^\n]+)/s;

  // Execute regular expressions on the text and extract the matched data
  const fullNameMatch = fullNameRegex.exec(text);
  const addressMatch = addressRegex.exec(text);
  const issueDateMatch = issueDateRegex.exec(text);
  const expirationDateMatch = expirationDateRegex.exec(text);

  // Check if a match was found for each field and store it in the data object
  if (fullNameMatch) {
    data.fullName = fullNameMatch[1].split("\n")[0];
  }

  if (addressMatch) {
    data.address = addressMatch[1].trim();
  }

  if (issueDateMatch) {
    data.issueDate = issueDateMatch[1].trim().replace(/[^0-9/]/g, "");
  }

  if (expirationDateMatch) {
    data.expirationDate = expirationDateMatch[1].trim().replace(/[^0-9/]/g, "");
  }

  return data;
};

const startScan = () => {
  Swal.fire({
    title: "Scanning...",
    allowEscapeKey: true,
  });
  Tesseract.recognize(capturedImage.value, "eng", {
    logger: (m) => console.log(m),
  })
    .then(({ data: { text } }) => {
      console.log("Extracted text:", text);
      const parsedData = parseData(text);
      // extractedData.value = { ...extractedData.value, ...parsedData };
      console.log(parsedData);
      extractedData.value = parsedData;
      showImage.value = false; // Hide the captured image after scan
      Swal.fire({
        title: "Scan Completed",
        allowEscapeKey: true,
      });
    })
    .catch((error) => {
      console.error("Error extracting text:", error);
    });
};
</script>
