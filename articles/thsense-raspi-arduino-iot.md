---
title: "【個人開発】Raspberry Pi × Arduinoで、危険な温度をメールで知らせる見守りシステム"
emoji: "🌡️"
type: "tech"
topics: ["raspberrypi", "arduino", "python", "flask", "iot"]
published: true
---

## はじめに

暑い夏が続くなか、一人暮らしをしている高齢の方が、部屋の気温が上がっていることに気づかないまま熱中症で救急搬送されるケースが増えています。

そこで、**離れた場所で暮らす家族を介護している人**に向けて、部屋の温度・湿度を見守り、危険な温度になったらメールで知らせるシステム「THSense」を個人開発しました。この記事では、その構成と実装を紹介します。

- 部屋の温度・湿度を1分ごとに測る
- 35℃を超えたら、メールで知らせる
- ブラウザで、今の温度と過去の推移をグラフで確認できる

グラフで推移を見られるようにしたのは、「温度が上がり続けている」といった危険の予兆に、早めに気づけるようにするためです。

ソースコードはGitHubで公開しています。

https://github.com/Taito1608/THsense_system

## 完成したもの

Arduino（Grove Beginner Kit）で温度・湿度を測り、USBでつないだRaspberry Piにデータを送ります。Arduino側の小さな画面にも、測った値を表示しています。

![Raspberry PiとArduinoをUSBで接続した実機](/images/thsense-raspi-arduino-iot/device.jpg)

ブラウザでは、今の温度と、温度・湿度の推移を確認できます。

![現在の温度を表示する画面](/images/thsense-raspi-arduino-iot/screen-now.png =400x)

![温度と湿度の推移を表示するグラフの画面](/images/thsense-raspi-arduino-iot/screen-graph.png =400x)

35℃を超えると、次のようなメールが届きます。

![基準外の温度を知らせる通知メール](/images/thsense-raspi-arduino-iot/mail.png)

## システム構成

![システム構成図](/images/thsense-raspi-arduino-iot/system.png)

| 部分 | 役割 |
| --- | --- |
| Arduino | 温湿度センサー（DHT20）から値を読み取り、USBシリアル通信でRaspberry Piに送る |
| Raspberry Pi（`sensor.py`） | 受け取った値を判定し、35℃を超えていたらメールを送る。値をデータベースに保存する |
| データベース | Azure上の仮想マシンに構築したMariaDB |
| Webサーバー（`server.py`） | データベースから値を取り出し、ブラウザに表示する |

使った技術は次のとおりです。

| 分類 | 技術 |
| --- | --- |
| ハードウェア | Raspberry Pi（Raspberry Pi OS）、Arduino、DHT20 |
| フロントエンド | HTML、CSS、JavaScript（Chart.js） |
| バックエンド | Python、Flask、SQLAlchemy |
| データベース | MariaDB（Azure上の仮想マシン） |

### なぜArduinoを使ったのか

センサーの制御は、Raspberry Piではなく、あえてArduinoで行っています。Arduinoはリアルタイムな制御に向いているため、DHT20の読み取りをArduinoに任せることで、より正確なデータを取得することを目的としました。Raspberry Piには、データの判定・保存・通知といった処理を担当させています。

## 実装

### 1. Arduino：センサーの値をシリアル通信で送る

Arduinoでは、DHT20から温度・湿度を読み取り、決まった形式の文字列にしてUSBシリアル通信でRaspberry Piに送ります。送信は60秒ごとです。

```cpp:data_send_to_raspi.ino
temp = float( dht.readTemperature() );
humi = float( dht.readHumidity() );

if( sendState == 1 ) {
  Serial.print("Temperature=");
  Serial.print(temp);
  Serial.print(":");
  Serial.print("Humidity=");
  Serial.print(humi);
  Serial.print(":");
  // ...
  Serial.println("");
}
```

送られる文字列は `Temperature=28.40:Humidity=55.10:...` のような形です。`:` で区切っておくと、受け取る側で分けやすくなります。

また、ボタンを押すと送信を止めたり再開したりできます。今の状態は、LEDとArduinoの画面の「SENDING / STOPPED」でわかるようにしました。

### 2. Raspberry Pi：受け取った文字列を数値に戻す

Raspberry Pi側では `pyserial` で受け取り、`:` と `=` で分けて数値に変換します。

```python:function_pkg/data_receive_from_arduino.py
import serial

ser = serial.Serial('/dev/ttyUSB0', 9600)

def read_usb():
  in_txt = ser.readline()
  while ser.in_waiting:
    in_txt = in_txt + ser.readline()

  # 受信したデータを分割
  txt1 = in_txt.split(b':')
  txt2 = txt1[0].split(b'Temperature=')
  txt3 = txt1[1].split(b'Humidity=')

  # 文字列から数値に変換
  temp = float( txt2[1] )
  humid = float( txt3[1] )

  return temp, humid
```

:::message
`/dev/ttyUSB0` は環境によって変わります。Arduinoをつないだ状態で `ls /dev/tty*` を実行し、つなぐ前後で増えたものを確認してください。
:::

### 3. 温度をチェックして、メールで知らせる

受け取った温度が基準を超えていたらメールを送り、そのあと値をデータベースに保存します。

```python:sensor.py
def main():
    while True:
        # Arduinoからデータを受信
        temp, humid = data_receive_from_arduino.read_usb()

        # 温度のチェック
        if data_check.check_temperature(temp):
            send_mail.send_mail()  # メール送信

        # データベースに保存
        data_save_db.put_data_record(temp, humid)
```

メールはPython標準の `smtplib` で、GmailのSMTPサーバーから送っています。アカウントやパスワードはコードに書かず、`.env` ファイルから読み込むようにしました。

```python:function_pkg/send_mail.py
msg = MIMEText(message, "html")
msg["Subject"] = "【重要】THSenseシステムからの通知"
msg["To"] = to_email
msg["From"] = from_email

server = smtplib.SMTP("smtp.gmail.com", 587)
server.starttls()
server.login(account, password)
server.send_message(msg)
server.quit()
```

:::message alert
Gmailから送る場合、普段のログインパスワードは使えません。Googleアカウントで2段階認証を有効にし、「アプリパスワード」を発行して使います。
:::

### 4. データベースに保存する

データベースは、Azure上に仮想マシンを作り、そこにMariaDBをインストールして用意しました。テーブルは次のように設計しています。

| カラム | 型 | 内容 |
| --- | --- | --- |
| `id` | varchar(8) | 測定した機器のホスト名 |
| `dt` | datetime | 測定した日時 |
| `temp` | float | 温度 |
| `humid` | float | 湿度 |

Python側では、SQLAlchemyでこのテーブルを定義しています。

```python:db/models.py
class TH_tbl(Base):
    __tablename__ = 'TH_tbl'
    id = Column(String(8), primary_key=True)
    dt = Column(TIMESTAMP, primary_key=True)
    temp = Column(Float, nullable=False)
    humid = Column(Float, nullable=False)
```

### 5. Flaskで、今の温度とグラフを表示する

画面は2つです。トップページに最新の温度、`/hour` に過去60件分（1分ごとなので約1時間分）のグラフを表示します。

```python:server.py
@app.route("/hour", methods=["GET"])
def display_hourdata():
    hour_data = get_hour_data()  # 新しい順に60件

    graph_data = [
        [dt.strftime("%Y %m/%d %H:%M"), temp, hum]
        for _, dt, temp, hum in hour_data
    ]
    return render_template("hour.html", graph_data=graph_data)
```

グラフはChart.jsで描いています。温度（℃）と湿度（%）は単位が違うので、左右に別々の縦軸を用意しました。また、データは新しい順に取り出しているため、表示する前に並びを反転して、最新の値が右端に来るようにしています。

```javascript:templates/hour.html（抜粋）
graphData = graphData.reverse(); // 最新のデータを右端に

new Chart(ctx, {
  // ...
  options: {
    scales: {
      'y-temp': { position: 'left',  title: { display: true, text: '温度 (℃)' } },
      'y-hum':  { position: 'right', title: { display: true, text: '湿度 (%)' } },
    },
  },
});
```

## 工夫したところ

### 機能ごとにファイルを分け、1つずつテストできるようにした

受信・判定・メール送信・保存・取得を、それぞれ `function_pkg/` の別ファイルに分けました。各ファイルに `if __name__ == '__main__':` のテスト用コードを入れているので、例えばメール送信だけ、データベースへの保存だけを単体で試せます。

```python:function_pkg/data_check.py
if __name__ == '__main__':
    temp = float(input("温度を入力してください: "))
    if check_temperature(temp):
        print('【危険】基準外の温度です。')
    else:
        print('温度は基準内です。')
```

センサー・ネットワーク・データベースが関わるシステムは、うまく動かないときに原因がどこにあるのかわかりにくくなります。機能ごとに動作を確認しながら組み立てることで、問題の切り分けがしやすくなりました。

### SQLAlchemyで、データベースの種類に依存しないコードにした

データベースの操作には、SQLAlchemyを使いました。SQLを直接書くのではなく、PythonのコードからSQLを組み立てる仕組みなので、データベースの種類に依存しないコードを書けます。将来、MariaDB以外のデータベースに移すことになっても、コードを大きく変えずに済むようにしています。

## 今後の改善

- **メールが送られ続けてしまう**：35℃を超えている間は、測るたびにメールが送られます。「30分に1回まで」のように、送信の間隔を設定できるようにしたいです。
- **画面が自動で更新されない**：今はページを開き直さないと新しい値が表示されません。データが変わったら、自動で画面も変わるようにしたいです。

## おわりに

センサーの読み取りから、判定・通知・保存・表示まで、IoTシステムの基本的な流れを一通り作ることができました。離れて暮らす家族の見守りのように、身近な課題をIoTで解決したい方の参考になればうれしいです。
