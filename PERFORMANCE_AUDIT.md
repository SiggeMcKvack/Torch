# Torch Codebase Performance Audit

## Executive Summary
This audit identifies performance bottlenecks in the Torch codebase, focusing on inefficient algorithms, suboptimal data structure usage, memory management issues, and I/O performance. The analysis reveals multiple optimization opportunities across code-level operations, I/O patterns, and memory management.

---

## 1. CODE-LEVEL BOTTLENECKS

### 1.1 Linear Search Operations (O(n) complexity)

#### A. SearchTable() - Linear Table Search
**File:** `/home/user/Torch/src/Companion.cpp` (lines 1491-1499)
```cpp
std::optional<Table> Companion::SearchTable(uint32_t addr){
    for(auto& table : this->gTables){  // O(n) linear search
        if(addr >= table.start && addr <= table.end){
            return table;
        }
    }
    return std::nullopt;
}
```

**Issue:** Every asset processing performs a linear search through the entire gTables vector.
- Called frequently from DisplayListFactory.cpp:106, TextureFactory, and other factories
- Vector size can grow with config, but remains unsorted
- Repeated searches during batch processing

**Recommendation:** Use interval tree or sorted container with binary search
```cpp
// Option 1: Use std::lower_bound on sorted vector
std::optional<Table> Companion::SearchTable(uint32_t addr){
    auto it = std::lower_bound(this->gTables.begin(), this->gTables.end(), addr,
        [](const Table& t, uint32_t addr){ return t.end < addr; });
    if(it != this->gTables.end() && addr >= it->start && addr <= it->end)
        return *it;
    return std::nullopt;
}
```

**Impact:** Reduces from O(n) to O(log n), significant for large projects with many tables


#### B. GetNodesByType() - Full Map Iteration
**File:** `/home/user/Torch/src/Companion.cpp` (lines 1647-1668)
```cpp
std::optional<std::vector<std::tuple<std::string, YAML::Node>>> Companion::GetNodesByType(const std::string& type){
    std::vector<std::tuple<std::string, YAML::Node>> nodes;
    for(auto& [addr, tpl] : this->gAddrMap[this->gCurrentFile]){
        auto [name, node] = tpl;
        const auto n_type = GetTypeNode(node);  // Repeated YAML parsing
        if(n_type == type){
            nodes.push_back(tpl);  // Creates new tuples
        }
    }
    return nodes;  // RVO helps but still creates vector
}
```

**Issue:** 
- Iterates entire address map for each type query
- GetTypeNode(node) re-parses YAML node each call
- Called multiple times during processing (DisplayListFactory, etc.)
- Builds result vector even if result is small

**Recommendation:** Build index during parsing
```cpp
// In class header
std::unordered_map<std::string, std::vector<std::tuple<std::string, YAML::Node>>> gNodesByType;

// During ParseNode - index nodes by type
this->gNodesByType[type].emplace_back(name, node);
```

**Impact:** Changes from O(n) to O(1) lookups; saves repeated YAML parsing


#### C. std::find() in Linear Chains
**File:** `/home/user/Torch/src/Companion.cpp` (line 1260)
```cpp
if(std::find(entries.begin(), entries.end(), key) != entries.end()) {
    continue;
}
entries.push_back(key);  // Add if not found
```

**Issue:** O(n) search followed by push_back
- Called in loop during initialization (lines 1256-1266)
- Repeated for each factory type

**Recommendation:** Use unordered_set first, then convert
```cpp
std::unordered_set<std::string> seen(entries.begin(), entries.end());
for (auto& [key, _] : this->gFactories) {
    if(!seen.contains(key)) {  // O(1) average
        entries.push_back(key);
    }
}
```

**Impact:** O(n²) → O(n) algorithm complexity


#### D. std::find() in Audio Processing
**File:** `/home/user/Torch/src/factories/naudio/v0/AudioManager.cpp` (lines 246, 557)
```cpp
if(std::find(usedEnvOffsets.begin(), usedEnvOffsets.end(), addr) == usedEnvOffsets.end())
// and
if(std::find(samples.begin(), samples.end(), entry.second) == samples.end())
```

**Issue:** Linear searches in loops through samples/envelopes
- Potentially large collections
- Called during audio parsing loop

**Recommendation:** Use unordered_set
```cpp
std::unordered_set<uint32_t> usedEnvOffsets;
if(usedEnvOffsets.insert(addr).second) {
    // New item added
}
```

**Impact:** O(n) → O(1) per lookup


---

### 1.2 String Operations & Allocations

#### A. Excessive String Concatenation
**File:** `/home/user/Torch/src/Companion.cpp` (multiple locations)

Line 694: `auto output = (this->gCurrentDirectory / entryName).string();`
Line 767: `std::string output = (this->gCurrentDirectory / entryName).string();`
- Created twice per asset
- Path manipulations repeated

Line 529-530, 536-537:
```cpp
this->gFileHeader += line->as<std::string>() + "\n";  // Repeated allocations
```

**Issue:**
- String concatenation with `+` operator creates temporary strings
- Multiple path conversions to strings (.string() on filesystem::path)
- File headers built with repeated += operations

**Recommendation:** Use ostringstream or string builder
```cpp
// For file header
std::ostringstream headerStream;
for(auto line = header["header"].begin(); line != header["header"].end(); ++line) {
    headerStream << line->as<std::string>() << '\n';
}
this->gFileHeader = headerStream.str();

// Cache output paths
std::string output = (this->gCurrentDirectory / entryName).string();
std::replace(output.begin(), output.end(), '\\', '/');
// Reuse 'output' instead of creating new strings
```

**Impact:** Reduces temporary allocations and copies


#### B. Repeated String Operations
**File:** `/home/user/Torch/src/utils/StringHelper.cpp` (lines 12-25, 42-52, 54-63)

```cpp
std::vector<std::string> StringHelper::Split(std::string s, const std::string& delimiter) {
    while ((pos_end = s.find(delimiter, pos_start)) != std::string::npos) {
        token = s.substr(pos_start, pos_end - pos_start);  // Creates new strings
        pos_start = pos_end + delim_len;
        res.push_back(token);
    }
    res.push_back(s.substr(pos_start));  // Another allocation
    return res;
}

std::string StringHelper::Replace(std::string str, const std::string& from, const std::string& to) {
    size_t start_pos = str.find(from);
    while (start_pos != std::string::npos) {
        str.replace(start_pos, from.length(), to);  // Modifies in-place each iteration
        start_pos = str.find(from);  // Rescans entire string
    }
    return str;
}
```

**Issue:**
- Split() creates intermediate strings for each token
- Replace() rescans string from beginning each iteration
- Parameter passed by value, creating copy

**Recommendation:**
```cpp
// Use string_view for non-owning references
std::vector<std::string_view> StringHelper::Split(std::string_view s, const std::string_view& delimiter) {
    // ... use string_view throughout
}

// Cache find position
std::string StringHelper::Replace(const std::string& str, const std::string& from, const std::string& to) {
    std::string result = str;
    size_t start_pos = 0;
    while ((start_pos = result.find(from, start_pos)) != std::string::npos) {
        result.replace(start_pos, from.length(), to);
        start_pos += to.length();  // Skip past replacement
    }
    return result;
}
```

**Impact:** Reduces string allocations and rescans


#### C. String Parsing with Regex
**File:** `/home/user/Torch/src/Companion.cpp` (line 343)
```cpp
line = std::regex_replace(line, std::regex(R"((/\*.*?\*/)|(//.*$)|([^a-zA-Z0-9=_\-\.]))"), "");
```

**Issue:**
- Regex compiled every iteration in ParseEnums()
- Complex regex pattern
- Called for every line in enum header files

**Recommendation:** Compile regex once
```cpp
class Companion {
private:
    static const std::regex gEnumCommentRegex;
};

const std::regex Companion::gEnumCommentRegex(R"((/\*.*?\*/)|(//.*$)|([^a-zA-Z0-9=_\-\.]))");

// In ParseEnums:
line = std::regex_replace(line, gEnumCommentRegex, "");
```

**Impact:** Single regex compilation vs O(n) compilations


---

### 1.3 Unnecessary Data Copies

#### A. Vector Copies in Writes
**File:** `/home/user/Torch/src/Companion.cpp` (line 935)
```cpp
entries.insert(entries.end(), raw.begin(), raw.end());  // Copies all WriteEntry objects
```

**Issue:**
- WriteEntry contains std::string members
- Repeated for each type in gWriteMap
- WriteEntry is 88+ bytes

**Recommendation:** Use move semantics or reference
```cpp
// Option 1: Use move
for (auto& [type, raw] : this->gWriteMap[this->gCurrentFile]) {
    for(auto& entry : raw) {
        entries.push_back(std::move(entry));  // Move instead of copy
    }
}

// Option 2: Sort in-place (if possible)
std::vector<WriteEntry> entries;
for (auto& [type, raw] : this->gWriteMap[this->gCurrentFile]) {
    entries.insert(entries.end(), std::make_move_iterator(raw.begin()),
                   std::make_move_iterator(raw.end()));
}
```

**Impact:** Eliminates string copies and allocations


#### B. Vector Construction in File Reads
**File:** `/home/user/Torch/src/Companion.cpp` (lines 392, 407, 601, 1099, 1421)
```cpp
std::vector<uint8_t> data = std::vector<uint8_t>( std::istreambuf_iterator( input ), {} );
// and
auto data = std::vector( std::istreambuf_iterator( input ), {} );
```

**Issue:**
- Creates two temporary vectors (explicit + implicit)
- Redundant type specification
- No reserve() call for large files

**Recommendation:**
```cpp
std::ifstream input(path, std::ios::binary);
input.seekg(0, std::ios::end);
std::vector<uint8_t> data;
data.reserve(input.tellg());  // Pre-allocate
input.seekg(0, std::ios::beg);
data.assign(std::istreambuf_iterator<char>(input), {});
```

**Impact:** Single vector allocation vs potential multiple reallocations


#### C. YAML Node Copies
**File:** `/home/user/Torch/src/Companion.cpp` (line 722)
```cpp
root = YAML::LoadFile(this->gCurrentFile);  // Complete YAML reload
```

**Issue:**
- File is reloaded after iteration (line 722 after iterating lines 691-719)
- YAML parsing is expensive
- Node already has what's needed

**Recommendation:** Cache during first iteration or avoid reloading
- Store reference if possible
- Avoid modifying node during iteration


---

### 1.4 Inefficient Container Usage

#### A. nested unordered_map Performance
**File:** `/home/user/Torch/src/Companion.h` (line 233)
```cpp
std::unordered_map<std::string, std::unordered_map<uint32_t, std::tuple<std::string, YAML::Node>>> gAddrMap;
```

**Issue:**
- Nested hash map with string keys (slow hashing)
- Used for lookups per file + per address
- Tuple unpacking overhead

**Recommendation:** Use structure binding with indices
```cpp
struct AddressEntry {
    std::string name;
    YAML::Node node;
};

std::unordered_map<std::string, std::unordered_map<uint32_t, AddressEntry>> gAddrMap;

// Or for faster lookups by filename
std::unordered_map<std::string, std::vector<std::pair<uint32_t, AddressEntry>>> gAddrMapByFile;
```

**Impact:** Reduces unpacking overhead; clearer code


#### B. Linear Table Vector vs Sorted Container
**File:** `/home/user/Torch/src/Companion.h` (line 221)
```cpp
std::vector<Table> gTables;  // Linear search needed for intervals
```

**Recommendation:** Sort on initialization
```cpp
// Sort tables by start address
std::sort(gTables.begin(), gTables.end(), 
    [](const Table& a, const Table& b) { return a.start < b.start; });
```

This enables binary search (see SearchTable optimization above)


---

### 1.5 Missing const References

**File:** `/home/user/Torch/src/Companion.cpp` and factories
Multiple functions take parameters by value that should be const references:

```cpp
// Line 108
static std::string ConvertType(std::string type)  // Should be const std::string&

// Line 119
static std::string GetTypeNode(YAML::Node& node)  // Should be const YAML::Node&

// Line 438
void ParseCurrentFileConfig(YAML::Node node)  // Should be const YAML::Node&

// Line 1569
std::optional<std::tuple<...>> GetSafeNodeByAddr(const uint32_t addr, std::string type)
// 'type' should be const std::string&

// Line 1402
static void Pack(const std::string& folder, const std::string& output, const ArchiveType otrMode)
// Already good, but check all factories
```

**Recommendation:** Audit all function signatures for unnecessar copies
- Use `const T&` for large types (strings, vectors, nodes)
- Use `std::string_view` for string parameters that don't modify

---

## 2. I/O PERFORMANCE ISSUES

### 2.1 File Reading Patterns

#### A. Multiple File Reads Per File
**File:** `/home/user/Torch/src/Companion.cpp` (lines 600-602)
```cpp
std::ifstream yaml(path);
const std::vector<uint8_t> data = std::vector<uint8_t>(std::istreambuf_iterator( yaml ), {});
```

**Issue:**
- YAML file read for hash calculation
- Same file read again in YAML::LoadFile() (line 722)

**Recommendation:** Cache file contents
```cpp
class Companion {
    std::unordered_map<std::string, std::vector<uint8_t>> gFileCache;
};

// Read once, use multiple times
auto& data = gFileCache[path];
if(data.empty()) {
    std::ifstream file(path, std::ios::binary);
    file.seekg(0, std::ios::end);
    data.reserve(file.tellg());
    file.seekg(0);
    data.assign(...);
}
```

**Impact:** Single disk read vs multiple reads per file


#### B. Sequential Directory Iteration
**File:** `/home/user/Torch/src/Companion.cpp` (lines 1352-1375, 1415-1424)

Process() and Pack() both iterate directory recursively:
```cpp
for (const auto & entry : Torch::getRecursiveEntries(this->gAssetPath)){
    // Process all YAML files
}

// Later in Pack():
for (const auto & entry : Torch::getRecursiveEntries(folder)){
    // Read all files again
}
```

**Issue:**
- getRecursiveEntries() is called multiple times
- Each call re-traverses directory tree
- Files read into separate storage

**Recommendation:** Cache directory listing
```cpp
std::vector<fs::path> cachedEntries;
if(cachedEntries.empty()) {
    for(const auto& e : Torch::getRecursiveEntries(path)) {
        cachedEntries.push_back(e.path());
    }
}
```

**Impact:** Single directory traversal vs multiple


### 2.2 Archive Creation Efficiency

#### A. Archive File Addition Pattern
**File:** `/home/user/Torch/src/archive/ZWrapper.cpp` (lines 22-38)
```cpp
bool ZWrapper::AddFile(const std::string& path, std::vector<char> data) {
    // ... debug write ...
    this->mZip->writebytes(path, data);  // Per-file compression
    return true;
}
```

**Issue:**
- Files compressed individually
- No batching or streaming
- Vector passed by value

**Recommendation:**
```cpp
bool ZWrapper::AddFile(const std::string& path, const std::vector<char>& data) {
    // Pass by const reference
    this->mZip->writebytes(path, data);
}
```

**Impact:** Reduces vector copy overhead


#### B. SWrapper File Addition
**File:** `/home/user/Torch/src/archive/SWrapper.cpp` (lines 32-85)
```cpp
if(!SFileWriteFile(hFile, (void*) raw, size, MPQ_COMPRESSION_ZLIB)){
```

**Issue:**
- Files compressed with ZLIB individually
- No optimization for similar file types
- Consider compression level settings

**Recommendation:**
- Profile compression vs time trade-off
- Consider parallel compression for large files
- Batch similar types together


---

## 3. MEMORY USAGE & MANAGEMENT

### 3.1 Decompressor Cache Management

**File:** `/home/user/Torch/src/utils/Decompressor.cpp` (lines 14-75)
```cpp
static std::unordered_map<uint32_t, DataChunk*> gCachedChunks;

DataChunk* Decompressor::Decode(...) {
    if(!ignoreCache && gCachedChunks.contains(offset)){
        return gCachedChunks[offset];
    }
    
    const auto decompressed = new uint8_t[head.dest_size];
    gCachedChunks[offset] = new DataChunk{ decompressed, head.dest_size };
    return gCachedChunks[offset];
}

void Decompressor::ClearCache() {
    for(auto& [key, value] : gCachedChunks){
        delete value->data;  // Manual delete
    }
    gCachedChunks.clear();
}
```

**Issues:**
- Manual new/delete instead of smart pointers
- No cache eviction policy (could grow unbounded)
- Single global cache - not thread-safe
- Potential memory leaks if exceptions occur

**Recommendation:** Use smart pointers with LRU cache
```cpp
class Decompressor {
private:
    struct CacheEntry {
        std::unique_ptr<uint8_t[]> data;
        size_t size;
        uint32_t accessCount;
    };
    
    static std::unordered_map<uint32_t, CacheEntry> gCachedChunks;
    static constexpr size_t MAX_CACHE_SIZE = 512 * 1024 * 1024;  // 512MB limit
    static size_t gCurrentCacheSize;
};

// Or use boost::compute::lru_cache or similar
```

**Impact:** 
- Safer memory management
- Prevents unbounded memory growth
- Exception-safe


### 3.2 Temporary String Allocations

**File:** `/home/user/Torch/src/Companion.cpp` (line 985)
```cpp
stream << "// WARNING: Overlap detected between 0x" << std::hex << startptr 
       << " and 0x" << end << " with size 0x" << std::abs(gap) << "\n";
```

**Issue:**
- String stream operations create temporaries for each `<<`
- Format conversions (hex, uppercase) add overhead
- Repeated logging in loops

**Recommendation:**
```cpp
stream << fmt::format("// WARNING: Overlap detected between 0x{:X} and 0x{:X} "
                      "with size 0x{:X}\n", startptr, end, std::abs(gap));
```

(Note: fmt library more efficient than stream operators)


### 3.3 Vector Pre-allocation Issues

**File:** `/home/user/Torch/src/factories/GenericArrayFactory.cpp`
```cpp
for (auto &datum : array->mData) {
    // Process each datum
}
```

**Issue:**
- GenericArray constructor iterates mData to compute max width/precision
- If mData is large, this is slow
- No pre-allocation before push_back operations during construction

**Recommendation:**
```cpp
GenericArray::GenericArray(std::vector<ArrayDatum> data) 
    : mData(std::move(data))  // Use move instead of copy
{
    mMaxWidth = 1;
    mMaxPrec = 1;
    for (const auto &datum : mData) {  // Use const reference
        // ... compute stats
    }
}
```


---

## 4. DEPENDENCY-RELATED PERFORMANCE

### 4.1 YAML Parsing Overhead

**File:** `/home/user/Torch/src/Companion.cpp` (multiple locations)

YAML parsing occurs repeatedly:
- Line 722: `root = YAML::LoadFile(this->gCurrentFile);`
- Line 1092: `YAML::Node config = YAML::LoadFile(configPath.string());`
- Line 1367: `YAML::Node root = YAML::LoadFile(yamlPath);`

**Issue:**
- yaml-cpp is relatively slow
- Full YAML tree parsed even if only accessing few fields
- No caching between passes

**Recommendation:**
```cpp
class YAMLCache {
    static std::unordered_map<std::string, YAML::Node> cache;
public:
    static YAML::Node LoadFile(const std::string& path) {
        auto it = cache.find(path);
        if(it != cache.end()) return it->second;
        
        auto node = YAML::LoadFile(path);
        cache[path] = node;
        return node;
    }
};
```

**Impact:** Eliminates redundant parsing


### 4.2 Compression Library Efficiency

**File:** `/home/user/Torch/src/utils/Decompressor.cpp`

Uses libmio0, libyay0 C libraries:
```cpp
mio0_decode(in_buf, decompressed, nullptr);
```

**Considerations:**
- C library integration requires C++ wrapping
- Allocation strategies set by C library
- Could consider alternative libraries (zstd for compression ratio)

**Recommendation:**
- Profile compression performance vs ratio
- Consider async decompression if system allows
- Document compression settings in config


### 4.3 Logging Overhead

**File:** Throughout codebase - uses spdlog

**Current:**
```cpp
SPDLOG_INFO("- [{}] Processing {} at 0x{:X}", type, name, offset);  // Line 364
```

**Issue:**
- Conditional logging only at runtime
- spdlog is good but still has overhead in tight loops
- Level checking occurs per call

**Recommendation:**
```cpp
// Use compile-time filtering
#if TORCH_LOG_LEVEL >= 2
    SPDLOG_INFO("...");
#endif

// Or structured logging
spdlog::debug("asset_processed", 
    spdlog::source_loc{__FILE__, __LINE__, __FUNCTION__},
    spdlog::kv("type", type), 
    spdlog::kv("name", name));
```

---

## 5. ALGORITHM IMPROVEMENTS

### 5.1 Batch Processing Optimization

**File:** `/home/user/Torch/src/Companion.cpp` ProcessFile()

Current: Sequential processing per asset
```cpp
for(auto asset = root.begin(); asset != root.end(); ++asset) {
    auto result = this->ParseNode(assetNode, output);
    // Process individually
}
```

**Recommendation:** Batch similar types
```cpp
struct ParseBatch {
    std::string type;
    std::vector<ParseTask> tasks;
};

// Group by type
std::unordered_map<std::string, ParseBatch> batches;
for(auto asset = root.begin(); asset != root.end(); ++asset) {
    batches[type].tasks.push_back(task);
}

// Process batches (can be parallelized)
for(auto& batch : batches) {
    ProcessBatch(batch);
}
```

**Impact:** 
- Better cache locality
- Enables batch optimizations in factories
- Potential for parallelization


---

## PRIORITY RECOMMENDATIONS SUMMARY

### High Impact (Do First)
1. **SearchTable() optimization** - Binary search (1491-1499)
   - Impact: Eliminates frequent O(n) searches
   - Effort: Low

2. **GetNodesByType() caching** - Build index during parsing (1647-1668)
   - Impact: O(n) → O(1) queries
   - Effort: Medium

3. **Decompressor cache** - Use smart pointers (Decompressor.cpp)
   - Impact: Safety and memory control
   - Effort: Medium

4. **String concatenation** - Use ostringstream (lines 529, 529, 536)
   - Impact: Reduces allocations
   - Effort: Low

### Medium Impact
5. **Vector copy in writes** - Use move semantics (line 935)
   - Impact: Reduces string copies
   - Effort: Low

6. **std::find() → unordered_set** - Multiple locations
   - Impact: O(n²) → O(n) algorithms
   - Effort: Medium

7. **File read caching** - Avoid duplicate reads (lines 600, 722)
   - Impact: Disk I/O reduction
   - Effort: Medium

### Lower Priority
8. **const references** - Audit function signatures
   - Impact: Reduced copies
   - Effort: High (broad changes)

9. **YAML parsing caching** - Cache LoadFile results
   - Impact: Parsing overhead reduction
   - Effort: Medium

10. **Regex compilation** - Compile once (line 343)
    - Impact: Small but measurable
    - Effort: Low

---

## TESTING RECOMMENDATIONS

1. **Profile with perf/VTune**
   ```bash
   perf record ./torch
   perf report
   ```

2. **Memory profiling with valgrind**
   ```bash
   valgrind --tool=massif ./torch
   ```

3. **Micro-benchmarks**
   ```cpp
   auto start = std::chrono::high_resolution_clock::now();
   SearchTable(addr);  // Before/after optimization
   auto end = std::chrono::high_resolution_clock::now();
   ```

4. **Regression testing** - Track metrics:
   - Total processing time
   - Peak memory usage
   - File I/O count
   - Cache hit rates

