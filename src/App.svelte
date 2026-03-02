<script lang="ts">
  import CloudBackground from './lib/CloudBackground.svelte';

  const text = 'On Cloud Nine';
  const letters = text.split('');

  const rainbowColors = [
    '#ff6b6b', '#ff9f43', '#feca57', '#48dbfb',
    '#0abde3', '#a29bfe', '#fd79a8', '#e056fd',
    '#6c5ce7', '#00cec9', '#55efc4', '#fdcb6e', '#e17055',
  ];

  let scattered = $state(false);
  let idleTimer: ReturnType<typeof setTimeout>;
  let cycleTimer: ReturnType<typeof setTimeout>;

  let trailCanvas: HTMLCanvasElement;
  let thunderFlash: HTMLDivElement;
  let letterEls: HTMLSpanElement[] = [];
  let trailRaf: number;
  let vortexRaf: number;
  let particles: { x: number; y: number; color: string; alpha: number }[] = [];

  // Per-letter vortex state
  let vortex: {
    angle: number;       // current angle in radians
    radius: number;      // current distance from origin
    targetRadius: number; // how far out it should go
    spinSpeed: number;   // radians per frame
    growSpeed: number;    // radius growth per frame
    rotation: number;     // letter self-rotation
    rotSpeed: number;
    color: string;
  }[] = [];

  function triggerThunder() {
    if (!thunderFlash) return;
    thunderFlash.style.opacity = '0.7';
    setTimeout(() => { thunderFlash.style.opacity = '0'; }, 60);
    setTimeout(() => { thunderFlash.style.opacity = '0.4'; }, 120);
    setTimeout(() => { thunderFlash.style.opacity = '0'; }, 200);

    document.body.style.transition = 'none';
    document.body.style.transform = 'translate(4px, -3px)';
    setTimeout(() => { document.body.style.transform = 'translate(-3px, 2px)'; }, 50);
    setTimeout(() => { document.body.style.transform = 'translate(2px, 4px)'; }, 100);
    setTimeout(() => { document.body.style.transform = 'translate(-4px, -2px)'; }, 150);
    setTimeout(() => {
      document.body.style.transition = 'transform 0.15s ease-out';
      document.body.style.transform = 'translate(0, 0)';
    }, 200);
  }

  function startVortex() {
    const maxR = Math.min(window.innerWidth, window.innerHeight) * 0.45;

    vortex = letters.map((l, i) => {
      if (l === ' ') return { angle: 0, radius: 0, targetRadius: 0, spinSpeed: 0, growSpeed: 0, rotation: 0, rotSpeed: 0, color: '' };
      const angle = (i / letters.length) * Math.PI * 2 + (Math.random() - 0.5) * 1.2;
      return {
        angle,
        radius: 0,
        targetRadius: maxR * (0.5 + Math.random() * 0.5),
        spinSpeed: (0.02 + Math.random() * 0.03) * (Math.random() > 0.5 ? 1 : -1),
        growSpeed: 3 + Math.random() * 4,
        rotation: 0,
        rotSpeed: (Math.random() - 0.5) * 8,
        color: rainbowColors[Math.floor(Math.random() * rainbowColors.length)],
      };
    });

    function animate() {
      for (let i = 0; i < letterEls.length; i++) {
        const el = letterEls[i];
        const v = vortex[i];
        if (!el || !v || !v.color) continue;

        // Grow radius toward target, spin the angle
        if (v.radius < v.targetRadius) {
          v.radius = Math.min(v.radius + v.growSpeed, v.targetRadius);
        }
        v.angle += v.spinSpeed;
        v.rotation += v.rotSpeed;
        // Slow down spin as it reaches the edge
        v.spinSpeed *= 0.998;
        v.rotSpeed *= 0.995;

        const x = Math.cos(v.angle) * v.radius;
        const y = Math.sin(v.angle) * v.radius;

        el.style.transform = `translate(${x}px, ${y}px) rotate(${v.rotation}deg)`;
        el.style.color = v.color;
        el.style.textShadow = `0 0 20px ${v.color}, 0 0 40px ${v.color}`;
      }

      vortexRaf = requestAnimationFrame(animate);
    }

    vortexRaf = requestAnimationFrame(animate);
  }

  function stopVortex() {
    cancelAnimationFrame(vortexRaf);

    // Animate back to origin
    function returnHome() {
      let allHome = true;

      for (let i = 0; i < letterEls.length; i++) {
        const el = letterEls[i];
        const v = vortex[i];
        if (!el || !v || !v.color) continue;

        v.radius *= 0.9;
        v.angle += v.spinSpeed * 2;
        v.rotation *= 0.9;

        if (v.radius > 1) {
          allHome = false;
        } else {
          v.radius = 0;
          v.rotation = 0;
        }

        const x = Math.cos(v.angle) * v.radius;
        const y = Math.sin(v.angle) * v.radius;

        el.style.transform = `translate(${x}px, ${y}px) rotate(${v.rotation}deg)`;

        if (v.radius === 0) {
          el.style.color = '';
          el.style.textShadow = '';
        }
      }

      if (allHome) return;
      vortexRaf = requestAnimationFrame(returnHome);
    }

    vortexRaf = requestAnimationFrame(returnHome);
  }

  function startTrail() {
    const ctx = trailCanvas.getContext('2d');
    if (!ctx) return;
    trailCanvas.width = window.innerWidth;
    trailCanvas.height = window.innerHeight;
    particles = [];

    function draw() {
      ctx.clearRect(0, 0, trailCanvas.width, trailCanvas.height);

      for (let i = 0; i < letterEls.length; i++) {
        const el = letterEls[i];
        const v = vortex[i];
        if (!el || !v?.color) continue;
        const rect = el.getBoundingClientRect();
        particles.push({
          x: rect.left + rect.width / 2,
          y: rect.top + rect.height / 2,
          color: v.color,
          alpha: 1,
        });
      }

      for (let j = particles.length - 1; j >= 0; j--) {
        const p = particles[j];
        p.alpha -= 0.02;
        if (p.alpha <= 0) {
          particles.splice(j, 1);
          continue;
        }
        ctx.globalAlpha = p.alpha;
        ctx.beginPath();
        ctx.arc(p.x, p.y, 5, 0, Math.PI * 2);
        ctx.fillStyle = p.color;
        ctx.shadowColor = p.color;
        ctx.shadowBlur = 12;
        ctx.fill();
      }
      ctx.globalAlpha = 1;
      ctx.shadowBlur = 0;

      trailRaf = requestAnimationFrame(draw);
    }

    trailRaf = requestAnimationFrame(draw);
  }

  function stopTrail() {
    cancelAnimationFrame(trailRaf);
    const ctx = trailCanvas?.getContext('2d');
    if (!ctx) return;

    function fadeOut() {
      ctx.clearRect(0, 0, trailCanvas.width, trailCanvas.height);
      for (let j = particles.length - 1; j >= 0; j--) {
        const p = particles[j];
        p.alpha -= 0.04;
        if (p.alpha <= 0) {
          particles.splice(j, 1);
          continue;
        }
        ctx.globalAlpha = p.alpha;
        ctx.beginPath();
        ctx.arc(p.x, p.y, 5, 0, Math.PI * 2);
        ctx.fillStyle = p.color;
        ctx.shadowColor = p.color;
        ctx.shadowBlur = 12;
        ctx.fill();
      }
      ctx.globalAlpha = 1;
      ctx.shadowBlur = 0;
      if (particles.length > 0) requestAnimationFrame(fadeOut);
    }
    requestAnimationFrame(fadeOut);
  }

  function resetIdleTimer() {
    clearTimeout(idleTimer);
    clearTimeout(cycleTimer);
    if (!scattered) return;
    idleTimer = setTimeout(() => {
      // No mouse movement for 5s — reassemble then restart
      reassemble();
      cycleTimer = setTimeout(() => {
        scatter();
      }, 1500);
    }, 5000);
  }

  function scatter() {
    scattered = true;
    startVortex();
    startTrail();
    triggerThunder();
    resetIdleTimer();
  }

  function reassemble() {
    scattered = false;
    clearTimeout(idleTimer);
    clearTimeout(cycleTimer);
    stopVortex();
    stopTrail();
  }

  function onMouseMove() {
    if (scattered) resetIdleTimer();
  }
</script>

<svelte:window onmousemove={onMouseMove} />

<CloudBackground />

<div bind:this={thunderFlash} class="thunder-flash"></div>
<canvas bind:this={trailCanvas} class="trail-canvas"></canvas>

<div class="content-overlay">
  <!-- svelte-ignore a11y_no_static_element_interactions -->
  <h1
    class="branding"
    onmouseenter={scatter}
    onmouseleave={reassemble}
  >
    {#each letters as letter, i}
      {#if letter === ' '}
        <span class="letter space">&nbsp;</span>
      {:else}
        <span
          bind:this={letterEls[i]}
          class="letter"
        >{letter}</span>
      {/if}
    {/each}
  </h1>
</div>

<style>
  .thunder-flash {
    position: fixed;
    inset: 0;
    z-index: 3;
    background: radial-gradient(ellipse at 50% 40%, rgba(200, 210, 255, 0.9), rgba(150, 170, 255, 0.4), transparent 70%);
    opacity: 0;
    pointer-events: none;
  }

  .trail-canvas {
    position: fixed;
    inset: 0;
    z-index: 1;
    width: 100%;
    height: 100%;
    pointer-events: none;
  }

  .letter {
    display: inline-block;
    cursor: default;
    will-change: transform;
  }

  .letter.space {
    width: 0.3em;
  }
</style>
