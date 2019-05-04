<template>
  <div>
  </div>
</template>

<script lang="ts">
import { Component, Prop, Vue } from 'vue-property-decorator';
import MediaStreamRecorder from 'msr';

(window as any).AudioContext = (window as any).AudioContext || (window as any).webkitAudioContext;

console.log(MediaStreamRecorder);

function blobToArrayBuffer(blob: Blob): Promise<ArrayBuffer> {
  return new Promise((resolve, reject) => {
    const fileReader = new FileReader();
    fileReader.onload = () => {
      resolve(fileReader.result as ArrayBuffer);
    };
    fileReader.onerror = () => {
      reject(fileReader.error);
    };
    fileReader.readAsArrayBuffer(blob);
  });
}


navigator.getUserMedia({audio: true}, onMediaSuccess, (err) => console.error((err)));


function onMediaSuccess(stream: MediaStream): void {
  const mediaRecorder = new MediaStreamRecorder(stream);
  const blobs: Blob[] = [];
  mediaRecorder.mimeType = 'audio/wav'; // check this line for audio/wav
  console.log('record start');
  mediaRecorder.ondataavailable = (blob: Blob) => {
    blobs.push(blob);
    console.log(blob);

    if (blobs.length === 100) {
      (async () => {
        console.log("record end");
        mediaRecorder.stop();
        console.log("play start");

        const context = new AudioContext();
        console.log('context.currentTime', context.currentTime);
        let prevStartTime = 0;
        let prevDuration = 0;

        for (const blob of blobs) {
          console.log(blob);
          const arrayBuffer = await blobToArrayBuffer(blob);
          console.log('arraybuffer', arrayBuffer);
          const audioBuffer = await context.decodeAudioData(arrayBuffer);
          console.log('audio buffer', audioBuffer);
          console.log('audio buffer duration', audioBuffer.duration);
          const source = context.createBufferSource();
          source.buffer = audioBuffer;
          source.connect(context.destination);
          const currentTime = context.currentTime;
          const startTime = Math.max(prevDuration - (currentTime - prevStartTime), 0);
          console.log('startTime', startTime);
          prevStartTime = currentTime + startTime;
          prevDuration = audioBuffer.duration;
          source.start(startTime);
        }
      })();
    }
  };

  mediaRecorder.start(100);
}

@Component
export default class PipingPhone extends Vue {
  @Prop() private msg!: string;
}
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
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
