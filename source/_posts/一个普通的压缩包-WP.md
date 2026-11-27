---
title: 一个普通的压缩包 WP
date: 2026-11-27 09:35:14
tags:
---

# Writeup｜Bugku《一个普通的压缩包》
## 一、题目概述

下载得到一个压缩包，解压后是一个损坏的 RAR 文件。RAR 内部含有一个干扰文本 `flag.txt` 和一个名为 `secret.png` 的文件；该文件后缀虽为 PNG，实际却是一个 **GIF89a 双帧动画**，二维码被拆分隐藏在两帧图像的红色通道最低有效位（LSB / Red plane 0）中。

整体解题链路：

```
外层 ZIP
└── 损坏的 RAR（HEAD_TYPE 字节被篡改）
    ├── flag.txt     干扰文件
    └── secret.png   伪装后缀，本质为双帧 GIF
        ├── 帧 1：R 通道 LSB 藏二维码一部分（含右上角定位块）
        └── 帧 2：R 通道 LSB 藏二维码另一部分（含左下角定位块）
```

## 二、解题步骤

### 1. 修复 RAR 文件头，解压提取文件

用十六进制编辑器（010 Editor / HxD）打开 RAR 文件，查看文件头结构。RAR5/RAR4 块头中 `HEAD_TYPE` 字段被人为改成了错误值 `0x7A`，标准文件头类型应为 **`0x74`（FILE 块）**。

修改前：

```
65 A8 3C 7A 20 90 2F 00 ...
```

修改后：

```
65 A8 3C 74 20 90 2F 00 ...
```

> 注意：修改 `HEAD_TYPE` 后，头部 CRC（`A8 3C`）会与内容不匹配，WinRAR 可能报损坏。**使用 7-Zip 打开压缩包即可无视 CRC 校验直接提取**；命令行可用 `unrar x -kb flag.rar`，`-kb` 参数表示遇到坏块继续保留提取。

提取得到 `flag.txt`（内容为干扰提示，不含 flag）和 `secret.png`。

### 2. 识别真实文件类型

查看 `secret.png` 文件头，实际魔数为 `GIF89a`，即后缀被伪装。将其重命名为 `secret.gif`，再进行图像处理分析。

### 3. 分离 GIF 帧

该 GIF 共 2 帧，使用 Python Pillow 逐帧导出：

```python
from PIL import Image, ImageSequence
import os

INPUT_GIF = "secret.gif"
OUT_DIR = "out"

os.makedirs(OUT_DIR, exist_ok=True)

def extract_red_plane0(img_rgb: Image.Image) -> Image.Image:
    r, g, b = img_rgb.split()
    # 逐像素取最低bit
    data = [255 if (px & 1) else 0 for px in r.getdata()]
    plane0 = Image.new("L", r.size)
    plane0.putdata(data)
    return plane0

with Image.open(INPUT_GIF) as im:
    for idx, frame in enumerate(ImageSequence.Iterator(im)):
        frame_rgb = frame.convert("RGB")
        raw_path = os.path.join(OUT_DIR, f"frame_{idx}_raw.png")
        frame_rgb.save(raw_path)

        p0 = extract_red_plane0(frame_rgb)
        p0_path = os.path.join(OUT_DIR, f"frame_{idx}_red_plane0.png")
        p0.save(p0_path)
        print(f"[+] 帧{idx} 已输出：{raw_path} | {p0_path}")

print("\n✅ 全部完成！去 out 文件夹拿图")
```

也可使用 ImageMagick（Kali / WSL 自带或 `apt install imagemagick`），`-coalesce` 参数用于把增量帧展开为完整独立画面：

```bash
magick secret.gif -coalesce frame_%d.png
```

> 也可用 StegSolve：`Analyse → Frame Browser` 切换帧，主界面切换到 Red plane 0 查看。注意 StegSolve 老版本保存 1bit 图时会报 `can not be encoded with 2 bits per pixel`，可直接截图，或改用上面的脚本导出。

### 5. 两张半码按位或合并

两张图黑色像素互补：同一位置任意一张为黑，合并结果即为黑。**不是上下拼接成竖长条**，而是在同尺寸画布上做像素或运算：

```python
from PIL import Image

im0 = Image.open("frame_0_red_plane0.png").convert("1")
im1 = Image.open("frame_1_red_plane0.png").convert("1")

w, h = im0.size
full = Image.new("1", (w, h), 1)

for y in range(h):
    for x in range(w):
        v0 = im0.getpixel((x, y))
        v1 = im1.getpixel((x, y))
        full.putpixel((x, y), 0 if v0 == 0 or v1 == 0 else 1)

full.save("qrcode_or_merged.png")
print("✅ 已生成：qrcode_or_merged.png")
```

### 6. 降噪去除孤立噪点

合并后图片存在零星白色雪花点，会干扰识别。将四周全为黑色的孤立白点涂黑：

```python
# 简单降噪：单个孤立白点（周围都是黑）涂黑，去掉杂雪花
px = out.load()
for y in range(1, h-1):
    for x in range(1, w-1):
        if px[x,y] == 255:
            neighbors = [px[x-1,y], px[x+1,y], px[x,y-1], px[x,y+1]]
            if all(n == 0 for n in neighbors):
                px[x,y] = 0

out.save("final_clean_qr.png")
```

### 7. 补齐左上角 QR 定位块

标准 QR 码需要 3 个 Finder Pattern（定位方块），原图缺少左上角一个。**补画的定位块尺寸必须与另外两个原生定位块一致**（本题为 21×21），否则二维码校验失败：

```python
size = 21
finder = []
for y in range(size):
    for x in range(size):
        if y == 0 or y == size - 1 or x == 0 or x == size - 1:
            finder.append(0)
        elif 4 <= y <= 16 and 4 <= x <= 16:
            if y == 4 or y == 16 or x == 4 or x == 16:
                finder.append(255)
            elif 8 <= y <= 12 and 8 <= x <= 12:
                finder.append(0)
            else:
                finder.append(255)
        else:
            finder.append(255)

finder_img = Image.new("L", (size, size))
finder_img.putdata(finder)
out.paste(finder_img, (0, 0))
```
>也可以直接用PS工具或者是画图工具截取一下自己挪一下位置就可以了。
![alt text](image.png)

### 8. 解码二维码

将修复完成的二维码上传至草料二维码在线解码（https://cli.im/deqr），或使用微信扫码，读取得到 flag。

## 三、踩坑记录

| 序号 | 错误尝试 | 原因 / 正解 |
| --- | --- | --- |
| 1 | 直接把两张半码上下拼接，得到竖长条 | 两图画布尺寸相同，应在同画布上做像素或合并，而非纵向叠放 |
| 2 | 左上角定位块按 7×7 绘制 | 尺寸远小于原生定位块（21×21），QR 校验失败；需与另外两个定位块等大 |
| 3 | 合并后未降噪直接扫码 | 孤立白色噪点干扰识别，需先去除单点雪花 |
| 4 | 用 WinRAR 解压修复后的 RAR | CRC 不匹配会报错；改用 7-Zip 或 `unrar -kb` 强制提取 |
| 5 | 直接按 PNG 分析 `secret.png` | 文件真实格式为 GIF，需先改后缀为 `.gif`，StegSolve 的 Frame Browser 才会可用 |

## 四、涉及知识点

- RAR 文件结构：HEAD_TYPE 字段与头部 CRC 校验；
- 文件格式识别：通过魔数（`GIF89a`）判断真实类型；
- GIF 多帧分离（ImageMagick `-coalesce` / Pillow `ImageSequence`）；
- LSB 最低有效位隐写：单通道 bit 平面提取；
- QR 码结构：三个 Finder Pattern 定位方块的作用与尺寸一致性；
- 二值图像像素运算（按位或合并）与简单形态学降噪。

