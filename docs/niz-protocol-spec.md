# NIZ通信プロトコル仕様書

cho45/niz-tools-ruby の `niz.rb` を解析した結果をまとめる。

## 接続情報

| 項目 | 値 |
|---|---|
| Vendor ID | `0x0483` |
| Product ID | `0x512A` (cho45氏のコード。Niz L84 では要実機確認) |
| 通信方式 | USB HID |
| Report ID | `0x00` |
| 送信バッファサイズ | 65 bytes |
| 受信バッファサイズ | 64 bytes |

## コマンド一覧

| 名前 | 値 | 説明 |
|---|---|---|
| `COMMAND_VERSION` | `0xf9` | ファームウェアバージョン取得 |
| `COMMAND_READ_ALL` | `0xf2` | 全キーマップ読み取り開始 |
| `COMMAND_WRITE_ALL` | `0xf1` | 全キーマップ書き込み開始（既存をリセット） |
| `COMMAND_KEY_DATA` | `0xf0` | 1キーのデータ送信（write時） |
| `COMMAND_DATA_END` | `0xf6` | 書き込み終了 |
| `COMMAND_KEYLOCK` | `0xd9` | キーロック/アンロック |
| `COMMAND_READ_COUNTER` | `0xe3` | キー押下カウンター読み取り |
| `COMMAND_READ_SERIAL` | `0x10` | シリアル番号読み取り |
| `COMMAND_CALIB` | `0xda` | キャリブレーション |
| `COMMAND_INITIAL_CALIB` | `0xdb` | 初期キャリブレーション |
| `COMMAND_PRESS_CALIB` | `0xdd` | 押下キャリブレーション |
| `COMMAND_PRESS_CALIB_DONE` | `0xde` | 押下キャリブレーション完了 |
| `COMMAND_XXX_DATA` | `0xe0` | 不明（マクロ関連？） |
| `COMMAND_READ_XXX` | `0xe2` | 不明（マクロ関連？） |
| `COMMAND_XXX_END` | `0xe6` | 不明 |

## 送信フォーマット (send_command)

65バイト固定。未使用部分は `0x00` パディング。

```
Byte 0:    Report ID (0x00)
Byte 1-2:  Command (big-endian, 2 bytes)
Byte 3-64: Data (最大62 bytes)
```

## プロトコル詳細

### version (0xf9)

**送信:**
```
[0x00] [0x00 0xf9] [0x00 * 62]
```

**受信 (64 bytes):**
```
Byte 0-1: Command echo (big-endian)
Byte 2+:  Version string (ASCII)
```

バージョン文字列の先頭数字がキー数を表す（例: `"66..."` → 66キー）。

### read_all (0xf2) — 全キーマップ読み取り

**送信:**
```
[0x00] [0x00 0xf2] [0x00 * 62]
```

**受信 (ループ、1キーにつき64 bytes):**
```
Byte 0:    継続フラグ (0x00 = データあり、それ以外 = 終了)
Byte 0-1:  Command echo (big-endian)
Byte 2:    Level (1始まり: 1=Normal, 2=Right Fn, 3=Left Fn)
Byte 3:    Key ID (1始まり)
Byte 4-5:  Unknown (2 bytes)
Byte 6:    HWCODE
Byte 7+:   Rest (未使用)
```

- 合計受信回数: **キー数 × 3レイヤー** (例: 66キー → 198回)
- Level はコード内で `level - 1` して 0始まりに変換:
  - 0 = Normal
  - 1 = Right Fn
  - 2 = Left Fn

### write_all (0xf1) — 全キーマップ書き込み

**重要: `0xf1` を送信した時点で既存キーマップが全リセットされる。必ず全キー分のデータを書き込むこと。**

**手順:**

1. 書き込み開始:
```
[0x00] [0x00 0xf1] [0x00 * 62]
```

2. 各キーのデータ送信 (3レイヤー × キー数 回繰り返し):
```
send_command(0xf0, data)
```
data の構造:
```
Byte 0:    Level (1始まり: level + 1)
Byte 1:    Key ID (1始まり)
Byte 2-3:  Flag (hwcode が 0以外 → 0x00 0x01、hwcode が 0 → 0x00 0x00)
Byte 4:    HWCODE
```

3. 書き込み終了:
```
send_command(0xf6, 0xf6 * 62)
```

### keylock / keyunlock (0xd9)

```
send_command(0xd9, 0x00)  → キーロック
send_command(0xd9, 0x01)  → キーアンロック
```

### read_counter (0xe3) — キー押下カウンター

**送信:**
```
send_command(0xe3)
```

**受信 (ループ):**
```
Byte 0:    継続フラグ (0x00 = データあり)
Byte 0-1:  Command echo
Byte 2-3:  Unknown (2 bytes)
Byte 4+:   Counts (4 bytes each, little-endian uint32)
```

## HWCODE マッピング

NIZ独自のキーコード体系。0〜255の値。

### 基本キー (0-90)

| HWCODE | キー名 | HWCODE | キー名 |
|---|---|---|---|
| 0 | (未割当) | 1 | ESC |
| 2 | F1 | 3 | F2 |
| 4 | F3 | 5 | F4 |
| 6 | F5 | 7 | F6 |
| 8 | F7 | 9 | F8 |
| 10 | F9 | 11 | F10 |
| 12 | F11 | 13 | F12 |
| 14 | \` | 15 | 1 |
| 16 | 2 | 17 | 3 |
| 18 | 4 | 19 | 5 |
| 20 | 6 | 21 | 7 |
| 22 | 8 | 23 | 9 |
| 24 | 0 | 25 | - |
| 26 | = | 27 | BS |
| 28 | TAB | 29 | Q |
| 30 | W | 31 | E |
| 32 | R | 33 | T |
| 34 | Y | 35 | U |
| 36 | I | 37 | O |
| 38 | P | 39 | [ |
| 40 | ] | 41 | \\ |
| 42 | Caps Lock | 43 | A |
| 44 | S | 45 | D |
| 46 | F | 47 | G |
| 48 | H | 49 | J |
| 50 | K | 51 | L |
| 52 | ; | 53 | ' |
| 54 | RET | 55 | L-Shift |
| 56 | Z | 57 | X |
| 58 | C | 59 | V |
| 60 | B | 61 | N |
| 62 | M | 63 | , |
| 64 | . | 65 | / |
| 66 | R-Shift | 67 | L-CTRL |
| 68 | Left-Super | 69 | L-Alt |
| 70 | Space | 71 | R-Alt |
| 72 | Right-Super | 73 | ContextMenu |
| 74 | R-Ctrl | 75 | Wakeup |
| 76 | Sleep | 77 | Power |
| 78 | PriSc | 79 | SclLk |
| 80 | Pause | 81 | Ins |
| 82 | Home | 83 | PageUp |
| 84 | Del | 85 | End |
| 86 | PageDown | 87 | Up Arrow |
| 88 | Left Arrow | 89 | Down Arrow |
| 90 | Right Arrow | | |

### テンキー (91-107)

| HWCODE | キー名 | HWCODE | キー名 |
|---|---|---|---|
| 91 | Num Lock | 92 | (/) |
| 93 | (*) | 94 | (7) |
| 95 | (8) | 96 | (9) |
| 97 | (4) | 98 | (5) |
| 99 | (6) | 100 | (1) |
| 101 | (2) | 102 | (3) |
| 103 | (0) | 104 | (.) |
| 105 | (-) | 106 | (+) |
| 107 | (Enter) | | |

### メディア・WWWキー (108-125)

| HWCODE | キー名 | HWCODE | キー名 |
|---|---|---|---|
| 108 | Media Next Track | 109 | Media Previous Track |
| 110 | Media Stop | 111 | Media Play/Pause |
| 112 | Media Mute | 113 | Media VolumeUp |
| 114 | Media VolumeDn | 115 | Media Select |
| 116 | WWW Email | 117 | Media Calculator |
| 118 | Media My Computer | 119 | WWW Search |
| 120 | WWW Home | 121 | WWW Back |
| 122 | WWW Forward | 123 | WWW Stop |
| 124 | WWW Refresh | 125 | WWW Favorites |

### マウスキー (126-134)

| HWCODE | キー名 | HWCODE | キー名 |
|---|---|---|---|
| 126 | Mouse Left | 127 | Mouse Right |
| 128 | Mouse Up | 129 | Mouse Down |
| 130 | Mouse Key Left | 131 | Mouse Key Right |
| 132 | Mouse Key Middle | 133 | Mouse Wheel Up |
| 134 | Mouse Wheel Dn | | |

### バックライト・設定系 (135-177)

| HWCODE | キー名 | HWCODE | キー名 |
|---|---|---|---|
| 135 | Backlight Switch | 136 | Backlight Macro |
| 137 | Demonstrate | 138 | Star shower |
| 139 | Riffle | 140 | Demo Stop |
| 141 | Breathe | 142 | Breathe Sequence- |
| 143 | Breathe Sequence+ | 144 | Backlight Lightness- |
| 145 | Backlight Lightness+ | 146 | Sunset or Relax/Aurora |
| 147 | Color Breathe | 148 | Back Color Exchange |
| 149 | Adjust Trigger Point | 150 | Keyboard Lock |
| 151 | Shift&Up | 152 | Ctrl&Caps Exchange |
| 153 | WinLock | 154 | MouseLock |
| 155 | Win/Mac Exchange | 156 | R-Fn |
| 157 | Mouse Unit Pixel | 158 | Mouse Unit Time |
| 159 | Programmable keyboard | 160 | Backlight Record1 |
| 161 | Backlight Record2 | 162 | Backlight Record3 |
| 163 | Backlight Record4 | 164 | Backlight Record5 |
| 165 | Backlight Record6 | 166 | L-Fn |
| 167 | Wire/Wireless exchange | 168 | BTD1 |
| 169 | BTD2 | 170 | BTD3 |
| 171 | Game | 172 | ECO |
| 173 | Mouse First Delay | 174 | Key Repeat Rate |
| 175 | Key Response Delay | 176 | USB Report Rate |
| 177 | Key Scan Period | | |

### その他

| HWCODE | キー名 |
|---|---|
| 178-198 | unknown |
| 199 | Mouse Left Double Click |
| 200-203 | unknown |
| 204 | <>&#124; |
| 205-255 | unknown |

## 注意事項

- Product ID `0x512A` は cho45 氏が使用していた機種のもの。Niz L84 では異なる可能性があるため実機で確認が必要。
- write_all は送信開始時点で全キーマップがリセットされるため、必ず read_all → 変更 → write_all の流れで使用すること。
- キー数はバージョン文字列から取得する（機種によって異なる）。
