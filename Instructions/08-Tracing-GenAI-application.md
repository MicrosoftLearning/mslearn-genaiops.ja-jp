---
lab:
  title: トレースを使用して生成 AI アプリを分析およびデバッグする
  description: ユーザー入力からモデルの応答と後処理までのワークフローをトレースして、生成 AI アプリケーションをデバッグする方法について学習します。
---

# トレースを使用して生成 AI アプリを分析およびデバッグする

この演習は約 **30** 分かかります。

> **注**: この演習では、Azure AI Foundry に関するある程度の知識を前提としているため、よりアクティブな探索と実践的な学習を促すために、一部の手順は意図的にあまり詳細に説明されていません。

## はじめに

この演習では、お勧めのハイキング旅行を提示しアウトドア用品を提案する複数ステップ生成 AI アシスタントを実行します。 Azure AI 推論 SDK のトレース機能を使用して、アプリケーションがどのように実行されているかを分析し、モデルとそれを取り巻くロジックによってなされた重要な意思決定ポイントを特定します。

デプロイ済みモデルとやり取りして、実際のユーザー体験をシミュレートし、ユーザーによる入力からモデルの応答を経由してポスト プロセスに至るアプリケーションの各ステージをトレースし、Azure AI Foundry でトレース データを表示します。 これにより、トレースによってどのように監視が強化され、デバッグが簡略化され、生成 AI アプリケーションのパフォーマンス最適化がサポートされるかを理解することができます。

## 環境を設定する

この演習のタスクを完了するには、以下が必要です。

- Azure AI Foundry ハブ
- Azure AI Foundry プロジェクト
- デプロイされたモデル (GPT-4o など)
- 接続された Application Insights リソース

### Azure AI Foundry プロジェクトにモデルをデプロイする

Azure AI Foundry プロジェクトをすばやくセットアップするため、Azure AI Foundry ポータル UI を使用する簡単な手順を以下に示します。

1. Web ブラウザーで [Azure AI Foundry ポータル](https://ai.azure.com) (`https://ai.azure.com`) を開き、Azure 資格情報を使用してサインインします。
1. ホーム ページの **[モデルと機能を調査する]** セクションで、プロジェクトで使用する `gpt-4o` モデルを検索します。
1. 検索結果で **gpt-4o** モデルを選んで詳細を確認してから、モデルのページの上部にある **[このモデルを使用する]** を選択します。
1. プロジェクトの作成を求められたら、プロジェクトの有効な名前を入力し、**[詳細]** オプションを展開します。
1. **[カスタマイズ]** を選択し、プロジェクトに次の設定を指定します。
    - **Azure AI Foundry リソース**: *Azure AI Foundry リソースの有効な名前*
    - **[サブスクリプション]**:"*ご自身の Azure サブスクリプション*"
    - **リソース グループ**: *リソース グループを作成または選択します*
    - **リージョン**: ***AI サービスでサポートされている場所を選択します***\*

    > \* 一部の Azure AI リソースは、リージョンのモデル クォータによって制限されます。 演習の後半でクォータ制限を超えた場合は、別のリージョンに別のリソースを作成する必要が生じる可能性があります。

1. **[作成]** を選択し、選んだ gpt-4 モデル デプロイを含むプロジェクトが作成されるまで待ちます。
1. 左側のナビゲーション ウィンドウで **[概要]** を選択すると、プロジェクトのメイン ページが表示されます。
1. **[エンドポイントとキー]** の領域で、**[Azure AI Foundry]** ライブラリが選択されていることを確認し、**[Azure AI Foundry プロジェクト エンドポイント]** を表示します。
1. エンドポイントをメモ帳に**保存**します。 クライアント アプリケーションで、このエンドポイントを使用してプロジェクトに接続します。

### Application Insights の接続

Application Insights を Azure AI Foundry のプロジェクトに接続して、分析するデータの収集を開始します。

1. 左側のメニューを使用し、**[トレース]** ページを選択します。
1. アプリに接続する Application Insights リソースを**新規作成**します。
1. Application Insights リソース名を入力し、**[作成]** を選択します。

Application Insights がプロジェクトに接続され、分析のためにデータの収集が開始されます。

## Cloud Shell で生成 AI アプリを実行する

Azure Cloud Shell から Azure AI Foundry プロジェクトに接続し、生成 AI アプリケーションの一部としてプログラムでデプロイ済みモデルとやり取りします。

### デプロイ済みモデルとやり取りする

まず、デプロイ済みモデルとやり取りするための認証に必要な情報を取得します。 次に、Azure Cloud Shell にアクセスし、生成 AI アプリのコードを更新します。

1. 新しいブラウザー タブを開きます (既存のタブで Azure AI Foundry ポータルを開いたままにします)。
1. 新しいブラウザー タブで [Azure portal](https://portal.azure.com) (`https://portal.azure.com`) を参照し、メッセージに応じて Azure 資格情報を使用してサインインします。
1. ページ上部の検索バーの右側にある **[\>_]** ボタンを使用して、Azure portal に新しい Cloud Shell を作成し、サブスクリプションにストレージがない ***PowerShell*** 環境を選択します。
1. Cloud Shell ツール バーの **[設定]** メニューで、**[クラシック バージョンに移動]** を選択します。

    **<font color="red">続行する前に、クラシック バージョンの Cloud Shell に切り替えたことを確認します。</font>**

1. Cloud Shell ペインで、次のコマンドを入力して実行します。

    ```
    rm -r mslearn-genaiops -f
    git clone https://github.com/microsoftlearning/mslearn-genaiops mslearn-genaiops
    ```

    このコマンドは、この演習のコード ファイルを含んだ GitHub リポジトリを複製します。

1. リポジトリが複製されたら、アプリケーション コード ファイルを含んだフォルダーに移動します。  

    ```
   cd mslearn-genaiops/Files/08
    ```

1. Cloud Shell コマンド ライン ペインで、次のコマンドを入力して、必要なライブラリをインストールします。

    ```
   python -m venv labenv
   ./labenv/bin/Activate.ps1
   pip install python-dotenv openai azure-identity azure-ai-projects opentelemetry-instrumentation-openai-v2 azure-monitor-opentelemetry
    ```

1. 次のコマンドを入力して、提供されている構成ファイルを開きます。

    ```
   code .env
    ```

    このファイルをコード エディターで開きます。

1. コード ファイルで以下を行います。

    1. コード ファイルで、**[your_project_endpoint]** プレースホルダーをプロジェクトのエンドポイント (Azure AI Foundry ポータルでプロジェクトの **[概要]** ページからコピーしたもの) に置き換えます。
    1. **your_model_deployment** プレースホルダーを、GPT-4o モデル デプロイに割り当てた名前 (既定では `gpt-4o`) に置き換えます。

1. プレースホルダーを置き換えた "後"、コード エディター内で、**CTRL + S** コマンドまたは**右クリック > [保存]** を使用して**変更を保存**し、**CTRL + Q** コマンドまたは**右クリック > [終了]** を使用して、Cloud Shell コマンド ラインを開いたままコード エディターを閉じます。**

### 生成 AI アプリのコードを更新する

環境が設定され、.env ファイルが構成されたので、次は AI アシスタント スクリプトを実行用に準備します。 AI プロジェクトと接続し Application Insights を有効にしたあと、次の手順を実行する必要があります。

- デプロイ済みモデルとやり取りします。
- プロンプトを指定する関数を定義します。
- すべての関数を呼び出すメイン フローを定義します。

これら 3 つの部分を開始スクリプトに追加します。

1. 次のコマンドを実行して、指定された**スクリプトを開きます**。

    ```
   code start-prompt.py
    ```

    複数の主要な行が空白のままになっているか、空の # コメントでマークされていることがわかります。 次の正しい行をコピーして適切な場所に貼り付けることで、スクリプトを完了する必要があります。

1. スクリプトで、**# Function to call the model and handle tracing** (モデルを呼び出しトレースを処理する関数) というコメントを見つけます。
1. このコメントの下に、次のコードを貼り付けます。

    ```
   # Function to call the model and handle tracing
   def call_model(system_prompt, user_prompt, span_name):
       with tracer.start_as_current_span(span_name) as span:
           span.set_attribute("session.id", SESSION_ID)
           span.set_attribute("prompt.user", user_prompt)
           start_time = time.time()
    
           response = chat_client.chat.completions.create(
               model=model_deployment,
               messages=[
                   { 
                       "role": "system", 
                       "content": system_prompt 
                   },
                   { 
                       "role": "user", 
                       "content": user_prompt
                   }
               ]
           )
    
           duration = time.time() - start_time
           output = response.choices[0].message.content
           span.set_attribute("response.time", duration)
           span.set_attribute("response.tokens", len(output.split()))
           return output
    ```

1. スクリプトで、**# Function to recommend a hike based on user preferences** (ユーザー設定に基づいてお勧めのハイキングを提示する関数) というコメントを見つけます。
1. このコメントの下に、次のコードを貼り付けます。

    ```
   # Function to recommend a hike based on user preferences 
   def recommend_hike(preferences):
        with tracer.start_as_current_span("recommend_hike") as span:
            prompt = f"""
            Recommend a named hiking trail based on the following user preferences.
            Provide only the name of the trail and a one-sentence summary.
            Preferences: {preferences}
            """
            response = call_model(
                "You are an expert hiking trail recommender.",
                prompt,
                "recommend_model_call"
            )
            span.set_attribute("hike_recommendation", response.strip())
            return response.strip()
    ```

1. スクリプトで、**# ---- Main Flow ----** (---- メイン フロー ----) というコメントを見つけます。
1. このコメントの下に、次のコードを貼り付けます。

    ```
   if __name__ == "__main__":
       with tracer.start_as_current_span("trail_guide_session") as session_span:
           session_span.set_attribute("session.id", SESSION_ID)
           print("\n--- Trail Guide AI Assistant ---")
           preferences = input("Tell me what kind of hike you're looking for (location, difficulty, scenery):\n> ")

           hike = recommend_hike(preferences)
           print(f"\n✅ Recommended Hike: {hike}")

           # Run profile function


           # Run match product function


           print(f"\n🔍 Trace ID available in Application Insights for session: {SESSION_ID}")
    ```

1. スクリプトで行った**変更を保存**します。
1. Cloud Shell コマンド ライン ペインで、次のコマンドを入力してアプリを実行します。

    ```
   az login
    ```

    **<font color="red">Cloud Shell セッションが既に認証されている場合でも、Azure にサインインする必要があります。</font>**

    > **注**: ほとんどのシナリオでは、*az ログイン*を使用するだけで十分です。 ただし、複数のテナントにサブスクリプションがある場合は、*[--tenant]* パラメーターを使用してテナントを指定する必要があります。 詳細については、「[Azure CLI を使用して対話形式で Azure にサインインする](https://learn.microsoft.com/cli/azure/authenticate-azure-cli-interactively)」を参照してください。
    
1. メッセージが表示されたら、指示に従って新しいタブでサインイン ページを開き、指定された認証コードと Azure 資格情報を入力します。 次に、コマンド ラインでサインイン プロセスを完了し、プロンプトが表示されたら、Azure AI Foundry ハブを含むサブスクリプションを選択します。
1. サインインしたら、次のコマンドを入力してアプリケーションを実行します。

    ```
   python start-prompt.py
    ```

1. 探しているハイキングの種類について説明します。次に例を示します。

    ```
   A one-day hike in the mountains
    ```

    応答がモデルによって生成され、Application Insights でキャプチャされます。 **Azure AI Foundry ポータル**でトレースを視覚化できます。

> **注**: 監視データが Azure Monitor に表示されるまでに数分かかる場合があります。

## Azure AI Foundry ポータルでトレース データを表示する

スクリプトを実行した後、AI アプリケーションの実行のトレースをキャプチャしました。 次は、Azure AI Foundry で Application Insights を使用してトレースを調べます。

> **注:** 後で、コードをもう一度実行し、Azure AI Foundry ポータルでトレースをもう一度表示します。 まず、視覚化するためにトレースを探す場所を調べてみましょう。

### Azure AI Foundry ポータルに移動します。

1. **Cloud Shell を開いたままにしておきます。** ここに戻ってコードを更新し、もう一度実行します。
1. **Azure AI Foundry ポータル**が開いている、ブラウザーのタブに移動します。
1. 左側のメニューを使用し、**[トレース]** を選択します。
1. *データが表示されない場合*は、ビューを**最新の情報に更新**します。
1. トレース **train_guide_session** を選択して、詳細を表示する新しいウィンドウを開きます。

### トレースを確認します。

このビューには、Trail Guide AI アシスタントの 1 つの完全なセッションのトレースが表示されます。

- **トップレベル スパン**: trail_guide_session。これは親スパンです。 アシスタントの開始から終了までの実行全体を表します。

- **入れ子になった子スパン**: インデントされた各行は、入れ子になった操作を表します。 次の情報が表示されます。

    - **recommend_hike**: ハイキングを決定するロジックをキャプチャします。
    - **recommend_model_call**: recommend_hike 内で call_model() によって作成されたスパンです。
    - **chat gpt-4o**: Azure AI 推論 SDK によって自動的にインストルメント化されて、LLM との実際のやり取りを表示します。

1. 任意のスパンをクリックすると、次の情報が表示されます。

    1. 実行継続時間。
    1. ユーザー プロンプト、使用されたトークン、応答時間などの属性。
    1. エラーまたは、**span.set_attribute(...)** でアタッチされたカスタム データ。

## コードにさらに関数を追加する

1. ブラウザーで、**Azure portal** が開いているタブに移動します。
1. 次のコマンドを実行して、**スクリプトをもう一度開きます**。

    ```
   code start-prompt.py
    ```

1. スクリプトで、**# Function to generate a trip profile for the recommended hike** (お勧めのハイキングの旅行プロファイルを生成する関数) というコメントを見つけます。
1. このコメントの下に、次のコードを貼り付けます。

    ```
   def generate_trip_profile(hike_name):
       with tracer.start_as_current_span("trip_profile_generation") as span:
           prompt = f"""
           Hike: {hike_name}
           Respond ONLY with a valid JSON object and nothing else.
           Do not include any intro text, commentary, or markdown formatting.
           Format: {{ "trailType": ..., "typicalWeather": ..., "recommendedGear": [ ... ] }}
           """
           response = call_model(
               "You are an AI assistant that returns structured hiking trip data in JSON format.",
               prompt,
               "trip_profile_model_call"
           )
           print("🔍 Raw model response:", response)
           try:
               profile = json.loads(response)
               span.set_attribute("profile.success", True)
               return profile
           except json.JSONDecodeError as e:
               print("❌ JSON decode error:", e)
               span.set_attribute("profile.success", False)
               return {}
    ```

1. スクリプトで、**# Function to match recommended gear with products in the catalog** (お勧めの用具をカタログ内の商品とマッチングする関数) というコメントを見つけます。
1. このコメントの下に、次のコードを貼り付けます。

    ```
   def match_products(recommended_gear):
       with tracer.start_as_current_span("product_matching") as span:
           matched = []
           for gear_item in recommended_gear:
               for product in mock_product_catalog:
                   if any(word in product.lower() for word in gear_item.lower().split()):
                       matched.append(product)
                       break
           span.set_attribute("matched.count", len(matched))
           return matched
    ```

1. スクリプトで、**# Run profile function** (プロファイル関数を実行する) というコメントを見つけます。
1. このコメントの下に、コメントに**揃えて**、次のコードを貼り付けます。

    ```
           profile = generate_trip_profile(hike)
           if not profile:
               print("Failed to generate trip profile. Please check Application Insights for trace.")
               exit(1)

           print(f"\n📋 Trip Profile for {hike}:")
           print(json.dumps(profile, indent=2))
    ```

1. スクリプトで、**# Run match product function** (商品マッチング関数を実行する) というコメントを見つけます。
1. このコメントの下に、コメントに**揃えて**、次のコードを貼り付けます。

    ```
           matched = match_products(profile.get("recommendedGear", []))
           print("\n🛒 Recommended Products from Lakeshore Retail:")
           print("\n".join(matched))
    ```

1. スクリプトで行った**変更を保存**します。
1. コード エディターの下の Cloud Shell コマンド ライン ペインで、次のコマンドを入力して**スクリプトを実行**します。

    ```
   python start-prompt.py
    ```

1. 探しているハイキングの種類について説明します。次に例を示します。

    ```
   I want to go for a multi-day adventure along the beach
    ```

<br>
<details>
<summary><b>ソリューション スクリプト</b>: コードが動作していない場合。</summary><br>
<p>generate_trip_profile 関数の LLM トレースを調べると、出力をコード ブロックとして書式設定するためのバッククオートと json という単語がアシスタントの応答に含まれていることに気付きます。

これは表示には役に立ちますが、コードの問題の原因になっています。出力が有効な JSON ではなくなったからです。 この結果、さらに処理を進めると解析エラーが発生します。

このエラーは、特定の出力形式に従うように LLM に指示する方法に起因して発生する可能性があります。 指示をユーザー プロンプトに含める方が、システム プロンプトに含めるよりも効果的のように思われます。</p>
</details>


> **注**: 監視データが Azure Monitor に表示されるまでに数分かかる場合があります。

### Azure AI Foundry ポータルで新しいトレースを表示する

1. Azure AI Foundry ポータルに戻ります。
1. 同じ **trail_guide_session** という名前の新しいトレースが表示されます。 必要に応じて、ビューを最新の情報に更新します。
1. 新しいトレースを選択して、より詳細なビューを開きます。
1. 入れ子になった新しい子スパン **trip_profile_generation** と **product_matching** を確認します。
1. **product_matching** を選択し、表示されるメタデータを確認します。

    product_matching 関数には、**span.set_attribute("matched.count", len(matched))** を含めました。 キーと値のペア **matched.count** と変数の長さが一致する属性を設定することで、この情報を **product_matching** トレースに追加しました。 このキーと値のペアは、メタデータの **attributes** 下にあります。

## (オプション) エラーをトレースする

時間に余裕があれば、エラー発生時のトレースの使用方法を確認してもかまいません。 エラーをスローする可能性が高いスクリプトが用意されています。 それを実行し、トレースを確認します。

これは、チャレンジ課題となるように設計された演習です。つまり、手順はわざとあまり詳細に説明されていません。

1. Cloud Shell で **error-prompt.py** スクリプトを開きます。 このスクリプトは、**start-prompt.py** スクリプトと同じディレクトリにあります。 スクリプトの内容を確認します。
1. **error-prompt.py** スクリプトを実行します。 入力を求めるメッセージが表示されたら、コマンド ラインに回答を入力します。
1. *うまくいけば*、出力メッセージに **Failed to generate trip profile. Please check Application Insights for trace.** (旅行プロファイルを生成できませんでした。Application Insights でトレースを確認してください) が含まれています。
1. **trip_profile_generation** のトレースに移動し、エラーが発生した理由を調べます。

<br>
<details>
<summary>エラーが発生した可能性がある理由についての<b>回答を取得します</b>。</summary><br>
<p>generate_trip_profile 関数の LLM トレースを調べると、出力をコード ブロックとして書式設定するためのバッククオートと json という単語がアシスタントの応答に含まれていることに気付きます。

これは表示には役に立ちますが、コードの問題の原因になっています。出力が有効な JSON ではなくなったからです。 この結果、さらに処理を進めると解析エラーが発生します。

このエラーは、特定の出力形式に従うように LLM に指示する方法に起因して発生する可能性があります。 指示をユーザー プロンプトに含める方が、システム プロンプトに含めるよりも効果的のように思われます。</p>
</details>

## 他のラボの参照先

その他のラボと演習については、[Azure AI Foundry ラーニング ポータル](https://ai.azure.com)で調べてください。また、他の利用可能なアクティビティについては、このコースの**ラボ セクション**をご覧ください。
