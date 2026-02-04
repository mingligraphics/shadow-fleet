<script>
    export let currentStep;
    import Scrolly from "../helpers/Scrolly.svelte";
  
    $: steps = [
    `There are 1300 shadow fleet vessels`,
    `Most of them have flags from five countries, here are five groups, including four major countries and one other`,
    `We are grouping them based on their ages, altogether four groups`,
    `We are now grouping them based on their sizes, about five groups`,
    `Here are how they are grouped based on who sanctioned them, about five groups`,
    `Here's how it looks if we put them in a timeline of when they were sanctioned in the past 20 years`
  ];
  
  </script>
  <div class="steps">
    <Scrolly bind:value={currentStep}>
      {#each steps as text, i}
      <div class="step" class:active={currentStep === i}>
        <div class="step-content">
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
  
  .step.active {
  opacity: 1;
  }
  
  .steps {
  position: relative;
  z-index: 2;
  pointer-events: none;
  /* display: flex;
  justify-content: center;
  place-items: center; */
  }
  
  </style>