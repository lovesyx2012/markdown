函数this的指向，是当我们调用函数确定的，调用方式的不同决定了this指向不同，一般指向我们的调用者。

| 调用方式     | this指向                                    |
| ------------ | ------------------------------------------- |
| 普通函数调用 | window                                      |
| 构造函数调用 | 实例对象， 原型对象里面的方法也指向实例对象 |
| 对象方法调用 | 该方法所属对象                              |
| 事件绑定方法 | 绑定事件对象                                |
| 定时任务     | window                                      |
| 立即执行函数 | window                                      |