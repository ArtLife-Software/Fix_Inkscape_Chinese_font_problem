# Inkscape 中文字型問題完整解決教學（Windows）

Oct 14, 2026 · @linyancheng

## 👤 發布

**林彥丞**  
✉️ [lin.yancheng@outlook.com](mailto:lin.yancheng@outlook.com)  
🐙 [ArtLife-Software](https://github.com/ArtLife-Software)  
💬 [O && C VBA 研究社](https://www.facebook.com/groups/vba.club)  
🇹🇼 中華民國台灣

## 總覽

Windows 上的 Inkscape 使用中文字型時，常見的問題有四個，各有不同原因。本教學已在 Inkscape 1.x、Windows 10/11 實際解決以下問題。依序虛理，前一個問題解決後，後一個問題才會出現或看得出來。

| # | 症狀 | 原因 | 解決方法 |
| --- | --- | --- | --- |
| 1 | 放在 D: 、以「捷徑」安裝的字型，在 Inkscape 找不到 | Inkscape 的字型系統 fontconfig 只掃描資料夾，不讀 Windows 的安裝記錄 | 在 `fonts.conf` 加入字型資料夾 |
| 2 | 中文字型顯示英文名稱（如 DFKaiShuW5-B5） | fontconfig 不知道你是中文使用者，預設英文名稱優先 | 設定環境變數 `FC_LANG=zh-tw` |
| 3 | 設定中文名稱後，部分舊字型消失或變亂碼 | 字型的中文名稱用 Big5 編碼儲存，fontconfig 讀不懂 | 用 `font_names.py` 產生名稱修正規則 |
| 4 | 看得到字型名稱，但套用後變成別的字型 | 字型只有 Big5 查字表，沒有 Unicode 查字表 | 用 `fixbig5.py` 產生修正版字型 |

本教學用到的全部工具：一個設定檔 `fonts.conf`、兩個 Python 程式。完整內容都在文末「附錄」，可以直接複製使用。如果只是要在新電腦上重新設定，直接看「新電腦安裝步驟」。

## 最簡單的做法：一鍵安裝

工具包裡的 `安裝.bat` 會自動完成問題 1～4 的全部設定。不想了解原理的人，只要做這四步：

1. 安裝 Python：到 python.org 下載，安裝第一個畫面勾選「Add python.exe to PATH」。
2. 解壓縮工具包，雙擊 `安裝.bat`。
3. 輸入字型資料夾路徑（直接按 Enter 使用 `D:\Program_Files\Fonts`），看完將要做的事後輸入 Y。
4. 完成後登出 Windows 再登入，開啟 Inkscape。

想還原時，雙擊 `解除安裝.bat`。

| 安全設計 | 說明 |
| --- | --- |
| 先說明再執行 | 列出每一步，輸入 Y 才開始 |
| 不動原檔 | 不修改、不刪除任何原本的字型檔 |
| 自動備份 | 原本已有的 `fonts.conf` 會先備份，解除安裝時還原 |
| 發生錯誤就停 | 任一步失敗會停下並顯示是哪一步 |
| 可以重複執行 | 新增字型後再執行一次即可，不會重複備份 |

**尚未在其他電腦實測**：安裝與解除安裝已在模擬環境測試通過，但尚未在不同的 Windows 上實際執行。分享前請先在自己電腦試一次。批次檔是純文字，可以用記事本打開確認內容。

以下是各問題的原理與手動做法，供需要了解或自行調整的人參考。

## 問題 1：以捷徑安裝的字型，Inkscape 找不到

**症狀**：字型檔放在 D: 等其他磁碟，在 Windows 字型設定中勾選「允許使用捷徑安裝字型」後安裝。Word 能用，Inkscape 的字型清單內大部分中文字型不見了。

**原理**

- Windows 的捷徑安裝不複製字型檔，只在登錄檔記下「這個字型在 D:\\某處」。Windows 程式讀登錄檔，所以能用。
- Inkscape 不用 Windows 的字型系統，而是用跨平台的 **fontconfig**。fontconfig 不讀登錄檔，只掃描設定檔裡列出的資料夾（預設只有 `C:\Windows\Fonts`）。
- Windows 10/11 「只為目前使用者安裝」的字型放在 `AppData\Local\Microsoft\Windows\Fonts`，也可能有同樣問題。
- 這是 Inkscape 的已知問題，官方未修正：[Inkscape doesn't see fonts that installed "by shortcuts"](https://bugs.qastaging.launchpad.net/inkscape/+bug/1482823)。

**解決方法**：建立 fontconfig 使用者設定檔，用 `<dir>` 加入字型資料夾。fontconfig 會自動掃描所有子資料夾，不用逐一列出。

1. 建立資料夾 `C:\Users\你的帳號\.config\fontconfig\`。
2. 放入附錄的 `fonts.conf`，按檔案上方的說明改成你的帳號與字型資料夾。
3. 關閉 Inkscape，刪除 `%LOCALAPPDATA%\fontconfig\cache` 後重開。

另一種做法：Inkscape 1.0 以後的偏好設定有「額外的字型資料夾」（Additional font directories）可以加入資料夾（[發行說明](https://wiki.inkscape.org/wiki/Translations:Release_notes/1.0/321/en)）。但問題 3、4 的修正仍需 `fonts.conf`，所以建議一併用設定檔處理。

## 問題 2：中文字型顯示英文名稱

**症狀**：字型找得到，但清單顯示 `DFKaiShuW5-B5`，而不是「華康楷書體W5」。

**原理**

- 中文字型檔裡通常存有多個名稱。用 FontForge 開啟字型，在「元素 → 字型資訊 → TTF 名稱」可以看到：

| 語言 | 字串類型 | 內容 |
| --- | --- | --- |
| 中文（中華民國） | 字族 | 華康楷書體W5 |
| 英語（美國） | 字族 | DFKaiShuW5-B5 |

- Inkscape 的字型清單用的是「**字族**」（Name ID 1），不是「全名」或「惟一 ID」。
- fontconfig 依「語言設定」決定哪個名稱排第一，Inkscape 只顯示第一個。Windows 預設沒有提供這個設定，fontconfig 就當作英文使用者。
- 實測結果：不設定時拿到 `DFKaiShuW5-B5`，設定 `FC_LANG=zh-tw` 後拿到「華康楷書體W5」。

**解決方法**：開啟 PowerShell，執行下面指令，然後把 Inkscape 完全關閉再開。若仍顯示英文，登出 Windows 再登入一次。

```powershell
setx FC_LANG zh-tw
```

還原的指令：

```powershell
reg delete HKCU\Environment /v FC_LANG /f
```

注意：存成 SVG 時，字型名稱會記錄成中文。字型檔本身仍保有英文名稱，所以別的電腦只要裝了同一個字型就能正確辨認。

## 問題 3：舊字型消失或名稱變亂碼

**症狀**：設定 `FC_LANG` 後，部分舊字型從清單消失，或名稱變成「뛘놤ꧺ엩圳」、「§§∞Í¿s」這類亂碼。常見於華康、中國龍等早期字型。

**原理**

- 這些字型的中文名稱用台灣舊式的 **Big5 編碼** 儲存。Windows 讀得懂，fontconfig 讀不懂。
- 有些字型還把 Big5 中文名稱放在「英文」欄位，當成英文讀就變成 `§§∞Í¿s§Ú∑¢≈È`，用 Big5 解讀其實是「中國龍毛楷體」。
- 修正方式：用 fontconfig 規則，在掃描字型時依「PostScript 名稱」辨認字型，把亂碼名稱整個換成正確的中文與英文名稱。字型檔完全不動。

規則範例（程式會自動產生，不用手寫）：

```xml
<match target="scan">
  <test name="postscriptname"><string>IDDragonKai</string></test>
  <edit name="family" mode="assign_replace"><string>中國龍毛楷體</string><string>IDDragonKai</string></edit>
  <edit name="familylang" mode="assign_replace"><string>zh-tw</string><string>en</string></edit>
</match>
```

**解決方法**：執行附錄的 `font_names.py`。

```powershell
py -m pip install fonttools
py font_names.py "D:\Program_Files\Fonts"
```

程式會做三件事：

1. 在程式所在資料夾產生 `字型對照表.html`（可搜尋）與 `字型對照表.csv`（可用 Excel 開啟），列出每個字型的中英文名稱與檔名。
2. 找出中文名稱為 Big5 編碼的字型，寫入 `%USERPROFILE%\.config\fontconfig\fonts-big5-fix.conf`。
3. 在畫面最後列出「需要修正的字型」。

`fonts.conf` 範本已經寫好載入這份規則的設定。執行後清除 fontconfig 快取，再重開 Inkscape。

如果 `py` 指令無法執行，請依序試 `python -m pip`、Anaconda Prompt，或到 python.org 重新安裝 Python，安裝時勾選「Add python.exe to PATH」。

## 問題 4：看得到字型名稱，套用後卻變成別的字型

**症狀**：字型名稱正常出現，但套用後文字仍是其他字型。本例為「中國龍」系列 9 個字型（id005、id007、id009、id011、id013、id026、id027、id028、id029）。

**原理**

- 字型檔裡有三樣東西：字形（筆畫圖案）、查字表（哪個編號對應哪個字形）、名稱。
- 同一個字在不同編碼的編號不同，例如「中」：Big5 是 A4A4，Unicode 是 4E2D。
- 這些字型只有 Big5 查字表。Windows 會自動轉換，所以 Word 正常；Inkscape 的 FreeType 只查 Unicode 表，查不到任何字，就改用其他字型。
- 實測：用原檔直接繪製文字，結果一片空白。

| 查字表 | 編號範例（中） | 誰讀得懂 | 中國龍字型有嗎 |
| --- | --- | --- | --- |
| Big5（平台 3、編碼 4） | A4A4 | Windows | 有 |
| Unicode（平台 3、編碼 1） | 4E2D | Windows、Inkscape、幾乎所有程式 | 沒有 |

**修正方式**：程式把 Big5 表的每一筆轉成 Unicode（例如「A4A4 → 中的字形」轉成「4E2D → 中的字形」），加入一張 Unicode 查字表，並把名稱改成 Unicode 編碼。字形完全不變。本例每個字型轉換 13,870 個字，且字型內的字形名稱（如 uni4E2D）與轉換結果完全吻合。

**解決方法**：執行附錄的 `fixbig5.py`。

```powershell
py fixbig5.py "D:\Program_Files\Fonts"
```

程式會做三件事：

1. 自動找出只有 Big5（或簡體 GB）查字表的字型。
2. 產生修正版到 `字型資料夾\00.修正版\U_原檔名`。**原檔完全不動**，Windows 與 Word 照常用原檔。
3. 產生 `%USERPROFILE%\.config\fontconfig\fonts-big5-reject.conf`，讓 Inkscape 忽略原檔、只用修正版，避免同一個字型出現兩次。

執行後清除 fontconfig 快取，重開 Inkscape。修正版字型不用在 Windows 安裝，因為 fontconfig 會自己掃描到 `00.修正版` 資料夾。

**授權注意**：修正版是改過的字型檔。商業字型的授權通常不允許修改後散布，只限自己電腦使用。幫別人處理時，請對方在自己電腦執行程式，不要傳送修正後的字型檔。

## 新電腦安裝步驟

以下步驟一次完成問題 1～4 的全部設定，約 15 分鐘。先安裝好 Inkscape 與 Python，並把字型以捷徑方式安裝到 Windows。

1. **準備工具**：建立一個資料夾（例如桌面的「字型工具」），放入 `font_names.py` 與 `fixbig5.py`。
2. **安裝 fontTools**：在該資料夾開啟 PowerShell（檔案總管網址列輸入 `powershell` 按 Enter），執行 `py -m pip install fonttools`。
3. **放入設定檔**：建立 `C:\Users\你的帳號\.config\fontconfig\`，放入 `fonts.conf`，把「你的帳號」與字型資料夾改成實際值。（問題 1）
4. **設定中文名稱優先**：執行 `setx FC_LANG zh-tw`。（問題 2）
5. **修正無法套用的字型**：執行 `py fixbig5.py "D:\Program_Files\Fonts"`。（問題 4，先做這步）
6. **修正亂碼名稱**：執行 `py font_names.py "D:\Program_Files\Fonts"`。（問題 3）
7. **清除快取**：關閉 Inkscape，刪除 `%LOCALAPPDATA%\fontconfig\cache`（若有 `%LOCALAPPDATA%\inkscape\fonts` 也一併刪除）。
8. **登出 Windows 再登入**，讓 `FC_LANG` 生效。
9. **開啟 Inkscape 檢查**：第一次啟動會重建字型快取，字型多時需等數分鐘。

**檢查是否成功**

- [ ] D: 的字型出現在清單中
- [ ] 中文字型以中文名稱顯示（例如「華康楷書體W5」）
- [ ] 沒有亂碼名稱
- [ ] 「中國龍」等舊字型可以正常套用

**新增字型以後**：重新執行步驟 5、6、7。

**全部還原**

| 要還原的項目 | 做法 |
| --- | --- |
| fontconfig 設定 | 刪除 `%USERPROFILE%\.config\fontconfig` 資料夾 |
| 中文名稱優先 | `reg delete HKCU\Environment /v FC_LANG /f` |
| 修正版字型 | 刪除字型資料夾裡的 `00.修正版` |
| 快取 | 刪除 `%LOCALAPPDATA%\fontconfig\cache` |

本教學全程不修改原本的字型檔，也不影響 Windows 與 Office。

## 附錄：設定檔與程式碼

以下三個檔案可直接複製使用（一鍵安裝用的 安裝.bat、解除安裝.bat、setup\_tool.py 只放在工具包裡），存檔時請用 UTF-8 編碼。程式需要 Python 3 與 fontTools。

| 檔案 | 用途 | 放置位置 |
| --- | --- | --- |
| `fonts.conf` | fontconfig 設定檔（問題 1，並載入問題 3、4 的規則） | `C:\Users\你的帳號\.config\fontconfig\` |
| `font_names.py` | 字型對照表 + Big5 名稱修正規則（問題 3） | 任意資料夾 |
| `fixbig5.py` | 產生 Unicode 修正版字型（問題 4） | 任意資料夾 |

### fonts.conf

```xml
<?xml version="1.0"?>
<!DOCTYPE fontconfig SYSTEM "fonts.dtd">
<!--
  Inkscape（fontconfig）使用者設定檔
  放置位置：C:\Users\你的帳號\.config\fontconfig\fonts.conf
  ★ 使用前，請把下面所有「你的帳號」改成你的 Windows 帳號名稱，
    「D:/Program_Files/Fonts」改成你的字型資料夾。
  路徑一律用正斜線 /。刪除此檔即可還原。
-->
<fontconfig>
  <!-- 問題 1：讓 Inkscape 看到這些資料夾裡的字型（含所有子資料夾） -->
  <dir>D:/Program_Files/Fonts</dir>
  <dir>C:/Users/你的帳號/AppData/Local/Microsoft/Windows/Fonts</dir>
  <dir>C:/Windows/Fonts</dir>

  <!-- 中文後備字型：介面或缺字時，依序找第一個存在的 -->
  <alias>
    <family>sans-serif</family>
    <prefer>
      <family>Noto Sans TC</family>
      <family>Microsoft JhengHei</family>
    </prefer>
  </alias>
  <alias>
    <family>serif</family>
    <prefer>
      <family>Noto Serif TC</family>
      <family>PMingLiU</family>
    </prefer>
  </alias>

  <!-- 問題 3：修正 Big5 編碼的中文字型名稱（由 font_names.py 產生） -->
  <include ignore_missing="yes">C:/Users/你的帳號/.config/fontconfig/fonts-big5-fix.conf</include>

  <!-- 問題 4：忽略只有 Big5 查字表的原檔，改用修正版（由 fixbig5.py 產生） -->
  <include ignore_missing="yes">C:/Users/你的帳號/.config/fontconfig/fonts-big5-reject.conf</include>
</fontconfig>
```

### font\_names.py

```python
# -*- coding: utf-8 -*-
"""
字型中英文名稱對照表產生器

用法：
    py -m pip install fonttools
    py font_names.py                      （預設掃描 D:\\Program_Files\\Fonts）
    py font_names.py "E:\\其他字型資料夾"

輸出（放在本程式所在的資料夾）：
    字型對照表.csv          可用 Excel 開啟
    字型對照表.html         用瀏覽器開啟，可即時搜尋
另外會寫入：
    %USERPROFILE%\\.config\\fontconfig\\fonts-big5-fix.conf
                           修正 Big5 編碼中文名稱的規則（讓這些字型在 Inkscape 正常顯示）
"""
import csv
import html
import os
import sys
from xml.sax.saxutils import escape

try:
    from fontTools.ttLib import TTFont, TTCollection
except ImportError:
    sys.exit("請先安裝 fontTools：py -m pip install fonttools")

DEFAULT_DIR = r"D:\Program_Files\Fonts"
FONT_EXT = (".ttf", ".otf", ".ttc", ".otc")

# Windows 平台 (platformID 3) 的語言代碼
ZH_LANGS = [0x0404, 0x0C04, 0x1404, 0x0804, 0x1004]   # 台灣、香港、澳門、中國、新加坡
EN_LANGS = [0x0409, 0x0809]                            # 美國、英國
# Mac 平台 (platformID 1) 的語言代碼
MAC_ZH = [19, 33]   # 19 = 繁體中文, 33 = 簡體中文
MAC_EN = [0]


def decode(rec):
    """把 name 記錄轉成文字；舊字型常用 Big5 編碼，失敗時改用 Big5 試試。"""
    try:
        s = rec.toUnicode()
    except Exception:
        s = None
    # 舊的中文字型常把 Big5 位元組放在標示為「Mac 英文」的記錄裡，
    # 照 Mac Roman 解碼會變成「§§∞Í¿s」這種亂碼，改用 Big5 / GBK 解碼
    if rec.platformID == 1 and any(b >= 0x80 for b in rec.string):
        for enc in ("big5", "gbk"):
            try:
                t = rec.string.decode(enc)
                if any("\u4e00" <= ch <= "\u9fff" for ch in t):
                    return t.strip()
            except Exception:
                pass
    if rec.platformID == 3 and rec.platEncID in (3, 4):   # 3 = GB, 4 = Big5
        raw = rec.string
        raw = bytes(b for b in raw if b != 0)   # 有些字型把 Big5 存成雙位元組並補 0
        for enc in ("big5", "cp950", "gbk"):
            try:
                return raw.decode(enc)
            except Exception:
                pass
    return s.strip() if s else s


def get_name(font, name_ids, win_langs, mac_langs):
    table = font["name"]
    for nid in name_ids:
        for lang in win_langs:
            for rec in table.names:
                if rec.nameID == nid and rec.platformID == 3 and rec.langID == lang:
                    s = decode(rec)
                    if s:
                        return s
        for lang in mac_langs:
            for rec in table.names:
                if rec.nameID == nid and rec.platformID == 1 and rec.langID == lang:
                    s = decode(rec)
                    if s:
                        return s
    return ""


def first_name(font, name_ids):
    """不分語言，取第一個找得到的名稱（備用）。"""
    for nid in name_ids:
        for rec in font["name"].names:
            if rec.nameID == nid:
                s = decode(rec)
                if s:
                    return s
    return ""


def has_unicode_zh_family(font):
    """字型裡是否有 fontconfig 讀得懂的（Windows Unicode）中文字族名稱。"""
    for rec in font["name"].names:
        if (rec.nameID in (1, 16) and rec.platformID == 3
                and rec.platEncID in (1, 10) and rec.langID in ZH_LANGS):
            return True
    return False


def ascii_or_empty(s):
    return s if s and all(32 <= ord(c) < 127 for c in s) else ""


def describe(font):
    fam_ids = [16, 1]     # 優先用「印刷字族」，沒有才用「字族」
    zh_family = get_name(font, fam_ids, ZH_LANGS, MAC_ZH)
    en_family = get_name(font, fam_ids, EN_LANGS, MAC_EN) or first_name(font, fam_ids)
    zh_full = get_name(font, [4], ZH_LANGS, MAC_ZH)
    en_full = get_name(font, [4], EN_LANGS, MAC_EN) or first_name(font, [4])
    style = get_name(font, [17, 2], EN_LANGS, MAC_EN) or first_name(font, [17, 2])
    ps = first_name(font, [6])
    needs_fix = bool(zh_family) and not has_unicode_zh_family(font)
    return zh_family, en_family, zh_full, en_full, style, ps, needs_fix


FIXES = []   # 需要修正名稱的字型（中文名稱是 Big5 等舊編碼，fontconfig 讀不懂）


def scan(folder):
    rows, errors = [], []
    for root, _, files in os.walk(folder):
        for fn in sorted(files):
            if not fn.lower().endswith(FONT_EXT):
                continue
            path = os.path.join(root, fn)
            rel = os.path.relpath(path, folder)
            try:
                if fn.lower().endswith((".ttc", ".otc")):
                    holder = TTCollection(path, lazy=True)
                    fonts = list(holder.fonts)
                else:
                    holder = TTFont(path, lazy=True)
                    fonts = [holder]
                try:
                    for i, f in enumerate(fonts):
                        *info, fix = describe(f)
                        rows.append((*info, rel, i if len(fonts) > 1 else ""))
                        if fix:
                            FIXES.append((info, path, i))
                finally:
                    holder.close()   # 讀完整個檔案才關閉（TTC 內的字型共用同一個檔案）
            except Exception as e:
                errors.append((rel, str(e)))
    return rows, errors


HEAD = ["中文字族", "英文字族（Inkscape 顯示）", "中文全名", "英文全名",
        "樣式", "PostScript 名稱", "檔案", "TTC 序號"]


def write_csv(rows, path):
    with open(path, "w", newline="", encoding="utf-8-sig") as f:
        w = csv.writer(f)
        w.writerow(HEAD)
        w.writerows(rows)


def write_html(rows, path, folder):
    body = "\n".join(
        "<tr>" + "".join(f"<td>{html.escape(str(c))}</td>" for c in r) + "</tr>"
        for r in rows)
    page = f"""<!doctype html><html lang="zh-Hant"><head><meta charset="utf-8">
<title>字型對照表</title><style>
body{{font-family:"Microsoft JhengHei",sans-serif;margin:16px;}}
input{{font-size:16px;padding:6px;width:320px;}}
table{{border-collapse:collapse;margin-top:12px;font-size:14px;}}
th,td{{border:1px solid #ccc;padding:4px 8px;text-align:left;}}
th{{background:#f0f0f0;position:sticky;top:0;}}
tr:hover td{{background:#fffbe6;}}
td:nth-child(2){{font-weight:bold;}}
</style></head><body>
<h2>字型對照表</h2><p>{html.escape(folder)}，共 {len(rows)} 個字型</p>
<input id="q" placeholder="搜尋中文或英文名稱、檔名…" autofocus>
<table><thead><tr>{"".join(f"<th>{h}</th>" for h in HEAD)}</tr></thead>
<tbody id="t">{body}</tbody></table>
<script>
document.getElementById('q').addEventListener('input',e=>{{
 const k=e.target.value.toLowerCase();
 for(const tr of document.getElementById('t').rows)
  tr.style.display=tr.textContent.toLowerCase().includes(k)?'':'none';
}});
</script></body></html>"""
    with open(path, "w", encoding="utf-8") as f:
        f.write(page)


def write_fix_conf(path):
    """為 Big5 名稱的字型產生 fontconfig 規則，用正確的中文名稱取代亂碼。"""
    rules = []
    for (zh, en, zh_full, en_full, style, ps), fpath, idx in FIXES:
        names = [zh]
        langs = ["zh-tw"]
        en_ok = ascii_or_empty(en) or ascii_or_empty(ps)
        if en_ok and en_ok != zh:
            names.append(en_ok)
            langs.append("en")
        ps_ok = ascii_or_empty(ps)
        if ps_ok:
            test = f'    <test name="postscriptname"><string>{escape(ps_ok)}</string></test>\n'
        else:
            fp = os.path.abspath(fpath).replace("\\", "/")
            test = (f'    <test name="file"><string>{escape(fp)}</string></test>\n'
                    f'    <test name="index"><int>{idx}</int></test>\n')
        rules.append(
            f"  <!-- {escape(zh)} -->\n"
            '  <match target="scan">\n' + test +
            '    <edit name="family" mode="assign_replace">'
            + "".join(f"<string>{escape(n)}</string>" for n in names) + "</edit>\n"
            '    <edit name="familylang" mode="assign_replace">'
            + "".join(f"<string>{l}</string>" for l in langs) + "</edit>\n"
            "  </match>")
    with open(path, "w", encoding="utf-8") as f:
        f.write('<?xml version="1.0"?>\n<!DOCTYPE fontconfig SYSTEM "fonts.dtd">\n')
        f.write("<!-- 由 font_names.py 產生：修正 Big5 編碼的中文字型名稱 -->\n<fontconfig>\n")
        f.write("\n".join(rules))
        f.write("\n</fontconfig>\n")
    return len(rules)


def main():
    folder = sys.argv[1] if len(sys.argv) > 1 else DEFAULT_DIR
    if not os.path.isdir(folder):
        sys.exit(f"找不到資料夾：{folder}")
    out = os.path.dirname(os.path.abspath(__file__))
    print(f"掃描中：{folder}")
    rows, errors = scan(folder)
    rows.sort(key=lambda r: (r[1].lower(), r[0]))
    write_csv(rows, os.path.join(out, "字型對照表.csv"))
    write_html(rows, os.path.join(out, "字型對照表.html"), folder)
    fc_dir = os.path.join(os.path.expanduser("~"), ".config", "fontconfig")
    os.makedirs(fc_dir, exist_ok=True)
    nfix = write_fix_conf(os.path.join(fc_dir, "fonts-big5-fix.conf"))
    no_zh = sum(1 for r in rows if not r[0])
    print(f"完成：{len(rows)} 個字型，其中 {no_zh} 個沒有中文字族名稱")
    print("已產生：字型對照表.csv、字型對照表.html")
    print(f"輸出位置：{out}")
    print(f"Big5 名稱修正：{nfix} 個字型 → {os.path.join(fc_dir, 'fonts-big5-fix.conf')}")
    if FIXES:
        print("  需要修正的字型：" + "、".join(i[0] for i, _, _ in FIXES))
    if errors:
        print(f"\n有 {len(errors)} 個檔案無法讀取：")
        for p, e in errors:
            print(f"  {p}：{e}")


if __name__ == "__main__":
    main()
```

### fixbig5.py

```python
# -*- coding: utf-8 -*-
"""
修正「看得到名稱、卻無法套用」的舊中文字型（只有 Big5 / GB 查字表的字型）

用法：
    py -m pip install fonttools
    py fixbig5.py                         （預設處理 D:\\Program_Files\\Fonts）
    py fixbig5.py "E:\\我的字型"

做的事：
    1. 找出沒有 Unicode 查字表、只有 Big5（或 GB）查字表的字型
    2. 為每個字型產生修正版：加上 Unicode 查字表，並把名稱改成 Unicode 編碼
       修正版存到「字型資料夾\\00.修正版\\U_原檔名」，原檔完全不動
    3. 產生 %USERPROFILE%\\.config\\fontconfig\\fonts-big5-reject.conf
       讓 Inkscape 忽略原檔、只用修正版（Windows 仍使用原檔，不受影響）

注意：修正版是修改過的字型檔，只限自己電腦使用，請勿散布（尤其是商業字型）。
"""
import os
import struct
import sys
from xml.sax.saxutils import escape

try:
    from fontTools.ttLib import TTFont, newTable
    from fontTools.ttLib.tables._c_m_a_p import CmapSubtable
except ImportError:
    sys.exit("請先安裝 fontTools：py -m pip install fonttools")

DEFAULT_DIR = r"D:\Program_Files\Fonts"
OUT_SUBDIR = "00.修正版"
# (平台, 編碼) → Python 解碼方式
LEGACY_CMAPS = {(3, 4): "cp950", (3, 3): "gbk", (1, 2): "cp950", (1, 25): "gbk"}
UNICODE_CMAPS = {(3, 1), (3, 10), (0, 0), (0, 1), (0, 2), (0, 3), (0, 4), (0, 6)}


def raw_subtables(data):
    n = struct.unpack(">H", data[2:4])[0]
    for i in range(n):
        pid, eid, off = struct.unpack(">HHI", data[4 + 8 * i:12 + 8 * i])
        fmt = struct.unpack(">H", data[off:off + 2])[0]
        yield pid, eid, fmt, off


def decode_subtable(data, fmt, off, font):
    """直接解析查字表（舊字型的表常有格式小錯誤，fontTools 正常讀取會失敗）。"""
    if fmt == 0 or fmt >= 8:
        return {}
    length = struct.unpack(">H", data[off + 2:off + 4])[0]
    st = CmapSubtable.newSubtable(fmt)
    st.decompileHeader(data[off:off + length], font)
    st.decompile(None, None)
    return st.cmap


def code_to_unicode(code, enc):
    try:
        if code < 0x100:
            return code if code < 0x80 else None
        return ord(bytes([code >> 8, code & 0xFF]).decode(enc))
    except Exception:
        return None


def check(path):
    """回傳 (需要修正?, 舊編碼)。"""
    f = TTFont(path, lazy=True)
    try:
        subs = [(pid, eid) for pid, eid, _, _ in raw_subtables(f.reader["cmap"])]
    finally:
        f.close()
    if any(s in UNICODE_CMAPS for s in subs):
        return False, None
    for s in subs:
        if s in LEGACY_CMAPS and s[0] == 3:
            return True, LEGACY_CMAPS[s]
    return False, None


def legacy_name(rec):
    raw = bytes(b for b in rec.string if b)   # 有些字型把每個位元組補成兩個位元組
    for enc in ("cp950", "gbk"):
        try:
            return raw.decode(enc).strip()
        except Exception:
            pass
    return None


def fix(src, dst, enc):
    f = TTFont(src)
    data = f.reader["cmap"]
    mapping = {}
    for pid, eid, fmt, off in raw_subtables(data):
        if (pid, eid) in LEGACY_CMAPS and pid == 3:
            for code, glyph in decode_subtable(data, fmt, off, f).items():
                u = code_to_unicode(code, enc)
                if u is not None and u not in mapping:
                    mapping[u] = glyph
    if not mapping:
        raise ValueError("讀不到任何字元")

    cmap = newTable("cmap")
    cmap.tableVersion = 0
    uni = CmapSubtable.newSubtable(4)
    uni.platformID, uni.platEncID, uni.language = 3, 1, 0
    uni.cmap = {u: g for u, g in mapping.items() if u <= 0xFFFF}
    mac = CmapSubtable.newSubtable(6)
    mac.platformID, mac.platEncID, mac.language = 1, 0, 0
    mac.cmap = {u: g for u, g in mapping.items() if u < 0x80}
    cmap.tables = [uni, mac]
    f["cmap"] = cmap

    # 名稱：舊編碼的中文名稱 → Unicode；英文名稱用 PostScript 名稱
    nm = f["name"]
    zh = {}
    for r in nm.names:
        if (r.platformID, r.platEncID) in ((3, 4), (3, 3), (1, 2), (1, 25)) or \
                (r.platformID == 1 and any(b >= 0x80 for b in r.string)):
            t = legacy_name(r)
            if t and r.nameID not in zh:
                zh[r.nameID] = t
    ps = None
    for r in nm.names:
        if r.nameID == 6:
            s = r.string.decode("utf-16-be", "ignore") if r.platformID == 3 else r.string.decode("latin-1")
            s = s.replace("\x00", "").strip()
            if s and s.isascii():
                ps = s
                break
    ps = ps or os.path.splitext(os.path.basename(src))[0]
    nm.names = [r for r in nm.names
                if (r.platformID == 3 and r.platEncID in (1, 10))
                or (r.platformID == 1 and r.platEncID == 0 and all(b < 0x80 for b in r.string))]
    zh_lang = 0x0804 if enc == "gbk" else 0x0404
    for nid, txt in zh.items():
        nm.setName(txt, nid, 3, 1, zh_lang)
    style = zh.get(2) if zh.get(2, "").isascii() and zh.get(2) else "Regular"
    for nid, txt in {1: ps, 2: style, 4: ps, 6: ps}.items():
        nm.setName(txt, nid, 3, 1, 0x0409)
    if not zh.get(1):
        zh[1] = ps
    f.save(dst)
    return zh[1], ps, len(mapping)


def main():
    folder = sys.argv[1] if len(sys.argv) > 1 else DEFAULT_DIR
    if not os.path.isdir(folder):
        sys.exit(f"找不到資料夾：{folder}")
    out_dir = os.path.join(folder, OUT_SUBDIR)
    print(f"檢查中：{folder}")
    todo = []
    for root, dirs, files in os.walk(folder):
        dirs[:] = [d for d in dirs if d != OUT_SUBDIR]
        for fn in sorted(files):
            if fn.lower().endswith((".ttf", ".otf")):
                p = os.path.join(root, fn)
                try:
                    need, enc = check(p)
                except Exception as e:
                    print(f"  略過（無法讀取）：{fn}  {e}")
                    continue
                if need:
                    todo.append((p, enc))
    if not todo:
        print("沒有需要修正的字型。")
        return
    os.makedirs(out_dir, exist_ok=True)
    done = []
    for p, enc in todo:
        fn = os.path.basename(p)
        dst = os.path.join(out_dir, "U_" + fn)
        try:
            zh, ps, n = fix(p, dst, enc)
            done.append(fn)
            print(f"  已修正：{fn} → U_{fn}　{zh}（{ps}），{n} 個字")
        except Exception as e:
            print(f"  修正失敗：{fn}  {e}")
    fc_dir = os.path.join(os.path.expanduser("~"), ".config", "fontconfig")
    os.makedirs(fc_dir, exist_ok=True)
    conf = os.path.join(fc_dir, "fonts-big5-reject.conf")
    with open(conf, "w", encoding="utf-8") as f:
        f.write('<?xml version="1.0"?>\n<!DOCTYPE fontconfig SYSTEM "fonts.dtd">\n')
        f.write("<!-- 由 fixbig5.py 產生：Inkscape 忽略這些原檔，改用 00.修正版 裡的修正版 -->\n")
        f.write("<fontconfig>\n  <selectfont>\n    <rejectfont>\n")
        for fn in done:
            f.write(f"      <glob>*/{escape(fn)}</glob>\n")
        f.write("    </rejectfont>\n  </selectfont>\n</fontconfig>\n")
    print(f"\n完成：修正 {len(done)} 個字型，存放在 {out_dir}")
    print(f"已產生：{conf}")
    if not os.environ.get("INKSCAPE_ZH_SETUP"):   # 一鍵安裝時由安裝程式統一處理
        print("接著請關閉 Inkscape、刪除 %LOCALAPPDATA%\\fontconfig\\cache，再重新開啟 Inkscape。")


if __name__ == "__main__":
    main()
```
