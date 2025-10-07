# ===== 非交互切到 WORKGROUP，清掉域成员状态 =====
$ErrorActionPreference = 'Stop'
# 有些环境中直接切组即可，不必先 Remove-Computer
Add-Computer -WorkGroup 'WORKGROUP' -Force -ErrorAction Stop -Confirm:$false
Restart-Computer -Force


这个就可以吗（机器已经被ad删掉）



Remove-Computer -UnjoinDomainCredential "AWS-PVT.IT\webadmin" -Force
Add-Computer -WorkGroup "WORKGROUP"
Restart-Computer -Force


ENDPOINT="<your-elasticache-endpoint>"
PORT=6379

echo "1) slowlog の閾値を一時的に 1 µs に変更"
redis-cli -h $ENDPOINT -p $PORT CONFIG SET slowlog-log-slower-than 1

echo "2) 遅いクエリを発生させる (DEBUG SLEEP 1)"
redis-cli -h $ENDPOINT -p $PORT DEBUG SLEEP 1

echo "3) slowlog の記録を確認"
redis-cli -h $ENDPOINT -p $PORT SLOWLOG GET 5

echo "4) Engine log エラーを発生させる (FOOBAR)"
redis-cli -h $ENDPOINT -p $PORT FOOBAR || true

echo "5) slowlog の閾値をデフォルト (10000 µs = 10ms) に戻す"
redis-cli -h $ENDPOINT -p $PORT CONFIG SET slowlog-log-slower-than 10000

echo "6) 閾値が戻ったことを確認"
redis-cli -h $ENDPOINT -p $PORT CONFIG GET slowlog-log-slower-than


Start-Process msiexec.exe -ArgumentList '/i "C:\Windows\Temp\package.msi" /qn /norestart /L*v "C:\Windows\Temp\install.log"' -Wait



皆さん、連絡が遅くなってすみません。
TechDayの新人セッションについて、会話ベースで以下を確認したいと思っています：
	1.	発表形式（準備した内容を発表する／自由対話）
	2.	各自が話したい内容（概要でOK）
	3.	発表時に必要な他メンバーからのサポート（例：同チームとの対話、ファシリ対応 など）

私が議事メモのようにまとめ、鬼島さんへ要約を提出します。
急ですが、本日16–17時に時間を確保しました（早く終われば解散）。都合が合わない方はこのスレで教えてください。可能な限り調整します。

事前に「1～2点、自分が話したい内容」を軽くイメージしておいてもらえるとありがたいです。例：
	•	話したい内容：配属後のコミットメント理解
	•	具体例：配属前はXXXXXと思っていたが、配属後に〇〇を経験し、XXXと感じた

また、AI関連を担当している皆さんは、もし良いネタがあればこのスレに投げてもらえると参考になります。
どうぞよろしくお願いします！
