<script>
    export let currentStep;
    import Scrolly from "../helpers/Scrolly.svelte";
  
    // Add logging to see if the value updates
    $: console.log('Steps component - currentStep:', currentStep);
  
    $: steps = [
      `There are 1300 plus shadow fleet vessels`,
      `Most of them have flags from four countries`,
      `We are grouping them based on their ages`,
      `by their sizes`,
      `by countries who sanctioned them`,
      `Most of the sanctions happened after 2020. Here are when they were sanctioned last time`
    ];
  
  </script>
  
  <div class="steps">
    <Scrolly bind:value={currentStep}>
      {#each steps as text, i}
      <div class="step" class:active={currentStep === i}>
        <div class="step-content">
          <p class="step-number">Step {i}</p>
          {@html text}
        </div>
      </div>
      {/each}
    </Scrolly>
  </div>
  
  <style>
    .step {
      height: 90vh;
      opacity: 0.3;
      transition: opacity 300ms ease;
  
      /* vertically and horizontally center the steps */
      display: flex;
      justify-content: center;
      place-items: center;
    }
  
    .step-content {
      position: relative;
      box-sizing: border-box;
      z-index: 1;
      -webkit-transform: translate3d(0, 0, 0); /*for safari z-index issue*/
      height: auto;
      width: 100%;
      max-width: 500px; 
      margin: 0 auto;
      font-family: RetinaLight, sans-serif;
      font-size: 17px;
      line-height: 27px;
      font-weight: 400;
      color: #333;
      padding: 20px 25px 20px;
      box-shadow: 0 0 1px 0 rgba(0, 0, 0, .4);
      overflow: visible;
      background-color: rgba(255, 255, 255, .9);
    }
  
    .step-number {
      font-size: 12px;
      color: #999;
      margin: 0 0 10px 0;
      font-weight: 600;
    }
  
    .step.active {
      opacity: 1;
    }
  
    .steps {
      position: relative;
      z-index: 2;
      pointer-events: none;
    }
  </style>