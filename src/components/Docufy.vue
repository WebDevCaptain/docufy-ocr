<template>
  <v-container class="fill-height">
    <v-responsive class="align-centerfill-height mx-auto" max-width="900">
      <v-textarea
        v-show="detectedText"
        v-model="detectedText"
        label="Detected Text"
        readonly
      ></v-textarea>

      <v-progress-circular
        v-if="showSpinner"
        indeterminate
        color="primary"
        class="mt-4 mb-6 mx-auto d-block"
      ></v-progress-circular>

      <v-file-input
        v-model="image"
        label="Upload Document"
        accept="image/*"
        @change="handleImageUpload"
      ></v-file-input>

      <p class="text-center">OR</p>

      <div class="text-center">
        <v-btn color="primary" class="mt-4 mb-4" @click="captureFromCamera"
          >Capture from Camera</v-btn
        >
      </div>

      <v-img :src="imagePreview" v-if="imagePreview" class="border"></v-img>
    </v-responsive>
  </v-container>
</template>

<script setup>
import { ref, watch } from "vue";
import Tesseract from "tesseract.js";
import { pipeline } from "@huggingface/transformers";

const imagePreview = ref(null);
const detectedText = ref("");
const showSpinner = ref(false);

const updateImageData = (blob) => {
  const reader = new FileReader();
  reader.onload = () => {
    imagePreview.value = reader.result;
  };
  reader.readAsDataURL(blob);
};

const handleImageUpload = (e) => {
  const file = e.target.files[0];

  if (file) {
    updateImageData(file);
  }
};

const captureFromCamera = () => {
  navigator.mediaDevices
    .getUserMedia({ video: true })
    .then((stream) => {
      const video = document.createElement("video");
      video.srcObject = stream;
      video.onloadedmetadata = () => {
        video.play();
        const canvas = document.createElement("canvas");
        canvas.width = video.videoWidth;
        canvas.height = video.videoHeight;
        const context = canvas.getContext("2d");
        context.drawImage(video, 0, 0, canvas.width, canvas.height);
        const imageData = context.getImageData(
          0,
          0,
          canvas.width,
          canvas.height
        );
        const blob = new Blob([imageData], { type: "image/png" });

        handleImageUpload(blob);
      };
    })
    .catch((error) => {
      console.error("Error accessing camera:", error);
    });
};

const runOCRwebGPU = async (imageUrl) => {
  console.info("Running OCR on WebGPU");

  const captioner = await pipeline(
    "image-to-text",
    "Xenova/trocr-small-handwritten"
  );

  const result = await captioner(imageUrl);
  console.log(result);

  detectedText.value = result.text;
  showSpinner.value = false;
};

const performOCR = async (image) => {
  showSpinner.value = true;

  // Check WebGPU support
  if (navigator.gpu) {
    await runOCRwebGPU(image);
    return;
  }

  if (image) {
    const {
      data: { text },
    } = await Tesseract.recognize(image, "eng", {
      logger: (m) => console.log(m),
    });

    showSpinner.value = false;
    console.log(text);

    detectedText.value = text;
  }
};

watch(imagePreview, () => {
  if (imagePreview.value) {
    performOCR(imagePreview.value);
  }
});
</script>

<style></style>
