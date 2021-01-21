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
          :src-object.prop.camel="streams[0]"
          autoplay
      />

      <div v-if="source === 'both'" :class="['secondary-buffer', camPosition]">
        <video
            :src-object.prop.camel="streams[1]"
            autoplay
        />
      </div>
    </div>

    <div v-if="!isRecording && mainBufferUrl != ''" class="main-buffer">
      <video>
        <source :src="mainBufferUrl" type="video/webm">
      </video>

      <div v-if="source === 'both'" :class="['secondary-buffer', camPosition]">
        <video>
          <source :src="secondaryBufferUrl" type="video/webm">
        </video>
      </div>
    </div>

    <div v-if="mainBufferUrl !== ''">
      <p class="green">Video recorded.</p>

      <input type="text" v-model="mainBufferFilename" placeholder="File name...">

      <a :href="mainBufferUrl" :download="mainBufferDownloadLink">
        <button>
          Download
        </button>
      </a>

      <button @click="playPause">Play / Pause</button>
    </div>

    <div v-if="secondaryBufferUrl !== ''">
      <input type="text" v-model="secondaryBufferFilename" placeholder="File name...">

      <a :href="secondaryBufferUrl" :download="secondaryBufferDownloadLink">
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
      camPosition: 'bottom-left',
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
      options: {mimeType: "video/webm; codecs=vp9"},
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
        this.mainBufferUrl = URL.createObjectURL(blob);
      } else if (type === 'secondary') {
        this.secondaryBufferUrl = URL.createObjectURL(blob);
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

          // start recording
          mediaRecorder.start();

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
          let camStream = await navigator.mediaDevices.getUserMedia({...this.displayOptions, audio:false});
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

          console.log('waiting...');

          setTimeout(() => {
            console.log('recording...');
            // start recording
            screenMediaRecorder.start();
            camMediaRecorder.start();
          }, 3000);

        }

        this.isRecording = true;
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
      this.source = '';
      this.streams = [];
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
  margin-top: 20px;
}

button {
  margin: 5px;
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
