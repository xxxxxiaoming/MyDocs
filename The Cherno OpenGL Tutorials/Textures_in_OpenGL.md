# OpenGL 纹理 (Textures) 知识要点与代码实例总结

根据 [LearnOpenGL CN - 纹理](https://learnopengl-cn.github.io/01%20Getting%20started/06%20Textures/) 整理。

### 1. 纹理坐标 (Texture Coordinates)
*   **范围**：纹理坐标在 x 和 y 轴（通常称为 s 和 t 轴）上，范围是从 **0.0 到 1.0**。
*   **原点**：(0, 0) 位于纹理图像的**左下角**，(1, 1) 位于**右上角**。
*   **采样 (Sampling)**：使用纹理坐标获取纹理颜色的过程。
*   **插值**：顶点着色器接收纹理坐标并传给片段着色器，片段着色器会对这些坐标进行插值，从而确定每个像素该采样的位置。

**顶点数据示例：**
```cpp
float vertices[] = {
//     ---- 位置 ----       ---- 颜色 ----     - 纹理坐标 -
     0.5f,  0.5f, 0.0f,   1.0f, 0.0f, 0.0f,   1.0f, 1.0f,   // 右上
     0.5f, -0.5f, 0.0f,   0.0f, 1.0f, 0.0f,   1.0f, 0.0f,   // 右下
    -0.5f, -0.5f, 0.0f,   0.0f, 0.0f, 1.0f,   0.0f, 0.0f,   // 左下
    -0.5f,  0.5f, 0.0f,   1.0f, 1.0f, 0.0f,   0.0f, 1.0f    // 左上
};
```

---

### 2. 纹理环绕方式 (Texture Wrapping)
当纹理坐标超出 [0, 1] 范围时，OpenGL 定义了不同的行为：
*   **GL_REPEAT**：默认行为，重复纹理图像。
*   **GL_MIRRORED_REPEAT**：重复图像，但每次重复时会进行镜像。
*   **GL_CLAMP_TO_EDGE**：坐标被约束在 0 到 1 之间，超出的部分会重复边缘像素，产生拉伸效果。
*   **GL_CLAMP_TO_BORDER**：超出的部分显示用户指定的边缘颜色。

**设置代码：**
```cpp
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_S, GL_MIRRORED_REPEAT);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_T, GL_MIRRORED_REPEAT);

// 如果使用 GL_CLAMP_TO_BORDER，还需要指定颜色
float borderColor[] = { 1.0f, 1.0f, 0.0f, 1.0f };
glTexParameterfv(GL_TEXTURE_2D, GL_TEXTURE_BORDER_COLOR, borderColor);
```

---

### 3. 纹理过滤 (Texture Filtering)
决定如何将纹理像素 (Texel) 映射到纹理坐标，主要有两种方式：
*   **GL_NEAREST (邻近过滤)**：选择中心点最接近纹理坐标的像素。产生颗粒感（像素化），适合 8-bit 风格。
*   **GL_LINEAR (线性过滤)**：基于邻近像素计算插值。产生更平滑的效果。
*   **应用场景**：可分别为放大 (Magnify, `GL_TEXTURE_MAG_FILTER`) 和缩小 (Minify, `GL_TEXTURE_MIN_FILTER`) 设置过滤方式。

**设置代码：**
```cpp
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_NEAREST);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_LINEAR);
```

---

### 4. 多级渐远纹理 (Mipmaps)
*   **原理**：生成一系列纹理图像，后一个是前一个的二分之一大小。
*   **目的**：解决远处物体因分辨率过高产生的走样（Aliasing）问题，并提高性能。
*   **生成**：调用 `glGenerateMipmap(GL_TEXTURE_2D)` 自动生成。
*   **过滤选项**：在切换 Mipmap 级别时，可以使用特殊的过滤方式，这些选项**仅适用于缩小过滤**。
    *   `GL_NEAREST_MIPMAP_NEAREST`
    *   `GL_LINEAR_MIPMAP_NEAREST`
    *   `GL_NEAREST_MIPMAP_LINEAR`
    *   `GL_LINEAR_MIPMAP_LINEAR`

**设置代码：**
```cpp
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_LINEAR_MIPMAP_LINEAR);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_LINEAR);
```

---

### 5. 纹理加载与创建
*   **加载库**：常用 `stb_image.h` 单头文件库。
*   **翻转 Y 轴**：由于 OpenGL 要求 y 轴 0.0 在底部，加载前需调用 `stbi_set_flip_vertically_on_load(true)`。

**完整创建流程代码：**
```cpp
unsigned int texture;
glGenTextures(1, &texture);
glBindTexture(GL_TEXTURE_2D, texture);

// 设置环绕和过滤
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_S, GL_REPEAT);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_T, GL_REPEAT);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_LINEAR);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_LINEAR);

// 加载并生成纹理
int width, height, nrChannels;
stbi_set_flip_vertically_on_load(true); 
unsigned char *data = stbi_load("container.jpg", &width, &height, &nrChannels, 0);
if (data)
{
    glTexImage2D(GL_TEXTURE_2D, 0, GL_RGB, width, height, 0, GL_RGB, GL_UNSIGNED_BYTE, data);
    glGenerateMipmap(GL_TEXTURE_2D);
}
stbi_image_free(data);
```

---

### 6. 纹理单元 (Texture Units)
*   **目的**：允许在着色器中同时使用多个纹理。
*   **默认值**：默认激活纹理单元 0 (`GL_TEXTURE0`)。
*   **使用方法**：激活单元 -> 绑定纹理 -> 设置着色器 Uniform。

**着色器代码：**
```glsl
// 片段着色器
uniform sampler2D texture1;
uniform sampler2D texture2;

void main()
{
    // 混合两个纹理
    FragColor = mix(texture(texture1, TexCoord), texture(texture2, TexCoord), 0.2);
}
```

**渲染循环设置：**
```cpp
glActiveTexture(GL_TEXTURE0); // 在绑定纹理之前激活纹理单元
glBindTexture(GL_TEXTURE_2D, texture1);
glActiveTexture(GL_TEXTURE1);
glBindTexture(GL_TEXTURE_2D, texture2);

ourShader.use(); 
glUniform1i(glGetUniformLocation(ourShader.ID, "texture1"), 0); 
ourShader.setInt("texture2", 1); // 或者使用封装好的函数
```
