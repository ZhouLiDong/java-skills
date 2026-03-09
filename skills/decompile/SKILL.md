---
name: "decompile"
description: "反编译JAR文件中的Java类以读取源代码。触发场景：需要读取第三方或内部JAR中的类、方法、字段；验证库类方法或字段是否存在；理解框架类的实现逻辑；确认方法的准确签名；了解框架组件的工作原理或调用链路；遇到编译错误需排查API使用问题。当项目源码中找不到某个类时，也应调用此skill从依赖JAR中查找。"
---

# 反编译Java类

此skill使您能够反编译JAR文件中的Java类文件，从而读取和理解第三方库或内部JAR依赖中类、方法和字段的实际实现。

## 触发时机

**在以下情况下调用此skill：**
- 需要读取JAR文件中的类（第三方或内部依赖）
- 需要验证库类中是否存在某个方法或字段
- 需要理解框架类的实现逻辑
- 用户要求查看或检查JAR类的源代码
- 遇到编译错误，可能是由于API使用不当导致
- 在生成代码前需要确认方法的准确签名
- 需要了解某个框架组件的工作原理或调用链路时

## 工作原理

### 缓存目录结构

反编译后的源码会被缓存，避免重复反编译。缓存目录位于用户主目录下：

```
<用户主目录>/.skill-cache/decompiled/
├── spring-core-5.3.20/
│   └── org/springframework/core/
│       └── SpringVersion.java
├── fastjson-1.2.83/
│   └── com/alibaba/fastjson/
│       └── JSONObject.java
```

**注意**：`<用户主目录>` 需要根据实际操作系统确定：
- Windows: 通常是 `C:\Users\{用户名}`
- Linux/macOS: 通常是 `/home/{用户名}` 或 `/Users/{用户名}`

### 工作流程

1. **优先检查缓存**：反编译前，先检查缓存中是否已存在该类
2. **按需反编译**：如果未缓存，使用Quiltflower反编译JAR
3. **保存到缓存**：将反编译后的源码存储到缓存目录
4. **返回源码**：提供反编译后的Java源代码

## 使用说明

### 步骤1：定位JAR文件

首先，确定JAR文件路径和版本：

**方法1：从项目配置中查找依赖版本（推荐）**

对于Maven项目：
1. 读取项目根目录下的 `pom.xml` 文件
2. 在 `<dependencies>` 中查找所有依赖
3. **关键步骤**：在本地Maven仓库中搜索包含目标类的JAR文件：
   ```bash
   # Windows PowerShell - 搜索包含特定类的JAR文件
   # 示例：搜索包含 SpringVersion 类的 JAR
   $className = "SpringVersion.class"
   $m2Repo = "$env:USERPROFILE\.m2\repository"
   Get-ChildItem -Path $m2Repo -Filter "*.jar" -Recurse | ForEach-Object {
       $jar = $_.FullName
       $result = jar tf $jar 2>$null | Select-String $className
       if ($result) {
           Write-Output "Found in: $jar"
       }
   }
   ```
4. 从搜索结果中筛选出项目依赖中存在的JAR（根据pom.xml中的依赖）
5. 如果找到多个版本，优先选择pom.xml中指定的版本

对于Gradle项目：
1. 读取项目根目录下的 `build.gradle` 文件
2. 在 `dependencies` 中查找所有依赖
3. 在本地Gradle缓存中搜索包含目标类的JAR文件

**方法2：使用Maven依赖树（如果项目可构建）**

```bash
# 查看完整依赖树
mvn dependency:tree

# 或者查看特定依赖
mvn dependency:tree -Dincludes=org.springframework:spring-core
```

**方法3：询问用户（最后手段）**

如果以上方法都无法确定，向用户询问：
- 该类所属的依赖名称和版本
- 或者直接提供JAR文件路径

### 步骤2：检查缓存

检查反编译后的类是否已存在于缓存中：
```
缓存路径：<用户主目录>/.skill-cache/decompiled/{jar名称-版本}/{包路径}/{类名}.java
```

示例：
```
C:\Users\ibank\.skill-cache\decompiled\spring-core-5.3.20\org\springframework\core\SpringVersion.java
```

**注意**：需要将 `<用户主目录>` 替换为实际的用户主目录路径。

### 步骤3：反编译（如果未缓存）

使用Quiltflower反编译JAR：

```bash
java -jar "<skill目录>/quiltflower-1.8.1.jar" "<jar路径>" "<缓存目录>"
```

示例（Windows系统）：
```bash
# 全局skill环境
java -jar "C:\Users\ibank\.trae-cn\skills\decompile\quiltflower-1.8.1.jar" "C:\Users\ibank\.m2\repository\org\springframework\spring-core\5.3.20\spring-core-5.3.20.jar" "C:\Users\ibank\.skill-cache\decompiled\spring-core-5.3.20"

# 本地skill环境（skill目录为项目中的相对路径）
java -jar ".trae\skills\decompile\quiltflower-1.8.1.jar" "C:\Users\ibank\.m2\repository\org\springframework\spring-core\5.3.20\spring-core-5.3.20.jar" "C:\Users\ibank\.skill-cache\decompiled\spring-core-5.3.20"
```

**注意**：
- `<skill目录>` 是 skill 所在的目录路径
- `<缓存目录>` 是反编译结果的输出目录
- `<jar路径>` 是要反编译的 JAR 文件路径
- Quiltflower 会反编译整个 JAR 到缓存目录，之后可直接读取需要的类

### 步骤4：读取反编译后的源码

反编译完成后，读取生成的Java文件：
```
<缓存目录>/<包路径>/<类名>.java
```

示例：
```
C:\Users\ibank\.skill-cache\decompiled\spring-core-5.3.20\org\springframework\core\SpringVersion.java
```

## 重要注意事项

1. **缓存优先**：始终在反编译前检查缓存，以节省时间
2. **版本感知**：不同版本的JAR有独立的缓存目录
3. **包路径转换**：将全限定类名转换为文件路径（如 `com.example.Foo` -> `com/example/Foo.java`）
4. **JAR命名**：使用JAR文件名（包含版本号）作为缓存子目录名
5. **错误处理**：如果反编译失败，向用户报告问题
6. **缓存目录强制约束**：
   - 缓存目录必须位于 `<用户主目录>/.skill-cache/decompiled/` 下
   - **禁止**在项目目录或其他任何位置创建缓存目录
   - 如果无法在用户主目录下创建缓存目录（如权限不足），**必须立即停止任务**，向用户报告问题并询问如何处理
   - 未经用户明确指示，不得擅自更改缓存目录位置

## 反编译工具信息

### Quiltflower（默认工具）

- **工具位置**：`<skill目录>/quiltflower-1.8.1.jar`
  - 全局环境：通常是 `C:\Users\{用户名}\.trae-cn\skills\decompile\quiltflower-1.8.1.jar`（Windows）
  - 本地环境：通常是 `.trae\skills\decompile\quiltflower-1.8.1.jar`（相对于项目根目录）
- **版本**：Quiltflower 1.8.1
- **来源**：https://github.com/QuiltMC/quiltflower

### CFR（备用工具）

- **工具位置**：`<skill目录>/cfr-0.152.jar`
  - 全局环境：通常是 `C:\Users\{用户名}\.trae-cn\skills\decompile\cfr-0.152.jar`（Windows）
  - 本地环境：通常是 `.trae\skills\decompile\cfr-0.152.jar`（相对于项目根目录）
- **版本**：CFR 0.152
- **来源**：https://github.com/leibnitz27/cfr

## 使用示例

### 示例1：反编译Spring类

用户问："org.springframework.util.StringUtils有哪些方法？"

1. 查找JAR：在Maven仓库中找到 `spring-core-5.3.20.jar`
2. 检查缓存：`C:\Users\ibank\.skill-cache\decompiled\spring-core-5.3.20\org\springframework\util\StringUtils.java`
3. 如果未缓存，执行反编译：
   ```bash
   # 全局skill环境
   java -jar "C:\Users\ibank\.trae-cn\skills\decompile\quiltflower-1.8.1.jar" "C:\Users\ibank\.m2\repository\org\springframework\spring-core\5.3.20\spring-core-5.3.20.jar" "C:\Users\ibank\.skill-cache\decompiled\spring-core-5.3.20"
   
   # 本地skill环境
   java -jar ".trae\skills\decompile\quiltflower-1.8.1.jar" "C:\Users\ibank\.m2\repository\org\springframework\spring-core\5.3.20\spring-core-5.3.20.jar" "C:\Users\ibank\.skill-cache\decompiled\spring-core-5.3.20"
   ```
4. 读取反编译后的文件并列出方法

### 示例2：验证方法签名

用户问："fastjson的JSONObject是否有put(String, Object)方法？"

1. 查找JAR：`fastjson-1.2.83.jar`
2. 检查缓存或反编译 `com.alibaba.fastjson.JSONObject`
3. 读取源码验证方法签名

## 局限性

- 反编译后的代码可能与原始源码不完全相同
- 某些结构（如lambda表达式）可能被反编译成不同的形式
- 原始源码中的注释和格式会丢失
- 此skill需要Java运行时环境来执行反编译工具

## 反编译失败处理

### 常见失败原因

某些类或方法可能无法被正确反编译，常见原因包括：

1. **混淆代码**：经过ProGuard等工具混淆的代码，控制流可能被改变
2. **复杂控制流**：异常复杂的try-catch-finally结构
3. **字节码优化**：编译器优化后的字节码结构
4. **特殊字节码指令**：某些不常见的字节码模式

### 失败时的表现

反编译失败时，反编译工具会在代码中插入错误信息：

```java
/*
 * Exception decompiling
 */
public SomeType someMethod() {
    /*
     * This method has failed to decompile.  When submitting a bug report...
     */
    throw new IllegalStateException("Decompilation failed");
}
```

### 处理流程

当遇到反编译失败时，按以下顺序处理：

**步骤0：删除整个JAR的缓存目录（重要）**

由于两个工具都会反编译整个JAR，如果目标类反编译失败，需要删除整个JAR的缓存目录，避免后续使用到错误的缓存结果：
```bash
# 删除整个JAR的反编译缓存目录
Remove-Item -Path "<缓存目录>/<jar名称-版本>" -Recurse -Force
```

示例：
```bash
Remove-Item -Path "C:\Users\ibank\.skill-cache\decompiled\spring-core-5.3.20" -Recurse -Force
```

**步骤1：尝试其他反编译工具**

本skill提供多个反编译工具，按以下顺序尝试：

1. **Quiltflower**（默认）：`<skill目录>/quiltflower-1.8.1.jar`
2. **CFR**（备用）：`<skill目录>/cfr-0.152.jar`

使用 CFR 的命令：
```bash
java -jar "<skill目录>/cfr-0.152.jar" --outputdir "<缓存目录>" "<jar路径>"
# 注意：CFR的参数顺序与Quiltflower不同
```

**注意**：两个工具都会反编译整个 JAR 文件，Quiltflower 参数顺序是 `<jar路径> <输出目录>`，CFR 参数顺序是 `--outputdir <输出目录> <jar路径>`。

**步骤2：使用 javap 查看字节码（最后手段）**

如果所有反编译器都失败，使用 javap 查看类结构：
```bash
# 从 JAR 中提取 class 文件
jar xf "<jar路径>" "<类路径>.class"

# 使用 javap 查看
javap -p -c -v "<类路径>.class"
```

**步骤3：告知用户**

向用户说明反编译失败的原因，并提供已获取的信息（如方法签名、字段列表等）

### 示例：处理反编译失败

当 `InvokeBo` 方法反编译失败时：

1. 删除整个JAR的缓存目录
2. 尝试使用 CFR 反编译
3. 如果仍失败，使用 javap 查看字节码
4. 向用户报告：
   ```
   反编译警告：方法 InvokeBo 反编译失败，已尝试 Quiltflower、CFR 两个工具。
   
   已获取的信息：
   - 方法签名：public AjaxResult InvokeBo(JSONObject invokeParam)
   - 方法注解：@PostMapping(value={"InvokeBo"})
   - 依赖的类：TokenService, ISysBusiExceptionLogService, IDao, AjaxResult, JSONObject
   
   如需查看完整实现，建议：
   1. 使用 javap -v 查看字节码
   2. 联系代码维护者获取源码
   ```
