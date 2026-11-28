# FCL 能跑、ZPCL 崩：差在 MobileGlues 版本

2026-11-26 初记，2026-11-28 补了日志对比。

同一台 vivo V2410A（SM6450，Adreno 710，Android 15）跑 MC Java 26.3 原版，无 mod、无资源包，两个启动器都没手动放 .so。

ZPCL 1.5.6：启动到资源加载阶段就崩，terrain 那组 program 全部编译失败（日志里列了 12 个），报 `ERROR: 0:224: '_uniform' : undeclared identifier`，RuntimeException 退出，exit code -1。

FCL 1.3.4.9（Version Code 1349）：同一个 jar、同一台手机，资源加载完整走完，纹理图集全部创建，着色器没有报错，正常进游戏后手动退出，exit code 0。

两边日志里的渲染器行，原样抄下来：

```
ZPCL: Graphics Drivers: 4.0.0 MobileGlues 2.0.0
FCL : using drivers: 4.0.0 MobileGlues 2.0.1·Dev1 (with Simple FPE Wrapper 26.11-dev, GIT@3042d8a)
```

这就是两边唯一的实质差别，其余都比过：

- 扩展列表完全一样，都只有 `GL_ARB_buffer_storage, GL_ARB_vertex_attrib_binding`。所以“缺 ARB_shader_draw_parameters 就会崩”这条不成立，FCL 也缺，但它不崩。
- customGLVersion 两边都是 4.0.0，ignoreError 都是 0。
- 另有三个小项不同：maxGlslCacheSize（FCL 32，ZPCL 0）、enableExtTimerQuery（FCL true，ZPCL false）、FCL 带 `-XX:ActiveProcessorCount=8`。这三样看着都不像能单独造成编译失败，但没排。

结论：崩的是 MG 2.0.0 这个版本，2.0.1·Dev1 不崩。2.0.0 的这个问题大概在新版本里已经修了。

没做的事：没拿 2.0.0 的 .so 反向放进 FCL 跑，所以“是版本还是版本顺带的配置”严格说还没分开。要分开才需要拆包比对 .so，不然不用动。

相关：notes/mobileglues-mc263-shader-issue.md。上游 MobileGlues#66 已关闭并锁定，那次报告里“缺扩展即必崩”的推断已被这份 FCL 日志推翻。
