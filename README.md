# fun-personality-test
# 项目说明

这不是一个正经的 MBTI 测试。

表面上它伪装成一个“人格类型测试”，首页文案、进度条、四维测评的壳子都照搬 MBTI 风格，用来发给朋友整蛊。实际测的是男娘潜质，题目来自一套自建的 130 题题库，答完不显示分数，而是把结果编码进一个二维码，让对方“发给作者获取解读”。

朋友扫码或截图发回来，作者用配套的解码器还原真实结果。

---

## 文件结构

```
├── index.html      # 测评页（伪装成 MBTI）
└── decoder.html    # 二维码解码器（自用）
```

两个文件都是纯静态单页，无后端、无构建、无依赖安装，直接用浏览器打开就能跑。

---

## 部署

最简单的方式是 GitHub Pages：

1. 新建仓库，把两个文件推到根目录
2. Settings → Pages → Source 选 `main` / 根目录
3. 访问 `https://<用户名>.github.io/<仓库名>/` 就是测评页
4. 解码器在 `https://<用户名>.github.io/<仓库名>/decoder.html`

也可以用 Vercel、Netlify、Cloudflare Pages，拖进去就行。

**注意**：测评页的二维码识别依赖 CDN 加载 `qrcode` 库，首次打开需要联网；如果加载失败会回退到 `api.qrserver.com` 生成图片二维码。解码器依赖 `jsqr` 库，同样走 CDN。

---

## 测评页是怎么伪装的

- **首页**：完全照抄 MBTI 文案——“人格类型测试”、“基于荣格心理类型理论”、“四个维度帮你找到 4 字母人格类型”，配 65 题 / 4 维度 / ~8 分钟三组数据。没有任何“男娘”字样。
- **题目**：65 道题里混了 15 道正经 MBTI 题（E/I、S/N、T/F、J/P 各若干），标记为 `decoy: true`，只出现在答题流程里，**不计分、不影响结果**。剩下 50 道是真正的核心题，分 A/B/C/D 四个维度，题目措辞做过降敏处理，不会一眼看出来。
- **抽题**：每次从 130 题里按维度分布随机抽 50 道核心题，再随机抽 15 道伪装题，打乱顺序，所以每次点开题目都不一样。
- **结果页**：不显示任何分数、段位、雷达图，只有一个二维码和一句“请截图或保存二维码，发送给作者获取完整解读”。

---

## 结果编码格式

答题结束后，页面会把结果打包成一个中文 JSON：

```json
{
  "男娘测试综合分": "117/250",
  "维度百分比": {
    "审美偏好": 48,
    "行为习惯": 45,
    "心理共情": 55,
    "社交亚文化": 38
  }
}
```

**没有答案串，没有时间戳，没有设备信息**，只有综合分和四维百分比。

编码流程：

```js
var json = JSON.stringify(payload);
var b64 = btoa(unescape(encodeURIComponent(json)));
```

用 `unescape + encodeURIComponent` 是为了让中文能安全通过 `btoa`（`btoa` 原生只接受 Latin-1）。解码时对应地：

```js
var json = decodeURIComponent(escape(atob(b64)));
```

二维码里存的就是这个 Base64 字符串。

---

## 解码方式

**方式一：用 decoder.html**

打开解码器页面，两种输入方式都支持：

- 上传或拖拽二维码图片，自动识别、自动解码、自动渲染结果
- 直接把 Base64 字符串粘贴到输入框，点“解码结果”

解码成功后会显示综合分、四维进度条，还能展开看原始 JSON。

**方式二：手动解码**

任何支持 Base64 的工具都行，命令行：

```bash
echo 'eyLnlLflq5jmtYvor5Xnu7zlkIjliIYi...' | base64 -d | python3 -c "import sys,urllib.parse;print(urllib.parse.unquote(sys.stdin.read()))"
```

或者在浏览器控制台：

```js
JSON.parse(decodeURIComponent(escape(atob('粘贴Base64'))))
```

---

## 分数与段位

综合分 = 四个维度百分比的平均值 × 2.5，范围 0–250。

解码器只展示分数和四维百分比，段位判定逻辑仍然保留在测评页代码里（`generateResult()` 中的 `tierIndex` 计算），只是没有写进二维码。需要的话可以自己加回去。

段位对照：

| 分数区间 | 段位 |
|---|---|
| 0–75 | 绝对钢铁壁垒 |
| 76–105 | 硬核直男本色 |
| 106–135 | 薛定谔的萌新 |
| 136–160 | 浅度粉红觉醒 |
| 161–185 | 深度觉醒预备役 |
| 186–205 | 软妹基因初醒 |
| 206–225 | 纯度极高的男娘 |
| 226–250 | 究极天选男娘 |

---

## 想改点什么

**改题库**：`rawQuestionBank` 数组，每条格式：

```js
// 态度自评题，1-5 分
{ type:'likert', d:'A', r:false, t:"题目文本" }

// 反向计分题
{ type:'likert', d:'A', r:true, t:"【反向】题目文本" }

// 情景选择题
{ type:'choice', d:'B', t:"题干", options: [
  {v:1, l:"选项一"}, {v:2, l:"选项二"}, {v:4, l:"选项三"}, {v:5, l:"选项四"}
]}

// MBTI 伪装题，不计分
{ type:'likert', d:'M', r:false, decoy:true, t:"题目文本" }
```

`d` 是维度（A 审美偏好 / B 行为习惯 / C 心理共情 / D 社交亚文化 / M 伪装题），`r: true` 表示反向计分。

**改抽题数量**：`initGame()` 里的 `targetDistribution`：

```js
var targetDistribution = {
    A: { likert: 9, choice: 3 },
    B: { likert: 9, choice: 4 },
    C: { likert: 9, choice: 3 },
    D: { likert: 9, choice: 4 }
};
```

伪装题数量在同一个函数里，`var decoyCount = Math.min(15, decoyPool.length);`。

**改伪装文案**：首页的 `.sub`、`.meta` 和 `<title>` 都是硬编码的，直接改 HTML 就行。

**改二维码内容结构**：`generateResult()` 里的 `payload` 对象，改完记得同步改 `decoder.html` 里 `renderResult()` 的字段读取逻辑。

---

## 已知限制

- 二维码内容**没有签名校验**，理论上任何人都能手动构造 JSON 再 Base64 生成假二维码。
- 解码器只认 `男娘测试综合分` 和 `维度百分比` 两个字段，结构不对会报错。
- 伪装题虽然不计分，但会占用答题时间，65 题完整答完大约 5–8 分钟。
- 二维码数据量已经很小，但如果以后往 payload 里加字段，注意别超过二维码容量（当前 M 级纠错下余量还很足）。

---

## 免责

整蛊归整蛊，发给朋友之前想清楚对方能不能接受。别用在职场、长辈、不熟的人身上。
