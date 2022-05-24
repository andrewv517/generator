<template>

  <div style="font-family: ProximaBold,serif;"> </div>
  <img src="./assets/blankboard2.png" width="1280" height="720" class="hidden" alt="blankboard" ref="image"/>

  <div
      class="p-6 bg-slate-700 w-11/12 sm:w-1/2 m-auto mt-10 rounded-lg drop-shadow-2xl  flex flex-col justify-center items-center space-y-4 mb-10">

    <p class="text-white text-center font-semibold text-2xl mb-2">Enter all scores (Pro and Am) before generating
      images</p>

    <div>
      <label for="round" class="text-white">Round: </label>
      <select class="form-select rounded-md text-black p-1" id="round" v-model="round">
        <option :value="1">1</option>
        <option :value="2">2</option>
        <option :value="3">3</option>
      </select>
    </div>

    <div
        class="flex flex-col md:flex-row justify-center md:justify-between items-center space-x-4 space-y-4 md:space-y-0">

      <div class="space-x-2 flex justify-center items-center">
        <label for="flight" class="text-white">Flight: </label>
        <select class="form-select rounded-md text-black p-1" id="flight" v-model="flight">
          <option value="pro">Pro</option>
          <option value="am">Am</option>
        </select>
      </div>

      <div class="space-x-2 flex justify-center items-center">
        <label for="name" class="text-white">Name: </label>
        <input v-model="inputValue" id="name" class="border border-slate-500 rounded-md p-1"/>
      </div>

      <div class="space-x-2 flex justify-center items-center" v-for="i in round" :key="i">
        <label for="name" class="text-white">Round {{ i }}: </label>
        <input v-model="roundValue[i-1]" id="round" class="border border-slate-500 rounded-md p-1"/>
      </div>
    </div>


    <button @click="addPlayer" class="bg-green-400 pl-4 pr-4 pt-2 pb-2 rounded-xl w-fit text-white">Add player</button>

    <hr class="border-gray-500 w-full">

    <ul class="text-white list-decimal space-y-1">
      <li v-for="(player, index) in proPlayers" :key="player">{{ player.name }} {{ player.scores }}

        <span class="text-red-600 font-semibold cursor-pointer" @click="removePlayer(index, true)">X</span>

      </li>
    </ul>

    <ul class="text-white list-decimal space-y-1">
      <li v-for="(player, index) in amPlayers" :key="player">{{ player.name }} {{ player.scores }}

        <span class="text-red-600 font-semibold cursor-pointer" @click="removePlayer(index, false)">X</span>

      </li>
    </ul>

    <hr class="border-gray-500 w-full">

    <canvas ref="procanvas" :width="1280" :height="720" class="hidden"></canvas>
    <canvas ref="amcanvas" :width="1280" :height="720" class="hidden"></canvas>

    <button
        class="bg-cyan-500 pl-4 pr-4 pt-2 pb-2 rounded-xl w-fit text-white"
        @click="generateImage(true)"
    >Download Pro Image
    </button>

    <button
        class="bg-yellow-500 pl-4 pr-4 pt-2 pb-2 rounded-xl w-fit text-white"
        @click="generateImage(false)"
    >Download Am Image
    </button>

    <a ref="link"/>
    <p class="text-slate-500 text-center">Copyright © 2022 Andrew Vadeika</p>
  </div>


</template>

<script setup lang="ts">
import {ref} from "vue";
import {Player} from "@/types";

const procanvas = ref<HTMLCanvasElement | null>(null);
const amcanvas = ref<HTMLCanvasElement | null>(null);
const link = ref<HTMLAnchorElement | null>(null);
const proPlayers = ref<Player[]>([]);
const amPlayers = ref<Player[]>([]);
const inputValue = ref("");
const round = ref(1);
const roundValue = ref([]);
const flight = ref("pro");
const image = ref<HTMLImageElement | null>(null);
let topScorers: Player[] = []

const addPlayer = () => {

  if (flight.value === "pro") {
    proPlayers.value.push({
      name: inputValue.value,
      scores: [...roundValue.value.map(el => parseInt(el))],
      pro: true,
    })

    proPlayers.value.sort((a, b) => a.scores.reduce((prev, cur) => prev + cur) - b.scores.reduce((prev, cur) => prev + cur))

  } else {
    amPlayers.value.push({
      name: inputValue.value,
      scores: [...roundValue.value.map(el => parseInt(el))],
      pro: false,
    })

    amPlayers.value.sort((a, b) => a.scores.reduce((prev, cur) => prev + cur) - b.scores.reduce((prev, cur) => prev + cur))

  }
}

const removePlayer = (indexToDelete: number, pro: boolean) => {
  if (pro) {
    proPlayers.value = proPlayers.value.filter((_, index) => index !== indexToDelete);
  } else {
    amPlayers.value = amPlayers.value.filter((_, index) => index !== indexToDelete);
  }
}

const generateImage = (pro: boolean) => {

  if (pro) {
    // add any am flighters who made it
    const worstProFlightScore = proPlayers.value[proPlayers.value.length - 1].scores.reduce((prev, cur) => prev + cur);
    topScorers = [
      ...proPlayers.value,
      ...amPlayers.value.filter(el => el.scores.reduce((prev, cur) => prev + cur) < worstProFlightScore)
    ]
    topScorers.sort((a, b) => a.scores.reduce((prev, cur) => prev + cur) - b.scores.reduce((prev, cur) => prev + cur))

    if (procanvas.value) {
      const ctx = procanvas.value?.getContext("2d");
      if (ctx && image.value) {

        ctx.drawImage(image.value, 0, 0);

        ctx.fillStyle = "white";
        ctx.font = "33px ProximaBold";

        let inTie = false;
        topScorers.forEach((player, index) => {

          const y = 265 + index * 45;
          const endTie = getEndTieYCoordinate(index, true);
          const score = getScoreString(player);

          ctx.strokeStyle = "white";
          if (!inTie) {
            inTie = y != endTie;
            ctx.fillText((index + 1) + "", 305 - 50, y);
            ctx.moveTo(305 - 20, y - 25);
            ctx.lineTo(305 - 20, endTie + 5);
          } else if (y == endTie) {
            inTie = false;
          }

          ctx.stroke();
          ctx.fillStyle = score[0] === "-" ? "red" : "white";
          ctx.fillText(score, score === "E" ? 942 : (score.length === 2 ? 935 : 930), y);

          ctx.fillStyle = "white";
          ctx.fillText(player.name + (!player.pro ? " (a)" : ""), 305, y);

          ctx.fillStyle = "gold";
          ctx.fillText(player.scores[round.value - 1] + "", 1026, y);
          ctx.fillStyle = "white";

          // const grd = ctx.createLinearGradient(305, y + 10, 1032 + 30, y + 10);
          // grd.addColorStop(0, "#2b82b7");
          // grd.addColorStop(1, "white");

          // Fill with gradient
          ctx.strokeStyle = "gray";

          ctx.moveTo(305, y + 10);
          ctx.lineTo(1032 + 30, y + 10);
          ctx.stroke();


        })

      }
      if (link.value) {
        // here is the most important part because if you dont replace you will get a DOM 18 exception.
        link.value?.setAttribute('download', 'PRO.png');
        link.value?.setAttribute('href', procanvas.value?.toDataURL("image/png").replace("image/png", "image/octet-stream"));
      }
      link.value?.click();
      ctx?.clearRect(0, 0, 1280, 720);
    }

  } else {
    // AM FLIGHT
    if (amcanvas.value) {
      const ctx = amcanvas.value?.getContext("2d");
      if (ctx && image.value) {
        ctx.drawImage(image.value, 0, 0);

        ctx.fillStyle = "white";
        ctx.font = "33px ProximaBold";

        let inTie = false;
        amPlayers.value.forEach((player, index) => {

          const y = 265 + index * 45;
          const endTie = getEndTieYCoordinate(index, false);
          const score = getScoreString(player);

          ctx.strokeStyle = "white";
          if (!inTie) {
            inTie = y != endTie;
            ctx.fillText((index + 1) + "", 305 - 50, y);
            ctx.moveTo(305 - 20, y - 25);
            ctx.lineTo(305 - 20, endTie + 5);
          } else if (y == endTie) {
            inTie = false;
          }

          ctx.stroke();
          ctx.fillStyle = score[0] === "-" ? "red" : "white";
          ctx.fillText(score, score === "E" ? 942 : (score.length === 2 ? 935 : 930), y);

          ctx.fillStyle = "white";
          ctx.fillText(player.name, 305, y);

          ctx.fillStyle = "gold";
          ctx.fillText(player.scores[round.value - 1] + "", 1026, y);
          ctx.fillStyle = "white";

          // Fill with gradient
          ctx.strokeStyle = "gray";

          ctx.moveTo(305, y + 10);
          ctx.lineTo(1032 + 30, y + 10);
          ctx.stroke();


        })

      }
      if (link.value) {
        // here is the most important part because if you dont replace you will get a DOM 18 exception.
        link.value?.setAttribute('download', 'AM.png');
        link.value?.setAttribute('href', amcanvas.value?.toDataURL("image/png").replace("image/png", "image/octet-stream"));
      }
      link.value?.click();
      ctx?.clearRect(0, 0, 1280, 720);
    }
  }

}

const getScoreString = (player: Player): string => {

  const par = 35 * round.value;
  const score = player.scores.slice(0, round.value).reduce((previousValue, currentValue) => previousValue + currentValue);

  if (score > par) {
    return "+" + (score - par);
  } else if (score < par) {
    return "-" + (par - score);
  }
  return "E";
}

const getEndTieYCoordinate = (indexToStart: number, pro: boolean): number => {
  for (let i = indexToStart + 1; i < (pro ? topScorers.length : amPlayers.value.length); i++) {
    let scoreCur;
    let scorePrev;
    if (pro) {
      scoreCur = topScorers[i].scores.slice(0, round.value).reduce((previousValue, currentValue) => previousValue + currentValue);
      scorePrev = topScorers[i - 1].scores.slice(0, round.value).reduce((previousValue, currentValue) => previousValue + currentValue);
    } else {
      scoreCur = amPlayers.value[i].scores.slice(0, round.value).reduce((previousValue, currentValue) => previousValue + currentValue);
      scorePrev = amPlayers.value[i - 1].scores.slice(0, round.value).reduce((previousValue, currentValue) => previousValue + currentValue);
    }

    if (scoreCur !== scorePrev) {
      return 265 + (i - 1) * 45;
    }
  }
  return 265 + ((pro ? topScorers.length : amPlayers.value.length) - 1) * 45;
}

</script>

<style>
</style>
