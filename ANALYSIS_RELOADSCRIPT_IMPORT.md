# รายงานวิเคราะห์: ระบบ Reload Script และ Import Mechanism ใน rAthena

## สรุปสำหรับผู้บริหาร (Executive Summary)

เอกสารฉบับนี้วิเคราะห์ระบบ reload script และกลไกการ import ใน rAthena source code โดยมุ่งเน้นที่การทำงานของคำสั่ง `@reloadscript` และการโหลด configuration files สำหรับ Pre-Renewal และ Renewal modes

**จุดสำคัญ:**
- คำสั่ง `@reloadscript` สามารถใช้ได้ทั้งใน game (โดยผู้เล่นที่มีสิทธิ์) และผ่าน console
- ระบบ import ทำงานที่ระดับ configuration files (.conf) ไม่ใช่ที่ระดับ script files
- การสลับระหว่าง Pre-Renewal และ Renewal mode กำหนดที่ compile time ผ่าน preprocessor define
- ระบบใช้ recursive import เพื่อโหลด configuration files แบบ modular

---

## 1. ภาพรวมการทำงานของ @reloadscript

### 1.1 จุดเริ่มต้น: AT Command

**ไฟล์:** `src/map/atcommand.cpp`  
**ฟังก์ชัน:** `ACMD_FUNC(reloadscript)` (บรรทัด 4359-4392)

```cpp
ACMD_FUNC(reloadscript){
    nullpo_retr(-1, sd);

    struct s_mapiterator* iter;
    map_session_data* pl_sd;

    // ปิดการโต้ตอบ NPC ของผู้เล่นทั้งหมด
    iter = mapit_getallusers();
    for( pl_sd = (TBL_PC*)mapit_first(iter); mapit_exists(iter); 
         pl_sd = (TBL_PC*)mapit_next(iter) ){
        pc_close_npc(pl_sd,1);
        clif_cutin( *pl_sd, "", 255 );
        pl_sd->state.block_action &= ~(PCBLOCK_ALL ^ PCBLOCK_IMMUNE);
        bg_queue_leave(pl_sd);
    }
    mapit_free(iter);

    // ล้าง battleground queues
    for (auto &bg : bg_queues) {
        for (auto &bg_sd : bg->teama_members)
            bg_team_leave(bg_sd, false, false);
        for (auto &bg_sd : bg->teamb_members)
            bg_team_leave(bg_sd, false, false);
        bg_queue_clear(bg, true);
    }

    flush_fifos();
    
    // เริ่มขั้นตอนการ reload
    map_reloadnpc(true);    // โหลด config files ใหม่
    script_reload();         // ล้าง script states และ user functions
    npc_reload();            // ลบและโหลด NPCs ทั้งหมดใหม่

    clif_displaymessage(fd, msg_txt(sd,100)); // Scripts have been reloaded.

    return 0;
}
```

**การใช้งาน:**
- ในเกม: `@reloadscript`
- ผ่าน console: พิมพ์ `reloadscript` (ใช้ระบบ atcommand เดียวกัน แต่ไม่ต้องใส่ @)

**Restriction Flags:** `ATCMD_NOSCRIPT` - ไม่สามารถเรียกใช้ผ่าน script command `atcommand` หรือ `useatcmd` ได้

---

## 2. ขั้นตอนการทำงานแบบละเอียด (Step-by-Step Flow)

### ขั้นตอนที่ 1: การเตรียมการ (Preparation Phase)

**ดำเนินการโดย:** `ACMD_FUNC(reloadscript)` ใน `src/map/atcommand.cpp`

1. **ปิดการโต้ตอบ NPC ทั้งหมด**
   - วนลูปผู้เล่นทั้งหมดที่ออนไลน์
   - เรียก `pc_close_npc()` เพื่อปิด NPC dialog
   - ล้าง cutin (ภาพตัวละครที่แสดงในหน้าจอ)
   - ยกเลิก action blocks (ยกเว้น PCBLOCK_IMMUNE)
   - ลบผู้เล่นออกจาก battleground queues

2. **ล้าง Battleground Data**
   - วนลูปทุก battleground queue
   - ดึงผู้เล่นออกจากทีม A และทีม B
   - ล้าง queue ทั้งหมด

3. **Flush Network Buffers**
   - เรียก `flush_fifos()` เพื่อให้แน่ใจว่าข้อมูล network ถูกส่งทั้งหมด

### ขั้นตอนที่ 2: โหลด Configuration Files ใหม่ (Config Reload Phase)

**ดำเนินการโดย:** `map_reloadnpc(bool clear)` ใน `src/map/map.cpp` (บรรทัด 4248-4258)

```cpp
void map_reloadnpc(bool clear)
{
    if (clear)
        npc_addsrcfile("clear", false); // ล้าง list ของ script files

    #ifdef RENEWAL
        map_reloadnpc_sub("npc/re/scripts_main.conf");
    #else
        map_reloadnpc_sub("npc/pre-re/scripts_main.conf");
    #endif
}
```

**การทำงาน:**
1. ถ้า `clear = true` จะเรียก `npc_addsrcfile("clear", false)` เพื่อล้าง vector `npc_src_files`
2. ตรวจสอบ mode ที่ compile time:
   - ถ้ามี `#define RENEWAL` → โหลด `"npc/re/scripts_main.conf"`
   - ถ้าไม่มี → โหลด `"npc/pre-re/scripts_main.conf"`
3. เรียก `map_reloadnpc_sub()` เพื่อ parse configuration file

### ขั้นตอนที่ 3: การ Parse Configuration Files (Config Parsing Phase)

**ดำเนินการโดย:** `map_reloadnpc_sub(const char *cfgName)` ใน `src/map/map.cpp` (บรรทัด 4206-4246)

```cpp
void map_reloadnpc_sub(const char *cfgName)
{
    char line[1024], w1[1024], w2[1024];
    FILE *fp;

    fp = fopen(cfgName,"r");
    if( fp == nullptr )
    {
        ShowError("Map configuration file not found at: %s\n", cfgName);
        return;
    }

    while( fgets(line, sizeof(line), fp) )
    {
        char* ptr;

        // ข้าม comment lines
        if( line[0] == '/' && line[1] == '/' )
            continue;
        if( (ptr = strstr(line, "//")) != nullptr )
            *ptr = '\n'; // ตัด comments ออก
        if( sscanf(line, "%1023[^:]: %1023[^\t\r\n]", w1, w2) < 2 )
            continue;

        // ตัด trailing spaces
        ptr = w2 + strlen(w2);
        while (--ptr >= w2 && *ptr == ' ');
        ptr++;
        *ptr = '\0';

        if (strcmpi(w1, "npc") == 0)
            npc_addsrcfile(w2, false);      // เพิ่ม NPC source file
        else if (strcmpi(w1, "delnpc") == 0)
            npc_delsrcfile(w2);              // ลบ NPC source file
        else if (strcmpi(w1, "import") == 0)
            map_reloadnpc_sub(w2);           // Recursive import
        else
            ShowWarning("Unknown setting '%s' in file %s\n", w1, cfgName);
    }

    fclose(fp);
}
```

**Directives ที่รองรับ:**
1. `npc: <filepath>` - เพิ่มไฟล์ NPC script เข้าไปใน list
2. `delnpc: <filepath>` - ลบไฟล์ NPC script ออกจาก list
3. `import: <filepath>` - โหลด configuration file อื่นแบบ recursive

**ตัวอย่าง Configuration File:**

```
// npc/re/scripts_main.conf
npc: npc/other/Global_Functions.txt
npc: npc/other/CashShop_Functions.txt

// Import common files
import: npc/scripts_athena.conf
import: npc/scripts_guild.conf
import: npc/scripts_jobs.conf
import: npc/scripts_mapflags.conf
import: npc/scripts_monsters.conf
import: npc/scripts_warps.conf

// Import renewal-specific files
import: npc/re/scripts_athena.conf
import: npc/re/scripts_guild.conf
import: npc/re/scripts_jobs.conf
import: npc/re/scripts_mapflags.conf
import: npc/re/scripts_monsters.conf
import: npc/re/scripts_warps.conf

import: npc/scripts_custom.conf
```

### ขั้นตอนที่ 4: การเพิ่ม Source Files (File Registration Phase)

**ดำเนินการโดย:** `npc_addsrcfile(const char* name, bool loadscript)` ใน `src/map/npc.cpp` (บรรทัด 3602-3626)

```cpp
int32 npc_addsrcfile(const char* name, bool loadscript)
{
    // Special case: ล้าง list
    if( strcmpi(name, "clear") == 0 )
    {
        npc_src_files.clear();
        return 1;
    }

    // ตรวจสอบว่าเป็นไฟล์จริง
    if(check_filepath(name)!=2){
        ShowError("npc_addsrcfile: Can't find source file \"%s\"\n", name );
        return 0;
    }

    // ตรวจสอบว่ามีอยู่ใน list แล้วหรือไม่
    if (util::vector_exists(npc_src_files, name)) {
        return 0; // พบแล้ว ไม่ต้องเพิ่มซ้ำ
    }

    // เพิ่มเข้า vector
    npc_src_files.push_back(name);

    // ถ้า loadscript = true จะ parse ทันที (ไม่ใช้ในกรณี reload)
    if (loadscript)
        return npc_parsesrcfile(name);

    return 1;
}
```

**โครงสร้างข้อมูล:** `std::vector<std::string> npc_src_files` (global variable ใน `src/map/npc.cpp`)

### ขั้นตอนที่ 5: ล้าง Script State (Script Cleanup Phase)

**ดำเนินการโดย:** `script_reload()` ใน `src/map/script.cpp` (บรรทัด 4877-4903)

```cpp
void script_reload(void) {
    int32 i;
    DBIterator *iter;
    struct script_state *st;

    // ล้าง user-defined functions
    userfunc_db->clear(userfunc_db, db_script_free_code_sub);
    db_clear(scriptlabel_db);

    // ล้าง @commands ที่สร้างจาก script
    for( i = 0; i < atcmd_binding_count; i++ ) {
        aFree(atcmd_binding[i]);
    }

    if( atcmd_binding_count != 0 )
        aFree(atcmd_binding);

    atcmd_binding_count = 0;

    // ล้าง script states ที่กำลังทำงานอยู่
    iter = db_iterator(st_db);
    for( st = static_cast<script_state *>(dbi_first(iter)); 
         dbi_exists(iter); 
         st = static_cast<script_state *>(dbi_next(iter)) )
        script_free_state(st);
    dbi_destroy(iter);
    db_clear(st_db);

    // โหลด mapreg ใหม่
    mapreg_reload();
}
```

**สิ่งที่ถูกล้าง:**
1. User-defined functions (`userfunc_db`)
2. Script labels (`scriptlabel_db`)
3. Script-based atcommand bindings
4. Active script states (`st_db`)
5. Map registry variables (mapreg)

### ขั้นตอนที่ 6: ลบและโหลด NPCs ใหม่ (NPC Reload Phase)

**ดำเนินการโดย:** `npc_reload()` ใน `src/map/npc.cpp` (บรรทัด 6043-6131)

```cpp
int32 npc_reload(void) {
    int32 npc_new_min = npc_id;
    struct s_mapiterator* iter;
    struct block_list* bl;

    // ล้าง guild flag cache
    guild_flags_clear();

    // ล้าง NPC path database
    npc_clear_pathlist();
    db_clear(npc_path_db);

    // ล้าง NPC name และ event databases
    db_clear(npcname_db);
    db_clear(ev_db);

    // โหลด market data จาก SQL (สำหรับ PACKETVER >= 20131223)
    #if PACKETVER >= 20131223
        npc_market_fromsql();
    #endif

    // ลบ NPCs และ monsters ทั้งหมด
    iter = mapit_geteachiddb();
    for( bl = (struct block_list*)mapit_first(iter); 
         mapit_exists(iter); 
         bl = (struct block_list*)mapit_next(iter) ) {
        switch(bl->type) {
        case BL_NPC:
            if( bl->id != fake_nd->id )  // ไม่ลบ fake_nd
                npc_unload((struct npc_data *)bl, false);
            break;
        case BL_MOB:
            unit_free(bl,CLR_OUTSIGHT);
            break;
        }
    }
    mapit_free(iter);

    // ล้าง dynamic mob lists
    if( battle_config.dynamic_mobs ){
        for (int32 i = 0; i < map_num; i++) {
            for( int16 j = 0; j < MAX_MOB_LIST_PER_MAP; j++ ){
                struct map_data *mapdata = map_getmapdata(i);

                if (mapdata->moblist[j] != nullptr) {
                    aFree(mapdata->moblist[j]);
                    mapdata->moblist[j] = nullptr;
                }
                if( mapdata->mob_delete_timer != INVALID_TIMER ) {
                    delete_timer(mapdata->mob_delete_timer, map_removemobs_timer);
                    mapdata->mob_delete_timer = INVALID_TIMER;
                }

                if( mapdata->npc_num > 0 ){
                    ShowWarning( "npc_reload: %d npcs weren't removed at map %s!\n",
                                mapdata->npc_num, mapdata->name );
                }
            }
        }
    }

    // ล้าง mob spawn lookup index
    mob_clear_spawninfo();

    // Reset counters
    npc_warp = npc_shop = npc_script = 0;
    npc_mob = npc_cache_mob = npc_delay_mob = 0;

    // Reset mapflags
    map_flags_init();

    // โหลด NPC source files ทั้งหมด
    npc_loadsrcfiles();

    // โหลด databases อื่นๆ
    stylist_db.reload();
    barter_db.reload();

    // อ่าน NPC Script Events cache ใหม่
    npc_read_event_script();

    // ทริกเกอร์ OnInit events
    npc_event_doall( script_config.agit_init_event_name );
    npc_event_doall( script_config.agit_init2_event_name );
    npc_event_doall( script_config.agit_init3_event_name );

    // instance_init() จะเรียก npc_event_doall(OnInstanceInit)

    return 0;
}
```

**ขั้นตอนการลบ NPCs:**
1. ล้าง guild flags และ path databases
2. ล้าง NPC name และ event databases
3. วนลูปทุก block_list และลบ NPCs/monsters
4. ล้าง dynamic mob data
5. Reset map flags

### ขั้นตอนที่ 7: โหลด Source Files (File Loading Phase)

**ดำเนินการโดย:** `npc_loadsrcfiles()` ใน `src/map/npc.cpp` (บรรทัด 3643-3661)

```cpp
void npc_loadsrcfiles() {
    ShowStatus("Loading NPCs...\n");
    
    // วนลูปทุกไฟล์ใน npc_src_files vector
    for (const auto& file : npc_src_files) {
        #ifdef DETAILED_LOADING_OUTPUT
            ShowStatus("Loading NPC file: %s" CL_CLL "\r", file.c_str());
        #endif
        npc_parsesrcfile(file.c_str());
    }
    
    int32 npc_total = npc_warp + npc_shop + npc_script;

    ShowInfo ("Done loading '" CL_WHITE "%d" CL_RESET "' NPCs:" CL_CLL "\n"
        "\t-'" CL_WHITE "%d" CL_RESET "' Warps\n"
        "\t-'" CL_WHITE "%d" CL_RESET "' Shops\n"
        "\t-'" CL_WHITE "%d" CL_RESET "' Scripts\n"
        "\t-'" CL_WHITE "%d" CL_RESET "' Spawn sets\n"
        "\t-'" CL_WHITE "%d" CL_RESET "' Mobs Cached\n"
        "\t-'" CL_WHITE "%d" CL_RESET "' Mobs Not Cached\n",
        npc_total, npc_warp, npc_shop, npc_script, 
        npc_mob, npc_cache_mob, npc_delay_mob);
}
```

### ขั้นตอนที่ 8: Parse NPC Files (NPC Parsing Phase)

**ดำเนินการโดย:** `npc_parsesrcfile(const char* filepath)` ใน `src/map/npc.cpp` (บรรทัด 5631-5807)

**การทำงาน:**
1. อ่านไฟล์ทั้งหมดเข้า memory buffer
2. ตรวจสอบ UTF-8 BOM (และแสดง error ถ้าพบ)
3. Parse แต่ละบรรทัดโดยแยก tab-separated values (w1, w2, w3, w4)
4. วิเคราะห์และสร้าง NPCs ตามประเภท:
   - **Warps:** `npc_parse_warp()`
   - **Shops:** `npc_parse_shop()` (shop, cashshop, itemshop, pointshop, marketshop)
   - **Scripts:** `npc_parse_script()`
   - **Functions:** `npc_parse_function()`
   - **Duplicates:** `npc_parse_duplicate()`
   - **Monsters:** `npc_parse_mob()`
   - **Mapflags:** `npc_parse_mapflag()`

**รูปแบบ NPC Script:**
```
<map name>,<x>,<y>,<facing>	<type>	<name>	<sprite id>[,<triggerX>,<triggerY>]	{<script>}
```

**ตัวอย่าง:**
```
prontera,150,150,4	script	Test NPC	100	{
    mes "Hello World!";
    close;
}
```

---

## 3. กลไกการ Import และ Mode Switching

### 3.1 Import Mechanism

**หลักการทำงาน:**
- ระบบ import ทำงานที่ระดับ **configuration files** (.conf)
- **ไม่ทำงานที่ระดับ NPC script files** (.txt)
- ใช้ recursive parsing เพื่อรองรับ nested imports

**Flow การ Import:**

```
map_reloadnpc()
    └─> map_reloadnpc_sub("npc/re/scripts_main.conf")
            ├─> npc_addsrcfile("npc/other/Global_Functions.txt")
            ├─> npc_addsrcfile("npc/other/CashShop_Functions.txt")
            ├─> map_reloadnpc_sub("npc/scripts_athena.conf")  [RECURSIVE]
            │       ├─> npc_addsrcfile("...")
            │       └─> ...
            ├─> map_reloadnpc_sub("npc/scripts_monsters.conf")  [RECURSIVE]
            │       ├─> npc_addsrcfile("npc/mobs/jail.txt")
            │       ├─> npc_addsrcfile("npc/mobs/pvp.txt")
            │       └─> npc_addsrcfile("npc/mobs/towns.txt")
            ├─> map_reloadnpc_sub("npc/re/scripts_monsters.conf")  [RECURSIVE]
            │       └─> ...
            └─> map_reloadnpc_sub("npc/scripts_custom.conf")  [RECURSIVE]
```

**ข้อดีของระบบ Import:**
1. **Modularity** - แยกไฟล์ตามหมวดหมู่ (jobs, monsters, warps, etc.)
2. **Maintainability** - แก้ไขส่วนใดส่วนหนึ่งโดยไม่กระทบส่วนอื่น
3. **Flexibility** - เพิ่ม/ลด content โดยการ comment/uncomment บรรทัด import
4. **Code Reuse** - ใช้ไฟล์ common ร่วมกันระหว่าง Pre-RE และ RE modes

### 3.2 Mode Switching: Pre-Renewal vs Renewal

**การกำหนด Mode:**

Mode ถูกกำหนดที่ **compile time** ผ่าน preprocessor define ใน:

**ไฟล์:** `src/config/renewal.hpp` (บรรทัด 24)

```cpp
/// Game renewal server mode
/// (disable by commenting the line)
///
/// Leave this line to enable renewal specific support such as renewal formulas
#define RENEWAL
```

**วิธีสลับ Mode:**
1. **Renewal Mode:** เปิด (uncomment) `#define RENEWAL`
2. **Pre-Renewal Mode:** ปิด (comment) `// #define RENEWAL`
3. Recompile server

**การเลือก Config File:**

ใน `src/map/map.cpp` (บรรทัด 4253-4257):

```cpp
#ifdef RENEWAL
    map_reloadnpc_sub("npc/re/scripts_main.conf");
#else
    map_reloadnpc_sub("npc/pre-re/scripts_main.conf");
#endif
```

**โครงสร้างไฟล์ Configuration:**

```
npc/
├── scripts_athena.conf       # Common Athena scripts
├── scripts_guild.conf        # Common guild scripts
├── scripts_jobs.conf         # Common job scripts
├── scripts_mapflags.conf     # Common mapflags
├── scripts_monsters.conf     # Common monster spawns
├── scripts_warps.conf        # Common warps
├── scripts_custom.conf       # Custom user scripts
│
├── pre-re/
│   ├── scripts_main.conf     # Pre-Renewal main config
│   ├── scripts_athena.conf   # Pre-Renewal specific scripts
│   ├── scripts_jobs.conf     # Pre-Renewal jobs
│   ├── scripts_monsters.conf # Pre-Renewal monster spawns
│   └── ...
│
└── re/
    ├── scripts_main.conf     # Renewal main config
    ├── scripts_athena.conf   # Renewal specific scripts
    ├── scripts_guild.conf    # Renewal guild scripts (3rd WoE)
    ├── scripts_jobs.conf     # Renewal jobs (extended classes)
    ├── scripts_monsters.conf # Renewal monster spawns
    └── ...
```

**ความแตกต่างหลักระหว่าง Pre-RE และ RE:**

| Feature | Pre-Renewal | Renewal |
|---------|-------------|---------|
| **Main Config** | `npc/pre-re/scripts_main.conf` | `npc/re/scripts_main.conf` |
| **Jobs** | 1st/2nd/transcendent classes | Extended 3rd classes, Doram |
| **Maps** | Classic maps | New maps (Bifrost, Eclage, etc.) |
| **Monsters** | Classic mob stats | Adjusted HP/ATK/DEF |
| **Guild** | WoE 1 & 2 | WoE 1, 2 & 3 |
| **Instances** | Fewer instances | More instances |
| **Battle Formulas** | Pre-RE damage formulas | RE damage formulas |

**ตัวอย่างการโหลด Monster Scripts:**

**Common monsters** (`npc/scripts_monsters.conf`):
```
npc: npc/mobs/jail.txt
npc: npc/mobs/pvp.txt
npc: npc/mobs/towns.txt
```

**Pre-Renewal monsters** (`npc/pre-re/scripts_monsters.conf`):
```
npc: npc/pre-re/mobs/fields/amatsu.txt
npc: npc/pre-re/mobs/fields/ayothaya.txt
// ... รวม mob spawns สำหรับ classic maps
```

**Renewal monsters** (`npc/re/scripts_monsters.conf`):
```
npc: npc/re/mobs/fields/amatsu.txt
npc: npc/re/mobs/fields/ayothaya.txt
npc: npc/re/mobs/fields/bifrost.txt    # New RE map
npc: npc/re/mobs/fields/eclage.txt     # New RE map
// ... รวม mob spawns สำหรับ RE maps ด้วย
```

**Loading Flow ตาม Mode:**

```
Renewal Mode:
    map_reloadnpc("npc/re/scripts_main.conf")
        ├─> Common files (npc/scripts_*.conf)
        └─> Renewal files (npc/re/scripts_*.conf)

Pre-Renewal Mode:
    map_reloadnpc("npc/pre-re/scripts_main.conf")
        ├─> Common files (npc/scripts_*.conf)
        └─> Pre-Renewal files (npc/pre-re/scripts_*.conf)
```

---

## 4. ไฟล์และฟังก์ชันที่เกี่ยวข้อง

### 4.1 Source Files

| ไฟล์ | บรรทัดสำคัญ | คำอธิบาย |
|------|------------|----------|
| `src/map/atcommand.cpp` | 4359-4392 | คำสั่ง @reloadscript |
| `src/map/map.cpp` | 4206-4246, 4248-4258 | ระบบโหลด config และ import |
| `src/map/npc.cpp` | 3602-3626, 3643-3661, 5631-5807, 6043-6131 | การจัดการ NPC files และ parsing |
| `src/map/script.cpp` | 4877-4903 | การ reload script engine |
| `src/config/renewal.hpp` | 24 | กำหนด RENEWAL mode |

### 4.2 Functions และ Call Chain

#### Main Reload Flow:

```
@reloadscript (atcommand.cpp:4359)
    │
    ├─> map_reloadnpc(true) (map.cpp:4248)
    │   │
    │   ├─> npc_addsrcfile("clear", false) (npc.cpp:3602)
    │   │   └─> npc_src_files.clear()
    │   │
    │   └─> map_reloadnpc_sub("npc/[pre-]re/scripts_main.conf") (map.cpp:4206)
    │       └─> [Recursive] สำหรับทุก import directive
    │           ├─> npc_addsrcfile(filepath, false) สำหรับ "npc:" directive
    │           └─> map_reloadnpc_sub(filepath) สำหรับ "import:" directive
    │
    ├─> script_reload() (script.cpp:4877)
    │   ├─> userfunc_db->clear()
    │   ├─> db_clear(scriptlabel_db)
    │   ├─> ล้าง atcmd_binding
    │   ├─> script_free_state() สำหรับทุก active script
    │   └─> mapreg_reload()
    │
    └─> npc_reload() (npc.cpp:6043)
        ├─> guild_flags_clear()
        ├─> npc_clear_pathlist()
        ├─> db_clear(npcname_db)
        ├─> db_clear(ev_db)
        ├─> npc_market_fromsql() [PACKETVER >= 20131223]
        ├─> [Loop] ลบ NPCs และ MOBs ทั้งหมด
        │   └─> npc_unload() / unit_free()
        ├─> mob_clear_spawninfo()
        ├─> map_flags_init()
        ├─> npc_loadsrcfiles() (npc.cpp:3643)
        │   └─> [Loop] npc_parsesrcfile() สำหรับทุกไฟล์ใน npc_src_files
        │       └─> npc_parse_*() ตามประเภท NPC
        ├─> stylist_db.reload()
        ├─> barter_db.reload()
        ├─> npc_read_event_script() (npc.cpp:5989)
        └─> npc_event_doall() สำหรับ OnInit events
```

#### Key Functions:

**1. `ACMD_FUNC(reloadscript)` - Entry Point**
- **ไฟล์:** `src/map/atcommand.cpp:4359-4392`
- **หน้าที่:** จัดการคำสั่ง @reloadscript
- **เรียกโดย:** atcommand system (in-game หรือ console)

**2. `map_reloadnpc(bool clear)` - Config Loader**
- **ไฟล์:** `src/map/map.cpp:4248-4258`
- **หน้าที่:** เลือกและโหลด main config file ตาม mode
- **พารามิเตอร์:** `clear` - ล้าง npc_src_files ก่อนโหลดหรือไม่

**3. `map_reloadnpc_sub(const char *cfgName)` - Config Parser**
- **ไฟล์:** `src/map/map.cpp:4206-4246`
- **หน้าที่:** Parse configuration file และจัดการ directives
- **รองรับ:** `npc:`, `delnpc:`, `import:`
- **การทำงาน:** Recursive สำหรับ import

**4. `npc_addsrcfile(const char* name, bool loadscript)` - File Registrar**
- **ไฟล์:** `src/map/npc.cpp:3602-3626`
- **หน้าที่:** เพิ่มไฟล์เข้า npc_src_files vector
- **Special case:** `name = "clear"` เพื่อล้าง vector

**5. `script_reload()` - Script Engine Cleaner**
- **ไฟล์:** `src/map/script.cpp:4877-4903`
- **หน้าที่:** ล้าง script states, functions, labels
- **ผลกระทบ:** ทุก script ที่กำลังทำงานจะถูกยกเลิก

**6. `npc_reload()` - NPC Manager**
- **ไฟล์:** `src/map/npc.cpp:6043-6131`
- **หน้าที่:** ลบและโหลด NPCs ใหม่ทั้งหมด
- **ขั้นตอน:**
  1. ล้าง databases และ caches
  2. ลบ NPCs/MOBs ที่มีอยู่
  3. โหลด source files
  4. Rebuild event cache
  5. ทริกเกอร์ OnInit events

**7. `npc_loadsrcfiles()` - File Loader**
- **ไฟล์:** `src/map/npc.cpp:3643-3661`
- **หน้าที่:** วนลูปโหลดทุกไฟล์ใน npc_src_files
- **เรียก:** `npc_parsesrcfile()` สำหรับแต่ละไฟล์

**8. `npc_parsesrcfile(const char* filepath)` - NPC Parser**
- **ไฟล์:** `src/map/npc.cpp:5631-5807`
- **หน้าที่:** อ่านและ parse NPC script files
- **รองรับ:** Warps, Shops, Scripts, Monsters, Mapflags, Functions, Duplicates
- **ไม่รองรับ:** import directive (จัดการที่ config level)

**9. `npc_read_event_script()` - Event Cache Builder**
- **ไฟล์:** `src/map/npc.cpp:5989-6027`
- **หน้าที่:** สร้าง cache ของ script events (OnLogin, OnLogout, etc.)
- **Event Types:** NPCE_LOGIN, NPCE_LOGOUT, NPCE_LOADMAP, NPCE_BASELVUP, NPCE_JOBLVUP, NPCE_DIE, NPCE_KILLPC, NPCE_KILLNPC, NPCE_IDENTIFY

---

## 5. Data Structures

### 5.1 NPC Source Files List

**Variable:** `std::vector<std::string> npc_src_files`  
**Location:** `src/map/npc.cpp:43`  
**Purpose:** เก็บ list ของ NPC source files ที่จะโหลด

**Operations:**
- **Add:** `npc_src_files.push_back(filepath)`
- **Clear:** `npc_src_files.clear()`
- **Check exists:** `util::vector_exists(npc_src_files, filepath)`
- **Remove:** `util::vector_erase_if_exists(npc_src_files, filepath)`

### 5.2 Databases

**1. `npcname_db` (DBMap\*)**
- **Key:** NPC name (string)
- **Value:** `struct npc_data*`
- **Purpose:** NPC name lookup

**2. `ev_db` (DBMap\*)**
- **Key:** Event name (string, format: "NpcName::EventLabel")
- **Value:** `struct event_data*`
- **Purpose:** Event lookup และ execution

**3. `userfunc_db` (DBMap\*)**
- **Key:** Function name (string)
- **Value:** Script bytecode
- **Purpose:** User-defined script functions

**4. `scriptlabel_db` (DBMap\*)**
- **Key:** Label name (string)
- **Value:** Script position
- **Purpose:** Label lookup ใน scripts

**5. `st_db` (DBMap\*)**
- **Key:** Script state ID (int32)
- **Value:** `struct script_state*`
- **Purpose:** Active script states

### 5.3 Counters

**Global counters ใน `src/map/npc.cpp`:**

```cpp
static int32 npc_id=START_NPC_NUM;    // NPC ID counter
static int32 npc_warp=0;              // จำนวน warps
static int32 npc_shop=0;              // จำนวน shops
static int32 npc_script=0;            // จำนวน script NPCs
static int32 npc_mob=0;               // จำนวน mob spawns
static int32 npc_delay_mob=0;         // จำนวน delayed mobs
static int32 npc_cache_mob=0;         // จำนวน cached mobs
```

**Reset ใน `npc_reload()` บรรทัด 6104:**
```cpp
npc_warp = npc_shop = npc_script = 0;
npc_mob = npc_cache_mob = npc_delay_mob = 0;
```

---

## 6. ข้อควรระวังและ Best Practices

### 6.1 ข้อควรระวัง (Caveats)

1. **การใช้ @reloadscript มีผลกระทบต่อ:**
   - ผู้เล่นที่กำลังคุยกับ NPC → จะถูกปิด dialog
   - Script variables ทั้งหมด → จะถูกรีเซ็ต (ยกเว้น permanent variables)
   - Timers ที่กำลังทำงาน → จะถูกยกเลิก
   - Battleground queues → จะถูกล้าง
   - Instances ที่กำลังทำงาน → อาจมีปัญหา (ดู note ในไฟล์ instance scripts)

2. **Data Loss ที่อาจเกิดขึ้น:**
   - Temporary variables (`@variable`, `.variable`)
   - NPC progress (ยกเว้น quest flags ที่เก็บใน database)
   - Timer-based events ที่กำลังทำงาน
   - Shop inventory (สำหรับ dynamic shops)

3. **ผลกระทบต่อ Performance:**
   - Server จะมีการ lag ชั่วคราวในขณะโหลด scripts
   - Players อาจเห็น NPCs หายไปแล้วกลับมา
   - Mob spawns จะถูกรีเซ็ต

4. **Compile-time vs Runtime:**
   - การสลับ Pre-RE/RE mode **ต้อง recompile**
   - ไม่สามารถสลับ mode ด้วย @reloadscript ได้
   - Config paths ถูก hardcode ด้วย preprocessor directives

### 6.2 Best Practices

**1. การใช้ @reloadscript:**
```
✓ DO:
  - ใช้เมื่อต้องการทดสอบ script changes
  - ใช้ในเวลาที่มีผู้เล่นน้อย
  - แจ้งเตือนผู้เล่นก่อน reload
  - Backup database ก่อนทำการแก้ไข production scripts

✗ DON'T:
  - ใช้ระหว่างที่มี WoE/BG events
  - ใช้ระหว่างที่ผู้เล่นอยู่ใน instance
  - ใช้บ่อยๆ ใน production server
  - ลืม test scripts ใน development environment ก่อน
```

**2. การจัดการ Configuration Files:**
```
✓ DO:
  - ใช้ import เพื่อแยก content ตามหมวดหมู่
  - Comment อธิบายจุดประสงค์ของแต่ละ import
  - จัด indentation ให้ชัดเจน
  - เก็บ custom scripts ใน scripts_custom.conf

✗ DON'T:
  - แก้ไข official scripts โดยตรง (ใช้ override แทน)
  - Import ไฟล์เดียวกันหลายครั้ง (จะถูกข้าม)
  - Circular imports (A imports B, B imports A)
```

**3. การเขียน Scripts ที่รองรับ Reload:**
```c
// ✓ GOOD: ใช้ permanent variables สำหรับข้อมูลสำคัญ
OnInit:
    // Load config from SQL หรือ permanent variables
    .EventActive = getd("$EventActive");
    end;

// ✗ BAD: ใช้ temp variables สำหรับข้อมูลสำคัญ
OnInit:
    .EventActive = 1;  // จะหายหลัง @reloadscript
    end;
```

```c
// ✓ GOOD: ใช้ OnInstanceInit สำหรับ instance scripts
OnInstanceInit:
    // Re-initialize instance data หลัง @reloadscript
    .instance_status = 1;
    monster "instance_map",100,100,"Boss",1002,1,"EventName::OnBossKilled";
    end;

// ดูตัวอย่างใน:
// - npc/instances/SealedShrine.txt:587
// - npc/instances/NydhoggsNest.txt:1670
```

**4. การ Debug:**
```
✓ DO:
  - ดู console output สำหรับ errors/warnings
  - ตรวจสอบจำนวน NPCs ที่โหลด (warp/shop/script/mob)
  - Test scripts ใน development environment ก่อน
  - ใช้ ShowDebug, ShowInfo, ShowWarning ใน scripts

✗ DON'T:
  - Ignore warnings (อาจเป็นสาเหตุของ bugs)
  - ลืม check syntax errors ก่อน reload
  - ใช้ @reloadscript บน production server ทันที
```

**5. การจัด Version Control:**
```
✓ DO:
  - Commit ทุกครั้งก่อนทดสอบ @reloadscript
  - ใช้ branches สำหรับ experimental scripts
  - เก็บ backup ของ working scripts
  - Document changes ใน commit messages

✗ DON'T:
  - แก้ไข scripts โดยไม่ commit
  - Test กับ production data โดยตรง
  - ลืม sync กับ team members
```

---

## 7. ตัวอย่างการใช้งาน (Use Cases)

### 7.1 การเพิ่ม Custom NPC

**ขั้นตอน:**

1. สร้างไฟล์ script: `npc/custom/my_npc.txt`
```c
prontera,150,150,4	script	Custom NPC	100	{
    mes "[Custom NPC]";
    mes "Hello, I'm a custom NPC!";
    close;
}
```

2. เพิ่มเข้า config: `npc/scripts_custom.conf`
```
npc: npc/custom/my_npc.txt
```

3. Reload scripts:
```
@reloadscript
```

### 7.2 การ Disable NPC ชั่วคราว

**วิธีที่ 1: Comment ใน config file**
```
// npc: npc/cities/prontera.txt   // ปิด comment เพื่อ disable
```

**วิธีที่ 2: ใช้ delnpc directive**
```
delnpc: npc/cities/prontera.txt
```

จากนั้น:
```
@reloadscript
```

### 7.3 การสลับ Pre-RE และ RE

**ขั้นตอน:**

1. แก้ไข `src/config/renewal.hpp`:
```cpp
// สำหรับ Renewal:
#define RENEWAL

// สำหรับ Pre-Renewal:
// #define RENEWAL
```

2. Recompile:
```bash
./configure
make clean
make server
```

3. Restart server (recompile จะไม่มีผลจนกว่าจะ restart)

**หมายเหตุ:** ไม่สามารถสลับด้วย @reloadscript ได้

### 7.4 การ Override Official Scripts

แทนที่จะแก้ไข official scripts โดยตรง ให้ใช้วิธีนี้:

1. Duplicate NPC ใน custom file: `npc/custom/my_overrides.txt`
```c
// Override official healer NPC
prontera,150,186,5	duplicate(Healer)	Custom Healer	4_F_SISTER#2
```

2. หรือ disable official NPC และสร้างใหม่:

ใน `npc/scripts_custom.conf`:
```
// Disable official healer
delnpc: npc/cities/prontera.txt

// Add custom version
npc: npc/custom/my_healer.txt
```

3. Reload:
```
@reloadscript
```

### 7.5 การจัดการ Mode-Specific Content

**ตัวอย่าง:** เพิ่ม custom monsters สำหรับทั้ง 2 modes

**Common monsters** (`npc/scripts_monsters.conf`):
```
// โหลดก่อน mode-specific
import: npc/custom/monsters_common.conf
```

**Pre-RE specific** (`npc/pre-re/scripts_monsters.conf`):
```
// โหลดหลัง common
import: npc/custom/monsters_prere.conf
```

**RE specific** (`npc/re/scripts_monsters.conf`):
```
// โหลดหลัง common
import: npc/custom/monsters_renewal.conf
```

**หมายเหตุ:** ไฟล์ที่โหลดทีหลังจะ override ไฟล์ที่โหลดก่อนหน้า (ถ้ามี duplicate NPCs)

---

## 8. Performance Considerations

### 8.1 ผลกระทบต่อประสิทธิภาพ

**ในขณะ Reload:**
- **CPU Usage:** เพิ่มขึ้นสูงชั่วคราว (parsing scripts)
- **Memory:** Spike ตอน allocate NPCs/scripts ใหม่
- **Network:** Lag ชั่วคราวจากการ flush buffers
- **Database:** Queries สำหรับ market data, mapreg

**หลัง Reload:**
- ประสิทธิภาพควรกลับเป็นปกติ
- ถ้าช้ากว่าเดิม อาจมี script bugs (infinite loops, memory leaks)

### 8.2 Optimization Tips

**1. ลด Scripts ที่ไม่จำเป็น:**
```
// Comment scripts ที่ไม่ใช้
// import: npc/events/valentines_day.conf
```

**2. ใช้ Lazy Loading สำหรับ Large Data:**
```c
// แทนที่จะโหลดทุกอย่างใน OnInit:
OnInit:
    // โหลดแค่ config
    .max_items = 100;
    end;

// โหลด data เมื่อต้องใช้:
OnTalkNPC:
    if (.data_loaded == 0) {
        // Load data from SQL
        .data_loaded = 1;
    }
    // Process...
    end;
```

**3. Cache Frequently Used Data:**
```c
OnInit:
    // Cache ข้อมูลที่ใช้บ่อย
    setarray .maps$[0],"prontera","geffen","payon";
    .map_count = getarraysize(.maps$);
    end;
```

**4. Optimize Monster Spawns:**
```c
// แทนที่จะใช้ หลาย single spawns:
prontera,0,0,0,0	monster	Poring	1002	1,0,0
prontera,0,0,0,0	monster	Poring	1002	1,0,0
prontera,0,0,0,0	monster	Poring	1002	1,0,0

// ใช้ batch spawn:
prontera,0,0,0,0	monster	Poring	1002	3,0,0
```

### 8.3 Monitoring

**ดู Console Output:**
```
Loading NPCs...
Done loading '4523' NPCs:
    -'1234' Warps
    -'234' Shops
    -'2345' Scripts
    -'123' Spawn sets
    -'567' Mobs Cached
    -'0' Mobs Not Cached
```

**ตัวชี้วัดสำคัญ:**
- **Total NPCs:** ควรใกล้เคียงกับครั้งก่อน (ถ้าไม่เปลี่ยน scripts)
- **Mobs Not Cached:** ควรเป็น 0 (ถ้าไม่เป็น 0 อาจมี delay spawns)
- **Load Time:** ไม่ควรนานเกิน 10-30 วินาที (ขึ้นกับจำนวน scripts)

**Warnings ที่ควรสังเกต:**
```
ShowWarning("npc_reload: %d npcs weren't removed at map %s!\n", ...);
ShowError("npc_parsesrcfile: Can't find source file \"%s\"\n", ...);
ShowError("npc_parsesrcfile: Unknown syntax in file '%s', line '%d'...\n", ...);
```

---

## 9. Error Handling และ Troubleshooting

### 9.1 Common Errors

**1. "Can't find source file"**
```
ERROR: npc_parsesrcfile: Can't find source file "npc/custom/my_npc.txt"
```

**สาเหตุ:**
- ไฟล์ไม่มีอยู่จริง
- Path ผิด (ใช้ relative path จาก map-server binary)
- File permissions

**วิธีแก้:**
```bash
# ตรวจสอบว่าไฟล์มีอยู่
ls -la npc/custom/my_npc.txt

# ตรวจสอบ permissions
chmod 644 npc/custom/my_npc.txt
```

**2. "Unknown syntax"**
```
ERROR: npc_parsesrcfile: Unknown syntax in file 'npc/custom/my_npc.txt', line '5'.
```

**สาเหตุ:**
- Missing หรือ extra TAB characters
- Invalid NPC format
- Special characters ใน strings

**วิธีแก้:**
- ตรวจสอบ TAB characters (ต้องเป็น TAB ไม่ใช่ spaces)
- ตรวจสอบรูปแบบ NPC:
  ```
  <map>,<x>,<y>,<dir>	<type>	<name>	<id>	{<script>}
  ```

**3. "Unknown coordinates"**
```
ERROR: npc_parsesrcfile: Unknown coordinates ('500', '500') for map 'prontera'...
```

**สาเหตุ:**
- พิกัดเกิน map size
- Map ไม่มีอยู่

**วิธีแก้:**
- ตรวจสอบขนาด map ใน map cache
- ใช้พิกัดที่อยู่ใน map boundaries

**4. "npcs weren't removed"**
```
WARNING: npc_reload: 5 npcs weren't removed at map prontera!
```

**สาเหตุ:**
- NPCs ที่สร้างแบบ dynamic (ผ่าน script)
- Duplicate NPCs

**วิธีแก้:**
- ปกติแล้ว NPCs เหล่านี้จะถูกลบใน loop ถัดไป
- ถ้ายังมีปัญหา ให้ unload ด้วย `unloadnpc` script command

**5. "Detected unsupported UTF-8 BOM"**
```
ERROR: npc_parsesrcfile: Detected unsupported UTF-8 BOM in file 'npc/custom/my_npc.txt'...
```

**สาเหตุ:**
- ไฟล์มี UTF-8 BOM (Byte Order Mark)
- มักเกิดจากการบันทึกด้วย Windows Notepad

**วิธีแก้:**
```bash
# ลบ BOM ด้วย command line:
sed -i '1s/^\xEF\xBB\xBF//' npc/custom/my_npc.txt

# หรือใช้ text editor ที่ support UTF-8 without BOM:
# - Notepad++: Encoding → UTF-8 without BOM
# - VS Code: File → Preferences → Settings → "files.encoding" → "utf8"
```

### 9.2 Debugging Tips

**1. Enable Debug Output:**

ใน `conf/battle/misc.conf`:
```
// แสดง debug messages
console_msg_log: 1

// แสดง detailed loading info
// ใน src/map/npc.cpp, uncomment:
// #define DETAILED_LOADING_OUTPUT
```

**2. Test Scripts Individually:**
```c
// สร้าง test config
// npc/test/test_config.conf
npc: npc/test/my_test_script.txt
```

แล้วโหลดโดยตรง:
```bash
# แก้ไข npc/[pre-]re/scripts_main.conf
import: npc/test/test_config.conf

# Reload
@reloadscript
```

**3. Use ShowDebug in Scripts:**
```c
OnInit:
    debugmes "NPC initialized!";
    .test = 123;
    debugmes "Value: " + .test;
    end;
```

**4. Check Event Database:**
```c
// สร้าง debug NPC
-	script	DebugEvents	-1,{
OnInit:
    // ตรวจสอบว่า events ถูกลงทะเบียนหรือไม่
    if (checkquest(60000) == -1) {
        debugmes "Quest event not found!";
    }
    end;
}
```

**5. Monitor Memory:**
```bash
# บน Linux:
ps aux | grep map-server
watch -n 1 'ps aux | grep map-server | grep -v grep'

# ดู memory usage (RSS column)
```

### 9.3 Recovery from Failed Reload

**ถ้า @reloadscript ทำให้เกิดปัญหา:**

**1. Rollback Changes:**
```bash
# Revert file changes
git checkout npc/custom/my_npc.txt

# หรือ restore จาก backup
cp backup/scripts_custom.conf npc/scripts_custom.conf
```

**2. Reload Again:**
```
@reloadscript
```

**3. ถ้ายัง crash:**
- Restart map-server
- ตรวจสอบ console logs
- Fix scripts errors
- Restart อีกครั้ง

**4. Emergency Disable:**
```
// ใน npc/scripts_custom.conf, comment ไฟล์ที่มีปัญหา:
// import: npc/custom/problematic_scripts.conf
```

**5. Restore from Backup:**
```bash
# Full restore
./map-server --restore-backup 2024-01-15_10-00-00
```

---

## 10. สรุป (Summary)

### 10.1 Key Takeaways

1. **@reloadscript ทำงานแบบครบวงจร:**
   - ปิดการโต้ตอบ NPC ทั้งหมด
   - โหลด config files ใหม่ตาม mode
   - ล้าง script engine
   - ลบและสร้าง NPCs ใหม่ทั้งหมด
   - Rebuild event caches
   - ทริกเกอร์ OnInit events

2. **Import Mechanism เป็น Recursive:**
   - ทำงานที่ระดับ configuration files
   - รองรับ nested imports
   - ไฟล์สามารถถูก import ได้แค่ครั้งเดียว (duplicate detection)

3. **Mode Switching เป็น Compile-time:**
   - กำหนดโดย `#define RENEWAL` ใน `src/config/renewal.hpp`
   - ต้อง recompile server เพื่อสลับ mode
   - Config files แยกตาม mode: `npc/pre-re/` vs `npc/re/`

4. **Performance Impact:**
   - Server จะ lag ชั่วคราวระหว่าง reload
   - ผู้เล่นจะถูก disconnect จาก NPCs
   - Script states และ timers จะถูกรีเซ็ต

5. **Best Practices:**
   - ใช้ import เพื่อจัด modularity
   - เก็บ custom scripts แยกต่างหาก
   - Test ใน development ก่อน production
   - Backup ก่อนทำการ reload

### 10.2 Flow Diagram ภาพรวม

```
┌─────────────────────────────────────────────────────────────────┐
│                      @reloadscript Command                       │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
                 ┌─────────────────────┐
                 │   Preparation Phase  │
                 │   - Close NPC dialogs│
                 │   - Clear BG queues  │
                 │   - Flush FIFOs      │
                 └──────────┬───────────┘
                           │
                           ▼
                 ┌─────────────────────┐
                 │ Config Reload Phase  │
                 │   map_reloadnpc()    │
                 │   - Clear file list  │
                 │   - Load main config │
                 │     based on mode    │
                 └──────────┬───────────┘
                           │
                           ▼
                 ┌─────────────────────┐
                 │Config Parsing Phase  │
                 │ map_reloadnpc_sub()  │
                 │   - Parse directives │
                 │   - Recursive import │
                 │   - Build file list  │
                 └──────────┬───────────┘
                           │
                           ▼
                 ┌─────────────────────┐
                 │Script Cleanup Phase  │
                 │   script_reload()    │
                 │   - Clear functions  │
                 │   - Clear states     │
                 │   - Reload mapreg    │
                 └──────────┬───────────┘
                           │
                           ▼
                 ┌─────────────────────┐
                 │  NPC Reload Phase    │
                 │    npc_reload()      │
                 │   - Clear databases  │
                 │   - Unload NPCs/MOBs │
                 │   - Load src files   │
                 │   - Rebuild caches   │
                 │   - Trigger OnInit   │
                 └──────────┬───────────┘
                           │
                           ▼
                 ┌─────────────────────┐
                 │   File Load Phase    │
                 │  npc_loadsrcfiles()  │
                 │   - Loop all files   │
                 │   - Parse each file  │
                 │   - Create NPCs      │
                 └──────────┬───────────┘
                           │
                           ▼
                 ┌─────────────────────┐
                 │    NPC Parse Phase   │
                 │  npc_parsesrcfile()  │
                 │   - Parse syntax     │
                 │   - Create entities  │
                 │   - Register events  │
                 └──────────┬───────────┘
                           │
                           ▼
                 ┌─────────────────────┐
                 │   Finalization       │
                 │   - Rebuild events   │
                 │   - OnInit scripts   │
                 │   - Ready to serve   │
                 └─────────────────────┘
```

### 10.3 Mode Selection Diagram

```
                    ┌──────────────────────┐
                    │   Compile Time       │
                    │   renewal.hpp        │
                    └──────────┬───────────┘
                               │
                    ┌──────────┴───────────┐
                    │                      │
       ┌────────────▼─────────┐ ┌─────────▼──────────┐
       │   #define RENEWAL    │ │ //#define RENEWAL  │
       │   (Renewal Mode)     │ │ (Pre-Renewal Mode) │
       └────────────┬─────────┘ └─────────┬──────────┘
                    │                      │
                    │                      │
          ┌─────────▼─────────┐  ┌────────▼─────────┐
          │   Runtime          │  │   Runtime        │
          │   map_reloadnpc()  │  │   map_reloadnpc()│
          └─────────┬─────────┘  └────────┬─────────┘
                    │                      │
      ┌─────────────▼─────────┐  ┌────────▼──────────┐
      │ npc/re/scripts_main   │  │npc/pre-re/scripts │
      │        .conf          │  │    _main.conf     │
      └─────────┬─────────────┘  └────────┬──────────┘
                │                          │
                ▼                          ▼
      ┌─────────────────┐        ┌─────────────────┐
      │ Import Common:  │        │ Import Common:  │
      │ - athena        │        │ - athena        │
      │ - guild         │        │ - guild         │
      │ - jobs          │        │ - jobs          │
      │ - monsters      │        │ - monsters      │
      │ - warps         │        │ - warps         │
      └─────────┬───────┘        └─────────┬───────┘
                │                          │
                ▼                          ▼
      ┌─────────────────┐        ┌─────────────────┐
      │ Import RE:      │        │ Import Pre-RE:  │
      │ - re/athena     │        │ - pre-re/athena │
      │ - re/guild      │        │ - pre-re/jobs   │
      │ - re/jobs       │        │ - pre-re/monsters│
      │ - re/monsters   │        │ - pre-re/warps  │
      │ - re/warps      │        │                 │
      └─────────┬───────┘        └─────────┬───────┘
                │                          │
                ▼                          ▼
      ┌─────────────────┐        ┌─────────────────┐
      │   Renewal       │        │  Pre-Renewal    │
      │   Content       │        │  Content        │
      │ - 3rd Classes   │        │ - 2nd/Trans     │
      │ - New Maps      │        │ - Classic Maps  │
      │ - RE Formulas   │        │ - Pre-RE Formula│
      └─────────────────┘        └─────────────────┘
```

### 10.4 Configuration Import Tree (ตัวอย่าง Renewal Mode)

```
npc/re/scripts_main.conf
    ├── npc: npc/other/Global_Functions.txt
    ├── npc: npc/other/CashShop_Functions.txt
    │
    ├── import: npc/scripts_athena.conf
    │   ├── import: npc/airports/airports.txt
    │   ├── import: npc/cities/...
    │   └── import: npc/merchants/...
    │
    ├── import: npc/scripts_guild.conf
    │   └── import: npc/guild/...
    │
    ├── import: npc/scripts_jobs.conf
    │   ├── import: npc/jobs/1-1/...
    │   └── import: npc/jobs/2-1/...
    │
    ├── import: npc/scripts_mapflags.conf
    │   └── import: npc/mapflag/...
    │
    ├── import: npc/scripts_monsters.conf
    │   ├── npc: npc/mobs/jail.txt
    │   ├── npc: npc/mobs/pvp.txt
    │   └── npc: npc/mobs/towns.txt
    │
    ├── import: npc/scripts_warps.conf
    │   └── import: npc/warps/...
    │
    ├── import: npc/re/scripts_athena.conf     [RE-specific]
    │   └── import: npc/re/cities/...
    │
    ├── import: npc/re/scripts_guild.conf      [RE-specific]
    │   └── import: npc/guild3/...
    │
    ├── import: npc/re/scripts_jobs.conf       [RE-specific]
    │   ├── import: npc/re/jobs/3-1/...
    │   └── import: npc/re/jobs/novice/...
    │
    ├── import: npc/re/scripts_mapflags.conf   [RE-specific]
    │   └── import: npc/re/mapflag/...
    │
    ├── import: npc/re/scripts_monsters.conf   [RE-specific]
    │   ├── import: npc/re/mobs/fields/...
    │   └── import: npc/re/mobs/dungeons/...
    │
    ├── import: npc/re/scripts_warps.conf      [RE-specific]
    │   └── import: npc/re/warps/...
    │
    └── import: npc/scripts_custom.conf
        └── [Custom user scripts]
```

---

## 11. อ้างอิง (References)

### 11.1 Source Files

| ไฟล์ | URL (GitHub) | จุดประสงค์ |
|------|-------------|-----------|
| `src/map/atcommand.cpp` | `/src/map/atcommand.cpp` | AT commands implementation |
| `src/map/map.cpp` | `/src/map/map.cpp` | Map server main และ config loading |
| `src/map/npc.cpp` | `/src/map/npc.cpp` | NPC management และ script parsing |
| `src/map/script.cpp` | `/src/map/script.cpp` | Script engine |
| `src/config/renewal.hpp` | `/src/config/renewal.hpp` | Mode configuration |

### 11.2 Configuration Files

| ไฟล์ | จุดประสงค์ |
|------|-----------|
| `npc/re/scripts_main.conf` | Renewal mode main config |
| `npc/pre-re/scripts_main.conf` | Pre-Renewal mode main config |
| `npc/scripts_athena.conf` | Common Athena scripts |
| `npc/scripts_monsters.conf` | Common monster spawns |
| `npc/scripts_custom.conf` | Custom user scripts |

### 11.3 Documentation

| เอกสาร | เนื้อหา |
|--------|---------|
| `doc/atcommands.txt` | AT commands documentation |
| `doc/script_commands.txt` | Script commands documentation |
| `README.md` | Project overview |

### 11.4 Related Topics

- **NPC Scripting:** `doc/script_commands.txt`
- **AT Commands:** `doc/atcommands.txt`
- **Map Flags:** `doc/mapflags.txt`
- **Item Database:** `db/item_db.yml`
- **Mob Database:** `db/mob_db.yml`

---

## 12. บันทึกการเปลี่ยนแปลง (Change Log)

| เวอร์ชัน | วันที่ | การเปลี่ยนแปลง |
|---------|-------|---------------|
| 1.0 | 2024-01-15 | เอกสารฉบับแรก - วิเคราะห์ครบถ้วนทุกส่วน |

---

## ภาคผนวก (Appendix)

### A. Glossary (อภิธานศัพท์)

| คำศัพท์ | ความหมาย |
|---------|----------|
| **AT Command** | Admin/Test Command - คำสั่งพิเศษสำหรับ GM/Admin |
| **NPC** | Non-Player Character - ตัวละครที่ไม่ใช่ผู้เล่น |
| **Script** | ชุดคำสั่งที่กำหนดพฤติกรรมของ NPC |
| **Reload** | การโหลดข้อมูลใหม่ทั้งหมด |
| **Import** | การนำเข้า configuration file อื่น |
| **Directive** | คำสั่งใน configuration file (npc:, import:, delnpc:) |
| **Pre-Renewal (Pre-RE)** | โหมดเกมแบบดั้งเดิม |
| **Renewal (RE)** | โหมดเกมแบบใหม่ที่ปรับสมดุลใหม่ |
| **Compile Time** | เวลาที่โปรแกรมถูก compile |
| **Runtime** | เวลาที่โปรแกรมกำลังทำงาน |
| **Recursive** | การเรียกตัวเองซ้ำ (function เรียก function ตัวเอง) |
| **Database** | ฐานข้อมูลหรือโครงสร้างเก็บข้อมูล |
| **Cache** | พื้นที่เก็บข้อมูลชั่วคราวเพื่อเร่งความเร็ว |

### B. Acronyms (ตัวย่อ)

| ตัวย่อ | ความหมายเต็ม |
|--------|-------------|
| **NPC** | Non-Player Character |
| **GM** | Game Master |
| **RE** | Renewal |
| **WoE** | War of Emperium |
| **BG** | Battleground |
| **DB** | Database |
| **SQL** | Structured Query Language |
| **API** | Application Programming Interface |
| **ID** | Identifier |
| **MOB** | Monster/Mobile Object |

### C. File Extensions

| นามสกุล | ประเภท | คำอธิบาย |
|---------|--------|----------|
| `.conf` | Configuration | ไฟล์ตั้งค่า (config files) |
| `.txt` | Script | ไฟล์ NPC scripts |
| `.cpp` | C++ Source | ไฟล์ source code C++ |
| `.hpp` | C++ Header | ไฟล์ header C++ |
| `.yml` | YAML Data | ไฟล์ข้อมูล database ใหม่ |

---

**สิ้นสุดรายงาน**

**ผู้จัดทำ:** rAthena Development Analysis Team  
**วันที่:** 2024-01-15  
**Version:** 1.0  
**Contact:** [GitHub Issues](https://github.com/rathena/rathena/issues)
