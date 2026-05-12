---
title: "Perceptron 代码讲解"
date: 2026-05-13T01:20:00+08:00
draft: false
description: "讲解一个用 C 语言实现的感知机演示程序：如何生成矩形和圆形样本、训练权重矩阵，并把训练过程可视化。"
searchHidden: true
ShowToc: true
TocOpen: false
---

这份项目是一个用 C 语言写的感知机演示程序。它随机生成两类 `20x20` 的二值图像：矩形和圆形，然后训练一个单层权重矩阵去区分它们。训练过程中，每次权重发生变化都会保存成一张 PPM 图片，最后可以合成为 GIF，观察模型是如何逐步形成分类特征的。

<!--more-->

## 运行方式

```console
$ ./build.sh
$ ./main
```

如果想重新生成动画：

```console
$ ./make-gif.sh
```

`make-gif.sh` 依赖 `ffmpeg`。程序运行时会生成 `data/weights-*.ppm`，这些文件是训练过程中的权重快照。

## 文件结构

- `main.c`：核心代码，包含图像生成、模型计算、训练、评估和图片保存。
- `config.h`：配置项，例如画布尺寸、样本数量、训练轮数、随机种子和输出目录。
- `build.sh`：编译脚本。
- `make-gif.sh`：编译、运行，并把 PPM 序列合成为 GIF。
- `README.md`：项目的简短说明和启动方式。

## 核心数据结构

项目中最重要的数据结构是 `Layer`：

```c
typedef float Layer[HEIGHT][WIDTH];
```

它是一个二维浮点数组，尺寸由 `config.h` 里的 `WIDTH` 和 `HEIGHT` 控制。当前配置是：

```c
#define WIDTH 20
#define HEIGHT 20
```

`Layer` 在程序里有两个用途：

- `inputs`：当前输入图像，像素值通常是 `0.0f` 或 `1.0f`。
- `weights`：模型权重，每个像素位置对应一个权重值。

可以把 `weights` 理解成一张“模板图”。输入图像覆盖到正权重区域时，模型输出会变大；覆盖到负权重区域时，模型输出会变小。

## 图像生成

程序会生成两类训练样本。

### 矩形

`layer_random_rect()` 会先清空整张图，然后随机选择矩形的位置和大小，再调用 `layer_fill_rect()` 把矩形区域填成 `1.0f`。

### 圆形

`layer_random_circle()` 也会先清空整张图，然后随机选择圆心和半径，再调用 `layer_fill_circle()` 把圆内像素填成 `1.0f`。

圆形判断使用的是标准公式：

```c
dx*dx + dy*dy <= r*r
```

也就是某个点到圆心的距离不超过半径时，这个点就在圆内。

## 模型计算

模型的前向计算在 `feed_forward()` 中完成：

```c
output += inputs[y][x] * weights[y][x];
```

它会遍历整张 `20x20` 图，把输入像素和对应位置的权重相乘，然后全部加起来。

这个输出值会和 `BIAS` 比较：

```c
#define BIAS 20.0
```

当前代码里的分类规则可以理解为：

- 输出大于等于 `BIAS`：更像圆形。
- 输出小于等于 `BIAS`：更像矩形。

## 训练规则

训练逻辑在 `train_pass()` 里。

每轮训练会生成 `SAMPLE_SIZE` 组矩形和圆形。当前配置是：

```c
#define SAMPLE_SIZE 75
```

### 矩形被误判

矩形应该让输出低于或等于 `BIAS`。如果矩形的输出大于 `BIAS`，说明模型把它判得太像圆形了：

```c
if (feed_forward(inputs, weights) > BIAS) {
    sub_inputs_from_weights(inputs, weights);
}
```

于是程序会从权重中减去这个矩形：

```text
weights -= inputs
```

这样下次遇到类似矩形时，这些区域的权重会更低，输出也会降低。

### 圆形被误判

圆形应该让输出大于或等于 `BIAS`。如果圆形的输出小于 `BIAS`，说明模型没有把它识别成圆形：

```c
if (feed_forward(inputs, weights) < BIAS) {
    add_inputs_from_weights(inputs, weights);
}
```

于是程序会把这个圆形加到权重中：

```text
weights += inputs
```

这样下次遇到类似圆形时，这些区域的权重会更高，输出也会提高。

## 权重可视化

每当模型权重发生调整，程序都会调用 `layer_save_as_ppm()` 保存一张图片：

```c
layer_save_as_ppm(weights, file_path);
```

PPM 图片中的颜色来自权重值：

- 权重越低，颜色越偏浅。
- 权重越高，颜色越偏蓝。

`PPM_SCALER` 用来放大每个像素，方便肉眼查看：

```c
#define PPM_SCALER 25
```

原始模型只有 `20x20`，放大后输出图片会变成 `500x500`。

## 程序入口

`main()` 的流程如下：

1. 创建 `data/` 输出目录。
2. 使用 `CHECK_SEED` 生成检查样本，测试未训练模型的失败率。
3. 进入训练循环，最多训练 `TRAIN_PASSES` 轮。
4. 每轮调用 `train_pass()`。
5. 如果某一轮没有发生任何权重调整，说明模型已经在当前训练样本上收敛，提前停止。
6. 再次使用 `CHECK_SEED` 检查训练后的失败率。

核心循环是：

```c
for (int i = 0; i < TRAIN_PASSES; ++i) {
    srand(TRAIN_SEED);
    int adj = train_pass(inputs, weights);
    printf("[INFO] Pass %d: adjusted %d times\n", i, adj);
    if (adj <= 0) break;
}
```

`adjusted` 表示这一轮中权重被调整了多少次。这个数字逐渐变小，说明模型在训练样本上的错误越来越少。

## 需要注意的地方

### 每轮训练使用相同随机种子

训练循环里每次都会调用：

```c
srand(TRAIN_SEED);
```

这意味着每一轮训练看到的随机样本序列都是一样的。因此模型可能只是在固定训练样本上收敛，不一定能很好地泛化到新的随机样本。

### 矩形高度疑似使用了错误变量

`layer_random_rect()` 中有一行：

```c
int h = HEIGHT - x;
```

从含义上看，高度更应该由 `y` 决定：

```c
int h = HEIGHT - y;
```

当前写法会让矩形高度受横坐标影响，生成的数据分布可能不符合预期。

### `layer_load_from_bin()` 尚未实现

代码中有保存二进制权重的函数 `layer_save_as_bin()`，但对应的加载函数还没有实现：

```c
assert(0 && "TODO: layer_load_from_bin is not implemented yet!");
```

当前主流程没有调用这个函数，所以不会影响正常运行。

### PPM 颜色映射没有限制范围

`layer_save_as_ppm()` 会把权重映射到颜色值，但没有对映射后的值做 clamp。如果权重超过配置里的显示范围，颜色可能溢出，导致可视化结果不够准确。

## 学习重点

这个项目适合用来理解以下概念：

- 二维数组如何表示简单图像。
- 感知机如何通过点积得到分类分数。
- 误分类时如何用加减输入样本更新权重。
- 随机样本、训练集、检查集和泛化之间的关系。
- 如何把模型内部状态保存成图片，用可视化辅助理解训练过程。

一句话总结：这个项目用很少的 C 代码演示了感知机的核心思想，输入是一张形状图，权重也是一张图，训练就是不断把误判样本加到或减出权重图。
