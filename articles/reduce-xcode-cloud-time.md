---
title: "クラシルリワードにおけるXcodeCloudの実行時間削減対応まとめ（2025/04）"
emoji: "⏱️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [
  "ios",
  "xcode",
  "xcodecloud"
]
published: true
published_at: "2025-04-30 18:00"
publication_name: "dely_jp"
---

こんにちは。dely株式会社のiOSエンジニア [uetyo](https://x.com/psnzbss) です！

この記事では、2025年4月現在までに行ったクラシルリワードにおけるXcodeCloudの実行時間削減対応についてまとめます。


## 実行時間削減の経緯
クラシルリワードのiOSアプリ開発では、XcodeCloudを使ったCI/CD環境を構築しています。  
プロジェクトはSwift Package Managerによるマルチモジュール構成で、各モジュールごとにテストターゲットも分割しています[^1]。

XcodeCloudは実行環境として**Intel系のMacを使用している**ことが知られていますが、Apple Silicon系のMacに比べてビルド時間がかなり長くなる傾向があります。クラシルリワードにおいてもチームメンバーや機能開発の増加に伴い、XcodeCloudの**実行時間が生産性に大きく影響**を与えるようになってきました。

そこで、あらゆる方面からXcodeCloudの実行時間を削減するための対応を行いました。


## 対応内容
※ 一部の内容はクラシルリワードのプロジェクト構成に合わせて作成しているため、他のプロジェクトではそのまま利用できない場合があります。

### 1. 適切なキャッシュの利用設定
XcodeCloudのワークフロー設定を見直しました。

XcodeCloudのワークフロー設定では、利用するリポジトリの選択やXcode、MacOSのバージョン指定、**クリーンビルドをするかどうか**を設定できます。以下の画像のようにチェックを外すことで、クリーンビルドを利用しない（= キャッシュを利用する）ようにできます。

![](/images/reduce-xcode-cloud-time/xcode_cloud_clean_build.png)

:::message
なお、TestFlightに配信するビルドやAppStoreConnectに配信するビルドでは**クリーンビルドすることが推奨**されています[^2]。
:::

### 2. 無駄なビルドをスキップ
XcodeCloudのワークフローを実行するかどうかの条件も見直しました。

XcodeCloudをPRが作成された際に実行するためには「開始条件→プル要求の変更」を設定する必要があります。ただし、初期設定ではドキュメントの変更等の本来ビルドもテストも不要なものでもワークフローが実行されてしまいます。

そこで以下のように、実行する条件をカスタムルールによって細かく設定することで実行時間の削減が可能です。

![](/images/reduce-xcode-cloud-time/xcode_cloud_run_build_custom_rule.png)

クラシルリワードでは、実行する条件を「アプリターゲットのディレクトリ配下が変更された場合」「xcodeproj, xcworkspace配下が変更された場合」「SPMのマルチモジュール配下が変更された場合」に設定しています。

また、ビルドを自動でキャンセルする設定も忘れずに設定しておくと安心です。

![](/images/reduce-xcode-cloud-time/xcode_cloud_run_build_pulls.png)

### 3. ビルド時間の短縮
地道にビルド時間の削減も行いました。

#### ビルドシステムの最適化
Xcode16より利用できる "**Explicitly Built Modules**" という機能を有効にします。

:::message
Xcode16の時点ではオプトインの機能になるため、導入する際は注意が必要です。
:::

WWDC24にて発表された新しいビルド機能で、モジュールのプリコンパイルを先に行っておくことで無駄なビルドを削減し、結果的にビルド時間も削減できるというものです。詳細については公式ドキュメント[^3]や参考にした資料[^4]を確認してください。

この対応によって、2分弱のビルド時間を削減することができました。

| Task                                 | Previous time | Explicity time |   reduced time |
| :----------------------------------- | ------------: | -------------: | -------------: |
| SwiftCompile                         |     1311.9900 |      1215.2200 |        96.7700 |
| SwiftEmitModule                      |      182.0170 |       169.0920 |        12.9250 |
| CompileAssetCatalog                  |      139.8910 |       133.6330 |         6.2580 |
| CompileC                             |       45.1475 |        40.0250 |         5.1225 |
| ScanDependencies                     |       45.0315 |        42.5430 |         2.4885 |
| Ld                                   |       24.2650 |        22.3180 |         1.9470 |
| SwiftDriver                          |       13.4345 |        13.1420 |         0.2925 |
| CodeSign                             |        8.4605 |         4.7145 |         3.7460 |
| RegisterExecutionPolicyException     |        6.7245 |         5.8680 |         0.8565 |
| GenerateAssetSymbols                 |        7.0685 |         2.1665 |         4.9020 |
| Copy                                 |        3.9320 |         3.5210 |         0.4110 |
| WriteAuxiliaryFile                   |        3.0495 |         3.0930 |        -0.0435 |
| ProcessInfoPlistFile                 |        2.5405 |         3.0890 |        -0.5485 |
| Touch                                |        1.5655 |         1.0330 |         0.5325 |
| CpResource                           |        1.4130 |         1.6890 |        -0.2760 |
| SwiftMergeGeneratedHeaders           |        0.4970 |         0.4225 |         0.0745 |
| SwiftDriver Compilation              |        0.1875 |         0.2655 |        -0.0780 |
| SwiftDriver Compilation Requirements |        0.0705 |         0.0130 |         0.0575 |
|                                      |               |                |                |
| **Total**                            |    1797.2900s |      1661.8500 | **-135.4370s** |

#### 推論時間の削減
次に、コードの推論時間の削減も行いました。

他社でも実践されている型推論時間の削減方法を参考に、型推論に時間がかかる箇所を特定し、必要のない型推論を削減することでビルド時間の削減を行いました[^5]。

Swift Package Manager の場合、Build Arguments に `-Xfrontend -warn-long-expression-type-checking=100` を追加することで、型推論に100ms以上かかる場合に警告を出すことができます。これを利用して、型推論に時間がかかっている箇所を特定し、必要のない型推論を削減することでビルド時間の削減をしました。

例えば以下のように修正することで型推論の時間削減が可能です。

```swift
// 🤔
let data: SomeType = .init(
  id: UUID()
)

// ✅ .init より実態を明示したほうが型推論が早い
let data = SomeType(
  id: UUID()
)
```

一つひとつは微々たる変更ですが、数百カ所の変更を行うことで、数十秒のビルド時間を削減できました。

### 4. アーカイブ時間の短縮
テスト前のアーカイブ時間も削減しました。

以下のスクリプトを実行することで、成果物の容量が削減され、アーカイブ時間の削減に繋がりました。この対応により、**CI時間を5分削減**できました。

```bash
# ci_post_xcodebuild.sh

if [[ $CI_WORKFLOW = "Test" ]]; then
  find "$CI_TEST_PRODUCTS_PATH/Binaries/0/Debug-iphonesimulator" \
    -maxdepth 1 \
    \( -type d -name "*.dSYM" -o \
      -type f -name "*.o" -o \
      -type d -name "*.bundle" -o \
      -type f -name "*.a" \) \
    -exec rm -rf {} +
fi
```

詳細は以下のリンクに記載されています。

https://zenn.dev/dely_jp/articles/d6d84cdb8dc8de

### 5. 変更によって影響したテストのみ実行
テスト時間の削減も行いました。

テスト時に変更によって影響したモジュールのみをテストするようにスクリプトを作成しました。この対応により、**CI時間を最大18分削減**できました。

| 改善前                                                                               | 改善後                                                                                                    |
| ------------------------------------------------------------------------------------ | --------------------------------------------------------------------------------------------------------- |
| ![](/images/selective-testing-script/test-full-modules.png)<br> *全てのテストを実行* | ![](/images/selective-testing-script/test-selective-modules.png)<br> *関連するモジュールのテストのみ実行* |

詳細は以下のリンクに記載されています。

https://zenn.dev/dely_jp/articles/selective-testing-script


## まとめ
クラシルリワードにおけるXcodeCloudの実行時間削減対応についてまとめました。

XcodeCloudはBitriseやGitHubActionsに比べて実行時間あたりのコストが安いですが、利用されているマシンスペックが低いことや、スクリプト実行のタイミングが厳しく制限されているため、実行時間の削減が難しいという特徴があります。

こういった実行時間の削減方法が網羅的にまとめられている記事も少ないため、どなたかの参考になれば幸いです。

---

[^1]: [dely - Swift Package Managerを活用したクラシルリワードのiOSアプリ構成](https://tech.dely.jp/entry/2023/05/30/113128)
[^2]: [Apple Developer Documentation - Xcode Cloud workflow reference](https://developer.apple.com/documentation/xcode/xcode-cloud-workflow-reference#Perform-a-clean-build)
[^3]: [Apple Developer Documentation - Building your project with explicit module dependencies](https://developer.apple.com/documentation/xcode/building-your-project-with-explicit-module-dependencies)
[^4]: [giginet - 5分でわかるExplicitly Built Modules](https://speakerdeck.com/giginet/explicitly-built-modules-bullet-tour)
[^5]: [freddi - Swiftの型チェックが遅くなる理由を探求してみたときの話](https://engineering.linecorp.com/ja/blog/swift-compile-build-time)
