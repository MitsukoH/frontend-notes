# Hacky！但有用！語系處理方式

##  問題描述
這是目前專案碰到的問題，目前專案站已經頗具規模，許多修改都顯得笨重，但某些站並無多國語要求，感謝同事提供了一個超Hacky的方式。在我們的 Vue/Nuxt 專案中，語系切換時，有些後端 API 撈取的資料 **不會即時變更**，導致畫面上的翻譯內容 **慢半拍**。**特別是在 JavaScript 內部自訂義的群組名稱（xx Groups）**，這些標題是透過程式碼手動設定的，而非直接來自 API 回傳的資料。這導致語系切換時，這些標題**不會立即更新**，需要強制重新渲染。

##  解法
同事A某用了一個超Hacky的方式來強制讓 Vue 重新渲染：
1. **動態更新 `rootKey` 來強制 Vue 重新掛載**
2. **監聽 `i18n.locale` 變更後，強制改變組件 `key`**
3. **使用 `setTimeout` 確保 Vue 重新計算**

### **主要修改點**

#### 新增 preference
1. 設定 `preference` 作為全域管理
假設專案使用 **Vue** 或 Pinia 來管理狀態
新增 store/preference.js
```
export const state = () => ({
  language: 'en', // 預設語言
  timezone: 'UTC' // 預設時區
})

export const mutations = {
  SET_LANGUAGE(state, lang) {
    state.language = lang
  },
  SET_TIMEZONE(state, tz) {
    state.timezone = tz
  }
}

export const actions = {
  changeLanguage({ commit }, lang) {
    commit('SET_LANGUAGE', lang)
  },
  changeTimezone({ commit }, tz) {
    commit('SET_TIMEZONE', tz)
  }
}
```
這樣就可以在組件中透過Vuex來存取`preference`

2. 在 layout 或 page 組件中使用 preference
在 layout 層或主要頁面裡，可以使用 Vuex 的 state 來生成 rootKey
```
import { computed } from '@nuxtjs/composition-api'
import { useStore } from '@nuxtjs/composition-api'

export default {
  setup() {
    const store = useStore()
    
    // 根據語言與時區變更 rootKey，強制 Vue 重新渲染
    const rootKey = computed(() => {
      return 'root-' + store.state.preference.language + '-' + store.state.preference.timezone
    })

    return {
      rootKey
    }
  }
}

```
在 layout 裡的 最外層 div 加上 :key="rootKey"
```
<template>
  <div :key="rootKey">
    <slot />
  </div>
</template>

import { computed } from '@nuxtjs/composition-api'

const rootKey = computed(() => {
  return 'root-' + preference.state.language.value + preference.state.timezone.value
})

return{
  rootKey,
}
```

這樣每當 language 或 timezone 變更時，整個 layout會被重新掛載，確保api資料重新載入

### 為什麼這樣可以強制vue重新渲染?
Vue 會根據 key 來決定是否要復用或重建一個組件：

當 key 變更時，Vue 會銷毀 (destroy) 原本的組件，然後重新掛載，這會強迫 Vue 重新觸發 mounted() & setup()。
這個技巧常用來解決語系切換、表單重置、或是某些狀態不同步的問題。
簡單來說：
```
<div :key="someKey"> 這裡的內容會被 Vue 重新建立 </div>
```
當 someKey 改變時，Vue 不會重繪 內部 DOM，而是整個 div 直接被重新創建。

### 這樣做有什麼潛在風險？
這個方法雖然有效，但還是有幾點要注意
1. 可能導致效能問題
如果 layout 太大，每次切換語系時，整個 layout 會全部重新渲染，這可能造成
- 所有子組件都會重新初始化，影響效能。
- 某些組件內部的 state 會被清空，導致使用者體驗下降

2. 如果有表單，可能會導致輸入內容丟失
取例來說，如果  layout 裡有個表單：
```
<input v-model="userInput" />

```
當 rootKey 變更時，整個 layout 會被重新載入，導致：
- 輸入中的表單資料會被清空(因為vue會銷毀再重新掛載)

當然這次的方式只是因為網站已具規模所以才使用此方法，更溫和的方式(監聽語系變更後，手動刷新API、使用 watchEffect 等)需要在建置初期就考慮在內。 
