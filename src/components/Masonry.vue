<template>
  <div :class="wfClass">
    <v-masonry v-if="isMasonry" v-bind="masonryProps">
      <slot></slot>
    </v-masonry>
    <div v-else class="flexbin">
      <slot></slot>
    </div>
  </div>
</template>

<script>
import { LocalStorage } from '@/utils/storage';

export default {
  props: {
    layout: {
      type: String,
      required: false
    }
  },
  data() {
    return {
      wfType: LocalStorage.get('__WF_TYPE', 'Masonry'),
      masonryProps: {
        gutter: "8px",
        cols: {
          300: 1,
          600: 2,
          900: 3,
          1200: 4,
          1600: 5,
          1920: 6,
          2400: 7,
          2700: 8,
          3000: 9,
          default: 6
        }
      }
    }
  },
  computed: {
    isMasonry() {
      if (this.layout == 'Grid') return true
      if (this.layout == 'Justified') return false
      return ['Masonry', 'Grid'].includes(this.wfType)
    },
    wfClass() {
      return {
        'wf-grid': this.layout == 'Grid' || this.wfType == 'Grid'
      }
    }
  }
}
</script>

<style>
.wf-grid .image-card {
  padding-bottom: 100% !important;
}

.flexbin {
  display: flex !important;
  overflow: hidden;
  flex-wrap: wrap;
  margin: 2.5px;
  transition: 0.2s;
}

.flexbin:after {
  content: '';
  flex-grow: 999999999;
  min-width: 15vw;
  height: 0;
}

.flexbin .image-card {
  position: relative;
  display: block !important;
  height: 15vw !important;
  margin: 2.5px;
  padding-bottom: 0 !important;
  flex-grow: 1;
  transition: 0.2s;
}

.flexbin .image[lazy='loading'] {
  top: unset !important;
  left: unset !important;
  transform: scale(0.5) !important;
}

.flexbin .image {
  position: initial !important;
  width: auto !important;
  height: 15vw !important;
  object-fit: cover;
  max-width: 100%;
  min-width: 100%;
  vertical-align: bottom;
  transition: 0.2s;
}

@media (min-width: 768px) {
  .flexbin .meta .title {
    font-size: 23px !important;
  }

  .flexbin .meta .author {
    font-size: 18px !important;
  }
}

@media (max-width: 1280px) {
  .flexbin:after {
    min-width: 20vw !important;
  }

  .flexbin .image-card {
    height: 20vw !important;
  }

  .flexbin .image {
    height: 20vw !important;
  }
}

@media (max-width: 980px) {
  .flexbin:after {
    min-width: 30vw !important;
  }

  .flexbin .image-card {
    height: 30vw !important;
  }

  .flexbin .image {
    height: 30vw !important;
  }
}

@media (max-width: 768px) {
  .flexbin:after {
    min-width: 40vw !important;
  }

  .flexbin .image-card {
    height: 40vw !important;
  }

  .flexbin .image {
    height: 40vw !important;
  }
}

@media (max-width: 570px) {
  .flexbin:after {
    min-width: 60vw !important;
  }

  .flexbin .image-card {
    height: 60vw !important;
  }

  .flexbin .image {
    height: 60vw !important;
  }
}
</style>
