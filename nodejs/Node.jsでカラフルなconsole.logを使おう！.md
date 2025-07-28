# Node.jsでカラフルなconsole.logを使おう！

# はじめに
普段、一面の白いconsole.logを見て、イライラしたり、重要な情報を一目で見つけるのが難しいと感じたことはありませんか？今日は、console.logをカラフルに変える方法を皆さんに共有します。

# クラスの実装
```typescript:ColorfulLog.ts
class ColorfulLog {
    export const success = (msg: string) => {
      console.log("\x1b[32m%s \x1b[0m", `SUCCESS: ${msg}`);
    };
    
    export const error = (msg: string) => {
      console.log("\x1b[31m%s \x1b[0m", `ERROR: ${msg}`);
    };
    
    export const warning = (msg: string) => {
      console.log("\x1b[33m%s \x1b[0m", `WARNING: ${msg}`);
    };
    
    export const task = (msg: string) => {
      console.log("\x1b[35m%s \x1b[0m", `TASK: ${msg}`);
    };
    
    export const info = (msg: string) => {
      if (!msg) return;
      const lines = msg.split("\n").filter(Boolean);
      for (const line of lines) {
        console.log("\x1b[34m%s \x1b[0m", `INFO: ${line}`);
      }
    };
}

export default new ColorfulLog();
```

# いざ使おう！

```ts
import "module-alias/register";
import ColorfulLog from "@utils/class/ColorfulLog";

ColorfulLog.info("infoです");
ColorfulLog.task("taskです");
ColorfulLog.warning("warningです");
ColorfulLog.error("errorです");
```

**📷 実際に使うイメージ**

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3760374/a60a74ae-778a-44a4-b26d-96c0374ef778.png)

# まとめ

この記事では、一般的な白黒のconsole.log出力を、より視認性が高く、情報が把握しやすいカラフルなログに変える方法をご紹介しました。messageUtil.tsを用いて、ログの種類ごとに異なる色を割り当てることで、エラー、警告、情報などのログを一目で区別できるようになります。また、dayjsライブラリを使用してログにタイムスタンプを追加することで、ログが記録された正確な時刻を把握することが可能になります。