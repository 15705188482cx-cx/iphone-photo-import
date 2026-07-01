# iPhone Photo Import Skill

> **skill_name:** `iphone-photo-import`  
> **description:** 把 Android 手机的 JPG 照片和 MP4 视频转换成 iPhone 能正确识别 GPS/时间/设备名/颜色的格式  
> **version:** 2.0  
> **last_updated:** 2026-07-01

---

## 背景

从 Android 手机（荣耀/小米/三星/一加等）把照片视频传到 iPhone 后，经常遇到三个问题：

| 问题 | 现象 | 根因 |
|------|------|------|
| ❌ 地理信息丢失 | iPhone 相册不显示拍摄地点 | Android 写入的私有 EXIF MakerNote + APP6-APP10 段让 iOS 跳过整个 EXIF 解析 |
| ❌ 颜色异常 | 导入后变黑白，需点"编辑"才恢复 | HEIC 格式的 ICC 色彩配置在 iOS 26 上有渲染 bug |
| ❌ 拍摄时间/设备名丢失 | 显示"2026年"（导入时间）或无设备名 | MP4 的元数据命名空间（com.apple.quicktime.*）与 Android 写入的 key 不兼容 |

## 解决方案

### JPG 照片

**原则：不转 HEIC，只清理 EXIF 私有段。**

不再做 JPG→HEIC 转换。iOS 26 的 HEIC 渲染有 bug（导入变黑白，要点编辑才恢复）。相反，方案是：

1. 用 `pyexiv2` 读取 EXIF
2. 删除 `MakerNote` + Thumbnail（私有信息段）
3. 调用 `clear_exif()` + `modify_exif(md)` 重新写入干净的 EXIF
4. 字节级重组：跳过 APP6-APP10（0xFFE6-0xFFEA）标记段
5. 保留：GPS / DateTimeOriginal / Make / Model / ICC

> ⚠️ **禁止**：pyexiv2 不能直接读中文路径。通过 `subst` 虚拟盘或临时拷贝到英文路径解决。

### MP4 视频

**原则：保留原视频编码，只修复 moov box 元数据。**

iOS 26 需要 `com.apple.quicktime.*` 命名空间的 7 个元数据 key。从原始 moov box 中读取：

- 拍摄时间（creation_time）
- GPS 坐标（location）
- 设备名（make/model）

然后构造兼容的 meta box 字节级注入，同时同步更新 `stco`/`co64` 的 chunk offset。

> ⚠️ **禁止**：不要用 ffmpeg remux（会丢失命名空间）；不要人为编写设备信息（必须从原视频读取后映射）。

### 分类合并

| 来源 | 归类 |
|------|------|
| Screenshots/ | 截图 |
| WeiXin/ / mmexport / share_ | 微信 |
| 其余所有 | 相机拍摄 |

支持按设备名建立子文件夹 + 全量去重（基于 MD5）。

## 使用方式

### 作为 Codex Skill

本 Skill 注册后会出现在 Codex 的可用技能列表中。当用户提到"把 Android 照片导入 iPhone"或类似问题时，Codex 会自动加载并按照本流程执行。

### 作为独立脚本

脚本位于 `scripts/` 目录（需要时可按流程生成）：

```bash
# JPG 批量清理
python batch_clean_jpg.py --input D:\photos --output D:\photos_clean

# MP4 批量修复
python mp4_fix_v13.py --input D:\videos --output D:\videos_fixed

# 分类合并
python merge_camera.py --input D:\mixed --output D:\organized
```

## 依赖

- Python 3.14+（或兼容版本）
- pyexiv2（EXIF 读写）
- ffmpeg（仅用于读取元数据）
- 环境变量 `EXIV2_DLL_PATH`（指向 pyexiv2 的 DLL 路径）

## 已验证的手机品牌

| 品牌 | 照片 | 视频 |
|------|------|------|
| 荣耀/Honor | ✅ | ✅ |
| 小米/Xiaomi | ✅ | ✅ |
| 三星/Samsung | ✅ | ✅ |
| OPPO/一加 | ✅ | ✅ |
| vivo | ✅ | 待测 |

## License

MIT
