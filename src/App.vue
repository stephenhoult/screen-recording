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

    <video
        v-if="(source === 'camera' || source === 'screen') && isRecording"
        :src-object.prop.camel="stream"
        autoplay
    />

    <video
        v-if="source === 'both' && isRecording"
        :src-object.prop.camel="stream"
        autoplay
    />

    <div v-if="url !== ''">
      <p class="green">Video recorded.</p>

      <input type="text" v-model="filename" placeholder="File name...">

      <a :href="url" :download="downloadLink" >
        <button>
          Download
        </button>
      </a>
    </div>

  </div>
</template>

<script>

export default {
  name: 'App',
  computed: {
    downloadLink: function() {
      if (this.url !== '') {
        if (this.filename !== '') {
          return this.filename + '.webm';
        } else {
          return this.url + '.webm';
        }
      }
      return '';
    }
  },
  data() {
    return {
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
      filename: '',
      isRecording: false,
      options: { mimeType: "video/webm; codecs=vp9" },
      mediaRecorder: {},
      recordedChunks: [],
      source: '',
      stream: {},
      url: '',
    }
  },
  methods: {
    createVideo() {
      let blob = new Blob(this.recordedChunks, {
        type: "video/webm",
      });
      this.createDownloadLink(blob);
    },
    createDownloadLink(blob) {
      this.url =  URL.createObjectURL(blob);
    },
    handleDataAvailable(event) {
      if (event.data.size > 0) {
        this.recordedChunks.push(event.data);
        this.createVideo();
      } else {
        console.log('No data available')
      }
    },
    async record(source) {
      this.reset();

      this.source = source;

      try {

        // what are we trying to record
        if (source === 'camera' || source === 'screen') {

          if (source === 'camera') {
            this.stream = await navigator.mediaDevices.getUserMedia(this.displayOptions);
          } else {
            this.stream = await navigator.mediaDevices.getDisplayMedia(this.displayOptions);
          }

          // create the media recorder instance
          this.mediaRecorder = new MediaRecorder(this.stream, this.options);

          // bind the method we'll call when data is available
          this.mediaRecorder.ondataavailable = this.handleDataAvailable; // once finished capturing screen

          // start recording
          this.mediaRecorder.start();

        } else { // we're trying to record both
          // ...
        }

        this.isRecording = true;
      } catch (err) {
        console.error(err);
        this.isRecording = false;
      }
    },
    reset() {
      // resets our variables to the defaults
      this.filename = '';
      this.isRecording = false;
      this.mediaRecorder = {};
      this.recordedChunks = [];
      this.source = '';
      this.stream = {};
      this.url = '';
    },
    stopRecording() {
      // stop the recorder
      if (this.source === 'both') {
        // ...
      } else {
        this.mediaRecorder.stop();
      }

      // stop the camera tracks
      let tracks;
      if (this.source === 'both') {
        //...
      } else {
        tracks = this.stream.getTracks();
      }
      tracks.forEach((track) => {
        track.stop();
      });

      this.isRecording = false;
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
  margin-top: 60px;
}

button {
  margin: 5px;
}

.green {
  color: darkseagreen;
}

video {
  clear:both;
  height:400px;
}

</style>
