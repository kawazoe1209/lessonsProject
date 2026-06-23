# 第2回：Node / React / npm — フロントエンド開発のサプライチェーンリスク

## この回の目的

React / Node.js開発者が日常的に扱うnpmの仕組みを題材に、ライブラリ汚染のリスクが具体的にどこに存在するかを理解する。

さらに、lockfile運用を前提としても混入リスクは残ることは認識し、**混入確率を下げる運用と混入時の初動判断**を実務で行えるようになることを目指す。

最終的に、新しいnpmパッケージを追加するときのチェックリストと、依存更新PRのレビュー意思決定基準を確認する。

---

## 1. npmの依存関係の基本構造

### 1.1 package.jsonとdependencyの種類

```json
{
  "name": "my-react-app",
  "dependencies": {
    "react": "^18.3.1",
    "react-dom": "^18.3.1",
    "axios": "^1.7.0"
  },
  "devDependencies": {
    "vite": "^5.4.0",
    "eslint": "^9.0.0",
    "vitest": "^2.0.0"
  }
}
```

| 区分 | 役割 | リスクの観点 |
|------|------|-------------|
| dependencies | 本番実行時に必要 | 本番環境に入り込む |
| devDependencies | 開発・ビルド時に必要 | 開発者端末・CI/CD上で動作する |

**devDependenciesは本番には入らないが、開発者端末やCI/CD上では実行される。** ビルドツールやテストフレームワークに悪性コードが含まれていれば、ビルド時に認証情報を窃取できる。

### 1.2 semantic versioningとバージョン指定

```
"axios": "^1.7.0"
```

`^`（キャレット）は「メジャーバージョンが同じ範囲で最新」を意味する。

```
^1.7.0 → 1.7.0 以上 2.0.0 未満の最新
~1.7.0 → 1.7.0 以上 1.8.0 未満の最新
1.7.0  → 1.7.0 固定
```

バージョン範囲指定は、**lockfileがなければ**インストールのたびに異なるバージョンが入る可能性がある。

攻撃者がmaintainerアカウントを侵害し、バージョン範囲内の新バージョン（例：1.7.1）に悪性コードを含めて公開した場合、lockfileを更新するタイミングで取り込まれる。

**重要：lockfileはこのリスクを低減する必要条件だが、完全な防御ではない**

### 1.3 transitive dependency（推移的依存関係）

直接追加したパッケージ（direct dependency）が依存するパッケージがtransitive dependencyである。

```
自分が追加:  axios (direct dependency)
axiosが依存: follow-redirects, form-data, proxy-from-env (transitive)
form-dataが依存: asynckit, combined-stream, mime-types (transitive)
...
```

実際のReactアプリケーションでは、direct dependencyが30個でも、transitive dependencyを含めると500〜1,500個になることがある。

**自分が選んでいないパッケージが大量に信頼境界に入っている。**

重要な区分け：
- **direct dependency**：　追加の判断責任は開発者にある
- **transitive dependency**：　監査と運用で守る必要がある

> **この章のまとめ**
> 依存追加は「新しい機能を追加する」という開発作業ではなく、「信頼の連鎖を広げる」という**意思決定**である。

---

## 2. package-lock.jsonの役割と重要性

### 2.1 lockfileとは何か

package-lock.jsonは、依存解決の結果を固定するファイルである。

```
lockfileがない場合:
  npm install → その時点の最新（バージョン範囲内）を取得
  → 実行するたびに結果が変わる可能性がある

lockfileがある場合:
  npm install → lockfileに記録されたバージョンを取得
  → 誰がいつ実行しても同じ結果になる
```

lockfileは再現性のためだけでなく、**セキュリティ上も重要**である。lockfileがなければ、攻撃者がバージョン範囲内の悪性バージョンを公開するだけで、次回のnpm installで自動的に取り込まれる。

**重要な注意： lockfile運用は必要だが、以下の条件で想定外の解決が起きる可能性がある。**
- Node/npmのバージョン更新時の依存再解決
- lockfile更新操作時のhoist再計算
- ツールチェーン変更時の挙動差分

これらの場合、lockfileに記録されたバージョンと異なる版が入る余地が残る。

### 2.2 lockfileの差分を見る

lockfileが更新されたとき、PRの差分で以下を確認する。

```diff
package-lock.json の差分で見るべきポイント:

1. 意図していないパッケージが追加されていないか
+  "node_modules/unexpected-package": {
+    "version": "1.0.0",
+    "resolved": "https://registry.npmjs.org/unexpected-package/-/unexpected-package-1.0.0.tgz",

2. resolved URLが正規のregistryか
   "resolved": "https://registry.npmjs.org/..."  ← 正常
   "resolved": "https://unknown-registry.example.com/..."  ← 要確認

3. パッケージ数が大幅に増えていないか
   依存追加1個で transitive dependency が50個増えた → 要確認

4. 既存パッケージのバージョンが意図せず変わっていないか
   lockfileに記載の1.13.6のまま → 正常
   lockfileと無関係に1.14.1に上がっている → 要確認
```

### 2.3 npm ciとnpm installの違い

| コマンド | 動作 | 用途 | セキュリティ観点 |
|---------|------|------|------|
| `npm install` | package.jsonを元に依存解決し、lockfileを更新する | 開発時（新規依存追加時のみ） | 依存解決ロジックが複雑で意図外の変更が起きる可能性 |
| `npm ci` | lockfileを元に依存を取得する。lockfileと不整合があればエラー | CI/CD、開発時通常運用 | lockfileを信頼し、再現性を優先 |

**重要ルール：**
- **CI/CDでは`npm ci`を使う。** `npm install`をCI/CDで使うと、lockfileの意図しない更新が発生する可能性がある。
- **開発端末でも、lockfileがある場合は`npm ci`を基本手順にする。** チーム全員の環境を同一に保つため。
- lockfile更新が必要な場合のみ`npm install`を実行し、その結果を必ずレビューする。

### 2.4 直近事例から学ぶ： lockfile運用の落とし穴（2026/03/31 Axios汚染インシデント）

#### 事実

- 2026/03/31 9:21 に汚染版 `axios@1.14.1` が`npm registry`に公開される
- 2026/03/31 10:36 に特定の開発端末1台で `npm install` を実行してインストールされる
- ただし、そのプロジェクトの `package-lock.json` には `axios@1.13.6` が固定されていた
- 14:30 MDRサービスが感染端末から悪性通信先 (C2サーバー) への接続を検知
- 15:16 感染端末をネットワーク隔離
- 認証情報のローテーション、開発端末の初期化を実施
- 横展開の痕跡なし、情報漏洩リスクなし (Netskopeが悪性通信をブロック)

#### 推測 (技術詳細は検証中)

> **注意：以下は推測であり、完全に検証されていません。**

- Node 18 から 22 へのバージョンアップに伴い、npm自体もバージョンが更新された
- その直後が初めての `npm install` だった
- npm のバージョン更新に伴う依存関係の再解決やhoist再計算が、予期した版 (1.13.6) を取り込むのではなく、新しい版 (1.14.1) を取り込んだ可能性

#### 教訓

1. **lockfile管理は重要だが、ツールチェーン変更時は通常より慎重に**
   - Node/npmバージョン更新直後の依存操作は、lockfile再確認を必須にする
   - 更新直後は、すぐに `npm install` せず、チーム内で確認してから実施

2. **手順固定の重要性**
   - 開発端末でも、常時 `npm ci` を基本にすることで、想定外の変更を検知しやすくなる
   - lockfile がある場合に `npm install` を実行する特別な理由を明確にする

3. **差分監査の重要性**
   - 想定と異なるバージョン更新がlockfileに反映された場合、PR時点で検知できる仕組み
   - `npm install` の結果をマージ前に必ずレビュー

4. **更新直後の慎重運用**
   - 新しいツールチェーン、フレームワークで初めての依存操作をするときは、強化レビューを実施
   - lockfile差分の異常さや、新パッケージの混入を多重確認

---

## 3. npm install時に何が起きるか — lifecycle scripts

### 3.1 lifecycle scriptsとは

npmでは、パッケージのinstall時にスクリプトを自動実行する仕組みがある。

```json
{
  "name": "some-package",
  "scripts": {
    "preinstall": "node setup.js",
    "install": "node-gyp rebuild",
    "postinstall": "node postinstall.js"
  }
}
```

| script | 実行タイミング |
|--------|--------------|
| preinstall | パッケージのinstall前 |
| install | パッケージのinstall時 |
| postinstall | パッケージのinstall後 |

**これらはnpm installを実行するだけで自動的に実行される。** ユーザーに確認なく、パッケージに含まれるスクリプトが開発者端末やCI/CD上で動作する。

### 3.2 install scriptを悪用した攻撃

悪性パッケージのpostinstall scriptの例（概念的な説明）：

```
攻撃の流れ:
  1. npm install を実行
  2. postinstall script が自動実行
  3. 環境変数を列挙（HOME, NPM_TOKEN, AWS_ACCESS_KEY_ID 等）
  4. ~/.npmrc、~/.aws/credentials などのファイルを読む
  5. 収集した情報を攻撃者のサーバーにHTTPで送信
  6. 開発者はエラーも警告も見ない
```

この攻撃は、開発者がnpm installを実行するだけで成立する。パッケージのコードをimportしたり実行したりする必要すらない。

### 3.3 install scriptへの対策

#### ignore-scriptsの設定

npmのドキュメントでは、`ignore-scripts`の設定により、package.jsonに定義されたscriptsの実行を抑制できることが説明されている。

```ini
# .npmrc
ignore-scripts=true
```

```bash
# コマンドラインで指定
npm install --ignore-scripts
```

ただし、`ignore-scripts=true`にすると、正当なinstall scriptも動作しなくなる。ネイティブモジュール（node-gypによるビルドが必要なパッケージ）などは手動で対応が必要になる場合がある。

#### install scriptの有無を確認する

パッケージ追加前にinstall scriptの有無を確認する。

```bash
# パッケージのpackage.jsonを確認
npm view <package-name> scripts

# install後にnode_modules内を確認
cat node_modules/<package-name>/package.json | grep -A5 '"scripts"'
```

新しいパッケージを追加するときは、preinstall / install / postinstall scriptの有無を確認し、存在する場合はその内容を確認する。

---

## 4. npmのセキュリティ機能

### 4.1 npm audit

npm auditは、プロジェクトの依存関係に既知の脆弱性がないかを確認するコマンドである。

```bash
# 脆弱性を確認
npm audit

# JSON形式で出力
npm audit --json

# 本番dependencyのみ確認
npm audit --omit=dev
```

出力例：

```
┌───────────────┬──────────────────────────────────────────────┐
│ High          │ Prototype Pollution in lodash                │
├───────────────┼──────────────────────────────────────────────┤
│ Package       │ lodash                                       │
│ Dependency of │ some-framework                               │
│ Path          │ some-framework > some-util > lodash           │
│ More info     │ https://github.com/advisories/GHSA-xxxx-xxxx │
└───────────────┴──────────────────────────────────────────────┘
```

**注意点：** npm auditは既知の脆弱性のみを検出する。悪性パッケージ（意図的に仕込まれたマルウェア）がCVEとして登録されているとは限らない。npm auditだけで安全とは言えない。

### 4.2 npm provenance

npm provenanceは、パッケージがどこでビルドされ、どのソースコードから生成されたかを検証可能にする仕組みである。

```bash
# provenance情報を確認
npm audit signatures
```

provenanceが付いているパッケージは、以下を検証できる。

- ビルドがCI/CD（GitHub Actions等）で行われたこと
- ビルド元のソースコードリポジトリ
- ビルド時のcommit hash

ただし、provenanceの付与はパッケージ作成者の任意であり、すべてのパッケージに付いているわけではない。

### 4.3 npm Trusted Publishing

npm Trusted Publishingは、CI/CDからOIDCを使ってパッケージを公開する仕組みである。

npm公式ドキュメントでは、Trusted Publishingは短命でスコープされたcredentialをworkflow実行時に生成し、長期tokenを不要にする仕組みとして説明されている。

```
従来のnpm publish:
  1. npm token を生成（長期有効）
  2. CI/CD secrets に保存
  3. npm publish 時にそのtokenを使用
  → tokenが漏洩すると、誰でもパッケージを公開できる

Trusted Publishing:
  1. npm に GitHub リポジトリを登録
  2. GitHub Actions workflow が OIDC token を取得
  3. npm が OIDC token を検証し、短命の publish token を発行
  4. そのtokenでパッケージを公開
  → 長期tokenが不要。特定のworkflowからしか公開できない
```

自分たちがパッケージを公開する場合に関係する仕組みだが、利用するパッケージがTrusted Publishingを採用しているかどうかも、信頼性の判断材料になる。

---

## 5. React開発で注意すべきポイント

### 5.1 Create React App / Vite / Next.jsの依存関係

Reactアプリケーションの初期構成ツールが持つ依存関係も攻撃対象になる。

```bash
# Viteプロジェクトの依存関係数を確認
ls node_modules | wc -l

# transitive dependencyの確認
npm ls --all
```

テンプレートから生成した直後でも、数百のtransitive dependencyが含まれている。これらすべてが信頼境界に入っている。

### 5.2 よく使われるカテゴリのパッケージ

React開発で追加されがちなパッケージカテゴリと、追加時の確認観点：

| カテゴリ | 例 | 確認観点 |
|---------|-----|---------|
| UIコンポーネント | MUI, Ant Design, shadcn/ui | transitive dependency数、install script有無 |
| 状態管理 | Redux, Zustand, Jotai | maintainer、更新頻度 |
| データ取得 | axios, SWR, TanStack Query | セキュリティアドバイザリ |
| フォーム | React Hook Form, Formik | バージョン範囲 |
| 日付処理 | date-fns, dayjs, luxon | typosquattingリスク |
| ユーティリティ | lodash, ramda | サブパッケージのtyposquatting |
| テスト | Jest, Vitest, Testing Library | devDependencyでもCI上で実行 |
| Linter/Formatter | ESLint, Prettier | plugin経由のリスク |

### 5.3 ESLintプラグイン・Babelプラグインのリスク

ESLintプラグインやBabelプラグインは、ビルド時にコードを解析・変換するため、任意のコード実行が可能である。

```json
{
  "devDependencies": {
    "eslint-plugin-some-name": "^1.0.0",
    "@babel/plugin-some-name": "^7.0.0"
  }
}
```

これらは「devDependencyだから本番には関係ない」と思われがちだが、**開発者端末やCI/CD上でコードを実行する権限を持つ**。

---

## 6. 依存関係の調査と評価

### 6.1 パッケージを追加する前に確認すること

```bash
# パッケージの基本情報
npm view <package-name>

# maintainer情報
npm view <package-name> maintainers

# 依存関係
npm view <package-name> dependencies

# バージョン履歴
npm view <package-name> versions

# 最近のバージョン公開日
npm view <package-name> time
```

### 6.2 警戒すべきシグナル

以下のようなシグナルがある場合は、追加前に慎重に調査する。

| シグナル | 確認方法 | リスク |
|---------|---------|-------|
| パッケージ名のtypo | 正規名と比較 | typosquatting |
| maintainerの最近の変更 | npm view maintainers、GitHubリポジトリ | アカウント乗っ取り |
| 直近の大量リリース | npm view time | 侵害後の悪性バージョン連続公開 |
| install scriptの存在 | npm view scripts | install時のコード実行 |
| README/descriptionがない・不自然 | npm registry, GitHub | 偽パッケージ |
| GitHubリポジトリがない・空 | npm view repository | 偽パッケージ |
| ダウンロード数が極端に少ない | npm registry | 偽パッケージ |
| 依存関係が極端に多い | npm view dependencies | 攻撃面の拡大 |

### 6.3 OSV（Open Source Vulnerabilities）の活用

npm auditに加えて、OSV（Open Source Vulnerabilities）データベースを利用することで、より広範な脆弱性情報を確認できる。

```bash
# osv-scannerを使った確認
osv-scanner --lockfile package-lock.json
```

---

## 7. Dependabot / RenovateのPR対応

### 7.1 自動依存更新のリスク

DependabotやRenovateは、依存関係の更新PRを自動で作成する。これは脆弱性の修正を素早く取り込むために有用だが、以下のリスクがある。

```
リスクシナリオ:
  1. 正規パッケージのmaintainerが侵害される
  2. 悪性バージョンが公開される
  3. Dependabot/Renovateが自動でPRを作成
  4. 「dependencyの更新はいつものこと」としてレビューが甘くなる
  5. mergeされる
  6. CI/CDで悪性コードが実行される
```

### 7.2 依存更新PRのレビュー観点

依存更新PRを見るときの確認ポイント：

1. **何が更新されたか** — direct dependencyだけか、transitive dependencyも変わったか
2. **バージョンの変更幅** — patch/minor/majorのどれか。patchなのにlockfileの変更が大きい場合は要確認
3. **CHANGELOGの確認** — 更新内容が妥当か
4. **lockfileの差分** — 新しいパッケージが追加されていないか、resolvedのURLが正規か
5. **install scriptの変化** — 新たにinstall scriptが追加されていないか
6. **maintainerの変化** — パッケージのmaintainerが変わっていないか

---

## 8. private registryとscoped packages

### 8.1 dependency confusion対策

社内でprivate registryを使っている場合、dependency confusionのリスクがある。

```ini
# .npmrc での設定例

# scopeを使って社内registryを指定
@mycompany:registry=https://npm.mycompany.com/

# これにより @mycompany/utils は社内registryから取得される
# scopeなしのパッケージは public npm registry から取得される
```

**対策：**

- 社内パッケージにはscope（`@mycompany/`）を付ける
- `.npmrc`でscopeごとにregistryを指定する
- public registryに同名のパッケージが存在しないか確認する

---

## 9. Node系ライブラリ導入チェックリスト

この回の成果物として、以下のチェックリストを共有する。

### 新しいnpmパッケージを追加するとき

```
□ パッケージ名にtypoがないか（正規名と比較）
□ npm registryでパッケージ情報を確認したか
  - maintainerは自然か
  - GitHubリポジトリが存在し、活動があるか
  - ダウンロード数は妥当か
  - 最近のowner変更や大量リリースがないか
□ install script（preinstall/install/postinstall）がないか
  - ある場合、その内容を確認したか
□ transitive dependencyがどの程度増えるか確認したか
□ npm auditやOSVで既知脆弱性を確認したか
□ 同等の機能を持つ、より信頼性の高い代替がないか検討したか
□ そもそもライブラリ追加が必要か検討したか（自前実装で済む場合）
```

### 依存更新PRをレビューするとき

```
□ 何が更新されたか把握したか
□ バージョン変更幅と lockfile 変更量が釣り合っているか
□ 新しいパッケージが追加されていないか
□ resolved URLが正規のregistryか
□ install scriptが新たに追加されていないか
□ maintainerの変更がないか
□ CHANGELOGの内容が妥当か
```

### CI/CDでのnpm運用

```
□ npm ci を使っているか（npm install ではなく）
□ lockfileがリポジトリにcommitされているか
□ ignore-scriptsの使用を検討したか
□ npm auditをCIに組み込んでいるか
□ node_modules のキャッシュが安全に管理されているか
```

---

## 10. まとめと次回への接続

### この回で学んだこと

1. **npmの依存関係構造** — direct dependency, transitive dependency, lockfileの役割
2. **npm install時のリスク** — lifecycle scriptsによる自動コード実行
3. **npmのセキュリティ機能** — npm audit, provenance, Trusted Publishing
4. **依存関係の調査方法** — パッケージ追加前の確認観点
5. **依存更新PRのレビュー観点** — Dependabot/RenovateのPRを見るときの注意点
6. **Node系ライブラリ導入チェックリスト** — チームで共有する確認観点

### 次回予告：第3回 Java / Kotlin / Gradle / Maven

次回は、JVM系のバックエンド開発における依存関係リスクを扱う。

- npmとJVM系のリスクの違い
- Gradle plugin、Maven plugin の権限
- annotation processorのリスク
- repository設定とSNAPSHOT dependency
- Dependency Verification
- OWASP Dependency-Check
- Java/Kotlin依存関係レビュー観点の作成

---

## 確認問題

1. devDependenciesは本番に入らないから安全か。その理由を説明できるか
2. npm installとnpm ciの違いを、セキュリティの観点から説明できるか
3. postinstall scriptが危険な理由と、その対策を説明できるか
4. Dependabot/RenovateのPRをmergeするとき、何を確認すべきか
5. dependency confusionの攻撃原理と、scoped packagesによる対策を説明できるか
