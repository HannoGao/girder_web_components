<script>
import md from 'markdown-it';

export default {
  props: {
    text: {
      default: '',
      type: String,
    },
  },
  computed: {
    markdown() {
      return md({
        linkify: true,
      }).render(this.text);
    },
  },
};
</script>

<template lang="pug">
.g-md-viewer(v-html="markdown")
</template>

<style lang="scss">
.g-md-viewer ul {
  padding-left: 20px;
}
</style>
