---
title: ImaginaryCTF - Questionably-Distributed
description: First blood writeup for questionably-distributed reverse engineering challenge using IDA Pro MCP server
date: 2025-09-30
tldr: A distributed system challenge where multiple servers obfuscate a simple transformation. We first blooded this hard difficulty challenge using the IDA Pro MCP server for automated analysis. The flag is hidden in the main diagonal of a 28×28 lookup table after applying XOR and arithmetic operations.
draft: false
tags: ["ctf", "writeup", "reverse-engineering", "ida-pro", "mcp", "first-blood"]
---

# Questionably-Distributed Challenge Writeup

## Challenge Overview

**Challenge Name:** questionably-distributed
**Category:** Reverse Engineering
**Difficulty:** Hard
**Points:** 418pts
**Solves:** 32
**Author:** Minerva-007
**Flag:** `ictf{51ngl3_4PP_mul71pl3_53rv1c35}`
**Achievement:** 🩸 **First Blood!**

### Challenge Description

> Distributed computing is the new tech on the horizon, so I decided to distribute the computer itself. Enjoy this horrendous simulation of a CPU running locally over TCP. This is not a steg chall.

## The MCP Server Advantage

For this challenge, I used the **IDA Pro MCP (Model Context Protocol) server** to automate the reverse engineering process. MCP is a protocol that allows AI assistants to interact with tools like IDA Pro programmatically, enabling automated analysis, renaming, and annotation of binaries.

The IDA Pro MCP server implementation I used is based on the work described in [Wil Gibbs' blog post about DEF CON Finals MCP](https://wilgibbs.com/blog/defcon-finals-mcp/). This approach allowed me to:

- Automatically retrieve function decompilations
- Programmatically rename variables and functions
- Add comments and annotations
- Extract data structures and binary data
- Perform complex analysis without manual clicking

This automated workflow was key to achieving **first blood** on this hard difficulty challenge - the combination of AI-assisted analysis and MCP automation allowed for rapid understanding of the binary's behavior.

## Initial Analysis

The challenge provides a Windows executable `chal.exe`. Running the binary shows that it starts three different servers on different ports:
- Port 5555: Lookup table server
- Port 6666: Arithmetic server
- Port 7777: Comparison server

The binary appears to implement a distributed system with multiple services, hence the name "questionably-distributed". As the author notes, this is a "horrendous simulation of a CPU running locally over TCP" - essentially distributing basic CPU operations across network services.

## Reverse Engineering Process

### 1. Binary Structure Analysis

Using IDA Pro, I identified the main components:

- **`main_function` (0x140002256)**: The primary function that processes data
- **`lookup_table_server` (0x140001687)**: Serves values from a lookup table
- **`arithmetic_server` (0x140001a42)**: Performs arithmetic operations (+, -, ^)
- **`comparison_server` (0x140001e45)**: Performs comparison operations
- **Lookup table data (0x140010020)**: 784 bytes of data (28×28 grid)

### 2. Algorithm Discovery

The main function implements the following algorithm:

```c
for (k = 0; k < 28; k++) {
    for (m = 0; m < 28; m++) {
        // 1. Get lookup table value
        send_to_port_5555("k m");
        value = receive_response();
        
        // 2. XOR with 66
        send_to_port_6666("value 66 ^");
        xor_result = receive_response();
        
        // 3. Add 15
        send_to_port_6666("xor_result 15 +");
        final_result = receive_response();
        
        // Result is discarded (not stored anywhere visible)
    }
}
```

This translates to the transformation: `(lookup_table[k][m] ^ 66) + 15`

### 3. Arithmetic Server Analysis

The arithmetic server supports three operations:
- `+` (addition)
- `-` (subtraction) 
- `^` (XOR)

Commands are sent in the format: `"operand1 operand2 operator"`

### 4. Flag Extraction Strategy

Since the main function processes all 784 values but doesn't store them, I suspected the flag was hidden in the lookup table data itself. The key insight was that the data forms a 28×28 grid, and flags are often hidden in geometric patterns.

## Solution

### Step 1: Extract Lookup Table Data

From IDA Pro, I extracted the 784 bytes of lookup table data starting at address 0x140010020.

### Step 2: Apply Transformation

Applied the discovered transformation to each byte:
```python
transformed_value = ((original_value ^ 66) + 15) & 0xFF
```

### Step 3: Convert to Grid and Extract Patterns

Arranged the transformed values into a 28×28 grid and tested various reading patterns:
- Row-wise reading
- Column-wise reading
- Main diagonal
- Anti-diagonal
- Border patterns
- Spiral patterns

### Step 4: Flag Discovery

The flag was found in the **main diagonal** of the transformed grid:

```
Position (0,0): '5'
Position (1,1): '1' 
Position (2,2): 'n'
Position (3,3): 'g'
...
Position (27,27): '5'
```

Reading the main diagonal gives: `51ngl3_4PP_mul71pl3_53rv1c35`

## Complete Solve Script

```python
# Extract lookup table data from IDA Pro
# Address: 0x140010020, Size: 784 bytes

def solve_questionably_distributed(lookup_table_data):
    """
    Solve the questionably-distributed challenge
    
    Args:
        lookup_table_data: 784 bytes of data from address 0x140010020
    
    Returns:
        The flag string
    """
    # Apply transformation: (value ^ 66) + 15
    transformed = []
    for byte in lookup_table_data:
        transformed_value = ((byte ^ 66) + 15) & 0xFF
        transformed.append(transformed_value)
    
    # Arrange into 28x28 grid
    grid = []
    for i in range(28):
        row = transformed[i*28:(i+1)*28]
        grid.append(row)
    
    # Extract main diagonal
    flag_chars = []
    for i in range(28):
        char_code = grid[i][i]
        if 32 <= char_code <= 126:  # Printable ASCII
            flag_chars.append(chr(char_code))
    
    flag_content = ''.join(flag_chars)
    return f"ictf{{{flag_content}}}"

# Usage:
# 1. Extract 784 bytes from IDA Pro at address 0x140010020
# 2. Pass to solve_questionably_distributed()
# 3. Get flag: ictf{51ngl3_4PP_mul71pl3_53rv1c35}
```

## Key Insights

1. **Distributed Architecture**: The challenge uses multiple servers to obfuscate a simple transformation
2. **Data Hiding**: The flag was embedded in the lookup table data, not in the algorithm logic
3. **Geometric Pattern**: Using the main diagonal of a square grid is a classic steganography technique
4. **Leetspeak**: The flag content uses number substitutions (5→s, 1→i, 3→e, etc.)

## Tools Used

- **IDA Pro**: For reverse engineering and static analysis
- **IDA Pro MCP Server**: For automated binary analysis and AI-assisted reverse engineering ([Wil Gibbs' implementation](https://wilgibbs.com/blog/defcon-finals-mcp/))
- **Python**: For data extraction and transformation
- **Claude AI with MCP**: For orchestrating the automated analysis workflow

## Technical Summary

The challenge demonstrates several reverse engineering concepts:

1. **Multi-service Architecture**: Understanding how distributed systems can obfuscate simple operations
2. **Data Transformation**: Recognizing XOR and arithmetic operations in assembly
3. **Steganography**: Finding hidden data in geometric patterns within data structures
4. **Static Analysis**: Extracting and analyzing binary data without dynamic execution
5. **Automated RE**: Using MCP to programmatically interact with IDA Pro for faster analysis

The transformation `(value ^ 66) + 15` was discovered by analyzing the network communication pattern in the main function, where each lookup table value is processed through the arithmetic server.

## Why This Approach Worked for First Blood

The combination of:
1. **MCP automation** - No manual clicking through IDA Pro
2. **AI-assisted analysis** - Pattern recognition and hypothesis generation
3. **Rapid iteration** - Quick testing of different extraction patterns
4. **Programmatic data extraction** - Automated extraction of the 784-byte lookup table

...allowed me to analyze the binary, understand the algorithm, extract the data, and test multiple geometric patterns in a fraction of the time traditional manual analysis would take. This is the power of combining AI with proper tooling infrastructure.

## Final Flag

`ictf{51ngl3_4PP_mul71pl3_53rv1c35}`

The flag content "51ngl3_4PP_mul71pl3_53rv1c35" appears to be leetspeak for "single app multiple services", which perfectly describes the challenge's distributed architecture concept.

---

**Note:** Despite the challenge description saying "This is not a steg chall", the flag extraction did involve finding a geometric pattern (main diagonal) in the data - though the real challenge was understanding the distributed architecture and transformation algorithm first!

