---
lab:
  title: RAG システムを調整する
  description: アプリに検索拡張生成 (RAG) システムを実装して、生成された応答の精度と関連性を高める方法について説明します。
---

## RAG システムを調整する

検索拡張生成 (RAG) システムは、大規模言語モデルの能力と効率的な取得メカニズムを組み合わせて、生成された応答の正確性と関連性を高めます。 オーケストレーション用の LangChain と AI 機能用の Azure AI Foundry を利用することで、データセットから関連情報を取得し、一貫性のある応答を生成する信頼性の高いパイプラインを作成できます。 この演習では、環境の設定、データの前処理、埋め込みの作成、インデックスの作成の手順を実行して、最終的に RAG システムを効果的に実装できるようにします。

この演習には約 **30** 分かかります。

## シナリオ

ロンドンのホテルに関する推奨事項を提供するアプリを構築するとします。 このアプリには、ホテルを推奨するだけでなく、ユーザーがそのホテルに関してしそうな質問に答えることができるエージェントを求めています。

生成型回答を提供するために GPT-4 モデルを選択しました。 ここでは、他のユーザー レビューに基づいてモデルにグラウンディング データを提供し、チャットの動作をパーソナル化された推奨事項に導く RAG システムを作成したいと考えています。

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
    cd ~/mslearn-genaiops/Files/04/
     ```

1. 次のコマンドを入力して仮想環境をアクティブ化し、必要なライブラリをインストールします。

    ```powershell
   python -m venv labenv
   ./labenv/bin/Activate.ps1
   pip install python-dotenv langchain-text-splitters langchain-community langchain-openai
    ```

1. 次のコマンドを入力して、提供されている構成ファイルを開きます。

    ```powershell
   code .env
    ```

    このファイルをコード エディターで開きます。

1. コード ファイルで、**your_azure_openai_service_endpoint** と **your_azure_openai_service_api_key** プレースホルダーを、先ほどコピーしたエンドポイントとキーの値に置き換えます。
1. プレースホルダーを置き換えたら、コード エディター内で、**Ctrl + S** コマンドを使用するか、**右クリックして保存**で変更を保存してから、**Ctrl + Q** コマンドを使用するか、**右クリックして終了**で、Cloud Shell コマンド ラインを開いたままコード エディターを閉じます。**

## RAG を実装する

次に、データの取り込みと前処理、埋め込みの作成、ベクトル ストアとインデックスの構築を行うスクリプトを実行し、最終的に RAG システムを効果的に実装できるようにします。

1. 次のコマンドを実行して、提供された**スクリプトを編集**します。

    ```powershell
   code RAG.py
    ```

1. スクリプトで、**# Initialize the components that will be used from LangChain's suite of integrations (# LangChain の統合スイートから使用されるコンポーネントを初期化する)** を探します。 このコメントの下に、次のコードを貼り付けます。

    ```python
   # Initialize the components that will be used from LangChain's suite of integrations
   llm = AzureChatOpenAI(azure_deployment=llm_name)
   embeddings = AzureOpenAIEmbeddings(azure_deployment=embeddings_name)
   vector_store = InMemoryVectorStore(embeddings)
    ```

1. スクリプトを確認し、ホテルのレビューが典拠データとして含まれる.csv ファイルが使用されていることに注目します。 コマンド ライン ペインでコマンド `download app_hotel_reviews.csv` を実行してファイルを開くと、このファイルの内容を確認できます。
1. 次に、**# Split the documents into chunks for embedding and vector storage (# 埋め込みとベクトル ストレージのチャンクにドキュメントを分割する)** を探します。 このコメントの下に、次のコードを貼り付けます。

    ```python
   # Split the documents into chunks for embedding and vector storage
   text_splitter = RecursiveCharacterTextSplitter(
       chunk_size=200,
       chunk_overlap=20,
       add_start_index=True,
   )
   all_splits = text_splitter.split_documents(docs)
    
   print(f"Split documents into {len(all_splits)} sub-documents.")
    ```

    上記のコードにより、一連の大きなドキュメントが小さなチャンクに分割されます。 多くの埋め込みモデル (セマンティック検索やベクトル データベースに使用されるモデルなど) にはトークンの制限があり、短いテキストの方がパフォーマンスが向上するため、これは重要です。

1. 次に、**# Embed the contents of each text chunk and insert these embeddings into a vector store (# 各テキスト チャンクの内容を埋め込み、その埋め込みをベクトル ストアに挿入する)** を探します。 このコメントの下に、次のコードを貼り付けます。

    ```python
   # Embed the contents of each text chunk and insert these embeddings into a vector store
   document_ids = vector_store.add_documents(documents=all_splits)
    ```

1. 次に、**# Retrieve relevant documents from the vector store based on user input (# ユーザーによる入力に基づいてベクトル ストアから関連するドキュメントを取得する)** を探します。 このコメントの下に、次のコードを貼り付けて、インデントが適切かどうかを観察します。

    ```python
   # Retrieve relevant documents from the vector store based on user input
   retrieved_docs = vector_store.similarity_search(question, k=10)
   docs_content = "\n\n".join(doc.page_content for doc in retrieved_docs)
    ```

    上記のコードにより、ベクトル ストアで、ユーザーが入力した質問に最も近いドキュメントが検索されます。 質問は、ドキュメントに使用されたのと同じ埋め込みモデルを使用してベクトルに変換されます。 その後、システムはこのベクトルを格納されているすべてのベクトルと比較し、最も近いベクトルを取得します。

1. 変更を保存。
1. コマンド ラインで次のコマンドを入力して、**スクリプトを実行**します。

    ```powershell
   python RAG.py
    ```

1. アプリケーションが実行されたら、`Where can I stay in London?` などの質問を開始し、より具体的な問い合わせをフォローアップできます。

## まとめ

この演習では、主要コンポーネントを含む一般的な RAG システムを構築しました。 独自のドキュメントを使用してモデルの応答をお知らせして、LLM が応答を作成するときに使用される典拠データを提供します。 エンタープライズ ソリューションの場合は、生成 AI をエンタープライズ コンテンツに制限できることを意味します。

## クリーンアップ

Azure AI サービスを確認し終わったら、不要な Azure コストが発生しないように、この演習で作成したリソースを削除する必要があります。

1. Azure portal が表示されているブラウザー タブに戻り (または、新しいブラウザー タブで [Azure portal](https://portal.azure.com?azure-portal=true) をもう一度開き)、この演習で使ったリソースがデプロイされているリソース グループの内容を表示します。
1. ツール バーの **[リソース グループの削除]** を選びます。
1. リソース グループ名を入力し、削除することを確認します。
