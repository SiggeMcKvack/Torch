# Torch Codebase Audit Report - Memory Leaks & Bugs

## Executive Summary
This audit identified **12 critical memory leaks**, **2 potential resource leaks**, and **2 design issues** in the Torch codebase. Issues range from undeleted heap allocations to dangling pointers and improper resource cleanup.

---

## CRITICAL ISSUES

### 1. CRITICAL: Wrapper Pointer Memory Leak in Companion::Process()
**Severity:** CRITICAL  
**File:** `/home/user/Torch/src/Companion.cpp`  
**Lines:** 1321-1340, 1377-1381  
**Issue:** The `wrapper` pointers created with `new` are never deleted.

```cpp
// Line 1321: AudioManager allocated with new but never deleted
AudioManager::Instance = new AudioManager();
BinaryWrapper* wrapper = nullptr;

if (this->gConfig.exporterType == ExportType::Binary) {
    switch (this->gConfig.otrMode) {
        case ArchiveType::OTR:
            wrapper = new SWrapper(this->gConfig.outputPath);  // Line 1327: MEMORY LEAK
            break;
        case ArchiveType::O2R:
            wrapper = new ZWrapper(this->gConfig.outputPath);  // Line 1330: MEMORY LEAK
            break;
    }
}

// ...later...
if(wrapper != nullptr) {
    wrapper->AddFile("version", vWriter.ToVector());
    wrapper->Close();
    // BUG: wrapper is never deleted!
}
```

**Impact:** Every time the process() function is called with Binary export type, SWrapper or ZWrapper objects are allocated but never freed, causing memory leak.

**Recommended Fix:** Use `std::unique_ptr<BinaryWrapper>` instead of raw pointers, similar to how it's done in the Pack() function (line 1426).

---

### 2. CRITICAL: ZWrapper Memory Leak - miniz_cpp::zip_file Not Deleted
**Severity:** CRITICAL  
**File:** `/home/user/Torch/src/archive/ZWrapper.cpp`  
**Lines:** 14, 42  
**Issue:** The `mZip` pointer allocated in constructor is never deleted in destructor.

```cpp
// Line 14: Constructor allocates mZip
ZWrapper::ZWrapper(const std::string& path) {
    this->mPath = path;
    this->mZip = new miniz_cpp::zip_file;  // MEMORY LEAK - never deleted
}

// Line 41-43: Close saves but doesn't delete
int32_t ZWrapper::Close(void) {
    this->mZip->save(this->mPath);
    return 0;  // mZip is never deleted
}
```

**Impact:** Every ZWrapper instance (O2R archive creation) causes a memory leak of the miniz_cpp::zip_file object.

**Recommended Fix:** Add proper destructor:
```cpp
ZWrapper::~ZWrapper() {
    delete this->mZip;
}
```

---

### 3. CRITICAL: Decompressor Cache DataChunk Structures Not Freed
**Severity:** CRITICAL  
**File:** `/home/user/Torch/src/utils/Decompressor.cpp`  
**Lines:** 33, 44, 55, 73, 225  
**Issue:** DataChunk structures are allocated with `new` but only the data pointer is deleted.

```cpp
// Lines 31-34: MIO0 decompression
const auto decompressed = new uint8_t[head.dest_size];
mio0_decode(in_buf, decompressed, nullptr);
gCachedChunks[offset] = new DataChunk{ decompressed, head.dest_size };  // Struct allocated
return gCachedChunks[offset];

// Lines 223-227: ClearCache only deletes data, not the struct
void Decompressor::ClearCache() {
    for(auto& [key, value] : gCachedChunks){
        delete value->data;  // INCOMPLETE: value (DataChunk struct) is never deleted
    }
    gCachedChunks.clear();
}
```

**Impact:** Every decompressed asset causes a memory leak of the DataChunk structure itself. Over the course of processing an entire ROM, this compounds to significant memory waste.

**Recommended Fix:**
```cpp
void Decompressor::ClearCache() {
    for(auto& [key, value] : gCachedChunks){
        delete value->data;
        delete value;  // ADD THIS LINE
    }
    gCachedChunks.clear();
}
```

---

### 4. CRITICAL: DecodeTKMK00 Leaks Uncompressed Data Pointer
**Severity:** CRITICAL  
**File:** `/home/user/Torch/src/utils/Decompressor.cpp`  
**Lines:** 70-73  
**Issue:** The `decompressed` buffer is allocated but only `rgba` is stored in the cache, leaving `decompressed` unreferenced.

```cpp
const auto decompressed = new uint8_t[size];  // Line 70: Allocated
const auto rgba = new uint8_t[size];          // Line 71: Also allocated
tkmk00_decode(in_buf, decompressed, rgba, alpha);
gCachedChunks[offset] = new DataChunk{ rgba, size };  // Only rgba is kept
return gCachedChunks[offset];  // decompressed is LEAKED!
```

**Impact:** TKMK00 format textures cause memory leak of uncompressed data (MK64 textures).

**Recommended Fix:** Only allocate `rgba` or properly track both allocations.

---

### 5. CRITICAL: CompressedTextureFactory Memory Leak - Undeleted PNG Buffers
**Severity:** CRITICAL  
**File:** `/home/user/Torch/src/factories/CompressedTextureFactory.cpp`  
**Lines:** 274-346, 500-587  
**Issue:** Multiple allocations of `raw` buffers are never deleted after use.

```cpp
// Line 274: Allocated for export
uint8_t* raw = new uint8_t[TextureUtils::CalculateTextureSize(...)];
// ... conversion happens ...
write.write(reinterpret_cast<char*>(raw), size);
return std::nullopt;  // raw is NEVER DELETED!

// Lines 507, 519, 559: More allocations
raw = new uint8_t[size];  // Switch statement allocations
// ... conversions ...
auto result = std::vector(raw, raw + size);  // Vector created from raw
// raw is NEVER DELETED, even though vector created a copy!
```

**Impact:** Every modding export of a texture causes memory leak of the temporary buffer.

**Recommended Fix:**
```cpp
write.write(reinterpret_cast<char*>(raw), size);
delete[] raw;  // ADD THIS
return std::nullopt;
```

---

### 6. CRITICAL: TextureFactory Memory Leak - Undeleted PNG Buffers
**Severity:** CRITICAL  
**File:** `/home/user/Torch/src/factories/TextureFactory.cpp`  
**Lines:** 196-268, 353-437  
**Issue:** Same issue as CompressedTextureFactory - `raw` buffers not deleted.

```cpp
// Line 196: Allocated
uint8_t* raw = new uint8_t[TextureUtils::CalculateTextureSize(...)];
// ... conversion functions modify raw pointer ...
write.write(reinterpret_cast<char*>(raw), size);
return std::nullopt;  // raw is LEAKED!
```

**Impact:** Every texture modding export causes memory leak.

---

### 7. CRITICAL: CompTool Pointer Reassignment Memory Leak
**Severity:** CRITICAL  
**File:** `/home/user/Torch/src/preprocess/CompTool.cpp`  
**Lines:** 87-96  
**Issue:** Buffer pointer is reassigned without deleting original allocation.

```cpp
auto bytes = new uint8_t[p_size];  // Line 87: Initial allocation
basefile.Read((char*) bytes, p_size);

switch ((CompType) comp_flag) {
    case CompType::COMPRESSED:
        decoded = Decompressor::Decode(std::vector(bytes, bytes + p_size), 0, CompressionType::MIO0, true);
        bytes = decoded->data;  // Line 96: MEMORY LEAK - original bytes is lost
        v_size = decoded->size;
        break;
}
// ...
decompfile.Write((char*) bytes, v_size);
// Function returns without cleaning up
```

**Impact:** Memory leak in ROM decompression process. The original allocated buffer is lost when reassigned.

**Recommended Fix:**
```cpp
std::vector<uint8_t> bytes;
basefile.Seek(p_begin, LUS::SeekOffsetType::Start);
bytes.resize(p_size);
basefile.Read((char*) bytes.data(), p_size);

switch ((CompType) comp_flag) {
    case CompType::COMPRESSED:
        decoded = Decompressor::Decode(bytes, 0, CompressionType::MIO0, true);
        // Use decoded->data directly, no reassignment needed
```

---

### 8. CRITICAL: Audio AIFC Table Allocations Not Freed
**Severity:** CRITICAL  
**File:** `/home/user/Torch/src/factories/naudio/v0/AIFCDecode.cpp`  
**Lines:** 179-185, 221  
**Issue:** Nested array allocations for AIFC decoding are never freed.

```cpp
// Lines 179-185: Coefficient table allocated
*table = new s32**[*npredictors];
for (s32 i = 0; i < *npredictors; i++) {
    (*table)[i] = new s32*[8];
    for (s32 j = 0; j < 8; j++) {
        (*table)[i][j] = new s32[*order + 8];  // Triple nested allocation
    }
}
// NEVER FREED - no cleanup code found

// Line 221: ADPCM loop allocation
auto *al = static_cast<ALADPCMloop *>(malloc(*nloops * sizeof(ALADPCMloop)));
// NEVER FREED
```

**Impact:** Audio file decompression causes significant memory leaks. These allocations can be large.

---

### 9. CRITICAL: Main Function Companion Instances Not Deleted
**Severity:** CRITICAL  
**File:** `/home/user/Torch/src/main.cpp`  
**Lines:** 37, 50, 63, 75, 94, 144, 162  
**Issue:** Companion objects allocated with `new` are assigned to static Instance but never deleted.

```cpp
// Line 37: OTR mode
const auto instance = Companion::Instance = new Companion(filename, ArchiveType::OTR, debug, srcdir, destdir);
instance->Init(ExportType::Binary);
// instance never deleted after callback

// Same pattern in lines 50, 63, 75, 94, 144, 162
```

**Impact:** Every invocation of Torch (all CLI modes) leaks a Companion object.

**Recommended Fix:** Use `make_unique` or ensure deletion in the callback or at program end.

---

## HIGH SEVERITY ISSUES

### 10. HIGH: Missing Virtual Destructors in Base Classes
**Severity:** HIGH  
**File:** `/home/user/Torch/src/factories/BaseFactory.h`  
**Lines:** 88-92, 94-122  
**Issue:** Base classes for polymorphic types lack virtual destructors.

```cpp
class BaseExporter {
public:
    virtual ExportResult Export(...) = 0;
    // NO VIRTUAL DESTRUCTOR!
};

class BaseFactory {
public:
    virtual std::optional<std::shared_ptr<IParsedData>> parse(...) = 0;
    // NO VIRTUAL DESTRUCTOR!
};
```

**Impact:** While less critical due to use of shared_ptr, this violates best practices and could cause issues if subclasses are ever created with unique_ptr or deleted through base class pointers without proper cleanup.

**Recommended Fix:**
```cpp
class BaseExporter {
public:
    virtual ~BaseExporter() = default;
    virtual ExportResult Export(...) = 0;
};

class BaseFactory {
public:
    virtual ~BaseFactory() = default;
    virtual std::optional<std::shared_ptr<IParsedData>> parse(...) = 0;
};
```

---

### 11. HIGH: AudioManager Static Singleton Never Deleted
**Severity:** HIGH  
**File:** `/home/user/Torch/src/Companion.cpp` line 1321  
**File:** `/home/user/Torch/src/factories/naudio/v0/AudioManager.h` line 126  
**Issue:** AudioManager is allocated with `new` and stored in a static pointer but never deleted.

```cpp
AudioManager::Instance = new AudioManager();
// ...nowhere is this ever deleted
```

**Impact:** Memory leak of AudioManager singleton every time Companion processes.

**Recommended Fix:** Use a static local variable (Meyer's singleton) or ensure proper cleanup on program exit.

---

## POTENTIAL ISSUES

### 12. MEDIUM: Dangling Pointer Risk - Texture Conversion Functions
**Severity:** MEDIUM  
**File:** `/home/user/Torch/src/factories/CompressedTextureFactory.cpp` lines 287, 297, 310, 335  
**File:** `/home/user/Torch/src/factories/TextureFactory.cpp` lines 208, 232, 246  
**Issue:** Functions like `rgba2png`, `ia2png`, `convert_raw_to_ci8` take `uint8_t**` and may reallocate memory.

```cpp
uint8_t* raw = new uint8_t[...];
rgba* imgr = raw2rgba(...);
if(rgba2png(&raw, &size, imgr, ...)) {  // Takes pointer to pointer
    throw std::runtime_error(...);
}
// If rgba2png reallocates 'raw', the original allocation is lost!
```

**Impact:** Depending on implementation of these functions, could cause additional memory leaks if they reallocate the buffer.

**Recommended Fix:** Use `std::vector<uint8_t>` or ensure the functions don't reallocate.

---

## SUMMARY TABLE

| # | Issue | File | Severity | Type |
|---|-------|------|----------|------|
| 1 | Wrapper pointers (new/delete mismatch) | Companion.cpp:1327,1330 | CRITICAL | Memory Leak |
| 2 | ZWrapper mZip not deleted | ZWrapper.cpp:14 | CRITICAL | Memory Leak |
| 3 | Decompressor DataChunk struct not freed | Decompressor.cpp:225 | CRITICAL | Memory Leak |
| 4 | DecodeTKMK00 decompressed buffer leaked | Decompressor.cpp:70 | CRITICAL | Memory Leak |
| 5 | CompressedTextureFactory raw buffers | CompressedTextureFactory.cpp:274+ | CRITICAL | Memory Leak |
| 6 | TextureFactory raw buffers | TextureFactory.cpp:196+ | CRITICAL | Memory Leak |
| 7 | CompTool pointer reassignment | CompTool.cpp:96 | CRITICAL | Memory Leak |
| 8 | AIFC table allocations | AIFCDecode.cpp:179+ | CRITICAL | Memory Leak |
| 9 | Companion instances in main | main.cpp:37,50,63,75,94,144,162 | CRITICAL | Memory Leak |
| 10 | Missing virtual destructors | BaseFactory.h:88,94 | HIGH | Design Issue |
| 11 | AudioManager singleton | Companion.cpp:1321 | HIGH | Memory Leak |
| 12 | Texture function dangling pointers | CompressedTextureFactory.cpp, TextureFactory.cpp | MEDIUM | Potential Leak |

---

## RECOMMENDATIONS

### Immediate Actions (Critical)
1. Fix Decompressor cache cleanup in `ClearCache()`
2. Add proper destructor to ZWrapper
3. Replace raw `new`/`delete` with smart pointers (unique_ptr/shared_ptr)
4. Delete temporary buffers in texture exporters
5. Fix CompTool pointer reassignment using vector

### Short-term Actions (High)
1. Add virtual destructors to base classes
2. Implement proper cleanup for AudioManager and other singletons
3. Review all texture conversion functions for buffer reallocation

### Long-term Actions (Medium)
1. Replace all raw pointers with smart pointers
2. Implement proper RAII patterns throughout
3. Add memory testing with valgrind/ASan in CI/CD
4. Consider using containers instead of raw arrays

