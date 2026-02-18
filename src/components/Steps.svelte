<script>
    export let currentStep;
    import Scrolly from "../helpers/Scrolly.svelte";
  
    $: steps = [
    `International maritime law requires every ship to be registered with a specific country—a flag
     state—granting it nationality. Many tankers use so-called flags of convenience to evade regulation.`,
    `Shadow-fleet vessels are typically old and lack proper insurance. The mean age of a shadow fleet ship
     is almost double the average for other tankers, based on data from the Brookings Institute.`,
    `Larger tankers generally undertake longer voyages so they can carry more cargo. Smaller vessels can load and unload more quickly, making them more efficient for shorter journeys.
    <br><br><span style="display: block; color: #727272;font-weight: 300; font-family: Retina, sans-serif;font-size: 13px;line-height: 20px; margin-top:6px;">*Very Large Crude Carrier or Ultra Large Crude Carrier</span>`,
    `The West has imposed sanctions on thousands of entities tied to shadow-fleet tankers to punish Russia for its invasion of Ukraine. The joint efforts are aimed at crippling Russia's 
    oil and gas exports, which have powered Moscow's economic engine.
    <br><br><span style="display: block; color: #727272;font-weight: 300; font-family: Retina, sans-serif;font-size: 13px;line-height: 20px; margin-top:4px;">Note: Sanctions to a tanker can be issued by several authorities. First sanction by major authority shown.</span>`,
    `In the latest round of sanctions, nearly 900 vessels were designated last year, more than five times as many as the year before.<br><br><span style="display: block; color: #727272;font-weight: 300; font-family: Retina, sans-serif;font-size: 13px;line-height: 20px; margin-top:6px;">Note: Latest sanctions shown</span>`
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
  }
  
  </style>