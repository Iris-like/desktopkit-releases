# DesktopKit Releases

DesktopKit 桌面应用的构建、签名与发布仓库。

源码在 [Iris-like/desktopkit](https://github.com/Iris-like/desktopkit)。

## 为什么拆成两个仓库

参照 [magpie](https://github.com/yetone/magpie) 的做法，收益在四件事：

1. **密钥隔离**：更新签名私钥只在本仓库的 Actions secrets 里。源码仓库接受 PR 和 issue，攻击面更大。
2. **写权限隔离**：源码仓库可以给更多人写权限；本仓库只留维护者。
3. **发布物与源码解耦**：更新 feed 和下载地址固定指向本仓库，源码仓库改名或迁移不影响已发布的客户端。
4. **构建隔离**：macOS runner 耗时且贵，放在本仓库跑，源码仓库的 CI 保持轻量。

## 发布流程

```bash
# 1. 改源码仓库的版本号（build/config.yml）
# 2. 本仓库打 tag
git tag desktop-v0.1.0
git push origin desktop-v0.1.0
```

tag 推送后自动触发 `.github/workflows/desktop-release.yml`：

```
checkout 源码（匿名，公开仓库）
  → wails3 package（arm64 + amd64 两个 job）
  → codesign（配了证书才签）
  → ditto 打 zip + 结构自检
  → 汇总产物
  → sha256sum
  → gen-feed.py 生成签名 feed
  → gh release create
```

## 必需的 secret

| Secret | 用途 |
| --- | --- |
| `UPDATE_SIGNING_KEY` | ed25519 私钥 PEM，给 feed 签名 |

**缺少它会直接失败**，不允许降级发版。

## 可选的 secret（代码签名）

| Secret | 用途 |
| --- | --- |
| `MAC_CERTIFICATE_P12` | Apple Developer ID 证书（base64） |
| `MAC_CERTIFICATE_PASSWORD` | 证书密码 |

不配也能发布，但产出的是 ad-hoc 签名的包，用户首次打开需右键 → 打开。

## 产物

| 文件 | 用途 |
| --- | --- |
| `desktopkit-darwin-arm64.zip` | Apple Silicon，含 `.app` |
| `desktopkit-darwin-amd64.zip` | Intel |
| `desktopkit-feed.json` | 更新 feed（已签名） |
| `SHA256SUMS` | 校验清单 |

## 更新签名密钥

私钥丢失等于存量用户永久无法收到更新——新版本必须由旧私钥签名，而私钥已经没了。

因此：

- 私钥必须**离线备份两份**，放在不同介质
- 每年做一次恢复演练
- 换钥需要双公钥过渡：先发带新公钥的版本，再用新私钥签名

详见 `PIPELINE.md` 第 16 节。
