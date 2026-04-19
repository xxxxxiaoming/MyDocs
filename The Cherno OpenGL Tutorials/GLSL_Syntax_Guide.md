# GLSL 基本语法与数据类型指南

GLSL (OpenGL Shading Language) 是一种类 C 语言，专门用于编写在 GPU 上运行的着色器程序。

## 1. 基础结构与版本
每个 GLSL 程序都必须以版本声明开头，随后是变量声明和 `main` 函数。

```glsl
#version 330 core

layout (location = 0) in vec3 aPos; // 顶点属性输入
out vec4 vertexColor;              // 传给片段着色器的输出

uniform mat4 model;                // 全局变量

void main() {
    gl_Position = model * vec4(aPos, 1.0);
    vertexColor = vec4(0.5, 0.0, 0.0, 1.0);
}
```

## 2. 基本数据类型

### 标量 (Scalars)
- `float`: 32 位浮点数
- `int`: 32 位有符号整数
- `uint`: 32 位无符号整数
- `bool`: 布尔值 (`true`/`false`)

### 向量 (Vectors)
向量是 GLSL 中最常用的类型，可包含 2、3 或 4 个分量。
- `vecn`: n 个浮点数分量 (默认)
- `ivecn`: n 个整数分量
- `uvecn`: n 个无符号整数分量
- `bvecn`: n 个布尔值分量

### 矩阵 (Matrices)
- `mat2`, `mat3`, `mat4`: 浮点矩阵（列主序）。
- 示例：`mat4` 表示 4x4 矩阵。

### 采样器 (Samplers)
- `sampler2D`: 用于访问 2D 纹理。
- `samplerCube`: 用于访问立方体贴图。

## 3. 分量访问 (Swizzling)
可以使用 `.x, .y, .z, .w` 等访问向量分量，且支持任意组合：
```glsl
vec4 v4 = vec4(1.0, 2.0, 3.0, 4.0);
vec2 v2 = v4.xy;     // vec2(1.0, 2.0)
vec3 v3 = v4.zzy;    // vec3(3.0, 3.0, 2.0)
v4.w = 5.0;          // 修改单个分量
```
- 位置：`x, y, z, w`
- 颜色：`r, g, b, a`
- 纹理：`s, t, p, q`

## 4. 类型限定符 (Qualifiers)
- **`in`**: 输入变量。
- **`out`**: 输出变量。
- **`uniform`**: 由 CPU 传递，在所有着色器中全局唯一且在绘制调用中不变。
- **`const`**: 编译时常量。
- **`layout (location = n)`**: 指定顶点属性的索引位置。

## 5. 控制流
与 C 语言一致：
- `if (condition) { ... } else { ... }`
- `for (int i = 0; i < 10; i++) { ... }`
- `while (condition) { ... }`
- **`discard`**: (仅片段着色器) 立即终止该片段的处理，不输出颜色到缓冲区。

## 6. 内置常用函数
- **数学**: `sin()`, `cos()`, `pow()`, `sqrt()`, `abs()`, `clamp(x, min, max)`, `mix(a, b, t)` (线性插值)
- **几何**: `length(v)`, `distance(p1, p2)`, `dot(a, b)`, `cross(a, b)`, `normalize(v)`, `reflect(I, N)`
- **纹理**: `texture(sampler2D, texCoord)`
