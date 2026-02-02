<script>
  import { onMount, onDestroy } from 'svelte';
  import { forceSimulation, forceX, forceY, forceCollide, forceManyBody } from 'd3-force';
  import Steps from './components/steps.svelte';
  import data from "./data/shadowfleet.json";
  
  let canvas;
  let ctx;
  let width = 550;
  let height = 550;
  let currentStep = 0;
  let dots = [];
  let simulation;
  let animationFrame;
  let mounted = false;
  
  // Define vessel properties - will be populated from JSON
  let FLAGS = [];
  let SIZE_GROUPS = [];
  const SANCTIONING_BODIES = ['USA', 'EU', 'UK', 'UN', 'Other'];
  const AGE_GROUPS = ['0-10 years', '11-20 years', '21-30 years', '31+ years'];
  
  // Color palettes
  let FLAG_COLORS = {};
  let SIZE_COLORS = {};
  const AGE_COLORS = ['#90E0EF', '#00B4D8', '#0077B6', '#03045E'];
  const SANCTION_COLORS = ['#2A9D8F', '#E76F51', '#F4A261', '#E9C46A', '#264653'];
  
  // Helper function to get age group index
  function getAgeGroupIndex(age) {
    if (age <= 10) return 0;
    if (age <= 20) return 1;
    if (age <= 30) return 2;
    return 3;
  }
  
  // Initialize dots from JSON data
  function initializeDots() {
    console.log(`Loading ${data.length} vessels from JSON`);
    
    // Extract unique flags and sizes from data
    const uniqueFlags = [...new Set(data.map(d => d.flag_country))].filter(f => f);
    const uniqueSizes = [...new Set(data.map(d => d.size_category))].filter(s => s);
    
    FLAGS = uniqueFlags;
    SIZE_GROUPS = uniqueSizes;
    
    console.log('Unique flags:', FLAGS);
    console.log('Unique sizes:', SIZE_GROUPS);
    
    // Create color mappings
    const flagColorPalette = ['#E63946', '#F77F00', '#06AED5', '#118AB2', '#073B4C', '#6A4C93', '#1982C4'];
    const sizeColorPalette = ['#FFB703', '#FB8500', '#E85D04', '#DC2F02', '#9D0208', '#D62828'];
    
    FLAGS.forEach((flag, i) => {
      FLAG_COLORS[flag] = flagColorPalette[i % flagColorPalette.length];
    });
    
    SIZE_GROUPS.forEach((size, i) => {
      SIZE_COLORS[size] = sizeColorPalette[i % sizeColorPalette.length];
    });
    
    // Convert JSON data to dots
    dots = data.map((vessel, i) => {
      const flagIndex = FLAGS.indexOf(vessel.flag_country);
      const ageIndex = getAgeGroupIndex(vessel.age_years);
      const sizeIndex = SIZE_GROUPS.indexOf(vessel.size_category);
      const sanctionIndex = SANCTIONING_BODIES.indexOf(vessel.sanctioned_by);
      
      return {
        id: i,
        x: Math.random() * width,
        y: Math.random() * height,
        vx: 0,
        vy: 0,
        
        // From JSON
        vesselName: vessel.vessel_name,
        flag: vessel.flag_country,
        flagIndex: flagIndex >= 0 ? flagIndex : 0,
        age: vessel.age_years,
        ageIndex,
        size: vessel.size_category,
        sizeIndex: sizeIndex >= 0 ? sizeIndex : 0,
        sanctionedBy: vessel.sanctioned_by,
        sanctionIndex: sanctionIndex >= 0 ? sanctionIndex : 0,
        sanctionYear: vessel.sanction_year,
        latestSanctionYear: vessel.latest_sanction_year,
        sanctionCount: vessel.sanction_count
      };
    });
    
    console.log(`Initialized ${dots.length} dots`);
    console.log('Sample dot:', dots[0]);
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
        return FLAG_COLORS[dot.flag] || '#073B4C';
      case 2:
        return AGE_COLORS[dot.ageIndex];
      case 3:
        return SIZE_COLORS[dot.size] || '#9D0208';
      case 4:
        return SANCTION_COLORS[dot.sanctionIndex];
      case 5:
        const yearProgress = (dot.latestSanctionYear - 2004) / 21;
        return `hsl(${240 - yearProgress * 120}, 70%, 60%)`;
      default:
        return '#4A90E2';
    }
  }
  
  // Get radius for dot based on step
  function getDotRadius(dot, step) {
    const isMobile = width < 450;
    const mobileScale = isMobile ? 0.6 : 1;
    
    if (step === 4 || step === 5) {
      // Scale radius based on sanction count (1-9)
      return (1.5 + (dot.sanctionCount - 1) * 0.5) * mobileScale;
    }
    return 2.5 * mobileScale;
  }
  
  // Update simulation based on current step
  function updateSimulation(step) {
    if (!mounted || dots.length === 0) return;
    
    console.log(`Updating simulation for step ${step}`);
    
    if (simulation) {
      simulation.stop();
    }
    
    simulation = forceSimulation(dots);
    
    const isMobile = width < 450;
    const collisionScale = isMobile ? 0.6 : 1;
    const chargeStrength = isMobile ? -0.5 : -1.5;
    
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
            const spacing = width / (FLAGS.length + 1);
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
          .force('collide', forceCollide(d => getDotRadius(d, 4) + 1))
          .force('charge', forceManyBody().strength(chargeStrength));
        break;
        
      case 5:
        const xMargin = isMobile ? 0.15 : 0.05;
        const xRange = isMobile ? 0.7 : 0.9;
        simulation
          .force('x', forceX(d => {
            const yearProgress = (d.latestSanctionYear - 2020) / 6;
            return width * xMargin + (width * xRange * yearProgress);
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
    max-width: 1200px;
    margin: 0 auto;
  }

  section {
    position: relative;
  }
  
  .canvas-container {
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