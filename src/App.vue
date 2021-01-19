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


    <div v-if="isRecording">
      <video
          v-for="stream in streams"
          :key="stream.id"
          :src-object.prop.camel="stream"
          autoplay
      />
    </div>

    <div v-if="mainBufferUrl !== ''">
      <p class="green">Video recorded.</p>

      <input type="text" v-model="mainBufferFilename" placeholder="File name...">

      <a :href="mainBufferUrl" :download="mainBufferDownloadLink" >
        <button>
          Download
        </button>
      </a>
    </div>

    <div v-if="secondaryBufferUrl !== ''">
      <input type="text" v-model="secondaryBufferFilename" placeholder="File name...">

      <a :href="secondaryBufferUrl" :download="secondaryBufferDownloadLink" >
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
      options: { mimeType: "video/webm; codecs=vp9" },
      mediaRecorders: [],
      mainBuffer: [],
      mainBufferFilename: '',
      mainBufferUrl: '',
      secondaryBuffer: [],
      secondaryBufferFilename: '',
      secondaryBufferUrl: '',
      source: '',
      streams: [],
    }
  },
  methods: {
    createVideo(buffer, type) {
      let blob = new Blob(buffer, {
        type: "video/webm",
      });
      this.createDownloadLink(blob, type);
    },
    createDownloadLink(blob, type) {
      if (type === 'main') {
        this.mainBufferUrl =  URL.createObjectURL(blob);
      } else if (type === 'secondary') {
        this.secondaryBufferUrl =  URL.createObjectURL(blob);
      }
    },
    handleMainBuffer(event) {
      if (event.data.size > 0) {
        this.mainBuffer.push(event.data);
        this.createVideo(this.mainBuffer, 'main');
      } else {
        console.log('No data available')
      }
    },
    handleSecondaryBuffer(event) {
      if (event.data.size > 0) {
        this.secondaryBuffer.push(event.data);
        this.createVideo(this.secondaryBuffer, 'secondary');
      } else {
        console.log('No data available')
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

          // start recording
          mediaRecorder.start();

          // add the media recorder to our array of media recorders
          this.mediaRecorders.push(mediaRecorder);

        } else { // we're trying to record multiple sources

          let stream;
          let mediaRecorder;

          //
          // camera
          //

          // camera stream
          stream = await navigator.mediaDevices.getUserMedia(this.displayOptions);
          this.streams.push(stream);

          // create the media recorder instance for the camera
          mediaRecorder = new MediaRecorder(stream, this.options);

          // bind the method we'll call when data is available for the camera
          mediaRecorder.ondataavailable = this.handleSecondaryBuffer;

          // start recording the camera
          mediaRecorder.start();

          // add the camera media recorder to our array of media recorders
          this.mediaRecorders.push(mediaRecorder);

          //
          // screen
          //

          // screen stream
          stream = await navigator.mediaDevices.getDisplayMedia(this.displayOptions);
          this.streams.push(stream);

          // create the media recorder instance for the screen
          mediaRecorder = new MediaRecorder(stream, this.options);

          // bind the method we'll call when data is available for the screen
          mediaRecorder.ondataavailable = this.handleMainBuffer;

          // start recording the screen
          mediaRecorder.start();

          // add the screen media recorder to our array of media recorders
          this.mediaRecorders.push(mediaRecorder);

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
      this.mediaRecorders = [];
      this.mainBuffer = [];
      this.secondaryBuffer = [];
      this.source = '';
      this.streams = [];
      this.url = '';
    },
    stopRecording() {
      // stop the media recorders
      this.mediaRecorders.forEach(mediaRecorder => {
        mediaRecorder.stop();
      })

      // stop the camera tracks
      this.streams.forEach(stream => {
        let tracks = stream.getTracks();
        tracks.forEach(track => {
          track.stop();
        })
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
