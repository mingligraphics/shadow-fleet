<script>
    export let currentStep;
    import Scrolly from "../helpers/Scrolly.svelte";
  
    $: steps = [
    `International maritime law requires every ship to be registered with a specific country—a flag state—granting it nationality. 
    Many tankers use “flags of convenience” to evade regulation.`,
    `Shadow fleet vessels are typically old, which means they are vulnerable to mechanical failures and leaks. 
    This is made worse by the fact that many lack proper insurance coverage. Should a spill or incident occur, 
    no insurance money will be available to cover the cost of a cleanup or rescue. 
    <br><br>The mean age of the shadow fleet ship was 22 years old, according to WSJ analysis of the TankerTrackers data, much higher than the average of 12.4 years for ships outside the shadow fleet, based on data from the Brookings Institute.`,
    `The distance of voyages determines the size of the tanker. On long voyages, such as from Venezuela to China, larger tankers that can carry more cargo would be more economical. 
    On the other hand, smaller vessels can load and unload more quickly, making them better fits for shorter voyages such as from Russia’s eastern port of Vladivostok to India through the Singapore Strait. 
    This route cuts down the travel time by 40% from the traditional routes from Russia’s west to India via the Suez Canal. 
    <br><br><span style="color: #727272;font-weight: 300; font-family: Retina, sans-serif;font-size: 13px;line-height: 17px; margin-top:6px;">*Very Large Crude Carrier or Ultra Large Crude Carrier</span>`,
    `Following the Russian invasion of Ukraine in February 2022, the U.S., U.K., European Union, and their allies have levied sanctions on thousands of entities tied to shadow fleet tankers. 
    The joint efforts are aimed at crippling Russia’s oil and gas exports, which has been Moscow’s most important economic engine.
    <br><br><span style="color: #727272;font-weight: 300; font-family: Retina, sans-serif;font-size: 13px;line-height: 17px; margin-top:6px;">Note: Only sanctions by major authorities shown</span>`,
    `In 2025, more than 700 first-time designations were added, representing a 77% surge from 2024, according to Kpler. However, 
    these sanctions have so far done little to stop the flow of sanctioned crude from Russia, Iran and Venezuela, and Western militaries have done little, 
    until recently, to physically stop the ships.<br><br><span style="color: #727272;font-weight: 300; font-family: Retina, sans-serif;font-size: 13px;line-height: 17px; margin-top:6px;">Note: Latest sanctions shown</span>`
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