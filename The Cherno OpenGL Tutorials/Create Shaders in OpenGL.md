
### 主要知识要点总结（按视频教学流程整理）

1. **着色器基础回顾与 GLSL 介绍**  
   - OpenGL 现代管线必须使用着色器（Shader）才能渲染。  
   - GLSL（OpenGL Shading Language）是着色器专用的类 C 语言。  
   - 本集重点：**顶点着色器**（处理每个顶点的位置）和**片段着色器**（决定每个像素的颜色）。

2. **在 C++ 中编写着色器源码**  
   - 将 GLSL 代码以 `std::string` 形式硬编码在 C++ 文件中（后续可改成从文件读取）。  
   - 推荐使用原始字符串字面量 `R"( ... )"` 避免转义符和换行问题。  
   - **顶点着色器示例**（GLSL 代码）：

     ```glsl
     #version 330 core
     layout(location = 0) in vec4 position;
     void main() {
         gl_Position = position;
     }
     ```

   - **片段着色器示例**（GLSL 代码）：

     ```glsl
     #version 330 core
     out vec4 color;
     void main() {
         color = vec4(1.0, 0.0, 0.0, 1.0);  // 纯红色
     }
     ```

   - `#version 330 core` 表示使用现代 OpenGL 核心模式；`layout(location = 0)` 绑定顶点属性位置；`gl_Position` 是顶点着色器必须输出的内置变量；`out vec4 color` 是片段着色器输出颜色。

3. **着色器编译与链接流程（核心 API）**  
   视频实现了两个辅助函数：
   - `compileShader`：编译单个着色器（GL_VERTEX_SHADER / GL_FRAGMENT_SHADER）。
   - `createShader`：完整创建着色器程序。

   关键 OpenGL 函数调用顺序：
   - `glCreateShader()` → `glShaderSource()`（传入源码）→ `glCompileShader()`  
   - `glCreateProgram()` → `glAttachShader()`（附加顶点+片段着色器）→ `glLinkProgram()` → `glValidateProgram()`  
   - 编译/链接成功后用 `glDeleteShader()` 删除中间着色器对象（释放资源）。  
   - 最终返回 program ID（类似之前的 VBO ID）。

4. **错误检查机制（非常重要）**  
   - 使用 `glGetShaderiv(GL_COMPILE_STATUS)` 检查编译状态。  
   - 失败时用 `glGetShaderInfoLog()` 或 `glGetProgramInfoLog()` 获取详细错误日志（例如漏分号会报 syntax error）。  
   - 常见坑：着色器类型混淆、VAO 未绑定、Intel 显卡编译器严格等。

5. **使用着色器渲染**  
   - 渲染循环中调用 `glUseProgram(shader)` 激活着色器程序。  
   - 之后再 `glDrawArrays()` / `glDrawElements()` 即可渲染。  
   - 清理：程序结束时 `glDeleteProgram()`。

6. **本集实践成果与注意事项**  
   - 成功渲染出一个纯红色三角形（通过片段着色器输出固定颜色）。  
   - 强调现代 OpenGL **必须绑定 VAO**，否则什么都不会画出来。  
   - 源码管理建议：着色器源码可来自字符串、文件或二进制；注意 `std::string::c_str()` 生命周期问题（字符串必须在 OpenGL 调用期间保持有效）。  
   - 为后续章节铺路：下一集会封装 Shader 类和 Uniform（动态参数传递）。

### 学习建议

- 配合视频 GitHub 仓库（https://github.com/speauty/ChernoOpenGL）中的对应代码跟着敲一遍。
- 重点掌握 `createShader` 函数的完整实现，这是整个系列后续所有渲染的基础。
- 如果遇到编译错误，先检查 GLSL 语法和 `glGetShaderInfoLog` 输出。