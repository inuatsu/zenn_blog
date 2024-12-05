---
title: ダミーの法人番号を生成する仕組みを作ろう
emoji: "\U0001F0CF"
type: tech
topics:
  - Python
  - Faker
published: true
published_at: "2023-12-13 07:00"
publication_name: simpleform_blog
---

エンジニアリングマネージャの犬束 ([@sekainoinuatsu](https://twitter.com/sekainoinuatsu)) です。
本記事は [SimpleForm Advent Calendar 2023](https://qiita.com/advent-calendar/2023/simpleform) の 13 日目の記事です。

当社では法人調査の情報基盤を作るため、法人に関する様々な情報を日々収集しています。
日本では、国税庁が全ての法人に 13 桁の法人番号を付与しており[^1]、当社が法人の情報を収集して紐付ける上で積極的に法人番号を活用しています。

[^1]: 法人番号については、国税庁が公開している[法人番号公表サイト](https://www.houjin-bangou.nta.go.jp/setsumei/)で詳しい説明があるのでご参照ください

この 13 桁の番号は、完全にランダムに決められている数字というわけではなく、一定の規則があります。
実装においては、法人番号の規則性を利用するケースもあります。
そのため、自動テストを行う際は、完全にランダムな 13 桁の数字の文字列を法人番号として生成する方法では困る場合があります。

そこで、法人番号の生成ルールに則ったダミーの法人番号を生成する仕組みを作りたくなったため、作ってみました。

# 前提知識

今回は Python で [Faker](https://github.com/joke2k/faker) というライブラリを使ってダミーの法人番号を生成する仕組みを実装する方法を紹介します。
Faker 自体の使い方に関しては丁寧な説明をしないので、利用されたことがない場合は他の記事などで基本的な使い方を学んだ上で、当記事を読んでいただくと良いかと思います。

# 法人番号の採番ルール

法人番号の採番ルールに関しては、法人番号公表サイトで公開されている PDF の資料[^2]に記載されています。
これによれば、13 桁目[^3]はチェックデジットになっており、他の桁の数字により自動的に定まるようです。

なお、チェックデジットを除いた 12 桁の数字は会社法人等番号とも呼ばれ、この数字は法人番号制度が開始[^4]される以前から存在していました。
この 12 桁の数字の採番ルールを見ていきましょう。

会社法などの法令の規定により設立の登記が必要な法人のことを「設立登記法人」[^5]と言いますが、設立登記法人の場合は 12 桁目〜 9 桁目の 4 桁は登記所コード、8 桁目〜 7 桁目の 2 桁は組織区分、6 桁目〜 1 桁目の 6 桁は一連番号 (設立登記の際、登記所で登記記録を起こした順に付ける番号) となっています。
登記所コードは、全国の全ての登記所に割り振られているコードです。全ての登記所の登記所コードは[登記・供託オンライン申請システム](https://www.touki-kyoutaku-online.moj.go.jp/toukinet/mock/SC01WS01.html)のページから調べられそうです。
組織区分は、「01」が株式会社、「02」が特例有限会社[^6]、「03」が合名会社・合資会社・合同会社、「05」はその他の法人を指します。

国の機関や地方公共団体は別の採番ルールとなっています。12 桁目〜 9 桁目の 4 桁は「0000」、8 桁目〜 7 桁目の 2 桁は「11」「12」「13」「20」「30」のいずれか、6 桁目〜 1 桁目の 6 桁は一連番号のようです。

設立登記のない法人、人格のない社団等はまた別の規則があり、法人番号公表サイトで公開されている資料によると 12 桁目は「7」、11 桁目〜 1 桁目は一連番号となっています。ただ、筆者が調べたところでは 12 桁目が 7 の法人は全て 12 桁目〜 7 桁目の 6 桁が「700150」となっており、6 桁目〜 1 桁目の 6 桁が一連番号となっているようです。

さて、採番ルールが分かればあとは実装するだけです。

[^2]: <https://www.houjin-bangou.nta.go.jp/setsumei/pamphlet/images/houjinbangou_gaiyou.pdf> の 2 ページを参照

[^3]: 本記事では、国税庁の資料に則って末尾の数字から 1 桁目、2 桁目、... と数えます。したがって、13 桁目は先頭の数字です

[^4]: ちなみに、法人番号制度が開始されたのは 2015 年 10 月です

[^5]: 株式会社、合名会社、合資会社、合同会社、一般社団法人、一般財団法人、学校法人、宗教法人、税理士法人、医療法人、特定非営利活動法人など、多くの法人が含まれます。逆に、設立登記法人ではない法人とは、税法上所定の届出書を提出することが求められ当該届出書に基づき法人番号が指定される法人や、国税庁への届出書に基づいて法人番号が指定される法人などです。具体的には、健康保険組合、企業年金基金、外国法人などが含まれます

[^6]: 2006 年に会社法が施行される以前に有限会社であった会社で、経過措置として有限会社として存続している会社のこと。会社法設立以前は有限会社法を根拠に設立が認められていましたが、会社法設立に伴い有限会社法が廃止され、現在は新設はできません

# faker でダミーデータを生成する仕組み

faker では住所、氏名、会社名など様々なダミーデータを生成できますが、これはそれぞれのダミーの値を生成するコードが provider としてパッケージングされており、provider を使うことによって実現しています。

provider の仕組みを理解するために、少し faker のコードを見てみます。provider の理解に必要な箇所だけ抜き出すと、以下のようなディレクトリ構造になっています。

```text
faker
  |-faker
  |   |-providers
  |   |     |-address
  |   |     |-automotive
  |   |     |-....
  |   |     |-job
  |   |     |  |-ar_AA
  |   |     |  |-az_AZ
  |   |     |  |-...
  |   |     |  |- ja_JP
  |   |     |  |   `-__init__.py
  |   |     |  |-...
  |   |     |  `-__init__.py
  |   |     |-...
```

job の provider が実装がシンプルで説明しやすいので、これを例に解説します。
job ディレクトリ配下には `__init__.py` と様々な言語のローカライズ用のディレクトリが存在しています。
`__init__.py` の中身は以下のようになっています。

```py
from .. import BaseProvider, ElementsType

localized = True


class Provider(BaseProvider):
    jobs: ElementsType[str] = (
        "Academic librarian",
        "Accommodation manager",
        ...
        "Youth worker",
    )

    def job(self) -> str:
        return self.random_element(self.jobs)
```

BaseProvider を継承して Provider クラスが定義されており、Provider クラスのアトリビュート jobs 内には職業の選択肢が羅列されています。Provider クラスの job メソッドで、jobs に定義された職業からランダムで値を返すようになっています。

おおよその実装が把握できたところで、実際にダミーの法人番号を生成する仕組みを実装しましょう。

# ダミーの法人番号を生成する仕組み

法人番号は大きく分けて 3 通りの採番ルールがあることを先ほど述べました。

1. 設立登記法人 : チェックデジット 1 桁 + 登記所コード 4 桁 + 法人種別 2 桁 (01, 02, 03, 05 いずれか) + 一連番号 6 桁
1. 国の機関または地方公共団体 : チェックデジット 1 桁 + 0000 + 法人種別 2 桁 (11, 12, 13, 20, 30 いずれか) + 一連番号 6 桁
1. 設立登記のない法人、人格のない社団等 : チェックデジット 1 桁 + 700150 + 一連番号 6 桁

チェックデジットを計算するところは共通化できそうなので、他の箇所に関して考えましょう。登記所コード、設立登記法人の場合の法人種別、国の機関または地方公共団体の場合の法人種別については決められた値からランダムで出せるようにする必要があるので、そこを定義します。また、一連番号はランダムな 6 桁の数字として定義することにします。

```py
class CorporateNumberProvider(BaseProvider):
    government_institution_codes = ("11", "12", "13", "20", "30")

    registry_office_codes = (
        "4300",
        "4301",
        "4302",
        "4304",
        "4306",
        ...
    )

    company_type_codes = ("01", "02", "03", "05")

    registration_order_formats: ElementsType[str] = ("######",)

```

上で定義した値の候補の中からランダムで返すメソッドも実装します。

```py
class CorporateNumberProvider(BaseProvider):
    ...

    def government_institution_code(self) -> str:
        return self.random_element(self.government_institution_codes)

    def registry_office_code(self) -> str:
        return self.random_element(self.registry_office_codes)

    def company_type_code(self) -> str:
        return self.random_element(self.company_type_codes)

    def registration_order(self) -> str:
        return self.numerify(self.random_element(self.registration_order_formats))
```

さて、ここまで来たら 12 桁の会社法人等番号のパターンを定義してランダムで返すメソッドと、会社法人等番号からチェックデジットを計算して 13 桁の法人番号を返すメソッドを加えれば完成です。

```py
class CorporateNumberProvider(BaseProvider):
    ...

    corporate_registration_number_formats = (
        "{{registry_office_code}}{{company_type_code}}{{registration_order}}",
        "0000{{government_institution_code}}{{registration_order}}",
        "700150{{registration_order}}",
    )

    def corporate_registration_number(self) -> str:
        pattern: str = self.random_element(self.corporate_registration_number_formats)
        return self.generator.parse(pattern)

    def corporate_number(self) -> str:
        corporate_registration_number = self.corporate_registration_number()

        even_sum = (
            int(corporate_registration_number[0])
            + int(corporate_registration_number[2])
            + int(corporate_registration_number[4])
            + int(corporate_registration_number[6])
            + int(corporate_registration_number[8])
            + int(corporate_registration_number[10])
        )
        odd_sum = (
            int(corporate_registration_number[1])
            + int(corporate_registration_number[3])
            + int(corporate_registration_number[5])
            + int(corporate_registration_number[7])
            + int(corporate_registration_number[9])
            + int(corporate_registration_number[11])
        )
        check_digit = 9 - (even_sum * 2 + odd_sum) % 9

        return f"{check_digit}{corporate_registration_number}"
```

コード全体としては以下のようになります。

::: details ダミーの法人番号を返す provider の実装

```py
from faker.providers import BaseProvider, ElementsType


class CorporateNumberProvider(BaseProvider):
    corporate_registration_number_formats = (
        "{{registry_office_code}}{{company_type_code}}{{registration_order}}",
        "0000{{government_institution_code}}{{registration_order}}",
        "700150{{registration_order}}",
    )

    government_institution_codes = ("11", "12", "13", "20", "30")

    registry_office_codes = (
        "4300",
        "4301",
        "4302",
        "4304",
        "4306",
        "4307",
        "4308",
        "4309",
        "4313",
        "4315",
        "4316",
        "4323",
        "4329",
        "4334",
        "4400",
        "4401",
        "4402",
        "4409",
        "4500",
        "4501",
        "4502",
        "4503",
        "4514",
        "4600",
        "4601",
        "4603",
        "4604",
        "4625",
        "3700",
        "3701",
        "3702",
        "3703",
        "3704",
        "3705",
        "3706",
        "3708",
        "3800",
        "3801",
        "3802",
        "3803",
        "3804",
        "3805",
        "3808",
        "3824",
        "3836",
        "3842",
        "3900",
        "3901",
        "3902",
        "3903",
        "3904",
        "3905",
        "3909",
        "4000",
        "4001",
        "4002",
        "4003",
        "4004",
        "4005",
        "4006",
        "4015",
        "4027",
        "4100",
        "4101",
        "4102",
        "4103",
        "4104",
        "4105",
        "4106",
        "4200",
        "4201",
        "4202",
        "4204",
        "4211",
        "4234",
        "0100",
        "0101",
        "0104",
        "0105",
        "0106",
        "0107",
        "0108",
        "0109",
        "0110",
        "0111",
        "0112",
        "0113",
        "0114",
        "0115",
        "0116",
        "0117",
        "0118",
        "0123",
        "0124",
        "0127",
        "0128",
        "0131",
        "0132",
        "0133",
        "0134",
        "0200",
        "0201",
        "0202",
        "0203",
        "0204",
        "0205",
        "0206",
        "0207",
        "0208",
        "0210",
        "0212",
        "0213",
        "0217",
        "0218",
        "0219",
        "0224",
        "0225",
        "0300",
        "0301",
        "0302",
        "0303",
        "0304",
        "0305",
        "0306",
        "0307",
        "0308",
        "0309",
        "0310",
        "0312",
        "0314",
        "0315",
        "0316",
        "0318",
        "0320",
        "0322",
        "0328",
        "0334",
        "0400",
        "0401",
        "0402",
        "0403",
        "0404",
        "0405",
        "0406",
        "0407",
        "0408",
        "0410",
        "0414",
        "0416",
        "0423",
        "0424",
        "0426",
        "0443",
        "0447",
        "0500",
        "0501",
        "0502",
        "0503",
        "0504",
        "0506",
        "0524",
        "0527",
        "0530",
        "0532",
        "0539",
        "0600",
        "0601",
        "0602",
        "0603",
        "0604",
        "0605",
        "0608",
        "0624",
        "0627",
        "0700",
        "0701",
        "0702",
        "0703",
        "0704",
        "0705",
        "0706",
        "0707",
        "0708",
        "0800",
        "0801",
        "0802",
        "0803",
        "0804",
        "0805",
        "0806",
        "0808",
        "0809",
        "0810",
        "0816",
        "0820",
        "0828",
        "0831",
        "0900",
        "0901",
        "0903",
        "0908",
        "0917",
        "0918",
        "1000",
        "1001",
        "1002",
        "1003",
        "1004",
        "1005",
        "1006",
        "1007",
        "1008",
        "1009",
        "1100",
        "1101",
        "1102",
        "1103",
        "1104",
        "1105",
        "1106",
        "1107",
        "1108",
        "1109",
        "1113",
        "1131",
        "1800",
        "1801",
        "1802",
        "1803",
        "1804",
        "1805",
        "1806",
        "1811",
        "1814",
        "1827",
        "1831",
        "1836",
        "1840",
        "1844",
        "1900",
        "1901",
        "1902",
        "1903",
        "1904",
        "1905",
        "1906",
        "1923",
        "1934",
        "2000",
        "2001",
        "2002",
        "2004",
        "2005",
        "2017",
        "2030",
        "2035",
        "2100",
        "2101",
        "2103",
        "2104",
        "2200",
        "2201",
        "2202",
        "2203",
        "2209",
        "2300",
        "2301",
        "2302",
        "2303",
        "1200",
        "1201",
        "1202",
        "1203",
        "1206",
        "1207",
        "1209",
        "1211",
        "1215",
        "1216",
        "1220",
        "1223",
        "1300",
        "1301",
        "1302",
        "1303",
        "1304",
        "1305",
        "1308",
        "1310",
        "1312",
        "1314",
        "1328",
        "1400",
        "1401",
        "1402",
        "1403",
        "1405",
        "1406",
        "1407",
        "1408",
        "1409",
        "1410",
        "1411",
        "1412",
        "1413",
        "1415",
        "1420",
        "1437",
        "1443",
        "1500",
        "1501",
        "1503",
        "1510",
        "1512",
        "1600",
        "1601",
        "1602",
        "1603",
        "1605",
        "1606",
        "1608",
        "1617",
        "1700",
        "1702",
        "1703",
        "1704",
        "1708",
        "1711",
        "1716",
        "2400",
        "2401",
        "2402",
        "2403",
        "2404",
        "2405",
        "2407",
        "2409",
        "2416",
        "2420",
        "2434",
        "2500",
        "2501",
        "2502",
        "2503",
        "2504",
        "2505",
        "2525",
        "2600",
        "2602",
        "2603",
        "2605",
        "2607",
        "2612",
        "2619",
        "2644",
        "2700",
        "2701",
        "2702",
        "2800",
        "2801",
        "2802",
        "2803",
        "2804",
        "2806",
        "4700",
        "4701",
        "4702",
        "4705",
        "4708",
        "4800",
        "4801",
        "4802",
        "4900",
        "4902",
        "4903",
        "4904",
        "4906",
        "4913",
        "5000",
        "5001",
        "5002",
        "5003",
        "5004",
        "5005",
        "5009",
        "5022",
        "2900",
        "2901",
        "2902",
        "2903",
        "2904",
        "2905",
        "2906",
        "2907",
        "2908",
        "2909",
        "2910",
        "2911",
        "2912",
        "2914",
        "2915",
        "2917",
        "2935",
        "3000",
        "3001",
        "3002",
        "3003",
        "3011",
        "3100",
        "3102",
        "3103",
        "3104",
        "3105",
        "3106",
        "3107",
        "3117",
        "3200",
        "3201",
        "3202",
        "3203",
        "3204",
        "3205",
        "3207",
        "3208",
        "3213",
        "3231",
        "3300",
        "3302",
        "3303",
        "3304",
        "3305",
        "3306",
        "3307",
        "3308",
        "3311",
        "3315",
        "3400",
        "3402",
        "3403",
        "3404",
        "3405",
        "3409",
        "3411",
        "3416",
        "3422",
        "3431",
        "3442",
        "3450",
        "3453",
        "3455",
        "3500",
        "3501",
        "3502",
        "3503",
        "3507",
        "3523",
        "3600",
        "3601",
        "3602",
        "3603",
        "3604",
        "3608",
    )

    company_type_codes = ("01", "02", "03", "05")

    registration_order_formats: ElementsType[str] = ("######",)

    def government_institution_code(self) -> str:
        return self.random_element(self.government_institution_codes)

    def registry_office_code(self) -> str:
        return self.random_element(self.registry_office_codes)

    def company_type_code(self) -> str:
        return self.random_element(self.company_type_codes)

    def registration_order(self) -> str:
        return self.numerify(self.random_element(self.registration_order_formats))

    def corporate_registration_number(self) -> str:
        pattern: str = self.random_element(self.corporate_registration_number_formats)
        return self.generator.parse(pattern)

    def corporate_number(self) -> str:
        corporate_registration_number = self.corporate_registration_number()

        even_sum = (
            int(corporate_registration_number[0])
            + int(corporate_registration_number[2])
            + int(corporate_registration_number[4])
            + int(corporate_registration_number[6])
            + int(corporate_registration_number[8])
            + int(corporate_registration_number[10])
        )
        odd_sum = (
            int(corporate_registration_number[1])
            + int(corporate_registration_number[3])
            + int(corporate_registration_number[5])
            + int(corporate_registration_number[7])
            + int(corporate_registration_number[9])
            + int(corporate_registration_number[11])
        )
        check_digit = 9 - (even_sum * 2 + odd_sum) % 9

        return f"{check_digit}{corporate_registration_number}"
```

:::

以下のように、実際にダミー (だがそれっぽい) の法人番号が返ってくることを確認できます。

```py
>>> from faker import Faker
>>> fake = Faker()
>>> fake.add_provider(CorporateNumberProvider)
>>> fake.corporate_number()
'3280105264187'
>>> fake.corporate_number()
'4000030514502'
>>> fake.corporate_number()
'1000020292995'
>>> fake.corporate_number()
'9700150185181'
>>> fake.corporate_number()
'3700150890892'
>>> fake.corporate_number()
'7291501829494'
>>>
```

ダミーのデータが作りやすくなることで、自動テストの実装がより一層捗りそうです。

アドベントカレンダーも折返し地点に差し掛かりましたが、明日以降も面白い記事がたくさん出てくると思いますので、引き続きお楽しみください！
