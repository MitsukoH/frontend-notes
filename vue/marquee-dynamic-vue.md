# Vue + JavaScript 動態跑馬燈（Marquee）

如何使用 **Vue + JavaScript** 來控制跑馬燈的滾動效果，而不是只靠 CSS `animation`。

---

## **1️⃣ 使用 JavaScript 控制滾動**

我們可以用 `setInterval` 來讓 `div` 內的文字往左移動：

```vue
<template>
  <div class="marquee-container" ref="marquee">
    <div class="marquee-text" :style="{ transform: `translateX(${position}px)` }">
      <slot />
    </div>
  </div>
</template>

<script>
import { ref, onMounted, onUnmounted } from 'vue'

export default {
  setup() {
    const position = ref(0)
    const marquee = ref(null)
    let interval = null

    const startScrolling = () => {
      interval = setInterval(() => {
        position.value -= 2 // 每次移動 2px
        if (position.value < -marquee.value.scrollWidth) {
          position.value = marquee.value.offsetWidth
        }
      }, 30)
    }

    onMounted(() => {
      startScrolling()
    })

    onUnmounted(() => {
      clearInterval(interval)
    })

    return {
      position,
      marquee
    }
  }
}
</script>

<style scoped>
.marquee-container {
  overflow: hidden;
  white-space: nowrap;
  width: 100%;
}
.marquee-text {
  display: inline-block;
  white-space: nowrap;
}
</style>
```

---

## **2️⃣ 讓使用者可以調整速度**

我們可以增加一個 `speed` 屬性，讓使用者可以 **動態改變滾動速度**：

```vue
<script>
import { ref, watch } from 'vue'

export default {
  props: {
    speed: {
      type: Number,
      default: 2
    }
  },
  setup(props) {
    const position = ref(0)
    let interval = null

    const startScrolling = () => {
      clearInterval(interval)
      interval = setInterval(() => {
        position.value -= props.speed
      }, 30)
    }

    watch(() => props.speed, startScrolling, { immediate: true })

    return { position }
  }
}
</script>
```

這樣使用者可以：

```html
<Marquee :speed="5">跑馬燈內容</Marquee>
```

來調整滾動速度。

---

## **📌 這篇文檔的總結**
✅ **使用 Vue `ref` 來動態控制 `translateX`**  
✅ **使用 `setInterval` 來讓內容不斷滾動**  
✅ **可以讓使用者改變 `speed` 屬性來調整速度**  

這樣我們就能用 Vue + JavaScript **動態控制跑馬燈效果** 了！🚀