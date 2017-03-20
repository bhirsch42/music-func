<template>
  <div id="app">
    <input type="file" v-on:change="loadSong">
    <audio id="audio" controls></audio>
    <button v-on:click="audio.play()">Play</button>
    <button v-on:click="audio.pause()">Pause</button>
    <div id="waveform" v-on:click="setPlaybackPosition">
      <div id="playhead" :style="{
        left: `${playbackPosition * 100}%`
      }"></div>
      <div id="left-channel" class="channel">
        <div v-for="peak in peaks.left" class="peak" :style="{
          height: `${peak * 100}px`
        }"></div>        
      </div>
      <div id="right-channel" class="channel">
        <div v-for="peak in peaks.right" class="peak" :style="{
          height: `${peak * 100}px`
        }"></div>
      </div>
    </div>
    <div id="function-tracks">
      <div v-for="track in functionTracks" class="function-track">
        Function: <input v-model="track.name" type="text">
        Input: <input v-on:keydown="addCue(track.calls, $event)" type="text">
        <div class="call-timeline">
          <div v-for="call in track.calls" class="call-marker" :style="{
            left: `${call.playbackPosition * 100}%`
          }"></div>
        </div>
      </div>
      <button v-on:click="addFunctionTrack">Add function track</button>
      <button v-on:click="exportFunctionTracks">Export function tracks</button>
    </div>
  </div>
</template>

<script>
import Hello from './components/Hello'
import _ from 'lodash'

let data = {
  audio: document.getElementById('audio'),
  playbackPosition: 0,
  peaks: {
    left: [],
    right: []
  },
  functionTracks: []
}

function downloadText(filename, text) {
  var element = document.createElement('a');
  element.setAttribute('href', 'data:text/plain;charset=utf-8,' + encodeURIComponent(text));
  element.setAttribute('download', filename);

  element.style.display = 'none';
  document.body.appendChild(element);

  element.click();

  document.body.removeChild(element);
}

function loadAndGenerateWaveform(data) {
  let audioContext = new AudioContext()
  audioContext.decodeAudioData(data, generateWaveform)
}

function generateWaveform(buffer) {
  let leftChannel = buffer.getChannelData(0)
  let rightChannel = buffer.getChannelData(1)

  let numSamples = 3000
  let leftSamples = []
  let rightSamples = []

  for (let i = 0; i < numSamples; i++) {
    let start = parseInt(leftChannel.length * (i / (numSamples + 1)))
    let end = parseInt(leftChannel.length * ((i + 1) / (numSamples + 1)))
    data.peaks.left = data.peaks.left.concat(_.max(leftChannel.slice(start, end)))
    data.peaks.right = data.peaks.right.concat(_.max(rightChannel.slice(start, end)))
  }
}

export default {
  name: 'app',
  components: {
    Hello
  },
  data () {
    return data
  },
  methods: {
    loadSong (e) {
      let file = e.target.files[0]
      let audioReader = new FileReader()
      audioReader.onloadend = (event) => {
        if (event.target.readyState != FileReader.DONE) {
          throw Error("The song didn't load right :(")
        }
        this.audio = document.getElementById('audio')
        console.log('set attribute')
        this.audio.setAttribute('src', event.target.result)
        setInterval(() => {
          data.playbackPosition = data.audio.currentTime / data.audio.duration
        }, 100)
      }
      audioReader.readAsDataURL(file)

      let waveformReader = new FileReader()
      waveformReader.onloadend = (event) => {
        loadAndGenerateWaveform(event.target.result)
      }
      waveformReader.readAsArrayBuffer(file)
    },
    setPlaybackPosition(e) {
      let waveform = document.getElementById('waveform')
      let leftBound = waveform.getBoundingClientRect().left
      let rightBound = waveform.getBoundingClientRect().right
      let position = (e.screenX - leftBound) / (rightBound - leftBound)
      let newTime = position * data.audio.duration
      data.audio.currentTime = newTime
    },
    addFunctionTrack(e) {
      data.functionTracks.push({
        name: "myFunc",
        calls: []
      })
    },
    addCue(calls, e) {
      calls.push({
        key: e.key,
        playbackPosition: data.audio.currentTime / data.audio.duration
      })
      e.target.value = ''
    },
    exportFunctionTracks() {
      downloadText('songFuncs.json', JSON.stringify(data.functionTracks))
    }
  }
}
</script>

<style>
audio {
  visibility: none;
}

#waveform, .call-timeline {
  position: relative;
  width: 100%;
  min-height: 100px;
}

.channel {
  display: flex;
}

#right-channel {
  align-items: flex-start;
}

#left-channel {
  align-items: flex-end;
}

#playhead, .call-marker {
  position: absolute;
  top: 0;
  height: 100%;
  width: 1px;
  background-color: red;
}

.call-marker {
  background-color: blue;
}

.peak {
  width: 100%;
  background-color: lightblue;
}
</style>
