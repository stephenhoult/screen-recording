<template>
  <div id="app">

    <div>
      <button v-if="!isRecording" type="button" @click="record('camera')">
        Record Camera
      </button>

      <button v-if="!isRecording" type="button" @click="record('screen')">
        Record Screen
      </button>

      <button v-if="!isRecording" type="button" @click="record('both')">
        Record Screen & Camera
      </button>

      <button v-else type="button" @click="stopRecording">
        Stop recording
      </button>
    </div>

    <div v-if="source === 'both'">
      <p>Set camera position</p>
      <button @click="camPosition='top-left'">Top left</button>
      <button @click="camPosition='top-right'">Top right</button>
      <br>
      <button @click="camPosition='bottom-left'">Bottom left</button>
      <button @click="camPosition='bottom-right'">Bottom right</button>
    </div>


    <div v-if="isRecording" class="main-buffer">
      <video
          id="mainBuffer"
          :src-object.prop.camel="streams[0]"
          autoplay
          muted
      />

      <div v-if="source === 'both'" :class="['secondary-buffer', camPosition]">
        <video
            id="secondaryBuffer"
            :src-object.prop.camel="streams[1]"
            autoplay
            muted
        />
      </div>
    </div>

    <div v-if="!isRecording && mainBufferUrl !== '' && showRecording" class="main-buffer">
      <video id="mainBuffer" :src="mainBufferUrl" />

      <div v-if="source === 'both'" :class="['secondary-buffer', camPosition]">
        <video id="secondaryBuffer" :src="secondaryBufferUrl" />
      </div>
    </div>

    <div v-if="mainBufferUrl !== ''">

      <div>
        <input type="text" v-model="trimStart">
        <input type="text" v-model="trimEnd">
        <button @click="applyTrim">Apply Trim</button>
      </div>

      <p class="green">Video recorded.</p>

      <button @click="playPause">Play / Pause</button>

      <div>
        <input type="text" v-model="mainBufferFilename" placeholder="File name...">
        <a :href="mainBufferUrl" :download="mainBufferDownloadLink">
          <button>
            Download Main
          </button>
        </a>
      </div>

    </div>

    <div v-if="secondaryBufferUrl !== ''">
      <input type="text" v-model="secondaryBufferFilename" placeholder="File name...">

      <a :href="secondaryBufferUrl" :download="secondaryBufferDownloadLink">
        <button>
          Download Camera
        </button>
      </a>
    </div>

    <div id="countdown" v-if="countdownTime"><p>{{countdownTime}}</p></div>
  </div>
</template>

<script>

export default {
  name: 'App',
  computed: {
    mainBufferDownloadLink() {
      if (this.mainBufferUrl !== '') {
        if (this.mainBufferFilename !== '') {
          return this.mainBufferFilename + '.webm';
        } else {
          return this.mainBufferUrl + '.webm';
        }
      }
      return '';
    },
    secondaryBufferDownloadLink() {
      if (this.secondaryBufferUrl !== '') {
        if (this.secondaryBufferFilename !== '') {
          return this.secondaryBufferFilename + '.webm';
        } else {
          return this.secondaryBufferUrl + '.webm';
        }
      }
      return '';
    }
  },
  data() {
    return {
      camPosition: 'bottom-left',
      chunkSize: 1000, // milliseconds
      countdownTime: 0,
      displayOptions: {
        audio: {
          echoCancellation: true,
          noiseSuppression: true,
          sampleRate: 44100,
        },
        video: {
          cursor: "always",
        },
      },
      isRecording: false,
      options: {mimeType: "video/webm;codecs=h264"},
      mediaRecorders: [],
      mainBuffer: [],
      mainBufferFilename: '',
      mainBufferUrl: '',
      secondaryBuffer: [],
      secondaryBufferFilename: '',
      secondaryBufferUrl: '',
      showRecording: false,
      source: '',
      streams: [],
      trimStart: 0,
      trimEnd: null,
    }
  },
  methods: {
    applyTrim() {
      if (this.trimStart < 0 || this.trimEnd > this.mainBuffer.length) {
        console.log('invalid trim values');
        return;
      }
      let trimmed = this.mainBuffer.slice(this.trimStart, this.trimEnd);

      this.createVideo(trimmed, 'main');

      if (this.source === 'both') {

        if (this.trimStart < 0 || this.trimEnd > this.secondaryBuffer.length) {
          console.log('invalid trim values');
          return;
        }
        trimmed = this.secondaryBuffer.slice(this.trimStart, this.trimEnd);
        this.createVideo(trimmed, 'secondary');
      }
    },
    countdown() {
      setTimeout(() => {
        this.countdownTime -= 1;
        if (this.countdownTime > 0) {
          this.countdown();
        } else {
          this.startRecording();
        }
      }, 1000);
    },
    async createVideo(buffer, type) {
      let blob = await new Blob(buffer, {
        type: "video/webm",
      });

      if (type === 'main') {
        this.mainBuffer = buffer;
      } else if (type === 'secondary') {
        this.secondaryBuffer = buffer;
      }
      this.createDownloadLink(blob, type);
    },
    createDownloadLink(blob, type) {
      if (type === 'main') {
        this.mainBufferUrl = URL.createObjectURL(blob);
        this.mainBufferUrl += '#t='+this.mainBuffer.length;
        this.trimEnd = this.mainBuffer.length
      } else if (type === 'secondary') {
        this.secondaryBufferUrl = URL.createObjectURL(blob);
        this.secondaryBufferUrl += '#t='+this.secondaryBuffer.length;
      }
    },
    handleMainBuffer(event) {
      if (event.data.size > 0) {
        this.mainBuffer.push(event.data);
      } else {
        console.log('No data available')
      }
    },
    handleSecondaryBuffer(event) {
      if (event.data.size > 0) {
        this.secondaryBuffer.push(event.data);
      } else {
        console.log('No data available')
      }
    },
    playPause() {
      let videos = document.getElementsByTagName("video");

      for (let i = 0; i < videos.length; i++) {
        if (videos[i].paused) {
          videos[i].play();
        } else {
          videos[i].pause();
        }
      }
    },
    async record(source) {
      this.reset();

      this.source = source;

      try {

        // we're recording a single source
        if (source === 'camera' || source === 'screen') {

          let stream;
          if (source === 'camera') {
            stream = await navigator.mediaDevices.getUserMedia(this.displayOptions);
            this.streams.push(stream);
          } else if (screen) {
            stream = await navigator.mediaDevices.getDisplayMedia(this.displayOptions);
            this.streams.push(stream);
          }

          // create the media recorder instance
          let mediaRecorder = new MediaRecorder(stream, this.options);

          // bind the method we'll call when data is available
          mediaRecorder.ondataavailable = this.handleMainBuffer;

          // add the media recorder to our array of media recorders
          this.mediaRecorders.push(mediaRecorder);

        } else { // we're trying to record multiple sources

          // screen
          // screen stream
          let screenStream = await navigator.mediaDevices.getDisplayMedia(this.displayOptions);
          // create the media recorder instance for the screen
          let screenMediaRecorder = new MediaRecorder(screenStream, this.options);
          // bind the method we'll call when data is available for the screen
          screenMediaRecorder.ondataavailable = this.handleMainBuffer;

          // camera
          // camera stream
          let camStream = await navigator.mediaDevices.getUserMedia({...this.displayOptions});
          // create the media recorder instance for the camera
          let camMediaRecorder = new MediaRecorder(camStream, this.options);
          // bind the method we'll call when data is available for the camera
          camMediaRecorder.ondataavailable = this.handleSecondaryBuffer;

          // streams
          this.streams.push(screenStream);
          this.streams.push(camStream);

          // add the media recorder to our array of media recorders
          this.mediaRecorders.push(screenMediaRecorder);
          this.mediaRecorders.push(camMediaRecorder);
        }

        // this setTimeout is very important! It makes sure the webcam is ready and ensures the audio is in sync
        this.countdownTime = 3;
        this.countdown();

      } catch (err) {
        console.error(err);
        this.isRecording = false;
      }
    },
    reset() {
      // resets our variables to the defaults
      this.isRecording = false;
      this.mainBufferFilename = '';
      this.mainBuffer = [];
      this.mediaRecorders = [];
      this.secondaryBuffer = [];
      this.secondaryBufferFilename = '';
      this.showRecording = false;
      this.source = '';
      this.streams = [];
    },
    startRecording() {
      this.mediaRecorders.map(mediaRecorder => mediaRecorder.start(this.chunkSize));
      this.isRecording = true;
    },
    async stopRecording() {
      // stop the media recorders
      this.mediaRecorders.map(mediaRecorder => mediaRecorder.stop());

      // stop the camera tracks
      this.streams.map(stream => stream.getTracks().map(track => track.stop()));

      this.isRecording = false;

      await this.createVideo(this.mainBuffer, 'main');
      await this.createVideo(this.secondaryBuffer, 'secondary');

      this.showRecording = true;
    },
  }
}
</script>

<style>

#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 20px;
}

button {
  margin: 5px;
}

#countdown {
  position:absolute;
  width:100%;
  height:100%;
  background: rgba(0, 0, 0, 0.7);
  top:0;
  left:0;
  margin:0;
  display: flex;
  justify-content: center;
}

#countdown p {
  font-size:100pt;
  color:#fff;
  font-weight: bold;
  align-self: center;
}

.green {
  color: darkseagreen;
}

.main-buffer {
  position: relative;
  width: fit-content;
  margin: 0 auto;
}

.secondary-buffer {
  border-radius: 50%;
  overflow: hidden;
  z-index: 99;
  position: absolute;
  background: green;
  width: 200px;
  height: 200px;
  padding: 0;
  margin: 0;
  box-shadow: 0px 0px 30px #ccc;
}

.secondary-buffer.bottom-left {
  bottom: 40px;
  left: 20px;
}

.secondary-buffer.bottom-right {
  bottom: 40px;
  right: 20px;
}

.secondary-buffer.top-left {
  top: 40px;
  left: 20px;
}

.secondary-buffer.top-right {
  top: 40px;
  right: 20px;
}

.secondary-buffer video {
  height: 300px;
  width: 300px;
  margin-top: -25%;
  margin-left: -23%;

}

video {
  clear: both;
  height: 800px;
}

</style>
