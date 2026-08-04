<script setup lang="ts">
import JsBarcode from 'jsbarcode'
import { computed, onMounted, ref } from 'vue'

type BarcodeFormat =
  | 'CODE128'
  | 'CODE39'
  | 'EAN13'
  | 'EAN8'
  | 'UPC'
  | 'ITF14'
  | 'ITF'
  | 'MSI'
  | 'pharmacode'

type PaperPreset = 'A4' | 'A5' | 'LETTER' | 'LABEL_100_150' | 'LABEL_80_50' | 'CUSTOM'
type Orientation = 'portrait' | 'landscape'
type HorizontalAlign = 'start' | 'center' | 'end'
type VerticalAlign = 'start' | 'center' | 'end'

interface ProductRow {
  id: number
  code: string
  name: string
  price: string
  quantity: number
}

interface LabelItem extends ProductRow {
  instanceId: string
}

interface PaperSize {
  width: number
  height: number
}

interface Notice {
  level: 'error' | 'warning' | 'info'
  text: string
}

const EPSILON = 0.01
const PAPER_SIZES: Record<Exclude<PaperPreset, 'CUSTOM'>, PaperSize> = {
  A4: { width: 210, height: 297 },
  A5: { width: 148, height: 210 },
  LETTER: { width: 215.9, height: 279.4 },
  LABEL_100_150: { width: 100, height: 150 },
  LABEL_80_50: { width: 80, height: 50 },
}

const format = ref<BarcodeFormat>('CODE128')
const rawProducts = ref(
  [
    '6901234567892 | 示例商品 A | ¥19.90 | 4',
    'SKU-2026-0002 | 示例商品 B | ¥39.00 | 3',
    'SKU-2026-0003 | 示例商品 C | ¥8.80 | 5',
  ].join('\n'),
)

const paperPreset = ref<PaperPreset>('A4')
const orientation = ref<Orientation>('portrait')
const customPaperWidthMm = ref(210)
const customPaperHeightMm = ref(297)

const marginTopMm = ref(10)
const marginRightMm = ref(10)
const marginBottomMm = ref(10)
const marginLeftMm = ref(10)

const labelWidthMm = ref(50)
const labelHeightMm = ref(30)
const labelPaddingMm = ref(2)
const barcodeWidthMm = ref(42)
const barcodeHeightMm = ref(16)
const gapXmm = ref(3)
const gapYmm = ref(3)

const autoLayout = ref(true)
const manualColumns = ref(3)
const manualRows = ref(8)
const horizontalAlign = ref<HorizontalAlign>('center')
const verticalAlign = ref<VerticalAlign>('start')

const showBarcodeValue = ref(true)
const showProductName = ref(true)
const showPrice = ref(true)
const showLabelBorder = ref(true)
const previewZoom = ref(70)

const mounted = ref(false)
onMounted(() => {
  mounted.value = true
})

const clampNumber = (value: number, min: number, max = Number.POSITIVE_INFINITY) => {
  if (!Number.isFinite(value)) return min
  return Math.min(max, Math.max(min, value))
}

const products = computed<ProductRow[]>(() => {
  return rawProducts.value
    .split(/\r?\n/)
    .map((line, index) => {
      const [code = '', name = '', price = '', quantityText = '1'] = line
        .split('|')
        .map((part) => part.trim())

      const parsedQuantity = Number.parseInt(quantityText, 10)

      return {
        id: index + 1,
        code,
        name,
        price,
        quantity: Number.isFinite(parsedQuantity) ? clampNumber(parsedQuantity, 1, 9999) : 1,
      }
    })
    .filter((row) => row.code.length > 0)
})

const labels = computed<LabelItem[]>(() => {
  return products.value.flatMap((product) =>
    Array.from({ length: product.quantity }, (_, index) => ({
      ...product,
      instanceId: `${product.id}-${index + 1}`,
    })),
  )
})

const basePaperSize = computed<PaperSize>(() => {
  const preset = paperPreset.value

  if (preset === 'CUSTOM') {
    return {
      width: clampNumber(customPaperWidthMm.value, 10),
      height: clampNumber(customPaperHeightMm.value, 10),
    }
  }

  return PAPER_SIZES[preset]
})

const paperSize = computed<PaperSize>(() => {
  const size = basePaperSize.value
  return orientation.value === 'portrait'
    ? size
    : { width: size.height, height: size.width }
})

const usableWidthMm = computed(() =>
  Math.max(0, paperSize.value.width - marginLeftMm.value - marginRightMm.value),
)
const usableHeightMm = computed(() =>
  Math.max(0, paperSize.value.height - marginTopMm.value - marginBottomMm.value),
)

const maxColumns = computed(() => {
  const label = clampNumber(labelWidthMm.value, 1)
  const gap = clampNumber(gapXmm.value, 0)
  return Math.max(0, Math.floor((usableWidthMm.value + gap + EPSILON) / (label + gap)))
})

const maxRows = computed(() => {
  const label = clampNumber(labelHeightMm.value, 1)
  const gap = clampNumber(gapYmm.value, 0)
  return Math.max(0, Math.floor((usableHeightMm.value + gap + EPSILON) / (label + gap)))
})

const columns = computed(() =>
  autoLayout.value ? maxColumns.value : Math.max(1, Math.floor(manualColumns.value || 1)),
)
const rows = computed(() =>
  autoLayout.value ? maxRows.value : Math.max(1, Math.floor(manualRows.value || 1)),
)

const layoutWidthMm = computed(() =>
  columns.value > 0
    ? columns.value * labelWidthMm.value + Math.max(0, columns.value - 1) * gapXmm.value
    : 0,
)
const layoutHeightMm = computed(() =>
  rows.value > 0
    ? rows.value * labelHeightMm.value + Math.max(0, rows.value - 1) * gapYmm.value
    : 0,
)

const labelsPerPage = computed(() => Math.max(0, columns.value * rows.value))

const pageLabels = computed<LabelItem[][]>(() => {
  if (labelsPerPage.value <= 0) return []

  const pages: LabelItem[][] = []
  for (let index = 0; index < labels.value.length; index += labelsPerPage.value) {
    pages.push(labels.value.slice(index, index + labelsPerPage.value))
  }

  return pages.length > 0 ? pages : [[]]
})

const metadataLineCount = computed(
  () => Number(showProductName.value) + Number(showPrice.value),
)

function createBarcodeSvg(code: string): string | null {
  if (!mounted.value || typeof document === 'undefined') return null

  const svg = document.createElementNS('http://www.w3.org/2000/svg', 'svg')

  try {
    JsBarcode(svg, code, {
      format: format.value,
      width: 2,
      height: 80,
      margin: 0,
      displayValue: showBarcodeValue.value,
      textMargin: 4,
      fontSize: 18,
      lineColor: '#111827',
      background: 'transparent',
    })

    const width = Number.parseFloat(svg.getAttribute('width') ?? '0')
    const height = Number.parseFloat(svg.getAttribute('height') ?? '0')

    if (width > 0 && height > 0) {
      svg.setAttribute('viewBox', `0 0 ${width} ${height}`)
    }

    svg.removeAttribute('width')
    svg.removeAttribute('height')
    svg.setAttribute('preserveAspectRatio', 'none')
    svg.setAttribute('aria-label', `条码 ${code}`)
    svg.setAttribute('role', 'img')

    return svg.outerHTML
  } catch {
    return null
  }
}

const barcodeSvgMap = computed(() => {
  const map = new Map<string, string | null>()
  for (const code of new Set(products.value.map((item) => item.code))) {
    map.set(code, createBarcodeSvg(code))
  }
  return map
})

const invalidCodes = computed(() => {
  if (!mounted.value) return []
  return products.value
    .filter((item) => !barcodeSvgMap.value.get(item.code))
    .map((item) => item.code)
})

const notices = computed<Notice[]>(() => {
  const result: Notice[] = []

  if (products.value.length === 0) {
    result.push({ level: 'error', text: '请至少输入一条有效的商品条码数据。' })
  }

  if (usableWidthMm.value <= 0 || usableHeightMm.value <= 0) {
    result.push({ level: 'error', text: '页边距已经超过纸张尺寸，请减小页边距。' })
  }

  if (labelWidthMm.value <= 0 || labelHeightMm.value <= 0) {
    result.push({ level: 'error', text: '标签宽高必须大于 0 mm。' })
  }

  if (barcodeWidthMm.value <= 0 || barcodeHeightMm.value <= 0) {
    result.push({ level: 'error', text: '条码宽高必须大于 0 mm。' })
  }

  if (barcodeWidthMm.value + labelPaddingMm.value * 2 > labelWidthMm.value + EPSILON) {
    result.push({ level: 'error', text: '条码宽度加标签内边距已经超过标签宽度。' })
  }

  const estimatedMetadataHeight = metadataLineCount.value * 4
  if (
    barcodeHeightMm.value + labelPaddingMm.value * 2 + estimatedMetadataHeight >
    labelHeightMm.value + EPSILON
  ) {
    result.push({
      level: 'warning',
      text: '条码、商品文字和内边距可能超过标签高度，请检查预览或增大标签高度。',
    })
  }

  if (columns.value <= 0 || rows.value <= 0) {
    result.push({ level: 'error', text: '当前纸张可用区域放不下任何标签。' })
  }

  if (layoutWidthMm.value > usableWidthMm.value + EPSILON) {
    result.push({
      level: 'error',
      text: `布局宽度 ${layoutWidthMm.value.toFixed(1)} mm 超过可用宽度 ${usableWidthMm.value.toFixed(1)} mm。`,
    })
  }

  if (layoutHeightMm.value > usableHeightMm.value + EPSILON) {
    result.push({
      level: 'error',
      text: `布局高度 ${layoutHeightMm.value.toFixed(1)} mm 超过可用高度 ${usableHeightMm.value.toFixed(1)} mm。`,
    })
  }

  if (invalidCodes.value.length > 0) {
    result.push({
      level: 'error',
      text: `${format.value} 格式无法编码：${invalidCodes.value.slice(0, 3).join('、')}${invalidCodes.value.length > 3 ? ' 等' : ''}。`,
    })
  }

  if (pageLabels.value.length > 1) {
    result.push({
      level: 'info',
      text: `共 ${labels.value.length} 张标签，将自动分页为 ${pageLabels.value.length} 页。`,
    })
  }

  return result
})

const hasBlockingError = computed(() => notices.value.some((notice) => notice.level === 'error'))

const paperStyle = computed(() => ({
  width: `${paperSize.value.width}mm`,
  height: `${paperSize.value.height}mm`,
  padding: `${marginTopMm.value}mm ${marginRightMm.value}mm ${marginBottomMm.value}mm ${marginLeftMm.value}mm`,
}))

const gridStyle = computed(() => ({
  width: `${usableWidthMm.value}mm`,
  height: `${usableHeightMm.value}mm`,
  gridTemplateColumns: `repeat(${Math.max(1, columns.value)}, ${labelWidthMm.value}mm)`,
  gridAutoRows: `${labelHeightMm.value}mm`,
  columnGap: `${gapXmm.value}mm`,
  rowGap: `${gapYmm.value}mm`,
  justifyContent: horizontalAlign.value,
  alignContent: verticalAlign.value,
}))

const labelStyle = computed(() => ({
  width: `${labelWidthMm.value}mm`,
  height: `${labelHeightMm.value}mm`,
  padding: `${labelPaddingMm.value}mm`,
}))

const barcodeStyle = computed(() => ({
  width: `${barcodeWidthMm.value}mm`,
  height: `${barcodeHeightMm.value}mm`,
}))

const formatHint = computed(() => {
  const hints: Record<BarcodeFormat, string> = {
    CODE128: '支持数字、英文和常见符号，适合内部商品编码。',
    CODE39: '支持大写英文、数字和少量符号。',
    EAN13: '常用于零售商品，通常为 12 或 13 位数字。',
    EAN8: '短商品码，通常为 7 或 8 位数字。',
    UPC: '常用于北美零售商品，通常为 11 或 12 位数字。',
    ITF14: '常用于物流包装，通常为 13 或 14 位数字。',
    ITF: '仅支持偶数位数字。',
    MSI: '仅支持数字。',
    pharmacode: '制药行业条码，仅支持规定范围内的整数。',
  }
  return hints[format.value]
})

function setPrintPageSize() {
  const styleId = 'barcode-runtime-page-size'
  let style = document.getElementById(styleId) as HTMLStyleElement | null

  if (!style) {
    style = document.createElement('style')
    style.id = styleId
    document.head.appendChild(style)
  }

  style.textContent = `@page { size: ${paperSize.value.width}mm ${paperSize.value.height}mm; margin: 0; }`
}

function printLabels() {
  if (hasBlockingError.value) return
  setPrintPageSize()
  window.requestAnimationFrame(() => window.print())
}

function resetLayout() {
  paperPreset.value = 'A4'
  orientation.value = 'portrait'
  marginTopMm.value = 10
  marginRightMm.value = 10
  marginBottomMm.value = 10
  marginLeftMm.value = 10
  labelWidthMm.value = 50
  labelHeightMm.value = 30
  labelPaddingMm.value = 2
  barcodeWidthMm.value = 42
  barcodeHeightMm.value = 16
  gapXmm.value = 3
  gapYmm.value = 3
  autoLayout.value = true
  manualColumns.value = 3
  manualRows.value = 8
}
</script>

<template>
  <div class="barcode-app">
    <aside class="barcode-controls">
      <div class="brand-block">
        <span class="brand-kicker">PRINT STUDIO</span>
        <h1>商品条码打印</h1>
        <p>毫米级尺寸、纸张布局和越界检测，打印时保持 100% 实际大小。</p>
      </div>

      <section class="control-card">
        <div class="section-heading">
          <span class="step-badge">01</span>
          <div>
            <h2>商品数据</h2>
            <p>每行：条码 | 商品名 | 价格 | 数量</p>
          </div>
        </div>

        <label class="field field--full">
          <span>条码格式</span>
          <select v-model="format">
            <option value="CODE128">CODE 128</option>
            <option value="CODE39">CODE 39</option>
            <option value="EAN13">EAN-13</option>
            <option value="EAN8">EAN-8</option>
            <option value="UPC">UPC</option>
            <option value="ITF14">ITF-14</option>
            <option value="ITF">ITF</option>
            <option value="MSI">MSI</option>
            <option value="pharmacode">Pharmacode</option>
          </select>
          <small>{{ formatHint }}</small>
        </label>

        <label class="field field--full">
          <span>批量商品</span>
          <textarea v-model="rawProducts" rows="7" spellcheck="false" />
        </label>
      </section>

      <section class="control-card">
        <div class="section-heading">
          <span class="step-badge">02</span>
          <div>
            <h2>纸张设置</h2>
            <p>所有数值均使用毫米</p>
          </div>
        </div>

        <div class="field-grid">
          <label class="field">
            <span>纸张</span>
            <select v-model="paperPreset">
              <option value="A4">A4 · 210 × 297</option>
              <option value="A5">A5 · 148 × 210</option>
              <option value="LETTER">Letter · 215.9 × 279.4</option>
              <option value="LABEL_100_150">标签纸 · 100 × 150</option>
              <option value="LABEL_80_50">标签纸 · 80 × 50</option>
              <option value="CUSTOM">自定义</option>
            </select>
          </label>

          <label class="field">
            <span>方向</span>
            <select v-model="orientation">
              <option value="portrait">纵向</option>
              <option value="landscape">横向</option>
            </select>
          </label>

          <template v-if="paperPreset === 'CUSTOM'">
            <label class="field">
              <span>纸张宽度</span>
              <div class="unit-input"><input v-model.number="customPaperWidthMm" type="number" min="10" step="0.1" /><b>mm</b></div>
            </label>
            <label class="field">
              <span>纸张高度</span>
              <div class="unit-input"><input v-model.number="customPaperHeightMm" type="number" min="10" step="0.1" /><b>mm</b></div>
            </label>
          </template>
        </div>

        <h3 class="subheading">页边距</h3>
        <div class="field-grid field-grid--four">
          <label class="field"><span>上</span><div class="unit-input"><input v-model.number="marginTopMm" type="number" min="0" step="0.5" /><b>mm</b></div></label>
          <label class="field"><span>右</span><div class="unit-input"><input v-model.number="marginRightMm" type="number" min="0" step="0.5" /><b>mm</b></div></label>
          <label class="field"><span>下</span><div class="unit-input"><input v-model.number="marginBottomMm" type="number" min="0" step="0.5" /><b>mm</b></div></label>
          <label class="field"><span>左</span><div class="unit-input"><input v-model.number="marginLeftMm" type="number" min="0" step="0.5" /><b>mm</b></div></label>
        </div>
      </section>

      <section class="control-card">
        <div class="section-heading">
          <span class="step-badge">03</span>
          <div>
            <h2>标签与条码尺寸</h2>
            <p>条码 SVG 将按设定毫米尺寸输出</p>
          </div>
        </div>

        <div class="field-grid">
          <label class="field"><span>标签宽度</span><div class="unit-input"><input v-model.number="labelWidthMm" type="number" min="1" step="0.5" /><b>mm</b></div></label>
          <label class="field"><span>标签高度</span><div class="unit-input"><input v-model.number="labelHeightMm" type="number" min="1" step="0.5" /><b>mm</b></div></label>
          <label class="field"><span>条码宽度</span><div class="unit-input"><input v-model.number="barcodeWidthMm" type="number" min="1" step="0.5" /><b>mm</b></div></label>
          <label class="field"><span>条码高度</span><div class="unit-input"><input v-model.number="barcodeHeightMm" type="number" min="1" step="0.5" /><b>mm</b></div></label>
          <label class="field"><span>标签内边距</span><div class="unit-input"><input v-model.number="labelPaddingMm" type="number" min="0" step="0.5" /><b>mm</b></div></label>
        </div>

        <div class="toggle-grid">
          <label class="toggle"><input v-model="showBarcodeValue" type="checkbox" /><span />显示条码数字</label>
          <label class="toggle"><input v-model="showProductName" type="checkbox" /><span />显示商品名</label>
          <label class="toggle"><input v-model="showPrice" type="checkbox" /><span />显示价格</label>
          <label class="toggle"><input v-model="showLabelBorder" type="checkbox" /><span />打印标签边框</label>
        </div>
      </section>

      <section class="control-card">
        <div class="section-heading">
          <span class="step-badge">04</span>
          <div>
            <h2>布局</h2>
            <p>自动计算或手动指定行列数</p>
          </div>
        </div>

        <label class="toggle toggle--standalone">
          <input v-model="autoLayout" type="checkbox" />
          <span />自动使用最大可容纳行列数
        </label>

        <div class="field-grid">
          <label class="field"><span>横向间距</span><div class="unit-input"><input v-model.number="gapXmm" type="number" min="0" step="0.5" /><b>mm</b></div></label>
          <label class="field"><span>纵向间距</span><div class="unit-input"><input v-model.number="gapYmm" type="number" min="0" step="0.5" /><b>mm</b></div></label>
          <label class="field"><span>列数</span><input v-model.number="manualColumns" type="number" min="1" step="1" :disabled="autoLayout" /></label>
          <label class="field"><span>行数</span><input v-model.number="manualRows" type="number" min="1" step="1" :disabled="autoLayout" /></label>
          <label class="field">
            <span>水平对齐</span>
            <select v-model="horizontalAlign">
              <option value="start">左对齐</option>
              <option value="center">居中</option>
              <option value="end">右对齐</option>
            </select>
          </label>
          <label class="field">
            <span>垂直对齐</span>
            <select v-model="verticalAlign">
              <option value="start">顶部</option>
              <option value="center">居中</option>
              <option value="end">底部</option>
            </select>
          </label>
        </div>
      </section>

      <div class="action-bar">
        <button class="button button--ghost" type="button" @click="resetLayout">恢复默认</button>
        <button class="button button--primary" type="button" :disabled="hasBlockingError" @click="printLabels">
          打印 {{ labels.length }} 张标签
        </button>
      </div>
    </aside>

    <main class="preview-panel">
      <header class="preview-toolbar">
        <div>
          <span class="eyebrow">LIVE PREVIEW</span>
          <strong>{{ paperSize.width.toFixed(1) }} × {{ paperSize.height.toFixed(1) }} mm</strong>
        </div>

        <div class="preview-stats">
          <div><span>布局</span><b>{{ columns }} × {{ rows }}</b></div>
          <div><span>单页</span><b>{{ labelsPerPage }}</b></div>
          <div><span>页数</span><b>{{ pageLabels.length }}</b></div>
        </div>

        <label class="zoom-control">
          <span>预览 {{ previewZoom }}%</span>
          <input v-model.number="previewZoom" type="range" min="30" max="120" step="5" />
        </label>
      </header>

      <div class="notice-stack" aria-live="polite">
        <div v-if="notices.length === 0" class="notice notice--success">
          <b>布局检查通过</b>
          <span>当前设置未检测到尺寸越界。</span>
        </div>
        <div v-for="notice in notices" :key="notice.text" class="notice" :class="`notice--${notice.level}`">
          <b>{{ notice.level === 'error' ? '需要处理' : notice.level === 'warning' ? '请注意' : '分页信息' }}</b>
          <span>{{ notice.text }}</span>
        </div>
      </div>

      <div class="print-note">
        <b>实际尺寸打印：</b>
        打印对话框中选择“实际大小 / 100%”，关闭“适合页面”，并关闭页眉页脚。打印机驱动仍可能受不可打印边缘影响。
      </div>

      <div class="paper-viewport">
        <div class="paper-stack" :style="{ zoom: `${previewZoom}%` }">
          <section
            v-for="(page, pageIndex) in pageLabels"
            :key="pageIndex"
            class="paper-page"
            :style="paperStyle"
          >
            <div class="label-grid" :style="gridStyle">
              <article
                v-for="label in page"
                :key="label.instanceId"
                class="label-cell"
                :class="{ 'label-cell--border': showLabelBorder }"
                :style="labelStyle"
              >
                <div class="barcode-svg" :style="barcodeStyle">
                  <div v-if="barcodeSvgMap.get(label.code)" v-html="barcodeSvgMap.get(label.code)" />
                  <div v-else class="invalid-barcode">格式不匹配<br />{{ label.code }}</div>
                </div>
                <div v-if="showProductName && label.name" class="product-name">{{ label.name }}</div>
                <div v-if="showPrice && label.price" class="product-price">{{ label.price }}</div>
              </article>
            </div>
            <span class="page-number">{{ pageIndex + 1 }} / {{ pageLabels.length }}</span>
          </section>
        </div>
      </div>
    </main>
  </div>
</template>

<style>
:root {
  font-family:
    Inter, ui-sans-serif, system-ui, -apple-system, BlinkMacSystemFont, "Segoe UI", "Microsoft YaHei",
    sans-serif;
  color: #18232f;
  background: #eaf0f4;
  font-synthesis: none;
  text-rendering: optimizeLegibility;
}

* {
  box-sizing: border-box;
}

html,
body,
#app {
  min-width: 1180px;
  min-height: 100%;
  margin: 0;
}

button,
input,
select,
textarea {
  font: inherit;
}

.barcode-app {
  min-height: 100vh;
  display: grid;
  grid-template-columns: 460px minmax(720px, 1fr);
  background: #eaf0f4;
}

.barcode-controls {
  position: relative;
  z-index: 2;
  min-height: 100vh;
  padding: 30px 24px 38px;
  overflow-y: auto;
  background: #f8fafb;
  border-right: 1px solid #d8e0e6;
  box-shadow: 12px 0 36px rgba(42, 58, 70, 0.08);
}

.brand-block {
  padding: 0 6px 22px;
}

.brand-kicker,
.eyebrow {
  display: block;
  margin-bottom: 8px;
  color: #d9663d;
  font-size: 11px;
  font-weight: 800;
  letter-spacing: 0.18em;
}

.brand-block h1 {
  margin: 0;
  font-family: Georgia, "Times New Roman", serif;
  font-size: 34px;
  font-weight: 600;
  letter-spacing: -0.04em;
}

.brand-block p {
  margin: 10px 0 0;
  color: #657380;
  font-size: 13px;
  line-height: 1.7;
}

.control-card {
  margin-bottom: 14px;
  padding: 18px;
  background: #fff;
  border: 1px solid #dce4e9;
  border-radius: 14px;
  box-shadow: 0 5px 18px rgba(44, 61, 73, 0.045);
}

.section-heading {
  display: flex;
  gap: 12px;
  align-items: flex-start;
  margin-bottom: 16px;
}

.section-heading h2 {
  margin: 0;
  font-size: 15px;
}

.section-heading p {
  margin: 4px 0 0;
  color: #8a959e;
  font-size: 11px;
}

.step-badge {
  width: 30px;
  height: 30px;
  display: grid;
  flex: 0 0 auto;
  place-items: center;
  color: #fff;
  background: #1f5b68;
  border-radius: 9px;
  font-size: 11px;
  font-weight: 800;
}

.subheading {
  margin: 17px 0 10px;
  color: #53616c;
  font-size: 12px;
}

.field-grid {
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));
  gap: 12px;
}

.field-grid--four {
  grid-template-columns: repeat(4, minmax(0, 1fr));
  gap: 8px;
}

.field {
  display: flex;
  min-width: 0;
  flex-direction: column;
  gap: 7px;
  color: #53616c;
  font-size: 12px;
  font-weight: 650;
}

.field--full {
  margin-top: 12px;
}

.field input,
.field select,
.field textarea {
  width: 100%;
  border: 1px solid #ccd6dc;
  border-radius: 9px;
  outline: none;
  color: #19252f;
  background: #fbfcfd;
  transition: 0.16s ease;
}

.field input,
.field select {
  height: 38px;
  padding: 0 11px;
}

.field textarea {
  min-height: 126px;
  padding: 10px 11px;
  resize: vertical;
  line-height: 1.55;
  font-family: "SFMono-Regular", Consolas, monospace;
  font-size: 11px;
}

.field input:focus,
.field select:focus,
.field textarea:focus {
  border-color: #287485;
  box-shadow: 0 0 0 3px rgba(40, 116, 133, 0.11);
}

.field input:disabled {
  color: #9ca6ad;
  background: #eef2f4;
}

.field small {
  color: #8b969e;
  font-size: 10px;
  font-weight: 450;
  line-height: 1.55;
}

.unit-input {
  position: relative;
}

.unit-input input {
  padding-right: 36px;
}

.unit-input b {
  position: absolute;
  top: 50%;
  right: 10px;
  color: #99a3aa;
  font-size: 10px;
  transform: translateY(-50%);
  pointer-events: none;
}

.toggle-grid {
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));
  gap: 10px 14px;
  margin-top: 16px;
}

.toggle {
  display: flex;
  align-items: center;
  gap: 8px;
  color: #53616c;
  font-size: 11px;
  font-weight: 600;
  cursor: pointer;
}

.toggle--standalone {
  margin-bottom: 14px;
}

.toggle input {
  position: absolute;
  opacity: 0;
  pointer-events: none;
}

.toggle > span {
  position: relative;
  width: 32px;
  height: 18px;
  flex: 0 0 auto;
  border-radius: 99px;
  background: #cbd4da;
  transition: 0.2s ease;
}

.toggle > span::after {
  position: absolute;
  top: 3px;
  left: 3px;
  width: 12px;
  height: 12px;
  content: "";
  border-radius: 50%;
  background: #fff;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.2);
  transition: 0.2s ease;
}

.toggle input:checked + span {
  background: #1f6674;
}

.toggle input:checked + span::after {
  transform: translateX(14px);
}

.action-bar {
  position: sticky;
  bottom: 0;
  display: grid;
  grid-template-columns: 0.75fr 1.5fr;
  gap: 10px;
  padding: 14px 0 0;
  background: linear-gradient(to top, #f8fafb 74%, rgba(248, 250, 251, 0));
}

.button {
  height: 44px;
  border: 0;
  border-radius: 10px;
  font-size: 13px;
  font-weight: 760;
  cursor: pointer;
}

.button--ghost {
  color: #46545e;
  background: #e6ecef;
}

.button--primary {
  color: #fff;
  background: #d9663d;
  box-shadow: 0 8px 20px rgba(217, 102, 61, 0.22);
}

.button--primary:disabled {
  cursor: not-allowed;
  opacity: 0.42;
  box-shadow: none;
}

.preview-panel {
  min-width: 0;
  padding: 30px;
  overflow: hidden;
}

.preview-toolbar {
  min-height: 76px;
  display: grid;
  grid-template-columns: 1fr auto auto;
  gap: 28px;
  align-items: center;
  margin-bottom: 18px;
  padding: 14px 18px;
  background: rgba(255, 255, 255, 0.78);
  border: 1px solid rgba(202, 214, 221, 0.9);
  border-radius: 14px;
  backdrop-filter: blur(12px);
}

.preview-toolbar strong {
  display: block;
  font-family: Georgia, "Times New Roman", serif;
  font-size: 20px;
}

.preview-stats {
  display: flex;
  gap: 24px;
}

.preview-stats div {
  display: grid;
  gap: 3px;
  text-align: center;
}

.preview-stats span,
.zoom-control span {
  color: #84919a;
  font-size: 10px;
}

.preview-stats b {
  font-size: 14px;
}

.zoom-control {
  display: grid;
  width: 150px;
  gap: 7px;
}

.zoom-control input {
  accent-color: #1f6674;
}

.notice-stack {
  display: grid;
  gap: 8px;
  margin-bottom: 12px;
}

.notice {
  display: grid;
  grid-template-columns: 82px 1fr;
  gap: 12px;
  padding: 10px 13px;
  border: 1px solid;
  border-radius: 10px;
  font-size: 11px;
  line-height: 1.5;
}

.notice--success {
  color: #20604f;
  background: #edf8f3;
  border-color: #c6e8d9;
}

.notice--error {
  color: #8f2e2e;
  background: #fff1f1;
  border-color: #f0caca;
}

.notice--warning {
  color: #855d1d;
  background: #fff7e5;
  border-color: #f1deb3;
}

.notice--info {
  color: #315f7a;
  background: #eef7fc;
  border-color: #c9e2f0;
}

.print-note {
  margin-bottom: 14px;
  padding: 10px 13px;
  color: #596670;
  background: #dfe8ed;
  border-radius: 9px;
  font-size: 11px;
  line-height: 1.6;
}

.paper-viewport {
  height: calc(100vh - 265px);
  min-height: 560px;
  padding: 36px;
  overflow: auto;
  background-color: #ccd6db;
  background-image: linear-gradient(rgba(255, 255, 255, 0.34) 1px, transparent 1px),
    linear-gradient(90deg, rgba(255, 255, 255, 0.34) 1px, transparent 1px);
  background-size: 20px 20px;
  border: 1px solid #becbd2;
  border-radius: 16px;
  box-shadow: inset 0 1px 12px rgba(44, 58, 67, 0.1);
}

.paper-stack {
  width: fit-content;
  display: grid;
  gap: 30px;
  margin: 0 auto;
  transform-origin: top center;
}

.paper-page {
  position: relative;
  flex: 0 0 auto;
  overflow: hidden;
  color: #111827;
  background: #fff;
  box-shadow: 0 16px 42px rgba(27, 41, 50, 0.22);
}

.label-grid {
  display: grid;
  overflow: hidden;
}

.label-cell {
  display: flex;
  overflow: hidden;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  color: #111827;
  background: #fff;
}

.label-cell--border {
  outline: 0.2mm solid #c5cbd0;
  outline-offset: -0.2mm;
}

.barcode-svg {
  flex: 0 0 auto;
}

.barcode-svg > div,
.barcode-svg svg {
  width: 100%;
  height: 100%;
  display: block;
}

.invalid-barcode {
  width: 100%;
  height: 100%;
  display: grid;
  place-items: center;
  padding: 1mm;
  color: #9e2d2d;
  background: #fff0f0;
  border: 0.3mm dashed #d69494;
  font-size: 2.5mm;
  text-align: center;
}

.product-name {
  width: 100%;
  margin-top: 0.8mm;
  overflow: hidden;
  font-size: 3mm;
  font-weight: 650;
  line-height: 3.6mm;
  text-align: center;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.product-price {
  width: 100%;
  overflow: hidden;
  font-size: 3.2mm;
  font-weight: 800;
  line-height: 3.8mm;
  text-align: center;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.page-number {
  position: absolute;
  right: 3mm;
  bottom: 2mm;
  color: #a2aab0;
  font-size: 2.5mm;
}

@media print {
  html,
  body,
  #app {
    width: auto;
    min-width: 0;
    height: auto;
    min-height: 0;
    margin: 0 !important;
    padding: 0 !important;
    background: #fff !important;
  }

  .barcode-app {
    display: block;
    min-height: 0;
    background: #fff;
  }

  .barcode-controls,
  .preview-toolbar,
  .notice-stack,
  .print-note {
    display: none !important;
  }

  .preview-panel {
    padding: 0;
    overflow: visible;
  }

  .paper-viewport {
    height: auto;
    min-height: 0;
    padding: 0;
    overflow: visible;
    background: #fff;
    border: 0;
    border-radius: 0;
    box-shadow: none;
  }

  .paper-stack {
    width: auto;
    display: block;
    margin: 0;
    zoom: 100% !important;
  }

  .paper-page {
    margin: 0;
    break-after: page;
    page-break-after: always;
    box-shadow: none;
    print-color-adjust: exact;
    -webkit-print-color-adjust: exact;
  }

  .paper-page:last-child {
    break-after: auto;
    page-break-after: auto;
  }

  .page-number {
    display: none;
  }
}
</style>
