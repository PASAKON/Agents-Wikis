# Higgsfield × Blender — the official add-on, and what it would cost us

**Question** higgsfield blender addon and video reference specs  
**Researched** 2026-09-03 · workflow run `w6syck6mr`  
**Method** 5 agents, ~0.5M tokens

> Agents were instructed to return empty rather than invent sources.
> Evidence levels are stated per finding: official / practitioner /
> secondhand / inference. See `README.md` for why that matters.

---


## ANSWER

# Higgsfield × Blender — คำตอบ 5 หัวข้อ

---

## 1. Higgsfield x Blender คืออะไร

**มีจริง เป็นของทางการ ไม่ใช่ของ community** — ประกาศใน changelog ของ Higgsfield เอง 2026-08-21 หัวข้อ "Higgsfield. Now in Blender" (tag: Release | MCP & Integrations)

| ประเด็น | ข้อเท็จจริง | ระดับหลักฐาน |
|---|---|---|
| ชื่อ | ไม่มีชื่อผลิตภัณฑ์ตายตัว หน้าเว็บเรียกสลับกันว่า "Higgsfield for Blender" / "the Blender add-on" / "the Higgsfield bar" | OFFICIAL |
| ดาวน์โหลด | `.zip` ไฟล์เดียวจาก https://higgsfield.ai/plugins/blender ลากทับหน้าต่าง Blender แล้วติดตั้งเอง ไม่ได้อยู่บน extensions.blender.org ไม่มี public source repo | OFFICIAL |
| เวอร์ชัน | Blender 4.2 – 5.1, Windows หรือ macOS เท่านั้น **Linux ไม่เคยถูกพูดถึงเลย** | OFFICIAL |
| UI | floating bar ลอยเหนือ viewport (ไม่ใช่ sidebar ไม่ใช่หน้าต่างแยก) 7 แท็บ: Scene builder, 3D Model, Character animation, Image, Video, Camera, Asset | OFFICIAL |
| ทิศทางข้อมูล **ลง** | ชัดเจนมาก — ผลลัพธ์ลงมาเป็น Blender data จริง แก้ไขได้ (mesh พร้อม quads/UV/PBR ที่ 3D cursor, armature + keyframes บน timeline, geometry + lights) | OFFICIAL |
| ทิศทางข้อมูล **ขึ้น** | Higgsfield พูดสองประโยคสั้น ๆ เท่านั้น: Image = "viewport in, frame out", Video = Seedance 2.5 "fed straight from your viewport" | OFFICIAL แต่คลุมเครือ |
| **กลไกของการอัปโหลด** | **ไม่มีเอกสารเลย** — ไม่บอกว่าเป็น OpenGL viewport grab, EEVEE/Cycles render, หรือ structured scene data | ไม่มีเอกสาร |
| ส่ง camera transform / F-curve เป็น motion conditioning | **ไม่มีคำยืนยันจากทางการที่ไหนเลย** ห้ามวางแผนโดยสมมติว่าทำได้ | ไม่มีเอกสาร |
| แท็บ Camera | วิ่ง**กลับทาง** — ขยับมือถือจริง แล้ว camera ใน Blender ตามแบบ real time (phone → Blender ไม่ใช่ Blender → Higgsfield) มาจาก blog table แหล่งเดียว หน้า product กับ changelog ไม่พูดถึง ไม่บอกชื่อ companion app ไม่มีใครยืนยัน | OFFICIAL แต่แหล่งเดียว |
| inference | **ทุกอย่างรันบน server ของ Higgsfield** offline ใช้ไม่ได้ GPU ในเครื่องไม่เกี่ยวกับการ generate | OFFICIAL |
| ค่าใช้จ่าย | ใช้ credit pool เดียวกับเว็บและ plugin อื่น ราคาโชว์บนปุ่ม Generate ก่อนกด | OFFICIAL |
| Bridge | `bridge.higgsfield.ai/mcp` เป็นคนละตัวกับ MCP ปกติ ให้ agent ภายนอก (ระบุชื่อ Claude ตรง ๆ) สั่ง add-on ใน scene ที่เปิดอยู่ เพิ่มเป็น custom connector ชื่อ "Higgsfield Bridge" | OFFICIAL |

**ช่องว่างเอกสารที่ควรรู้:** Help Center ของ Higgsfield เองในหน้า "External Integrations and Plugins" **ไม่ได้ list Blender เลย** (list แค่ Figma, Photoshop, Premiere/AE, DaVinci, Minecraft) เอกสารจริงมีแค่หน้า marketing กับ blog หนึ่งชิ้น ไม่มี reference doc ไม่มี privacy/data-handling note เฉพาะของ add-on และไม่มี source ให้ audit ว่าอะไรออกจากเครื่องบ้าง (github.com/higgsfield-ai มี 8 repo ไม่มี Blender สักตัว)

### แล้ว skill file ของเราพูดถึงตัวเดียวกันหรือเปล่า — น่าจะใช่ แต่ไม่ตรง

`/Users/gob/Projects/Agents/.claude/skills/blender-previz/SKILL.md` บรรทัด 30-57 เขียนว่า add-on เปิด local exec bridge ที่ `127.0.0.1:9876` พูด JSON `{type:"execute", code, strict_json:false}` และ recovery คือเรียก `mcp_service.start()` หลัง auth event `SIGNED_IN`

- **ตรงกับของทางการ:** มี MCP service ในตัว, ต้อง sign in ก่อนถึงทำงาน, รันบน Windows, Blender 5.x อยู่ในช่วง 4.2–5.1 ที่ support
- **ไม่ตรง:** skill เราเรียก panel ว่า `fnf_*`, "Generate 3D/HDRI/Motion/Retexture, Realtime" ซึ่ง**ไม่ตรงกับ 7 แท็บที่ Higgsfield ประกาศ**สักชื่อเดียว และ Higgsfield ไม่เคยเขียนถึง local socket 9876 ที่ไหนเลย — bridge ที่เขา document เป็น remote (`bridge.higgsfield.ai/mcp`)
- หมายเหตุ: prefix `fnf_` โผล่ในฝั่งเว็บด้วย — operator log จับ endpoint `POST /fnf/jobs/v2/seedance_2_5` และ `/fnf/favourites/liked-by/v2` ได้ ([absence-handover-20260902-2.md](file:///Users/gob/Projects/Agents/docs/reports/absence-handover-20260902-2.md)) แปลว่า `fnf_` เป็น namespace ภายในของ Higgsfield จริง ไม่ใช่ของเจ้าอื่น — **สนับสนุนว่าเราติดตั้งของทางการ แค่คนละ build/รุ่นกับที่หน้า marketing บรรยาย**

**เช็คถูก 2 นาที:** เปิด Blender บน winbox → Edit ▸ Preferences ▸ Add-ons → อ่านชื่อกับ version ที่มันลงทะเบียนไว้ นี่จะปิดคำถามนี้ถาวร และเป็นข้อมูลที่ไม่มีที่ไหนเผยแพร่ (ผมยืนยัน bl_info name จากภายนอกไม่ได้)

---

## 2. มันช่วยตัดขั้นตอนไหนของเราออกได้ไหม

**ในทางทฤษฎีตัดได้ 4 ขั้น ในทางปฏิบัติ "อย่าเพิ่ง" เพราะมันแลกกับเงิน**

pipeline ปัจจุบัน (จาก SKILL.md §Playblast → MP4 และ §Handoff):

```
bridge job → PNG seq 1280x720 → ffmpeg mux บน winbox → scp กลับ Mac
→ operator เปิด Chrome → อัปโหลดเข้า composer → พิมพ์ @Video 1 → กด Generate
```

ถ้า Video tab ทำงานอย่างที่ Higgsfield บอก ("Seedance 2.5 fed straight from your viewport") ขั้นที่หายไปคือ **PNG sequence, ffmpeg mux, scp, และการอัปโหลดด้วยมือทั้งหมด** เหลือแค่ตั้งกล้องใน Blender แล้วกด Generate ในแท็บ Video

**แต่มีสี่ข้อที่ทำให้ยังไม่ควรย้าย:**

1. **เสียเครดิตจริง** — นี่คือข้อชี้ขาด Higgsfield เขียนไว้ตรง ๆ ว่า *"Unlimited access and free generations apply only on higgsfield.ai—so anything generated through MCP, CLI, Canvas, Supercomputer, or other automated tools deducts credits at standard rates"* และหน้า FAQ ของ plugin เองก็บอกว่าราคาโชว์บนปุ่ม Generate งานที่ fail ของเราตอนนี้คืนเงิน $0 ทุกครั้งเพราะยิงบนเว็บใน Unlimited — ย้ายเข้า plugin แปลว่า**ทุก fail 25 นาทีจะเริ่มมีราคา** (plugin ไม่ได้ถูกระบุชื่อในลิสต์ "MCP, CLI, Canvas, Supercomputer" ตรง ๆ แต่ประโยค "other automated tools" + ราคาบนปุ่ม ทำให้ตีความเป็นอย่างอื่นได้ยาก — ถ้าจะเดิมพันควรถาม support ก่อน)
2. **กลไกไม่มีเอกสาร** — ไม่รู้ว่า "from your viewport" คือ grab แบบไหน ความละเอียดเท่าไร ส่งกี่เฟรม ซึ่งบังเอิญเป็นตัวแปรเดียวกับที่เรากำลังสงสัยอยู่พอดี ย้ายไปตอนนี้ = เปลี่ยนตัวแปรที่ควบคุมไม่ได้ ระหว่างที่กำลังดีบั๊กตัวแปรนั้นอยู่
3. **มันไม่ได้ปลดล็อกความสามารถใหม่** — workflow "เรนเดอร์ grey blockout แล้วโยนเข้า Seedance เป็น video reference" เป็นของเดิมที่มีมาก่อน add-on และเราทำอยู่แล้ว add-on เป็นชั้นความสะดวก ไม่ใช่ capability ใหม่ (อันนี้เป็นการตีความของผม ไม่ใช่คำพูดของ Higgsfield)
4. **CEO ruling ยังไม่ครอบคลุม** — SKILL.md บรรทัด 152-154 บันทึกว่า "Blender/AE เป็น editing-class tools ยกเว้นจาก platform-only; generation ยังเกิดบน Higgsfield" ถ้า generate ในแท็บ Video ของ Blender มันยัง "เกิดบน Higgsfield" อยู่ไหม — ตีความได้สองทาง อันนี้เป็นคำถามถึง CEO ไม่ใช่คำตอบทางเทคนิค

### ส่วนที่ premise ของโจทย์คลาดเคลื่อน

"GUI ไม่ได้รันและ agent เปิด headless ไม่ได้" — ถูกครึ่งเดียว **headless เปิดไม่ได้จริง แต่เรามีวิธีเปิดที่จดไว้แล้ว** SKILL.md บรรทัด 51-57:

```
blender.exe --python C:\Users\UsEr\Downloads\boot_bridge.py
```
สั่งผ่าน `schtasks /create … /it` + `/run` เพื่อให้ GUI ไปโผล่ใน session ของ user รอ ~2-4 นาทีให้ :9876 listen แล้ว poll (ห้ามเดา) — bridge จะไม่ขึ้นเองหลัง restart เพราะ add-on start มันเฉพาะตอน `SIGNED_IN` แบบ interactive ส่วน silent token restore ยิง `AUTH_SETTLED` แทน

**ฉะนั้น blocker จริงคือ "ต้องมี Windows user session ที่ล็อกอินอยู่" ไม่ใช่ "เปิดไม่ได้"** — และการซ่อม pipeline ฟรีตัวนี้ควรมาก่อนการไปประเมิน add-on

---

## 3. สเปกของ video reference ที่หาเจอ

### ประกาศชัดว่า: **Higgsfield ไม่ได้เผยแพร่ข้อกำหนดเรื่อง resolution ของ video reference ที่ไหนเลย**

ไม่มีทั้งค่าสัมบูรณ์ ไม่มีทั้งความสัมพันธ์กับ output resolution ค้นครบแล้วทั้ง: help center ทั้ง 74 บทความ (enumerate จาก `higgsfield.ai/creator-hub/sitemap.xml` — **ไม่มีบทความเรื่อง file requirements / supported formats เลยแม้แต่บทเดียว**), docs.higgsfield.ai ทั้ง 20 หน้า, OpenAPI spec ฉบับเต็ม (168KB, 50-51 endpoints, parse ด้วยเครื่อง), product page ของ Seedance 2.0 และ 2.5, blog, และ repo `github.com/higgsfield-ai/skills`

**ทฤษฎีปัจจุบันของเราวางอยู่บนพฤติกรรมที่ไม่มีเอกสารรองรับ 100%** — และไม่มี practitioner report จากใครที่ไหนที่บอกว่า video ref ถูก reject เพราะขนาดหรือความละเอียด

### สิ่งที่ Higgsfield เผยแพร่จริง (ทั้งหมดที่มี)

| รายการ | ค่า | ที่มา |
|---|---|---|
| จำนวน video clip ต่อ generation | 3 | help center + product page (ตรงกัน) |
| ความยาวต่อ clip | up to 15 seconds each | product page Seedance 2.0 เท่านั้น (help center ไม่พูดถึง) |
| format ผ่าน REST API | `video/mp4` **อย่างเดียว** | docs.higgsfield.ai/docs/concepts/file-uploads.md |
| format ผ่าน CLI/MCP | `mp4`, `mov`, `webm` | github.com/higgsfield-ai/skills |
| จำนวน video ref ของ `gemini_omni` | max 1 (และทำให้ image ref ลดจาก 7 → 5) | skills repo — **นี่คือ limit เชิงตัวเลขเรื่อง video ref เพียงอันเดียวที่ Higgsfield เขียนไว้** |
| resolution / file size / fps / aspect ratio ของ reference | **ไม่มีเอกสาร** | — |

### ความขัดแย้งในเอกสารของ Higgsfield เอง (ยังไม่คลี่คลาย)

- **จำนวน ref:** product page Seedance 2.5 = "Up to 50 multimodal inputs" vs help center = "9 images, 3 video clips, 3 audio files" — เป็นเว็บเดียวกันทั้งคู่
- **ความยาว:** Higgsfield = 15s ต่อ clip (รวม 45s) vs fal.ai (host เดียวกันของโมเดล ByteDance ตัวเดียวกัน) = "Combined duration 2–15 s" คือ 15s รวมทุก clip
- **`video_url`:** หน้า file-uploads สั่งให้ส่ง `public_url` เข้า parameter ชื่อ `video_url` — แต่ parameter นั้น**ไม่มีอยู่บน endpoint ไหนเลย**ใน OpenAPI spec

### สเปก resolution เดียวที่มีอยู่ในโลก — และมันไม่ใช่ของ Higgsfield

fal.ai เผยแพร่ตารางเต็มสำหรับ Seedance 2.0 Reference-to-Video (โมเดล ByteDance ตัวเดียวกัน คนละ host):

> Videos | Up to 3 | MP4, MOV | Combined duration 2–15 s, total under 50 MB, **480p–720p resolution per video**

สองข้อสังเกตที่สำคัญกับเราโดยตรง:

1. **640x360 คือ 360p ซึ่งอยู่ต่ำกว่าพื้น 480p ของ band นี้** — 1280x720 อยู่พอดีเพดานบน นี่เป็นเหตุผลอิสระที่ทำให้ทฤษฎีของเราน่าเชื่อขึ้น
2. **แต่รูปแบบของกฎไม่ใช่ "ref ต้อง ≥ output"** — fal ปล่อย output ถึง 1080p แต่ cap reference ไว้ที่ 720p คือ reference ถูก cap **ต่ำกว่า** output สูงสุด ถ้าเรารับกฎเชิงสัมพัทธ์ไปใช้ วันหนึ่งจะไปสรุปผิดว่า output 4K ต้องใช้ ref 4K ซึ่งจะหลุด band ทันที **กฎที่หลักฐานรองรับคือ "พื้นสัมบูรณ์ ~480p" ไม่ใช่ "≥ output"** — และแม้แต่ข้อนี้ก็เป็นการอนุมานของผมว่ามันถ่ายทอดมาถึง Higgsfield ไม่ใช่กฎที่ Higgsfield ประกาศ

### หลักฐานจาก repo ของเราเอง — สำคัญกว่าทุกอย่างข้างบน

ผม ffprobe previz ทั้ง 37 ไฟล์ใน `/Users/gob/Projects/Agents/docs/` แล้ว ทุกไฟล์เป็น 24fps ตรงกันหมด และ:

| กลุ่ม | ไฟล์ | ผล |
|---|---|---|
| ยิงพร้อม `@Video 1` แล้ว **ผ่าน** | S5, S6, S7, S8a, S8b, S9, S10b, S11, S2, S14, S16, S18a … | **ทุกไฟล์ 1280x720, 15s หรือ 20s** |
| ยิงพร้อม `@Video 1` แล้ว **fail** | `X3-Render.MP4`, `S1C-Render.MP4` (เวอร์ชันเดิม) | **ทั้งคู่ 640x360, 8s** |

**สี่ข้อที่ต้องรู้:**

1. **`S1C-Render.MP4` เดิมคือ 640x360 / 8s / 713,630 bytes ยืนยันจาก git blob commit `a963beb`** ตอนนี้ถูกแทนแล้วเป็น **1280x720 / 8s / 1,762,989 bytes** ใน commit `bb9b044` ("the previz goes to 720p, because it was the last variable left") — **และเวอร์ชัน 720p นี้ยังไม่เคยถูกยิง** การทดลองจึง staged พร้อมอยู่แล้ว
2. **X3 ก็ fail แบบเดียวกัน** และ previz ของ X3 ก็ 640x360 — ทฤษฎีนี้มีข้อมูลสองจุดอิสระ ไม่ใช่จุดเดียว
3. **ระยะเวลา generation ถูก exonerate แล้ว** — commit `209f1cd` บันทึกว่า X2/X3/X4 ยิงใหม่แบบ **refless แล้วผ่านหมด** X3 refless ที่ 8s ผ่าน แปลว่า "8 วินาที" ไม่ใช่ตัวปัญหา ปัญหาอยู่ที่ตัว reference
4. **ตัวแปรที่ยังพัวพันกันอยู่: previz ทั้งสองไฟล์เป็น 360p *และ* 8s พร้อมกัน** — `X2-Render.MP4` (360p/20s) กับ `X4-Render.MP4` (360p/10s) ถูกยิงแบบ refless เท่านั้น และ `S8a-Render-v2.MP4` (360p/20s) ไม่เคยถูกผูกใน prompt ไหนเลย จึงไม่มีตัวอย่างที่แยกสองตัวแปรนี้ออกจากกัน **"640x360" คือคำอธิบายที่ดีที่สุด แต่ยังไม่ใช่คำอธิบายเดียวที่รอด**

### สัญญาณที่ขัดกับทฤษฎี "ไฟล์ถูก reject"

จาก `/Users/gob/Projects/Agents/docs/reports/absence-handover-20260902-2.md`:

> **It failed at completion time**, ~24-25 minutes after firing, with a generic platform error: *"Something went wrong. Please try again, or change your input files or prompt."*

และ upload สะอาด — *"clean, ~30-60 seconds, thumbnail resolved, 'Added to prompt box' confirmed. Not a hang."*

**ถ้าไฟล์ผิดสเปก มันควรถูกปฏิเสธตอน upload หรือตอน submit ไม่ใช่หลังเผา GPU ไป 25 นาที** ข้อความ error นั้นเป็น boilerplate ที่บอกอะไรไม่ได้ ฉะนั้นแม้ 720p จะแก้ได้จริง กลไกก็ไม่ใช่ "validation reject" — อย่าเขียนลง skill ว่า Higgsfield "reject ref ที่ต่ำกว่า 480p" เพราะเรายังพิสูจน์ไม่ได้ว่ามันทำอย่างนั้น

### สิ่งที่ห้ามไล่ตาม

- **"อัปโหลด OBJ/FBX เป็น blockout reference"** — จาก seedance25.tools ซึ่งเป็น SEO site ไม่เป็นทางการ help center ของ Higgsfield list เฉพาะ image/video/audio ไม่มี practitioner คนไหนทำ ถือว่าไม่รองรับจนกว่าจะพิสูจน์ได้
- **"500MB / 2 นาทีต่อคลิป"** — ลอยอยู่ตาม marketing blog เจ้าอื่น ไม่มีแหล่งที่ Higgsfield เป็นเจ้าของ ห้ามออกแบบตามนี้
- **แท็บ Camera ที่ขยับด้วยมือถือ** — มาจาก blog table แหล่งเดียว ไม่มีชื่อ app ไม่มีใครยืนยัน

---

## 4. ยิงผ่าน API/MCP/CLI แทน browser ได้ไหม

**ได้ — แต่เฉพาะทาง CLI/MCP เท่านั้น REST API ทำไม่ได้เด็ดขาด** และนี่เป็นเรื่องที่ต้องแยกให้ขาด เพราะใครที่ดูแค่ docs.higgsfield.ai จะสรุปผิดว่าทำไม่ได้เลย

### ทางที่ได้ผล: CLI / MCP

```
npm i -g @higgsfield/cli
higgsfield auth login
higgsfield generate create seedance_2_5 --prompt "..." --video ./ref.mp4 --wait
```

- `--video <path-or-id>` เป็น documented flag ระบุว่า "reference or analyzed video" ใช้ได้กับ `seedance_2_0`, `seedance_2_5`, `brain_activity`
- `seedance_2_0` รับ media role ครบ 5: `image`, `start_image`, `end_image`, `video`, `audio`
- flag รับได้ทั้ง local path (auto-upload), upload id, **หรือ job id ของงานก่อนหน้า** — chain ผลงานเก่าเป็น video ref ได้โดยไม่ต้องอัปใหม่
- **CLI validate media role ในเครื่องก่อนส่ง** ถ้า role ผิดมัน error ทันทีโดยไม่ submit = fail ฟรี
- MCP endpoint: `https://mcp.higgsfield.ai/mcp` — Higgsfield เขียนเองว่า CLI media handling คือ "Mirrored from MCP server media-handling logic"

### ทางที่ตัน: REST API `api.higgsfield.ai`

Parse OpenAPI spec ครบทุก schema แล้ว (เก็บไว้ที่ `/private/tmp/claude-501/-Users-gob-Projects-Agents/00f1fc3c-9dae-4a9c-8642-52ef116d7688/scratchpad/hf-openapi.json`):

- `video_url` ปรากฏ **0 ครั้ง** ทั้ง spec ไม่มี endpoint ไหนรับ video input เลย
- **Seedance 2.x ไม่มีบน REST API เลย** มีแค่ Seedance v1 (lite / pro-fast)
- กับดักชื่อ: endpoint `/veo3.1/reference-to-video` **ไม่รับวิดีโอ** — parameter คือ `image_urls` (1-3 รูป)
- camera control ทาง REST คือ `motions` array (preset UUID + strength, สูงสุด 2) ไม่ใช่วิดีโอที่อัปโหลด

### ทำไมเรื่องนี้สำคัญมากกับความเจ็บของ operator — และทำไมยังไม่ควรย้ายทั้งหมด

CLI ตัดอาการที่ log ของเราจดไว้ออกได้หมด: Generate ปุ่มตาย, HTTP 429 จาก slot Unlimited ที่ยิงซ้อน, Unlimited toggle reset ทุกครั้งที่ reload, composer settings reset เอง, tab freeze จน `Page.captureScreenshot` timeout

**แต่ราคาคือเงินจริง** Higgsfield ระบุตรง ๆ ว่า Unlimited/free generation มีเฉพาะบน higgsfield.ai และของที่ผ่าน MCP/CLI/Canvas/Supercomputer "deducts credits at standard rates" ตอนนี้ session ที่ fail 3 ครั้งของเราคืนเงินครบ net $0 — ถ้าย้ายไป CLI ความล้มเหลวแบบเดิมจะเริ่มมีราคาต่อครั้ง **นี่เป็นการตัดสินใจงบประมาณของ CEO ไม่ใช่การตัดสินใจทางเทคนิคของ CTO**

ข้อจำกัดอีกข้อ: **Seedance 2.5 cap ที่ 720p** บน CLI (Higgsfield เขียนเองว่ามันไม่ใช่ 2.0 รุ่นใหม่กว่า) งานที่ต้อง 1080p/4K ต้องอยู่กับ Seedance 2.0 ซึ่งก็รับ `video` role เหมือนกัน

**ทางสายกลางที่ควรใช้ทันที:** ใช้ CLI เฉพาะคำสั่งที่**ไม่เสียเครดิต** — `higgsfield model get` (อ่าน schema) และ REST `POST /estimate/{model-path}` (คืน credits + USD ก่อนจ่าย) แล้วยังยิงงานจริงบนเว็บใน Unlimited ต่อไป ได้ความจริงโดยไม่จ่ายเงิน

---

## 5. แนะนำให้ทำอะไรต่อ (เรียงลำดับ)

### 1. ยิง S1C ด้วย previz 720p ที่ commit ไว้แล้ว — เปลี่ยนตัวแปรเดียว
**หลักฐาน:** `docs/S1C-Render.MP4` ถูกแทนเป็น 1280x720 / 8s แล้วใน commit `bb9b044` ความยาวยังเป็น 8s เท่าเดิม prompt เดิม Element เดิม → เป็น single-variable experiment ที่สะอาดที่สุดเท่าที่จะทำได้ และการทดลองนี้ staged พร้อมแล้ว
**ต้นทุน:** $0 (Unlimited) + ~25 นาที wall-clock + 1 operator slot
**อ่านผลยังไง:** ผ่าน = ทฤษฎี 360p ยืนยัน และ "ref ยาว 8s" ถูก exonerate ไปพร้อมกัน (เพราะ 8s ไม่เปลี่ยน) / fail = ทฤษฎีตาย ไปข้อ 2 ทันที อย่ายิงซ้ำที่ 3

### 2. รัน `higgsfield model get` — ได้ ground truth ฟรี
```
npm i -g @higgsfield/cli && higgsfield auth login
higgsfield model get seedance_2_5 --json | jq '{aspect_ratios, durations, parameters, medias}'
higgsfield model get seedance_2_0 --json | jq '{aspect_ratios, durations, parameters, medias}'
```
**หลักฐาน:** Higgsfield บอกเองว่านี่คืนสคีมาเต็ม — aspect ratios, durations (list ปิดหรือ min/max), parameters พร้อม default, และ media role ต่อ slot คำสั่งนี้จะปิดข้อขัดแย้ง mp4-vs-mov และ 15s-ต่อคลิป-vs-15s-รวม ได้ในครั้งเดียว ผมรันเองไม่ได้เพราะ `api.higgsfield.ai/models` คืน 401 ถ้าไม่ auth
**ต้นทุน:** 0 credits, ~10 นาที เป็นคำสั่งอ่านอย่างเดียว
**ความเสี่ยงที่ต้องยอมรับ:** ไม่มีใครยืนยันว่า `auth login` บน CLI กระทบสถานะ Unlimited บนเว็บหรือไม่ — น่าจะไม่ แต่ไม่มีเอกสาร ถ้ากังวลให้ทำหลังข้อ 1 เสร็จ

### 3. ซ่อม Blender GUI + bridge บน winbox ตามสูตรที่เราจดไว้เอง
**หลักฐาน:** `blender-previz/SKILL.md` บรรทัด 51-57 — `boot_bridge.py` ผ่าน `schtasks /create … /it` + `/run`, รอ ~2-4 นาที, poll ห้ามเดา นี่คือ blocker จริงที่หยุด previz pipeline ฟรีของเรา และเป็นสิ่งที่ต้องมีอยู่ดีไม่ว่าจะตัดสินใจเรื่อง add-on ยังไง
**ต้นทุน:** $0 + ต้องมี Windows session ที่ล็อกอินอยู่ (เปิด headless ไม่ได้จริงตามที่โจทย์ว่า)

### 4. ตั้ง 1280x720 เป็นพื้นแข็งใน skill พร้อมเหตุผล
**หลักฐาน:** SKILL.md §Playblast บรรทัด 112 เขียน 1280x720 อยู่แล้ว แต่มีสี่ไฟล์หลุดออกมาเป็น 640x360 (`X2`, `X3`, `X4`, `S8a-Render-v2`) แปลว่ามันเป็น convention ไม่ใช่ gate เพิ่มบรรทัดว่า **ห้ามผูก `@Video 1` กับคลิปที่ต่ำกว่า 1280x720** และเขียนกำกับว่านี่มาจาก band 480p-720p ของ fal สำหรับโมเดล ByteDance ตัวเดียวกัน **ไม่ใช่กฎที่ Higgsfield ประกาศ** — และ 720p คือ**เพดาน**ที่มีหลักฐาน ไม่ใช่พื้น อย่าเรนเดอร์ previz 1080p ขึ้นไปเพราะคิดว่าดีกว่า
**ต้นทุน:** 10 นาที
**พร้อมกันนี้เพิ่มกฎ hygiene ที่มาจากภายนอกและเรายังไม่มี:** ลบ trajectory line, coordinate axes, controller, camera frustum ออกจาก source ก่อนเรนเดอร์ (ไม่งั้นโมเดลเรนเดอร์มันเป็นวัตถุในฉากจริง), coarse blockout อย่าใส่แขน/ขา/ปีก, และประโยค negative ที่ควรอยู่ในทุก prompt: *"Do not use its blockout appearance, materials, or scene"* — ตอนนี้ prompt S1C ของเรามี *"@Video 1 carries ONLY camera path, timing, and positions. It is NOT a style reference"* ซึ่งดีอยู่แล้ว แต่ยังไม่มีข้อ scope สำหรับกรณีใช้ ref แค่ช่วงหนึ่งของคลิป

### 5. ถ้าข้อ 1 fail — ยิง X3 ด้วย previz 720p เป็นจุดข้อมูลที่สอง
**หลักฐาน:** X3 fail แบบเดียวกันด้วย previz 360p และผ่านเมื่อยิง refless ถ้า X3 ที่ 720p ผ่าน แต่ S1C ที่ 720p fail แปลว่าปัญหาอยู่ที่ S1C เฉพาะตัว (prompt/Element/cart) ไม่ใช่ที่ระบบ ref
**ต้นทุน:** $0 + เรนเดอร์ previz ใหม่ 1 ตัว + 25 นาที

### 6. ประเมิน add-on / Bridge — **ยังไม่ใช่ตอนนี้**
**เหตุผล:** เสียเครดิตจริง, กลไก upload ไม่มีเอกสารและดันเป็นตัวแปรเดียวกับที่กำลังดีบั๊ก, ไม่ปลดล็อก capability ใหม่, และ CEO ruling เรื่อง platform-only ยังไม่ครอบคลุมเคสนี้
**สิ่งเดียวที่ควรทำตอนนี้คือเช็คฟรี 2 นาที:** เปิด Edit ▸ Preferences ▸ Add-ons บน winbox อ่านชื่อ+เวอร์ชันที่ติดตั้งอยู่ แล้วอัปเดต SKILL.md บรรทัด 23-24 ให้ตรงกับ panel ที่มีจริง (ตอนนี้เขียน "Generate 3D/HDRI/Motion/Retexture, Realtime" ซึ่งไม่ตรงกับ 7 แท็บที่ประกาศ)

### 7. ย้ายการยิงงานไป CLI — **ชง CEO ยังไม่ตัดสินเอง**
**หลักฐาน:** `--video` ทำได้จริงและจะลบอาการปุ่มตาย/toggle reset/429 ออกทั้งหมด แต่แลกกับการเปลี่ยน generation ฟรีเป็น generation ที่มีราคา
**ต้นทุน:** เป็นเรื่องงบ ไม่ใช่เรื่องเทคนิค — ถ้าจะลอง ให้ pilot ทีละ 1 คลิป โดยมี `POST /estimate/{model-path}` เป็นประตูกันเงินก่อนทุกครั้ง และตาม `feedback_ask_before_paid_api` ต้องบอกตัวเลข $ ที่แน่นอนแล้วรอ OK

### สิ่งที่ควร "ไม่ทำ"
- **ไม่ยิง S1C ซ้ำด้วย previz 360p อีก** — 3 ครั้งพอแล้ว ครั้งละ 25 นาที ไม่ได้ข้อมูลใหม่
- **ไม่เขียนลง skill ว่า Higgsfield มี minimum resolution** — สัญญาณ fail-at-completion-time ขัดกับการเป็น validation reject และไม่มีเอกสารรองรับ เขียนได้แค่ "เรามีหลักฐานเชิงสหสัมพันธ์ 2 จุด + band 480p-720p ของ fal สำหรับโมเดลเดียวกัน"
- **ไม่ต้องรีบแก้ความยาว previz 20s** ทั้งที่ Higgsfield เขียนว่า "up to 15 seconds each" — previz 20s ของเรายิงผ่านมาแล้วหลายตัว (S7, S10b, S2, S11 …) แปลว่าลิมิตนี้ไม่ได้ถูก enforce เป็น hard reject บน composer เว็บ นี่เป็นกรณีที่ข้อมูลของเราแม่นกว่าเอกสารของเขา — จดไว้ใน skill ได้เลย

**ไฟล์อ้างอิง:** `/Users/gob/Projects/Agents/.claude/skills/blender-previz/SKILL.md` · `/Users/gob/Projects/Agents/docs/PREVIZ-INDEX.md` · `/Users/gob/Projects/Agents/docs/reports/absence-handover-20260902-2.md` · `/Users/gob/Projects/Agents/docs/reports/absence-handover-20260903-3.md` · `/Users/gob/Projects/Agents/docs/S1C-Render.MP4` (720p ปัจจุบัน) · blob เดิม 360p กู้ได้จาก `git cat-file -p a963beb:docs/S1C-Render.MP4`
