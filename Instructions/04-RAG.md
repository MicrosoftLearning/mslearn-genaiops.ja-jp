---
lab:
  title: RAG システムを調整する
---

## RAG システムを調整する

検索拡張生成 (RAG) システムは、大規模言語モデルの能力と効率的な取得メカニズムを組み合わせて、生成された応答の正確性と関連性を高めます。 オーケストレーション用の LangChain と AI 機能用の Azure AI Foundry を利用することで、データセットから関連情報を取得し、一貫性のある応答を生成する信頼性の高いパイプラインを作成できます。 この演習では、環境の設定、データの前処理、埋め込みの作成、インデックスの作成の手順を実行して、最終的に RAG システムを効果的に実装できるようにします。

## シナリオ

ホテルに関する推奨事項を提供するアプリをビルドしたいとします。 このアプリには、ホテルを推奨するだけでなく、ユーザーがそのホテルに関してしそうな質問に答えることができるエージェントを求めています。

生成型回答を提供するために GPT-4 モデルを選択しました。 ここでは、他のユーザー レビューに基づいてモデルにグラウンディング データを提供し、チャットの動作をパーソナル化された推奨事項に導く RAG システムを作成したいと考えています。

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
1. VS Code Explorer (左側のペイン) で、**Files/04** フォルダーのノートブック **04-RAG.ipynb** を開きます。
1. ノートブック内のすべてのセルを実行します。

## クリーンアップ

Azure AI サービスを確認し終わったら、不要な Azure コストが発生しないように、この演習で作成したリソースを削除する必要があります。

1. Azure portal が表示されているブラウザー タブに戻り (または、新しいブラウザー タブで [Azure portal](https://portal.azure.com?azure-portal=true) をもう一度開き)、この演習で使ったリソースがデプロイされているリソース グループの内容を表示します。
1. ツール バーの **[リソース グループの削除]** を選びます。
1. リソース グループ名を入力し、削除することを確認します。
