---
name: cn-image-host-upload
description: 把本地图片上传到国内可直连的免费图床并拿到公网直链（给需要公网图片 URL 的国内 AI API 用）。包含 2026 年实测可用的图床接口、已失效图床清单、手写 multipart 编码、以及 urllib 的重定向/路径坑。当用户要做「图生图需要先上传参考图」「接一个免费图床」「图片转公网链接」这类需求时使用。
agent_created: true
---

# 国内免费图床上传

## 何时用

需求里出现「本地图片要先变成公网 URL 才能给 API 用」的场景，典型如国内 AI 生图 API 的
参考图参数（速创 image_gpt、火山方舟 `image_urls`、ModelScope 等）只接受 http(s) 直链。

## 第一步：先确认哪些图床还活着

**不要凭记忆写接口。** 免费图床死得极快，下面这张表是 2026-09 实测结果：

| 图床 | 结果 |
|------|------|
| **uapis.cn** | ✅ **唯一实测免注册可用的国内图床** |
| sm.ms | ❌ `308` 跳转到 `s.ee/api/v1/file/upload`，且返回 `Unauthorized`（必须带 token） |
| imgurl.org | ❌ `/api/v2/upload` 强制要求 `uid` + `token` |
| cdnjson.com | ❌ Chevereto 接口，`Invalid API key` |
| 0x0.st | ❌ 官方停服（"uploads disabled"） |
| picui.cn | ⚠️ 需 token；无 token 时请求会挂到超时 |
| imgbb / freeimage.host | ⚠️ 需自备 API Key，且是海外站点 |
| catbox.moe / tmpfiles.org | ⚠️ 时好时坏，返回空响应常见 |
| telegraph | ⚠️ 国内不可达 |
| 路过图床 / 牛图网 / Hello图床 / Picbed / Z4A / hualigs / uomg | ❌ 无公开上传 API，404 或返回 HTML |

新图床要先探一下：拿一张小 PNG `POST` 上去看返回，别只看官网宣传。

## 首选方案：uapis.cn（免注册）

```
POST https://uapis.cn/api/v1/image/upload
Content-Type: multipart/form-data
field: file

→ {"deduplicated":false,"message":"success","size":73,
   "url":"https://uapis.cn/static/uploads/xxx.webp"}
```

- 返回的是**国内域名**直链，国内 API 服务器能抓到（这点比 catbox 之类海外图床关键）。
- 会把图转成 **webp**；一般 AI API 都接受，如果下游只认 jpg/png 要另行处理。
- **匿名访客每日 10 张**，超了返回：
  `图片上传接口限流中，每日限制10次…登录后携带 API Key 调用不受此限`
- 带 Key 解除限制（免费注册，每月 3500 积分）：`Authorization: Bearer <api_key>`
  （`X-API-Key` 也可，是旧写法）。接口本身不在公开 OpenAPI 里，属于实测接口，要留降级链。

## 推荐结构：适配器链 + 自动降级

不要只写一个图床。做成 `key -> (name, need, tip, timeout)` 的注册表 + 顺序列表，
逐个尝试，第一个成功即用，全部失败回退本地地址并把每个图床的失败原因回传给前端。

要点：

- `need == 'none'` 的图床（免注册）永远参与；`need != 'none'` 但**没配凭据的要直接跳过**，
  否则每次上传都要白等一圈超时。
- 免注册的海外兜底图床给**短 timeout**（15–20s），避免整体太慢。
- 要能区分「拿到了公网 URL」和「回退成本地 URL」，前端要如实提示后者大概率不可用。

## 代码要点

标准库没有 multipart 编码器，需要手写（`\r\n` 拼接 + boundary），注意最后一个 boundary 要加 `--` 后缀。

`urllib` 两个坑：

```python
# 1) POST + 307/308 不会自动跟随重定向，会抛 HTTPError（308 不在 urllib 的 POST 重定向白名单里）
#    要自己读 Location 重发，或者换用不会重定向的端点。
# 2) 发 multipart 时不要走 http_request 里"dict 就 json.dumps"的分支，直接传 bytes 并自带 Content-Type
```

响应字段名各家不同，取值要写得宽容：Chevereto 系是 `image.url`，sm.ms 系是 `data.url`，
picui 是 `data.links.url`，uapis 是顶层 `url`，uguu 是 `files[0].url`，catbox 直接返回纯文本 URL。
可以写一个 `_pick(obj, 'a.b.c')` 按路径取，且校验取到的值以 `http` 开头。

## 排错

- 想确认某图床是否还能用：`curl -s -X POST -F "file=@x.png" <url>`，看返回体而不是状态码。
- 图片传上去了但下游 API 还是报错：先 `curl` 一下拿到的直链，确认**公网可访问**；
  再考虑下游服务器所在网络能不能到（海外图床国内直连经常不行）。
- 需要公网 IP 才能直连的场景不要用 `http://127.0.0.1/...` 冒充，下游一定抓不到。
