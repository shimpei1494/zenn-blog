---
title: "TanStack Tableの「なぜ？」を理解する（accessor / cell / rowModel の正体）"
emoji: "📊"
type: "tech"
topics: ["tanstacktable", "react", "typescript", "frontend"]
published: true
---
:::message
この記事は私がAIと壁打ちしながらTanstack Tableへの理解を深め、自分が理解しやすいようにまとめた記事になります。私が100%ドキュメントを読み込んではいませんが、Tanstack Tableの要点を理解しやすいようにある程度まとめられたとは思うので、参考になれば幸いです。
:::

# TanStack Tableの「なぜ？」を理解する

TanStack Tableを触り始めたとき、こんな疑問を持ちませんでしたか？

- `getRowModel()` って何？　`data` と何が違うの？
- `accessorKey` と `accessorFn` の違いは？
- `cell` で表示を変えたのに、なぜソートは影響を受けないの？
- なぜただの `<table>` を書くより、こんなに複雑なの？

この記事では、TanStack Tableの核心にある3つの概念

1. **accessor** — 列の値の定義
2. **cell** — 表示の担当
3. **rowModel** — データ変換パイプライン

を整理します。

理解を深めるために実際に動くサンプルリポジトリも用意しました。

https://github.com/shimpei1494/tanstack-table-lab

GitHub Pages でもデプロイしているので、以下のサイトですぐ確認ができます。

https://shimpei1494.github.io/tanstack-table-lab/#/

![サンプルアプリのトップページ](/images/f1fd1e859d4cfe/1.png)
*サンプルアプリのトップページ*

リポジトリは **TanStack Table × Mantine** の組み合わせで実装しています（`@tanstack/react-table` v8・`@mantine/core` v8）。各 Step で概念を段階的に確認できます。

| Step | 内容 | 確認できること |
|------|------|--------------|
| ★Step00 | Basic | useReactTable の最小構成、getRowModel |
| ★Step01 | Accessor vs Cell | accessorFn と cell の独立性 |
| ★Step02 | Sorting | row model パイプラインの仕組み |
| ★Step03 | Filtering | フィルタの pipeline への追加 |
| Step04 | Pagination | ページングの pipeline への追加 |
| Step05 | Column Visibility | row.getVisibleCells() の意味を列ON/OFFで体感 |
| Step06 | Row Selection | rowSelection state と selected row model |
| Step07 | Editing | 外部 React state を更新することで編集を実現 |
| Step08 | Virtual | 10,000件でも軽い仮想スクロール。row model と描画の分離 |
| Step09 | Grouping + Expanding | getGroupedRowModel / getExpandedRowModel でグループ化と開閉 |
| Step10 | Fullscreen | Mantine の useFullscreen フックでテーブルを全画面表示 |

★ がついている Step をこの記事で解説しています。

:::message
リポジトリは更新される可能性があります。Step の内容や数が変わっている場合は、リポジトリの README を確認してください。
:::

---

## TanStack Table は「UIライブラリではない」

最初に最重要ポイントを押さえます。

**TanStack Table はテーブルUIライブラリではありません。**

TanStack Table は

> テーブルの状態管理とデータ計算を行う「ヘッドレス」ライブラリ

です。

https://tanstack.com/table/latest

UIは自分で実装します。TanStack Table が担うのは「どの行を表示するか」「どの順番で並べるか」という計算だけです。

```
data（元データ）
  ↓
TanStack Table（フィルタ→ソート→ページング）
  ↓
getRowModel()（表示する行の最終結果）
  ↓
<table>（自分でHTMLを組む）
```

これが**ヘッドレスUI**の考え方です。

### Mantine などの UIライブラリとの違い

サンプルリポジトリではMantineというUIライブラリを使用しています。MantineのTableコンポーネントは `<Table>` を渡すと見た目ごと提供してくれます。一方、TanStack Tableは `useReactTable()` というフックを提供するだけで、見た目は一切関与しません。

| 比較 | Mantine Table | TanStack Table |
|------|--------------|----------------|
| 提供するもの | UIコンポーネント | ロジック（フック） |
| 描画 | ライブラリが担当 | 自分で実装 |
| 柔軟性 | UIに依存 | 任意のUIで使える |

---

## 最小構成を見る（Step00）

まず最もシンプルな例から始めます。

```tsx
// Step00: 最小構成
const [data] = useState(() => makeData(50));

const columns = useMemo<ColumnDef<Person>[]>(() => [
  { accessorKey: "id",   header: "ID" },
  { accessorKey: "name", header: "名前" },
  { accessorKey: "age",  header: "年齢" },
], []);

const table = useReactTable({
  data,
  columns,
  getCoreRowModel: getCoreRowModel(),  // ← これが必須
});
```

`useReactTable()` に渡すのは3つだけ：

- `data` — 元データ
- `columns` — 列の定義
- `getCoreRowModel` — 最低限必要な行モデル

`table` インスタンスが返ってくるので、あとは自分でHTMLを描くだけです。

```tsx
// Mantine の Table コンポーネントを使って描画
import { Table } from "@mantine/core";
import { flexRender } from "@tanstack/react-table";

<Table striped withTableBorder>
  <Table.Thead>
    {table.getHeaderGroups().map(headerGroup => (
      <Table.Tr key={headerGroup.id}>
        {headerGroup.headers.map(header => (
          <Table.Th key={header.id}>
            {flexRender(header.column.columnDef.header, header.getContext())}
          </Table.Th>
        ))}
      </Table.Tr>
    ))}
  </Table.Thead>
  <Table.Tbody>
    {table.getRowModel().rows.map(row => (
      <Table.Tr key={row.id}>
        {row.getVisibleCells().map(cell => (
          <Table.Td key={cell.id}>
            {flexRender(cell.column.columnDef.cell, cell.getContext())}
          </Table.Td>
        ))}
      </Table.Tr>
    ))}
  </Table.Tbody>
</Table>
```

`flexRender()` は TanStack Table が提供するユーティリティで、列定義の `header` や `cell` をReactコンポーネントとして描画します。

サンプルアプリのStep00を開くと、このDebug Panelも確認できます。

![Step00の画面](/images/f1fd1e859d4cfe/2.png)
*Step00の画面*

以下のように`getCoreRowModel` のみの場合、加工なしの全ての行（Core）と最終出力のgetRowModelの行数が同じ数になることが可視化されています。

![Step00のDebugPanel](/images/f1fd1e859d4cfe/3.png)
*Step00のDebugPanel*

---

## accessor と cell の違い（Step01）

ここが**TanStack Tableで最もよく混乱するポイント**です。

### accessor は「列の値」を決める

```tsx
// accessorKey: データのキーをそのまま使う
{ accessorKey: "name" }

// accessorFn: 計算した値を列として使う
{ id: "totalScore", accessorFn: (row) => row.math + row.english + row.science }
```

accessorは

- ソートの基準
- フィルタリングの対象
- `getValue()` で取得できる値

を決めます。

### cell は「表示」だけを担う

```tsx
{
  id: "totalScore",
  accessorFn: (row) => row.math + row.english + row.science,  // ← 内部の値
  cell: ({ getValue }) => {                                    // ← 表示の担当
    const score = getValue() as number;
    const color = score >= 240 ? "green" : score >= 180 ? "blue" : "gray";
    return <Badge color={color}>{score}点</Badge>;
  }
}
```

`cell` をどれだけ装飾しても、**ソートは `accessorFn` の値で動きます。**

### getValue() を使うべき理由

```tsx
// ✅ 推奨: accessorFn の結果を getValue() で取得
cell: ({ getValue }) => getValue() as number

// ❌ 非推奨: row.original から直接取得
cell: ({ row }) => row.original.math + row.original.english + row.original.science
```

`getValue()` を使うことで、表示とソートの基準が一致します。`row.original` から直接計算すると、`accessorFn` と計算ロジックが2箇所に分散してしまいます。

### 実際に確認する

Step01では「表示モード」を切り替えられるようになっています。

- **Mode A**: `cell: ({ getValue }) => getValue()` — 数値をそのまま表示
- **Mode B**: `getValue()` で取得した値を `score` に入れて `<Badge>{score}点</Badge>` で装飾表示

どちらのモードでも「合計点でソート」すると**同じ順序**になります。表示が変わっても、ソートの基準（`accessorFn`）は変わらないからです。

---

## rowModel パイプラインを理解する（Step02）

TanStack Tableは「どの行を表示するか」を段階的に計算します。これが**row model パイプライン**です。

```
data（元データ）
  ↓ getCoreRowModel()       ← 必須。全行をそのまま通す
  ↓ getFilteredRowModel()   ← 渡せばフィルタが有効になる
  ↓ getSortedRowModel()     ← 渡せばソートが有効になる
  ↓ getPaginationRowModel() ← 渡せばページングが有効になる
  ↓
getRowModel()（最終的に表示する行）
```

`getRowModel()` は「パイプラインの最後の出力」です。

### 使いたい機能を useReactTable に渡すだけで有効になる

ポイントは、`getXxxRowModel` を `useReactTable` に渡すことが**その機能の有効化スイッチ**になっている点です。

```tsx
// ソートだけ有効
useReactTable({
  getCoreRowModel: getCoreRowModel(),
  getSortedRowModel: getSortedRowModel(),
})

// ソート + フィルタを有効
useReactTable({
  getCoreRowModel: getCoreRowModel(),
  getFilteredRowModel: getFilteredRowModel(),
  getSortedRowModel: getSortedRowModel(),
})

// ソート + フィルタ + ページングを有効
useReactTable({
  getCoreRowModel: getCoreRowModel(),
  getFilteredRowModel: getFilteredRowModel(),
  getSortedRowModel: getSortedRowModel(),
  getPaginationRowModel: getPaginationRowModel(),
})
```

必要な機能だけをパイプラインに追加していくイメージです。不要な処理は含まれないので、シンプルなテーブルから多機能なテーブルまで同じパターンで構成できます。

なお、有効/無効は列単位でも制御できます。`enableSorting: false` を列定義に指定すると、その列だけソート対象から外せます。

```tsx
{ accessorKey: "id", header: "ID", enableSorting: false }
```

### state を持つだけでは機能しない

Step02では「ソートあり」と「ソートなし」の2テーブルを並べて比較しています。

```tsx
// ✅ ソートあり
const table = useReactTable({
  data, columns,
  state: { sorting },
  onSortingChange: setSorting,
  getCoreRowModel: getCoreRowModel(),
  getSortedRowModel: getSortedRowModel(),  // ← これを渡して初めてソートが動く
});

// ❌ ソートなし（getSortedRowModel を省略）
const table = useReactTable({
  data, columns,
  state: { sorting },
  onSortingChange: setSorting,
  getCoreRowModel: getCoreRowModel(),
  // getSortedRowModel がない → sorting state が変わっても行は並び変わらない
});
```

ヘッダーをクリックすると `sorting` state は両方のテーブルで更新されます。しかし、`getSortedRowModel` を渡していないテーブルは**行が並び変わりません**。

「state を持つ」と「その state を処理する関数をパイプラインに登録する」は別のことです。この2つがセットになって初めて機能します。

---

## フィルタリングとパイプラインの確認（Step03）

Step03 では `getFilteredRowModel` をパイプラインに追加することでフィルタが有効になります。グローバルフィルタ（全列を横断して部分一致）と列ごとのフィルタの両方を確認できます。

フィルタをかけると、Debug Panel の行数がパイプラインの各段階でどう変化するかが見えます。
グローバルフィルタを「3」で絞り込んだ状態。Core: 8 は元データのまま変わらず、Filtered: 3 に絞られ、getRowModel も 3件になります。

![Step03のフィルターとDebug Panel](/images/f1fd1e859d4cfe/4.png)
*Step03のフィルターとDebug Panel（一部表示）*

`Core` は元データの総数で常に変わりません。フィルタをかけると `Filtered` が減り、最終出力の `getRowModel` も同じ数になります。`getRowModel()` が画面表示に使われるため、表にはフィルター済みのデータだけが表示される、という仕組みです。

---

## TanStack Table を使うとどう変わるか

### 複合機能の連携が自動で正しく動く

自前でフィルタ・ソート・ページングを実装すると、それぞれの連携でバグが出やすくなります。

```
よくある自前実装の落とし穴：
- フィルタを変えても、ページが2ページ目のままになる
- ソートをかけた後にフィルタすると順序がおかしくなる
- 「全件数」と「フィルタ後件数」の計算がズレる
```

TanStack Table ではフィルタ→ソート→ページングのパイプラインが内部で一貫して処理されるため、`state` を渡すだけで各機能が正しく連携します。

### 型安全に列を定義できる

```tsx
// Person 型に存在しないキーは型エラーになる
const columns: ColumnDef<Person>[] = [
  { accessorKey: "nmae" }, // ← typo → 型エラー
]
```

`ColumnDef<T>` の型パラメータが効くため、列定義のミスをコンパイル時に検出できます。

### 後から機能を追加しやすい

最初はソートだけ、後からフィルタとページングを追加、という拡張がパイプラインへの追記だけで済みます。自前実装だと機能を追加するたびに既存のデータ処理ロジックと絡み合いが増えていきます。

---

## なぜこの設計なのか

TanStack Table の設計思想を一言でいうと

> **データ管理と表示を完全に分離する**

です。

| 担当 | 責務 |
|------|------|
| React state | 元データの管理 |
| TanStack Table | 派生データの計算（フィルタ・ソート・ページング） |
| UIコンポーネント | 表示（HTML/CSS） |

この分離によって、任意のUIライブラリ（今回のMantineなど）と組み合わせられる柔軟性が生まれています。逆に言えば、この分離を理解せずに「とりあえず動かす」状態になりやすいのが TanStack Table の難しさです。

---

## まとめ

TanStack Tableを理解する3つのポイント：

| 概念 | 役割 |
|------|------|
| `accessor` | 列の値（ソート・フィルタの基準）を決める |
| `cell` | 表示だけを担う（accessorの値に影響しない） |
| `getRowModel()` | パイプラインの最終出力（表示する行） |

そして最重要の前提：

> TanStack Table はヘッドレス。UIは自分で実装する。

この3つを押さえると、TanStack Table のドキュメントが読みやすくなります。

---


## 余談
仕事でMantineで表を作りたいとなった時に安易に以下のMantine React Table（以下MRTと略す）というライブラリを使用してしまいました。ただ、Mantineのバージョンアップが早いのに対し、MRTではメンテナンスが活発とはいえず、Mantineのバージョンアップの妨げになっていました。
https://www.mantine-react-table.com/
ただ、MRTはMantineとTanstack Tableを組み合わせたものなので、MRTライブラリを使用しなくても今回のように同じような表を作ることが可能でした。AIにMantineとTanstack Tableのドキュメントなど参照させたりもできるので、思ったより簡単に移行することができました。
