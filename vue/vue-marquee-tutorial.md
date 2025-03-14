# Vue 動態跑馬燈 (Marquee) 教學

如何在 **Vue (Nuxt 2)** 中實現一個 **動態跑馬燈 (Marquee)**，並且會自動偵測 **容器大小變化**，確保 **內容超出才會滾動**，使用 **Anime.js** 來實現動畫效果。

---

## **📌 目標功能**
✅ **文字長度短時不滾動**，超出容器時才開始滾動  
✅ **使用 Anime.js** 來實現流暢動畫  
✅ **支援 Nuxt 2，使用 Composition API**  
✅ **支援 `ResizeObserver`，監聽大小變化**  

---

## **📂 檔案結構**
這是一個 Vue **可重複使用的組件**，你可以將它存為 `Marquee.vue`：

```
📂 components
 ├── 📄 Marquee.vue  # 跑馬燈組件
```

---

## **🚀 完整 `Marquee.vue` 組件**

```vue
<template>
  <div ref="marqueeContainer" class="marquee-container">
    <div ref="marqueeText" class="marquee-text" :class="styleClass">
      <slot />
    </div>
  </div>
</template>

<script>
import { ref, onMounted, onUnmounted } from '@nuxtjs/composition-api'
import anime from 'animejs'

export default {
  props: {
    shouldScroll: {
      type: Boolean,
      default: false,
    },
    styleClass: {
      type: String,
      default: null,
    },
  },
  setup(props) {
    const marqueeContainer = ref(null)
    const marqueeText = ref(null)

    const checkForScroll = () => {
      if (!marqueeContainer.value || !marqueeText.value) return

      const containerWidth = marqueeContainer.value.offsetWidth
      const textWidth = marqueeText.value.offsetWidth

      if (props.shouldScroll && textWidth > containerWidth) {
        anime({
          targets: marqueeText.value,
          translateX: [containerWidth, -textWidth],
          duration: 10000,
          loop: true,
          easing: 'linear',
          delay: 500,
        })
      }
    }

    let resizeObserver = null

    onMounted(() => {
      resizeObserver = new ResizeObserver(checkForScroll)
      resizeObserver.observe(marqueeContainer.value)
      checkForScroll()
    })

    onUnmounted(() => {
      if (resizeObserver) {
        resizeObserver.disconnect()
      }
    })

    return {
      marqueeContainer,
      marqueeText,
    }
  },
}
</script>

<style scoped lang="postcss">
.marquee-container {
  overflow: hidden;
  white-space: nowrap;
}

.marquee-text {
  display: inline-block;
  white-space: nowrap;
  will-change: transform;
}
</style>
```

---

## **🔧 如何使用？**

你可以在 Vue/Nuxt 的 `pages/index.vue` 中這樣使用它：

```vue
<template>
  <div>
    <Marquee :shouldScroll="true" styleClass="custom-style">
      這是一個會自動滾動的文字！🚀🔥
    </Marquee>
  </div>
</template>

<script>
import Marquee from '~/components/Marquee.vue'

export default {
  components: {
    Marquee,
  },
}
</script>

<style scoped>
.custom-style {
  color: red;
  font-size: 20px;
}
</style>
```

---

## **📌 主要技術解析**

### **1️⃣ 使用 Anime.js 來製作動畫**
`Anime.js` 是一個強大的動畫庫，我們在 `checkForScroll()` 裡面用它來控制 `translateX`，讓文字從 **右往左滾動**。

```js
anime({
  targets: marqueeText.value,
  translateX: [containerWidth, -textWidth],
  duration: 10000,
  loop: true,
  easing: 'linear',
  delay: 500,
})
```

✅ **`translateX: [containerWidth, -textWidth]`** 讓文字從右側進入，向左移動直到完全滾出視窗  
✅ **`duration: 10000`** 讓滾動速度固定  
✅ **`loop: true`** 讓動畫無限循環  

---

### **2️⃣ 監聽 `ResizeObserver` 來偵測大小變化**
當畫面縮放或視窗大小變更時，`ResizeObserver` 會觸發 `checkForScroll()`，確保文字寬度發生變化時能 **重新啟動動畫**。

```js
let resizeObserver = null

onMounted(() => {
  resizeObserver = new ResizeObserver(checkForScroll)
  resizeObserver.observe(marqueeContainer.value)
  checkForScroll()
})

onUnmounted(() => {
  if (resizeObserver) {
    resizeObserver.disconnect()
  }
})
```

✅ **畫面改變時，動畫會重新計算，確保滾動效果正常**  
✅ **避免容器變化後動畫仍用舊數據導致視覺錯誤**  

---

