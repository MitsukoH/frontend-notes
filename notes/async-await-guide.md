# 非同步處理技術筆記（async / await & Promise）

這是一份用來幫助自己與他人理解 JavaScript 中非同步操作的筆記，從最基本的觀念到實務應用，搭配情境範例與小結語。

---

## 🌕 什麼是非同步（Asynchronous）？

非同步代表「不會馬上完成，等一下才會有結果」，但我們不會停下來等它，先做別的事，等結果回來再處理。

**生活例子：**

> 去飲料店點珍奶 → 你不會站在原地等，而是先找座位或滑手機。等店員叫號時再取餐。

---

## 🧃 Promise 是什麼？

Promise 是一種「承諾」，表示某個動作會在未來某個時間完成，不論成功或失敗。

```js
const drink = new Promise((resolve, reject) => {
  setTimeout(() => {
    resolve("珍奶來了");
  }, 2000);
});
```

- `resolve`：成功時呼叫（傳回資料）
- `reject`：失敗時呼叫（發生錯誤）

使用 `.then()` 和 `.catch()` 處理結果：

```js
drink
  .then((result) => console.log(result))
  .catch((error) => console.error(error));
```

---

## ✨ async / await 是什麼？

- `async`：定義一個能使用 `await` 的函式
- `await`：暫停等這個 Promise 回來，再繼續往下執行

```js
async function orderDrink() {
  try {
    const result = await drink;
    console.log(result);
  } catch (error) {
    console.error(error);
  }
}
```

這讓程式更像「同步邏輯」那樣好閱讀，不會一直 .then().then()。

---

## 🚀 高手觀念：同時發出，再一起等

### 錯誤寫法（每次都等完才做下一件）

```js
async function run() {
  const a = await wait(1000, "A");
  const b = await wait(1000, "B");
  const c = await wait(1000, "C");
  console.log(a, b, c);
}
```

> ⏱️ 總共約 3 秒（每個都等完才跑下一個）

### 正確寫法（同時啟動 → 一起等）

```js
async function run() {
  const p1 = wait(1000, "A");
  const p2 = wait(1000, "B");
  const p3 = wait(1000, "C");

  const [a, b, c] = await Promise.all([p1, p2, p3]);
  console.log(a, b, c);
}
```

> ⏱️ 總共約 1 秒（三個任務同時發出、並行處理）

### 🌟 技巧總結：

> 要快，就**先啟動任務**，再用 `Promise.all()` **同時等待全部結果**。

---

## ❗ await 的使用限制

`await` 只能寫在 `async function` 裡面！

```js
// ❌ 錯誤！
function run() {
  const result = await fetchData()
}

// ✅ 正確
async function run() {
  const result = await fetchData()
}
```

---

## ✅ 總結

- 非同步不是可怕的東西，它只是「我不馬上給你結果」
- Promise 是未來會發生的事，`async/await` 是讓你更好管理這些事
- 想提升效能，要善用：**先啟動，再一起等待** 的技巧
- 不要在非 async 函式中用 `await`，會直接報語法錯誤！
