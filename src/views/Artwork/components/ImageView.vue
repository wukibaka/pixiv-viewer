<template>
  <div class="image-view" :class="{ shrink: isShrink, loaded: artwork.images, censored: isCensored(artwork) }"
    @click="showFull" ref="view">
    <div class="image-box" v-for="(url, index) in artwork.images" :key="index"
      :style="index === 0 ? { width: `${displayWidth}px`, height: `${displayWidth / (artwork.width / artwork.height)}px` } : null">
      <!-- :style="{height: `${(375/artwork.width*artwork.height).toFixed(2)}px`}" -->
      <img v-if="lazy" v-lazy="url.l" :alt="`${artwork.title} - Page ${index + 1}`" class="image"
        @click.stop="view(index, isCensored(artwork))" />
      <img v-else :src="url.l" :alt="`${artwork.title} - Page ${index + 1}`" class="image"
        :style="{ width: displayWidth, height: ((artwork.width / displayWidth) * artwork.height) * (artwork.width / artwork.height) }"
        @click.stop="view(index, isCensored(artwork))" />
      <canvas v-if="artwork.type === 'ugoira'" class="ugoira" :width="artwork.width" :height="artwork.height" id="ugoira"
        ref="ugoira" @click="openDownloadPanel()"></canvas>
    </div>
    <Icon v-if="isShrink" class="dropdown" name="dropdown" scale="4"></Icon>
    <div v-if="artwork.type === 'ugoira'" class="ugoira-controls">
      <div v-if="ugoiraPlaying" class="btn-pause" @click="drawCanvas('pause')">
        <Icon class="pause" name="pause" scale="6"></Icon>
      </div>
      <div v-else class="btn-play" @click="playUgoira()">
        <Icon class="play" name="play" scale="6"></Icon>
      </div>
      <div class="progress-bar" v-if="progressShow" :style="{ width: `${progress * 100}%` }">
        <div class="background"></div>
      </div>
    </div>
  </div>
</template>

<script>
import { mapState, mapGetters } from "vuex";
import { ImagePreview } from "vant";
import axios from "axios";
import JSZip from "jszip";
import GIF from "gif.js";
import Whammy from "whammy";
import FileSaver from "file-saver";
import api from "@/api";
export default {
  watch: {
    artwork(val) {
      if (val.images && val.images.length > 0) {
        this.init();
      }
    }
  },
  props: {
    artwork: {
      type: Object,
      required: true
    },
    lazy: {
      type: Boolean,
      default: true
    }
  },
  computed: {
    original() {
      return this.artwork.images.map(url => url.o);
    },
    ...mapState(["$swiper"]),
    ...mapGetters(["isCensored"])
  },
  data() {
    return {
      displayWidth: 0,
      displayHeight: 0,
      isShrink: false,
      ugoira: null,
      ugoiraPlaying: false,
      curIndex: 0,
      progressShow: false,
      progress: 0
    };
  },
  methods: {
    view(index, censored) {
      if (censored) {
        this.$toast({
          message: "根据当前设置，此内容将不予显示",
          icon: require("@/icons/ban-view.svg")
        });
      } else {
        ImagePreview({
          className: "image-preview",
          images: this.original,
          startPosition: index,
          closeOnPopstate: true,
          closeable: true
        });
      }
    },
    showFull() {
      if (this.isShrink) this.isShrink = false;
    },
    async ugoiraMetadata() {
      let res = await api.ugoiraMetadata(this.artwork.id);
      if (res.status === 0) {
        return Object.freeze(res.data);
      } else {
        this.$toast({
          message: res.msg
        });
      }
    },
    async playUgoira() {
      if (this.progressShow) return;

      if (this.ugoira) {
        this.drawCanvas("play");
        return;
      }

      const ugoira = await this.ugoiraMetadata();
      const frames = {};
      ugoira.frames.forEach(frame => {
        frames[frame.file] = frame;
      });

      this.ugoira = {
        frames,
        zip: ugoira.zip
      };
      // console.log(this.ugoira);
      this.progressShow = true;
      axios
        .get(ugoira.zip, {
          responseType: "blob",
          timeout: 1000 * 30,
          onDownloadProgress: progress => {
            this.progress = progress.loaded / progress.total;
          }
        })
        .then(resp => {
          const jszip = new JSZip();
          jszip.loadAsync(resp.data).then(zip => {
            let index = 0;
            const files = Object.keys(zip.files);
            files.forEach(name => {
              zip
                .file(name)
                .async("blob")
                .then(async blob => {
                  return {
                    blob: blob,
                    bmp: await createImageBitmap(blob)
                  };
                })
                .then(({ blob, bmp }) => {
                  this.ugoira.frames[name].blob = blob;
                  this.ugoira.frames[name].bmp = bmp;

                  if (++index === files.length) {
                    console.info(
                      "Frames loaded:",
                      `frames ${files.length}`,
                      `size ${resp.data.size}`
                    );
                    this.progressShow = false;
                    this.drawCanvas("play");
                  }
                });
            });
          });
        })
        .catch(error => {
          this.resetUgoira();
          this.$toast({
            message: error.message
          });
        });
    },
    drawCanvas(action) {
      const ctx = this.$refs.ugoira[0].getContext("2d");
      // console.log(ctx);
      const { width, height } = this.artwork;

      const frames = Object.values(this.ugoira.frames);

      const draw = () => {
        this.curIndex++;
        setTimeout(
          () => {
            if (!this.ugoira || !this.ugoiraPlaying) return;

            // const imgUri = URL.createObjectURL(frames[this.curIndex - 1].data);
            // const imgData = new Image();
            // imgData.onload = () => {
            ctx.clearRect(0, 0, width, height);
            ctx.drawImage(frames[this.curIndex - 1].bmp, 0, 0, width, height);

            if (this.curIndex >= frames.length) this.curIndex = 0;
            draw();
            // };
            // imgData.src = imgUri;
          },
          this.curIndex === 0 ? 0 : frames[this.curIndex - 1].delay
        );
      };

      if (action === "play") {
        this.ugoiraPlaying = true;
        draw();
      } else if (action === "pause") {
        this.ugoiraPlaying = false;
      }
    },
    downloadZIP() {
      FileSaver.saveAs(
        this.ugoira.zip,
        `[${this.artwork.author.name}] ${this.artwork.title} - ${this.artwork.id}.zip`
      );
    },
    downloadWebM() {
      if (/iPad|iPhone|iPod/.test(navigator.userAgent) && !window.MSStream) {
        this.$toast({
          message: "iOS 设备暂不支持 WebM 格式下载",
          icon: require("@/icons/error.svg")
        });
        return;
      }

      const [width, height] = [this.displayWidth, this.displayHeight];

      const cacheCanvas = document.createElement("canvas");
      cacheCanvas.width = width;
      cacheCanvas.height = height;
      const ctx = cacheCanvas.getContext("2d");

      const encoder = new Whammy.Video();
      Object.values(this.ugoira.frames).forEach(frame => {
        ctx.clearRect(0, 0, width, height);
        ctx.drawImage(frame.bmp, 0, 0, width, height);
        encoder.add(ctx, frame.delay);
      });
      const webm = encoder.compile();
      FileSaver.saveAs(
        webm,
        `[${this.artwork.author.name}] ${this.artwork.title} - ${this.artwork.id}.webm`
      );
    },
    downloadGIF() {
      let images = Object.values(this.ugoira.frames),
        offset = 1;
      if (images.length >= 100) {
        // 抽帧间隔
        offset = 2;
        images = images.filter((frame, idx) => idx % offset === 0); // 抽帧
        // .map(frame => URL.createObjectURL(frame.blob));
      }

      const [width, height] = [this.displayWidth, this.displayHeight];

      const cacheCanvas = document.createElement("canvas");
      cacheCanvas.width = width;
      cacheCanvas.height = height;
      const ctx = cacheCanvas.getContext("2d");

      const gif = new GIF({
        workers: 4,
        quality: 10,
        width,
        height,
        workerScript: "./static/js/gif.worker.js"
      });
      Object.values(images).forEach(frame => {
        ctx.clearRect(0, 0, width, height);
        ctx.drawImage(frame.bmp, 0, 0, width, height);
        gif.addFrame(ctx, { copy: true, delay: frame.delay * offset });
      });
      gif.on("finished", blob => {
        FileSaver.saveAs(
          blob,
          `[${this.artwork.author.name}] ${this.artwork.title} - ${this.artwork.id}.gif`
        );
      });
      gif.render();
    },
    download(type) {
      switch (type) {
        case "ZIP":
          this.downloadZIP();
          break;

        case "GIF":
          this.downloadGIF();
          break;

        case "WebM":
          this.downloadWebM();
          break;

        default:
          break;
      }
    },
    openDownloadPanel() {
      if (this.progressShow) return;

      if (this.ugoira) {
        if (!window.plus) this.$emit("open-download");
      } else {
        this.playUgoira();
      }
    },
    resetUgoira() {
      this.ugoira = null;
      this.ugoiraPlaying = false;
      this.curIndex = 0;
      this.progress = 0;
      this.progressShow = false;
    },
    init() {
      this.resetUgoira();
      this.$nextTick(() => {
        this.displayWidth = document.getElementById('app').getBoundingClientRect().width;
        this.displayHeight =
          this.displayWidth / (this.artwork.width / this.artwork.height);
        setTimeout(() => {
          if (this.artwork.images && this.artwork.images.length >= 3) {
            this.isShrink = true;
          } else {
            this.isShrink = false;
          }
        }, 0);
      });
    }
  },
  mounted() {
    this.init();
  },
  deactivated() {
    this.resetUgoira();
  }
};
</script>

<style lang="stylus" scoped>
.image-view {
  position: relative;
  min-height: 600px;
  // background-color: #fafafa;

  &.censored {
    pointer-events: none;
  }

  &.loaded {
    min-height: unset;
  }

  &.shrink {
    max-height: 1000px;
    overflow: hidden;

    &::after {
      content: '';
      position: absolute;
      top: 0;
      right: 0;
      bottom: 0;
      left: 0;
      background: linear-gradient(to top, rgb(255, 255, 255), rgba(#fff, 0));
    }

    .dropdown {
      position: absolute;
      bottom: 26px;
      left: 50%;
      transform: translateX(-50%);
      z-index: 1;
      color: #fafafa;
      filter: drop-shadow(1px 4px 8px rgba(0, 0, 0, 0.2));
      animation: ani-dropdown 2s ease-in-out infinite;
    }

    @keyframes ani-dropdown {
      0%, 100% {
        transform: translate(-50%, 0);
      }

      50% {
        transform: translate(-50%, 6px);
      }
    }
  }

  .image-box {
    position: relative;
    // background: #fafafa;

    &:nth-of-type(n+2) {
      min-height: 600px;
      // max-height: 1000px;
    }

    .image {
      display: block;
      width: 100%;
      height: 100%;
      // min-height: 600px;
      // max-height: 1000px;
      object-fit: cover;

      &[lazy='loading'] {
        position: absolute;
        top: 50%;
        left: 50%;
        transform: translate(-50%, -50%);
        width: 120px !important;
        height: 120px !important;
        min-height: auto;
      }
    }

    .ugoira {
      position: absolute;
      top: 50%;
      left: 50%;
      transform: translate(-50%, -50%);
      width: auto;
      max-width: 100%;
      height: auto;
      max-height 100%
    }
  }

  .ugoira-controls {
    position: absolute;
    bottom: 0;
    width: 100%;

    .btn-play, .btn-pause {
      position: absolute;
      right: 16px;
      bottom: 16px;
      color: rgba(122, 172, 208, 0.9);
    }

    .progress-bar {
      position: absolute;
      bottom: 0;
      width: 0;
      height: 4px;
      overflow: hidden;
      transition: width 0.1s;

      .background {
        position: absolute;
        bottom: 0;
        width: 100%;
        height: 4px;
        background: linear-gradient(to right, #3fffa2 0%, #1a9be0 100%);
      }
    }
  }
}
</style>
