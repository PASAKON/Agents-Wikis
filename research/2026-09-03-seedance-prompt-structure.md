# Seedance 2.0/2.5 prompt structure — community practice vs our own skill

**Researched** 2026-09-03 · CTO session 116d7688 · workflow run `wmamv83fu`  
**Method** CEO asked whether the community writes Seedance prompts the way our skill does. 10 agents, ~1.1M tokens, six search angles plus an adversarial critic.

> Agents were instructed to return empty rather than invent sources. Confidence levels are stated per finding inside.

---


## COMPARISON

# Seedance prompt structure: community vs ของเรา

สรุปหัวข้อเดียวก่อน: **โครงสร้างของเราตรงกับรูปทรงที่ดีที่สุดที่ community ใช้อยู่แล้ว ไม่ต้อง rewrite** สิ่งที่ควรทำมี 3-4 อย่างเล็กๆ ที่เป็น additive ล้วน แต่มีข้อเท็จจริงจาก official doc หนึ่งข้อที่ต้องเช็คด่วน เพราะมันกระทบว่า timeline ของเราถูกอ่านหรือไม่

---

## 1. โครงสร้างที่ community ใช้จริง

### ชั้นของหลักฐาน (สำคัญกว่าตัวโครงสร้าง)

- **[ทางการ]** BytePlus/ModelArk docs อ่านถึงตัวหน้าจริง: Seedance 2.5 guide (docs.byteplus.com/en/docs/ModelArk/2607689), 1.0-pro guide (1631633) อ่านเต็ม; 2.0 guide (2222480) และ 1.5-pro (2168087) กู้จาก embedded JSON ได้แค่บางส่วน + ByteDance Seed technical report (arXiv 2506.09113)
- **[ผู้ใช้จริง]** fal.ai (เจ้าของ inference, ลง prompt คู่กับคลิปที่ generate เอง), beech (Substack, prompt เต็มชุด), MindStudio (ทำหนังสั้น 2 นาทีจริง), Curious Refuge (stress test 15 refs), Higgsfield blog (แพลตฟอร์มที่เราใช้)
- **[มือสอง]** OpenArt, Segmind, Kapwing, DeepBrain, GitHub skills (dexhunter, issastash, Emily2040)
- **[อ่อน]** SEO farm อีกหลายสิบเว็บ ก๊อปกันเอง ไม่มี citation ตัดทิ้งหมด

**Reddit / X / Discord = ศูนย์** ทุก route โดนบล็อกหมด (reddit ไม่อยู่ใน index, x.com คืน HTTP 402) เพราะฉะนั้นอย่าเข้าใจว่านี่คือ "เสียงชุมชน" มันคือ docs + blog ของคนที่ผลิตงานจริงประมาณ 5 คน

### ข้อแรกที่ต้องรู้: "official six-part formula" ส่วนใหญ่เป็นตำนาน

เว็บจำนวนมาก (Segmind พาดหัวว่า "ByteDance's Six-Part Formula" ตรงๆ) อ้างสูตร 6 ช่อง Subject + Action + Scene + Style + Camera + Audio ว่าเป็นของ ByteDance **แต่ไม่มีใคร link doc เลย** และคนที่ไปเปิดหน้า 1.0-pro จริงยืนยันว่าประโยคนั้นไม่มีอยู่ในหน้านั้น

ของจริงที่อ่านได้ที่ต้นทาง:

| version | โครงสร้างทางการ |
|---|---|
| **2.5** | 4 บล็อก: (1) Asset referencing (2) One-sentence summary "Subject + Location + Event + Genre/Style + Camera movement" (3) Detailed plot ตาม timeline หรือ shot (4) Additional notes = สิ่งที่ต้องคงที่ตลอด |
| **2.0** | 5 ขั้น: นิยาม subject → bind reference ต่อชนิดไฟล์ → shot list "Shot 1, Shot 2" → ในแต่ละ shot: camera / actions+expressions / position changes / audio → tail: image quality, style, constraint words |
| **1.5-pro** | สูตรเดียวที่ ByteDance เขียนเป็นสูตรจริงๆ: `Subject + Movement + Environment (optional) + Camera movement (optional) + Aesthetic (optional) + Sound (optional)` **บังคับแค่ 2 ช่องแรก** |
| **1.0-pro** | ไม่มีสูตรหลายช่องเลย มีแค่ "Beginner: subject+action" |

### ข้อที่สำคัญที่สุดต่อเรา (verbatim จาก official 2.5 guide)

> "Seedance 2.0 does not respond to timestamps and only responds to shot numbers, while Seedance 2.5 supports integer-second timestamps."

นี่ [ทางการ] และมันอธิบายความขัดแย้งใหญ่ที่สุดใน community ได้หมด: ค่าย `[0-6s]` (fal, beech, techhalla, OpenArt) กับค่าย `Shot 1:/Shot 2:` (Higgsfield 2.5 guide, Emily2040 repo) **ไม่ได้เถียงกัน คนละ version กัน** Emily2040 เขียนสำหรับ 2.0 และพูดตรงๆ ว่าอย่าใช้ bracketed block แทน label ซึ่งถูกต้องสำหรับ 2.0

กฎ timestamp ทางการเพิ่ม: ใช้หน่วย 1 วินาที, ห้ามมีช่องว่างระหว่างช่วง (ห้าม "0-3s... 5-6s"), รับได้ทั้ง `0-3 seconds`, `[1s-4s]`, `at the 5-second mark`, `After 3 seconds`

### โครงกระดูกที่ตรงกันจริง (เรียงตามความแข็งของหลักฐาน)

1. **Reference declaration มาก่อน แต่ละอันมี job เดียว + ประโยคห้ามชัดๆ** [ทางการ + ผู้ใช้จริง หลายเจ้าอิสระกัน] เป็นเทคนิคที่ถูกพูดซ้ำมากที่สุดในทั้ง corpus
   - fal: `@Image1 controls only [X]. Do not copy [pose, background, lighting, text] from @Image1.`
   - official 2.0: `Define [Core_Subject_Features] in <Image_N> as <Subject_N>` และเตือนตรงๆ ว่าอย่าเขียนชื่อบนรูปแล้วอ้างชื่อในprompt "can easily cause character confusion or duplication"
   - official 2.5: binding ต้องอยู่ใน**ข้อความ** เรียงตามลำดับ upload เท่านั้น
2. **Beat แบ่งเวลา** [ทางการสำหรับ 2.5] รูปแบบตาม version ข้างบน
3. **Camera ใช้ศัพท์ operator ตรงๆ** [ทางการ] official 2.0: "The model has a strong understanding of camera movement terms" ทั้ง 2.0/2.5 บอกว่าเขียน push in / pan / orbit / dolly zoom ได้เลย และ "cinematic" ไม่ใช่ camera move
4. **Continuity = ไล่ชื่อสิ่งที่ห้ามเปลี่ยน ไม่ใช่คำว่า consistent** [ผู้ใช้จริง, fal + Higgsfield + Curious Refuge อิสระกัน]
5. **Audio ระบุเสมอ** [ทางการ + ผู้ใช้จริง] ทุกคนตรงกันว่า music ใส่ตอน edit ไม่ใช่ตอน generate
6. **Physical verb ไม่ใช่ adjective** [ผู้ใช้จริง] official 2.0 มีหัวข้อ "Body-movement refinement + degree quantification" รองรับ

### ที่ไม่ตรงกัน (พูดตรงๆ ว่าไม่ตรง ไม่เฉลี่ย)

| ประเด็น | ฝั่ง A | ฝั่ง B | อ่านยังไง |
|---|---|---|---|
| **Timestamp vs Shot N** | fal/beech ใช้ `[0-6s]` | Higgsfield 2.5 guide + Emily2040 ใช้ `Shot 1:` | **แก้แล้วด้วย official**: เป็นเรื่อง version |
| **Beat density** | beech: montage 13 beats/15s ยิงแล้วไปเลือกตัดเอง | MindStudio (ทำหนังจริง): 2.5 "resists prompt styles built around rapid, one-second-style cuts" scene ที่หายใจได้ออกมาดีกว่า | ขัดกันจริง แต่น่าจะเป็น montage vs narrative คนละงาน ทั้งคู่ agree ว่า narrative = 3-4 beats/15s |
| **จำนวน reference** | vendor: 50 refs! (ByteDance, Higgsfield, fal API) | MindStudio: 3 refs พาหนัง 2 นาทีทั้งเรื่อง; Curious Refuge: ~15 refs แล้วตัวเอก "randomly turned into the Green Lantern" | **หลักฐานมือหนึ่งเอนไปทาง น้อยแต่ดี** ตัวเลข 50 คือ spec sheet ไม่ใช่ผลทดสอบ |
| **Negative** | official 2.5: แคบมาก รองรับแค่ subtitles กับ audio ("no subtitles", "no BGM") ไม่มี general negative | ทุกคนที่ยิงงานจริงเขียน negative ยาวๆ inline แล้วบอกว่าใช้ได้ + official 2.0 **มีช่อง constraint words จริง** + ตัวอย่างของ official 2.5 เองมีบล็อก "[Strictly exclude]" ขัดกับกฎในหน้าเดียวกัน | doc ขัดกับตัวอย่างของตัวเอง เชื่อคนที่ยิงงาน |
| **ความยาว prompt** | SEO farm: 50-70 คำคือ sweet spot | Higgsfield 2.5 (ทดสอบเอง): 800-3,500 คำ; beech: <3,500 ตัวอักษร | ตัวเลข 50-70 ไม่มีการทดสอบรองรับเลย **ไม่มี first-party max ที่ไหนทั้งสิ้น** |
| **@ syntax** | Higgsfield help center: @character/@style/@motion/@audio; Elements: @ElementName; blog หนึ่ง: @Image 1 | Higgsfield 2.5 flagship guide: **ไม่มี @ เลย** ใช้ CAPS name + POSITIVE LOCKS | vendor เดียวมี 4 แบบ ไม่มี "the Higgsfield way" |

### เรื่องที่ต้องระวังเพิ่ม

- **ไม่มี negative_prompt field ใน API ของ Seedance** (Kapwing ยืนยัน, fal guide ไม่พูดถึงเลย) ใครบอกให้กรอกช่อง negative คือกำลังพูดถึง Kling อยู่
- **@Video ไม่มีใน doc ทางการของ Higgsfield** tag video ของเขาคือ @motion แหล่งเดียวที่อ้าง @Video คือ GitHub skill ของบุคคลที่สาม
- **JSON prompting = Veo ไม่ใช่ Seedance** อย่าเอามาใช้
- **มี rewriter คั่นอยู่** technical report ระบุว่า user prompt ถูก SFT+DPO rewrite เป็น "dense caption expression" ก่อนถึง DiT ซึ่งแปลว่าลำดับ field อาจสำคัญน้อยกว่าที่ทุกคนคิด สิ่งที่สำคัญคือ**ข้อมูล dynamic/static มีครบและไม่กำกวม** (นี่คือ framing ของ paper เอง ส่วนข้อสรุปเรื่องลำดับเป็นการตีความ)
- **ByteDance มี official skill ชื่อ `sd25-pe`** และ doc บอกว่า "We strongly recommend using the Seedance 2.5 Skill" ยังดึงตัวไฟล์มาไม่ได้ นี่คือของชิ้นเดียวที่จะปิดคำถามเรื่องโครงสร้างได้ขาด

---

## 2. โครงสร้างของเรา

เขียนด้วยศัพท์เดียวกันเพื่อวางเทียบได้ 10 ชั้น

| ชั้น | ของเรา | เทียบกับศัพท์ community |
|---|---|---|
| 0 | **File preamble** provenance guillemets + คำเตือน Element id ตัวพิมพ์ใหญ่ + project URL + cast bible พร้อม fixed head-counts | ไม่มีใครมี |
| 1 | **Divider + title** `===` = scene/pack, `---` = shot; `ID · ALL-CAPS NAME — gloss` | ไม่มีใครมี |
| 2 | **Tech header** บรรทัดเดียวคั่นด้วย `·`: duration · res · aspect · camera doctrine (ตั้งชื่อ + ย้ำเป็นประโยคปฏิเสธ) · dialogue flag | = fal `FORMAT`, = Dreamina ช่อง 1, = Higgsfield "specify shot structure upfront" |
| 3 | **REFERENCES — each with a job** รูปแบบ `@element_id — หน้าที่` job เขียนเป็นหน้าที่ ไม่ใช่คำบรรยาย มี ignore-clause (`Not the framing.`, `take the look, never the count`) มี ⚠️ ล็อกจำนวน inline และบางครั้ง correction ทั้งย่อหน้าฝังอยู่ใน job | = fal `REFERENCE ROLES` + exclusion, = official 2.0 subject definition |
| 4 | **Set-dressing fact line** บรรทัดเดียวไม่มี label บอกสิ่งที่เปลี่ยนจากซีนก่อน | ไม่มีใครมี (ใกล้ที่สุดคือ fal `STARTING STATE`) |
| 5 | **`[Ns]` beats** วินาที absolute, 2-4 beat, beat แรก = เฟรม, beat สุดท้าย = hold; dialogue inline พร้อม tone tag นำหน้า | = fal `TIMELINE`, = official 2.5 timestamp |
| 6 | **Per-shot override block** ⚠️ หัวข้อระบุวันที่ + เหตุผล bullets แบบ "อาการที่เจอ → กฎ" | ไม่มีใครมี |
| 7 | **Colour grade** ระดับไฟล์ ไม่เคยอยู่ในบล็อก | ≈ beech "one style prompt, copy paste don't rewrite", ≈ Higgsfield `Style:` prefix |
| 8 | **`AUDIO:`** 3 ส่วนตายตัว: แหล่งเสียง → `No music.` → ประโยคห้ามเสียงหัวท้าย | = fal `AUDIO` แต่ท่อนหัวท้ายไม่มีใครมี |
| 9 | **`CRITICAL NEGATIVES:`** boilerplate 2 ท่อนก่อน (positive form ห้ามหน้าซ้ำ + ห้ามเสียงหัวท้าย) แล้วค่อยข้อห้ามของ take นี้ ปิดด้วย pointer `+ house negatives.` | = fal `CONSTRAINTS` แต่ **การจัดลำดับ aimed-first เป็นของเราเอง** |
| 10 | **House negatives** ระดับไฟล์ ~233 คำใน s6-s18, ~110 ใน s1-angles, **ไม่มีเลยใน s-extras** | ไม่มีใครมี |

ที่ยิงจริงอยู่ราว **300-950 คำ** และ skill ยังพก: interview ก่อนเขียนทุกครั้ง, 10 house rules, 8 hard rules, เพดาน chip ~10, เพดาน 20s (วัดเอง), `@Video 1` มีเว้นวรรค, เศรษฐศาสตร์ของ chain, one-noun-per-character, describe-before-use, roster ต่อ role

---

## 3. ตรงกันตรงไหน / ต่างตรงไหน

### ตรงกัน และเราไม่ได้ตามหลัง

| หัวข้อ | ของเรา | ของเขา | ตัดสิน |
|---|---|---|---|
| Tech spec บรรทัดแรก | มี + ย้ำ camera doctrine สองรอบ | fal มี, Higgsfield 2.0 บอกให้ระบุ upfront, ByteDance ไม่มี (เป็น UI param) | **เราดีกว่านิดหน่อย** การย้ำซ้ำถูกทางเมื่อรู้ว่ามี rewriter คั่น แต่เรายังไม่เคย A/B เหมือนกัน |
| Per-reference job | มีทุกอัน + ignore clause | fal เป็นสูตรตายตัว `controls only X / Do not copy Y` | **เสมอ เอียงไปทางเขานิดเดียวเรื่องความสม่ำเสมอ** ของเราเป็น ad hoc phrasing |
| Beat density | 2-4 beat ต่อคลิป | beech: 3-4/15s narrative; MindStudio: อย่ายัดคัต 1 วิ | **เราอยู่ฝั่งที่ถูก** และมีหลักฐานมือหนึ่งสองรายหนุน |
| ลำดับใน beat | shot type → subject+action → camera → atmosphere | official 2.0 ระบุ 4 อย่างต่อ shot เกือบตรงกัน | **ตรงกับ primary source** |
| Dialogue tone ก่อนคำพูด | มี + budget 45-50 คำ/20s | fal/beech/official ทำ tone-first เหมือนกัน แต่**ไม่มีใครให้ตัวเลข word budget** | **เราดีกว่า** |
| Audio block ท้าย ไม่ใช่ per-beat | block เดียว | official 2.0 ใส่ audio ในทุก shot, beech ก็ inline per beat; **แต่ official 2.5 ตัวอย่างแพนด้าใส่ audio เป็นบล็อกท้าย** | **เราตรงกับ 2.5 primary** เรื่อง per-beat เป็นของ 2.0 อีกแล้ว |
| One noun per character | กฎบ้าน 10 | official 2.0 บอกให้ define แล้วใช้ label เดิมตลอด (police officer / thief) | **primary ยืนยันของเรา** |
| Describe before use แม้มี ref | hard rule 1 | Higgsfield/issastash ทำเหมือนกัน, official 2.0 ก็ define จาก image | **ตรงกัน** |
| Head-count | roster ต่อ role + role/เสื้อผ้า/ตำแหน่ง/งาน ต่างกันทุกคน + เหตุผลว่าตัวเลขเปล่าไม่ช่วยเพราะโมเดลเช็คตัวเองไม่ได้ตอนวาด | official 2.0 มี boilerplate ห้ามตัวซ้ำ, Higgsfield มี "MEMBER COUNT LOCK" (ตัวเลขเปล่า) | **เราดีกว่าชัดเจน** เหตุผลของเราคมกว่าและใช้ได้จริงกว่า |
| Negative จัดลำดับ | 5-8 ตัวที่เคยฆ่า take ขึ้นก่อน แล้วค่อยกำแพง | fal/beech/AIwithkhan เขียนยาวรวดเดียวไม่จัดลำดับ | **เราดีกว่า** และสอดคล้องกับพฤติกรรม front-loading ของโมเดล |
| Shared file-level block | grade + house negatives + cast bible แยกออกมา | beech "copy paste don't rewrite", Higgsfield `Style:` prefix เท่านั้น | **แนวคิดมีคนคิดแล้ว แต่วินัยระดับนี้ไม่มีใครทำ เราดีกว่า** |

### ต่างกัน และเขาดีกว่า

1. **`ENDING STATE`** [fal, ผู้ใช้จริง] เราไม่มี beat สุดท้ายของเราคือ "Hold." ซึ่งบอกว่าไม่ขยับ แต่ไม่ได้บอกว่า**ต้องจบที่ตำแหน่งไหน** สำหรับงานที่ chain ด้วย `@Video 1` และการแก้ที่ link 3 จาก 14 = re-render 12 ครั้ง ~4 ชม. อันนี้ตรงกับเศรษฐศาสตร์ของเราเป๊ะ
2. **Exclusion เป็นช่องตายตัว** ของเรากระจายเป็นสำนวนหลายแบบ ทำให้บางครั้งลืม
3. **Accent/language line ใน dialogue** [MindStudio, ผู้ใช้จริงรายเดียว] เขาเจอว่าเสียงถูก**เดาจาก image reference** ตัวละครพูดสำเนียงบริติชโดยไม่ได้สั่ง แก้ด้วยคำว่า "American" ได้ แต่ "American English" ไม่ได้เพราะคำว่า English ดึงกลับไปทางบริติช เรามี tone แต่ไม่มีช่องสำเนียง

### ต่างกัน และเราดีกว่า (ของที่ต้องปกป้อง ไม่ต้องแก้ตาม)

- **20s ceiling** ของเราเป็นค่าที่**วัดบนแพลตฟอร์ม** community พูดถึง 30s ซึ่งเป็น API-level ของ 2.5 ไม่ใช่สิ่งเดียวกัน (แต่ควรวัดใหม่ ดูข้อ 4)
- **chip ceiling ~10** ตัวเลข community คือ 9 / 12 / 50 ซึ่งขัดกันเองในเว็บ vendor เดียว ของเราคือ**เกณฑ์ที่ผลลัพธ์เริ่มพัง** ไม่ใช่เพดาน spec และ Curious Refuge พังที่ ~15 ซึ่งสอดคล้องกัน
- **`@Video 1` มีเว้นวรรค** เป็นกฎที่วัดจากแพลตฟอร์มจริง ส่วน @Video ของ community ไม่มี first-party รองรับเลย

### จุดอ่อนที่ไม่เกี่ยวกับ community เลย เป็นของเราเอง

- **s-extras ไม่มีทั้ง grade block และ house negatives** X pack ยิงแบบ ungraded และมีแค่ `NEGATIVE:` บรรทัดเดียว นี่คือความไม่สอดคล้องภายใน ไม่ใช่ช่องว่างจากงานวิจัย
- **Override block ลงคนละที่ในสามไฟล์** (หัวข้อของตัวเอง / ย่อหน้า ⚠️ ก่อน beats / ฝังใน reference job) ควรมีสล็อตเดียว
- **Provenance ของ house structure อ้าง 4 ชื่อ ไม่มี URL ไม่มีวันที่เข้าถึง** และ sweep 6 มุมนี้**หา "ChatCut's five formulas" ไม่เจอเลยแม้แต่ครั้งเดียว** อีกสามชื่อ (Higgsfield, fal.ai, MindStudio) เจอครบและตรวจสอบได้ ไม่ได้แปลว่า ChatCut ผิด แต่แปลว่าตอนนี้มันตรวจสอบไม่ได้

---

## 4. สิ่งที่ควรเพิ่มเข้าสกิลเรา

เรียงตามค่า/ต้นทุน ทุกข้อมี source กำกับ

**1. เช็คว่ายิงเข้า 2.0 หรือ 2.5 แล้วระบุใน skill** [ทางการ, verbatim] doc 2.5 บอกตรงๆ ว่า 2.0 ไม่อ่าน timestamp อ่านแต่ shot number ส่วนหัว house structure ของเราเขียนว่า "Seedance 2.0 & 2.5" ซึ่งเป็นความเสี่ยง

ประเมินความเสียหายตามจริง: shot ของเราเป็น one-take ล็อกเฟรมเกือบทั้งหมด `[Ns]` ทำหน้าที่**คุมจังหวะ ไม่ได้คุมการตัด** ถ้าโดน 2.0 กินทิ้ง ลำดับ prose ยังลงอยู่ ผลคือ pacing เพี้ยน ไม่ใช่โครงสร้างพัง แต่มันอธิบายอาการ "beat ที่สองมาเร็ว/ช้ากว่าที่เขียน" ได้พอดี **ทำ:** ระบุ target model ในบรรทัด tech header และถ้าเป็น 2.0 ให้ใส่ `Shot 1:` กำกับควบคู่ไปด้วย (ใส่แล้วไม่เสียหายกับ 2.5)

**2. เพิ่ม `ENDING STATE` หนึ่งบรรทัดต่อ shot ที่อยู่ใน chain** [fal.ai, ผู้ใช้จริง เจ้าของ inference ลงคลิปคู่ prompt] ระบุตำแหน่งสุดท้ายของตัวละคร ของ และกล้อง ให้ link ถัดไปมีจุดเริ่มจริง ไม่ใช่ให้โมเดลเดา ตรงกับต้นทุน re-render ของเรา

**3. ทำ exclusion เป็นสูตรตายตัวในช่อง REFERENCES** [fal + Curious Refuge + Emily2040, สามแหล่งอิสระ] เปลี่ยนจากสำนวนอิสระเป็น `@x — <job>. Do not take <รายการ> from @x.` เราทำอยู่แล้วแต่ไม่สม่ำเสมอ นี่คือ systematize ไม่ใช่ของใหม่

**4. เพิ่มบรรทัดสำเนียง/ภาษาในบล็อก dialogue** [MindStudio, **ผู้ใช้จริงรายเดียว** ระบุชัดว่าคนเดียว] ต้นทุนแทบเป็นศูนย์ และถ้าเราเคยเจอเสียงหลุดสำเนียงมาก่อน อันนี้อธิบายได้

**5. วัดเพดาน duration ใหม่** เพดาน 20s วัดวันที่ 2026-08-28 ส่วน 2.5 มี 30s ยืนยันจาก first-party สามแหล่ง (ByteDance blog, fal API, Higgsfield product page) ถ้า Higgsfield เปิด 30s แล้ว การเขียนได้ 30s เปลี่ยนวิธีวางซีนพอสมควร

**6. ซ่อมความไม่สอดคล้องของเราเอง** อันนี้ ROI สูงสุดในลิสต์และไม่ได้มาจากงานวิจัยเลย: ให้ s-extras มี grade block + house negatives, และกำหนดสล็อตเดียวสำหรับ override block

**7. ใส่ URL + วันที่เข้าถึง ในบรรทัด provenance ของ house structure** และตัดหรือหาหลักฐานให้ "ChatCut's five formulas"

**8. ไปเอา `sd25-pe` skill ของ ByteDance มา** [ทางการ] doc 2.5 แนะนำเองตรงๆ ยังไม่มีใครดึงตัวไฟล์มาได้ นี่เป็นการ**ตรวจสอบ** ไม่ใช่การเปลี่ยน แต่มันจะปิดคำถามนี้ได้ขาดกว่างานวิจัยที่เหลือทั้งหมดรวมกัน

### ที่ไม่ควรทำ

- **ไม่ rewrite เป็นโครง 9-section ของ fal** เรามีข้อมูลเดียวกันครบ แค่จัดวางต่างกัน และเรายิงมา ~30 คลิปแล้ว
- **ไม่ตัด negative wall ตาม official 2.5** ที่บอกว่า negative รองรับแค่ subtitles/audio เพราะตัวอย่างในหน้าเดียวกันนั้นเองมีบล็อก "[Strictly exclude]" ยาวเหยียด และคนที่ยิงงานจริงทุกคนเขียน negative ยาว ถ้าจะทดสอบให้ A/B กำแพง 233 คำกับ 5-8 ตัวที่ aimed แล้ววัด ไม่ใช่ตัดเพราะ doc บอก
- **ไม่เอา word cap 50-70 คำ** มาจาก SEO farm ไม่มีการทดสอบ และขัดกับ prompt ที่ Higgsfield ทดสอบเอง (800-3,500 คำ)
- **ไม่เอา JSON prompting (Veo) และไม่เอา negative field (Kling)**
- **ไม่เปลี่ยน `CONTINUES <first> UNBROKEN` pattern** fal มีกฎ "do not retell the first clip" ซึ่งห้ามเล่า**แอ็กชัน**ซ้ำ ไม่ได้ห้ามย้ำ**สิ่งที่ต้องคงที่** ของเราย้ำ invariant (same frame, same light, same positions) ซึ่งถูกอยู่แล้ว
- **bracket audio channel `( ) < > { } 【 】`** ยังไม่ควรใส่ในสกิล มี 4 แหล่งพูดตรงกันแต่ทั้งหมดน่าจะเป็น echo ของ doc ByteDance ฉบับเดียว และหาไม่เจอในหน้าที่อ่านถึงต้นทาง ถ้า burned-in text เป็นปัญหาซ้ำๆ ค่อยลองกับคลิปเดียว

---

## 5. สิ่งที่เรารู้แต่ community ไม่มีใครเขียนถึง

นี่คือส่วนที่ทำให้สกิลเราไม่ใช่ blog post อีกอันหนึ่ง ทุกข้อมีวันที่และมีของเสียเป็นค่าเรียน

**1. Video reference ได้แค่หนึ่ง** take 1 ของ S1C ตายด้วย generic platform error ตอนส่งวิดีโอสองไฟล์ ตัวเลขที่ community มีคือ fal 10 คลิป / ByteDance 10 คลิป / Higgsfield 3 คลิป ไม่มีใครพูดถึงเพดานจริงระดับ composer และที่แย่กว่านั้นคือ**มันแจ้งเป็น error กลางๆ ไม่บอกสาเหตุ** ซึ่งแปลว่าไม่มีทางเจอด้วยการอ่าน doc มีแต่ต้องเผาไปหนึ่ง take

**2. Reference chip ไม่เท่ากับ attachment** รถเข็นต้องเป็น Element chip ไม่ใช่ไฟล์แนบ และ `@Video 1` จะไม่ผูกอะไรเลยจนกว่าจะเลือกคลิปผ่าน `+ VIDEO TO EXTEND` ในตัว composer ถ้าไม่ทำ มัน**เรนเดอร์แบบไม่ผูก เงียบๆ และคิดเงินเต็ม** ทั้ง corpus นี้ไม่มีใครแตะเรื่องนี้เลย ทุกคู่มือปฏิบัติกับ @tag เหมือนมันเป็นแค่ข้อความ

**3. Protected-content scanner บล็อก Element ที่ผูกไว้แล้วได้ตรงๆ** และปุ่ม NO เป็น terminal ทางแก้คือทำ location plate ใหม่แบบไม่มีคน ไม่ใช่แก้ prompt community ไม่มีใครรู้ว่าแพลตฟอร์มปฏิเสธ asset ที่ผูกแล้วได้

**4. Defect ที่อบมากับ plate ลบด้วย negative ไม่ได้** กฎของเราคือ: รายละเอียดที่สั่งห้ามแล้วยังโผล่**สองครั้ง** = plate defect ให้ไปเปิด Element ดู อย่าไปเสริม prompt ต่อ อันนี้เป็นกฎวินิจฉัยที่ทั้ง community ขาด เพราะทุกคนตอบสนองต่อความล้มเหลวด้วยการเติมคำ MindStudio ถึงกับรายงานว่า morphing รอดจาก prompt ที่โครงสร้างดีมาได้และ upscale ก็ไม่หาย แต่ไม่ได้บอกว่าต้องหยุดเขียนแล้วไปดู reference

**5. เพดาน chip ~10 ก่อนผลจะเละ** community มีแต่ตัวเลข spec (9/12/50 ซึ่งขัดกันเองในเว็บ vendor เดียว) ของเราคือจุดที่หน้าคนเริ่มเบลนด์และเสื้อผ้าเริ่มสลับ Curious Refuge พังที่ ~15 คือ datapoint ภายนอกเพียงอันเดียวที่มี และมันสอดคล้องกับเรา

**6. Prompt สู้กับ Element ที่ตัวเองผูกไว้** plate ชุดทหารของเราโชว์หกคน ซีนต้องการอีกจำนวน จึงต้องเขียน "UNIFORM AND FACES ONLY, take the look, never the count" หลักการทั่วไป: plate พก**จำนวน กรอบภาพ พื้นหลัง และแสง**ที่เราไม่ได้ขอมาด้วยเสมอ และข้อความอย่างเดียวลบมันไม่ออก ต้องระบุการลบเป็นราย reference fal มี exclusion clause ก็จริงแต่วางมันเป็นเรื่องความเรียบร้อย เรารู้ว่ามันคือการต่อสู้ และรู้ว่าฝั่งไหนชนะถ้าไม่เขียน

**7. คำสั่งที่ปรากฏที่เดียวถูกทิ้ง** กฎ CEO 2026-08-21: ทั้งข้อห้ามหน้าซ้ำและข้อห้ามเสียงหัวท้าย ต้องอยู่**ทั้งใน SOUND block และในบรรทัด negative** เพราะที่เดียวโดนดร็อป นี่เป็นการค้นพบเชิงประจักษ์เกี่ยวกับพฤติกรรมโมเดล ไม่มีใครในโลกเขียนถึง ทุกคู่มือสมมติว่าเขียนครั้งเดียวพอ

**8. เพดานคำพูด 45-50 คำต่อคลิป 20s** community พูดแค่เชิงคุณภาพ ("keep lines short", "long monologues drift out of lip-sync") ไม่มีใครให้ตัวเลข

**9. ข้อห้ามเสียงหัวท้าย** no intro sting / no riser / no tail / no final chord / no fade ทั้งสองด้าน อันนี้เป็นของเราล้วน community หยุดแค่ "no music"

**10. เศรษฐศาสตร์ของ chain** แก้ที่ link 3 ของ chain 14 ตัว = re-render 12 ครั้ง ~4 ชม. เพราะฉะนั้น: chain ยาว 2-3 คลิป, ตัดที่จังหวะมีอะไรบังเฟรม, ข้าม hard break ให้ `@Character` plate แบกหน้าไว้ (plate จึงยิ่งสำคัญขึ้นตอนตัด ไม่ใช่ลดลง), เผา calibration render ทิ้งก่อนคลิปแรก, lateral track เย็บง่ายกว่าการเคลื่อนกล้อง 3 มิติ ไม่มีใครตีพิมพ์ต้นทุนของการแก้ช้าใน chain เลย ซึ่งเป็นเหตุผลทั้งหมดว่าทำไมจุดตัดถึงสำคัญ

**11. วินัยกวาดไฟล์** เติมกฎใหม่แล้วต้องกวาดทุกไฟล์ prompt ที่ยังไม่ได้ยิง (2026-08-21 เจอ 14 ไฟล์ตกกฎเสียง) นี่เป็นข้อค้นพบระดับกระบวนการผลิต ไม่ใช่ระดับ prompt และไม่มีที่ไหนพูดถึงเพราะไม่มีใครใน community ทำงานที่มีไฟล์ prompt เป็นสิบไฟล์รอคิวยิงอยู่

---

### สรุปให้ CEO บรรทัดเดียว

Community ไม่มีโครงสร้างเดียว มี 2 ชั้น (ประโยคสั้น 6 ช่อง กับบล็อกยาวมี label) และมันแตกตาม version ไม่ใช่แตกเพราะเถียงกัน โครงของเราคือชั้นยาวซึ่งเป็นตัวที่คนทำงานจริงใช้ ไม่ต้องรื้อ เพิ่ม 3 อย่าง (ENDING STATE, exclusion เป็นสูตร, บรรทัดสำเนียง) เช็ค 2 อย่าง (ยิงเข้า model ไหน, เพดาน 30s) ซ่อมของตัวเอง 2 อย่าง (s-extras ขาด shared block, override ลงคนละที่) และของที่เราแพงที่สุดคือสิบเอ็ดข้อในหมวด 5 ที่ไม่มีอยู่ในบล็อกไหนบนอินเทอร์เน็ตเลย


## CRITIC

โครงสรุปหลัก ("อย่ารื้อ, เพิ่ม 3 เช็ค 2") ยังยืนอยู่ แต่ข้อสนับสนุนหลายอันพัง โดยเฉพาะฝั่ง 2.0/2.5 ซึ่งเป็นจุดที่กระทบเรามากที่สุด

## 1. อ้างเกินกว่าที่ sweep รองรับ

- **"@Video ไม่มี first-party รองรับ" ผิด** รายงาน: "แหล่งเดียวที่อ้าง @Video คือ GitHub skill ของบุคคลที่สาม" / "@Video ของ community ไม่มี first-party รองรับเลย" แต่ sweep angle 5 มี ByteDance Seed launch blog ติดป้าย `confidence: primary_source` ระบุ field_order = "@-tagged reference declarations (@Image N / **@Video N** / @Clay Render N / @Audio N)" และ fal.ai (เจ้าของ inference) ใช้ "@Video1 controls only [camera path…]" ทั้งไกด์ ที่ไม่มี @Video คือ *บล็อก Higgsfield* ไม่ใช่ first-party
- **"คนที่ยิงงานจริงทุกคนเขียน negative ยาว" ผิด** นี่คือฐานเดียวที่ใช้ป้องกำแพง 233 คำ แต่ beech (คนเดียวใน corpus ที่เป็นบุคคลจริง) เขียนว่า "negative prompts stay short and only list problems that have actually shown up in your generations" และ Higgsfield 2.5: "There is no NEGATIVE section; the inverse is 'POSITIVE LOCKS'"
- **เอา mechanism ที่ sweep ตีตกแล้วกลับมาใช้** รายงาน: "สอดคล้องกับพฤติกรรม front-loading ของโมเดล" sweep angle 3 เขียนไว้ตรงๆ ว่า "'locks in subject before processing the rest' is not how these models work in any published sense; I regard it as invented rationalisation" คือปฏิเสธ word-cap ของ SEO farm แต่เก็บเหตุผลของมันไว้เงียบๆ
- **"ทั้งคู่ agree ว่า narrative = 3-4 beats/15s" ปั้นความเห็นพ้อง** beech คนเดียวที่ให้ตัวเลข MindStudio ไม่เคยให้ตัวเลข พูดแค่ว่า 2.5 "resists prompt styles built around rapid, one-second-style cuts"
- **"30s เป็น API-level ไม่ใช่แพลตฟอร์ม" ผิด และขัดกับ §4.5 ของตัวเอง** Higgsfield product page ใน sweep (primary_source) เขียน "Up to 30 seconds per generation" ผมเช็คซ้ำวันนี้ยังขึ้นแบบนั้น เพดาน 20s ของเราคือค่าวัดของ composer เรา ไม่ใช่หลักฐานว่าแพลตฟอร์มปิดที่ 20
- **"six-part formula เป็นตำนาน" ยิงผิดเป้า** sweep หักล้างสตริงคนละอัน ("Prompt = subject + movement + scene + camera, style…" บนหน้า 1.0-pro) ส่วน Segmind อ้างสำหรับ 2.5 และ doc 2.5 ที่อ่านถึงต้นทางมีสูตรบรรทัดเดียวจริง "Subject + Location + Event + Genre/Style + Camera movement…" (ellipsis เป็นของ doc เอง) ใกล้เกินกว่าจะเรียกตำนาน
- **ตาราง "2.0 = 5 ขั้น" ทิ้ง caveat สำคัญ** sweep: "the section headings did not survive extraction… Treat the field CONTENTS as confirmed and the ORDER as **high-confidence-but-inferred**"

## 2. 2.0 กับ 2.5 ปนกันหนัก (ข้อที่แพงที่สุดสำหรับเรา)

- **"primary ยืนยันของเรา" ใน §3 เกือบทั้งหมดคือ doc 2.0** ลำดับใน beat, one-noun-per-character, describe-before-use, anti-duplicate, ศัพท์กล้อง, body-movement และที่หนักสุดคือฐานป้องกำแพง negative ("official 2.0 **มีช่อง constraint words จริง**") สำหรับ pipeline ที่เป็น 2.5-only นี่ไม่ใช่การยืนยัน
- **"Audio block ท้าย = เราตรงกับ 2.5 primary" ไม่จริง** ยกตัวอย่างแพนด้าตัวเดียวมาสู้กับ field order ของ doc เดียวกัน ซึ่งเขียนว่า "per segment: visuals, camera movement, actions, dialogue, **sound effects**" คือ 2.5 เองก็วาง sound รายบีต
- **ข้อเสนอ #1 สวนทางกับ sweep และไม่ตรงกับเรา** "ถ้าเป็น 2.0 ให้ใส่ `Shot 1:` ควบคู่ ใส่แล้วไม่เสียหายกับ 2.5" ไม่มี source และ Emily2040 ใน sweep เตือนตรงข้าม: "Match the convention of the active surface; **do not mix both skeletons in one prompt**" เราเป็น 2.5-only อยู่แล้ว ทิศที่ควรถามคือทางกลับ ซึ่ง sweep มีคำตอบมือหนึ่งอยู่แล้วแต่รายงานไม่พูดถึงเลยสักคำ: MindStudio "prompts optimized for Seedance 2.0 don't reliably transfer to 2.5. Running an established 2.0 prompt through 2.5 produced **visibly broken, glitchy results**"
- **rewriter argument เป็นของ 1.0** ที่ใช้เชียร์ "การย้ำซ้ำถูกทาง" มาจาก arXiv 2506.09113 = Seedance **1.0** และ sweep ยังกำกับว่าอ่านผ่าน summarizer ไม่ได้อ่านดิบ ไม่มีหลักฐานว่ายังมี rewriter คั่นใน 2.5

## 3. ข้อขัดแย้งที่ปิดเร็วเกินไป

- **"Timestamp vs Shot N แก้แล้วด้วย official = เป็นเรื่อง version" ไม่จริง** ผมเปิดไกด์ 2.5 ของ Higgsfield เองวันนี้: ใช้ทั้งสองอย่างพร้อมกัน มี "Shot 1 / Shot 2 / SEGMENT 1" และมี timecode ในบีต "At 3.5s he raises the cigarette and draws" / "At 5.4s she says, soft and unsteady" ซ้ำร้ายเป็น**เศษวินาที** ซึ่งขัดกฎ official 2.5 ที่ว่า "Use 1-second intervals as the basic unit" / "integer-second timestamps" นี่เป็นคำถามเปิดจริงสำหรับ pipeline ที่ยิงผ่าน Higgsfield อย่างเดียว และ sweep ก็ขัดกันเอง (angle 2 "Beats inside a shot are timecoded in prose" vs angle 5 "Cuts are named, not timestamped")
- **counter-datapoint เรื่องจำนวน ref ถูกตัดทิ้ง** รายงาน: "Curious Refuge พังที่ ~15 คือ datapoint ภายนอกเพียงอันเดียวที่มี" แต่ MindStudio (แหล่งที่รายงานเชื่อในข้ออื่น) ทดสอบตู้เสื้อผ้าทั้งชุด "The result held up well across multiple outfit changes in a single generation"
- **rec #4 ไม่ใช่ single-source** ป้ายว่า "[MindStudio, ผู้ใช้จริงรายเดียว]" แต่ OpenArt ใน sweep เดียวกันเขียน "name the language before the delivery style… 'Spoken language: **American English**.'" ซึ่งเป็นวลีเป๊ะที่ MindStudio บอกว่าใช้ไม่ได้ มีสองแหล่งและมันขัดกัน มีค่ากว่าบอกว่าแหล่งเดียว
- **"Higgsfield ทดสอบเอง 800-3,500 คำ"** sweep: "No methodology, sample size, or failure rate is published, so this is an assertion, not data" และรายงานจับ beech "<3,500 **ตัวอักษร**" (~500-600 คำ) ไปอยู่คอลัมน์เดียวกับ 3,500 **คำ** ต่างกัน 6 เท่า คนละฝั่งกัน

## 4. angle ที่ว่าง แต่หาเจอ

- **"ChatCut's five formulas" หาเจอในการค้นครั้งเดียว** รายงานบอก "sweep 6 มุมนี้หาไม่เจอเลยแม้แต่ครั้งเดียว" ของจริงอยู่ที่ https://chatcut.io/blog/seedance-2-prompt-guide ("Seedance 2.0 Prompt Guide: 5 Formulas + Worked Examples", 18 มี.ค. 2026) ห้าสูตรคือ Core Asset Assignment / Prompt Structure / Camera Replication / Video Extension / Beat Synchronization และ skeleton สูตร 2 คือ "[asset] + [job of asset] + [what happens] + [when it happens] + [camera behavior] + [sound behavior] + [constraints]" ซึ่งคือทรงบ้านเราเลย **แต่มันเป็นหน้า Seedance 2.0** และมันสั่ง "0-3s, 3-6s, and 6-10s will almost always outperform a vague narrative blob" สำหรับ 2.0 ซึ่งคือสิ่งที่ doc 2.5 บอกว่า 2.0 ไม่อ่าน แปลว่าธรรมเนียม `[Ns]` ของเราอาจสืบมาจากบล็อก 2.0 ที่ vendor เถียง — สำคัญกว่าการเติม URL ตามข้อเสนอ #7 มาก
- **negative_prompt: ตรวจได้ใน 1 fetch ไม่มีใครตรวจ** รายงานบอก "Kapwing ยืนยัน" ทั้งที่ sweep เขียนว่า "the one negative-prompt assertion I would actually go verify" ผมเปิด schema ของ fal (seedance-2.5/reference-to-video): input มี `prompt, image_urls, video_urls, audio_urls, resolution, duration, aspect_ratio, generate_audio, bitrate_mode, end_user_id` **ไม่มี negative_prompt** ข้อสรุปรอด แต่หน้าเดียวกันปิดตัวเลขที่รายงานยกธง "ขัดกันเอง" ได้หมด: 30 ภาพ / 10 วิดีโอ (แต่ละคลิป 1.8-30.2s) / 10 เสียง, duration 4-30s, resolution 480p/720p/1080p (ไม่มี native 4K) นี่คือของฟรีที่ทั้งรายงานเว้นไว้
- **"X = ศูนย์" ไม่จริง** Reddit ผมยืนยันว่าบล็อกจริง (search index ไม่มี, www + old.reddit ปฏิเสธทั้งคู่) แต่ sweep angle 5 กู้ prompt verbatim จาก X มา 20+ อันผ่าน ZeroLu/awesome-seedance-2.5 + twitter-thread mirror และรายงานก็ใช้ของพวกนั้นอยู่ (techhalla, AIwithkhan, doctorwasif, Curious Refuge) ประโยค "docs + blog ของคนที่ผลิตงานจริงประมาณ 5 คน" ประเมินคลังตัวเองต่ำไป
- **sd25-pe ยังเอาไม่ได้จริง** ผมยิง bucket ทั้งสองตัว (มี CN อีกอันที่รายงานไม่รู้: `arkdocs.tos-cn-beijing.volces.com/skills/`) bucket มีชีวิต แต่ listing 403 และ key ที่เดาได้ 404 หมด ข้อเสนอ #8 ยังเปิดจริง ทางเข้าเหลือแค่ตัว installer หรือ browser session
- เล็กน้อย: skill ที่ sweep อยากได้ของ Higgsfield (`higgsfield-seedance-shotlist-director`) ก็เป็นของ **2.0** เหมือนกัน

## 5. เรื่องท่าที

- คอลัมน์ "เราดีกว่า" มี 5-6 ช่อง ทุกช่องเป็นการตัดสินโดยไม่มีการวัด รายงานยอมรับแค่ครั้งเดียว ("เรายังไม่เคย A/B เหมือนกัน") ควรยอมรับทั้งคอลัมน์ ไม่งั้นมันคือ self-assessment ที่แต่งเป็น finding
- sweep ขัดกันเองเรื่องความน่าเชื่อของแหล่ง แล้วรายงานเลือกข้างที่เข้าทางตัวเองโดยไม่บอก: angle 3 จัด **mindstudio.ai** และ **dreamina.capcut.com** ไว้ในกลุ่ม "SEO content farms" ขณะที่ angle 5 ยก MindStudio เป็นรายงานมือหนึ่งชั้นดี และ angle 4 ยก Dreamina เป็น anchor ที่ใกล้ first-party ที่สุด

Sources: [chatcut.io/blog/seedance-2-prompt-guide](https://chatcut.io/blog/seedance-2-prompt-guide), [fal.ai seedance-2.5 reference-to-video API](https://fal.ai/models/bytedance/seedance-2.5/reference-to-video/api), [higgsfield.ai/blog/seedance-2-5-prompting-guide](https://higgsfield.ai/blog/seedance-2-5-prompting-guide), [higgsfield.ai/seedance/2.5](https://higgsfield.ai/seedance/2.5), [help.scenario.com Seedance 2.5 essentials](https://help.scenario.com/articles/1651934064-seedance-2-5-the-essentials), [mindstudio.ai timeline prompting Seedance 2.0](https://www.mindstudio.ai/blog/timeline-prompting-seedance-2-cinematic-ai-video)
