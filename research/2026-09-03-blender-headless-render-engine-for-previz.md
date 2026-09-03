# Rendering Blender with nobody at the keyboard — and the right engine for grey-proxy previz

**Question** blender headless render engine for previz  
**Researched** 2026-09-03 · workflow run `wqriorl7t`  
**Method** 6 agents, ~0.9M tokens

> Agents were instructed to return empty rather than invent sources.
> Evidence levels are stated per finding: official / practitioner /
> secondhand / inference. See `README.md` for why that matters.

---


## ANSWER

# 1. วิธีที่คน render Blender แบบไม่มีหน้าจอ

มาตรฐานคือ **background render ผ่าน CLI** ไม่ใช่ playblast — Blender ออกแบบมาให้ทำแบบนี้ตั้งแต่แรก manual มีหน้าเฉพาะชื่อ "Rendering From The Command Line" และบอกตรง ๆ ว่า "we do not need a graphical display ... we can render via a remote shell (typically SSH)"

รูปแบบคำสั่งที่ถูกต้อง:

```
blender -b <file.blend> [-E ...] [-o ...] [-F ...] [-x 1] [-s N -e M] [-P script.py] -a
```

**กฎที่พังบ่อยที่สุดคือลำดับ argument** — manual เตือนไว้สองที่ ไฟล์ .blend ต้องมาก่อน setter ทั้งหมด และ `-f` / `-a` ต้องอยู่ท้ายสุดเสมอ ตัวอย่างจาก manual แบบคำต่อคำ:

```
blender -b file.blend -a -x 1 -o //render     ← ไม่ทำงาน
blender -b file.blend -x 1 -o //render -a     ← ถูกต้อง
```

ที่อันตรายกว่านั้น: `-E`, `-F`, `-x` ถ้าวางผิดที่ (ก่อนโหลด .blend) Blender จะ print error ลง stderr แล้ว **render ต่อด้วยค่าเดิม exit code 0** มีแค่ `-E` ที่ชื่อ engine ผิดเท่านั้นที่ exit 1 — เพราะฉะนั้นอย่าเชื่อ exit code อย่างเดียว ต้องเช็คว่าไฟล์ output ออกมาจริง

Studio จริง ๆ ทำ 3 ชั้นเสมอ เหมือนกันหมดทั้ง Blender Studio (Flamenco), AYON/OpenPype และ Ubisoft Shot Manager:
1. render เป็น **image sequence** ด้วย `blender -b` (นี่คือชั้นเดียวที่รันแบบไม่มีคนได้)
2. **ffmpeg** แปลง sequence เป็น MP4 เป็น step แยก
3. playblast จาก viewport เป็นงานที่ artist กดเองใน GUI เท่านั้น — Flamenco farm ของ Blender Studio ส่ง `-b -y` ให้ Blender ทุก task ซึ่งแปลว่า playblast **ไม่มีทางอยู่บน farm ของเขาได้เลย**

**Evidence level: official docs ล้วน** — manual + source code ของ Blender เอง (`creator_args.cc`) + test suite ของ Blender เอง + source ของ Flamenco/AYON/Shot Manager ไม่ใช่ blog ไม่ใช่ tutorial

---

# 2. เครื่องยนต์ไหนถูกต้องสำหรับ previz ของเรา

**ตอบ: Workbench (`-E BLENDER_WORKBENCH`) ชัดเจน ไม่ต้องคิดต่อ**

เหตุผลเรียงตามน้ำหนัก:

**1. Workbench คือ engine ตัวเดียวกับที่ viewport Solid mode ใช้** — ไม่ใช่ของเลียนแบบ manual ระบุว่า "Solid mode uses the render settings of Workbench" เพราะฉะนั้น **ใช่ Workbench headless ให้ภาพแบบเดียวกับ viewport playblast ที่เราทำมา 33 ตัว** แค่เรียกผ่าน render pipeline แทน viewport

**2. "Workbench ไม่ใช้ไฟในซีน" เป็นข้อดีของเคสเรา ไม่ใช่ข้อจำกัด** — manual: "The Workbench engine does not use the lights of the scene" ตั้ง Studio light อย่างเดียวก็อ่านรูปทรงได้ทันที ตรงข้ามกับ EEVEE ที่ต้องพึ่ง world/lights ถ้าไม่มีการจัดไฟ grey cylinder จะออกมาแบน ๆ อ่านฟอร์มไม่ได้ — แปลว่าต้องไป**สร้างงานที่เราบอกว่าไม่ต้องการ**เพื่อให้ previz อ่านออก

**3. `BLENDER_WORKBENCH` เป็น engine id ตัวเดียวที่ไม่เคยเปลี่ยนชื่อ** — EEVEE เปลี่ยนจาก `BLENDER_EEVEE` → `BLENDER_EEVEE_NEXT` (4.2–4.5) → `BLENDER_EEVEE` (5.0+) ส่ง id ผิด = exit(1) ทันที Workbench ไม่มีปัญหานี้เลย pipeline ทนข้ามเวอร์ชัน

**4. Ubisoft Shot Manager แยก "playblast engine" ออกจาก "render engine" และให้เลือก Workbench ได้ตรง ๆ** ส่วน Blender Studio กับ AYON กำหนด look ของ blocking pass ไว้เหมือนกันเป๊ะโดยไม่ได้นัดกัน: Workbench SOLID + material color + overlay/gizmo ปิดหมด นี่คือ preset ที่ industry ลงเอยเหมือนกัน

**ทำไมไม่ใช่ EEVEE ที่ samples ต่ำ:** นอกจากเรื่องไฟข้างบนแล้ว EEVEE Next ยังมี shader compilation cost ก้อนใหญ่ที่เฟรมแรก, API property เปลี่ยนยกชุดตอน 4.2 (`use_gtao`, `use_bloom`, `use_ssr`, `use_soft_shadows`, `use_volumetric_lights` หายหมด) และมันคือเส้นทางที่เพิ่งให้เรา 20 นาที ไม่มีเหตุผลจะกลับไป

### สิ่งที่เสียไปจาก viewport playblast — พูดตรง ๆ

| เสีย | ผลกระทบ |
|---|---|
| **Overlays ทั้งหมด** — grid floor, empties, camera passepartout, motion path, annotation | **นี่คือข้อเสียจริงข้อเดียวที่สำคัญ** ถ้า previz เดิมอาศัย grid floor เป็นจุดอ้างอิงพื้น ต้องใส่ plane เป็น object จริงเข้าไปแทน ไม่งั้นกล้องจะลอยไม่มี ground reference |
| shading state ที่ artist เห็นอยู่บนจอ | ต้องเซ็ตใน script แทน (ซึ่งดีกว่าในแง่ reproducibility) |
| Wireframe / solid+wireframe / shading.type อื่น ๆ | Workbench-as-engine เดินเส้น solid อย่างเดียว |
| ความเร็ว | **ไม่เท่ากัน** ดู §5 — อย่าคาดหวังว่าเร็วเท่า playblast เดิม |

สิ่งที่ยังได้ครบ: camera DoF, cast shadow (ถ้าอยากได้), per-object random color เพื่อแยก proxy, transparent film, burn-in stamp เลขเฟรม

---

# 3. playblast headless ทำได้จริงไหม

**ไม่ได้ ทุกเวอร์ชัน ไม่มีข้อยกเว้น**

`bpy.ops.render.opengl()` ถูกบล็อกด้วย guard ตัวแรกใน `screen_opengl_render_init` ก่อนเช็คอย่างอื่นทั้งหมด:

```c
if (G.background) {
    BKE_report(op->reports, RPT_ERROR,
        "Cannot use OpenGL render in background mode (no opengl context)");
    return false;
}
```

ตรวจ source แบบ byte-for-byte แล้วที่ tag **v2.79b, v2.93.0, v3.3.0, v3.6.0, v4.2.0, v4.5.0, v5.0.0 และ main** — เหมือนกันหมด ไม่เคยมีเวอร์ชันไหนที่ทำได้ และ fork ทุกตัว (UPBGE, Bforartists, goo-engine) ก็ยังมี guard นี้

**สองข้อที่สำคัญกว่าตัว guard เอง เพราะมันปิดทางแก้ที่ดูน่าจะได้:**

1. **ไม่ใช่ปัญหา GPU context** — มีคนทดสอบบน Blender 5.2.1 เรียก `gpu.init()` ใน background mode สำเร็จ ได้ OpenGL context จริง (backend `OPENGL`, renderer `AMD Radeon RX6500 XT`) จอง `GPUOffScreen` ได้ แล้วเรียก `render.opengl()` — **ยัง error ข้อความเดิมเป๊ะ** เพราะฉะนั้น EGL, xvfb, `--gpu-backend`, GPU headless — ช่วยไม่ได้สักอย่าง ข้อความ "(no opengl context)" ในวงเล็บนั้นทำให้เข้าใจผิด มันเช็ค flag `-b` ไม่ได้เช็ค context

2. **ไม่ใช่ปัญหา poll เหมือนกัน** — `bpy.ops.render.opengl.poll()` return **True** ใน background mode (bpy ยังมี window กับ `bpy.data.screens['Layout']` อยู่) เพราะฉะนั้น `temp_override` / context override ก็ช่วยไม่ได้ ใครสอนให้แก้ด้วยวิธีนี้คือผิด

ที่สำคัญคือ **ไม่มีใครกำลังจะแก้** — task #54638 "OpenGL headless rendering" ที่ Clément Foucault เปิดไว้ปี 2018 ยังเปิดค้างอยู่ และคุยกันเรื่อง EGL context ไม่เคยพูดถึงการปลด guard ตัวนี้เลย

**ทางออกที่มีจริงถ้าวันหนึ่งจำเป็นต้องได้ viewport shading + overlay จริง ๆ:** Blender 5.2 เพิ่ม `gpu.init()` มาให้ใช้ `GPUOffScreen.draw_view3d()` ใน background mode ได้ — ทดสอบแล้วว่าทำงาน เป็น programmatic viewport render ตัวจริง แต่ต้องเขียนโค้ดเอง (ดึง VIEW_3D area จาก `bpy.data.screens` เอง จัดการ matrix เอง อ่าน pixel เอง เขียนไฟล์เอง) และต้อง 5.2+ เท่านั้น — **ไม่แนะนำสำหรับงานนี้ Workbench ให้ผลเดียวกันด้วยโค้ดหนึ่งบรรทัด**

ทางที่เหลืออีกทาง (รัน Blender แบบ GUI ใต้ xvfb ให้ `G.background` เป็น false) คือทางที่เราปิดไปแล้วด้วยเหตุผลของเราเอง และไม่มีใครใน production ยืนยันว่าใช้จริง — เป็น inference ล้วน อย่าไปไล่ต่อ

---

# 4. คำสั่งที่ควรใช้จริง

**ทำ 2 อย่างนี้ก่อนเสมอ** (คนละ 5 วินาที ตัดปัญหาได้เยอะ):

```
blender -E help
blender --version
```

`-E help` จะ print `type.idname` ออกมาตรง ๆ ยืนยันว่าเห็น `BLENDER_WORKBENCH` บนเครื่องนั้นจริง

### Smoke test — 1 เฟรม ก่อนทำทั้ง sequence

```
blender -b --factory-startup "C:\previz\shot_s2.blend" -E BLENDER_WORKBENCH -o "C:\previz\out\smoke_" -F PNG -x 1 -f 1
```

แล้ว**เช็คว่าไฟล์ `smoke_0001.png` เกิดขึ้นจริง** อย่าดู exit code อย่างเดียว ถ้าเจอ `Cannot initialize the GPU` ให้เพิ่ม `--gpu-backend vulkan` ต่อจาก `-b`

### คำสั่งจริง (เขียนบรรทัดเดียว — อย่าใช้ `^` หรือ backtick เพราะไม่รู้ว่า ssh เข้าไปเจอ cmd.exe หรือ PowerShell)

```
blender -b --factory-startup --python-exit-code 1 "C:\previz\shot_s2.blend" -E BLENDER_WORKBENCH -o "C:\previz\out\s2_" -F PNG -x 1 -s 1 -e 192 -P "C:\previz\previz_settings.py" -a
```

`-o "...\s2_"` ไม่มี `#` Blender จะเติม `####` ให้เอง → `s2_0001.png` (manual ระบุไว้ตรง ๆ)

`--python-exit-code 1` สำคัญ: ถ้า script พังโดยไม่มีตัวนี้ Blender จะ print traceback แล้ว **render ต่อด้วยค่า default** ได้ไฟล์ครบ เนียนมาก แต่ผิดหมด — ตัวนี้ทำให้ job ตายแทน

### `previz_settings.py`

```python
# previz_settings.py — headless Workbench previz preset
import bpy

s = bpy.context.scene
r = s.render

# --- resolution / timing ---
r.resolution_x = 1280
r.resolution_y = 720
r.resolution_percentage = 100
r.fps = 24
r.fps_base = 1.0

# --- Workbench look (อยู่บน scene.display ไม่ใช่ viewport → ทำงาน headless) ---
d = s.display
d.render_aa = 'OFF'          # default คือ '8' = วาดซีนซ้ำ 8 รอบ นี่คือปุ่มความเร็วอันดับ 1
sh = d.shading
sh.light = 'STUDIO'          # ไม่แตะไฟในซีนเลย
sh.color_type = 'SINGLE'     # เปลี่ยนเป็น 'RANDOM' ถ้าอยากให้ proxy แต่ละตัวสีต่างกัน
sh.show_shadows = False
sh.show_cavity = False
sh.show_object_outline = False
sh.show_specular_highlight = False

# --- ตัด fixed per-frame cost ---
s.view_settings.view_transform = 'Standard'   # ไม่เอา AgX/Filmic (colour management กิน CPU)
s.use_nodes = False                            # ปิด compositor
r.use_motion_blur = False
r.film_transparent = False

# --- burn-in เลขเฟรม (optional แต่คุ้มมากสำหรับ previz ที่ขาย timing) ---
r.use_stamp = True
r.use_stamp_frame = True
r.use_stamp_lens = True
r.stamp_font_size = 24
r.use_stamp_labels = True

print("[previz] engine=%s aa=%s %dx%d" % (r.engine, d.render_aa, r.resolution_x, r.resolution_y))
```

### ffmpeg → MP4

```
ffmpeg -y -framerate 24 -i "C:\previz\out\s2_%04d.png" -c:v libx264 -crf 20 -g 18 -pix_fmt yuv420p -vf "pad=ceil(iw/2)*2:ceil(ih/2)*2" -r 24 "C:\previz\out\s2_previz.mp4"
```

(ลอกมาจาก Flamenco `frames-to-video` task ตรง ๆ — 1280x720 เป็นเลขคู่อยู่แล้ว `pad` เลยไม่ทำอะไร แต่ปล่อยไว้ไม่เสียหาย)

### สิ่งที่ยืนยันไม่ได้ ต้องธงไว้

- **ความเสี่ยงอันดับ 1: winbox เป็น Windows** manual ของ Blender 4.2+ เขียนว่า "Headless rendering is not supported on headless Windows systems" และ EGL fallback ที่แก้เรื่องนี้เป็น Linux-only Jeroen Bakker (คนดูแล Workbench) เคยบอกตรง ๆ ว่า OpenGL ต้องมี GUI session กับจอต่ออยู่ — แต่**มีคนทดสอบบนเครื่อง Windows + AMD GPU ผ่าน ssh แล้ว Workbench render สำเร็จจริง (2.6s)** ปัญหาคือ**ยืนยันไม่ได้ว่าตอนนั้นมี desktop session login ค้างอยู่หรือเปล่า** ถ้ามัน work เพราะมีคน login ค้าง วันที่ไม่มีคน login มันจะพัง → **ทดสอบ smoke test จาก ssh session สดตอนที่มั่นใจว่าไม่มีใคร login** นี่คือการทดสอบที่คุ้มที่สุดก่อนสร้าง pipeline ทับ
- `sh.single_color` / `sh.show_xray` — ผมไม่ได้ verify ชื่อ property พวกนี้บน `scene.display.shading` โดยตรง (ที่ verify คือ `.light` กับ `.color_type` จาก production code จริง) ทั้งคู่เป็น `View3DShading` เหมือนกันเลยควรจะมี แต่ถ้าเจอ AttributeError ให้ลบบรรทัดนั้นทิ้ง render ยังทำงานปกติ
- ตำแหน่งของ `--gpu-backend` และ `--python-exit-code` ในบรรทัด — ผมวางตาม issue #154123 กับหลักการ "argument ทำงานตามลำดับ" ไม่ใช่จาก doc ที่ระบุตำแหน่งชัด
- Blender **ไม่มี exit code สำหรับ render ที่ล้มเหลว** ที่เป็นทางการเลย `--python-exit-code` คุมแค่ Python exception → wrapper ควรเช็ค stderr หา substring `Error:` และเช็คจำนวนไฟล์ที่ออกมา
- path ของ `blender.exe` บน winbox — ผมไม่รู้ ต้องไปเติมเอง

`-noaudio` ไม่ต้องใส่แล้ว: ตั้งแต่ 4.2 audio ปิดอยู่แล้วโดย default ใน background mode (source comment ของ Blender บอกเหตุผลไว้ตรง ๆ ว่าเพื่อไม่ให้คนต้องจำ `-noaudio` เวลาทำ batch)

---

# 5. ประมาณการเวลา

**ผมไม่ได้วัดเอง** — ไม่มี Blender บนเครื่องนี้ และตัวเลขที่หาได้จาก research ไม่มีอันไหนตรงเคสเรา (720p, grey proxy, เครื่องเรา) เลย ดังนั้นให้ช่วงกับฐานที่มา

**ประมาณ: 192 เฟรมที่ 1280x720 → น่าจะ 30 วินาที – 3 นาที, worst case ~8 นาที**

ฐานที่มาของช่วงนี้:

| หลักฐาน | ค่า | ใช้ได้แค่ไหน |
|---|---|---|
| **EEVEE บนเครื่องเรา งานจริง** | 6.25 s/frame (20 นาที ÷ 192) | แม่นที่สุด เป็นตัวเลขของเราเอง — ใช้เป็นเพดาน |
| Workbench vs EEVEE, Blender 2.83, 2020, ซีนเดียวกัน | 37s vs 52s = เร็วกว่า **1.4x** | เก่ามาก ไม่รู้ frame count/resolution ใช้ได้แค่เป็นอัตราส่วนขั้นต่ำ |
| Workbench vs EEVEE บนเครื่องเรา, 1 เฟรม, Blender 5.2.1 | 2.6s vs 16.9s = **6.5x** | เฟรมเดียว รวม startup + EEVEE shader compile → **เกินจริง** สำหรับ animation |
| per-frame overhead floor, 2019, 4K, ซีนว่างเปล่า | 0.27 s/frame | เก่า และ 4K = pixel มากกว่า 720p ~9 เท่า |

เอา 1.4x เป็นขั้นต่ำ → 4.5 s/frame → **14 นาที** (แย่กว่าเดิม? ไม่ใช่ — เพราะ 1.4x นั้นวัดตอน AA เปิด default ทั้งคู่ ซึ่งเราปิดไปแล้ว) เอา 6.5x เป็นเพดานบน → ~1 s/frame → **3 นาที** แล้วบวกผลของ `render_aa = 'OFF'` (default `'8'` = วาดซีน 8 รอบ) + view transform Standard + ปิด compositor ทับลงไปอีก

**ตัวเลขนี้ขึ้นกับ 4 อย่าง เรียงตามน้ำหนัก:**

1. **`scene.display.render_aa`** — default `'8'` วาดซีนซ้ำ 8 รอบ นี่คือปุ่มเดียวที่ใหญ่ที่สุดฝั่ง Workbench
2. **`resolution_percentage`** — cost ทุกอย่างใน Workbench ผูกกับจำนวน pixel เกือบทั้งหมด ลด 50% = pixel เหลือ 1/4
3. **Colour management** — Filmic/AgX เคยวัดได้ ~0.5 s/frame ฝั่ง CPU (Blender dev วัดเอง ปี 2019) `Standard` ตัดตรงนี้ทิ้ง และ output เป็น **PNG 8-bit** จะได้ GPU colour-management fast path (EXR/16-bit ไม่ได้)
4. **Fixed per-frame overhead** — encode + GPU→CPU readback + เขียน disk **ไม่ขึ้นกับความซับซ้อนของซีนเลย** มีคนวัดซีนที่ไม่มี object เลยยังเสีย 0.27 s/frame และยังมีคนบ่นเรื่องนี้อยู่ใน Blender 4 (2024) → **อย่าไปเสียเวลา optimize geometry ของ proxy มันฟรีอยู่แล้ว**

### วัดจริงใน 30 วินาที แทนที่จะเดา

```
blender -b --factory-startup "C:\previz\shot_s2.blend" -E BLENDER_WORKBENCH -o "C:\previz\out\t_" -F PNG -x 1 -s 1 -e 24 -P "C:\previz\previz_settings.py" -a
```

24 เฟรม จับเวลา คูณ 8 (บวก startup ~5-15s ที่จ่ายครั้งเดียว) — ตัวเลขนี้ชนะทุกตัวเลขที่ผมอ้างมาข้างบน

**ถ้าออกมาช้ากว่าที่รับได้:** ลด `resolution_percentage` เป็น 50 **อย่าใช้ `-j 2` ข้ามเฟรม** — previz ตัวนี้ขาย timing กับ camera path การข้ามเฟรมคือการทำลายสิ่งเดียวที่เราต้องการ (และมันยังทำให้ `%04d` ของ ffmpeg พังด้วย)

### สุดท้าย — ตอบข้อสังเกตของ CEO ตรง ๆ

ถูกครึ่งเดียว previz เดิมเร็วเพราะมันเป็น **viewport playblast ที่ข้าม render pipeline ทั้งก้อน** ไม่ใช่เพราะ Blender เร็ว ส่วน 20 นาทีคือ EEVEE render จริงที่ค่า default — anomaly ทั้งคู่ คนละทาง ทางปกติอยู่ตรงกลางและ**ใกล้ playblast มากกว่ามาก** แต่ไม่เท่า: การวัดเปรียบเทียบที่หาได้ชิ้นเดียว (2020) พบว่า render path สมัยใหม่ช้ากว่า OpenGL render ของ 2.79 อยู่ ~7 เท่า และปัญหา fixed per-frame overhead นั้นยังไม่ถูกแก้จนถึง Blender 4 คาดหวังว่า "เร็วพอจะ iterate กล้อง" ได้ แต่อย่าคาดหวังว่าจะเร็วเท่าเดิมเป๊ะ
