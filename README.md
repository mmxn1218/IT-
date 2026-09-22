# IT工具箱 · 自动更新发布仓库

这个仓库**只放发布产物**，不放源码。

## 里面是什么

| 文件 | 作用 |
| --- | --- |
| `IT工具箱.dat` | 就是 `IT工具箱.exe`，只是改了后缀。**必须叫 `.dat`** —— jsDelivr 对 `.exe` 返回 403 |
| `version.json` | 版本描述，客户端启动时用它比对版本并校验下载 |

`version.json` 格式：

```json
{
  "version": "2026.09.22.1500",
  "md5": "e222005f37488dbdececfa0dc31bea1d",
  "size": 109056
}
```

- `version` 格式 `YYYY.MM.DD.HHMM`，**必须和客户端编译进去的 `IT_VERSION` 完全一致**
- `md5` / `size` 必须是 `IT工具箱.dat` 的真实值，客户端下载后会逐一核对，不符就丢弃

## 客户端怎么更新

```
GET https://cdn.jsdelivr.net/gh/mmxn1218/IT-@main/version.json
  → 版本号比本机新？
GET https://cdn.jsdelivr.net/gh/mmxn1218/IT-@main/IT工具箱.dat
  → 校验 size + md5
  → 换成新的 exe，重启
```

主地址不通时自动改走备用地址：

```
https://raw.githubusercontent.com/mmxn1218/IT-/main/
```

## 怎么发新版本

1. 改源码里的 `IT_VERSION`（`update.h`），格式 `YYYY.MM.DD.HHMM`
2. 编译出新的 `IT工具箱.exe`
3. 用发布助手一键发版（自动：改名成 `.dat` → 算 md5/size → 写 `version.json` → commit → push → 清 CDN 缓存 → 回读验证）

> 发版顺序不能错：**先改版本号再编译**。版本号没变的话，客户端会认为已经是最新，不会下载。

## 注意

- 仓库必须是 **public**，否则 raw / jsDelivr 取不到文件
- jsDelivr 有缓存。发完版要清一下：
  `https://purge.jsdelivr.net/gh/mmxn1218/IT-@main/version.json`
- 客户端已经带了 `no-cache` 请求头，正常情况下不需要手动清缓存
