---
title: "XcodeCloudの実行時間を34分→16分に改善した"
emoji: "🛠️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [
  "ios",
  "xcodecloud"
]
published: true
published_at: "2025-04-24 12:00"
publication_name: "dely_jp"
---

こんにちは。dely株式会社のiOSエンジニア [uetyo](https://x.com/psnzbss) です！

この記事では、XcodeCloudの実行時間改善をAIと協業した結果、爆速で成果が出たので改善内容とAIの活用方法について紹介します 🤖

:::message
XcodeCloud の実行環境が Apple Silicon系のチップになれば丸く解決するのになぁ、というポエムではありません（2025/04/23現在、XcodeCloudはIntel系チップを搭載したマシンで実行されています）
:::


## 課題の整理
クラシルリワードのiOSアプリ開発では、XcodeCloudを使ったCI/CD環境を構築しています。  
プロジェクトはSwift Package Managerによるマルチモジュール構成で、各モジュールごとにテストターゲットも分割しています（[詳細](https://tech.dely.jp/entry/2023/05/30/113128)）。

[以前の改善](https://zenn.dev/dely_jp/articles/d6d84cdb8dc8de#%E5%89%8D%E6%8F%90)でXcodeCloudのテスト実行時間を**24分→19分**まで短縮しましたが、機能追加やテスト拡充により平均実行時間が**34分**まで悪化してしまいました。

直近の開発メンバー増加に伴い、CIの待ち時間が開発効率の大きなボトルネックとして顕在化する中、実行時間短縮が急務となっていたため、再度の改善に取り組みました。


## XcodeCloudの実行時間削減対象の決定
まずは現状の把握を詳細に行いました（異なるテストターゲット、ブランチにて5回ほど計測）

| 項目                                   | 平均継続時間  |
| -------------------------------------- | ------------- |
| Fetch source code                      | 28.0秒        |
| Run ci_post_clone.sh script            | 2.2秒         |
| Resolve package dependencies           | 2分38.0秒     |
| Run ci_pre_xcodebuild.sh script        | 0.7秒         |
| Run xcodebuild build-for-testing       | 10分54.9秒    |
| Check project configuration            | 1分47.2秒     |
| Run ci_post_xcodebuild.sh script       | 1.3秒         |
| 🤔 Run xcodebuild test-without-building | **9分38.9秒** |
| 🤔 Generate test report                 | **46.3秒**    |
|                                        |               |
| **合計平均実行時間**                   | 34分19.5秒    |

この結果より、今回は `Run xcodebuild test-without-building`, `Generate test report` の2つの項目を削減対象として考えることにしました。

:::details 目星をつけた背景
- Fetch source code
  - ✅ 削減インパクトが少ない
  - XcodeCloud側の通信速度と処理に依存するためほぼ改善できない認識
  - 今後もリポジトリサイズは肥大化していくので、XcodeCloud側で改善するより、GitHub/Repository側で改善すべき
- Run ci_post_clone.sh script
  - ✅ 問題ではない
- Resolve package dependencies
  - ⚠️ 可能であれば削減したい
  - SPMでライブラリを導入している関係で肥大化は避けられない
  - 各ライブラリをPre-Buildしておき、ダウンロードすることで改善できる可能性もあるが、できる限り依存解決等もSPM側に任せたいため手を加えない
- Run ci_pre_xcodebuild.sh script
  - ✅ 問題ではない
- Run xcodebuild build-for-testing
  - ⚠️ 改善したい
  - XcodeCloud のビルド環境（スペック）が向上すれば容易に改善可能
  - モジュール構成やビルド設定の見直し、リアーキテクチャ等も対象に検討する
  - 今回はXcodeCloud側で改善できることに絞るため、手を加えない
- Check project configuration
  - ✅ 削減方法が不明なため対象外
- Run ci_post_xcodebuild.sh script
  - ✅ 問題ではない
- Run xcodebuild test-without-building
  - ❌ 改善対象
  - テスト実行数を削減すれば短縮可能
- Generate test report
  - ❌ 改善対象
  - テスト実行数を削減されることで結果的に短縮可能
:::


## XcodeCloudの実行時間削減アプローチ
`Run xcodebuild test-without-building`, `Generate test report` の実行時間、つまりはテストの実行時間を削減するため、CI実行時に毎回すべてのテストを実行するのではなく、**変更のあったモジュールに関連するテストのみを実行するように制御する**方法を採用しました。

クラシルリワードでは、Appに近いモジュール（Feature）により多くのユニットテストが記載されているため、このアプローチを採用することでテスト対象がかなり抑えられ、実行時間の大幅な短縮が見込めます。

※ DesignSystemモジュールに変更があった場合のテスト対象例（黄色背景が実行対象）

| 改善前                                                                       | 改善後                                                                                            |
| ---------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------- |
| ![](/images/selective-testing-script/test-full-modules.png)<br> *全てのテストを実行* | ![](/images/selective-testing-script/test-selective-modules.png)<br> *関連するモジュールのテストのみ実行* |

この制御を実現するためカスタムスクリプトを実行する必要があります。XcodeCloud では特定のタイミングでのみ独自のスクリプトを実行して挙動を調整することができます。

![](/images/selective-testing-script/xcode-cloud-ciscripts.png)
*https://developer.apple.com/documentation/xcode/writing-custom-build-scripts*

今回は `Pre-XcodeBuild` のタイミングで、変更のあったモジュールの特定、XCTestPlanを書き換えるスクリプトを実行することで実現しました。

:::message
当初は [mikeger/XcodeSelectiveTesting](https://github.com/mikeger/XcodeSelectiveTesting) というツールを用いて実現しようとしましたが、クラシルリワードのプロジェクト構成に合わなかったため、自作することにしました。
:::


## AIと共に作る自作スクリプト
AIを積極的に活用して改善していきます。

### GitHub API を利用したルートコミットハッシュの取得
XcodeCloud では実行されているブランチ名やベースブランチ名、PullRequestの番号を環境変数として取得することができます。

https://developer.apple.com/documentation/xcode/environment-variable-reference

XcodeCloud のワークスペースにも .git は存在しており、実行対象のブランチのデータ（ブランチを作成したルートコミット）を取得できそうですが、XcodeCloudがCloneされた段階で**単一のブランチしか存在せず（ローカル、リモート共に）タグも存在しない**ため、ベースブランチ名からルートコミットを取得することはできません。

そのため GitHub API を使用してベースブランチのコミットハッシュを取得する必要があります。幸い、GitHubのAPIが非常に充実しており、以下のコマンドで取得できます。

```bash
ROOT_COMMIT_HASH=$(curl -s -H "Authorization: token ${GITHUB_TOKEN}" "https://api.github.com/repos/${CI_PULL_REQUEST_TARGET_REPO}/pulls/${CI_PULL_REQUEST_NUMBER}" | jq -r '.base.sha')
```

### ルートコミットから最新コミットまでの差分を取得
ルートコミットから現在の最新コミットまでの差分を取得します。変更によって影響を受けたモジュールの特定ができればよいため、Pathのみ取得します。  
このとき `Package.swift` の変更は不要なため除外しておきます。

```bash
git diff --name-only ${ROOT_COMMIT_HASH} HEAD
```

### 差分より影響を受けたターゲットの特定
:::message
※ 以下はクラシルリワードのPackage.swiftに合わせて構築しているため、他のプロジェクトでは適宜修正が必要です。  
※ 実際に構築する場合は以下の流れをChatGPT等に読み込ませつつ、利用しているPackage.swiftを読み込ませることでスクリプトが生成されるかと思います。
:::

#### 1. Package.swift の構成情報を取得
Package.swiftを読み込み、 Target と TestTarget の `name`, `path（存在する場合は）` を適切に取得します（Regex、Python、SwiftSyntaxParser等）。結果は再利用しやすいようにJSON形式で出力しました。

※ 本来は `swift package` コマンド等で取得したいところですが、pathの情報が取得できなかったため、今回は自作しました。

:::details 取得結果
```json
[
    {
        "name": "NetworkClient",
        "path": "Sources/NetworkClient/Core"
    },
    {
        "name": "ConcurrencyUtils",
        "path": "Sources/ConcurrencyUtils"
    },
    {
        "name": "MemoryTestHelper"
    }
]
```
:::

#### 2. 差分から変更されたターゲットを特定
上記で取得した差分とTargetの情報を元に、変更されたターゲットを特定します。  
ターゲット名とターゲットのPathが差分のPathと一致する場合は、ターゲットが特定できたことにします。

:::details 取得結果
```json
[
    {
        "file": "AppFoundation/Sources/UIComponents/Generated/Icon.swift",
        "name": "UIComponents"
    }
]
```
:::

#### 3. ターゲットの依存関係を取得
Package.swiftに定義されたターゲットデータを取得します。このデータは `swift package dump-package` コマンドを実行することで取得できます。

```bash
swift package dump-package
```

取得したJSONデータは不要なパラメータも多く付与されているため、必要最低限にクレンジングします。

:::details クレンジング結果
```json
[
    {
        "name": "RewardEventDispatcher",
        "dependencies": [
            "BaseExtension",
            "FeatureModel"
        ]
    },
    {
        "name": "AdEventDispatcher",
        "dependencies": [
            "AnalyticsLogger"
        ]
    },
    {
        "name": "AffiliateAppDispatcher",
        "dependencies": [
            "TrackingService",
            "InstallValidationService"
        ]
    }
]
```
:::

#### 4. ターゲットの依存関係を1対1でマッピング
ターゲット名にTestと付与されているものと、依存（dependencies）を1対1でマッピングします。

:::details マッピング結果
```json
[
    {
        "target_name": "NetworkCore",
        "test_target_name": "NetworkCoreTests"
    },
    {
        "target_name": "AdSkipperService",
        "test_target_name": "AdSkipperServiceTests"
    },
    {
        "target_name": "LocalStorageClient",
        "test_target_name": "LocalStorageClientTests"
    },
    {
        "target_name": "FlyerFeature",
        "test_target_name": "FlyerFeatureTests"
    },
    {
        "target_name": "CommonViewModel",
        "test_target_name": "CommonViewModelTests"
    },
    {
        "target_name": "AppEntryFeature",
        "test_target_name": "AppEntryFeatureTests"
    }
]
```
:::

例えば `HogeTests` というテストターゲットが依存先として `FugaModel`, `FooLogic`, `BarService` を持っている場合は、3つのマッピング結果を生成します。
```json
[
    {
        "target_name": "FugaModel",
        "test_target_name": "HogeTests"
    },
    {
        "target_name": "FooLogic",
        "test_target_name": "HogeTests"
    },
    {
        "target_name": "BarService",
        "test_target_name": "HogeTests"
    }
]
```

#### 5. 変更されたターゲットから依存するターゲットを特定
クレンジング済みのターゲットデータと変更されたターゲットデータを再帰的に処理し、依存ターゲットを特定していきます。もし変更したファイルがどのモジュールにも属さない場合は、空配列を返却してください。

変更されたターゲット名を Seed（再帰処理の起点）として、クレンジング済みのターゲットデータの name と一致するものを特定し、そのターゲットに紐づく dependencies を seed に追加し、全ての seed が出現しきるまでループ処理することで特定できます。

:::details Pythonでの例
```python
def recursive_extract_targets(seeds, targets):
    """
    seeds (set) を初期シードとして、targets の各オブジェクトについて
    'name' または 'dependencies' にシードが含まれているものを再帰的に抽出します。
    重複なく抽出されたターゲットオブジェクトのリストを、名前順にソートして返します。
    """
    extracted = {}
    updated = True
    while updated:
        updated = False
        for target in targets:
            tname = target.get("name")
            if not tname:
                continue
            if tname in extracted:
                continue
            deps = target.get("dependencies", [])
            # 'name' が seeds に含まれるか、dependencies に seeds のいずれかが含まれている場合
            if tname in seeds or any(dep in seeds for dep in deps):
                extracted[tname] = target
                if tname not in seeds:
                    seeds.add(tname)
                    updated = True
    result = list(extracted.values())
    result.sort(key=lambda x: x.get("name", ""))
    return result
```
:::

#### 6. 変更されたターゲットに依存するテストターゲットを特定
上記の処理により、変更されたファイルに影響するターゲットが特定できたため、テストターゲットも特定できます。

4でマッピングした結果を利用することで、変更されたターゲットに依存するテストターゲットを特定できます。

:::message alert
5の処理で空配列が返却された場合、XCTestPlanの実行対象がなくなります。  
テスト対象が存在しない場合、CIが失敗するため、`DummyTarget` などのダミーターゲットを用意しておくと良いかもしれません。
:::

:::details 変更によって影響を受けたターゲットのテストターゲット一覧
```json
{
  "test_targets": [
    "AdAcquisitionCompleteTests"
  ]
}
```
:::

::: details XCTestPlanの上書き後
```json
{
  "configurations": [
    {
      "id": "XXXXXXXX",
      "name": "Configuration 1",
      "options": {}
    }
  ],
  "defaultOptions": {
    "codeCoverage": false,
    "maximumTestRepetitions": 10,
    "testRepetitionMode": "retryOnFailure"
  },
  "testTargets": [ // ここから下の部分を書き換える
    {
      "target": {
        "containerPath": "container:",
        "identifier": "AdAcquisitionCompleteTests",
        "name": "AdAcquisitionCompleteTests"
      }
    }
  ],
  "version": 1
}
```
:::

### XCTestPlanの上書き
最後に、プロジェクトのXCTestPlanを6で特定したテストターゲットに上書きすることで差分テストが実現できます。


## まとめ
上記のスクリプトをCIのテストにて実行するようにしたところ、XcodeCloudの実行時間が**34分→16分**まで短縮されました。

| 項目                                   | 改善後平均                   |
| -------------------------------------- | ---------------------------- |
| Fetch source code                      | 10.8秒                       |
| Run ci_post_clone.sh script            | 1.1秒                        |
| Resolve package dependencies           | 2分17.8秒                    |
| Run ci_pre_xcodebuild.sh script        | 13.7秒                       |
| Run xcodebuild build-for-testing       | 7分54.8秒                    |
| Check project configuration            | 1分36.8秒                    |
| Run ci_post_xcodebuild.sh script       | 1.0秒                        |
| ✅ Run xcodebuild test-without-building | 9分38.9秒 -> **1分9.5秒**    |
| ✅ Generate test report                 | 46.3秒 -> **5.6秒**          |
|                                        |                              |
| 合計実行時間                           | 34分19.5秒 -> **16分28.5秒** |

1ヶ月ほど運用した結果、実測値でも **50%** ほどCIの実行時間が削減され、非常に大きな成果となりました。
![](/images/selective-testing-script/asc-xcodecloud-selective-testing-result.png)
*3月は改善前と改善後のCIを並列で動かしていた関係で外れ値になっています*

:::message alert
アプリ側に近いモジュールを変更した場合、影響するモジュール数も少ないため実行時間が短いですが、アプリの基盤側（APIClient、EventLogger等）の場合は、影響するモジュール数が多くなるため、必ずしも実行時間が50％に削減されるわけではありません。
:::

:::message
念のため、毎日朝と夕方に全てのテストを実行することで差分テストによる安全性も担保しています。
:::

今回、ClineやChatGPTなどAIを活用して開発したところ、1日程度でスクリプトが完成しました！

むしろ、XcodeCloud側の仕様調査やAppleのドキュメントを読む時間のほうが長かった印象です。AppleのドキュメントについてはDeepResearchも利用しましたが、知りたい部分が書かれていないこともあり（XcodeCloud内のGit周りなど）調査に苦労しました。

日々の開発が格段に快適になったため、こういったAIがボトルネックを見つけにくい部分の改善活動などを今年は考えていきたいと思います。


## 参考
- [mikeger/XcodeSelectiveTesting](https://github.com/mikeger/XcodeSelectiveTesting)
- [hoangatuan/SPM-Selective-Testing](https://github.com/hoangatuan/SPM-Selective-Testing)
