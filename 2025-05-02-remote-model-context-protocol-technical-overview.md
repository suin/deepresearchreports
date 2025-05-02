# 技術的詳細: リモートMCP (Model Context Protocol)

## 1. リモートMCPの概要

リモートMCP（Model Context Protocol）は、AI主導のツールとアプリケーションがリモートデータソースと安全に接続できるようにする標準化されたプロトコルです。このプロトコルはAnthropicが開発し、現在は多くの業界のパートナーによってサポートされています。

リモートMCPは、従来のローカルMCPサーバーの進化版であり、ウェブアクセス可能なインターフェースを通じてAIモデルとリモートサービスの間のシームレスな通信を可能にします。本質的には、「AIアプリケーションのためのUSB-Cポート」として機能し、標準化されたインターフェイスを通じてAIが外部ツール、データソース、サービスに接続できるようにします[MCP Introduction](https://modelcontextprotocol.io/introduction)。

## 2. アーキテクチャと核心コンポーネント

### 2.1 アーキテクチャの概要

リモートMCPサーバーのアーキテクチャは、以下の主要なコンポーネントで構成されています：

- **MCP ホスト**: Claude Desktopなどのプログラムで、MCPを通じてデータにアクセスしたいもの
- **MCP クライアント**: サーバーとの1:1接続を維持するプロトコルクライアント
- **MCP サーバー**: 標準化されたModel Context Protocolを通じて特定の機能を公開する軽量プログラム
- **データソース/サービス**: MCPサーバーがアクセスするローカルまたはリモートのデータまたはサービス

ローカルMCPサーバーとは対照的に、リモートMCPサーバーはインターネット上に展開され、複数のクライアントやユーザーからアクセス可能です[Cloudflare Blog](https://github.com/cloudflare/workers-oauth-provider)。

### 2.2 主要コンポーネント

1. **プロトコル層**:
   - `Protocol`: 基本的な通信インフラを処理
   - `Client`: サーバーとの接続を開始・維持
   - `Server`: クライアントからのリクエストを処理し応答

2. **トランスポート層**:
   - リモートMCPでは、以下の通信メカニズムをサポート:
     - **Server-Sent Events (SSE)**: サーバーからクライアントへのストリーミングにHTTP GETを使用し、クライアントからサーバーへの通信にHTTP POSTを使用
     - **Streamable HTTP**: 単一のHTTPエンドポイントを使用して双方向のメッセージングを簡素化する新しいトランスポート方式（2025年3月導入）[Cloudflare Developers](https://modelcontextprotocol.io/specification/2025-03-26/basic/transports#streamable-http)

3. **メッセージタイプ**:
   - **リクエスト**: 相手側からの応答を期待
   - **結果**: リクエストへの成功応答
   - **エラー**: リクエストの失敗を示す
   - **通知**: 応答を期待しない一方向のメッセージ

4. **機能コンポーネント**:
   - **リソース**: ユーザーまたはAIモデルが使用するコンテキストとデータ
   - **プロンプト**: ユーザー向けのテンプレート化されたメッセージとワークフロー
   - **ツール**: AIモデルが実行する関数

## 3. トランスポートプロトコル

### 3.1 Server-Sent Events (SSE)

SSEは現在、ほとんどのリモートMCPクライアントでサポートされているトランスポート方式です。

**技術的特徴**:
- 2つの別々のエンドポイントを使用:
  - **GETエンドポイント**: 接続確立用（SSEストリームを返し、セッションIDを生成）
  - **POSTエンドポイント**: クライアントからのメッセージ処理用
- セッションIDはクエリパラメータとして追加される
- サーバーからクライアントへの一方向ストリーミングに最適化

**実装例**:
```javascript
// SSEサーバーの例
app.get('/sse', (req, res) => {
  res.setHeader('Content-Type', 'text/event-stream');
  res.setHeader('Cache-Control', 'no-cache');
  res.setHeader('Connection', 'keep-alive');
  
  const sessionId = generateSessionId();
  const messagingUrl = `/message?sessionId=${sessionId}`;
  
  // エンドポイントイベントを送信
  res.write(`event: endpoint\ndata: ${JSON.stringify({ messagingUrl })}\n\n`);
  
  // クライアントとの接続を維持
  const keepAlive = setInterval(() => {
    res.write(': keepalive\n\n');
  }, 30000);
  
  req.on('close', () => {
    clearInterval(keepAlive);
  });
});

// メッセージングエンドポイント
app.post('/message', (req, res) => {
  const { sessionId } = req.query;
  const message = req.body;
  
  // セッションの検証
  if (!isValidSession(sessionId)) {
    return res.status(401).json({ error: 'Invalid session' });
  }
  
  // メッセージの処理
  const response = processMessage(sessionId, message);
  res.json(response);
});
```

### 3.2 Streamable HTTP

Streamable HTTPは、2025年3月に導入された新しいトランスポート方式で、今後SSEに取って代わることが期待されています。

**技術的特徴**:
- 双方向メッセージングに単一のHTTPエンドポイントを使用
- セッションIDはHTTPヘッダー（例: `mcp-session-id`）を通じて渡される
- GETとPOSTの両方のメソッドをサポートする単一エンドポイント
- より柔軟なセッション管理と、必要に応じてステートレス操作をサポート

**実装例**:
```javascript
// Streamable HTTPサーバーの例
app.all('/mcp', (req, res) => {
  const sessionId = req.headers['mcp-session-id'];
  
  if (req.method === 'GET') {
    // SSEストリームの確立（オプション）
    res.setHeader('Content-Type', 'text/event-stream');
    res.setHeader('Cache-Control', 'no-cache');
    res.setHeader('Connection', 'keep-alive');
    
    // クライアントとの接続を維持
    const keepAlive = setInterval(() => {
      res.write(': keepalive\n\n');
    }, 30000);
    
    req.on('close', () => {
      clearInterval(keepAlive);
    });
  } 
  else if (req.method === 'POST') {
    const message = req.body;
    
    // 初期化リクエストの場合
    if (!sessionId && message.method === 'initialize') {
      const newSessionId = generateSessionId();
      // 新しいセッションIDでレスポンス
      res.setHeader('mcp-session-id', newSessionId);
      const response = handleInitialize(newSessionId, message);
      res.json(response);
    } 
    // 既存セッションへのメッセージ
    else if (sessionId && isValidSession(sessionId)) {
      const response = processMessage(sessionId, message);
      res.json(response);
    } 
    else {
      res.status(401).json({ error: 'Invalid session' });
    }
  } 
  else {
    res.status(405).json({ error: 'Method not allowed' });
  }
});
```

### 3.3 トランスポートの比較

| 特徴 | SSE | Streamable HTTP |
|------|-----|-----------------|
| エンドポイント | 2つの別々のエンドポイント (GET, POST) | 単一のエンドポイント |
| セッション管理 | クエリパラメータ | HTTPヘッダー |
| ステートレス操作 | 限定的 | サポート |
| クライアントのサポート | 広範（レガシー） | 採用が拡大中 |
| 実装の複雑さ | 別々のエンドポイント管理が必要 | 単一エンドポイントで簡略化 |
| スケーラビリティ | 制限あり | より高い |

## 4. 認証と承認

リモートMCPサーバーでは、リソースへのアクセスを制御するために認証と承認が不可欠です。MCPはOAuth 2.1標準に基づく認証メカニズムを実装しています。

### 4.1 OAuth 2.1実装

MCPの認証仕様は、以下を必要とします:

- OAuthのベストプラクティスへの準拠（RFC 6749および8252）
- すべてのクライアントに対するPKCE（Proof Key for Code Exchange）の義務化
- クライアントとサーバーの両方に対するメタデータ検出サポート
- 動的クライアント登録サポート（RFC 7591）

### 4.2 認証フロー

リモートMCPサーバーの一般的な認証フローは次のとおりです:

1. MCPクライアントがMCPサーバーとの標準的なOAuthフローを開始
2. MCPサーバーがユーザーをサードパーティの認証サーバー（例: GitHub、Auth0）にリダイレクト
3. ユーザーがサードパーティのサーバーで認証
4. サードパーティのサーバーが認可コードとともにMCPサーバーにリダイレクト
5. MCPサーバーが認可コードをサードパーティのアクセストークンと交換
6. MCPサーバーがサードパーティのセッションに結び付けられた独自のアクセストークンを生成
7. MCPサーバーが元のOAuthフローをMCPクライアントと完了

このプロセスにより、MCPサーバーは認証をサードパーティにデリゲートしながら、クライアントに対する独自のアクセス制御を維持できます[Auth0](https://<your-tenant>.auth0.com/.well-known/openid-configuration)。

### 4.3 実装例

Cloudflareでのリモートサーバーの認証実装例:

```javascript
import OAuthProvider from "@cloudflare/workers-oauth-provider";
import MyMCPServer from "./my-mcp-server";
import MyAuthHandler from "./auth-handler";

export default new OAuthProvider({
  apiRoute: "/sse",
  apiHandler: MyMCPServer.mount('/sse'),
  defaultHandler: MyAuthHandler,
  authorizeEndpoint: "/authorize",
  tokenEndpoint: "/token",
  clientRegistrationEndpoint: "/register",
});

// 認証コールバック処理
const { redirectTo } = await c.env.OAUTH_PROVIDER.completeAuthorization({
  request: oauthReqInfo,
  userId: login,
  metadata: { label: name },
  scope: oauthReqInfo.scope,
  props: {
    accessToken,  // 暗号化されて保存され、MCPクライアントには送信されない
  },
});
return Response.redirect(redirectTo);
```

## 5. セキュリティの考慮事項

リモートMCPサーバーは、外部からアクセス可能なため、強力なセキュリティ対策が不可欠です。

### 5.1 主なセキュリティリスク

1. **認証情報の漏洩**: リモートMCPサーバーは外部サービスに接続するための認証情報を必要とし、これらの漏洩リスクがある
2. **ネットワーク露出**: 構成データが通信ネットワークを通過することによる潜在的な傍受リスク
3. **攻撃表面の拡大**: リモート接続の開放性により、脆弱性が増加
4. **サプライチェーン攻撃**: 信頼されていないMCPサーバーが悪意のあるコードを含む可能性

### 5.2 セキュリティのベストプラクティス

1. **サーバー開発者向け**:
   - サーバーのソースコードに認証情報を埋め込まないこと
   - すべての通信に安全なチャネル（TLS）を使用すること
   - クライアントの認証情報を保存しないこと
   - ログに認証情報が露出しないようにすること
   - セキュリティを念頭に置いた開発（適切なユーザー入力の検証など）

2. **MCPプロトコルユーザー向け**:
   - 敏感な操作の前に手動承認を義務付ける
   - MCPレベルでLLMが実行できるコマンドを制限する
   - 可能な場合は敏感なローカルサーバーをコンテナ化する
   - 信頼されていないサーバーからのコンテンツを敏感な情報と同じプロンプトに含めない
   - 機能の説明が検証されフリーズされない限り、信頼されていないMCPサーバーでユーザー意図検出を実行しない
   - 公開されたMCPサービスも同様にセキュリティテストを行う

### 5.3 トランスポートセキュリティ

- **SSEトランスポート**:
  - DNSリバインディング攻撃に対する脆弱性がある
  - 受信SSE接続のOriginヘッダーを常に検証すること
  - ローカルで実行する場合、すべてのネットワークインターフェイス（0.0.0.0）にサーバーをバインドするのを避け、代わりにlocalhostのみ（127.0.0.1）にバインドすること
  - すべてのSSE接続に適切な認証を実装すること

- **Streamable HTTP**:
  - 単一エンドポイント設計は、適切なアクセス制御が実装されていれば、セキュリティプロファイルの向上につながる
  - ヘッダーベースのセッション管理は適切な検証が必要

## 6. パフォーマンスとスケーリング

### 6.1 スケーリング手法

リモートMCPサーバーは、以下の方法でスケールできます:

1. **水平スケーリング (スケールアウト)**:
   - トラフィック需要に応じて複数のMCPサーバーノードをスピンアップ
   - ロードバランサーを使用して需要を分散
   - ステートレスな設計を優先してノード間の依存関係を最小化

2. **垂直スケーリング (スケールアップ)**:
   - より強力なハードウェアリソースにサーバーをデプロイ
   - 単一インスタンスのパフォーマンスを最適化

3. **キャッシング**:
   - 頻繁にアクセスされるデータをキャッシュしてレイテンシを削減
   - データベースの負荷を減らすために結果をキャッシュ

4. **非同期処理**:
   - 長時間実行されるタスクは非同期プロセスにオフロード
   - 進行状況の通知メカニズムを使用してユーザーに更新を提供

### 6.2 ステート管理

リモートMCPサーバーは、Cloudflareの場合、以下のようにステートを管理できます:

- **Durable Objects**: セッションごとの状態を維持し、プロトコルのインタラクティブ性を保証
- **KV Storage**: 認証トークンや稀にアクセスされるデータに

実装例:
```javascript
type State = { counter: number };

export class MyMCP extends McpAgent<Env, State, {}> {
  server = new McpServer({
    name: "Demo",
    version: "1.0.0",
  });

  initialState: State = {
    counter: 1,
  };

  async init() {
    this.server.resource(`counter`, `mcp://resource/counter`, (uri) => {
      return {
        contents: [{ uri: uri.href, text: String(this.state.counter) }],
      };
    });

    this.server.tool('add', 'Add to the counter', { a: z.number() }, async ({ a }) => {
      this.setState({ ...this.state, counter: this.state.counter + a });
      return {
        content: [{ type: 'text', text: String(`Added ${a}, total is now ${this.state.counter}`) }],
      };
    });
  }

  onStateUpdate(state: State) {
    console.log({ stateUpdate: state });
  }
}
```

## 7. 実装例と産業応用

### 7.1 Atlassianのリモートサーバー実装

Atlassianは、JiraとConfluenceのデータをClaudeなどのAIツールと接続するリモートMCPサーバーを提供しています:

- **主な機能**:
  - Jira作業項目やConfluenceページの要約
  - Claude内から直接Jira作業項目やConfluenceページを作成
  - 複数のアクションを一度に実行（一括でのissueやページの作成など）
  - 多様なソースからのコンテキストを用いたJira作業項目の充実

- **技術詳細**:
  - クラウドホストされたサービスとしてのリモートMCP
  - OAuthを使用した認証と既存の権限制御の尊重
  - CloudflareのAgents SDKを使用した構築
  - 内部的には、Teamwork Graphと呼ばれるデータインテリジェンスレイヤーを活用

### 7.2 Cloudflareによるデプロイメント例

Cloudflareは、リモートMCPサーバーの開発と展開を簡素化する包括的なソリューションを提供しています:

```javascript
// 最小限のMCPサーバー
import { McpAgent } from "agents/mcp";
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { z } from "zod";

export class MyMCP extends McpAgent {
  server = new McpServer({
    name: "Demo",
    version: "1.0.0",
  });
  async init() {
    this.server.tool("add", { a: z.number(), b: z.number() }, async ({ a, b }) => ({
      content: [{ type: "text", text: String(a + b) }],
    }));
  }
}

// デプロイメント
export default {
  fetch(request: Request, env: Env, ctx: ExecutionContext) {
    const { pathname } = new URL(request.url);
    if (pathname.startsWith('/sse')) {
      return MyMcpAgent.serveSSE('/sse').fetch(request, env, ctx);
    }
    if (pathname.startsWith('/mcp')) {
      return MyMcpAgent.serve('/mcp').fetch(request, env, ctx);
    }
  },
};
```

### 7.3 産業応用例

リモートMCPサーバーは、さまざまな業界で採用されています:

1. **プロダクティビティツール**:
   - Atlassianのような企業があれば、統合されたワークフローが可能
   - チケットやドキュメントの作成、要約、充実が自動化

2. **開発者ツール**:
   - GitHubとGitLabのようなコード管理プラットフォームとの統合
   - コードレビュー、課題管理、コミットのスマート要約

3. **データベース**:
   - PineconeなどのベクターデータベースとLLMの接続
   - 複雑なデータクエリと分析の自然言語インターフェイス

4. **コラボレーションプラットフォーム**:
   - Slackや通信ツールとの統合
   - AIアシスタントによる会話の分析と会議のフォローアップ

## 8. 将来の方向性と展望

### 8.1 プロトコルの進化

- **トランスポートの統合**: StreamableHTTPへの移行の継続とSSEの段階的廃止
- **認証メカニズムの強化**: OAuth実装の改良と拡張
- **標準化の進展**: さらなる業界採用に伴うプロトコル仕様の調整

### 8.2 展望とトレンド

- **エンタープライズ採用**: 企業レベルでのAIツール統合のためのリモートMCPサーバーの増加
- **セキュリティと規制対応**: より厳格なセキュリティ基準とコンプライアンス機能
- **エコシステムの拡大**: さまざまなサービスやプラットフォームにわたる統合MCPサーバーの増加
- **パフォーマンス最適化**: より効率的なリモート通信のための新しいトランスポート技術

### 8.3 オープンソースと標準化

- リモートMCPサーバーの開発を刺激するオープンソースツールとプロジェクトの継続的成長
- AI業界におけるさらなる相互運用性と標準化の進展
- プロトコルベストプラクティスの継続的改良

## 9. まとめ

リモートMCP（Model Context Protocol）は、AIツールとエンタープライズシステムの間の標準化されたコミュニケーションを可能にする革新的なプロトコルです。ローカルのスタンダード入出力ベースの接続から進化し、OAuth対応認証とさまざまなトランスポート層をサポートするウェブアクセス可能なインターフェイスを提供します。

主な利点は以下の通りです:

- **標準化されたインターフェイス**: AIアプリケーションとデータソース間の一貫したコミュニケーション
- **セキュリティ**: オープンスタンダードに基づく強力な認証と承認
- **スケーラビリティ**: 効率的なリモート通信のための設計
- **柔軟性**: 複数のトランスポート層と実装オプションをサポート

AtlassianやCloudflareなどの主要テクノロジー企業の採用により、リモートMCPサーバーは、AIアプリケーション統合の基礎技術となっています。StreamableHTTPの導入と継続的なプロトコル仕様の改良により、リモートMCPはより広範な業界採用に向けてさらに改善されると予想されます。

## 参考文献

- [Model Context Protocol Introduction](https://modelcontextprotocol.io/introduction)
- [Model Context Protocol Specification](https://modelcontextprotocol.io/specification/2025-03-26)
- [Cloudflare Remote MCP Servers](https://blog.cloudflare.com/remote-model-context-protocol-servers-mcp/)
- [NCC Group: 5 MCP Security Tips](https://github.com/modelcontextprotocol)
- [Atlassian Remote MCP Server](https://community.atlassian.com/forums/Atlassian-Platform-articles/Using-the-Atlassian-Remote-MCP-Server/ba-p/3005104)
- [Cloudflare Agents Documentation](https://modelcontextprotocol.io/specification/2025-03-26/basic/transports#streamable-http)
- [Auth0 Introduction to MCP and Authorization](https://<your-tenant>.auth0.com/.well-known/openid-configuration)
- [GitGuardian Blog: MCP Security](https://blog.gitguardian.com)
- [GitHub: Remote-MCP](https://github.com/ssut/Remote-MCP/blob/main/examples/cloudflare-workers)