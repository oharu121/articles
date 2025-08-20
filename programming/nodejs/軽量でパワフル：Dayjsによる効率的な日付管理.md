## 軽量でパワフル：Dayjs による効率的な日付管理

## はじめに

日付と時刻の操作は、Web 開発において一般的かつ重要なタスクです。JavaScript の標準 Date オブジェクトは使いづらいと感じる人も多く、これを解決するためのライブラリがいくつか存在します。

Dayjs は、そのようなライブラリの中でも特に軽量でありながら、必要十分な機能を提供することで人気を博しています。

**📅 Dayjs の特徴**

> - 1.  軽量性: Dayjs は非常に小さいサイズで、パフォーマンスに影響を与えることなくプロジェクトに組み込むことができます
> - 2.  使いやすさ: Dayjs は一連の操作はチェインできます
> - 3.  不変性: Dayjs は操作を行っても元のインスタンスを変更せず、新しいインスタンスを返すため、バグの発生を防ぎやすいです
> - 4.  拡張性: プラグインを通じて機能を拡張でき、必要な機能を柔軟に追加することができます
> - 5.  国際化対応: 多言語に対応しており、グローバルなプロジェクトにも適しています

## 1. 日付の作成、フォーマット変更

```typescript:example.ts
import dayjs from "dayjs";

const date = dayjs().format("YYYY/MM/DD");
console.log(date); // 2024/04/08

const date2 = dayjs("1995-12-01").format("YYYY/MM/DD HH:mm:ss");
console.log(date2); // 1995/12/01 00:00:00

const date3 = dayjs("2022/04").format("YYYY/MM/DD hh:mm:ss A");
console.log(date3); // 2022/04/01 12:00:00 AM

const isoString = dayjs().toISOString();
console.log(isoString); // 2024-04-08T06:10:26.303Z

const timeStamp = dayjs().valueOf();
console.log(timeStamp); // 1712556626303
```

フォーマット一覧は下記です：

https://day.js.org/docs/en/display/format

## 2. 日付の加算、減算、調整

```typescript:example.ts
import dayjs from "dayjs";

const date1 = dayjs("2024-04-08").add(7, "day").format("YYYY-MM-DD");
console.log(date1); // 2024-04-15

const date2 = dayjs("2024-04-08").subtract(1, "year").format("YYYY-MM-DD");
console.log(date2); // 2023-04-08

const date3 = dayjs("2024-04-08").startOf("year").format("YYYY-MM-DD");
console.log(date3); // 2024-01-01

const date4 = dayjs("2024-04-08").endOf("month").format("YYYY-MM-DD");
console.log(date4); // 2024-04-30
```

## 3. 日付の比較

```typescript:example.ts
import dayjs from "dayjs";

const today = dayjs();
const date2 = dayjs("2018-06-05");

const diffMilliSecond = today.diff(date2);
console.log(diffMilliSecond); // 184434897202

const diffDay = today.diff(date2, "day");
console.log(diffDay); // 2134

const diffMonth = today.diff(date2, "month");
console.log(diffMonth); // 70

const diffYear = today.diff(date2, "year", true);
console.log(diffYear); // 5.843180543172541

const compare1 = today.isBefore(date2);
console.log(compare1); // false

const compare2 = today.isAfter(date2);
console.log(compare2); // true

const compare3 = dayjs("2024-04-15").isSame("2024-04-30", "month");
console.log(compare3); // true
```

## 4. タイムゾーンに基づいた日付表示

```typescript:example.ts
import dayjs from "dayjs";
import utc from "dayjs/plugin/utc";
import timezone from "dayjs/plugin/timezone";

// プラグインを適用
dayjs.extend(utc);
dayjs.extend(timezone);

// ユーザーのタイムゾーンを設定（この例では自動的に検出）
dayjs.tz.setDefault();

// 日付をフォーマットして表示（例: "YYYY-MM-DD HH:mm:ss"）
const currentTimeZone = dayjs().tz().format("YYYY-MM-DD HH:mm:ss");
console.log(`日本時間：${currentTimeZone}`); // 日本時間：2024-04-08 16:35:02

const taiwanTime = dayjs().tz("Asia/Taipei").format("YYYY-MM-DD HH:mm:ss");
console.log(`台湾時間：${taiwanTime}`); // 台湾時間：2024-04-08 15:35:02
```

## 5. 日付の判定

```typescript:example.ts
import dayjs from "dayjs";
import isToday from "dayjs/plugin/isToday";
import isTomorrow from "dayjs/plugin/isTomorrow";
import isYesterday from "dayjs/plugin/isYesterday";
import isLeapYear from "dayjs/plugin/isLeapYear";

dayjs.extend(isToday);
dayjs.extend(isTomorrow);
dayjs.extend(isYesterday);
dayjs.extend(isLeapYear);

const result1 = dayjs("2000-01-01").isLeapYear();
console.log(result1); // true

const result2 = dayjs().isToday();
console.log(result2); // true

const result3 = dayjs().add(1, "day").isTomorrow();
console.log(result3); // true

const result4 = dayjs().add(-1, "day").isYesterday();
console.log(result4); // true
```

## まとめ

Dayjs は、その軽量さと簡単な API、拡張性の高さから、多くの JavaScript プロジェクトで日付と時刻の操作に選ばれています。Dayjs を使うことで、開発者はより読みやすく、保守しやすいコードを書くことができます。
