---
layout: post
title: "Pure Storage Interview: Reverse File Content Chunk by Chunk"
date: 2025-11-16
categories: [Interview Preparation, Pure Storage, Technical Interview, File I/O, System Design]
excerpt: "Complete solution for reversing large file content in-place: chunk-based reading/writing, two-pointer approach, and crash recovery strategies. Includes C++ implementations and error handling."
---

## Problem Overview

**Pure Storage Interview: Reverse File Content**

This problem tests your ability to:
1. Handle large file I/O efficiently
2. Implement in-place algorithms
3. Design crash-resistant systems
4. Manage memory constraints
5. Handle edge cases and errors

### Problem Statement

Given a file, reverse its content character by character **in-place** (read and write to the same file).

**Requirements:**
- File is very large (cannot load entire file into memory)
- Must read and write **chunk by chunk**
- Input and output use the **same file**
- Handle system crashes during operation

**Example:**
```
Input File:  ABCDEFG
Output File: GFEDCBA
```

### Challenges

1. **Memory constraint**: Cannot load entire file
2. **In-place operation**: Same file for input/output
3. **Chunk-based I/O**: Process in small chunks
4. **Crash recovery**: Ensure data integrity if system crashes

## Part 1: Basic Approach - Two-Pointer Method

### Algorithm Overview

1. Use two file pointers: one at start, one at end
2. Read chunks from both ends
3. Swap chunks
4. Move pointers inward
5. Repeat until pointers meet

### Implementation

```cpp
#include <fstream>
#include <iostream>
#include <algorithm>
#include <vector>
#include <cstring>
using namespace std;

class FileReverser {
private:
    static const size_t CHUNK_SIZE = 4096;  // 4KB chunks
    
    void swapChunks(fstream& file, size_t pos1, size_t pos2, size_t size) {
        vector<char> chunk1(size);
        vector<char> chunk2(size);
        
        // Read chunks
        file.seekg(pos1);
        file.read(chunk1.data(), size);
        
        file.seekg(pos2);
        file.read(chunk2.data(), size);
        
        // Reverse chunks
        reverse(chunk1.begin(), chunk1.end());
        reverse(chunk2.begin(), chunk2.end());
        
        // Write swapped chunks
        file.seekp(pos2);
        file.write(chunk1.data(), size);
        
        file.seekp(pos1);
        file.write(chunk2.data(), size);
        
        file.flush();  // Ensure data is written
    }
    
public:
    bool reverseFile(const string& filename) {
        fstream file(filename, ios::in | ios::out | ios::binary);
        if (!file.is_open()) {
            cerr << "Error: Cannot open file " << filename << endl;
            return false;
        }
        
        // Get file size
        file.seekg(0, ios::end);
        size_t fileSize = file.tellg();
        file.seekg(0, ios::beg);
        
        if (fileSize == 0) {
            return true;  // Empty file
        }
        
        size_t leftPos = 0;
        size_t rightPos = fileSize;
        
        while (leftPos < rightPos) {
            // Calculate chunk sizes
            size_t leftChunkSize = min(CHUNK_SIZE, rightPos - leftPos);
            size_t rightChunkSize = min(CHUNK_SIZE, rightPos - leftPos);
            
            // Ensure chunks don't overlap
            if (leftPos + leftChunkSize > rightPos - rightChunkSize) {
                // Handle middle section
                size_t remaining = rightPos - leftPos;
                leftChunkSize = remaining / 2;
                rightChunkSize = remaining - leftChunkSize;
            }
            
            // Read and swap
            size_t rightStart = rightPos - rightChunkSize;
            swapChunks(file, leftPos, rightStart, leftChunkSize);
            
            // Update positions
            leftPos += leftChunkSize;
            rightPos -= rightChunkSize;
        }
        
        file.close();
        return true;
    }
};
```

**Problem:** This approach has issues with chunk alignment and overlapping.

## Part 2: Correct Chunk-by-Chunk Implementation

### Improved Algorithm

```cpp
#include <fstream>
#include <vector>
#include <algorithm>
#include <iostream>
using namespace std;

class FileReverser {
private:
    static const size_t CHUNK_SIZE = 4096;  // 4KB chunks
    
    // Read chunk from file
    vector<char> readChunk(fstream& file, size_t pos, size_t size) {
        vector<char> chunk(size);
        file.seekg(pos);
        file.read(chunk.data(), size);
        size_t bytesRead = file.gcount();
        chunk.resize(bytesRead);  // Adjust if read less than requested
        return chunk;
    }
    
    // Write chunk to file
    void writeChunk(fstream& file, size_t pos, const vector<char>& chunk) {
        file.seekp(pos);
        file.write(chunk.data(), chunk.size());
        file.flush();  // Ensure written to disk
    }
    
public:
    bool reverseFile(const string& filename) {
        fstream file(filename, ios::in | ios::out | ios::binary);
        if (!file.is_open()) {
            return false;
        }
        
        // Get file size
        file.seekg(0, ios::end);
        size_t fileSize = file.tellg();
        
        if (fileSize == 0) {
            return true;  // Empty file
        }
        
        size_t leftPos = 0;
        size_t rightPos = fileSize;
        
        while (leftPos < rightPos) {
            // Determine chunk sizes
            size_t leftChunkSize = min(CHUNK_SIZE, rightPos - leftPos);
            size_t rightChunkSize = min(CHUNK_SIZE, rightPos - leftPos);
            
            // Adjust if chunks would overlap
            if (leftPos + leftChunkSize > rightPos - rightChunkSize) {
                // Process remaining bytes
                size_t remaining = rightPos - leftPos;
                leftChunkSize = remaining;
                rightChunkSize = 0;
            }
            
            if (rightChunkSize > 0) {
                // Read chunks from both ends
                vector<char> leftChunk = readChunk(file, leftPos, leftChunkSize);
                vector<char> rightChunk = readChunk(file, rightPos - rightChunkSize, rightChunkSize);
                
                // Reverse chunks
                reverse(leftChunk.begin(), leftChunk.end());
                reverse(rightChunk.begin(), rightChunk.end());
                
                // Swap and write
                writeChunk(file, rightPos - rightChunkSize, leftChunk);
                writeChunk(file, leftPos, rightChunk);
                
                leftPos += leftChunkSize;
                rightPos -= rightChunkSize;
            } else {
                // Handle middle section (odd number of bytes)
                vector<char> chunk = readChunk(file, leftPos, leftChunkSize);
                reverse(chunk.begin(), chunk.end());
                writeChunk(file, leftPos, chunk);
                break;
            }
        }
        
        file.close();
        return true;
    }
};
```

## Part 3: Character-by-Character Approach

### More Precise Implementation

```cpp
class FileReverser {
private:
    static const size_t CHUNK_SIZE = 4096;
    
public:
    bool reverseFile(const string& filename) {
        fstream file(filename, ios::in | ios::out | ios::binary);
        if (!file.is_open()) {
            return false;
        }
        
        // Get file size
        file.seekg(0, ios::end);
        size_t fileSize = file.tellg();
        
        if (fileSize == 0) {
            return true;
        }
        
        size_t leftPos = 0;
        size_t rightPos = fileSize - 1;
        
        while (leftPos < rightPos) {
            // Read chunks
            size_t leftChunkSize = min(CHUNK_SIZE, rightPos - leftPos + 1);
            size_t rightChunkSize = min(CHUNK_SIZE, rightPos - leftPos + 1);
            
            // Ensure chunks don't overlap
            if (leftPos + leftChunkSize > rightPos - rightChunkSize + 1) {
                leftChunkSize = (rightPos - leftPos + 1) / 2;
                rightChunkSize = (rightPos - leftPos + 1) - leftChunkSize;
            }
            
            // Read left chunk
            vector<char> leftChunk(leftChunkSize);
            file.seekg(leftPos);
            file.read(leftChunk.data(), leftChunkSize);
            
            // Read right chunk
            vector<char> rightChunk(rightChunkSize);
            file.seekg(rightPos - rightChunkSize + 1);
            file.read(rightChunk.data(), rightChunkSize);
            
            // Reverse chunks
            reverse(leftChunk.begin(), leftChunk.end());
            reverse(rightChunk.begin(), rightChunk.end());
            
            // Swap chunks
            file.seekp(rightPos - rightChunkSize + 1);
            file.write(leftChunk.data(), leftChunkSize);
            
            file.seekp(leftPos);
            file.write(rightChunk.data(), rightChunkSize);
            
            file.flush();
            
            // Update positions
            leftPos += leftChunkSize;
            rightPos -= rightChunkSize;
        }
        
        file.close();
        return true;
    }
};
```

## Part 4: Complete Robust Implementation

### Production-Ready Version

```cpp
#include <fstream>
#include <vector>
#include <algorithm>
#include <iostream>
#include <stdexcept>
#include <cstring>
using namespace std;

class FileReverser {
private:
    static const size_t CHUNK_SIZE = 4096;  // 4KB chunks
    
    // Read chunk with error handling
    vector<char> readChunk(fstream& file, size_t pos, size_t size) {
        file.seekg(pos);
        if (!file.good()) {
            throw runtime_error("Failed to seek to position " + to_string(pos));
        }
        
        vector<char> chunk(size);
        file.read(chunk.data(), size);
        size_t bytesRead = file.gcount();
        
        if (file.fail() && !file.eof()) {
            throw runtime_error("Failed to read chunk at position " + to_string(pos));
        }
        
        chunk.resize(bytesRead);
        return chunk;
    }
    
    // Write chunk with error handling
    void writeChunk(fstream& file, size_t pos, const vector<char>& chunk) {
        file.seekp(pos);
        if (!file.good()) {
            throw runtime_error("Failed to seek to write position " + to_string(pos));
        }
        
        file.write(chunk.data(), chunk.size());
        if (!file.good()) {
            throw runtime_error("Failed to write chunk at position " + to_string(pos));
        }
        
        file.flush();
        if (!file.good()) {
            throw runtime_error("Failed to flush after writing at position " + to_string(pos));
        }
    }
    
public:
    bool reverseFile(const string& filename) {
        fstream file(filename, ios::in | ios::out | ios::binary);
        if (!file.is_open()) {
            cerr << "Error: Cannot open file " << filename << endl;
            return false;
        }
        
        try {
            // Get file size
            file.seekg(0, ios::end);
            size_t fileSize = file.tellg();
            file.seekg(0, ios::beg);
            
            if (fileSize == 0) {
                return true;  // Empty file
            }
            
            if (fileSize == 1) {
                return true;  // Single character, already reversed
            }
            
            size_t leftPos = 0;
            size_t rightPos = fileSize - 1;
            
            while (leftPos < rightPos) {
                // Calculate chunk sizes
                size_t leftChunkSize = min(CHUNK_SIZE, rightPos - leftPos + 1);
                size_t rightChunkSize = min(CHUNK_SIZE, rightPos - leftPos + 1);
                
                // Adjust if chunks would overlap
                if (leftPos + leftChunkSize > rightPos - rightChunkSize + 1) {
                    // Process remaining bytes
                    size_t remaining = rightPos - leftPos + 1;
                    leftChunkSize = remaining / 2;
                    rightChunkSize = remaining - leftChunkSize;
                }
                
                // Read chunks
                vector<char> leftChunk = readChunk(file, leftPos, leftChunkSize);
                vector<char> rightChunk = readChunk(file, rightPos - rightChunkSize + 1, rightChunkSize);
                
                // Reverse chunks
                reverse(leftChunk.begin(), leftChunk.end());
                reverse(rightChunk.begin(), rightChunk.end());
                
                // Swap chunks
                writeChunk(file, rightPos - rightChunkSize + 1, leftChunk);
                writeChunk(file, leftPos, rightChunk);
                
                // Update positions
                leftPos += leftChunkSize;
                rightPos -= rightChunkSize;
            }
            
            file.close();
            return true;
            
        } catch (const exception& e) {
            cerr << "Error: " << e.what() << endl;
            file.close();
            return false;
        }
    }
};
```

## Part 5: Crash Recovery Strategy

### Problem: System Crash During Operation

If the system crashes while reversing:
- File might be partially reversed
- Data might be corrupted
- Need to recover and complete the operation

### Solution 1: Backup File Approach

```cpp
#include <filesystem>
#include <fstream>
#include <vector>
#include <algorithm>
using namespace std;
namespace fs = filesystem;

class CrashResistantFileReverser {
private:
    static const size_t CHUNK_SIZE = 4096;
    
    struct Progress {
        size_t leftPos;
        size_t rightPos;
        size_t fileSize;
    };
    
    // Save progress to metadata file
    void saveProgress(const string& progressFile, const Progress& progress) {
        ofstream file(progressFile, ios::binary);
        file.write(reinterpret_cast<const char*>(&progress), sizeof(Progress));
        file.flush();
        fsync(file.fileno());  // Force write to disk
    }
    
    // Load progress from metadata file
    bool loadProgress(const string& progressFile, Progress& progress) {
        ifstream file(progressFile, ios::binary);
        if (!file.is_open()) {
            return false;
        }
        file.read(reinterpret_cast<char*>(&progress), sizeof(Progress));
        return file.good();
    }
    
public:
    bool reverseFileWithRecovery(const string& filename) {
        string backupFile = filename + ".backup";
        string progressFile = filename + ".progress";
        
        // Check if recovery is needed
        Progress progress;
        bool needRecovery = loadProgress(progressFile, progress);
        
        if (!needRecovery) {
            // First time: create backup
            fs::copy_file(filename, backupFile, fs::copy_options::overwrite_existing);
            
            // Initialize progress
            ifstream file(filename, ios::binary);
            file.seekg(0, ios::end);
            progress.fileSize = file.tellg();
            file.close();
            progress.leftPos = 0;
            progress.rightPos = progress.fileSize - 1;
        }
        
        // Open file for reading/writing
        fstream file(filename, ios::in | ios::out | ios::binary);
        if (!file.is_open()) {
            return false;
        }
        
        try {
            size_t leftPos = progress.leftPos;
            size_t rightPos = progress.rightPos;
            
            while (leftPos < rightPos) {
                // Calculate chunk sizes
                size_t leftChunkSize = min(CHUNK_SIZE, rightPos - leftPos + 1);
                size_t rightChunkSize = min(CHUNK_SIZE, rightPos - leftPos + 1);
                
                if (leftPos + leftChunkSize > rightPos - rightChunkSize + 1) {
                    size_t remaining = rightPos - leftPos + 1;
                    leftChunkSize = remaining / 2;
                    rightChunkSize = remaining - leftChunkSize;
                }
                
                // Read chunks
                vector<char> leftChunk(leftChunkSize);
                file.seekg(leftPos);
                file.read(leftChunk.data(), leftChunkSize);
                
                vector<char> rightChunk(rightChunkSize);
                file.seekg(rightPos - rightChunkSize + 1);
                file.read(rightChunk.data(), rightChunkSize);
                
                // Reverse and swap
                reverse(leftChunk.begin(), leftChunk.end());
                reverse(rightChunk.begin(), rightChunk.end());
                
                file.seekp(rightPos - rightChunkSize + 1);
                file.write(leftChunk.data(), leftChunkSize);
                file.flush();
                
                file.seekp(leftPos);
                file.write(rightChunk.data(), rightChunkSize);
                file.flush();
                
                // Update progress
                leftPos += leftChunkSize;
                rightPos -= rightChunkSize;
                
                progress.leftPos = leftPos;
                progress.rightPos = rightPos;
                saveProgress(progressFile, progress);
            }
            
            // Operation complete: cleanup
            file.close();
            fs::remove(progressFile);
            fs::remove(backupFile);
            
            return true;
            
        } catch (const exception& e) {
            // On error, restore from backup
            file.close();
            fs::copy_file(backupFile, filename, fs::copy_options::overwrite_existing);
            return false;
        }
    }
};
```

### Solution 2: Atomic Write with Temporary File

```cpp
class AtomicFileReverser {
private:
    static const size_t CHUNK_SIZE = 4096;
    
public:
    bool reverseFileAtomic(const string& filename) {
        string tempFile = filename + ".tmp";
        
        // Read original file and reverse in memory (chunk by chunk)
        ifstream input(filename, ios::binary);
        ofstream output(tempFile, ios::binary);
        
        if (!input.is_open() || !output.is_open()) {
            return false;
        }
        
        // Get file size
        input.seekg(0, ios::end);
        size_t fileSize = input.tellg();
        input.seekg(0, ios::beg);
        
        // Read and write in reverse order
        size_t pos = fileSize;
        while (pos > 0) {
            size_t chunkSize = min(CHUNK_SIZE, pos);
            pos -= chunkSize;
            
            input.seekg(pos);
            vector<char> chunk(chunkSize);
            input.read(chunk.data(), chunkSize);
            
            reverse(chunk.begin(), chunk.end());
            output.write(chunk.data(), chunkSize);
            output.flush();
        }
        
        input.close();
        output.close();
        
        // Atomic replace: rename is atomic on most systems
        fs::rename(tempFile, filename);
        
        return true;
    }
};
```

### Solution 3: Two-Pass with Checksum

```cpp
#include <openssl/md5.h>
#include <iomanip>
#include <sstream>

class ChecksumFileReverser {
private:
    static const size_t CHUNK_SIZE = 4096;
    
    string calculateChecksum(const string& filename) {
        ifstream file(filename, ios::binary);
        MD5_CTX md5Context;
        MD5_Init(&md5Context);
        
        vector<char> buffer(CHUNK_SIZE);
        while (file.read(buffer.data(), CHUNK_SIZE) || file.gcount() > 0) {
            MD5_Update(&md5Context, buffer.data(), file.gcount());
        }
        
        unsigned char digest[MD5_DIGEST_LENGTH];
        MD5_Final(digest, &md5Context);
        
        stringstream ss;
        for (int i = 0; i < MD5_DIGEST_LENGTH; i++) {
            ss << hex << setw(2) << setfill('0') << (int)digest[i];
        }
        return ss.str();
    }
    
public:
    bool reverseFileWithChecksum(const string& filename) {
        string checksumFile = filename + ".checksum";
        string originalChecksum = calculateChecksum(filename);
        
        // Save original checksum
        ofstream csFile(checksumFile);
        csFile << originalChecksum << endl;
        csFile.close();
        
        // Perform reversal
        FileReverser reverser;
        if (!reverser.reverseFile(filename)) {
            return false;
        }
        
        // Verify: reverse again should get original
        string reversedChecksum = calculateChecksum(filename);
        reverser.reverseFile(filename);
        string restoredChecksum = calculateChecksum(filename);
        
        if (restoredChecksum == originalChecksum) {
            // Success: reverse one more time to get final result
            reverser.reverseFile(filename);
            fs::remove(checksumFile);
            return true;
        }
        
        return false;
    }
};
```

## Part 6: Complete Solution with Crash Recovery

```cpp
#include <fstream>
#include <vector>
#include <algorithm>
#include <filesystem>
#include <iostream>
#include <fstream>
using namespace std;
namespace fs = filesystem;

class RobustFileReverser {
private:
    static const size_t CHUNK_SIZE = 4096;
    
    struct State {
        size_t leftPos;
        size_t rightPos;
        size_t fileSize;
        bool isValid;
    };
    
    void saveState(const string& stateFile, const State& state) {
        ofstream file(stateFile, ios::binary);
        file.write(reinterpret_cast<const char*>(&state), sizeof(State));
        file.flush();
    }
    
    bool loadState(const string& stateFile, State& state) {
        ifstream file(stateFile, ios::binary);
        if (!file.is_open()) {
            return false;
        }
        file.read(reinterpret_cast<char*>(&state), sizeof(State));
        return file.good() && state.isValid;
    }
    
    void invalidateState(const string& stateFile) {
        State state = {0, 0, 0, false};
        saveState(stateFile, state);
    }
    
public:
    bool reverseFile(const string& filename) {
        string backupFile = filename + ".backup";
        string stateFile = filename + ".state";
        
        fstream file(filename, ios::in | ios::out | ios::binary);
        if (!file.is_open()) {
            return false;
        }
        
        State state;
        bool recovering = loadState(stateFile, state);
        
        if (!recovering) {
            // First run: create backup and initialize state
            fs::copy_file(filename, backupFile, fs::copy_options::overwrite_existing);
            
            file.seekg(0, ios::end);
            state.fileSize = file.tellg();
            state.leftPos = 0;
            state.rightPos = (state.fileSize > 0) ? state.fileSize - 1 : 0;
            state.isValid = true;
        }
        
        try {
            size_t leftPos = state.leftPos;
            size_t rightPos = state.rightPos;
            
            while (leftPos < rightPos) {
                // Calculate chunk sizes
                size_t leftChunkSize = min(CHUNK_SIZE, rightPos - leftPos + 1);
                size_t rightChunkSize = min(CHUNK_SIZE, rightPos - leftPos + 1);
                
                if (leftPos + leftChunkSize > rightPos - rightChunkSize + 1) {
                    size_t remaining = rightPos - leftPos + 1;
                    leftChunkSize = remaining / 2;
                    rightChunkSize = remaining - leftChunkSize;
                }
                
                // Read chunks
                vector<char> leftChunk(leftChunkSize);
                file.seekg(leftPos);
                file.read(leftChunk.data(), leftChunkSize);
                
                vector<char> rightChunk(rightChunkSize);
                file.seekg(rightPos - rightChunkSize + 1);
                file.read(rightChunk.data(), rightChunkSize);
                
                // Reverse and swap
                reverse(leftChunk.begin(), leftChunk.end());
                reverse(rightChunk.begin(), rightChunk.end());
                
                file.seekp(rightPos - rightChunkSize + 1);
                file.write(leftChunk.data(), leftChunkSize);
                file.flush();
                
                file.seekp(leftPos);
                file.write(rightChunk.data(), rightChunkSize);
                file.flush();
                
                // Update and save state
                leftPos += leftChunkSize;
                rightPos -= rightChunkSize;
                
                state.leftPos = leftPos;
                state.rightPos = rightPos;
                saveState(stateFile, state);
            }
            
            // Success: cleanup
            file.close();
            invalidateState(stateFile);
            fs::remove(stateFile);
            fs::remove(backupFile);
            
            return true;
            
        } catch (const exception& e) {
            // Error: restore from backup
            file.close();
            fs::copy_file(backupFile, filename, fs::copy_options::overwrite_existing);
            return false;
        }
    }
};
```

## Part 7: Test Program

```cpp
#include <iostream>
#include <fstream>
#include <cassert>
#include <filesystem>
using namespace std;

void testFileReverser() {
    cout << "=== Testing File Reverser ===" << endl;
    
    // Test 1: Simple case
    {
        string filename = "test1.txt";
        ofstream file(filename);
        file << "ABCDEFG";
        file.close();
        
        FileReverser reverser;
        assert(reverser.reverseFile(filename));
        
        ifstream result(filename);
        string content;
        result >> content;
        assert(content == "GFEDCBA");
        cout << "Test 1 passed: ABCDEFG -> GFEDCBA" << endl;
        
        fs::remove(filename);
    }
    
    // Test 2: Empty file
    {
        string filename = "test2.txt";
        ofstream file(filename);
        file.close();
        
        FileReverser reverser;
        assert(reverser.reverseFile(filename));
        cout << "Test 2 passed: Empty file" << endl;
        
        fs::remove(filename);
    }
    
    // Test 3: Single character
    {
        string filename = "test3.txt";
        ofstream file(filename);
        file << "A";
        file.close();
        
        FileReverser reverser;
        assert(reverser.reverseFile(filename));
        
        ifstream result(filename);
        string content;
        result >> content;
        assert(content == "A");
        cout << "Test 3 passed: Single character" << endl;
        
        fs::remove(filename);
    }
    
    // Test 4: Large file simulation
    {
        string filename = "test4.txt";
        ofstream file(filename);
        for (int i = 0; i < 10000; i++) {
            file << (char)('A' + (i % 26));
        }
        file.close();
        
        FileReverser reverser;
        assert(reverser.reverseFile(filename));
        
        // Verify: reverse twice should get original
        string original;
        ifstream origFile(filename);
        origFile >> original;
        origFile.close();
        
        reverser.reverseFile(filename);
        
        string restored;
        ifstream restFile(filename);
        restFile >> restored;
        restFile.close();
        
        // Original should equal restored
        assert(original.length() == restored.length());
        for (size_t i = 0; i < original.length(); i++) {
            assert(original[i] == restored[original.length() - 1 - i]);
        }
        
        cout << "Test 4 passed: Large file (10000 chars)" << endl;
        
        fs::remove(filename);
    }
    
    cout << "\nAll tests passed!" << endl;
}

int main() {
    testFileReverser();
    return 0;
}
```

## Part 8: Interview Discussion Points

### Key Questions and Answers

**Q1: Why chunk-by-chunk instead of loading entire file?**

**A:**
- **Memory constraint**: File might be larger than available RAM
- **Efficiency**: Process large files without memory issues
- **Scalability**: Works for files of any size

**Q2: How to handle overlapping chunks?**

**A:**
- Check if `leftPos + leftChunkSize > rightPos - rightChunkSize`
- Adjust chunk sizes to prevent overlap
- Handle middle section separately if needed

**Q3: What if system crashes during reversal?**

**A:** Several strategies:
1. **Backup file**: Keep original, restore on failure
2. **Progress tracking**: Save state, resume on restart
3. **Atomic operations**: Use temporary file + atomic rename
4. **Checksum verification**: Verify integrity after operation

**Q4: Why flush after each write?**

**A:**
- Ensure data is written to disk immediately
- Critical for crash recovery
- Prevents data loss if system crashes

**Q5: How to optimize performance?**

**A:**
- **Chunk size**: Balance between I/O overhead and memory
- **Sequential access**: Minimize seeks when possible
- **Buffer management**: Reuse buffers
- **Parallel processing**: Process multiple chunks (complex)

## Key Takeaways

### Algorithm Design

1. **Two-pointer approach**: Start from both ends
2. **Chunk-based processing**: Handle large files efficiently
3. **In-place operation**: Modify same file
4. **State management**: Track progress for recovery

### Crash Recovery Strategies

1. **Backup + Restore**: Simple but requires 2x storage
2. **Progress tracking**: Resume from last position
3. **Atomic operations**: All-or-nothing approach
4. **Checksum verification**: Ensure data integrity

### Implementation Tips

1. **Error handling**: Check all file operations
2. **Flush operations**: Ensure data written to disk
3. **State persistence**: Save progress regularly
4. **Cleanup**: Remove temporary files on success

### Interview Tips

1. **Start simple**: Basic two-pointer algorithm
2. **Handle edge cases**: Empty file, single char, overlapping chunks
3. **Discuss crash recovery**: Show system design thinking
4. **Optimize**: Chunk size, I/O patterns
5. **Test thoroughly**: Multiple test cases

## Summary

File reversal with chunk-based processing demonstrates:

- **Large file handling**: Memory-efficient algorithms
- **In-place operations**: Modifying files efficiently
- **Crash recovery**: System design for reliability
- **Error handling**: Robust file I/O operations

**Key implementation:**
- **Two-pointer method**: Process from both ends
- **Chunk-based I/O**: Handle large files
- **State tracking**: Enable crash recovery
- **Atomic operations**: Ensure data integrity

Understanding these concepts is essential for:
- System programming
- File system operations
- Data integrity
- Crash recovery systems

This knowledge is particularly valuable for storage companies like Pure Storage, where reliable file operations and crash recovery are critical.

