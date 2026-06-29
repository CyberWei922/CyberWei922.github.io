---
title: 给个人网站接入 Strava API：从零开始的折腾全记录
date: 2025-07-01
tags: [软件, web, Linux, 教程]
desc: 记录我如何在纯静态网站上实现 Strava 骑行数据的自动同步展示，从 OAuth 授权到 GitHub Actions 的完整过程。
---

# 给个人网站接入 Strava API：从零开始的折腾全记录

最近在重构自己的个人网站，想把 Strava 上的骑行数据展示出来——累计里程、最近的活动、爬升数据这些。看起来很酷，但实际折腾下来踩了不少坑。这篇文章记录整个过程，希望能帮到同样想接入 Strava 的人。

---

## 先搞清楚问题在哪

我的网站是纯静态页面，托管在 GitHub Pages 上，没有任何后端服务器。

一开始我以为直接调 Strava 的开放 API 就行了，查了一圈才发现没那么简单：

**Strava API 强制要求 OAuth 2.0 认证。** 也就是说，每一次 API 请求都必须带上一个有效的 `access_token`，而这个 token **每 6 小时就会过期一次**。

想在纯静态网页里实时调 API？不可能的事，因为：
- token 放在前端代码里会直接暴露给所有人
- token 过期后没有服务器去自动刷新

所以直接在 HTML 里写 fetch 调 Strava API 这条路死掉了。

---

## 解决思路：GitHub Actions 中转

想了一圈，最后想到一个优雅的方案：

**用 GitHub Actions 做中间层。**

流程是这样的：

```
GitHub Actions（定时触发）
  → 用 refresh_token 换新的 access_token
  → 调 Strava API 拉数据
  → 把数据写入仓库里的 data/strava.json
  → commit & push

静态网页
  → 直接 fetch data/strava.json（本仓库文件，无需鉴权）
  → 渲染数据
```

这样网页还是纯静态，完全不需要后端，数据定期自动更新，token 也安全地存在 GitHub Secrets 里。

---

## 第一步：注册 Strava 开发者应用

去 [https://www.strava.com/settings/api](https://www.strava.com/settings/api) 创建一个应用。

几个字段的填写建议：

| 字段 | 填什么 |
|------|--------|
| Application Name | 随意，比如 `MyWebsite` |
| Category | `Other` |
| Website | 你的网站域名，比如 `https://cyberwei.me` |
| Authorization Callback Domain | 你的域名，比如 `cyberwei.me` |
| 应用图标 | 随便传一张图就行，不影响功能 |

创建完成后，页面会给你两个关键值：
- **Client ID**：一串纯数字
- **Client Secret**：一串字母数字混合的字符串

这两个保存好，后面要用。

---

## 第二步：完成 OAuth 授权，拿到 refresh_token

Strava 的 OAuth 流程是标准的授权码模式。

### 2.1 构造授权链接

把下面的 `YOUR_CLIENT_ID` 换成你的 Client ID，在浏览器里打开：

```
https://www.strava.com/oauth/authorize?client_id=YOUR_CLIENT_ID&response_type=code&redirect_uri=https://你的域名&approval_prompt=force&scope=activity:read_all,read
```

注意 `scope` 一定要带上 `activity:read_all`，只有 `read` 的话拿不到骑行统计数据。

### 2.2 授权并拿到 code

点击授权后，浏览器会跳转到你的域名，URL 里会有一段类似这样的参数：

```
https://你的域名/?code=820f11c05c2119bbe3e2f812fb2c07e203c75cdc&scope=read,activity:read_all
```

复制 `code=` 后面那串值。**注意这个 code 只能用一次，拿到之后马上进行下一步。**

页面报错没关系，只要能看到 URL 里的 code 就够了。

### 2.3 用 code 换取 refresh_token

在终端执行（替换掉 `YOUR_CLIENT_ID`、`YOUR_CLIENT_SECRET`、`YOUR_CODE`）：

```bash
curl -X POST https://www.strava.com/oauth/token \
  -d client_id=YOUR_CLIENT_ID \
  -d client_secret=YOUR_CLIENT_SECRET \
  -d code=YOUR_CODE \
  -d grant_type=authorization_code
```

返回的 JSON 里找 `refresh_token` 这个字段，把它保存下来。这个 token 长期有效，是我们后续自动刷新的核心。

---

## 第三步：把密钥存入 GitHub Secrets

绝对不能把 Client Secret 和 refresh_token 写在代码里，要放在 GitHub 的加密环境变量里。

进入你的仓库 → **Settings** → 左侧 **Secrets and variables** → **Actions** → **New repository secret**

依次添加三个：

| Secret 名称 | 值 |
|------------|-----|
| `STRAVA_CLIENT_ID` | 你的 Client ID（数字） |
| `STRAVA_CLIENT_SECRET` | 你的 Client Secret |
| `STRAVA_REFRESH_TOKEN` | 刚才拿到的 refresh_token |

---

## 第四步：写 GitHub Actions 工作流

在仓库里新建文件 `.github/workflows/strava-sync.yml`：

```yaml
name: Sync Strava Data

on:
  schedule:
    - cron: '0 * * * *'   # 每小时整点运行
  workflow_dispatch:        # 支持手动触发

jobs:
  sync:
    runs-on: ubuntu-latest
    permissions:
      contents: write

    steps:
      - uses: actions/checkout@v4

      - name: Fetch and write strava.json
        env:
          CLIENT_ID: ${{ secrets.STRAVA_CLIENT_ID }}
          CLIENT_SECRET: ${{ secrets.STRAVA_CLIENT_SECRET }}
          REFRESH_TOKEN: ${{ secrets.STRAVA_REFRESH_TOKEN }}
        run: |
          python3 << 'PYEOF'
          import json, datetime, urllib.request, urllib.parse, os

          def post(url, data):
              req = urllib.request.Request(url, urllib.parse.urlencode(data).encode())
              with urllib.request.urlopen(req) as r:
                  return json.loads(r.read())

          def get(url, token):
              req = urllib.request.Request(url, headers={"Authorization": f"Bearer {token}"})
              with urllib.request.urlopen(req) as r:
                  return json.loads(r.read())

          def km(m):   return round((m or 0) / 1000, 1)
          def hrs(s):  return round((s or 0) / 3600, 1)
          def elev(e): return round(e or 0)

          # 用 refresh_token 换新的 access_token
          token_data = post("https://www.strava.com/oauth/token", {
              "client_id":     os.environ["CLIENT_ID"],
              "client_secret": os.environ["CLIENT_SECRET"],
              "refresh_token": os.environ["REFRESH_TOKEN"],
              "grant_type":    "refresh_token",
          })
          access = token_data["access_token"]

          # 拉取运动员统计数据
          stats = get("https://www.strava.com/api/v3/athletes/YOUR_ATHLETE_ID/stats", access)

          # 拉取最近活动
          acts = get("https://www.strava.com/api/v3/athlete/activities?per_page=8", access)

          # 筛选骑行类型
          ride_types = {"Ride", "MountainBikeRide", "GravelRide", "VirtualRide"}
          recent = []
          for a in acts:
              if a.get("sport_type") in ride_types:
                  recent.append({
                      "name":          a.get("name", ""),
                      "date":          (a.get("start_date_local") or "")[:10],
                      "distance_km":   km(a.get("distance", 0)),
                      "moving_time_h": hrs(a.get("moving_time", 0)),
                      "elevation_m":   elev(a.get("total_elevation_gain", 0)),
                      "avg_speed_kmh": round((a.get("average_speed") or 0) * 3.6, 1),
                  })

          ar = stats.get("all_ride_totals", {})
          yr = stats.get("ytd_ride_totals", {})
          rr = stats.get("recent_ride_totals", {})

          output = {
              "updated_at":    datetime.datetime.utcnow().strftime("%Y-%m-%dT%H:%M:%SZ"),
              "all_time": {
                  "distance_km":   km(ar.get("distance")),
                  "moving_time_h": hrs(ar.get("moving_time")),
                  "elevation_m":   elev(ar.get("elevation_gain")),
                  "count":         ar.get("count", 0),
              },
              "ytd": {
                  "distance_km":   km(yr.get("distance")),
                  "moving_time_h": hrs(yr.get("moving_time")),
                  "elevation_m":   elev(yr.get("elevation_gain")),
                  "count":         yr.get("count", 0),
              },
              "recent_4w": {
                  "distance_km": km(rr.get("distance")),
                  "count":       rr.get("count", 0),
              },
              "biggest_ride_km":   km(stats.get("biggest_ride_distance")),
              "recent_activities": recent,
              "strava_url":        "https://www.strava.com/athletes/YOUR_ATHLETE_ID",
          }

          os.makedirs("data", exist_ok=True)
          with open("data/strava.json", "w", encoding="utf-8") as f:
              json.dump(output, f, ensure_ascii=False, indent=2)

          print("done")
          PYEOF

      - name: Commit & push
        run: |
          git config user.name  "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add data/strava.json
          git diff --staged --quiet || git commit -m "chore: sync strava $(date -u +%Y-%m-%d) [skip ci]"
          git push
```

注意最后 commit message 里的 `[skip ci]`，这个标记可以防止每次 Action 提交都触发 GitHub Pages 重新部署，避免部署页面一直转圈。

---

## 第五步：在网页里读取数据

有了 `data/strava.json` 之后，网页里直接 fetch 这个文件就行，不需要任何鉴权：

```javascript
async function loadStrava() {
  const data = await fetch('data/strava.json').then(r => r.json());

  // 累计距离
  console.log(data.all_time.distance_km + ' km');

  // 今年数据
  console.log(data.ytd.distance_km + ' km');

  // 最近活动列表
  data.recent_activities.forEach(a => {
    console.log(a.name, a.distance_km + 'km', a.avg_speed_kmh + 'km/h');
  });
}
```

`strava.json` 的数据结构如下：

```json
{
  "updated_at": "2025-07-01T10:00:00Z",
  "all_time": {
    "distance_km": 1234.5,
    "moving_time_h": 56.7,
    "elevation_m": 12345,
    "count": 42
  },
  "ytd": {
    "distance_km": 500.0,
    "moving_time_h": 22.0,
    "elevation_m": 5000,
    "count": 18
  },
  "recent_4w": {
    "distance_km": 120.5,
    "count": 5
  },
  "biggest_ride_km": 312.0,
  "recent_activities": [
    {
      "name": "晨骑",
      "date": "2025-06-30",
      "distance_km": 45.2,
      "moving_time_h": 1.5,
      "elevation_m": 320,
      "avg_speed_kmh": 30.1
    }
  ],
  "strava_url": "https://www.strava.com/athletes/xxxxxxx"
}
```

---

## 踩过的坑

**1. scope 只选了 read**

第一次授权的时候没注意，scope 只选了 `read`，结果 API 返回的统计数据全是 0。必须要 `activity:read_all` 才能拿到骑行数据。解决方法是重新走一遍授权流程，拿新的 code 再换 token。

**2. code 只能用一次**

授权后 URL 里的 `code` 用 curl 换完 token 就失效了。如果换 token 的时候报错，不要以为是参数写错了，很可能就是 code 已经过期或者已经用过了，重新授权拿新 code 就行。

**3. Action 每次提交触发 Pages 部署**

Action 每小时自动 commit 一次 `strava.json`，每次 commit 都会触发 GitHub Pages 重新部署，导致仓库页面一直有个黄色转圈。在 commit message 末尾加 `[skip ci]` 就能跳过部署触发。

**4. 本地 push 冲突**

Action 在自动提交，本地也在改代码，push 的时候会因为远端有新 commit 而被拒绝。养成习惯，每次 push 前先 `git pull origin main --rebase` 同步一下就没问题了。

---

## 总结

整个方案的核心思路就是：**用 GitHub Actions 把需要鉴权的 API 请求从浏览器端移到服务端，把结果缓存成静态 JSON 文件，网页直接读文件。**

这个思路不只适用于 Strava，任何需要 token 鉴权、但又想在静态网站展示的 API 都可以用这套方法——比如 GitHub 贡献图、Last.fm 听歌记录、Notion 数据库等等。

如果你也在搭自己的个人网站，希望这篇记录对你有帮助。
