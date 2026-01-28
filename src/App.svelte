<script>
  import { onMount, onDestroy } from 'svelte';
  import { forceSimulation, forceX, forceY, forceCollide, forceManyBody } from 'd3-force';
  import Steps from './components/steps.svelte';
  
  let canvas;
  let ctx;
  let width = 550;
  let height = 550;
  let currentStep = 0;
  let dots = [];
  let simulation;
  let animationFrame;
  let mounted = false;
  
  const TOTAL_VESSELS = 1300;
  
  // Define vessel properties
  const FLAGS = ['Panama', 'Liberia', 'Marshall Islands', 'Gabon', 'Other'];
  const AGE_GROUPS = ['0-5 years', '6-10 years', '11-15 years', '16+ years'];
  const SIZE_GROUPS = ['Small', 'Medium', 'Large', 'Very Large', 'Mega'];
  const SANCTIONING_BODIES = ['USA', 'EU', 'UK', 'UN', 'Other'];
  
  // Color palettes
  const FLAG_COLORS = ['#E63946', '#F77F00', '#06AED5', '#118AB2', '#073B4C'];
  const AGE_COLORS = ['#90E0EF', '#00B4D8', '#0077B6', '#03045E'];
  const SIZE_COLORS = ['#FFB703', '#FB8500', '#E85D04', '#DC2F02', '#9D0208'];
  const SANCTION_COLORS = ['#2A9D8F', '#E76F51', '#F4A261', '#E9C46A', '#264653'];
  
  // Initialize vessels with properties
  function initializeDots() {
    dots = Array.from({ length: TOTAL_VESSELS }, (_, i) => {
      const flagIndex = i < 260 ? 0 : i < 520 ? 1 : i < 780 ? 2 : i < 1040 ? 3 : 4;
      const ageIndex = Math.floor(Math.random() * AGE_GROUPS.length);
      const sizeIndex = Math.floor(Math.random() * SIZE_GROUPS.length);
      const sanctionIndex = Math.floor(Math.random() * SANCTIONING_BODIES.length);
      const sanctionYear = 2004 + Math.floor(Math.random() * 21);
      const randomSize = Math.floor(Math.random() * 5) + 1; // Random 1-5
      
      return {
        id: i,
        x: Math.random() * width,
        y: Math.random() * height,
        vx: 0,
        vy: 0,
        radius: 2.5,
        flag: FLAGS[flagIndex],
        flagIndex,
        age: AGE_GROUPS[ageIndex],
        ageIndex,
        size: SIZE_GROUPS[sizeIndex],
        sizeIndex,
        sanctionedBy: SANCTIONING_BODIES[sanctionIndex],
        sanctionIndex,
        sanctionYear,
        randomSize
      };
    });
    console.log(`Initialized ${dots.length} dots`);
  }
  
  // Setup high-DPI canvas
  function setupCanvas() {
    if (!canvas) return;
    
    const dpr = window.devicePixelRatio || 1;
    const rect = canvas.getBoundingClientRect();
    
    // Force square canvas - use the smaller dimension
    const size = Math.min(rect.width, rect.height);
    width = size;
    height = size;
    
    canvas.width = width * dpr;
    canvas.height = height * dpr;
    
    ctx = canvas.getContext('2d');
    ctx.scale(dpr, dpr);
    
    canvas.style.width = `${width}px`;
    canvas.style.height = `${height}px`;
    
    console.log(`Canvas setup: ${width}x${height}, DPR: ${dpr}`);
  }
  
  // Get color for dot based on step
  function getDotColor(dot, step) {
    switch(step) {
      case 0:
        return '#4A90E2';
      case 1:
        return FLAG_COLORS[dot.flagIndex];
      case 2:
        return AGE_COLORS[dot.ageIndex];
      case 3:
        return SIZE_COLORS[dot.sizeIndex];
      case 4:
        return SANCTION_COLORS[dot.sanctionIndex];
      case 5:
        const yearProgress = (dot.sanctionYear - 2004) / 20;
        return `hsl(${240 - yearProgress * 120}, 70%, 60%)`;
      default:
        return '#4A90E2';
    }
  }
  
  // Get radius for dot based on step
  function getDotRadius(dot, step) {
    // Scale down on mobile
    const isMobile = width < 450;
    const mobileScale = isMobile ? 0.6 : 1;
    
    if (step === 5) {
      // Scale radius based on random size (1-5)
      return (1.5 + dot.randomSize * 0.8) * mobileScale; // Radius from 2.3 to 5.5
    }
    return 2.5 * mobileScale; // Default radius for other steps
  }
  
  // Update simulation based on current step
  function updateSimulation(step) {
    if (!mounted || dots.length === 0) return;
    
    console.log(`Updating simulation for step ${step}`);
    
    if (simulation) {
      simulation.stop();
    }
    
    simulation = forceSimulation(dots);
    
    // Adjust collision radius for mobile
    const isMobile = width < 450;
    const collisionScale = isMobile ? 0.6 : 1;
    const chargeStrength = isMobile ? -0.5 : -1.2; // Stronger repulsion on larger screens
    
    switch(step) {
      case 0:
        simulation
          .force('x', forceX(width / 2).strength(0.05))
          .force('y', forceY(height / 2).strength(0.05))
          .force('collide', forceCollide(4 * collisionScale))
          .force('charge', forceManyBody().strength(chargeStrength));
        break;
        
      case 1:
        simulation
          .force('x', forceX(d => {
            const spacing = width / 6;
            return spacing * (d.flagIndex + 1);
          }).strength(0.1))
          .force('y', forceY(height / 2).strength(0.1))
          .force('collide', forceCollide(3 * collisionScale))
          .force('charge', forceManyBody().strength(chargeStrength));
        break;
        
      case 2:
        simulation
          .force('x', forceX(width / 2).strength(0.05))
          .force('y', forceY(d => {
            const spacing = height / 5;
            return spacing * (d.ageIndex + 1);
          }).strength(0.1))
          .force('collide', forceCollide(3 * collisionScale))
          .force('charge', forceManyBody().strength(chargeStrength));
        break;
        
      case 3:
        simulation
          .force('x', forceX(d => {
            const angle = (d.sizeIndex / SIZE_GROUPS.length) * Math.PI * 2;
            return width / 2 + Math.cos(angle) * (Math.min(width, height) * 0.25);
          }).strength(0.1))
          .force('y', forceY(d => {
            const angle = (d.sizeIndex / SIZE_GROUPS.length) * Math.PI * 2;
            return height / 2 + Math.sin(angle) * (Math.min(width, height) * 0.25);
          }).strength(0.1))
          .force('collide', forceCollide(3 * collisionScale))
          .force('charge', forceManyBody().strength(chargeStrength));
        break;
        
      case 4:
        simulation
          .force('x', forceX(d => {
            const col = d.sanctionIndex % 3;
            return width * 0.25 + (col * width * 0.25);
          }).strength(0.1))
          .force('y', forceY(d => {
            const row = Math.floor(d.sanctionIndex / 3);
            return height * 0.3 + (row * height * 0.35);
          }).strength(0.1))
          .force('collide', forceCollide(3 * collisionScale))
          .force('charge', forceManyBody().strength(chargeStrength));
        break;
        
      case 5:
        simulation
          .force('x', forceX(d => {
            const yearProgress = (d.sanctionYear - 2004) / 20;
            return width * 0.15 + (width * 0.7 * yearProgress);
          }).strength(0.15))
          .force('y', forceY(d => {
            return height / 2 + (Math.random() - 0.5) * height * 0.5;
          }).strength(0.08))
          .force('collide', forceCollide(d => getDotRadius(d, 5) + 1))
          .force('charge', forceManyBody().strength(-0.3));
        break;
        
      default:
        simulation
          .force('x', forceX(width / 2).strength(0.05))
          .force('y', forceY(height / 2).strength(0.05))
          .force('collide', forceCollide(3 * collisionScale));
    }
    
    simulation.alpha(1).restart();
  }
  
  // Render loop with requestAnimationFrame
  function render() {
    if (!ctx || !mounted) return;
    
    ctx.clearRect(0, 0, width, height);
    
    dots.forEach(dot => {
      const radius = getDotRadius(dot, currentStep);
      ctx.beginPath();
      ctx.arc(dot.x, dot.y, radius, 0, Math.PI * 2);
      ctx.fillStyle = getDotColor(dot, currentStep);
      ctx.fill();
    });
    
    animationFrame = requestAnimationFrame(render);
  }
  
  // Watch for step changes
  $: {
    console.log('Current step changed to:', currentStep);
    if (mounted && dots.length > 0) {
      updateSimulation(currentStep);
    }
  }
  
  onMount(() => {
    console.log('App mounted');
    mounted = true;
    setupCanvas();
    initializeDots();
    updateSimulation(0);
    render();
    
    const handleResize = () => {
      setupCanvas();
      if (dots.length > 0) {
        // Reposition dots proportionally
        const oldWidth = width;
        const oldHeight = height;
        dots.forEach(dot => {
          dot.x = (dot.x / oldWidth) * width;
          dot.y = (dot.y / oldHeight) * height;
        });
        updateSimulation(currentStep);
      }
    };
    
    window.addEventListener('resize', handleResize);
    
    return () => {
      mounted = false;
      if (animationFrame) {
        cancelAnimationFrame(animationFrame);
      }
      if (simulation) {
        simulation.stop();
      }
      window.removeEventListener('resize', handleResize);
    };
  });
</script>

<main>
  <section>
    <div class="sticky">
  <div class="canvas-container">
    <canvas bind:this={canvas}></canvas>
  </div>
    </div>
  <Steps bind:currentStep />
</section>
</main>

<style>
  :global(body) {
    margin: 0;
    padding: 0;
    background: #f8f9fa;
    overflow-x: hidden;
  }
  
  main {
    /* position: relative;
    width: 100%;
    min-height: 100vh; */
    max-width: 1200px;
    margin: 0 auto;
  }

  section {
    position: relative;
  }
  
  .canvas-container {
    /* position: fixed;
    top: 0;
    left: 0;
    width: 100%;
    height: 100vh;
    pointer-events: none;
    z-index: 1; */
    display: flex;
    align-items: center;
    justify-content: center;
    width: 100%;
    max-width: 1200px;
    margin: auto;
  }

    .sticky {
    position: sticky;
    z-index: 1;
    height: 90vh;
    top: 5vh;
    margin-bottom: 1rem;
    display: flex;
    align-items: center;
    justify-content: center;
  }
  
  canvas {
    width: 100%;
    height: 100%;
    max-width: 100%;
    max-height: 100%;
    aspect-ratio: 1 / 1;
    display: block;
  }
</style>