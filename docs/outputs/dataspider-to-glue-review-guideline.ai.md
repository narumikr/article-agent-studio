# DataSpider → AWS Glue 移行コード AIレビュールール

本ファイルは [dataspider-to-glue-review-guideline.md](./dataspider-to-glue-review-guideline.md) から、AIレビュー (Claude Code等) 用にルールだけを抜き出した版です。各ルールの根拠・詳細なNG/OK例・出典は本編を参照してください。

`.claude/rules/glue-migration-review.md` として配置する場合は、先頭に `paths` フロントマター (例: `glue_jobs/**/*.py`) を追加してください。

<context>
対象は DataSpider Servista の「ファイル取得 → 変換 → CSV作成 → S3アップロード」処理を AWS Glue (Python Shell 3.9 または PySpark glueetl) へ移植したコードである。
入力CSVは CP932 (Shift_JIS系) と UTF-8 が混在する。
移行元の DataSpider は GUI 設定と既定値で多くの処理を暗黙的に行っていたため、Python では明示が必要になる。
既定値は DataSpider のバージョンと稼働OSで異なる (例: CSV書き込みのエンコード既定は 4.2 SP8 ではOS依存、5.0.2 では UTF-8)。
</context>

<instructions>
1. 差分の中で、ジョブ種別 (Python Shell / Spark) と実行環境のバージョンを最初に特定する。Python Shell (library-set=analytics) は Python 3.9 / pandas 1.4.2 / boto3 1.21.21 である。library-set=none は boto3 が同梱一覧に無い。Glue 5.0 の botocore は 1.34.131 である。
2. 下記ルールに照らして指摘する。指摘には必ずルールID・重大度・`file:line` の引用を付ける。
3. 開いていないコードや、命名からの推測で指摘しない。DataSpider の旧設定値がPRから読み取れない場合は断定せず「要確認」として質問にとどめる。
4. ruff / mypy で検出できる項目 (bare except、print、f-string ログ等) は、CIで有効になっている場合は報告しない。
5. 出力の冒頭に「Must n / Should n / Nice n」を1行で書き、Must → Should → Nice の順に並べる。Nice は最大5件とする。
</instructions>

<severity>
- Must: パリティ破壊・データ欠損・障害検知不能・セキュリティ。マージ前に修正が必要。
- Should: 保守性・運用性・将来の事故要因。理由があれば見送り可。
- Nice: 改善提案。
</severity>

## 初級ルール

| ID | 重大度 | 検出条件 | 理由 |
|---|---|---|---|
| ENC-001 | Must | `encoding="shift_jis"/"sjis"`、Spark の `encoding=Shift_JIS` | ①㈱髙～∥ 等で UnicodeError。`cp932` / Spark は `windows-31j` を使う |
| ENC-002 | Must | `errors="ignore"/"replace"`、`encoding_errors=` | 文字化けが無言で出力される。ignore はバイト境界がずれて別文字になる |
| ENC-003 | Should | UTF-8 入力を `utf-8` で読み列名アクセスしている | BOM 付きで1列目が `\ufeffid` になる。`utf-8-sig` を使う |
| CSV-001 | Must | CSV の `open()` に `newline=""` か `encoding=` が無い | クォート内改行が変わる、ロケール依存になる |
| CSV-002 | Must | `split(",")`、`splitlines()`、`iter_lines()`、`",".join()` でCSVを処理 | RFC 4180 のクォート・エスケープを処理できない |
| PD-001 | Must | `read_csv` に `dtype=str` と `keep_default_na=False` (または `na_filter=False`) が無い | 先頭ゼロ消失、19桁の桁落ち、"NA"/"NULL" が NaN になる。`dtype=str` だけでは不十分 |
| PD-002 | Must | `to_csv` に `index=False` が無い | 行番号列が出力される |
| SPK-001 | Must | Spark CSV 読み込みに `header`/`encoding`/`multiLine`/`escape='"'` の明示が無い、`inferSchema=True` | 既定 escape は `\`、multiLine=false でレコード分割、inferSchema で `001`→`1` |
| GLU-001 | Must | DynamicFrame の `optimizePerformance: True` | SIMD CSV reader は日本語などマルチバイト文字に非対応と公式に明記 |
| NUM-001 | Must | 金額・数量で `round()`、`float()`、float リテラル演算 | Python の round は偶数丸め + float 誤差。DataSpider「四捨五入」は HALF_UP 相当 (負数の .5 は旧出力で要確認)。`Decimal(str)` + `ROUND_HALF_UP` |
| ERR-001 | Must | `except: pass`、`except Exception: print/log` で終了、ループ内 `continue` 後に raise しない | ジョブが SUCCEEDED になり失敗通知が飛ばない |
| ERR-002 | Should | ジョブ本体の `print`、except 内の `logger.error(str(e))` | スタックトレースが失われる。`logger.exception` を使う |
| CFG-001 | Should | バケット名・プレフィックス・環境名・列番号のリテラル、ハイフン付き引数名、引数値を真偽値として直接判定 | ジョブ引数化する。引数値は常に文字列 |
| SEC-001 | Must | `aws_access_key_id=`、パスワード等のリテラル、パスワードのジョブ引数 | ジョブ引数はログに残りうると公式に明記。ジョブロール + Secrets Manager |
| S3-001 | Must | `list_objects_v2` の単発呼び出し、`resp["Contents"]` の無条件参照 | 1,000件で打ち切り、0件で KeyError。paginator を使う |
| S3-002 | Should | サイズ想定なしの `Body.read()` + `decode()` + `splitlines()` | 0.0625 DPU (推測で約1GB) で OOM。`/tmp` (約14GiB) 経由で逐次処理 |
| SRC-001 | Must | `C:\`、`\\server\share`、オンプレパス | Glue からは読めない |
| LOG-001 | Must | 行データ・`df.show()` をログ出力 | CloudWatch Logs に個人情報が残る |
| PAR-001 | Must | 旧出力と新出力の比較結果 (ハッシュ一致または差分分類と合意) がPRに無い | パリティ未確認のままマージされる |

<example type="NG" rule="PD-003,PD-002">
s3.put_object(Bucket=b, Key=k, Body=df.to_csv(encoding="cp932").encode())
</example>
<example type="OK" rule="PD-003,PD-002">
buf = io.StringIO(newline="")
df.to_csv(buf, index=False)
s3.put_object(Bucket=b, Key=k, Body=buf.getvalue().encode("cp932"))
</example>

<example type="NG" rule="NUM-001">
tax = round(float(row["amount"]) * 0.1)
</example>
<example type="OK" rule="NUM-001">
tax = (Decimal(row["amount"]) * Decimal("0.1")).quantize(Decimal("1"), rounding=ROUND_HALF_UP)
</example>

<example type="NG" rule="ERR-001">
try:
    main()
except Exception as e:
    print(e)
</example>
<example type="OK" rule="ERR-001">
try:
    main()
except Exception:
    logger.exception("job failed")
    raise
</example>

## 中級ルール

| ID | 重大度 | 検出条件 | 理由 |
|---|---|---|---|
| CSV-003 | Must | 出力改行 (`lineterminator`/`line_terminator`/`lineSep`) が根拠なく既定値 | DataSpider は AUTO=OS依存 (Windows なら CRLF)。csv は `\r\n`、pandas は `os.linesep` (Glue では LF)、Spark は LF |
| CSV-004 | Must | UTF-8 出力の `utf-8`/`utf-8-sig` 選択に根拠が無い | DataSpider は `x-UTF-8-BOM` 指定時のみ BOM |
| CSV-005 | Must | ヘッダ出力・読み込みが既定値任せ | DataSpider 書き込みは列名挿入 OFF が既定、DynamicFrame `writeHeader` は既定 true |
| CSV-006 | Must | 読み込み後に `''`/`None`/`NaN` が混在、Spark で null と空文字を区別して出力 | DataSpider は 空→null で読み、null→空文字で書く。Spark は null を `,,`、空文字を `,"",` |
| CSV-007 | Should | 列単位の常時クォート設定があったのに `QUOTE_MINIMAL` のまま | DataSpider は列単位でクォート設定。csv モジュールに列単位指定は無い |
| CSV-010 | Must | アダプタ設定の実値 (バージョン・OS・エンコード・改行・ヘッダ・上書き・トランザクション・トリガー終了時オプション) がPRや設計書に無い | 既定値はバージョンとOSで変わる |
| ENC-004 | Must | Spark で非UTF-8を読むのに U+FFFD 件数チェック等の検知が無い | Spark (Java) は不正バイトを置換文字にしてエラーにしない |
| ENC-005 | Should | CP932 ↔ Unicode 往復があるのに文字単位の突合が無い | `0x8160` は shift_jis で U+301C、cp932 で U+FF5E。`‖` は cp932 往復で `∥` になる |
| PD-003 | Must | `to_csv(encoding=...)` をパス未指定やテキストバッファで使用 | str が返り encoding は無効。後続の `.encode()` で UTF-8 になる |
| PY-001 | Must | Python Shell 向けで `X \| None` 型ヒント (future import 無し)、`match` 文、pandas `lineterminator=`、`dtype_backend`。Python Shell・Glue 5.0 での boto3 `IfNoneMatch` | Python 3.9 / pandas 1.4.2 で動かない。`IfNoneMatch` は botocore 1.35.2 以上が必要で、古い版では `ParamValidationError` (ClientError ではない) になる |
| SPK-002 | Should | `mode` が PERMISSIVE のまま破損レコード件数を確認しない | 壊れた行が null 化され黙って出力される。`FAILFAST` 推奨 |
| SPK-003 | Must | Spark CSV 書き込みに `escape='"'`、`ignoreLeadingWhiteSpace=false`、`ignoreTrailingWhiteSpace=false` が無い | 既定で `\"` エスケープ (RFC 4180 非準拠)、前後空白が削られる |
| SPK-004 | Must | Spark 出力でファイル名指定できる前提、引数由来パスへの `mode("overwrite")`、大データへの `coalesce(1)` | part ファイルになる。overwrite は既存出力を削除する。一時プレフィックス + copy で最終キー化 |
| DYF-001 | Must | CP932 を `create_dynamic_frame.from_options(format="csv")` で直接読む | format_options に encoding が無い。`spark.read.option("encoding","windows-31j")` + `DynamicFrame.fromDF` |
| NUM-002 | Should | Mapper「数値フォーマッティング」「割り算」の移植で丸めモード・精度の根拠が無い | 数値フォーマッティングは HALF_EVEN、割り算は既定で1番目の入力の小数精度 |
| STR-001 | Should | Mapper「トリム」を引数なし `strip()` で置換 | DataSpider は対象文字を個別指定。strip() は全角スペースも除去する |
| DATE-001 | Must | `datetime.now()`、`date.today()`、Spark `current_date()` を TZ 指定なしで使用 | 移行元は JST 前提と推測。Glue の TZ は未確認で9時間ずれの恐れ。`ZoneInfo("Asia/Tokyo")` |
| DATE-002 | Must | Java 書式 `yyyy/MM/dd` を `strptime` に渡す、Spark `dateFormat` 省略 | Python は `%Y/%m/%d`、Spark 既定は `yyyy-MM-dd` |
| VAR-001 | Should | スクリプト変数相当の値が None になりうるのに扱いが無い | DataSpider は null を型デフォルト値 (空文字/0/false) に暗黙変換 |
| ERR-003 | Must | 例外監視の置換箇所で「捕捉後に正常終了」か「異常終了」かの根拠が無い | DataSpider の例外監視は捕捉後に正常終了しトランザクションもコミットされる。終了ステータス値を後続が参照していた可能性 |
| TX-001 | Must | 変換しながら S3 へ逐次書き込み、例外後も出力処理に進む経路がある | DataSpider のトランザクション有効時は失敗時に出力先が更新されない。`/tmp` に書き切ってからアップロード |
| FILE-001 | Should | ファイル操作を S3 API に置換したが存在チェックが無い | DataSpider は既定で「既存ならエラー」「無ければエラー」。S3 は既定で上書き |
| CFG-002 | Must | ジョブ定義に `Timeout`/`MaxRetries`/`MaxConcurrentRuns`/`GlueVersion`/`PythonVersion` の明示が無い | Timeout 既定は Glue 4.0 以前 2,880分、5.0 以降 480分。タイムアウトはリトライされない |
| CFG-003 | Should | `--debug`/`--mode`/`--JOB_NAME`/`--endpoint` と衝突、任意引数を必須リストに含める | 予約名。getResolvedOptions に任意引数のサポート記載なし |
| IAM-001 | Must | `s3:*`、`Resource: "*"`、AmazonS3FullAccess、KMS 権限漏れ | AWSGlueServiceRole は `aws-glue-*` バケットのみ。プレフィックス限定で付与 |
| MON-001 | Should | FAILED/TIMEOUT 通知ルール、ログ保持期間が IaC に無い | `Glue Job State Change` イベントで通知する |
| LOG-002 | Should | 開始・終了・入出力キー・件数・処理時間のログが無い | DataSpider はサーバー側で自動記録していた |
| TEST-001 | Should | 変換ロジックがトップレベル直書き、テストが実S3にアクセス | 純粋関数化して pytest、S3 は moto 等でモック |
| DEP-001 | Should | `--additional-python-modules` がバージョン未固定、共通処理のコピペ | Glue 5.0 以降は venv (`--python-virtual-env`) が公式推奨。実行時の PyPI 解決は highly risky とされ、次善策は zip of wheels |
| PAR-002 | Should | パリティ検証データに境界値が無い | 空ファイル、null/空文字、"NA"、先頭ゼロ、セル内改行、`""`、①髙～𠮷、.5 境界、月末・うるう年 |

<example type="NG" rule="SPK-001,SPK-003">
df = spark.read.csv(path, header=True, inferSchema=True)
df.write.csv(out, header=True)
</example>
<example type="OK" rule="SPK-001,SPK-003">
df = (spark.read.option("header", "true").option("encoding", "windows-31j")
      .option("multiLine", "true").option("escape", '"').option("mode", "FAILFAST")
      .schema(schema).csv(path))
(df.repartition(1).write.mode("errorifexists").option("header", "true")
   .option("escape", '"').option("ignoreLeadingWhiteSpace", "false")
   .option("ignoreTrailingWhiteSpace", "false").csv(tmp_uri))
</example>

<example type="NG" rule="DATE-001">
fname = f"out_{datetime.now():%Y%m%d}.csv"
</example>
<example type="OK" rule="DATE-001">
fname = f"out_{datetime.now(ZoneInfo('Asia/Tokyo')):%Y%m%d}.csv"
</example>

## 上級ルール

| ID | 重大度 | 検出条件 | 理由 |
|---|---|---|---|
| ARCH-001 | Should | ジョブ種別の選定理由が無い、Spark ジョブの本体が pandas/boto3 のみ | Spark は最小2ワーカーで、Python Shell 相当の処理では1ワーカーしか使われずコスト増と公式に明記 |
| ARCH-002 | Must | DataSpiderServer ローカルから S3 への転送方式・転送完了判定が設計に無い | Glue はローカルディスクを読めない。転送途中のファイルを処理する恐れ |
| IDEM-001 | Must | 出力キーに実行時刻、`MaxRetries > 0` で非冪等、処理済み入力の管理なし | Python Shell はブックマーク不可。決定的出力キー + `IfNoneMatch="*"` (Glue 5.1 以上か boto3>=1.35.2) + 処理済み移動。409 はリトライ対象 |
| IDEM-002 | Must | EventBridge 起動で重複起動対策が無い、ジョブ内ポーリング (`while True` + `sleep`) | Glue は EventBridge の配信保証も重複排除もしないと公式に明記 |
| TRG-001 | Must | 旧ファイルトリガーの終了時オプション (入力ファイル削除) 相当の処理が無い | 設定がスクリプト本体に現れず移植漏れしやすい |
| TX-002 | Should | 複数ファイルを1セットで渡すのに完了マーカーが無い | S3 はキーをまたいだアトミック更新ができない |
| ENC-006 | Should | UTF-8 → CP932 経路で CP932 に無い文字 (𠮷・絵文字) の扱いが仕様化されていない | 明示的な置換表と置換件数ログを用意し、表外の文字は失敗させる |
| VER-001 | Should | GlueVersion 変更・Glue 6.0 採用で挙動差の検証が無い | Spark 3.3 で日付変換方式変更、Spark 4.0 で ANSI 既定 ON、Glue 6.0 で EMRFS 削除 |
| SEC-002 | Should | 出力バケットに TLS 強制・KMS キー指定が無い、`--TempDir` 未指定 | バケットポリシーで `aws:SecureTransport` と KMS キーを強制する |
| COST-001 | Nice | Python Shell 1 DPU、Spark ワーカー数が既定や過大、1ファイル1ジョブ実行 | 0.0625 DPU と 1 DPU で単価16倍。Auto Scaling / Flex / バッチングを検討 |
| PAR-003 | Should | 並行稼働期間・合格基準・切り戻し手順が無い | AWS は安定稼働を確認してから旧環境を廃止することを推奨 |

<example type="NG" rule="IDEM-001">
out_key = f"converted/{datetime.now():%Y%m%d%H%M%S}_{name}"
</example>
<example type="OK" rule="IDEM-001">
# Glue 5.1 以上、または boto3>=1.35.2 が前提
out_key = output_key_for(in_key, head["ETag"], out_prefix)
s3.put_object(Bucket=b, Key=out_key, Body=f, IfNoneMatch="*")
</example>

## 指摘の出力形式

<output_format>
Must 2 / Should 1 / Nice 0

[Must] ENC-001 jobs/csv_convert.py:42
`text = body.decode("shift_jis")`
CP932 由来の入力で ① や 髙 を含むと UnicodeDecodeError になります。`cp932` を指定してください。

[要確認] CSV-003 jobs/csv_convert.py:88
改行コードが既定値 (pandas は Glue 上で LF) です。旧 DataSpiderServer の OS と改行コード設定を棚卸し表に記載してください。
</output_format>

## 参考リンク

- [本編: DataSpider → AWS Glue (Python) 移行コード レビューガイドライン](./dataspider-to-glue-review-guideline.md)
- [Claude Code - Memory (CLAUDE.md / rules)](https://code.claude.com/docs/en/memory)
- [Claude Code - Code Review](https://code.claude.com/docs/en/code-review)
- [Prompting best practices](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/claude-prompting-best-practices)
- [Using the CSV format in AWS Glue](https://docs.aws.amazon.com/glue/latest/dg/aws-glue-programming-etl-format-csv-home.html)
- [Python shell jobs in AWS Glue](https://docs.aws.amazon.com/glue/latest/dg/add-job-python.html)
- [Spark CSV Files (3.5.6)](https://spark.apache.org/docs/3.5.6/sql-data-sources-csv.html)
- [CSVファイル書き込み処理 (DataSpider 5.0.2)](https://www.hulft.com/help/ja-jp/DataSpider/dss502/help/ja/adapter/file/csv_put_data.html)

最終更新日: 2026/10/09
