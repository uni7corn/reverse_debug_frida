# Frida代码

* Frida代码=Frida源码
  * 总入口: https://github.com/frida/frida
    * `core`: [frida-core](https://github.com/frida/frida-core)
      * Frida core library intended for static linking into bindings
    * `gum`: [frida-gum](https://github.com/frida/frida-gum)
      * Cross-platform instrumentation and introspection library written in C
        * This library is consumed by frida-core through its `JavaScript` bindings, `GumJS`.
    * `tools`: [frida-tools](https://github.com/frida/frida-tools)
      * Frida CLI tools
    * `bindings`
      * python -> [frida-python](https://github.com/frida/frida-python)
      * Node.js -> [frida-node](https://github.com/frida/frida-node)
      * Swift -> [frida-swift](https://github.com/frida/frida-swift)
      * .NET -> [frida-clr](https://github.com/frida/frida-clr)
      * GO -> [frida-go](https://github.com/frida/frida-go)
      * Rust -> [frida-rust](https://github.com/frida/frida-rust)
      * QT/qml -> [frida-qml](https://github.com/frida/frida-qml)
    * `其他`
      * Frida支持多个移动端的互操作，所以有分别的内部的互操作相关的内容
        * iOS端的：[frida-objc-bridge](https://github.com/frida/frida-objc-bridge)
          * Objective-C runtime interop from Frida
        * Android端的：[frida-java-bridge](https://github.com/frida/frida-java-bridge)
          * Java runtime interop from Frida
      * [Capstone](https://github.com/frida/capstone)
        * 记得是：Frida内部用到了Capstone，但是有些额外的内容要微调，所以单独fork了Capstone源码，自己同步更新了
