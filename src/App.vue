<template>
  <div class="app">
    <div class="form-container">
      <input v-model="text" placeholder="Enter rank text..." class="input dark-input" />

      <canvas ref="canvas" class="preview-canvas"></canvas>
      <img :src="imageSrc" alt="Preview" class="preview-img"
        :style="{ backgroundColor: bgColor, borderColor: borderColor }" />
      <div class="color-pickers">
        <label>
          Background:
          <input type="color" v-model="bgColor" class="color-picker" />
        </label>
        <label>
          Border:
          <input type="color" v-model="borderColor" class="color-picker" />
          <input type="checkbox" v-model="showBorder" checked />
        </label>
        <label>
          Shadow:
          <input type="color" v-model="shadowColor" class="color-picker" />
          <input type="checkbox" v-model="showShadow" checked />
        </label>
        <label>
          <select v-model="selectedPreset" @change="applyPreset" class="dark-select" size="10">
            <option disabled value="">Select preset</option>
            <option v-for="(_, name) in presets" :key="name" :value="name">{{ name }}</option>
          </select>
        </label>
      </div>
      <button @click="downloadImage" class="dark-button">Download</button>
    </div>
  </div>
</template>

<script setup>
import { ref, watch, onMounted } from 'vue'

const text = ref('Rank')
const canvas = ref(null)
const tileSize = 8
const padding = 1
const spacing = 1
const height = tileSize
const width = ref(0)
const imageSrc = ref('')
const bgColor = ref('#282828')
const borderColor = ref('#a0a0a0')
const shadowColor = ref('#505050')
const selectedPreset = ref('')
const showBorder = ref(true)
const showShadow = ref(true)

const saturation = 0.08
const presets = {
  Classic: { bg: '#282828', border: '#a0a0a0' },
  Emerald: { bg: '#003e2f', border: '#00ffba' },
  Gold: { bg: '#3b2c00', border: '#ffcc00' },
  Nether: { bg: '#2b0f0f', border: '#ff3b3b' },
  Ice: { bg: '#0f2b3b', border: '#3bafff' },
  Diamond: { bg: '#0f3b3b', border: '#3bffff' },
  Ruby: { bg: '#3b0f0f', border: '#ff3b6b' },
  Amethyst: { bg: '#2b0f3b', border: '#a03bff' },
  Obsidian: { bg: '#0f0f2b', border: '#3b3bff' },
  Sandstone: { bg: '#3b2b0f', border: '#ffcc66' },
  Lapis: { bg: '#0f0f3b', border: '#3b6bff' },
  Ender: { bg: '#1a0f2b', border: '#7f3bff' },
  Prismarine: { bg: '#0f3b2b', border: '#3bffcc' },
  Copper: { bg: '#3b1f0f', border: '#ff9966' },
  Glowstone: { bg: '#3b3b0f', border: '#ffff66' },
  Crimson: { bg: '#3b0f1f', border: '#ff6699' },
  Warped: { bg: '#0f3b1f', border: '#66ff99' }
}

for (const name in presets) {
  const preset = presets[name];
  preset.shadow = adjustHSL(preset.bg, saturation);
}

function adjustHSL(colorHex, lightnessAdjustment) {
  const r = parseInt(colorHex.slice(1, 3), 16) / 255;
  const g = parseInt(colorHex.slice(3, 5), 16) / 255;
  const b = parseInt(colorHex.slice(5, 7), 16) / 255;

  const max = Math.max(r, g, b);
  const min = Math.min(r, g, b);
  const l = (max + min) / 2;

  const s = max === min ? 0 : l < 0.5 ? (max - min) / (max + min) : (max - min) / (2 - max - min);

  const h = (() => {
    if (max === min) return 0;
    if (max === r) return ((g - b) / (max - min) + (g < b ? 6 : 0)) / 6;
    if (max === g) return ((b - r) / (max - min) + 2) / 6;
    return ((r - g) / (max - min) + 4) / 6;
  })();

  const adjustedL = Math.min(Math.max(l + lightnessAdjustment, 0), 1);

  const q = adjustedL < 0.5 ? adjustedL * (1 + s) : adjustedL + s - adjustedL * s;
  const p = 2 * adjustedL - q;

  const toRGB = t => {
    if (t < 0) t += 1;
    if (t > 1) t -= 1;
    if (t < 1 / 6) return p + (q - p) * 6 * t;
    if (t < 1 / 2) return q;
    if (t < 2 / 3) return p + (q - p) * (2 / 3 - t) * 6;
    return p;
  };

  const newR = Math.round(toRGB(h + 1 / 3) * 255);
  const newG = Math.round(toRGB(h) * 255);
  const newB = Math.round(toRGB(h - 1 / 3) * 255);

  return `#${newR.toString(16).padStart(2, '0')}${newG.toString(16).padStart(2, '0')}${newB.toString(16).padStart(2, '0')}`;
}

const applyPreset = () => {
  const preset = presets[selectedPreset.value]
  if (preset) {
    bgColor.value = preset.bg
    borderColor.value = preset.border
    shadowColor.value = preset.shadow
  }
}

const fontImage = new Image()
fontImage.src = 'ascii.png'

let charWidths = {}

const charToIndex = char => {
  const code = char.charCodeAt(0)
  return code >= 32 && code <= 127 ? code - 32 : 0
}

const analyzeCharWidths = () => {
  const offCanvas = document.createElement('canvas')
  offCanvas.width = 128
  offCanvas.height = 128
  const ctx = offCanvas.getContext('2d')
  ctx.drawImage(fontImage, 0, 0)

  for (let i = 0; i < 96; i++) {
    const sx = (i % 16) * tileSize
    const sy = Math.floor(i / 16) * tileSize
    const imageData = ctx.getImageData(sx, sy, tileSize, tileSize)
    const data = imageData.data

    let minX = tileSize
    let maxX = 0

    for (let y = 0; y < tileSize; y++) {
      for (let x = 0; x < tileSize; x++) {
        const idx = (y * tileSize + x) * 4
        const alpha = data[idx + 3]
        if (alpha > 0) {
          if (x < minX) minX = x
          if (x > maxX) maxX = x
        }
      }
    }

    const width = maxX >= minX ? maxX - minX + 1 : 0
    charWidths[i] = { width, offsetX: minX }
  }
}

const draw = () => {
  const ctx = canvas.value.getContext('2d')
  ctx.imageSmoothingEnabled = false

  const chars = text.value.toLowerCase().split('')
  let totalWidth = padding + 1 // Extra padding for left and right

  for (const char of chars) {
    const i = charToIndex(char)
    const w = charWidths[i]?.width ?? tileSize
    totalWidth += w + spacing
  }

  const finalWidth = totalWidth - spacing + 2 // Extra padding for right
  const finalHeight = height + 1 // Extra padding for top, bottom, and 1 extra pixel at the bottom
  canvas.value.width = finalWidth
  canvas.value.height = finalHeight
  width.value = finalWidth

  ctx.fillStyle = bgColor.value
  ctx.fillRect(0, 0, finalWidth, finalHeight)

  if (showBorder.value) {
    ctx.strokeStyle = borderColor.value
    ctx.lineWidth = 1 // Border thickness
    ctx.strokeRect(0, 0, finalWidth, finalHeight)
  }

  let cursor = padding + 1 // Start with extra padding
  for (const char of chars) {
    const i = charToIndex(char)
    const tileX = (i % 16) * tileSize
    const tileY = Math.floor(i / 16) * tileSize
    const info = charWidths[i] ?? { width: tileSize, offsetX: 0 }

    if (showShadow.value) {
      const shadowImage = colorize(fontImage, shadowColor.value);
      ctx.drawImage(
        shadowImage,
        tileX + info.offsetX, tileY,
        info.width, tileSize,
        cursor + 1, 0, // Extra padding for top
        info.width, tileSize
      )
    }

    ctx.drawImage(
      fontImage,
      tileX + info.offsetX, tileY,
      info.width, tileSize,
      cursor, 0, // Extra padding for top
      info.width, tileSize
    )

    cursor += info.width + spacing
  }

  imageSrc.value = canvas.value.toDataURL()
}

function downloadImage() {
  const link = document.createElement('a')
  link.href = imageSrc.value
  link.download = 'rank.png'
  link.click()
}

function colorize(image, colorHex) {
  const r = parseInt(colorHex.slice(1, 3), 16) / 255;
  const g = parseInt(colorHex.slice(3, 5), 16) / 255;
  const b = parseInt(colorHex.slice(5, 7), 16) / 255;
  const imageSize = image.width;

  const offscreen = new OffscreenCanvas(imageSize, imageSize);
  const ctx = offscreen.getContext("2d");

  ctx.drawImage(image, 0, 0);

  const imageData = ctx.getImageData(0, 0, imageSize, imageSize);

  for (let i = 0; i < imageData.data.length; i += 4) {
    imageData.data[i + 0] *= r;
    imageData.data[i + 1] *= g;
    imageData.data[i + 2] *= b;
  }

  ctx.putImageData(imageData, 0, 0);

  return offscreen;
}

watch([text, bgColor, borderColor, shadowColor, showBorder, showShadow], () => {
  if (fontImage.complete) draw()
})

onMounted(() => {
  fontImage.onload = () => {
    analyzeCharWidths()
    draw()
  }
})
</script>

<style>
body {
  margin: 0;
  font-family: sans-serif;
  background: #1a1a1a;
  color: white;
  display: flex;
  justify-content: flex-start;
  align-items: flex-start;
  padding: 0;
  /* Prevent scrolling */
  overflow: hidden;
}

.app {
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: flex-start;
  gap: 1rem;
  padding: 1rem;
  height: 100vh;
  background: #1a1a1a;
}

.input {
  padding: 0.25rem;
  font-size: 1rem;
  width: 200px;
}

.color-pickers {
  display: flex;
  gap: 1rem;
  align-items: center;
}

.preview-canvas {
  display: none;
}

.preview-img {
  image-rendering: pixelated;
  width: auto;
  height: 128px;
}

button {
  padding: 0.4rem 0.8rem;
  background: #333;
  border: 1px solid #666;
  color: white;
  cursor: pointer;
}

.form-container {
  display: flex;
  flex-direction: column;
  gap: 1.5rem;
  align-items: center;
  background: #2a2a2a;
  padding: 2rem;
  border-radius: 8px;
  box-shadow: 0 4px 10px rgba(0, 0, 0, 0.5);
  border: 1px solid #4a4a4a;
}

.input.dark-input {
  padding: 0.5rem;
  font-size: 1.2rem;
  width: 250px;
  border: 1px solid #4a4a4a;
  border-radius: 4px;
  background: #1e1e2e;
  color: #e0e0ff;
}

.input.dark-input::placeholder {
  color: #a0a0c0;
}

.color-pickers {
  display: flex;
  flex-direction: column;
  gap: 1rem;
  align-items: flex-start;
}

.color-picker {
  margin-left: 0.5rem;
  border: none;
  background: #1e1e2e;
  border-radius: 4px;
  padding: 0.2rem;
}

.dark-select {
  margin-left: 0.5rem;
  padding: 0.4rem;
  border: 1px solid #4a4a4a;
  border-radius: 4px;
  background: #1e1e2e;
  color: #e0e0ff;
}

.dark-button {
  padding: 0.6rem 1.2rem;
  background: #3a3a5a;
  border: 1px solid #5a5a8a;
  color: #e0e0ff;
  border-radius: 4px;
  cursor: pointer;
  transition: background 0.3s;
}

.dark-button:hover {
  background: #4a4a7a;
}
</style>
