# Security Audit Report
# CTF Reverse Engineering Walkthrough: Spooky Pass
# This script contains the ordered sequence of commands used to crack, inspect, and solve the challenge.

# ==========================================
# PHASE 1: Extracting and Cracking the Archive
# ==========================================

# 1. Prepare the hash from the password-protected ZIP archive for John the Ripper
zip2john a12c739e-dddf-43d7-bbf0-c4389ea79b09.zip > clean_hash.txt

# 2. Crack the archive hash using John the Ripper with a wordlist and rules
john --wordlist=/usr/share/seclists/Passwords/Common-Credentials/top-passwords-shortlist.txt --rules=best64 clean_hash.txt

# 3. View the cracked password to extract the contents
john --show clean_hash.txt

# 4. Extract the ZIP archive using the recovered password
unzip a12c739e-dddf-43d7-bbf0-c4389ea79b09.zip


# ==========================================
# PHASE 2: Static Analysis of the Binary
# ==========================================

# 5. Navigate into the extracted directory containing the binary
cd rev_spookypass/

# 6. Check the file type and architecture properties of the executable
file pass

# 7. List the compiled functions and symbol table since the binary is not stripped
nm pass | grep -v ' U '


# ==========================================
# PHASE 3: Dynamic Analysis & Debugging with GDB
# ==========================================

# 8. Launch the GNU Debugger on the target binary
gdb ./pass

# (Inside GDB interactive prompt):
# ------------------------------------------

# Set a breakpoint at the start of the main function to inspect initialization flow
break main

# Run the program inside the debugger
run

# Disassemble the main function to analyze instruction logic and comparison checks
disassemble main

# Set a breakpoint right before the 'strcmp' password evaluation routine
break *0x0000555555555250

# Continue execution until it hits the password prompt input check
continue

# (Type any test input when prompted, e.g., 'test')

# Extract the hardcoded password string directly from the comparison memory address
p (char*)0x555555556080

# Clear previous breakpoints and set a new breakpoint right after a successful password check
delete
break *0x0000555555555260

# Resume program execution
continue

# (Provide the correct cracked password: s3cr3t_p455_f0r_gh05t5_4nd_gh0ul5 when prompted)

# Step over/through the execution loop to let the binary print the final flag
next
