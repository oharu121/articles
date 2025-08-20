## TypeScript で Kubernetes Pod の IP・ステータスを簡単取得する方法

## はじめに

本記事では、**Kubernetes クラスター上の Pod 情報をリモート環境から取得**するための TypeScript クラスを紹介します。

Node.js 環境で `kubectl` コマンドをリモート実行し、対象の Pod の「名前」「ステータス」「IP アドレス」を効率的に取得・加工する仕組みを構築しました。

以下のような課題を感じたことはありませんか？

- kubectl exec を毎回手動で叩くのが面倒
- 複数の Pod を対象に検索・情報取得したい
- 実行結果をコード上で加工・再利用したい

このようなニーズに応えるために作ったのが、今回の `Pod` クラスです。

## 1. クラス構造

```typescript:Pod.ts
import { PodTable } from "@utils/types/internal/data";
import Logger from "../common/Logger";
import SessionDB from "@class/db/SessionDB";
import settings from "@utils/constants/settings";
import Remote from "./Remote";

class Pod { ... }

export default new Pod();
```

**📌 ポイント**

> - **PodTable**: Pod 情報を保持する型定義
> - **Logger**: 実行ログの出力に使用
> - **SessionDB**: 現在の環境（本番 or ステージング）を取得
> - **Remote**: SSH 経由でリモートサーバーに接続し、コマンドを実行するクラス
> - インスタンスを即時生成・エクスポートしておくことで、**グローバルに使えるユーティリティとして再利用**できます（シングルトンパターン）。

## 2. Pod 検索

```typescript:Pod.ts
public async searchPods(search: string): Promise<Array<PodTable>> {
  const cmd = `kubectl get pods | grep ${search}`;
  const searchResult = (await Remote.execRemote(cmd)) as string;
  const podLines = searchResult.trim().split("\n");

  const podObjects = await Promise.all(
    podLines.map(async (line) => {
      const [podName, , status] = line.split(/\s+/);
      const podIP = await this.getPodIp(podName);
      return { name: podName || "", status: status || "", ip: podIP || "" };
    })
  );

  Logger.info(`Search result for: ${search}`);
  console.table(podObjects);

  return podObjects;
}
```

**📌 ポイント**

> - 指定文字列を含む Pod を検索し、**名前 / ステータス / IP** を一覧化して返します。
> - `console.table` により視覚的にも分かりやすい形式でログに出力。

## 3. Pod の IP アドレス取得: `getPodIp`

```typescript:Pod.ts
private async getPodIp(podName: string): Promise<string> {
  const cmd = `kubectl get pod ${podName} -o jsonpath='{.status.podIP}'`;
  const podIP = (await this.execRemote(cmd)) as string;
  return podIP;
}
```

**📌 ポイント**

> - 単一 Pod の IP アドレスを `jsonpath` で取得します。

## 4. Pod 取得（サービス別）

```typescript:Pod.ts
public async getPodFooService(): Promise<string[]> {
  const pods = await this.getRunningPod("foo-service");
  return pods.map((pod) => pod.name);
}
```

```typescript:Pod.ts
private async getRunningPod(search: string): Promise<PodTable[]> {
  const pods = await this.searchPods(search);
  const runningPods = pods.filter((pod) => pod.status === "Running");

  if (runningPods.length > 0) {
    Logger.success(`Found running pod(s)`);
    return runningPods;
  } else {
    throw new Error(`No running pod found for keyword: ${search}`);
  }
}
```

**📌 ポイント**

> - 指定サービス名で **"Running" 状態の Pod だけを取得**。
> - 取得対象を明示的にサービス名ごとに分けられます。
> - 実運用では、**異常状態の Pod を除外**して Running のものだけ使いたい場面が多いため、有用なフィルタ関数です。

## おわりに

本記事では、Node.js + TypeScript で**Kubernetes 上の Pod 情報をリモートで取得するクラス**の構築方法を紹介しました。

**✅ 特徴**

> - SSH リモート経由での `kubectl` 実行
> - Pod 検索・IP 取得・稼働状態フィルタの機能分離
> - 環境ごとの Pod 取得に対応した柔軟な設計

**今後の展望（ToDo）**

> - Pod の **ログ取得機能** の追加（`kubectl logs`）
> - **Restart 回数** や **Ready 数の取得** によるヘルスチェックの強化
> - リソース（CPU/Memory）の使用状況取得
> - Pod イベントやエラー検出の統合
