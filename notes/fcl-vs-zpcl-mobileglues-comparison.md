# FCL 能跑、ZPCL 崩：差在 MobileGlues 版本

2026-11-26 初记，2026-11-28 补了日志对比。

同一台 vivo V2410A（SM6450，Adreno 710，Android 15）跑 MC Java 26.3 原版，无 mod、无资源包，两个启动器都没手动放 .so。

ZPCL 1.5.6：启动到资源加载阶段就崩，terrain 那组 program 全部编译失败（日志里列了 12 个），报 `ERROR: 0:224: '_uniform' : undeclared identifier`，RuntimeException 退出，exit code -1。

FCL 1.3.4.9（Version Code 1349）：同一个 jar、同一台手机，资源加载完整走完，纹理图集全部创建，着色器没有报错，正常进游戏后手动退出，exit code 0。

两边日志里的渲染器行：

```
ZPCL: Graphics Drivers: 4.0.0 MobileGlues 2.0.0
FCL : using drivers: 4.0.0 MobileGlues 2.0.1·Dev1 (with Simple FPE Wrapper 26.11-dev, GIT@3042d8a)
```

其他能对比的项：

- 扩展列表两边完全一样，都只有 `GL_ARB_buffer_storage, GL_ARB_vertex_attrib_binding`。所以“缺 ARB_shader_draw_parameters 就会崩”这条不成立，FCL 也缺，但它不崩。
- customGLVersion 两边都是 4.0.0，ignoreError 都是 0。
- 不一样的三处：maxGlslCacheSize（FCL 32，ZPCL 0）、enableExtTimerQuery（FCL true，ZPCL false）、FCL 带 `-XX:ActiveProcessorCount=8`。

现在手里的判断：最可能的原因是 MG 版本，2.0.0 的这个问题在 2.0.1·Dev1 里已经修掉。这只是推测，没验证。

还没做的一步：从两个 APK 里取出 libmobileglues.so 比哈希，确认二进制确实不一样。

```
unzip -p /sdcard/Download/ZPCL.apk lib/arm64-v8a/libmobileglues.so > /sdcard/zpcl-mg.so
unzip -p /sdcard/Download/FCL.apk  lib/arm64-v8a/libmobileglues.so > /sdcard/fcl-mg.so
sha256sum /sdcard/zpcl-mg.so /sdcard/fcl-mg.so
```

哈希不同之后，想证明因果还得把 2.0.0 那份 .so 放进 FCL 跑一次，看能不能复现。这一步比塞进 ZPCL 容易，因为 FCL 的渲染器目录是自己的。

相关：notes/mobileglues-mc263-shader-issue.md。上游 MobileGlues#66 已被关闭并锁定，那次报告里“缺扩展即必崩”的推断已被这份 FCL 日志推翻。
