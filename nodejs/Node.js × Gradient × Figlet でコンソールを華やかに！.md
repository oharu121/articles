# Node.js × Gradient × Figlet でコンソールを華やかに！

# はじめに

以下の記事を参考に [`oh-my-logo`](https://www.npmjs.com/package/oh-my-logo) を試してみたところ…
実際に動かしてみると、`ERR_PACKAGE_PATH_NOT_EXPORTED` というエラーが発生してしまいました。

https://zenn.dev/shinshin86/articles/5abe789dc2f4f3

そこで、**自分でロゴ出力ツールを実装することにしました！**
その名も `OhMyLogo`、800スターおめでとうございます 🎉

**👇 実際の出力：**

![banner](https://storage.googleapis.com/zenn-user-upload/40d272d2ba33-20250717.png)

**📦 使用技術**

>* [`figlet`](https://www.npmjs.com/package/figlet)：アスキーアート風の文字生成
>* [`gradient-string`](https://www.npmjs.com/package/gradient-string)：文字列にグラデーションカラーを適用

**✨ 主な機能**

>* `figlet` による **アスキーアート風フォントの装飾**
>* `gradient-string` による **グラデーション文字装飾**
>* シンプルかつ直感的な **メソッドチェーン構文**
>* カスタムパレット + `gradient-string` のプリセット両対応

# 1. クラスの実装

```ts:OhMyLogo.ts
import figlet from "figlet";
import gradient from "gradient-string";

const DEFAULT_PALETTE: PaletteName = "sunset";
const DEFAULT_FONT: figlet.Fonts = "ANSI Shadow";
const DEFAULT_TEXT_WIDTH = 100;

class OhMyLogo {
  private text: string = "";
}

export default new OhMyLogo();
```

**📌 ポイント**

>* `text` で内部で維持することで、各メソッドにリレーできる
>* `new OhMyLogo()` を毎回書かなくても良いように、クラスをインスタンス化した
>* **メソッドチェーン** で流れるように組み立てられるAPI

# 2. メインな処理

```ts:OhMyLogo.ts
public createLogo(
  text: string,
  palette: PaletteName = DEFAULT_PALETTE
): string {
  return this.setText(text).addFontStyle().addGradient(palette);
}
```

```ts:OhMyLogo.ts
public createRandomLogo(text: string): string {
  const randomIndex = Math.floor(Math.random() * PALETTE_NAMES.length);
  const randomPalette = PALETTE_NAMES[randomIndex]!;
  return this.setText(text).addFontStyle().addGradient(randomPalette);
}
```

- 使用例

```ts
// 通常のロゴを作る
const logo = OhMyLogo.createLogo("Hello World!");
console.log(logo);

// ランダムなロゴを作る
const logo = OhMyLogo.createRandomLogo("Hello World!");
console.log(logo);

// カスタマイズなロゴを作る
const logo =
  OhMyLogo
    .setText("Hello World!")
    .addFontStyle("bloody")
    .addGradient("rainbow");
    
console.log(logo);
```

**📌 ポイント**

>* `createLogo` や `createRandomLogo` を使えば簡単にロゴを作れます
>* カスタマイズをしたい場合、直接メソッドチェーンを呼ぶのも◎

# 3. メソッドチェーンの実装

```ts:OhMyLogo.ts
public setText(text: string): this {
  this.text = text;
  return this;
}
```

```ts:OhMyLogo.ts
public addFontStyle(
  font: figlet.Fonts = DEFAULT_FONT,
  width: number = DEFAULT_TEXT_WIDTH
): this {
  this.text = figlet.textSync(this.text, {
    font,
    horizontalLayout: "default",
    verticalLayout: "default",
    width,
    whitespaceBreak: true,
  });
  return this;
}
```

```ts:OhMyLogo.ts
public addGradient(palette: PaletteName): string {
  let gradientFunc;

  if (palette in PRESET_GRADIENTS) {
    gradientFunc = PRESET_GRADIENTS[palette as PresetGradient];
  } else if (palette in CUSTOM_GRADIENTS) {
    gradientFunc = gradient(CUSTOM_GRADIENTS[palette as CustomeGradient]);
  } else {
    gradientFunc = gradient(CUSTOM_GRADIENTS[DEFAULT_PALETTE]);
  }

  return gradientFunc.multiline(this.text);
}
```

**📌 ポイント**

>* `text` は `プレーンテキスト` → `描画` → `着色` の順で加工します
>* `setText()`: プレーンな文字列をセットする
>* `addFontStyle()`: figletでアスキーアートに変換する
>* `addGradient()`: gradient-stringで着色する
>* `プリセット` → `カスタム` → `フォールバック` と段階的にチェック
>* `multiline()` で複数行対応

# 4. 定数と型

```ts:OhMyLogo.ts
const PRESET_GRADIENTS = {
  cristal: gradient.cristal,
  teen: gradient.teen,
  mind: gradient.mind,
  morning: gradient.morning,
  vice: gradient.vice,
  passion: gradient.passion,
  fruit: gradient.fruit,
  instagram: gradient.instagram,
  atlas: gradient.atlas,
  retro: gradient.retro,
  summer: gradient.summer,
  pastel: gradient.pastel,
  rainbow: gradient.rainbow,
} as const;

const CUSTOM_GRADIENTS: { [key: string]: string[] } = {
  gradBlue: ["#4ea8ff", "#7f88ff"],
  blackWhite: ["#000000", "#FFFFFF"],
  sunset: ["#ff9966", "#ff5e62", "#ffa34e"],
  dawn: ["#00c6ff", "#0072ff"],
  nebula: ["#654ea3", "#eaafc8"],
  mono: ["#f07178", "#f07178"],
  ocean: ["#667eea", "#764ba2"],
  fire: ["#ff0844", "#ffb199"],
  forest: ["#134e5e", "#71b280"],
  gold: ["#f7971e", "#ffd200"],
  purple: ["#667db6", "#0082c8", "#0078ff"],
  mint: ["#00d2ff", "#3a7bd5"],
  coral: ["#ff9a9e", "#fecfef"],
  matrix: ["#00ff41", "#008f11"],
};

const PALETTE_NAMES = [
  ...Object.keys(CUSTOM_GRADIENTS),
  ...Object.keys(PRESET_GRADIENTS),
] as const;

type CustomeGradient = keyof typeof CUSTOM_GRADIENTS;
type PresetGradient = keyof typeof PRESET_GRADIENTS;
type PaletteName = (typeof PALETTE_NAMES)[number];
```

**📌 ポイント**

>* `PRESET_GRADIENTS`：`gradient-string` に元から用意されている （プリセット）
>* `CUSTOM_GRADIENTS`：HEXで定義した （カスタム）

# まとめ

CLIやNode.jsプロジェクトで印象的なロゴ出力をしたいなら、`OhMyLogo`はとても簡単に導入できます。

* 見た目が映える！
* 自由にパレット追加できる！
* シンプルで直感的なAPI！

あなたのプロジェクトの冒頭を、**鮮やかなロゴで飾ってみませんか？**