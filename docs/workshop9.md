## チャットAPI

まず、チャットAPIのコードを作成します。このAPIは[ChatBootAI OpenAPI仕様](https://editor.swagger.io/?url=https://raw.githubusercontent.com/ChatBootAI/chatbootai-openapi/main/openapi/openapi-chatbootai.yml)を実装し、ウェブサイトがメッセージの回答を取得するために使用されます。

### Fastifyの紹介

チャットAPIを作成するために[Fastify](https://www.fastify.io/)を使用します。Fastifyは、最小限のオーバーヘッドと強力なプラグインアーキテクチャで最高の開発者体験を提供することに重点を置いたWebフレームワークです。

[Express](https://expressjs.com)に非常に似ていますが、はるかに高速で軽量であり、マイクロサービスに適した選択肢です。また、TypeScriptのサポートが一流であり、これをベーステンプレートで使用します。

### チャットプラグインの設定

まず、チャットサービスを実装するためのFastifyプラグインを作成します。プラグインはFastifyで機能をカプセル化する方法であり、コードを整理するのに適しています。

ファイル `src/backend/src/plugins/chat.ts` を開きます。下部に次のコードが表示されます:

```ts
export default fp(
  async (fastify, options) => {
    const config = fastify.config;

    // TODO: initialize clients here

    const chatService = new ChatService(/* config, model, vectorStore */);

    fastify.decorate('chat', chatService);
  },
  {
    name: 'chat',
    dependencies: ['config'],
  },
);
```

ここにチャットサービスを実装するための出発点があります。ここでの各部分を見てみましょう:

1. まず、`const config = fastify.config;` でサービスに必要な設定を取得します。これは `src/backend/src/plugins/config.ts` ファイルの環境変数から初期化されます。
2. 次に、Azureサービスを呼び出すために必要なクライアントを作成します。これについては次のセクションで説明します。
3. その後、APIが回答を生成するために使用する `ChatService` インスタンスを作成します。作成したクライアントをコンストラクタのパラメータとして渡します。
4. 最後に、Fastifyインスタンスを `ChatService` インスタンスで装飾し、ルートから `fastify.chat` を使用してアクセスできるようにします。

### SDKクライアントの初期化

`// TODO: initialize clients here` を実際のクライアント設定コードに置き換えます。

#### Azure資格情報の管理

クライアントを作成する前に、Azureサービスにアクセスするための資格情報を取得する必要があります。[Azure Identity SDK](https://learn.microsoft.com/javascript/api/overview/azure/identity-readme?view=azure-node-latest) を使用してこれを行います。

ファイルの先頭に次のインポートを追加します:

```ts
import { DefaultAzureCredential, getBearerTokenProvider } from '@azure/identity';
```

次に、`const config = fastify.config;` 行の下に次のコードを追加します:

```ts
// 現在のユーザーIDを使用して認証します。
// シークレットは不要で、ローカルでは `az login` または `azd auth login` を使用し、Azureにデプロイされた場合はマネージドIDを使用します。
const credentials = new DefaultAzureCredential();

// OpenAIトークンプロバイダーを設定
const getToken = getBearerTokenProvider(credentials, 'https://cognitiveservices.azure.com/.default');
const azureADTokenProvider = async () => {
  try {
    return await getToken();
  } catch {
    // ローカルコンテナ環境ではAzure IDがサポートされていないため、ダミーキーを使用します（OpenAIプロキシを使用する場合のみ機能します）。
    fastify.log.warn('Azure OpenAIトークンの取得に失敗しました。ダミーキーを使用します');
    return '__dummy';
  }
};
```

これにより、現在のユーザーIDを使用してAzure OpenAIに認証します。シークレットを提供する必要はなく、ローカルでは `az login`（または `azd auth login`）を使用し、Azureにデプロイされた場合は[マネージドID](https://learn.microsoft.com/entra/identity/managed-identities-azure-resources/overview)を使用します。

<div class="info" data-title="note">

> ローカルコンテナ内で実行される場合、Azure Identity SDKはAzure Developer CLIから現在のユーザーIDを取得できません。簡単のため、この場合はダミーキーを使用しますが、これはこのワークショップに参加する場合に提供されるOpenAIプロキシを使用する場合にのみ機能します。
> ローカルで適切に認証する必要がある場合は、アプリをコンテナ外で `npm run dev` で実行するか、[サービスプリンシパル](https://learn.microsoft.com/entra/identity-platform/howto-create-service-principal-portal)を作成し、必要な権限を割り当てて環境変数をコンテナに渡す必要があります。

</div>

#### LangChain.jsクライアント

次にLangChain.jsクライアントを作成します。まず、ファイルの先頭に次のインポートを追加します:

```ts
import { AzureChatOpenAI, AzureOpenAIEmbeddings } from '@langchain/openai';
import { AzureAISearchVectorStore } from '@langchain/community/vectorstores/azure_aisearch';
```

次に、資格情報の取得の下に次のコードを追加します:

```ts
// LangChain.jsクライアントを設定
fastify.log.info(`OpenAIを使用しています: ${config.azureOpenAiApiEndpoint}`);

const model = new AzureChatOpenAI({
  azureADTokenProvider,
  // OpenAIエンドポイントを設定可能にするために必要
  azureOpenAIBasePath: `${config.azureOpenAiApiEndpoint}/openai/deployments`,
  // ランダム性を制御します。0 = 決定論的、1 = 最大ランダム性
  temperature: 0.7,
  // 生成するトークンの最大数
  maxTokens: 1024,
  // 生成する完了の数
  n: 1,
});
const embeddings = new AzureOpenAIEmbeddings({
  azureADTokenProvider,
  // OpenAIエンドポイントを設定可能にするために必要
  azureOpenAIBasePath: `${config.azureOpenAiApiEndpoint}/openai/deployments`,
});
const vectorStore = new AzureAISearchVectorStore(embeddings, { credentials });
```

ここでは、Azure OpenAIチャットモデルとAzure OpenAI埋め込みモデルのクライアントを作成し、先ほど作成した `azureADTokenProvider` メソッドを使用します。チャットモデルの動作を制御するためにいくつかのオプションを渡します:
- `temperature` はモデルのランダム性を制御します。値が0の場合、モデルは決定論的になり、値が1の場合、最もランダムな回答を生成します。
- `maxTokens` はモデルが生成するトークンの最大数です。値が低すぎると、モデルは長い回答を生成できません。値が高すぎると、モデルは長すぎる回答を生成する可能性があります。
- `n` はモデルが生成する回答の数です。今回は1つの回答のみが必要なので、1に設定します。

ベクトルデータベースには、AI Searchサービスと対話するために使用される `AzureAISearchVectorStore` インスタンスを作成します。

### ChatServiceの作成

すべてのクライアントを作成したので、`ChatService` インスタンスを適切に初期化する時が来ました。`ChatService` コンストラクタ呼び出しのパラメータを次のようにコメント解除します:

```ts
const chatService = new ChatService(config, model, vectorStore);
```

現在の設定、Azure OpenAIチャットモデルクライアント、およびAzure AI Searchベクトルストアクライアントを `ChatService` インスタンスに渡します。

#### ドキュメントの取得

RAGパターンの実装を開始する時が来ました！最初のステップは、ベクトルデータベースからドキュメントを取得することです。`ChatService` クラスには、現在 `// TODO: implement Retrieval Augmented Generation (RAG) here` とコメントされている `run` メソッドがあります。ここでRAGパターンを実装します。

ドキュメントを取得する前に、質問を取得する必要があります:

```ts
// 最後のメッセージの内容（質問）を取得
const query = messages[messages.length - 1].content;
```

次に、ベクトルデータベースから最も一致する3つのドキュメントを取得します。LangChain.jsはクエリの埋め込みを自動的に計算してから検索を実行します。

```ts
// ベクトル類似性検索を実行
// クエリの埋め込みは自動的に計算されます
const documents = await this.vectorStore.similaritySearch(query, 3);
```

検索結果を処理してドキュメントの内容を抽出します:

```ts
const results: string[] = [];
for (const document of documents) {
  const source = document.metadata.source;
  const content = document.pageContent.replaceAll(/[\n\r]+/g, ' ');
  results.push(`${source}: ${content}`);
}

const content = results.join('\n');
```

結果の各ドキュメントについて、ページ情報とドキュメントの内容を抽出し、それから文字列を作成します。
内容については、正規表現を使用してすべての改行をスペースに置き換え、後でGPTモデルにフィードするのが簡単になるようにします。

最後に、すべての結果を1つの文字列に結合し、各ドキュメントを改行で区切ります。この内容を使用して拡張プロンプトを生成します。

#### システムプロンプトの作成

ドキュメントの内容を取得したので、GPTモデルに送信される基本プロンプトを作成します。インポートの下に次のコードを追加します:

```ts
const SYSTEM_MESSAGE_PROMPT = `Assistant helps the Consto Real Estate company customers with support questions regarding terms of service, privacy policy, and questions about support requests. Be brief in your answers.
Answer ONLY with the facts listed in the list of sources below. If there isn't enough information below, say you don't know. Do not generate answers that don't use the sources below. If asking a clarifying question to the user would help, ask the question.
For tabular information return it as an html table. Do not return markdown format. If the question is not in English, answer in the language used in the question.
Each source has a name followed by colon and the actual information, always include the source name for each fact you use in the response. Use square brackets to reference the source, for example: [info1.txt]. Don't combine sources, list each source separately, for example: [info1.txt][info2.pdf].
`;
```

```ts
const SYSTEM_MESSAGE_PROMPT = `アシスタントは、Consto Real Estate社の顧客に対して、利用規約、プライバシーポリシー、およびサポートリクエストに関する質問のサポートを行います。回答は簡潔にしてください。
以下のソースリストに記載されている事実のみを使用して回答してください。以下に十分な情報がない場合は、知らないと言ってください。ソースを使用しない回答を生成しないでください。ユーザーに明確化の質問をすることが役立つ場合は、質問してください。
表形式の情報はHTMLテーブルとして返してください。Markdown形式では返さないでください。質問が英語でない場合は、質問に使用された言語で回答してください。
各ソースには名前があり、その後に実際の情報が続きます。回答に使用する各事実について、必ずソース名を含めてください。ソースを参照するには角括弧を使用してください。例えば：[info1.txt]。ソースを組み合わせず、各ソースを個別にリストしてください。例えば：[info1.txt][info2.pdf]。
`;
```

定数にすることで、後でコードに潜らずにプロンプトを調整しやすくします。

プロンプトを分解して、何が起こっているのかを理解しましょう。プロンプトを作成する際には、最良の結果を得るためにいくつかの点に注意する必要があります:

- プロンプトのドメインを明確にします。今回は、このフレーズでコンテキストを設定します: `Assistant helps the Consto Real Estate company customers with support questions regarding terms of service, privacy policy, and questions about support requests.`。これはデフォルトで提供されるドキュメントセットに関連しているので、独自のドキュメントを使用する場合は変更してください。

- 回答の長さを指定します。今回は回答を短く保ちたいので、このフレーズを追加します: `Be brief in your answers.`。

- RAGの文脈では、提供するドキュメントの内容のみを使用するように指示します: `Answer ONLY with the facts listed in the list of sources below.`。これはモデルを*グラウンディング*することと呼ばれます。

- モデルが事実を発明しないようにするために、ドキュメントに情報がない場合は知らないと答えるように指示します: `If there isn't enough information below, say you don't know. Do not generate answers that don't use the sources below.`。これは*エスケープハッチ*を追加することと呼ばれます。

- 必要に応じてモデルが明確化の質問をすることを許可します: `If asking a clarifying question to the user would help, ask the question.`。

- 回答の形式と言語を指定します: `Do not return markdown format. If the question is not in English, answer in the language used in the question.`

- モデルがソース形式を理解し、回答で引用する方法を指示します: `Each source has a name followed by colon and the actual information, always include the source name for each fact you use in the response. Use square brackets to reference the source, for example: [info1.txt]. Don't combine sources, list each source separately, for example: [info1.txt][info2.pdf].`

- 可能な場合は例を使用します。ソース形式を説明するために例を使用します。

#### 拡張プロンプトの作成

前のプロンプトでは、ソース内容を追加しなかったことに注意してください。これは、モデルが長いシステムメッセージをうまく処理できないためです。代わりに、ソースを最新のユーザーメッセージに注入します。

モデルが期待する入力は、最新のメッセージがユーザーメッセージであるメッセージの配列です。各メッセージには `system`（コンテキストを設定）、`user`（ユーザーの質問）、または `assistant`（AI生成の回答）という役割があります。

このメッセージの配列を作成するために、`src/backend/src/lib/message-builder.ts` ファイルに作成した `MessageBuilder` クラスを使用します。次のコードでRAGパターンの実装を続けます:

```ts
// システムメッセージでコンテキストを設定
const systemMessage = SYSTEM_MESSAGE_PROMPT;

// 最新のユーザーメッセージ（質問）を取得し、ソースを注入
const userMessage = `${messages[messages.length - 1].content}\n\nSources:\n${content}`;

// メッセージプロンプトを作成
const messageBuilder = new MessageBuilder(systemMessage, this.config.azureOpenAiApiModelName);
messageBuilder.appendMessage('user', userMessage);
```

会話の前のメッセージもモデルに役立つ場合があるため、プロンプトに追加します。ただし、GPTモデルには処理できるトークンの数に制限があるため、設定したトークン制限に達するまでメッセージを追加します。

```ts
// トークン制限を超えない限り、前のメッセージをプロンプトに追加
for (const historyMessage of messages.slice(0, -1).reverse()) {
  if (messageBuilder.tokens > this.tokenLimit) {
    messageBuilder.popMessage();
    break;
  };
  messageBuilder.appendMessage(historyMessage.role, historyMessage.content);
}
```

最後の仕上げとして、モデルが何をしているのかを理解するのに役立つデバッグ情報を作成することが有用です。

```ts
// デバッグ目的の処理詳細
const conversation = messageBuilder.messages.map((m) => `${m.role}: ${m.content}`).join('\n\n');
const thoughts = `検索クエリ:\n${query}\n\n会話:\n${conversation}`.replaceAll('\n', '<br>');
```

ここでは、検索クエリとモデルに送信されたメッセージを含む `thoughts` 文字列を作成し、回答と一緒に返します。

#### 応答の生成

モデルからの応答を生成する準備が整いました。前のコードの下に次のコードを追加します:

```ts
const completion = await this.model.invoke(messageBuilder.getMessages());
```

`invoke` メソッドを呼び出して応答を生成し、先ほど作成したメッセージを入力として渡します。

最後のステップは、Chat仕様形式で結果を返すことです:

```ts
// Chat仕様形式で応答を返す
return {
  message: {
    content: completion.content as string,
    role: 'assistant',
  },
  context: {
    data_points: results,
    thoughts: thoughts,
  },
};
```

完了の結果は `completion.content` プロパティにあります。また、検索ドキュメントの結果を含む `data_points` と `thoughts` プロパティを `context` オブジェクトに追加し、ウェブサイトがデバッグ情報を表示できるようにします。

ふぅ、たくさんのコードでしたね！しかし、RAGパターンの実装は完了しました。

### APIルートの作成

`ChatService` インスタンスができたので、それを呼び出すAPIルートを作成する必要があります。ファイル `src/backend/src/routes/root.ts` を開きます。`// TODO: create /chat endpoint` というコメントが次に何をすべきかのヒントを与えてくれます。

では、`/chat` エンドポイントを作成しましょう:

```ts
fastify.post('/chat', async function (request, reply) {
  const { messages } = request.body as any;
  try {
    return await fastify.chat.run(messages);
  } catch (_error: unknown) {
    const error = _error as Error;
    fastify.log.error(error);
    return reply.internalServerError(error.message);
  }
});
```

`fastify.post('/chat', ...)` を使用して、`/chat` ルートにPOSTエンドポイントを作成します。
リクエストボディから `messages` プロパティを取得し、先ほど作成した `ChatService` インスタンスの `run` メソッドを呼び出します。
また、発生する可能性のあるエラーをキャッチし、それらをログに記録し、クライアントに内部サーバーエラー（HTTPステータス500）を返します。

<div class="info" data-title="note">

> ここでは、リクエストボディの検証を省略して簡単にするために `any` にキャストする必要があります（ブー！）。実際のアプリケーションでは、リクエストボディが期待される形式と一致することを確認するために常に検証する必要があります。Fastifyは、[JSONスキーマを提供してボディを検証](https://fastify.dev/docs/latest/Reference/Validation-and-Serialization/#validation)することでこれを行うことができます。これにより、`as any` キャストを削除し、リクエストボディが無効な場合により良いエラーメッセージを取得できます。

</div>

APIのテストを行う準備が整いました！

### APIのテスト

ターミナルを開き、次のコマンドを実行してAPIを起動します:

```bash
cd src/backend
npm run dev
```

これにより、開発モードでAPIが起動します。開発モードでは、コードに変更を加えると自動的に再起動されます。

このAPIをテストするには、[REST Client](https://marketplace.visualstudio.com/items?itemName=humao.rest-client)拡張機能を使用するか、cURLリクエストを使用できます。

#### オプション1: REST Client拡張機能を使用する

ファイル `src/backend/test.http` を開きます。「Chat with the bot」コメントの下にある **Send Request** ボタンをクリックしてAPIをテストします。

質問を変更してモデルの動作を確認することができます。

テストが完了したら、各ターミナルで `Ctrl+C` を押してサーバーを停止します。

すべてが期待通りに動作することを確認したら、リポジトリに変更をコミットして進捗を追跡することを忘れないでください。

#### オプション2: cURLリクエストを使用する

VS Codeで新しいターミナルを開き、次のコマンドを実行します:

```bash
curl -X POST "http://localhost:3000/chat" \
  -H "Content-Type: application/json" \
  -d '{
    "messages": [{
      "content": "How to search and book rentals?",
      "role": "user"
    }]
  }'
```

質問を変更してモデルの動作を確認することができます。

テストが完了したら、各ターミナルで `Ctrl+C` を押してサーバーを停止します。

すべてが期待通りに動作することを確認したら、リポジトリに変更をコミットして進捗を追跡することを忘れないでください。


---