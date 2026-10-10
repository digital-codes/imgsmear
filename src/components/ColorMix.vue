<script setup lang="ts">
import { ref, onMounted, onUnmounted } from 'vue'

const running = ref(false)

const cv = ref<HTMLCanvasElement | null>(null)
const glCtx = ref<WebGLRenderingContext | null>(null)
let cleanupColorMix: (() => void) | null = null
let animationFrameId: number | null = null

const props = defineProps({
  container: HTMLElement | undefined
})

// Install Vite globally if not installed
// npm create vite@latest my-webgl-project -- --template vanilla

onMounted(() => {
  console.log("Mounted, container: ",props.container)
  colorMix(props.container);
  running.value = true;
});

onUnmounted(() => {
    cleanupColorMix?.();
    cleanupColorMix = null;

    const gl = glCtx.value;
    if (gl) {
        const loseContext = gl.getExtension('WEBGL_lose_context');
        loseContext?.loseContext();
        glCtx.value = null;
    }

    if (cv.value) {
        cv.value.remove();
        cv.value = null;
    }

    running.value = false;
});

const colorMix = (el:HTMLElement) => {
    cleanupColorMix?.();

    // Setup WebGL canvas
    const canvas = document.createElement('canvas');
    canvas.width = el.clientWidth || el.offsetWidth || 1280;
    canvas.height = el.clientHeight || el.offsetHeight || 720;
    cv.value = canvas
    //document.body.appendChild(canvas);
    el.appendChild(canvas);
    const gl = canvas.getContext('webgl');
    if (!gl) {
        return;
    }
    glCtx.value = gl;

    console.log('colorMix.js');

    let dropPositions = new Float32Array(800); // 400 drops, 2 values each (x, y)
    let dropColors = new Float32Array(1200); // 400 drops, 3 values each (r, g, b)
    let dropTimes = new Float32Array(400);
    let stirring = false;
    let selectedColor = [1.0, 0.0, 0.0];
    let maxDrops = 200;
    let dropIndex = 0;
    let diffusion = 0.03;
    let dissolution = 0.1;
    let gravity = 0.01;
    let running = true;

    let mouseDown = false;

    const handleMouseDown = (event: MouseEvent) => {
        if (event.button === 0) { // Left click
            mouseDown = true;
            addDrop(event.offsetX, event.offsetY);
        } else if (event.button === 2) { // Right click
            stirring = true;
        }
    };

    const handleMouseUp = (event: MouseEvent) => {
        if (event.button === 0) {
            mouseDown = false;
        } else if (event.button === 2) {
            stirring = false;
        }
    };

    const handleMouseMove = (event: MouseEvent) => {
        if (mouseDown) {
            addDrop(event.offsetX, event.offsetY);
        }
        if (stirring) {
            console.log("Stirring at", event.offsetX, event.offsetY);
        }
    };

    const handleWheel = (event: WheelEvent) => {
        console.log("Wheel event", event.deltaY);
        diffusion = Math.max(0.01, Math.min(0.05, diffusion += -.0005 * Math.sign(event.deltaY)));
        event.preventDefault();
    };

    const handleKeyDown = (event: KeyboardEvent) => {
        if (event.key === '1') selectedColor = [1.0, 0.0, 0.0]; // Red
        else if (event.key === '2') selectedColor = [0.0, 1.0, 0.0]; // Green
        else if (event.key === '3') selectedColor = [0.0, 0.0, 1.0]; // Blue
        else if (event.key === '4') selectedColor = [1.0, 1.0, 0.0]; // Yellow
        else if (event.key === '5') selectedColor = [1.0, 0.0, 1.0]; // Magenta
        else if (event.key === '6') selectedColor = [0.0, 1.0, 1.0]; // Cyan
        else if (event.key === '9') selectedColor = [1.0, 1.0, 1.0]; // White
        else if (event.key === '0') selectedColor = [0.0, 0.0, 0.0]; // Erase color
        else if (event.key === ' ') running = !running; // Toggle simulation
    };

    canvas.addEventListener('mousedown', handleMouseDown);
    canvas.addEventListener('mouseup', handleMouseUp);
    canvas.addEventListener('mousemove', handleMouseMove);
    canvas.addEventListener('wheel', handleWheel, { passive: false });
    window.addEventListener('keydown', handleKeyDown);

    cleanupColorMix = () => {
        canvas.removeEventListener('mousedown', handleMouseDown);
        canvas.removeEventListener('mouseup', handleMouseUp);
        canvas.removeEventListener('mousemove', handleMouseMove);
        canvas.removeEventListener('wheel', handleWheel);
        window.removeEventListener('keydown', handleKeyDown);

        if (animationFrameId !== null) {
            cancelAnimationFrame(animationFrameId);
            animationFrameId = null;
        }
    };

    function addDrop(x, y) {
        let normalizedX = x / canvas.width;
        let normalizedY = 1.0 - y / canvas.height;

        if (selectedColor[0] === 0.0 && selectedColor[1] === 0.0 && selectedColor[2] === 0.0) {
            eraseColorAt(normalizedX, normalizedY);
            return;
        }

        let index = (dropIndex % maxDrops) * 2;
        dropPositions[index] = normalizedX;
        dropPositions[index + 1] = normalizedY;

        index = (dropIndex % maxDrops) * 3;
        dropColors[index] = selectedColor[0];
        dropColors[index + 1] = selectedColor[1];
        dropColors[index + 2] = selectedColor[2];

        dropTimes[dropIndex % maxDrops] = performance.now() * 0.001;
        dropIndex++;
    }

    function eraseColorAt(x, y) {
        for (let i = 0; i < maxDrops; i++) {
            let index = i * 2;
            let dropX = dropPositions[index];
            let dropY = dropPositions[index + 1];

            let dist = Math.hypot(dropX - x, dropY - y);
            if (dist < 0.05) { // Define erase radius
                dropColors[i * 3] = 0.0;
                dropColors[i * 3 + 1] = 0.0;
                dropColors[i * 3 + 2] = 0.0;
            }
        }
    }
    // Vertex Shader (GLSL ES compatible)
    const vertexShaderSource = `
    attribute vec2 position;
    void main() {
        gl_Position = vec4(position, 0.0, 1.0);
    }`;

    // Fragment Shader (GLSL ES compatible)
    const fragmentShaderSource = `
    precision mediump float;
    uniform float time;
    uniform vec2 drops[200];
    uniform vec3 colors[200];
    uniform float drop_times[200];
    uniform float diffusion;
    uniform float dissolution;
    uniform float gravity;
    
    void main() {
        vec2 uv = gl_FragCoord.xy / vec2(1280.0, 720.0);
        vec3 color = vec3(0.0);
        for (int i = 0; i < 200; i++) {
            float local_time = time - drop_times[i];
            vec2 dropPos = drops[i] - vec2(0.0, gravity * local_time);
            float dist = length(uv - dropPos);
            float ageFactor = exp(-local_time * dissolution);
            float spread = exp(-dist / diffusion) * ageFactor;
            color += colors[i] * spread;
        }
        gl_FragColor = vec4(color, 1.0);
    }`;

    // Compile shader function
    function createShader(gl, type, source) {
        const shader = gl.createShader(type);
        gl.shaderSource(shader, source);
        gl.compileShader(shader);
        if (!gl.getShaderParameter(shader, gl.COMPILE_STATUS)) {
            console.error(gl.getShaderInfoLog(shader));
            gl.deleteShader(shader);
            return null;
        }
        return shader;
    }

    // Create and link WebGL program
    const vertexShader = createShader(gl, gl.VERTEX_SHADER, vertexShaderSource);
    const fragmentShader = createShader(gl, gl.FRAGMENT_SHADER, fragmentShaderSource);
    const program = gl.createProgram();
    gl.attachShader(program, vertexShader);
    gl.attachShader(program, fragmentShader);
    gl.linkProgram(program);
    gl.useProgram(program);

    // Define fullscreen quad
    const vertices = new Float32Array([
        -1, -1, 1, -1, -1, 1,
        -1, 1, 1, -1, 1, 1
    ]);
    const vbo = gl.createBuffer();
    gl.bindBuffer(gl.ARRAY_BUFFER, vbo);
    gl.bufferData(gl.ARRAY_BUFFER, vertices, gl.STATIC_DRAW);
    const position = gl.getAttribLocation(program, 'position');
    gl.enableVertexAttribArray(position);
    gl.vertexAttribPointer(position, 2, gl.FLOAT, false, 0, 0);

    // Render loop
    function render(time) {
        if (cleanupColorMix === null) {
            return;
        }

        if (!running) {
            animationFrameId = requestAnimationFrame(render);
            return;
        }
        gl.clear(gl.COLOR_BUFFER_BIT);
        gl.uniform1f(gl.getUniformLocation(program, 'time'), time * 0.001);
        gl.uniform1f(gl.getUniformLocation(program, 'diffusion'), diffusion);
        gl.uniform1f(gl.getUniformLocation(program, 'dissolution'), dissolution);
        gl.uniform1f(gl.getUniformLocation(program, 'gravity'), gravity);

        // Update drop positions, colors, and times in shader
        gl.uniform2fv(gl.getUniformLocation(program, 'drops'), dropPositions);
        gl.uniform3fv(gl.getUniformLocation(program, 'colors'), dropColors);
        gl.uniform1fv(gl.getUniformLocation(program, 'drop_times'), dropTimes);

        gl.drawArrays(gl.TRIANGLES, 0, 6);
        animationFrameId = requestAnimationFrame(render);
    }
    animationFrameId = requestAnimationFrame(render);
}


</script>

<template>
</template>
