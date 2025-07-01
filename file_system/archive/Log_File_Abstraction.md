# Indexlib 

**:**
* `file_system/archive/LogFile.h`
* `file_system/archive/LogFile.cpp`

## 1. 

`LogFile`  Indexlib **
**

:

*   **
**
: `Write(key, value)`  `Read(key)` 

*   **
**
: `LogFile`  (value)  (key)  :

    *   (.lfd)
    *   (.lfm)
*   **
**
: 
*   **
**
: 

`LogFile`  Log-Structured Storage

## 2. 

`LogFile` 

### 2.1. 

`LogFile`  :

1.  ** (`.lfd` - Log File Data)**:
    *   **
**
: (values)
    *   **
**
: 
    *   **
**
: I/O 

2.  ** (`.lfm` - Log File Meta)**:
    *   **
**
: 
    *   **
**
: `key_length\tvalue_offset\tvalue_length\tkey`
        *   `key_length`: `key` 
        *   `value_offset`: `key` `value` `.lfd` 
        *   `value_length`: `value` 
        *   `key`: 
    *   **
**
:
        *   **
**
: `LogFile` (`_metaInfos` `map` )
        *   **
**
: `.lfm` 
        *   **
**
: `key_length` `key` 

### 2.2. 

`LogFile` `std::map<std::string, RecordInfo>` `_metaInfos`

*   **`_metaInfos`**:
    *   ** (`std::string`)**
: `key`
    *   ** (`RecordInfo`)**
: `valueOffset`  `valueLength` `.lfd` 

`LogFile` `.lfm` `_metaInfos` `Read`  `HasKey` `map` 

### 2.3. 

*   **
**
: `LogFile` **
**
**
**
**
**

*   **
**
: B、LSM
*   **
**
: `LogFile` `.lfm` `.lfd` `.lfd` 

## 3. 

### 3.1. (`Open`  `LoadLogFile`)

`Open` `LogFile` (`fslib::READ`  `fslib::APPEND`)

*   ** (`fslib::READ`)**:
    1.  `LineReader` `.lfm` 
    2.  `Parse` `key`  `RecordInfo` (
    3.  `_metaInfos` `map` 
    4.  `.lfd` `FileReader` (`_dataFileReader`)

*   ** (`fslib::APPEND`)**:
    1.  `.lfd`  `.lfm` `BufferedFileWriter` (`_dataFileWriter`  `_metaFileWriter`)
    2.  
    3.  `_metaInfos` 

** (`LogFile::LoadLogFile`)**:
```cpp
FSResult<void> LogFile::LoadLogFile(const std::shared_ptr<IDirectory>& directory, const std::string& metaFile,
                                    const std::string& dataFile, fslib::Flag openMode) noexcept
{
    if (openMode == fslib::READ) {
        LineReader lineReader;
        auto [ec, fileReader] = directory->CreateFileReader(metaFile, ReaderOption::NoCache(FSOT_BUFFERED));
        RETURN_IF_FS_ERROR(ec, "CreateFileReader failed");
        lineReader.Open(fileReader);
        string line;
        while (lineReader.NextLine(line, ec)) {
            string key;
            RecordInfo recordInfo;
            if (Parse(line, key, recordInfo)) {
                _metaInfos[key] = recordInfo;
            } else {
                AUTIL_LOG(DEBUG, "log file [%s] record is illegal format", _filePath.c_str());
            }
        }
        RETURN_IF_FS_ERROR(ec, "NextLine failed");
        std::tie(ec, _dataFileReader) =
            directory->CreateFileReader(dataFile, ReaderOption::NoCache(FSOT_BUFFERED)).CodeWith();
        RETURN_IF_FS_ERROR(ec, "CreateFileReader failed");
    } else if (openMode == fslib::APPEND) {
        // ... _dataFileWriter  _metaFileWriter ...
    }
    return FSEC_OK;
}
```

### 3.2. (`Write`)

`Write` `LogFile` 

1.  **
**
: `ScopedLock` 
2.  **
**
: `.lfd` (`_dataFileWriter->GetLength()`) 
3.  **
**
: `value` `.lfd` (`_dataFileWriter->Write`)
4.  **
**
: `_dataFileWriter->Flush()`
5.  **
**
: `WriteMeta` `key` 
6.  **
**
: `RecordInfo` `_metaInfos` 

** (`LogFile::Write`)**:
```cpp
FSResult<void> LogFile::Write(const std::string& key, const char* value, int64_t valueSize) noexcept
{
    ScopedLock lock(_fileLock);
    RecordInfo recordInfo;
    recordInfo.valueOffset = _dataFileWriter->GetLength();
    recordInfo.valueLength = valueSize;
    RETURN_IF_FS_ERROR(_dataFileWriter->Write(value, valueSize).Code(), "write data failed");
    auto [ec, flushRet] = _dataFileWriter->Flush();
    RETURN_IF_FS_ERROR(ec, "flush data failed");
    if (!flushRet) {
        AUTIL_LOG(ERROR, "flush data file [%s.%s] failed", _filePath.c_str(), DATA_FILE_SUFFIX.c_str());
        return FSEC_ERROR;
    }
    RETURN_IF_FS_ERROR(WriteMeta(key, recordInfo), "WriteMeta failed");
    _metaInfos[key] = recordInfo;
    return FSEC_OK;
}
```

### 3.3. (`Read`)

`Read` `Open` 

1.  **
**
: 
2.  **
**
: `_metaInfos` `map` `key`
3.  **
**
: `key` 
4.  **
**
: `key` `valueOffset`  `valueLength`
5.  **
**
: `_dataFileReader->Read` `.lfd` 
6.  **
**
: 

** (`LogFile::Read`)**:
```cpp
FSResult<size_t> LogFile::Read(const std::string& key, char* readBuffer, int64_t bufferSize) noexcept
{
    ScopedLock lock(_fileLock);
    auto iter = _metaInfos.find(key);
    if (iter == _metaInfos.end()) {
        return {FSEC_OK, 0};
    }
    if (bufferSize < iter->second.valueLength) {
        // ... , ...
        return {FSEC_BADARGS, 0};
    }
    auto [ec, readLength] = _dataFileReader->Read(readBuffer, iter->second.valueLength, iter->second.valueOffset);
    RETURN2_IF_FS_ERROR(ec, readLength, "Read failed");

    if ((int64_t)readLength != iter->second.valueLength) {
        // ... , ...
        return {FSEC_ERROR, readLength};
    }
    return {FSEC_OK, readLength};
}
```

## 4. 

1.  **
**
: `Write` `_dataFileWriter->Flush()` `_metaFileWriter->Flush()` `.lfd` `.lfm` “”。WAL

2.  **
**
: `LogFile` `_metaInfos` `map` `LogFile` 

3.  **
**
: `LogFile` `key` `value` `key` `Write` `_metaInfos` 

4.  **
**
: `LogFile` `FileReader`  `FileWriter` `LogFile` 

## 5. 

`LogFile` Indexlib 

**
**
**
**
**
**

`LogFile` Indexlib 
