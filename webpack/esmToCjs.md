## esm to cjs

虽然现代主流浏览器都已支持`esm`,但`webpack`仍然会将 ESM 转化为 CommonJS,并注入到运行时代码。

在 esm 中有两种导入导出的方式,但是在 cjs 中只有一种。
因此在转化时，将 `default import/export` 转化为 `module.exports.default`，而 `named import/export` 转化为 `module.exports` 对应的属性。
例如:

- 导出

  ```js
  const sum = (x, y) => x + y;
  // esm
  export default sum;
  export const a = "a";
  // cjs
  module.exports.default = sum;
  module.exports.a = "a";
  ```

- 导入

  ```js
  // esm
  import sum from "./sum";
  import { a } from "./sum";
  // cjs
  const s = require("./sum");
  const sum = s.default;
  const a = s.a;
  ```

### ESM 运行时代码分析:

相比 commonjs,多出了不少代码:

```js
(() => {
  // 使用getter定义exports的属性
  __webpack_require__.d = (exports, definition) => {
    for (var key in definition) {
      if (
        __webpack_require__.o(definition, key) &&
        !__webpack_require__.o(exports, key)
      ) {
        Object.defineProperty(exports, key, {
          enumerable: true,
          get: definition[key],
        });
      }
    }
  };
})();

// 用 __webpack_require__.o 简写 Object.prototype.hasOwnProperty
(() => {
  __webpack_require__.o = (obj, prop) =>
    Object.prototype.hasOwnProperty.call(obj, prop);
})();

/* webpack/runtime/make namespace object */
(() => {
  // 标记一个ESM模块
  __webpack_require__.r = (exports) => {
    if (typeof Symbol !== "undefined" && Symbol.toStringTag) {
      // 判断浏览器支不支持Symbol.toStringTag标准 详情: https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Symbol
      // let obj = {}
      // Object.prototype.toString.call(obj) ——> [object Object]
      // Object.defineProperty(obj, Symbol.toStringTag, { value: "Module" });
      // Object.prototype.toString.call(obj) ——> [object Module]
      Object.defineProperty(exports, Symbol.toStringTag, { value: "Module" });
    }
    Object.defineProperty(exports, "__esModule", { value: true });
  };
})();
```

webpack 运行时代码:

```js
// 源码
const sum = (...args) => args.reduce((x, y) => x + y, 0);

// 运行时代码:
export default sum;
export const name = "sum"(
  // 运行时代码

  // 该模块被 module_wrapper 包裹,和cjs一样
  (__unused_webpack_module, __webpack_exports__, __webpack_require__) => {
    // 标记该模块是一个 ESM 模块
    __webpack_require__.r(__webpack_exports__);

    // 导出所有的属性，即 __webpack_exports__，但通过 getter 方式，可以懒加载属性
    __webpack_require__.d(__webpack_exports__, {
      default: () => __WEBPACK_DEFAULT_EXPORT__,
      name: () => /* binding */ name,
    });

    // 执行代码，配置 getter 的属性
    const sum = (...args) => args.reduce((x, y) => x + y, 0);

    // export default
    const __WEBPACK_DEFAULT_EXPORT__ = sum;
    // export const name
    const name = "sum";
  }
);
```
