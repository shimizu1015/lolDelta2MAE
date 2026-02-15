# 📖 Lolalytics 先出しチェッカー 利用ガイドガイド

このドキュメントは、LolalyticsのCounterデータから、試合数による重み付けを行った**平均絶対誤差（MAE）**をDelta2の指標より算出・評価するためのガイドです。

---

## 🚀 1. ツールで何ができるのか

[Lolalytics](https://lolalytics.com/)のサイトを使いながらLOLにおいて特定のチャンピオンが先出しの強さを測る指標を出力します
具体的な内容こちらのサイトご確認ください。　　https://lolninja.net/2026/02/15/49370/
サイト内でも説明がある通り、勝率と試合数をもとにMAEを算出していますが、当プログラムは Lolalytics の各対面ごとに算出されている値（Delta2）を使います

---

## 🛠 2. 実行手順

### Step 1: Lolalyticsで対象とするチャンピオンを開く

[Lolalytics](https://lolalytics.com/)
![lolalytics_ahri](step1.png)

### Step 2: 解析対象のページで「Counter」が見える位置までスクロールをする

目的のチャンピオンページへ行き、画面中央の **「Matchups」** タブをクリックしてください。

```javascript
(async () => {
  // 1. タイトルからレーンを特定
  const rawTitle = document.title;
  const lanes = ["top", "middle", "jungle", "bottom", "support"];
  const activeLane = lanes.find((lane) =>
    rawTitle.toLowerCase().includes(lane),
  );

  if (!activeLane) {
    console.error("タイトルからレーンを判別できませんでした:", rawTitle);
    return;
  }

  // 2. 【改善】全コンテナから対象レーンのデータが入っている「正しい箱」を自動探索
  const allContainers = Array.from(document.querySelectorAll(".cursor-grab"));
  const scrollContainer = allContainers.find((c) => {
    // 箱の中にそのレーン(vslane=...)のリンクが含まれているかチェック
    return c.querySelector(`a[href*="vslane=${activeLane}"]`);
  });

  if (!scrollContainer) {
    console.error(
      `${activeLane.toUpperCase()} 用のデータコンテナが見つかりません。Matchupsタブを確認してください。`,
    );
    return;
  }

  console.log(
    `[START] "${activeLane.toUpperCase()}" レーンの解析を開始します...`,
  );

  let allStats = new Map();

  const fetchData = () => {
    const items = scrollContainer.querySelectorAll(":scope > div > div");
    items.forEach((el) => {
      const aTag = el.querySelector('a[href*="/vs/"]');
      if (!aTag || !aTag.getAttribute("href").includes(`vslane=${activeLane}`))
        return;

      const name = el.querySelector("img")?.alt;
      const divs = Array.from(
        el.querySelectorAll("div.my-1, div.text-\\[9px\\]"),
      );

      if (name && divs.length >= 5) {
        const delta2 = parseFloat(divs[2].innerText.replace(/,/g, ""));
        const games = parseInt(divs[4].innerText.replace(/,/g, ""));

        if (!isNaN(delta2) && !isNaN(games) && games > 0) {
          allStats.set(name, { delta2, games });
        }
      }
    });
  };

  // 3. スクロール実行
  let lastScroll = -1;
  while (true) {
    const beforeScroll = scrollContainer.scrollLeft;
    fetchData();
    scrollContainer.scrollLeft += 800;
    await new Promise((r) => setTimeout(r, 1000));

    if (
      scrollContainer.scrollLeft === beforeScroll ||
      scrollContainer.scrollLeft === lastScroll
    )
      break;
    lastScroll = beforeScroll;
  }

  // 4. 解析データの成形
  const finalArray = Array.from(allStats.entries()).map(([name, data]) => ({
    name,
    delta2: data.delta2,
    games: data.games,
    absDelta: Math.abs(data.delta2),
  }));

  if (finalArray.length === 0) {
    console.warn("データが取得できませんでした。");
    return;
  }

  // 5. 計算 (Weighted MAE)
  const totalGames = finalArray.reduce((acc, curr) => acc + curr.games, 0);
  const weightedSum = finalArray.reduce(
    (acc, curr) => acc + curr.absDelta * curr.games,
    0,
  );
  const weightedMae = weightedSum / totalGames;
  const simpleMae =
    finalArray.reduce((acc, curr) => acc + curr.absDelta, 0) /
    finalArray.length;

  // 6. 結果表示（順序を入れ替え：テーブル → 集計スコア）
  console.clear();
  console.log(
    `--- 対面データ一覧 (${activeLane.toUpperCase()} / 試合数順) ---`,
  );
  console.table(finalArray.sort((a, b) => b.games - a.games));

  console.log(`--- 解析レーン: ${activeLane.toUpperCase()} ---`);
  console.log(`\n--- 重み付け解析まとめ ---`);
  //console.log(`判定タイトル: "${rawTitle}"`);
  console.log(`対面種類数: ${finalArray.length} 体`);
  console.log(`全対面の総試合数: ${totalGames.toLocaleString()} games`);
  console.log(`-----------------------------------`);
  //console.log(`単純平均 MAE: ${simpleMae.toFixed(4)}`);
  console.log(`重み付け MAE: ${weightedMae.toFixed(4)}`);
  console.log(`-----------------------------------`);
})();
```
