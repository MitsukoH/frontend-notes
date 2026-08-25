AutoFitText

Vue 3 自動調整文字大小的共用元件，主要用來處理「文字長度不固定，但必須塞進指定容器」的情況。

適合：

多國語系
API 回傳長度不固定的名稱
卡片標題
Dashboard 數字
固定尺寸 UI
主要功能
shrink：文字塞不下時自動縮小
fill：盡量放大文字填滿空間
minFontSize / maxFontSize
group：多個文字共用相同字級
measureTarget：指定實際測量容器
sideGap：預留左右安全距離
支援 px、rem、vw
使用 ResizeObserver 監聽尺寸變化
使用 MutationObserver 監聽文字變化
核心概念
容器尺寸
   +
文字尺寸
   ↓
判斷是否 Overflow
   ↓
自動調整 font-size
使用原則

優先使用正常 CSS：

Grid / Flex
↓
clamp()
↓
Container Query
↓
line-clamp
↓
AutoFitText

AutoFitText 比較適合處理「內容本身不可預測」的情況，不建議拿來取代一般 RWD。

備註

Nuxt / SSR 使用時，DOM 尺寸必須等 mounted 後才能測量，元件卸載時記得清除 Observer。

```
<script lang="ts">
  /**
   * AutoFitText 支援的字級格式。
   *
   * number：
   * 直接視為 px。
   *
   * string：
   * 支援 px / rem / vw。
   *
   * 例如：
   *
   * 12
   * "12px"
   * "0.75rem"
   * "2vw"
   */
  type AutoFitFontSize = number | `${number}px` | `${number}rem` | `${number}vw`

  type AutoFitGroupMember = {
    id: symbol

    /**
     * 這一顆文字自己可以使用的最大 px。
     */
    maxFontSize: number

    /**
     * 原始 CSS font-size，單位 px。
     */
    defaultFontSize: number

    applyFontSize: (fontSize: number) => void
  }

  /**
   * 所有 AutoFitText 共用的群組資料。
   *
   * 相同 group 名稱：
   * 最後統一使用相同 font-size。
   */
  const autoFitGroups = new Map<string, Map<symbol, AutoFitGroupMember>>()

  /**
   * 同一個 group 裡：
   *
   * 每一顆先算：
   *
   * 「我最多可以使用多少 px」
   *
   * 最後整組取最小值。
   */
  const syncAutoFitGroup = (groupName: string) => {
    const group = autoFitGroups.get(groupName)

    if (!group || group.size === 0) {
      return
    }

    const sharedFontSize = Math.min(
      ...Array.from(group.values()).map((member) => member.maxFontSize),
    )

    group.forEach((member) => {
      member.applyFontSize(sharedFontSize)
    })
  }

  /**
   * 新增 / 更新 group 成員。
   */
  const updateAutoFitGroupMember = (groupName: string, member: AutoFitGroupMember) => {
    let group = autoFitGroups.get(groupName)

    if (!group) {
      group = new Map()

      autoFitGroups.set(groupName, group)
    }

    group.set(member.id, member)

    syncAutoFitGroup(groupName)
  }

  /**
   * 從 group 移除成員。
   */
  const removeAutoFitGroupMember = (groupName: string, memberId: symbol) => {
    const group = autoFitGroups.get(groupName)

    if (!group) {
      return
    }

    group.delete(memberId)

    if (group.size === 0) {
      autoFitGroups.delete(groupName)

      return
    }

    /**
     * 少了一個成員，
     * 其他成員可能可以恢復成更大的字。
     */
    syncAutoFitGroup(groupName)
  }
</script>

<script setup lang="ts">
  const props = withDefaults(
    defineProps<{
      /**
       * 最小字級。
       *
       * 支援：
       *
       * 10
       * "10px"
       * "0.625rem"
       * "1vw"
       *
       * number 一律視為 px。
       *
       * shrink：
       * 不傳時預設 12px。
       *
       * fill：
       * 不傳時完全自動。
       *
       * 注意：
       *
       * fill 模式下 minFontSize 是「正常下限」。
       *
       * 如果 minFontSize 本身已經會造成 overflow，
       * AutoFitText 會允許繼續往下縮，
       * 優先確保文字不超出父層。
       */
      minFontSize?: AutoFitFontSize

      /**
       * 最大字級。
       *
       * 支援：
       *
       * number / px / rem / vw
       *
       * fill 不傳：
       * 自動尋找最大安全尺寸。
       *
       * maxFontSize 是硬上限，
       * 不會被 AutoFit 突破。
       */
      maxFontSize?: AutoFitFontSize

      /**
       * 左右額外預留空間。
       *
       * 單位固定 px。
       */
      sideGap?: number

      /**
       * sideGap 倍率。
       */
      sideGapMultiplier?: number

      /**
       * 相同 group：
       * 最後統一使用相同 font-size。
       */
      group?: string

      /**
       * shrink 模式測量目標。
       *
       * self：
       * AutoFitText 本身。
       *
       * parent：
       * 直接父層。
       *
       * fill 會自動使用 parent。
       */
      measureTarget?: 'self' | 'parent'

      /**
       * shrink：
       * 只縮小，不主動放大。
       *
       * fill：
       * 自動依照直接父層可用寬度，
       * 找到最大安全 font-size。
       */
      fitMode?: 'shrink' | 'fill'
    }>(),
    {
      sideGap: 6,
      sideGapMultiplier: 2,
      group: undefined,
      measureTarget: 'self',
      fitMode: 'shrink',
    },
  )

  const containerRef = ref<HTMLElement | null>(null)
  const textRef = ref<HTMLElement | null>(null)

  /**
   * 每一顆 AutoFitText 唯一 ID。
   */
  const memberId = Symbol('auto-fit-text')

  /**
   * 真正計算完成的 font-size。
   *
   * 內部統一使用 px。
   *
   * null：
   * 不產生 inline font-size，
   * 使用原始 CSS。
   */
  const fittedFontSize = ref<number | null>(null)

  /**
   * 原始 CSS font-size。
   *
   * 單位統一是 px。
   */
  const defaultFontSize = ref(0)

  /**
   * shrink 模式舊有預設值。
   */
  const DEFAULT_SHRINK_MIN_FONT_SIZE = 12

  /**
   * fill 自動搜尋最低起點。
   *
   * 即使使用者設定 minFontSize，
   * 如果 min 已經會 overflow，
   * 最低仍允許往這個值搜尋。
   */
  const DEFAULT_FILL_SEARCH_MIN_FONT_SIZE = 1

  /**
   * 自動放大最多嘗試幾輪。
   */
  const MAX_AUTO_GROW_STEPS = 20

  /**
   * Binary Search 次數。
   */
  const BINARY_SEARCH_STEPS = 12

  let resizeObserver: ResizeObserver | null = null
  let mutationObserver: MutationObserver | null = null
  let animationFrameId: number | null = null

  let isFitting = false
  let needsRefit = false

  let registeredGroup: string | null = null
  let pendingGroupFontSize: number | null = null

  /**
   * ==========================================
   * 字級單位轉換
   * ==========================================
   *
   * AutoFitText 內部所有計算統一使用 px。
   *
   * rem / vw 都先轉成實際 px。
   */
  const resolveFontSizeToPx = (value: AutoFitFontSize | undefined): number | undefined => {
    if (value === undefined) {
      return undefined
    }

    /**
     * number：
     *
     * :min-font-size="10"
     *
     * =
     *
     * 10px
     */
    if (typeof value === 'number') {
      return Number.isFinite(value) ? value : undefined
    }

    const normalizedValue = value.trim().toLowerCase()

    const numericValue = Number.parseFloat(normalizedValue)

    if (!Number.isFinite(numericValue)) {
      return undefined
    }

    /**
     * px
     */
    if (normalizedValue.endsWith('px')) {
      return numericValue
    }

    /**
     * rem
     *
     * 以 html font-size 為基準。
     */
    if (normalizedValue.endsWith('rem')) {
      if (typeof window === 'undefined') {
        return undefined
      }

      const rootFontSize = Number.parseFloat(
        window.getComputedStyle(document.documentElement).fontSize,
      )

      if (!Number.isFinite(rootFontSize)) {
        return undefined
      }

      return numericValue * rootFontSize
    }

    /**
     * vw
     *
     * 1vw = viewport width 的 1%。
     */
    if (normalizedValue.endsWith('vw')) {
      if (typeof window === 'undefined') {
        return undefined
      }

      return (window.innerWidth * numericValue) / 100
    }

    return undefined
  }

  /**
   * 取得目前 min 實際換算後的 px。
   */
  const getMinFontSizePx = () => resolveFontSizeToPx(props.minFontSize)

  /**
   * 取得目前 max 實際換算後的 px。
   */
  const getMaxFontSizePx = () => resolveFontSizeToPx(props.maxFontSize)

  /**
   * 最後真正套到文字上的 style。
   *
   * 注意：
   *
   * 這裡不再強制套 minFontSize。
   *
   * 因為 fill 模式如果父層真的太小，
   * 必須允許突破 min 往下縮，
   * 才能避免 overflow。
   *
   * maxFontSize 仍然是硬限制。
   */
  const textStyle = computed(() => {
    if (fittedFontSize.value === null) {
      return {}
    }

    let fontSize = fittedFontSize.value

    const maxFontSize = getMaxFontSizePx()

    /**
     * max 是硬上限。
     */
    if (maxFontSize !== undefined) {
      fontSize = Math.min(fontSize, maxFontSize)
    }

    return {
      fontSize: `${fontSize}px`,
    }
  })

  /**
   * 套用 group 共用 font-size。
   */
  const applyGroupFontSize = (fontSize: number) => {
    if (isFitting) {
      pendingGroupFontSize = fontSize

      return
    }

    if (Math.abs(fontSize - defaultFontSize.value) < 0.1) {
      fittedFontSize.value = null

      return
    }

    fittedFontSize.value = fontSize
  }

  /**
   * 更新自己的 group 資料。
   */
  const updateGroup = (maxFontSize: number, originalFontSize: number) => {
    const groupName = props.group

    if (!groupName) {
      return
    }

    if (registeredGroup && registeredGroup !== groupName) {
      removeAutoFitGroupMember(registeredGroup, memberId)
    }

    registeredGroup = groupName

    updateAutoFitGroupMember(groupName, {
      id: memberId,
      maxFontSize,
      defaultFontSize: originalFontSize,
      applyFontSize: applyGroupFontSize,
    })
  }

  /**
   * 決定要測量哪個 DOM。
   */
  const getMeasureElement = (container: HTMLElement) => {
    /**
     * fill 永遠使用直接 parent。
     */
    if (props.fitMode === 'fill' || props.measureTarget === 'parent') {
      return container.parentElement ?? container
    }

    return container
  }

  /**
   * CSS px 字串轉 number。
   */
  const parseCssPixelValue = (value: string) => {
    const result = Number.parseFloat(value)

    return Number.isFinite(result) ? result : 0
  }

  /**
   * 取得真正可用寬度。
   *
   * 父層 clientWidth
   * - padding-left
   * - padding-right
   * - sideGap
   */
  const getAvailableWidth = (element: HTMLElement) => {
    const style = window.getComputedStyle(element)

    const paddingLeft = parseCssPixelValue(style.paddingLeft)

    const paddingRight = parseCssPixelValue(style.paddingRight)

    return Math.max(
      0,
      element.clientWidth - paddingLeft - paddingRight - props.sideGap * props.sideGapMultiplier,
    )
  }

  /**
   * 清掉 Binary Search
   * 暫時產生的 inline style。
   */
  const clearTestFontSize = (text: HTMLElement) => {
    text.style.fontSize = ''
  }

  /**
   * 測試指定 font-size
   * 是否可以塞進可用寬度。
   *
   * AutoFitText 是單行元件，
   * 所以只檢查寬度。
   */
  const doesFontSizeFit = (text: HTMLElement, fontSize: number, availableWidth: number) => {
    text.style.fontSize = `${fontSize}px`

    return text.scrollWidth <= availableWidth + 0.5
  }

  /**
   * 在一個已知範圍中，
   * 找出最大的可用 font-size。
   *
   * 前提：
   *
   * low 應該是能放得下的。
   * high 則可能放不下。
   */
  const binarySearchFontSize = (
    text: HTMLElement,
    lowValue: number,
    highValue: number,
    availableWidth: number,
  ) => {
    let low = lowValue
    let high = highValue
    let bestFontSize = lowValue

    for (let index = 0; index < BINARY_SEARCH_STEPS; index += 1) {
      const testFontSize = (low + high) / 2

      if (doesFontSizeFit(text, testFontSize, availableWidth)) {
        bestFontSize = testFontSize

        low = testFontSize
      } else {
        high = testFontSize
      }
    }

    return bestFontSize
  }

  /**
   * ==========================================
   * fill
   * ==========================================
   *
   * 自動尋找最大安全字級。
   *
   * minFontSize：
   *
   * 是「正常情況下的最低值」。
   *
   * 如果 min 本身就會 overflow，
   * 則允許繼續往下縮。
   *
   * maxFontSize：
   *
   * 是硬上限。
   */
  const findFillFontSize = (
    text: HTMLElement,
    originalFontSize: number,
    availableWidth: number,
  ) => {
    const resolvedMinFontSize = getMinFontSizePx()

    const resolvedMaxFontSize = getMaxFontSizePx()

    /**
     * 使用者希望的正常下限。
     *
     * 沒傳就從 1px 開始，
     * 等同完全自動。
     */
    let preferredMinFontSize = Math.max(
      DEFAULT_FILL_SEARCH_MIN_FONT_SIZE,
      resolvedMinFontSize ?? DEFAULT_FILL_SEARCH_MIN_FONT_SIZE,
    )

    /**
     * max 是硬上限。
     */
    const hardMaxFontSize =
      resolvedMaxFontSize !== undefined
        ? Math.max(DEFAULT_FILL_SEARCH_MIN_FONT_SIZE, resolvedMaxFontSize)
        : undefined

    /**
     * 如果使用者寫：
     *
     * min = 20
     * max = 15
     *
     * 那 max 應該優先，
     * 正常下限也不能高於硬上限。
     */
    if (hardMaxFontSize !== undefined) {
      preferredMinFontSize = Math.min(preferredMinFontSize, hardMaxFontSize)
    }

    /**
     * ======================================
     * min 本身就放不下
     * ======================================
     *
     * 這就是我們這次修改的重點。
     *
     * 例如：
     *
     * min = 12px
     *
     * 但 breakpoint 讓父層突然縮小，
     * 12px 已經超出。
     *
     * 此時不能：
     *
     * return 12
     *
     * 而是必須繼續往：
     *
     * 11
     * 10
     * 9
     *
     * 搜尋。
     */
    if (!doesFontSizeFit(text, preferredMinFontSize, availableWidth)) {
      const emergencyMin = DEFAULT_FILL_SEARCH_MIN_FONT_SIZE

      /**
       * 如果連 1px 都放不下，
       * 已經沒有合理空間可以再縮。
       */
      if (!doesFontSizeFit(text, emergencyMin, availableWidth)) {
        return emergencyMin
      }

      /**
       * 在：
       *
       * 1px ~ preferred min
       *
       * 之間尋找最大可以塞下的值。
       */
      return binarySearchFontSize(text, emergencyMin, preferredMinFontSize, availableWidth)
    }

    /**
     * ======================================
     * 有 maxFontSize
     * ======================================
     */
    if (hardMaxFontSize !== undefined) {
      /**
       * max 本身可以塞下：
       * 直接使用 max。
       */
      if (doesFontSizeFit(text, hardMaxFontSize, availableWidth)) {
        return hardMaxFontSize
      }

      /**
       * min 可以塞下、
       * max 放不下。
       *
       * 在中間搜尋。
       */
      return binarySearchFontSize(text, preferredMinFontSize, hardMaxFontSize, availableWidth)
    }

    /**
     * ======================================
     * 沒有 maxFontSize
     * ======================================
     *
     * 真正全自動模式。
     */

    let low = preferredMinFontSize

    let high = Math.max(originalFontSize, preferredMinFontSize, 16)

    let bestFontSize = preferredMinFontSize

    /**
     * 目前 high 是否塞得下。
     */
    let highFits = doesFontSizeFit(text, high, availableWidth)

    let growStep = 0

    /**
     * 不斷倍增：
     *
     * 16
     * ↓
     * 32
     * ↓
     * 64
     * ↓
     * 128
     *
     * 找到第一次放不下的位置。
     */
    while (highFits && growStep < MAX_AUTO_GROW_STEPS) {
      bestFontSize = high
      low = high

      high *= 2

      highFits = doesFontSizeFit(text, high, availableWidth)

      growStep += 1
    }

    /**
     * 極端情況保護。
     */
    if (highFits) {
      return high
    }

    /**
     * 現在：
     *
     * low  放得下
     * high 放不下
     *
     * Binary Search。
     */
    bestFontSize = binarySearchFontSize(text, low, high, availableWidth)

    return bestFontSize
  }

  /**
   * ==========================================
   * shrink
   * ==========================================
   *
   * 保留原本只縮不放大的功能。
   */
  const findShrinkFontSize = (
    text: HTMLElement,
    originalFontSize: number,
    availableWidth: number,
  ) => {
    const resolvedMinFontSize = getMinFontSizePx()

    /**
     * shrink 沒有設定 min：
     * 維持原本 12px。
     */
    const configuredMinFontSize = resolvedMinFontSize ?? DEFAULT_SHRINK_MIN_FONT_SIZE

    /**
     * 原始 CSS 如果比 min 還小，
     * 不要反過來放大。
     */
    const minFontSize = Math.min(configuredMinFontSize, originalFontSize)

    /**
     * 原始大小已經放得下。
     */
    if (text.scrollWidth <= availableWidth) {
      return originalFontSize
    }

    let low = minFontSize
    let high = originalFontSize
    let bestFontSize = minFontSize

    for (let index = 0; index < BINARY_SEARCH_STEPS; index += 1) {
      const testFontSize = (low + high) / 2

      text.style.fontSize = `${testFontSize}px`

      if (text.scrollWidth <= availableWidth) {
        bestFontSize = testFontSize

        low = testFontSize
      } else {
        high = testFontSize
      }
    }

    return bestFontSize
  }

  /**
   * 測量完成後：
   *
   * 有 group → 交給 group。
   *
   * 沒 group → 自己使用。
   */
  const applyMeasuredFontSize = (fontSize: number, originalFontSize: number) => {
    /**
     * 保留一位小數。
     */
    const roundedFontSize = Math.floor(fontSize * 10) / 10

    if (props.group) {
      updateGroup(roundedFontSize, originalFontSize)

      return
    }

    /**
     * shrink 結果等於原始 CSS：
     * 不產生 inline style。
     */
    if (props.fitMode === 'shrink' && Math.abs(roundedFontSize - originalFontSize) < 0.1) {
      fittedFontSize.value = null

      return
    }

    fittedFontSize.value = roundedFontSize
  }

  /**
   * 真正執行 AutoFit。
   */
  const fitText = async () => {
    if (isFitting) {
      needsRefit = true
      return
    }

    const container = containerRef.value

    const text = textRef.value

    if (!container || !text) {
      return
    }

    /**
     * 空文字不用 fit。
     */
    if (!text.textContent?.trim()) {
      fittedFontSize.value = null

      if (registeredGroup) {
        removeAutoFitGroupMember(registeredGroup, memberId)

        registeredGroup = null
      }

      return
    }

    isFitting = true
    needsRefit = false

    pendingGroupFontSize = null

    try {
      /**
       * 先回到原始 CSS 字級。
       *
       * 避免上一次 AutoFit 結果
       * 影響這一次計算。
       */
      fittedFontSize.value = null

      await nextTick()

      const measureElement = getMeasureElement(container)

      const availableWidth = getAvailableWidth(measureElement)

      if (availableWidth <= 0) {
        return
      }

      /**
       * getComputedStyle 回來的
       * font-size 一律是 px。
       */
      const originalFontSize = Number.parseFloat(window.getComputedStyle(text).fontSize)

      if (!Number.isFinite(originalFontSize) || originalFontSize <= 0) {
        return
      }

      defaultFontSize.value = originalFontSize

      /**
       * ======================================
       * fill
       * ======================================
       */
      if (props.fitMode === 'fill') {
        const bestFontSize = findFillFontSize(text, originalFontSize, availableWidth)

        clearTestFontSize(text)

        applyMeasuredFontSize(bestFontSize, originalFontSize)

        return
      }

      /**
       * ======================================
       * shrink
       * ======================================
       */
      const bestFontSize = findShrinkFontSize(text, originalFontSize, availableWidth)

      clearTestFontSize(text)

      applyMeasuredFontSize(bestFontSize, originalFontSize)
    } finally {
      /**
       * 不論中間發生什麼，
       * 都清掉測試中的 inline style。
       */
      clearTestFontSize(text)

      isFitting = false

      /**
       * 套用延後的 group font-size。
       */
      if (pendingGroupFontSize !== null) {
        const pendingSize = pendingGroupFontSize

        pendingGroupFontSize = null

        applyGroupFontSize(pendingSize)
      }

      /**
       * fit 過程中如果又發生 resize，
       * 再算一次。
       */
      if (needsRefit) {
        needsRefit = false

        queueFitText()
      }
    }
  }

  /**
   * 合併同一個 frame 內的
   * 多次重新計算。
   */
  const queueFitText = () => {
    if (animationFrameId !== null) {
      cancelAnimationFrame(animationFrameId)
    }

    animationFrameId = requestAnimationFrame(() => {
      animationFrameId = null

      void fitText()
    })
  }

  /**
   * 重新設定 ResizeObserver。
   */
  const refreshResizeObserver = () => {
    const container = containerRef.value

    if (!container) {
      return
    }

    resizeObserver?.disconnect()

    if (!resizeObserver) {
      resizeObserver = new ResizeObserver(() => {
        queueFitText()
      })
    }

    resizeObserver.observe(getMeasureElement(container))
  }

  /**
   * vw 依賴 viewport 寬度。
   *
   * resize 時重新換算 vw / rem
   * 並重新執行 AutoFit。
   */
  const handleWindowResize = () => {
    queueFitText()
  }

  onMounted(() => {
    const container = containerRef.value

    const text = textRef.value

    if (!container || !text) {
      return
    }

    /**
     * 監聽 parent / self 寬度。
     */
    refreshResizeObserver()

    /**
     * 監聽 viewport。
     *
     * vw 需要這個。
     */
    window.addEventListener('resize', handleWindowResize)

    /**
     * slot 文字改變時重新算。
     */
    mutationObserver = new MutationObserver(() => {
      queueFitText()
    })

    mutationObserver.observe(text, {
      childList: true,
      subtree: true,
      characterData: true,
    })

    /**
     * 第一次 render。
     */
    queueFitText()

    /**
     * Web Font 載入完成後再算一次。
     */
    void document.fonts?.ready.then(() => {
      queueFitText()
    })
  })

  /**
   * min / max / gap 改變時重新計算。
   */
  watch(
    () => [props.minFontSize, props.maxFontSize, props.sideGap, props.sideGapMultiplier],
    () => {
      queueFitText()
    },
  )

  /**
   * fitMode / measureTarget 改變：
   *
   * ResizeObserver 監控的 DOM
   * 也可能需要改變。
   */
  watch(
    () => [props.measureTarget, props.fitMode],
    () => {
      refreshResizeObserver()

      queueFitText()
    },
  )

  /**
   * group 改變。
   */
  watch(
    () => props.group,
    (newGroup, oldGroup) => {
      if (oldGroup) {
        removeAutoFitGroupMember(oldGroup, memberId)
      }

      registeredGroup = null
      fittedFontSize.value = null

      queueFitText()
    },
  )

  onBeforeUnmount(() => {
    resizeObserver?.disconnect()

    mutationObserver?.disconnect()

    window.removeEventListener('resize', handleWindowResize)

    if (animationFrameId !== null) {
      cancelAnimationFrame(animationFrameId)
    }

    if (registeredGroup) {
      removeAutoFitGroupMember(registeredGroup, memberId)
    }
  })
</script>

<template>
  <span ref="containerRef" class="ui-auto-fit-text">
    <span ref="textRef" class="ui-auto-fit-text__content" :style="textStyle">
      <slot />
    </span>
  </span>
</template>

<style scoped>
  .ui-auto-fit-text {
    display: block;
    min-width: 0;
    max-width: 100%;
  }

  .ui-auto-fit-text__content {
    display: flex;
    justify-content: center;
    align-items: center;

    /**
     * AutoFitText 是單行文字。
     */
    white-space: nowrap;

    /**
     * 行高永遠跟著 font-size。
     *
     * font-size: 10px
     * → line-height: 10px
     *
     * font-size: 20px
     * → line-height: 20px
     *
     * 避免 AutoFit 縮小文字後，
     * 舊有過大的 line-height
     * 還把文字位置撐開。
     */
    line-height: 1;
  }
</style>

```