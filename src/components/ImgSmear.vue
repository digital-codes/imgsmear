<script setup lang="ts">
import { ref, onMounted, onUnmounted, watch, nextTick, withDefaults } from 'vue'

type RenderMode = 'cpu' | 'shader'

const props = withDefaults(
  defineProps<{
    container?: HTMLElement
    img?: string
    /** Optional second image: when set the smear becomes a transition that
     * reveals imgB behind the sweeping front (A ahead, streak band mixing
     * A -> B at the front, B behind). Same canvas size as img (stretched).
     * Changing this prop reloads the images live. */
    imgB?: string
    mode?: RenderMode
    fwd?: boolean
    tgl?: boolean
    /** Milliseconds between front steps. Lower = faster. Live-updatable
     * (both renderers pick it up without a rebuild). */
    delayMs?: number
    // Width of the smear band per tick, in native image pixels. Default is
    // 1/120 of the image width, computed in onMounted. Override with an
    // explicit value if you want a fixed band size.
    stepColumns?: number
    /** Transition band width as a fraction of the image width (0..1).
     * Default 0.125 (1/8). Ignored in single-image mode. Live-updatable. */
    band?: number
  }>(),
  {
    mode: 'cpu',
    fwd: true,
    tgl: true,
    delayMs: 30,
    stepColumns: 1,
    band: 0.125,
  },
)

const direction = ref(props.fwd)
const directionToggle = ref(props.tgl)

/** fired once per sweep wrap, in both renderers — the streak sequencer
 *  advances to the next image pair on it */
const emit = defineEmits<{ wrapped: [forward: boolean] }>()

const canvasEl = ref<HTMLCanvasElement | null>(null)

// ---------- CPU state ----------
interface CpuState {
  ctx: CanvasRenderingContext2D
  originalImageData: ImageData
  currentImageData: ImageData
  bImageData: ImageData | null
  indexColumn: number
}
// Resolved step size (native pixels). If the prop is 0 we pick a value
// scaled to the image width so the smear band is visually meaningful at
// any resolution.
let stepColumnsResolved = 1
// Resolved transition band width in canvas px (from props.band as a width
// fraction). Recomputed when the image or the band prop changes; both
// renderers read it (CPU per tick, shader via a uniform).
let bandResolved = 8

function bandFromWidth(width: number): number {
  const f = props.band && props.band > 0 ? props.band : 0.125
  return Math.max(1, Math.round(width * f))
}

let cpuState: CpuState | null = null
let cpuTimerId: number | null = null

// ---------- Shader state ----------
interface ShaderState {
  gl: WebGLRenderingContext
  program: WebGLProgram
  texture: WebGLTexture
  textureB: WebGLTexture | null
  buffer: WebGLBuffer
  indexColumnLoc: WebGLUniformLocation | null
  directionLoc: WebGLUniformLocation | null
  hasBLoc: WebGLUniformLocation | null
  bandLoc: WebGLUniformLocation | null
  width: number
  height: number
  indexColumn: number
  rafId: number | null
  lastTickTime: number
  disposed: boolean
}
let shaderState: ShaderState | null = null

function directionUniformValue(isForward: boolean): number {
  // Shader treats values < 0.5 as forward.
  return isForward ? 0 : 1
}

// ---------- Shaders ----------
// Fullscreen quad covering clip space [-1, 1] x [-1, 1].
// v_uv maps position -> [0, 1] in each axis for texture sampling.
const VERTEX_SHADER_SRC = `
attribute vec2 position;
varying vec2 v_uv;
void main() {
  v_uv = position * 0.5 + 0.5;
  gl_Position = vec4(position, 0.0, 1.0);
}
`

// The smear is a per-fragment decision. Single-image mode (u_hasB = 0):
// pixels on the smeared side of the index column take the colour of the
// index column on their row; the rest keeps the original texture. Because
// the texture holds the pristine image, no reset pass is needed when the
// index wraps -- we just snap indexColumn back to width - 1 on the next
// tick and the shader instantly redraws the original.
//
// Two-image mode (u_hasB = 1): the smear becomes a transition. Ahead of the
// front: image A untouched. At the front: a streak band of width u_band that
// mixes A's front-column colour into B's front-column colour. Behind the
// front: image B revealed.
const FRAGMENT_SHADER_SRC = `
#ifdef GL_FRAGMENT_PRECISION_HIGH
precision highp float;
#else
precision mediump float;
#endif
uniform sampler2D u_image;
uniform sampler2D u_imageB;
uniform float u_indexColumn;
uniform float u_width;
uniform float u_direction;
uniform float u_hasB;
uniform float u_band;
varying vec2 v_uv;

void main() {
  // Flip Y explicitly so output orientation matches 2D canvas rendering.
  // This is deterministic across source types (including ImageBitmap).
  vec2 uv = vec2(v_uv.x, 1.0 - v_uv.y);
  float px = uv.x * u_width;
  float u = (u_indexColumn + 0.5) / u_width;
  bool shouldSmear = false;

  if (u_direction > 0.5) {
    // Forward direction: smear from left to index column
    shouldSmear = px < u_indexColumn;
  } else {
    // Backward direction: smear from index column to right
    shouldSmear = px > u_indexColumn;
  }

  if (shouldSmear && u_hasB > 0.5) {
    // Transition: A ahead of the front, streak band at the front mixing
    // A -> B, B revealed behind it.
    float dist = abs(px - u_indexColumn);
    if (dist > u_band) {
      gl_FragColor = texture2D(u_imageB, uv);
    } else {
      float t = clamp(dist / max(u_band, 1.0), 0.0, 1.0);
      vec3 ca = texture2D(u_image, vec2(u, uv.y)).rgb;
      vec3 cb = texture2D(u_imageB, vec2(u, uv.y)).rgb;
      // t = 0 at the front (adjacent to the unsmeared A region) -> A colour;
      // t = 1 at the band's outer edge (adjacent to the revealed B) -> B
      gl_FragColor = vec4(mix(ca, cb, t), 1.0);
    }
  } else if (shouldSmear) {
    // Sample the index column for this row. +0.5 lands on the pixel
    // centre when using nearest/linear filtering.
    gl_FragColor = texture2D(u_image, vec2(u, uv.y));
  } else {
    gl_FragColor = texture2D(u_image, uv);
  }
}
`

// ---------- WebGL helpers ----------
function compileShader(gl: WebGLRenderingContext, type: number, source: string): WebGLShader | null {
  const shader = gl.createShader(type)
  if (!shader) return null
  gl.shaderSource(shader, source)
  gl.compileShader(shader)
  if (!gl.getShaderParameter(shader, gl.COMPILE_STATUS)) {
    console.error('ImgSmear shader compile error:', gl.getShaderInfoLog(shader))
    gl.deleteShader(shader)
    return null
  }
  return shader
}

function linkProgram(gl: WebGLRenderingContext, vs: WebGLShader, fs: WebGLShader): WebGLProgram | null {
  const program = gl.createProgram()
  if (!program) return null
  gl.attachShader(program, vs)
  gl.attachShader(program, fs)
  gl.linkProgram(program)
  if (!gl.getProgramParameter(program, gl.LINK_STATUS)) {
    console.error('ImgSmear program link error:', gl.getProgramInfoLog(program))
    gl.deleteProgram(program)
    return null
  }
  return program
}

// ---------- Shared logic ----------
function advanceIndex(
  state: { indexColumn: number; width: number } | { indexColumn: number; currentImageData: ImageData },
  step: number
): boolean {
  // Returns true if the index wrapped (so caller can reset CPU buffer).
  // On wrap, skip ahead by one step so the next visible tick already has
  // a band on screen (otherwise we'd smear at width-1, which is a no-op,
  // and the loop looks like it pauses for one tick before restarting).
  const next = state.indexColumn - step
  if (next < 0) {
    const width = 'width' in state ? state.width : state.currentImageData.width
    // fixme
    // state.indexColumn = state.width - 1 //  - step
    state.indexColumn = width - 1 //  - step
    return true
  }
  state.indexColumn = next
  return false
}

// ---------- CPU renderer ----------
const cpuTick = (): void => {
  if (!cpuState) return

  const { ctx, currentImageData, indexColumn } = cpuState
  const data = currentImageData.data
  const width = currentImageData.width
  const height = currentImageData.height
  const originalImageData = cpuState.originalImageData.data // read-only source
  const bData = cpuState.bImageData?.data ?? null
  // transition band width in px (shared with the shader mode)
  const band = bandResolved

  for (let y = 0; y < height; y++) {
    const srcOffset = (y * width + indexColumn) * 4
    const r = originalImageData[srcOffset]
    const g = originalImageData[srcOffset + 1]
    const b = originalImageData[srcOffset + 2]
    const a = originalImageData[srcOffset + 3]

    if (!direction.value) {
      for (let x = 0; x < indexColumn; x++) {
        const offset = (y * width + x) * 4
        if (bData) {
          const dist = indexColumn - x
          if (dist > band) {
            // B revealed
            data[offset] = bData[offset]
            data[offset + 1] = bData[offset + 1]
            data[offset + 2] = bData[offset + 2]
            data[offset + 3] = bData[offset + 3]
          } else {
            // streak band mixing A front column -> B
            const t = dist / band // 0 at front -> 1 at band edge
            data[offset] = bData[offset] * t + r * (1 - t)
            data[offset + 1] = bData[offset + 1] * t + g * (1 - t)
            data[offset + 2] = bData[offset + 2] * t + b * (1 - t)
            data[offset + 3] = a
          }
        } else {
          data[offset] = r
          data[offset + 1] = g
          data[offset + 2] = b
          data[offset + 3] = a
        }
      }
    } else {
      for (let x = indexColumn; x < width; x++) {
        const offset = (y * width + x) * 4
        if (bData) {
          const dist = x - indexColumn
          if (dist > band) {
            data[offset] = bData[offset]
            data[offset + 1] = bData[offset + 1]
            data[offset + 2] = bData[offset + 2]
            data[offset + 3] = bData[offset + 3]
          } else {
            const t = dist / band
            data[offset] = bData[offset] * t + r * (1 - t)
            data[offset + 1] = bData[offset + 1] * t + g * (1 - t)
            data[offset + 2] = bData[offset + 2] * t + b * (1 - t)
            data[offset + 3] = a
          }
        } else {
          data[offset] = r
          data[offset + 1] = g
          data[offset + 2] = b
          data[offset + 3] = a
        }
      }
    }
  }

  ctx.putImageData(currentImageData, 0, 0)

  const wrapped = advanceIndex(cpuState, stepColumnsResolved)
  if (wrapped) {
    if (directionToggle.value) {
      direction.value = !direction.value
    }
    // Reset the working buffer from the pristine copy so the next cycle
    // starts from the original image again. Also push that reset to the
    // canvas on the same frame -- otherwise the all-one-color frame we
    // just drew lingers for one tick and the loop looks like it stops.
    cpuState.currentImageData = new ImageData(
      new Uint8ClampedArray(cpuState.originalImageData.data),
      width,
      height
    )
    ctx.putImageData(cpuState.currentImageData, 0, 0)
    emit('wrapped', direction.value)
  }
}

// ---------- Shader renderer ----------
function renderShaderFrame(): void {
  if (!shaderState) return
  const gl = shaderState.gl
  gl.viewport(0, 0, shaderState.width, shaderState.height)
  gl.clear(gl.COLOR_BUFFER_BIT)
  gl.drawArrays(gl.TRIANGLES, 0, 6)
}

const shaderRender = (time: number): void => {
  if (!shaderState || shaderState.disposed) return

  const elapsed = time - shaderState.lastTickTime
  if (elapsed >= props.delayMs) {
    const wrapped = advanceIndex(shaderState, stepColumnsResolved)
    if (wrapped) {
      if (directionToggle.value) {
        direction.value = !direction.value
      }
      emit('wrapped', direction.value)
    }
    shaderState.gl.uniform1f(shaderState.indexColumnLoc, shaderState.indexColumn)
    shaderState.gl.uniform1f(shaderState.directionLoc, directionUniformValue(direction.value))
    shaderState.lastTickTime = time
    renderShaderFrame()
  }

  shaderState.rafId = requestAnimationFrame(shaderRender)
}

function setupCpu(canvas: HTMLCanvasElement, bitmap: ImageBitmap, bitmapB: ImageBitmap | null): void {
  const ctx = canvas.getContext('2d')
  if (!ctx) return

  ctx.drawImage(bitmap, 0, 0)
  const originalImageData = ctx.getImageData(0, 0, canvas.width, canvas.height)
  const currentImageData = new ImageData(
    new Uint8ClampedArray(originalImageData.data),
    originalImageData.width,
    originalImageData.height
  )

  let bImageData: ImageData | null = null
  if (bitmapB) {
    // B stretched into A's canvas geometry (same-spot photos share aspect)
    ctx.drawImage(bitmapB, 0, 0, canvas.width, canvas.height)
    bImageData = ctx.getImageData(0, 0, canvas.width, canvas.height)
    ctx.drawImage(bitmap, 0, 0)
  }

  cpuState = {
    ctx,
    originalImageData,
    currentImageData,
    bImageData,
    indexColumn: canvas.width - 1
  }
  cpuTimerId = window.setInterval(cpuTick, props.delayMs)
}

function setupShader(
  canvas: HTMLCanvasElement,
  bitmap: ImageBitmap,
  bitmapB: ImageBitmap | null,
): boolean {
  const gl = canvas.getContext('webgl')
  if (!gl) {
    console.warn('ImgSmear: WebGL unavailable, shader mode requested')
    return false
  }

  const vs = compileShader(gl, gl.VERTEX_SHADER, VERTEX_SHADER_SRC)
  const fs = compileShader(gl, gl.FRAGMENT_SHADER, FRAGMENT_SHADER_SRC)
  if (!vs || !fs) return false

  const program = linkProgram(gl, vs, fs)
  if (!program) return false
  gl.useProgram(program)

  // Fullscreen quad
  const vertices = new Float32Array([
    -1, -1,  1, -1, -1,  1,
    -1,  1,  1, -1,  1,  1
  ])
  const buffer = gl.createBuffer()
  if (!buffer) return false
  gl.bindBuffer(gl.ARRAY_BUFFER, buffer)
  gl.bufferData(gl.ARRAY_BUFFER, vertices, gl.STATIC_DRAW)
  const positionLoc = gl.getAttribLocation(program, 'position')
  gl.enableVertexAttribArray(positionLoc)
  gl.vertexAttribPointer(positionLoc, 2, gl.FLOAT, false, 0, 0)

  // Image texture.
  // Keep unpack flip disabled and handle orientation in shader (FRAGMENT_SHADER_SRC)
  // for consistent behavior with ImageBitmap uploads.
  gl.pixelStorei(gl.UNPACK_FLIP_Y_WEBGL, false)

  const texture = gl.createTexture()
  if (!texture) return false
  gl.bindTexture(gl.TEXTURE_2D, texture)
  gl.texImage2D(gl.TEXTURE_2D, 0, gl.RGBA, gl.RGBA, gl.UNSIGNED_BYTE, bitmap)
  gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MIN_FILTER, gl.LINEAR)
  gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MAG_FILTER, gl.LINEAR)
  gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_S, gl.CLAMP_TO_EDGE)
  gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_T, gl.CLAMP_TO_EDGE)

  const imageLoc = gl.getUniformLocation(program, 'u_image')
  const indexColumnLoc = gl.getUniformLocation(program, 'u_indexColumn')
  const widthLoc = gl.getUniformLocation(program, 'u_width')
  const directionLoc = gl.getUniformLocation(program, 'u_direction')
  const hasBLoc = gl.getUniformLocation(program, 'u_hasB')
  const bandLoc = gl.getUniformLocation(program, 'u_band')
  if (!imageLoc || !indexColumnLoc || !widthLoc || !directionLoc || !hasBLoc || !bandLoc) return false

  gl.uniform1i(imageLoc, 0)
  gl.uniform1f(widthLoc, canvas.width)
  gl.uniform1f(indexColumnLoc, canvas.width - 1)
  gl.uniform1f(directionLoc, directionUniformValue(direction.value))

  // Optional second texture for the transition mode.
  let textureB: WebGLTexture | null = null
  if (bitmapB) {
    textureB = gl.createTexture()
    if (!textureB) return false
    gl.activeTexture(gl.TEXTURE1)
    gl.bindTexture(gl.TEXTURE_2D, textureB)
    gl.texImage2D(gl.TEXTURE_2D, 0, gl.RGBA, gl.RGBA, gl.UNSIGNED_BYTE, bitmapB)
    gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MIN_FILTER, gl.LINEAR)
    gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MAG_FILTER, gl.LINEAR)
    gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_S, gl.CLAMP_TO_EDGE)
    gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_T, gl.CLAMP_TO_EDGE)
    const imageBLoc = gl.getUniformLocation(program, 'u_imageB')
    if (!imageBLoc) return false
    gl.uniform1i(imageBLoc, 1)
    gl.uniform1f(hasBLoc, 1)
    gl.uniform1f(bandLoc, bandResolved)
  } else {
    gl.uniform1f(hasBLoc, 0)
    gl.uniform1f(bandLoc, 0)
  }

  shaderState = {
    gl,
    program,
    texture,
    textureB,
    buffer,
    indexColumnLoc,
    directionLoc,
    hasBLoc,
    bandLoc,
    width: canvas.width,
    height: canvas.height,
    indexColumn: canvas.width - 1,
    rafId: null,
    lastTickTime: performance.now(),
    disposed: false
  }

  // Render the initial frame so we don't wait one delayMs for the first
  // image to appear, and the canvas isn't blank during the first RAF gap.
  renderShaderFrame()

  shaderState.rafId = requestAnimationFrame(shaderRender)
  return true
}

// ---------- Lifecycle ----------
// Module-level bitmaps so the watcher-driven rebuild can re-use the
// already-loaded pixels instead of re-fetching.
let currentBitmap: ImageBitmap | null = null
let bitmapB: ImageBitmap | null = null
let unmounted = false
// A counter that, when changed, forces Vue to destroy and re-create the
// <canvas> DOM element. We bump it whenever the mode changes so each
// renderer gets a fresh, context-free canvas -- getContext('2d') returns
// null on a canvas that already has a WebGL context (and vice versa), so
// reusing the same canvas across modes silently breaks the swap.
const canvasEpoch = ref(0)

function sizeCanvas(canvas: HTMLCanvasElement, bitmap: ImageBitmap): void {
  canvas.width = bitmap.width
  canvas.height = bitmap.height
  if (!props.container) return
  const containerRect = props.container.getBoundingClientRect()
  const maxWidth = containerRect.width > 0 ? containerRect.width : bitmap.width
  const maxHeight = containerRect.height > 0 ? containerRect.height : bitmap.height
  const scale = Math.min(maxWidth / bitmap.width, maxHeight / bitmap.height)
  const safeScale = Number.isFinite(scale) && scale > 0 ? scale : 1
  canvas.style.width = `${Math.max(1, Math.floor(bitmap.width * safeScale))}px`
  canvas.style.height = `${Math.max(1, Math.floor(bitmap.height * safeScale))}px`
}

function teardownRenderer(): void {
  if (cpuTimerId !== null) {
    window.clearInterval(cpuTimerId)
    cpuTimerId = null
  }
  if (shaderState) {
    shaderState.disposed = true
    if (shaderState.rafId !== null) {
      cancelAnimationFrame(shaderState.rafId)
      shaderState.rafId = null
    }
    try {
      const loseContext = shaderState.gl.getExtension('WEBGL_lose_context')
      loseContext?.loseContext()
    } catch {
      // Context may already be invalidated by the canvas being removed.
    }
    shaderState = null
  }
  console.log('teardownRenderer called')
  cpuState = null
}

function setupRenderer(mode: RenderMode): void {
  if (unmounted || !canvasEl.value || !currentBitmap) return
  teardownRenderer()

  const canvas = canvasEl.value
  // Re-apply dimensions: when the canvas DOM element is recreated via
  // :key, width/height reset to the defaults (300x150).
  sizeCanvas(canvas, currentBitmap)

  if (mode === 'shader') {
    const ok = setupShader(canvas, currentBitmap, bitmapB)
    if (!ok) {
      console.warn('ImgSmear: shader setup failed, falling back to CPU mode')
      setupCpu(canvas, currentBitmap, bitmapB)
    }
  } else {
    setupCpu(canvas, currentBitmap, bitmapB)
  }
  console.log(`ImgSmear: mode=${mode} ${currentBitmap.width}x${currentBitmap.height} delayMs=${props.delayMs} stepColumns=${stepColumnsResolved} twoImage=${!!bitmapB}`)
}

async function loadAndStart(): Promise<void> {
  if (!props.img || !props.container || !canvasEl.value) return

  const response = await fetch(props.img)
  const blob = await response.blob()
  let bitmap = await createImageBitmap(blob)

  // Scale image so largest dimension matches container's largest dimension
  const containerRect = props.container.getBoundingClientRect()
  const containerMaxDim = Math.max(containerRect.width, containerRect.height)
  const bitmapMaxDim = Math.max(bitmap.width, bitmap.height)

  if (containerMaxDim > 0 && bitmapMaxDim > containerMaxDim) {
    const scale = containerMaxDim / bitmapMaxDim
    const scaledWidth = Math.round(bitmap.width * scale)
    const scaledHeight = Math.round(bitmap.height * scale)

    const offCanvas = new OffscreenCanvas(scaledWidth, scaledHeight)
    const ctx = offCanvas.getContext('2d')
    if (ctx) {
      ctx.drawImage(bitmap, 0, 0, scaledWidth, scaledHeight)
      bitmap = await createImageBitmap(offCanvas)
    }
  }

  if (currentBitmap) currentBitmap.close()
  currentBitmap = bitmap

  // Optional transition target image, stretched into A's geometry.
  if (bitmapB) {
    bitmapB.close()
    bitmapB = null
  }
  if (props.imgB) {
    try {
      const respB = await fetch(props.imgB)
      const blobB = await respB.blob()
      const rawB = await createImageBitmap(blobB)
      const offB = new OffscreenCanvas(bitmap.width, bitmap.height)
      const octx = offB.getContext('2d')
      if (octx) {
        octx.drawImage(rawB, 0, 0, bitmap.width, bitmap.height)
        bitmapB = await createImageBitmap(offB)
      }
      rawB.close()
    } catch {
      console.warn('ImgSmear: failed to load imgB, running single-image smear')
      bitmapB = null
    }
  }

  // Pick a sensible step if the caller left it at 0. Target ~120 distinct
  // smear steps across the full image width -- yields a clearly visible
  // band on any display scale.
  stepColumnsResolved = props.stepColumns > 0
    ? props.stepColumns
    : Math.max(1, Math.round(bitmap.width / 120))

  bandResolved = bandFromWidth(bitmap.width)

  setupRenderer((props.mode ?? 'cpu') as RenderMode)
}

onMounted(loadAndStart)

// React to prop changes. We bump canvasEpoch to force a fresh canvas DOM
// element, then wait for Vue to swap it in before (re)building the
// renderer. Without the canvas swap, switching from shader -> cpu (or
// after a ColorMix intermission that left a WebGL context in the
// browser's memory) gives a blank canvas.
watch(
  () => props.mode,
  async (newMode) => {
    if (unmounted) return
    canvasEpoch.value++
    await nextTick()
    setupRenderer((newMode ?? 'cpu') as RenderMode)
  },
)
watch(
  () => props.fwd,
  async (newFwd) => {
    if (unmounted) return
    direction.value = newFwd
    if (shaderState) {
      shaderState.gl.uniform1f(shaderState.directionLoc, directionUniformValue(direction.value))
      renderShaderFrame()
    }
  },
)

watch(
  () => props.tgl,
  async (newTgl) => {
    if (unmounted) return
    directionToggle.value = newTgl
  }
)

// Switching images reloads the bitmaps and rebuilds the renderer on the
// fresh canvas (no remount needed).
watch(
  () => [props.img, props.imgB] as const,
  async () => {
    if (unmounted) return
    canvasEpoch.value++
    await nextTick()
    await loadAndStart()
  },
)

// Band width is a uniform in the shader and read per tick by the CPU
// renderer -- update live, no rebuild.
watch(
  () => props.band,
  () => {
    if (unmounted || !currentBitmap) return
    bandResolved = bandFromWidth(currentBitmap.width)
    if (shaderState) {
      shaderState.gl.uniform1f(shaderState.bandLoc, bandResolved)
      renderShaderFrame()
    }
  },
)

// The CPU renderer starts a setInterval with the initial delayMs; restart
// it so speed changes apply live (the shader reads props.delayMs each
// frame and picks changes up on its own).
watch(
  () => props.delayMs,
  () => {
    if (unmounted || cpuTimerId === null) return
    window.clearInterval(cpuTimerId)
    cpuTimerId = window.setInterval(cpuTick, props.delayMs)
  },
)

onUnmounted(() => {
  unmounted = true
  teardownRenderer()
  if (currentBitmap) {
    currentBitmap.close()
    currentBitmap = null
  }
  if (bitmapB) {
    bitmapB.close()
    bitmapB = null
  }
})
</script>

<template>
  <canvas :key="canvasEpoch" ref="canvasEl"></canvas>
</template>

<style>
</style>
