# คู่มือการใช้คำสั่ง @reloadmonsters

## 🎯 วัตถุประสงค์

คำสั่ง `@reloadmonsters` ใช้สำหรับโหลดเฉพาะ **Monster Spawns** โดยไม่กระทบ Scripts อื่นๆ (NPCs, Warps, Shops)

---

## 📋 คำสั่งที่มีให้ใช้งาน

| คำสั่ง | NPCs | Warps | Shops | Monsters | ความเร็ว |
|--------|------|-------|-------|----------|----------|
| `@reloadscript` | ✅ Reload | ✅ Reload | ✅ Reload | ✅ Reload | ช้า (ครบถ้วน) |
| `@reloadscriptnomobs` | ✅ Reload | ✅ Reload | ✅ Reload | ❌ ข้าม | ปานกลาง |
| **`@reloadmonsters`** | ❌ **ไม่แตะ** | ❌ **ไม่แตะ** | ❌ **ไม่แตะ** | ✅ **Reload** | เร็ว |

---

## 🚀 การใช้งาน

### In-Game:
```
@reloadmonsters
```

### Console:
```
reloadmonsters
```

---

## 💡 Use Cases

### 1. แก้ไข Monster Spawns
```bash
# แก้ไขไฟล์ monster spawns
nano npc/mobs/prontera.txt

# Reload เฉพาะ monsters
@reloadmonsters
```

### 2. เปลี่ยน Rate การ Spawn
```bash
# แก้ไข spawn rate ในไฟล์
# prontera,150,150,0,0	monster	Poring	1002	10,60000,0,0
#                                               ^^  ^^^^^^
#                                            amount  respawn

# Reload
@reloadmonsters
```

### 3. เพิ่ม/ลด Monster ใน Map
```bash
# แก้ไขจำนวน monsters
nano npc/re/mobs/fields/prontera.txt

# Reload
@reloadmonsters

# ไม่กระทบ NPCs, Warps, Shops ที่มีอยู่
```

### 4. Test Monster Balance
```bash
# Scenario: ทดสอบ balance ของ monsters

# 1. Spawn monsters แบบเก่า
@reloadscript

# 2. Fight & test
# ...

# 3. แก้ไข monster spawns
nano npc/mobs/fields/geffen.txt

# 4. Reload เฉพาะ monsters (เร็ว!)
@reloadmonsters

# 5. Test อีกครั้ง
# NPCs/Quests ยังทำงานปกติ
```

---

## 🔧 ขั้นตอนการทำงาน

### Step-by-Step Flow:

```
1. Display: "Clearing all monsters..."

2. ลบ monsters ทั้งหมดที่มีอยู่
   - Loop ผ่านทุก block_list
   - ลบเฉพาะ BL_MOB type
   - ใช้ unit_free(bl, CLR_OUTSIGHT)

3. Clear mob spawn lookup index
   - mob_clear_spawninfo()
   - ล้าง cache ของ spawn data

4. Flush FIFOs
   - ส่ง packets ที่ค้างอยู่ออกไป

5. Display: "Reloading monster spawns..."

6. โหลด monster spawn files
   - npc/mobs/jail.txt
   - npc/mobs/pvp.txt
   - npc/mobs/towns.txt
   
7. โหลดตาม mode:
   [Renewal Mode]
   - โหลดจาก npc/re/scripts_monsters.conf
   
   [Pre-Renewal Mode]
   - โหลดจาก npc/pre-re/scripts_monsters.conf

8. Rebuild NPC events
   - npc_reload()
   - Rebuild event cache

9. Display: "Monster spawns have been reloaded."
10. Display: "Note: Other scripts (NPCs/Warps/Shops) were NOT reloaded."
```

---

## 📊 ผลลัพธ์ที่คาดหวัง

### Console Output:

```
[Info]: Clearing all monsters...
[Info]: Reloading monster spawns...
[Info]: Loading Renewal monster spawns...
[Status]: Loading [npc/mobs/jail.txt].
[Status]: Loading [npc/mobs/pvp.txt].
[Status]: Loading [npc/mobs/towns.txt].
[Status]: Loading [npc/re/mobs/fields/prontera.txt].
[Status]: Loading [npc/re/mobs/fields/geffen.txt].
... (monster files)

[Info]: Done loading 'X' NPCs:
    -'0' Warps         ← ไม่มี Warps ใหม่
    -'0' Shops         ← ไม่มี Shops ใหม่
    -'0' Scripts       ← ไม่มี NPCs ใหม่
    -'123' Spawn sets  ← มีเฉพาะ monster spawns
    -'567' Mobs Cached
    -'0' Mobs Not Cached

[Info]: Monster spawns have been reloaded.
[Info]: Note: Other scripts (NPCs/Warps/Shops) were NOT reloaded.
```

### In-Game Message:

```
Clearing all monsters...
Reloading monster spawns...
Monster spawns have been reloaded.
Note: Other scripts (NPCs/Warps/Shops) were NOT reloaded.
```

---

## ⚠️ ข้อควรระวัง

### 1. NPCs/Warps/Shops จะไม่ถูกแตะ
```
✅ ข้อดี: ไม่ต้อง reload scripts ทั้งหมด
✅ ข้อดี: Quest states ของผู้เล่นไม่หาย
✅ ข้อดี: NPC dialogs ยังคงทำงาน

⚠️ ข้อควรระวัง: ถ้ามี NPC script ที่เกี่ยวข้องกับ monster spawns 
   (เช่น spawn monsters ผ่าน script) จะไม่ถูก reload
```

### 2. Monsters เก่าจะถูกลบ
```
⚠️ Monsters ทั้งหมดที่มีอยู่จะถูกลบทันที
⚠️ Players ที่กำลังต่อสู้จะเห็น monsters หาย
⚠️ Drops จาก monsters จะไม่เกิด
```

### 3. Spawn ใหม่ตาม Config
```
✅ Monsters จะ spawn ใหม่ตาม config
✅ Spawn time จะเริ่มนับใหม่
✅ Boss timers จะ reset
```

---

## 🎯 เปรียบเทียบคำสั่ง

### Scenario: แก้ไข Monster Spawns

#### วิธีเก่า (ใช้ @reloadscript):
```
@reloadscript

ผลลัพธ์:
✅ โหลด monsters ใหม่
⚠️ โหลด NPCs ใหม่ (ไม่จำเป็น)
⚠️ โหลด Warps ใหม่ (ไม่จำเป็น)
⚠️ โหลด Shops ใหม่ (ไม่จำเป็น)
❌ Quest states อาจหาย
❌ ช้า (15-30 วินาที)
```

#### วิธีใหม่ (ใช้ @reloadmonsters):
```
@reloadmonsters

ผลลัพธ์:
✅ โหลดเฉพาะ monsters
✅ NPCs ยังทำงานต่อ
✅ Warps ไม่ถูกแตะ
✅ Shops ไม่ถูกแตะ
✅ Quest states ปลอดภัย
✅ เร็ว (3-5 วินาที)
```

---

## 📂 ไฟล์ Monster Spawns ที่จะถูกโหลด

### Common Files (ทุก Mode):
```
npc/mobs/jail.txt      - Jail monster spawns
npc/mobs/pvp.txt       - PvP arena monsters
npc/mobs/towns.txt     - Town monsters (Porings, etc.)
```

### Renewal Mode:
```
npc/re/scripts_monsters.conf
├── npc/re/mobs/fields/amatsu.txt
├── npc/re/mobs/fields/ayothaya.txt
├── npc/re/mobs/fields/brasilis.txt
├── npc/re/mobs/fields/comodo.txt
├── ... (all RE field maps)
├── npc/re/mobs/dungeons/abbey.txt
├── npc/re/mobs/dungeons/abyss.txt
└── ... (all RE dungeons)
```

### Pre-Renewal Mode:
```
npc/pre-re/scripts_monsters.conf
├── npc/pre-re/mobs/fields/amatsu.txt
├── npc/pre-re/mobs/fields/ayothaya.txt
├── ... (all Pre-RE field maps)
├── npc/pre-re/mobs/dungeons/abbey.txt
└── ... (all Pre-RE dungeons)
```

---

## 💻 ตัวอย่างการแก้ไข Monster Spawns

### ไฟล์: `npc/mobs/prontera.txt`

```c
//===== rAthena Script =======================================
//= Prontera Monster Spawns
//============================================================

//---------------- Normal Spawns -----------------
// prontera,<x>,<y>,<xs>,<ys>	monster	<name>	<mob_id>	<amount>,<respawn_time>,<respawn_time2>,<mob_event>

// เพิ่มจำนวน Poring
prontera,0,0,0,0	monster	Poring	1002	20,60000,0,0
//                                        ^^  ^^^^^^
//                                     จำนวน  Respawn time (ms)

// เพิ่ม Drops
prontera,0,0,0,0	monster	Drops	1113	10,60000,0,0

// เพิ่ม MVP Boss (ทดสอบ)
prontera,156,191,0,0	boss_monster	Eddga	1115	1,7200000,600000,0
//                                                   ^  ^^^^^^^  ^^^^^^
//                                                  จำนวน  2 ชม.  10 นาที
```

### หลังแก้ไข:
```bash
@reloadmonsters
```

**ผลลัพธ์:**
- Porings จะเพิ่มจาก 10 → 20 ตัว
- Drops จะปรากฏ
- Eddga boss จะ spawn (หาก respawn time ผ่านแล้ว)

---

## 🔄 Workflow แนะนำ

### สำหรับ Development:

```bash
# 1. Start server
./map-server

# 2. ใช้ @reloadscriptnomobs เพื่อไม่มี monsters
@reloadscriptnomobs

# 3. พัฒนา/ทดสอบ NPCs
# ... edit NPCs ...
@reloadscriptnomobs

# 4. เมื่อต้องการทดสอบ monsters
# แก้ไข monster spawns
nano npc/mobs/prontera.txt

# 5. Reload เฉพาะ monsters
@reloadmonsters

# 6. ทดสอบ monster balance
# ...

# 7. แก้ไข monsters อีก
nano npc/mobs/prontera.txt

# 8. Reload อีกครั้ง (เร็ว!)
@reloadmonsters

# 9. เสร็จแล้ว - reload ทั้งหมดเพื่อให้แน่ใจ
@reloadscript
```

---

## 🎓 Best Practices

### ✅ DO:

1. **ใช้เมื่อแก้ไขเฉพาะ monster spawns**
   ```
   @reloadmonsters  # เร็ว, ไม่กระทบ NPCs
   ```

2. **Test monster balance แบบ iterative**
   ```
   แก้ไข → @reloadmonsters → Test → repeat
   ```

3. **Backup ก่อนแก้ไข**
   ```bash
   cp npc/mobs/prontera.txt npc/mobs/prontera.txt.backup
   ```

4. **ใช้ร่วมกับ @killmonster**
   ```
   @killmonster prontera  # ฆ่า monsters เก่า
   @reloadmonsters        # โหลด config ใหม่
   ```

### ❌ DON'T:

1. **อย่าใช้เมื่อแก้ไข NPCs/Scripts**
   ```diff
   - @reloadmonsters  # จะไม่โหลด NPCs
   + @reloadscript    # ใช้คำสั่งนี้แทน
   ```

2. **อย่าใช้บน Production ที่มีผู้เล่นเยอะ**
   ```
   ⚠️ Monsters จะหายหมด → แจ้ง players ก่อน
   ```

3. **อย่าลืม reload ครั้งสุดท้ายก่อน deploy**
   ```
   @reloadscript  # Reload ทั้งหมดเพื่อความแน่ใจ
   ```

---

## 🐛 Troubleshooting

### ปัญหา: ไม่มี monsters spawn

**สาเหตุ:**
1. ไฟล์ monster spawns มี syntax error
2. Path ไฟล์ผิด
3. Map ไม่มีอยู่

**วิธีแก้:**
```bash
# ดู console errors
tail -f log/map-server.log

# ตรวจสอบ syntax
# ตัวอย่าง: ขาดเครื่องหมาย comma
prontera,0,0,0,0	monster	Poring	1002	10 60000,0,0
#                                           ^ ขาดตรงนี้
```

### ปัญหา: Monsters spawn ซ้ำ

**สาเหตุ:**
- ใช้ `@reloadmonsters` หลายครั้งโดยไม่ clear

**วิธีแก้:**
```
@reloadscript  # Reload ทั้งหมดเพื่อ reset
```

### ปัญหา: Monsters spawn แต่ไม่ขยับ

**สาเหตุ:**
- AI script มีปัญหา
- Monster database ไม่ match

**วิธีแก้:**
```
@reloadmobdb     # Reload monster database
@reloadmonsters  # Reload spawns
```

---

## 📝 ตัวอย่างการใช้งานจริง

### Example 1: Event - Double Poring Spawns

```bash
# Goal: เพิ่ม Poring spawns 2 เท่าสำหรับ event

# 1. Backup
cp npc/mobs/prontera.txt npc/mobs/prontera.txt.backup

# 2. แก้ไข
nano npc/mobs/prontera.txt
# เปลี่ยน:
# prontera,0,0,0,0	monster	Poring	1002	10,60000,0,0
# เป็น:
# prontera,0,0,0,0	monster	Poring	1002	20,60000,0,0

# 3. Reload เฉพาะ monsters
@reloadmonsters

# 4. ประกาศใน game
@broadcast Poring Event! Double spawns for 2 hours!

# 5. หลัง 2 ชั่วโมง - restore
mv npc/mobs/prontera.txt.backup npc/mobs/prontera.txt
@reloadmonsters
```

### Example 2: Boss Testing

```bash
# Goal: ทดสอบ MVP boss spawns

# 1. สร้างไฟล์ test
nano npc/mobs/test_mvp.txt

# เนื้อหา:
# prontera,156,191,0,0	boss_monster	Eddga	1115	1,30000,0,0
# //                                                 ^^^^^ 30 sec respawn

# 2. เพิ่มในconfig (ชั่วคราว)
echo "npc: npc/mobs/test_mvp.txt" >> npc/re/scripts_monsters.conf

# 3. Reload
@reloadmonsters

# 4. Test boss mechanics
# ...

# 5. ลบออกเมื่อเสร็จ
# ลบบรรทัดจาก npc/re/scripts_monsters.conf
@reloadmonsters
```

### Example 3: Map-Specific Monster Adjustment

```bash
# Goal: ปรับ monsters ในแมพ geffen เท่านั้น

# 1. แก้ไขเฉพาะ geffen
nano npc/re/mobs/fields/geffen.txt

# 2. Reload (จะโหลด geffen ด้วย)
@reloadmonsters

# 3. ทดสอบที่ geffen
@go geffen

# 4. ถ้า OK - commit changes
```

---

## 🎉 สรุป

คำสั่ง `@reloadmonsters` เหมาะสำหรับ:

✅ **แก้ไข monster spawns อย่างเดียว**  
✅ **Test monster balance แบบ quick iteration**  
✅ **Event ที่ต้องเปลี่ยน spawn rates**  
✅ **ไม่ต้องการกระทบ NPCs/Quests**  

**ข้อแตกต่างสำคัญ:**

| คำสั่ง | Target | Speed | Quest Safe? |
|--------|--------|-------|-------------|
| `@reloadscript` | ทุกอย่าง | ช้า | ⚠️ อาจหาย |
| `@reloadscriptnomobs` | ยกเว้น monsters | ปานกลาง | ✅ ปลอดภัย |
| **`@reloadmonsters`** | **เฉพาะ monsters** | **เร็ว** | ✅ **ปลอดภัย** |

**Happy Monster Spawning! 🐉🎮**
