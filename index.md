---
title: GenAI 運用の演習
permalink: index.html
layout: home
---

# GenAI 運用 (GenAIOps) ワークロード ラボ

次の実践的な演習では、GenAI 運用のパターンとプラクティスに関する実践的な環境を提供します。 Microsoft Foundry と Azure サービスを使用して、インフラストラクチャのデプロイ、プロンプトの管理、評価ワークフローの実装、運用 GenAI アプリケーションの監視について学習します。

> **注**:演習を完了するには、Azure AI サービスのプロビジョニングと、Microsoft Foundry ワークスペースのデプロイに十分なアクセス許可とクォータを持つ Azure サブスクリプションが必要です。 Azure サブスクリプションをお持ちでない場合は、新規ユーザーの無料クレジット付きで [[Azure アカウント]](https://azure.microsoft.com/free) に新規登録できます。

## クイック スタート演習

{% assign labs = site.pages | where_exp:"page", "page.url contains '/docs'" %} {% ラボでのアクティビティ用  %}
<hr>
### [{{ activity.lab.title }}]({{ site.github.url }}{{ activity.url }})

{{activity.lab.description}}

{% endfor %}

> **注**: これらの演習は単独でも完了できますが、[Microsoft Learn](https://learn.microsoft.com/training/paths/operationalize-gen-ai-apps/) のモジュールを補完するように設計されています。このモジュールでは、これらの演習の基になる概念の一部について詳しく説明しています。
