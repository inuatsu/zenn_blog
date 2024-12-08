---
title: Lambda が timeout する際に graceful に落としたい日々だった
emoji: ⌛
type: tech
topics:
  - Python
  - AWS
  - Lambda
published: true
published_at: "2023-12-03 07:00"
publication_name: simpleform_blog
---

エンジニアリングマネージャーの[犬束](https://twitter.com/sekainoinuatsu)です。
本記事は [SimpleForm Advent Calendar 2023](https://qiita.com/advent-calendar/2023/simpleform) の 3 日目の記事です。

当社では AWS Lambda を多く活用しています。
今回は、Lambda を運用する中で当社において活用している Tips を紹介します。

## 前提知識

Python ランタイムの Lambda を活用したことがある方向けの記事です。Lambda のしくみやデプロイ方法などは説明しないので、Lambda を活用されたことがない方は別記事で調べていただけますと幸いです。

## Lambda を使っていて困ること

Lambda にはタイムアウトが存在します。タイムアウトまでの時間は最大 15 分 (900 秒) まで 1 秒単位で調整できますが、Lambda の処理実行中にタイムアウト時間が到来すると、以下のようなログを吐いてスパッと処理が停止されます。

```sh:Lambda のタイムアウト発生時のログ例
2023-10-01T00:00:00.000Z 3a1e28ee-19ba-3ae8-3af1-2a2ae2cf1a2b Task timed out after 300.49 seconds
```

Lambda を実装する上では冪等性が重要で、このようにタイムアウトが発生したとしても次に開始される処理には影響がないようにすることが求められます。
が、場合によっては上記のタイムアウト時の仕様が困ったことになる場合があります。
当社で Lambda を使った処理を実装している上でも、Lambda 内の特定の処理が完了しない内にタイムアウトしてしまうことで、運用上想定していない挙動をしてしまうことがあり頭を悩ませていました。

## Lambda を graceful にシャットダウンさせたい

Python にはデコレータ[^1]という概念があり、これを使うことで関数の前後に任意の処理を追加できます。
Lambda のハンドラ関数にデコレータを適用することで、Lambda がタイムアウトする際に必要な処理をさせた上で graceful に落とすことができるのではないか、というのが今回試したことです。

[^1]: <https://docs.python.org/ja/3/glossary.html#term-decorator>

Lambda で関数が実行されると、context オブジェクトがハンドラに渡されます。
context オブジェクトは関数の実行などに関するいくつかのメソッドやプロパティを持っています。
この中に実行がタイムアウトするまでの残り時間をミリ秒で返す `get_remaining_time_in_millis` というメソッドがあります[^2]。
デコレータでこのメソッドを使って残り時間を取得することで、Lambda がタイムアウトする前に処理をとめて任意の処理をさせることができそうです。

[^2]: <https://docs.aws.amazon.com/ja_jp/lambda/latest/dg/python-context.html>

実際に以下のようなデコレータを用意しました。

```py:lambda_timeout_handler.py
from __future__ import annotations

import signal
from functools import wraps
from typing import Any, Callable

from aws_lambda_powertools import Logger
from aws_lambda_powertools.utilities.typing import LambdaContext

logger = Logger()


def timeout_handler(_signal, _frame):
    raise TimeoutError


def lambda_timeout_handler(lambda_handler: Callable[[dict, Any], Any]) -> Callable:
    @wraps(lambda_handler)
    def wrapper(event: dict[str, Any], context: LambdaContext, **kwargs):
        signal.signal(signal.SIGALRM, timeout_handler)
        try:
            # Lambda のタイムアウト時に Graceful にシャットダウンできるよう、Lambda 関数のタイムアウトの 3 秒前にシグナルを送る
            signal.alarm(int(context.get_remaining_time_in_millis() / 1000) - 3)
            result = lambda_handler(event, context)
        except Exception:
            logger.exception("An error occurred during the timeout process")
            result = None
        signal.alarm(0)
        return result

    return wrapper
```

上では Lambda 関数のタイムアウトの 3 秒前にシグナルを送るようにしていますが、この数字は必要に応じて適宜変更してください。

このデコレータを以下のように Lambda のハンドラ関数に付けることで、Lambda のタイムアウト時間が到来する際に必ず特定の処理をしてから終了するようにできます。
デコレータを使わない場合、Lambda がタイムアウト時間に到達しても finally 句内の処理は実行されません。

```py :main.py
from __future__ import annotations

from aws_lambda_powertools import Logger
from aws_lambda_powertools.utilities.typing import LambdaContext

from lambda_timeout_handler import lambda_timeout_handler

logger = Logger()


@logger.inject_lambda_context
@lambda_timeout_handler
def handler(event: dict[str, Any], context: LambdaContext):
    try:
        # Lambda で実行したい処理を実装
    except TimeoutError:
        logger.exception("Lambda function timed out")
    finally:
        # Lambda を終了する前に実行したい処理を実装
    return
```

上記のハンドラ関数を Lambda で実行すると、タイムアウト時間が到来した場合に以下のようなログが出て終了するはずです。
finally 句内でログを出せば、以下の処理の前に finally 句内の処理が実行されることもわかります。

```json
{
  "level": "ERROR",
  "location": "handler:xxx",
  "message": "Lambda function timed out",
  "timestamp": "2023-10-01 00:00:00.000+0900"
}
```

Lambda 関数を graceful にシャットダウンできることで、さらに Lambda 活用が捗りそうです。

<!-- textlint-disable ja-technical-writing/no-exclamation-question-mark -->

明日は中山さんによる、「Lambda の同時実行数でハマった話」の予定です。こちらもお楽しみに！

<!-- textlint-enable ja-technical-writing/no-exclamation-question-mark -->
