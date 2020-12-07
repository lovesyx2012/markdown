#### react-loadable 实现动态加载

###### 安装依赖

```
npm i react-loadable -S
```

###### 创建components/Loading/index.js

```
import React, { Component } from 'react';

class Loading extends Component {
    render() {
        return (
            <div>
                loading...
            </div>
        );
    }
}

export default Loading;
```

###### 调整导入组件的方式

```
import Loading from '../components/Loading'
import Loadable from 'react-loadable'

//import Dashboard from "../views/Dashboard/Dashboard"
const Dashboard = Loadable({
    Loader: () => import("../views/Dashboard/Dashboard"),
    Loading: Loading
})
```

###### 可以控制台网络请求里面看倒相关的1.chunk.js



#### 自定义Loadable替换react-loadable

###### 创建自己的views/loadable.js

```
import React, { Component } from 'react';


const Loadable = ({ loader, loading: Loading }) => {
    return class LoadableComponent extends Component {
        state = {
            LoadedComponent: null
        }

        componentDidMount() {
            // loader() 等同于 import("xxx")，返回的是一个promise
            loader()
                .then(resp => {
                    // resp.default即导入的组件
                    console.log(resp)
                    this.setState({
                        LoadedComponent: resp.default
                    })
                })
        }

        render() {
            const { LoadedComponent } = this.state
            return (
                LoadedComponent
                    ?
                    <LoadedComponent />
                    :
                    <Loading />
            );
        }
    }
}

export default Loadable;
```

###### 将react-loadable替换成自定义Loadable

```
import Loading from '../components/Loading'
// import Loadable from 'react-loadable'
import Loadable from '../views/loadable'

const Dashboard = Loadable({
    loader: () => import("../views/Dashboard/Dashboard"),
    loading: Loading
})
```

### 一般页面级组件才会动态加载，普通组件不需要动态加载!!!