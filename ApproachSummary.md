# SIGNATE mynavi 2019 まとめ
yCarbon (2019/11/08)

注意：自分は後半はベースラインを上げられず、最終サブはアンサンブルしないものだったのでweight 0でした。


## 1. チームメインアプローチ (LB:20th PB:10th) ~ 13000+
### 1. 前処理：19000
- チームマージ前なので詳しくは知りません。
- 自分のアプローチは後述します。

### 2. チームマージ（yCarbon）：18000
- チームマージしました。
- 1. 前処理し終えたデータで機械的に特徴量を3000個くらい作成
- 2. LightGBMで500roundsのみ回帰
- 3. 上記モデルの重要度上位250個で回帰

### 3. チームマージアンサンブル：17000
- 上記のサブと、既存のチームサブでアンサンブルしました。
    - 単純な平均です。
- 相関は0.99くらいでしたが、割とスコアは上昇しました。
    - 前処理とモデルが異なっていたことが原因かと思います。

### 4. 誤った値（特に賃料・面積）の修正: 16000
- データについて、賃料や面積が一桁ずれているものがありました。
- 修正することで、シングルモデルで16000が出ました。

### 5. チームマージ（copasta）：13500
- 地価（＝賃料/面積）へ回帰する＋重要そうな特徴量作成でこのスコアを出していました。めっちゃ強い。

### 6. 地価を追加・機械的に特徴量作成: 13000
- 外部データとして地価がOKだったので、
- 1. 上記のcopastaベースで地価を追加
- 2. 機械的に特徴量を2000個くらい作成
- 3. LightGBMでEarly Stoppingするまで回帰
- 4. 上記モデルの重要度上位100個、200個、300個で回帰・アンサンブル

- 以下は、最終モデルを作成したチームメイト(hiromu : https://twitter.com/hiromu30694350 )さんのメッセージです。
'''<br>
自分の取り組みは簡単にまとめると、copastaさんのベストに以下のアプローチを加えたものになります。<br>
1.地価情報を住所と文字列的に近いものを対象にして結合する(何丁目までしかないものは複数件、類似度が高いものが出てくるので平均をとる)<br>
2.東京のランドマーク的な建物(六本木ヒルズ、東京タワーなど)からの距離を加える(主要５区のみ)<br>
2-2.東京からの距離と標高で土地の安全性に関する特徴をつくる<br>
3.各区ごとの大使館と私立高校の数を加える<br>
4.出店戦略のサイトから重要そうな情報をいれてみる<br>
5.区と地域で集計とる<br>
6.lightgbmを複数回使用して重要な特徴のみで学習<br>
'''


## 1.5. 自分のアプローチ
### 前処理
- 誤った値を修正
- 区切れそうなところで区切って、文字・数字を抽出

### 特徴量作成
- Count Encoding
- mean, max, min, std等のグループ統計量を作成。
- グループ統計量との差・比率を作成
- 座標データを用いて、物件データの平均・平均との差（最近傍4件・10件・20件）を作成
- 地価データを用いて、地価（最近傍2点・8点・16点の平均）を作成

### モデル
- 地価（＝賃料/面積）への回帰
- KFold:4, rounds=500程度でLightGBMにつっこむ
- 重要度が平均以上のもののみで再度LightGBM(Kold:4, iters=10000)
- パラメータはKernel0のものからNum_leaves:255→63, max_bin:255→63, min_child_in_leaf:defalut→31です。
    - 過学習気味だったので、正規化強めにしました。
    - 最適なパラメータは同じデータでもFoldによって変わるので、細かいチューニングはしてません。

### スコア
- 15000前後


## 2. 試したこと
### 2値変数のみで線形回帰しスタッキング
- 前処理後、大量に2値変数が出てきました。
    - エアコンがある・ない、バストイレ別である・ない等
- そのまま使ってもあまり重要度が高くないので、線形回帰した予測値をLightGBMの特徴量にしようとしました。
    - Leakしました。
    - OOF使いましょう。

### 23区毎にモデル作成
- 23区ごとにモデルを作成しました。
    - 4foldなので23x4=92個のモデルを作成しました。
- そのままだとLeakするので、住所＋経過年数で同一の建物を特定し、GroupKFoldしても微妙にLeakしました。
- 実行時間が長く、実験の管理が大変でした。
    - 管理しきれず何回か作ったり壊したりしました。
- CVは7000～30000(港区）で重み付けて合計すると13000くらいでした
    - 捨てきれずに何度か試していました。
- そのままだとデータ数が少ないので、Pseudo Labelingが必要でしたが、Leakする自信があったのでやりませんでした。

### NNで地価の回帰、LightGBMで残差の回帰
- 地価の分布が標準分布っぽかったのでNNを使いたくなりました。
- 精度が今一つだったうえ、Leakしました。

### CatBoost
- Early Stoppingするまでにかなり時間がかかったため、使用しませんでした。

### BERT
- 自然言語処理は門外漢だったため、チームメイトに実験を頼みました。
- 成果は出なかったようです。
- LSTMも試してみたそうですが、うまく学習できなかったそうです。


## 3. 個人的なこと
### コンペについて
- 扱いやすい大きさ、実際に近い汚さ・Leakがあるデータで面白かったです。
- 前処理は比較的得意（統計分析での前処理経験がそのまま使える）だったので序盤は順位が高かったですが、後半は想像通りにボコられました。

### 情報共有について
- クソ雑魚Kernelを上げたので、TLのKagglerなら『大変参考になりました！』とコメントしてきた後、激つよKernel投稿してくるかと期待していました。
    - Validationきちんと切っていなかったのでKagglerだと気づかれなかったのが敗因かな、と思います。
- df['hogehoge']=df['hoge'].apply(lambda x: func(x))を愚直に繰り返せば18000代には届きますが、序盤では18000代が意外と少なかったので情報公開が難しかったです。
    - こんなこと悩む機会はもうないと思うので多分あまり意味ない反省点です。

### 実験の流れについて
- 上手く回せませんでした。反省点です。
- 行ったり来たりして何回か同じことを繰り返しました。
- Leakがかなりの回数発生しました。
    - 基本的なLeakに関してはKaggle本がかなり役立ちました。
        - 残りのLeakについても、本の内容を理解できてないだけで中に記載されていそうです。

### Weight 0
- チームで初めてコンペやりましたが、楽しかったです。
- kaggleで組むのはもうちょい実力の近い人か、実力がついてからにします。
