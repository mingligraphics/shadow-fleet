<script>
    export let currentStep;
    import Scrolly from "../helpers/Scrolly.svelte";
  
    $: steps = [
    `As of January 21st, there were 1,308 tankers sanctioned by different authorities. 
    Merchant vessels must officially register a flag state under whose laws they agree to operate. 
    Shadow fleet tankers frequently adopt flags of convenience due to insurance requirements or safety standards. 
    Many also reflag to Russia seeking protection against U.S. interception and drone attacks.`,
    `Shadow fleet vessels’ advanced age and poor repair make them vulnerable to mechanical failures and catastrophic leaks. 
    Shadow fleet ships’ lack of insurance means that, should a spill occur, Western states could be forced to pay for the cleanup out of their own budgets, 
    creating a fiscal crisis.<br>The mean age of shadow fleet ships in our dataset was 19.0 years old, much higher than the average of 12.4 years for ships outside the shadow fleet.`,
    `Voyage distance determines which size tanker would be more economical to use. 
    On long voyages, a tanker spends a greater portion of total voyage time at sea as opposed to loading or unloading in port, 
    so larger tankers that can carry more cargo would be more economical. Since shorter voyages entail a greater portion of total voyage time in port, 
    smaller vessels that can load and unload more quickly are often more economical.`,
    `Following the Russian invasion of Ukraine in February 2022, the United States, United Kingdom, European Union, and their allies levied a barrage of sanctions on the Russian Federation. 
    Russia’s energy industry has been the main target of these sanctions, and for good reason. Exports of fossil fuels, particularly crude oil, drive Russia’s economy.`,
    `Between January and December 2025, vessel sanctions increased 77% year-over-year—over 700 first-time designations in twelve months. Despite designating hundreds of shadow fleet tankers, 
    the fleet didn't shrink—it adapted. They modified behaviour, employing increasingly sophisticated deceptive practices to continue operations.`
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