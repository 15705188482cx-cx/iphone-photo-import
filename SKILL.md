# iPhone 照片导入 Skill

> skill_name: iphone-photo-import
> description: 把 Android 手机的 JPG 照片和 MP4 视频转换成 iPhone 能正确识别 GPS/时间/设备名/颜色的格式
> version: 2.0
> last_updated: 2026-07-01

---

## 核心结论

**JPG 问题**：不是数据丢了，是 iOS 26 不认 HONOR 私有 EXIF 字段（MakerNote + APP6-APP10），删掉即可。

**MP4 问题**：iOS 26 需要 com.apple.quicktime.* 命名空间的 7 个 key。V13 算法字节级注入 moov box，零损失。

**绝对不要转 HEIC**（iOS 26 渲染 bug，刚导入变黑白）。

## 工具路径

Python: C:\Users\ChenDebiao\AppData\Local\Python\pythoncore-3.14-64\python.exe
ffmpeg: E:\ffmpeg\ffmpeg-8.1.1-full_build\bin\
work:   C:\Users\ChenDebiao\Documents\Codex\2026-06-29\ni\work\

## 处理流程

### JPG 清理 (batch_clean_jpg.py / batch_clean_tree.py)
1. pyexiv2 读 EXIF → 删 MakerNote + Thumbnail
2. clear_exif() + modify_exif(md)
3. 字节级重组：跳过 APP6-APP10 (0xFFE6-0xFFEA)
4. 保留：GPS / DateTimeOriginal / Make / Model / ICC

### MP4 修复 (mp4_fix_v13.py + mp4_meta_builder.py)
1. 从原始 moov 读 location/make/model/creation_time
2. 构造 7 个 com.apple.quicktime.* key 的 meta box
3. 字节级替换 moov.meta + trak.meta
4. 同步更新 stco/co64 chunk offset（关键 Bug 修复）

### 分类合并 (merge_camera.py)
Screenshots/ → 截图
WeiXin/微信图/mmexport/share_ → 微信
其余所有 → 相机拍摄

### 去重 + 爱思助手导入

## Windows 注意事项
- pyexiv2 不可读中文路径 → subst 虚拟盘 或 临时拷贝到英文路径
- 先 clear_exif() 再 modify_exif(md)
- 设置 EXIV2_DLL_PATH

## 禁止
- 不转 HEIC
- 不用 ffmpeg remux (丢 namespace)
- 不人为修改元数据
