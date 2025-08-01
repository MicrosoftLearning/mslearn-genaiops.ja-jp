---
lab:
  title: prompty を使用してプロンプト エンジニアリングを探索する
  description: Prompty で、言語モデルを使用してさまざまなプロンプトをすばやくテストおよび改善し、最適な結果を得るためにそれらが構築および調整されていることを確認する方法について説明します。
---

## prompty を使用してプロンプト エンジニアリングを探索する

この演習は約 **45** 分かかります。

> **注**: この演習では、Azure AI Foundry に関するある程度の知識を前提としているため、よりアクティブな探索と実践的な学習を促すために、一部の手順は意図的にあまり詳細に説明されていません。

## はじめに

アイデア出しをする間に、言語モデルを使用してさまざまなプロンプトをすばやくテストして改善したいと考えています。 プロンプト エンジニアリングにアプローチするには、Azure AI Foundry ポータルのプレイグラウンドを使用したり、Prompty を使用して、よりコード優先のアプローチをとったりするなど、さまざまな方法があります。

この演習では、Azure AI Foundry を使用してデプロイされたモデルを使用して、Azure Cloud Shell で Prompty を使用したプロンプト エンジニアリングについて探索します。

## 環境を設定する

この演習のタスクを完了するには、以下が必要です。

- Azure AI Foundry ハブ
- Azure AI Foundry プロジェクト
- デプロイ済みのモデル (GPT-4o など)。

### Azure AI ハブとプロジェクトを作成する

> **注**:Azure AI プロジェクトが既にある場合は、この手順をスキップして既存のプロジェクトを使用できます。

Azure AI Foundry ポータルを使用して Azure AI プロジェクトを手動で作成したり、演習で使用するモデルをデプロイしたりできます。 ただし、[Azure Developer CLI (azd)](https://aka.ms/azd) でテンプレート アプリケーションを使用して、このプロセスを自動化することもできます。

1. Web ブラウザーで、`https://portal.azure.com`にある [Azure portal](https://portal.azure.com) を開き、自分の Azure 資格情報を使用してサインインします。

1. ページ上部の検索バーの右側にある **[\>_]** ボタンを使用して、Azure portal に新しい Cloud Shell を作成します。***PowerShell*** 環境を選択します。 Azure portal の下部にあるペインに、Cloud Shell のコマンド ライン インターフェイスが表示されます。 Azure Cloud Shell の使い方について詳しくは、[Azure Cloud Shell のドキュメント](https://docs.microsoft.com/azure/cloud-shell/overview)をご覧ください。

    > **注**: *Bash* 環境を使用するクラウド シェルを以前に作成した場合は、それを ***PowerShell*** に切り替えます。

1. PowerShell ペインで、次のコマンドを入力して、この演習のリポジトリを複製します。

     ```powershell
    rm -r mslearn-genaiops -f
    git clone https://github.com/MicrosoftLearning/mslearn-genaiops
     ```

1. リポジトリが複製されたら、次のコマンドを入力してスターター テンプレートを初期化します。

     ```powershell
    cd ./mslearn-genaiops/Starter
    azd init
     ```

1. プロンプトが表示されたら、新しい環境に名前を付けます。これは、プロビジョニングされたすべてのリソースに一意の名前を付けるために使用されます。

1. 次に、次のコマンドを入力してスターター テンプレートを実行します。 依存リソース、AI プロジェクト、AI サービス、オンライン エンドポイントを使用して AI ハブをプロビジョニングします。

     ```powershell
    azd up
     ```

1. メッセージが表示されたら、使用するサブスクリプションを選択してから、リソース プロビジョニング用に次のいずれかの場所を選択します。
   - 米国東部
   - 米国東部 2
   - 米国中北部
   - 米国中南部
   - スウェーデン中部
   - 米国西部
   - 米国西部 3

1. スクリプトの完了まで待ちます。通常、約 10 分かかりますが、さらに時間がかかる場合もあります。

    > **注**: Azure OpenAI リソースは、リージョンのクォータによってテナント レベルで制限されます。 上で一覧表示されているリージョンには、この演習で使用されるモデル タイプの既定のクォータが含まれています。 リージョンをランダムに選択すると、1 つのリージョンがクォータ制限に達するリスクが軽減されます。 クォータ制限に達した場合は、別のリージョンに別のリソース グループを作成する必要が生じる可能性があります。 詳しくは、[リージョンごとのモデルの可用性](https://learn.microsoft.com/en-us/azure/ai-services/openai/concepts/models?tabs=standard%2Cstandard-chat-completions#global-standard-model-availability)を参照してください

    <details>
      <summary><b>トラブルシューティングのヒント</b>: 特定のリージョンには使用可能なクォータがありません</summary>
        <p>選択したリージョンに使用可能なクォータがないためにいずれかのモデルに対してデプロイ エラーが発生した場合は、次のコマンドを実行してみてください。</p>
        <ul>
          <pre><code>azd env set AZURE_ENV_NAME new_env_name
   azd env set AZURE_RESOURCE_GROUP new_rg_name
   azd env set AZURE_LOCATION new_location
   azd up</code></pre>
        <code>new_env_name</code>、<code>new_rg_name</code>、および<code>new_location</code>を新しい値に置き換えます。 新しい場所は、演習の開始時に一覧表示されるリージョン (<code>eastus2</code>、<code>northcentralus</code>など) のいずれかである必要があります。
        </ul>
    </details>

1. すべてのリソースがプロビジョニングされたら、次のコマンドを使用して、エンドポイントとアクセス キーを AI サービス リソースに取り込みます。 `<rg-env_name>`と`<aoai-xxxxxxxxxx>`は、リソース グループと AI サービス リソースの名前に置き換える必要があることに注意してください。 どちらもデプロイの出力に印刷されます。

     ```powershell
    Get-AzCognitiveServicesAccount -ResourceGroupName <rg-env_name> -Name <aoai-xxxxxxxxxx> | Select-Object -Property endpoint
    Get-AzCognitiveServicesAccountKey -ResourceGroupName <rg-env_name> -Name <aoai-xxxxxxxxxx> | Select-Object -Property Key1
     ```

1. これらの値は、後で使用するのでコピーします。

### Cloud Shell で仮想環境を設定する

実験と反復処理をすばやく行うには、Cloud Shell で一連の Python スクリプトを使用します。

1. Cloud Shell のコマンド ライン ペインで、次のコマンドを入力して、この演習で使用するコード ファイルを含むフォルダーに移動します。

     ```powershell
    cd ~/mslearn-genaiops/Files/03/
     ```

1. 次のコマンドを入力して仮想環境をアクティブ化し、必要なライブラリをインストールします。

    ```powershell
   python -m venv labenv
   ./labenv/bin/Activate.ps1
   pip install python-dotenv openai tiktoken azure-ai-projects prompty[azure]
    ```

1. 次のコマンドを入力して、提供されている構成ファイルを開きます。

    ```powershell
   code .env
    ```

    このファイルをコード エディターで開きます。

1. コード ファイルで、**ENDPOINTNAME** と **APIKEY** のプレースホルダーを、前にコピーしたエンドポイントとキーの値に置き換えます。
1. プレースホルダーを置き換えたら、コード エディター内で、**Ctrl + S** コマンドを使用するか、**右クリックして保存**で変更を保存してから、**Ctrl + Q** コマンドを使用するか、**右クリックして終了**で、Cloud Shell コマンド ラインを開いたままコード エディターを閉じます。**

## システム プロンプトを最適化する

生成 AI の機能を維持しながらシステム プロンプトの長さを最小限に抑えることが、大規模なデプロイの基本です。 プロンプトが短いほど、AI モデルが処理するトークンが少なくなり、使用されるコンピューティング リソースも少なくなるので、応答時間を短縮できます。

1. 次のコマンドを入力して、指定されたアプリケーション ファイルを開きます。

    ```powershell
   code optimize-prompt.py
    ```

    コードをレビューし、既に定義済みのシステム プロンプトがある `start.prompty` テンプレート ファイルがスクリプトによって実行されることを確認してください。

1. `code start.prompty` を実行して、システム プロンプトをレビューします。 意図を明確かつ効果的に保ちながら、短縮する方法を検討してください。 次に例を示します。

   ```python
   original_prompt = "You are a helpful assistant. Your job is to answer questions and provide information to users in a concise and accurate manner."
   optimized_prompt = "You are a helpful assistant. Answer questions concisely and accurately."
   ```

   冗長な単語は削除し、重要な手順に焦点を当てます。 最適化されたプロンプトをファイルに保存します。

### 最適化のテストと検証

品質を損なうことなくトークンの使用を確実に減らすには、プロンプトの変更をテストすることが重要です。

1. `code token-count.py` を実行して、演習で提供されたトークン カウンター アプリを開いてレビューします。 上記の例で提供したものとは異なる最適化されたプロンプトを使用した場合は、このアプリでもそれを使用できます。

1. `python token-count.py` を使用してスクリプトを実行し、トークン数の違いを確認します。 最適化されたプロンプトでも高品質の応答が生成されることを確認してください。

## ユーザーの操作を分析する

ユーザーがアプリとどのようにやり取りするかを理解することは、トークンの使用を増やすパターンを特定するのに役立ちます。

1. ユーザー プロンプトのサンプル データセットをレビューします。

    - **"『*戦争と平和*』のあらすじを要約してください。"**
    - **"猫に関する豆知識を教えてください。"**
    - **"AI を使用してサプライ チェーンを最適化するスタートアップ企業向けの詳細なビジネス計画を作成してください。"**
    - **""Hello, how you?" をフランス語に翻訳してください。"**
    - **"量子もつれについて 10 歳に説明してください。"**
    - **"SF の短編ストーリー向けのクリエイティブなアイデアを 10 個教えてください。"**

    それぞれに対して、AI から**短い**、**中程度**、または**長い、もしくは複雑**な応答が得られる可能性があるかどうかを特定します。

1. 分類をレビューします。 どのようなパターンに気づきましたか? 以下を検討してください。

    - **抽象化のレベル** (クリエイティブなものか、事実に基づいたものか) が長さに影響していますか?
    - **オープンエンド型のプロンプト**は長くなる傾向がありますか?
    - **構造の複雑さ** (例: "私を 10 歳だと思って説明して") は応答にどのように影響しますか?

1. 次のコマンドを入力して、**optimize-prompt** アプリケーションを実行します。

    ```
   python optimize-prompt.py
    ```

1. 上記のサンプルの一部を使用して、分析を確認します。
1. ここで、次の長い形式のプロンプトを使用して、その出力を確認します。

    ```
   Write a comprehensive overview of the history of artificial intelligence, including key milestones, major contributors, and the evolution of machine learning techniques from the 1950s to today.
    ```

1. このプロンプトを次のように書き直します。

    - スコープを制限する
    - 簡潔にするための期待値を設定する
    - 書式設定または構造を使用して応答をガイドする

1. 回答を比較して、より簡潔な回答が得られたことを確認します。

> **注**:`token-count.py` を使用すると、両方の応答でのトークンの使用を比較できます。
<br>
<details>
<summary><b>書き換えられたプロンプトの例:</b></summary><br>
<p>"AI の歴史における 5 つの重要なマイルストーンを箇条書きの概要で教えてください。"</p>
</details>

## [**省略可能**] 実際のシナリオで最適化を適用する

1. 迅速に正確な回答を提供する必要があるカスタマー サポートのチャットボットを構築しているとします。
1. 最適化されたシステム プロンプトとテンプレートをチャットボットのコードに統合します (*出発点として `optimize-prompt.py` を使用できます*)。
1. さまざまなユーザー クエリを使用してチャットボットをテストし、効率的かつ効果的に応答することを確認します。

## まとめ

プロンプトの最適化は、生成 AI アプリケーションのコストを削減し、パフォーマンスを向上させるための重要なスキルです。 プロンプトの短縮、テンプレートの使用、ユーザー操作の分析を行うことで、より効率的でスケーラブルなソリューションを作成できます。

## クリーンアップ

Azure AI サービスを確認し終わったら、不要な Azure コストが発生しないように、この演習で作成したリソースを削除する必要があります。

1. Azure portal が表示されているブラウザー タブに戻り (または、新しいブラウザー タブで [Azure portal](https://portal.azure.com?azure-portal=true) をもう一度開き)、この演習で使ったリソースがデプロイされているリソース グループの内容を表示します。
1. ツール バーの **[リソース グループの削除]** を選びます。
1. リソース グループ名を入力し、削除することを確認します。
