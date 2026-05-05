<script setup lang="ts">
import { ref } from "vue";
import Tesseract from "tesseract.js";
import * as pdfjsLib from "pdfjs-dist";

// ต้องระบุ Worker สำหรับ pdf.js
pdfjsLib.GlobalWorkerOptions.workerSrc = `//cdnjs.cloudflare.com/ajax/libs/pdf.js/${pdfjsLib.version}/pdf.worker.min.mjs`;

const ocrResult = ref("");
const isProcessing = ref(false);

const handleFileUpload = async (event: any) => {
  const file = event.target.files[0];
  if (!file) return;

  isProcessing.value = true;
  ocrResult.value = "กำลังประมวลผล...";

  if (file.type === "application/pdf") {
    await processPdf(file);
  } else {
    await recognizeText(file);
  }
  isProcessing.value = false;
};

const processPdf = async (file: File) => {
  const arrayBuffer = await file.arrayBuffer();
  const pdf = await pdfjsLib.getDocument(arrayBuffer).promise;

  // ตัวอย่างนี้จะอ่านเฉพาะหน้าแรก (Page 1)
  const page = await pdf.getPage(1);
  const viewport = page.getViewport({ scale: 2 }); // scale 2 เพื่อความชัด
  const canvas = document.createElement("canvas");
  const context = canvas.getContext("2d");

  canvas.height = viewport.height;
  canvas.width = viewport.width;

  if (context) {
    await page.render({ canvasContext: context, viewport }).promise;
    // แปลง Canvas เป็น Blob (รูปภาพ) เพื่อส่งให้ Tesseract
    canvas.toBlob((blob) => {
      if (blob) recognizeText(blob);
    }, "image/png");
  }
};

const recognizeText = async (data: File | Blob) => {
  try {
    const {
      data: { text },
    } = await Tesseract.recognize(data, "tha+eng");
    ocrResult.value = text;
  } catch (error) {
    ocrResult.value = "เกิดข้อผิดพลาดในการอ่านไฟล์";
  }
};
</script>

<template>
  <div style="padding: 20px">
    <h1>Vue OCR (Support PDF & Image) 📄📸</h1>
    <!-- เพิ่ม .pdf ใน accept -->
    <input type="file" @change="handleFileUpload" accept="image/*,application/pdf" />

    <div style="margin-top: 20px">
      <p v-if="isProcessing">กำลังประมวลผล... ⏳</p>
      <div v-else>
        <h3>ผลลัพธ์:</h3>
        <pre style="white-space: pre-wrap; background: #000; padding: 15px">{{ ocrResult }}</pre>
      </div>
    </div>
  </div>
</template>
