<script>
    /**
     * Created by Russell Samora
     * https://svelte.dev/repl/5535ec729e594e97bf73590e688233a9?version=3.42.4
     * This component manages which item is most in view for scroll triggering
     * example:
     * <Scrolly
     * 	bind:value={scrollIndex}
     * >
     * **items here**
     * </Scrolly>
     *
     * optional params with defaults
     * <Scrolly root={null} top={0} bottom={0} increments={100}>
     */
    import { onMount } from "svelte";
    
    export let root = null;
    export let top = 0;
    export let bottom = 0;
    export let increments = 100;
    export let value = undefined;
  
    const steps = [];
    const threshold = [];
  
    let nodes = [];
    let intersectionObservers = [];
    let container;
  
    $: top, bottom, update();
  
    const update = () => {
      if (!nodes.length) return;
      nodes.forEach(createObserver);
    };
  
    const mostInView = () => {
      let maxRatio = 0;
      let maxIndex = 0;
      for (let i = 0; i < steps.length; i++) {
        if (steps[i] > maxRatio) {
          maxRatio = steps[i];
          maxIndex = i;
        }
      }
  
      console.log('Scrolly mostInView - maxIndex:', maxIndex, 'maxRatio:', maxRatio, 'steps:', steps);
  
      if (maxRatio > 0) value = maxIndex;
      else value = undefined;
    };
  
    const createObserver = (node, index) => {
      const handleIntersect = (e) => {
        const intersecting = e[0].isIntersecting;
        const ratio = e[0].intersectionRatio;
        steps[index] = ratio;
        console.log(`Step ${index} - intersecting: ${intersecting}, ratio: ${ratio}`);
        mostInView();
      };
  
      const marginTop = top ? top * -1 : 0;
      const marginBottom = bottom ? bottom * -1 : 0;
      const rootMargin = `${marginTop}px 0px ${marginBottom}px 0px`;
      const options = { root, rootMargin, threshold };
  
      if (intersectionObservers[index]) intersectionObservers[index].disconnect();
  
      const io = new IntersectionObserver(handleIntersect, options);
      io.observe(node);
      intersectionObservers[index] = io;
    };
  
    onMount(() => {
      console.log('Scrolly mounted');
      for (let i = 0; i < increments + 1; i++) {
        threshold.push(i / increments);
      }
      nodes = container.querySelectorAll(":scope > *");
      console.log('Scrolly found', nodes.length, 'step nodes');
      update();
    });
  </script>
  
  <div bind:this={container}>
    <slot />
  </div>