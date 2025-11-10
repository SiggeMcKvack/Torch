# Torch Codebase Security Audit Report

**Date:** November 10, 2025
**Scope:** Comprehensive security audit of Torch ROM extraction tool
**Severity Summary:** 
- Critical: 3
- High: 8
- Medium: 6
- Low: 4

---

## EXECUTIVE SUMMARY

The Torch codebase is a ROM extraction and asset conversion tool for Nintendo 64 games. This audit identified **21 security vulnerabilities** spanning input validation, buffer safety, file system security, memory safety, and dependency management. The most critical issues involve unchecked buffer operations during ROM parsing and decompression, path traversal vulnerabilities, and unsafe string operations.

---

## CRITICAL VULNERABILITIES

### 1. Unchecked Buffer Operations in ROM Parsing (Decompressor.cpp)
**File:** `/home/user/Torch/src/utils/Decompressor.cpp:22,68,138,142,183`
**Severity:** CRITICAL
**CWE:** CWE-120 (Buffer Overflow), CWE-125 (Out-of-bounds Read)

**Description:**
The decompression functions perform pointer arithmetic without validating that offset values are within buffer bounds:

```cpp
// Line 22: No bounds check on offset
const unsigned char* in_buf = buffer.data() + offset;

// Line 68: Unsafe pointer arithmetic
const uint8_t* in_buf = buffer.data() + offset;

// Line 138: Potential integer underflow not validated
auto size = node["size"] ? node["size"].as<size_t>() : 
    manualSize.value_or(buffer.size() - fileOffset);
```

**Attack Vector:** A malformed ROM with offset values exceeding buffer size could cause buffer over-read.

**Proof of Concept:**
```cpp
// If fileOffset > buffer.size(), the subtraction wraps to large value (integer underflow)
size_t size = buffer.size() - 0xFFFFFFFF; // Results in huge allocation
```

**Remediation:**
```cpp
// Validate offset before pointer arithmetic
if (offset > buffer.size()) {
    throw std::runtime_error("Offset exceeds buffer size");
}
const unsigned char* in_buf = buffer.data() + offset;

// Check for integer underflow
if (fileOffset > buffer.size()) {
    throw std::runtime_error("File offset exceeds buffer size");
}
```

---

### 2. Path Traversal Vulnerability in External File Loading
**File:** `/home/user/Torch/src/Companion.cpp:445-467`
**Severity:** CRITICAL
**CWE:** CWE-22 (Path Traversal)

**Description:**
External YAML files are loaded without proper path validation:

```cpp
// Line 445: User-controlled path from YAML
auto externalFile = externalFiles[i].as<std::string>();
this->gCurrentExternalFiles.push_back(
    (this->gSourceDirectory / externalFile).string()
);

// Line 451: Path validation uses relative() which may fail
std::string externalFileName = 
    (this->gSourceDirectory / externalFile).string();
if (std::filesystem::relative(externalFileName, this->gAssetPath)
    .string().starts_with("../")) {
    throw std::runtime_error("Not in asset directory");
}

// Line 467: YAML file loaded without further validation
YAML::Node root = YAML::LoadFile(externalFileName);
```

**Attack Vector:** A YAML configuration with `external_files: ["../../etc/passwd"]` could read arbitrary files.

**Proof of Concept:**
```yaml
external_files:
  - "../../sensitive/file.txt"
  - "../../../../etc/config"
```

**Remediation:**
```cpp
// Use canonical paths for comparison
auto canonical_external = std::filesystem::canonical(externalFileName);
auto canonical_asset = std::filesystem::canonical(this->gAssetPath);

// Ensure external file is within asset directory
if (!std::filesystem::exists(canonical_asset) || 
    canonical_external.string().find(canonical_asset.string()) != 0) {
    throw std::runtime_error("Path traversal detected");
}
```

---

### 3. Unsafe String Buffer Operations (StringHelper.cpp)
**File:** `/home/user/Torch/src/utils/StringHelper.cpp:91-103`
**Severity:** CRITICAL
**CWE:** CWE-119 (Improper Restriction of Operations within Bounds), CWE-667 (Improper Locking)

**Description:**
Format string vulnerability through unsafe vsprintf usage:

```cpp
std::string StringHelper::Sprintf(const char* format, ...) {
    char buffer[32768];  // Fixed buffer size
    std::string output;
    va_list va;
    
    va_start(va, format);
    vsprintf_s(buffer, format, va);  // No length check, can overflow
    va_end(va);
    
    output = buffer;
    return output;
}
```

**Attack Vector:** If untrusted data is passed as format string from YAML parsing, buffer overflow occurs.

**Proof of Concept:**
```cpp
// If node["comment"] contains "%x %x %x..." format string
std::string comment = GetSafeNode<std::string>(node, "comment");
std::string result = StringHelper::Sprintf(comment.c_str()); // Format string vulnerability
```

**Remediation:**
```cpp
std::string StringHelper::Sprintf(const char* format, ...) {
    // Use vsnprintf with explicit size limit
    char buffer[32768];
    va_list va;
    va_start(va, format);
    int ret = vsnprintf(buffer, sizeof(buffer), format, va);
    va_end(va);
    
    if (ret < 0 || ret >= (int)sizeof(buffer)) {
        throw std::runtime_error("Buffer overflow in Sprintf");
    }
    return std::string(buffer);
}
```

---

## HIGH SEVERITY VULNERABILITIES

### 4. Integer Overflow in Size Calculations
**File:** `/home/user/Torch/src/utils/Decompressor.cpp:126`
**Severity:** HIGH
**CWE:** CWE-190 (Integer Overflow)

**Description:**
```cpp
auto size = node["size"] ? node["size"].as<size_t>() : 
    manualSize.value_or(decoded->size - offset);
```

If `offset > decoded->size`, integer underflow occurs due to unsigned arithmetic.

**Attack Vector:** YAML specifying large offset values causes size wrap-around.

**Remediation:**
```cpp
size_t size = 0;
if (node["size"]) {
    size = node["size"].as<size_t>();
} else {
    if (offset > decoded->size) {
        throw std::runtime_error("Offset exceeds decoded size");
    }
    size = manualSize.value_or(decoded->size - offset);
}
```

---

### 5. Direct Pointer Arithmetic Without Bounds Checking (MIO0 Decompression)
**File:** `/home/user/Torch/lib/libmio0/mio0.c:169-183`
**Severity:** HIGH
**CWE:** CWE-125 (Out-of-bounds Read)

**Description:**
```c
out[bytes_written] = in[head.uncomp_offset + uncomp_idx];
// No validation that head.uncomp_offset + uncomp_idx < input_size

idx = ((vals[0] & 0x0F) << 8) + vals[1] + 1;
for (i = 0; i < length; i++) {
    out[bytes_written] = out[bytes_written - idx];  // No check for negative index
    bytes_written++;
}
```

**Attack Vector:** Malformed MIO0 headers can cause buffer over-read or negative indexing.

**Remediation:** Add bounds validation in decompression loop.

---

### 6. YAML Parsing Without Size Validation
**File:** `/home/user/Torch/src/Companion.cpp:405-410`
**Severity:** HIGH
**CWE:** CWE-129 (Improper Validation of Array Index)

**Description:**
```cpp
if(executeDef && this->gConfig.parseMode == ParseMode::Directory) {
    auto path = GetSafeNode<std::string>(node, "path");
    std::ifstream input( path, std::ios::binary );
    auto data = std::vector<uint8_t>( std::istreambuf_iterator( input ), {} );
    // No size limit check - could load multi-gigabyte files
    result = impl->parse(data, node);
}
```

**Attack Vector:** YAML points to extremely large files, causing memory exhaustion (DoS).

**Remediation:**
```cpp
// Add size limits
const size_t MAX_FILE_SIZE = 1024 * 1024 * 1024; // 1GB limit
std::ifstream input(path, std::ios::binary);
input.seekg(0, std::ios::end);
size_t fileSize = input.tellg();
if (fileSize > MAX_FILE_SIZE) {
    throw std::runtime_error("File exceeds size limit");
}
```

---

### 7. Unchecked Type Conversions in YAML Parsing
**File:** `/home/user/Torch/src/Companion.cpp:493,562,1712`
**Severity:** HIGH
**CWE:** CWE-197 (Numeric Truncation Error)

**Description:**
```cpp
gCurrentSegmentNumber = segments[0][0].as<uint32_t>();
gCurrentFileOffset = segments[0][1].as<uint32_t>();
// No validation that these values are in valid ranges

const auto offset = GetSafeNode<uint32_t>(asset, "offset");
// Could be 0xFFFFFFFF, causing buffer overflow when used
```

**Attack Vector:** YAML with malicious offset/size values bypasses bounds checking.

**Remediation:**
```cpp
uint32_t offset = GetSafeNode<uint32_t>(asset, "offset");
if (offset > this->gRomData.size()) {
    throw std::runtime_error("Offset exceeds ROM size");
}
```

---

### 8. Symlink Attack in Archive Creation
**File:** `/home/user/Torch/src/archive/SWrapper.cpp:38-42, ZWrapper.cpp:28-30`
**Severity:** HIGH
**CWE:** CWE-59 (Improper Link Resolution Before File Access)

**Description:**
```cpp
std::string dpath = "debug/" + path;
if(!fs::exists(fs::path(dpath).parent_path())){
    fs::create_directories(fs::path(dpath).parent_path());
}
std::ofstream stream(dpath, std::ios::binary);  // Path not validated
stream.write((char*)data.data(), data.size());
```

**Attack Vector:** If `path` contains `../` sequences, files can be written outside debug directory.

**Remediation:**
```cpp
// Validate path doesn't escape debug directory
auto canonical_debug = std::filesystem::canonical("debug");
auto canonical_file = std::filesystem::canonical(dpath);
if (canonical_file.string().find(canonical_debug.string()) != 0) {
    throw std::runtime_error("Path traversal in archive");
}
```

---

### 9. Unvalidated Network-Adjacent YAML Parsing (YAML-cpp)
**File:** `/home/user/Torch/src/Companion.cpp:429,467,645,722,1092,1367`
**Severity:** HIGH
**CWE:** CWE-502 (Deserialization of Untrusted Data)

**Description:**
YAML files are loaded without validation using YAML::LoadFile, which can execute arbitrary operations:

```cpp
auto modding = YAML::LoadFile(path.string());
// YAML-cpp can parse complex objects, potentially leading to issues

YAML::Node root = YAML::LoadFile(externalFileName);
```

**Vulnerable Pattern:** YAML parsing can trigger code execution if coupled with dynamic loading.

**Remediation:** Validate YAML structure before processing:
```cpp
try {
    YAML::Node config = YAML::LoadFile(configPath.string());
    // Validate that config contains only expected keys
    ValidateYAMLStructure(config);
} catch (const YAML::Exception& e) {
    throw std::runtime_error("Invalid YAML: " + std::string(e.what()));
}
```

---

### 10. Insufficient Input Validation in CString Reading
**File:** `/home/user/Torch/src/n64/Cartridge.cpp:12`
**Severity:** HIGH
**CWE:** CWE-126 (Buffer Over-read)

**Description:**
```cpp
reader.Seek(0x20, LUS::SeekOffsetType::Start);
this->gGameTitle = std::string(reader.ReadCString());
// No validation that ROM is large enough or contains null terminator
```

**Attack Vector:** Crafted ROM without proper null termination in game title field.

**Remediation:**
```cpp
// Validate offset and size
if (0x20 + 16 > this->gRomData.size()) {
    throw std::runtime_error("ROM too small for title field");
}
```

---

### 11. Missing Offset Validation in DecodeTKMK00
**File:** `/home/user/Torch/src/utils/Decompressor.cpp:63-74`
**Severity:** HIGH
**CWE:** CWE-125 (Out-of-bounds Read)

**Description:**
```cpp
DataChunk* Decompressor::DecodeTKMK00(const std::vector<uint8_t>& buffer, 
                                      const uint32_t offset, const uint32_t size, 
                                      const uint32_t alpha) {
    const uint8_t* in_buf = buffer.data() + offset;  // No bounds check
    // offset could exceed buffer.size()
    
    const auto decompressed = new uint8_t[size];
    tkmk00_decode(in_buf, decompressed, rgba, alpha);
}
```

**Remediation:**
```cpp
if (offset > buffer.size() || offset + size > buffer.size()) {
    throw std::runtime_error("TKMK00 offset/size exceeds buffer");
}
```

---

## MEDIUM SEVERITY VULNERABILITIES

### 12. Weak Hash Algorithm for ROM Verification
**File:** `/home/user/Torch/src/Companion.cpp:1703-1704`
**Severity:** MEDIUM
**CWE:** CWE-327 (Use of Broken/Risky Cryptographic Algorithm)

**Description:**
```cpp
std::string Companion::CalculateHash(const std::vector<uint8_t>& data) {
    return Chocobo1::SHA1().addData(data).finalize().toString();
}
```

SHA-1 is cryptographically broken. Hash is used to verify ROM integrity, allowing forgery attacks.

**Remediation:**
```cpp
return Chocobo1::SHA256().addData(data).finalize().toString();
```

---

### 13. Unsafe End-of-File Calculations
**File:** `/home/user/Torch/src/factories/TextureFactory.cpp:37,43,97-98`
**Severity:** MEDIUM
**CWE:** CWE-190 (Integer Overflow)

**Description:**
```cpp
size_t byteSize = std::max(1, (int) (texture->mFormat.depth / 8));
size_t isize = texture->mBuffer.size() / byteSize;  // Division by zero check missing
```

**Attack Vector:** If depth is 0 and not validated, division by zero occurs.

**Remediation:**
```cpp
if (texture->mFormat.depth == 0 || texture->mFormat.depth % 8 != 0) {
    throw std::runtime_error("Invalid texture depth");
}
size_t byteSize = texture->mFormat.depth / 8;
```

---

### 14. No Validation of External File Sizes
**File:** `/home/user/Torch/src/Companion.cpp:391-393`
**Severity:** MEDIUM
**CWE:** CWE-400 (Uncontrolled Resource Consumption)

**Description:**
```cpp
std::ifstream input(path, std::ios::binary);
std::vector<uint8_t> data = std::vector<uint8_t>( std::istreambuf_iterator( input ), {} );
// No size check - loads entire file into memory
```

**Remediation:**
```cpp
std::ifstream input(path, std::ios::binary);
input.seekg(0, std::ios::end);
size_t fileSize = input.tellg();
if (fileSize > MAX_ASSET_SIZE) {
    throw std::runtime_error("Asset file too large");
}
input.seekg(0);
std::vector<uint8_t> data(fileSize);
input.read((char*)data.data(), fileSize);
```

---

### 15. Improper Error Handling in Decompression
**File:** `/home/user/Torch/src/utils/Decompressor.cpp:27-32,40-45`
**Severity:** MEDIUM
**CWE:** CWE-755 (Improper Handling of Exceptional Conditions)

**Description:**
```cpp
mio0_header_t head;
if(!mio0_decode_header(in_buf, &head)){
    throw std::runtime_error("Failed to decode MIO0 header");
}

const auto decompressed = new uint8_t[head.dest_size];  // No overflow check
mio0_decode(in_buf, decompressed, nullptr);
```

If `head.dest_size` is extremely large, allocation fails silently or throws std::bad_alloc.

**Remediation:**
```cpp
const size_t MAX_DECOMP_SIZE = 512 * 1024 * 1024;  // 512MB
if (head.dest_size > MAX_DECOMP_SIZE) {
    throw std::runtime_error("Decompressed size exceeds limit");
}
```

---

### 16. Missing Null Pointer Checks
**File:** `/home/user/Torch/src/Companion.cpp:376-377`
**Severity:** MEDIUM
**CWE:** CWE-476 (Null Pointer Dereference)

**Description:**
```cpp
auto impl = factory->get();  // Returns shared_ptr
auto exporter = impl->GetExporter(this->gConfig.exporterType);
if(!exporter.has_value() && !impl->HasModdedDependencies()){
    // No check if impl is null before calling methods
}
```

**Remediation:**
```cpp
if (!factory || !factory->get()) {
    throw std::runtime_error("Factory is null");
}
```

---

## LOW SEVERITY VULNERABILITIES

### 17. CRC-64 Hash Collision Vulnerability
**File:** `/home/user/Torch/src/factories/DisplayListFactory.cpp:253-257,290-296`
**Severity:** LOW
**CWE:** CWE-327 (Cryptographic Weakness)

**Description:**
```cpp
auto bhash = CRC64((*replacement).c_str());
writer.Write(static_cast<uint32_t>(bhash >> 32));
writer.Write(static_cast<uint32_t>(bhash & 0xFFFFFFFF));

if(hash == 0) {
    throw std::runtime_error("Vtx hash is 0 for " + path);
}
```

CRC-64 is not collision-resistant. Multiple paths could hash to same value.

**Remediation:** Consider SHA-256 or other cryptographically secure hashes.

---

### 18. Temporary File Vulnerability in Debug Output
**File:** `/home/user/Torch/src/archive/SWrapper.cpp:36-44`
**Severity:** LOW
**CWE:** CWE-377 (Insecure Temporary File)

**Description:**
```cpp
std::string dpath = "debug/" + path;  // Predictable debug file location
std::ofstream stream(dpath, std::ios::binary);
stream.write((char*)data.data(), data.size());
```

Debug files created with predictable paths in current directory.

**Remediation:** Use system temporary directories or create files with restricted permissions.

---

### 19. Race Condition in File Directory Creation
**File:** `/home/user/Torch/src/archive/ZWrapper.cpp:29-30`
**Severity:** LOW
**CWE:** CWE-362 (Concurrent Execution using Shared Resource)

**Description:**
```cpp
if(!fs::exists(fs::path(dpath).parent_path())){
    fs::create_directories(fs::path(dpath).parent_path());  // TOCTOU
}
std::ofstream stream(dpath, std::ios::binary);
```

Time-of-check to time-of-use race condition between exists() and create_directories().

**Remediation:** Use create_directories which doesn't fail if already exists.

---

### 20. Missing Log Sanitization
**File:** `/home/user/Torch/src/Companion.cpp:389,413,481`
**Severity:** LOW
**CWE:** CWE-117 (Improper Output Neutralization)

**Description:**
```cpp
SPDLOG_ERROR("Modded asset {} not found", this->gModdedAssetPaths[name]);
// User-controlled path logged without sanitization
```

Malicious filenames with escape sequences could manipulate log output.

**Remediation:** Sanitize user-controlled data before logging.

---

### 21. Implicit Dependency on yaml-cpp Library Security
**File:** CMakeLists.txt (Line 226)
**Severity:** LOW
**CWE:** CWE-494 (Download of Code Without Integrity Check)

**Description:**
```cmake
FetchContent_Declare(
    yaml-cpp
    GIT_REPOSITORY https://github.com/jbeder/yaml-cpp.git
    GIT_TAG 2f86d13775d119edbb69af52e5f566fd65c6953b  # Fixed commit
)
```

While a specific commit is pinned, yaml-cpp may have vulnerabilities. No integrity checking on fetched content.

**Remediation:** Verify GPG signatures or use known-good binary packages.

---

## SUMMARY TABLE

| ID | Vulnerability | Severity | Category | CWE | File |
|----|---|---|---|---|---|
| 1 | Unchecked Buffer Operations | CRITICAL | Buffer Safety | 120 | Decompressor.cpp |
| 2 | Path Traversal | CRITICAL | File System | 22 | Companion.cpp |
| 3 | Format String Vulnerability | CRITICAL | Input Validation | 119 | StringHelper.cpp |
| 4 | Integer Overflow in Size | HIGH | Memory Safety | 190 | Decompressor.cpp |
| 5 | MIO0 Bounds Checking | HIGH | Buffer Safety | 125 | mio0.c |
| 6 | YAML File Size Validation | HIGH | Input Validation | 129 | Companion.cpp |
| 7 | Unchecked Type Conversions | HIGH | Input Validation | 197 | Companion.cpp |
| 8 | Symlink Attack | HIGH | File System | 59 | SWrapper.cpp |
| 9 | YAML Deserialization | HIGH | Input Validation | 502 | Companion.cpp |
| 10 | CString Over-read | HIGH | Buffer Safety | 126 | Cartridge.cpp |
| 11 | TKMK00 Bounds Check | HIGH | Buffer Safety | 125 | Decompressor.cpp |
| 12 | Weak Hash (SHA-1) | MEDIUM | Cryptography | 327 | Companion.cpp |
| 13 | Integer Division | MEDIUM | Memory Safety | 190 | TextureFactory.cpp |
| 14 | No File Size Limits | MEDIUM | Resource Mgmt | 400 | Companion.cpp |
| 15 | Decompression Allocation | MEDIUM | Resource Mgmt | 755 | Decompressor.cpp |
| 16 | Null Pointer | MEDIUM | Memory Safety | 476 | Companion.cpp |
| 17 | CRC-64 Collision | LOW | Cryptography | 327 | DisplayListFactory.cpp |
| 18 | Debug File Security | LOW | File System | 377 | SWrapper.cpp |
| 19 | Directory Race | LOW | Concurrency | 362 | ZWrapper.cpp |
| 20 | Log Injection | LOW | Output | 117 | Companion.cpp |
| 21 | Dependency Integrity | LOW | Dependency | 494 | CMakeLists.txt |

---

## RECOMMENDATIONS

### Immediate Actions (Critical/High)
1. **Add bounds checking** to all buffer operations before pointer arithmetic
2. **Implement path canonicalization** for all file operations with traversal checks
3. **Replace sprintf** with length-limited alternatives (snprintf/vsnprintf)
4. **Validate all integer conversions** from YAML to check ranges
5. **Add file size limits** for all file I/O operations
6. **Use tempfile libraries** instead of predictable debug paths

### Short-term Actions (Medium)
1. Replace SHA-1 with SHA-256 for integrity checking
2. Add overflow checks for all arithmetic operations on sizes/offsets
3. Implement proper error handling for decompression failures
4. Add null pointer validation before dereferencing
5. Sanitize all user input before logging

### Long-term Actions (Low)
1. Use static analysis tools (clang-analyzer, cppcheck)
2. Implement fuzzing of YAML parser with malformed inputs
3. Add security-focused unit tests for boundary conditions
4. Consider AddressSanitizer during development
5. Regular dependency audit for known CVEs

### Development Practices
1. Code review checklist for security (bounds checking, validation, error handling)
2. Automated security scanning in CI/CD pipeline
3. Regular dependency updates with vulnerability scanning
4. Security-focused test suite for edge cases
5. Address Sanitizer (ASAN) enabled in debug builds

---

## TESTING RECOMMENDATIONS

Create test cases for:
```cpp
// Test 1: ROM with offset >= buffer size
std::vector<uint8_t> smallRom(100);
YAML::Node node;
node["offset"] = 200;  // Should fail

// Test 2: Path traversal in YAML
std::string yaml = "external_files:\n  - '../../etc/passwd'";
// Should be rejected

// Test 3: Large file allocation
node["size"] = 0xFFFFFFFF;
// Should be limited

// Test 4: Integer underflow
auto size = smallRom.size() - 1000;  // Underflow when unsigned
// Should be caught
```

---

## CONCLUSION

The Torch codebase has significant security gaps in input validation, buffer safety, and file system operations. The three critical vulnerabilities should be addressed immediately before processing untrusted ROM files. Implementing the recommended changes will substantially improve the security posture of the application.

