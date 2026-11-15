# Longthinx gobbiefix 修正内容のご説明（コード付き）

*English version: [README.md](README.md)*

## 株式会社スクウェア・エニックス Creative Studio 3 の皆様へ

拝啓

突然のご連絡となりますこと、何卒ご容赦くださいませ。私どもは、FINAL FANTASY XI の Windows 版クライアント向けに、
非公式の改善用ソフトウェア「Longthinx gobbiefix」を制作しております有志でございます。長年にわたりヴァナ・ディールの
世界をお届けくださっている皆様に、まずは心より御礼申し上げます。

本書は、gobbiefix に含まれる各修正が PC 版クライアントの何をどのように変更しているのか、またその理由について、
実際のコードを添えてご説明するものでございます。

もし本書に記載いたしました Dellingr のコードや着想の中に、ゲーム本体へお取り入れいただけるものがございましたら、
どうぞご自由にお使いくださいませ。その際は、**Dellingr および Geno** の名をクレジットに加えていただけましたら
幸甚に存じます。法務上の理由などにより私どもの名を直接お載せいただくことが難しい場合には、ささやかな遊び心として
**「獣人同盟（Beastmen Alliance）」** への謝辞をお添えいただければ、それで十分でございます。

なお、本リポジトリのライセンス（[LICENSE](LICENSE)）は、スクウェア・エニックス様ならびにグループ各社様、委託先の
皆様に対し、MIT No Attribution と同等の条件をお認めするものでございます。本コードは条件なく使用・改変・配布して
いただけます。上記のクレジットはあくまで私どもからのお願いであり、義務ではございません（それ以外の方は
GPL-3.0-only の条件でのみご利用いただけます。LICENSE には日本語の参考訳を併記しております）。本ライセンスの
対象は Dellingr 自身の成果物に限られます。他の制作者による MOD に由来する着想については、該当箇所および
[クレジット](#クレジット) にその旨を明記しております。それらの方々のコードは本リポジトリには含まれておりません。

ここに掲載いたしますコードは、いずれも皆様のコードではございません。gobbiefix は実行中のゲームに外部から手を加える
方式のため、各修正はクライアントの関数を囲む「フック」として書かれております。実際のソースコードをお持ちの皆様で
あれば、その多くは該当関数の中の数行の変更で実現できるものと拝察いたします。

至らぬ点も多々あろうかと存じますが、ご参考の一端となりましたら幸いでございます。

敬具

### 私へのご連絡方法

株式会社スクウェア・エニックスの方で、本書の内容のご利用について、あるいはすべてのアドレスとオフセットを含む完全版の
ご提供についてご連絡をいただける場合は、本リポジトリの非公開報告機能をご利用くださいませ。**Security** タブを開き、
**Report a vulnerability** をお選びください（[直接のリンク](https://github.com/WGINC/Longthinx-Gobbiefix-Source-Files/security/advisories/new)）。お送りいただいた内容は、本リポジトリの管理者
のみが拝読いたします。フォームの名称は「脆弱性の報告」となっておりますが、脆弱性に関するご連絡である必要はございません。
件名に「Contact」とご記入のうえ、ご自由にお書きくださいませ。

完全版をご希望の場合は、ご本人であることを確認させていただくため、メッセージに貴社のメールアドレスをお書き添え
くださいますようお願い申し上げます。確認のご連絡を差し上げた後、お使いの GitHub アカウントに非公開リポジトリの
閲覧権限をお付けいたします。

## 本書の読み方

- **本書の基となったビルド。** 本書の内容はすべて、PE タイムスタンプ `6A995428`（2026 年 9 月 3 日）の FFXiMain.dll
  （アンパック後）に基づいております。sha256 は `1b9b59aedc18a5b7dcbd531fff598c3b43a359eeb5dc2798763bc0ff8a21a3ad`、サイズは 6,208,512 バイトでございます。アドレスは既定の
  ベースアドレス `0x10000000` を前提としております。このハッシュはアンパック後の DLL のものであり、インストールされて
  いる圧縮された状態のファイルとは一致いたしません。特に記載のない限り、アドレス・オフセット・定数はすべて当該ビルドの
  コードから読み取ったものでございます。
- **最新のビルド**は 11 月のもの、PE タイムスタンプ `6AE818B4`（2026 年 11 月 2 日）でございます。gobbiefix は、解析した
  ビルドからの変換によりこのビルドで動作しております。キットが一覧に掲げる 4,826 の関数のうち 4,746 がこのビルドでも同一
  であることを確認しており、speedfix が用いるアドレスはすべて確定し、変更された関数の外にございます。同ビルドの
  インストールされているファイル（圧縮された状態）の sha256 は `d809ebe333019395ad43ab7ec1d94d9f58fe1f62c2ae730334f730fa8e7d48fb` でございます。
- **関数名**（`Shadow_DrawByMode`、`CXiActor::NameCalc` など）は、私どもが便宜上付けた名称でございます。皆様の
  ソースにおける名称とは異なりますので、アドレスにて関数をご特定くださいませ。
- **フックの書き方について。** コード中の `prev_x` は、フックから呼び出すゲーム本来の関数でございます。
  `orig::X(...)` はゲームの関数を直接呼び出すものでございます。`XI_GAME(a)` は、上記ビルドでのアドレスを
  実行中のプロセスにおけるアドレスへ変換するものでございます。
- **掲載コードの範囲。** 掲載コードはすべて gobbiefix 独自のものでございます。修正が皆様の関数の書き直し版の中に
  ある場合は、私どもが追加した行のみを掲載し、関数のその他の部分は一切掲載しておりません。省略箇所は `...` または
  「（抜粋）」と示しております。
- **プレースホルダについて。** 本公開版では、ゲームのデータアドレス、構造体オフセット、個々の命令の位置およびパッチ
  内容を省略しております。`g_shadowSize`、`ACTOR_BOX_CORNERS(actor)`、`RET_IN_RESTART` などの名称がそれらの代わりで
  ございます。関数のアドレスは、gobbiefix の公開版にすでに記載されているもののみを掲載しております。すべての値を
  含む完全版は、スクウェア・エニックス様からのご要望に応じてご提供いたします。
- **判明していることと、判明していないこと。** 各項目では、数値の出所を明記しております。判明していない事柄
  （多くは PS2 版での正確な挙動）については、その旨を記しております。

## 修正の一覧

| # | 修正 | 修正がない場合の PC 版 | 修正を入れるべき箇所 | 設定（既定値） | ゲーム内での確認 |
|---|---|---|---|---|---|
| 1 | 地面のデカール | 魔法陣が引き伸ばされ、大きさが視野角に追従 | `BeginEffectOffscreenPass` | `decal_fix`（有効） | 検証担当者が確認済み |
| 2 | 丸影 | 全キャラクターの影が同じ大きさ | `Shadow_DrawByMode` | `fitted_shadows`（有効） | 動作を確認。倍率は未測定 |
| 3 | 名前の大きさ | 移動に伴い名前の大きさが脈打つ | `CXiActor::NameCalc` | `smooth_names`（有効） | 動作を確認 |
| 4 | 街灯の光 | 光がランプの奥でちらつく | 光の描画の呼び出し元 | `flare_fix`（有効） | 検証担当者が確認済み |
| 5 | 球面マップの金属 | PS2 版よりくすむ | `CYyModelDt::SkinAndDrawV2` | `sphere_shine`（有効） | ゲーム内では未検証 |
| 6 | オーラのキャプチャサイズ | オーラがギザギザ | `Post_CreateScreenEffectTargets`、`Post_CaptureSceneToTarget` | `aura_size`（1024） | 2048 で確認済み |
| 7 | オーラの広がりと最後の層 | 大画面で層が詰まり、縁が硬い | `Post_DrawRotatingGhostLayers` | `aura_soft`（有効） | 確認済みのオーラ検証で使用。単独では未検証 |
| 8 | オーラの強化（新機能） | — | `Post_DrawRotatingGhostLayers` | `aura_boost`（0、無効） | 動作を確認 |
| 9 | オーラの残像（新機能） | — | `Post_DrawRotatingGhostLayers` | `aura_trail`（0、無効）および `aura_trail_*` | 動作を確認。最新の変更は未実行 |
| 10 | 崩壊エフェクト | PS2 版の分解表現がない | —（portfix の手法） | `disintegrate`（1） | 両版とも検証担当者が確認済み |
| 11 | ループ音 | 雨や風の音が完全に止まる | `CYySoundElem::UpdateTimerAndPosition` | `sound_retry`（有効） | 未確認：測定した 1 回では一度も作動せず |
| 12 | 60 fps のリミッター | 58.8 fps | `Frame_Render` のリミッター | `precise_pacing`（speedfix.ini、有効） | 測定済み：作動。フレーム間のばらつき 15.2% → 11.1% |
| 13 | ゲーム内時間とフレームレート | 60 fps 未満で遅く、超過で速い | `Frame_Render`、`Camera_GetFrameScale` | `frame_step`（speedfix.ini、3） | 測定済み：31〜184 fps でゲーム内時間が実時間に追従 |
| 14 | 均等なフレームレート上限 | — | Present の時点 | `frame_cap`（0、無効） | 120 と上限なしとで画面上の差は見られず |
| 15 | すべての CPU で SSE | ベンダー文字列が一致しない CPU で x87 の経路 | CPU 判定、`CreateProcessorInstance` | `cpu_tier`、`sse_skinning`（speedfix.ini、有効） | ログで確認：有効時は SSE、無効時は x87 |
| 16 | オクルージョンの読み戻し | CPU が GPU を待つ | オクルージョンの読み戻し | `occlusion_late`（speedfix.ini、有効） | 使用中。既知の副作用であるポップインを確認。負荷はゲーム内で未測定 |
| 17 | 描画エンジンのテクスチャテーブル | — | d3d8on11 | — | 測定済み：1 描画あたり 0.72〜0.88 µs → 0.34〜0.36 µs |

- **設定：** `gobbiefix.ini` における名称でございます（`speedfix.ini` と記したものを除きます）。設定を無効にすると
  ゲーム本来の挙動となりますので、ゲーム内で修正の前後をお比べいただけます。
- **ゲーム内での確認：** 実際のゲームのセッションで確認・測定された内容でございます。「検証担当者が確認済み」は、
  gobbiefix の検証担当者が修正の動作を確認したことを意味いたします。測定値は、それらのセッションにおける gobbiefix
  自身のログによるものでございます。
- `frame_step` の既定値である**モード 3** は、第 13 項でご説明したモード 1・2 に加え、1 ティックごとに 1 回実行される
  5 つのループ（カメラの追従、キャラクターの位置と向き）にも端数を与えるものでございます。

---

## 描画の修正

### 1. 地面のデカールを正円かつ本来の大きさで描画（`decal_fix`）

**症状。** 召喚の魔法陣や地面指定の魔法陣が楕円に引き伸ばされ、視野角によって大きさも変わってしまいます。

**原因。**
- `BeginEffectOffscreenPass` は、256×256 の蓄積用ターゲットをバインドし、VIEW（エフェクトから
  2 ユニット離れた位置からの `D3DXMatrixLookAtRH`）と WORLD（エフェクトの行列に 0.25 倍のスケール）を設定いたします。
- ところが **PROJECTION は設定されておりません**。そのため、デカールは正方形のターゲットへシーンの透視投影のまま
  描かれ、x と y のスケールが画面の縦横比の分だけ異なり、さらに両方がシーンの視野角に追従いたします。
- その後、`EndEffectOffscreenPass`（0x10054F80）が `DrawProjectedTexture`（0x10182700）により、サイズ 10 で
  ターゲットを地面へ貼り付けます（`u = (x − px) / 10 + 0.5`）。

**計算。** エフェクト中心から距離 `d` の点は NDC `s · 0.25 · d / 2` に描かれます（`s` は投影行列の `_11` または
`_22`）。これは地面上では `5 · NDC = (5s/8) · d` の位置となり、**s = 8/5 = 1.6** のとき、両軸ともちょうど `d`
（等倍）となります。あるプレイヤーの画面では `_11 = 1.7032`、`_22 = 2.5729` であり、横 1.06 d、縦 1.61 d の楕円と
なっておりました。

**修正。** オフスクリーンパスでは `_11 = _22 = ±1.6` の投影を用い、`EndEffectOffscreenPass` の冒頭でシーンの投影を
元に戻します。ターゲットのバインド後に行われる呼び出しであるため、`CMoDX::PushViewMatrix` をフックし、Begin からの
呼び出し（戻りアドレス `RET_FROM_BEGIN_PUSHVIEW` で判別）の場合にのみ処理いたします。

```cpp
static void __thiscall sf_pushview(void *self) {
  prev_pushview(self);
  if ((uintptr_t)__builtin_return_address(0) != RET_FROM_BEGIN_PUSHVIEW || !sf_cfg.decal_fix || sf_decal_set) return;
  void *cdx = CDX_OF(g_pResourceMng);
  float p[16];
  orig::CDx_GetTransformCached(cdx, 3, sf_decal_saved);           // 3 = D3DTS_PROJECTION
  for (int i = 0; i < 16; i++) p[i] = sf_decal_saved[i];
  p[0] = p[0] < 0 ? -1.6f : 1.6f; p[5] = p[5] < 0 ? -1.6f : 1.6f; // _11, _22; depth terms kept
  orig::CDx_SetTransform(cdx, 3, p);
  sf_decal_set = 1;
}
static float * __stdcall sf_effend(float *a1, int a2) {           // EndEffectOffscreenPass
  if (sf_decal_set) { orig::CDx_SetTransform(CDX_OF(g_pResourceMng), 3, sf_decal_saved); sf_decal_set = 0; }
  return prev_effend(a1, a2);
}
```

*ソースにおいては：* `BeginEffectOffscreenPass` 内で、VIEW・WORLD と並べてこの投影を設定いただくのが自然かと存じます。

**判明していないこと：** PS2 版がこれらをモデル本来の大きさちょうどで描いていたかどうかは分かっておりません。PC 版の
定数がそれを示すのは、パイプラインが等倍を意図していた場合に限られます。Dellingr の以前の修正（dagnyfix、
Daleterrence 氏の試作に基づくもの）は目視で 1.529 を見いだしており、1.6 との差は 4% でございます。

### 2. 丸影をキャラクターごとの大きさに（`shadow_fit`）

**症状。** 簡易の丸影が全キャラクター同じ大きさで、ガルカもタルタルも同じ影となっております。

**原因。**
- `Shadow_DrawByMode`（0x10033DA0）は、地面の四角形（x・z 方向に ±1）を `g_shadowSize / 3` で
  拡大いたします。
- この値は全キャラクターで共有される単一のグローバル変数でございます。初期値 1.0 の後は、0x100CBDE0 が*いずれか一体*の
  キャラクターの範囲から最後に書き込んだ値となっております。

**修正。** キャラクターごとの呼び出しの間だけこの値を設定し、呼び出し後に元へ戻します。キャラクターの大きさは、
キャラクターの OnMove が毎ティック書き込む、バウンディングボックスのワールド座標上の 8 頂点（`ACTOR_BOX_CORNERS`）から求めます。
- 足元の対角線（頂点間の x-z 距離の最大値）は、キャラクターが向きを変えても変わりません。
- 投影影（分岐 2）には 2 倍の値を与えます。`DrawProjectedShadow`（0x10184FE0）はテクスチャを `s` の幅に、地面の
  四角形は `2s` の幅に広げるためでございます。この係数がない場合、影を落とす者が 20 ユニット以内に入ると影の幅が
  半分になってしまいます。

```cpp
static undefined4 __thiscall sf_shadow(void *self_, float *pos) {
  ...
    if (br <= 2) {                                        // the round-shadow branches
      const float *c = ACTOR_BOX_CORNERS(self_);                                    // the box's 8 corners
      float d2 = 0;
      for (int i = 0; i < 8; i++) for (int j = i + 1; j < 8; j++) {
        float dx = c[4 * i] - c[4 * j], dz = c[4 * i + 2] - c[4 * j + 2], e = dx * dx + dz * dz;
        if (e > d2) d2 = e;
      }
      if (d2 >= 0.05f * 0.05f && d2 <= 20.0f * 20.0f) {   // a box is there
        float d; __asm__("fsqrt" : "=t"(d) : "0"(d2));
        volatile float *g = &g_shadowSize; float keep = *g;
        *g = (br == 2 ? 2.0f : 1.0f) * 3.0f * d * (float)sf_cfg.shadow_fit_scale / 100.0f;
        undefined4 r = prev_shadow(self_, pos);
        *g = keep;
        return r;
      }
    }
  ...
  return prev_shadow(self_, pos);
}
```

**判明していないこと：** 適切な倍率は分かっておりません。`shadow_fit_scale = 46`（足元の対角線の 46% を影の半幅と
するもの）は暫定値であり、測定に基づくものではございません。0.05〜20 の範囲も、測定ではなく私どもが定めたものでございます。

### 3. 名前表示の距離による縮小をなめらかに（`name_smooth`）

**症状。** 移動に伴い、名前表示の大きさが脈打つように変化いたします（改変のないゲームでも生じます）。

**原因。**
- `CXiActor::NameCalc`（0x100830C0）は、`z · 4096` を `_ftol` で整数に切り捨てた後の深度から名前の大きさを決めて
  おります。
- 近・遠の限界値（値を読み取ったビルドではそれぞれ 4020 と 4100）の間では、大きさは
  `(4100 − depth) / 80` となります。
- そのため、名前の大きさは目に見える 80 段階で変化いたします。

**修正。** ゲーム本来の関数の実行後、同じ投影をゲーム自身の関数で同じ順序に再計算いたします。その切り捨て値がゲームの
格納した整数と一致する場合（同じ数値であることの確認）に限り、大きさを丸める前の値に置き換えます。

```cpp
static uint32_t __thiscall sf_namecalc(void *self, undefined4 *pos, int out, float *scale, uint32_t *color) {
  uint32_t r = prev_namecalc(self, pos, out, scale, color);
  ...
  long double zf = (long double)v[2] * 4096.0L;                     // the same depth, before truncation
  if ((int32_t)zf != o[2]) return r;                                // not the game's number: keep its size
  long double sz = ((long double)farz - zf) / (long double)(int32_t)(farz - nearz);
  *scale = (float)(sz < 0 ? 0 : sz > 1 ? 1 : sz);
  return r;
}
```

*ソースにおいては：* 整数化する前の深度から大きさを算出いただければ解消するものと存じます。

### 4. 街灯の光をランプの手前に（`flare_fix`）

**症状。** 街灯やランタンの光が、ランプ本体の面の奥でちらつきます。

**原因と修正。** Dellingr の dagnyfix（flarefix v3.1）に由来する修正でございます。dagnyfix は、光の描画を次の
特徴で特定いたしました。
- 36 バイトストライドの頂点バッファからの `DrawPrimitive`（頂点数 8 以下）
- 深度テスト有効、深度書き込み無効
- ブレンド `SRCALPHA / ONE`

dagnyfix は各頂点のローカル z を −0.4 移動いたしますが、gobbiefix は同じ移動をその描画に限って WORLD で行います
（ロックもコピーも不要）。`WORLD' = T(0, 0, −0.4) × WORLD`。

```cpp
static bool flare_match(D3DPRIMITIVETYPE t, UINT prims) {
  UINT nv = t == D3DPT_TRIANGLELIST ? prims * 3 : t == D3DPT_TRIANGLESTRIP || t == D3DPT_TRIANGLEFAN ? prims + 2 : 0;
  if (!nv || nv > 8 || !g_ssV[0] || g_ssStride[0] != 36 || fvf_rhw(g_curVS)) return false;
  static const D3DRENDERSTATETYPE k[5] = {D3DRS_ZENABLE, D3DRS_ZWRITEENABLE, D3DRS_ALPHABLENDENABLE, D3DRS_SRCBLEND, D3DRS_DESTBLEND};
  static const DWORD want[5] = {D3DZB_TRUE, FALSE, TRUE, D3DBLEND_SRCALPHA, D3DBLEND_ONE};
  for (int i = 0; i < 5; i++) if (g_rs[k[i]] != want[i]) return false;   // (abridged: "state known" checks removed)
  return true;
}
```

*ソースにおいては：* このオフセットは光そのものの描画箇所に入れるのが本来の形でございます（gobbiefix は呼び出し元を
集計しておりますが、ソース上での修正には至っておりません）。

### 5. 球面マップの金属を PS2 版の輝きに（`sphere_shine`）

**症状。** 球面マッピングされた金属が、PS2 版に比べて PC 版ではくすんで見えます。

**原因。**
- `CYyModelDt::SkinAndDrawV2` の球面マップ描画は、環境テクスチャをステージ 1 に置き、`TCI_CAMERASPACENORMAL` にて
  `MODULATEALPHA_ADDCOLOR` を行っております。マップを `(nx, ny)` で参照し、0xFF = 1.0 の尺度で加算いたします。
- voetre 氏の portfix が PCSX2 上で PS2 版を測定したところでは、`u = 0.5 + 0.5·nx`、`v = 0.5 − 0.5·ny` で、
  0x80 = 1.0 の尺度（すなわち 2 倍の強さ）で加算されております。

**修正。** portfix の手法を、gobbiefix 独自の形でゲーム自身の描画設定の中に実装したものでございます。
- PS2 版のマッピングを表すテクスチャ行列を設定
- ステージ 1 で `2 × マップ × アルファ` を TEMP へ出力
- ステージ 2 で TEMP を色に加算

変更するステートは事前に読み取り、描画後に元へ戻します。

```cpp
static const struct { DWORD st; D3DTEXTURESTAGESTATETYPE ty; DWORD val; } kSmSet[10] = {
  {1, D3DTSS_COLOROP, D3DTOP_MODULATE2X}, {1, D3DTSS_COLORARG1, D3DTA_CURRENT | D3DTA_ALPHAREPLICATE}, {1, D3DTSS_RESULTARG, D3DTA_TEMP},
  {2, D3DTSS_COLOROP, D3DTOP_ADD}, {2, D3DTSS_COLORARG1, D3DTA_CURRENT}, {2, D3DTSS_COLORARG2, D3DTA_TEMP},
  {2, D3DTSS_ALPHAOP, D3DTOP_SELECTARG1}, {2, D3DTSS_ALPHAARG1, D3DTA_TEMP}, {3, D3DTSS_COLOROP, D3DTOP_DISABLE}, {3, D3DTSS_ALPHAOP, D3DTOP_DISABLE}};
// u = 0.5 + 0.5 nx, v = 0.5 - 0.5 ny (the PS2's mapping, as portfix measured it)
static const D3DMATRIX kSmPs2 = {{{0.5f, 0, 0, 0, 0, -0.5f, 0, 0, 0, 0, 0, 0, 0.5f, 0.5f, 0, 1}}};
```

### 6. オーラをなめらかに：キャプチャの拡大（`aura_size`）

**症状。** キャラクターのオーラがギザギザに見えます。

**原因。**
- `Post_CaptureSceneToTarget`（0x10038A80）は、シーンを 256×256 の四角形として `g_sceneCaptureTex` にコピーいたします。
- `Post_DrawRotatingGhostLayers`（0x10038E10）はそれをビューポートの大きさで描き戻すため、オーラは 256 ピクセルの画像を
  引き伸ばしたものとなっております。
- コード上、256 に依存している箇所は他にございません。ターゲットは深度サーフェスなしでバインドされ、ビューポートは
  ターゲット自身の大きさから設定されます（`CDx::PushRenderTargetTexture`、0x10009F20）。

**修正。** `Post_CreateScreenEffectTargets` でテクスチャを大きく作成し、キャプチャの四角形をその大きさで描きます。
同関数内に私どもが追加した行は以下のとおりでございます。

```cpp
unsigned short n = xi_aura_size >= 0x100u && xi_aura_size <= 0x1000u ? (unsigned short)xi_aura_size : 0x100u;
g_sceneCaptureTex = (int)orig::CDx_CreateTexture(g_pCDx, n, n, &g_sceneCaptureDepth, 1, 0);
if ( g_sceneCaptureTex == 0 && n != 0x100u )        // not made at that size: the game's
  g_sceneCaptureTex = (int)orig::CDx_CreateTexture(g_pCDx, 0x100u, 0x100u, &g_sceneCaptureDepth, 1, 0);
```

gobbiefix の既定値は 1024 でございます。立体視用のテクスチャ（`g_sceneCaptureTexEye2`）は 256 のままといたしました。
このテクスチャを拡大するという着想は、Dellingr の dagnyfix が移植した Krauerlabs 氏の SpectralFix に由来いたします。
gobbiefix は、外部から描画を照合するのではなく、作成元でこの変更を行っております。

### 7. オーラをなめらかに：広がりと最後の層の半透明化（`aura_soft`）

**原因。** `Post_DrawRotatingGhostLayers` には次の 2 点がございます。
- 回転した各層は `sin/cos(角度) × 半径 × 0.0625` **ピクセル**ずつずらされます。この値はどの画面でも同じため、
  大きな画面では層が詰まって見えます。
- 最後のずらさない層は、アルファテスト（`ALPHAREF 0x60`、ブレンド無効）で不透明に描かれるため、縁が硬くなります。

**修正。** SpectralFix（Krauerlabs 氏）の 2 つの描画変更を、ゲーム自身の関数の中で行うものでございます。
- ずれ量を `レンダーターゲットの幅 / 1024` 倍（1〜16 の範囲）にいたします。
- 最後の層を 25% のブレンドで描きます（ブレンド有効、アルファテスト無効、アルファ = テクスチャ × ディフューズ、
  ディフューズのアルファは 64）。

私どもが追加した行は以下のとおりでございます。

```cpp
if ( xi_aura_soft != 0 )                    // the spread
{
  float sp = (float)RT_WIDTH(orig::CDx_GetRenderTargetTop_v1(g_pCDx_as_v1)) / 1024.0f;
  sp = sp < 1.0f ? 1.0f : sp > 16.0f ? 16.0f : sp;
  fStack_a4 *= sp; fVar11 *= sp;            // the copy's x and y offsets
}
...
if ( xs_comp )                              // the last copy at 25% (states read first, put back after the draw)
{
  xs_d->lpVtbl->SetRenderState(xs_d, D3DRS_ALPHABLENDENABLE, TRUE); xs_d->lpVtbl->SetRenderState(xs_d, D3DRS_SRCBLEND, D3DBLEND_SRCALPHA);
  xs_d->lpVtbl->SetRenderState(xs_d, D3DRS_DESTBLEND, D3DBLEND_INVSRCALPHA); xs_d->lpVtbl->SetRenderState(xs_d, D3DRS_BLENDOP, D3DBLENDOP_ADD);
  xs_d->lpVtbl->SetRenderState(xs_d, D3DRS_ALPHATESTENABLE, FALSE);
  xs_d->lpVtbl->SetTextureStageState(xs_d, 0, D3DTSS_ALPHAOP, D3DTOP_MODULATE); xs_d->lpVtbl->SetTextureStageState(xs_d, 0, D3DTSS_ALPHAARG1, D3DTA_TEXTURE);
  xs_d->lpVtbl->SetTextureStageState(xs_d, 0, D3DTSS_ALPHAARG2, D3DTA_DIFFUSE);
  // each of the quad's four vertex colours gets alpha 0x40 (64)
}
```

### 8. 新機能：オーラの強化（`aura_boost`）

gobbiefix 独自の機能でございます（PS2 版や SpectralFix にはございません）。段階（0〜5）を 1 つ上げるごとに、各層の
ずれ量を 25% 広げ、アルファを 25% 強めます。

```cpp
if ( xi_aura_boost != 0 )
{
  float b = 1.0f + 0.25f * (float)xi_aura_boost;
  fStack_a4 *= b; fVar11 *= b;
  uint32_t a = (uStack_a0 >> 24) * (4 + xi_aura_boost) / 4; if (a > 255) a = 255;
  uStack_a0 = (uStack_a0 & 0xFFFFFFu) | (a << 24);
}
```

### 9. 新機能：オーラの残像（`aura_trail`）

こちらも gobbiefix 独自の機能でございます。移動するキャラクターの後ろに、オーラの残像を最大 50 個、それぞれ前の残像
から一定の距離をおいて表示いたします。静止時には残像が淡い 1 つにまとまり、静止中のキャラクターが雑然と見えないよう
にしております。

**組み込み位置。**
- CXiSkeletonActor の描画処理は、オーラ関連の関数をキャラクター内部を指すポインタで呼び出しております。
- キャラクターのワールド座標には、スキニング用パレット（`g_boneWeightMatrixTable`）の
  先頭ボーンの平行移動成分を用いております。直前にそのキャラクターのモデルが描かれているため、この時点ではパレットは
  まだ当該キャラクターのものでございます。
- `GetPos` は移動中もほとんど変化しないため、使用できませんでした。

各残像はキャプチャテクスチャのコピーで、記録時のワールド座標と VIEW×PROJECTION を保持しております。描画時には、その
ワールド座標の*現在の*画面上の位置に置き、`(P._11 / w_now) / (P._11_then / w_then)` で拡大縮小いたします。これにより、
カメラが動いても残像はその場に留まります。

```cpp
float ax, ay, wn, wo;
if (!now_ok || !to_screen(vp, s.pos, rt_w, rt_h, ax, ay, &wn) || !to_screen(s.vp, s.pos, rt_w, rt_h, tx0, ty0, &wo)) continue;
float sc = (p._11 / wn) / (s.p11 / wo);   // (abridged: range checks removed)
float fade = fade_of(k, (int)n, now - s.at, lone, 0.35f, 0.8f) * (float)sf_cfg.aura_trail_alpha / 100.0f;
d->SetTexture(0, s.tex);
quad(d, ax - tx0 * sc, ay - ty0 * sc, ax + (rt_w - tx0) * sc, ay + (rt_h - ty0) * sc, (colour & 0xFFFFFF) | (a << 24));
```

残像が長いほど、各残像の持続時間を長くし、新しいものから古いものへの減衰を急にしております。

```cpp
DWORD life_ms(int n) { return 300u + 60u * (DWORD)n; }
float fade_of(int k, int n, DWORD age, bool lone, float base_lone, float base) {   // k = 1: the newest of n
  if (lone) return base_lone;
  float g = 1.0f + (float)n / 16.0f, x = 1.0f - (float)(k - 1) / (float)(n + 1), a = 1.0f - (float)age / (float)life_ms(n);
  if (a < 0) a = 0;
  return base * powf(x, g) * a;
}
```

キャラクターが 0.5 秒以上姿を消した後に再び現れた場合、またはオーラの色がいずれかのチャンネルで 64 を超えて変化した
場合は、新しい残像の列を開始いたします。こうしないと、前の召喚獣と同じ位置に現れた新しい召喚獣が、前の召喚獣の残像を
引き継いでしまいます。

上級モード（`aura_trail_advanced`）では、画像の代わりに CPU でスキニングされた身体のポーズ済み三角形を記録いたします。
各残像は現在のカメラから 3D で描き直され、テクスチャなし・加算合成・深度テストあり・深度書き込みなしで描画されます。
残像 1 つにつき身体 1 体分の描画負荷がかかります。各時間・係数（300 + 60n ms、指数 1 + n/16、500 ms、64）は設計上の
選択であり、測定値ではございません。

### 10. 崩壊エフェクト：PS2 版の分解表現

PS2 版では、一部のモデル（闇の王の変身）が破片に分かれ、煙と揺らぎを伴って崩れていきますが、PC 版クライアントでは
これが描かれておりません。**voetre 氏の portfix** がこれを再現しております。デバイス層でモデルのスキニング済みの部位を
取得し、個別の破片として描画する手法でございます。gobbiefix の既定（`disintegrate=1`）は portfix のコードを同氏の MIT
ライセンスのもとで使用したものでございます。また、portfix による PS2 版の解析をもとに、gobbiefix 独自の版
（`disintegrate=2`、PS2 版のムービーと同様にちらつくもの）もございます。いずれの版のコードも本書には含めておりません。
原典は portfix をご参照くださいませ。

---

## サウンド

### 11. ループ音が二度と鳴らなくなる問題（`sound_retry`）

**症状。** 雨や風の音が 1 分ほどで消え、エリアや天候が改めて再生するまで無音のままとなります。改変のないゲームでも
生じます。レジストリの同時発音数上限（キー 0029）を 12 から 20 にした場合は、5〜10 分で生じます。

**原因。**
- `CYySoundElem::UpdateTimerAndPosition`（0x100364E0）は、ハンドルが 0 の要素を再始動いたします。
- そこでは、`SetVolumeVector`（0x10036F50）の戻り値がハンドルとして格納されます。
- 戻り値 **0** は「今回の更新のキューが満杯」を意味します。要素は保持され、次の更新で再度要求いたします。
- 戻り値 **−1** は発音数上限による拒否を意味します（`QueueSoundCmdIfChannelsBelowLimit`、0x10038330）。負の
  ハンドルの場合、要素はデストラクタへ送られ、その音は完全に失われます。

**修正。** その呼び出しでの拒否を、キュー満杯の場合と同じく 0 として返します。対象は、聞こえていない間もゲーム自身が
保持する要素（要素自身の「聞こえていない間も保持する」仮想関数 `VT_KEEP_WHEN_UNHEARD`、ゲームがその箇所で行うのと同じ判定）に
限っております。`RET_IN_RESTART` はその呼び出しの戻りアドレスでございます。

```cpp
static undefined4 __thiscall sf_setvol(void *self_, undefined4 a1, undefined4 a2) {   // (abridged: counters removed)
  undefined4 r = prev_setvol(self_, a1, a2);
  if ((uintptr_t)__builtin_return_address(0) != RET_IN_RESTART || !snd_elem) return r;
  void *e = snd_elem;
  if ((int32_t)r < 0 && sf_cfg.sound_retry) {
    typedef uint32_t (__thiscall *KeepFn)(void *);
    if ((uint8_t)((KeepFn)VT_KEEP_WHEN_UNHEARD(e))(e)) return 0;   // kept; it asks again next update
  }
  return r;
}
```

*ソースにおいては：* `UpdateTimerAndPosition` にて、持続する要素については発音拒否をキュー満杯と同様に扱って
いただければ解消するものと存じます。

---

## フレームタイミング

### 12. 正確な 60 fps のリミッター（`precise_pacing`）

**症状。** ゲームが 60 fps ではなく 58.8 fps で動作いたします。

**原因。**
- `Frame_Render`（0x10011A00）内のリミッターは、`60 / (scale × rate) < 下限`（リミッターの 1 フレームあたりの最小ティック数）の間 `Sleep(1)` を繰り返します。
- rate は `timeGetTime` による整数ミリ秒からの `1000 / ms` でございます（0x10019550）。
- 下限が 1 ティックの場合、待機は 17 ms で終わり、1000 / 17 = 58.8 となります。

**修正。** リミッター自身の rate の呼び出しを `QueryPerformanceCounter` で計時し、`Sleep` の代わりに、`v` が下限に
達するちょうどその瞬間まで待機いたします（高分解能の待機タイマーの後、短いスピン）。

```cpp
// the moment v = 60 / (scale * rate) reaches ticks: rate = 1000 / ms, so ms = ticks * 1000 * scale / 60
double want = ticks * 1000.0 * scale / 60.0 + TIMER_PAUSED_MS(timer) + 0.002;   // + the paused ms, + 2 us
LONGLONG end = g_pp_base + (LONGLONG)(want * (double)g_pp_qpf.QuadPart / 1000.0);
...
if (remain > 0) sf_sleep_until(end, g_pp_qpf.QuadPart);
```

### 13. 60 fps 未満・超過時のゲーム内時間（`frame_step`）

**原因。**
- `Frame_Render` はフレームのステップを
  `min(max(trunc(直近 4 フレームの時間の平均、1/60 秒単位), 下限), 20)` として求めます。
- 218 箇所の呼び出しが、`Camera_GetFrameScale`（0x10014CF0）を通じて `max(step, 1)` として読み取っております。
- この切り捨てにより、60 fps 未満では時間が少なく数えられます（私どものダンプでは実時間の 62〜99%）。60 fps を超えて
  下限を外すとステップは 0 となり、1 として返されるため、ゲームが実時間より速く進みます。

**修正。** 切り捨てずに同じ値を保持し、端数をフレーム間で繰り越して整数ティックにいたします。

```cpp
double c = xi_step_carry + f;    // f: the untruncated step, floored and capped as the game's
int w = (int)c;                  // whole ticks this frame
xi_step_carry = c - w;           // the fraction carried to the next frame
xi_step_whole = (float)w;
xi_step_frac = (float)f;
```

モード 1 では、すべての呼び出し元に整数ティックを返します。その合計は、長い目で見ればゲーム自身によるフレーム時間の
計測と一致いたします。モード 2 では、ステップを累積するだけであることを私どもが証明した 14 箇所（すべての経路が
`x += f(step)` で終わるもの）に、正確な端数も与えます。

**判明していないこと：** ステップを読まずに毎フレーム一定量だけ物を動かすコードがどれほどあるかは分かっておりません
（そのような関数の一例は 0x1024DE20 にございます）。そうしたコードは、60 fps を超えると依然として速く動きます。

### 14. 均等なフレームレート上限（`frame_cap_fps`）

Present の時点で、指定のレートでフレーム間隔を均等にいたします。待機の大半を高分解能タイマーで眠り、最後の約 1 ms を
スピンいたします。1 周期を超えて遅れた場合は基準を取り直します。ゲーム本来のステップは 1/60 秒単位の整数ティックで
あるため、ゲーム内時間と実時間が一致するのは 60・30・20・15 の場合のみでございます。

```cpp
LONGLONG period = g_qpf.QuadPart / fps;
if (!g_next || now.QuadPart - g_next > period) g_next = now.QuadPart;   // late by over a frame: re-anchor
LONGLONG remain = g_next - now.QuadPart;
if (remain > 0) {
  double ms = remain * 1000.0 / (double)g_qpf.QuadPart;
  if (ms > 1.5 && g_timer) {                 // sleep most of it on a high-resolution timer, spin the last ~1 ms
    LARGE_INTEGER due; due.QuadPart = -(LONGLONG)((ms - 1.0) * 10000.0);
    if (SetWaitableTimer(g_timer, &due, 0, NULL, NULL, FALSE)) WaitForSingleObject(g_timer, 200);
  }
  LARGE_INTEGER t;
  do { YieldProcessor(); QueryPerformanceCounter(&t); } while (t.QuadPart < g_next);
}
g_next += period;
```

---

## CPU と描画の負荷

### 15. SSE 対応 CPU すべてで SSE を使用（`cpu_tier`、`sse_skinning`）

**原因。** SSE の有無を確認する前に、CPU のベンダー文字列を確認している箇所が 2 つございます。
- 演算ティアの選択処理では、条件分岐（`VENDOR_TEST_JUMP`）により、ベンダー文字列が一致しない限り SSE の判定が
  飛ばされます。
- CPU 判定処理は、ベンダー文字列が「Intel」または「Transmeta」の場合にのみ、スキニング処理用の SSE フラグを
  立てます。

そのため、SSE を備えていてもベンダー文字列が一致しない CPU（たとえば、別のベンダーを報告する場合の Windows on ARM の
x86 エミュレーション）は x87 の経路で動作いたします。0x1002F1D0 ほか 1 つのスキニング処理については、AMD 製の CPU
もすべてこれに該当いたします。演算ティアの判定がどのベンダー文字列を受け付けるかは、本書では扱って
おりません。

**修正**（`cpu_tier`）：`CreateProcessorInstance`（0x1006CB50）がティアを選択する前に、その `je` を 2 つの NOP に
置き換えます（アラインされた dword への 1 回の compare-exchange として書き込み）。

```cpp
volatile LONG *dw = (volatile LONG *)VENDOR_TEST_JUMP;            // test al, al; je <past the SSE test>
const LONG expect = VENDOR_TEST_BYTES, want = VENDOR_TEST_NOPPED;  // the same dword with the je as two NOPs
...
LONG prev = InterlockedCompareExchange(dw, want, expect);
```

*ソースにおいては：* ベンダー文字列ではなく、CPUID の機能ビットのみで判定いただければ解消するものと存じます。

### 16. 待機なしのオクルージョン読み戻し（`occlusion_late`）

**原因。** オクルージョン判定は結果を GPU から即座に読み戻すため、CPU が GPU を待つことになります。

**修正。**
- 各クエリの画像をクエリ専用のロック可能なターゲットへコピーし、そのクエリの*次回*の判定時（描画はとうに完了して
  いる時点）に読み取ります。
- そのため、結果は 1 回分遅れます（視界に入ってきたキャラクターは、次回の判定から描画されます）。
- `occlusion_hold` は、「非表示」の結果が 2 回続いた場合にのみ物体を非表示といたします。遅れた結果では、判定の 3〜4%
  で表示から非表示への反転が生じるためでございます。
- `occlusion_stagger` は、各キャラクターの再判定を 4 フレームごとではなく 8 フレームごとにいたします。

これらの規則は私どもが定めたもので、ゲーム本来のものではございません。

### 17. 描画エンジン：Direct3D 12・11 上の Direct3D 8（d3d8on11）

gobbiefix は独自の Direct3D 8 実装（`d3d8.dll`）を同梱しており、ゲームの Direct3D 8 の呼び出しを Direct3D 12 または
11 上で実行いたします。測定済みの改善例として、各描画のテクスチャテーブルを、生成したピクセルシェーダーが実際に参照する
ビューのみから構築するようにしたところ、その処理が 1 描画あたり 0.72〜0.88 µs から 0.34〜0.36 µs に短縮されました。
描画エンジン本体は本リポジトリには含まれておりません。

---

## 本書で扱っていない設定

gobbiefix には、上記の修正のほかにも設定がございます。以下は本書ではご説明しておりません。

- `skin_merge`：モデルの三角形リストの連続部分を 1 回の `DrawIndexedPrimitive` で描画いたします。
- `gpu_skin`：キャラクターのスキニングを CPU ではなく d3d8on11 の頂点シェーダーで行います。
- `lazy_state`、`block_cache`、`vs_dedupe`、`state_cache`、`extended_state_cache`：何も変えない、または次の描画までに
  元に戻されるデバイス呼び出しを省きます。
- `merge_scenes`：`BeginScene`/`EndScene` を 1 フレームにつき 1 組といたします。
- `batch_up_draws`：連続する `DrawPrimitiveUP` の呼び出しを統合いたします。
- `uncap`：ゲームの 1 フレームあたりの最小待機を外します（60 fps を超える場合に必要です）。
- `smooth_pacing`（`pace_adaptive`）と `step_exact`：フレーム間隔の調整とステップの計時の別方式でございます。
- `liminality`：ゲームのティックの間のフレームを、位置とアニメーションを補間して描画いたします。
- `texture_compression`：読み込み時のテクスチャ圧縮のしきい値でございます。
- d3d8on11 自身の設定：`swap_buffers`、`mipmaps`、`anisotropy`、`renderer`。
- 測定専用：`motion_stats`、`task_draws`、`null_draws`、`ab_levers`。

## クレジット

- **Dellingr**：gobbiefix（d3d8on11 描画エンジン、xikit キット、speedfix）、speedfix_hook、および dagnyfix（その
  flarefix・オーラ修正・decalfix が、`flare_fix`・`aura_size`・`decal_fix` の起点となりました）。
- **Geno**：近日公開予定の gobbiefix の Linux 対応へのご協力。本書に記載した解析および修正には関わっておりません。
- **voetre 氏**：portfix（MIT ライセンス）。`disintegrate=1` は portfix のコードでございます。`sphere_shine` と
  `disintegrate=2` は、portfix による PS2 版の測定に基づいております。
- **Krauerlabs 氏**：SpectralFix。`aura_soft` の 2 つの変更、およびオーラのキャプチャ拡大の着想の出所でございます。
- **Daleterrence 氏**：dagnyfix の decalfix の元となった試作。

FINAL FANTASY、FINAL FANTASY XI、PlayOnline、SQUARE ENIX は、株式会社スクウェア・エニックスの商標または登録商標です。
本プロジェクトは株式会社スクウェア・エニックスが制作・承認・推奨したものではございません。ゲームのファイルおよび
ゲームのコードは一切含まれておりません。

末筆ながら、皆様のますますのご健勝とご活躍を心よりお祈り申し上げます。
