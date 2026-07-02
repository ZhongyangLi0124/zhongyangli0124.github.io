# 会话交接 — 给接手的新 Claude Code session

> **一句话背景**:这是 aiming-lab/WebHarbor 的贡献 PR #29(新增 PhET
> Interactive Simulations 镜像站)。PR 收到 review,五条意见已在代码上
> 全部修好并验证,现在只差**在用户本地机器上落地**:应用 patch → 预览 →
> 重传 HF 资产 → 重钉 .assets-revision → push。
>
> 之前的会话跑在云端沙箱里,碰不到用户的 Mac、push 不了 fork、连不上
> Hugging Face。你(本地会话)没有这些限制,负责把最后一公里跑完。

## 你要读的文件(都在本仓库 `webharbor-phet/dist/`)

| 文件 | 用途 |
| --- | --- |
| `SESSION-STATE.md` | 本文件 —— 全局状态 |
| `REVIEW-FIXES.md` | **主运行手册**,落地 review 修复的分步命令 + PR 回复草稿 |
| `0001-fix-phet_simulations-address-PR-29-review-feedback.patch` | review 修复补丁(4 文件,+93/−34) |
| `phet_simulations.tar.gz` | 新 seed DB 包(**只含 DB,不含图片**,见下方警告) |
| `APPLY.md` | 最初建站时的运行手册(历史参考) |
| `0001-feat-phet_simulations-add-new-site.patch` | 最初建站补丁(已合进 PR,历史参考) |

## 环境与坐标

- **GitHub fork**: `ZhongyangLi0124/WebHarbor`,PR 分支 `add-phet-simulations`
  = PR #29,基于上游 `aiming-lab/WebHarbor:main`。
- **PR 分支当前 head**: `b790166`(review 补丁就是基于它生成的)。
- **HF 数据集**: `ChilleD/WebHarbor`(用户 fork 后上传自己的副本)。
- **用户本地 clone**: `~/WebHarbor`(如果丢了就重新
  `git clone https://github.com/ZhongyangLi0124/WebHarbor.git`)。
- **用户本地 HF 资产包(带图片)**: `/Users/lzy/Downloads/phet_simulations (1).tar.gz`
  —— 这个是用户之前上传到 HF 的那份,含 `static/images/`。
- **新 seed DB 的 md5**: `48ca438b8ac6dab37a6503a4e5574503`。
- 依赖版本钉死在根 Dockerfile:Flask 3.1.0 / Flask-SQLAlchemy 3.1.1 /
  Flask-Login 0.6.3 / Flask-WTF 1.2.2 / Flask-Bcrypt 1.0.1 /
  Werkzeug 3.1.3 / SQLAlchemy 2.0.36。站点无 per-site requirements.txt。

## PR #29 的五条 review 意见 与 修复(均已在补丁里)

审阅者 MufanQiu,REQUEST CHANGES:

1. **BLOCKER** 首页渲染零仿真(index.html 忽略 most_played/featured/new_sims)
   → `index.html` 加了 Featured / New Sims / Most Played 三个 sim-card 栅格(20 张卡)。
2. **MAJOR** "New" 目录过滤器是摆设(后端忽略 release= 参数)
   → `simulations()` 现在读 `release=new`(is_new)和 `release=updated`
   (release_date ≥ 2024-01-01);复选框状态、chip、排序、分页都带该参数。
   任务 --21 可解:7 个 New,全是 2025 年。
3. **MAJOR** GET 请求变更数据库(play_count/download_count 每次 GET 自增)
   → 删掉 `simulation_detail` 和 `activity_detail` 里的自增;计数成为固定 seed。
4. **MINOR** 部分任务有歧义/多解:
   - --12 活动下载数并列 → 改为显式互异,唯一最大 = Net Force
     Investigation(8742),对应 Forces and Motion: Basics。
   - --28 所有版本都是 1.0.0 → 版本从 seed 常量确定性推导(47 种不同,
     Wave Interference = 1.5.2)。
   - --6 / --24 / --28 / --31 改写任务措辞,加约束使每题恰好一个答案
     (已程序化验证:Plinko Probability / Magnet and Compass /
     Wave Interference / Predator-Prey Dynamics)。
5. **BLOCKER** `.assets-revision` 钉在 `main`,但新站 tarball 只在 HF PR ref 上
   → 需重钉到 **HF PR 合并后的 merge commit SHA**(只能等 HF 侧合并后做,
   见下方步骤)。

已在云端复验:66/66 路由返回 200;reset 循环字节级幂等
(md5 `48ca438b8ac6dab37a6503a4e5574503`);同页连续两次 GET 数据库不变。

## 你(本地会话)要执行的步骤

### ⚠️ 头号陷阱:不要用交付的 tarball 直接覆盖 HF

`phet_simulations.tar.gz`(本目录里那个)**只含 instance_seed/,没有图片**。
用户站点的图片(`static/images/ui/*.svg`、`static/images/sims/*.png`)在
用户本地树和现有 HF tarball 里。正确做法:只从它解出 `.db`,然后在用户
的工作副本里用 `scripts/extract_assets.sh` **重新打包**(会连图片一起打进去),
再上传那个重打的包。

### 完整命令

```bash
cd ~/WebHarbor
git checkout add-phet-simulations
git log --oneline -1          # 确认 b790166

# 1. 应用 review 修复补丁
#    (若 ~/Downloads 没有,从下方 raw 地址下载)
git am ~/Downloads/0001-fix-phet_simulations-address-PR-29-review-feedback.patch

# 2. 恢复图片:解开用户本地的 HF 资产包(注意文件名有空格)
tar -xzf '/Users/lzy/Downloads/phet_simulations (1).tar.gz' -C sites/

# 3. 用新 seed DB 覆盖(从交付 tarball 只解出 .db)
tar -xzf ~/Downloads/phet_simulations.tar.gz -C sites \
    phet_simulations/instance_seed/phet_simulations.db
md5 sites/phet_simulations/instance_seed/phet_simulations.db
# macOS 用 md5;必须 = 48ca438b8ac6dab37a6503a4e5574503

# 4. 本地预览
cd sites/phet_simulations
python3 -m venv .venv
.venv/bin/pip install Flask==3.1.0 Flask-SQLAlchemy==3.1.1 Flask-Login==0.6.3 \
    Flask-WTF==1.2.2 Flask-Bcrypt==1.0.1 Werkzeug==3.1.3 SQLAlchemy==2.0.36
rm -rf instance && cp -a instance_seed instance
PORT=40015 .venv/bin/python app.py
# 浏览器验收:
#   http://127.0.0.1:40015/                      → 首页三个栅格,图片正常
#   http://127.0.0.1:40015/simulations?release=new → 7 results + NEW chip
#   http://127.0.0.1:40015/simulation/wave-interference → v1.5.2,刷新两次 Plays 不变
cd ../..    # 回到仓库根

# 5. 预览满意后:重打包资产(含图片)并上传到用户 HF fork
mkdir -p ../wh-static-pr
./scripts/extract_assets.sh ../wh-static-pr/ phet_simulations
tar -tzf ../wh-static-pr/phet_simulations.tar.gz    # sanity:必须能看到 static/images/*
hf upload <用户HF用户名>/WebHarbor \
    ../wh-static-pr/phet_simulations.tar.gz \
    phet_simulations.tar.gz --repo-type dataset
# 请用户在 HF 上合并该 PR,拿到 merge commit SHA

# 6. 重钉 .assets-revision 到 HF merge SHA(解决 blocker #5)
sed -i.bak "s/^revision:.*/revision: <HF-MERGE-SHA>/" .assets-revision && rm .assets-revision.bak
git add .assets-revision
git commit -m "chore(phet_simulations): pin assets to merged HF revision <HF-MERGE-SHA>"

# 7. push,PR 自动更新
git push origin add-phet-simulations
```

### 顺序注意

第 6 步的重钉需要第 5 步的 HF merge SHA。如果维护者想 HF 和 GitHub 一起
合,可以先 push 第 1+7 步(`.assets-revision` 暂留 main),在 PR 回复里说明
"repin commit 会在 HF 侧合并后立刻跟上",让维护者确认顺序。

## 下载地址(补丁/文档在用户个人站仓库上)

Base raw URL:
`https://raw.githubusercontent.com/ZhongyangLi0124/zhongyangli0124.github.io/claude/evaluate-docker-setup-KFBgT/webharbor-phet/dist/`

- `0001-fix-phet_simulations-address-PR-29-review-feedback.patch`
- `phet_simulations.tar.gz`(新 seed，只含 DB)
- `REVIEW-FIXES.md`(含 PR 回复英文草稿)

## PR #29 回复草稿(落地后贴到 PR)

见 `REVIEW-FIXES.md` 末尾 "Draft reply to post on PR #29",把
`<HF-MERGE-SHA>` 替换为真实值后粘贴。
