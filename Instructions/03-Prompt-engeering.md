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

> **注**: Azure AI ハブとプロジェクトが既にある場合は、この手順をスキップして既存のプロジェクトを使用できます。

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
   
### ローカルの開発環境を設定する

実験と反復をすばやく行うために、Visual Studio (VS) Code で Prompty を使用します。 VS Code をローカルの考案作業に使用する準備をしましょう。

1. VS Code を開き、次の Git リポジトリを**クローン**します: [https://github.com/MicrosoftLearning/mslearn-genaiops.git](https://github.com/MicrosoftLearning/mslearn-genaiops.git)
1. クローンをローカル ドライブに保存し、クローンによりできたフォルダーを開きます。
1. VS Code の拡張機能ウィンドウで、**Prompty** 拡張機能を検索してインストールします。
1. VS Code エクスプローラー (左ペイン) で、 **Files/03** フォルダーを右クリックします。
1. ドロップダウン メニューの **[New Prompty]** を選択します。
1. **basic.prompty** という名前の新しく作成されたファイルを開きます。
1. 右上隅にある**再生**ボタンを選択して (または F5 キーを押して)、Prompty ファイルを実行します。
1. サインインを求められたら、**[許可する]** を選択します。
1. 自分の Azure アカウントを選択してサインインします。
1. VS Code に戻ると、エラー メッセージが表示された **[出力]** ペインが開きます。 エラー メッセージは、デプロイされたモデルが指定されていないまたは見つからないという内容になっているはずです。

このエラーを修正するには、Prompty で使用するモデルを構成する必要があります。

## プロンプト メタデータを更新する

Prompty ファイルを実行するには、応答の生成に使用する言語モデルを指定する必要があります。 メタデータは、Prompty ファイルの *frontmatter* で定義されます。 モデルの構成やその他の情報を使用してメタデータを更新しましょう。

1. Visual Studio Code ターミナル ペインを開きます。
1. **basic.prompty** ファイルをコピー (同じフォルダー内で) し、そのコピーの名前を `chat-1.prompty` に変更します。
1. **chat-1.prompty** を開き、次のフィールドをいくつかの基本情報を変更して更新します。

    - **名前:**

        ```yaml
        name: Python Tutor Prompt
        ```

    - **説明**:

        ```yaml
        description: A teaching assistant for students wanting to learn how to write and edit Python code.
        ```

    - **デプロイされたモデル**:

        ```yaml
        azure_deployment: ${env:AZURE_OPENAI_CHAT_DEPLOYMENT}
        ```

1. 次に、**azure_deployment** パラメーターの下に、API キーに対する次のプレースホルダーを追加します。

    - **エンドポイント キー**:

        ```yaml
        api_key: ${env:AZURE_OPENAI_API_KEY}
        ```

1. 更新した Prompty ファイルを保存します。

Prompty ファイルに必要なすべてのパラメーターが追加されましたが、一部のパラメーターではプレースホルダーを使用して必要な値を取得します。 プレースホルダーは、同じフォルダー内の **.env** ファイルに格納されます。

## モデル構成を更新する

Prompty で使用するモデルを指定するには、.env ファイルにモデルの情報を記載する必要があります。

1. **Files/03** フォルダー内の **.env** ファイルを開きます。
1. 各プレースホルダーを、以前に Azure portal のモデル デプロイの出力からコピーした値で更新します。

    ```yaml
    - AZURE_OPENAI_CHAT_DEPLOYMENT="gpt-4"
    - AZURE_OPENAI_ENDPOINT="<Your endpoint target URI>"
    - AZURE_OPENAI_API_KEY="<Your endpoint key>"
    ```

1. .env ファイルを保存します。
1. **chat-1.prompty** ファイルをもう一度実行します。

サンプル入力のみが使用されるため、シナリオとは無関係になりますが、AI によって生成された応答が得られるはずです。 テンプレートを更新して、AI 教育アシスタントにしましょう。

## sample のセクションを編集する

サンプル セクションでは、Prompty への入力を指定したり、入力がない場合に使用する既定値を指定したりします。

1. 次のパラメーターのフィールドを編集します。

    - **firstName**: 他の名前を選択します。
    - **context**: このセクション全体を削除します。
    - **question**: 設定されているテキストを次のように置き換えます。

    ```yaml
    What is the difference between 'for' loops and 'while' loops?
    ```

    **sample** のセクションは次のようになります。
    
    ```yaml
    sample:
    firstName: Daniel
    question: What is the difference between 'for' loops and 'while' loops?
    ```

    1. 更新された Prompty ファイルを実行し、出力を確認します。

