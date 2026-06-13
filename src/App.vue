<template>
  <div class="app">
    <div class="card">
      <h1 class="title">QR Coder</h1>
      <div class="input-row">
        <input
          v-model="url"
          type="text"
          placeholder="Вставьте ссылку..."
          class="url-input"
          @keydown.enter="generate"
        />
        <button class="generate-btn" @click="generate">Сгенерировать</button>
      </div>

      <div class="presets">
        <button
          v-for="preset in PRESETS"
          :key="preset.id"
          class="preset-btn"
          :class="{ active: activePreset === preset.id }"
          @click="selectPreset(preset.id)"
        >
          {{ preset.label }}
        </button>
      </div>

      <div v-if="generated" class="result">
        <div ref="qrContainer" class="qr-wrap"></div>
        <div class="actions">
          <button class="action-btn" @click="downloadPng">Скачать PNG</button>
          <button class="action-btn" @click="copyToClipboard">Скопировать</button>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, nextTick } from "vue";
import QRCodeStyling from "qr-code-styling";

const url = ref("");
const generated = ref(false);
const qrContainer = ref<HTMLDivElement | null>(null);
const activePreset = ref("corporate");

const PRESETS = [
  { id: "corporate", label: "Корпоративный" },
  { id: "minimal", label: "Минимализм" },
  { id: "gradient", label: "Градиент" },
  { id: "neon", label: "Неон" },
  { id: "dots", label: "Точки" },
] as const;

let currentQr: QRCodeStyling | null = null;

function selectPreset(id: string) {
  activePreset.value = id;
  if (generated.value) generate();
}

function getOptions(presetId: string) {
  const base = {
    width: 400,
    height: 400,
    data: url.value,
    margin: 10,
    qrOptions: { typeNumber: 0 as const, mode: "Byte" as const, errorCorrectionLevel: "Q" as const },
    imageOptions: { hideBackgroundDots: true, imageSize: 0.4, margin: 0 },
  };

  switch (presetId) {
    case "corporate":
      return {
        ...base,
        dotsOptions: { type: "rounded" as const, color: "#0f172a" },
        cornersSquareOptions: { type: "extra-rounded" as const, color: "#0f172a" },
        cornersDotOptions: { type: "dot" as const, color: "#0f172a" },
        backgroundOptions: { color: "#f8fafc" },
      };
    case "minimal":
      return {
        ...base,
        dotsOptions: { type: "dots" as const, color: "#000000" },
        cornersSquareOptions: { type: "square" as const, color: "#000000" },
        cornersDotOptions: { type: "dot" as const, color: "#000000" },
        backgroundOptions: { color: "#ffffff" },
      };
    case "gradient":
      return {
        ...base,
        dotsOptions: {
          type: "rounded" as const,
          gradient: { type: "linear" as const, rotation: 0, colorStops: [{ offset: 0, color: "#6366f1" }, { offset: 1, color: "#ec4899" }] },
        },
        cornersSquareOptions: {
          type: "extra-rounded" as const,
          gradient: { type: "linear" as const, rotation: 0, colorStops: [{ offset: 0, color: "#6366f1" }, { offset: 1, color: "#ec4899" }] },
        },
        cornersDotOptions: {
          type: "dot" as const,
          gradient: { type: "linear" as const, rotation: 0, colorStops: [{ offset: 0, color: "#6366f1" }, { offset: 1, color: "#ec4899" }] },
        },
        backgroundOptions: { color: "#ffffff" },
      };
    case "neon":
      return {
        ...base,
        dotsOptions: { type: "rounded" as const, color: "#22d3ee" },
        cornersSquareOptions: { type: "extra-rounded" as const, color: "#22d3ee" },
        cornersDotOptions: { type: "dot" as const, color: "#22d3ee" },
        backgroundOptions: { color: "#0f172a" },
      };
    case "dots":
      return {
        ...base,
        dotsOptions: { type: "dots" as const, color: "#16a34a" },
        cornersSquareOptions: { type: "dot" as const, color: "#16a34a" },
        cornersDotOptions: { type: "dot" as const, color: "#16a34a" },
        backgroundOptions: { color: "#f0fdf4" },
      };
    default:
      return base;
  }
}

function generate() {
  if (!url.value.trim()) return;
  generated.value = true;
  nextTick(() => {
    if (!qrContainer.value) return;
    qrContainer.value.innerHTML = "";
    currentQr = new QRCodeStyling(getOptions(activePreset.value));
    currentQr.append(qrContainer.value);
  });
}

async function downloadPng() {
  if (!currentQr) return;
  currentQr.download({ name: "qr-code", extension: "png" });
}

async function copyToClipboard() {
  if (!currentQr) return;
  const blob = await currentQr.getRawData("png");
  if (!blob) return;
  await navigator.clipboard.write([
    new ClipboardItem({ [blob.type]: blob }),
  ]);
}
</script>

<style scoped>
.app {
  min-height: 100vh;
  display: flex;
  align-items: center;
  justify-content: center;
  background: #e2e8f0;
  font-family: "Inter", system-ui, -apple-system, sans-serif;
}

.card {
  background: #ffffff;
  border-radius: 1.5rem;
  padding: 2.5rem;
  width: 100%;
  max-width: 480px;
  box-shadow: 0 20px 60px rgba(0, 0, 0, 0.08);
  display: flex;
  flex-direction: column;
  gap: 1.25rem;
}

.title {
  margin: 0;
  font-size: 1.75rem;
  font-weight: 700;
  color: #0f172a;
  text-align: center;
  letter-spacing: -0.025em;
}

.input-row {
  display: flex;
  gap: 0.5rem;
}

.url-input {
  flex: 1;
  padding: 0.75rem 1rem;
  border: 1px solid #cbd5e1;
  border-radius: 0.75rem;
  font-size: 1rem;
  outline: none;
  transition: border-color 0.2s, box-shadow 0.2s;
  color: #0f172a;
}

.url-input:focus {
  border-color: #6366f1;
  box-shadow: 0 0 0 3px rgba(99, 102, 241, 0.15);
}

.generate-btn {
  padding: 0.75rem 1.25rem;
  border: none;
  border-radius: 0.75rem;
  background: #0f172a;
  color: #ffffff;
  font-weight: 600;
  font-size: 0.95rem;
  cursor: pointer;
  transition: background 0.2s, transform 0.1s;
  white-space: nowrap;
}

.generate-btn:hover {
  background: #1e293b;
}

.generate-btn:active {
  transform: scale(0.97);
}

.presets {
  display: flex;
  flex-wrap: wrap;
  gap: 0.5rem;
}

.preset-btn {
  padding: 0.4rem 0.8rem;
  border: 1px solid #e2e8f0;
  border-radius: 9999px;
  background: #f8fafc;
  color: #475569;
  font-size: 0.85rem;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.2s;
}

.preset-btn:hover {
  background: #f1f5f9;
  border-color: #cbd5e1;
}

.preset-btn.active {
  background: #0f172a;
  color: #ffffff;
  border-color: #0f172a;
}

.result {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 1rem;
  animation: fadeIn 0.3s ease-out;
}

.qr-wrap {
  width: 400px;
  height: 400px;
  display: flex;
  align-items: center;
  justify-content: center;
}

.actions {
  display: flex;
  gap: 0.5rem;
  width: 100%;
}

.action-btn {
  flex: 1;
  padding: 0.65rem 0;
  border: 1px solid #e2e8f0;
  border-radius: 0.75rem;
  background: #f8fafc;
  color: #0f172a;
  font-weight: 600;
  font-size: 0.9rem;
  cursor: pointer;
  transition: background 0.2s;
}

.action-btn:hover {
  background: #f1f5f9;
}

@keyframes fadeIn {
  from {
    opacity: 0;
    transform: translateY(8px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}
</style>
