---
lab:
  title: 合成データセットを使用してモデルを最適化する
---

## 合成データセットを使用してモデルを最適化する

生成 AI アプリケーションを最適化するには、データセットを利用してモデルのパフォーマンスと信頼性を向上させる必要があります。 合成データを使用することで、開発者は実際のデータに存在しない可能性があるさまざまなシナリオやエッジ ケースをシミュレートできます。 さらに、モデルの出力の評価は、高品質で信頼性の高い AI アプリケーションを得るのに不可欠です。 最適化と評価のプロセス全体は、これらのタスクを効率化するための信頼性の高いツールとフレームワークを提供する Azure AI Evaluation SDK を使用して効率的に管理できます。

## シナリオ

博物館での訪問者の体験を強化するために、AI 搭載スマート ガイド アプリをビルドしたいとします。 このアプリは、歴史的人物に関する質問に答えるのを目的としています。 アプリからの応答を評価するには、そのような人物とその業績のさまざまな側面をカバーする包括的な合成質問回答データセットを作成する必要があります。

生成型回答を提供するために GPT-4 モデルを選択しました。 ここでは、コンテキストに関連する対話を生成するシミュレーターを作成し、さまざまなシナリオで AI のパフォーマンスを評価したいと考えています。

まず、このアプリケーションをビルドするのに必要なリソースをデプロイしましょう。

## Azure AI ハブとプロジェクトを作成する

Azure AI Foundry ポータルを使用して Azure AI ハブとプロジェクトを手動で作成したり、演習で使用するモデルをデプロイしたりできます。 ただし、[Azure Developer CLI (azd)](https://aka.ms/azd) でテンプレート アプリケーションを使用して、このプロセスを自動化することもできます。

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

1. 次に、次のコマンドを入力してスターター テンプレートを実行します。 依存リソース、AI プロジェクト、AI サービス、オンライン エンドポイントを使用して AI ハブをプロビジョニングします。 また、モデル GPT-4 Turbo、GPT-4o、GPT-4o mini もデプロイします。

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

## ローカルの開発環境を設定する

実験と反復をすばやく行うために、Visual Studio (VS) Code で Python コードを含むノートブックを使用します。 VS Code をローカルの考案作業に使用する準備をしましょう。

1. VS Code を開き、次の Git リポジトリを**クローン**します: [https://github.com/MicrosoftLearning/mslearn-genaiops.git](https://github.com/MicrosoftLearning/mslearn-genaiops.git)
1. クローンをローカル ドライブに保存し、クローンによりできたフォルダーを開きます。
1. VS Code エクスプローラー (左側のペイン) で、**Files/06** フォルダーのノートブック **06-Optimize-your-model.ipynb** を開きます。
1. ノートブック内のすべてのセルを実行します。

## クリーンアップ

Azure AI サービスを確認し終わったら、不要な Azure コストが発生しないように、この演習で作成したリソースを削除する必要があります。

1. Azure portal が表示されているブラウザー タブに戻り (または、新しいブラウザー タブで [Azure portal](https://portal.azure.com?azure-portal=true) をもう一度開き)、この演習で使ったリソースがデプロイされているリソース グループの内容を表示します。
1. ツール バーの **[リソース グループの削除]** を選びます。
1. リソース グループ名を入力し、削除することを確認します。
