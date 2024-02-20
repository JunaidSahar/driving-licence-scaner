<template>
  <div class="bg-zinc-900 h-screen">
    <div class="grid md:grid-cols-12 gap-6 mx-auto">
      <div class="space-y-4 col-span-3 bg-zinc-900 px-3 py-6 h-full">
        <h1 class="text-xl font-semibold text-zinc-200">
          Driver's License Scanner
        </h1>
        <div class="flex flex-col items-start gap-2">
          <button
            class="bg-zinc-900 flex items-center gap-2 hover:text-zinc-800 font-medium transition-all hover:bg-zinc-400 text-white border border-slate-700 px-5 py-2 rounded-lg"
            @click="openWebcam"
            :disabled="showWebcam"
          >
            <svg
              xmlns="http://www.w3.org/2000/svg"
              width="18"
              height="18"
              viewBox="0 0 24 24"
            >
              <path
                fill="none"
                stroke="currentColor"
                stroke-linecap="round"
                stroke-linejoin="round"
                stroke-width="1.5"
                d="M10 12V6h1m-1 6h1V6m-1 12v-3h1m0 0v3h-1M7 6v6m0 3v3m7-12v6m0 3v3m3-12v6m0 3v3M6 3H3v3m-1 6h20m-4-9h3v3M6 21H3v-3m15 3h3v-3"
              />
            </svg>
            Scan Front
          </button>
          <!-- <button
            class="bg-gray-100 border border-slate-700 px-5 py-2 rounded-lg text-slate-600"
            @click="startScan"
            :disabled="!showImage"
          >
            Start Scan
          </button> -->
        </div>
        <div
          class="space-y-4 bg-zinc-800 p-4 rounded-lg"
          v-if="overallExtractData.length"
        >
          <p class="text-zinc-100 font-semibold">
            Overall generated data from AI:
          </p>
          <p class="text-zinc-200 font-normal">{{ overallExtractData }}</p>
        </div>
      </div>
      <div class="col-span-9 bg-zinc-800 h-screen px-4 py-6">
        <p v-show="showWebcam" class="text-zinc-100 pb-3">
          Please refrain from scanning the card in a vertical orientation.
          <br />
          Ensure that the card is scanned horizontally to ensure accurate
          processing. <br />
          Thank you for your cooperation.
        </p>
        <div class="w-fit relative">
          <video
            v-if="!showImage"
            ref="video"
            id="video"
            v-show="showWebcam"
            autoplay
          ></video>
          <img v-show="showImage" :src="capturedImage" alt="Captured Screen" />
          <button
            class="bg-gray-100 absolute text-sm font-medium top-2 right-2 border border-slate-700 p-2 rounded-full text-slate-600"
            @click="toggleCameraMode"
            v-show="showWebcam"
          >
            <svg
              xmlns="http://www.w3.org/2000/svg"
              width="30"
              height="30"
              viewBox="0 0 24 24"
            >
              <g
                fill="none"
                stroke="currentColor"
                stroke-linecap="round"
                stroke-linejoin="round"
                stroke-width="2"
              >
                <path
                  d="M11 19H4a2 2 0 0 1-2-2V7a2 2 0 0 1 2-2h5m4 0h7a2 2 0 0 1 2 2v10a2 2 0 0 1-2 2h-5"
                />
                <circle cx="12" cy="12" r="3" />
                <path d="m18 22l-3-3l3-3M6 2l3 3l-3 3" />
              </g>
            </svg>
          </button>
          <button
            class="bg-gray-100 border border-slate-700 absolute right-3 top-1/2 -translate-y-1/2 p-2 rounded-full text-slate-600"
            @click="captureScreen"
            :disabled="!showWebcam"
            v-show="showWebcam"
          >
            <svg
              xmlns="http://www.w3.org/2000/svg"
              width="32"
              height="32"
              viewBox="0 0 24 24"
            >
              <path
                fill="none"
                stroke="currentColor"
                stroke-linecap="round"
                stroke-linejoin="round"
                stroke-width="2"
                d="M4 8V6a2 2 0 0 1 2-2h2M4 16v2a2 2 0 0 0 2 2h2m8-16h2a2 2 0 0 1 2 2v2m-4 12h2a2 2 0 0 0 2-2v-2M9 12a3 3 0 1 0 6 0a3 3 0 1 0-6 0"
              />
            </svg>
          </button>
        </div>
        <div
          id="dataDisplay"
          class="space-y-2 text-zinc-300"
          v-if="!showWebcam"
        >
          <h2 class="text-3xl pb-6 font-semibold text-zinc-200">
            Extracted Data:
          </h2>
          <p class="font-semibold">
            Full Name:
            <span v-if="extractedData.firsName" class="font-normal">{{
              extractedData.firsName + ` ` + extractedData.lastName
            }}</span>
          </p>
          <p class="font-semibold">
            Address:
            <span v-if="extractedData.address" class="font-normal">
              {{ extractedData.address }}</span
            >
          </p>
          <p class="font-semibold">
            DL Issuance:
            <span
              v-if="extractedData.driverLicenseNumber"
              class="font-normal"
              >{{ extractedData.driverLicenseNumber }}</span
            >
          </p>
          <p class="font-semibold">
            Expiration Date:
            <span v-if="extractedData.expirationDate" class="font-normal">{{
              extractedData.expirationDate
            }}</span>
          </p>
        </div>
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
const overallExtractData = ref("");

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

function preprocessImage(canvas) {
  const image = canvas
    .getContext("2d")
    .getImageData(0, 0, canvas.width, canvas.height);
  thresholdFilter(image.data, 0.8);
  return image;
}

const captureScreen = () => {
  const video = document.getElementById("video");
  if (!video) {
    console.error("Video element not found.");
    return;
  }

  const canvas = document.createElement("canvas");
  const context = canvas.getContext("2d");
  context.drawImage(video, 0, 0, canvas.width, canvas.height);
  context.putImageData(preprocessImage(canvas), 0, 0);

  capturedImage.value = canvas.toDataURL();
  showImage.value = true;

  stopStreaming();

  startScan();
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
  const firstNameRegex = /FN\s+([A-Za-z\s]+)\b/;
  const lastNameRegex = /LN\s+([^ ]+)/;
  const addressRegex = /\b\d+\b\s+[A-Za-z\s,]+[A-Z]{2}\s+\d{5}\b/;
  const dlRegex = /DL\s+(\w+)/s;
  const expiryDateRegex = /EXP\s+(.+)/i;

  // Execute regular expressions on the text and extract the matched data
  const firstNameMatch = firstNameRegex.exec(text);
  const lastNameMatch = lastNameRegex.exec(text);
  const addressMatch = addressRegex.exec(text);
  const dlMatch = dlRegex.exec(text);
  const expirationDateMatch = expiryDateRegex.exec(text);

  // Check if a match was found for each field and store it in the data object
  if (firstNameMatch) {
    data.firsName = firstNameMatch[1].trim();
  }
  if (lastNameMatch) {
    data.lastName = lastNameMatch[1].trim();
  }

  if (addressMatch) {
    data.address = addressMatch[0].trim();
  }

  if (dlMatch) {
    data.driverLicenseNumber = dlMatch[1].trim();
  }

  if (expirationDateMatch) {
    data.expirationDate = expirationDateMatch[1].trim();
  }

  return data;
};

const startScan = async () => {
  Swal.fire({
    title: "Scanning... Please Wait",
    allowEscapeKey: false,
    allowOutsideClick: false,
    background: "#343a40",
    color: "#fff",
    showConfirmButton: false,
    didOpen: () => {
      Swal.showLoading();
    },
  });

  try {
    const {
      data: { text },
    } = await Tesseract.recognize(capturedImage.value, "eng", {
      lang: "eng",
      config: {
        tessedit_char_blacklist: "!@#$%^&*()_+-=[]{};:'\",.<>?/|\\`~",
        tessedit_char_whitelist:
          "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789",
        psm: 3,
      },
      logger: (m) => console.log(m),
    });

    console.log("Extracted text:", text);

    // Further processing if needed
    const parsedData = parseData(text);
    overallExtractData.value = text;
    extractedData.value = parsedData;
    overallExtractData.value = text;
    showImage.value = false; // Hide the captured image after scan
    showWebcam.value = false;
    Swal.fire({
      title: "Scan Completed",
      allowEscapeKey: true,
      background: "#343a40",
      color: "#fff",
    });
  } catch (error) {
    console.error("Error extracting text:", error);
    Swal.fire({
      icon: "error",
      title: "Error",
      text: "An error occurred while processing the image. Please try again.",
      background: "#343a40",
      color: "#fff",
    });
  }
};

function thresholdFilter(pixels, level) {
  if (level === undefined) {
    level = 0.5;
  }
  const thresh = Math.floor(level * 255);
  for (let i = 0; i < pixels.length; i += 4) {
    const red = pixels[i];
    const green = pixels[i + 1];
    const blue = pixels[i + 2];

    const gray = 0.2126 * red + 0.7152 * green + 0.0722 * blue;
    let value;
    if (gray >= thresh) {
      value = 255;
    } else {
      value = 0;
    }
    pixels[i] = pixels[i + 1] = pixels[i + 2] = value;
  }
}
</script>
