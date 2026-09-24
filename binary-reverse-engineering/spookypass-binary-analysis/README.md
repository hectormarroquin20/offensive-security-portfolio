# Security Assessment & Binary Analysis: Spooky Pass

## 📌 Executive Summary
This technical assessment documents the reverse engineering workflow performed on a protected compiled binary. The evaluation focuses on static symbol analysis, memory inspection via dynamic debugging, and control-flow evaluation to understand how local authentication checks are enforced and how sensitive reference data resides within executable memory segments.

---

## 🛠️ Methodology & Tooling
* **Archive Extraction & Cryptographic Auditing:** Utilizing archive hash isolation tools (`zip2john`) and dictionary-based recovery utilities (`john`) to obtain cleartext archive access credentials.
* **Static Binary Inspection:** Employing file-type profiling (`file`) and symbol table enumeration (`nm`) to review unstripped function structures.
* **Dynamic Debugging & Memory Inspection:** Using the GNU Debugger (`gdb`) to set breakpoints, inspect runtime registers, analyze disassembly blocks, and evaluate string comparison logic.

---

## 🔍 Technical Analysis & Workflow

### Phase 1: Archive Retrieval & Password Recovery
Protected compressed archives often employ standard or legacy encryption formats. The initial phase involves isolating the compressed structure, extracting its cryptographic hash representation, and performing an offline dictionary recovery process to secure the cleartext phrase required for extraction.

### Phase 2: Static Reconnaissance
Once the target binary is extracted from its container, static analysis determines compilation properties and target architecture:
* **Binary Profiling:** Verifying whether the executable is stripped or contains debugging symbols.
* **Symbol Enumeration:** Inspecting exported function names and symbol tables to map out the application's internal control flow before execution.

### Phase 3: Dynamic Analysis & Control Flow Evaluation
To examine how authentication logic handles runtime user input:
1. **Debugger Initialization:** Launching the binary within a controlled debugging context.
2. **Breakpoint Placement:** Setting execution breakpoints at core initialization routines (such as `main`) and directly preceding validation or comparison functions (such as `strcmp`).
3. **Memory Inspection:** Intercepting program execution during input validation to inspect how reference strings or memory addresses store sensitive comparison tokens.
4. **Control Flow Navigation:** Navigating execution checkpoints to verify how the application responds to successful validation paths.

---

### 💻 Quick Reference: Technical Commands & Workflow

```bash
# 1. Extract hash from password-protected ZIP archive
zip2john <archive_name>.zip > archive_hash.txt

# 2. Recover password using John the Ripper with wordlists and rules
john --wordlist=/path/to/wordlist.txt --rules=best64 archive_hash.txt
john --show archive_hash.txt

# 3. Extract the archive contents
unzip <archive_name>.zip

# 4. Static Binary Inspection & Symbol Listing
file <binary_name>
nm <binary_name> | grep -v ' U '

# 5. Dynamic Debugging with GDB
gdb ./<binary_name>
(gdb) break main
(gdb) run
(gdb) disassemble main
(gdb) break *<memory_address_before_strcmp>
(gdb) continue
(gdb) p (char*)<memory_address_containing_secret>