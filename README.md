﻿//M5_DE_NAVI

#include <TinyGPSPlus.h>
#include <M5Core2.h>
TinyGPSPlus gps;

static void smartDelay(unsigned long ms);
static void printFloat(float val, bool valid, int len, int prec);
static void printInt(unsigned long val, bool valid, int len); //
static void printDateTime(TinyGPSDate &d, TinyGPSTime &t);
static void printStr(const char *str, int len);

void setup() {
    Serial.begin(115200);
    Serial2.begin(38400, SERIAL_8N1, 13, 14); //pin番号要確認

    Serial.println(); //改行
    Serial.println(F(
        "Sats HDOP  Latitude   Longitude   Fix  Date       Time     Date Alt   "
        " Course Speed Card  Distance Course Card  Chars Sentences Checksum")); //表のタイトルとかの文字列表示
    Serial.println(
        F("           (deg)      (deg)       Age                      Age  (m) "
          "   --- from GPS ----  ---- to London  ----  RX    RX        Fail")); //単位表示
    Serial.println(F(
        "----------------------------------------------------------------------"
        "------------------------------------------------------------------")); //上二つの区切り線
    M5.Lcd.begin(); //core2に画面表示させるときに使うから今回はいらない  ←使うことにした
    delay(10000); //あえて１０秒プログラムを停止させてコードを見れるようにしてるぽい
}


void loop() {
    static const double LONDON_LAT = 51.508131, LONDON_LON = -0.128002; //ロンドンの緯度経度

    printInt(gps.satellites.value(), gps.satellites.isValid(), 5); //有効な衛星の数を表示する
    printFloat(gps.hdop.hdop(), gps.hdop.isValid(), 6, 1); //hdop(水平方向の測定の正確さ)を表示する
    printFloat(gps.location.lat(), gps.location.isValid(), 11, 6);  //緯度を表示する
    printFloat(gps.location.lng(), gps.location.isValid(), 12, 6);  //経度を表示する
    printInt(gps.location.age(), gps.location.isValid(), 5);  //GPS情報の更新からの時間
    printDateTime(gps.date, gps.time);  //今の時間表示
    printFloat(gps.altitude.meters(), gps.altitude.isValid(), 7, 2);  //高度(メートル)を表示
    printFloat(gps.course.deg(), gps.course.isValid(), 7, 2);  //方位を度数で表示
    printFloat(gps.speed.kmph(), gps.speed.isValid(), 6, 2);  //速度を表示
    printStr(
        gps.course.isValid() ? TinyGPSPlus::cardinal(gps.course.deg()) : "*** ",  //方位が正しかどうかを判断。無効なら*。
        6);// ？と：で区切ってif文の代わり。値を返してくれる。

    unsigned long distanceKmToLondon =      //ロンドンから現在地までの距離
        (unsigned long)TinyGPSPlus::distanceBetween(
            gps.location.lat(), gps.location.lng(), LONDON_LAT, LONDON_LON) /
        1000;  //1000で割ってｋｍ表示
    printInt(distanceKmToLondon, gps.location.isValid(), 9);  //表示

    double courseToLondon = TinyGPSPlus::courseTo(   //現在地からみるロンドンの方向
        gps.location.lat(), gps.location.lng(), LONDON_LAT, LONDON_LON);

    printFloat(courseToLondon, gps.location.isValid(), 7, 2);

    const char *cardinalToLondon = TinyGPSPlus::cardinal(courseToLondon);  //cardinal = 方位ぽい

    // cardinal = 方位(文字表示), course = 方向(北から何度かを数字表示)

    printStr(gps.location.isValid() ? cardinalToLondon : "*** ", 6);  //位置が有効かどうかを判断。有効なら表示。無効なら*。

    printInt(gps.charsProcessed(), true, 6);  //解析したＧＰＳのデータの文字列数を返す
    printInt(gps.sentencesWithFix(), true, 10);  //不明
    printInt(gps.failedChecksum(), true, 9); //失敗した数を表示
    Serial.println();

    smartDelay(10000);  //この関数を１秒間かけて実行してる。

    Serial.printf("value =  %d\n");  //不明 %dて誰。現在時刻を表示してる説。

    if (millis() > 5000 && gps.charsProcessed() < 10)  //マイコンが電源入ってからの時間(ミリ秒)が5000よりも大きく、かつGPSデータの文字列が10より小さいなら
        Serial.println(F("No GPS data received: check wiring"));  //←こいつを表示
} //つまりデータが来てなかったらそれを知らせる役割

// This custom version of delay() ensures that the gps object
// is being "fed".
static void smartDelay(unsigned long ms) {  //ミリ秒間の間GPSのデータを読むよ
    unsigned long start = millis();
    do {
        while (Serial2.available()) gps.encode(Serial2.read());
    } while (millis() - start < ms);
}

static void printFloat(float val, bool valid, int len, int prec) {  //小数点を含む数の表示
    if (!valid) {
        while (len-- > 1) Serial.print('*');
        Serial.print(' ');
    } else {
        Serial.print(val, prec);
        int vi   = abs((int)val);
        int flen = prec + (val < 0.0 ? 2 : 1);  // . and -
        flen += vi >= 1000 ? 4 : vi >= 100 ? 3 : vi >= 10 ? 2 : 1;
        for (int i = flen; i < len; ++i) Serial.print(' ');
    }
    smartDelay(0);

    //以下、GPTから入手した奴
    while (gps.charsProcessed() > 0) {
    gps.encode(Serial2.read());  //gps.encode(ss.read());から書き換えた
    
    // GPSの位置が更新されていない場合
    if (!gps.location.isUpdated()) {
      Serial.println("ERROR: GPS data not available!");
      M5.Lcd.fillScreen(RED);
      M5.Lcd.setCursor(0, 0);
      M5.Lcd.println("GPS ERROR!");
      
      // プログラム停止（無限ループにして停止）
      while (true) {
        
        Serial.println("GPSが上手くいってないよ")

        // ここで何もせず無限ループを使って停止
      }
    }

    // 位置情報が更新されていたら表示
    if (gps.location.isUpdated()) {
      double lat = gps.location.lat();
      double lng = gps.location.lng();

      Serial.printf("Lat: %.6f, Lng: %.6f\n", lat, lng);

      M5.Lcd.fillScreen(BLACK);
      M5.Lcd.setCursor(0, 0);
      M5.Lcd.printf("Lat: %.6f\n", lat);
      M5.Lcd.printf("Lng: %.6f\n", lng);
    }
  }
}

//一旦ここは休んでもらう。GPTから貰った文字表示のやつをぶっこみたい
"

static void printInt(unsigned long val, bool valid, int len) {//整数を表示 データが届けば表示、届かなけりゃ*で済ます。
    char sz[32] = "*****************";
    if (valid) sprintf(sz, "%ld", val);
    sz[len] = 0;
    for (int i = strlen(sz); i < len; ++i) sz[i] = ' ';
    if (len > 0) sz[len - 1] = ' ';
    Serial.print(sz);
    smartDelay(0);
}

static void printDateTime(TinyGPSDate &d, TinyGPSTime &t) {//時間の表示 データが（以下略
    if (!d.isValid()) {
        Serial.print(F("********** "));
    } else {
        char sz[32];
        sprintf(sz, "%02d/%02d/%02d ", d.month(), d.day(), d.year());
        Serial.print(sz);
    }

    if (!t.isValid()) {
        Serial.print(F("******** "));
    } else {
        char sz[32];
        sprintf(sz, "%02d:%02d:%02d ", t.hour(), t.minute(), t.second());
        Serial.print(sz);
    }

    printInt(d.age(), d.isValid(), 5);
    smartDelay(0);
}

"

static void printStr(const char *str, int len) { //文字列の表示
    int slen = strlen(str);
    for (int i = 0; i < len; ++i) Serial.print(i < slen ? str[i] : ' ');
    smartDelay(0);
}



"void loop() {
  while (gps.charsProcessed() > 0) {
    gps.encode(ss.read());
    
    // GPSの位置が更新されていない場合
    if (!gps.location.isUpdated()) {
      Serial.println("ERROR: GPS data not available!");
      M5.Lcd.fillScreen(RED);
      M5.Lcd.setCursor(0, 0);
      M5.Lcd.println("GPS ERROR!");
      
      // プログラム停止（無限ループにして停止）
      while (true) {
        
        println("GPSが上手くいってないよ")

        // ここで何もせず無限ループを使って停止
      }
    }

    // 位置情報が更新されていたら表示
    if (gps.location.isUpdated()) {
      double lat = gps.location.lat();
      double lng = gps.location.lng();

      Serial.printf("Lat: %.6f, Lng: %.6f\n", lat, lng);

      M5.Lcd.fillScreen(BLACK);
      M5.Lcd.setCursor(0, 0);
      M5.Lcd.printf("Lat: %.6f\n", lat);
      M5.Lcd.printf("Lng: %.6f\n", lng);
    }
  }
}"
