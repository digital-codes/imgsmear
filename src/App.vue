<script setup lang="ts">
import { ref } from 'vue'
import ColorMix from './components/ColorMix.vue'
import ImgSmear from './components/ImgSmear.vue'

const type = ref(true)
const isMode = ref<'cpu' | 'shader'>('shader')
const isFwd = ref(true)
const isTgl = ref(true)
const two = ref(false)
const container = ref<HTMLElement | null>(null)

// demo state driving the component props
const pairs = [
  { a: '/photoA.jpg', b: '/photoB.jpg', label: 'Wiese 2021: Sommer → Schnee' },
  { a: '/test.jpg', b: '/test2.jpg', label: 'Testbilder' },
]
const pair = ref(0)
const bandPct = ref(12) // transition band, % of image width
const speedMs = ref(30) // ms per front step (lower = faster)

function cyclePair(): void {
  pair.value = (pair.value + 1) % pairs.length
}
</script>

<template>
  <div class="ctl">
    <button @click="type = !type">{{ type ? 'IS' : 'CM' }}</button>
    <button v-if="type" @click="isMode = isMode === 'cpu' ? 'shader' : 'cpu'">
      {{ isMode }}
    </button>
    <button v-if="type" @click="isFwd = !isFwd">
      {{ isFwd ? 'FWD' : 'BWD' }}
    </button>
    <button v-if="type" @click="isTgl = !isTgl">
      {{ isTgl ? 'TGL ON' : 'TGL OFF' }}
    </button>
    <button v-if="type" @click="two = !two">
      {{ two ? 'A→B' : '1 IMG' }}
    </button>
    <button v-if="type && two" @click="cyclePair">{{ pairs[pair].label }}</button>
    </div>
    <div class="ctl" v-if="type">
    <label>Bandbreite <input type="range" min="2" max="40" step="1" v-model.number="bandPct"> {{ bandPct }} %</label>
    <label>Speed <input type="range" min="5" max="150" step="5" v-model.number="speedMs"> {{ speedMs }} ms</label>
    </div>
    <div class="status">
    <p>Selected: {{ type ? `IS (${isMode})` : 'CM' }}, Direction: {{ isFwd ? 'FWD' : 'BWD' }}, {{ isTgl ? 'TGL ON' : 'TGL OFF' }}{{ type ? (two ? `, transition: ${pairs[pair].label}, Band ${bandPct} %, ${speedMs} ms/step` : ', single image') : '' }}</p>
  </div>
  <div ref="container" class="CC">
    <ImgSmear
      v-if="type && container"
      :container="container"
      :img="pairs[pair].a"
      :img-b="two ? pairs[pair].b : undefined"
      :mode="isMode"
      :fwd="isFwd"
      :tgl="isTgl"
      :delay-ms="speedMs"
      :band="bandPct / 100"
    />
    <ColorMix v-else-if="container" :container="container" />
  </div>
</template>


<style>
.ctl {
  margin: 0.5rem auto 0.5rem auto;
  display: flex;
  gap: 1rem;
  align-items: center;
}
.status {
  margin: 0.5rem 0;
}

.CC {
  width: 100%;
  margin: 1rem auto;
  height: 80vh;
}

button {
  width: 5rem;
  height: 2rem;
}

label {
  display: flex;
  align-items: center;
  gap: 0.5rem;
}
</style>
