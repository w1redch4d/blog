+++
title = 'PlaidCTF'
date = 2025-04-11T11:14:17+05:30
draft = false
summary = "My Writeups for PlaidCTF 2025"
[params]
  author = 'w1redch4d'
+++

## V8 my beloved (or Excav8) 
```
>be me
>doing PlaidCTF 2025
>challenge name smells like v8
>ok lets peek
```

They give us a `chall.py` and `gen.js`.
The Python script basically reads from `secret.txt` (yea, that’s the flag lol), turns every character into 8-bit binary.
For each bit:
- if it’s 0, it runs `d8 gen.js`
- if it’s 1, it runs `node gen.js`
Both generate 24 random floats.

gen.js is just:
```js
for (let i = 0; i < 24; i++) {
    console.log(Math.random());
}
```
That's it. Peak simplicity. But here’s where it gets cursed.
Turns out, they’re cooking with two versions of V8:
- d8 uses Chrome’s V8 version 13.6.1
- node is dragging an older V8 version like it’s still 2017
And plot twist: **both use different RNG implementations.**
Look at this goofy ahh RNG from V8:
```cpp
static inline double ToDouble(uint64_t state0) {
  double random = static_cast<double>(state0 >> 11);
  constexpr double k2_53{static_cast<uint64_t>(1) << 53};
  return random / k2_53;
}
```
Clean. Simple. Divide and conquer.
Now Node’s V8 is on some old head stuff:
```cpp
static inline double ToDouble(uint64_t state0) {
  static const uint64_t kExponentBits = uint64_t{0x3FF0000000000000};
  uint64_t random = (state0 >> 12) | kExponentBits;
  return base::bit_cast<double>(random) - 1;
}
```
They’re literally doing bitcast shenanigans. Old school.

> why does this matter ?

Because, brainlet, if you see the output numbers, you can tell which RNG barfed them out.
V8 and Node RNGs leave different "fingerprints" because of how they convert state to float.
Big brain moment incoming.

Ok, but how do we actually leak the flag?
> enter Z3

Z3 is basically satan in SMT solver form.
You throw it some constraints like "make these fake random numbers match these outputs", and it does all the unholy math for you.
So we:
- Model Node.js RNG in Z3
- Feed it the batch of 24 numbers
- If Z3 is like "yeah I can solve this", it’s Node RNG => bit is 1
- If Z3 is like "nah fam", then it’s V8 RNG => bit is 0

Simple as that.
Bonus round: turns out V8 is turbo cursed and has an internal LIFO cache for RNG numbers.
So when they ask for a number, they pre-gen a batch and pop them off stack style.
So you gotta reverse the batch before feeding it to Z3. Classic gotcha.

Anyway here’s the script I slapped together while running on fumes and caffeine:
```py
import z3
import struct

def read_random_outputs(filename):
    with open(filename, 'r') as file:
        return [float(line.strip()) for line in file if line.strip()]

def is_nodejs_rng(reversed_numbers):
    solver = z3.Solver()
    initial_state_0, initial_state_1 = z3.BitVecs("initial_state_0 initial_state_1", 64)
    state_0, state_1 = initial_state_0, initial_state_1

    for random_number in reversed_numbers:
        next_state_0 = state_1
        temp_state = state_0

        # Node.js RNG state transition
        temp_state ^= temp_state << 23
        temp_state ^= z3.LShR(temp_state, 17)
        temp_state ^= next_state_0
        temp_state ^= z3.LShR(next_state_0, 26)
        next_state_1 = temp_state

        # Extract mantissa from float
        float_plus_one = random_number + 1
        packed_float = struct.pack('d', float_plus_one)
        unpacked_bits = struct.unpack('<Q', packed_float)[0]
        mantissa_bits = unpacked_bits & ((1 << 52) - 1)

        solver.add(z3.LShR(next_state_0, 12) == mantissa_bits)

        state_0, state_1 = next_state_0, next_state_1

    return solver.check() == z3.sat

def main():
    random_numbers = read_random_outputs('output.txt')
    numbers_per_batch = 24
    batches = [random_numbers[i:i + numbers_per_batch] for i in range(0, len(random_numbers), numbers_per_batch)]
    recovered_bits = []

    for batch in batches:
        if len(batch) != numbers_per_batch:
            continue
        reversed_batch = batch[::-1]
        is_node = is_nodejs_rng(reversed_batch)
        bit = '1' if is_node else '0'
        recovered_bits.append(bit)
        print(bit)

    print(''.join(recovered_bits))

if __name__ == "__main__":
    main()
```

I run this cursed thing and boom:
```
011001100110110001100001011001110011101000100000010100000100001101010100010001100111101101000010011101010110100101101100010001000011000101101110010001110101111101110110001110000101111101101001001101010101111101010011011101010100001101101000010111110011010001011111011100000110000100110001010011100010111000101110001011100111110100001010011100000110000101110011011100110111011101101111011100100110010000100000011101000110111100100000011100000110000101110010011101000010000000110010001110100010000001101111011000010111000100110001010011010100010000111001001100100110010101110110010100100111001101000100010110100111011001001000
```
Converting binary to ascii:
```
flag: PCTF{BuilD1nG_v8_i5_SuCh_4_pa1N...}
password to part 2: oaq1MD92evRsDZvH
```
free flag ez clap
turns out someone already made V8 RNG solvers with Z3, check YouTube for more cursed content:
- [Nathanial Lattimer](https://www.youtube.com/watch?v=_Iv6fBrcbAM)
- [Pwn Function](https://youtu.be/-h_rj2-HP2E)

## Fool's Gulch

```
>be me
>x86-64 machine
>get handed this aarch64 binary for reverse
>ohno.jpg
>alright let's make this work
>install aarch64 libs on local box
>copy ld-linux-aarch64.so.1 to your challenge folder like a pro /usr/aarch64-linux-gnu/lib/ld-linux-aarch64.so.1
>do the same for libc because binary go brrrr
>cp /usr/aarch64-linux-gnu/lib/libc.so.6 aarch64-libs/lib/
>now use qemu-aarch64 to emulate:
qemu-aarch64 -g 1234 -L aarch64-libs ./prospectors_claim
>-g 1234 is for GDB to attach, because we’re not savages
>binary will wait until you hook gdb-multiarch to it
>goodstuff.png
>time to decompile
>use Ghidra because why not
>decompiles into absolute spaghetti
```
Here is the decompiled output:
```cpp
undefined8 main(void)

{
  byte local_55;
  byte local_54;
  byte local_53;
  byte local_52;
  byte local_51;
  byte local_50;
  byte local_4f;
  byte local_4e;
  byte local_4d;
  byte local_4c;
  byte local_4b;
  byte local_4a;
  byte local_49;
  byte local_48;
  byte local_47;
  byte local_46;
  byte local_45;
  byte local_44;
  byte local_43;
  byte local_42;
  byte local_41;
  byte local_40;
  byte local_3f;
  byte local_3e;
  byte local_3d;
  byte local_3c;
  byte local_3b;
  byte local_3a;
  byte local_39;
  byte local_38;
  byte local_37;
  byte local_36;
  byte local_35;
  byte local_34;
  byte local_33;
  byte local_32;
  char local_31;
  byte local_30;
  byte local_2f;
  char local_2e;
  byte local_2d;
  byte local_2c;
  byte local_2b;
  byte local_2a;
  byte local_29;
  byte local_28;
  byte local_27;
  byte local_26;
  byte local_25;
  byte local_24;
  byte local_23;
  byte local_22;
  byte local_21;
  byte local_20;
  byte local_1f;
  byte local_1e;
  byte local_1d;
  byte local_1c;
  byte local_1b;
  byte local_1a;
  byte local_19;
  byte local_18;
  byte local_17;
  byte local_16;
  undefined4 local_14;
  
  local_14 = 0;
  setvbuf(_stdout,(char *)0x0,2,0);
  printf("=== The Prospector\'s Claim ===\n");
  printf("Old Man Jenkins\' map to his modest gold claim has been floating around\n");
  printf("Fool\'s Gulch for years. Most folks think it\'s worthless, but you\'ve\n");
  printf("noticed something peculiar in the worn-out corners...\n\n");
  printf("Enter the claim sequence: ");
  fgets((char *)&local_55,0x41,_stdin);
  if (local_3a == 0x32) {
    bump(&score);
  }
  if ((local_36 ^ local_3d) == 0xaa) {
    bump(&score);
  }
  if ((byte)(local_36 + local_27) == -0x70) {
    bump(&score);
  }
  if ((byte)(local_1a + local_31) == -0x7f) {
    bump(&score);
  }
  if (local_48 == 0xbe) {
    bump(&score);
  }
  if (local_35 == 100) {
    bump(&score);
  }
  if (local_3b == 0x31) {
    bump(&score);
  }
  if (local_2b == 0x39) {
    bump(&score);
  }
  if ((byte)(local_28 + local_16) == -0x6d) {
    bump(&score);
  }
  if (local_23 == 0x31) {
    bump(&score);
  }
  if ((byte)(local_54 + local_46) == '8') {
    bump(&score);
  }
  if ((local_48 ^ local_32) == 0x8e) {
    bump(&score);
  }
  if ((local_54 ^ local_27) == 0x71) {
    bump(&score);
  }
  if (local_2b == 0xb4) {
    bump(&score);
  }
  if (local_42 == 0x36) {
    bump(&score);
  }
  if (local_39 == local_25) {
    bump(&score);
  }
  if ((local_4b ^ local_34) == 3) {
    bump(&score);
  }
  if (local_37 == 0x76) {
    bump(&score);
  }
  if (local_50 == 0x32) {
    bump(&score);
  }
  if ((local_54 ^ local_25) == 0x85) {
    bump(&score);
  }
  if (local_53 == 0x54) {
    bump(&score);
  }
  if (local_3e == 0xe1) {
    bump(&score);
  }
  if ((local_4c ^ local_1f) == 0x81) {
    bump(&score);
  }
  if ((byte)(local_20 + local_3b) == 'a') {
    bump(&score);
  }
  if ((local_43 ^ local_1c) == 7) {
    bump(&score);
  }
  if ((local_43 ^ local_50) == 0x56) {
    bump(&score);
  }
  if ((byte)(local_47 + local_20) == 'f') {
    bump(&score);
  }
  if ((local_25 ^ local_1d) == 0x54) {
    bump(&score);
  }
  if (local_21 == 0xe5) {
    bump(&score);
  }
  if (local_31 == 'o') {
    bump(&score);
  }
  if ((byte)(local_30 + local_37) == 'f') {
    bump(&score);
  }
  if (local_23 == 0x31) {
    bump(&score);
  }
  if (local_3e == 0x38) {
    bump(&score);
  }
  if (local_2e == '4') {
    bump(&score);
  }
  if ((byte)(local_1d + local_42) == -0x69) {
    bump(&score);
  }

[...]

  if ((local_30 ^ local_53) == 0x67) {
    bump(&score);
  }
  if ((local_1f ^ local_45) == 0x7b) {
    bump(&score);
  }
  if ((local_28 ^ local_43) == 0x91) {
    bump(&score);
  }
  if (local_43 == 100) {
    bump(&score);
  }
  if (score < 0x119) {
    if (score < 0xFC) {
      if (score < 0x8C) {
        printf("\nThat claim\'s as empty as a desert well in August.\n",
        printf("Not a speck of gold to be found. Try another spot, prospector!\n");
      }
      else {
        printf("The saloon erupts in laughter as you show off your \'treasure\'.\n");
        printf("Keep prospecting - or take up farming instead!\n");
      }
    }
    else {
      printf("The assayer laughs you out of his office. \"Come back when you\'ve got\n");
      printf("something worth my time, greenhorn!\"\n");
    }
  }
  else {
    printf("You\'ve struck a rich vein of gold! Your claim is officially recorded\n");
    printf("at the assayer\'s office, and the flag is yours: %s\n",&local_55);
  }
  return 0;
}
```
```
>literally hundreds of these conditions
>ohgodwhy.gif
>turns out you don’t need all of them to hit gold
>just enough to cross score > 0x119
>optimal_strategy.png
```
### Obviously a job for z3 solver
```
>parse all the conditions
>translate to Z3 expressions
>let Z3 figure out what values for flag make score high enough
>profit
```
Here is the ghidra python script:
```py
import re
from ghidra.app.decompiler import DecompInterface
from ghidra.util.task import ConsoleTaskMonitor

def decompile_function(func):
    decomp_interface = DecompInterface()
    decomp_interface.openProgram(currentProgram)
    result = decomp_interface.decompileFunction(func, 60, ConsoleTaskMonitor())
    if result.decompileCompleted():
        return result.getDecompiledFunction().getC()
    else:
        print(f"Failed to decompile function: {func.getName()}")
        return ""

def extract_conditions_from_code(code_text):
    condition_regex = re.compile(r'if\s*\((.*?)\)\s*{\s*bump\(&score\);', re.DOTALL)
    conditions = condition_regex.findall(code_text)
    return conditions

def transform_condition(condition):
    # Replace variable names like local_55 with flag[index]
    variable_regex = re.compile(r'local_([0-9a-fA-F]{2})')
    def replace_variable(match):
        local_offset = int(match.group(1), 16)
        flag_index = 0x55 - local_offset
        return f'flag[{flag_index}]'
    condition = variable_regex.sub(replace_variable, condition)

    # Simplify (byte)(expression) to (expression & 0xFF)
    byte_cast_regex = re.compile(r'\(byte\)\s*\((.*?)\)')
    condition = byte_cast_regex.sub(r'(\1 & 0xFF)', condition)

    # Replace negative hex like -0x2A with positive equivalent
    negative_hex_regex = re.compile(r'-0x([0-9a-fA-F]+)')
    def replace_negative_hex(match):
        value = int(match.group(1), 16)
        positive_equiv = (0x100 - value) & 0xFF
        return f'0x{positive_equiv:02x}'
    condition = negative_hex_regex.sub(replace_negative_hex, condition)

    return condition

def main():
    output_file = askFile("Select output file", "Save")
    target_function_name = askString("Function Name", "Enter the target function name (e.g., main):")

    function_manager = currentProgram.getFunctionManager()
    target_function = function_manager.getFunctionAt(toAddr(0))
    for func in function_manager.getFunctions(True):
        if func.getName() == target_function_name:
            target_function = func
            break

    if not target_function:
        print(f"Function '{target_function_name}' not found.")
        return

    decompiled_code = decompile_function(target_function)
    conditions = extract_conditions_from_code(decompiled_code)

    with open(output_file.absolutePath, 'w') as out:
        out.write("# Auto-generated conditions for Z3 solver\n\n")
        out.write("score = 0\n")
        out.write("flag = [BitVec(f'flag_{i}', 8) for i in range(FLAG_LENGTH)]\n\n")  # Replace FLAG_LENGTH as needed

        for idx, raw_condition in enumerate(conditions, 1):
            clean_condition = transform_condition(raw_condition)
            out.write(f"# Condition {idx}\n")
            out.write(f"cond{idx} = ({clean_condition})\n")
            out.write(f"score += If(cond{idx}, 1, 0)\n\n")

    print(f"Extraction complete. Conditions written to: {output_file.absolutePath}")

if __name__ == '__main__':
    main()
```
```
>feed output to Z3
>let the solver cook
>Z3: here’s your flag, enjoy
```
```bash
=== The Prospector's Claim ===
Old Man Jenkins' map to his modest gold claim has been floating around
Fool's Gulch for years. Most folks think it's worthless, but you've
noticed something peculiar in the worn-out corners...

Enter the claim sequence: PCTF{24d16126d6739d6ada82b125534d2ae2324b39ed72e5a1200c5ac96200} 

🌟 PAYDIRT! 🌟
You've struck a rich vein of gold! Your claim is officially recorded
at the assayer's office, and the flag is yours: PCTF{24d16126d6739d6ada82b125534d2ae2324b39ed72e5a1200c5ac96200}
```
```
>ezpz lemon squeezy
>mfw I spent more time installing qemu than solving the binary
```

## Prospector (Reversing)

Just another redundant z3 baby rev so wont waste your time elaborating the thing, here is the solution i wrote to get the flag:
```py
#!/usr/bin/env python
from z3 import *

# Create an Optimize instance
opt = Optimize()

# Create all symbolic 8-bit variables
names = ["s", "v5", "v6", "v7", "v8", "v9", "v10", "v11", "v12", "v13", "v14", "v15",
         "v16", "v17", "v18", "v19", "v20", "v21", "v22", "v23", "v24", "v25", "v26",
         "v27", "v28", "v29", "v30", "v31", "v32", "v33", "v34", "v35", "v36", "v37",
         "v38", "v39", "v40", "v41", "v42", "v43", "v44", "v45", "v46", "v47", "v48",
         "v49", "v50", "v51", "v52", "v53", "v54", "v55", "v56", "v57", "v58", "v59",
         "v60", "v61", "v62", "v63", "v64", "v65", "v66", "v67"]

vars = { name: BitVec(name, 8) for name in names }

# For convenience, assign names
s   = vars["s"]
v5  = vars["v5"]
v6  = vars["v6"]
v7  = vars["v7"]
v8  = vars["v8"]
v9  = vars["v9"]
v10 = vars["v10"]
v11 = vars["v11"]
v12 = vars["v12"]
v13 = vars["v13"]
v14 = vars["v14"]
v15 = vars["v15"]
v16 = vars["v16"]
v17 = vars["v17"]
v18 = vars["v18"]
v19 = vars["v19"]
v20 = vars["v20"]
v21 = vars["v21"]
v22 = vars["v22"]
v23 = vars["v23"]
v24 = vars["v24"]
v25 = vars["v25"]
v26 = vars["v26"]
v27 = vars["v27"]
v28 = vars["v28"]
v29 = vars["v29"]
v30 = vars["v30"]
v31 = vars["v31"]
v32 = vars["v32"]
v33 = vars["v33"]
v34 = vars["v34"]
v35 = vars["v35"]
v36 = vars["v36"]
v37 = vars["v37"]
v38 = vars["v38"]
v39 = vars["v39"]
v40 = vars["v40"]
v41 = vars["v41"]
v42 = vars["v42"]
v43 = vars["v43"]
v44 = vars["v44"]
v45 = vars["v45"]
v46 = vars["v46"]
v47 = vars["v47"]
v48 = vars["v48"]
v49 = vars["v49"]
v50 = vars["v50"]
v51 = vars["v51"]
v52 = vars["v52"]
v53 = vars["v53"]
v54 = vars["v54"]
v55 = vars["v55"]
v56 = vars["v56"]
v57 = vars["v57"]
v58 = vars["v58"]
v59 = vars["v59"]
v60 = vars["v60"]
v61 = vars["v61"]
v62 = vars["v62"]
v63 = vars["v63"]
v64 = vars["v64"]
v65 = vars["v65"]
v66 = vars["v66"]
v67 = vars["v67"]

# Optionally constrain all bytes to be printable (for example, ASCII 32 to 126)
for var in vars.values():
    opt.add(var >= 32, var <= 126)

# Define the score as the sum of all bumps.
# Each bump is modeled as If(condition, 1, 0)
score = 0
score += If(v31 == 50, 1, 0)
score += If(v35 ^ v28 == 0xAA, 1, 0)
score += If(v35 + v50 == 144, 1, 0)
score += If(v63 + v40 == 129, 1, 0)
score += If(v17 == 190, 1, 0)
score += If(v36 == 100, 1, 0)
score += If(v30 == 49, 1, 0)
score += If(v46 == 57, 1, 0)
score += If(v49 + v67 == 147, 1, 0)
score += If(v54 == 49, 1, 0)
score += If(v5 + v19 == 56, 1, 0)
score += If(v17 ^ v39 == 0x8E, 1, 0)
score += If(v5 ^ v50 == 0x71, 1, 0)
score += If(v46 == 180, 1, 0)
score += If(v23 == 54, 1, 0)
score += If(v32 == v52, 1, 0)
score += If(v14 ^ v37 == 3, 1, 0)
score += If(v34 == 118, 1, 0)
score += If(v9 == 50, 1, 0)
score += If(v5 ^ v52 == 0x85, 1, 0)
score += If(v6 == 84, 1, 0)
score += If(v27 == 225, 1, 0)
score += If(v13 ^ v58 == 0x81, 1, 0)
score += If(v57 + v30 == 97, 1, 0)
score += If(v22 ^ v61 == 7, 1, 0)
score += If(v22 ^ v9 == 0x56, 1, 0)
score += If(v18 + v57 == 102, 1, 0)
score += If(v52 ^ v60 == 0x54, 1, 0)
score += If(v56 == 229, 1, 0)
score += If(v40 == 111, 1, 0)
score += If(v41 + v34 == 102, 1, 0)
score += If(v54 == 49, 1, 0)
score += If(v27 == 56, 1, 0)
score += If(v43 == 52, 1, 0)
score += If(v60 + v23 == 151, 1, 0)
score += If(v23 ^ v54 == 0x2D, 1, 0)
score += If(v38 + v5 == 164, 1, 0)
score += If(v51 + v42 == 244, 1, 0)
score += If(v31 + v51 == 151, 1, 0)
score += If(v51 + v14 == 150, 1, 0)
score += If(v6 == 114, 1, 0)
score += If(v63 == 54, 1, 0)
score += If(v32 == 177, 1, 0)
score += If(v12 ^ v62 == 8, 1, 0)
score += If(v6 ^ v50 == 0x3A, 1, 0)
score += If(v63 == 68, 1, 0)
score += If(v61 == 18, 1, 0)
score += If(v21 ^ v44 == 0x5B, 1, 0)
score += If(v45 == 42, 1, 0)
score += If(v45 == 51, 1, 0)
score += If(v30 + v42 == 63, 1, 0)
score += If(v65 ^ s == 0x60, 1, 0)
score += If(v25 == 36, 1, 0)
score += If(v9 + v47 == 45, 1, 0)
score += If(v61 == 99, 1, 0)
score += If(v59 ^ v56 == 5, 1, 0)
score += If(v8 == 123, 1, 0)
score += If(v20 + v13 == 173, 1, 0)
score += If(v26 + v49 == 152, 1, 0)
score += If(v28 == 50, 1, 0)
score += If(v52 == 53, 1, 0)
score += If(v58 ^ v23 == 0x5B, 1, 0)
score += If(v58 + v18 == 153, 1, 0)
score += If(v57 == 210, 1, 0)
score += If(s ^ v52 == 0x65, 1, 0)
score += If(v46 + v53 == 154, 1, 0)
score += If(v24 + v33 == 150, 1, 0)
score += If(v44 ^ v54 == 0xDA, 1, 0)
score += If(v64 + v36 == 150, 1, 0)
score += If(v43 + v11 == 152, 1, 0)
score += If(v54 == v14, 1, 0)
score += If(v61 ^ v14 == 0x52, 1, 0)
score += If(v60 == 97, 1, 0)
score += If(v24 == 97, 1, 0)
score += If(v66 ^ v12 == 1, 1, 0)
score += If(v48 ^ v38 == 0x30, 1, 0)
score += If(v37 == 50, 1, 0)
score += If(v51 ^ v53 == 4, 1, 0)
score += If(v31 ^ v26 == 0x53, 1, 0)
score += If(s + v45 == 131, 1, 0)
score += If(v22 == 241, 1, 0)
score += If(v32 == 53, 1, 0)
score += If(v20 == 211, 1, 0)
score += If(v10 + v51 == 153, 1, 0)
score += If(v32 + v29 == 175, 1, 0)
score += If(v67 + v23 == 179, 1, 0)
score += If(v33 == 53, 1, 0)
score += If(v65 == 48, 1, 0)
score += If(v19 + v29 == 133, 1, 0)
score += If(v24 ^ v65 == 0x4F, 1, 0)
score += If(v31 ^ v54 == 3, 1, 0)
score += If(v66 + v55 == 98, 1, 0)
score += If(v24 + v48 == 197, 1, 0)
score += If(v47 + v22 == 201, 1, 0)
score += If(v13 + v40 == 104, 1, 0)
score += If(v13 == v23, 1, 0)
score += If(v23 ^ v37 == 4, 1, 0)
score += If(v59 + s == 133, 1, 0)
score += If(v26 == 97, 1, 0)
score += If(v21 == 57, 1, 0)
score += If(v14 == 49, 1, 0)
score += If(v57 == 48, 1, 0)
score += If(v53 + v65 == 233, 1, 0)
score += If(v54 == 49, 1, 0)
score += If(v59 + v7 == 191, 1, 0)
score += If(v49 + v29 == 153, 1, 0)
score += If(v50 == 50, 1, 0)
score += If(v28 + v47 == 151, 1, 0)
score += If(v30 == 49, 1, 0)
score += If(v18 == 54, 1, 0)
score += If(v7 + v66 == 118, 1, 0)
score += If(v31 == 50, 1, 0)
score += If(v67 ^ v49 == 0x4A, 1, 0)
score += If(v48 + v18 == 240, 1, 0)
score += If(v13 == 216, 1, 0)
score += If(v62 + v66 == 98, 1, 0)
score += If(v65 + v26 == 147, 1, 0)
score += If(v55 ^ v57 == 2, 1, 0)
score += If(v31 == 50, 1, 0)
score += If(v54 ^ v63 == 7, 1, 0)
score += If(v56 == 48, 1, 0)
score += If(v37 == 121, 1, 0)
score += If(v65 == 51, 1, 0)
score += If(v25 + v53 == 197, 1, 0)
score += If(v40 == 83, 1, 0)
score += If(v19 == 9, 1, 0)
score += If(s + v36 == 180, 1, 0)
score += If(v18 + v55 == 104, 1, 0)
score += If(v47 ^ v18 == 0xFB, 1, 0)
score += If(v62 == 79, 1, 0)
score += If(v14 == 62, 1, 0)
score += If(v51 == 236, 1, 0)
score += If(v49 == 55, 1, 0)
score += If(v26 + v63 == 255, 1, 0)
score += If(v28 ^ v6 == 0x28, 1, 0)
score += If(v52 == 209, 1, 0)
score += If(v59 ^ v11 == 0xE8, 1, 0)
score += If(v51 == 101, 1, 0)
score += If(v9 + v61 == 252, 1, 0)
score += If(v28 + v9 == 100, 1, 0)
score += If(v38 ^ v58 == 2, 1, 0)
score += If(v13 == 54, 1, 0)
score += If(v9 == 50, 1, 0)
score += If(v30 == 49, 1, 0)
score += If(v64 == 50, 1, 0)
score += If(v8 + v11 == 223, 1, 0)
score += If(v35 == 52, 1, 0)
score += If(v54 ^ v56 == 0x59, 1, 0)
score += If(v62 == 57, 1, 0)
score += If(v16 ^ v47 == 0x53, 1, 0)
score += If(v11 == 97, 1, 0)
score += If(v57 ^ v11 == 0x54, 1, 0)
score += If(v63 + v10 == 106, 1, 0)
score += If(v21 + v32 == 110, 1, 0)
score += If(v8 == 123, 1, 0)
score += If(v39 == 101, 1, 0)
score += If(v40 + v48 == 150, 1, 0)
score += If(v67 == 125, 1, 0)
score += If(v31 ^ v45 == 0x5D, 1, 0)
score += If(v44 == 98, 1, 0)
score += If(v28 ^ v25 == 0x63, 1, 0)
score += If(v25 ^ v23 == 0x52, 1, 0)
score += If(v14 == 49, 1, 0)
score += If(v41 + v10 == 103, 1, 0)
score += If(v10 == 123, 1, 0)
score += If(v60 == 147, 1, 0)
score += If(v13 ^ v33 == 3, 1, 0)
score += If(v51 == 199, 1, 0)
score += If(v33 == 53, 1, 0)
score += If(v67 == 125, 1, 0)
score += If(v55 + v42 == 71, 1, 0)
score += If(v64 + v24 == 147, 1, 0)
score += If(v5 ^ v19 == 0x74, 1, 0)
score += If(v15 ^ v62 == 0x16, 1, 0)
score += If(v54 == 49, 1, 0)
score += If(s + v67 == 205, 1, 0)
score += If(v64 == 50, 1, 0)
score += If(v58 == 99, 1, 0)
score += If(v11 == 75, 1, 0)
score += If(v33 ^ v44 == 0x57, 1, 0)
score += If(v62 == 57, 1, 0)
score += If(v37 ^ v48 == 0x56, 1, 0)
score += If(v21 == 150, 1, 0)
score += If(v23 == 171, 1, 0)
score += If(v63 + v55 == 175, 1, 0)
score += If(v25 == 127, 1, 0)
score += If(v39 ^ v15 == 0xD1, 1, 0)
score += If(v44 == 98, 1, 0)
score += If(v22 ^ v17 == 0x5E, 1, 0)
score += If(v11 == 22, 1, 0)
score += If(v58 + v10 == 151, 1, 0)
score += If(v54 == 49, 1, 0)
score += If(v49 == 55, 1, 0)
score += If(v23 + v9 == 104, 1, 0)
score += If(v8 == 26, 1, 0)
score += If(v66 == 48, 1, 0)
score += If(v16 ^ v28 == 4, 1, 0)
score += If(v50 + v36 == 150, 1, 0)
score += If(v38 == 97, 1, 0)
score += If(v7 == 70, 1, 0)
score += If(v15 + v51 == 151, 1, 0)
score += If(v50 == 50, 1, 0)
score += If(v29 == 96, 1, 0)
score += If(v13 == 54, 1, 0)
score += If(v30 == 58, 1, 0)
score += If(v59 ^ v46 == 0xC, 1, 0)
score += If(v38 ^ v23 == 0x57, 1, 0)
score += If(v60 == 97, 1, 0)
score += If(v47 ^ v35 == 0x51, 1, 0)
score += If(v59 == 53, 1, 0)
score += If(v13 == 169, 1, 0)
score += If(v50 + v7 == 120, 1, 0)
score += If(v52 == 31, 1, 0)
score += If(v39 + v27 == 157, 1, 0)
score += If(v40 == 109, 1, 0)
score += If(v34 ^ v32 == 6, 1, 0)
score += If(v57 == 48, 1, 0)
score += If(v24 == 247, 1, 0)
score += If(v54 == 49, 1, 0)
score += If(v64 + v49 == 105, 1, 0)
score += If(v45 ^ v16 == 5, 1, 0)
score += If(v13 ^ v46 == 0x85, 1, 0)
score += If(v36 == 100, 1, 0)
score += If(v57 == 48, 1, 0)
score += If(v42 + v59 == 103, 1, 0)
score += If(v36 == v17, 1, 0)
score += If(v40 == 50, 1, 0)
score += If(v27 == 155, 1, 0)
score += If(v17 == 100, 1, 0)
score += If(v12 == 49, 1, 0)
score += If(v40 == 50, 1, 0)
score += If(v18 == 195, 1, 0)
score += If(v45 + v25 == 151, 1, 0)
score += If(v49 + v41 == 106, 1, 0)
score += If(v36 == 50, 1, 0)
score += If(v48 == 100, 1, 0)
score += If(v6 == 84, 1, 0)
score += If(v39 ^ v54 == 0x54, 1, 0)
score += If(v55 == 50, 1, 0)
score += If(v52 == 53, 1, 0)
score += If(v45 == 51, 1, 0)
score += If(v31 == 50, 1, 0)
score += If(v53 + v63 == 151, 1, 0)
score += If(v40 == 50, 1, 0)
score += If(v26 == 97, 1, 0)
score += If(v59 + v8 == 18, 1, 0)
score += If(v50 == 158, 1, 0)
score += If(v26 ^ v35 == 0x55, 1, 0)
score += If(v54 == 67, 1, 0)
score += If(v27 == 56, 1, 0)
score += If(v56 == 12, 1, 0)
score += If(v56 == 48, 1, 0)
score += If(v26 == 97, 1, 0)
score += If(v18 == 54, 1, 0)
score += If(v6 == 84, 1, 0)
score += If(v12 ^ v26 == 0x50, 1, 0)
score += If(v39 + s == 167, 1, 0)
score += If(v43 + v6 == 199, 1, 0)
score += If(v45 ^ v10 == 7, 1, 0)
score += If(v37 + v15 == 165, 1, 0)
score += If(v29 == 98, 1, 0)
score += If(v30 == 49, 1, 0)
score += If(v24 + v31 == 35, 1, 0)
score += If(v10 ^ v39 == 0x9C, 1, 0)
score += If(v18 ^ v15 == 4, 1, 0)
score += If(v43 == 52, 1, 0)
score += If(v44 + v35 == 150, 1, 0)
score += If(v61 == 99, 1, 0)
score += If(v11 == 100, 1, 0)
score += If(v18 == 54, 1, 0)
score += If(v9 == 67, 1, 0)
score += If(v27 ^ v26 == 0x59, 1, 0)
score += If(v25 == 100, 1, 0)
score += If(v26 ^ v48 == 5, 1, 0)
score += If(v57 == 48, 1, 0)
score += If(v35 == 52, 1, 0)
score += If(v26 == 97, 1, 0)
score += If(v50 + v12 == 122, 1, 0)
score += If(v29 ^ v7 == 0x24, 1, 0)
score += If(v47 == 101, 1, 0)
score += If(v48 == 15, 1, 0)
score += If(v20 ^ v25 == 0x57, 1, 0)
score += If(v13 ^ v66 == 6, 1, 0)
score += If(v31 == 50, 1, 0)
score += If(v22 + v58 == 209, 1, 0)
score += If(v36 == 100, 1, 0)
score += If(v61 + v22 == 199, 1, 0)
score += If(v47 ^ v36 == 1, 1, 0)
score += If(v21 == 164, 1, 0)
score += If(v12 ^ v13 == 7, 1, 0)
score += If(v44 == 98, 1, 0)
score += If(v60 + v8 == 10, 1, 0)
score += If(v25 + v32 == 153, 1, 0)
score += If(v16 == 124, 1, 0)
score += If(v67 + v22 == 225, 1, 0)
score += If(v31 == 50, 1, 0)
score += If(v27 == 56, 1, 0)
score += If(v48 + v22 == 200, 1, 0)
score += If(v13 + v23 == 108, 1, 0)
score += If(v65 + v15 == 231, 1, 0)
score += If(v25 == 100, 1, 0)
score += If(v61 + v35 == 151, 1, 0)
score += If(v32 ^ v44 == 0x57, 1, 0)
score += If(v49 == 55, 1, 0)
score += If(v17 + v51 == 201, 1, 0)
score += If(v22 == 100, 1, 0)
score += If(v19 == 55, 1, 0)
score += If(v62 ^ v20 == 0xA, 1, 0)
score += If(s ^ v25 == 0x34, 1, 0)
score += If(v66 + v53 == 145, 1, 0)
score += If(v50 == 50, 1, 0)
score += If(v61 + s == 168, 1, 0)
score += If(v40 == 50, 1, 0)
score += If(v54 == 49, 1, 0)
score += If(v52 == 53, 1, 0)
score += If(v46 == 57, 1, 0)
score += If(v65 + v39 == 149, 1, 0)
score += If(v31 + v5 == 117, 1, 0)
score += If(v52 == 53, 1, 0)
score += If(v39 ^ v37 == 0x51, 1, 0)
score += If(v33 ^ v61 == 0x56, 1, 0)
score += If(v27 == 75, 1, 0)
score += If(v35 + v17 == 172, 1, 0)
score += If(v43 == 74, 1, 0)
score += If(v43 == 52, 1, 0)
score += If(v37 == 115, 1, 0)
score += If(v11 == 100, 1, 0)
score += If(s == 80, 1, 0)
score += If(v26 == 97, 1, 0)
score += If(v66 + v59 == 101, 1, 0)
score += If(v56 == 48, 1, 0)
score += If(v11 + v26 == 204, 1, 0)
score += If(v30 ^ v18 == 7, 1, 0)
score += If(v6 == 84, 1, 0)
score += If(v50 == 50, 1, 0)
score += If(v56 == 48, 1, 0)
score += If(v58 ^ v39 == 6, 1, 0)
score += If(v35 ^ v38 == 0x55, 1, 0)
score += If(v37 == 50, 1, 0)
score += If(v44 + v37 == 148, 1, 0)
score += If(v11 == 100, 1, 0)
score += If(v19 == 55, 1, 0)
score += If(v14 ^ v47 == 0x54, 1, 0)
score += If(v19 == 55, 1, 0)
score += If(v33 + v34 == 104, 1, 0)
score += If(v47 ^ v17 == 1, 1, 0)
score += If(v62 + v28 == 217, 1, 0)
score += If(v21 + v65 == 105, 1, 0)
score += If(v45 == 166, 1, 0)
score += If(v58 ^ v28 == 0x51, 1, 0)
score += If(v66 == 48, 1, 0)
score += If(v61 + v41 == 150, 1, 0)
score += If(v65 ^ v8 == 0x4B, 1, 0)
score += If(v41 + v56 == 99, 1, 0)
score += If(v25 ^ v30 == 0x3D, 1, 0)
score += If(v54 == 233, 1, 0)
score += If(v9 ^ v33 == 7, 1, 0)
score += If(v35 == 52, 1, 0)
score += If(v57 == 1, 1, 0)
score += If(v19 + v11 == 155, 1, 0)
score += If(v45 == 51, 1, 0)
score += If(v39 + v31 == 151, 1, 0)
score += If(v46 == 57, 1, 0)
score += If(v31 == 50, 1, 0)
score += If(v51 == 101, 1, 0)
score += If(v49 ^ v34 == 4, 1, 0)
score += If(v34 == 51, 1, 0)
score += If(v26 == 97, 1, 0)
score += If(v53 == 97, 1, 0)
score += If(v35 + v25 == 152, 1, 0)
score += If(v28 == 50, 1, 0)
score += If(v19 == 55, 1, 0)
score += If(v39 == 203, 1, 0)
score += If(v8 + v41 == 174, 1, 0)
score += If(v53 == 97, 1, 0)
score += If(v11 ^ v35 == 0x50, 1, 0)
score += If(v19 + v23 == 109, 1, 0)
score += If(v34 ^ v26 == 0xC7, 1, 0)
score += If(v20 ^ v64 == 1, 1, 0)
score += If(v16 == 54, 1, 0)
score += If(v32 == 1, 1, 0)
score += If(v48 + v62 == 157, 1, 0)
score += If(v53 ^ v41 == 0x52, 1, 0)
score += If(v10 == 52, 1, 0)
score += If(v42 == 50, 1, 0)
score += If(v66 == 48, 1, 0)
score += If(v41 ^ v25 == 0x57, 1, 0)
score += If(v44 + v9 == 148, 1, 0)
score += If(v15 + v38 == 147, 1, 0)
score += If(v47 == 153, 1, 0)
score += If(v26 + v31 == 136, 1, 0)
score += If(v50 == v42, 1, 0)
score += If(v52 ^ v65 == 5, 1, 0)
score += If(v16 == 86, 1, 0)
score += If(v48 ^ v32 == 0x51, 1, 0)
score += If(v49 == 55, 1, 0)
score += If(v41 ^ v6 == 0x67, 1, 0)
score += If(v58 ^ v20 == 0x7B, 1, 0)
score += If(v49 ^ v22 == 0x91, 1, 0)
score += If(v22 == 100, 1, 0)
score += If(v19 == 55, 1, 0)
score += If(v62 ^ v20 == 0xA, 1, 0)
score += If(s ^ v25 == 0x34, 1, 0)
score += If(v66 + v53 == 145, 1, 0)
score += If(v50 == 50, 1, 0)
score += If(v61 + s == 168, 1, 0)
score += If(v40 == 50, 1, 0)
score += If(v54 == 49, 1, 0)
score += If(v52 == 53, 1, 0)
score += If(v46 == 57, 1, 0)
score += If(v65 + v39 == 149, 1, 0)
score += If(v31 + v5 == 117, 1, 0)
score += If(v52 == 53, 1, 0)
score += If(v39 ^ v37 == 0x51, 1, 0)
score += If(v33 ^ v61 == 0x56, 1, 0)
score += If(v27 == 75, 1, 0)
score += If(v35 + v17 == 172, 1, 0)
score += If(v43 == 74, 1, 0)
score += If(v43 == 52, 1, 0)
score += If(v37 == 115, 1, 0)
score += If(v11 == 100, 1, 0)
score += If(s == 80, 1, 0)
score += If(v26 == 97, 1, 0)
score += If(v66 + v59 == 101, 1, 0)
score += If(v56 == 48, 1, 0)
score += If(v11 + v26 == 204, 1, 0)
score += If(v30 ^ v18 == 7, 1, 0)
score += If(v6 == 84, 1, 0)
score += If(v50 == 50, 1, 0)
score += If(v56 == 48, 1, 0)
score += If(v58 ^ v39 == 6, 1, 0)
score += If(v35 ^ v38 == 0x55, 1, 0)
score += If(v37 == 50, 1, 0)
score += If(v44 + v37 == 148, 1, 0)
score += If(v11 == 100, 1, 0)
score += If(v19 == 55, 1, 0)
score += If(v14 ^ v47 == 0x54, 1, 0)
score += If(v19 == 55, 1, 0)
score += If(v33 + v34 == 104, 1, 0)
score += If(v47 ^ v17 == 1, 1, 0)
score += If(v62 + v28 == 217, 1, 0)
score += If(v21 + v65 == 105, 1, 0)
score += If(v45 == 166, 1, 0)
score += If(v58 ^ v28 == 0x51, 1, 0)
score += If(v66 == 48, 1, 0)
score += If(v61 + v41 == 150, 1, 0)
score += If(v65 ^ v8 == 0x4B, 1, 0)
score += If(v41 + v56 == 99, 1, 0)
score += If(v25 ^ v30 == 0x3D, 1, 0)
score += If(v54 == 233, 1, 0)
score += If(v9 ^ v33 == 7, 1, 0)
score += If(v35 == 52, 1, 0)
score += If(v57 == 1, 1, 0)
score += If(v19 + v11 == 155, 1, 0)
score += If(v45 == 51, 1, 0)
score += If(v39 + v31 == 151, 1, 0)
score += If(v46 == 57, 1, 0)
score += If(v31 == 50, 1, 0)
score += If(v51 == 101, 1, 0)
score += If(v49 ^ v34 == 4, 1, 0)
score += If(v34 == 51, 1, 0)
score += If(v26 == 97, 1, 0)
score += If(v53 == 97, 1, 0)
score += If(v35 + v25 == 152, 1, 0)
score += If(v28 == 50, 1, 0)
score += If(v19 == 55, 1, 0)
score += If(v39 == 203, 1, 0)
score += If(v8 + v41 == 174, 1, 0)
score += If(v53 == 97, 1, 0)
score += If(v11 ^ v35 == 0x50, 1, 0)
score += If(v19 + v23 == 109, 1, 0)
score += If(v34 ^ v26 == 0xC7, 1, 0)
score += If(v20 ^ v64 == 1, 1, 0)
score += If(v16 == 54, 1, 0)
score += If(v32 == 1, 1, 0)
score += If(v48 + v62 == 157, 1, 0)
score += If(v53 ^ v41 == 0x52, 1, 0)
score += If(v10 == 52, 1, 0)
score += If(v42 == 50, 1, 0)
score += If(v66 == 48, 1, 0)
score += If(v41 ^ v25 == 0x57, 1, 0)
score += If(v44 + v9 == 148, 1, 0)
score += If(v15 + v38 == 147, 1, 0)
score += If(v47 == 153, 1, 0)
score += If(v26 + v31 == 136, 1, 0)
score += If(v50 == v42, 1, 0)
score += If(v52 ^ v65 == 5, 1, 0)
score += If(v16 == 86, 1, 0)
score += If(v48 ^ v32 == 0x51, 1, 0)
score += If(v49 == 55, 1, 0)
score += If(v41 ^ v6 == 0x67, 1, 0)
score += If(v58 ^ v20 == 0x7B, 1, 0)
score += If(v49 ^ v22 == 0x91, 1, 0)
score += If(v22 == 100, 1, 0)

# (Optional) Set a threshold for success.
# For example, if we want a score at least THRESHOLD to reach the "rich vein" branch:
THRESHOLD = 100  # adjust as needed
opt.add(score >= THRESHOLD)

# Tell the optimizer to maximize the score.
h = opt.maximize(score)

def extract_solution(model, variables):
    # Create a dictionary mapping each variable's name to its ASCII character
    sol = {}
    for name, var in variables.items():
        # Convert the integer value to a character.
        # If you want to print it as a two-digit hex value or similar, you can adjust here.
        sol[name] = chr(model[var].as_long())
    return sol

# Print the extracted solution in string format:

# Check satisfiability and print result.
if opt.check() == sat:
    m = opt.model()
    max_score = m.evaluate(score)
    # For demonstration, we print the value for each variable.
    print("Maximum score achieved:", max_score)
    print("Solution:")
    solution_str = "".join(chr(m[vars[name]].as_long()) for name in names)
    print("Solution string:")
    print(solution_str)
    # Print only the input s and a selection of variables, or all if needed.
    print("s =", chr(m[s].as_long()))
    for name in names:
        # Print each variable as a character.
        val = m[vars[name]].as_long()
        print(f"{name} = {val} (char: {chr(val)})")
else:
    print("No solution found.")
```
```
Maximum score achieved: 355
Solution:
Solution string:
PCTF{24d16126d6739d6ada82b125534d2ae2324b39ed72e5a1200c5ac96200}
s = P
s = 80 (char: P)
v5 = 67 (char: C)
v6 = 84 (char: T)
v7 = 70 (char: F)
v8 = 123 (char: {)
v9 = 50 (char: 2)
v10 = 52 (char: 4)
v11 = 100 (char: d)
v12 = 49 (char: 1)
v13 = 54 (char: 6)
v14 = 49 (char: 1)
v15 = 50 (char: 2)
v16 = 54 (char: 6)
v17 = 100 (char: d)
v18 = 54 (char: 6)
v19 = 55 (char: 7)
v20 = 51 (char: 3)
v21 = 57 (char: 9)
v22 = 100 (char: d)
v23 = 54 (char: 6)
v24 = 97 (char: a)
v25 = 100 (char: d)
v26 = 97 (char: a)
v27 = 56 (char: 8)
v28 = 50 (char: 2)
v29 = 98 (char: b)
v30 = 49 (char: 1)
v31 = 50 (char: 2)
v32 = 53 (char: 5)
v33 = 53 (char: 5)
v34 = 51 (char: 3)
v35 = 52 (char: 4)
v36 = 100 (char: d)
v37 = 50 (char: 2)
v38 = 97 (char: a)
v39 = 101 (char: e)
v40 = 50 (char: 2)
v41 = 51 (char: 3)
v42 = 50 (char: 2)
v43 = 52 (char: 4)
v44 = 98 (char: b)
v45 = 51 (char: 3)
v46 = 57 (char: 9)
v47 = 101 (char: e)
v48 = 100 (char: d)
v49 = 55 (char: 7)
v50 = 50 (char: 2)
v51 = 101 (char: e)
v52 = 53 (char: 5)
v53 = 97 (char: a)
v54 = 49 (char: 1)
v55 = 50 (char: 2)
v56 = 48 (char: 0)
v57 = 48 (char: 0)
v58 = 99 (char: c)
v59 = 53 (char: 5)
v60 = 97 (char: a)
v61 = 99 (char: c)
v62 = 57 (char: 9)
v63 = 54 (char: 6)
v64 = 50 (char: 2)
v65 = 48 (char: 0)
v66 = 48 (char: 0)
v67 = 125 (char: })
```