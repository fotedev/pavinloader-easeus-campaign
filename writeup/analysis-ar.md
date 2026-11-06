# من كراك «EaseUS Partition Master 2026» المزيف إلى حمولة ‎.NET‎ بلا ملفات

تحليل حادثة — fotedev، 2026-10-30. النسخة العربية؛ الإنجليزية: analysis-en.md

## الزبدة

- في 2026-10-28، وبينما كان مستخدم يبحث عن «كراك EaseUS Partition Master 2026» على موقع برمجيات عربي، أوصلته زرارياً مزروعة في الصفحة إلى بوابة استضافة ملفات مزيفة على النطاق `*.m5u1iioax.store` (بأسماء: DropVault وIcecloud وGoFiles وWeShare وShareBox)، ونزّل «أرشيفًا» بحجم ~700–800 ميغابايت.
- الأرشيف ليس EaseUS: حزمة محرك ألعاب Ren'Py 8.1.3 مع ملف `Setup.exe` غير موقّع. تشغيله ثبّت ملفّي MSI وهميين باسم «Killer Network Service 2.6» (Qualcomm) و«PhysX Runtime 7.0» (NVIDIA): لا وجود لأيٍّ من الإصدارين أصلًا، والجهاز لا يملك كارت NVIDIA ولا شبكة Killer أصلًا.
- كل MSI يُسقط حلقة ‎.NET‎ وهمية اسمها «CatalogIngestor» مع نسخة MSBuild.exe مرفقة: موقّعة توقيع Microsoft حقيقي لكنها **عُدّلت بعد التوقيع** (نتيجة الفحص `HashMismatch`).
- المحمّل يعتمد على **دوال الخصائص في MSBuild فقط** بلا أي Inline Tasks: يقرأ ملف `Resources.resx` بحجم 31.6 ميغابايت، يستخرج Res1–Res4، يحذف فواصل `p`، يطبّق XOR-6 على كل نايبل، يدير التدفق السداسي عند الإزاحة 142,132، ثم `SoapHexBinary.Parse` وأخيرًا `AppDomain.CurrentDomain.Load(...).CreateInstance("CloneExtensions.FallbackProvider5")`: حمولة ‎~72KB‎ تُنفَّذ **بالكامل في الذاكرة**.
- Malwarebytes سمّى العائلة **Trojan.PavinLoader** وسجّن 109 عناصر في 2026-10-29. سُجّلت أيضًا إصابات مرافقة (Trojan.Glupteba وTrojan.Ranumbot وعائلتا Adware) أقدم من الحملة أو موازية لها، وليست بالضرورة لنفس الجهة.
- نقطة الدخول: ودجت (widget) طرفية مزروعة في صفحة المصدر (قسم التنزيل الحقيقي فيها 264 زرار Short-Links سليم)، والبوابة نفسها تحويل متدرّج لكل رابط على نطاق أم عمره يومان تقريبًا.

## السلسلة

```
farescd.com/software/easeus-partition-master/
   |  زرار مزروع «تحميل الملف مع التفعيل» (class file-download-btnn، جافاسكربت base64)
   |  كشف DevTools + فحص هندسة النافذة + قناة أوامر «mr2127a58» (صلاحية ساعة)
   v
*.m5u1iioax.store  (save-njbg / blob-imrx / upload-atd5 / box-tfdg / mirrorgcve)
   |  أسماء مستعارة: DropVault - Icecloud - GoFiles - WeShare - ShareBox
   |  «ملفك جاهز للتنزيل» -> Download File      [خلف Cloudflare؛ 404 للماسحات]
   v
"EaseUS_Partition_Master_2026..._Archive_free_3295.zip"  (~700-800 MB)
   |  حزمة Ren'Py 8.1.3 + Setup.exe غير موقّع  (ليس EaseUS إطلاقًا)
   v
ملفا MSI وهميان: "Killer Network Service 2.6" / "PhysX Runtime 7.0"
   |  Custom Action نوع 51+242:  cmd /c start "" <Project>.lnk
   v
حل وهمية "CatalogIngestor": MSBuild.exe (HashMismatch) + Resources.resx بحجم 31.6MB
   |  دوال الخصائص: حذف 'p' -> XOR-6 -> تدوير @142132 -> SoapHexBinary.Parse
   v
AppDomain.CurrentDomain.Load -> CreateInstance("CloneExtensions.FallbackProvider5")
   |  حمولة ~72KB بلا ملفات  ->  Trojan.PavinLoader (مهام dllhost / taskhostw)
   +  إصابات مرافقة: Glupteba وRanumbot وAdware (ليست بالضرورة نفس الجهة)
```

## المرحلة 1: الودجت المزروعة

صفحة المصدر هي صفحة تنزيل EaseUS Partition Master على مكتبة برمجيات عربية «بنسخ مفعّلة». قسم التنزيل الحقيقي فيها يحوي 264 زر `short-links` عاديًا بلا أي مؤشر خبيث. لكن لقطة الـ HTML (Elementor) احتوت ثلاث نسخ من زر إضافي: class `file-download-btnn` والتسمية «تحميل الملف مع التفعيل»، يحمل سكربت `data:text/javascript;base64,…`. لم تُثبت الأدلة هل وصل عبر فتحة إعلانية مخترقة أم حقن طرفي (انظر ملاحظة الإسناد).

بعد فك الترميز ساكنًا، السكربت:

1. يبدأ برأس `87942;` ويستخدم مُبسّط سلاسل مخصصًا لبناء السكربت النهائي.
2. يكشف فتح أدوات المطور عبر مصيدة توقيت `Function('debugger')` ومصيدة `console.log` getter، ويتوقف عند أيٍّ منهما.
3. لا يرسم إلا في تبويب حقيقي: `window.top === window` و`outerWidth − innerWidth > 160`، وليس في نافذة صغيرة.
4. يقرأ قناة أوامره من `localStorage['mr2127a58']` ومن الكوكيز `mr2127a58N…` بصيغة `timestamp|command`، ويبقى الأمر صالحًا ساعة واحدة (`t − ct < 36e5`)؛ أوامر المرحلة الثانية بترميز base64 مع إزاحة ‎−1 لكل حرف وتُنفَّذ عبر `Function(...)`.
5. ينتهي بذيل خداعي: `(()=>{const _kpo=1,_nvz=2})();(()=>{let _xfk=0})()`.

الرسم مشروط باجتياز البوابات الثلاث، وهذا يفسّر ظاهرة «الإعلانات اختفت» (انظر الدروس).

## المرحلة 2: بوابة الاستضافة المزيفة

الزر يقود لنطاقات فرعية من نطاق أم جديد `m5u1iioax.store` (`save-njbg` و`blob-imrx` و`upload-atd5` و`box-tfdg` و`mirrorgcve`)، كلٌّ منها يعرض واحدة من خمس أغلفة استضافة مزيفة: DropVault وIcecloud وGoFiles وWeShare وShareBox، بصفحة «ملفك جاهز» وشارات ثقة مزيّفة وزر Download File. الصفحات ذكرت أن الأرشيف رُفع في 2026-10-27.

بيانات الفحص العامة (urlscan.io بتاريخ 2026-10-28) لنطاق شقيق تكشف التخفّي: واجهة Cloudflare (AS13335)، شهادة TLS صدرت في 2026-10-24، استجابة HTTP 404 للماسح، وكتلة حالة لكل رابط:

```json
{"id":"bxlz4","tag":"6bnS3H2T:C1","file":"Download Link - Archive Free Software And Tools","redirectStep":0}
```

المتصفح المُحال يحصل على القمع؛ وكل ما عدا ذلك لا يحصل على شيء.

## المرحلة 3: حزمة التنكّر

ملف ZIP المسلَّم (~700–800MB باسم `…Archive_free_3295.zip`) يحتوي توزيعة Ren'Py 8.1.3 كاملة (مُشغّلات Python وأرشيف لعبة RPA بحجم 145KB)، **وبلا أي ملفات EaseUS إطلاقًا**. ملف `Setup.exe` في الجذر غير موقّع وبلا معلومات إصدار، ببصمة `7123e151…888d4ab`، ووصفه لاحقًا Malwarebytes بأنه Generic.Malware/Suspicious. عند تشغيله في 2026-10-28 جهّز عدة المحمّل وثبّت ملفّي MSI خلال دقائق؛ ثم ثُبّت منتج EaseUS الحقيقي في اليوم نفسه بعده.

## المرحلة 4: ملفّا MSI المنتكران

بُني المثبّتان بـ WiX 3.11 في 2026-10-26 من قالب واحد شبه متطابق:

| | «Killer Network Service 2.6» | «PhysX Runtime 7.0» |
|---|---|---|
| المُصنِّع | Qualcomm Technologies Inc. | NVIDIA Corporation |
| ProductCode | `{96447065-A735-4A79-BA59-A2679E7BAFEF}` | `{3B086246-E935-41F8-BE45-676EF3138112}` |
| مجلد الإسقاط | `%LOCALAPPDATA%\Qualcomm\Killer Network Service` | `%LOCALAPPDATA%\NVIDIA\PhysX Runtime` |
| يُشغّل | `Antlr3Runtime.lnk` | `MasterDataManager.lnk` |

حمّلهما السلوكي الوحيد زوج Custom Actions (نوع 51 + 242، التسلسل 6601/6602، بشرط `NOT REMOVE~="ALL"`، أي عند التثبيت/الإصلاح فقط) ينفّذ `cmd.exe /d /s /c "cd /d <drop dir> && start "" <Project>.lnk"`.

التنكّر ينهار أمام فحصين: برنامج NVIDIA PhysX System Software الحقيقي لم يصدر قط بأي إصدار غير 9.x، وحزم Killer الحقيقية من جيل Rivet (1.x) أو جيل Intel (40.x/50.x): لا وجود لـ«7.0» ولا لـ«2.6». وكارت الشاشة AMD والشبكة Realtek في الجهاز لم يعطيا أي مبرر لوجود أيٍّ من المنتجين أصلًا.

## المرحلة 5: المحمّل داخل MSBuild

كل MSI يُسقط حل تطوير وهمية: «CatalogIngestor 2.3.0.0» من «Ingestion Labs»، بسجل تغييرات مفبرك ببطاقات `DQ-*` ومراجع NuGet خداعية و`Directory.Build.props` سليم تمامًا. الملفان المهمان:

- **`MSBuild.exe`** (إصدار 4.8.9032.0): شهادة Microsoft حقيقية (بصمة `5A858500A0262E237FBA6BFEF80FA39C59ECEE76`) لكن `Get-AuthenticodeSignature` تُعيد `HashMismatch`: أي عُدّل بعد التوقيع.
- **`Resources.resx`** (31,618,579 بايت): XML مقصوء عمدًا (resheaders غير مكتملة) بحيث لا يُقرأ أبدًا إلا كنص خام. Res1–Res4 تحمل الحمولة؛ وRes5–Res127 (~31MB) كتلة خداع.

**`Internal.targets`** (3,671 بايت) بلا `UsingTask` ولا `CodeTaskFactory` ولا أي خطّافات Targets. كل شيء يجري في دوال الخصائص التي تُقيَّم أثناء تقييم الخصائص، قبل أي Target، وبأقل ظهور أمام الأدوات الساكنة:

1. `File.ReadAllText` يسحب ملف resx كاملًا داخل خاصية.
2. أربعة تعابير نمطية تستخرج الكتل عبر أسماء مستعارة معرّفة في ملفات أخرى «بريئة» (`PackageIcon=Res1` و`NuGetAuditSuppress=Res3` في `.csproj.user`؛ و`PackageTags=Res2` و`Description=Res4` في `Shared.props`).
3. الكتل المجمّعة تُنزع فواصل `p` لكل بايت؛ ثم استبدال محرفي من مرحلتين يطبّق XOR-6 على كل نايبل سداسي.
4. `Substring(142132) + Substring(0,142132)` تدير التدفق السداسي.
5. `SoapHexBinary.Parse(...).Value` تحوّل السداسي إلى بايتات.
6. `AppDomain.CurrentDomain.Load(bytes).CreateInstance("CloneExtensions.FallbackProvider5")` تحمّل التجميعة في الذاكرة وتُنشئ النوع: نقطة الدخول بلا ملفات.

من أطوال الكتل المخزنة (216,575 محرفًا) فإن حجم الحمولة ≈72KB: نتيجة مشتقة؛ لم تُفكّ ولم تُنفَّذ قط.

## الاستمرارية وما تلاها

فحص Malwarebytes في 2026-10-29 سجّن 109 عناصر وسمّى العائلة Trojan.PavinLoader: مهام مجدولة مزيفة في `C:\Windows\System32\Tasks\dllhost` و`…\taskhostw` بمشغّلات تسجيل دخول، ومجلدات تجهيز متنكّرة تحت `Microsoft\Protect` و`Crypto` و`Themes` و`Libraries` و`Windows\INetCache` (باسمَي `ANTLR3RUNTIME`/`MASTERDATAMANAGER`) و`WebCache` و`Notifications` و`Explorer`.

كشف الفحص نفسه إصابات مرافقة أقدم من الحملة أو موازية لها، وليست بالضرورة لنفس الجهة: Trojan.Glupteba (`EpicNet Inc\CloudNet\cloudnet.exe`)، وTrojan.Ranumbot (عدة في `%TEMP%\csrss`: `tor.exe` و`obfs4proxy` و`i2pd` وماسحات SMB)، وزوج الإعلانات PhoenixInvicta (Chrome) + PUP.Optional.SquareStartPage (Edge). تاريخ طويل من الكراكرات على الجهاز هو المفسّر المشترك الأرجح.

## واقع الكشف

- **VirusTotal (الـ MSI الأول، `fc870a5e…3536c5`): 2/62**: Kaspersky `HEUR:Trojan.WinLNK.Agent.gen` (عائلة الاختصارات الخبيثة؛ ينسجم مع الإطلاق عبر `.lnk`)، تسمية `trojan.winlnk`؛ وسم سلوكي `checks-usb-bus` (غير موثق علنًا؛ استنتاجنا: تعداد USB/PnP في البيئة الرملية، مقدمة لانتشار عبر الأقراص المتنقلة)؛ تقييم مجتمعي −51.
- **Smadav: صفر اكتشافات من 1,401 ملف**: محرك «الحماية الإضافية» المثبّت كان أعمى عن الحزمة بأكملها.
- **Microsoft Defender:** لا اكتشافات ولا Run Keys وقت الفحص؛ المهام المجدولة كانت الاستمرارية الوحيدة المكتشفة وسجّنها Malwarebytes لاحقًا.

## الخط الزمني (تواريخ فقط)

| التاريخ | الحدث |
|---|---|
| 2026-10-24 | إصدار شهادة TLS للبوابة؛ النطاق الأم عمره يومان تقريبًا وقت الحادثة. |
| 2026-10-26 | بناء ملفّي MSI الوهميين (من أختام حزم WiX). |
| 2026-10-27 | صفحات الاستضافة المزيفة تعرض الأرشيف كمرفوع. |
| 2026-10-28 | يوم الإصابة: بوابة شقيقة تخدم القمع علنًا؛ تشغيل `Setup.exe`؛ تثبيت الـ MSI معًا؛ استمرارية PavinLoader؛ ثم تثبيت EaseUS الحقيقي بعدها. |
| 2026-10-29 | كتابة ذاكرة Ren'Py المؤقتة (تشغيل الحزمة)؛ Malwarebytes يسجّن 109 عناصر؛ هذا التحليل. |

## الدروس المستفادة

1. **«الإعلانات اختفت» تعني تحكيمًا في العرض لا تنظيفًا.** الرسم يتطلب لا DevTools، وهندسة تبويب حقيقي، وأمرًا أحدث من ساعة في `mr2127a58`؛ وعندما تتوقف الحملة عن إصدار الأوامر يبقى الإطار فارغًا.
2. **أرقام الإصدارات IOCs مجانية.** «PhysX Runtime 7.0» و«Killer Network Service 2.6» لم يوجدا يومًا؛ قارن التثبيتات بخطوط المورّدين وبالعتاد الفعلي (لا NVIDIA ولا Killer هنا).
3. **التوقيع الصالح ليس حكمًا نهائيًا.** اشتغل المحمّل تحت ثنائية موقّعة بتوقيع Microsoft حقيقي؛ تحقق من *حالة* التوقيع — `HashMismatch` تعني عُدّل بعد التوقيع — لا من مجرد وجود الشهادة.
4. **دوال الخصائص تتفوق على Inline Tasks — للمهاجم.** قواعد الكشف التي تبحث عن `CodeTaskFactory` لن تجد شيئًا هنا؛ راقب `MSBuild.exe` خارج مجلدات Framework، وملفات resx بحجم عشرات الميغابايتات، و`AppDomain.CurrentDomain.Load` داخل ملفات المشاريع (MITRE ATT&CK T1127.001).
5. **2/62 على VirusTotal متوقَّع لا مُطمئن.** بناء كل حملة على حدة يُبقي الانتشار قريبًا من الصفر؛ الدرجة المجتمعية −51 قالت أكثر من عدد المحركات.
6. **محرك ضعيف واحد لا يساوي محركًا.** Smadav: صفر من 1,401 ملف؛ أدوات سلوكية طبقية: 109 عناصر.
7. **التقط ودجات الصفحة الحقيقية قبل الإسناد.** أزرار الـ short-links الأصلية الـ264 تثبت أن الزر الخبيث «مزروع في الصفحة» لكنه ليس «منها».
8. **الاستضافة المزيفة عدة جاهزة تتداول.** أغلفة DropVault/Icecloud موثقة في مشاهد تنزيل متعددة؛ احظر *النمط*: نطاق جديد، نطاقات فرعية عشوائية، 404 للماسح، كتل حالة لكل رابط، لا نطاق اليوم فقط.

## الأدلة وIOCs

**SHA-256**

```
fc870a5ed14512ab04816681ce7fdab545e455e8e74f76f394486192f33536c5   MSI-1 "Killer Network Service" (VT 2/62)
55c42ed6c1c610b78e99ade2ac9e47648aafdc2d8112c621b4a1f25c2497866d   MSI-2 "PhysX Runtime"
6bc58e85a42bcf60df74c23de212c882d8fdc5e6a60f20f5f3c5e392da6b0ad0   MSBuild.exe المرفق 4.8.9032.0 (MD5 2024ddf63513a9ea309091339a84a656)
30d16380654c4cb4eebd6c6cdc5ce1d13c04f2ad3467fd9e1514c7df1471c993   Resources.resx (حاملة 31.6MB)
e20c494d8d081353f4d188ba88304028496165467486903338142a94b634ef5a   Internal.targets (المحمّل)
7123e1514b939b165985560057fe3c761440a9fff9783a3b84e861fd2888d4ab   Setup.exe داخل حزمة Ren'Py (غير موقّع)
b1f19e6504e31fbe0e3af05da792cb7fac8a9b7427f0928d6d26cdfe75d3c7bf   ملف ZIP المسلَّم
```

**هويات الـ MSI:** UpgradeCodes `{392F3156-6442-4173-B0B8-20E198E37E9D}` / `{E147E7CC-0C80-4705-826C-4D9982F08D37}`؛ PackageCodes `{6230FAE6-CC8B-4372-8054-09EC986D56AB}` / `{FC482E92-9A99-4FAF-BAB8-7C57E01C0720}`؛ الإجراءات المخصصة `SetCA_/CA_c61e54380f464eee81bcd885b78b865f` و`SetCA_/CA_072d6bab34de4524a75d7d80ce1ea576`.

**الشبكة:** النطاق الأم `m5u1iioax.store`؛ النطاقات الفرعية `save-njbg` و`blob-imrx` و`upload-atd5` و`box-tfdg` و`mirrorgcve` و`inspectv2lf`؛ صفحة المصدر `farescd.com/software/easeus-partition-master/` (الصفحة التي رُصدت فيها الودجت المزروعة)؛ Cloudflare AS13335، 188.114.96.3.

**أدلة على الجهاز:** `%LOCALAPPDATA%\Qualcomm\Killer Network Service\`؛ `%LOCALAPPDATA%\NVIDIA\PhysX Runtime\`؛ `C:\Windows\System32\Tasks\dllhost` و`…\taskhostw`؛ تجهيزات تحت `AppData\{Roaming,Local}\Microsoft\{Protect, Crypto, Themes, Libraries, Windows\INetCache, WebCache, Notifications, Explorer}`؛ `%TEMP%\csrss\` (Ranumbot)؛ `%APPDATA%\EpicNet Inc\CloudNet\` (Glupteba)؛ إضافة Chrome‏ `iaddfgegjgjelgkanamleadckkpnjpjc`؛ إضافة Edge‏ `hnjlcpngjhdoehbjfkmhjagmmojcimnc`.

**سلاسل وأسماء ملفات:** `CloneExtensions.FallbackProvider5`؛ إزاحة التدوير `142132`؛ خريطة XOR-6؛ مفتاح القناة `mr2127a58`؛ رأس السكربت `87942;`؛ مسار NuGet‏ `iqbydwfv`؛ التنكّر «Ingestion Labs» / «CatalogIngestor» 2.3.0.0 / بطاقات `DQ-*`؛ `Res1`–`Res127` (الأسماء المستعارة `PackageIcon=Res1`، `PackageTags=Res2`، `NuGetAuditSuppress=Res3`، `Description=Res4`)؛ الملفات `build.cmd` و`Antlr3Runtime.lnk` و`MasterDataManager.lnk` و`CatalogIngestor.csproj` و`Internal.targets`.

## ملاحظة الإسناد

الزر الخبيث كان ودجت طرفية مزروعة رُصدت في صفحة المصدر؛ تنزيلات الموقع الحقيقية روابط short-links عادية، ولا يوجد أي تقرير علني يربط الموقع نفسه بتوزيع مالوير. عدة الأغلفة من نمط DropVault/Icecloud تعمل بنفس الشكل في مشاهد تنزيل متعددة — بنية تحتية سلعية مشتركة، وليست دليلًا على جهة واحدة.

كل الهاشات والقواعد منشورة في هذا المستودع.
