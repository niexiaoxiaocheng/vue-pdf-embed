<script setup lang="ts">
import VuePdfEmbed from '../src/index'

// 使用本地PDF文件
const pdfSource = '/assets/sample.pdf'
</script>

<template>
  <VuePdfEmbed :source="pdfSource" :textLayer="true" :page="2" />
</template>

<style lang="scss">
@import 'pdfjs-dist/web/pdf_viewer.css';

body {
  padding: 16px;
  background-color: #ccc;
}

.vue-pdf-embed {
  margin: auto;
  max-width: 480px;

  & > div {
    margin-bottom: 4px;
    box-shadow: 0 2px 8px 4px rgba(0, 0, 0, 0.1);
  }
}
</style>
