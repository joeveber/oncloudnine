<script lang="ts">
  import { untrack } from 'svelte';

  let canvas: HTMLCanvasElement;
  let gl: WebGL2RenderingContext | null = null;
  let program: WebGLProgram | null = null;
  let animId: number;
  let resizeTimer: ReturnType<typeof setTimeout>;

  const isMobile = typeof navigator !== 'undefined' && /Mobi|Android/i.test(navigator.userAgent);
  const octaves = isMobile ? 3 : 5;
  const targetFps = isMobile ? 30 : 60;
  const maxDpr = isMobile ? 1.5 : 2;

  const VERT = `#version 300 es
    void main() {
      vec2 pos = vec2(gl_VertexID & 1, gl_VertexID >> 1) * 4.0 - 1.0;
      gl_Position = vec4(pos, 0.0, 1.0);
    }`;

  const FRAG = `#version 300 es
    precision mediump float;
    uniform float u_time;
    uniform vec2 u_resolution;
    uniform int u_octaves;
    out vec4 fragColor;

    // Hash-based 3D noise
    vec3 hash3(vec3 p) {
      p = vec3(
        dot(p, vec3(127.1, 311.7, 74.7)),
        dot(p, vec3(269.5, 183.3, 246.1)),
        dot(p, vec3(113.5, 271.9, 124.6))
      );
      return -1.0 + 2.0 * fract(sin(p) * 43758.5453123);
    }

    float noise3d(vec3 p) {
      vec3 i = floor(p);
      vec3 f = fract(p);
      vec3 u = f * f * (3.0 - 2.0 * f);

      return mix(
        mix(
          mix(dot(hash3(i + vec3(0,0,0)), f - vec3(0,0,0)),
              dot(hash3(i + vec3(1,0,0)), f - vec3(1,0,0)), u.x),
          mix(dot(hash3(i + vec3(0,1,0)), f - vec3(0,1,0)),
              dot(hash3(i + vec3(1,1,0)), f - vec3(1,1,0)), u.x), u.y),
        mix(
          mix(dot(hash3(i + vec3(0,0,1)), f - vec3(0,0,1)),
              dot(hash3(i + vec3(1,0,1)), f - vec3(1,0,1)), u.x),
          mix(dot(hash3(i + vec3(0,1,1)), f - vec3(0,1,1)),
              dot(hash3(i + vec3(1,1,1)), f - vec3(1,1,1)), u.x), u.y),
        u.z
      );
    }

    float fbm(vec3 p, int octaves) {
      float val = 0.0;
      float amp = 0.5;
      float freq = 1.0;
      for (int i = 0; i < 5; i++) {
        if (i >= octaves) break;
        val += amp * noise3d(p * freq);
        freq *= 2.0;
        amp *= 0.5;
      }
      return val;
    }

    void main() {
      vec2 uv = gl_FragCoord.xy / u_resolution;
      float t = u_time * 0.04;

      // Sky gradient: dark navy → deep blue → muted steel
      vec3 sky = mix(
        vec3(0.03, 0.05, 0.15),
        mix(vec3(0.06, 0.1, 0.25), vec3(0.12, 0.18, 0.35), uv.y),
        uv.y
      );

      // Cloud layers at different scales and drift speeds
      float c1 = fbm(vec3(uv * 3.0 + vec2(t * 0.5, t * 0.2), t * 0.3), u_octaves);
      float c2 = fbm(vec3(uv * 5.0 + vec2(t * 0.3, -t * 0.15), t * 0.2 + 10.0), u_octaves);
      float c3 = fbm(vec3(uv * 8.0 + vec2(-t * 0.2, t * 0.1), t * 0.15 + 20.0), u_octaves);

      // Cloud colors: dim silver, slate blue, muted purple
      vec3 cloud1 = vec3(0.55, 0.55, 0.65) * smoothstep(-0.1, 0.5, c1);
      vec3 cloud2 = vec3(0.3, 0.35, 0.55) * smoothstep(0.0, 0.55, c2) * 0.6;
      vec3 cloud3 = vec3(0.45, 0.35, 0.55) * smoothstep(0.05, 0.6, c3) * 0.3;

      vec3 color = sky + cloud1 + cloud2 + cloud3;

      // Vignette
      vec2 vc = uv - 0.5;
      float vignette = 1.0 - dot(vc, vc) * 0.8;
      color *= vignette;

      fragColor = vec4(color, 1.0);
    }`;

  function createShader(gl: WebGL2RenderingContext, type: number, source: string): WebGLShader | null {
    const shader = gl.createShader(type);
    if (!shader) return null;
    gl.shaderSource(shader, source);
    gl.compileShader(shader);
    if (!gl.getShaderParameter(shader, gl.COMPILE_STATUS)) {
      console.error('Shader error:', gl.getShaderInfoLog(shader));
      gl.deleteShader(shader);
      return null;
    }
    return shader;
  }

  function resize() {
    if (!gl || !canvas) return;
    const dpr = Math.min(window.devicePixelRatio, maxDpr);
    canvas.width = canvas.clientWidth * dpr;
    canvas.height = canvas.clientHeight * dpr;
    gl.viewport(0, 0, canvas.width, canvas.height);
  }

  $effect(() => {
    gl = canvas.getContext('webgl2', { alpha: false, antialias: false });
    if (!gl) {
      canvas.style.background = 'linear-gradient(135deg, #080d1a 0%, #0f1a33 50%, #1a2844 100%)';
      return;
    }

    const vs = createShader(gl, gl.VERTEX_SHADER, VERT);
    const fs = createShader(gl, gl.FRAGMENT_SHADER, FRAG);
    if (!vs || !fs) return;

    program = gl.createProgram();
    if (!program) return;
    gl.attachShader(program, vs);
    gl.attachShader(program, fs);
    gl.linkProgram(program);

    if (!gl.getProgramParameter(program, gl.LINK_STATUS)) {
      console.error('Program link error:', gl.getProgramInfoLog(program));
      return;
    }

    gl.useProgram(program);

    const uTime = gl.getUniformLocation(program, 'u_time');
    const uRes = gl.getUniformLocation(program, 'u_resolution');
    const uOctaves = gl.getUniformLocation(program, 'u_octaves');

    gl.uniform1i(uOctaves, octaves);
    resize();

    const frameInterval = 1000 / targetFps;
    let lastFrame = 0;
    const start = performance.now();

    function render(now: number) {
      animId = requestAnimationFrame(render);
      if (now - lastFrame < frameInterval) return;
      lastFrame = now;

      if (!gl || !program) return;
      gl.uniform1f(uTime, (now - start) / 1000);
      gl.uniform2f(uRes, gl.drawingBufferWidth, gl.drawingBufferHeight);
      gl.drawArrays(gl.TRIANGLES, 0, 3);
    }

    animId = requestAnimationFrame(render);

    return () => {
      cancelAnimationFrame(animId);
      if (gl && program) {
        gl.deleteProgram(program);
      }
    };
  });

  $effect(() => {
    const onResize = () => {
      clearTimeout(resizeTimer);
      resizeTimer = setTimeout(() => untrack(() => resize()), 150);
    };
    window.addEventListener('resize', onResize);
    return () => window.removeEventListener('resize', onResize);
  });
</script>

<canvas bind:this={canvas} class="cloud-canvas"></canvas>
