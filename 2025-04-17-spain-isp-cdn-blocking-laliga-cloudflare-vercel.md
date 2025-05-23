# **スペインにおけるISPによるCDNトラフィック遮断：LaLiga、Cloudflare、Vercelを巡る技術・法・影響分析**

## **要旨**

本レポートは、スペインの主要インターネットサービスプロバイダー（ISP）が、スペインプロサッカーリーグ（LaLiga）が取得した裁判所命令に基づき、CloudflareやVercelなどのコンテンツデリバリーネットワーク（CDN）が使用するIPアドレスへの通信を遮断している問題について、その背景、現状、および影響を分析するものである。この遮断は、LaLigaの著作権で保護されたサッカー試合のライブ配信における著作権侵害（海賊版配信）対策を目的としているが、CDNが採用する共有IPアドレスアーキテクチャの性質上、著作権侵害とは無関係の数千もの正当なウェブサイトやオンラインサービスへのアクセスを巻き添えで妨害する「巻き添え被害」を引き起こしている[^1]。

LaLigaは、CDNが著作権侵害活動を助長していると主張し、IPアドレス遮断は暗号化技術（ECH）の普及により唯一有効な手段であると正当化している[^3]。一方、CloudflareやVercelは、この措置を「不均衡」かつ「違法」であり、オープンなインターネットへの脅威であると強く非難し、法的措置を講じている[^1]。裁判所は当初、Cloudflareの異議申し立てを却下し、LaLigaの立場を強化した[^6]。

遮断は主に試合開催中の週末に実施されるとされるが、試合時間外にも継続しているとの報告もあり、ISP（Telefónica、Vodafone、Orangeなど）は裁判所命令に従って遮断を実行している[^5]。この状況は、スペイン国内のユーザーや企業に広範な混乱と経済的損害を与え、ネット中立性、表現の自由、デジタル権に関する重大な懸念を引き起こしている[^9]。本件は、著作権保護とインターネットインフラの現実との間の緊張関係を浮き彫りにする重要な事例となっている。

## **はじめに**

現代のインターネットにおいて、コンテンツデリバリーネットワーク（CDN）は不可欠な役割を担っている。CloudflareやVercelのようなCDNプロバイダーは、世界中に分散配置されたサーバーネットワークを利用し、ウェブサイトのコンテンツをユーザーに近い場所から配信することで、表示速度の向上、サーバー負荷の軽減、そしてDDoS攻撃などのセキュリティ脅威からの保護を提供する[^1]。これらの効率性、セキュリティ、パフォーマンス向上のため、CDNは多くの場合、「共有IPアドレス」を使用する。これは、単一のIPアドレスが、小規模な個人ブログから大規模なプラットフォーム（例：ChatGPT）に至るまで、数百、数千もの異なるウェブサイトやサービスによって共有されることを意味する[^2]。このアーキテクチャは、インターネットの効率的な運用に貢献する一方で、特定のIPアドレスを標的とするブロッキング措置が広範な影響を及ぼす可能性を内包している。

一方、スペインのプロサッカーリーグであるLaLigaは、その放映権から得られる収益を守るため、長年にわたりオンラインでの著作権侵害（無許可の試合ストリーミング）との戦いを続けてきた[^1]。過去にも様々な法的・技術的手段を用いて海賊版配信サイトへの対策を講じてきた経緯がある[^13]。

本レポートの中心テーマは、このLaLigaの著作権保護の目的と、それを実現するために裁判所の承認を得て行使されるようになったIPアドレス遮断という強硬な手段が、現代インターネットの基盤技術であるCDNの運用アーキテクチャと衝突し、スペイン国内で広範なインターネットアクセス障害を引き起こしている現状である。この衝突は、ブロッキング措置の比例原則、ネット中立性、そしてデジタル時代の基本的権利に関する深刻な問題を提起している。

## **ブロッキング事案の確認と時系列**

スペインのISPがCloudflareおよびVercelのインフラに影響を与えるブロッキング措置を実施している事実は、影響を受けたユーザーからの報告、関連企業（Cloudflare、Vercel）の公式声明、および多数の報道によって確認されている[^1]。具体的には、自身のウェブサイトにアクセスできなくなったという報告[^14] や、多数のウェブサイトが閲覧不能になるという広範なブラウジング問題[^4] が挙げられる。

この問題の主要な出来事を時系列で整理すると以下のようになる。これは主に情報源[^12] に基づき、他の情報源で補足したものである。

-   **2024年12月:**
    > バルセロナ商事裁判所第6号法廷が、LaLigaに対し、著作権侵害に関連するIPアドレスを特定し、ISPにその遮断を週単位で要求する権限を与える判決を下す
    > [^5]。(*注記: [^7]
    > は2022年の判決が支持されたと言及しており、法的根拠は2024年後半より前に存在する可能性を示唆するが、具体的なIP遮断命令は2024年後半/2025年初頭から強調されているように見える*)

-   **2025年2月9日:**
    > 最初の主要なブロッキング措置が報告される。海賊版プラットフォームDuckVisionが使用しているとされるCloudflareインフラが標的となり、広範なアクセス障害が発生
    > [^12]。ユーザーからの苦情が表面化し始める [^17]。

-   **2025年2月15日:**
    > LaLigaが声明を発表。Cloudflareが犯罪組織と共謀し、違法コンテンツを保護していると非難。遮断されたIP経由での児童ポルノへのアクセスを確認し警察に通報したとも主張
    > [^3]。

-   **2025年2月16日:**
    > さらなるブロッキングが報告される。Cloudflareを利用しているとされる海賊版IPTVサービスDazcFutboliosおよびRBTV77が標的となる
    > [^1]。

-   **2025年2月19日:**
    > CloudflareがLaLigaに対し法的措置を開始したと発表。ブロッキング命令は不均衡かつ違法であり、オープンインターネットへの脅威であると主張
    > [^1]。

-   **2025年3月:**
    > バルセロナ商事裁判所が、Cloudflareによるブロッキング命令に対する異議申し立て（無効事由の申立て）を却下。損害の証明が不十分であるとし、LaLigaの立場を強化
    > [^6]。RootedCON（影響を受けた可能性のある関連団体、おそらくカンファレンス主催者
    > [^7]）も異議申し立てに関与。

-   **2025年3月6日:** 双方から強硬な声明が出され、論争が継続・激化
    > [^12]。

-   **2025年4月12日～13日:**
    > LaLigaが特定の著作権侵害サイトに関してVercelに侵害報告を送付
    > [^19]。

-   **2025年4月15日:**
    > Vercelが声明を発表。自社インフラがブロッキングされていることを確認し、IPブロックの無差別性を批判。違法コンテンツを積極的に削除しているにもかかわらず、警告なしにブロックされたと主張
    > [^5]。特定のVercel IPアドレスがブロックされていることを確認 [^5]。

-   **2025年4月16日:**
    > LaLigaの主張（ブロックは試合日のみ、Vercelの対応が遅い）と、Vercelおよびユーザー側の主張（ブロックは試合日以外も継続、Vercelは迅速に対応した）との間に食い違いがあることが報道される
    > [^8]。

この時系列は、事態が段階的にエスカレートしたことを示している。まず裁判所命令があり、次にその実行（最初はCloudflare、後にVercelを標的）、そして法的対抗措置と公然たる論争へと発展した。裁判所がCloudflareの最初の異議申し立てを退けたことはLaLigaの立場を強め、さらなる行動を促した可能性がある。これは、法的権限付与、執行措置、技術的影響、そして法的対抗措置という、動的な相互作用を示している。つまり、最初の裁判所命令（2024年12月またはそれ以前）が法的ツールを提供し、LaLigaはおそらく以前の方法の非効率性や著作権侵害の増加を理由に、2025年2月からそれを積極的に使用し始めた。その広範な影響は直ちにCloudflareからの反応（法的措置）を引き起こした。裁判所がLaLiga側を支持したこと（2025年3月）は先例となり、CDNがその特定の法的枠組み内で巻き添え被害を理由にブロックに異議を唱えることを困難にし、LaLigaが標的を拡大する（Vercel、2025年4月）ことにつながった可能性がある。

## **ブロッキングの根拠：LaLiga、著作権侵害、裁判所命令**

LaLigaが表明している動機は明確である。それは、貴重なライブサッカー試合放送の無許可配信を阻止し、知的財産権と放送収入を保護することである[^1]。LaLigaは著作権侵害による重大な経済的損害を主張し、自らの行動を違法行為の撲滅と位置付けている[^1]。LaLigaのコンテンツを違法配信するIPアドレスの50%以上がCloudflareによって保護されているとも主張している[^1]。

このブロッキング措置の法的根拠となっているのは、バルセロナ商事裁判所第6号法廷による命令である[^1]。この命令は、LaLigaが著作権侵害に関連すると特定したIPアドレスのリストをISPに提供し、ISPにそのIPアドレスへのアクセスを（しばしば週単位またはそれ以上の頻度で）遮断するよう義務付ける権限をLaLigaに与えている[^4]。

スペインの法律（Act
21/2014）および欧州司法裁判所（CJEU）の判例法は、コンテンツを直接ホストしていなくても、インターネットアクセスを提供する中間事業者（ISP）に対して差止請求を行うことを認めている[^13]。これが、ISPを著作権侵害対策の標的とすることを法的に可能にしている背景にある。

Cloudflareによる異議申し立てが却下された際、裁判所は特に、Cloudflareがブロックによって引き起こされた損害の十分な証拠を提出できなかった点を理由として挙げた。裁判官は、この措置を損害を引き起こすものではなく、「海賊版コンテンツへのアクセスを阻止する」ものと位置付けた[^7]。この判断は、著作権侵害の防止という目的を、インフラを共有する無関係な第三者への潜在的損害よりも優先するものと解釈でき、CDNやデジタル権利擁護団体から強く批判されている。

特定のドメインやURLではなく、IPアドレス全体をブロックする手法への移行は、ブラウザ（例：Chrome）におけるEncrypted
Client Hello（ECH）のような技術の利用拡大が背景にあると報告されている[^4]。ECHは、HTTPS通信の初期段階（TLSハンドシェイク）でアクセス先のホスト名（SNI）を暗号化するため、ISPが通信内容を検査して特定のサイトを識別することが困難になる[^5]。LaLigaは、このような状況下では、裁判所の著作権侵害対策命令を遵守するための唯一実行可能な技術的手段がIPアドレスブロッキングであると主張したと考えられる[^4]。

この法的枠組みは、LaLigaに大きな権限を与え、特に最初の異議申し立てが却下された後は、巻き添え被害に対する直接的な抑制策が限定的であるように見える。裁判所は、共有インフラに内在する巻き添え被害の可能性を過小評価したか、法的に軽視した上で、IPブロッキングというより抜本的な手段を承認した可能性がある。Cloudflareの異議申し立てが却下された際、裁判所が申立人への「証明された」損害の欠如と著作権侵害阻止という主たる目標に焦点を当てたことで、この立場はさらに強化された。これにより、広範で不均衡な可能性のあるブロッキングが法的に容認される環境が生まれたと言える。

さらに、一部で報告されている手続き上の側面、すなわちTelefónica（スペインの大手ISP）が自身および他の通信事業者を相手取って訴訟を起こしたという点は[^4]、ブロッキングプロセスを正当化し迅速化するための協調的な動きを示唆している。LaLigaはISPに迅速なブロック実行を求めている。著作権侵害サイトやCDNに対する通常の法的手続きは時間がかかったり、異議に直面したりする可能性がある。もしTelefónicaのような大手ISPが、LaLiga/裁判所の命令を引用し、自身および他の協力的なISPに対して（形式的に）訴訟を起こせば、ブロック義務を確認する判決を迅速に得ることができる。これは業界全体での法的必要性とコンプライアンスの外観を作り出し、他者がLaLigaによって指示された技術的実装に抵抗したり異議を唱えたりすることを困難にする可能性がある。これは執行を効率化する戦略と考えられる。

## **技術的実行と影響範囲**

ブロッキングの技術的メカニズムは、従来のDNSベースやSNI（Server Name
Indication）ベースのフィルタリングから、ISPによる直接的なIPアドレスブロッキング（ヌールルーティングまたはインターセプト）へと移行した点が重要である[^1]。この移行の主な理由は、前述の通り、ECH（Encrypted Client
Hello）技術の普及により、ISPが暗号化されたトラフィック内のSNIヘッダーを検査して特定のターゲットホスト名を識別することが困難になったためである[^4]。これは、通常、特定のドメインを対象に行われるISPレベルのブロッキングとは対照的である[^5]。ただし、ISPによる実装にはばらつきがある可能性も指摘されており、一部のISPはトラフィックを完全に破棄（ブラックホール化）する一方、他のISPはDPI（Deep
Packet
Inspection）を使用するものの、その実装が不適切である可能性も示唆されている[^6]。

裁判所命令に基づきブロッキングを実施している、または実施していると報告されているスペインのISPには、Telefónica（Movistar、O2ブランドを含む）、Vodafone
Spain、Orange Spain、およびDigiMobilが含まれる[^5]。特にMovistar/Telefónicaは、ユーザーからの苦情や報告で頻繁に言及されている[^9]。また、Vodafoneは裁判手続きにおいてLaLiga側に同調したと報じられている[^7]。

ブロッキングの範囲に関して最も重要な点は、それがドメインではなくIPアドレスを対象としていることである[^4]。そして、CloudflareやVercelのようなCDNは、効率化のために共有IPアドレスを使用しており、単一のIPアドレスが数百から数千もの異なる、無関係なウェブサイトやサービスをホストしている[^1]。

この結果、ブロッキングの影響は意図された著作権侵害サイトをはるかに超え、正当なビジネス、開発者ツール、個人ブログ、さらには主要なプラットフォームまでもが、LaLigaが標的としたIPアドレスを共有しているという理由だけでアクセス不能になる事態が発生している[^1]。具体的に影響を受けたとされるサービスには、ChatGPT、GitHub、スペインのスタートアップTinybird、雑誌Hello
Magazineなどが含まれる[^2]。ブロックされたIPアドレスの具体例としては、Vercelが使用する
66.33.60.129 および 76.76.21.142 が挙げられている[^5]。ユーザーからは他のCloudflare IP（例：104.21.x.x [^14]、188.114.x.x
[^23]）への影響も報告されている。

**表1: ISPの関与とブロッキング手法**

  -----------------------------------------------------------------------------------------------------------------------------------
  **ISP名**         **親会社（該当する場合）**   **確認された関与（情報源）**   **報告されているブロッキング手法（情報源）**
  ----------------- ---------------------------- ------------------------------ -----------------------------------------------------
  Movistar          Telefónica                   [^8]                            IPヌールルーティング/インターセプト [^17]

  O2                Telefónica                   [^8]                            IPヌールルーティング/インターセプト [^17]

  Vodafone Spain    Vodafone Group               [^5]                            IPブロッキング（DPIによる不完全な実装の可能性あり）
                                                                                [^7]

  Orange Spain      Orange S.A.                  [^5]                            IPブロッキング（DPIによる不完全な実装の可能性あり）
                                                                                [^21]

  DigiMobil Spain   Digi Communications          [^8]                            IPヌールルーティング/インターセプト [^21]
  -----------------------------------------------------------------------------------------------------------------------------------

ECHの採用によってSNIインスペクションが困難になったことが、IPアドレスブロッキングという技術的決定につながり、これが不均衡な巻き添え被害の直接的な原因となっている。これは、洗練され階層化されたインフラ（CDN）に対して、鈍器のような執行ツールを適用している状況と言える。暗号化技術（ECH）はプライバシーとセキュリティを向上させるが、ISPレベルでのフィルタリングに必要なSNIを不明瞭にする。著作権者（LaLiga）は依然としてブロッキングを要求し、裁判所は唯一残された選択肢とされる方法、すなわちIPブロッキングを承認する。ISPはこの技術的に単純だが過度に広範な方法を実装する。CDNは効率のために共有IPに依存しているため、1つのIPをブロックすると、そのIPでホストされている可能性のある数千もの無関係なサイトがダウンする。選択された技術的解決策が、広範な負の外部性を直接生み出しているのである。

また、ISPによる実装のばらつき [^21]
は、ブロッキング命令に対する一貫性の欠如、あるいは異なる技術的能力や解釈を示唆しており、ユーザーが利用するISPによって異なる体験をもたらしている可能性がある。裁判所命令は特定のIPのブロックを義務付けているが、正確な技術的方法（ヌールルート、IP/SNIに対するDPIフィルタリングなど）は指定されていないか、均一に適用されていない可能性がある。異なるISPは、自社のネットワークアーキテクチャや解釈に基づいて異なる技術を使用する可能性がある。一部は「正しく」実装（IPへの全トラフィックをブロック）するかもしれないが、他はより微妙なフィルタリング（[^21]
でOrange/Vodafoneについて言及されているDPIチェックなど）を試みるが失敗し、異なるネットワーク上のユーザーに対して一貫性のないブロッキング効果をもたらしているのかもしれない。

## **関係者の反応と進行中の論争**

この問題に関与する主要な関係者は、それぞれ異なる立場を取り、論争は継続している。

-   **Cloudflare:**

    -   **立場:** ブロック措置を「不均衡 (disproportionate)」、「違法
        > (unlawful)」、そして「オープンインターネットへの脅威」として激しく反対している
        > [^1]。LaLigaが基本的権利よりも商業的利益を優先していると主張
        > [^1]。LaLigaが通知なしに命令を取得し、潜在的な損害を隠蔽したと主張
        > [^1]。

    -   **行動:** 命令に異議を唱えるため法的措置を提起
        > [^1]。バルセロナ裁判所により最初の異議申し立ては却下された
        > [^6]。ヨーロッパの他の国々でも同様の法的課題に直面している
        > [^7]。影響を受けるユーザーと積極的にコミュニケーションを取っている
        > [^14]。

    -   **主張:** 共有IPアーキテクチャと避けられない巻き添え被害を強調
        > [^1]。LaLigaによる共謀の非難を、責任転嫁の戦術であるとして拒否
        > [^1]。膨大なCDNトラフィックから特定の侵害ストリームをフィルタリングすることの困難さ/不可能性を指摘
        > [^4]。

-   **Vercel:**

    -   **立場:**
        > Cloudflareと同様に、無差別なブロッキングと正当なサイトへの巻き添え被害に対する懸念を表明
        > [^5]。専門の不正利用対策チームを通じて違法コンテンツを積極的に監視・削除していることを強調
        > [^5]。

    -   **行動:** ブロックを非難する公式声明を発表
        > [^5]。LaLiga/当局とのコミュニケーションを試みた
        > [^8]。緩和策を検討中 [^5]。

    -   **主張:**
        > コンプライアンスプロセスにもかかわらず、事前の警告なしにブロックされたと主張
        > [^5]。影響を受けた具体的な正当な顧客（Tinybird、Hello
        > Magazine）を例示
        > [^5]。ブロックが試合中のみ行われるというLaLigaの主張に反論
        > [^19]。

-   **LaLiga:**

    -   **立場:**
        > CDNによって助長される広範な著作権侵害に対抗するために必要な措置であると正当化
        > [^1]。Cloudflare（および暗に他のCDN）を「利益のために犯罪組織を故意に保護」し、違法活動の「デジタルシールド」として機能していると非難
        > [^1]。CDNが協力（著作権侵害対策への）を拒否していると主張
        > [^3]。影響を軽視し、影響を受けたユーザーを「インターネット上の一部のマニア（cuatro
        > frikis）」と呼び、ブロックは「選択的」であると主張 [^7]。

    -   **行動:** IPブロックを可能にする裁判所命令を確保
        > [^4]。ISPにブロック対象IPリストを週単位または定期的に提供
        > [^4]。自らの行動を擁護し、Cloudflareの慣行を攻撃する強硬な公式声明を発表
        > [^3]。ブロックされたIP経由での児童ポルノアクセス疑惑を警察に通報
        > [^3]。影響を受けたCloudflare顧客向けの連絡窓口を提供
        > [^11]。Vercelが削除要求への対応が遅かったと主張 [^19]。

-   **ISP (Telefónica/Movistar, Vodafone, Orangeなど):**

    -   **立場:** 一般的に裁判所命令に従っているように見える
        > [^4]。Telefónica/Movistarは当初、ユーザーからの苦情に対しCloudflareに責任転嫁していた
        > [^9]。Vodafoneは裁判手続きでLaLiga側に同調し、Cloudflareの無効申し立てに反対したと報じられている
        > [^7]。一部のユーザー報告では、ISPがブロックを一貫性なく、または不適切に実装している可能性が示唆されている
        > [^21]。Telefónicaの（自身を訴えるという）法的策略は、戦略的なコンプライアンスを示唆している
        > [^4]。

    -   **行動:** LaLigaから提供されたIPブロックを実装
        > [^4]。ユーザーからの苦情に対応しているが、時に混乱を招く、または役に立たない回答をしている
        > [^9]。

-   **デジタル権利団体 & ユーザーコミュニティ:**

    -   **立場:**
        > ブロッキング措置を、不均衡で、オープンインターネット、ネット中立性、表現・情報への自由への脅威であるとして強く批判
        > [^1]。司法による監督と適正手続きの欠如に対する懸念
        > [^4]。権威主義体制下の検閲との比較
        > [^6]。正当なユーザーやビジネスへの悪影響を強調
        > [^2]。スペイン既存のデジタル権利憲章との潜在的な矛盾を指摘
        > [^28]。スペインにおける過去のデジタル権訴訟（例：Women on
        > Webのブロッキング [^29]）や政府権限への懸念（デジタル王政令
        > [^10]）に言及。

    -   **行動:** 公の場でのコメント（Hacker News, Reddit,
        > NANOGメーリングリスト
        > [^4]）、法的分析、潜在的なアドボカシー活動（RootedCONはさらなる上訴を計画
        > [^7]）。

ここには根本的な認識のずれが存在する。LaLigaはCDNを著作権侵害の（意図的かどうかにかかわらず）助長者とみなし、裁判所命令を利用して広範な技術的コンプライアンスを強制している。一方、CDNは自らを、より広範なインターネットに損害を与える不均衡な措置によって不当に標的にされた、不可欠な中立的インフラ提供者と見なしている。ISPは法的には遵守義務があるものの、ユーザーからの反発に直面し、板挟みになっている。デジタル権利団体は、検閲と基本的権利の軽視につながる危険な前例と見ている。LaLigaの焦点は狭く、自らの条件で著作権侵害を阻止することにある。CDNの焦点は広く、ネットワークの完全性を維持し、すべてのクライアントにサービスを提供することにある。裁判所命令は、ISPに対し、IPブロッキングという鈍器のようなツールを通じてLaLigaの狭い焦点に合わせることを強制する。これは必然的にCDNの運用モデルや、信頼性の高いインターネットアクセスに対するユーザー/企業の期待と衝突する。結果として生じる論争は、著作権保護と、オープンインターネットの原則および比例原則という、対立する価値観を浮き彫りにしている。

LaLigaの攻撃的なレトリック（犯罪保護の非難、影響の軽視）と、Cloudflare/Vercelの防御（コンプライアンスの主張、巻き添え被害の強調）は、コミュニケーションの断絶と立場の硬化を示しており、さらなる法的または規制的介入なしには、相互に合意可能な解決策を見出すことが困難であることを示唆している。LaLigaの強い非難
[^3]
は、Cloudflareを無責任であると描き出し、抜本的なブロッキング措置を正当化することを目的としている。Cloudflare/Vercelは、自らの正当性とブロックの不当性を強調することで対抗する
[^1]。出来事に関する矛盾した説明 [^19]
を含むこの公然たる対立は、深い不信感を示唆し、（例えば、より的を絞った技術的解決策を見つけるなどの）協力的な問題解決を困難にしている。法廷闘争が解決への主要な道筋となっている。

## **影響と結果の分析**

このブロッキング措置の最も重大かつ直接的な結果は、正当なサービスへの広範な「巻き添え被害」である。共有IPアドレスのブロッキングが、著作権侵害とは全く無関係な多数の正当なウェブサイトやサービスに必然的に影響を及ぼすことを詳述する必要がある
[^1]。以下に具体的な影響例を挙げる。

-   **ビジネスウェブサイト:**
    > オンラインストアや建築事務所など、顧客との接点や業務遂行に不可欠なサイトがアクセス不能になる
    > [^2]。

-   **開発者プラットフォームとツール:** GitHub
    > Pages、Vercel自体、データ分析プラットフォームのTinybird、開発支援ツールのCursorなど、開発者の生産性に不可欠なサービスが利用できなくなる
    > [^5]。

-   **主要なオンラインサービス:**
    > ChatGPTのようなAIサービスや、Instagram、X（旧Twitter）、Blueskyといった主要プラットフォームの一部も、IPアドレスを共有している場合、影響を受ける可能性がある
    > [^2]。

-   **個人・コミュニティサイト:**
    > 個人ブログ、フォーラム、文化情報ポータルなども影響を受ける [^2]。

-   **公共機関:**
    > 公共機関のウェブサイトさえも影響を受ける可能性が指摘されている
    > [^2]。

**表2: 影響を受けた正当なサービス/ウェブサイトタイプの例**

  -------------------------------------------------------------------------------------------------------------------------------
  **サービス/ウェブサイトタイプ**   **具体例（言及されている場合）**   **情報源**        **影響の性質（アクセス不能、断続的）**
  --------------------------------- ---------------------------------- ----------------- ----------------------------------------
  ビジネス/Eコマース                オンラインストア、建築事務所       [^2]               アクセス不能

  開発者ツール/プラットフォーム     GitHub Pages, Vercel, Tinybird,    [^5]               アクセス不能
                                    Cursor                                               

  主要AI/SaaS                       ChatGPT                            [^2]               アクセス不能

  ソーシャルメディア                Instagram, X, Bluesky              [^12]              断続的/アクセス不能
                                    （部分的/断続的な可能性あり）                        

  個人/コミュニティサイト           ブログ、フォーラム                 [^2]               アクセス不能

  ニュース/情報                     Hello Magazine                     [^5]               アクセス不能
  -------------------------------------------------------------------------------------------------------------------------------

スペイン国内のエンドユーザーに対する影響も深刻である。ブロッキング期間中、不可欠な業務ツール、オンラインショップ、情報源、エンターテイメントサイトへのアクセスが不可能になる
[^4]。特にISPからの説明が不十分な場合、ユーザーのフラストレーションと混乱は増大する
[^9]。ブロックは主に試合が行われる週末に実施されると報告されているが
[^2]、これらの時間帯以外にも、時には数日間にわたって継続しているとの報告もある
[^5]。この時間的な広がりは、ユーザーへの影響をさらに悪化させる要因となる。

より広範な経済的およびエコシステムへの影響も無視できない。

-   顧客にリーチできない、または不可欠なオンラインツールを使用できないことによる、企業の収益損失と生産性の低下
    > [^8]。

-   デジタルビジネスや開発の拠点としてのスペインの評判へのダメージ [^8]。

-   ISPおよびインターネットアクセスの安定性に対する信頼の低下 [^9]。

-   ブロックが恣意的または不可避であると認識された場合の、特定のCDNやISPの利用に対する萎縮効果（あるプロバイダーは顧客にISPの変更を推奨している
    > [^9]）。

-   オープンでアクセス可能なインターネットという原則を損ない、他の権利者や当局による同様のブロッキング措置の前例となる可能性
    > [^1]。スペインのデジタル権利憲章 [^28]
    > に概説されている原則との潜在的な矛盾。

選択された執行メカニズム（IPブロッキング）は、著作権侵害を標的とするという意図された範囲をはるかに超える負の外部性を生み出している。LaLigaを支持する法的および執行上の計算において、正当なユーザーや企業が負担する経済的および社会的コストは、大部分が無視されているように見える。LaLigaはIPブロックを通じて著作権侵害を阻止することに集中し、裁判所はこれを承認、ISPはそれを実行する。その結果、これらのIPを使用する数千もの無関係な事業体が混乱に見舞われる。この混乱は、ビジネスの損失、生産性の低下、ユーザーのフラストレーション、そしてデジタルエコシステムへの損害へとつながる。これらのコストはLaLigaが負担するものではなく、裁判所によって十分に考慮されたわけでもない（異議申し立て却下の判断
[^7]
に見られるように）が、罪のない第三者に課せられている。これは、執行方法とインターネットの構造との間の著しい不整合を示しており、非効率的で不公平な結果を招いている。

この状況は、同様の広範なブロッキング命令が発令される可能性のある管轄区域において、CDNインフラに依存するあらゆるビジネスやサービスにとっての潜在的な脆弱性を浮き彫りにしている。スペインの事例は、権利保有者が裁判所の支持を得て、ISPに共有CDN
IPのブロックを強制できることを示している。この前例が維持されるか、他の場所で模倣される場合、一般的なCDN（現代のウェブの大部分が利用している）を使用するウェブサイトは、同じIPインフラを共有する無関係な「悪質な隣人」の行動に基づいて、巻き添えでブロックされる潜在的な脆弱性にさらされることになる。これは、標準的で効率的なウェブホスティング慣行に依存するオンラインビジネスにとって、システミックなリスクを生み出す。

## **現状と緩和策**

最新の報告（2025年4月時点）によると、このブロッキングは依然として進行中の問題である
[^5]。ブロッキングのタイミングについては、矛盾する報告が存在する。

-   当初の報告やLaLigaの立場は、ブロックが主に週末のサッカー試合中に有効化され、その後解除されることを示唆している
    > [^2]。

-   しかし、Vercel、ユーザー、および一部の報告は、侵害コンテンツが削除された後でさえ、ブロックが試合時間外にも、時には数日間継続していると主張している
    > [^5]。この食い違いは、問題の性質を理解する上で極めて重要である。

ユーザーや関係者は、この状況に対処するために様々な緩和策や回避策を講じている。

-   **ユーザー側:**
    > 最も一般的な回避策は、VPN（仮想プライベートネットワーク）やCloudflare自身のWARPサービスを利用することである。これらは、トラフィックを影響を受けていないサーバー/IP経由でトンネリングすることにより、ISPレベルのブロックを迂回する
    > [^6]。皮肉なことに、ブロックされているプロバイダー（Cloudflare）のツールであるWARPがブロック回避に利用できる点が指摘されている
    > [^7]。

-   **CDN側:**
    > Cloudflareの主な緩和策は、法的手段による異議申し立てであるように見える
    > [^1]。Vercelは「緩和戦略を検討中」と述べているが、具体的な内容は明らかにされていない
    > [^5]。IPアドレスを変更することは、新しいIPもブロックされる可能性があるため、スケーラブルな解決策ではない
    > [^14]。専用IPアドレスは通常、高価なエンタープライズプランの機能であり、影響を受けるほとんどのユーザーにとって現実的な選択肢ではない
    > [^14]。侵害ドメインの停止は行われているが、LaLigaが主張すればIPブロック自体は止められない
    > [^6]。

-   **ISP側:**
    > 積極的な緩和策は報告されておらず、LaLigaのリストに従うことに焦点が当てられているように見える
    > [^4]。一部ISPの混乱した対応は、内部的な解決策や明確性の欠如を示唆している
    > [^9]。

試合時間外にもブロックが継続しているという報告が事実であれば、それはLaLigaの正当化理由や、ライブ配信の著作権侵害を対象とするという裁判所命令の意図（と推測されるもの）と矛盾する。これは、ブロック解除における技術的な失敗、意図的な過剰ブロッキング、あるいは著作権侵害インフラの動的な性質に対する誤解を示唆している可能性がある。ブロックの根拠は主にライブ試合の著作権侵害に関連しているため、論理的にはブロックは試合中のみ有効であるべきである。持続的なブロックの報告
[^8] は、この論理の破綻を示唆している。考えられる理由としては、1)
ISPがブロックを迅速に解除するのが遅い、または技術的に不可能である、2)
LaLigaが提供するリストが、削除されたコンテンツや試合スケジュールを反映するために十分に迅速に更新されない、3)
ブロッキングメカニズム自体が扱いにくく、正確に切り替えるのが難しい、4)
LaLigaが懲罰的措置または抑止力として意図的にブロックを長期間有効にしている、などが挙げられる。この食い違いは、措置の表明された比例原則を損なうものである。

ユーザーがVPNのような回避策に頼らざるを得ない状況は、このブロッキング措置が、元々標的としていた（おそらく既にそのようなツールを使用している）断固たる著作権侵害者に対しては効果が薄く、主に正当なユーザーに不便を強いていることを浮き彫りにしている。IPブロックは著作権侵害を阻止することを目的としているが、洗練された著作権侵害者はしばしばVPNや他の難読化技術を使用する。巻き添え被害を受けた正当なユーザーも、通常のウェブサイトにアクセスするためにVPN/WARPに頼るようになっている
[^6]。これは、ブロッキングが主に法を遵守するユーザーに影響を与え、彼らに回避ツールを採用させる一方で、単純なIPブロックを回避する準備ができている元の著作権侵害ターゲットには限定的な効果しかない可能性があることを意味する。これは、戦略全体の有効性と費用対効果に疑問を投げかける。

## **結論と展望**

スペインにおけるISPによるCDNトラフィック遮断問題は、LaLigaによる法的措置に裏打ちされた、しかし技術的には粗削りな著作権執行アプローチが、現代インターネットのアーキテクチャ（共有IP、CDN）と衝突し、深刻かつ不均衡な巻き添え被害をもたらしているという核心的な対立を示している。

この状況は、いくつかの重要な法的および倫理的問題を提起している。

-   **比例原則:**
    > 少数の著作権侵害ストリームを標的とするために、数千もの正当なサイトをブロックすることは、比例原則に合致するか？現在の裁判所のスタンス
    > [^7]
    > はこれを軽視しているように見えるが、CDNや権利団体は強く異議を唱えている
    > [^1]。

-   **司法的監督:**
    > 民間団体（LaLiga）が、影響に関する継続的な司法的レビューが限定的である可能性のある中で、ブロックを指示できるプロセスに対する懸念
    > [^4]。

-   **ネット中立性:**
    > ISPが、合法性に関わらずIPアドレスに基づいて広範なコンテンツをブロックすることを許可することは、たとえ裁判所命令によるものであっても、ネット中立性の原則に違反しないか？
    > [^4]。

-   **デジタル権:**
    > 正当なコンテンツへのアクセスが不可能になった場合の、情報へのアクセス権および表現の自由への影響
    > [^10]。

今後の展開として、いくつかのシナリオが考えられる。

-   **法的帰結:**
    > 比例原則や基本的権利に焦点を当てた、より高位のスペインの裁判所（RootedCONは憲法裁判所に言及
    > [^7]）または欧州の裁判所への訴訟の継続。Cloudflareの（却下された予備的異議申し立てを超えた）本案訴訟の結果が重要となる。

-   **技術的調整:**
    > ECHが課題として残るものの、より洗練されたブロッキング技術の探求への圧力が高まる可能性がある。CDNはアーキテクチャの変更を検討するかもしれないが、コストがかかる可能性が高い。

-   **政策/規制的介入:**
    > スペインまたはEUの政策立案者が、著作権執行とインターネットインフラの現実との間の対立に対処し、ブロッキング命令の法的枠組みを（デジタル王政令
    > [^10] やデジタル権利憲章 [^28] を参照しつつ）見直す可能性。

-   **現状維持:**
    > 法的挑戦が失敗し、政策変更が行われない場合、状況は持続し、継続的な混乱、ユーザーによる回避策への依存、そしてスペインのデジタル環境への長期的な損害につながる可能性がある。

結論として、スペインの事例は、複雑で階層化され、ますます暗号化される現代のインターネットに対して、従来の著作権執行パラダイムを適用する際の課題を示す重要なケーススタディとなっている。これは、侵害に対して効果的であると同時に、インターネットのアーキテクチャとユーザーの基本的権利を尊重する解決策の緊急の必要性を浮き彫りにしている。

#### 引用文献

[^1]: Cloudflare takes legal action over LaLiga\'s "disproportionate \..., https://www.broadbandtvnews.com/2025/02/19/cloudflare-takes-legal-action-over-laligas-disproportionate-blocking-efforts/

[^2]: Los bloqueos masivos de LaLiga saltan al debate global: Cloudflare, redes legítimas y miles de webs afectadas en España - Administración de Sistemas, https://administraciondesistemas.com/los-bloqueos-masivos-de-laliga-saltan-al-debate-global-cloudflare-redes-legitimas-y-miles-de-webs-afectadas-en-espana/

[^3]: LaLiga accuses Cloudflare of "knowingly protecting" piracy sites - Broadcast, https://www.broadcastnow.co.uk/broadcasting/laliga-accuses-cloudflare-of-knowingly-protecting-piracy-sites/5202278.article

[^4]: Cloudflare takes legal action over LaLiga\'s \"disproportionate blocking efforts\" \| Hacker News, https://news.ycombinator.com/item?id=43157000

[^5]: Update on Spain and LaLiga blocks of the internet - Vercel, https://vercel.com/blog/update-on-spain-and-laliga-blocks-of-the-internet

[^6]: The judge who authorized Cloudflare\'s IP address blocking in Spain due to football piracy dismisses the appeal filed by Cloudflare and RootedCDN - Reddit, https://www.reddit.com/r/CloudFlare/comments/1jke3pc/the_judge_who_authorized_cloudflares_ip_address/

[^7]: Duro revés para Cloudflare: la Justicia rechaza su demanda contra \..., https://www.xataka.com/empresas-y-economia/duro-reves-para-cloudflare-justicia-rechaza-su-demanda-laliga-bloqueo-masivo-webs

[^8]: Los bloqueos de LaLiga ya no afectan sólo a Cloudflare. El último proveedor en caer ha sido Vercel (y fuera de días de partido) - Genbeta, https://www.genbeta.com/actualidad/bloqueos-laliga-no-afectan-solo-a-cloudflare-ultimo-proveedor-caer-ha-sido-vercel-fuera-dias-partido

[^9]: El efecto dominó del bloqueo de Cloudflare por Movistar en España: \"Tuvimos un estudio de arquitectura dos días parado\" - Xataka, https://www.xataka.com/empresas-y-economia/efecto-domino-bloqueo-cloudflare-espana-tuvimos-estudio-arquitectura-dos-dias-parado

[^10]: Spain: The Digital Royal Decree may give Government powers to censor and takedown online content - ARTICLE 19, https://www.article19.org/resources/spain-the-digital-royal-decree-may-give-government-powers-to-censor-and-takedown-online-content/

[^11]: Caos en Internet: La Liga Ordena Bloquear Cloudflare - YouTube, https://m.youtube.com/watch?v=2GM9tSYdgrY

[^12]: LaLiga is blocking the internet in Spain. A VPN can help \| Proton VPN, https://protonvpn.com/blog/spain-laliga

[^13]: Several Spanish telecommunication companies are ordered to block Internet access to certain links websites - Kluwer Copyright Blog, https://copyrightblog.kluweriplaw.com/2018/03/14/several-spanish-telecommunication-companies-ordered-block-internet-access-certain-links-websites/

[^14]: CloudFlare IP blocking by Spanish LaLiga - DNS & Network, https://community.cloudflare.com/t/cloudflare-ip-blocking-by-spanish-laliga/789094

[^17]: Many websites using Cloudflare services are not accessible from Movistar/O2 (Spain), https://community.cloudflare.com/t/many-websites-using-cloudflare-services-are-not-accessible-from-movistar-o2-spain/767610

[^19]: Aclarando qué ha pasado entre LaLiga y Vercel (y por qué se están \..., https://www.genbeta.com/actualidad/aclarando-que-ha-pasado-laliga-vercel-que-se-estan-bloqueando-sus-webs-versiones-no-cuadran

[^21]: Some context the article misses: there\'s a court order that allows the Spanish F\... \| Hacker News, https://news.ycombinator.com/item?id=43157259

[^23]: Domains proxied are inaccessible from Movistar Spain - Cloudflare Community, https://community.cloudflare.com/t/domains-proxied-are-inaccessible-from-movistar-spain/400270

[^28]: Challenges and opportunities of Spain\'s Digital Rights Charter - - Telefónica, https://www.telefonica.com/en/communication-room/blog/challenges-and-opportunities-of-spains-digital-rights-charter/

[^29]: Women\'s rights website blocked in Spain - Digital Freedom Fund, https://digitalfreedomfund.org/womens-rights-website-blocked-in-spain/