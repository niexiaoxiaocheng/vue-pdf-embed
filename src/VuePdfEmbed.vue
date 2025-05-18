<script setup lang="ts">
import { computed, onBeforeUnmount, ref, shallowRef, toRef, watch, onMounted } from 'vue'
import { AnnotationLayer, TextLayer } from 'pdfjs-dist/legacy/build/pdf.mjs'
import { PDFLinkService } from 'pdfjs-dist/web/pdf_viewer.mjs'
import type {
  OnProgressParameters,
  PDFDocumentProxy,
  PDFPageProxy,
  PageViewport,
} from 'pdfjs-dist'

import type { PasswordRequestParams, Source } from './types'
import {
  addPrintStyles,
  createPrintIframe,
  downloadPdf,
  emptyElement,
  releaseChildCanvases,
} from './utils'
import { useVuePdfEmbed } from './composables'

// 定义高亮数据类型
interface Highlight {
  pageNumber: number
  x: number
  y: number
  width: number
  height: number
  text: string
  pageWidth: number
  pageHeight: number
}

const props = withDefaults(
  defineProps<{
    /**
     * Whether to enable an annotation layer.
     */
    annotationLayer?: boolean
    /**
     * Desired page height.
     */
    height?: number
    /**
     * Root element identifier (inherited by page containers with page number
     * postfixes).
     */
    id?: string
    /**
     * Path for annotation icons, including trailing slash.
     */
    imageResourcesPath?: string
    /**
     * Document navigation service.
     */
    linkService?: PDFLinkService
    /**
     * Number of the page to display.
     */
    page?: number
    /**
     * Desired page rotation angle.
     */
    rotation?: number
    /**
     * Desired ratio of canvas size to document size.
     */
    scale?: number
    /**
     * Source of the document to display.
     */
    source: Source
    /**
     * Whether to enable a text layer.
     */
    textLayer?: boolean
    /**
     * Desired page width.
     */
    width?: number
    /**
     * Whether to show all pages.
     */
    allPages?: boolean
    /**
     * Array of highlights to display.
     */
    highlights?: Highlight[]
  }>(),
  {
    rotation: 0,
    scale: 1,
    allPages: false,
    highlights: () => []
  }
)

const emit = defineEmits<{
  (e: 'internal-link-clicked', value: number): void
  (e: 'loaded', value: PDFDocumentProxy): void
  (e: 'loading-failed', value: Error): void
  (e: 'password-requested', value: PasswordRequestParams): void
  (e: 'progress', value: OnProgressParameters): void
  (e: 'rendered'): void
  (e: 'rendering-failed', value: Error): void
}>()

const pageNums = shallowRef<number[]>([])
const pageScales = ref<number[]>([])
const root = shallowRef<HTMLDivElement | null>(null)

let renderingController: { isAborted: boolean; promise: Promise<void> } | null =
  null

const { doc } = useVuePdfEmbed({
  onError: (e) => {
    pageNums.value = []
    emit('loading-failed', e)
  },
  onPasswordRequest({ callback, isWrongPassword }) {
    emit('password-requested', { callback, isWrongPassword })
  },
  onProgress: (progressParams) => {
    emit('progress', progressParams)
  },
  source: toRef(props, 'source'),
})

const linkService = computed(() => {
  if (!doc.value || !props.annotationLayer) {
    return null
  } else if (props.linkService) {
    return props.linkService
  }

  const service = new PDFLinkService()
  service.setDocument(doc.value)
  service.setViewer({
    scrollPageIntoView: ({ pageNumber }: { pageNumber: number }) => {
      emit('internal-link-clicked', pageNumber)
    },
  })
  return service
})

/**
 * Downloads the PDF document.
 * @param filename - Predefined filename to save.
 */
const download = async (filename: string) => {
  if (!doc.value) {
    return
  }

  const data = await doc.value.getData()
  const metadata = await doc.value.getMetadata()
  const suggestedFilename =
    // @ts-expect-error: contentDispositionFilename is not typed
    filename ?? metadata.contentDispositionFilename ?? ''
  downloadPdf(data, suggestedFilename)
}

/**
 * Returns an array of the actual page width and height based on props and
 * aspect ratio.
 * @param ratio - Page aspect ratio.
 */
const getPageDimensions = (ratio: number): [number, number] => {
  let width: number
  let height: number

  if (props.height && !props.width) {
    height = props.height
    width = height / ratio
  } else {
    width = props.width ?? root.value!.clientWidth
    height = width * ratio
  }

  return [width, height]
}

/**
 * Prints a PDF document via the browser interface.
 * @param dpi - Print resolution.
 * @param filename - Predefined filename to save.
 * @param allPages - Whether to ignore the page prop and print all pages.
 */
const print = async (dpi = 300, filename = '', allPages = false) => {
  if (!doc.value) {
    return
  }

  const printUnits = dpi / 72
  const styleUnits = 96 / 72
  let container: HTMLDivElement
  let iframe: HTMLIFrameElement
  let title: string | undefined

  try {
    container = window.document.createElement('div')
    container.style.display = 'none'
    window.document.body.appendChild(container)
    iframe = await createPrintIframe(container)

    const pageNums =
      props.page && !allPages
        ? [props.page]
        : [...Array(doc.value.numPages + 1).keys()].slice(1)

    await Promise.all(
      pageNums.map(async (pageNum, i) => {
        const page = await doc.value!.getPage(pageNum)
        const viewport = page.getViewport({
          scale: 1,
          rotation: 0,
        })

        if (i === 0) {
          const sizeX = (viewport.width * printUnits) / styleUnits
          const sizeY = (viewport.height * printUnits) / styleUnits
          addPrintStyles(iframe, sizeX, sizeY)
        }

        const canvas = window.document.createElement('canvas')
        canvas.width = viewport.width * printUnits
        canvas.height = viewport.height * printUnits
        container.appendChild(canvas)
        const canvasClone = canvas.cloneNode() as HTMLCanvasElement
        iframe.contentWindow!.document.body.appendChild(canvasClone)

        await page.render({
          canvasContext: canvas.getContext('2d')!,
          intent: 'print',
          transform: [printUnits, 0, 0, printUnits, 0, 0],
          viewport,
        }).promise

        canvasClone.getContext('2d')!.drawImage(canvas, 0, 0)
      })
    )

    if (filename) {
      title = window.document.title
      window.document.title = filename
    }

    iframe.contentWindow?.focus()
    iframe.contentWindow?.print()
  } finally {
    if (title) {
      window.document.title = title
    }

    releaseChildCanvases(container!)
    container!.parentNode?.removeChild(container!)
  }
}

/**
 * Renders the PDF document as canvas element(s) and additional layers.
 */
const render = async () => {
  if (!doc.value || renderingController?.isAborted) {
    return
  }

  try {
    pageNums.value = props.allPages
      ? [...Array(doc.value.numPages + 1).keys()].slice(1)
      : props.page
      ? [props.page]
      : [1]
    pageScales.value = Array(pageNums.value.length).fill(1)

    await Promise.all(
      pageNums.value.map(async (pageNum, i) => {
        const page = await doc.value!.getPage(pageNum)
        if (renderingController?.isAborted) {
          return
        }
        const pageRotation =
          ((props.rotation % 90 === 0 ? props.rotation : 0) + page.rotate) % 360
        const [canvas, div1, div2] = Array.from(
          root.value!.getElementsByClassName('vue-pdf-embed__page')[i].children
        ) as [HTMLCanvasElement, HTMLDivElement, HTMLDivElement]
        const isTransposed = !!((pageRotation / 90) % 2)
        const viewWidth = page.view[2] - page.view[0]
        const viewHeight = page.view[3] - page.view[1]
        const [actualWidth, actualHeight] = getPageDimensions(
          isTransposed ? viewWidth / viewHeight : viewHeight / viewWidth
        )
        const cssWidth = `${Math.floor(actualWidth)}px`
        const cssHeight = `${Math.floor(actualHeight)}px`
        const pageWidth = isTransposed ? viewHeight : viewWidth
        const pageScale = actualWidth / pageWidth
        const viewport = page.getViewport({
          scale: pageScale,
          rotation: pageRotation,
        })

        pageScales.value[i] = pageScale

        canvas.style.display = 'block'
        canvas.style.width = cssWidth
        canvas.style.height = cssHeight

        const renderTasks = [
          renderPage(
            page,
            viewport.clone({
              scale: viewport.scale * window.devicePixelRatio * props.scale,
            }),
            canvas
          ),
        ]

        if (props.textLayer) {
          renderTasks.push(
            renderPageTextLayer(
              page,
              viewport.clone({
                dontFlip: true,
              }),
              div1
            )
          )
        }

        if (props.annotationLayer) {
          renderTasks.push(
            renderPageAnnotationLayer(
              page,
              viewport.clone({
                dontFlip: true,
              }),
              div2 || div1
            )
          )
        }

        return Promise.all(renderTasks)
      })
    )

    if (!renderingController?.isAborted) {
      emit('rendered')
    }
  } catch (e) {
    pageNums.value = []
    pageScales.value = []

    if (!renderingController?.isAborted) {
      emit('rendering-failed', e as Error)
    }
  }
}

/**
 * Renders the page content.
 * @param page - Page proxy.
 * @param viewport - Page viewport.
 * @param canvas - HTML canvas.
 */
const renderPage = async (
  page: PDFPageProxy,
  viewport: PageViewport,
  canvas: HTMLCanvasElement
) => {
  canvas.width = viewport.width
  canvas.height = viewport.height
  await page.render({
    canvasContext: canvas.getContext('2d')!,
    viewport,
  }).promise
}

/**
 * Renders the annotation layer for the specified page.
 * @param page - Page proxy.
 * @param viewport - Page viewport.
 * @param container - HTML container.
 */
const renderPageAnnotationLayer = async (
  page: PDFPageProxy,
  viewport: PageViewport,
  container: HTMLDivElement
) => {
  emptyElement(container)
  new AnnotationLayer({
    accessibilityManager: null,
    annotationCanvasMap: null,
    annotationEditorUIManager: null,
    div: container,
    page,
    structTreeLayer: null,
    viewport,
  }).render({
    annotations: await page.getAnnotations(),
    div: container,
    imageResourcesPath: props.imageResourcesPath,
    linkService: linkService.value!,
    page,
    renderForms: false,
    viewport,
  })
}

/**
 * Renders the text layer for the specified page.
 * @param page - Page proxy.
 * @param viewport - Page viewport.
 * @param container - HTML container.
 */
const renderPageTextLayer = async (
  page: PDFPageProxy,
  viewport: PageViewport,
  container: HTMLElement
) => {
  emptyElement(container)
  new TextLayer({
    container,
    textContentSource: await page.getTextContent(),
    viewport,
  }).render()

  // 渲染高亮
  renderHighlights(page.pageNumber, container, viewport)
}

/**
 * Renders highlights for a specific page.
 * @param pageNumber - Page number.
 * @param container - Text layer container.
 * @param viewport - Page viewport.
 */
const renderHighlights = (pageNumber: number, container: HTMLElement, viewport: PageViewport) => {
  // 获取当前页面的高亮
  const pageHighlights = props.highlights.filter(h => h.pageNumber === pageNumber)
  
  pageHighlights.forEach(highlight => {
    const highlightElement = document.createElement('div')
    highlightElement.className = 'pdf-highlight'
    
    // 计算实际显示比例
    const displayScale = viewport.width / highlight.pageWidth
    
    // 计算位置和尺寸
    const x = highlight.x * displayScale
    // 注意：PDF 坐标系原点在左下角，需要转换 Y 坐标
    const y = (highlight.pageHeight - highlight.y - highlight.height) * displayScale
    const width = highlight.width * displayScale
    const height = highlight.height * displayScale
    
    // 设置高亮样式
    highlightElement.style.position = 'absolute'
    highlightElement.style.left = `${x}px`
    highlightElement.style.top = `${y}px`
    highlightElement.style.width = `${width}px`
    highlightElement.style.height = `${height}px`
    highlightElement.style.backgroundColor = 'rgba(255, 255, 0, 0.8)' // 非常深的高亮颜色
    highlightElement.style.pointerEvents = 'none' // 确保不影响文本选择
    
    // 添加调试信息
    console.log('Highlight rendering:', {
      pageNumber,
      original: highlight,
      calculated: { x, y, width, height },
      viewport: {
        width: viewport.width,
        height: viewport.height,
        scale: viewport.scale
      },
      displayScale
    })
    
    container.appendChild(highlightElement)
  })
}

watch(
  doc,
  (newDoc) => {
    if (newDoc) {
      emit('loaded', newDoc)
    }
  },
  { immediate: true }
)

watch(
  () => [
    doc.value,
    props.annotationLayer,
    props.height,
    props.imageResourcesPath,
    props.page,
    props.rotation,
    props.scale,
    props.textLayer,
    props.width,
  ],
  async ([newDoc]) => {
    if (newDoc) {
      if (renderingController) {
        renderingController.isAborted = true
        await renderingController.promise
      }

      releaseChildCanvases(root.value)
      renderingController = {
        isAborted: false,
        promise: render(),
      }

      await renderingController.promise
      renderingController = null
    }
  },
  { immediate: true }
)

onBeforeUnmount(() => {
  releaseChildCanvases(root.value)
})

onMounted(() => {
  if (root.value) {
    setTimeout(() => {
      const pages = root.value?.getElementsByClassName('vue-pdf-embed__page')
      if (pages) {
        // 优先使用高亮数据中的页码
        const targetPage = props.highlights.length > 0 
          ? props.highlights[0].pageNumber 
          : props.page
        
        if (targetPage && pages[targetPage - 1]) {
          pages[targetPage - 1].scrollIntoView({ behavior: 'smooth' })
        }
      }
    }, 500)
  }
})

defineExpose({
  doc,
  download,
  print,
})
</script>

<template>
  <div :id="id" ref="root" class="vue-pdf-embed">
    <div v-for="(pageNum, i) in pageNums" :key="pageNum">
      <slot name="before-page" :page="pageNum" />

      <div
        :id="id && `${id}-${pageNum}`"
        class="vue-pdf-embed__page"
        :style="{
          '--scale-factor': pageScales[i],
          position: 'relative',
        }"
      >
        <canvas />

        <div v-if="textLayer" class="textLayer" />

        <div v-if="annotationLayer" class="annotationLayer" />
      </div>

      <slot name="after-page" :page="pageNum" />
    </div>
  </div>
</template>

<style lang="scss">
.vue-pdf-embed {
  position: relative;

  &__page {
    position: relative;
    margin-bottom: 4px;
  }

  .textLayer {
    position: absolute;
    left: 0;
    top: 0;
    right: 0;
    bottom: 0;
    overflow: hidden;
    opacity: 0.2;
    line-height: 1.0;

    > span {
      color: transparent;
      position: absolute;
      white-space: pre;
      cursor: text;
      transform-origin: 0% 0%;
    }

    .pdf-highlight {
      position: absolute;
      background-color: rgba(255, 255, 0, 0.3);
      pointer-events: none;
      z-index: 1;
    }
  }

  .annotationLayer {
    position: absolute;
    left: 0;
    top: 0;
    right: 0;
    bottom: 0;
    overflow: hidden;
    z-index: 3;
  }
}
</style>
