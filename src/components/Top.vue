<template>
  <v-container fluid grid-list-md>
    <v-layout row wrap>
      <v-flex d-flex md3 offset-md1>
        <v-layout row wrap>
          <v-flex d-flex md12>
            <v-card>
              <v-card-text>
                Your peer ID is {{peerId}}
              </v-card-text>
            </v-card>
          </v-flex>
          <v-flex d-flex md12>
            <v-card>
              <v-card-text>
                <v-select
                  :items="audios"
                  v-model="selectedAudio"
                  label="Audio input"
                  single-line
                  @change="onChange"
                ></v-select>
                <v-select
                  :items="videos"
                  v-model="selectedVideo"
                  label="Video input"
                  single-line
                  @change="onChange"
                ></v-select>
                <v-select
                  :items="codecs"
                  v-model="selectedCodec"
                  label="Video codec"
                  single-line
                ></v-select>
              </v-card-text>
              <v-card-actions>
                <v-card-text>
                  Screen share settings
                </v-card-text>
                <v-btn icon @click.native="show = !show">
                  <v-icon>{{ show ? 'keyboard_arrow_down' : 'keyboard_arrow_up' }}</v-icon>
                </v-btn>
              </v-card-actions>
              <v-slide-y-transition>
                <v-card-text v-show="show">
                  <v-text-field
                    label="Width (px)"
                    v-model="width"
                    @change="onChange"
                  ></v-text-field>
                  <v-text-field
                    label="Height (px)"
                    v-model="height"
                    @change="onChange"
                  ></v-text-field>
                  <v-text-field
                    label="Frame Rate (fps)"
                    v-model="frameRate"
                    @change="onChange"
                  ></v-text-field>
                  <v-select
                    :items="mediaSourceItem"
                    v-model="mediaSource"
                    label="Media source"
                    single-line
                    @change="onChange"
                  ></v-select>
                </v-card-text>
              </v-slide-y-transition>
            </v-card>
          </v-flex>
          <v-flex d-flex md12>
            <v-card>
              <v-card-text>
                <v-text-field
                  label="Call ID"
                  v-model="callId"
                  required
                ></v-text-field>
              </v-card-text>
              <v-card-actions>
                <v-btn
                  @click="callByName"
                  color="success"
                  :disabled="(!callId || existingCall)"
                >
                  Call
                </v-btn>
                <v-btn
                  @click="close"
                  color="error"
                >
                  Close
                </v-btn>
              </v-card-actions>
            </v-card>
          </v-flex>
          <v-flex d-flex md12>
            <v-card>
              <v-card-title>
                Your video
              </v-card-title>
              <v-card-media>
                <video
                  id="my-video"
                  muted="true"
                  autoplay
                  playsinline
                  controls
                ></video>
              </v-card-media>
            </v-card>
          </v-flex>
        </v-layout>
      </v-flex>
      <v-flex d-flex md7>
        <v-card>
          <v-card-title v-if="existingCall">
            Connecting {{ existingCall.remoteId }}
          </v-card-title>
          <v-card-media>
            <video
              id="their-video"
              autoplay
              playsinline
              controls
            ></video>
          </v-card-media>
        </v-card>
      </v-flex>
    </v-layout>

    <v-dialog v-model="dialog" persistent max-width="300px">
      <v-card>
        <v-card-title v-if="call">
          <span class="headline">Calling from {{ call.remoteId }}</span>
        </v-card-title>
        <v-card-actions>
          <v-spacer></v-spacer>
          <v-btn color="success" @click="connect">Connect</v-btn>
          <v-btn color="error" @click="disconnect">Disconnect</v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>

    <v-snackbar
      color="error"
      v-model="snackbar"
    >
      {{ errorMessage.toString() }}
      <v-btn dark flat @click="closeSnackbar">Close</v-btn>
    </v-snackbar>
  </v-container>
</template>

<script>
import Vue from 'vue'
import Peer from 'skyway-js'
import { key } from '../credentials'

export default {
  props: ['name'],
  data () {
    return {
      peer: {},
      screenShare: {},
      peerId: '',
      callId: '',
      show: false,
      dialog: false,
      width: 600,
      height: 400,
      frameRate: 24,
      mediaSource: 'window',
      audios: [],
      videos: [{
        text: 'None',
        value: null
      },
      {
        text: 'Screen share',
        value: 'screenShare'
      }],
      codecs: ['VP8', 'VP9', 'H264'],
      mediaSourceItem: ['window', 'application', 'screen'],
      selectedAudio: '',
      selectedVideo: '',
      selectedCodec: 'VP9',
      localStream: null,
      call: null,
      existingCall: null,
      errorMessage: ''
    }
  },
  mounted: function () {
    this.peer = new Peer(this.name, {
      key: key,
      debug: 3
    })

    this.screenShare = ScreenShare.create({ debug: true })

    this.peer.on('open', () => {
      this.peerId = this.peer.id
    })

    this.peer.on('close', () => {
      this.close()
    })

    this.peer.on('error', () => {
      this.close()
    })

    this.peer.on('disconnected', () => {
      this.close()
    })

    this.peer.on('call', (call) => {
      this.dialog = true
      this.call = call
    })

    navigator.mediaDevices.enumerateDevices()
    .then((deviceInfos) => {
      for (let i = 0; i !== deviceInfos.length; ++i) {
        const deviceInfo = deviceInfos[i]
        if (deviceInfo.kind === 'audioinput') {
          this.audios.push({
            text: deviceInfo.label ||
            `Microphone ${this.audios.length + 1}`,
            value: deviceInfo.deviceId
          })
        } else if (deviceInfo.kind === 'videoinput') {
          this.videos.push({
            text: deviceInfo.label ||
            `Camera  ${this.videos.length - 1}`,
            value: deviceInfo.deviceId
          })
        }
      }
    })
  },
  computed: {
    snackbar: {
      get: function () {
        return !!this.errorMessage
      },
      set: function () {}
    }
  },
  methods: {
    onChange: function () {
      Vue.nextTick(() => {
        // See. https://github.com/vuejs/vue/issues/293#issuecomment-265716984
        if (this.selectedVideo === 'screenShare') {
          if (!this.screenShare.isScreenShareAvailable()) {
            this.errorMessage = 'Screen Share cannot be used. Please install the Chrome extension.'
            return
          }

          this.screenShare.start({
            width: this.width,
            height: this.height,
            frameRate: this.frameRate,
            mediaSource: this.mediaSource
          }).then((screenStream) => {
            if (this.selectedAudio) {
              const constraints = {
                audio: { deviceId: { exact: this.selectedAudio } },
                video: false
              }
              navigator.mediaDevices.getUserMedia(constraints).then((audioStream) => {
                const stream = new MediaStream()
                screenStream.getVideoTracks().forEach((track) => stream.addTrack(track.clone()))
                audioStream.getAudioTracks().forEach((track) => stream.addTrack(track.clone()))
                this.replaceStream(stream)
              }).catch((err) => {
                this.errorMessage = err
              })
            } else {
              this.replaceStream(screenStream)
            }
          }).catch((err) => {
            this.errorMessage = err
          })
        } else {
          const constraints = {
            audio: this.selectedAudio ? { deviceId: { exact: this.selectedAudio } } : false,
            video: this.selectedVideo ? { deviceId: { exact: this.selectedVideo } } : false
          }
          navigator.mediaDevices.getUserMedia(constraints).then((stream) => {
            this.replaceStream(stream)
          }).catch((err) => {
            this.errorMessage = err
          })
        }
      })
    },
    callByName: function () {
      this.receive(this.peer.call(this.callId, this.localStream, {
        videoCodec: this.selectedCodec
      }))
    },
    connect: function () {
      this.dialog = false
      this.call.answer(this.localStream, {
        videoCodec: this.selectedCodec
      })
      this.receive(this.call)
      this.call = null
    },
    disconnect: function () {
      this.dialog = false
      this.call.close()
      this.call = null
    },
    receive: function (call) {
      this.close()
      call.on('stream', (stream) => {
        const el = document.getElementById('their-video')
        el.srcObject = stream
        el.play()
      })
      this.existingCall = call
    },
    close: function () {
      if (this.existingCall) {
        this.existingCall.close()
        this.existingCall = null
      }
    },
    closeSnackbar: function () {
      this.errorMessage = ''
    },
    replaceStream: function (stream) {
      document.getElementById('my-video').srcObject = stream
      this.localStream = stream

      if (this.existingCall) {
        this.existingCall.replaceStream(stream)
      }
    }
  }
}
</script>

<style scoped>
video {
  width: 100%;
}
</style>
