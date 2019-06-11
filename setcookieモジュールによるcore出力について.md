# setcookieモジュールによるcore出力について
2019年6月11日
株式会社アベリオシステムズ

## coreを gdb でバックトレース

スタックトレースの解析結果は以下である。

1. トレース出力のパターンはいくつかに分類できる。
2. 発生しているエラーはいくつかに分類できる。
3. いずれも不正アドレスを参照してエラーとなっている。
4. 1についてはapr_pcallocでのパターンが多いが結果のため、何が原因であるかはログ、再現検証等からは不明である。

* gdb出力: [core/gdbout](file:///material/core/gdbout/)
* コア一覧: [error_log_list.xlsx](file:///material/error_log_list.xlsx)

## 環境の確認

検証環境では再現されず、本番環境で再現するため、環境を比較した。

* 検証環境で internavi/base 以下のファイルのチェックサムを取り、本番環境で md5sum -c を行った。
  + チェックサム: [checksum.txt](file:///material/checksum.txt)
  + 検証結果: [checksum_result.txt](file:///material/checksum_result.txt)

## 負荷をかけて再現を試行

access_log を1分ごとに集計し、core出力の前後3分間のアクセス状況を確認すると、coreを吐いた頃にピークがある観測が得られた（最多 1189 件）。

* 1分ごとのアクセス数を集計

```
for f in htc_web1_logs/access_log.*.zip; do unzip -p $f | awk '{ sub("^\\[","",$5); sub(":[0-9][0-9]$","",$5); print $5 }'; done | sort | uniq -c > access_log_aggregation.txt
```

* [coreの日時](file:///material/core_0529-0602_ls-ltr.txt) で集計結果を grep

```
for t in `cat core_0529-0602_ls-ltr.txt | perl -ne '$f = ""; while (<>) { @F = split(" "); $tmp = sprintf("%02d/%s/2019:%s\n",$F[6],$F[5],$F[7]); if ($f ne $tmp) { $f = $tmp ; } else { next; } print $f }'`; do grep `echo $t` access_log_aggregation.txt; done
    618 29/May/2019:21:21
    713 29/May/2019:23:17
    346 29/May/2019:23:18
    340 31/May/2019:21:46
    290 01/Jun/2019:08:19
    716 01/Jun/2019:17:41
   1189 01/Jun/2019:18:14
    880 02/Jun/2019:08:57
    799 02/Jun/2019:08:59
```

* core出力の前後3分間のアクセス状況
  [accesses_around_coredump.txt](file:///material/accesses_around_coredump.txt)

検証環境で、負荷をかけて再現を試みた。

* テスト会員が13人ほどの状態で同時リクエストをしたが、coreを吐くことはなかった。
* 次に、検証環境に有効な会員が560人いることがわかったので、20グループに分割して、次々に 10URLほどにリクエストを送るスクリプトをバックグラウンドで実行させてみた。
* 結果、やはりcoreは吐かなかった。
* 12:47:17から12:58:40 までの約1分20秒ほどで6500件ほどだったので、それなりに負荷はかかったと推測できる。
* 負荷によってcoreを吐く再現はできないと思われる。

```
[sync-se@ver-htc-web1 ~]$ grep '\[06/Jun/2019:12:\(48:1[789]\|49:[0-5][0-9]\|5[0-8]:[0-3][0-9]\|58:40\)' /usr/local/internavi/base/httpd/logs/access_log | wc -l
6508
```

## データを確認

coreを吐くとき error_log に Backtrace が出力されるケースがあることがわかった。
* [access_log_backtrace_time.txt](file:///material/access_log_backtrace_time.txt)

その時刻と一致する access_log により、会員を特定したところ、5会員にしぼれることがわかった。

```
$ awk -F'People=' '$2 !~ /^$/ { sub(";.*","",$2); print $2 }' access_log_backtrace_time.txt | sort -u
2d9efb83e770846c9e3018fa2992d26e
5efd7bf89b9f2e7b23cd6542a300032e
94c6cb950d6d689fb6452211b6aca001
9a9673808c7d8431eda92e01b5cb7506
ad3182e02f9a9e10175a45e13db9740d
```

この5会員の会員データを確認した。

- DB
  - car_navi_master
  - dealer_info
  - owner_car_master
  - person_car
  - premium_car
  - user_car
  - user_dealer
  - user_main
  - user_maint_rec
  - user_maintenance
  - user_person
  - user_service
- API
  - crpf
  - rule_contents

特に問題点となるようなデータはないようだ。データが原因であれば、これ以外のなにかのはずだが、問題点に行き着くには膨大な時間が必要と思われる。
