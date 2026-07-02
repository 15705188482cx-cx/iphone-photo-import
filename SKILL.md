# iPhone 照片导入 Skill

> skill_name: iphone-photo-import
> description: 把 Android 手机/iPhone 备份/微信/截屏的照片视频转换成 iPhone 能正确识别 GPS/时间/设备名/颜色的格式
> version: 3.0
> last_updated: 2026-07-02

---

## 核心结论

| 格式 | 问题 | 解决方案 |
|------|------|---------|
| **JPG** | iOS 26 不认 HONOR 私有 EXIF (MakerNote + APP6-APP10) | pyexiv2 删私有段，保留 GPS/DateTime/Make/Model/ICC |
| **MP4 (有 moov-meta)** | iOS 26 需 com.apple.quicktime.* 7 个 key | V13 字节级 patch 注入 |
| **MP4 (无 moov-meta, 微信 share_)** | moov box 没 meta 段 | ffmpeg `-c copy` remux + `-metadata creation_time=...` |
| **PNG** | iOS 26 也不读 PNG EXIF | PIL PngInfo tEXt chunk 写 Creation Time |
| **JPG (文件名可推时间)** | EXIF 无时间但文件名有时间戳 | pyexiv2 注入 DateTimeOriginal + DateTime |
| **JPG (无解, 老照片)** | 文件名和 EXIF 都没时间 | 2025-12-30 兜底 + mtime 时分秒 |
| **HEIC** | iOS 26 渲染 bug, 刚导入变黑白 | **禁止转换 HEIC** |

## 工具路径

```
Python:  C:\Users\ChenDebiao\AppData\Local\Python\pythoncore-3.14-64\python.exe
ffmpeg:  E:\ffmpeg\ffmpeg-8.1.1-full_build\bin\
pyexiv2: C:\Users\ChenDebiao\AppData\Local\Python\pythoncore-3.14-64\Lib\site-packages\pyexiv2\lib
work:    C:\Users\ChenDebiao\Documents\Codex\2026-06-29\ni\work\
```

## 文件名时间戳模式（27 种，按优先级匹配）

```python
TS_PATTERNS = [
    # iOS 录屏分享
    (r"^RPReplay_Final(\d{10})$", "rprelay", "unix_s"),                    # 10位Unix秒
    # 微信类
    (r"^mmexport(\d{13})", "mmexport_ms", "ms"),                          # 13位毫秒
    (r"^mmexport[a-f0-9]+_(\d{13})", "mmexport_hash_ms", "ms"),
    (r"^microMsg_(\d{10,13})", "micromsg", "ms"),
    (r"^wx_camera_(\d{13})", "wx_camera_ms", "ms"),
    (r"^微信图片_(\d{14})", "wx_img", "compact"),                          # 14位紧凑
    (r"^share_[a-f0-9]+?(\d{13})(?:\.|$)", "share_hash_ms", "ms"),
    (r"^share_app_(\d{8})$", "share_app_date", "ymd"),                     # share_app_YYYYMMDD
    (r"^Image_(\d{13})", "image_ms", "ms"),                                # QQ浏览器
    # 安卓截图
    (r"^.+_screenshot_(\d{4})-(\d{2})-(\d{2})-(\d{2})-(\d{2})-(\d{2})", "ss_android_prefix", "ymd_hms"),
    (r"^screenshot_(\d{4})-(\d{2})-(\d{2})-(\d{2})-(\d{2})-(\d{2})", "ss_android", "ymd_hms"),
    # 相机标准
    (r"^IMG_(\d{8})_(\d{6})[A-Z]?$", "img_ymdhms", "ymd_hms"),
    (r"^SVID_(\d{8})_(\d{6})_?\d*$", "svid", "ymd_hms"),
    (r"^Screenshot_(\d{8})_(\d{6})", "ss_ymdhms", "ymd_hms"),
    (r"^Screenshot_(\d{4})-(\d{2})-(\d{2})-(\d{2})-(\d{2})-(\d{2})", "ss_y_m_d", "ymd_hms"),
    (r"^(\d{4})-(\d{2})-(\d{2})\s+\d+$", "date_seq", "ymd_only"),          # 2022-02-08 47894.jpg
    # 小红书/淘宝/美图
    (r"^Camera_XHS_(\d{13})", "xhs_ms", "ms"),
    (r"^IMG_MIRROR_(\d{8})_(\d{6})$", "img_mirror_ymdhms", "ymdhms"),     # 大小写不敏感
    (r"^IMG_MIRROR_(\d{8})_(\d{6})_\d+$", "img_mirror_ymdhms_n", "ymdhms"), # _1 后缀
    (r"^beautybox_(\d{14})", "beautybox", "compact"),                       # 大小写不敏感, 美图秀秀
    (r"^null-(\d{13})$", "null_ms", "ms"),                                 # null-<ms> 前缀
    # 数字类
    (r"^(\d{13})(?:_|$)", "ms_ts", "ms"),
    (r"^(\d{8})_(\d{6})$", "ymdhms_plain", "ymdhms"),
    (r"^(\d{8})_(\d{6})_\d+$", "ymdhms_3digit", "ymdhms"),                 # 20150820_135327_044
    (r"^(\d{10})\(\d+\)$", "unix_10_paren", "unix_s"),                     # 1531341145(2).jpg
    (r"^(\d{13})--\d+$", "ms_double_dash", "ms"),
    (r"^(\d{10})$", "unix_s_10", "unix_s"),                                # 1454537310.jpg
    (r"^(\d{4})\.(\d{2})\.(\d{2})_(\d{2})\.(\d{2})\.(\d{2})$", "ymd_dot", "ymd_hms"),  # 2024.06.24_08.23.50
]
```

## 注入方法

### JPG/JPEG 注入 (pyexiv2)

```python
import pyexiv2, tempfile, shutil, os
# 中文路径 -> ASCII temp
fd, tmp = tempfile.mkstemp(suffix=".jpg", dir=r"C:\Windows\Temp")
os.close(fd)
shutil.copy2(p, tmp)
img = pyexiv2.Image(tmp)
exif = img.read_exif()
exif["Exif.Image.DateTime"] = "2026:07:01 10:43:10"
exif["Exif.Photo.DateTimeOriginal"] = "2026:07:01 10:43:10"
exif["Exif.Photo.DateTimeDigitized"] = "2026:07:01 10:43:10"
img.modify_exif(exif)
img.close()
shutil.copy2(tmp, p)  # 关键: copy 回原文件
os.unlink(tmp)
```

**注意**：tag namespace 是 `Exif.Photo.*`（不是 `Exif.Image.*`）。

### MP4 注入 (V13 / ffmpeg)

**Path 1 - 有 moov-meta (V13)**:
- 字节级 patch moov box + 7 个 com.apple.quicktime.* key
- 必须同步更新 stco/co64 chunk offset
- 详见 `mp4_fix_v13.py`

**Path 2 - 无 moov-meta (ffmpeg fallback)**:
```python
import subprocess
cmd = [
    FFMPEG, "-y", "-loglevel", "error",
    "-i", str(p), "-c", "copy",
    "-metadata", f"creation_time={iso_time}",
    "-metadata", f"com.apple.quicktime.creationdate={iso_time}",
    "-movflags", "use_metadata_tags", tmp,
]
```

### PNG 注入 (PIL PngInfo tEXt chunk)

```python
from PIL import Image, PngImagePlugin
img = Image.open(p)
info = PngImagePlugin.PngInfo()
info.add_text("Creation Time", "2025-04-05 10:43:10")
img.save(p, "PNG", pnginfo=info, icc_profile=img.info.get("icc_profile"))
```

### GIF 注入 (ffmpeg - 已知限制)

```python
# ffmpeg 写入 creation_time 但 GIF 格式不读 metadata
# iPhone 不会显示 GIF 的拍摄时间, 等于无效果
```

### 损坏 EXIF 修复 (lqpic.jpg 之类)

```python
from PIL import Image
img = Image.open(p).convert("RGB")
img.save(tmp, "JPEG", quality=95, icc_profile=img.info.get("icc_profile"))
img.close()
# 然后再用 pyexiv2 注入时间
```

## 注入策略（mtime 注入前必须确认）

| 场景 | 策略 |
|------|------|
| 文件名可推时间 | 用文件名时间 |
| mtime 离群 (e.g. 2025-04-25) | **必须用户确认后**用 mtime |
| mtime 是复制时间 (2026-07-01 11:xx) | 用 2025-12-30 + mtime 时分秒 兜底 |
| 完全无解 | 跳过 / 移到 `_unfixable/` |

**mtime 离群判断**：mtime 不在批处理时间窗（2026-07-01 上午 10:43 / 11:03 / 12:31）就算离群。

## 微信文件夹特殊处理

| 文件名 | 模式 | 修复方法 |
|--------|------|---------|
| `mmexport<13位ms>.jpg` | mmexport_ms | 注入 ms 转换的时间 |
| `wx_camera_<13位ms>.jpg` | wx_camera_ms | 同上 |
| `share_<hash>.png/jpg` | share_hash_ms | **mtime 是复制时间, 用 2025-04-05 + mtime 时分秒** |
| `share_app_YYYYMMDD.jpeg` | share_app_date | 注入日期 |
| `涂鸦_screenshot_<...>_微信.png` | ss_android_prefix | 注入时间 |
| `mm_facetoface_collect_qrcode_*.png` | mtime 真实 (e.g. 2025-04-25) | **必须确认**用 mtime |
| `lqpic.jpg` (损坏 EXIF) | 无 | PIL 重保存 + pyexiv2 |
| `陈鑫.jpg` / `陈鑫证件照.jpg` | 中文命名 | 用 mtime 注入"导入时间" |

## 分类合并

| 来源 | 归类 | 子文件夹 |
|------|------|---------|
| 相机拍摄 (Camera) | 相机拍摄 | 按设备名分子文件夹 |
| Screenshots/ | 截图 | 单独 |
| WeiXin/, 微信图, mmexport, share_ | 微信 | 单独 |
| 自拍 (自拍) | 相机拍摄 | 同相机 |
| 截图 (Screenshots) | 截图 | 单独 |

## 设备检测 (EXIF Make/Model)

常见设备字段示例:
- OnePlus 12 (一加 12)
- OPPO Reno 10x Zoom
- Smartisan U3 Pro (锤子)
- SONY ILCE-7M4 (A7M4 相机)
- Apple iPhone 11
- samsung Galaxy S23 Ultra
- HONOR BVL-AN20 (荣耀 Magic6 至臻版)
- Xiaomi 22041211AC
- PJD110 (一加 11)
- NIKON D800 / COOLPIX S8000
- HUAWEI MHA-AL00

## 去重 (基于 MD5)

```python
import hashlib
def md5(p):
    h = hashlib.md5()
    with open(p, "rb") as f:
        h.update(f.read())
    return h.hexdigest()
```

## Windows 注意事项

- **pyexiv2 不支持中文路径** → 复制到 `C:\Windows\Temp` 临时处理
- **必须设置** `EXIV2_DLL_PATH` 指向 pyexiv2 的 DLL 目录
- pyexiv2 写入的 EXIF tag namespace 是 `Exif.Photo.*`（不是 `Exif.Image.*`）
- iOS 跳过视频 < 1 秒 → 必要时 ffmpeg `tpad=stop_duration=0.8` 延长

## 禁止

- ❌ 不转 HEIC（iOS 26 渲染 bug）
- ❌ 不用 ffmpeg remux 处理有 moov-meta 的 MP4（V13 才能保 namespace）
- ❌ 不人为修改元数据（除时间戳从文件名推 / mtime 离群确认后用）
- ❌ 不用 mtime 注入复制时间窗的 share_<hash> 系列（mtime 是复制时间）
