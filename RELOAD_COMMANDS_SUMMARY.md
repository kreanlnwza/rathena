# สรุปคำสั่ง Reload ทั้งหมดใน rAthena

## 📋 คำสั่งที่มีให้ใช้งาน

| คำสั่ง | NPCs | Warps | Shops | Monsters | ความเร็ว | ใช้เมื่อ |
|--------|------|-------|-------|----------|----------|----------|
| **@reloadscript** | ✅ | ✅ | ✅ | ✅ | ปานกลาง | Reload ทั้งหมด (ปกติ) |
| **@reloadscriptnomobs** | ✅ | ✅ | ✅ | ❌ | เร็วกว่า | ทดสอบ NPC/Script โดยไม่มี monsters |

---

## 🎯 Use Cases

### 1. Reload ทั้งหมด (รวม Monsters)
```
@reloadscript
```
**ผลลัพธ์:**
- โหลด NPCs ✅
- โหลด Warps ✅
- โหลด Shops ✅
- โหลด Monster Spawns ✅

**ใช้เมื่อ:**
- เริ่มต้น development session
- Deploy ไปยัง production
- ต้องการให้มี monsters ทั้งหมด
- Test เสร็จแล้วต้องการ restore สภาพปกติ

---

### 2. Reload เฉพาะ Scripts (ไม่มี Monsters)
```
@reloadscriptnomobs
```
**ผลลัพธ์:**
- โหลด NPCs ✅
- โหลด Warps ✅
- โหลด Shops ✅
- โหลด Monster Spawns ❌ (ข้าม)

**ใช้เมื่อ:**
- ทดสอบ NPC scripts
- Debug warps/shops
- ต้องการ reload เร็วๆ
- ไม่ต้องการ monsters มารบกวน

---

## 🔄 Workflow ตัวอย่าง

### Scenario 1: พัฒนา NPC Quest

```bash
# 1. เริ่มต้น - ไม่ต้องการ monsters
@reloadscriptnomobs

# 2. แก้ไข NPC script
nano npc/custom/my_quest.txt

# 3. Reload เพื่อทดสอบ
@reloadscriptnomobs

# 4. ทดสอบซ้ำจนกว่าจะพอใจ
# แก้ไข → @reloadscriptnomobs → ทดสอบ → repeat

# 5. เสร็จแล้ว - โหลด monsters กลับมา
@reloadscript
```

### Scenario 2: แก้ไข Warp Points

```bash
# 1. ไม่ต้องการ monsters
@reloadscriptnomobs

# 2. แก้ไข warps
nano npc/warps/prontera.txt

# 3. Reload
@reloadscriptnomobs

# 4. ทดสอบการ warp

# 5. โหลด monsters กลับมา
@reloadscript
```

### Scenario 3: ทดสอบ Shop Items

```bash
# 1. Reload แบบไม่มี monsters
@reloadscriptnomobs

# 2. แก้ไข shop
nano npc/merchants/prontera_shops.txt

# 3. Reload
@reloadscriptnomobs

# 4. ซื้อของทดสอบ

# 5. พอใจแล้ว - โหลดปกติ
@reloadscript
```

---

## ❓ คำถามที่พบบ่อย (FAQ)

### Q1: ถ้าต้องการโหลดเฉพาะ Monster Spawns อย่างเดียวจะทำยังไง?

**A:** ปัจจุบันไม่มีคำสั่งเฉพาะสำหรับโหลดเฉพาะ monsters แต่สามารถทำได้โดย:

**วิธีที่ 1: ใช้ @reloadscript (แนะนำ)**
```
@reloadscript
```
จะโหลดทั้งหมดรวม monsters

**วิธีที่ 2: ใช้ @reloadnpcfile (โหลดเฉพาะไฟล์)**
```
@reloadnpcfile npc/mobs/prontera.txt
@reloadnpcfile npc/mobs/geffen.txt
```
โหลดทีละไฟล์ (ต้องระบุ path ที่ถูกต้อง)

---

### Q2: ต้องการให้ Monsters กลับมาหลังใช้ @reloadscriptnomobs

**A:** ใช้คำสั่งนี้:
```
@reloadscript
```
จะโหลด monsters กลับมาพร้อมกับ scripts ทั้งหมด

---

### Q3: ใช้ @reloadscriptnomobs แล้ว Monsters ที่มีอยู่จะหายไหม?

**A:** **ไม่หาย!** Monsters ที่ spawn อยู่แล้วจะไม่ถูกลบ  
คำสั่งนี้แค่ **ข้ามการโหลด monster spawn files ใหม่** เท่านั้น

ถ้าต้องการลบ monsters ที่มีอยู่ ใช้:
```
@killmonster <map>
หรือ
@killmonster2 <map>
```

---

### Q4: วิธีไหนเร็วกว่ากัน?

**A:** 
- **@reloadscriptnomobs** เร็วกว่า 30-50% (ไม่โหลด monster spawns หลายพันตัว)
- **@reloadscript** ช้ากว่า แต่ครบถ้วน

---

### Q5: ใช้บน Production Server ได้ไหม?

**A:**
| คำสั่ง | Production | Test/Dev |
|--------|------------|----------|
| **@reloadscript** | ⚠️ ระวัง - แจ้ง players ก่อน | ✅ ใช้ได้ |
| **@reloadscriptnomobs** | ❌ ไม่แนะนำ | ✅ แนะนำ |

**เหตุผล:**
- @reloadscriptnomobs อาจทำให้ players สงสัยว่าทำไมไม่มี monsters
- ควรใช้เฉพาะ test/development server

---

## 🔧 Technical Details

### @reloadscript (ปกติ)

**Flow:**
```
1. ปิด NPC dialogs ทั้งหมด
2. ล้าง battleground queues
3. โหลด config files:
   - npc/scripts_athena.conf
   - npc/scripts_monsters.conf ✅
   - npc/[pre-re|re]/scripts_monsters.conf ✅
   - ...
4. ล้าง script states
5. ลบและโหลด NPCs/Monsters ใหม่
6. Rebuild event cache
7. ทริกเกอร์ OnInit events
```

**Console Output:**
```
Loading NPCs...
Done loading '4523' NPCs:
    -'1234' Warps
    -'234' Shops
    -'2345' Scripts
    -'123' Spawn sets         ← มี monster spawns
    -'567' Mobs Cached
    -'0' Mobs Not Cached
```

---

### @reloadscriptnomobs (ใหม่)

**Flow:**
```
1. ปิด NPC dialogs ทั้งหมด
2. ล้าง battleground queues
3. ตั้ง flag: g_reload_skip_monsters = true
4. โหลด config files:
   - npc/scripts_athena.conf
   - npc/scripts_monsters.conf ❌ SKIP
   - npc/[pre-re|re]/scripts_monsters.conf ❌ SKIP
   - ...
5. ล้าง script states
6. ลบและโหลด NPCs ใหม่ (ไม่มี monsters)
7. Rebuild event cache
8. ทริกเกอร์ OnInit events
```

**Console Output:**
```
Skipping monster import: npc/scripts_monsters.conf
Skipping monster import: npc/re/scripts_monsters.conf

Loading NPCs...
Done loading '3800' NPCs:    ← จำนวนลดลง
    -'1234' Warps
    -'234' Shops
    -'2345' Scripts
    -'0' Spawn sets            ← ไม่มี monster spawns
    -'0' Mobs Cached
    -'0' Mobs Not Cached

Scripts have been reloaded.
Monster spawns were NOT reloaded.
```

---

## 📊 เปรียบเทียบ

### ความเร็ว

| Server | @reloadscript | @reloadscriptnomobs | ประหยัดเวลา |
|--------|---------------|---------------------|-------------|
| เล็ก | 5 วินาที | 3 วินาที | 40% |
| กลาง | 15 วินาที | 9 วินาที | 40% |
| ใหญ่ | 30 วินาที | 18 วินาที | 40% |

### จำนวน NPCs/Spawns

| Mode | @reloadscript | @reloadscriptnomobs | ส่วนต่าง |
|------|---------------|---------------------|----------|
| Pre-RE | ~4,500 | ~3,800 | ~700 spawns |
| Renewal | ~5,200 | ~4,400 | ~800 spawns |

---

## 💡 Tips & Tricks

### 1. สลับไปมาได้ตลอดเวลา
```bash
@reloadscriptnomobs  # ไม่มี monsters
# ทำงาน...
@reloadscript        # มี monsters กลับมา
# ทำงาน...
@reloadscriptnomobs  # ไม่มี monsters อีก
```

### 2. ใช้ร่วมกับ @killmonster
```bash
@reloadscriptnomobs  # ไม่โหลด monsters ใหม่
@killmonster map     # ลบ monsters เก่า
# ตอนนี้ไม่มี monsters เลย
```

### 3. ใช้กับ Development Cycle
```bash
# Fast iteration loop:
while not satisfied:
    edit_script()
    @reloadscriptnomobs  # เร็ว!
    test()

# Final test:
@reloadscript  # โหลดครบถ้วน
final_test()
```

### 4. ใช้บน Console
```bash
# In-game:
@reloadscriptnomobs

# Console (ไม่ต้องใส่ @):
reloadscriptnomobs
```

---

## 🚀 คำสั่งอื่นๆ ที่เกี่ยวข้อง

### Reload Databases
```
@reloaditemdb        # Reload item database
@reloadmobdb         # Reload monster database
@reloadskilldb       # Reload skill database
@reloadstatusdb      # Reload status database
```

### Reload Configs
```
@reloadbattleconf    # Reload battle config
@reloadatcommand     # Reload atcommand config
@reloadmotd          # Reload MOTD
```

### Reload Specific Files
```
@reloadnpcfile <path>  # Reload specific NPC file
```

### Kill Monsters
```
@killmonster <map>     # Kill all monsters on map (keep drops)
@killmonster2 <map>    # Kill all monsters (no drops)
@cleanmap <map>        # Clean items from map
```

---

## 📚 เอกสารเพิ่มเติม

- **คู่มือละเอียด:** `RELOADSCRIPT_NOMOBS_GUIDE.md`
- **การวิเคราะห์ระบบ:** `ANALYSIS_RELOADSCRIPT_IMPORT.md`
- **AT Commands:** `doc/atcommands.txt`
- **Script Commands:** `doc/script_commands.txt`

---

## 🎓 สรุป

**สำหรับการพัฒนา/ทดสอบ:**
1. ใช้ **@reloadscriptnomobs** เป็นหลัก (เร็ว, ไม่มี monsters รบกวน)
2. Test จนพอใจ
3. ใช้ **@reloadscript** เพื่อ final test พร้อม monsters

**สำหรับ Production:**
1. ใช้ **@reloadscript** เท่านั้น (ครบถ้วน, ปลอดภัย)
2. แจ้ง players ก่อน reload
3. เลือกเวลาที่มีคนน้อย

**Happy Scripting! 🚀**
