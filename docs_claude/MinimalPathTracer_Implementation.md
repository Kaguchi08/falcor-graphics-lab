# MinimalPathTracer実装解説

本ドキュメントは、[MinimalPathTracer.cpp](../Source/RenderPasses/MinimalPathTracer/MinimalPathTracer.cpp)の実装について、コードを上から順番に解説します。

## 目次

1. [概要](#概要)
2. [プラグイン登録](#プラグイン登録)
3. [定数とチャネル定義](#定数とチャネル定義)
4. [コンストラクタ](#コンストラクタ)
5. [プロパティ管理](#プロパティ管理)
6. [リフレクション](#リフレクション)
7. [実行処理](#実行処理)
8. [UI描画](#ui描画)
9. [シーン設定](#シーン設定)
10. [変数準備](#変数準備)
11. [理論的背景](#理論的背景)

---

## 概要

MinimalPathTracerは、検証目的で設計された最小限のパストレーシング実装です。以下の特徴を持ちます：

- **目的**: より複雑なレンダラーの検証用リファレンス実装
- **手法**: ブルートフォース（総当たり）パストレーシング
- **特性**: 不偏推定量（unbiased）による一貫性のある（consistent）グラウンドトゥルース画像の生成
- **制限事項**: 透過（transmission）とネストされた誘電体（nested dielectrics）は未対応

このクラスは[RenderPass](../Source/RenderPasses/MinimalPathTracer/MinimalPathTracer.h)を継承し、Falcorのレンダーグラフシステムと統合されます。

---

## プラグイン登録

### コード (行28-35)

```cpp
#include "MinimalPathTracer.h"
#include "RenderGraph/RenderPassHelpers.h"
#include "RenderGraph/RenderPassStandardFlags.h"

extern "C" FALCOR_API_EXPORT void registerPlugin(Falcor::PluginRegistry& registry)
{
    registry.registerClass<RenderPass, MinimalPathTracer>();
}
```

### 解説

- **ヘッダーインクルード**:
  - `MinimalPathTracer.h`: クラス定義
  - `RenderPassHelpers.h`: レンダーパス用ヘルパー関数
  - `RenderPassStandardFlags.h`: 標準的なフラグ定義

- **プラグイン登録関数**:
  - `extern "C"`: C言語リンケージを指定（名前マングリングを防ぐ）
  - `FALCOR_API_EXPORT`: DLLエクスポート用マクロ
  - `registerPlugin()`: Falcorが起動時に呼び出す関数
  - `registry.registerClass<RenderPass, MinimalPathTracer>()`: MinimalPathTracerをRenderPassとして登録

この仕組みにより、MinimalPathTracerは動的にロード可能なプラグインとして機能します。

---

## 定数とチャネル定義

### コード (行37-64)

```cpp
namespace
{
const char kShaderFile[] = "RenderPasses/MinimalPathTracer/MinimalPathTracer.rt.slang";

// Ray tracing settings that affect the traversal stack size.
// These should be set as small as possible.
const uint32_t kMaxPayloadSizeBytes = 72u;
const uint32_t kMaxRecursionDepth = 2u;

const char kInputViewDir[] = "viewW";

const ChannelList kInputChannels = {
    // clang-format off
    { "vbuffer",        "gVBuffer",     "Visibility buffer in packed format" },
    { kInputViewDir,    "gViewW",       "World-space view direction (xyz float format)", true /* optional */ },
    // clang-format on
};

const ChannelList kOutputChannels = {
    // clang-format off
    { "color",          "gOutputColor", "Output color (sum of direct and indirect)", false, ResourceFormat::RGBA32Float },
    // clang-format on
};

const char kMaxBounces[] = "maxBounces";
const char kComputeDirect[] = "computeDirect";
const char kUseImportanceSampling[] = "useImportanceSampling";
} // namespace
```

### 解説

#### シェーダーファイル
- **`kShaderFile`**: レイトレーシングシェーダーのパス
  - `.rt.slang`: レイトレーシング用Slangシェーダー

#### レイトレーシング設定
- **`kMaxPayloadSizeBytes = 72u`**:
  - レイペイロードの最大サイズ（バイト単位）
  - トラバーサルスタックサイズに影響
  - 小さい値を推奨（メモリ効率のため）
  - 72バイトは[`ScatterRayData`構造体](../Source/RenderPasses/MinimalPathTracer/MinimalPathTracer.rt.slang#L103-L127)のサイズに対応

- **`kMaxRecursionDepth = 2u`**:
  - 再帰トレースの最大深度
  - 値2は以下を意味:
    - レベル0: レイ生成（raygen）
    - レベル1: スキャッターレイ（scatter ray）
    - レベル2: シャドウレイ（shadow ray）

#### 入力チャネル
- **`vbuffer` (必須)**:
  - Visibility Buffer（可視性バッファ）
  - パックされた形式でヒット情報を格納
  - プライマリレイキャストの結果

- **`viewW` (オプション)**:
  - ワールド空間のビュー方向
  - Depth of Field（被写界深度）使用時に必要
  - float3形式

#### 出力チャネル
- **`color`**:
  - 最終出力色（直接照明＋間接照明の合計）
  - RGBA32Float形式（32ビット浮動小数点×4チャネル）

#### プロパティキー
プロパティディクショナリで使用する文字列キー:
- `maxBounces`: 最大バウンス数
- `computeDirect`: 直接照明を計算するか
- `useImportanceSampling`: 重点サンプリングを使用するか

---

## コンストラクタ

### コード (行66-73)

```cpp
MinimalPathTracer::MinimalPathTracer(ref<Device> pDevice, const Properties& props) : RenderPass(pDevice)
{
    parseProperties(props);

    // Create a sample generator.
    mpSampleGenerator = SampleGenerator::create(mpDevice, SAMPLE_GENERATOR_UNIFORM);
    FALCOR_ASSERT(mpSampleGenerator);
}
```

### 解説

#### 初期化シーケンス
1. **基底クラス初期化**: `RenderPass(pDevice)` を呼び出し
2. **プロパティ解析**: `parseProperties(props)` でユーザー設定を読み込み
3. **サンプルジェネレータ作成**:
   - `SampleGenerator::create()` でサンプル生成器を作成
   - `SAMPLE_GENERATOR_UNIFORM`: 一様分布サンプラー
   - `FALCOR_ASSERT()`: 作成成功を確認

#### サンプルジェネレータの役割
- 各ピクセルに対して擬似乱数列を生成
- パストレーシングで使用するランダムサンプル（方向、位置など）を生成
- 一様分布（uniform distribution）により不偏推定を保証

---

## プロパティ管理

### parseProperties (行75-88)

```cpp
void MinimalPathTracer::parseProperties(const Properties& props)
{
    for (const auto& [key, value] : props)
    {
        if (key == kMaxBounces)
            mMaxBounces = value;
        else if (key == kComputeDirect)
            mComputeDirect = value;
        else if (key == kUseImportanceSampling)
            mUseImportanceSampling = value;
        else
            logWarning("Unknown property '{}' in MinimalPathTracer properties.", key);
    }
}
```

### 解説

#### 処理内容
- プロパティディクショナリを走査し、既知のキーを処理
- 範囲ベースforループで全プロパティをチェック
- 構造化束縛 `[key, value]` でキーと値を取得

#### サポートされるプロパティ

| プロパティ | 型 | デフォルト | 説明 |
|----------|-----|----------|------|
| `maxBounces` | uint | 3 | 間接照明の最大パス長 |
| `computeDirect` | bool | true | 直接照明を計算 |
| `useImportanceSampling` | bool | true | マテリアルの重点サンプリング使用 |

- **未知のプロパティ**: 警告をログ出力（エラーではない）

### getProperties (行90-97)

```cpp
Properties MinimalPathTracer::getProperties() const
{
    Properties props;
    props[kMaxBounces] = mMaxBounces;
    props[kComputeDirect] = mComputeDirect;
    props[kUseImportanceSampling] = mUseImportanceSampling;
    return props;
}
```

### 解説

現在の設定を`Properties`オブジェクトとして返します。
- シリアライゼーション（保存/復元）に使用
- UIやスクリプトからの設定取得に使用

---

## リフレクション

### コード (行99-108)

```cpp
RenderPassReflection MinimalPathTracer::reflect(const CompileData& compileData)
{
    RenderPassReflection reflector;

    // Define our input/output channels.
    addRenderPassInputs(reflector, kInputChannels);
    addRenderPassOutputs(reflector, kOutputChannels);

    return reflector;
}
```

### 解説

#### リフレクションの目的
レンダーグラフシステムに対して、このパスが必要とする入出力リソースを宣言します。

#### 処理内容
1. `RenderPassReflection` オブジェクト作成
2. `addRenderPassInputs()`: 入力チャネル登録
   - `vbuffer` (必須)
   - `viewW` (オプション)
3. `addRenderPassOutputs()`: 出力チャネル登録
   - `color` (RGBA32Float)

#### レンダーグラフでの役割
- グラフコンパイル時に依存関係を解析
- 必要なリソースの自動割り当て
- データフロー検証

---

## 実行処理

### コード (行110-199)

```cpp
void MinimalPathTracer::execute(RenderContext* pRenderContext, const RenderData& renderData)
{
    // Update refresh flag if options that affect the output have changed.
    auto& dict = renderData.getDictionary();
    if (mOptionsChanged)
    {
        auto flags = dict.getValue(kRenderPassRefreshFlags, RenderPassRefreshFlags::None);
        dict[Falcor::kRenderPassRefreshFlags] = flags | Falcor::RenderPassRefreshFlags::RenderOptionsChanged;
        mOptionsChanged = false;
    }

    // If we have no scene, just clear the outputs and return.
    if (!mpScene)
    {
        for (auto it : kOutputChannels)
        {
            Texture* pDst = renderData.getTexture(it.name).get();
            if (pDst)
                pRenderContext->clearTexture(pDst);
        }
        return;
    }

    if (is_set(mpScene->getUpdates(), IScene::UpdateFlags::RecompileNeeded) ||
        is_set(mpScene->getUpdates(), IScene::UpdateFlags::GeometryChanged))
    {
        FALCOR_THROW("This render pass does not support scene changes that require shader recompilation.");
    }

    // Request the light collection if emissive lights are enabled.
    if (mpScene->getRenderSettings().useEmissiveLights)
    {
        mpScene->getLightCollection(pRenderContext);
    }

    // Configure depth-of-field.
    const bool useDOF = mpScene->getCamera()->getApertureRadius() > 0.f;
    if (useDOF && renderData[kInputViewDir] == nullptr)
    {
        logWarning("Depth-of-field requires the '{}' input. Expect incorrect shading.", kInputViewDir);
    }

    // Specialize program.
    // These defines should not modify the program vars. Do not trigger program vars re-creation.
    mTracer.pProgram->addDefine("MAX_BOUNCES", std::to_string(mMaxBounces));
    mTracer.pProgram->addDefine("COMPUTE_DIRECT", mComputeDirect ? "1" : "0");
    mTracer.pProgram->addDefine("USE_IMPORTANCE_SAMPLING", mUseImportanceSampling ? "1" : "0");
    mTracer.pProgram->addDefine("USE_ANALYTIC_LIGHTS", mpScene->useAnalyticLights() ? "1" : "0");
    mTracer.pProgram->addDefine("USE_EMISSIVE_LIGHTS", mpScene->useEmissiveLights() ? "1" : "0");
    mTracer.pProgram->addDefine("USE_ENV_LIGHT", mpScene->useEnvLight() ? "1" : "0");
    mTracer.pProgram->addDefine("USE_ENV_BACKGROUND", mpScene->useEnvBackground() ? "1" : "0");

    // For optional I/O resources, set 'is_valid_<name>' defines to inform the program of which ones it can access.
    // TODO: This should be moved to a more general mechanism using Slang.
    mTracer.pProgram->addDefines(getValidResourceDefines(kInputChannels, renderData));
    mTracer.pProgram->addDefines(getValidResourceDefines(kOutputChannels, renderData));

    // Prepare program vars. This may trigger shader compilation.
    // The program should have all necessary defines set at this point.
    if (!mTracer.pVars)
        prepareVars();
    FALCOR_ASSERT(mTracer.pVars);

    // Set constants.
    auto var = mTracer.pVars->getRootVar();
    var["CB"]["gFrameCount"] = mFrameCount;
    var["CB"]["gPRNGDimension"] = dict.keyExists(kRenderPassPRNGDimension) ? dict[kRenderPassPRNGDimension] : 0u;

    // Bind I/O buffers. These needs to be done per-frame as the buffers may change anytime.
    auto bind = [&](const ChannelDesc& desc)
    {
        if (!desc.texname.empty())
        {
            var[desc.texname] = renderData.getTexture(desc.name);
        }
    };
    for (auto channel : kInputChannels)
        bind(channel);
    for (auto channel : kOutputChannels)
        bind(channel);

    // Get dimensions of ray dispatch.
    const uint2 targetDim = renderData.getDefaultTextureDims();
    FALCOR_ASSERT(targetDim.x > 0 && targetDim.y > 0);

    // Spawn the rays.
    mpScene->raytrace(pRenderContext, mTracer.pProgram.get(), mTracer.pVars, uint3(targetDim, 1));

    mFrameCount++;
}
```

### 解説

#### 1. リフレッシュフラグ更新 (行113-119)

```cpp
if (mOptionsChanged)
{
    auto flags = dict.getValue(kRenderPassRefreshFlags, RenderPassRefreshFlags::None);
    dict[Falcor::kRenderPassRefreshFlags] = flags | Falcor::RenderPassRefreshFlags::RenderOptionsChanged;
    mOptionsChanged = false;
}
```

- レンダーオプション変更時に他のパスに通知
- テンポラルアキュムレーション（時間的累積）のリセットをトリガー
- `RenderOptionsChanged` フラグを設定

#### 2. シーンチェック (行122-131)

```cpp
if (!mpScene)
{
    for (auto it : kOutputChannels)
    {
        Texture* pDst = renderData.getTexture(it.name).get();
        if (pDst)
            pRenderContext->clearTexture(pDst);
    }
    return;
}
```

- シーンが存在しない場合、出力テクスチャをクリアして終了
- 早期リターンによるエラー回避

#### 3. シーン更新検証 (行133-137)

```cpp
if (is_set(mpScene->getUpdates(), IScene::UpdateFlags::RecompileNeeded) ||
    is_set(mpScene->getUpdates(), IScene::UpdateFlags::GeometryChanged))
{
    FALCOR_THROW("This render pass does not support scene changes that require shader recompilation.");
}
```

- シェーダー再コンパイルが必要な変更を検出
- ジオメトリ変更も同様に検出
- 未対応のため例外をスロー（この実装の制限事項）

#### 4. ライトコレクション (行140-143)

```cpp
if (mpScene->getRenderSettings().useEmissiveLights)
{
    mpScene->getLightCollection(pRenderContext);
}
```

- 発光マテリアルが有効な場合、ライトコレクションを要求
- ライトコレクション: シーン内の発光サーフェスをライトとして管理

#### 5. 被写界深度設定 (行146-150)

```cpp
const bool useDOF = mpScene->getCamera()->getApertureRadius() > 0.f;
if (useDOF && renderData[kInputViewDir] == nullptr)
{
    logWarning("Depth-of-field requires the '{}' input. Expect incorrect shading.", kInputViewDir);
}
```

- 絞り半径（aperture radius）が0より大きい場合、DOF有効
- DOF使用時に`viewW`入力がない場合は警告
- 警告のみで処理は継続（ただし結果は不正確）

#### 6. プログラム特殊化 (行154-165)

```cpp
mTracer.pProgram->addDefine("MAX_BOUNCES", std::to_string(mMaxBounces));
mTracer.pProgram->addDefine("COMPUTE_DIRECT", mComputeDirect ? "1" : "0");
mTracer.pProgram->addDefine("USE_IMPORTANCE_SAMPLING", mUseImportanceSampling ? "1" : "0");
mTracer.pProgram->addDefine("USE_ANALYTIC_LIGHTS", mpScene->useAnalyticLights() ? "1" : "0");
mTracer.pProgram->addDefine("USE_EMISSIVE_LIGHTS", mpScene->useEmissiveLights() ? "1" : "0");
mTracer.pProgram->addDefine("USE_ENV_LIGHT", mpScene->useEnvLight() ? "1" : "0");
mTracer.pProgram->addDefine("USE_ENV_BACKGROUND", mpScene->useEnvBackground() ? "1" : "0");

mTracer.pProgram->addDefines(getValidResourceDefines(kInputChannels, renderData));
mTracer.pProgram->addDefines(getValidResourceDefines(kOutputChannels, renderData));
```

シェーダープリプロセッサ定義を追加:

| 定義 | 説明 |
|-----|------|
| `MAX_BOUNCES` | 最大バウンス数（数値） |
| `COMPUTE_DIRECT` | 直接照明計算フラグ（0/1） |
| `USE_IMPORTANCE_SAMPLING` | 重点サンプリングフラグ（0/1） |
| `USE_ANALYTIC_LIGHTS` | 解析的ライト使用フラグ（0/1） |
| `USE_EMISSIVE_LIGHTS` | 発光ライト使用フラグ（0/1） |
| `USE_ENV_LIGHT` | 環境マップライト使用フラグ（0/1） |
| `USE_ENV_BACKGROUND` | 環境マップ背景使用フラグ（0/1） |
| `is_valid_<name>` | オプションリソースの有効性フラグ |

これらの定義により、シェーダーコードが実行時設定に応じて最適化されます。

#### 7. 変数準備 (行169-171)

```cpp
if (!mTracer.pVars)
    prepareVars();
FALCOR_ASSERT(mTracer.pVars);
```

- 初回実行時のみ`prepareVars()`を呼び出し
- シェーダーコンパイルをトリガー可能
- コンパイル失敗時は例外をスロー

#### 8. 定数設定 (行174-176)

```cpp
auto var = mTracer.pVars->getRootVar();
var["CB"]["gFrameCount"] = mFrameCount;
var["CB"]["gPRNGDimension"] = dict.keyExists(kRenderPassPRNGDimension) ? dict[kRenderPassPRNGDimension] : 0u;
```

シェーダー定数バッファ（CB）に設定:
- **`gFrameCount`**: シーンロード後のフレーム数（乱数シード用）
- **`gPRNGDimension`**: 擬似乱数生成器（PRNG）の開始次元
  - 他のパスとPRNG次元を共有する場合に使用
  - 辞書に存在しない場合は0

#### 9. I/Oバッファバインド (行179-189)

```cpp
auto bind = [&](const ChannelDesc& desc)
{
    if (!desc.texname.empty())
    {
        var[desc.texname] = renderData.getTexture(desc.name);
    }
};
for (auto channel : kInputChannels)
    bind(channel);
for (auto channel : kOutputChannels)
    bind(channel);
```

- ラムダ式でバインド処理を定義
- 全入出力チャネルをシェーダー変数にバインド
- フレームごとにバッファが変わる可能性があるため、毎フレーム実行

#### 10. レイディスパッチ (行192-196)

```cpp
const uint2 targetDim = renderData.getDefaultTextureDims();
FALCOR_ASSERT(targetDim.x > 0 && targetDim.y > 0);

mpScene->raytrace(pRenderContext, mTracer.pProgram.get(), mTracer.pVars, uint3(targetDim, 1));
```

- 出力テクスチャの解像度を取得
- `raytrace()` でレイトレーシングを実行
  - `uint3(targetDim, 1)`: (幅, 高さ, 1) のディスパッチ次元
  - 各ピクセルに対してレイ生成シェーダーを起動

#### 11. フレームカウンタ更新 (行198)

```cpp
mFrameCount++;
```

- 次フレーム用にカウンタをインクリメント
- 乱数シード生成に使用

---

## UI描画

### コード (行201-220)

```cpp
void MinimalPathTracer::renderUI(Gui::Widgets& widget)
{
    bool dirty = false;

    dirty |= widget.var("Max bounces", mMaxBounces, 0u, 1u << 16);
    widget.tooltip("Maximum path length for indirect illumination.\n0 = direct only\n1 = one indirect bounce etc.", true);

    dirty |= widget.checkbox("Evaluate direct illumination", mComputeDirect);
    widget.tooltip("Compute direct illumination.\nIf disabled only indirect is computed (when max bounces > 0).", true);

    dirty |= widget.checkbox("Use importance sampling", mUseImportanceSampling);
    widget.tooltip("Use importance sampling for materials", true);

    // If rendering options that modify the output have changed, set flag to indicate that.
    // In execute() we will pass the flag to other passes for reset of temporal data etc.
    if (dirty)
    {
        mOptionsChanged = true;
    }
}
```

### 解説

#### UI要素

1. **Max bounces スライダー**
   ```cpp
   dirty |= widget.var("Max bounces", mMaxBounces, 0u, 1u << 16);
   ```
   - 範囲: 0〜65536 (2^16)
   - 0 = 直接照明のみ
   - 1 = 1回の間接バウンス
   - n = n回の間接バウンス

2. **Evaluate direct illumination チェックボックス**
   ```cpp
   dirty |= widget.checkbox("Evaluate direct illumination", mComputeDirect);
   ```
   - 直接照明の計算を制御
   - 無効時は間接照明のみ（maxBounces > 0の場合）

3. **Use importance sampling チェックボックス**
   ```cpp
   dirty |= widget.checkbox("Use importance sampling", mUseImportanceSampling);
   ```
   - マテリアルの重点サンプリング使用を制御

#### 変更フラグ管理

```cpp
if (dirty)
{
    mOptionsChanged = true;
}
```

- いずれかの設定が変更された場合、`mOptionsChanged`フラグを設定
- `execute()`でリフレッシュフラグとして他のパスに伝播
- テンポラルアキュムレーションのリセットをトリガー

---

## シーン設定

### コード (行222-301)

```cpp
void MinimalPathTracer::setScene(RenderContext* pRenderContext, const ref<Scene>& pScene)
{
    // Clear data for previous scene.
    // After changing scene, the raytracing program should to be recreated.
    mTracer.pProgram = nullptr;
    mTracer.pBindingTable = nullptr;
    mTracer.pVars = nullptr;
    mFrameCount = 0;

    // Set new scene.
    mpScene = pScene;

    if (mpScene)
    {
        if (pScene->hasGeometryType(Scene::GeometryType::Custom))
        {
            logWarning("MinimalPathTracer: This render pass does not support custom primitives.");
        }

        // Create ray tracing program.
        ProgramDesc desc;
        desc.addShaderModules(mpScene->getShaderModules());
        desc.addShaderLibrary(kShaderFile);
        desc.setMaxPayloadSize(kMaxPayloadSizeBytes);
        desc.setMaxAttributeSize(mpScene->getRaytracingMaxAttributeSize());
        desc.setMaxTraceRecursionDepth(kMaxRecursionDepth);

        mTracer.pBindingTable = RtBindingTable::create(2, 2, mpScene->getGeometryCount());
        auto& sbt = mTracer.pBindingTable;
        sbt->setRayGen(desc.addRayGen("rayGen"));
        sbt->setMiss(0, desc.addMiss("scatterMiss"));
        sbt->setMiss(1, desc.addMiss("shadowMiss"));

        if (mpScene->hasGeometryType(Scene::GeometryType::TriangleMesh))
        {
            sbt->setHitGroup(
                0,
                mpScene->getGeometryIDs(Scene::GeometryType::TriangleMesh),
                desc.addHitGroup("scatterTriangleMeshClosestHit", "scatterTriangleMeshAnyHit")
            );
            sbt->setHitGroup(
                1, mpScene->getGeometryIDs(Scene::GeometryType::TriangleMesh), desc.addHitGroup("", "shadowTriangleMeshAnyHit")
            );
        }

        if (mpScene->hasGeometryType(Scene::GeometryType::DisplacedTriangleMesh))
        {
            sbt->setHitGroup(
                0,
                mpScene->getGeometryIDs(Scene::GeometryType::DisplacedTriangleMesh),
                desc.addHitGroup("scatterDisplacedTriangleMeshClosestHit", "", "displacedTriangleMeshIntersection")
            );
            sbt->setHitGroup(
                1,
                mpScene->getGeometryIDs(Scene::GeometryType::DisplacedTriangleMesh),
                desc.addHitGroup("", "", "displacedTriangleMeshIntersection")
            );
        }

        if (mpScene->hasGeometryType(Scene::GeometryType::Curve))
        {
            sbt->setHitGroup(
                0, mpScene->getGeometryIDs(Scene::GeometryType::Curve), desc.addHitGroup("scatterCurveClosestHit", "", "curveIntersection")
            );
            sbt->setHitGroup(1, mpScene->getGeometryIDs(Scene::GeometryType::Curve), desc.addHitGroup("", "", "curveIntersection"));
        }

        if (mpScene->hasGeometryType(Scene::GeometryType::SDFGrid))
        {
            sbt->setHitGroup(
                0,
                mpScene->getGeometryIDs(Scene::GeometryType::SDFGrid),
                desc.addHitGroup("scatterSdfGridClosestHit", "", "sdfGridIntersection")
            );
            sbt->setHitGroup(1, mpScene->getGeometryIDs(Scene::GeometryType::SDFGrid), desc.addHitGroup("", "", "sdfGridIntersection"));
        }

        mTracer.pProgram = Program::create(mpDevice, desc, mpScene->getSceneDefines());
    }
}
```

### 解説

#### 1. 前シーンのクリア (行224-230)

```cpp
mTracer.pProgram = nullptr;
mTracer.pBindingTable = nullptr;
mTracer.pVars = nullptr;
mFrameCount = 0;
```

- シーン変更時、既存のレイトレーシングデータを破棄
- プログラム、バインディングテーブル、変数をリセット
- フレームカウンタを0に初期化

#### 2. カスタムプリミティブチェック (行236-239)

```cpp
if (pScene->hasGeometryType(Scene::GeometryType::Custom))
{
    logWarning("MinimalPathTracer: This render pass does not support custom primitives.");
}
```

- カスタムプリミティブは未対応
- 警告のみで処理継続

#### 3. プログラム記述子作成 (行242-247)

```cpp
ProgramDesc desc;
desc.addShaderModules(mpScene->getShaderModules());
desc.addShaderLibrary(kShaderFile);
desc.setMaxPayloadSize(kMaxPayloadSizeBytes);
desc.setMaxAttributeSize(mpScene->getRaytracingMaxAttributeSize());
desc.setMaxTraceRecursionDepth(kMaxRecursionDepth);
```

プログラム記述子の設定:
- **シェーダーモジュール**: シーンが必要とするモジュールを追加
- **シェーダーライブラリ**: MinimalPathTracer専用のシェーダーファイル
- **最大ペイロードサイズ**: 72バイト（`ScatterRayData`のサイズ）
- **最大属性サイズ**: シーンの交差属性サイズ
- **最大再帰深度**: 2（scatter + shadow）

#### 4. シェーダーバインディングテーブル（SBT）作成

```cpp
mTracer.pBindingTable = RtBindingTable::create(2, 2, mpScene->getGeometryCount());
```

SBTパラメータ:
- **2 ray types**: スキャッターレイ（0）、シャドウレイ（1）
- **2 miss programs**: scatterMiss、shadowMiss
- **geometry count entries**: ジオメトリ数分のヒットグループ

#### 5. レイ生成とミスシェーダー (行250-252)

```cpp
sbt->setRayGen(desc.addRayGen("rayGen"));
sbt->setMiss(0, desc.addMiss("scatterMiss"));
sbt->setMiss(1, desc.addMiss("shadowMiss"));
```

| インデックス | タイプ | シェーダー | 用途 |
|------------|-------|----------|------|
| - | RayGen | `rayGen` | レイ生成 |
| 0 | Miss | `scatterMiss` | スキャッターレイミス時 |
| 1 | Miss | `shadowMiss` | シャドウレイミス時 |

#### 6. TriangleMeshヒットグループ (行254-264)

```cpp
if (mpScene->hasGeometryType(Scene::GeometryType::TriangleMesh))
{
    sbt->setHitGroup(
        0,
        mpScene->getGeometryIDs(Scene::GeometryType::TriangleMesh),
        desc.addHitGroup("scatterTriangleMeshClosestHit", "scatterTriangleMeshAnyHit")
    );
    sbt->setHitGroup(
        1, mpScene->getGeometryIDs(Scene::GeometryType::TriangleMesh), desc.addHitGroup("", "shadowTriangleMeshAnyHit")
    );
}
```

三角形メッシュ用のヒットグループ:

| レイタイプ | Closest Hit | Any Hit | 用途 |
|----------|------------|---------|------|
| 0 (Scatter) | `scatterTriangleMeshClosestHit` | `scatterTriangleMeshAnyHit` | マテリアル評価 |
| 1 (Shadow) | なし | `shadowTriangleMeshAnyHit` | 可視性テスト |

- **Any Hit**: アルファテスト用（透明マテリアル処理）
- **Closest Hit**: 最も近い交差点での処理

#### 7. DisplacedTriangleMeshヒットグループ (行266-279)

```cpp
if (mpScene->hasGeometryType(Scene::GeometryType::DisplacedTriangleMesh))
{
    sbt->setHitGroup(
        0,
        mpScene->getGeometryIDs(Scene::GeometryType::DisplacedTriangleMesh),
        desc.addHitGroup("scatterDisplacedTriangleMeshClosestHit", "", "displacedTriangleMeshIntersection")
    );
    sbt->setHitGroup(
        1,
        mpScene->getGeometryIDs(Scene::GeometryType::DisplacedTriangleMesh),
        desc.addHitGroup("", "", "displacedTriangleMeshIntersection")
    );
}
```

ディスプレイスメントマッピング用:
- **Intersection Shader**: カスタム交差テスト（`displacedTriangleMeshIntersection`）
- ディスプレイスメントマップによる形状変形を処理

#### 8. Curveヒットグループ (行281-287)

```cpp
if (mpScene->hasGeometryType(Scene::GeometryType::Curve))
{
    sbt->setHitGroup(
        0, mpScene->getGeometryIDs(Scene::GeometryType::Curve), desc.addHitGroup("scatterCurveClosestHit", "", "curveIntersection")
    );
    sbt->setHitGroup(1, mpScene->getGeometryIDs(Scene::GeometryType::Curve), desc.addHitGroup("", "", "curveIntersection"));
}
```

カーブプリミティブ（髪、毛皮など）用:
- **Intersection Shader**: カーブとレイの交差計算（`curveIntersection`）
- 球掃引（sphere-swept）カーブとして処理

#### 9. SDFGridヒットグループ (行289-297)

```cpp
if (mpScene->hasGeometryType(Scene::GeometryType::SDFGrid))
{
    sbt->setHitGroup(
        0,
        mpScene->getGeometryIDs(Scene::GeometryType::SDFGrid),
        desc.addHitGroup("scatterSdfGridClosestHit", "", "sdfGridIntersection")
    );
    sbt->setHitGroup(1, mpScene->getGeometryIDs(Scene::GeometryType::SDFGrid), desc.addHitGroup("", "", "sdfGridIntersection"));
}
```

SDF（Signed Distance Field）グリッド用:
- **Intersection Shader**: SDFレイマーチング（`sdfGridIntersection`）
- ボリューメトリックな形状表現

#### 10. プログラム作成 (行299)

```cpp
mTracer.pProgram = Program::create(mpDevice, desc, mpScene->getSceneDefines());
```

- 記述子とシーン定義からプログラムを作成
- シーン定義: ジオメトリタイプ、マテリアルシステムなどの設定

---

## 変数準備

### コード (行303-319)

```cpp
void MinimalPathTracer::prepareVars()
{
    FALCOR_ASSERT(mpScene);
    FALCOR_ASSERT(mTracer.pProgram);

    // Configure program.
    mTracer.pProgram->addDefines(mpSampleGenerator->getDefines());
    mTracer.pProgram->setTypeConformances(mpScene->getTypeConformances());

    // Create program variables for the current program.
    // This may trigger shader compilation. If it fails, throw an exception to abort rendering.
    mTracer.pVars = RtProgramVars::create(mpDevice, mTracer.pProgram, mTracer.pBindingTable);

    // Bind utility classes into shared data.
    auto var = mTracer.pVars->getRootVar();
    mpSampleGenerator->bindShaderData(var);
}
```

### 解説

#### 1. 前提条件チェック (行305-306)

```cpp
FALCOR_ASSERT(mpScene);
FALCOR_ASSERT(mTracer.pProgram);
```

- シーンとプログラムが存在することを確認
- 開発時のバグ検出用

#### 2. プログラム設定 (行309-310)

```cpp
mTracer.pProgram->addDefines(mpSampleGenerator->getDefines());
mTracer.pProgram->setTypeConformances(mpScene->getTypeConformances());
```

- **サンプルジェネレータ定義**:
  - サンプラーの種類を指定（`SAMPLE_GENERATOR_UNIFORM`など）
  - シェーダー内で適切なサンプラーを選択

- **型適合性（Type Conformances）**:
  - Slangのインターフェース/実装マッピング
  - 例: `IMaterialInstance`の具体的な実装を指定

#### 3. プログラム変数作成 (行314)

```cpp
mTracer.pVars = RtProgramVars::create(mpDevice, mTracer.pProgram, mTracer.pBindingTable);
```

- レイトレーシングプログラム変数を作成
- **シェーダーコンパイルをトリガー可能**
  - 失敗時は例外をスロー
  - 初回実行時やシェーダー変更時にコンパイル

#### 4. ユーティリティクラスバインド (行317-318)

```cpp
auto var = mTracer.pVars->getRootVar();
mpSampleGenerator->bindShaderData(var);
```

- サンプルジェネレータをシェーダーデータにバインド
- シェーダー側で`SampleGenerator`として利用可能

---

## 理論的背景

### パストレーシングの基礎

#### レンダリング方程式

パストレーシングは以下のレンダリング方程式を解きます:

$$
L_o(p, \omega_o) = L_e(p, \omega_o) + \int_{\Omega} f_r(p, \omega_i, \omega_o) L_i(p, \omega_i) (\omega_i \cdot n) d\omega_i
$$

記号の意味:
- $L_o(p, \omega_o)$: 点$p$から方向$\omega_o$への出射輝度
- $L_e(p, \omega_o)$: 点$p$での放射輝度（発光）
- $f_r(p, \omega_i, \omega_o)$: 双方向反射率分布関数（BRDF）
- $L_i(p, \omega_i)$: 方向$\omega_i$からの入射輝度
- $\Omega$: 半球全体
- $n$: 表面法線

#### モンテカルロ積分

半球上の積分をモンテカルロ法で近似:

$$
L_o(p, \omega_o) \approx L_e(p, \omega_o) + \frac{1}{N} \sum_{i=1}^{N} \frac{f_r(p, \omega_i, \omega_o) L_i(p, \omega_i) (\omega_i \cdot n)}{p(\omega_i)}
$$

- $N$: サンプル数
- $p(\omega_i)$: サンプリング確率密度関数（PDF）

### MinimalPathTracerのアルゴリズム

#### 1. 一様サンプリング（Uniform Sampling）

`useImportanceSampling = false`の場合:
- 半球上で一様にランダム方向をサンプル
- PDF: $p(\omega) = \frac{1}{2\pi}$
- 単純だが収束が遅い

#### 2. 重点サンプリング（Importance Sampling）

`useImportanceSampling = true`の場合:
- BRDFに基づいてサンプリング方向を選択
- 例: 鏡面反射では反射方向周辺を優先
- 分散を減らし、収束を高速化

#### 3. 直接照明（Direct Illumination）

```cpp
// コード内の実装: evalDirectAnalytic()
float3 evalDirectAnalytic(const ShadingData sd, const IMaterialInstance mi, inout SampleGenerator sg)
{
    const uint lightCount = gScene.getLightCount();
    if (lightCount == 0) return float3(0.f);

    // ライトを一様にランダム選択
    const uint lightIndex = min(uint(sampleNext1D(sg) * lightCount), lightCount - 1);
    float invPdf = lightCount;

    // ライトをサンプル
    AnalyticLightSample ls;
    if (!sampleLight(sd.posW, gScene.getLight(lightIndex), sg, ls))
        return float3(0.f);

    // シャドウレイでテスト
    bool V = traceShadowRay(origin, ls.dir, ls.distance);
    if (!V) return float3(0.f);

    // 寄与を評価
    return mi.eval(sd, ls.dir, sg) * ls.Li * invPdf;
}
```

数式:
$$
L_{direct} = \frac{1}{N_{lights}} \sum_{i} \frac{f_r(p, \omega_i, \omega_o) L_i V(\omega_i) (\omega_i \cdot n)}{p_{light}(\omega_i)}
$$

- $N_{lights}$: ライト数
- $V(\omega_i)$: 可視性関数（0または1）

#### 4. 間接照明（Indirect Illumination）

再帰的なパストレーシング:

```cpp
// 疑似コード
for (uint depth = 0; depth <= kMaxBounces && !rayData.terminated; depth++)
{
    traceScatterRay(rayData);  // 次の交差点へ
    // handleHit()内で:
    // - 発光を累積
    // - 直接照明を計算
    // - 次のレイ方向をサンプル
}
```

パススループット（経路スループット）:
$$
\text{thp}_{i+1} = \text{thp}_i \cdot \frac{f_r(p_i, \omega_i, \omega_o) (\omega_i \cdot n)}{p(\omega_i)}
$$

各バウンスで累積:
$$
L_{indirect} = \sum_{i=1}^{k} \text{thp}_i \cdot L_e(p_i)
$$

### 不偏推定（Unbiased Estimation）

MinimalPathTracerは不偏推定量を生成:

$$
\mathbb{E}[\hat{L}] = L
$$

- $\hat{L}$: 推定値
- $L$: 真の値
- 期待値が真の値と一致

これにより、十分なサンプル数で正しい結果に収束します（一貫性: consistency）。

### ライトタイプ

#### 解析的ライト（Analytic Lights）
- ポイントライト、ディレクショナルライト
- 明示的サンプリング（シャドウレイ）
- PDF: ライト選択確率 $\frac{1}{N_{lights}}$

#### 発光マテリアル（Emissive Lights）
- メッシュサーフェスからの発光
- スキャッターレイのヒット時に寄与
- 暗黙的サンプリング（Next Event Estimation未使用）

#### 環境マップ（Environment Map）
- 無限遠の照明
- スキャッターレイのミス時に寄与
- 背景としても使用可能

### レイタイプ

#### スキャッターレイ（Scatter Ray）
- パストレーシングのメインレイ
- サーフェスでBRDFサンプリング
- 再帰的にトレース

#### シャドウレイ（Shadow Ray）
- ライトへの可視性テスト
- `RAY_FLAG_ACCEPT_FIRST_HIT_AND_END_SEARCH`最適化
- Any Hitシェーダーで早期終了

### ペイロード構造

#### ScatterRayData（72バイト）
```cpp
struct ScatterRayData
{
    float3 radiance;     // 12バイト: 累積輝度
    bool terminated;     // 4バイト: 終了フラグ
    float3 thp;          // 12バイト: スループット
    uint pathLength;     // 4バイト: パス長
    float3 origin;       // 12バイト: レイ原点
    float3 direction;    // 12バイト: レイ方向
    SampleGenerator sg;  // 16バイト: 乱数生成器状態
};
```

合計: 72バイト（`kMaxPayloadSizeBytes`と一致）

#### ShadowRayData
```cpp
struct ShadowRayData
{
    bool visible;  // 可視性フラグのみ
};
```

最小限のペイロード（効率的）

### ジオメトリタイプ対応

| タイプ | Intersection | Any Hit | Closest Hit |
|-------|-------------|---------|------------|
| TriangleMesh | 組み込み | アルファテスト | マテリアル評価 |
| DisplacedTriangleMesh | カスタム | - | マテリアル評価 |
| Curve | カスタム | - | マテリアル評価 |
| SDFGrid | カスタム | - | マテリアル評価 |

各タイプに対して、スキャッターとシャドウの2つのヒットグループを設定。

---

## まとめ

MinimalPathTracerは以下の特徴を持つシンプルなパストレーサーです:

### 主要な設計方針
1. **シンプルさ優先**: 複雑な最適化を排除
2. **検証用途**: 他のレンダラーの正確性検証
3. **不偏推定**: 一貫性のある結果保証
4. **モジュラー設計**: Falcorレンダーグラフと統合

### アルゴリズムの流れ
1. V-bufferからプライマリヒットを読み込み
2. 直接照明を解析的ライトから計算（オプション）
3. BRDFサンプリングでスキャッターレイを生成
4. 再帰的にパスをトレース（最大深度まで）
5. 発光と環境マップを累積
6. 最終色を出力

### 性能特性
- **収束速度**: 遅い（ブルートフォース）
- **メモリ使用量**: 少ない（ペイロード72バイト）
- **品質**: 高い（不偏推定）

### 制限事項
- 透過マテリアル未対応
- ネストされた誘電体未対応
- シーン動的変更未対応
- Next Event Estimation未実装（発光マテリアル用）

このドキュメントは、コード実装と理論的背景の両面からMinimalPathTracerを解説しました。
