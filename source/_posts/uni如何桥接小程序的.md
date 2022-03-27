---
title: uni如何桥接小程序的
categories:
  - default
tags:
  - default
date: 2022-03-10 20:00:33
---

uni编译后的代码

```javascript
const getEmitter = (function () {
  let Emitter;
  return function getUniEmitter() {
    if (!Emitter) {
      Emitter = new Vue();
    }
    return Emitter;
  };
})();

function apply(ctx, method, args) {
  return ctx[method].apply(ctx, args);
}

function $on() {
  return apply(getEmitter(), "$on", [...arguments]);
}
function $off() {
  return apply(getEmitter(), "$off", [...arguments]);
}
function $once() {
  return apply(getEmitter(), "$once", [...arguments]);
}
function $emit() {
  return apply(getEmitter(), "$emit", [...arguments]);
}
```



##### uni连接vue和小程序的桥梁

```javascript
function initProperties(props, isBehavior = false, file = "") {
  const properties = {};
  if (!isBehavior) {
    properties.vueId = {
      type: String,
      value: "",
    };
    // 用于字节跳动小程序模拟抽象节点
    properties.generic = {
      type: Object,
      value: null,
    };
    properties.vueSlots = {
      // 小程序不能直接定义 $slots 的 props，所以通过 vueSlots 转换到 $slots
      type: null,
      value: [],
      observer(newVal, oldVal) {
        const $slots = Object.create(null);
        newVal.forEach((slotName) => {
          $slots[slotName] = true;
        });
        this.setData({//调用小程序的setData
          $slots,
        });
      },
    };
  }
  if (Array.isArray(props)) {
    // ['title']
    props.forEach((key) => {
      properties[key] = {
        type: null,
        observer: createObserver(key),
      };
    });
  } else if (isPlainObject(props)) {
    // {title:{type:String,default:''},content:String}
    Object.keys(props).forEach((key) => {
      const opts = props[key];
      if (isPlainObject(opts)) {
        // title:{type:String,default:''}
        let value = opts.default;
        if (isFn(value)) {
          value = value();
        }

        opts.type = parsePropType(key, opts.type);

        properties[key] = {
          type: PROP_TYPES.indexOf(opts.type) !== -1 ? opts.type : null,
          value,
          observer: createObserver(key),
        };
      } else {
        // content:String
        const type = parsePropType(key, opts);
        properties[key] = {
          type: PROP_TYPES.indexOf(type) !== -1 ? type : null,
          observer: createObserver(key),
        };
      }
    });
  }
  return properties;
}

```



##### uni创建页面和组件

```javascript
//小程序的component https://developers.weixin.qq.com/miniprogram/dev/reference/api/Component.html
function createPage(vuePageOptions) {
  {
    return Component(parsePage(vuePageOptions));
  }
}

function createComponent(vueOptions) {
  {
    return Component(parseComponent(vueOptions));
  }
}
```



##### uni创建App

```javascript
function createApp(vm) {
  App(parseApp(vm));//小程序的App方法https://developers.weixin.qq.com/miniprogram/dev/reference/api/App.html
  return vm;
}

function parseApp(vm) {
  return parseBaseApp(vm, {
    mocks,
    initRefs,
  });
}

const hooks = [
  "onShow",
  "onHide",
  "onError",
  "onPageNotFound",
  "onThemeChange",
  "onUnhandledRejection",
];

function parseBaseApp(vm, { mocks, initRefs }) {
  initEventChannel();
  {
    initScopedSlotsParams();
  }
  if (vm.$options.store) {
    Vue.prototype.$store = vm.$options.store;//挂在store
  }

  Vue.prototype.mpHost = "mp-weixin";

  Vue.mixin({
    beforeCreate() {
      if (!this.$options.mpType) {
        return;
      }

      this.mpType = this.$options.mpType;

      this.$mp = {
        data: {},
        [this.mpType]: this.$options.mpInstance,
      };

      this.$scope = this.$options.mpInstance;

      delete this.$options.mpType;
      delete this.$options.mpInstance;
      if (this.mpType === "page" && typeof getApp === "function") {
        // hack vue-i18n
        const app = getApp();
        if (app.$vm && app.$vm.$i18n) {
          this._i18n = app.$vm.$i18n;
        }
      }
      if (this.mpType !== "app") {
        initRefs(this);
        initMocks(this, mocks);
      }
    },
  });

  const appOptions = {
    onLaunch(args) {//这里定义了onLaunch，其他在hooks中
      if (this.$vm) {
        // 已经初始化过了，主要是为了百度，百度 onShow 在 onLaunch 之前
        return;
      }
      {
        if (wx.canIUse && !wx.canIUse("nextTick")) {
          // 事实 上2.2.3 即可，简单使用 2.3.0 的 nextTick 判断
          console.error(
            "当前微信基础库版本过低，请将 微信开发者工具-详情-项目设置-调试基础库版本 更换为`2.3.0`以上"
          );
        }
      }

      this.$vm = vm;

      this.$vm.$mp = {
        app: this,
      };

      this.$vm.$scope = this;
      // vm 上也挂载 globalData
      this.$vm.globalData = this.globalData;

      this.$vm._isMounted = true;
      this.$vm.__call_hook("mounted", args);

      this.$vm.__call_hook("onLaunch", args);
    },
  };

  // 兼容旧版本 globalData
  appOptions.globalData = vm.$options.globalData || {};
  // 将 methods 中的方法挂在 getApp() 中
  const { methods } = vm.$options;
  if (methods) {
    Object.keys(methods).forEach((name) => {
      appOptions[name] = methods[name];
    });
  }

  initHooks(appOptions, hooks);

  return appOptions;
}
```

