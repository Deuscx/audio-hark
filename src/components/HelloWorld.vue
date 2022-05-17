<template>
  <div>
    <audio :src="audioUrl"
           ref="audioRef"
           controls></audio>

    <meter min="0"
           max="100"
           :value="dataV"></meter>
  </div>
</template>

<script lang="ts">
import { defineComponent, onMounted, ref } from "vue";
import audioUrl from "@/assets/bgm1.mp3";
import { EventEmitter } from "events";
console.log(audioUrl);

const harkEvent = {
  start: "start",
  stop: "stop",
  volume: "volume",
  error: "error",
};

interface IHarkOptions {
  threshold: number;
}
class Hark extends EventEmitter {
  context: AudioContext;
  mic?: MediaStreamAudioSourceNode;
  analyser: AnalyserNode;
  started: boolean;
  fftBins!: Uint8Array;
  options: IHarkOptions;
  constructor(stream: MediaStream, options: Partial<IHarkOptions> = {}) {
    super();
    this.started = false;
    this.context = new AudioContext();
    this.analyser = this.context.createAnalyser();
    this.analyser.fftSize = 1024;
    // this.analyser.smoothingTimeConstant = 0.8;
    this.connectToSource(stream);
    this.options = Object.assign({}, this.defaultOptions, options);
  }

  connectToSource(stream: MediaStream) {
    this.mic = this.context.createMediaStreamSource(stream);
    this.mic.connect(this.analyser);
    this.analyser.connect(this.context.destination);
    this.fftBins = new Uint8Array(this.analyser.frequencyBinCount);
  }

  _update() {
    if (this.started === true) {
      this.analyser.getByteFrequencyData(this.fftBins);
      this.calculate();
      requestAnimationFrame(this._update.bind(this));
    }
  }
  calculate() {
    let sum = 0;
    for (let i = 0; i < this.analyser.frequencyBinCount; i++) {
      const value = this.fftBins[i];
      sum += Math.pow(value, 2);
    }
    let volume = Math.max(
      Math.floor(Math.sqrt(sum / this.analyser.frequencyBinCount)) -
        this.options.threshold - 25,
      0
    );
    if (volume > 100) {
      console.log(volume);
    }
    super.emit(harkEvent.volume, volume);
  }

  start() {
    this.started = true;
    super.emit(harkEvent.volume, "1");
    this._update();
  }

  stop() {
    this.started = false;
  }

  resume() {
    this.context.resume();
  }

  get defaultOptions() {
    return {
      threshold: 0,
    };
  }
}

export default defineComponent({
  name: "HelloWorld",
  props: {
    msg: String,
  },
  setup() {
    const audioRef = ref<HTMLAudioElement>();
    const dataV = ref<number>(0);
    let hark: Hark;
    onMounted(() => {
      audioRef.value!.volume = 0.1;

      audioRef.value!.addEventListener("loadeddata", () => {
        // eslint-disable-next-line @typescript-eslint/ban-ts-comment
        //@ts-ignore
        hark = new Hark(audioRef.value!.captureStream());
        // eslint-disable-next-line @typescript-eslint/ban-ts-comment
        //@ts-ignore
        window.hark = hark;
        hark.on(harkEvent.volume, (volume) => {
          dataV.value = volume;
        });
      });
      audioRef.value!.addEventListener("play", () => {
        console.log("😀 play");
        hark.start();
      });

      audioRef.value!.addEventListener("pause", () => {
        console.log("😀 pause");
        hark.stop();
      });
    });

    return { audioUrl, audioRef, dataV };
  },
});

document.addEventListener(
  "click",
  () => {
    console.log("😀 resume");
    window.hark.resume();
  },
  { once: true }
);
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped lang="scss">
h3 {
  margin: 40px 0 0;
}
ul {
  list-style-type: none;
  padding: 0;
}
li {
  display: inline-block;
  margin: 0 10px;
}
a {
  color: #42b983;
}
</style>
