# react-hook-form `src/` バグ調査レポート (2026-08-05)

`src/` (コアライブラリ) を中心に静的解析を行い、`app/`(手動E2Eテスト用デモアプリ) も併せて確認した。
以下は再現シナリオを具体的に示せた、確度の高い不具合のみを記載する(単なるスタイル指摘は含めない)。

---

## 1. 【重大度: 高】フィールド配列のルートエラーが `append`/`prepend`/`insert`/`remove` で消える

- **ファイル**: `src/logic/updateFieldArrayRootError.ts:16-18`, `src/logic/createFormControl.ts:198-209` (`_updateFieldArray`)
- **関連**: `src/utils/append.ts`, `src/utils/insert.ts`, `src/utils/remove.ts`, `src/utils/prepend.ts`

### 原因

`useFieldArray({ rules: { validate } })` で配列全体に対するエラー(`errors.items.root`)を保持する際、`updateFieldArrayRootError` は配列オブジェクトに直接 `root` という非インデックスプロパティを生やしている。

```ts
// updateFieldArrayRootError.ts
const fieldArrayErrors = compact(get(errors, name)); // 実体は配列
set(fieldArrayErrors, 'root', error[name]);           // fieldArrayErrors.root = {...}
set(errors, name, fieldArrayErrors);
```

一方、`append`/`prepend`/`insert`/`remove` は配列を **スプレッド構文や `slice` で新規生成**している。

```ts
// append.ts
return [...data, ...convertToArrayPayload(value)];
// insert.ts
return [...data.slice(0, index), ...convertToArrayPayload(value), ...data.slice(index)];
```

JS のスプレッド/`slice` はインデックス要素のみをコピーし、**カスタムプロパティ(`.root`)はコピーされない**。Node での検証:

```js
const arr = []; arr.root = { message: 'x' };
[...arr].root      // undefined
arr.slice().root   // undefined
```

`createFormControl.ts` の `_updateFieldArray` はフィールド配列操作のたびに `errors` をこの `method`(= `appendAt`/`insertAt`/`removeArrayAt` 等)で再生成しているため、`.root` エラーが黙って消える。

```ts
// createFormControl.ts:202-208
const errors = method(get(_formState.errors, name), args.argA, args.argB);
shouldSetValues && set(_formState.errors, name, errors);
```

`swap.ts`/`move.ts` は配列を **in-place で変更**しているため `.root` は残る ⇒ 操作間で挙動が不揃い。

### 再現シナリオ

```tsx
const { control, trigger } = useForm({ defaultValues: { items: [{ name: 'a' }] } }); // mode: 'onSubmit' (デフォルト)
const { append } = useFieldArray({
  control,
  name: 'items',
  rules: { validate: (v) => v.length >= 3 || '3件以上必要です' },
});

await trigger('items'); // errors.items.root.message === '3件以上必要です'
append({ name: 'b' });  // 要素数はまだ2件 → 本来エラーは継続すべき
```

`append` 直後に `errors.items.root` が `undefined` になり、フォームが有効に見えてしまう。デフォルトの `onSubmit` モードでは、`useFieldArray` 側の再検証エフェクト(`useFieldArray.ts:314-357`, `!isOnSubmit || isSubmitted` でガード)も走らないため、明示的に `trigger()` や送信するまでエラーが復活しない。

---

## 2. 【重大度: 中〜高】`valueAsDate: true` + `min`/`max` の組み合わせでバリデーションが無効化される

- **ファイル**: `src/logic/validateField.ts:126-159`, `src/logic/getFieldValueAs.ts:17-18`

### 原因

`valueAsDate: true` を指定すると、`getFieldValueAs` が文字列値を `Date` オブジェクトに変換して `_formValues` に保存する。

```ts
// getFieldValueAs.ts
: valueAsDate && isString(value)
? new Date(value)
```

`validateField` はこの値(`inputValue`)を使って `min`/`max` の分岐を決めているが、分岐条件が `isNaN(inputValue)` になっている。

```ts
// validateField.ts:126
if (!isNullOrUndefined(inputValue) && !isNaN(inputValue as number)) {
  // 数値用の分岐 (本来は date 分岐に行くべき)
  const valueNumber = ... ;
  exceedMax = valueNumber > maxOutput.value; // maxOutput.value は文字列 (例: '2010-01-01')
```

`isNaN(dateObject)` は `Number(dateObject)`(=タイムスタンプ)で評価されるため、**有効な日付では常に `false`** になり、本来通るべき「日付用分岐」(137行目以降)ではなく「数値用分岐」に入ってしまう。さらにそこではミリ秒のタイムスタンプ(数値)を `max` の**文字列**('2010-01-01' など)と直接比較するため、比較結果は常に `NaN` 相当 → `exceedMax`/`exceedMin` は常に `false`。

Node での検証:

```js
const d = new Date('2019-02-12');
isNaN(d);                 // false → 数値分岐に入ってしまう
+d > '2019-03-12';         // false (文字列は NaN に変換される)
```

### 再現シナリオ

```tsx
register('birthDate', { valueAsDate: true, max: '2010-01-01' });
// <input type="date" /> に 2025-01-01 を入力しても max エラーが一切発生しない
```

`validateField.test.tsx` の既存テスト(764-859行)は `inputValue` に生の文字列を直接渡しており、`register` → `getFieldValueAs` を経由する実際のパイプラインを通していないため、このバグはテストで検出されていない。

---

## 3. 【重大度: 中】`useFieldArray().update()` は他の操作と異なり `errors`/`touchedFields` を更新しない

- **ファイル**: `src/useFieldArray.ts:260-288`, `src/logic/createFormControl.ts:183-222`

### 原因

`update()` は `control._updateFieldArray` を最後の引数 `shouldUpdateFieldsAndState = false` で呼んでいる。

```ts
// useFieldArray.ts
control._updateFieldArray(name, updatedFieldArrayValues, updateAt, {
  argA: index,
  argB: updateValue,
}, true, false); // ← shouldUpdateFieldsAndState: false
```

`append`/`prepend`/`insert`/`remove` はこの引数を省略(デフォルト `true`)しており、`_fields`・`_formState.errors`・`_formState.touchedFields` を配列操作に合わせてシフト/更新する。しかし `update()` だけは `false` を渡すため、`createFormControl.ts:198-222` の該当ブロックが丸ごとスキップされ、`errors`/`touchedFields` がそのまま残る。

### 再現シナリオ

1. `fields.0.firstName` に `required` ルールを設定。
2. 一度バリデーションを走らせ `errors.fields[0].firstName` にエラーをセット。
3. `update(0, { firstName: '正しい値' })` を呼ぶ。

`dirtyFields` は(無条件ブロックのため)再計算されるが、`errors.fields[0]` は更新されないため、有効な値に更新したにもかかわらず古い「required」エラーが表示され続ける。別のバリデーションイベント(再マウントしたinputのblur、`trigger()`の再実行、送信など)が起きるまで解消されない。

---

## 4. 【重大度: 低】`deepEqual(NaN, NaN)` が `false` を返す

- **ファイル**: `src/utils/deepEqual.ts:6-9`, `src/utils/deepEqual.ts:37`

### 原因

```ts
export default function deepEqual(object1: any, object2: any) {
  if (isPrimitive(object1) || isPrimitive(object2)) {
    return object1 === object2; // NaN === NaN は false
  }
  ...
```

`NaN` は `isPrimitive` によりプリミティブ扱いされるため、`===` 比較に落ちて `NaN === NaN` → `false` になる。オブジェクト内のプロパティ比較(37行目 `val1 !== val2`)でも同様の問題がある。

`deepEqual` は `isDirty`/`dirtyFields` の判定(`createFormControl.ts` の `updateTouchAndDirty`, `_getDirty`, `getDirtyFields`)に使われており、`getFieldValueAs`(`valueAsNumber` で空文字を `NaN` に変換)経由で `NaN` が実際に発生しうる。デフォルト値と現在値が両方とも正当に `NaN` であるケース(数値フィールドの初期値が未入力状態など)で、誤って「変更あり」と判定される可能性がある。影響範囲は限定的だが、`Number.isNaN`/`Object.is` を使うのが本来正しい。

---

## まとめ

| # | 箇所 | 確度 | 影響度 |
|---|------|------|--------|
| 1 | `updateFieldArrayRootError` × 配列操作(append/prepend/insert/remove) | 高 | 高 (配列全体バリデーションが実質無効化) |
| 2 | `validateField` の `valueAsDate` + `min`/`max` | 高 | 中〜高 (日付フィールドの範囲バリデーションが無効化) |
| 3 | `useFieldArray().update()` の `errors`/`touchedFields` 未更新 | 中 | 中 (古いエラー表示が残留) |
| 4 | `deepEqual(NaN, NaN)` | 中 | 低 (`valueAsNumber` の空値比較など限定的なケース) |

`app/` 配下は手動確認用のデモページ群であり、ロジック上のバグではなく個別ページの実装例のため、今回のレポート対象外とした。
