---
name: cyberchef
description: Multi-layer payload deobfuscation, cryptographic decoding, heuristic recipe detection (magic), and entropy analysis via CyberChef MCP.
---

# CyberChef MCP Tooling Playbook

Official resources:
- https://github.com/gchq/CyberChef
- https://github.com/noor202401938-netizen/cyber-chef-mcp
- https://gchq.github.io/CyberChef/
- https://modelcontextprotocol.io

CyberChef provides over 500 data transformation and cryptographic operations. Connected via the Model Context Protocol (MCP) server `cyberchef`, it enables Strix agents to autonomously analyze, deobfuscate, unpack, and verify encoded exploit payloads, authorization tokens, and obfuscated attack vectors without manual intervention or guessing.

## Canonical MCP Tool Names & Signatures

When the `cyberchef` MCP connection is active, the following tools are available in the agent registry:

- `cyberchef_magic(input: string)`: Run heuristic detection across known encodings, ciphers, and hash formats. Returns recommended deobfuscation recipes and confidence scores.
- `cyberchef_bake(input: string, recipe: [{ op: string, args?: any[] }])`: Execute sequential operation chains (e.g. `[{"op": "From Base64"}, {"op": "URL Decode"}]`).
- `cyberchef_from_base64(input: string, urlSafe?: boolean)`: Decode standard or URL-safe Base64 strings.
- `cyberchef_to_base64(input: string, urlSafe?: boolean)`: Encode plaintext into Base64 / URL-safe Base64.
- `cyberchef_from_hex(input: string, delimiter?: "None"|"Space"|"0x"|"Comma")`: Convert hexadecimal sequences to text.
- `cyberchef_url_decode(input: string)`: Decode single or multi-round percent-encoded parameters.
- `cyberchef_rot13(input: string, amount?: number)`: Rotate characters by offset (default 13, Caesar cipher support).
- `cyberchef_xor(input: string, key: string, keyFormat?: "UTF8"|"Hex")`: Decrypt or apply bitwise XOR with secret key.
- `cyberchef_jwt_decode(token: string)`: Parse and inspect header, claims, alg, and signatures of JSON Web Tokens.
- `cyberchef_entropy(input: string)`: Calculate Shannon entropy (0.0 to 8.0) to distinguish plaintext, compressed data, and encrypted or packed payloads.
- `cyberchef_defang_url(url: string)`: Defang malicious or suspicious indicators (`hxxps://target[.]com`) for safe reporting.
- `cyberchef_extract_entities(text: string)`: Extract URLs, IP addresses, and email addresses from raw logs or memory strings.

## Agent-Safe Baseline for Automation

1. **Heuristic First**:
   Always run `cyberchef_magic` on unknown high-entropy or encoded strings before guessing transformations:
   ```json
   {
     "tool": "cyberchef_magic",
     "arguments": { "input": "ZXlKaGJHY2lPaUpTVXpVbkxh..." }
   }
   ```

2. **Sequential Multi-Layer Deobfuscation (Bake)**:
   For payloads with layered obfuscation (e.g. Hex inside Base64 inside URL-encoded query params):
   ```json
   {
     "tool": "cyberchef_bake",
     "arguments": {
       "input": "%34%38%36%35%36%63%36%63%36%66",
       "recipe": [
         { "op": "URL Decode" },
         { "op": "From Hex", "args": ["None"] }
       ]
     }
   }
   ```

3. **High-Entropy Verification**:
   Before analyzing suspicious parameters, assess randomness and encryption depth:
   ```json
   {
     "tool": "cyberchef_entropy",
     "arguments": { "input": "01a2fe89cb994821a0d8e4..." }
   }
   ```
   - **Entropy < 4.0**: Plain English text, uncompressed source code, or structured JSON/XML.
   - **Entropy 4.0 - 6.5**: Encoded payloads (Base64, Hex) or compressed data.
   - **Entropy > 7.0**: Strong encryption, cryptographic hashes, or packed binary shellcode.

## Common Security Analysis Patterns

### Pattern 1: Nested WAF Bypass / Obfuscated Injection Vector
When target web applications accept encoded input in parameters or cookies:
1. Extract candidate parameter from HTTP request or response.
2. Call `cyberchef_magic` to determine layers.
3. Call `cyberchef_bake` with the suggested pipeline to recover the plaintext injection string.
4. Verify whether the underlying query contains unsanitized SQLi (`UNION SELECT`), XSS, or SSRF vectors.

### Pattern 2: JWT Security Inspection
When encountering `Authorization: Bearer <token>` or session tokens:
1. Call `cyberchef_jwt_decode(token)`.
2. Inspect the header: check for `alg: "none"`, `alg: "HS256"` with potential asymmetric public key confusion, or empty signatures.
3. Inspect claims: verify expiry timestamps (`exp`), issuer (`iss`), role/privilege elevations, and user identities.

### Pattern 3: XOR Obfuscation Recovery
When inspecting hardcoded binary strings, PowerShell scripts, or obfuscated malware droppers:
1. Identify probable key length or common plaintext prefix (e.g., `MZ`, `http`, `function`).
2. Run `cyberchef_xor` iterating candidate keys to extract underlying C2 endpoints or script payloads.

## Critical Correctness Rules

- **Do Not Guess Encodings**: If a string contains `=, %, 0x` or unexpected symbols, run `cyberchef_magic` first rather than blindly applying base64 or URL decoding.
- **Preserve Raw Inputs**: Keep the original obfuscated string in agent memory/notes alongside the decoded output for accurate proof-of-concept (PoC) reporting.
- **Fail-Safe Fallback**: If an operation fails during `cyberchef_bake`, isolate the failing recipe step and execute individual tools (`cyberchef_from_base64`, `cyberchef_url_decode`) sequentially.
- **Safe Defanging**: Always run `cyberchef_defang_url` on confirmed malicious or C2 URLs before writing final markdown reports.

## Failure Recovery

- If `cyberchef_from_base64` throws a padding error, retry with `urlSafe: true` or inspect whether characters are URL percent-encoded first.
- If `cyberchef_from_hex` produces unprintable garbage characters, check if the input is big-endian or uses custom delimiters (`0x`, `Space`, `,`).
- If `cyberchef_bake` returns an error, use `cyberchef_help(query: "<operation>")` to verify supported operation names and argument formats.

If uncertain, query web_search with:
`site:gchq.github.io/CyberChef cyberchef <operation_name>`
