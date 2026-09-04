# Java 后端 OCR 集成：JNA 直调 PaddleOCR C++ 引擎

## 1. 文档说明

| 属性 | 说明 |
|------|------|
| 文档版本 | V1.0 |
| 生效日期 | 2026-09-04 |
| 信息来源 | 微信公众号《明月心技术学堂》 |
| 原文标题 | Java 开发者的福音！后端 OCR，可直接调 PaddleOCR 的 C++ 高性能引擎 |
| 原文链接 | https://mp.weixin.qq.com/s/orYjOGg5LzexmhmHkduChg |
| 整理 | pangpang-doc |

> **摘要**：本文档整理了微信公众号《明月心技术学堂》的一篇技术文章，介绍 Java 后端项目如何通过 **JNA 直接加载 PaddleOCRSharp 编译的 C++ 动态库**（Windows 下 `PaddleOCR.dll`，Linux 下 `PaddleOCR.so`）实现高精度中文 OCR，无需引入 Python 运行时、无需写 JNI、无需额外部署 OCR 微服务。

---

## 2. 核心结论

如果 Java 后端项目需要做 OCR，且能拿到 PaddleOCRSharp 编译的 C++ 动态库，**用 JNA 直接加载该库是很好的方案**，优点如下：

- 不引入 Python 运行时
- 不写 JNI
- 不额外部署微服务
- Spring Boot 里 `@Autowired` 就能用
- 模型离线、数据不出机器
- 识别结果与其他方案（如 Python/PaddleOCR）一致

### 2.1 传统做法（痛点）

```java
// 方式一：ProcessBuilder 起 Python 进程
ProcessBuilder pb = new ProcessBuilder(
    "python", "ocr_server.py", imagePath
);
Process p = pb.start();
String result = IOUtils.toString(p.getInputStream(), StandardCharsets.UTF_8);

// 方式二：调内部 Python HTTP 服务
RestTemplate rest = new RestTemplate();
OCRResponse resp = rest.postForObject(
    "http://ocr-service:5000/detect",
    new ImageRequest(base64),
    OCRResponse.class
);
```

两种方式都引入 Python 环境/独立服务，存在部署复杂、进程开销、链路长等问题。而 PaddleOCRSharp 的 C++ 内核 `PaddleOCR.dll`，**Java 一样能用**。

---

## 3. PaddleOCRSharp 简介

PaddleOCRSharp 是基于百度飞桨 PaddleOCR 的开源代码修改并优化的 **.NET 版本 OCR 可离线使用类库**：

- **核心组件**：`PaddleOCR.dll` 由 C++ 编写，根据 PaddleOCR 的 C++ 代码修改、优化与编译。
- **多语言支持**：已支持 C/C++、.NET、Python、Golang、Rust、Java、LabVIEW、Delphi 等众多语言的直接 API 调用。
- **功能范围**：文本识别、文本检测、表格识别。
- **识别能力**：支持超轻量级中文 OCR，单模型支持中、英、数字及 PaddleOCR 官方涵盖的多语种识别，同时支持竖排文本、长文本识别。
- **易用性**：封装极其简化，实际调用仅几行代码；NuGet 包即装即用，支持离线部署。

> **关键点**：Java 侧不需要 NuGet 包，也不需要 C# 运行时。只要拿到中间那层 C++ 动态库 + 它的 C 风格导出头文件，JNA 直接上。

**运行依赖与模型库**（拿来即用）：https://gitee.com/raoyutian/PaddleOCRSharp/tree/master/Demo/win_runtime_x64

---

## 4. JNA 接口定义（PaddleOCR.java）

作者仓库中已提供完整的 JNA 接口映射：

```java
import com.sun.jna.Pointer;
import com.sun.jna.Library;
import com.sun.jna.Native;

public interface PaddleOCR extends Library {
    PaddleOCR INSTANCE = Native.load("PaddleOCR", PaddleOCR.class);

    void libaddLicense(String licfile);

    // OCR
    Pointer Initializejson(String modelPath_det_infer, String modelPath_cls_infer,
                           String modelPath_rec_infer, String keys, String parameterjson);
    void EnableANSIResult(Pointer engine_p, boolean enable);
    void EnableJsonResult(Pointer engine_p, boolean enable);
    void libEnableDetUseRect(Pointer engine_p, boolean enable);
    String Detect(Pointer engine, String imagefile);
    String DetectByte(Pointer engine, byte[] imagebytedata, Pointer size);
    String DetectBase64(Pointer engine, String imagebase64);
    String DetectByteData(Pointer engine, byte[] img, int nWidth, int nHeight, int nChannel);
    void FreeEngine(Pointer engine);

    // Table
    boolean StructureInitializejson(String modelPath_det_infer, String modelPath_rec_infer,
                                    String keys, String table_model_dir, String table_char_dict_path,
                                    String parameterjson);
    String GetStructureDetectFile(String imagefile);
    String GetStructureDetectByte(byte[] imagebytedata, Pointer size);
    String GetStructureDetectBase64(String imagebase64);
    void FreeStructureEngine();
    String GetError();
}
```

---

## 5. 调用示例（PaddleOCRDemo.java）

```java
import com.sun.jna.Pointer;
import java.io.File;
import java.io.IOException;
import java.nio.charset.StandardCharsets;
import java.nio.file.Files;
import java.nio.file.Paths;

public class PaddleOCRDemo {
    public static void main(String[] args) throws IOException {
        String root = System.getProperty("user.dir");
        String jsonConfig = new String(
                Files.readAllBytes(Paths.get(root + "/inference/PaddleOCR.config.json")),
                StandardCharsets.UTF_8
        );
        String det_infer = root + "/inference/PP-OCRv6_small_det_infer";
        String cls_infer = root + "/inference/PP-OCRv5_mobile_cls_infer";
        String rec_infer = root + "/inference/PP-OCRv6_small_rec_infer";
        String keys      = root + "/inference/keys.txt";

        Pointer ptr = PaddleOCR.INSTANCE.Initializejson(det_infer, cls_infer, rec_infer, keys, jsonConfig);
        if (ptr == null || ptr.equals(Pointer.NULL)) {
            System.err.println("PaddleOCR Initializejson failed!");
            return;
        }
        PaddleOCR.INSTANCE.EnableANSIResult(ptr, true);
        // 返回纯文本
        PaddleOCR.INSTANCE.EnableJsonResult(ptr, false);
        // 返回 JSON 字符串：PaddleOCR.INSTANCE.EnableJsonResult(ptr, true);

        File imageDir = new File(root + "/image");
        File[] files = imageDir.listFiles();
        if (files == null || files.length == 0) {
            System.err.println("image dir is empty");
            return;
        }
        double totalTimes = 0;
        int count = 0;
        for (File file : files) {
            String imagePath = file.getAbsolutePath();
            long start = System.nanoTime();
            String ocrResult = PaddleOCR.INSTANCE.Detect(ptr, imagePath);
            long end = System.nanoTime();
            double ms = (end - start) / 1_000_000.0;
            totalTimes += ms;
            System.out.printf("--%d-----耗时:【 %.2f】ms,文件名【%s】-----%n", count, ms, file.getName());
            System.out.println(ocrResult);
            count++;
        }
        System.out.printf("--total times:%.2fms,平均:%.2fms-----------%n", totalTimes, totalTimes / count);
        // 防止程序直接退出
        System.out.println("Press Enter to exit...");
        System.in.read();
    }
}
```

> 没有 Python、没有额外进程、没有 HTTP 客户端、没有序列化胶水。

---

## 6. 返回结果格式

`Detect` 系列接口默认返回 JSON（可通过 `EnableJsonResult` 控制），大致结构：

```json
[
  {
    "Text": "发票代码：144001900111",
    "Score": 0.987,
    "BoxPoints": [[x1,y1], [x2,y2], [x3,y3], [x4,y4]]
  },
  {
    "Text": "开票日期：2024年03月15日",
    "Score": 0.972,
    "BoxPoints": [[x1,y1], [x2,y2], [x3,y3], [x4,y4]]
  }
]
```

Java 侧直接 `JSON.parseArray(result, OcrItem.class)` 即可使用。

---

## 7. 常见疑问解答

| 疑问 | 解答 |
|------|------|
| **JNA 性能够吗？** | JNA 调用 native 方法的开销大概在**几十纳秒级**，和 OCR 推理本身（几十到几百毫秒）相比可忽略。真正吃时间的是模型推理，不在 JNA 这一层。 |
| **和 PaddleOCRSharp 的 C# 版本效果一样吗？** | **一模一样**。底层是同一个 `PaddleOCR.dll` / `PaddleOCR.so`、同一个模型文件、同一个后处理逻辑。C# 的 P/Invoke 和 Java 的 JNA 只是不同的"门面"，进去后走同一条路。 |
| **Linux 服务器上能跑吗？** | 能。PaddleOCRSharp 有 Linux x64 的 `.so` 构建（也有信创 ARM/龙芯版本）。`Native.load("PaddleOCR", ...)` 在 Linux 下自动找 `PaddleOCR.so`，只要 `LD_LIBRARY_PATH` 或 `java.library.path` 包含其所在目录即可。 |
| **表格识别怎么做？** | 用 `StructureInitialize` 初始化带表格模型的引擎，调 `GetStructureDetectFile` / `GetStructureDetectByte`，返回 HTML 格式的表格结构。JNA 接口映射方式与普通 OCR 完全对称。 |
| **模型文件去哪拿？** | PaddleOCRSharp 的 Gitee 仓库包含各种模型；也可直接用 PaddleOCR 官方仓的 inference 模型，路径对上即可。 |

---

## 8. 方案对比

| 方案 | 中文/复杂版面识别率 | 部署复杂度 | 运行时依赖 | 适用场景 |
|------|---------------------|------------|------------|----------|
| **PaddleOCRSharp C++ 内核 + JNA** | 高 | 低（一个 dll/so + 模型目录） | 无 Python、无 C# 运行时 | Java 后端进程内 OCR，离线、数据不出机器 |
| Tess4J（Tesseract） | 低（中文/复杂版面/表格较弱） | 低 | JNA + Tesseract 本地库 | 简单印刷体识别 |
| Python PaddleOCR 独立服务 | 高 | 高（独立部署 + 网络链路） | Python 运行时 | 已有 Python 服务或 GPU 推理集群 |

> **原文观点**：Java 生态里最容易搜到的是 Tess4J，但 Tesseract 对中文场景、复杂版面、表格的识别率与 PaddleOCR 系列不在一个量级。PaddleOCRSharp 的 C++ 内核 + JNA，本质上给 Java 后端补上了"本地高精度 OCR 能力"这块短板，且接入成本极低。

---

## 9. 资源与参考

- 作者仓库（含 dll + 模型 + 头文件 + Java 可编译运行 Demo）：https://gitee.com/raoyutian/PaddleOCRSharp
- Windows 运行依赖与模型库（win_runtime_x64）：https://gitee.com/raoyutian/PaddleOCRSharp/tree/master/Demo/win_runtime_x64
- 原文链接：https://mp.weixin.qq.com/s/orYjOGg5LzexmhmHkduChg

---

## 10. 落地评估与注意事项

1. **动态库获取**：需要先拿到 PaddleOCRSharp 编译的 C++ 动态库（dll/so）与 C 风格导出头文件；自编译需配置飞桨 C++ 编译链。
2. **模型体积与部署**：模型文件（det/cls/rec/keys）需随应用分发，注意镜像体积与磁盘占用。
3. **平台差异**：Windows 与 Linux 使用不同动态库；信创环境需确认 ARM/龙芯构建版本。
4. **线程与并发**：引擎初始化较耗时，建议单例初始化 + 复用 engine 指针；并发场景需评估引擎线程安全性。
5. **授权与合规**：PaddleOCR 与 PaddleOCRSharp 均为开源（Apache-2.0），商用前需核对各自开源协议与依赖组件的许可证。
6. **备选方案**：若团队已有 GPU 或 Python OCR 服务，可对比"进程内 JNA"与"独立服务"在吞吐、维护成本上的取舍。

---

**文档结束**

*本文档由微信公众号文章整理，pangpang-doc 维护*
