<template>
  <div :id="id" class="vue-pdf-embed">
    <div
      v-for="pageNum in pageNums"
      :key="pageNum"
      :id="id && `${id}-${pageNum}`"
    >
      <canvas />

      <div v-if="!disableTextLayer" class="textLayer" />

      <div v-if="!disableAnnotationLayer" class="annotationLayer" />
    </div>
  </div>
</template>

<script>
import * as pdf from 'pdfjs-dist/legacy/build/pdf.js'
import PdfWorker from 'pdfjs-dist/legacy/build/pdf.worker.js'
import { PDFLinkService } from 'pdfjs-dist/legacy/web/pdf_viewer.js'
import {
  addPrintStyles,
  createPrintIframe,
  emptyElement,
  releaseChildCanvases,
} from './util.js'

pdf.GlobalWorkerOptions.workerPort = new PdfWorker()

export default {
  name: 'VuePdfEmbed',
  props: {
    /**
     * Whether the annotation layer should be disabled.
     * @values Boolean
     */
    disableAnnotationLayer: Boolean,
    /**
     * Whether the text layer should be disabled.
     * @values Boolean
     */
    disableTextLayer: Boolean,
    /**
     * Desired page height.
     * @values Number, String
     */
    height: [Number, String],
    /**
     * Component identifier (inherited by page containers with page number
     * postfixes).
     * @values String
     */
    id: String,
    /**
     * Path for annotation icons, including trailing slash.
     * @values String
     */
    imageResourcesPath: String,
    /**
     * Number of the page to display.
     * @values Number
     */
    page: Number,
    /**
     * Desired page rotation angle.
     * @values Number, String
     */
    rotation: {
      type: [Number, String],
      validator(value) {
        if (value % 90 !== 0) {
          throw new Error('Rotation must be 0 or a multiple of 90.')
        }
        return true
      },
    },
    /**
     * Desired ratio of canvas size to document size.
     * @values Number
     */
    scale: Number,
    /**
     * Source of the document to display.
     * @values String, URL, TypedArray
     */
    source: {
      type: [Object, String, Uint8Array],
      required: true,
    },
    /**
     * Desired page width.
     * @values Number, String
     */
    width: [Number, String],
  },
  data() {
    return {
      document: null,
      pageCount: null,
      pageNums: [],
    }
  },
  computed: {
    linkService() {
      if (!this.document || this.disableAnnotationLayer) {
        return null
      }

      const service = new PDFLinkService()
      service.setDocument(this.document)
      service.setViewer({
        scrollPageIntoView: ({ pageNumber }) => {
          this.$emit('internal-link-clicked', pageNumber)
        },
      })
      return service
    },
  },
  created() {
    this.$watch(
      () => [
        this.source,
        this.disableAnnotationLayer,
        this.disableTextLayer,
        this.height,
        this.page,
        this.rotation,
        this.width,
      ],
      async ([newSource], [oldSource]) => {
        if (newSource !== oldSource) {
          releaseChildCanvases(this.$el)
          await this.load()
        }
        this.render()
      }
    )
  },
  async mounted() {
    await this.load()
    this.render()
  },
  beforeDestroy() {
    releaseChildCanvases(this.$el)
    this.document?.destroy()
  },
  beforeUnmount() {
    releaseChildCanvases(this.$el)
    this.document?.destroy()
  },
  methods: {
    /**
     * Returns an array of the actual page width and height based on props and
     * aspect ratio.
     * @param {number} ratio - Page aspect ratio.
     */
    getPageDimensions(ratio) {
      let width, height

      if (this.height && !this.width) {
        height = this.height
        width = height / ratio
      } else {
        width = this.width || this.$el.clientWidth
        height = width * ratio
      }

      return [width, height]
    },
    /**
     * Loads a PDF document. Defines a password callback for protected
     * documents.
     *
     * NOTE: Ignored if source property is not provided.
     */
    async load() {
      if (!this.source) {
        return
      }

      try {
        if (this.source._pdfInfo) {
          this.document = this.source
        } else {
          const documentLoadingTask = pdf.getDocument(this.source)
          documentLoadingTask.onPassword = (callback, reason) => {
            const retry = reason === pdf.PasswordResponses.INCORRECT_PASSWORD
            this.$emit('password-requested', callback, retry)
          }
          this.document = await documentLoadingTask.promise
        }
        this.pageCount = this.document.numPages
        this.$emit('loaded', this.document)
      } catch (e) {
        this.document = null
        this.pageCount = null
        this.pageNums = []
        this.$emit('loading-failed', e)
      }
    },
    /**
     * Prints a PDF document via the browser interface.
     *
     * NOTE: Ignored if the document is not loaded.
     *
     * @param {number} dpi - Print resolution.
     * @param {string} filename - Predefined filename to save.
     * @param {boolean} allPages - Ignore page prop to print all pages.
     */
    async print(dpi = 300, filename = '', allPages = false) {
      if (!this.document) {
        return
      }

      const printUnits = dpi / 72
      const styleUnits = 96 / 72
      let container, iframe, title

      try {
        container = document.createElement('div')
        container.style.display = 'none'
        window.document.body.appendChild(container)
        iframe = await createPrintIframe(container)

        const pageNums =
          this.page && !allPages
            ? [this.page]
            : [...Array(this.document.numPages + 1).keys()].slice(1)

        await Promise.all(
          pageNums.map(async (pageNum, i) => {
            const page = await this.document.getPage(pageNum)
            const viewport = page.getViewport({ scale: 1 })

            if (i === 0) {
              const sizeX = (viewport.width * printUnits) / styleUnits
              const sizeY = (viewport.height * printUnits) / styleUnits
              addPrintStyles(iframe, sizeX, sizeY)
            }

            const canvas = document.createElement('canvas')
            canvas.width = viewport.width * printUnits
            canvas.height = viewport.height * printUnits
            container.appendChild(canvas)
            const canvasClone = canvas.cloneNode()
            iframe.contentWindow.document.body.appendChild(canvasClone)

            await page.render({
              canvasContext: canvas.getContext('2d'),
              intent: 'print',
              transform: [printUnits, 0, 0, printUnits, 0, 0],
              viewport,
            }).promise

            canvasClone.getContext('2d').drawImage(canvas, 0, 0)
          })
        )

        if (filename) {
          title = window.document.title
          window.document.title = filename
        }

        iframe.contentWindow.focus()
        iframe.contentWindow.print()
      } catch (e) {
        this.$emit('printing-failed', e)
      } finally {
        if (title) {
          window.document.title = title
        }

        releaseChildCanvases(container)
        container.parentNode?.removeChild(container)
      }
    },
    /**
     * Renders the PDF document as SVG element(s) and additional layers.
     *
     * NOTE: Ignored if the document is not loaded.
     */
    async render() {
      if (!this.document) {
        return
      }

      try {
        this.pageNums = this.page
          ? [this.page]
          : [...Array(this.document.numPages + 1).keys()].slice(1)

        await Promise.all(
          this.pageNums.map(async (pageNum, i) => {
            const page = await this.document.getPage(pageNum)
            const [canvas, div1, div2] = this.$el.children[i].children
            const [actualWidth, actualHeight] = this.getPageDimensions(
              page.view[3] / page.view[2]
            )

            if ((this.rotation / 90) % 2) {
              canvas.style.width = `${Math.floor(actualHeight)}px`
              canvas.style.height = `${Math.floor(actualWidth)}px`
            } else {
              canvas.style.width = `${Math.floor(actualWidth)}px`
              canvas.style.height = `${Math.floor(actualHeight)}px`
            }

            await this.renderPage(page, canvas, actualWidth)

            if (!this.disableTextLayer) {
              await this.renderPageTextLayer(page, div1, actualWidth)
            }

            if (!this.disableAnnotationLayer) {
              await this.renderPageAnnotationLayer(
                page,
                div2 || div1,
                actualWidth
              )
            }
          })
        )

        this.$emit('rendered')
      } catch (e) {
        this.document = null
        this.pageCount = null
        this.pageNums = []
        this.$emit('rendering-failed', e)
      }
    },
    /**
     * Renders the page content.
     * @param {PDFPageProxy} page - Page proxy.
     * @param {HTMLCanvasElement} canvas - HTML canvas.
     * @param {number} width - Actual page width.
     */
    async renderPage(page, canvas, width) {
      const viewport = page.getViewport({
        scale: this.scale ?? Math.ceil(width / page.view[2]) + 1,
        rotation: this.rotation,
      })

      canvas.width = viewport.width
      canvas.height = viewport.height

      await page.render({
        canvasContext: canvas.getContext('2d'),
        viewport,
      }).promise
    },
    /**
     * Renders the annotation layer for the specified page.
     * @param {PDFPageProxy} page - Page proxy.
     * @param {HTMLElement} container - HTML container.
     * @param {number} width - Actual page width.
     */
    async renderPageAnnotationLayer(page, container, width) {
      emptyElement(container)
      pdf.AnnotationLayer.render({
        annotations: await page.getAnnotations(),
        div: container,
        linkService: this.linkService,
        page,
        renderInteractiveForms: false,
        viewport: page
          .getViewport({
            scale: width / page.view[2],
            rotation: this.rotation,
          })
          .clone({
            dontFlip: true,
          }),
        imageResourcesPath: this.imageResourcesPath,
      })
    },
    /**
     * Renders the text layer for the specified page.
     * @param {PDFPageProxy} page - Page proxy.
     * @param {HTMLElement} container - HTML container.
     * @param {number} width - Actual page width.
     */
    async renderPageTextLayer(page, container, width) {
      emptyElement(container)
      await pdf.renderTextLayer({
        container,
        textContent: await page.getTextContent(),
        viewport: page.getViewport({
          scale: width / page.view[2],
          rotation: this.rotation,
        }),
      }).promise
    },
  },
}
</script>

<style lang="scss">
@import 'styles/text-layer';
@import 'styles/annotation-layer';

.vue-pdf-embed {
  & > div {
    position: relative;
  }

  canvas {
    display: block;
  }
}
</style>
