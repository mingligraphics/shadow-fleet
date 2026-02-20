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
  let displayStep = 0; // ← never goes undefined; freezes on last seen step

  // Only update displayStep when currentStep is a real value
  $: if (currentStep !== undefined) displayStep = currentStep;

  let dots = [];
  let simulation;
  let animationFrame;
  let mounted = false;
  let vesselIcons = {};
  let flagIcons = {};
  
  let FLAGS = [];
  let SIZE_GROUPS = [];
  const SANCTIONING_BODIES = ['U.S.', 'U.K.', 'EU', 'Other'];
  const AGE_GROUPS = ['0-10 years', '11-20', '21-30', '31 and more'];
  
  const PEACOCK_COLORS = ['#e3f0f6', '#c3e3ed', '#88c9d7', '#00b3c9', '#0091a2', '#006f7a'];
  const GINGER_COLORS = ['#f7deca', '#f2c7a1', '#e8ac76', '#dc6c00', '#b25200', '#883800'];
  const TIMELINE_COLORS = ['#c3e3ed', '#c3e3ed', '#88c9d7', '#00b3c9', '#0091a2', '#006f7a','#006f7a'];
  
  let FLAG_COLORS = {};
  let SIZE_COLORS = {};
  const AGE_COLORS = ['#88c9d7', '#00b3c9', '#0091a2', '#727272'];
  const SANCTION_COLORS = {
    'U.S.': '#00b3c9',
    'EU': '#c3e3ed',
    'U.K.': '#88c9d7',
    'Other': '#bfbfbf'
  };

  const FLAG_ICON_SOURCES = {
    'Russia':   'https://images.wsj.net/im-59819117',
    'Cameroon': 'https://images.wsj.net/im-53935436',
    'Panama':   'https://images.wsj.net/im-70904983',
    'Sierra Leone': 'https://images.wsj.net/im-16776081',
  };

  const flagIconOffsets = {
    'Russia':   { desktop: { dx:  -22, dy: -120 }, mobile: { dx: -10, dy: -75 } },
    'Cameroon': { desktop: { dx:  -24, dy: -120 }, mobile: { dx: -12, dy: -75 } },
    'Panama':   { desktop: { dx:  -22, dy: -120 }, mobile: { dx: -12, dy: -75 } },
    'Sierra Leone':     { desktop: { dx:  -22, dy: -120 }, mobile: { dx: -12, dy: -75 } },
  };

  const flagIconSize = {
    desktop: { w: 50, h: 50*2/3 },
    mobile:  { w: 24, h: 24*2/3 },
  };
  
  function getAgeGroupIndex(age) {
    if (age <= 10) return 0;
    if (age <= 20) return 1;
    if (age <= 30) return 2;
    return 3;
  }
  
  function drawTextWithBackground(text, x, y, fontSize = 15, textAlign = 'center') {
    ctx.font = `${fontSize}px Retina, sans-serif`;
    ctx.textAlign = textAlign;
    ctx.textBaseline = 'middle';
    
    ctx.strokeStyle = 'rgba(255, 255, 255, 0.9)';
    ctx.lineWidth = 4;
    ctx.strokeText(text, x, y);
    
    ctx.fillStyle = '#333';
    ctx.fillText(text, x, y);
  }
  
  function removeBackground(img, tolerance = 30) {
    const offscreen = document.createElement('canvas');
    offscreen.width  = img.naturalWidth  || img.width;
    offscreen.height = img.naturalHeight || img.height;
    const offCtx = offscreen.getContext('2d');
    offCtx.drawImage(img, 0, 0);

    const imageData = offCtx.getImageData(0, 0, offscreen.width, offscreen.height);
    const d = imageData.data;

    const bgR = d[0], bgG = d[1], bgB = d[2];

    for (let i = 0; i < d.length; i += 4) {
      const r = d[i], g = d[i + 1], b = d[i + 2];
      const diff = Math.abs(r - bgR) + Math.abs(g - bgG) + Math.abs(b - bgB);
      if (diff < tolerance * 3) {
        const alpha = Math.min(255, (diff / (tolerance * 3)) * 255 * 4);
        d[i + 3] = Math.round(alpha);
      }
    }

    offCtx.putImageData(imageData, 0, 0);
    return offscreen;
  }

  function loadVesselIcons() {
    const iconSources = {
      'Handysize/Handymax': 'https://images.wsj.net/im-87719691',
      'Aframax':            'https://images.wsj.net/im-82816473',
      'Suezmax':            'https://images.wsj.net/im-75328124',
      'VLCC/ULCC*':         'https://images.wsj.net/im-97627471'
    };

    Object.entries(iconSources).forEach(([size, src]) => {
      const img = new Image();
      img.crossOrigin = 'anonymous';
      img.onload  = () => {
        try {
          vesselIcons[size] = removeBackground(img);
        } catch (e) {
          vesselIcons[size] = img;
        }
      };
      img.onerror = () => console.error(`Failed to load vessel icon for ${size}`);
      img.src = src;
    });
  }

  function loadFlagIcons() {
    Object.entries(FLAG_ICON_SOURCES).forEach(([country, src]) => {
      const img = new Image();
      img.crossOrigin = 'anonymous';
      img.onload = () => {
        flagIcons[country] = img;
      };
      img.onerror = () => console.error(`Failed to load flag icon for ${country}`);
      img.src = src;
    });
  }
  
  function initializeDots() {
    console.log(`Loading ${data.length} vessels from JSON`);
    
    const uniqueSizes = [...new Set(data.map(d => d.size_category))].filter(s => s);
    
    const sizeOrderSmallToBig = ['Handysize/Handymax', 'Aframax', 'Suezmax', 'VLCC/ULCC*'];
    SIZE_GROUPS = sizeOrderSmallToBig.filter(size => uniqueSizes.includes(size));
    
    uniqueSizes.forEach(size => {
      if (!SIZE_GROUPS.includes(size) && size !== 'Other') SIZE_GROUPS.push(size);
    });
    if (uniqueSizes.includes('Other')) SIZE_GROUPS.push('Other');

    const flagCounts = {};
    data.forEach(d => { flagCounts[d.flag_country] = (flagCounts[d.flag_country] || 0) + 1; });
    
    const sortedFlags = Object.entries(flagCounts)
      .filter(([flag]) => flag !== 'Other')
      .sort((a, b) => b[1] - a[1])
      .map(([flag]) => flag);
    
    if (flagCounts['Other']) sortedFlags.push('Other');
    FLAGS = sortedFlags;
    
    const flagSpecificColors = {
      'Russia':   '#0091a2',
      'Cameroon': '#00b3c9',
      'Panama':   '#88c9d7',
      'Sierra Leone':     '#c3e3ed',
      'Other':    '#bfbfbf'
    };
    
    const sizeToGinger = {
      'VLCC/ULCC*':         '#0091a2',
      'Suezmax':            '#00b3c9',
      'Aframax':            '#88c9d7',
      'Handysize/Handymax': '#c3e3ed',
      'Other':              '#bfbfbf'
    };
    
    FLAGS.forEach(flag => {
      FLAG_COLORS[flag] = flagSpecificColors[flag] || '#00b3c9';
    });
    SIZE_GROUPS.forEach(size => {
      SIZE_COLORS[size] = sizeToGinger[size] || (size === 'Other' ? '#727272' : '#dc6c00');
    });
    
    dots = data.map((vessel, i) => {
      const flagIndex     = FLAGS.indexOf(vessel.flag_country);
      const ageIndex      = getAgeGroupIndex(vessel.age_years);
      const sizeIndex     = SIZE_GROUPS.indexOf(vessel.size_category);
      const sanctionIndex = SANCTIONING_BODIES.indexOf(vessel.sanctioned_by);
      
      return {
        id: i,
        x: Math.random() * width,
        y: Math.random() * height,
        vx: 0,
        vy: 0,
        vesselName:        vessel.vessel_name,
        flag:              vessel.flag_country,
        flagIndex:         flagIndex >= 0 ? flagIndex : 0,
        age:               vessel.age_years,
        ageIndex,
        size:              vessel.size_category,
        sizeIndex:         sizeIndex >= 0 ? sizeIndex : 0,
        sanctionedBy:      vessel.sanctioned_by,
        sanctionIndex:     sanctionIndex >= 0 ? sanctionIndex : 0,
        sanctionYear:      vessel.sanction_year,
        latestSanctionYear:vessel.latest_sanction_year,
        sanctionCount:     vessel.sanction_count,
        UANIOnly:          vessel.uani_only
      };
    });
  }
  
  function setupCanvas() {
    if (!canvas) return;
    const dpr  = window.devicePixelRatio || 1;
    const rect = canvas.getBoundingClientRect();
    const size = Math.min(rect.width, rect.height);
    width  = size;
    height = size;
    canvas.width  = width  * dpr;
    canvas.height = height * dpr;
    ctx = canvas.getContext('2d');
    ctx.scale(dpr, dpr);
    canvas.style.width  = `${width}px`;
    canvas.style.height = `${height}px`;
  }
  
  function getDotColor(dot, step) {
    switch(step) {
      case 0: return FLAG_COLORS[dot.flag] || '#00b3c9';
      case 1: return AGE_COLORS[dot.ageIndex];
      case 2: return SIZE_COLORS[dot.size] || '#727272';
      case 3: return SANCTION_COLORS[dot.sanctionedBy] || '#727272';
      case 4:
        const yearsSince2020 = Math.max(0, Math.min(dot.latestSanctionYear - 2020, 6));
        return TIMELINE_COLORS[yearsSince2020];
      default: return '#00b3c9';
    }
  }
  
  function getDotRadius(dot, step) {
    const isMobile = width < 450;
    const mobileScale = isMobile ? 0.6 : 1;
    if (step === 3 || step === 4) return (1.5 + (dot.sanctionCount - 1) * 0.5) * mobileScale;
    return 2.5 * mobileScale;
  }
  
  function updateSimulation(step) {
    if (!mounted || dots.length === 0) return;
    if (simulation) simulation.stop();
    
    simulation = forceSimulation(dots);
    const isMobile      = width < 450;
    const collisionScale = isMobile ? 0.6 : 1;
    const chargeStrength = isMobile ? -0.5 : -1.5;
    
    switch(step) {
      case 0:
        simulation
          .force('x', forceX(d => {
            const spacing = width / (FLAGS.length + 1.5);
            const baseX   = spacing * (d.flagIndex + 1);
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
        
      case 2: {
        const mobileOffset = isMobile ? 30 : 0;
        simulation
          .force('x', forceX(d => {
            const angle = (d.sizeIndex / SIZE_GROUPS.length) * Math.PI * 2 - Math.PI / 2;
            return width / 2 + Math.cos(angle) * (Math.min(width, height) * 0.25);
          }).strength(0.1))
          .force('y', forceY(d => {
            const angle = (d.sizeIndex / SIZE_GROUPS.length) * Math.PI * 2 - Math.PI / 2;
            return height / 2 + mobileOffset + Math.sin(angle) * (Math.min(width, height) * 0.25);
          }).strength(0.1))
          .force('collide', forceCollide(3 * collisionScale))
          .force('charge', forceManyBody().strength(chargeStrength));
        break;
      }
        
      case 3:
        simulation
          .force('x', forceX(d => {
            if (d.UANIOnly !== 'N') return width / 2;
            const spacing = width / (SANCTIONING_BODIES.length + 1.5);
            const baseX   = spacing * (d.sanctionIndex + 1) + width * 0.12;
            return Math.max(60, Math.min(width - 60, baseX));
          }).strength(d => d.UANIOnly === 'N' ? 0.1 : 0))
          .force('y', forceY(height / 2).strength(d => d.UANIOnly === 'N' ? 0.1 : 0))
          .force('collide', forceCollide(d => d.UANIOnly === 'N' ? getDotRadius(d, 3) + 1 : 0))
          .force('charge', forceManyBody().strength(chargeStrength));
        break;
        
      case 4:
        const xMargin = isMobile ? 0.15 : 0.05;
        const xRange  = isMobile ? 0.7  : 0.9;
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
  
  function render() {
    if (!ctx || !mounted) return;
    
    ctx.clearRect(0, 0, width, height);
    
    // Use displayStep (never undefined) for all rendering
    dots.forEach(dot => {
      if (displayStep === 3 && dot.UANIOnly !== 'N') return;
      const radius = getDotRadius(dot, displayStep);
      ctx.beginPath();
      ctx.arc(dot.x, dot.y, radius, 0, Math.PI * 2);
      ctx.fillStyle = getDotColor(dot, displayStep);
      ctx.fill();
    });
    
    const isMobile = width < 450;
    const fontSize  = isMobile ? 11 : 15;
    ctx.font        = `${fontSize}px Retina, sans-serif`;
    ctx.textAlign   = 'center';
    ctx.fillStyle   = '#333';
    
    switch(displayStep) {
      case 0: {
        const flagCounts = {};
        FLAGS.forEach(flag => {
          flagCounts[flag] = dots.filter(d => d.flag === flag).length;
        });

        const flagLabelOffsets = {
          'Russia':   { dx: isMobile ? -25 : -50, dy: 0 },
          'Cameroon': { dx: isMobile ? -20 : -30, dy: 0 },
          'Panama':   { dx: isMobile ? -20 : -30, dy: 0 },
          'Sierra Leone': { dx: isMobile ? -20 : -35, dy: 0 },
          'Other':    { dx: isMobile ?  15 :  20, dy: 0 },
        };

        FLAGS.forEach((flag, i) => {
          const spacing = width / (FLAGS.length + 1.5);
          const baseX   = spacing * (i + 1);
          const off     = flagLabelOffsets[flag] || { dx: 0, dy: 0 };
          const cx      = baseX + off.dx;
          const cy      = height / 2 + off.dy;

          drawTextWithBackground(flag, cx, cy, fontSize);

          if (flag !== 'Other') {
            drawTextWithBackground(
              flagCounts[flag].toString(),
              cx,
              cy + fontSize + 8,
              fontSize
            );
          }

          const icon = flagIcons[flag];
          if (icon) {
            const device   = isMobile ? 'mobile' : 'desktop';
            const iconOff  = (flagIconOffsets[flag] || {})[device] || { dx: 0, dy: -80 };
            const iconSize = flagIconSize[device];
            ctx.drawImage(icon, cx + iconOff.dx, cy + iconOff.dy, iconSize.w, iconSize.h);
          }
        });
        break;
      }
        
      case 1:
        AGE_GROUPS.forEach((ageGroup, i) => {
          const spacing = height / 4.5;
          const x = 15;
          let y = spacing * (i + 1);
          if (i < 2) y -= 65;
          if (i == 2) y -= 25;
          ctx.textAlign = 'left';
          ctx.fillText(ageGroup, x, y);
        });
        ctx.textAlign = 'center';
        break;
        
      case 2: {
        const mobileOffset = isMobile ? 30 : 0;

        const tonnageInfo = {
          'Handysize/Handymax': 'Carrying capacity: less than 400,000 barrels',
          'Aframax':            '750,000',
          'Suezmax':            'One million',
          'VLCC/ULCC*':         'Two to four million'
        };
        const footageInfo = {
          'Handysize/Handymax': 'Length: less than 625 ft.',
          'Aframax':            '800 ft.',
          'Suezmax':            '935 ft.',
          'VLCC/ULCC*':         '1,080–1,360 ft.'
        };
        const footageOffsets = {
          'Handysize/Handymax': { dx: isMobile ?  0 :  -8, dy: isMobile ? 0 : 16 },
          'Aframax':            { dx: isMobile ?  0 : -10, dy: isMobile ? 0 : 15 },
          'Suezmax':            { dx: isMobile ?  0 : -10, dy: isMobile ? 0 : 15 },
          'VLCC/ULCC*':         { dx: isMobile ?  0 :   0, dy: isMobile ? 0 : 15 },
        };
        const iconOffsets = {
          'Handysize/Handymax': { dx: isMobile ?   0 :   0, dy: isMobile ?  -80 : -140 },
          'Aframax':            { dx: isMobile ?  36 :  60, dy: isMobile ?  -55 :  -95 },
          'Suezmax':            { dx: isMobile ?  12 :  25, dy: isMobile ?   -5 :  -20 },
          'VLCC/ULCC*':         { dx: isMobile ? -15 : -25, dy: isMobile ?  -20 :  -35 },
        };
        const labelOffsets = {
          'Handysize/Handymax': { dx: isMobile ?   0 : -10, dy: isMobile ? -30 : -40 },
          'Aframax':            { dx: isMobile ?  25 :  40, dy: isMobile ?   0 :   0 },
          'Suezmax':            { dx: isMobile ?  10 :  15, dy: isMobile ?  25 :  35 },
          'VLCC/ULCC*':         { dx: isMobile ? -12 : -23, dy: isMobile ?  20 :  25 },
        };
        
        SIZE_GROUPS.forEach((size, i) => {
          const angle       = (i / SIZE_GROUPS.length) * Math.PI * 2 - Math.PI / 2;
          const labelRadius = Math.min(width, height) * 0.25;
          const cx          = width  / 2 + Math.cos(angle) * labelRadius;
          const cy          = height / 2 + mobileOffset + Math.sin(angle) * labelRadius;

          const iconOffset      = iconOffsets[size]  || { dx: 0, dy: -(isMobile ? 14 : 28) };
          const labelOffset     = labelOffsets[size] || { dx: 0, dy: 0 };
          const iconSize        = isMobile ? 20 : 50;
          const smallerFontSize = isMobile ?  9 : 11;

          const iconCx = cx + iconOffset.dx;
          const iconCy = cy + iconOffset.dy;

          if (footageInfo[size]) {
            const fOff = footageOffsets[size] || { dx: 0, dy: 0 };
            drawTextWithBackground(
              footageInfo[size],
              iconCx + fOff.dx,
              iconCy - iconSize / 2 - (isMobile ? 8 : 12) + fOff.dy,
              smallerFontSize
            );
          }

          const icon = vesselIcons[size];
          if (icon) {
            ctx.save();
            ctx.globalCompositeOperation = 'multiply';
            ctx.drawImage(icon, iconCx - iconSize / 2, iconCy - iconSize / 2, iconSize, iconSize);
            ctx.restore();
          }

          const lx = cx + labelOffset.dx;
          const ly = cy + labelOffset.dy;
          drawTextWithBackground(size, lx, ly, fontSize);
          if (tonnageInfo[size]) {
            drawTextWithBackground(tonnageInfo[size], lx, ly + fontSize + 6, smallerFontSize);
          }
        });
        break;
      }
        
      case 3: {
        const sanctionCounts = {};
        SANCTIONING_BODIES.forEach(body => {
          sanctionCounts[body] = dots.filter(d => d.UANIOnly === 'N' && d.sanctionedBy === body).length;
        });
        
        SANCTIONING_BODIES.forEach((body, i) => {
          const spacing = width / (SANCTIONING_BODIES.length + 1.5);
          let x = spacing * (i + 1) + width * 0.12;
          
          if (isMobile) {
            if (i === 0) x -= 10;
            else if (i === 1) x += 20;
            else if (i === 2) x += 35;
            else if (i === 3) x += 30;
          } else {
            if (i === 0) x -= 25;
            else if (i === 1) x += 40;
            else if (i === 2) x += 52;
            else if (i === 3) x += 25;
          }
          
          const y = height / 2;
          drawTextWithBackground(body, x, y, fontSize);
          drawTextWithBackground(sanctionCounts[body].toString(), x, y + fontSize + 8, fontSize);
        });
        break;
      }
        
      case 4: {
        const startYear = 2020;
        const endYear   = 2026;
        const xMargin   = isMobile ? 0.15 : 0.05;
        const xRange    = isMobile ? 0.7  : 0.9;

        const yearLabelY = isMobile ? height * 0.94 : height * 0.85;

        ctx.font      = '13px "Retina Narrow", sans-serif';
        ctx.fillStyle = '#222222';
        ctx.textBaseline = 'middle';

        for (let year = startYear; year <= endYear; year++) {
          const yearProgress = (year - startYear) / 6;
          const x = width * xMargin + (width * xRange * yearProgress);
          const yearLabel = year === 2020 ? "2016-'20" : `'${year.toString().slice(-2)}`;
          ctx.fillText(yearLabel, x, yearLabelY);
        }

        if (!isMobile) {
          const vessels2025Count = dots.filter(d => d.latestSanctionYear === 2025).length;
          ctx.font      = '13px Retina, sans-serif';
          ctx.fillStyle = '#333';
          ctx.textAlign = 'center';
          ctx.fillText(`${vessels2025Count} vessels`, width * 3.15 / 4, 92);

          const year2022Progress  = (2022 - startYear) / 6;
          const annotationX = width * xMargin + (width * xRange * year2022Progress) + 40;
          const annotationY = yearLabelY + 28;

          ctx.font      = '13px Retina, sans-serif';
          ctx.fillStyle = '#333';
          ctx.textAlign = 'center';
          ctx.fillText('Russia-Ukraine war', annotationX + 35, annotationY + 17);

          const arrowY      = annotationY;
          const arrowStartX = annotationX - 20;
          const arrowEndX   = width * 0.95;
          const headSize    = 4;

          ctx.beginPath();
          ctx.moveTo(arrowStartX, arrowY);
          ctx.lineTo(arrowEndX, arrowY);
          ctx.moveTo(arrowEndX - headSize, arrowY - headSize * 0.6);
          ctx.lineTo(arrowEndX, arrowY);
          ctx.lineTo(arrowEndX - headSize, arrowY + headSize * 0.6);
          ctx.strokeStyle = '#333';
          ctx.lineWidth   = 1.5;
          ctx.stroke();
        }
        
        ctx.font      = `${fontSize}px Retina, sans-serif`;
        ctx.fillStyle = '#333';
        break;
      }
    }
    
    // Draw legend for steps 3 and 4
    if (displayStep === 3 || displayStep === 4) {
      const legendX       = isMobile ? 15 : 20;
      const legendY       = isMobile ? 30 : 90;
      const bubbleSpacing = isMobile ? 20 : 25;
      
      ctx.font      = `${isMobile ? 10 : 12}px Retina, sans-serif`;
      ctx.textAlign = 'left';
      ctx.fillStyle = '#666';
      ctx.fillText('Number of sanctions per vessel', legendX, legendY);
      
      const sampleCounts = [1, 3, 5, 9];
      const sampleLabels = ['1', '3', '5', '9'];
      const maxRadius    = (1.5 + (9 - 1) * 0.5) * (isMobile ? 0.6 : 1);
      
      sampleCounts.forEach((count, i) => {
        const x      = legendX + bubbleSpacing * i;
        const y      = legendY + 20;
        const radius = (1.5 + (count - 1) * 0.5) * (isMobile ? 0.6 : 1);
        ctx.beginPath();
        ctx.arc(x + 10, y, radius, 0, Math.PI * 2);
        ctx.fillStyle = '#bbb';
        ctx.fill();
        ctx.textAlign = 'center';
        ctx.fillStyle = '#666';
        ctx.fillText(sampleLabels[i], x + 10, y + maxRadius + 12);
      });
      
      ctx.textAlign = 'center';
    }
    
    animationFrame = requestAnimationFrame(render);
  }
  
  // Only trigger simulation when currentStep has a real value (not undefined)
  $: if (mounted && dots.length > 0 && currentStep !== undefined) {
    updateSimulation(currentStep);
  }
  
  onMount(() => {
    mounted = true;
    loadVesselIcons();
    loadFlagIcons();
    setupCanvas();
    initializeDots();
    updateSimulation(0);
    render();
    
    const handleResize = () => {
      setupCanvas();
      if (dots.length > 0) updateSimulation(displayStep);
    };
    
    window.addEventListener('resize', handleResize);
    
    return () => {
      mounted = false;
      if (animationFrame) cancelAnimationFrame(animationFrame);
      if (simulation)     simulation.stop();
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
    <div class="after-steps"></div>
  </section>
</main>

<style>
  :global(body) {
    margin: 0;
    padding: 0;
    background: #FFFFFF;
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
    padding-bottom: 40px;
  }
  
  canvas {
    width: 90vh;
    height: 90vh;
    max-width: 100%;
    max-height: 100%;
    aspect-ratio: 1 / 1;
    display: block;
  }

  /* Keeps the sticky canvas pinned while the last step scrolls away */
  .after-steps {
    height: 100vh;
  }
</style>