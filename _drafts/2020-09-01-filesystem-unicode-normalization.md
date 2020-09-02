---
title: Filesystem & Unicode Normalization
date: 2020-08-01 00:00:00
tags: [linux]
---

```console
# touch $(echo -e "\u00C5")
# touch $(echo -e "\u212B")
# touch $(echo -e "\u0041\u030A")
# ls
./  ../  Å  Å  Å
```

## Unicode Decomposed and Precomposed Characters

部份 unicode 字元有<ruby>組合字元<rt>Decomposed Characters</rt></ruby>與<ruby>預組字元<rt>Precomposed Characters</rt></ruby>兩種表示方法，但這兩種表示方法並不是單純的一對一對應。

以 Å 為例：
* 組合字元的表示方法則可以拆成主要字元 [<ruby>`A`<rt>U+0041</rt></ruby>](https://www.compart.com/en/unicode/U+0041) 與 [附加符號](https://zh.wikipedia.org/wiki/%E7%B5%84%E5%90%88%E9%99%84%E5%8A%A0%E7%AC%A6%E8%99%9F)[<ruby>`◌̊`<rt>U+030A</rt></ruby>](https://www.compart.com/en/unicode/U+030A) 共兩個碼位
* 預組字元的表示方式則只佔用一個碼位，但表示 Å 預組字元的碼位卻不只一個，分別有 [Angstrom Sign <ruby>`Å`<rt>U+212B</rt></ruby>](https://www.compart.com/en/unicode/U+212Br) 和 [Swedish letter <ruby>`Å`<rt>U+00C5</rt></ruby>](https://www.compart.com/en/unicode/U+00C5)。

另外一個複雜一點的例子，ᾅ 字元：
* 用 [<ruby>`ᾅ`<rt>U+1F85</rt></ruby>](https://www.compart.com/en/unicode/U+1F85) 單獨表示
* 拆成 [<ruby>`ἅ`<rt>U+1F05</rt></ruby>](https://www.compart.com/en/unicode/U+1F05)-[<ruby>`◌ͅ`<rt>U+0345</rt></ruby>](https://www.compart.com/en/unicode/U+0345)
* 或 [<ruby>`ᾁ`<rt>U+1F81</rt></ruby>](https://www.compart.com/en/unicode/U+1F81)-[<ruby>`◌́`<rt>U+0301</rt></ruby>](https://www.compart.com/en/unicode/U+0301)
* ...
* 或 [<ruby>`α`<rt>U+03B1</rt></ruby>](https://www.compart.com/en/unicode/U+03B1)-[<ruby>`◌̔`<rt>U+0314</rt></ruby>](https://www.compart.com/en/unicode/U+0314)-[<ruby>`◌́`<rt>U+0301</rt></ruby>](https://www.compart.com/en/unicode/U+0301)-[<ruby>`◌ͅ`<rt>U+0345</rt></ruby>](https://www.compart.com/en/unicode/U+0345) 等多種表示方法。


---
## Unicode Equivalence and Normalization

由於 `U+212B`, `U+00C5`, `U+0041-U+030A` 都長成 `Å` 的模樣，應用程式處理 unicode 時必須考慮到<ruby>等價性<rt>Equivalence</rt></ruby>，否則在比對時找不到在視覺上無法區分的字形。但是也不代表等價，因此產生了[惡搞](https://github.com/reinderien/mimic)的空間，也額外衍生出安全性的問題。

* [Unicode Security Considerations](https://www.unicode.org/reports/tr36/)
* [MSDN: Security Considerations: International Features](https://docs.microsoft.com/zh-tw/windows/win32/intl/security-considerations--international-features)

<ruby>正規化<rt>Normalization</rt></ruby>可將所有等價的序列產生一個「唯一」的序列，因此字串要正規化完才能進行比較。先不考慮相容等價，Unicode 有 NFC 和 NFD 兩種標準正規化的方法：

| ***NFD***<br/> *Normalization Form Canonical Decomposition* | 以標準等價方式來分解<br/> Å ⟶ `U+0041-U+030A`<br/> ᾅ ⟶ `U+03B1 U+0314 U+0301 U+0345` |
| ***NFC***<br/> *Normalization Form Canonical Composition*   | 以標準等價方式來分解，然後以標準等價重組之。重組結果有可能和分解前不同。<br/> Å ⟶ `U+0041-U+030A` ⟶ `U+00C5`<br/> ᾅ ⟶ `U+03B1 U+0314 U+0301 U+0345` ⟶ `U+1F85` |


---
## Filesystem Implemetations

先講結論，大部分的檔案系統都**不會對檔案名稱做 unicode 正規化 (NFC/NFD)**。因此**不要假設檔案名稱是 NFC 或 NFD 形式**，可能還**沒有正規化**，或者同一個檔名有**兩者混合**的形式，甚至根本**不是合法的 unicode 編碼**，就只是一坨 raw bytes 從硬碟上被讀起來。

所以系統內任何對檔案名稱的操作，主要是 create 與 lookup，如果沒有統一的正規化來規範等價性，有機會[找不到存在的檔案](https://en.wikipedia.org/wiki/Unicode_equivalence#Errors_due_to_normalization_differences)而無法正常操作。

### Btrfs
保持檔案名稱原本的形式，不做額外的 unicode 正規化。

### Ext4
保持檔案名稱原本的形式，不做額外的 unicode 正規化。Linux-5.2 導入的 [case-insensitive](https://github.com/torvalds/linux/blob/master/Documentation/admin-guide/ext4.rst#case-insensitive-file-name-lookups) 會使用 NFD 正規化後再做檔名比較，但仍不改變檔案名稱儲存的形式。

* [Using the Linux kernel's Case-insensitive feature in Ext4](https://www.collabora.com/news-and-blog/blog/2020/08/27/using-the-linux-kernel-case-insensitive-feature-in-ext4/)

### HFS+
Mac [鍵盤輸入法](https://developer.apple.com/library/archive/qa/qa1235/_index.html)會產生 NFD (precomposed) 形式的字元。

Mac 專屬的 HFS+ 是少數會對檔案名稱加工的檔案系統，儲存前一律轉成 decomposed unicode 字元，但[並非嚴格的 NFD](https://developer.apple.com/library/archive/qa/qa1173/_index.html)。

> For example, HFS Plus (Mac OS Extended) uses a variant of Normal Form D in which U+2000 through U+2FFF, U+F900 through U+FAFF, and U+2F800 through U+2FAFF are not decomposed (this avoids problems with round trip conversions from old Mac text encodings).

* [Apple Developer: Technical Q&A QA1235. Converting to Precomposed Unicode](https://developer.apple.com/library/archive/qa/qa1235/_index.html)
* [Apple Developer: Technical Q&A QA1173. Text Encodings in VFS](https://developer.apple.com/library/archive/qa/qa1173/_index.html)

Linux 版本的 HFS+ 實作預設也是一樣的行為，檔案名稱會先轉成 NFD 形式再儲存，但讀取則會配合 linux 轉成 NFC 形式。此外，Linux 實作也提供了 [`nodecompose`](https://www.kernel.org/doc/html/latest/filesystems/hfsplus.html) 掛載選項，可以關閉自動轉成 NFD 形式的正規化，保持檔案名稱原本的形式。但如果有跨平台使用的需求，請勿開啟該選項。

### APFS
雖然 APFS 被定位用來取代 HFS+，但儲存檔案名稱時並不做任何 unicode 正規化，僅在算 filename hash 時使用到 NFD。

* [Apple File System Reference](https://developer.apple.com/support/downloads/Apple-File-System-Reference.pdf)
* [APFS's "Bag of Bytes" Filenames](https://mjtsai.com/blog/2017/03/24/apfss-bag-of-bytes-filenames/)

### NTFS
Windows 提供了各種正規化的機制，原生從 Windows 產生的 unicode 字元會是 NFC 形式，但仍然有機會從其他外部管道（網頁）獲得 NFD 形式的 unicode 字元

> Windows, Microsoft applications, and the .NET Framework generally generate characters in form C using normal input methods. For most purposes on Windows, form C is the preferred form. For example, characters in form C are produced by Windows keyboard input. However, characters imported from the Web and other platforms can introduce other normalization forms into the data stream.

NTFS 使用 unicode 的 UTF-16 編碼儲存檔案名稱，但本身並不會對檔案名稱做任何 unicode 正規化。

* [MSDN: Using Unicode Normalization to Represent Strings](https://docs.microsoft.com/zh-tw/windows/win32/intl/using-unicode-normalization-to-represent-strings)
* [MSDN: Naming Files, Paths, and Namespaces](https://docs.microsoft.com/zh-tw/windows/win32/fileio/naming-a-file)
* [MSDN: Character Sets Used in File Names](https://docs.microsoft.com/en-us/windows/win32/intl/character-sets-used-in-file-names)

### exFAT
exFAT 也是使用 UTF-16 編碼儲存檔案名稱。Samsung 貢獻的 [Linux exFAT](https://github.com/torvalds/linux/blob/master/fs/exfat/nls.c) 實作也僅做 UTF-8 🡘 UTF-16 之間的轉換，不做任何正規化。

### FAT family
FAT 家族的檔名機制因為歷史因素比較複雜，FAT16 之前僅支援 8.3 檔名，需搭配 codepage 處理。FAT32 之後支援的長檔名則使用 UTF-16 編碼，不做任何 unicode 正規化。

### How to Verify

**Å** 的三種表達方式雖然在 unicode 標準等價下視為相等，但大部份的檔案系統不處理 unicode 等價性，所以可以用下述的方法，建立出三個長得一樣的檔案名稱。
* **U+00C5** : NFC
* **U+212B** : not normalized
* **U+0041-U+030A** : NFD

```console
# touch $(echo -e "\u00C5")
# touch $(echo -e "\u212B")
# touch $(echo -e "\u0041\u030A")
# ls
# ls
./  ../  Å  Å  Å
```

目前已知不會建出三個檔案的檔案系統有：
* HFS+: 不意外
* Ext4 with case-insensitive: Ext4 比較大小寫前會先 NFD 再做 folding，因此只會建出第一個檔案，後續檔案會配判斷已存在而不建立。檔案名稱仍會保留原本的格式，不過 unicode 正規化。


---
## Which normalization form is recommended?

It should be **NFC**. NFD and NFKD are most useful for internal processing.

* **Unicode.org** [mentions](https://unicode.org/faq/normalization.html):
  * NFC is the best form for **general text**, since it is more compatible with strings converted from legacy encodings. NFD and NFKD are most useful for internal processing.
  * Much **legacy data** is automatically in NFC, since the character sets are constrained to that.
  * All **user-level comparison** should behave as if it normalizes the input to NFC. Most binary character matching that affects users should also behave as if it normalizes the input to NFC. Because it is rare to have non-NFC text, an optimized implementation can do such comparison very quickly.
  * Much of the **existing content** on the internet is already in NFC, and does not require re-normalization in contexts expecting to use NFC.
* **W3C** [recommends](http://www.w3.org/International/questions/qa-html-css-normalization) the use of NFC normalized text on the Web.
* **Windows** [generate characters](https://docs.microsoft.com/zh-tw/windows/win32/intl/using-unicode-normalization-to-represent-strings) in NFC using normal input methods.
* **Apple** developer document [recommends](https://developer.apple.com/library/archive/qa/qa1235/_index.html) to use NFC (precomposed) when you interact with other platforms.
  * If you implement a **network protocol** which is defined to use precomposed Unicode.
  * When creating a **cross-platform file (or volume)** whose specification dictates precomposed Unicode.
  * If you incorporate a large body of **cross-platform code into your application**, where that code is expecting precomposed Unicode.
* NFC is the preferred form for **Linux**.<sup><mark>找不到依據</mark></sup>


---
## References
* [Unicode equivalence](https://en.wikipedia.org/wiki/Unicode_equivalence)
* [Unicode Normalization FAQ](https://unicode.org/faq/normalization.html)
* [Unicode Normalization Test](https://www.unicode.org/Public/UNIDATA/NormalizationTest.txt)
* [Character Model for the World Wide Web: String Matching](https://www.w3.org/TR/charmod-norm/)
* [Something about Unicode](https://zhuanlan.zhihu.com/p/99989967)
* [Apache: Unicode Composition for Filename](http://svn.apache.org/repos/asf/subversion/trunk/notes/unicode-composition-for-filenames)
* [LWN: Working with UTF-8 in the kernel](https://lwn.net/Articles/784124/)
* [UTF-8 and Unicode FAQ for Unix/Linux](https://www.cl.cam.ac.uk/~mgk25/unicode.html)
* [Unicode Normalization](https://www.win.tue.nl/~aeb/linux/uc/nfc_vs_nfd.html)
