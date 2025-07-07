<template>
  <div class="container">
    <div class="note-grid">
      <button
        v-for="(note, key) in noteMap"
        :key="key"
        @mousedown="startTone(note.frequency)"
        @mouseup="stopTone"
        @touchstart.prevent="startTone(note.frequency)"
        @touchend.prevent="stopTone"
        @click="resumeAudio"
        class="note-button"
      >
        <div class="note-label">
          <span class="note-key">{{ key }}</span><br />
          <small class="note-name">{{ note.name }}</small>
        </div>
        <button class="add-button" @click.stop="addToSequence(key)" title="Add to sequence">＋</button>
      </button>
    </div>

    <div class="sequence-section" v-if="sequence.length > 0">
      <h2>Sequence</h2>
      <div class="sequence-notes">
        <div v-for="(key, index) in sequence" :key="index" class="note-button sequence-note">
          <div class="note-label">
            <span class="note-key">{{ key }}</span><br />
            <small class="note-name">{{ noteMap[key].name }}</small>
          </div>
          <button class="remove-button" @click="removeFromSequence(index)" title="Remove">&times;</button>
        </div>
      </div>
      <div class="controls">
        <div class="ptt-id-wrapper">
          <div class="ptt-label">nicFW&nbsp;&nbsp;PTT ID:</div>
          <div class="ptt-box">
            **<strong>{{ sequence.join('') }}</strong>
          </div>
        </div>
        <div class="button-row">
          <button @click="playSequence">Play Sequence</button>
          <button @click="saveCurrentSequence">Save Sequence</button>
        </div>
      </div>
    </div>

    <div class="saved-section">
      <h2 v-if="savedSequences.length > 0">Saved Sequences</h2>
      <ul class="saved-list" v-if="savedSequences.length > 0">
        <li v-for="(seq, i) in savedSequences" :key="i" class="saved-item">
          <button @click="loadSequence(i)">Load</button>
          {{ seq.join(' ') }}
          <button @click="deleteSequence(i)">Delete</button>
        </li>
      </ul>
      <div class="controls">
        <div v-if="savedSequences.length > 0">
          <button @click="copyExportToTextarea">Export JSON</button>
          <div v-if="showExportMessage" style="color: green">Save this to a .txt or .json file for future use.</div>
        </div>
        <textarea v-model="importText" placeholder="Paste JSON here to import" rows="4"></textarea>
        <button @click="importSequences">Import JSON</button>
        <div v-if="importError" style="color: red">{{ importError }}</div>
      </div>
    </div>

    <p style="margin-top: 2rem; font-size: 0.9rem; color: #555;">
      This app generates tones for the PTT ID "chirp" that are compatible with
      <a href="https://www.patreon.com/posts/nicfw-h3-v2-52-131755788" target="_blank" rel="noopener" style="color: #3498db; text-decoration: underline;">
        nicFW TD-H3 v2.52.20
      </a> or greater.
    </p>
    <p style="font-size: 0.9rem; color: #555;">
      Source code available on
      <a href="https://github.com/dapug/fw-tone-gen" target="_blank" rel="noopener" style="color: #3498db; text-decoration: underline;">
        GitHub</a>
    </p>
  </div>
</template>

<script setup lang="ts">
import { ref } from 'vue'

interface Note {
  name: string
  frequency: number
}

const noteMap: Record<string, Note> = {
  '0': { name: 'C-5', frequency: 523 },
  '1': { name: 'C#-5', frequency: 554 },
  '2': { name: 'D-5', frequency: 587 },
  '3': { name: 'D#-5', frequency: 622 },
  '4': { name: 'E-5', frequency: 659 },
  '5': { name: 'F-5', frequency: 698 },
  '6': { name: 'F#-5', frequency: 740 },
  '7': { name: 'G-5', frequency: 784 },
  '8': { name: 'G#-5', frequency: 831 },
  '9': { name: 'A-5', frequency: 880 },
  'A': { name: 'A#-5', frequency: 932 },
  'B': { name: 'B-5', frequency: 988 },
  'C': { name: 'C-6', frequency: 1047 },
  'D': { name: 'C#-6', frequency: 1109 },
  '*': { name: 'D-6', frequency: 1175 },
  '#': { name: 'D#-6', frequency: 1245 }
}

const validKeys = Object.keys(noteMap)
const audioCtx = new AudioContext()
let oscillator: OscillatorNode | null = null

const sequence = ref<string[]>([])
const savedSequences = ref<string[][]>(JSON.parse(localStorage.getItem('sequences') || '[]'))
const importText = ref('')
const importError = ref('')
const showExportMessage = ref(false)

const resumeAudio = () => {
  if (audioCtx.state === 'suspended') {
    audioCtx.resume()
  }
}

const startTone = (frequency: number) => {
  resumeAudio()
  playTone(frequency)
}

const playTone = (frequency: number) => {
  stopTone()
  oscillator = audioCtx.createOscillator()
  oscillator.type = 'sine'
  oscillator.frequency.setValueAtTime(frequency, audioCtx.currentTime)
  oscillator.connect(audioCtx.destination)
  oscillator.start()
}

const stopTone = () => {
  if (oscillator) {
    oscillator.stop()
    oscillator.disconnect()
    oscillator = null
  }
}

const addToSequence = (key: string) => {
  sequence.value.push(key)
}

const removeFromSequence = (index: number) => {
  sequence.value.splice(index, 1)
}

const playSequence = async () => {
  for (const key of sequence.value) {
    const note = noteMap[key]
    playTone(note.frequency)
    await new Promise(resolve => setTimeout(resolve, 100))
    stopTone()
  }
}

const saveCurrentSequence = () => {
  if (sequence.value.length > 0) {
    savedSequences.value.push([...sequence.value])
    localStorage.setItem('sequences', JSON.stringify(savedSequences.value))
    sequence.value = []
  }
}

const loadSequence = (index: number) => {
  sequence.value = [...savedSequences.value[index]]
}

const deleteSequence = (index: number) => {
  savedSequences.value.splice(index, 1)
  localStorage.setItem('sequences', JSON.stringify(savedSequences.value))
}

const copyExportToTextarea = () => {
  importText.value = JSON.stringify(savedSequences.value, null, 2)
  showExportMessage.value = true
  setTimeout(() => (showExportMessage.value = false), 4000)
}

const importSequences = () => {
  importError.value = ''
  try {
    const parsed = JSON.parse(importText.value)
    if (!Array.isArray(parsed) ||
        !parsed.every(arr =>
          Array.isArray(arr) &&
          arr.every(key => typeof key === 'string' && validKeys.includes(key)))) {
      throw new Error('Invalid format or unknown keys')
    }
    savedSequences.value = parsed
    localStorage.setItem('sequences', JSON.stringify(parsed))
    importText.value = ''
  } catch (e) {
    importError.value = 'Invalid JSON. Must be an array of sequences using only valid keys.'
  }
}
</script>

<style>
.container {
  padding: 1rem;
  font-family: sans-serif;
}

.note-grid {
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  gap: 0.5rem;
}

.note-button {
  padding: 0.5rem;
  border: none;
  background-color: #3498db;
  color: white;
  border-radius: 0.25rem;
  cursor: pointer;
  position: relative;
  text-align: left;
}

.note-label {
  display: inline-block;
}

.note-key {
  font-weight: bold;
  font-size: 1.25rem;
}

.note-name {
  font-size: 0.85rem;
}

.note-button:active {
  background-color: #1d6fa5;
}

.add-button {
  position: absolute;
  top: 4px;
  right: 4px;
  background: #28a745;
  border: none;
  color: white;
  font-size: 1rem;
  line-height: 1;
  padding: 0 0.4rem;
  border-radius: 3px;
  cursor: pointer;
}

.sequence-section,
.saved-section {
  margin-top: 1.5rem;
}

.sequence-notes {
  display: flex;
  flex-wrap: wrap;
  gap: 0.5rem;
  margin-top: 0.5rem;
  max-width: 100%;
  word-break: break-word;
}

.sequence-note {
  position: relative;
  word-break: break-word;
  white-space: normal;
  overflow-wrap: break-word;
  text-align: left;
}

.sequence-note .remove-button {
  position: absolute;
  top: 2px;
  right: 4px;
  background: #666;
  border: none;
  color: white;
  font-size: 0.75rem;
  line-height: 1;
  cursor: pointer;
  padding: 0 0.25rem;
  border-radius: 2px;
}

.saved-list {
  list-style: none;
  padding-left: 0;
  margin-top: 0.5rem;
}

.saved-item {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  justify-content: flex-start;
}

.controls {
  margin-top: 1rem;
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

.ptt-id-wrapper {
  display: flex;
  align-items: center;
  gap: 0.5rem;
}

.ptt-label {
  white-space: nowrap;
}

.ptt-box {
  border: 1px solid #888;
  padding: 0.5rem;
  font-family: monospace;
}

.button-row {
  display: flex;
  gap: 0.5rem;
}

button {
  user-select: none;
  padding: 0.5rem 1rem;
  border: none;
  background: #666;
  color: white;
  border-radius: 0.25rem;
  cursor: pointer;
}

button:hover {
  background: #444;
}
</style>
