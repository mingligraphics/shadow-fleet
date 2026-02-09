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
  const SANCTIONING_BODIES = ['EU', 'U.K.', 'U.S.', 'Other'];
  const AGE_GROUPS = ['0-10 years', '11-20', '21-30', '31 and more'];
  
  // Color palettes from WSJ style guide
  const PEACOCK_COLORS = ['#e3f0f6', '#c3e3ed', '#88c9d7', '#00b3c9', '#0091a2', '#006f7a'];
  const GINGER_COLORS = ['#f7deca', '#f2c7a1', '#e8ac76', '#dc6c00', '#b25200', '#883800'];
  const TIMELINE_COLORS = ['#f7deca', '#f2c7a1', '#e8ac76', '#dc6c00', '#b25200', '#883800', '#883800']; // 2020-2026: lightest to darkest ginger
  
  // Color palettes
  let FLAG_COLORS = {};
  let SIZE_COLORS = {};
  const AGE_COLORS = ['#88c9d7', '#00b3c9', '#0091a2', '#727272']; // 0-10 lightest, 21-30 darkest peacock, 31+ gray
  const SANCTION_COLORS = {
    'U.S.': '#00b3c9',
    'EU': '#dc6c00',    // Ginger
    'U.K.': '#e8ac76',    // Lighter ginger
    'Other': '#727272'  // Gray
  };
  
  // Helper function to get age group index
  function getAgeGroupIndex(age) {
    if (age <= 10) return 0;
    if (age <= 20) return 1;
    if (age <= 30) return 2;
    return 3;
  }
  
  // Helper function to draw text with white outline/stroke
  function drawTextWithBackground(text, x, y, fontSize = 15, textAlign = 'center') {
    ctx.font = `${fontSize}px Retina, sans-serif`;
    ctx.textAlign = textAlign;
    ctx.textBaseline = 'middle';
    
    // Draw white outline/stroke around text
    ctx.strokeStyle = 'rgba(255, 255, 255, 0.9)';
    ctx.lineWidth = 4; // Thickness of the white outline
    ctx.strokeText(text, x, y);
    
    // Draw the actual text on top
    ctx.fillStyle = '#333';
    ctx.fillText(text, x, y);
  }
  
  // Initialize dots from JSON data
  function initializeDots() {
    console.log(`Loading ${data.length} vessels from JSON`);
    
    // Extract unique flags and sizes from data
    const uniqueSizes = [...new Set(data.map(d => d.size_category))].filter(s => s);
    
    // Order sizes from smallest to biggest (clockwise arrangement), Other always last
    const sizeOrderSmallToBig = ['Handysize/Handymax', 'Aframax', 'Suezmax', 'VLCC/ULCC'];
    SIZE_GROUPS = sizeOrderSmallToBig.filter(size => uniqueSizes.includes(size));
    
    // Add any sizes not in the predefined order (except Other)
    uniqueSizes.forEach(size => {
      if (!SIZE_GROUPS.includes(size) && size !== 'Other') {
        SIZE_GROUPS.push(size);
      }
    });
    
    // Add Other at the end if it exists
    if (uniqueSizes.includes('Other')) {
      SIZE_GROUPS.push('Other');
    }
    const flagCounts = {};
    data.forEach(d => {
      flagCounts[d.flag_country] = (flagCounts[d.flag_country] || 0) + 1;
    });
    
    const sortedFlags = Object.entries(flagCounts)
      .filter(([flag]) => flag !== 'Other')
      .sort((a, b) => b[1] - a[1])
      .map(([flag]) => flag);
    
    // Add "Other" at the end if it exists
    if (flagCounts['Other']) {
      sortedFlags.push('Other');
    }
    
    FLAGS = sortedFlags;
    
    console.log('Flags ordered by size (Other last):', FLAGS, flagCounts);
    console.log('Unique sizes:', SIZE_GROUPS);
    
    // Create color mappings
    // Map specific flags to specific colors
    const flagSpecificColors = {
      'Russia': '#F48474',
      'Cameroon': '#DC7300',
      'Panama': '#00b3c9',
      'Iran': '#76BCE8',
      'Other': '#bfbfbf'
    };
    
    // Map sizes to Ginger colors (bigger = darker)
    const sizeToGinger = {
      'VLCC/ULCC': '#883800',        // Biggest - darkest
      'Suezmax': '#b25200',
      'Aframax': '#dc6c00',
      'Handysize/Handymax': '#e8ac76',
      'Other': '#bfbfbf'
    };
    
    FLAGS.forEach((flag) => {
      if (flagSpecificColors[flag]) {
        FLAG_COLORS[flag] = flagSpecificColors[flag];
      } else {
        // Fallback for any other flags
        FLAG_COLORS[flag] = '#00b3c9';
      }
    });
    
    SIZE_GROUPS.forEach((size) => {
      if (sizeToGinger[size]) {
        SIZE_COLORS[size] = sizeToGinger[size];
      } else if (size === 'Other') {
        SIZE_COLORS[size] = '#727272';
      } else {
        // Fallback
        SIZE_COLORS[size] = '#dc6c00';
      }
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
        return FLAG_COLORS[dot.flag] || '#00b3c9';
      case 1:
        return AGE_COLORS[dot.ageIndex];
      case 2:
        return SIZE_COLORS[dot.size] || '#727272';
      case 3:
        return SANCTION_COLORS[dot.sanctionedBy] || '#727272';
      case 4:
        // Timeline colors: all ginger shades, 2020 lightest to 2026 darkest
        const yearsSince2020 = Math.max(0, Math.min(dot.latestSanctionYear - 2020, 6));
        return TIMELINE_COLORS[yearsSince2020];
      default:
        return '#00b3c9';
    }
  }
  
  // Get radius for dot based on step
  function getDotRadius(dot, step) {
    const isMobile = width < 450;
    const mobileScale = isMobile ? 0.6 : 1;
    
    if (step === 3 || step === 4) {
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
          .force('x', forceX(d => {
            const spacing = width / (FLAGS.length + 1.5); // More spacing
            const baseX = spacing * (d.flagIndex + 1);
            // Keep dots within bounds with more padding
            return Math.max(60, Math.min(width - 60, baseX));
          }).strength(0.1))
          .force('y', forceY(height / 2).strength(0.1))
          .force('collide', forceCollide(3 * collisionScale))
          .force('charge', forceManyBody().strength(chargeStrength));
        break;
        
      case 1:
        simulation
          .force('x', forceX(width / 2).strength(0.05))
          .force('y', forceY(d => {
            const spacing = height / 5;
            return spacing * (d.ageIndex + 1);
          }).strength(0.1))
          .force('collide', forceCollide(3 * collisionScale))
          .force('charge', forceManyBody().strength(chargeStrength));
        break;
        
      case 2:
        simulation
          .force('x', forceX(d => {
            const angle = (d.sizeIndex / SIZE_GROUPS.length) * Math.PI * 2 - Math.PI / 2; // Start from top
            return width / 2 + Math.cos(angle) * (Math.min(width, height) * 0.25);
          }).strength(0.1))
          .force('y', forceY(d => {
            const angle = (d.sizeIndex / SIZE_GROUPS.length) * Math.PI * 2 - Math.PI / 2; // Start from top
            return height / 2 + Math.sin(angle) * (Math.min(width, height) * 0.25);
          }).strength(0.1))
          .force('collide', forceCollide(3 * collisionScale))
          .force('charge', forceManyBody().strength(chargeStrength));
        break;
        
      case 3:
        simulation
          .force('x', forceX(d => {
            const angle = (d.sanctionIndex / SANCTIONING_BODIES.length) * Math.PI * 2 - Math.PI / 2; // Start from top
            return width / 2 + Math.cos(angle) * (Math.min(width, height) * 0.18);
          }).strength(0.1))
          .force('y', forceY(d => {
            const angle = (d.sanctionIndex / SANCTIONING_BODIES.length) * Math.PI * 2 - Math.PI / 2; // Start from top
            return height / 2 + Math.sin(angle) * (Math.min(width, height) * 0.18);
          }).strength(0.1))
          .force('collide', forceCollide(d => getDotRadius(d, 3) + 1))
          .force('charge', forceManyBody().strength(chargeStrength));
        break;
        
      case 4:
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
          .force('collide', forceCollide(d => getDotRadius(d, 4) + 1))
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
    
    // Draw dots
    dots.forEach(dot => {
      const radius = getDotRadius(dot, currentStep);
      ctx.beginPath();
      ctx.arc(dot.x, dot.y, radius, 0, Math.PI * 2);
      ctx.fillStyle = getDotColor(dot, currentStep);
      ctx.fill();
    });
    
    // Draw labels for each step
    const isMobile = width < 450;
    const fontSize = isMobile ? 11 : 15;
    ctx.font = `${fontSize}px Retina, sans-serif`;
    ctx.textAlign = 'center';
    ctx.fillStyle = '#333';
    
    switch(currentStep) {
      case 0: // Flag labels - in the middle of groups with white background and counts
        // Calculate counts for each flag
        const flagCounts = {};
        FLAGS.forEach(flag => {
          flagCounts[flag] = dots.filter(d => d.flag === flag).length;
        });
        
        FLAGS.forEach((flag, i) => {
          const spacing = width / (FLAGS.length + 1.5);
          const x = spacing * (i + 1);
          const y = height / 2; // Middle of the group
          
          // Draw flag name with white background
          drawTextWithBackground(flag, x, y, fontSize);
          
          // Draw count below (except for Other)
          if (flag !== 'Other') {
            const count = flagCounts[flag];
            drawTextWithBackground(count.toString(), x, y + fontSize + 8, fontSize);
          }
        });
        break;
        
      case 1: // Age labels - left edge, spaced out more
        AGE_GROUPS.forEach((ageGroup, i) => {
          const spacing = height / 4.5; // More spacing between groups
          const x = 15; // Left edge
          let y = spacing * (i + 1);
          
          // Move first three groups up
          if (i < 2) {
            y -= 65;
          }
          if (i == 2) {
            y -= 25;
          }
          
          ctx.textAlign = 'left';
          ctx.fillText(ageGroup, x, y);
        });
        ctx.textAlign = 'center'; // Reset
        break;
        
      case 2: // Size labels (circular) - centered in groups with white background
        // Map size categories to tonnage ranges
        const tonnageInfo = {
          'Handysize/Handymax': '15,000-60,000 deadweight tonnage',
          'Aframax': '80,000-120,000',
          'Suezmax': '120,000-200,000',
          'VLCC/ULCC': 'more than 200,000'
        };
        
        SIZE_GROUPS.forEach((size, i) => {
          const angle = (i / SIZE_GROUPS.length) * Math.PI * 2 - Math.PI / 2; // Start from top
          const labelRadius = Math.min(width, height) * 0.25; // Match dot positioning radius
          const x = width / 2 + Math.cos(angle) * labelRadius;
          const y = height / 2 + Math.sin(angle) * labelRadius;
          
          // Draw size label with white background
          drawTextWithBackground(size, x, y, fontSize);
          
          // Draw tonnage info below if available
          if (tonnageInfo[size]) {
            const smallerFontSize = isMobile ? 9 : 11;
            drawTextWithBackground(tonnageInfo[size], x, y + fontSize + 6, smallerFontSize);
          }
        });
        break;
        
      case 3: // Sanction body labels - circular layout, centered in groups
        // Calculate counts for each sanctioning body
        const sanctionCounts = {};
        SANCTIONING_BODIES.forEach(body => {
          sanctionCounts[body] = dots.filter(d => d.sanctionedBy === body).length;
        });
        
        SANCTIONING_BODIES.forEach((body, i) => {
          const angle = (i / SANCTIONING_BODIES.length) * Math.PI * 2 - Math.PI / 2; // Start from top
          const labelRadius = Math.min(width, height) * 0.18; // Match dot positioning radius
          const x = width / 2 + Math.cos(angle) * labelRadius;
          const y = height / 2 + Math.sin(angle) * labelRadius;
          
          // Draw body name with white background
          drawTextWithBackground(body, x, y, fontSize);
          
          // Draw count below with white background
          const count = sanctionCounts[body];
          drawTextWithBackground(count.toString(), x, y + fontSize + 8, fontSize);
        });
        break;
        
      case 4: // Timeline labels with Retina Narrow font
        const startYear = 2020;
        const endYear = 2026;
        const xMargin = isMobile ? 0.15 : 0.05;
        const xRange = isMobile ? 0.7 : 0.9;
        
        ctx.font = '13px "Retina Narrow", sans-serif';
        ctx.fillStyle = '#222222';
        
        // Draw year labels
        for (let year = startYear; year <= endYear; year++) {
          const yearProgress = (year - startYear) / 6;
          const x = width * xMargin + (width * xRange * yearProgress);
          const y = height - 20;
          
          // Format year label
          let yearLabel;
          if (year === 2020) {
            yearLabel = "2016-'20";
          } else {
            yearLabel = `'${year.toString().slice(-2)}`;
          }
          
          ctx.fillText(yearLabel, x, y);
        }

// Add annotation for 2025 (desktop only)
if (!isMobile) {
  const vessels2025Count = dots.filter(d => d.latestSanctionYear === 2025).length;
  const year2025Progress = (2025 - startYear) / 6;
  const x2025 = width * xMargin + (width * xRange * year2025Progress);

  // Annotation text at top
  const annotationY = 80;
  ctx.font = '13px Retina, sans-serif';
  ctx.fillStyle = '#333';
  ctx.textAlign = 'center';
  const annotationText = `${vessels2025Count} vessels`;
  ctx.fillText(annotationText, width * 3 / 4, annotationY);
}

        // Draw line from annotation to 2025 bubbles
        // ctx.strokeStyle = '#666';
        // ctx.lineWidth = 1;
        // ctx.beginPath();
        // ctx.moveTo(x2025, annotationY + 10);
        // ctx.lineTo(x2025, annotationY + 30);
        // ctx.stroke();
        // ctx.setLineDash([]); 
        
        // Reset font for next render
        ctx.font = `${fontSize}px Retina, sans-serif`;
        ctx.fillStyle = '#333';
        break;
    }
    
    // Draw legend for steps 3 and 4 (bubble size represents sanction count)
    if (currentStep === 3 || currentStep === 4) {
      const legendX = isMobile ? 15 : 20;
      const legendY = isMobile ? 30 : 90;
      const bubbleSpacing = isMobile ? 20 : 25;
      
      ctx.font = `${isMobile ? 10 : 12}px Retina, sans-serif`;
      ctx.textAlign = 'left';
      ctx.fillStyle = '#666';
      
      // Legend title
      ctx.fillText('Number of sanctions', legendX, legendY);
      
      // Draw sample bubbles horizontally
      const sampleCounts = [1, 3, 5, 9];
      const sampleLabels = ['1', '3', '5', '9'];
      const maxRadius = (1.5 + (9 - 1) * 0.5) * (isMobile ? 0.6 : 1); // Largest bubble
      
      sampleCounts.forEach((count, i) => {
        const x = legendX + bubbleSpacing * i;
        const y = legendY + 20; // Fixed vertical position for all bubbles
        const radius = (1.5 + (count - 1) * 0.5) * (isMobile ? 0.6 : 1);
        
        // Draw sample bubble
        ctx.beginPath();
        ctx.arc(x + 10, y, radius, 0, Math.PI * 2);
        ctx.fillStyle = '#bbb';
        ctx.fill();
        
        // Draw label underneath - aligned to bottom
        ctx.textAlign = 'center';
        ctx.fillStyle = '#666';
        ctx.fillText(sampleLabels[i], x + 10, y + maxRadius + 12);
      });
      
      // Reset text align
      ctx.textAlign = 'center';
    }
    
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
    padding-bottom: 40px; /* Add padding for year labels */
  }
  
  canvas {
    width: 90vh; /* Match height */
    height: 90vh;
    max-width: 100%;
    max-height: 100%;
    aspect-ratio: 1 / 1;
    display: block;
  }
</style>