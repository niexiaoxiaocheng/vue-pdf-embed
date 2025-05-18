<script setup lang="ts">
import VuePdfEmbed from '../src/index'

// 使用本地PDF文件
const pdfSource = '/assets/sample.pdf'

// 高亮数据
const highlights = [
{
      "pageNumber": 15,
      "x": 85.92,
      "y": 611.39026,
      "width": 252.36284,
      "height": 15,
      "text": "经甲、乙双方平等、自愿协商同意，对本合同做以下变更",
      "pageWidth": 595.3,
      "pageHeight": 841.9
    }
]
</script>

<template>
  <div class="pdf-container">
    <VuePdfEmbed 
      :source="pdfSource" 
      :textLayer="true" 
      :allPages="true"
      :scale="1.0"
      :highlights="highlights"
    />
  </div>
</template>

<style lang="scss">
@import 'pdfjs-dist/web/pdf_viewer.css';

body {
  padding: 16px;
  background-color: #ccc;
}

.pdf-container {
  margin: auto;
  max-width: 800px;
  height: 800px;
  overflow-y: auto;
  background: white;
  border-radius: 8px;
  box-shadow: 0 2px 8px 4px rgba(0, 0, 0, 0.1);
  scroll-behavior: smooth;
}

.vue-pdf-embed {
  & > div {
    margin-bottom: 4px;
  }

  canvas {
    image-rendering: -webkit-optimize-contrast;
    image-rendering: crisp-edges;
  }
}
</style>
