<template>
  <div class="flex flex-col items-center gap-10 m-5">
    <div class="bg-black rounded-2xl min-w-[50vw] min-h-[25vw]">
      <div id="siri-container"></div>
    </div>
    <button @click="toggleMic"
            class="bg-gray-200 rounded-xl py-2 px-8 hover:bg-gray-300 active:translate-y-1 active:bg-gray-200 transition-all duration-75">
      {{ isMicActive ? "Stop" : "Start Listening" }}
    </button>
  </div>
</template>

<script setup>
import {ref} from "vue";
import SiriWave from "siriwave";

const siriWave = ref(null);
const isMicActive = ref(false);
const audioContext = ref(null);
const mediaStream = ref(null);
const analyser = ref(null);

function initSiriWave() {
  siriWave.value = new SiriWave({
    container: document.getElementById("siri-container"),
    style: "ios9",
    width: window.innerWidth / 2,
    height: window.innerWidth / 4,
  });
}

function startMicrophone() {
  navigator.mediaDevices.getUserMedia({audio: true}).then(stream => {
    mediaStream.value = stream;
    audioContext.value = new AudioContext();
    analyser.value = audioContext.value.createAnalyser();
    const microphone = audioContext.value.createMediaStreamSource(stream);
    microphone.connect(analyser.value);
    analyser.value.fftSize = 256;
    initSiriWave();
    updateSiriWave();
  }).catch(error => {
    console.error("Error accessing the microphone", error);
  });
}

function updateSiriWave() {
  const bufferLength = analyser.value.frequencyBinCount;
  const dataArray = new Uint8Array(bufferLength);

  analyser.value.getByteFrequencyData(dataArray);
  let sum = dataArray.reduce((a, b) => a + b, 0);
  let average = sum / bufferLength;
  let normalizedAmplitude = (average / 128) * 2;
  normalizedAmplitude = Math.min(Math.max(normalizedAmplitude, 0), 1);
  siriWave.value.setAmplitude(normalizedAmplitude);

  if (isMicActive.value) {
    requestAnimationFrame(updateSiriWave);
  }
}

function stopMicrophone() {
  audioContext.value.close();
  mediaStream.value.getTracks().forEach((track) => track.stop());
  siriWave.value.dispose();
}

function toggleMic() {
  if (isMicActive.value) {
    isMicActive.value = false;
    stopMicrophone();
  } else {
    isMicActive.value = true;
    startMicrophone();
  }
}
</script>
