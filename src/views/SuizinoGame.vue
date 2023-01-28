<script setup>
import {ref, onMounted,onUnmounted, reactive} from 'vue'
import {logo} from "../assets/icons";
import {useAuthStore} from "../stores/auth";
import {casinoAddress, moduleAddress} from "../helpers/constants";
import {useWallet} from "../helpers/wallet";
import {useUiStore} from "../stores/ui";

import { bcs } from "@mysten/bcs";

const authStore = useAuthStore();
const uiStore = useUiStore();
const {executeMoveCall, getAddress, getSuitableCoinId, getSigner} = useWallet();

const gameStatuses = {
  STANDBY: 'STANDBY',
  LOSS: 'LOSS',
  WIN: 'WIN'
}
const spinningList = ["❌", "🦄", "✅", "🐻️", "🤑"];
const gameStatus = ref(gameStatuses.STANDBY);
const gameStarted = ref(false);
const isLoading = ref(false);
const gameResultsObject = ref({});
const gameResults = ref([]);
const totalGames = ref(0);
const wheelSlots = reactive([
  {
    id: 0,
    started: false,
    randomSlides: [],
    ended: false
  },
  {
    id: 1,
    randomSlides: [],
    started: false,
    ended: false
  },
  {
    id: 2,
    randomSlides: [],
    started: false,
    ended: false
  }
]);

const amount = reactive({
  value: 10000000
})

const player_choice = reactive({
  value: 1
})

const coin_state = reactive({
  value: 2
})

const signer = getSigner();

let initialSpinInterval = ref();

onMounted(()=>{
  initialSpinInterval.value = setupSpinningInterval(120);
 });



const executeGamble = () => {
  const address = getAddress();
  if(!address) return;
  if(gameStarted.value) return;
  resetGame();
  gameStarted.value = true;
  isLoading.value = true;

  const coinId = getSuitableCoinId(amount.value);
  
  console.log(authStore.signer);

  const coins = authStore.coins.map(x => {
      return x.id;
  });

  console.log(coins);

  executeMoveCall({
    packageObjectId: moduleAddress,
    module: 'Base',
    typeArguments: [],
    arguments: [casinoAddress, coins, amount.value.toString(), player_choice.value],
    function: 'gamble',
    gasBudget: 1000
  }).then(res =>{
    totalGames.value++;
    const status = res?.effects?.status?.status;

    if(status === 'success'){
      let SuizinoEventResult = res?.effects?.events?.find(x => x.moveEvent) || {};

      let fields = SuizinoEventResult?.moveEvent?.fields;

      gameResultsObject.value = fields;
      gameResults.value = [fields.slot_1, fields.slot_2, fields.slot_3];

      // We just mark all slots as "started" and we let the interval that is already running
      // take care of showing the results. To make it more smooth,
      for(let [index, slot] of wheelSlots.entries()){
        slot.started = true;
        // setTimeout(()=>{
        //
        // }, (index+1) * 600); // start wih 300ms difference
      }

    }else{
      uiStore.setNotification(res?.effects?.status?.error);
      resetGame();
    }
  }).catch(e=>{
    resetGame();

    uiStore.setNotification(e.message);
  })
}


const setupSpinningInterval = (timeout) => {
  return setInterval(()=> {
    for (let slot of wheelSlots) {

      if(slot.ended) continue; // if slot selection ended. continue!

      // if slot slection has started, we pick one and then break
      if(slot.started){
        clearSpinningInterval();
        slot.randomSlides = [spinningList[gameResults.value[slot.id]]];
        slot.ended = true;

        if(slot.id === wheelSlots.length - 1) checkGameStatus();
        continue;
      }
      slot.randomSlides = [...spinningList].sort(() => 0.5 - Math.random()).slice(0,3);
    }
  }, timeout);
}

const clearSpinningInterval = () => {
  clearInterval(initialSpinInterval.value);
}

onUnmounted(()=>{
  clearSpinningInterval();
});

const setBettingValue = (para) => {  
  amount.value = para;
}

const setPlayerchoice = (para) => {
  player_choice.value = para;
}

const resetGame = () => {
  
  clearSpinningInterval();
  for(let slot of wheelSlots){
    slot.randomSlides = [];
    slot.started = false;
    slot.ended = false;
  }
  isLoading.value = false;
  gameStarted.value = false;
  gameResults.value = null;
  gameStatus.value = gameStatuses.STANDBY;
  gameResultsObject.value = {};
  coin_state.value = 2;
  setupSpinningInterval(120);
}
const startGame = () => {
}

const checkGameStatus = () =>{
  gameStarted.value = false;
  isLoading.value = false;
  let hasWon = true;
  let icon = null;
  gameStatus.value = gameResultsObject.value.winnings > 0 ? gameStatuses.WIN : gameStatuses.LOSS;
  

  if(gameStatus.value === gameStatuses.WIN){
    uiStore.setNotification("Congratulations 🎉! You won " + gameResultsObject.value.winnings + " MIST!", "success")
    coin_state.value = gameResultsObject.value.slot_1;
  }
  if(gameStatus.value === gameStatuses.LOSS){
    uiStore.setNotification("😔 You were unlucky this time. maybe try an extra spin?")
    coin_state.value = player_choice.value === 1 ? 0 : 1;
  }
  for(let slot of wheelSlots){
    if(!icon) {
      icon = slot.randomSlides[0];
      continue;
    }

    if(icon !== slot.randomSlides[0]){
      hasWon = false;
      break;
    }
  }
}

</script>

<style>

.lucky-wheel-slot.win{
  @apply border-2 border-green-600;
}
.lucky-wheel-slot.loss{
  @apply border-2 border-red-700;
}

.depth-front {
    -webkit-box-align: center;
    align-items: center;
    /* background: linear-gradient(to left, rgb(0, 0, 0), rgb(0, 0, 0)); */
    border-radius: 100%;
    display: flex;
    height: 10em;
    -webkit-box-pack: center;
    justify-content: center;
    transform-style: preserve-3d;
    width: 10em;
    margin:0px auto;
}

.gQRSty {
    /* background: rgb(42, 116, 202); */
    /* border: 2px solid rgb(250, 250, 250); */
    padding: 12px;
    border-top-left-radius: 35px;
    border-top-right-radius: 35px;
    display: -webkit-flex;
    gap: 0.75rem;
    overflow: hidden;
    width: 100%;
    z-index: 0;
    max-width: 700px;
    margin-top: 1.5rem;
    margin-bottom: 1.5rem
}

.cUYTSd {
    -webkit-box-align: center;
    align-items: center;
    background: rgb(0, 83, 138);
    border-radius: 4px;
    display: flex;
    gap: 2px;
    height: 40px;
    -webkit-box-pack: center;
    justify-content: center;
    position: relative;
    padding: 2px;
    line-height: 0;
    width: 100%;
    z-index: 0;
}

.jyukYf {
    -webkit-box-align: center;
    align-items: center;
    background: linear-gradient(90deg, rgb(255, 255, 255) 0%, rgb(221, 79, 250) 100%);
    border: 1px solid rgb(248, 202, 34);    
    border-radius: 2px;
    color: rgb(23, 23, 23);
    display: flex;
    flex: 1 1 0%;
    gap: 4px;
    height: 100%;
    -webkit-box-pack: center;
    justify-content: center;
    line-height: 1.5rem;
    padding: 0px 16px;
    text-align: center;
    text-transform: uppercase;
    transition: all 300ms ease-in-out 0s;
    user-select: none;
    white-space: nowrap;
}

.jQzBaH {
    -webkit-box-align: center;
    align-items: center;
    background: none;
    border: 1px solid rgb(36, 36, 36);
    border-left-width: 0;
    border-top-width: 0;
    border-bottom-width: 0;
    border-radius: 2px;
    color: white;
    display: flex;
    flex: 1 1 0%;
    gap: 4px;
    height: 100%;
    -webkit-box-pack: center;
    justify-content: center;
    line-height: 1.5rem;
    padding: 0px 16px;
    text-align: center;
    text-transform: uppercase;
    transition: all 300ms ease-in-out 0s;
    user-select: none;
    white-space: nowrap;
}

.gQRSty .coin-side {
    width: fit-content;
}

.gQRSty > div:not(:first-child) {
    width: calc(50% - 0.375rem);
}


.bZJbip {
  -webkit-box-align: center;
  align-items: center;
  background: rgb(38, 48, 59);
  border-radius: 0px 0px 35px 35px;
  border-right: 2px solid rgb(0, 0, 0);
  border-bottom: 2px solid rgb(0, 0, 0);
  border-left: 2px solid rgb(0, 0, 0);
  border-image: initial;
  border-top: none;
  display: flex;
  flex-direction: column;
  -webkit-box-pack: justify;
  justify-content: space-between;
  max-width: 700px;
  padding: 0px 0px 28px;
  position: relative;
  width: 100%;
}

.jJnyom {
  display: flex;
  flex-direction: column;
  gap: 12px;
  list-style: none;
  overflow-y: scroll;
  padding: 20px 20px 24px;
  width: 100%;
  height: calc(100vh - 680px);
  min-height: 200px;
}

.jJnyom li {
  border-bottom: 1px solid rgb(23, 23, 23);
  display: flex;
  gap: 100px;
  padding: 0px 0px 12px;
  width: 100%;
  font-size: 0.875rem;
  color: rgb(255, 255, 255);
  text-align: center;
}

.jJnyom li span:nth-last-child(1) {
  flex: 1 1 0%;
  text-align: right;
}
</style>

<template>
  <main class="container">

<!--    <h2>Please login to continue</h2>-->
    <div class="max-w-[700px] mx-auto mb-20 mt-6">

      <div class="py-6 text-center">
        <h2 class="text-4xl font-bold">
          DEVIL'S FLIP
        </h2>
        <p>
          A Game By The Brawlers
        </p>

      </div>
      <div class="py-6 text-center" style="display: flex; z-index: 0;">
        <div>
          <img class="side-img" :src="'./head.png'" style="visibility: visible; padding-right: 1rem;"/>
          <p>Head</p>
        </div>
        <div>
          <img class="side-img" :src="'./tail.png'" style="visibility: visible; padding-left: 1rem;"/>
          <p>Tail</p>
        </div>
        
      </div>

      <h3 class="text-2xl font-bold">
          Select Amount
      </h3>
        
      <div class="gQRSty">
        
        <div class="cUYTSd">
          <button :class="amount.value === 10000000 ? 'jyukYf' : 'jQzBaH'" @click="setBettingValue(10000000)">0.01<div v-html="logo" class="logo-icon"></div></button>
          <button :class="amount.value === 50000000 ? 'jyukYf' : 'jQzBaH'" @click="setBettingValue(50000000)">0.05<div v-html="logo" class="logo-icon"></div></button>
          <button :class="amount.value === 100000000 ? 'jyukYf' : 'jQzBaH'" @click="setBettingValue(100000000)">0.1<div v-html="logo" class="logo-icon"></div></button>
          <button :class="amount.value === 500000000 ? 'jyukYf' : 'jQzBaH'" @click="setBettingValue(500000000)">0.5<div v-html="logo" class="logo-icon"></div></button>
          <button :class="amount.value === 1000000000 ? 'jyukYf' : 'jQzBaH'" @click="setBettingValue(1000000000)">1<div v-html="logo" class="logo-icon"></div></button>
        </div>
        
      </div>

      <h3 class="text-2xl font-bold">
          Select Side
      </h3>
      <div class="gQRSty">
        <div class="cUYTSd">
          <button :class="player_choice.value === 1 ? 'jyukYf' : 'jQzBaH'" @click="setPlayerchoice(1)">HEAD</button>
          <button :class="player_choice.value === 0 ? 'jyukYf' : 'jQzBaH'" @click="setPlayerchoice(0)">TAIL</button>
        </div>
      </div>
      

      <div class="mt-6 text-center">

        <button v-if="!authStore.hasWalletPermission" class="bg-gray-800 mx-auto ease-in-out duration-500 hover:px-10 dark:bg-gray-800 flex items-center text-white px-5 py-2 rounded-full" @click="authStore.toggleWalletAuthModal = true">
          <div v-html="logo" class="logo-icon"></div> Connect your wallet to start playing!
        </button>
        <button v-else
                class="bg-gray-800 mx-auto ease-in-out duration-500 hover:px-10 dark:bg-gray-800 flex items-center text-white px-5 py-2 rounded-full"
                :class="isLoading || !authStore.hasWalletPermission ? 'opacity-70 cursor-default hover:px-5': ''"
                @click="executeGamble">

          <div v-if="isLoading">
            <svg aria-hidden="true" class="w-5 h-5 text-gray-200 animate-spin dark:text-white fill-gray-800" viewBox="0 0 100 101" fill="none" xmlns="http://www.w3.org/2000/svg">
              <path d="M100 50.5908C100 78.2051 77.6142 100.591 50 100.591C22.3858 100.591 0 78.2051 0 50.5908C0 22.9766 22.3858 0.59082 50 0.59082C77.6142 0.59082 100 22.9766 100 50.5908ZM9.08144 50.5908C9.08144 73.1895 27.4013 91.5094 50 91.5094C72.5987 91.5094 90.9186 73.1895 90.9186 50.5908C90.9186 27.9921 72.5987 9.67226 50 9.67226C27.4013 9.67226 9.08144 27.9921 9.08144 50.5908Z"
                    fill="currentColor"/>
              <path d="M93.9676 39.0409C96.393 38.4038 97.8624 35.9116 97.0079 33.5539C95.2932 28.8227 92.871 24.3692 89.8167 20.348C85.8452 15.1192 80.8826 10.7238 75.2124 7.41289C69.5422 4.10194 63.2754 1.94025 56.7698 1.05124C51.7666 0.367541 46.6976 0.446843 41.7345 1.27873C39.2613 1.69328 37.813 4.19778 38.4501 6.62326C39.0873 9.04874 41.5694 10.4717 44.0505 10.1071C47.8511 9.54855 51.7191 9.52689 55.5402 10.0491C60.8642 10.7766 65.9928 12.5457 70.6331 15.2552C75.2735 17.9648 79.3347 21.5619 82.5849 25.841C84.9175 28.9121 86.7997 32.2913 88.1811 35.8758C89.083 38.2158 91.5421 39.6781 93.9676 39.0409Z"
                    fill="currentFill"/>
            </svg>
          </div>
          <span v-else class="flex items-center">
            <div v-html="logo" class="logo-icon"></div>
            {{totalGames === 0 ? 'FLIP NOW': 'PLAY AGAIN'}} <span class="ml-2 text-sm">
        </span>
          </span>

        </button>
        <!-- <p class="text-xs mt-5">If you get all 3 slots equal, you win 25.000 MIST</p> -->
      </div>
    </div>
  </main>
</template>
