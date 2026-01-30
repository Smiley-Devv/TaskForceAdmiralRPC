# Custom Engine Reverse Engineering Guide



## 🔍 Why This Matters

Since it's a custom engine:
- ❌ No public documentation or tutorials
- ❌ No existing modding tools
- ❌ Memory layouts are unique to this game
- ✅ BUT: Once you find offsets, they're very stable!
- ✅ Simpler than Unity/Unreal (less abstraction layers)

## 🛠️ Tools You'll Need

### 1. Cheat Engine (Essential)
- Download: https://www.cheatengine.org/
- Used for: Finding memory addresses

### 2. x64dbg or IDA Pro (Advanced)
- For: Reverse engineering DLLs
- Finding function signatures

### 3. Process Explorer (Microsoft)
- For: Viewing loaded DLLs and modules
- Download: https://docs.microsoft.com/sysinternals/

### 4. HxD Hex Editor
- For: Examining memory dumps
- Download: https://mxhc.de/en/hxd/

## 📋 Step-by-Step Reverse Engineering

### Phase 1: Map the DLLs

1. **Launch the Memory Tools from the app**
2. **Click "Attach to Game"**
3. **Click "List All DLLs"**

Look for DLLs like:
- `TaskForceAdmiral.exe` (main executable)
- `imgui.dll` or similar (UI library)
- `engine.dll` / `core.dll` (game engine)
- Any custom-named DLLs

**Key Finding**: The DLLs marked with ⭐ are likely to contain game data!

### Phase 2: Find Scenario Name (Easy Start)

This is the EASIEST value to find and a great starting point!

1. **Start a scenario** - note the name (e.g., "Pearl Harbor")
2. **In Memory Tools**: Enter "Pearl Harbor" in search box
3. **Click "Scan for Strings"**
4. **Results will show all addresses** containing that string

Example output:
```
Address: 0x12345678
Offset from base: 0x345678  ← THIS IS YOUR OFFSET!
String: Pearl Harbor - December 7th, 1941
```

**Write down this offset!** This is `ScenarioNamePointer`

### Phase 3: Find Aircraft Counts (Moderate)

Since the engine is C++, aircraft counts are likely stored as:
- `int` (4 bytes, whole numbers)
- In a struct/class near each other

**Method A: Cheat Engine**
1. Note friendly aircraft count (e.g., 12)
2. Scan for value `12` (int, 4 bytes)
3. Launch more planes
4. Now it's 15
5. Next scan for `15`
6. Repeat until 1-5 addresses remain

**Method B: Structure Analysis**
1. Once you find scenario name address
2. In Memory Tools, paste that address
3. Click "Analyze Structure"
4. Look for integers in reasonable range (0-100)
5. Test by changing game state

### Phase 4: Understanding C++ Data Structures

Custom C++ engines typically use structures like:

```cpp
struct ScenarioData {
    char* scenarioName;      // +0x00 (pointer to string)
    int friendlyAircraft;    // +0x08
    int enemyAircraft;       // +0x0C
    int attackingGroups;     // +0x10
    float gameTime;          // +0x14
    // ... more fields
};
```

When you find one field (like scenario name), nearby offsets often contain related data!

### Phase 5: ImGui Detection

If the game uses ImGui (which it likely does):

1. **Look for ImGui strings** in DLLs
2. **Search for** "Dear ImGui" or "imgui" in memory
3. **ImGui stores UI state** - can be useful for finding:
   - Window titles
   - Button states
   - Current menu/phase

**In Memory Tools**: 
- Search for strings you see in the UI
- Map panels → back to game state

### Phase 6: Finding Nation Names & Target Types

These are **C++ std::string** or **char arrays**:

**Method 1: String Search**
- When under attack by Japan, search for "Japan"
- When Japan attacks, search for target like "Carrier"
- Map these addresses

**Method 2: Enum Analysis**
Nations might be stored as enums:
```cpp
enum Nation {
    USA = 0,
    Japan = 1,
    Britain = 2
};
```

So you might find `1` (int) instead of "Japan" (string)!

## 🎮 Practical Example Workflow

### Finding "Japan is striking carrier" data:

1. **Setup**: 
   - Start a scenario where Japan attacks
   - Note: "Under attack by Japan"
   - Note: "Target is a Carrier"

2. **Find Japan**:
   ```
   Memory Tools → Search "Japan"
   Find addresses: 0x1234000, 0x1234100, etc.
   ```

3. **Find Carrier**:
   ```
   Memory Tools → Search "Carrier"
   Find addresses: 0x1234200, 0x1234300, etc.
   ```

4. **Analyze Relationships**:
   ```
   If Japan is at 0x1234100
   And Carrier is at 0x1234200
   Difference: 0x100 (256 bytes)
   
   These might be in the same struct!
   ```

5. **Test & Verify**:
   - Start different scenario
   - Check if USA appears at similar offset
   - Check if Battleship appears at similar offset
   - If yes → You found the attack data structure!

## 📊 Custom Engine Memory Patterns

### Typical Memory Layout:
```
Base Address (e.g., 0x10000000)
│
├── Static Data Section
│   ├── Scenario Info       (+0x100000)
│   ├── Ship Lists          (+0x200000)
│   └── Aircraft Lists      (+0x300000)
│
├── Dynamic Game State
│   ├── Current Battle      (+0x1000000)
│   ├── Attack State        (+0x1100000)
│   └── UI State            (+0x1200000)
│
└── ImGui Context           (+0x2000000)
```

### How to Find Sections:

1. **Static Data**: Doesn't change between saves
2. **Dynamic Data**: Changes constantly during game
3. **ImGui**: Look for GUI element strings

## 🔧 Advanced: DLL Code Analysis

### If you want to find code (not just data):

1. **Use x64dbg** or **IDA Pro**
2. **Load TaskForceAdmiral.exe**
3. **Search for string references**:
   ```
   Right-click → Search for → All referenced text strings
   Search for: "Scenario", "Japan", "Carrier"
   ```

4. **Find the function that uses these strings**
5. **Set breakpoint** when that code executes
6. **Inspect registers/memory** when breakpoint hits

### Example Finding Nation String:
```assembly
; Code that sets nation name
mov rax, [rbp+attackerNation]  ; Load nation pointer
call std::string::assign        ; Assign "Japan"
```

The register `rbp+attackerNation` tells you the offset!

## 💾 Saving Your Work

### Create an Offset Map File:

```json
{
  "GameVersion": "1.0.0.0",
  "BaseModule": "TaskForceAdmiral.exe",
  
  "StaticOffsets": {
    "ScenarioNamePointer": 5457016,
    "FriendlyAircraftCount": 5457100,
    "EnemyAircraftCount": 5457104
  },
  
  "DynamicPointerPaths": {
    "AttackingNation": {
      "base": "TaskForceAdmiral.exe",
      "offsets": [0x123000, 0x50, 0x10]
    }
  },
  
  "Notes": {
    "ScenarioNamePointer": "Found via string search for Pearl Harbor",
    "FriendlyAircraftCount": "4 bytes after scenario name struct"
  }
}
```

## 🚨 Common Pitfalls

### 1. "All my offsets broke after game update!"
- **Solution**: Save your finding process, not just offsets
- Use signature scanning instead of fixed offsets
- Document the context around each offset

### 2. "Value is always 0 or garbage"
- **Cause**: Wrong data type or pointer chain
- **Solution**: Try reading as different types (int/float/pointer)

### 3. "Address changes every game launch"
- **Cause**: Dynamic allocation (ASLR)
- **Solution**: Use pointer paths, not absolute addresses

### 4. "Can't find the value I'm looking for"
- **Cause**: Might be calculated, not stored
- Example: "Total aircraft" might be `friendly + enemy`
- **Solution**: Find components, calculate result yourself

## 📚 Resources

### Learning Reverse Engineering:
- GuidedHacking Forums: https://guidedhacking.com/
- Cheat Engine Tutorial (built-in to CE)
- "Reverse Engineering for Beginners" (free book)

### C++ Memory Layouts:
- Learn about POD types, vtables, std::string
- Understanding how C++ objects are laid out in memory

### ImGui Specific:
- ImGui GitHub: https://github.com/ocornut/imgui
- Look at ImGuiContext structure definition

## 🎯 Your Mission

### Week 1: Foundation
- [ ] List all DLLs
- [ ] Find scenario name string
- [ ] Find aircraft counts
- [ ] Document your findings

### Week 2: Advanced
- [ ] Find nation names
- [ ] Find target types  
- [ ] Map attack state structure
- [ ] Test in different scenarios

### Week 3: Integration
- [ ] Update memory_offsets.json
- [ ] Test RPC narrative messages
- [ ] Fine-tune and enjoy!

Good luck, and remember: **patience and documentation** are your best friends in reverse engineering!

---

**Pro Tip**: Join the Task Force Admiral Discord or Steam forums. Other players might share findings or work together on modding tools!
