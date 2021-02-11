<template>
  <div>
    Server: <input type="text" v-model="serverUrl"><br>
    Your ID: <input type="text" v-model="connectionId"><br>
    Peer: ID:<input type="text" v-model="peerConnectionId"><br>
    <button v-on:click="connect()">Connect</button>
  </div>
</template>

<script lang="ts">
/* tslint:disable:no-console */

import { Component, Prop, Vue } from 'vue-property-decorator';
import MediaStreamRecorder from 'msr';
import urlJoin from 'url-join';

(window as any).AudioContext = (window as any).AudioContext || (window as any).webkitAudioContext;


/**
 * Get random ID
 * @param len
 */
function getRandomId(len: number): string {
  // NOTE: some similar shaped alphabets are not used
  const alphas  = [
    'a', 'b', 'c', 'd', 'e', 'f', 'h', 'i', 'j', 'k', 'm', 'n', 'p', 'r', 's', 't', 'u', 'v', 'w', 'x', 'y', 'z',
  ];
  const chars   = [...alphas];
  const randomArr = window.crypto.getRandomValues(new Uint32Array(len));
  return Array.from(randomArr).map((n) => chars[n % chars.length]).join('');
}

function createUrl(serverUrl: string, fromId: string, toId: string, chunkNum: number): string {
  return urlJoin(serverUrl, 'phone-web', `${fromId}-to-${toId}`, chunkNum.toString());
}

@Component
export default class PipingPhone extends Vue {
  private serverUrl: string = 'https://ppng.io';
  private connectionId: string = '';
  private peerConnectionId: string = '';

  public mounted() {
    this.connectionId = getRandomId(3);
  }

  private connect() {
    this.record();
    this.play();
  }

  private record() {
    const onMediaSuccess = (stream: MediaStream) => {
      const mediaRecorder = new MediaStreamRecorder(stream);
      mediaRecorder.mimeType = 'audio/wav'; // check this line for audio/wav
      console.log('record start');

      let chunkNum: number = 1;
      let nFetchingReqs: number = 0;

      mediaRecorder.ondataavailable = (blob: Blob) => {
        console.log(blob);

        // Skip fetch-request
        if (nFetchingReqs > 2) {
          console.log('send skip:', chunkNum);
          return;
        }

        // Send a chunk
        const resPromise = fetch(createUrl(this.serverUrl, this.connectionId, this.peerConnectionId, chunkNum), {
          method: 'POST',
          body: blob,
        });
        // Increment # of fetch-request
        nFetchingReqs++;

        // Wait for the request
        (async () => {
          const res = await resPromise;
          if (res.body === null) { return; }
          await res.body.pipeTo(new WritableStream());
        })().finally(() => {
          nFetchingReqs--;
        });

        // Increment chunk number
        chunkNum++;
      };

      mediaRecorder.start(100);
    };
    navigator.getUserMedia(
      {audio: {
        echoCancellation: true,
      }},
      onMediaSuccess,
      (err) => console.error((err)),
    );
  }

  private async play() {
    const context = new AudioContext();
    let prevStartTime = 0;
    let prevDuration = 0;

    let nFetchingReqs: number = 0;

    for (let chunkNum = 1; ; chunkNum++) {
      console.log('play chunk', chunkNum);

      // Skip fetch-request
      if (nFetchingReqs > 2) {
        console.log('get skip:', chunkNum);
        return;
      }

      // Get a chunk
      const res = await fetch(createUrl(this.serverUrl, this.peerConnectionId, this.connectionId, chunkNum));
      if (res.body === null) {
        console.error('Unexpected error: body is null');
        continue;
      }
      nFetchingReqs++;
      // Read all bytes
      // NOTE: whole body is not too big because whole body is also a chunk.
      const arrayBuffer = await res.arrayBuffer();
      // Decrement # of fetch-request
      nFetchingReqs--;
      // If no bytes
      if (arrayBuffer.byteLength === 0) {
        break;
      }
      const audioBuffer = await context.decodeAudioData(arrayBuffer);
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
