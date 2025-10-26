# คู่มือการใช้งาน @reloadscriptnomobs

## 📋 สรุป

คำสั่ง `@reloadscriptnomobs` เป็นคำสั่งใหม่ที่ช่วยให้คุณสามารถ reload scripts ทั้งหมดโดยไม่ต้องโหลด monster spawns  
เหมาะสำหรับการทดสอบ NPC scripts, warps, shops และ mapflags โดยไม่ต้องกังวลเรื่อง monsters

---

## 🚀 การใช้งาน

### คำสั่งที่มีให้ใช้:

| คำสั่ง | การทำงาน | Monster Spawns |
|--------|----------|----------------|
| `@reloadscript` | Reload ทั้งหมดตามปกติ | ✅ โหลด monsters |
| `@reloadscriptnomobs` | Reload แต่ข้าม monsters | ❌ ไม่โหลด monsters |

### ตัวอย่างการใช้งาน:

```
# In-game
@reloadscriptnomobs

# Console
reloadscriptnomobs
```

---

## 📊 เปรียบเทียบ

### @reloadscript (ปกติ)
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

### @reloadscriptnomobs (ใหม่)
```
Skipping monster import: npc/scripts_monsters.conf
Skipping monster import: npc/re/scripts_monsters.conf

Loading NPCs...
Done loading '3800' NPCs:    ← จำนวนลดลง
    -'1234' Warps
    -'234' Shops
    -'2345' Scripts
    -'0' Spawn sets            ← ไม่มี monster spawns
    -'0' Mobs Cached           ← ไม่มี monsters
    -'0' Mobs Not Cached
```

---

## 💡 Use Cases

### 1. ทดสอบ NPC Scripts
```
# แก้ไข NPC script
nano npc/custom/my_npc.txt

# Reload โดยไม่โหลด monsters
@reloadscriptnomobs

# ทดสอบ NPC ได้เลย ไม่มี monsters มารบกวน
```

### 2. Debug Warps และ Shops
```
# แก้ไข warps/shops
nano npc/cities/prontera.txt

# Reload เร็วกว่า เพราะไม่ต้องโหลด monsters
@reloadscriptnomobs
```

### 3. ทดสอบ Mapflags
```
# แก้ไข mapflags
nano npc/mapflag/pvp.txt

# Reload เพื่อดูผล
@reloadscriptnomobs
```

### 4. Development Workflow
```
# วนรอบ: แก้ไข → reload → ทดสอบ → แก้ไข
while true; do
    edit_script();
    @reloadscriptnomobs;
    test();
done
```

---

## ⚡ ข้อดี

✅ **เร็วกว่า** - ไม่ต้องโหลด monster spawns หลายพันตัว  
✅ **สะดวกกว่า** - ไม่ต้อง comment/uncomment ใน .conf files  
✅ **ปลอดภัยกว่า** - ไม่เปลี่ยนแปลง config files  
✅ **ยืดหยุ่นกว่า** - สลับระหว่าง 2 โหมดได้ง่าย  
✅ **ทดสอบง่ายกว่า** - ไม่มี monsters มารบกวนการทดสอบ

---

## 🔧 Technical Details

### กลไกการทำงาน:

1. **Global Flag**
   - ตัวแปร `g_reload_skip_monsters` ถูกตั้งเป็น `true`
   
2. **Config Parsing**
   - ฟังก์ชัน `map_reloadnpc_sub()` ตรวจสอบทุกบรรทัด `import:`
   - ถ้าพบคำว่า `"monsters"` ใน path และ flag = true → ข้ามบรรทัดนั้น
   
3. **Skip Logic**
   ```cpp
   if (g_reload_skip_monsters && 
       strcmpi(w1, "import") == 0 && 
       strstr(w2, "monsters") != nullptr) {
       ShowInfo("Skipping monster import: %s\n", w2);
       continue;  // ข้ามไม่โหลด
   }
   ```

4. **Files ที่ถูกข้าม:**
   - `npc/scripts_monsters.conf`
   - `npc/re/scripts_monsters.conf`  
   - `npc/pre-re/scripts_monsters.conf`

---

## 📝 ไฟล์ที่เกี่ยวข้อง

### Source Files:
- `src/map/map.cpp` - เพิ่ม `map_reloadnpc_nomobs()` และ skip logic
- `src/map/map.hpp` - เพิ่ม function declaration
- `src/map/atcommand.cpp` - เพิ่ม `ACMD_FUNC(reloadscriptnomobs)`
- `conf/atcommands.yml` - เพิ่ม command help

### Config Files:
- `npc/re/scripts_main.conf` - ไม่ต้องแก้ไข (ใช้ default)
- `npc/pre-re/scripts_main.conf` - ไม่ต้องแก้ไข (ใช้ default)

---

## 🎯 Workflow แนะนำ

### Development Mode (ไม่ต้องการ monsters):
```bash
# 1. Start server
./map-server

# 2. Edit scripts
nano npc/custom/my_quest.txt

# 3. Reload (in-game)
@reloadscriptnomobs

# 4. Test immediately
# → ไม่มี monsters, reload เร็ว

# 5. Repeat steps 2-4 จนกว่าจะพอใจ
```

### Production Mode (ต้องการ monsters):
```bash
# เมื่อทดสอบเสร็จแล้ว ต้องการ monsters กลับมา
@reloadscript

# หรือ restart server เพื่อให้แน่ใจ
./map-server restart
```

---

## ⚠️ ข้อควรระวัง

### 1. Monster Spawns จะไม่ถูกโหลด
- ❌ Maps จะไม่มี monsters
- ❌ MVP bosses จะไม่มี
- ❌ Event monsters จะไม่มี

### 2. ผลกระทบต่อ Gameplay
- Players อาจสงสัยว่าทำไมไม่มี monsters
- ระบบที่ต้องอาศัย monsters อาจทำงานผิดพลาด
- ควรใช้เฉพาะเวลาทดสอบเท่านั้น

### 3. ใช้กับ Production Server
```diff
- ❌ ไม่ควรใช้บน production server ที่มีผู้เล่นเล่นอยู่
+ ✅ ควรใช้บน test/development server
```

---

## 🔍 Troubleshooting

### ปัญหา: ไม่เห็นข้อความ "Skipping monster import"

**สาเหตุ:**  
- Console output อาจไม่เปิด

**วิธีแก้:**  
```cpp
// ตรวจสอบว่าใน src/map/map.cpp มีบรรทัดนี้
ShowInfo("Skipping monster import: %s\n", w2);
```

### ปัญหา: Monsters ยังโหลดอยู่

**สาเหตุ:**  
- Server ยังไม่ได้ recompile

**วิธีแก้:**  
```bash
make clean
make server
./map-server restart
```

### ปัญหา: ต้องการ spawn monsters บางตัว

**วิธีแก้:**  
สร้างไฟล์ custom:
```bash
# npc/custom/test_monsters.txt
prontera,150,150,0,0	monster	Poring	1002	5,0,0,0

# npc/scripts_custom.conf
npc: npc/custom/test_monsters.txt
```

แล้ว:
```
@reloadscriptnomobs
```

---

## 📚 Examples

### Example 1: Quest NPC Testing
```c
// npc/custom/quest_test.txt
prontera,155,185,4	script	Quest Tester	100	{
    mes "[Quest Tester]";
    mes "Hello! This is a test quest.";
    next;
    mes "Do you want to start the quest?";
    if (select("Yes:No") == 2) {
        close;
    }
    setquest 60000;
    mes "Quest started!";
    close;
}
```

**Workflow:**
```
1. Edit the script
2. @reloadscriptnomobs
3. Talk to NPC → Test quest flow
4. Edit again if needed
5. @reloadscriptnomobs
6. Test again
```

### Example 2: Shop Testing
```c
// npc/custom/shop_test.txt
prontera,150,150,4	shop	Test Shop	100,501:50,502:100,503:150
```

**Workflow:**
```
@reloadscriptnomobs
→ Test shop items
→ Adjust prices
@reloadscriptnomobs
→ Test again
```

### Example 3: Warp Testing
```c
// npc/custom/warp_test.txt
prontera,156,191,0	warp	testwarp	1,1,izlude,128,98
```

**Workflow:**
```
@reloadscriptnomobs
→ Walk through warp
→ Check coordinates
→ Adjust if needed
@reloadscriptnomobs
```

---

## 🎓 Best Practices

### ✅ DO:

1. **ใช้สำหรับ Development**
   ```
   # Local testing
   @reloadscriptnomobs
   ```

2. **Test Incrementally**
   ```
   # แก้ไขทีละเล็กทีละน้อย
   edit → @reloadscriptnomobs → test → repeat
   ```

3. **Reload ปกติก่อน Deploy**
   ```
   # ก่อน deploy to production
   @reloadscript  # โหลด monsters กลับมา
   ```

4. **Document การใช้งาน**
   ```
   // Comment ใน script
   // Tested with @reloadscriptnomobs
   ```

### ❌ DON'T:

1. **ใช้บน Production ที่มีผู้เล่น**
   ```diff
   - @reloadscriptnomobs  // บน production
   + ใช้บน test server แทน
   ```

2. **ลืม Reload ปกติก่อน Release**
   ```diff
   - Deploy หลังใช้ @reloadscriptnomobs เลย
   + Deploy หลัง @reloadscript (reload ปกติ)
   ```

3. **Assume Monsters จะอยู่**
   ```diff
   - เขียน script ที่ assume มี monsters
   + ทดสอบทั้ง 2 กรณี: มี/ไม่มี monsters
   ```

---

## 🔄 Comparison: ก่อนและหลัง

### ก่อนมีคำสั่งใหม่:

```bash
# ต้อง comment ใน config files
nano npc/re/scripts_main.conf
# ค้นหาและ comment:
# // import: npc/scripts_monsters.conf
# // import: npc/re/scripts_monsters.conf

# Reload
@reloadscript

# เสร็จแล้วต้อง uncomment กลับ
nano npc/re/scripts_main.conf
# ลบ // ออก
```

**ปัญหา:**
- ❌ ยุ่งยาก ต้องแก้ไขหลายครั้ง
- ❌ ง่ายต่อการลืม uncomment
- ❌ เสียเวลา

### หลังมีคำสั่งใหม่:

```bash
# ไม่ต้องแก้ไข config files

# ไม่ต้องการ monsters
@reloadscriptnomobs

# ต้องการ monsters
@reloadscript
```

**ข้อดี:**
- ✅ สะดวกมาก แค่พิมพ์คำสั่ง
- ✅ ไม่แก้ไข config files
- ✅ สลับโหมดได้ทันที

---

## 📞 Support

หากมีปัญหาหรือข้อสงสัย:

1. **ตรวจสอบ Console Output**
   ```
   Skipping monster import: ...
   ```

2. **ดู Error Messages**
   ```
   ERROR: Map configuration file not found at: ...
   ```

3. **ตรวจสอบ Compilation**
   ```bash
   make clean
   make server
   ```

4. **Test บน Clean Server**
   ```bash
   # Backup current setup
   cp -r npc npc.backup
   
   # Test with default setup
   @reloadscriptnomobs
   ```

---

## 🎉 สรุป

คำสั่ง `@reloadscriptnomobs` ช่วยให้การพัฒนาและทดสอบ scripts สะดวกและรวดเร็วยิ่งขึ้น:

✨ **สะดวก** - ไม่ต้องแก้ไข config files  
⚡ **เร็ว** - ไม่ต้องโหลด monsters หลายพันตัว  
🎯 **แม่นยำ** - ทดสอบได้เฉพาะส่วนที่ต้องการ  
🔄 **ยืดหยุ่น** - สลับโหมดได้ง่าย

**Happy Scripting! 🚀**
