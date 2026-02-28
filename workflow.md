以下是为你整理的完整经验总结，涵盖了从获取 FFmpeg 到转换验证的全流程，方便你下次复用。

---

## 🧩 透明底视频转换全流程经验总结（MOV → WebM）

### 一、获取 FFmpeg（避开 Homebrew 的复杂安装）

**推荐直接从官网下载静态二进制文件**，免安装、免依赖、免编译。

1. **下载地址**  
   - 推荐来源：[https://evermeet.cx/ffmpeg/](https://evermeet.cx/ffmpeg/)（macOS 专用，持续更新）  
   - 或 [https://www.gyan.dev/ffmpeg/builds/](https://www.gyan.dev/ffmpeg/builds/)（Windows/Linux 也可用）

2. **安装步骤**  
   ```bash
   # 将下载的 ffmpeg 文件放到系统路径
   sudo mv ~/Downloads/ffmpeg /usr/local/bin/
   # 赋予执行权限
   sudo chmod +x /usr/local/bin/ffmpeg
   ```

3. **处理 macOS 安全提示**  
   如果运行时报错“无法打开，因为无法验证开发者”，执行以下命令移除隔离属性：
   ```bash
   sudo xattr -rd com.apple.quarantine /usr/local/bin/ffmpeg
   ```

4. **验证安装**  
   ```bash
   ffmpeg -version
   # 应显示版本信息，并包含 --enable-libvpx
   ```

---

### 二、转换命令模版

#### 1. 单个文件转换
```bash
ffmpeg -i "/完整路径/输入.mov" \
       -c:v libvpx-vp9 \
       -pix_fmt yuva420p \
       -crf 30 \
       -b:v 4M \
       -c:a libopus \
       -b:a 128k \
       "/完整路径/输出.webm"
```
**参数说明**  
- `-c:v libvpx-vp9`：使用 VP9 编码器（支持透明通道）  
- `-pix_fmt yuva420p`：强制输出带 Alpha 的像素格式（关键！）  
- `-crf 30`：质量系数（0~63，越小质量越高，30 为平衡点）  
- `-b:v 4M`：视频码率上限（可根据源文件大小调整）  
- `-c:a libopus -b:a 128k`：音频采用 Opus 编码，码率 128kbps  

> **注意**：如果文件名含空格或特殊字符，必须用双引号 `" "` 括起来。

#### 2. 批量转换当前文件夹所有 `.mov`
```bash
for file in *.mov; do
  ffmpeg -i "$file" -c:v libvpx-vp9 -pix_fmt yuva420p -crf 30 -b:v 4M -c:a libopus -b:a 128k "${file%.mov}.webm"
done
```
- 输出文件与原文件在同一目录，文件名相同（仅扩展名改为 `.webm`）。

#### 3. 批量转换指定文件夹并输出到另一文件夹
```bash
# 创建输出文件夹（如果不存在）
mkdir -p "/目标文件夹路径"

# 遍历源文件夹中的 .mov
for file in "/源文件夹路径"/*.mov; do
    [ -e "$file" ] || continue   # 避免无文件时报错
    filename=$(basename "$file" .mov)
    ffmpeg -i "$file" -c:v libvpx-vp9 -pix_fmt yuva420p -crf 30 -b:v 4M -c:a libopus -b:a 128k \
           "/目标文件夹路径/${filename}.webm"
done
```
- 支持带中文、括号、空格的路径（已用双引号处理）。

---

### 三、验证转换结果（关键！）

#### 1. 检查源文件是否包含 Alpha 通道
```bash
ffprobe -v error -select_streams v:0 -show_entries stream=pix_fmt "输入文件.mov"
```
- 期望输出：`pix_fmt=yuva420p` 或 `rgba`（带 `a` 表示有透明）。

#### 2. 检查输出文件是否包含 Alpha
```bash
ffprobe -v error -select_streams v:0 -show_entries stream=pix_fmt "输出文件.webm"
```
- 期望输出：`pix_fmt=yuva420p`。

#### 3. 正确播放测试（非常重要！）
- **❌ 不要用** QuickTime Player、Windows 媒体播放器、Safari 浏览器 —— 它们不支持 WebM 透明，会显示黑底。
- **✅ 必须用** Chrome、Firefox、Edge 浏览器打开 `.webm` 文件（直接将文件拖入浏览器窗口）。如果背景透明（通常显示为棋盘格或透出浏览器背景），说明转换成功。

---

### 四、常见问题与避坑指南

1. **源文件编码必须是 FFmpeg 能读取 Alpha 的格式**  
   - ✅ **可用**：RLE（Animation）、ProRes 4444、未压缩的 RGBA 等。  
   - ❌ **不可用**：HEVC with Alpha（苹果专用格式，FFmpeg 无法直接转换其透明通道）。  
   - **经验**：若源文件是 HEVC Alpha，可先用 QuickTime 或 Compressor 导出为 RLE 格式（文件变大但兼容），再作为中转进行 FFmpeg 转换。

2. **文件名与路径**  
   - 始终用双引号包裹含空格、中文、括号的路径。  
   - 使用绝对路径可避免当前目录的干扰。

3. **性能提示**  
   - VP9 编码较慢，大文件或批量转换需耐心等待。可考虑先用 `-crf 35` 提高压缩速度（质量略降）。

4. **权限问题**  
   - 若移动 ffmpeg 时提示权限不足，使用 `sudo`。  
   - 若运行时报“权限被拒绝”，执行 `sudo chmod +x /usr/local/bin/ffmpeg`。

5. **批量转换中断**  
   - 如果中途按 `Ctrl+C` 停止，已完成的文件会保留，下次可重新运行命令，跳过已完成文件。

---

### 五、总结：一套完整的复用脚本示例

假设源文件夹在桌面 `素材-横版`，输出到 `成品-横版`：
```bash
# 1. 确保 FFmpeg 可用
ffmpeg -version || { echo "请先安装 FFmpeg"; exit 1; }

# 2. 设置路径
SRC="/Users/$(whoami)/Desktop/素材-横版"
DST="/Users/$(whoami)/Desktop/成品-横版"
mkdir -p "$DST"

# 3. 批量转换
for file in "$SRC"/*.mov; do
    [ -e "$file" ] || continue
    fname=$(basename "$file" .mov)
    echo "正在转换：$fname"
    ffmpeg -i "$file" -c:v libvpx-vp9 -pix_fmt yuva420p -crf 30 -b:v 4M -c:a libopus -b:a 128k "$DST/$fname.webm"
done

echo "所有转换完成！请用 Chrome 检查透明效果。"
```

下次需要转换时，只需修改 `SRC` 和 `DST` 路径即可复用。

---

希望这份总结能成为你日后处理透明视频的得力工具！如果有新需求或遇到新问题，随时可以调整参数或寻求帮助。
