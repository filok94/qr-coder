<template>
  <div class="app">
    <div class="card">
      <h1 class="title">QR Coder</h1>

      <div class="mode-tabs">
        <button
          v-for="mode in MODES"
          :key="mode.id"
          class="mode-tab"
          :class="{ active: activeMode === mode.id }"
          @click="selectMode(mode.id)"
        >
          {{ mode.label }}
        </button>
      </div>

      <!-- Режим: Ссылка -->
      <template v-if="activeMode === 'url'">
        <input
          v-model="url"
          type="text"
          placeholder="Вставьте ссылку..."
          class="url-input"
          @keydown.enter="generate"
        />
      </template>

      <!-- Режим: Банковские реквизиты -->
      <template v-if="activeMode === 'bank'">
        <div class="bank-form">
          <div class="form-group">
            <label>Получатель</label>
            <input v-model="bankForm.name" type="text" placeholder="ФИО или название организации" />
          </div>
          <div class="form-group">
            <label>Расчётный счёт</label>
            <input v-model="bankForm.personalAcc" type="text" placeholder="20 цифр" maxlength="20" />
          </div>
          <div class="form-group">
            <label>БИК</label>
            <input v-model="bankForm.bic" type="text" placeholder="9 цифр" maxlength="9" />
          </div>
          <div class="form-group">
            <label>Наименование банка</label>
            <input v-model="bankForm.bankName" type="text" placeholder="ПАО СБЕРБАНК РОССИИ" />
          </div>
          <div class="form-group">
            <label>ИНН</label>
            <input v-model="bankForm.inn" type="text" placeholder="10 или 12 цифр" maxlength="12" />
          </div>
          <div class="form-group">
            <label>КПП <span class="optional">(опционально)</span></label>
            <input v-model="bankForm.kpp" type="text" placeholder="9 цифр" maxlength="9" />
          </div>
          <div class="form-group">
            <label>Назначение платежа</label>
            <input v-model="bankForm.purpose" type="text" placeholder="Оплата за..." />
          </div>
          <div class="form-group">
            <label>Сумма, ₽ <span class="optional">(опционально)</span></label>
            <input v-model="bankForm.amount" type="text" placeholder="1500" />
          </div>
        </div>
      </template>

      <button class="generate-btn" @click="generate">Сгенерировать</button>

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
import { ref, reactive, nextTick } from "vue";
import QRCodeStyling from "qr-code-styling";

type Mode = "url" | "bank";
type Preset = "corporate" | "gradient";

const activeMode = ref<Mode>("url");
const activePreset = ref<Preset>("corporate");
const url = ref("");
const generated = ref(false);
const qrContainer = ref<HTMLDivElement | null>(null);

const MODES = [
  { id: "url" as Mode, label: "Ссылка" },
  { id: "bank" as Mode, label: "Реквизиты" },
] as const;

const PRESETS = [
  { id: "corporate" as Preset, label: "Основной" },
  { id: "gradient" as Preset, label: "Градиент" },
] as const;

const bankForm = reactive({
  name: "",
  personalAcc: "",
  bic: "",
  bankName: "",
  inn: "",
  kpp: "",
  purpose: "",
  amount: "",
});

let currentQr: QRCodeStyling | null = null;

function selectMode(mode: Mode) {
  activeMode.value = mode;
  generated.value = false;
}

function selectPreset(id: Preset) {
  activePreset.value = id;
  if (generated.value) generate();
}

function buildBankQrData(): string {
  const parts = ["ST", "1.0"];
  if (bankForm.name.trim()) parts.push(`Name=${bankForm.name.trim()}`);
  if (bankForm.personalAcc.trim()) parts.push(`PersonalAcc=${bankForm.personalAcc.trim()}`);
  if (bankForm.bankName.trim()) parts.push(`BankName=${bankForm.bankName.trim()}`);
  if (bankForm.bic.trim()) parts.push(`BIC=${bankForm.bic.trim()}`);
  if (bankForm.inn.trim()) parts.push(`PayeeINN=${bankForm.inn.trim()}`);
  if (bankForm.kpp.trim()) parts.push(`KPP=${bankForm.kpp.trim()}`);
  if (bankForm.purpose.trim()) parts.push(`Purpose=${bankForm.purpose.trim()}`);
  if (bankForm.amount.trim()) {
    const amountKop = Math.round(parseFloat(bankForm.amount.replace(/,/g, ".")) * 100);
    if (!isNaN(amountKop) && amountKop > 0) parts.push(`Sum=${amountKop}`);
  }
  return parts.join("|");
}

function validateBankForm(): boolean {
  if (!bankForm.name.trim()) { alert("Укажите получателя"); return false; }
  if (!/^\d{20}$/.test(bankForm.personalAcc.trim())) { alert("Расчётный счёт должен содержать 20 цифр"); return false; }
  if (!/^\d{9}$/.test(bankForm.bic.trim())) { alert("БИК должен содержать 9 цифр"); return false; }
  if (!bankForm.bankName.trim()) { alert("Укажите наименование банка"); return false; }
  if (!/^\d{10}$/.test(bankForm.inn.trim()) && !/^\d{12}$/.test(bankForm.inn.trim())) { alert("ИНН должен содержать 10 или 12 цифр"); return false; }
  if (bankForm.kpp.trim() && !/^\d{9}$/.test(bankForm.kpp.trim())) { alert("КПП должен содержать 9 цифр"); return false; }
  return true;
}

function getQrData(): string {
  if (activeMode.value === "url") return url.value.trim();
  return buildBankQrData();
}

function validate(): boolean {
  if (activeMode.value === "url") {
    if (!url.value.trim()) { alert("Вставьте ссылку"); return false; }
    return true;
  }
  return validateBankForm();
}

function getOptions(data: string, presetId: Preset) {
  const base = {
    width: 400,
    height: 400,
    data,
    margin: 10,
    qrOptions: { typeNumber: 0 as const, mode: "Byte" as const, errorCorrectionLevel: "Q" as const },
    imageOptions: { hideBackgroundDots: true, imageSize: 0.4, margin: 0 },
  };

  switch (presetId) {
    case "corporate":
      return {
        ...base,
        dotsOptions: { type: "rounded" as const, color: "#e2e8f0" },
        cornersSquareOptions: { type: "extra-rounded" as const, color: "#e2e8f0" },
        cornersDotOptions: { type: "dot" as const, color: "#e2e8f0" },
        backgroundOptions: { color: "#1e293b" },
      };
    case "gradient":
      return {
        ...base,
        dotsOptions: {
          type: "rounded" as const,
          gradient: { type: "linear" as const, rotation: 0, colorStops: [{ offset: 0, color: "#818cf8" }, { offset: 1, color: "#f472b6" }] },
        },
        cornersSquareOptions: {
          type: "extra-rounded" as const,
          gradient: { type: "linear" as const, rotation: 0, colorStops: [{ offset: 0, color: "#818cf8" }, { offset: 1, color: "#f472b6" }] },
        },
        cornersDotOptions: {
          type: "dot" as const,
          gradient: { type: "linear" as const, rotation: 0, colorStops: [{ offset: 0, color: "#818cf8" }, { offset: 1, color: "#f472b6" }] },
        },
        backgroundOptions: { color: "#0f172a" },
      };
  }
}

function generate() {
  if (!validate()) return;
  const data = getQrData();
  generated.value = true;
  nextTick(() => {
    if (!qrContainer.value) return;
    qrContainer.value.innerHTML = "";
    currentQr = new QRCodeStyling(getOptions(data, activePreset.value));
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
  background: #0b0f19;
  font-family: "Inter", system-ui, -apple-system, sans-serif;
}

.card {
  background: #111827;
  border-radius: 1.5rem;
  padding: 2.5rem;
  width: 100%;
  max-width: 520px;
  box-shadow: 0 20px 60px rgba(0, 0, 0, 0.4);
  display: flex;
  flex-direction: column;
  gap: 1.25rem;
}

.title {
  margin: 0;
  font-size: 1.75rem;
  font-weight: 700;
  color: #f8fafc;
  text-align: center;
  letter-spacing: -0.025em;
}

/* Tabs */
.mode-tabs {
  display: flex;
  gap: 0.5rem;
  border-bottom: 1px solid #1f2937;
  padding-bottom: 0.5rem;
}

.mode-tab {
  flex: 1;
  padding: 0.5rem 0;
  border: none;
  background: transparent;
  color: #94a3b8;
  font-weight: 600;
  font-size: 0.95rem;
  cursor: pointer;
  border-radius: 0.5rem;
  transition: background 0.2s, color 0.2s;
}

.mode-tab:hover {
  background: #1f2937;
}

.mode-tab.active {
  background: #6366f1;
  color: #ffffff;
}

/* Inputs */
.url-input {
  flex: 1;
  padding: 0.75rem 1rem;
  border: 1px solid #334155;
  border-radius: 0.75rem;
  font-size: 1rem;
  outline: none;
  transition: border-color 0.2s, box-shadow 0.2s;
  color: #f1f5f9;
  background: #0f172a;
  width: 100%;
}

.url-input::placeholder {
  color: #64748b;
}

.url-input:focus {
  border-color: #6366f1;
  box-shadow: 0 0 0 3px rgba(99, 102, 241, 0.2);
}

/* Bank form */
.bank-form {
  display: flex;
  flex-direction: column;
  gap: 0.85rem;
}

.form-group {
  display: flex;
  flex-direction: column;
  gap: 0.3rem;
}

.form-group label {
  font-size: 0.8rem;
  font-weight: 600;
  color: #94a3b8;
}

.form-group label .optional {
  font-weight: 400;
  color: #64748b;
}

.form-group input {
  padding: 0.65rem 0.9rem;
  border: 1px solid #334155;
  border-radius: 0.6rem;
  font-size: 0.95rem;
  outline: none;
  transition: border-color 0.2s, box-shadow 0.2s;
  color: #f1f5f9;
  background: #0f172a;
  width: 100%;
}

.form-group input::placeholder {
  color: #475569;
}

.form-group input:focus {
  border-color: #6366f1;
  box-shadow: 0 0 0 3px rgba(99, 102, 241, 0.2);
}

/* Generate button */
.generate-btn {
  padding: 0.75rem 1.25rem;
  border: none;
  border-radius: 0.75rem;
  background: #6366f1;
  color: #ffffff;
  font-weight: 600;
  font-size: 0.95rem;
  cursor: pointer;
  transition: background 0.2s, transform 0.1s;
  white-space: nowrap;
}

.generate-btn:hover {
  background: #818cf8;
}

.generate-btn:active {
  transform: scale(0.97);
}

/* Presets */
.presets {
  display: flex;
  flex-wrap: wrap;
  gap: 0.5rem;
}

.preset-btn {
  padding: 0.4rem 0.8rem;
  border: 1px solid #334155;
  border-radius: 9999px;
  background: #1e293b;
  color: #94a3b8;
  font-size: 0.85rem;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.2s;
}

.preset-btn:hover {
  background: #334155;
  border-color: #475569;
}

.preset-btn.active {
  background: #6366f1;
  color: #ffffff;
  border-color: #6366f1;
}

/* Result */
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
  border: 1px solid #334155;
  border-radius: 0.75rem;
  background: #1e293b;
  color: #e2e8f0;
  font-weight: 600;
  font-size: 0.9rem;
  cursor: pointer;
  transition: background 0.2s;
}

.action-btn:hover {
  background: #334155;
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
