# iPhone Photo Import Skill

> **skill_name:** `iphone-photo-import`  
> **description:** 把 Android 手机/iPhone 备份/微信/截屏的照片视频转换成 iPhone 能正确识别 GPS/时间/设备名/颜色的格式  
> **version:** 3.0  
> **last_updated:** 2026-07-02

---

## 背景

从 Android 手机（荣耀/小米/三星/一加等）或 iPhone 备份照片传到 iPhone 后，经常遇到以下问题：

| 问题 | 现象 | 根因 |
|------|------|------|
| ❌ 地理信息丢失 | iPhone 相册不显示拍摄地点 | Android 私有 EXIF (MakerNote + APP6-APP10) 让 iOS 跳过整个 EXIF 解析 |
| ❌ 颜色异常 | 导入后变黑白，需点"编辑"才恢复 | HEIC 格式的 ICC 色彩配置在 iOS 26 上有渲染 bug |
| ❌ 拍摄时间/设备名丢失 | 显示"2026年"（导入时间）或无设备名 | MP4 的 com.apple.quicktime.* 命名空间与 Android 不兼容 |
| ❌ 老照片无时间 | iPhone 显示 1970-01-01 或无时间 | EXIF 没时间，文件名也没时间戳 |

## 解决方案总览

### 1. JPG 照片 - 清理 EXIF 私有段

不转 HEIC（iOS 26 bug），只清理 EXIF：

1. 用 `pyexiv2` 读 EXIF
2. 删除 `MakerNote` + Thumbnail（私有信息段）
3. 调用 `clear_exif()` + `modify_exif(md)` 重新写入干净的 EXIF
4. 字节级重组：跳过 APP6-APP10（0xFFE6-0xFFEA）标记段
5. 保留：GPS / DateTimeOriginal / Make / Model / ICC

> ⚠️ pyexiv2 不能直接读中文路径。通过 `subst` 虚拟盘或临时拷贝到 `C:\Windows\Temp` 解决。

### 2. MP4 视频 - 注入 7 个 com.apple.quicktime.* key

iOS 26 需要 `com.apple.quicktime.*` 命名空间的 7 个元数据 key：

- `location.accuracy.horizontal` = "20.0"
- `full-frame-rate-playback-intent`
- `location.ISO6709` (GPS)
- `make` / `model` / `software`
- `creationdate` (拍摄时间)

**两条修复路径**：

| 路径 | 适用 | 工具 |
|------|------|------|
| V13 字节级 patch | MP4 已有 moov-meta | `mp4_fix_v13.py` |
| ffmpeg remux | MP4 没有 moov-meta（微信 share_） | `-c copy -metadata creation_time=...` |

> ⚠️ 关键 Bug：moov 在文件开头时必须同步更新 `stco`/`co64` chunk offset，否则 iPhone 读到错误位置。

### 3. PNG 图片 - PIL tEXt chunk

```python
from PIL import Image, PngImagePlugin
img = Image.open(p)
info = PngImagePlugin.PngInfo()
info.add_text("Creation Time", "2025-04-05 10:43:10")
img.save(p, "PNG", pnginfo=info, icc_profile=img.info.get("icc_profile"))
```

iOS 26 会读 PNG 的 tEXt chunk 里的 `Creation Time`。

### 4. 文件名时间戳解析（27 种模式）

完整正则列表见 `SKILL.md`。常见模式：

| 模式 | 例子 | 来源 |
|------|------|------|
| `mmexport<13位ms>` | `mmexport1714530535426.jpg` | 微信 |
| `wx_camera_<13位ms>` | `wx_camera_1714530535426.jpg` | 微信 |
| `screenshot_YYYY-MM-DD-HH-MM-SS` | `screenshot_2019-03-16-10-30-23-212.png` | 安卓 |
| `IMG_YYYYMMDD_HHMMSS` | `IMG_20240507_135850.jpg` | 相机 |
| `IMG_MIRROR_YYYYMMDD_HHMMSS` | `IMG_MIRROR_20250302_031125.jpg` | 拍立淘 |
| `Camera_XHS_<13位ms>` | `Camera_XHS_1703236957940...png` | 小红书 |
| `beautybox_YYYYMMDDHHMMSS` | `BeautyBox_20201019213814.jpg` | 美图秀秀 |
| `null-<13位ms>` | `null-1668240276000.jpg` | 系统截图 |
| `YYYYMMDD_HHMMSS` (10位 Unix) | `1454537310.jpg` | 老安卓 |
| `YYYYMMDD_HHMMSS_XXX` | `20150820_135327_044.jpg` | 三星 |
| `YYYY.MM.DD_HH.MM.SS` | `2024.06.24_08.23.50.jpg` | 拍立得 |
| `涂鸦_screenshot_<...>_微信` | `涂鸦_screenshot_2019-03-16-10-30-23-212_微信.png` | 微信涂鸦 |
| `share_app_YYYYMMDD` | `share_app_20190801.jpeg` | 微信PC版 |

### 5. mtime 兜底策略

| mtime 状态 | 策略 |
|-----------|------|
| 离群（不是 2026-07-01 / 2025-04-05 复制时间） | **必须用户确认后**用 mtime |
| 在复制时间窗（2026-07-01 10:43 / 11:03 / 12:31） | 用 2025-12-30 + mtime 时分秒 兜底 |
| 用户要求用 mtime 复制时间（如 share_<hash>） | 注入 2025-04-05 + mtime 时分秒 |

**重要：mtime 注入前必须用户确认。**

## 分类合并

| 来源 | 归类 | 子文件夹 |
|------|------|---------|
| 相机拍摄 (Camera) | 相机拍摄 | 按设备名分 |
| Screenshots/ | 截图 | 单独 |
| WeiXin/, 微信图, mmexport, share_ | 微信 | 单独 |
| 自拍 | 相机拍摄 | 同相机 |
| 截图 (Screenshots) | 截图 | 单独 |

## 微信文件夹特殊处理

| 文件名 | 修复方法 |
|--------|---------|
| `share_<hash>.png` | **mtime 是复制时间, 用 2025-04-05 + mtime 时分秒**（已确认方案） |
| `share_<hash>.mp4` | ffmpeg 注入 2025-04-05 + mtime 时分秒 |
| `mm_facetoface_collect_qrcode_*.png` | mtime 真实（2025-04-25），用 mtime |
| `lqpic.jpg` (损坏 EXIF) | PIL 重保存 + pyexiv2 注入 |
| `陈鑫.jpg` / `陈鑫证件照.jpg` | 用 mtime 注入"导入时间" |

## 使用方式

### 作为 Codex Skill

本 Skill 注册后会出现在 Codex 的可用技能列表中。当用户提到"把 Android 照片导入 iPhone"或类似问题时，Codex 会自动加载并按照本流程执行。

### 作为独立脚本

脚本位于 `C:\Users\ChenDebiao\Documents\Codex\2026-06-29\ni\work\`，关键脚本：

```bash
# JPG 批量清理 + 注入
python batch_clean_jpg.py

# MP4 修复 (V13 + ffmpeg fallback)
python mp4_fix_v14.py <src_dir> <out_dir>

# 文件名时间戳解析 + 注入
python rescan_camera_v5.py

# 微信文件夹兜底注入 (share_ 系列用 2025-04-05 占位)
python fix_share_mp4.py
python fix_share_unfixable.py

# PNG 注入
python fix_png.py
```

## 依赖

- Python 3.14+（或兼容版本）
- pyexiv2（EXIF 读写）
- Pillow (PIL) — PNG 注入
- ffmpeg — MP4 注入、ffprobe
- 环境变量 `EXIV2_DLL_PATH`（指向 pyexiv2 的 DLL 路径）

## 已验证的手机品牌

| 品牌 | 照片 | 视频 |
|------|------|------|
| 荣耀/Honor (BVL-AN20) | ✅ | ✅ |
| 小米/Xiaomi (22041211AC) | ✅ | ✅ |
| 三星/Samsung (Galaxy S23) | ✅ | ✅ |
| 一加/OnePlus (12) | ✅ | ✅ |
| OPPO (Reno 10x Zoom) | ✅ | ✅ |
| 锤子/Smartisan (U3 Pro) | ✅ | ✅ |
| 华为/HUAWEI (MHA-AL00) | ✅ | 待测 |
| 苹果/iPhone (11) | ✅ | ✅ |
| SONY (A7M4 / ZV-E1) | ✅ | ✅ |
| NIKON (D800 / D3000 / COOLPIX) | ✅ | ✅ |

## 已验证的注入场景

- 荣耀 Magic6 至臻版 JPG (1,218 张带 EXIF)
- 微信图 share_<hash>.png (99 张 mtime 兜底 2025-04-05)
- 微信视频 share_<hash>.mp4 (36 张 ffmpeg 注入)
- 美图秀书 beautybox 系列 (177 张)
- 小红书 Camera_XHS (8 张)
- iPhone 11 / iPhone 17 PM 备份 (312 张)

## License

MIT
