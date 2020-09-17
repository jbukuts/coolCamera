<template>
  <div id="app">
    <h3>The Stream</h3>
    <canvas id="video"></canvas>

    <canvas id="hidden"> </canvas>
  </div>
</template>

<script>
import * as bodyPix from '@tensorflow-models/body-pix';
import * as tfjs from '@tensorflow/tfjs';

export default {
  name: 'App',
  data() {
    return {
      videoTrack : null,
      width : 640,
      height : 480
    }
  },
  components: {
  },
  methods : {
    async getVideoTrack() {
      const vgaConstraints = {
        video: {width: {exact: 640}, height: {exact: 480}}
      };


      return await navigator.mediaDevices.getUserMedia(vgaConstraints)
        .then( function( mediaStream ) {
          return mediaStream.getVideoTracks()[0];
        })
        .catch( function( err ) {
          console.log( "Unable to access camera: " + err );
        });
    },
  },
  async mounted() {
    this.videoTrack = await this.getVideoTrack();
    let canvas = document.getElementById("video");
    let ctx = canvas.getContext('2d');

    let hidden = document.getElementById("hidden");
    let hiddenCtx = hidden.getContext('2d');


    let imageCapture = new ImageCapture(this.videoTrack);
    var img = new Image();

    canvas.width  = hidden.width = this.width;
    canvas.height = hidden.height = this.height;

    await tfjs.setBackend('webgl');

    const net = await bodyPix.load({
      architecture: 'MobileNetV1',
      outputStride: 16,
      multiplier: 0.75,
      quantBytes: 2
    });

    let d;
  
    setInterval(() => {
        imageCapture.takePhoto().then(async (frame) => {
    
          img.src = URL.createObjectURL(frame);
          img.onload = async () => {

            hiddenCtx.drawImage(img,0,0);
            d = hiddenCtx.getImageData(0, 0, this.width, this.height);


            const segmentation = await net.segmentPerson(img, {
              flipHorizontal: false,
              internalResolution: 'medium',
              segmentationThreshold: 0.7
            });

            d.data.set(d.data.map((curr, index) => {
                return ((index +1) % 4 == 0 && segmentation.data[((index+1)/4)-1] == 0 ) ? segmentation.data[((index+1)/4)-1] : curr;
            }));

            ctx.putImageData(d, 0, 0, 0, 0, d.width, d.height);
          };

        }).catch((e) => e);
    }, 33);
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

#hidden {
  display: none;
}

#video {
  padding-left: 0;
  padding-right: 0;
  margin-left: auto;
  margin-right: auto;
  display: block;
}
</style>
