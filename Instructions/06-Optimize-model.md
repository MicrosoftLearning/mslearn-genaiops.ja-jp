---
lab:
  title: 合成データセットを使用してモデルを最適化する
  description: 合成データセットを作成し、それらを使用してモデルのパフォーマンスと信頼性を向上する方法について学習します。
---

## 合成データセットを使用してモデルを最適化する

生成 AI アプリケーションを最適化するには、データセットを利用してモデルのパフォーマンスと信頼性を向上させる必要があります。 合成データを使用することで、開発者は実際のデータに存在しない可能性があるさまざまなシナリオやエッジ ケースをシミュレートできます。 さらに、モデルの出力の評価は、高品質で信頼性の高い AI アプリケーションを得るのに不可欠です。 最適化と評価のプロセス全体は、これらのタスクを効率化するための信頼性の高いツールとフレームワークを提供する Azure AI Evaluation SDK を使用して効率的に管理できます。

この演習には約 **30** 分かかります\*

> \* この推定時間には、演習の最後にある省略可能なタスクは含まれません。
## シナリオ

博物館での訪問者の体験を強化するために、AI 搭載スマート ガイド アプリをビルドしたいとします。 このアプリは、歴史的人物に関する質問に答えるのを目的としています。 アプリからの応答を評価するには、そのような人物とその業績のさまざまな側面をカバーする包括的な合成質問回答データセットを作成する必要があります。

生成型回答を提供するために GPT-4 モデルを選択しました。 ここでは、コンテキストに関連する対話を生成するシミュレーターを作成し、さまざまなシナリオで AI のパフォーマンスを評価したいと考えています。

まず、このアプリケーションをビルドするのに必要なリソースをデプロイしましょう。

## Azure AI ハブとプロジェクトを作成する

Azure AI Foundry ポータルを使用して Azure AI ハブとプロジェクトを手動で作成したり、演習で使用するモデルをデプロイしたりできます。 ただし、[Azure Developer CLI (azd)](https://aka.ms/azd) でテンプレート アプリケーションを使用して、このプロセスを自動化することもできます。

1. Web ブラウザーで、`https://portal.azure.com`にある [Azure portal](https://portal.azure.com) を開き、自分の Azure 資格情報を使用してサインインします。

1. ページ上部の検索バーの右側にある **[\>_]** ボタンを使用して、Azure portal に新しい Cloud Shell を作成します。***PowerShell*** 環境を選択します。 Azure portal の下部にあるペインに、Cloud Shell のコマンド ライン インターフェイスが表示されます。 Azure Cloud Shell の使い方について詳しくは、[Azure Cloud Shell のドキュメント](https://docs.microsoft.com/azure/cloud-shell/overview)をご覧ください。

    > **注**: *Bash* 環境を使用するクラウド シェルを以前に作成した場合は、それを ***PowerShell*** に切り替えます。

1. Cloud Shell ツール バーの **[設定]** メニューで、**[クラシック バージョンに移動]** を選択します。

    **<font color="red">続行する前に、クラシック バージョンの Cloud Shell に切り替えたことを確認します。</font>**

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
     ```

     ```powershell
    Get-AzCognitiveServicesAccountKey -ResourceGroupName <rg-env_name> -Name <aoai-xxxxxxxxxx> | Select-Object -Property Key1
     ```

1. これらの値は、後で使用するのでコピーします。

## Cloud Shell で開発環境を設定する

実験と反復処理をすばやく行うには、Cloud Shell で一連の Python スクリプトを使用します。

1. Cloud Shell のコマンド ライン ペインで、次のコマンドを入力して、この演習で使用するコード ファイルを含むフォルダーに移動します。

     ```powershell
    cd ~/mslearn-genaiops/Files/06/
     ```

1. 次のコマンドを入力して仮想環境をアクティブにし、必要なライブラリをインストールします。

    ```powershell
   python -m venv labenv
   ./labenv/bin/Activate.ps1
   pip install python-dotenv azure-ai-evaluation azure-ai-projects promptflow wikipedia aiohttp openai==1.77.0
    ```

1. 次のコマンドを入力して、提供されている構成ファイルを開きます。

    ```powershell
   code .env
    ```

    このファイルをコード エディターで開きます。

1. コード ファイルで、**your_azure_openai_service_endpoint** および **your_azure_openai_service_api_key** プレースホルダーを、先ほどコピーしたエンドポイントとキーの値に置き換えます。
1. プレースホルダーを置き換えた "後"、コード エディター内で、**CTRL + S** コマンドまたは**右クリック > [保存]** を使用して変更を保存し、**CTRL + Q** コマンドまたは**右クリック > [終了]** を使用して、Cloud Shell コマンド ラインを開いたままコード エディターを閉じます。**

## 合成データを生成する

次に、合成データセットを生成し、それを使用して事前トレーニング済みモデルの品質を評価するスクリプトを実行します。

1. 次のコマンドを実行して、提供された**スクリプトを編集**します。

    ```powershell
   code generate_synth_data.py
    ```

1. スクリプトで、**# Define callback function** を見つけます。
1. このコメントの下に、次のコードを貼り付けます。

    ```
    async def callback(
        messages: List[Dict],
        stream: bool = False,
        session_state: Any = None,  # noqa: ANN401
        context: Optional[Dict[str, Any]] = None,
    ) -> dict:
        messages_list = messages["messages"]
        # Get the last message
        latest_message = messages_list[-1]
        query = latest_message["content"]
        context = text
        # Call your endpoint or AI application here
        current_dir = os.getcwd()
        prompty_path = os.path.join(current_dir, "application.prompty")
        _flow = load_flow(source=prompty_path)
        response = _flow(query=query, context=context, conversation_history=messages_list)
        # Format the response to follow the OpenAI chat protocol
        formatted_response = {
            "content": response,
            "role": "assistant",
            "context": context,
        }
        messages["messages"].append(formatted_response)
        return {
            "messages": messages["messages"],
            "stream": stream,
            "session_state": session_state,
            "context": context
        }
    ```

    ターゲットのコールバック関数を指定すると、シミュレートする任意のアプリケーション エンドポイントを取り込むことができます。 この場合、使用するアプリケーションは、Prompty ファイル `application.prompty` を使用する LLM です。 上記のコールバック関数は、次のタスクを実行して、シミュレーターによって生成された各メッセージを処理します。
    * 最新のユーザー メッセージを取得します。
    * application.prompty からプロンプト フローを読み込む。
    * プロンプト フローを使用して応答を生成します。
    * OpenAI チャット プロトコルに準拠するように応答を書式設定します。
    * アシスタントの応答をメッセージ一覧に追加します。

    >**注**:Prompty の使用方法の詳細については、[Prompty のドキュメント](https://www.prompty.ai/docs)を参照してください。

1. 次に、**# Run the simulator** を見つけます。
1. このコメントの下に、次のコードを貼り付けます。

    ```
    model_config = {
        "azure_endpoint": os.getenv("AZURE_OPENAI_ENDPOINT"),
        "api_key": os.getenv("AZURE_OPENAI_API_KEY"),
        "azure_deployment": os.getenv("AZURE_OPENAI_DEPLOYMENT"),
    }
    
    simulator = Simulator(model_config=model_config)
    
    outputs = asyncio.run(simulator(
        target=callback,
        text=text,
        num_queries=1,  # Minimal number of queries
    ))
    
    output_file = "simulation_output.jsonl"
    with open(output_file, "w") as file:
        for output in outputs:
            file.write(output.to_eval_qr_json_lines())
    ```

   上記のコードはシミュレーターを初期化し、それを実行して、Wikipedia から以前に抽出されたテキストに基づいて合成会話を生成します。

1. 次に、**# Evaluate the model** を見つけます。
1. このコメントの下に、次のコードを貼り付けます。

    ```
    groundedness_evaluator = GroundednessEvaluator(model_config=model_config)
    eval_output = evaluate(
        data=output_file,
        evaluators={
            "groundedness": groundedness_evaluator
        },
        output_path="groundedness_eval_output.json"
    )
    ```

    データセットが作成されたので、生成 AI アプリケーションの品質と有効性を評価できます。 上記のコードでは、品質メトリックとして根拠性を使用します。

1. 変更を保存。
1. コード エディターの下の Cloud Shell コマンド ライン ペインで、次のコマンドを入力して**スクリプトを実行**します。

    ```
   python generate_synth_data.py
    ```

    スクリプトが完了したら、`download simulation_output.jsonl` と `download groundedness_eval_output.json` を実行して出力ファイルをダウンロードし、その内容を確認できます。 根拠性メトリックが 3.0 に近くない場合は、`application.prompty` ファイル内の `temperature`、`top_p`、`presence_penalty`、`frequency_penalty` などの LLM パラメーターを変更し、スクリプトを再実行して評価用の新しいデータセットを生成できます。 また、`wiki_search_term` を変更して、別のコンテキストに基づいて合成データセットを取得することもできます。

## (省略可能) モデルを微調整する

時間に余裕がある場合は、生成されたデータセットを使用して、Azure AI Foundry でモデルを微調整できます。 微調整はクラウド インフラストラクチャ リソースに依存します。データ センターの容量と同時需要に応じて、プロビジョニングにはさまざまな時間がかかります。

1. 新しいブラウザー タブを開き、[Azure AI Foundry ポータル](https://ai.azure.com) (`https://ai.azure.com`) に移動し、Azure 資格情報を使用してサインインします。
1. AI Foundry のホーム ページで、この演習の冒頭で作成したプロジェクトを選択します。
1. 左側のメニューを使用して、**[ビルドとカスタマイズ]** セクションの **[微調整]** ページに移動します。
1. 新しい微調整モデルを追加するボタンを選択し、**GPT-4o** モデルを選択してから、**[次へ]** を選択します。
1. 次の構成を使用してモデルを**微調整**します。
    - **モデルのバージョン**: *Select the default version (既定のバージョンの選択)*
    - **カスタマイズの方法**: 監督下
    - **[モデル サフィックス]**: `ft-travel`
    - **AI リソースに接続済み**: *ハブの作成時に作成された接続を選択する。既定で選択する必要がある。*
    - **[トレーニング データ]**: ファイルをアップロードします

    <details>  
    <summary><b>トラブルシューティングのヒント</b>: アクセス許可エラー</summary>
    <p>アクセス許可エラーが返された場合は、次のトラブルシューティングを試してください。</p>
    <ul>
        <li>Azure portal で、AI サービス リソースを選択します。</li>
        <li>[リソース管理] の [ID] タブで、システム割り当てのマネージド ID であることを確認します。</li>
        <li>関連付けられたストレージ アカウントに移動します。 [IAM] ページで、<em>[ストレージ BLOB データ所有者]</em> というロールの割り当てを追加します。</li>
        <li><strong>[アクセスの割り当て先]</strong> で、<strong>[マネージド ID]</strong>、<strong>+[メンバーの選択]</strong> を選択し、<strong>[すべてのシステム割り当てマネージド ID]</strong> を選択して、Azure AI サービス リソースを選択します。</li>
        <li>[確認と割り当て] で新しい設定を保存し、前の手順を繰り返します。</li>
    </ul>
    </details>

    - **[ファイルのアップロード]**: 前の手順でダウンロードした JSONL ファイルを選択します。
    - **[検証データ]**: なし
    - **[タスク パラメーター]**: *既定の設定のままにします*
1. 微調整が開始されます。完了するまでに時間がかかる場合があります。

    > **注**: 微調整とデプロイにはかなりの時間 (30 分以上) がかかる可能性があるため、定期的に確認する必要があります。 ここまでの進行状況の詳細を確認するには、微調整モデル ジョブを選択し、その **[ログ]** タブを表示します。

## (省略可能) 微調整されたモデルをデプロイする

微調整が正常に完了したら、微調整したモデルをデプロイできます。

1. 微調整ジョブのリンクを選択して、詳細ページを開きます。 次に、**[メトリック]** タブを選択し、微調整されたメトリックを調べます。
1. 次の構成を使用して、微調整されたモデルをデプロイします。
    - **デプロイ名**: モデル デプロイの有効な名前**
    - **デプロイの種類**:Standard
    - **1 分あたりのトークン数のレート制限 (1,000 単位)**:5K *(または 5K 未満の場合はサブスクリプションで使用可能な最大値)
    - **コンテンツ フィルター**: 既定
1. テストできるようになるには、デプロイが完了するまで待ちます。これには時間がかかる場合があります。 成功するまで、**プロビジョニングの状態**を確認します (更新された状態を表示するには、ブラウザーを再読み込みする必要がある場合があります)。
1. デプロイの準備ができたら、微調整したモデルに移動し、**[プレイグラウンドで開く]** を選択します。

    微調整されたモデルをデプロイしたので、任意の基本モデルと同様に、チャット プレイグラウンドでそれをテストできます。

## まとめ

この演習では、ユーザーとチャット入力候補アプリの間の会話をシミュレートする合成データセットを作成しました。 このデータセットを使用すると、アプリの応答の品質を評価し、モデルを微調整して目的の結果を得ることができます。

## クリーンアップ

Azure AI サービスを確認し終わったら、不要な Azure コストが発生しないように、この演習で作成したリソースを削除する必要があります。

1. Azure portal が表示されているブラウザー タブに戻り (または、新しいブラウザー タブで [Azure portal](https://portal.azure.com?azure-portal=true) をもう一度開き)、この演習で使ったリソースがデプロイされているリソース グループの内容を表示します。
1. ツール バーの **[リソース グループの削除]** を選びます。
1. リソース グループ名を入力し、削除することを確認します。
