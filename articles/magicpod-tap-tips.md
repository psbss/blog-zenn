---
title: "MagicPodで指定位置タップの座標指定を効率的に設定する"
emoji: "💯"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [
  "ios",
  "magicpod"
]
published: true
published_at: "2024-12-06 00:00"
publication_name: "dely_jp"
---

こんにちは。dely株式会社のiOSエンジニア、[uetyo](https://x.com/psnzbss)です！

この記事は「[ソフトウェアテストの小ネタ Advent Calendar 2024](https://qiita.com/advent-calendar/2024/software-testing-koneta)」の6日目の記事です。  
AIテスト自動化プラットフォーム『MagicPod』でテストケースを作成する際に「指定位置タップ」の「座標指定」を効率的に設定する方法を見つけたので紹介します📮

:::message
※多少パワー系💪な方法と思いつつ、当てずっぽうに行うよりも確実に効率が良くなったため紹介します
:::


## MagicPodとは
MagicPodは、iOSやAndroid、Webアプリを対象に実際にアプリを操作しながら事前に想定した挙動と一致するかを検証するためのE2Eテストツールです。基本的なタップやスワイプに加えて、テキストの入力、表示されている値が想定した値と一致するかなど一通りの動作を再現できます。

クラシルリワードでは、iOS・Android・Webの3つのプラットフォームでサービスを提供しています。特にiOSアプリに関しては週に3回を超える頻度でリリースを行っているため、MagicPodを導入することでリリース前のテスト工数を大幅に削減できました。

詳細な導入経緯については以下の記事で紹介しています。
[クラシルリワードにおける自動テストツール MagicPodの導入事例](https://tech.dely.jp/entry/rewards_qa)


## 指定位置タップ機能の利用経緯
クラシルリワードでは、広告表示にAppLovinのMaxという外部サービス・SDKを利用しています。

基礎体験として「ユーザが広告を閲覧できること」「閲覧したあと意図した挙動をすること」は非常に重要であり、人力では途方もない時間がかかるためMagicPodで担保したい項目の一つでした。

![](/images/magicpod-tap-tips/applovin_ads_flow.png)

広告の再生処理フローは「完了待機」「スキップ」「完了」の3つのステップで行います。テストケースを作成してしばらくは安定して完了する状態でしたが、ある時から急に失敗するようになりました。

調査したところ、SDKの広告配信内容に変更があり、表示しているスキップボタンの位置を取得できなくなったようです（MagicPodではViewIdentifierとViewHierarchyを取得することで要素の位置を確定しています）。

この問題の解決に向けて、できれば利用したくなかったのですが「指定位置タップ機能」を利用することにしました（詳細は後述）。


## 指定位置タップ機能とは
指定位置タップ機能は、MagicPodのテストケース作成画面で指定したエリア（もしくは座標）にタップを行う機能です。

![](/images/magicpod-tap-tips/specific_area_tap.png)

現時点では以下のエリアをざっくりと指定できます（目安です。詳細は実際に設定してみてください🙏）

![](/images/magicpod-tap-tips/iphone_area_map.png)

中でも「座標指定」は、指定した座標にタップを行うことができるため非常に強力です。しかし設定も非常に難しく、「どこを起点としてX%なのか」「実際にどこをタップするのか分からない」という問題がありました。

![](/images/magicpod-tap-tips/unknown_tap_coordinate_area.png)


## 指定位置タップ機能の効率的な設定方法
ここからは実際にMagicPodを操作して設定しています。

まずはじめに、MagicPod内で一時的に利用するテストケースを作成し、対象の画面全体をタップエリアに設定します。  
検証したところ以下の要素を選択することでアプリ全体をタップ対象とすることができるようです。

```text
xpath=(//XCUIElementTypeApplication)[1]
```

![](/images/magicpod-tap-tips/magicpod_locator.png)

次に、シミュレーターでSafariを起動し、ウェブ検索で画面のタップ位置を可視化するウェブサービスを開きます。（今回は「[ちょっとしたスマホ（タッチパネル）チェック・テスト](https://anysweb.co.jp/touchcheck/)」の「お絵描き感覚フリーテスト」を利用しています。）

![](/images/magicpod-tap-tips/visualization_tap_target.png)

今回はスキップボタンの「>>」をタップできる位置が見つかるまでひたすらタップして検証し、目的の座標を特定しました。

![](/images/magicpod-tap-tips/visualization_tap_ads_skip_button.png)

最終的なワークフローは以下の通りです。

![](/images/magicpod-tap-tips/magicpod_complete_workflow.png)


## まとめ
- テスト成功率
  - 改善前：40％  
  - 改善後：99％
- MagicPodは強力なE2Eテストツール
- 画面レイアウトが取得できない場合は「指定位置タップ機能（座標）」が利用できる
- 座標設定は難しいが、効率的な設定方法を見つけた
- MagicPodのGUIが簡単になることを祈ります🙏
