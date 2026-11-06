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
    </div>
    <div class="status">
    <p>Selected: {{ type ? `IS (${isMode})` : 'CM' }}, Direction: {{ isFwd ? 'FWD' : 'BWD' }}, {{ isTgl ? 'TGL ON' : 'TGL OFF' }}{{ type ? (two ? ', transition: summer → snow (2021 weekly photos)' : ', single image') : '' }}</p>
  </div>
  <div ref="container" class="CC">
    <ImgSmear
      v-if="type && container"
      :key="two ? 'two' : 'one'"
      :container="container"
      :img="two ? '/photoA.jpg' : '/test2.jpg'"
      :img-b="two ? '/photoB.jpg' : undefined"
      :mode="isMode"
      :fwd="isFwd"
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
</style>
