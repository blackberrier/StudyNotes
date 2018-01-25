# HBPUI appmanager server
1. logback方式实现日志滚动:log.info(xxx)
    logback.xml放在lib库中
2. 打开的文件流、客户端需要关闭，关闭顺序为先关外层，再关内层。
3. try catch finally 方式进行异常处理，关闭动作放进finally语句块，语句块中中途出现错误不会继续执行，所以不相关的client的关闭不能放在可能出现的异常后面。
