# Vue + CSS 跑馬燈（Marquee）

在這篇文章，我們將探討如何使用 **純 CSS** 來製作跑馬燈效果，並將其應用在 Vue 專案中。

---

## **1️⃣ `marquee` 標籤（舊方法，已淘汰）**
早期 HTML 提供 `<marquee>` 標籤來做文字滾動：

```html
<marquee>這是跑馬燈！</marquee>
```

不過，這個標籤已經被淘汰（deprecated），我們應該用 **CSS animation** 來實現。

---

## **2️⃣ 使用 CSS `animation`**
可以用 `@keyframes` 來讓文字滾動：

```css
.marquee {
  white-space: nowrap;
  overflow: hidden;
  display: block;
  animation: marquee 10s linear infinite;
}

@keyframes marquee {
  from {
    transform: translateX(100%);
  }
  to {
    transform: translateX(-100%);
  }
}
```

這樣，`div` 裡的內容就會無限循環滾動。

---

## **3️⃣ 在 Vue 裡應用 CSS 跑馬燈**

如果想在 Vue 裡封裝成組件，可以這樣寫：

```vue
<template>
  <div class="marquee">
    <slot />
  </div>
</template>

<style scoped>
.marquee {
  white-space: nowrap;
  overflow: hidden;
  display: block;
  animation: marquee 10s linear infinite;
}

@keyframes marquee {
  from {
    transform: translateX(100%);
  }
  to {
    transform: translateX(-100%);
  }
}
</style>
```

---

## **📌 這篇文檔的總結**
✅ **`marquee` 標籤已被淘汰，不建議使用**  
✅ **CSS `animation` 可以做出純 CSS 的跑馬燈效果**  
✅ **可以封裝成 Vue 組件來重複使用**  

這樣就能在 Vue 項目中 **快速使用跑馬燈效果** 了！🚀