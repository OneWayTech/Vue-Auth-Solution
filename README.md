# § Vue 权限控制

> 在看本文档之前，您需要阅读 [Vue 另类状态管理](https://github.com/kenberkeley/vue-state-management-alternative)

业界一向认为，权限只能是后端做  
但如果在前后端分离的前提下仍是这样实现，那么前后端分离是没有任何意义的，还不如直接后端渲染实在

目前有关 Vue 的权限控制并没有一个相对主流的解决方案，故在此抛砖引玉

首先先说明，我司并没有用 Vuex，仅仅就是 Vue + Vue Router  
稍微复杂一点的业务场景，都可以使用上述的“另类状态管理”以及 [eventbus](https://cn.vuejs.org/v2/guide/components.html#非父子组件的通信) 去解决 

***

## ⊙ 概览

一般我们项目的源码目录 `src/` 下都会有一个 `mixins/`，里面必然存在一个 `session.js`，有如：

> 我司约定 mixin 中的变量以及函数都应当使用 `$` 结尾（为什么不能用在开头？因为 Vue 不会代理 `$` / `_` 开头的变量，故统一置尾）

```js
/*** src/mixins/session.js ***/
import authService from '@/services/authService' // 权限相关的 API 封装服务
const goToSSO = () => {
  location.replace(`<我司单点登录 URL>?returnUrl=${encodeURIComponent(location.href)}`)
}

/* 登录凭据 */
export const session$ = {
  id: null,
  username: '',
  role: '',
  isLeader: null
}

/* 是否管理员 */
export const isAdmin$ = () => {
  return session$.role === 'admin'
}

/* 是否销售主管 */
export const isSalesLeader$ = () => {
  return session$.role === 'sales' && session$.isLeader
}

/* 挂载 DOM 前调用本函数 */
export const syncSession$ = () => {
  return authService.getSession().then(sess => {
    Object.assign(session$, sess) // 这里不建议写成 session$ = sess
  }).catch(() => {
    goToSSO() // 跳转到单点登录
    throw new Error('Redirecting to SSO') // 继续抛出，避免之后的 then 执行挂载 DOM
  })
}

// @export.default <mixin>
export default {
  data: () => ({
    session$
  }),
  computed: {
    isAdmin$,
    isSalesLeader$
  },
  methods: {
    logout$ () {
      authService.logout().then(goToSSO)
    }
  }
}
``` 

在启动文件中一般是这样的：

```js
/*** src/app.js ***/
import 'babel-polyfill'
import Vue from 'vue'
import App from '@/components/App'
import { syncSession$ } from '@/mixins/session'

// 同步 session 后才挂载 DOM
syncSession$().then(() => {
  /* eslint-disable no-new */
  new Vue({
    el: '#app',
    router: require('@/routes/').default, // 路由涉及权限，因此须在同步 session 后才执行
    render: h => h(App)
  })
})
```

以上就是我司权限管理的基石

***

## ⊙ 常见的疑问

Q：为什么不使用 LocalStorage / SessionStorage / cookies 去保存登录凭据？这样可以全局访问很方便啊  
A：可被篡改，没有安全性可言，而且还得考虑其过期时间以及解析错误等一系列不必要的麻烦

Q：用 mixin 全局共享状态的好处是什么？  
A：首先必须指出，要想让 `session$` 变成响应式，您必须要把 `src/mixins/session.js` 引入到任一组件中，这样就可以在组件内部（包括模板）访问到所有的变量与方法。而且，您还可以在非组件内部中访问。虽然 `export default` 的是 mixin 的固定格式，但 `export` 的却是直接的变量或方法，因此可以直接 `import { session$  } from '@/mixins/session'`，这几乎就像全局变量般便利，但又可以最大程度地保证安全性。更重要的，您还可以享受 Vue 带来的全局响应式、计算属性等一系列特性，而不仅仅是一个无法被外界篡改的闭包变量。举例说明：

```js
import session from '@/mixins/session'

export default {
  mixins: [session],
  watch: {
    /* 需求：测试模式下允许即时修改用户角色，请在控制台显示 */
    'session$.role' (newRole, oldRole) {
      console.info('角色已切换：', oldRole, ' => ', newRole)
    }
  }
}
```

Q：路由控制怎么处理？  
A：下面我们接着说

***

## ⊙ 路由级别控制

能在 Vue 层面上解决的事情没必要动用到 Vue Router 的特性，否则权限就写得太散了  
业内主流的方式都是通过 `beforeEach` 来获取 `meta` 信息以拦截  
不过话说回来，既然你都不想该角色看到的路由，为什么你还要挂载？画蛇添足多此一举莫过于此

举个例子，一个项目有 `/a`（默认）、`/b`、`/c`、`/d`、`/e` 五个路由，需满足：  
* 只有 管理员 可以看到 `/e`  
* 只有 管理员 与 销售 Leader 可以看到 `/d`

那么路由的定义可以这样写：

```js
import { isAdmin$, isSalesLeader$ } from '@/mixins/session'

export default [
  {
    path: '/a',
    alias: '/',
    component: require('@/views/a/')
  },

  {
    path: '/b',
    component: require('@/views/b/')
  },

  {
    path: '/c',
    component: require('@/views/c/')
  },

  (isAdmin$() || isSalesLeader$()) && {
    path: '/d',
    component: require('@/views/d/')
  },

  isAdmin$() && {
    path: '/e',
    component: require('@/views/e/')
  },
  
  { // 404 置尾
    path: '*',
    component: {
      beforeCreate () {
        this.$router.replace('/')
      },
      render: h => null
    }
  }
].filter(route => route) // 排除掉为 false 的项
```

这下你能可以理解为什么在启动文件中要在 `syncSession$` 完成后才引入路由了吧？  
如果在开头就 `import routes from '@/routes/'` 则无法实现控权（因为是先执行）

***

## ⊙ 按钮级别控制

基本就是把 `@/mixins/session` 引入到组件中就可以了，没有任何难度

***
***

### 2017/12/6 针对 SegmentFault 下[评论](https://segmentfault.com/p/1210000012206425?_ea=2945405)的更新

* 针对 `没有用动态路由，导致用户登录前不能初始化Vue应用，所以登陆页只能单独做，开始我也是这么做的，但始终觉得url跳转的体验不好，所以用动态路由解决了` 的解决方案：如果您的公司没有 SSO，那么每个项目都只能重复造轮子做登录页（之前我司就是如此），此时只能借助 `vue-router` 的钩子函数 `beforeEach` 控权：

```js
import { isLogin$ } from '@/mixins/session'
const LOGIN_PATH = '/auth/login'

export default function authInterceptor(to, from, next) {
  if (isLogin$()) {
    switch (to.path) {
      case LOGIN_PATH:
        next('/')
        return
      default:
        next()
    }
  } else {
    switch (to.path) {
      case LOGIN_PATH:
        next()
        return
      default:
        next(`${LOGIN_PATH}?referrer=${encodeURIComponent(to.fullPath)}`)
    }
  }
}

// 使用方式：router.beforeEach(authInterceptor)
```

* 针对 `在前端路由文件中根据角色做判断的做法不够灵活，路由权限还是由后端分发给前端比较好，这样当需要修改角色权限时，后端改一下配置，前端刷新就生效了` 的回应：把我司现行完善的权限设计一股脑搬出来说没有意义，以上例子只是为了简要说明，更重要的是思想
