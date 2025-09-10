EYE PIPELINE — 統合型アバターEye制作ワークフロー
Unityアバター用の高品質で表現力豊かな「目」を、半自動で生成するための統合パイプラインです。
3Dモデルのジオメトリ解析、AIによるテクスチャ生成、そして高度なカスタムシェーダ技術をシームレスに連携させることで、従来は多大な時間と専門スキルを要した制作工程を劇的な効率化と品質向上を実現します。
ワークフローダイアグラム (Workflow Diagram)
本パイプラインの全体像は以下の通りです。各ノードの詳細は後述の「アルゴリズムとワークフロー解説」セクションを参照してください。
flowchart TD
  %% ==============================
  %% EYE PIPELINE — Unified Diagram
  %% ==============================

  %% 0. 入力
  IN_FBX["入力FBX\navatar.fbx"]:::in --> ED_MENU

  %% 1. Unityエディタ処理（抽出→JSON→辞書学習→プロンプト）
  subgraph EDITOR["Unity Editor：抽出・書き出し"]
    direction TB
    ED_MENU["メニュー実行\nTools/Motirabbit/Eye JSON/Export"]:::op --> ED_CORE
    ED_CORE["MotirabbitEyeJSON.cs\n候補収集・スコア算出・左右性推定"]:::core
    ED_CORE -->|selection| J_SELE["selection\nobject_path / renderer / material / laterality / score"]:::data
    ED_CORE -->|geometry| J_GEOM["geometry\nuv[] / normals[] / tangents[] / triangles[]"]:::data
    ED_CORE --> J_PRETTY["出力: *.json（可読）"]:::file
    ED_CORE --> J_MIN["出力: *.min.json（軽量）"]:::file
    ED_CORE --> J_COMP["圧縮: *.min.json.br / .gz"]:::file
    ED_CORE --> P_TXT["出力: *_prompt.txt（AI用）"]:::file
    ED_CORE --> DIC["Motirabbit_Dictionary.json\n動的除外語 学習・保存"]:::aux
  end

  %% 2. AI生成（テクスチャ）
  subgraph AI["AI生成：瞳テクスチャ"]
    direction TB
    P_TXT --> AI_CFG["PromptBuilder 設定\nスタイル/配色/UVヒント"]:::op
    AI_CFG --> TX_PNG["生成: iris.png（1024x1024 PNG）"]:::file
  end

  %% 3. インポート＆マテリアル適用
  subgraph IMPORT["Unity取込・適用"]
    direction TB
    J_PRETTY --> IMP["Importer"]:::op
    J_MIN --> IMP
    J_COMP --> IMP
    TX_PNG --> IMP
    IMP --> MAT["マテリアル適用\n（左右Unknown時はミラー複製）"]:::op
    J_SELE --> MAT
  end

  %% 4. シェーダ統合（Built-in RP / Kaede / Geodesic）
  subgraph SHADER["カスタムEyeシェーダ（Built-in RP）"]
    direction TB
    MAT --> SH_IN["Uniform/Texture\n_IrisTex, _ScleraTex, _Normal 等"]:::data
    J_GEOM --> SH_TBN["UV＆TBN確立\n(uv, normals, tangents)"]:::core
    SH_TBN --> SH_CORNEA["角膜屈折\nGeodesic Bézier補間ベース"]:::core
    SH_CORNEA --> SH_IRIS["虹彩/リム（limbal）\n半径・縁エッジ・異方性"]:::core
    SH_IRIS --> SH_SCLERA["強膜シェーディング\n簡易SSS/AO"]:::core
    SH_SCLERA --> SH_SPEC["ハイライト/スペキュラ\n視線・光源追従"]:::core
    SH_SPEC --> SH_OUT["合成出力（Eye）"]:::out
  end

  %% 5. ランタイム制御（表情/視線→シェーダ）
  subgraph RUNTIME["ランタイム制御"]
    direction TB
    A3["Avatars 3.0 / 表情ブレンドシェイプ"]:::aux --> RT_PARAM["Shader Params\n瞳径/輝度/ハイライト偏位 など"]:::op
    EYE_TRK["Eye Tracking\n視線ベクトル"]:::aux --> RT_PARAM
    RT_PARAM --> SH_IRIS
    RT_PARAM --> SH_SPEC
  end

  %% 6. 出力先
  SH_OUT --> OUTPLAY["出力: 再生/VRChat/配信/収録"]:::out

  %% Styles
  classDef in fill:#eef6ff,stroke:#5b9cf0,color:#0b3360,stroke-width:1.1px;
  classDef out fill:#eafff3,stroke:#1fbf75,color:#083b2a,stroke-width:1.1px;
  classDef file fill:#fff7e6,stroke:#f0a000,color:#5a3b00,stroke-width:1.1px;
  classDef data fill:#f2f2ff,stroke:#7a68f8,color:#2f2b5a,stroke-width:1.1px;
  classDef op fill:#fff,stroke:#9aa4b2,color:#1f2937,stroke-width:1.1px;
  classDef core fill:#f7e8ff,stroke:#b24de6,color:#3c1156,stroke-width:1.1px;
  classDef aux fill:#f0f9ff,stroke:#38bdf8,color:#0c4a6e,stroke-width:1.1px;

アルゴリズムとワークフロー解説
1. Unity Editor：抽出・書き出し (EDITOR)
パイプラインの起点です。ユーザーが3Dモデル（.fbx）から目の情報を抽出します。
 * MotirabbitEyeJSON.cs: このコアスクリプトが、以下のインテリジェントな処理を実行します。
   * 候補収集: アバター内の全メッシュレンダラーを走査し、メッシュ名、マテリアル名、頂点数などから「目」である可能性が高いオブジェクトをリストアップします。
   * スコア算出: 事前定義されたルール（例：「Eye」「瞳」といったキーワードを含むか）に基づき、各候補にスコアを付け、最も確からしいオブジェクトを特定します。
   * 左右性推定: オブジェクトの名称（_L, _Rなど）やワールド座標から、左右の目を推定します。
   * データ書き出し:
     * selection: 特定された目のオブジェクトパス、マテリアル情報、左右の別、スコアなどを格納したデータです。
     * geometry: シェーダが直接利用するための生ジオメトリデータ（UV、法線、タンジェント、三角形インデックス）を格納したデータです。
     * *.json, *.min.json, *.min.json.br/.gz: 上記データを、可読形式、軽量形式、圧縮形式でファイルに出力します。
     * *_prompt.txt: 本パイプラインの核心の一つ。 抽出した情報（メッシュ名、マテリアル名など）を基に、AI画像生成に最適化されたプロンプトの雛形を自動生成します。
 * Motirabbit_Dictionary.json: スコア算出時に除外すべき単語（例：「Hair」「Body」）を動的に学習・保存し、次回以降の解析精度を向上させます。
2. AI生成：瞳テクスチャ (AI)
Editor処理で生成された *_prompt.txt を利用して、瞳のテクスチャを生成します。
 * ユーザーはプロンプトをコピーし、好みのスタイルや配色に関するキーワードを追記します。
 * AI画像生成サービス（Stable Diffusion, Midjourneyなど）を用いて、1024x1024のPNG形式でiris.png（虹彩テクスチャ）を生成します。
3. Unity取込・適用 (IMPORT)
生成されたアセットをUnityプロジェクトにインポートすると、自動化されたワークフローが起動します。
 * カスタムインポータ (Importer): プロジェクト内のアセット変更を監視しており、iris.pngおよび.jsonファイルが追加されると、以下の処理を実行します。
   * selectionデータ（*.json内）を読み込み、テクスチャを適用すべきマテリアルを特定します。
   * 特定したマテリアルのシェーダを、本パイプライン専用のカスタムEyeシェーダに切り替えます。
   * iris.pngをシェーダの _IrisTex スロットに自動で設定します。
   * 左右性が不明(Unknown)だった場合、片方の目を複製・ミラーリングして両目を設定するフォールバック処理を行います。
4. カスタムEyeシェーダ (SHADER)
本パイプラインの最終的な見た目を決定づける、心臓部です。
 * UV＆TBN確立: geometryデータから渡されたUV、法線、タンジェント情報を基に、オブジェクトの表面における光の計算基盤（TBN行列）を正確に構築します。
 * 角膜屈折: Geodesic Bézier補間などの手法に基づき、眼球の丸みに沿って虹彩が歪んで見える、リアルなレンズ効果をシミュレートします。
 * 虹彩/リム: 虹彩の半径、縁の表現、リム（黒目と白目の境界）の陰影、異方性ライティング（光の筋）などを細かく制御します。
 * 強膜シェーディング: 白目（強膜）部分に簡易的なSSS（Subsurface Scattering）やAO（Ambient Occlusion）を適用し、プラスチック的ではない、柔らかな質感を表現します。
 * ハイライト/スペキュラ: ランタイム制御と連携し、光源方向や視線ベクトルに応じて動的に変化する、生き生きとしたハイライトを描画します。
5. ランタイム制御 (RUNTIME)
アバターが実際に動いている最中の、動的な表現を司ります。
 * Avatars 3.0 / 表情ブレンドシェイプ: VRChatなどのプラットフォームで表情が変化した際に、そのパラメータ（例：「驚き」ブレンドシェイプの値）を受け取ります。
 * Eye Tracking: 外部の視線追跡デバイスから、ユーザーが見ている方向のベクトル情報を受け取ります。
 * Shader Params: 受け取ったこれらの情報を基に、シェーダに送るパラメータ（瞳孔の直径 ハイライトの偏位 虹彩の輝度など）をリアルタイムで計算し、シェーダに送ります。
 * これにより、驚いたときに瞳孔が小さくなったり、視線の先にハイライトが追従したりといった、極めて生命感のある表現が可能になります。
この一連のパイプラインを経て、最終的にアバターの目はVRChatや各種配信プラットフォームで、高品質かつインタラクティブな表現力を持つアセットとして機能します。
