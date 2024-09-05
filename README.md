### 使用方法
 1. 引入components下面的组件 并 进行注册
 2. 通过 ref 绑定dom原生 使用this.$refs.xxx 调用方法
 3. 具体有 redo() saveCanvas() 较为重要 redo将画布中的绘画清除  saveCanvas返回一个promise 是一个对象 一个是原图一个是画笔处理后的mask图
