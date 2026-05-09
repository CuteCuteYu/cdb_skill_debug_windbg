---
name: windbg-cdb-powershell
description: >
  CDB (Console Debugger) debugging via PowerShell on Windows. ⚠️ THIS SKILL USES ONLY CDB — 
  WinDbg GUI references are removed. USE THIS SKILL whenever the user wants to: debug a 
  Windows executable or crash dump; analyze .dmp files; use CDB (console debugger); set 
  breakpoints, disassemble, inspect memory/registers/call stacks; attach to a running 
  process for debugging; perform remote debugging sessions; or any general Windows debugging 
  workflow. This skill uses ONLY PowerShell to launch and interact with CDB — no MCP tools 
  involved. Trigger on keywords like: cdb, crash dump, .dmp, breakpoint, disassemble, debug 
  Windows, dump analysis, attach debugger, memory inspection, call stack. 中文触发词包括：动态调
  试、动态分析、调试器、调试Windows程序、转储分析、附加进程、断点调试、反汇编调试、cdb、
  cdbx64、cdbx86、cdbx。只要用户提到与动态调试（即运行时调试、交互式调试、实时调试）相关的
  任何需求，都应该使用本 skill。
---

# CDB Debugging via PowerShell

This skill documents how to use **CDB (Console Debugger)** entirely through PowerShell, without relying on MCP tools. All debugger interactions happen via PowerShell commands, Start-Process, and script files.

⚠️ **IMPORTANT**: This skill focuses exclusively on **CDB** (the command-line debugger). All examples use CDB paths explicitly.

## 🚫 NO REMOTE SYMBOL SERVERS

**This skill does NOT use Microsoft's public symbol server or any remote symbol servers.**

**Why:**
- Remote symbol servers require network access
- Symbol downloads can be slow and large (hundreds of MB)
- Not all environments have internet access
- Offline debugging is more reliable and faster

**What works without remote symbols:**
- Exported function names (e.g., `user32!MessageBoxA`, `kernel32!CreateFileA`)
- Your own module's symbols (if compiled with `-g` or `/DEBUG`)
- Raw addresses and offsets
- Module load addresses and sizes
- Full register states and call stacks
- Memory inspection and disassembly

**What doesn't work without remote symbols:**
- Internal function names in system DLLs (e.g., `ntdll!LdrpDoDebuggerBreak`)
- Line number information for system code
- Local variable names in system functions

**For most debugging tasks, exported symbols and addresses are sufficient.**

## ⚠️ CRITICAL: Encoding and Escaping Rules

**READ THIS BEFORE WRITING ANY CDB DEBUGGING CODE**

| Rule | Description | Example |
|------|-------------|---------|
| **1. ASCII Encoding** | CDB script files MUST use ASCII encoding | `Out-File -Encoding ASCII` |
| **2. English Only** | CDB commands/comments MUST be English only | `$$ Set breakpoint` ✅ <br> `$$ 设置断点` ❌ |
| **3. Here Strings** | Use `@'...'@` for CDB scripts to avoid escaping | See examples below |
| **4. File-Based** | Write script to file, don't pass complex commands inline | `\$<script.txt` |
| **5. No Nested Quotes** | Avoid complex nested quote escaping in PowerShell | Use here strings |

**Why these rules matter:**
- CDB does NOT support UTF-8/BOM encoding
- Chinese characters in CDB commands cause syntax errors
- PowerShell quote escaping with CDB's semicolon syntax is fragile

**Correct Pattern:**
```powershell
$script = @'
bp user32!MessageBoxA ".echo HIT; da poi(rdx) L1; g"
g
'@
$script | Out-File "debug.txt" -Encoding ASCII
& $cdb64 -c "`$<$pwd\debug.txt" $exe 2>$null
```

**Wrong Pattern (DO NOT DO THIS):**
```powershell
& $cdb64 -c "bp user32!MessageBoxA \".echo 命中; da poi(rdx) L1; g\"" $exe
```

---

## 1. CDB Paths (MANDATORY - Use These Exact Paths)

**CDB 64-bit (for debugging 64-bit applications):**
```powershell
$cdb64 = "C:\Users\j4543\AppData\Local\Microsoft\WindowsApps\cdbX64.exe"
```

**CDB 32-bit (for debugging 32-bit applications):**
```powershell
$cdb86 = "C:\Users\j4543\AppData\Local\Microsoft\WindowsApps\cdbX86.exe"
```

**CDB ARM64 (for debugging ARM64 applications):**
```powershell
$cdbArm = "C:\Users\j4543\AppData\Local\Microsoft\WindowsApps\cdbARM64.exe"
```

⚠️ **CRITICAL**: Always use the full path when invoking CDB. Do not rely on PATH environment variable.

**Path detection helper:**
```powershell
# Verify CDB installation
Test-Path "C:\Users\j4543\AppData\Local\Microsoft\WindowsApps\cdbX64.exe"
Test-Path "C:\Users\j4543\AppData\Local\Microsoft\WindowsApps\cdbX86.exe"
```

If CDB is not found, install WinDbg Preview:
```powershell
winget install "WinDbg"
```

---

## 2. Launching CDB via PowerShell

### 2.1 Basic CDB Usage

```powershell
$cdb64 = "C:\Users\j4543\AppData\Local\Microsoft\WindowsApps\cdbX64.exe"

# Load an EXE for debugging (starts suspended at initial breakpoint)
& $cdb64 "program.exe"

# Load an EXE and run commands on startup, then quit
& $cdb64 -c "x program!*; q" "program.exe"

# Load a crash dump for analysis
& $cdb64 -z "crash.dmp" -c "!analyze -v; q"

# Attach to a running process by PID
& $cdb64 -p 1234

# Attach to a running process by name
$pid = (Get-Process notepad).Id
& $cdb64 -p $pid

# Non-interactive mode: run commands, capture output, exit
& $cdb64 -c "command1; command2; q" "program.exe"
```

### 2.2 Critical CDB Flags

| Flag | Meaning |
|------|---------|
| `-c "cmds"` | Run commands on startup (semicolon-separated) |
| `-z file` | Open a crash dump (not an EXE) |
| `-p pid` | Attach to process by PID |
| `-pn name` | Attach to process by name |
| `-v` | Verbose mode |
| `-hd` | Don't suspend the process initially |
| `-o` | Debug child processes |

### 2.3 Suppress Output Noise

```powershell
# Redirect stderr to null to hide NatVis unloading messages
& $cdb64 -c "command; q" "program.exe" 2>$null
```

---

## 3. Symbol Handling (NO REMOTE SYMBOL SERVERS)

⚠️ **CRITICAL**: This skill does NOT use remote symbol servers. All debugging is done with local symbols only.

### 3.1 Symbol Path Configuration

**Default symbol path (local only):**
```powershell
# Use only local symbols - NO remote servers
.sympath C:\symbols
# Or use current directory
.sympath .
```

**WRONG - Do NOT use remote symbol servers:**
```powershell
# DO NOT DO THIS - requires network access and downloads symbols
.sympath SRV*c:\symbols*https://msdl.microsoft.com/download/symbols
```

### 3.2 When to Use .reload

```powershell
# Only reload if you have local PDB files
.reload

# Force reload (use sparingly)
.reload /f myapp.exe
```

**Note**: If symbols are not available, CDB will still work but will show raw addresses instead of symbol names.

### 3.3 CDB Script Files (Batch Debugging)

For complex debugging sessions, write CDB commands into a script file and execute it. This is the most reliable way to run multi-step debug sessions via PowerShell.

**Create a script file (LOCAL SYMBOLS ONLY):**
```powershell
$script = @"
$$ Load local symbols only - NO remote servers
.sympath .
.reload
$$ List available symbols
x myapp!*
$$ Set breakpoint on function
bp myapp!WinMain
g
dv
k
uf myapp!WinMain
q
"@

$script | Out-File -FilePath "debug.txt" -Encoding ascii
```

**Execute the script:**
```powershell
$cdb64 = "C:\Users\j4543\AppData\Local\Microsoft\WindowsApps\cdbX64.exe"
& $cdb64 -c "`$<`"$pwd\debug.txt`"" "program.exe" 2>$null
```

The `$<filename` syntax tells CDB to read and execute commands from a file.

**Common pitfalls:**
- Always use `-Encoding ascii` for script files (CDB doesn't handle BOM well)
- Use `2>$null` to suppress CDB's verbose unloading messages
- Use `q` as the last command to quit after execution
- Wrap paths in quotes when they contain spaces
- **NEVER** use SRV* syntax or Microsoft symbol server URLs

### 3.1 CRITICAL: Encoding and Escaping Rules

⚠️ **FOLLOW THESE RULES TO AVOID DEBUGGING FAILURES:**

#### Rule 1: ALWAYS Use ASCII Encoding
```powershell
# CORRECT
$script | Out-File -FilePath "debug.txt" -Encoding ASCII

# WRONG - CDB will fail with UTF-8/BOM
$script | Out-File -FilePath "debug.txt" -Encoding UTF8
$script | Out-File -FilePath "debug.txt" -Encoding Default
```

#### Rule 2: Use Single-Quote Here Strings for CDB Scripts
```powershell
# CORRECT - Single-quote here string (literal, no escaping)
$script = @'
bp user32!MessageBoxA ".echo CALLED; da poi(rdx) L1; g"
g
'@

# CORRECT - Double-quote here string (variable expansion OK)
$script = @"
bp $moduleName!$functionName ".echo CALLED; g"
g
"@

# WRONG - Complex nested quotes will break
$script = "bp user32!MessageBoxA \".echo CALLED; da poi(rdx) L1; g\""
```

#### Rule 3: NO Chinese Characters in CDB Scripts
```powershell
# CORRECT - English only for CDB commands
$script = @'
$$ Set breakpoint on MessageBoxA
bp user32!MessageBoxA ".echo *** CALLED ***; g"
g
'@

# WRONG - Chinese causes CDB syntax errors
$script = @'
$$ 在MessageBoxA下断点
bp user32!MessageBoxA ".echo 调用; g"
g
'@
```

#### Rule 4: Write Script to File, Don't Pass Inline
```powershell
# CORRECT - File-based approach
$script = @'
bp user32!MessageBoxA ".echo CALLED; g"
g
'@
$script | Out-File -FilePath "cdb_script.txt" -Encoding ASCII
& $cdb64 -c "`$<$pwd\cdb_script.txt" $exePath 2>$null

# WRONG - Inline escaping fails with complex commands
& $cdb64 -c "bp user32!MessageBoxA \".echo CALLED; g\"" $exePath
```

#### Rule 5: Chinese OK in PowerShell Output Messages Only
```powershell
# CORRECT - Chinese in PowerShell-only output
Write-Host "正在设置断点..." -ForegroundColor Cyan

# CORRECT - English in CDB script
$script = @'
bp user32!MessageBoxA ".echo *** BREAKPOINT HIT ***; g"
g
'@

# WRONG - Chinese in CDB command strings
$script = @'
bp user32!MessageBoxA ".echo *** 断点命中 ***; g"
g
'@
```

---

## 4. Essential CDB Commands Reference

### 4.1 Symbol and Module Commands

```cdb
x myapp!*                  # List all symbols in module (requires local PDB)
x myapp!WinMain            # Show address of a specific symbol
x *!*MessageBox*           # Search all modules for symbol containing "MessageBox"
lm                         # List loaded modules
lmv m myapp                # Verbose info about a specific module
.sympath C:\symbols        # Set LOCAL symbol path only
.sympath .                 # Use current directory for symbols
.reload                    # Reload symbols from local path
.reload /f myapp.dll       # Force reload symbols for a module
```

**Note**: System DLLs (user32, kernel32, etc.) will not show detailed symbols without Microsoft symbol server. This is expected and acceptable for most debugging tasks.

### 4.2 Breakpoints

```cdb
bp myapp!WinMain           # Set breakpoint at function entry
bp myapp!main+0x30         # Set breakpoint at offset
bp myapp!function          # Set breakpoint by name
bl                         # List all breakpoints
bc 1                       # Clear breakpoint #1
bc *                       # Clear all breakpoints
bd 1                       # Disable breakpoint #1
be 1                       # Enable breakpoint #1
```

### 4.3 Execution Control

```cdb
g                          # Go (resume execution)
p                          # Step over (step to next line, skipping calls)
t                          # Step into (trace into calls)
gu                         # Step out (run until return)
pa 0xaddress               # Step to address
pc                         # Step to next call instruction
gh                         # Go with exception handled
gn                         # Go with exception not handled
```

### 4.4 Call Stack and Registers

```cdb
k                          # Show call stack
kb                         # Call stack with first 3 params
kp                         # Call stack with all params (verbose)
kn                         # Call stack with frame numbers
dds esp                    # Dump stack with symbols
r                          # Show all registers
r eax                      # Show specific register
r eax=0x42                 # Set register value
dv                         # Display local variables
dv /t                      # Display locals with types
```

### 4.5 Memory Inspection

```cdb
db address                 # View memory as bytes
dw address                 # View memory as words (2 bytes)
dd address                 # View memory as dwords (4 bytes)
dq address                 # View memory as qwords (8 bytes)
da address                 # View as ASCII string
du address                 # View as Unicode string
dW address                 # View as wide characters
!address address           # Show memory region info
s -u 0 L?7fffffff "hello"  # Search Unicode string in memory
s -a 0 L?7fffffff "hello"  # Search ASCII string in memory
```

### 4.6 Disassembly

```cdb
u address                  # Disassemble (8 instructions by default)
u address L20              # Disassemble 20 instructions
uf myapp!WinMain           # Disassemble an entire function
uf myapp!WinMain L50       # Disassemble first 50 bytes of a function
ub address                 # Disassemble backwards
```

### 4.7 Crash Dump Analysis

```cdb
!analyze -v                # Verbose crash analysis (recommended first step)
!analyze -vv               # Even more verbose
.exr -1                    # Show last exception record
.kframes                   # Show stack frame info
!threads                   # Show all threads
!process                   # Show process info
!peb                       # Show Process Environment Block
!teb                       # Show Thread Environment Block
|                           # List processes
~                           # List threads
~0s                        # Switch to thread 0
```

### 4.8 Process and Module Control

```cdb
.sympath C:\path           # Set LOCAL symbol path (NO SRV* or URLs)
.sympath .                 # Use current directory for symbols
.srcpath+ C:\path          # Add source file search path
.lsrcpath+ C:\path         # Add local source path
!lmi myapp                 # Show module info
!dh myapp                  # Show PE header of module
.load <ext>                # Load a debugger extension
.chain                     # List loaded extensions
.version                   # Show debugger version
```

**PROHIBITED**: Never use `.sympath SRV*` or any Microsoft symbol server URLs.

### 4.9 Conditional and Scripting

```cdb
.if (condition) { cmds }   # Conditional execution
.for (init; cond; inc) {}  # For loop
.while (condition) {}      # While loop
.printf "%mu\n", ptr       # Formatted print (Unicode string)
.printf "%ma\n", ptr       # Formatted print (ASCII string)
$$                          # Single-line comment
$$ // Comment               # Also a comment
```

---

## 5. Common Workflows

### 5.1 Quick Crash Dump Analysis

```powershell
$cdb64 = "C:\Users\j4543\AppData\Local\Microsoft\WindowsApps\cdbX64.exe"
& $cdb64 -z "crash.dmp" -c "!analyze -v; .exr -1; k; q" 2>$null
```

### 5.2 Disassemble and Analyze a Function

```powershell
$cdb64 = "C:\Users\j4543\AppData\Local\Microsoft\WindowsApps\cdbX64.exe"

$script = @"
$$ Use local symbols only
.sympath .
.reload
x myapp!*
bp myapp!WinMain
g
uf myapp!WinMain
k
dv
q
"@ | Out-File debug.txt -Encoding ascii

& $cdb64 -c "`$<`"$pwd\debug.txt`"" "myapp.exe" 2>$null
```

### 5.3 Set Breakpoint and Inspect Parameters

```powershell
$cdb64 = "C:\Users\j4543\AppData\Local\Microsoft\WindowsApps\cdbX64.exe"

$script = @"
$$ Local symbols only - NO remote symbol server
.sympath .
.reload
bp user32!MessageBoxA ".echo === MessageBoxA called ===; .echo hWnd:; .printf \"0x%p\\n\", rcx; .echo Text:; da poi(rdx) L1; .echo Caption:; da poi(r8) L1; .echo Type:; .printf \"0x%x\\n\", r9d; g"
g
q
"@ | Out-File debug.txt -Encoding ascii

& $cdb64 -c "`$<`"$pwd\debug.txt`"" "messagebox.exe" 2>$null
```

**Note**: Even without full symbols for system DLLs like user32, you can still set breakpoints on exported functions (MessageBoxA) and inspect parameters via registers and stack.

### 5.4 Attach to a Running Process

```powershell
$cdb64 = "C:\Users\j4543\AppData\Local\Microsoft\WindowsApps\cdbX64.exe"

# Find the PID first
$pid = (Get-Process notepad).Id

# Attach CDB
& $cdb64 -p $pid
```

### 5.5 Remote Debugging

```powershell
$cdb64 = "C:\Users\j4543\AppData\Local\Microsoft\WindowsApps\cdbX64.exe"

# Server side (target machine): start a debug server
# In an admin terminal on the target:
# & $cdb64 -server tcp:port=5005 -p <pid>

# Client side (your machine): connect via CDB
& $cdb64 -remote "tcp:Port=5005,Server=192.168.1.100"
```

---

## 6. PowerShell Utilities for Debugging

### 6.1 Create a Minidump of a Running Process

```powershell
$cdb64 = "C:\Users\j4543\AppData\Local\Microsoft\WindowsApps\cdbX64.exe"

$pid = (Get-Process myapp).Id
& $cdb64 -c ".dump /ma `"$pwd\crash.dmp`"" -p $pid
```

### 6.2 Capture CDB Output to a File

```powershell
$cdb64 = "C:\Users\j4543\AppData\Local\Microsoft\WindowsApps\cdbX64.exe"

& $cdb64 -c "x myapp!*; q" "myapp.exe" 2>$null | Out-File "output.txt"
```

### 6.3 Check if CDB is Available

```powershell
$cdb64 = "C:\Users\j4543\AppData\Local\Microsoft\WindowsApps\cdbX64.exe"
$cdb86 = "C:\Users\j4543\AppData\Local\Microsoft\WindowsApps\cdbX86.exe"

if (Test-Path $cdb64) {
    Write-Host "CDB x64 is available at: $cdb64"
} else {
    Write-Host "CDB x64 not found. Install with: winget install Microsoft.WinDbg"
}

if (Test-Path $cdb86) {
    Write-Host "CDB x86 is available at: $cdb86"
} else {
    Write-Host "CDB x86 not found. Install with: winget install Microsoft.WinDbg"
}
```

### 6.4 Determine Architecture of an EXE

```powershell
$cdb64 = "C:\Users\j4543\AppData\Local\Microsoft\WindowsApps\cdbX64.exe"
$cdb86 = "C:\Users\j4543\AppData\Local\Microsoft\WindowsApps\cdbX86.exe"

# Try cdbX64 first
& $cdb64 -c "lm; q" "program.exe" 2>$null
# If it fails with "bad image" error, use cdbX86 for 32-bit EXE
```

---

## 7. Tips and Best Practices

1. **Architecture matching**: Always use the correct CDB architecture:
   - Use `cdbX64.exe` for 64-bit executables
   - Use `cdbX86.exe` for 32-bit executables
   - Use `cdbARM64.exe` for ARM64 executables
   - If unsure, try cdbX64 first — if it fails with a "bad image" error, use cdbX86

2. **Always use full paths**: Do not rely on PATH environment variable. Always use:
   - `C:\Users\j4543\AppData\Local\Microsoft\WindowsApps\cdbX64.exe`
   - `C:\Users\j4543\AppData\Local\Microsoft\WindowsApps\cdbX86.exe`

3. **Suppress noise**: CDB prints NatVis unloading messages on exit. Use `2>$null` to keep output clean, or pipe to a file.

4. **Script files are reliable**: For multi-step debugging, write commands to a `.txt` file and use `$<filename` to execute them. This avoids PowerShell quoting headaches with CDB's command syntax.

5. **Debug symbols**: Compile your programs with `-g` (gcc/MinGW) or `/DEBUG` (MSVC) to get meaningful symbol names and source-level debugging. Without symbols, you'll still see:
   - Exported function names (e.g., `user32!MessageBoxA`)
   - Raw addresses
   - Module names and load addresses
   - Full register states and call stacks

6. **Breakpoints without full symbols**: You can still set breakpoints on:
   - Exported functions (e.g., `bp user32!MessageBoxA`)
   - Raw addresses (e.g., `bp 0x7FFE12345678`)
   - Your own module's symbols if compiled with debug info

7. **PIDs change**: When attaching to processes, get the PID dynamically with `Get-Process -Name "name" | Select-Object -ExpandProperty Id`.

8. **Crash dump first step**: Always run `!analyze -v` first on a crash dump — it provides the exception code, faulting instruction, call stack, and often a bug description.

9. **Path quoting**: Always wrap paths with spaces in quotes when passing to CDB. Use PowerShell's `"` quoting or backtick escaping.

10. **CDB-only approach**: This skill uses CDB exclusively for all debugging tasks. For GUI-based debugging, consider using WinDbg Preview separately.