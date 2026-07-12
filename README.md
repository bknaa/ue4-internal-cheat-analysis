# ue4-internal-cheat-analysis
Static malware analysis and architectural breakdown of an obfuscated internal UE4 cheat DLL vs. Riot Vanguard mitigation mechanics.

# Technical Analysis Report: Unreal Engine 4 Internal Cheat Artifacts vs. Riot Vanguard

## 1. Executive Summary
This report covers the static malware analysis of an obfuscated internal Dynamic Link Library (.dll) designed to compromise tactical shooter games utilizing Unreal Engine 4. The binary attempts memory injection, User-mode API hooking via `ProcessEvent`, and implements various anti-detection mechanisms. 

---

## 2. File Identification & Metadata
* **File Type:** PE32+ x86-64 DLL (Windows GUI subsystem)
* **Sections (7):** `.text`, `.rdata`, `.data`, `.pdata`, `.fptable`, `.rsrc`, `.reloc`
* **Internal DLL Name:** `JJGBFRV368BY238RB8R3HFSNS.dll` (Randomized naming convention typical of obfuscation)
* **Compile Timestamp:** June 30, 2026

### Developer Environment Leak (PDB Path)
The compiler failed to strip debug symbols, exposing the absolute Program Database (PDB) path of the developer:
`c:\Users\yigit\Desktop\dvn\project\x64\Release\JJGBFRV368BY238RB8R3HFSNS.pdb`

---

## 3. Import Address Table (IAT) Analysis
The binary imports functions from 5 core Windows subsystems to interact with system memory and input devices:

### A. Memory Manipulation & Anti-Debugging (`KERNEL32.dll`)
* `VirtualAlloc` & `VirtualProtect`: Used to allocate memory and modify page protections (e.g., changing page access to Read/Write/Execute) to facilitate code injection.
* `CreateThread`: Used to spin up an independent execution thread inside the game process.
* `IsDebuggerPresent`: Standard anti-debugging check to detect if the binary is running within a analysis environment (like x64dbg).

### B. Input Interception (`USER32.dll`)
* `GetAsyncKeyState` & `GetKeyboardState`: Utilized to monitor keystrokes asynchronously, typically map custom hotkeys for the cheat overlay or triggering the aimbot.
* `GetCursorPos` & `ScreenToClient`: Used to map cursor coordinates relative to the game window viewport for Aimbot calculations.

---

## 4. String Extraction & Unreal Engine Indicators
Static string analysis revealed explicit internal components targeting Unreal Engine 4 infrastructure:

### A. Engine Hooking
* `[+] ProcessEvent hook initialized`
* **Mechanism:** Overriding the core `ProcessEvent` function in UE4 allows the binary to intercept every function call, entity update, and rendering routine executed by the game engine.

### B. Game-Specific Artifacts (Blueprint Classes)
The binary targets specific Valorant Blueprint classes and agents, explicitly hardcoded in the string table:
* **Character Controller Inferences:** `Phoenix_PC_C`, `Killjoy_PC_C`, `Breach_PC_C`, `Sarge_PC_C`, `Wraith_PC_C`
* **Weapon & Prop Subclasses:** `AssaultRifle_AK_C`, `SawedOffShotgun_C`, `LeverSniperRifle_C`, `Pawn_TrainingBot_Defuse_Ultimate_C`

---

## 5. Mitigation & Defensive Context (Riot Vanguard)
While the binary contains strings referencing features like `Fake lag`, `Jitter range`, `Resolver Intensity`, and `Desync range` to bypass basic server-side heuristics, its user-mode injection techniques are heavily mitigated by Riot Vanguard:

1. **Handle Stripping:** Vanguard's Ring 0 driver runs at boot-time and strips process handle rights. User-mode processes cannot easily obtain handles with `PROCESS_VM_WRITE` or `PROCESS_VM_OPERATION` access.
2. **Memory Page Monitoring:** Modifications via `VirtualProtect` on critical memory pages are dynamically monitored at the kernel level, triggering immediate integrity violation flags.
