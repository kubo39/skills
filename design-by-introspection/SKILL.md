---
name: design-by-introspection
description: ジェネリック/テンプレートな汎用部品（コンテナ・range・アロケータ・シリアライザ・数値ラッパ・出力先）を設計するとき、型の能力（メソッド/メンバの有無）でコンパイル時に分岐したいとき、Hook/ポリシー型を受け取る型の契約を設計するとき、継承階層やインターフェース爆発を解きほぐすときに使う。D で template constraint / static if / __traits / is(typeof) を書く場面では必ず参照。Rust / Zig / C++ / Nim / Go でジェネリック設計をしていて「D ならこう書ける手法」の可否・向き不向きを判断するときにも使う。
---

# Design by Introspection (DbI)

Andrei Alexandrescu が提唱した設計パラダイム。契約観を一言でいえば
**「必須は最小に、任意は内省で発見し、なければ既定か不在」**。
型が満たすべきインターフェースを事前に宣言させるのではなく、型が **たまたま何をできるか** を
コンパイル時に問い、その結果に応じて自分自身の構造・API・実装を組み替える。

| アプローチ | 契約の決め方 | 契約にない機能 | 実行時コスト |
|---|---|---|---|
| OOP インターフェース | 事前に固定 (vtable) | 表現できない / インターフェース爆発 | 仮想呼び出し |
| 素朴なジェネリクス（制約のみ） | 最小公倍数に固定 | 使えない（最弱の型に合わせる） | ゼロ |
| **DbI** | **必須は最小、任意は内省で発見** | **あれば使い、なければ代替かメンバごと消す** | ゼロ |

DbI は最適化テクニックではなく**設計手法**。成果物はインターフェース定義ではなく
「**プロトコル**（必須メンバ + 任意メンバ + それぞれの意味論と既定解釈）」の文書と、
それを内省する規約になる。D は表現力が最も高いので **D（Phobos）の例を正典** として示す。

## 中核原則（言語非依存）

- **構造的に能力を問う**：「名前付き型か」ではなく「`.foo()` を呼べるか」「`.length` を持つか」で分岐。conformance は宣言ではなく観測。
- **使わないものに金を払わない**：能力が無い型には、その能力に依存する経路もメンバも一切生成しない。
- **opt-in な高速経路 + フォールバック**：最適化フックがあれば使い、無ければ汎用経路に退避（graceful degradation）。
- **継承・強制インターフェースを避ける**：基底クラスや必須 trait を背負わせず、型は素のままで合成可能に保つ。
- **黙って選ばない、説明して止まる**：能力不足は constraint で呼び出し地点のコンパイルエラーに。曖昧な組み合わせは `static assert(0, "説明")` で止める。
- **プロトコルを文書化する**：任意メンバには「ないときの既定解釈」を必ず決め、表として明文化する（`std.checkedint` 冒頭表が手本）。

## D の道具箱

| 道具 | 役割 |
|---|---|
| `__traits(hasMember, T, "name")` | メンバの存在検査 |
| `is(typeof(expr))` / `__traits(compiles, ...)` | 式が型検査を通るか（存在 + 呼び出し可能性を一度に） |
| `static if` / `static foreach` | 条件付きコンパイル。**メンバ宣言の位置で使える**のが D の強み（「条件を満たすときだけメソッドが存在する型」を直接書ける） |
| テンプレート制約 `if (...)` | 適用条件・オーバーロード選択。エラーを呼び出し地点に出す |
| `enum` trait | 内省結果に名前を付けて再利用（`isInputRange`, `hasLength`） |
| 文字列 mixin | 内省結果からコード生成（`forwardToMember`） |
| `stateSize!T` | 「実体として領域を要するか」判定（パターンE の基盤）。※ 所属モジュール `std.experimental.allocator.common` は文書で「直接使用は非推奨」（internal 扱い）と明記——自前コードでは同等の数行テンプレートを書き写すのが安全 |

## パターンカタログ（Phobos 正典）

### A. 構造的プロトコル — 必須の核 + 任意の周辺

契約を「必須メンバの小さな核 + 任意メンバの大きな周辺」として定義・文書化する。
アロケータなら必須は `alignment`・`allocate` のみ、任意に `expand`/`owns`/`deallocate`/
`alignedAllocate`/... が並ぶ。Range も「核 = empty/front/popFront、周辺 = save/length/opSlice」で同型。
`NullAllocator` のような**全プリミティブの自明実装＝プロトコルの見本**を1つ用意するとよい（テスト素材にもなる）。

### B. 能力の条件付き露出と伝播

合成コンポーネントは、構成要素が任意プリミティブを持つかを内省し、**持つときだけ**合成結果にも生やす。
合成ロジックは機能ごとに違い、**条件式そのものが設計判断の証明になる**：

```d
// Segregator: alignedAllocate は AND —— どちらに振られるか分からないから両方必要
static if (__traits(hasMember, SmallAllocator, "alignedAllocate")
        && __traits(hasMember, LargeAllocator, "alignedAllocate"))
void[] alignedAllocate(size_t s, uint a) { ... }

// expand は OR —— 片方だけ持つならそのサイズ帯でだけ拡張、他方は false
// FallbackAllocator.expand は「Primary が owns を持つこと」が前提条件 ——
// どちらが確保したか判定できなければ転送先を決められない（意味論上の依存が条件式に現れる）
static if (__traits(hasMember, Primary, "owns")
    && (__traits(hasMember, Primary, "expand") || __traits(hasMember, Fallback, "expand")))
bool expand(ref void[] b, size_t delta) { ... }
```

合成結果の能力は手で列挙するのではなく**構成要素の能力から計算される**。
構成を1段変えれば API 表面も自動で再計算され、手で同期すべき宣言がどこにもない。

### C. フック規約 — 「あれば全権、なければ既定」

Hook/ポリシー型を受ける本体は、操作ごとに「フックがこの名前のメソッドを持つか」を検査し、
持つなら委譲、持たないなら既定動作。**粒度の異なる2種類のフック**を階層にするのが要点：

```d
// Checked!(T, Hook).opBinary の骨格 (std.checkedint)
static if (__traits(hasMember, Hook, "hookOpBinary"))
    return hook.hookOpBinary!op(payload, rhs);        // 全権フック: 演算ごと乗っ取る
else static if (__traits(hasMember, Hook, "onOverflow"))
{
    auto r = opChecked!op(payload, rhs, overflow);    // 既定: チェック付き演算
    if (overflow) r = hook.onOverflow!op(payload, rhs); // 通知フック: 異常時だけ相談
}
else
    return mixin("payload" ~ op ~ "rhs");             // 完全な既定動作
```

- **全権フック**（`hookOpBinary`）= 操作そのものを置き換える。あると通知フックは見られもしない。
- **通知フック**（`onOverflow`）= 既定パスが異常を検出したときだけ呼ばれる。

この階層により、フック実装者は関心のあるメソッド**だけ**書けばよく、
**実装量が「どこまで挙動を変えたいか」に比例**する（Saturate は onOverflow 系のみ、
WithNaN は演算一式 + `min`/`max`/`defaultValue` まで——フックは型の値域すら再定義できる）。

### D. 試行階層ディスパッチ

複数の実現手段を**良い順に**並べ、`static if`/`else static if` で最初に使えるものを選ぶ。
- 高速パス選択：`walkLength` は `hasLength` なら O(1)、なければ O(n)。
- 利用者の型の受容：`put` は「専用 `put` メソッド → スライス代入 → front/popFront → デリゲート呼び出し」と
  カスケードし、自然に書いたどの形の型でも出力先になる。`std.getopt` はデリゲートの引数 0/1/2 個まで判別。
- **等級付け**：`std.format` の `hasToString` は受理シグネチャを優先度つき等級で判定。
  利用者は `string toString()` から始め、writer 版を実装した瞬間に**呼び出し側を1行も変えず**割り当てゼロになる。
  等級付き内省は「より良い実装への移行インセンティブ」を API に埋め込む。

### E. 状態ゼロの部品にコストを払わない

ポリシー型・部品型をメンバに持つときは `stateSize` を検査し、状態がなければ `alias`（またはシングルトン）に置き換える：

```d
static if (stateSize!Hook > 0) Hook hook;
else alias hook = Hook;            // Checked!int は int と同サイズになる
```

「ポリシーを足してもメモリレイアウトが変わらない」ことが DbI 合成のスケーラビリティを支える。

### F. 「分からない」を正直に返す — Ternary

任意プリミティブの問い合わせ（例：アロケータの `owns`）は bool でなく三値
（`Ternary.yes/no/unknown`）で返す。DbI ではメソッドが「ない」ことが正当な状態なので、
「ない（unknown）」と「否（no）」を呼び出し側が区別できる語彙が要る。
bool で `false` を返すと合成アルゴリズム（「Primary が所有していなければ Fallback へ」）が誤動作する。
三値は論理演算で畳み込める：`primary.owns(b) | fallback.owns(b)`。

### G. 欠けたプリミティブの既定実装を外から供給する

任意プリミティブには、必須プリミティブだけから構成できる汎用既定実装を**フリー関数**で用意する
（汎用 `reallocate` は `expand` があれば使い、なければ確保 + コピー）。UFCS により
メンバ実装と同じ構文で呼べる。実装者は本当に最適化したいものだけ書けばよくなる。
定型転送は `forwardToMember`（文字列 mixin）で消す。

### H. 静的世界と動的世界のブリッジ

DbI の弱点は「型がすべてコンパイル時に決まる」こと。実行時に構成を選びたい場面のために、
静的部品をクラスインターフェースへ**一方向に**包むアダプタを用意する
（`allocatorObject` → `RCIAllocator`）。内省は包む瞬間に一度だけ行われ、結果が vtable 実装に
焼き込まれる（`owns` が無ければ `Ternary.unknown` を返す override が生成される）。
**最後の一皮だけ**動的にするのがコツ。

## 規律と落とし穴

1. **タイポはエラーにならず、既定動作に「静かに」落ちる。**
   `__traits(hasMember, Hook, "hookOpBinray")`（綴り間違い）は常に false で、コンパイルは通り、
   フックは呼ばれない。OOP の `override` なら即エラーになるのと対照的な、DbI 固有の構造的弱点。
   対策：(a) 検査式に名前を付けて一箇所に集約（`isOutputRange`, `canMatch`）、
   (b) プロトコルを表として文書化、(c) 提供フック/部品の挙動を網羅する unittest を本体側に持つ
   ——「フックが呼ばれていない」ことはテストで露見させる。
2. **組み合わせ爆発はテストにも及ぶ。** 任意メンバ n 個なら合成形は最大 2^n。
   「部品単体 + 代表的な合成」をテストし、最小プロトコル実装（NullAllocator 相当）を素材に使う。
   設計時点で任意メンバ同士の依存（`expand` には `owns` が要る等）を最小化して組み合わせを抑える。
3. **エラーメッセージは素の OOP より悪化しうる。** 深い `static if` の最奥で失敗すると遠いエラーが届く。
   constraint で呼び出し地点エラー化し、曖昧ケース（例：両辺の Hook が衝突）は
   `static assert(0, "説明")` で止める——黙って選ばない。
4. **属性は決め打ちしない。** 内省で選ばれるパスごとに `@safe`/`pure`/`nothrow` 性が変わるため推論に任せる。
5. **すべてを DbI にしない。** 任意メンバは「既定実装が定義できるもの」に絞る。向くのは
   「中核は同じで能力に幅がある部品ファミリ」（range・アロケータ・数値ラッパ・出力先）。
   能力の幅が1次元に並ばない領域は素直に別 API へ。
   なお `std.experimental.allocator` が20年近く experimental のままなのは、概念数・学習コスト・
   `@safe` 整合という「DbI を限界まで推した場合のコスト」の実例でもある。

## 他言語での可否判断（D の例 → 対象言語）

分かれ目は「型駆動のコンパイル時分岐（`static if` 相当）」と「構造的な能力検出」、
さらに「**条件付きメンバ宣言**」を持つか。

| 言語 | DbI 適性 | D の対応物 | 注意点 |
|------|---------|-----------|--------|
| **Zig** | ◎ ほぼ同等 | `comptime`, `@hasDecl`/`@hasField`, `@typeInfo` | 思想的に最も近い。`comptime` 反射で素直に書ける。 |
| **C++** | ○ 近い | concepts (`requires`), `if constexpr`, detection idiom | `if constexpr` は**関数本体内のみ**——「条件付きメンバ宣言」（パターンB/C）は特殊化や CRTP の迂回が要る。 |
| **Nim** | ○ 近い | `when compiles(x.foo())`, `declared()`, concepts | `when compiles` が `is(typeof)+static if` のほぼ直訳。 |
| **Rust** | △ 部分的 | trait + bound、blanket impl、（不安定）specialization | conformance が明示で構造的でない。「あれば速い経路」は specialization 未安定化のため困難（`Vec::from_iter` 等は内部の不安定機能で実現）。能力ごとに trait を切って bound で表現するのが筋。 |
| **Go** | ✗ 静的には不向き | interface 充足、実行時の型アサーション | コンパイル時の構造的反射が無い。ただし `io.Copy` が `io.WriterTo` を型アサーションで検出して高速化するのは **DbI の実行時版**——検査コストと失敗がランタイムに残る点が違い。素直に interface 設計へ倒す。 |

判断手順：D の例の分岐が「コンパイル時に能力を観測して経路（やメンバ）を消している」なら、
対象言語に `static if` 相当が無い限り**そのままは移植できない**——trait/interface で明示化するか、
実行時に倒すかを選ぶ。それもこの判断自体が DbI の一部。

## 使い方の要約

1. 継承・強制インターフェースで縛られた汎用コードを見たら、「必須の核 + 任意の周辺」のプロトコルに解きほぐせないか考える。
2. D で書くなら：constraint で弾く → 能力検出（trait に命名）→ `static if` で経路/メンバを生成し分け → 既定実装をフリー関数で供給。合成するならパターンB（AND/OR/前提条件）、Hook を受けるならパターンC + E、問い合わせ系は F。
3. 落とし穴チェック：タイポのサイレントフォールバック対策（命名 trait + 網羅 unittest）、曖昧時の `static assert`、属性は推論任せ。
4. 他言語なら対応表で can/cannot/should を判断。
