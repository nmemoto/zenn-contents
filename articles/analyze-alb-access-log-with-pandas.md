---
title: "AWS ALBのアクセスログをPandasで集計する"
emoji: "🐙"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["AWS", "ALB", "Pandas", "Python"]
published: true
---

AWS ALBのアクセスログをPandasを使い、ALBが返したレスポンスのステータスコード別分間件数、ALBの背後にあるターゲットグループがALBに返したステータスコード別分間件数、ターゲットグループの処理時間の分間パーセンタイル値を集計するためのPythonスクリプトを作成しました。

# Pythonスクリプトの説明

この集計を行うために作成したスクリプトを分割して説明します。

## ログをデータフレームに変換

以下のスクリプトでログをデータフレームとしました。なお、集計したい複数のアクセスログは予めスクリプト内の`dir_path`に格納している前提としています。

ALBのアクセスログの項目一覧は以下にあり、データフレーム化する際の項目名はこれを使用しました。
https://docs.aws.amazon.com/ja_jp/elasticloadbalancing/latest/application/load-balancer-access-logs.html#access-log-entry-format

```python
from pathlib import Path

import pandas as pd

dir_path = Path("./alb_access_log/raw")
output_dir_path = "./alb_access_log/processed"
file_list = dir_path.glob("*.log")
# %% ログのデータフレーム化
df = pd.DataFrame()
for file in file_list:
    df_file = pd.read_csv(
        file,
        delim_whitespace=True,  # スペース区切り
        header=None,
        index_col='time',       # timeをindexにするとresampleが使える
        parse_dates=True,       # indexをparseする
        names=(
            "type",
            "time",
            "elb",
            "client:port",
            "target:port",
            "request_processing_time",
            "target_processing_time",
            "response_processing_time",
            "elb_status_code",
            "target_status_code",
            "received_bytes",
            "sent_bytes",
            "request",
            "user_agent",
            "ssl_cipher",
            "ssl_protocol",
            "target_group_arn",
            "trace_id",
            "domain_name",
            "chosen_cert_arn",
            "matched_rule_priority",
            "request_creation_time",
            "actions_executed",
            "redirect_url",
            "error_reason",
            "target:port_list",
            "target_status_code_list",
            "classification",
            "classification_reason"
        ),
        dtype={
            "request_processing_time": float,
            "target_processing_time": float,
            "response_processing_time": float,
            "elb_status_code": str,
            "target_status_code": str,
        },
        na_values='-'       # '-'をnaとみなす
        )
    df = pd.concat([df, df_file, axis=0)
```

以降の処理で[pandas.DataFrame.resample](https://pandas.pydata.org/docs/reference/api/pandas.DataFrame.resample.html)を使用しますが、そのためにデータフレームのindexにログの記録日時である"time"を指定し、これを`parse_dates=True`でdatetime型にパースするようにしました。

## ALBが返したレスポンスのステータスコード別分間件数

以下の処理で、ALBが返したレスポンスのステータスコード別分間件数のCSVを出力しています。

```python
df_alb_status_all = pd.DataFrame()
alb_status_code_list = df['elb_status_code'].unique()
for v in alb_status_code_list:
    df_alb_status = df[(df['elb_status_code'] == v)].resample('1T').count().rename(columns={'type': v})[v]
    df_alb_status_all = pd.concat([df_alb_status_all, df_alb_status], axis=1).fillna(0)
df_alb_status_all_sorted = df_alb_status_all.sort_index(axis='columns').sort_index(axis='index')
df_alb_status_all_sorted.to_csv(f'{output_dir_path}/elb_status_code.csv')
```

これによって作成されるcsvは以下です。

```csv:elb_status_code.csv
,200,201,400,401,403,460,500,502,503,504
2022-09-XX 1X:00:00+00:00,23520,1295.0,85,28.0,0.0,8139.0,0.0,46.0,0.0,0.0
2022-09-XX 1X:01:00+00:00,16018,1324.0,295,5.0,0.0,19144.0,1.0,3240.0,1803.0,0.0
2022-09-XX 1X:02:00+00:00,14296,1056.0,564,8.0,0.0,10972.0,0.0,721.0,19255.0,0.0
2022-09-XX 1X:03:00+00:00,8994,360.0,405,5.0,0.0,11514.0,0.0,1424.0,27202.0,2.0
2022-09-XX 1X:04:00+00:00,5540,289.0,369,5.0,0.0,3912.0,0.0,0.0,47958.0,1.0
・・・・
```

この処理の肝は[`pandas.Dataframe.resample`](https://pandas.pydata.org/docs/reference/api/pandas.DataFrame.resample.html)で、引数に`1T`と指定すると1分毎の集計が可能となり、その後ろの`count()`で件数を集計できます。


## ターゲットグループがALBに返したステータスコード別分間件数

以下の処理で、そのCSVを出力しています。

```python
df_target_status_all = pd.DataFrame()
target_status_code_list = df['target_status_code'].unique()
for v in target_status_code_list:
    df_status = pd.DataFrame()
    if pd.isnull(v):
        df_status = df[(df['target_status_code'].isnull())].resample('1T').count().rename(columns={'type': '-'})['-']
    else:
        df_status = df[(df['target_status_code'] == v)].resample('1T').count().rename(columns={'type': v})[v]
    df_target_status_all = pd.concat([df_target_status_all, df_status], axis=1).fillna(0)
df_target_status_all_sorted = df_target_status_all.sort_index(axis='columns').sort_index(axis='index')
df_target_status_all_sorted.to_csv(f'{output_dir_path}/target_status_code.csv')
```

これによって作成されるcsvは以下です。

```csv:target_status_code.csv
,-,200,201,400,401,403,500,502,503
2022-09-XX 1X:00:00+00:00,4543.0,22939,1222.0,98.0,28.0,0.0,0.0,612.0,0.0
2022-09-XX 1X:01:00+00:00,10169.0,21049,2035.0,1195.0,18.0,0.0,1.0,1547.0,0.0
2022-09-XX 1X:02:00+00:00,7650.0,13202,1173.0,1061.0,6.0,0.0,0.0,3597.0,0.0
2022-09-XX 1X:03:00+00:00,12276.0,8150,351.0,389.0,7.0,0.0,0.0,2049.0,3551.0
2022-09-XX 1X:04:00+00:00,10652.0,9952,534.0,438.0,6.0,0.0,0.0,2983.0,0.0
```

ほとんどALBが返したレスポンスのステータスコード別分間件数を出したときと処理が同じです。
target_status_codeの場合、ターゲットグループからのレスポンスを返していない場合ログに`"-"`が出力されるのですが、今回はログをデータフレームに変えるときに、`"-"`をNAとみなすように`pandas.read_csv`のオプションで`na_values='-'`を指定していました。
上記処理内の変数`target_status_code_list`にはNAが入っているのですが、`df[(df['target_status_code'] == NA)]`という処理を行うことができないため、NAを判定する分岐(`if pd.isnull(v)`)を入れていました。
(最初からNAに変換せずに`"-"`のまま処理してたら、この分岐いらんのではというセルフツッコミ)

`pandas.isnull()`については以下を参照のこと。`pandas.isna()`のaliasとのこと。
https://pandas.pydata.org/docs/reference/api/pandas.isnull.html

`pandas.DataFrame.isnull()`については以下を参照のこと。こちらも`pandas.DataFrame.isna()`のaliasとのこと。
https://pandas.pydata.org/docs/reference/api/pandas.DataFrame.isnull.html


## ターゲットグループの処理時間の分間パーセンタイル値

以下の処理で、ログに出力されている処理時間('target_processing_time', 'request_processing_time', 'response_processing_time')のパーセンタイル値を含むCSVを出力しています。

```python
df_percetile_all = pd.DataFrame()
percentile_list=[0.5, 0.9, 0.95, 0.99, 1]
metrics_list=['target_processing_time', 'request_processing_time', 'response_processing_time']
for m in metrics_list:
    df_percetile_all = pd.DataFrame()
    for v in percentile_list:
        df_percetile = df.resample('1T')[m].quantile(v).rename(v)
        df_percetile_all = pd.concat([df_percetile_all, df_percetile], axis=1)
    df_percetile_all.to_csv(f'{output_dir_path}/{m}.csv')
```

これによって作成されるcsvの一つは以下です。

```csv:target_processing_time.csv
,0.5,0.9,0.95,0.99,1.0
 2022-09-XX 1X:00:00+00:00,0.639,19.158400000000007,25.250649999999982,28.753,28.999
 2022-09-XX 1X:01:00+00:00,0.104,22.991100000000003,28.2,28.886,29.0
 2022-09-XX 1X:02:00+00:00,0.792,21.015200000000004,27.0022,28.885,28.999
 2022-09-XX 1X:03:00+00:00,0.001,13.5946,20.051,28.878,28.999
 2022-09-XX 1X:04:00+00:00,0.032,7.197200000000026,27.414999999999974,28.92236,28.999
```

[`pandas.core.resample.Resampler.quantile`](https://pandas.pydata.org/docs/reference/api/pandas.core.resample.Resampler.quantile.html)を使用しています。

# まとめ

アクセスログの集計に`Pandas`の[`pandas.Dataframe.resample`](https://pandas.pydata.org/docs/reference/api/pandas.DataFrame.resample.html)を使うと、分間の件数やパーセンタイル値の算出が簡単にできる。
