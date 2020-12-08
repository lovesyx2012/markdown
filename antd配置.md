[TOC]



#### 基于customize-cra和react-app-rewired 对create-react-app进行自定义配置

###### 添加react-app-rewired customize-cra 依赖 

```
npm i react-app-rewired customize-cra -D
```

###### 创建自定义配置文件 config-overides.js

```
/**
 * 基于customize-cra和react-app-rewired 对create-react-app进行自定义配置
 */

const {
    override
} = require('customize-cra')

module.exports = override(
)
```

###### 修改package.json里面的scripts

```
"scripts": {
  "start": "react-app-rewired start",
  "build": "react-app-rewired build",
  "test": "react-app-rewired test",
  "eject": "react-scripts eject"
},
```

#### 添加less less-loader 依赖 

###### 增加依赖

```
npm i less less-loader -D
```

###### 调整config-overides.js

```
const {
  override,
  addLessLoader
} = require('customize-cra')

module.exports = override(
  addLessLoader({
    lessOptions: {
      javascriptEnabled: true,
    }
  })
)
```

#### 添加antd babel-plugin-import依赖

###### 增加依赖

```
npm i antd -S
npm i babel-plugin-import -D
```

###### 调整config-overrides.js

```
const {
  override,
  addLessLoader,
  fixBabelImports
} = require('customize-cra')

module.exports = override(
  addLessLoader({
    lessOptions: {
      javascriptEnabled: true
    }
  }),

  fixBabelImports("import", {
    libraryName: "antd",
    libraryDirectory: "es",
    style: true // change importing css to less
  }),
)
```

#### 自定义主题

 ###### 创建theme.js

```
module.exports = {
    '@primary-color': '#1890ff', // 全局主色
    '@link-color': '#1890ff', // 链接色
    '@success-color': '#52c41a', // 成功色
    '@warning-color': '#faad14', // 警告色
    '@error-color': '#f5222d', // 错误色
    '@font-size-base': '14px', // 主字号
    '@heading-color': 'rgba(0, 0, 0, 0.85)', // 标题色
    '@text-color': 'rgba(0, 0, 0, 0.65)', // 主文本色
    '@text-color-secondary': 'rgba(0, 0, 0, 0.45)', // 次文本色
    '@disabled-color': 'rgba(0, 0, 0, 0.25)', // 失效色
    '@border-radius-base': '2px', // 组件/浮层圆角
    '@border-color-base': '#d9d9d9', // 边框色
    '@box-shadow-base': '0 3px 6px -4px rgba(0, 0, 0, 0.12), 0 6px 16px 0 rgba(0, 0, 0, 0.08),0 9px 28px 8px rgba(0, 0, 0, 0.05)', // 浮层阴影
}
```

###### 调整config-overrides.js

```
const modifyVars = require('./theme')

module.exports = override(
  addLessLoader({
    lessOptions: {
      javascriptEnabled: true,
      modifyVars
    }
  }),
  
  fixBabelImports("import", {
    libraryName: "antd",
    libraryDirectory: "es",
    style: true // change importing css to less
  }),
)
```



#### 增加装饰器支持

###### 调整config-overrides.js 增加 addDecoratorsLegacy

```
const {
  override,
  addLessLoader,
  fixBabelImports,
  addDecoratorsLegacy
} = require('customize-cra')

const modifyVars = require('./theme')

module.exports = override(
  addLessLoader({
    lessOptions: {
      javascriptEnabled: true,
      modifyVars
      //modifyVars: { "@primary-color": "#1DA57A" }
    }
  }),

  addDecoratorsLegacy(),

  fixBabelImports("import", {
    libraryName: "antd",
    libraryDirectory: "es",
    style: true // change import
    ing css to less
  }),
)
```

###### 增加babel注解依赖

```
npm i -D @babel/plugin-proposal-decorators
```

###### 一个HOC的简单测试App.js

```
import React, { Component } from 'react'
import { Button } from 'antd'

const testHOC = (WapperComponent) => {
  return class HOCComponent extends Component {
    render() {
      return (
        <>
        <WapperComponent />
        THIS IS HOC...
        </>
      )
    }
  }
}

@testHOC
class App extends Component {
  render() {
    return (
        <div>
        	<Button>click</Button>
      </div>
    )
  }
}

export default App
```



#### LocaleProvider国际化

###### LocaleProvider 使用 React 的 [context](https://facebook.github.io/react/docs/context.html) 特性，只需在应用外围包裹一次即可全局生效

```
import { ConfigProvider } from 'antd';
import zh_CN from 'antd/lib/locale-provider/zh_CN';
 
render(
  <Provider store={store}>
    <ConfigProvider locale={zhCN}>
      <App />
    </ConfigProvider>
  </Provider>,
  document.getElementById('root')
)
```



#### 使用axios获取数据

###### 增加依赖

```
npm i axios -S
```

###### request请求封装utils/request.js

```
import axios from 'axios'

const isDev = process.env.NODE_ENV === 'development'

const request = axios.create({
    baseURL: isDev ? 'https://api.zisuye.com' : ''
})

const authToken = "Basic " + "ZDc3NzlhMmM2OGIwNjcwYmI5YTY2MjU0MzNmMmNiMTk6YWJmODY5YzU3ODIwOWE2ODIwODExZDg4ZjNiZGRkY2Y=";

request.interceptors.request.use(config => {
console.log(config)
    config.headers = Object.assign({}, config.headers, {'Authorization': authToken})
    return config
})

request.interceptors.response.use(resp => {
    if (resp.status === 200 && resp.data.code === "0000") {
        return resp.data.data
    } else {
        // 统一异常处理
    }
})

export default request
```

