---
lab:
  title: モデル カタログの言語モデルを比較する
---

## モデル カタログの言語モデルを比較する

ユース ケースを定義したら、モデル カタログを使用して、AI モデルによって問題が解決されるかどうかを調べることができます。 モデル カタログを使用してデプロイするモデルを選択し、それを比較して、ニーズに最も適したモデルを調べることができます。

この演習では、Azure AI Foundry ポータルのモデル カタログを使用して 2 つの言語モデルを比較します。

この演習には、約 **25** 分かかります。

## シナリオ

受講生が Python でコーディングする方法を学ぶのを手助けするアプリを構築したいとします。 このアプリでは、受講生がコードを記述して評価するのを手助けする自動チューターが必要です。 1 つの演習では、受講生は次の例の画像に基づいて、円グラフをプロットするために必要な Python コードを思い付く必要があります。

![数学 (34.9%)、物理 (28.6%)、化学 (20.6%)、英語 (15.9%) のセクションを含む試験で取得したマークを示す円グラフ](./images/demo.png)

画像を入力として受け入れ、正確なコードを生成できる言語モデルを選択する必要があります。 これらの条件を満たす使用可能なモデルは、GPT-4 Turbo、GPT-4o、GPT-4o mini です。

まず、Azure AI Foundry ポータルでこれらのモデルを操作するために必要なリソースをデプロイします。

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

## モデルを比較する

Azure によって完全に管理される推論インフラストラクチャを持つ、画像を入力として受け入れるモデルが 3 つあります。 ここでは、それらを比較して、ユース ケースに最適なものを決定する必要があります。

1. Web ブラウザー内で [Azure AI Foundry ポータル](https://ai.azure.com) (`https://ai.azure.com`) を開き、ご自身の Azure 資格情報を使用してサインインします。
1. メッセージが表示されたら、先ほど作成した AI プロジェクトを選択します。
1. 左側のメニューを使用して、**[モデル カタログ]** ページに移動します。
1. **[モデルの比較]** を選択します (検索ウィンドウのフィルターの横にあるボタンを見つけます)。
1. 選択したモデルを削除します。
1. 比較する 3 つのモデルを 1 つずつ追加します: **GPT-4**、**GPT-4o**、 **GPT-4o-mini**。 **GPT-4** については、選択したバージョンが **turbo-2024-04-09** であることを確認します。それが画像を入力として受け入れる唯一のバージョンだからです。
1. X 軸を **[正確性]** に変更します。
1. Y 軸が **[コスト]** に設定されていることを確認します。

プロットを確認し、次の質問に答えてみてください。

- *より正確なモデルはどれですか?*
- *どのモデルを使用する方が安いですか?*

ベンチマーク メトリックの正確性は、一般公開されている汎用データセットに基づいて計算されます。 プロットから、トークンあたりのコストは最も高くても正確性は最も高くないため、モデルの 1 つをまず除外できます。 最終決定する前に、ユース ケースに固有の残りの 2 つのモデルの出力の品質を調べてみましょう。

## ローカルの開発環境を設定する

実験と反復をすばやく行うために、Visual Studio (VS) Code で Python コードを含むノートブックを使用します。 VS Code をローカルの考案作業に使用する準備をしましょう。

1. VS Code を開き、次の Git リポジトリを**クローン**します: [https://github.com/MicrosoftLearning/mslearn-genaiops.git](https://github.com/MicrosoftLearning/mslearn-genaiops.git)
1. クローンをローカル ドライブに保存し、クローンによりできたフォルダーを開きます。
1. VS Code Explorer (左側のペイン) で、**Files/02** フォルダーのノートブック **02-Compare-models.ipynb** を開きます。
1. ノートブック内のすべてのセルを実行します。

## クリーンアップ

Azure AI サービスを確認し終わったら、不要な Azure コストが発生しないように、この演習で作成したリソースを削除する必要があります。

1. Azure portal が表示されているブラウザー タブに戻り (または、新しいブラウザー タブで [Azure portal](https://portal.azure.com?azure-portal=true) をもう一度開き)、この演習で使ったリソースがデプロイされているリソース グループの内容を表示します。
1. ツール バーの **[リソース グループの削除]** を選びます。
1. リソース グループ名を入力し、削除することを確認します。
