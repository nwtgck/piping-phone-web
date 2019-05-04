<template>
  <div>
    <input type="text" v-model="dataId">
    <button v-on:click="record()">Record</button>
    <button v-on:click="play()">play</button>
  </div>
</template>

<script lang="ts">
import { Component, Prop, Vue } from 'vue-property-decorator';
import MediaStreamRecorder from 'msr';
import urlJoin from 'url-join';

(window as any).AudioContext = (window as any).AudioContext || (window as any).webkitAudioContext;


async function readAllReadableStream(readableStream: ReadableStream<Uint8Array>): Promise<Uint8Array> {
  // Get all chunks
  const chunks: Uint8Array[] = [];
  let totalLength: number = 0;
  const reader = readableStream.getReader();
  while (true) {
    const {done, value} = await reader.read();
    if (done) {
      break;
    }
    chunks.push(value);
    totalLength += value.byteLength;
  }
  // Get whole bytes
  // (from: https://qiita.com/hbjpn/items/dc4fbb925987d284f491)
  const wholeBytes = new Uint8Array(totalLength);
  let pos = 0;
  for (const chunk of chunks) {
    wholeBytes.set(chunk, pos);
    pos += chunk.byteLength;
  }
  return wholeBytes;
}

@Component
export default class PipingPhone extends Vue {
  private serverUrl: string = 'http://localhost:8080';
  private dataId: string = 'mypath1'; // TODO: Hard code

  private record() {
    const onMediaSuccess = (stream: MediaStream) => {
      const mediaRecorder = new MediaStreamRecorder(stream);
      mediaRecorder.mimeType = 'audio/wav'; // check this line for audio/wav
      console.log('record start');

      let chunkNum: number = 1;

      mediaRecorder.ondataavailable = (blob: Blob) => {
        console.log(blob);

        // Send a chunk
        fetch(urlJoin(this.serverUrl, this.dataId, chunkNum.toString()), {
          method: 'POST',
          body: blob,
        });

        // Increment chunk number
        chunkNum++;
      };

      mediaRecorder.start(100);
    };
    navigator.getUserMedia({audio: true}, onMediaSuccess, (err) => console.error((err)));
  }

  private async play() {
    const context = new AudioContext();
    let prevStartTime = 0;
    let prevDuration = 0;

    for (let chunkNum = 1; ; chunkNum++) {
      // Get a chunk
      const res = await fetch(urlJoin(this.serverUrl, this.dataId, chunkNum.toString()));
      if (res.body === null){
        console.error('Unexpected error: body is null');
        continue;
      }
      // Read all bytes
      // NOTE: whole body is not too big because whole body is also a chunk.
      const bytes: Uint8Array = await readAllReadableStream(res.body);
      // If no bytes
      if (bytes.byteLength === 0) {
        break;
      }
      const audioBuffer = await context.decodeAudioData(bytes.buffer);
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
  }
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
