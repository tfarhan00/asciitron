<template>
  <Head>
    <Title>ASCIItron</Title>
  </Head>
  <div>
    <div
      class="fixed top-0 left-0 w-full h-screen flex items-center justify-center z-0"
    >
      <canvas
        ref="canvasElement"
        :class="[
          'h-[540px] w-full max-w-3xl transition-transform',
          isPlaying ? 'scale-y-100' : 'scale-y-0',
        ]"
      ></canvas>
      <div
        v-if="!isPlaying"
        class="text-sm font-mono absolute top-[50%] left-[50%] -translate-x-[50%] -translate-y-[50%] flex flex-col items-center gap-1"
      >
        <p class="font-bold">ASCIItron</p>
        <p class="text-xs">
          {{
            !isLoading
              ? "[ascii webcam video renderer]"
              : "[loading video data]"
          }}
        </p>
      </div>
    </div>

    <div
      class="w-full fixed bottom-0 left-0 p-6 flex justify-center z-50 pointer-events-none"
    >
      <div class="flex items-center gap-3 pointer-events-auto">
        <button
          @click="startWebcam"
          class="border border-black bg-white px-4 py-2 text-sm group flex items-center relative overflow-hidden hover:text-white"
        >
          <div
            class="absolute top-0 left-0 w-full h-full translate-y-[100%] group-hover:translate-y-[0%] bg-black transition-transform"
          ></div>
          <div
            class="w-0 group-hover:w-7 overflow-hidden transition-all relative"
          >
            <PlayCircle class="w-5 h-5" />
          </div>
          <p class="relative transition-colors">Start</p>
        </button>
        <button
          @click="stopWebcam"
          class="border border-black bg-white px-4 py-2 text-sm group flex items-center relative overflow-hidden hover:text-white"
        >
          <div
            class="absolute top-0 left-0 w-full h-full translate-y-[100%] group-hover:translate-y-[0%] bg-black transition-transform"
          ></div>
          <div
            class="w-0 group-hover:w-7 overflow-hidden transition-all relative"
          >
            <StopCircle class="w-5 h-5" />
          </div>
          <p class="relative transition-colors">Stop</p>
        </button>
        <button
          @click="captureImage"
          class="border border-black bg-white px-4 py-2 text-sm group flex items-center relative overflow-hidden hover:text-white"
        >
          <div
            class="absolute top-0 left-0 w-full h-full translate-y-[100%] group-hover:translate-y-[0%] bg-black transition-transform"
          ></div>
          <div
            class="w-0 group-hover:w-7 overflow-hidden transition-all relative"
          >
            <Camera class="w-5 h-5" />
          </div>
          <p class="relative transition-colors">Capture</p>
        </button>
        <button
          @click="toggleAscii"
          class="border border-black bg-white px-4 py-2 text-sm group flex items-center relative overflow-hidden hover:text-white"
        >
          <div
            class="absolute top-0 left-0 w-full h-full translate-y-[100%] group-hover:translate-y-[0%] bg-black transition-transform"
          ></div>
          <div
            class="w-0 group-hover:w-7 overflow-hidden transition-all relative"
          >
            <Circle v-if="isAsciiEnabled" class="relative w-5 h-5" />
            <Braces v-else class="relative w-5 h-5" />
          </div>
          <p class="relative transition-colors">
            {{ isAsciiEnabled ? "Normal" : "ASCII" }}
          </p>
        </button>
        <select
          v-model="mode"
          class="border border-black bg-white px-4 py-2 text-sm group flex items-center relative overflow-hidden"
        >
          <option value="grayscale">Grayscale</option>
          <option value="color">Color</option>
          <option value="thermal">Thermal</option>
        </select>
      </div>
    </div>
  </div>

  <!-- Modal for Captured Image -->
  <Transition>
    <div
      v-if="capturedImage"
      class="fixed inset-0 z-50 flex items-center justify-center bg-black bg-opacity-20 backdrop-blur-2xl"
    >
      <div class="bg-white flex flex-col items-start">
        <img
          :src="capturedImage"
          alt="Captured Image"
          class="max-w-full h-auto"
        />
        <div class="flex items-center w-full">
          <a
            :href="capturedImage"
            download="captured-image.png"
            class="flex-1 bg-white px-4 py-2.5 text-sm group flex items-center justify-between relative overflow-hidden text-black hover:bg-black/5"
          >
            <p class="relative transition-colors">Download</p>
            <Download class="w-4 h-4 relative" />
          </a>
          <button
            @click="closeModal"
            class="flex-1 bg-red-500 px-4 py-2.5 text-sm group flex items-center justify-between relative overflow-hidden text-white hover:bg-red-500/90"
          >
            <p class="relative transition-colors">Close</p>
            <X class="w-4 h-4 relative" />
          </button>
        </div>
      </div>
    </div>
  </Transition>
</template>

<script setup lang="ts">
import {
  Braces,
  Camera,
  Circle,
  Download,
  PlayCircle,
  StopCircle,
  X,
} from "lucide-vue-next";
import { ref, onUnmounted } from "vue";

const canvasElement = ref<HTMLCanvasElement | null>(null);
const stream = ref<MediaStream | null>(null);
const isPlaying = ref(false);
const isLoading = ref(false);
const videoElement = ref<HTMLVideoElement | null>(null);
const mode = ref("grayscale");
const isAsciiEnabled = ref(true);
const capturedImage = ref<string | null>(null);

let animationFrame: number | null = null;
const asciiChars = ["@", "#", "S", "%", "?", "*", "+", ";", ":", ",", "."];

const grayscaleToThermalColor = (value: number): string => {
  const normalizedValue = value / 255;
  let r = 0,
    g = 0,
    b = 0;

  if (normalizedValue <= 0.25) {
    r = 0;
    g = 0;
    b = 4 * normalizedValue * 255;
  } else if (normalizedValue <= 0.5) {
    r = 0;
    g = 4 * (normalizedValue - 0.25) * 255;
    b = 255;
  } else if (normalizedValue <= 0.75) {
    r = 4 * (normalizedValue - 0.5) * 255;
    g = 255;
    b = 255 - r;
  } else {
    r = 255;
    g = 255 - 4 * (normalizedValue - 0.75) * 255;
    b = 0;
  }

  return `rgb(${r},${g},${b})`;
};

const startWebcam = async (): Promise<void> => {
  if (isPlaying.value) return;
  isLoading.value = true;
  try {
    stream.value = await navigator.mediaDevices.getUserMedia({ video: true });
    videoElement.value = document.createElement("video");
    videoElement.value.srcObject = stream.value;
    videoElement.value.play();

    videoElement.value.onloadedmetadata = () => {
      if (canvasElement.value && videoElement.value) {
        canvasElement.value.width = videoElement.value.videoWidth;
        canvasElement.value.height = videoElement.value.videoHeight;
        drawToCanvas();
      }
      isPlaying.value = true;
      isLoading.value = false;
    };
  } catch (error) {
    console.error("Error accessing the webcam:", error);
    isPlaying.value = false;
  }
};

const stopWebcam = (): void => {
  if (stream.value) {
    stream.value.getTracks().forEach((track: MediaStreamTrack) => track.stop());
    stream.value = null;
  }
  if (animationFrame !== null) {
    cancelAnimationFrame(animationFrame);
    animationFrame = null;
  }
  if (videoElement.value) {
    videoElement.value.srcObject = null;
    videoElement.value = null;
  }
  isPlaying.value = false;
};

const drawToCanvas = (): void => {
  if (canvasElement.value && videoElement.value) {
    const ctx = canvasElement.value.getContext("2d");
    if (ctx) {
      const canvas = canvasElement.value;
      const video = videoElement.value;

      // Higher resolution settings
      const charWidth = 6;
      const charHeight = 10;

      const cols = Math.floor(canvas.width / charWidth);
      const rows = Math.floor(canvas.height / charHeight);

      ctx.drawImage(video, 0, 0, cols, rows);
      const imageData = ctx.getImageData(0, 0, cols, rows);
      ctx.fillStyle = "black";
      ctx.fillRect(0, 0, canvas.width, canvas.height);

      if (isAsciiEnabled.value) {
        for (let i = 0; i < imageData.data.length; i += 4) {
          const r = imageData.data[i];
          const g = imageData.data[i + 1];
          const b = imageData.data[i + 2];

          // Apply a weighted grayscale conversion for better dynamic range
          const avg = 0.2126 * r + 0.7152 * g + 0.0722 * b;

          const col = (i / 4) % cols;
          const row = Math.floor(i / 4 / cols);

          const asciiIndex = Math.floor((avg / 255) * (asciiChars.length - 1));
          const char = asciiChars[asciiIndex];

          let fillStyle;
          if (mode.value === "grayscale") {
            const gray = avg; // Make grayscale lighter
            fillStyle = `rgb(${gray},${gray},${gray})`;
          } else if (mode.value === "color") {
            fillStyle = `rgb(${r},${g},${b})`;
          } else if (mode.value === "thermal") {
            fillStyle = grayscaleToThermalColor(avg);
          } else {
            fillStyle = `rgb(${r},${g},${b})`;
          }

          ctx.fillStyle = fillStyle; // Set text color to white
          ctx.font = `${charHeight}px monospace`;
          ctx.fillText(char, col * charWidth, (row + 1) * charHeight);
        }
      } else {
        ctx.drawImage(video, 0, 0, canvas.width, canvas.height);
      }
    }
    animationFrame = requestAnimationFrame(drawToCanvas);
  }
};

const captureImage = (): void => {
  if (!isPlaying.value) return;
  if (canvasElement.value) {
    capturedImage.value = canvasElement.value.toDataURL("image/png");
  }
};

const closeModal = (): void => {
  capturedImage.value = null;
};

const toggleAscii = (): void => {
  isAsciiEnabled.value = !isAsciiEnabled.value;
};

onUnmounted(() => {
  stopWebcam();
});
</script>

<style scoped>
.v-enter-active,
.v-leave-active {
  transition: opacity 0.2s ease;
}

.v-enter-from,
.v-leave-to {
  opacity: 0;
}
</style>
