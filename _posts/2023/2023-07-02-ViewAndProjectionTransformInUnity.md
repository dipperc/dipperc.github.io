---
layout: post
title:  "Unity 中的视图变换和投影变换"
date:   2023-07-02 16:16:00 +800
category: Unity
---

- [1. 观察矩阵](#1-观察矩阵)
- [2. 正交投影和透视投影](#2-正交投影和透视投影)
  - [2.1. 正交投影](#21-正交投影)
  - [2.2. 透视投影](#22-透视投影)

## 1. 观察矩阵

在Unity中，约定使用的是 **左手坐标系** ，即 $\text{+x}$ 方向指向右方， $\text{+y}$ 方向指向上方， $\text{+z}$ 方向指向的是物体的前方。如下图所示， 相机的世界坐标为 `(0, 0, 0)` ，Cube的世界坐标为 `(0, 0, 1.5)` ：

![00_unity_worldspace_coordinates](/assets/images/2023/2023-07-02-ViewAndProjectionTransformInUnity/00_unity_worldspace_coordinates.jpeg)

而在观察空间中，Unity 使用的是右手坐标系，即： $\text{+x}$ 方向指向右方， $\text{+y}$ 方向指向上方， $\text{-z}$ 方向指向的是物体的前方，[官网文档](https://docs.unity3d.com/2021.3/Documentation/ScriptReference/Camera-worldToCameraMatrix.html)中也有说明：***Note that camera space matches OpenGL convention: camera's forward is the negative Z axis. This is different from Unity's convention, where forward is the positive Z axis.***。所以在 Unity 中，经过观察矩阵变换后， $z$ 坐标的值会被取反。看下面的例子，相机参数如下：

![01_camera_properties](/assets/images/2023/2023-07-02-ViewAndProjectionTransformInUnity/01_camera_properties.png)

相机在 `(0, 0, 0)` 的位置，不做任何旋转，此时输出的观察矩阵是：

![02_view_matrix](/assets/images/2023/2023-07-02-ViewAndProjectionTransformInUnity/02_view_matrix.png)

可以看到，观察空间的 $\text{z}$ 的方向和世界坐标系下的 $\text{z}$ 的方向是相反的。知道这个以后，在后面的投影变换中，就需要带入负的 $\text{z}$ 值来验证正确性。

如果再加上旋转和平移呢？这个时候就需要再计算旋转和平移的矩阵，需要注意的是：在世界坐标系下，旋转是按照 $\text{z-x-y}$ 的顺序旋转。

相机的基本属性如下：

![03_view_matrix_camera_properties](/assets/images/2023/2023-07-02-ViewAndProjectionTransformInUnity/03_view_matrix_camera_properties.png)

在C#中用代码验证：

```csharp
var cameraTransform = GetComponent<Camera>().transform;
Vector3 rotation = new Vector3(cameraTransform.eulerAngles.x, cameraTransform.eulerAngles.y, cameraTransform.eulerAngles.z);

// Rotate around z-axis
float cosAlpha = Mathf.Cos(Mathf.Deg2Rad * rotation.z);
float sinAlpha = Mathf.Sin(Mathf.Deg2Rad * rotation.z);
Matrix4x4 rotateZ = new Matrix4x4(
    new Vector4(cosAlpha, sinAlpha, 0, 0),
    new Vector4(-sinAlpha, cosAlpha, 0, 0),
    new Vector4(0, 0, 1, 0),
    new Vector4(0, 0, 0, 1));

// Rotate around x-axis
float cosBeta = Mathf.Cos(Mathf.Deg2Rad * rotation.x);
float sinBeta = Mathf.Sin(Mathf.Deg2Rad * rotation.x);
Matrix4x4 rotateX = new Matrix4x4(
    new Vector4(1, 0, 0, 0),
    new Vector4(0, cosBeta, sinBeta, 0),
    new Vector4(0, -sinBeta, cosBeta, 0),
    new Vector4(0, 0, 0, 1));

// Rotate around y-axis
float cosTheta = Mathf.Cos(Mathf.Deg2Rad * rotation.y);
float sinTheta = Mathf.Sin(Mathf.Deg2Rad * rotation.y);
Matrix4x4 rotateY = new Matrix4x4(
    new Vector4(cosTheta, 0, -sinTheta, 0),
    new Vector4(0, 1, 0, 0),
    new Vector4(sinTheta, 0, cosTheta, 0),
    new Vector4(0, 0, 0, 1));

// Translate matrix
Vector3 position = new Vector3(cameraTransform.position.x, cameraTransform.position.y, cameraTransform.position.z);
Matrix4x4 translate = new Matrix4x4(
    new Vector4(1, 0, 0, 0),
    new Vector4(0, 1, 0, 0),
    new Vector4(0, 0, 1, 0),
    new Vector4(position.x, position.y, position.z, 1));

// Negative z
Matrix4x4 negativeZ = new Matrix4x4(
    new Vector4(1, 0, 0, 0),
    new Vector4(0, 1, 0, 0),
    new Vector4(0, 0, -1, 0),
    new Vector4(0, 0, 0, 1));

// Local to world matrix (Multiply left)
Matrix4x4 localToWorld = translate * rotateY * rotateX * rotateZ;
// Inverse matrix and multiple negative z matrix
Matrix4x4 finalViewMatrix = negativeZ * localToWorld.inverse;

Debug.LogError(finalViewMatrix);
```

输出结果为：

![04_console_view_matrix](/assets/images/2023/2023-07-02-ViewAndProjectionTransformInUnity/04_console_view_matrix.png)

和Unity中输出的结果一致，验证正确：

![05_gpu_view_matrix](/assets/images/2023/2023-07-02-ViewAndProjectionTransformInUnity/05_gpu_view_matrix.png)

## 2. 正交投影和透视投影

下面再来看投影矩阵，先来看正交投影。

首先，先确定一下 Unity 中，正交矩阵和透视矩阵是把 `z` 值映射到 `[-1, 1]` 还是 `[0, 1]` ？其中 OpenGL 是把 `z` 映射到 `[-1, 1]` ，即 Near 近平面 `z` 为 `-1`，Far 远平面 `z` 为 `1`；而 `DirectX` ，`Vulkan` 和 `Metal` 中是把 `z` 映射到 `[0, 1]` ，Near 近平面 `z` 为 `0`，Far 远平面 `z` 为 `1`。

其中在 DirectX，Vulkan 和 Metal 下，Unity 使用了 **反向Z（Reversed-Z）技术**，也就是 Near 近平面 `z` 为 `1`，Far 远平面 `z` 为 `0`，这样可以使远离相机的位置有更好的 `z` 值精度，这样能更好的避免 **z-fighting** 的问题。可以通过 `UNITY_REVERSED_Z` 宏来判断是否有使用反向Z，此宏定义在 D3D11.hlsl、Vulkan.hlsl 等文件中。

### 2.1. 正交投影

相机参数如下：

![06_orthographic_camera_properties](/assets/images/2023/2023-07-02-ViewAndProjectionTransformInUnity/06_orthographic_camera_properties.png)

根据 Near，Far，Size 和 Aspect 可以确定正交投影矩阵，**其中需要注意的是，Size 定义的是正交投影视锥体一半的高度（Half Height）**， 所以矩阵中的计算从 $\frac{2}{\text{Size} * 2}$ 简化成 $\frac{1}{\text{Size}}$ ， $\frac{1}{\text{Aspect} * \text{Size}}$ 也是同理。 在 OpenGL 环境下(转换后 `z` 映射到 `[-1, 1]` )，矩阵如下：

$$
M_{ortho}=
\begin{bmatrix}
\frac{1}{Aspect \cdot Size} & 0 & 0 & 0\\
0 & \frac{1}{Size} & 0 & 0\\
0 & 0 & -\frac{2}{f - n} & -\frac{f + n}{f - n}\\
0 & 0 & 0 & 1
\end{bmatrix}
$$

其中 Aspect 是宽高比，可以通过

```csharp
float aspectRatio = (float)camera.pixelWidth / camera.pixelHeight;
```

计算输出得到 Aspect 为：

![07_orthographic_camera_aspect](/assets/images/2023/2023-07-02-ViewAndProjectionTransformInUnity/07_orthographic_camera_aspect.png)

根据相机参数可以知道 Size = 5，Near = 1，Far = 4，则计算出来的正交投影矩阵为：

$$
M_{ortho}=
\begin{bmatrix}
0.125026412 & 0 & 0 & 0\\
0 & 0.2 & 0 & 0\\
0 & 0 & -0.666666667 & -1.66666667\\
0 & 0 & 0 & 1
\end{bmatrix}
$$

上面的观察矩阵左乘此正交投影矩阵可以得到 VP 矩阵为：

$$
M_{vp}=
\begin{bmatrix}
0.125026412 & 0 & 0 & 0\\
0 & 0.2 & 0 & 0\\
0 & 0 & 0.666666667 & -1.66666667\\
0 & 0 & 0 & 1
\end{bmatrix}
$$

和 Unity 中输出的 VP 矩阵是一样的，公式验证正确：

![08_orthographic_view_projection_matrix_opengl](/assets/images/2023/2023-07-02-ViewAndProjectionTransformInUnity/08_orthographic_view_projection_matrix_opengl.png)

再来看非 OpenGL 环境呢？也就是上面说的把 `z` 映射到 `[0, 1]` ，而且由于反向Z，所以需要 Near 近平面 `z` 为 `1`，Far 远平面 `z` 为 `0` 。只有 `z` 是不一样的，所以矩阵需要变化的是 `m22` 和 `m23` 项，推导如下：

$$
M_{ortho}=
\begin{bmatrix}
\frac{1}{Aspect \cdot Size} & 0 & 0 & 0\\
0 & \frac{1}{Size} & 0 & 0\\
0 & 0 & A & B\\
0 & 0 & 0 & 1
\end{bmatrix}
$$

因为 `z = -n` 时，转换后的 `z = 1` ； `z = -f` 时，转换后的 `z = 0` ：

$$
-nA + B = 1,\quad
-fA + B = 0
$$

所以：

$$
B = fA,\quad
-nA + fA = 1
$$

可求得:

$$
A = \frac{1}{f-n},\quad
B = \frac{f}{f-n}
$$

求得非 OpenGL 环境，正交投影矩阵为：

$$
M_{ortho}=
\begin{bmatrix}
\frac{1}{Aspect \cdot Size} & 0 & 0 & 0\\
0 & \frac{1}{Size} & 0 & 0\\
0 & 0 & \frac{1}{f-n} & \frac{f}{f-n}\\
0 & 0 & 0 & 1
\end{bmatrix}
$$

切换到 Vulkan 环境验证一下，带入相机参数后的正交投影矩阵为：

$$
M_{ortho}=
\begin{bmatrix}
0.125026412 & 0 & 0 & 0\\
0 & 0.2 & 0 & 0\\
0 & 0 & 0.333333333 & 1.33333333\\
0 & 0 & 0 & 1
\end{bmatrix}
$$

上面的观察矩阵左乘此正交投影矩阵可以得到 VP 矩阵为：

$$
M_{vp}=
\begin{bmatrix}
0.125026412 & 0 & 0 & 0\\
0 & 0.2 & 0 & 0\\
0 & 0 & -0.333333333 & 1.33333333\\
0 & 0 & 0 & 1
\end{bmatrix}
$$

此时 Unity 中输出的矩阵为：

![09_orthographic_view_projection_matrix_not_opengl](/assets/images/2023/2023-07-02-ViewAndProjectionTransformInUnity/09_orthographic_view_projection_matrix_not_opengl.png)

发现有一点点不一样，那就是矩阵的 `m11` 项是 `-0.2`，但是上面计算出来的是 `0.2`，[这是因为在非 OpenGL 环境下，y 轴的坐标方向是向下的](https://matthewwellings.com/blog/the-new-vulkan-coordinate-system/)，而 OpenGL 环境下 y 轴是向上的，所以这里做了取反的计算。

![10_negative_y_not_opengl](/assets/images/2023/2023-07-02-ViewAndProjectionTransformInUnity/10_negative_y_not_opengl.png)

### 2.2. 透视投影

相机参数如下：

![11_perspective_camera_properties](/assets/images/2023/2023-07-02-ViewAndProjectionTransformInUnity/11_perspective_camera_properties.png)

根据 Near，Far，FOV 和 Aspect 可以确定正交投影矩阵，在OpenGL环境下(转换后 `z` 映射到 `[-1, 1]` )，矩阵如下：

$$
M_{persp}=
\begin{bmatrix}
\frac{\cot{(\frac{FOV}{2})}}{Asepct} & 0 & 0 & 0\\
0 & \cot{(\frac{FOV}{2})} & 0 & 0\\
0 & 0 & -\frac{f + n}{f - n} & -\frac{2  \cdot  n  \cdot  f}{f - n}\\
0 & 0 & -1 & 0
\end{bmatrix}
$$

其中，Aspect 是 1.600666：

![12_perspective_camera_aspect](/assets/images/2023/2023-07-02-ViewAndProjectionTransformInUnity/12_perspective_camera_aspect.png)

然后，FOV 为60度，Near 为1，Far 为4，其中， $\cot{(\frac{FOV}{2})}$ 可以通过如下代码求得：

```csharp
float HalfFovRadians = Mathf.Deg2Rad * (60.0f / 2);
float cotHalfFov = Mathf.Cos(HalfFovRadians)/Mathf.Sin(HalfFovRadians);
```

输出结果为：

![13_perspective_camera_cot_half_fov](/assets/images/2023/2023-07-02-ViewAndProjectionTransformInUnity/13_perspective_camera_cot_half_fov.png)

带入相机相关参数后可以计算出透视投影矩阵为：

$$
M_{persp}=
\begin{bmatrix}
1.08208146 & 0 & 0 & 0\\
0 & 1.732051 & 0 & 0\\
0 & 0 & -1.66666667 & -2.66666667\\
0 & 0 & -1 & 0
\end{bmatrix}
$$

观察矩阵左乘此透视投影矩阵可以得到 VP 矩阵为：

$$
M_{vp}=
\begin{bmatrix}
1.08208146 & 0 & 0 & 0\\
0 & 1.732051 & 0 & 0\\
0 & 0 & 1.66666667 & -2.66666667\\
0 & 0 & 1 & 0
\end{bmatrix}
$$

和 Unity 中输出的 VP 矩阵是一样的，公式验证正确：

![14_perspective_view_projection_matrix_opengl](/assets/images/2023/2023-07-02-ViewAndProjectionTransformInUnity/14_perspective_view_projection_matrix_opengl.png)

继续来看非 OpenGL 环境，和正交投影一样，也是变换后的 `z` 的范围不一样，而且因为反向Z的原因，Near 近平面 `z` 为 `1`，Far 远平面 `z` 为 `0`，所以也是只需要变化矩阵的 `m22` 和 `m23` 项，推导过程：

$$
\begin{bmatrix}
\frac{\cot{(\frac{FOV}{2})}}{Asepct} & 0 & 0 & 0\\
0 & \cot{(\frac{FOV}{2})} & 0 & 0\\
0 & 0 & A & B\\
0 & 0 & -1 & 0
\end{bmatrix}
\cdot
\begin{bmatrix}
x\\
y\\
z\\
1
\end{bmatrix}
$$

因为透视矩阵第4行并不是 `[0, 0, 0, 1]` ，通过矩阵变换后， `w` 项是有值的并不是 `1`，所以会多一个**透视除法**，所以条件是： `z = -n` 时，转换后的 `z = n` ，此时 `w` 分量为 $(-1) \cdot (-n) = n$ ，透视除法后 `z = 1` ； `z = -f` 时，转换后的 `z = 0` ，即：

$$
-nA + B = n,\quad
-fA + B = 0
$$

所以：

$$
B = fA,\quad
-nA + fA = n
$$

可求得:

$$
A = \frac{n}{f - n} ,\quad B = \frac{fn}{f - n}
$$

求得非 OpenGL 环境，透视投影矩阵为：

$$
M_{persp}=
\begin{bmatrix}
\frac{\cot{(\frac{FOV}{2})}}{Asepct} & 0 & 0 & 0\\
0 & \cot{(\frac{FOV}{2})} & 0 & 0\\
0 & 0 & \frac{n}{f - n} & \frac{fn}{f - n}\\
0 & 0 & -1 & 0
\end{bmatrix}
$$

切换到 Vulkan 环境验证一下，带入相机参数后的透视投影矩阵为：

$$
M_{persp}=
\begin{bmatrix}
1.08208146 & 0 & 0 & 0\\
0 & 1.732051 & 0 & 0\\
0 & 0 & 0.333333333 & 1.33333333\\
0 & 0 & -1 & 0
\end{bmatrix}
$$

上面的观察矩阵左乘此透视投影矩阵可以得到 VP 矩阵为：

$$
M_{vp}=
\begin{bmatrix}
1.08208146 & 0 & 0 & 0\\
0 & 1.732051 & 0 & 0\\
0 & 0 & -0.333333333 & 1.33333333\\
0 & 0 & 1 & 0
\end{bmatrix}
$$

此时 Unity 中输出的矩阵为：

![15_perspective_view_projection_matrix_not_opengl](/assets/images/2023/2023-07-02-ViewAndProjectionTransformInUnity/15_perspective_view_projection_matrix_not_opengl.png)

矩阵的 `m11` 项是反的这个问题上面已经解释了，这里就不再说明。

最终矩阵验证正确。

**补充一下，在没有使用反向Z时，求透视矩阵**。同理，条件为： `z = -n` 时，转换后的 `z = 0` ， `z = -f` 时，转换后的 `z = f` ，此时 `w` 分量为 $(-1) \cdot (-f) = f$ ，透视除法后 `z = 1` :

$$
-nA + B = 0,\quad
-fA + B = f
$$

可求得最终的透视矩阵为：

$$
M_{persp-non-reversed-z}=
\begin{bmatrix}
\frac{\cot{(\frac{FOV}{2})}}{Asepct} & 0 & 0 & 0\\
0 & \cot{(\frac{FOV}{2})} & 0 & 0\\
0 & 0 & \frac{f}{n - f} & \frac{nf}{n - f}\\
0 & 0 & -1 & 0
\end{bmatrix}
$$
