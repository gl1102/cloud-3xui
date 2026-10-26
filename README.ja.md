[简体中文](README.md) | [English](README.en.md) | [Русский](README.ru.md) | [日本語](README.ja.md) | [한국어](README.ko.md) | [Español](README.es.md) | [العربية](README.ar.md) | [Čeština](README.cs.md) | [Dansk](README.da.md) | [Deutsch](README.de.md) | [Esperanto](README.eo.md) | [فارسی](README.fa.md) | [Suomi](README.fi.md) | [Français](README.fr.md) | [Ελληνικά](README.gr.md) | [Magyar](README.hu.md) | [Bahasa Indonesia](README.id.md) | [Italiano](README.it.md) | [മലയാളം](README.ml.md) | [Nederlands](README.nl.md) | [Norsk](README.no.md) | [Polski](README.pl.md) | [Português](README.ptbr.md) | [Română](README.ro.md) | [Türkçe](README.tr.md) | [Українська](README.ua.md) | [Tiếng Việt](README.vn.md)

# ☁️ cloud-3xui — GitHub で作る無料 VLESS ノード

GitHub Actions の無料サーバーで 3x-ui パネル＋VLESS ノードを自動構築。固定ドメインで、切断時は自動で復旧します。

## アーキテクチャ

```mermaid
flowchart LR
    subgraph GitHub["GitHub Actions Runner (Ubuntu)"]
        XUI["3x-ui 面板\n:2053"]
        XRAY["Xray VLESS+WS\n:8080"]
    end
    subgraph CF["Cloudflare"]
        TUN["命名隧道\n(固定域名)"]
    end
    USER["用户客户端\nv2rayN / NekoBox 等"]

    XUI --> TUN
    XRAY --> TUN
    TUN --> USER

    style TUN fill:#f6821f,color:#fff
    style USER fill:#1f9d55,color:#fff
```

## 自動復旧フロー

```mermaid
flowchart TD
    A["手动 Run workflow\n(或上一轮自动触发)"] --> B["安装 3x-ui"]
    B --> C["API 创建 VLESS+WS 入站"]
    C --> D["启动 Cloudflare 命名隧道"]
    D --> E["输出节点链接"]
    E --> F["保活 N 分钟<br/>(手动输入，默认 290)"]
    F --> G["结束前提前触发下一轮<br/>(长运行提前 X 分钟,默认 5<br/>短运行提前 Y 秒,默认 30)"]
    G --> H["新旧 runner 重叠运行<br/>流量无缝切换"]
    H --> A

    style G fill:#f6821f,color:#fff
```

## 自動復旧

- **キープアライブ時間は自分で指定**：Run workflow をクリックするときに `run_minutes` を手動で入力します。入力した分数だけ実行され、空欄のままならデフォルトの 290 を使います
- **パラメータは自動継承**：各ラウンドは API 経由で `run_minutes` / `countdown_interval` / `countdown_unit` / `overlap_threshold` / `overlap_long` / `overlap_short` を次のラウンドに渡すため、チェーン全体で最初に入力した値が使われ続けます
- 一時的なテストなら `5` と入力し、日常利用では空欄のままデフォルトの 290 で使います
- **重複起動（シームレス切替）**：現在のラウンド終了前に次のラウンドを事前に開始します。新旧 2 つの runner が同時に動作し、Cloudflare トンネルは複数接続をサポートしているため、トラフィックは自動的に利用可能な接続に振り分けられ、旧ラウンド終了をユーザーはほとんど意識しません。開始タイミングは設定可能です：keep-alive が `overlap_threshold`（デフォルト 5 分）より長い場合は長時間ラウンドとみなされ、`overlap_long`（デフォルト 5 分）前に開始します。それより短い場合は短時間ラウンドとみなされ、`overlap_short`（デフォルト 30 秒）前に開始します
- ドメインと UUID は固定なので、クライアントは一度設定すればずっと使えます
- 止めたいときは Actions で Cancel をクリックすれば、自動で再起動されることはありません


## 固定アドレス

| 用途 | アドレス |
|------|----------|
| ノード | `vless.mallow.ccwu.cc:443`（VLESS + WebSocket + TLS） |
| パネル | `https://panel.mallow.ccwu.cc/panel`（ユーザー名 `admin`） |

> ⚠️ 上記のドメインは例です。変更するには、Settings → Secrets and variables → Actions → Variables で `VLESS_HOST` / `PANEL_HOST` の 2 つの変数を編集してください。ワークフローファイルを編集する必要はありません。

## ドメイン設定

ノードとパネルのドメインは GitHub 変数に保存されており、ワークフローファイルを編集する必要はありません。

| 変数 | 例 | 説明 |
|------|----|------|
| `VLESS_HOST` | `vless.mallow.ccwu.cc` | ノードのアドレス（完全なホスト名） |
| `PANEL_HOST` | `panel.mallow.ccwu.cc` | パネルのアドレス（完全なホスト名） |

**変更方法**：**Settings** → **Secrets and variables** → **Actions** → **Variables** タブ → **New repository variable** から、名前と値を入力します。変更は次回のワークフロー実行から有効になります。

**注意点（3つ）**：
- 各変数には**完全なホスト名**（サブドメインのプレフィックスを含む）が入るため、ノードとパネルは別々のドメインを指すことができ、同じ親ドメインである必要はありません
- 値は、ステップ 1 で Cloudflare トンネルに作成したパブリックホスト名（Public Hostname）と一致させる必要があります（`vless` → 127.0.0.1:8080、`panel` → 127.0.0.1:2053）。一致しないとリンクは使えません
- 変数が設定されていない場合、バナーにはその項目のプレースホルダー（`vless.YOUR_DOMAIN` / `panel.YOUR_DOMAIN`）が表示されます

## 詳細な設定手順

### 無料ドメイン（ドメインがない場合はこちら）

独自ドメインをお持ちでない？ https://my.dnshe.com/ で無料取得できます（例：`.ccwu.cc`、有効期間 10 年）。その後、ドメインの NS を Cloudflare に移管してください。

### ステップ1：Cloudflare で名前付きトンネルを作成


Cloudflare 公式サイト: https://www.cloudflare-cn.com （中国）/ https://www.cloudflare.com （国際版）
ダッシュボード: https://dash.cloudflare.com

```mermaid
flowchart TD
    A["控制台 → Zero Trust<br/>→ ネットワーク (Networks) → コネクター (Tunnels)"] --> B["右上角 创建隧道"]
    B --> C["选 Cloudflared<br/>(推荐) → 选择 Cloudflared"]
    C --> D["输入隧道名称<br/>(例：github-3xui)<br/>→ 保存隧道"]
    D --> E["安装并运行连接器页<br/>复制 Token<br/>(eyJ 开头的一长串)"]
    E --> F["下一步 → 路由隧道<br/>→ 已发布应用程序路由"]
    F --> G["添加已发布应用程序路由<br/>子域 vless + 你的域<br/>类型 HTTP → URL 127.0.0.1:8080"]
    F --> H["添加已发布应用程序路由<br/>子域 panel + 你的域<br/>类型 HTTP → URL 127.0.0.1:2053"]
    G --> I["保存完成"]
    H --> I

    style E fill:#f6821f,color:#fff
    style I fill:#1f9d55,color:#fff
```

詳細手順:

1. [Cloudflare Dashboard](https://dash.cloudflare.com) を開き、左メニューから **Zero Trust** → **ネットワーク (Networks)** → **コネクター (Tunnels)** に進む
2. 右上の **Create a tunnel** をクリック
3. **トンネルタイプの選択**: **Cloudflared**（推奨）を選び、**Select Cloudflared** をクリック
4. **トンネルに名前を付ける**: 名前（例: `github-3xui`）を入力し、**Save tunnel** をクリック
5. **コネクタのインストールと実行**: ページにインストールコマンドが表示されるので、`eyJ` で始まる長い **Token** をコピーして保存する（後で GitHub Secret に登録）。ローカルへのインストールは**不要**。そのまま次へ進む
6. **トンネルのルーティング**: **Public hostnames** タブを開き、**Add a public hostname** をクリックして 2 件追加する:

   **1 件目（ノード）:**
   - サブドメイン: `vless`
   - ドメイン: 自分のドメインを選択
   - タイプ: `HTTP`
   - URL: `127.0.0.1:8080`

   **2 件目（パネル）:**
   - サブドメイン: `panel`
   - ドメイン: 自分のドメインを選択
   - タイプ: `HTTP`
   - URL: `127.0.0.1:2053`
7. 保存する。DNS レコードは Cloudflare が自動作成する。コネクター (Tunnels) 一覧でステータスが **Healthy** になれば設定成功。


### ステップ2：UUID を生成

1. https://1024tools.com/uuid/ を開きます
2. 生成された UUID をコピーします（例：`cbfc229f-9d34-4fef-9990-8baa82a7327b`）
3. **この UUID は必ずメモしてください**。ワークフローのログには平文で表示されません。ノードリンクを組み立てるたびに必要になります

### ステップ3：GitHub に Secrets と Variables を追加

1. このリポジトリを開き、**Settings** タブをクリックします
2. 左側の **Secrets and variables** → **Actions** に入ります
3. 右上の **New repository secret** をクリックし、以下の3つを追加します：

   | Name | Secret（値） | 説明 |
   |------|-------------|------|
   | `CF_TUNNEL_TOKEN` | ステップ1でコピーした Tunnel Token | Cloudflare トンネルの認証情報 |
   | `PANEL_PASSWORD` | ご自身で設定したパネルパスワード | 3x-ui パネルのログインパスワード |
   | `VLESS_UUID` | ステップ2で生成した UUID | ノードの UUID、変更しません |

4. **Variables** タブに切り替え、**New repository variable** をクリックして、2 つの変数を追加します（上記の「ドメイン設定」セクションを参照）：

   | Name | 値 | 説明 |
   |------|----|------|
   | `VLESS_HOST` | `vless.mallow.ccwu.cc` | ノードのアドレス |
   | `PANEL_HOST` | `panel.mallow.ccwu.cc` | パネルのアドレス |

### ステップ4：ワークフローを実行

1. リポジトリ上部の **Actions** タブをクリックします
2. 左側で **3x-ui VLESS Node** を選択します
3. 右側の **Run workflow** ボタンをクリックします
4. 表示されたフォームで：
   - **Panel password**：空欄のまま（`PANEL_PASSWORD` secret を自動使用）
     パネルパスワード（空欄の場合は PANEL_PASSWORD secret を使用）
   - **Run duration**：デフォルト `290`（分）。初回テストでは `5` と入力して素早く確認できます。日常利用では空欄のままで構いません
   - **Countdown output interval**: カウントダウンの出力間隔、デフォルト `1`、`0` で無効化
   - **Countdown interval unit**: カウントダウン出力間隔の単位、ドロップダウンから `minutes` または `seconds` を選択
   - **Overlap threshold**: 重複起動のしきい値（分、デフォルト `5`）— keep-alive がこれより長い場合は長時間ラウンドとみなす
   - **Overlap long**: 長時間ラウンドで終了の何分前に次のラウンドを開始するか（デフォルト `5`）
   - **Overlap short**: 短時間ラウンドで終了の何秒前に次のラウンドを開始するか（デフォルト `30`）
     実行時間（分、デフォルト 290、最大 340）
5. 緑色の **Run workflow** をクリックして実行します

### ステップ5：ノードリンクを取得

1. 約2〜3分待ち、実行中のジョブをクリックして詳細ページを開きます
2. **Start named Cloudflare tunnel** ステップを展開し、ログ出力を確認します：
   ```
   ==================================================
     面板 Panel: https://panel.mallow.ccwu.cc/panel  (admin / 你的面板密码)

     VLESS 节点 node:
     vless://YOUR_UUID@vless.mallow.ccwu.cc:443?encryption=none&security=tls&sni=vless.mallow.ccwu.cc&type=ws&path=%2Fws#GitHub-3xui

     把 YOUR_UUID 换成你在 VLESS_UUID secret 里填的 UUID
     Replace YOUR_UUID with the UUID from your VLESS_UUID secret
     域名：去 Settings → Secrets and variables → Actions → Variables 改 VLESS_HOST / PANEL_HOST 变量
     Domain: set the VLESS_HOST / PANEL_HOST variables in Settings → Secrets and variables → Actions → Variables
   ==================================================
   ```
3. `vless://` リンクをコピーし、`YOUR_UUID` をご自身の UUID に置き換えます （ドメインは変数から取得した実際のものなので、置き換える必要はありません。）
4. v2rayN / v2rayNG / Streisand / NekoBox などのクライアントに貼り付ければ使えます

### ステップ6：自動復旧の確認（任意）

1. 初回実行時は実行時間を `5` 分に設定します
2. 5分後、最後の **Trigger next run** ステップのログを確認します。以下のように表示されるはずです：
   ```
   HTTP 204
   已触发下一次运行 Next run triggered
   ```
3. Actions の一覧に戻ると、新しく自動トリガーされた実行が表示されます
4. 問題なければ、実行時間を `290` に戻して再実行し、通常利用します

## ワークフローのステップ説明

| ステップ | 中文 | English | 説明 |
|----------|------|---------|------|
| Install 3x-ui | 安装 3x-ui | Install 3x-ui | 公式スクリプトでパネルを非対話インストール |
| Create VLESS+WS inbound via API | 通过 API 创建 VLESS+WS 入站 | Create VLESS+WS inbound via API | API Token でログインし、インバウンドを自動作成 |
| Start named Cloudflare tunnel | 启动 Cloudflare 命名隧道 | Start named Cloudflare tunnel | Token で固定ドメインのトンネルに接続 |
| Keep alive | 保活 | Keep alive | 指定した分数スリープし、サービスの状態を監視し、設定した間隔でカウントダウンを出力 |
| Trigger next run | 触发下一次运行 | Trigger next run | API を呼び出して次のラウンドを自動起動 |

## よくある質問

**Q: ログ中のノードリンクの UUID が `***` と表示されるのは？**
A: GitHub の secret 保護機能によるものです。`YOUR_UUID` を、`VLESS_UUID` secret に設定したご自身の UUID に置き換えてください。

**Q: ログ中のドメインが `YOUR_DOMAIN` と表示されるのは？**
A: Settings → Secrets and variables → Actions → Variables で、完全なホスト名を指定して `VLESS_HOST` / `PANEL_HOST` の 2 つの変数を作成してください。Cloudflare トンネルのパブリックホスト名と一致していることを確認してください。

**Q: クライアントのテストが通らない（-1）のは？**
A: 3点を確認してください：① UUID が正しく置き換えられているか；② クライアントのバージョンが十分新しいか（v2rayN 7.x 以上）；③ Cloudflare トンネルが正常に接続されているか（ログを確認）。

**Q: 自動再起動がトリガーされないのは？**
A: **Trigger next run** ステップのログを確認してください。204 以外の HTTP ステータスコードが表示されている場合は、リポジトリの Actions 権限設定を確認してください。

**Q: 5時間ごとに数分切断されるのは正常ですか？**
A: 重複起動を採用しています。現在のラウンド終了前に次のラウンドを事前に開始し、新旧 2 つの runner が同時に動作するため、理論上は無停止です。GitHub のキュー待ちなどの極端な状況では一時的な変動があるかもしれませんが、クライアントは自動で再接続します。

**Q: 自動復旧を止めたい場合は？**
A: Actions ページで現在の実行を Cancel するだけです。自動で再びトリガーされることはありません。

## 注意事項

- **速度**：トラフィックは Cloudflare → GitHub のデータセンター（米欧）を経由するため、一時的な利用・軽い利用にのみ適しています
- **シームレス切替**：重複起動を採用し、各ラウンド終了の 5 分前に次のラウンドを立ち上げます。新旧 runner が同時にオンラインになるため、理論上ユーザーは何も感じません。極端な場合（GitHub のキューなど）には短時間の変動が生じる可能性があります
- **コンプライアンス**：GitHub Actions でプロキシを動かすのは ToS 違反でアカウント停止になるリスクがあります。ご自身で判断してください
- **セキュリティ**：`CF_TUNNEL_TOKEN` / `PANEL_PASSWORD` / `VLESS_UUID` を公開の場に載せないでください
