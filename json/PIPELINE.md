# JSON 词汇短语增强 — 批量处理流水线

## 概述

将 JSON 词汇文件中每个单词的 `phrases` 字段替换为增强版短语（含 phrase / usage / example / example_translation），支持断点续传、原子写入、自动修复格式错误。

**数据规模**：7 个文件，共 54,356 词

| 文件 | 词数 |
|------|------|
| 1-初中-顺序.json | 3,223 |
| 2-高中-顺序.json | 6,008 |
| 3-CET4-顺序.json | 7,508 |
| 4-CET6-顺序.json | 5,651 |
| 5-考研-顺序.json | 9,602 |
| 6-托福-顺序.json | 13,477 |
| 7-SAT-顺序.json | 8,887 |

---

## 目录结构

```
json/
├── 1-初中-顺序.json          ← 主数据文件（被注入目标）
├── 2-高中-顺序.json
├── ...
├── inject_batch.ps1           ← 核心注入脚本
├── run_inject.ps1             ← 注入包装器（配置当前批次参数）
├── list_next.ps1              ← 列出下一批待处理单词
├── progress.json              ← 断点续传进度追踪
├── PIPELINE.md                ← 本文档
└── temp/
    ├── fix_batch.js           ← Node.js 格式修复脚本
    ├── batch_701_800.json     ← 当前批次数据
    └── ...
```

---

## 数据格式

### 主文件结构（每个单词）

```json
{
  "word": "ability",
  "translations": [
    { "translation": "能力，能耐；才能", "type": "n" }
  ],
  "phrases": [
    {
      "phrase": "ability to do sth",
      "usage": "后接动词不定式，表示「做某事的能力」",
      "example": "She has the ability to solve complex problems.",
      "example_translation": "她有解决复杂问题的能力。"
    }
  ]
}
```

### 批次文件结构（temp/batch_NNN_MMM.json）

```json
[
  {
    "word": "ability",
    "phrases": [
      {
        "phrase": "ability to do sth",
        "usage": "表示「做某事的能力」",
        "example": "She has the ability to solve complex problems.",
        "example_translation": "她有解决复杂问题的能力。"
      }
    ]
  }
]
```

每个单词 **5 条短语**，紧凑 JSON 格式（每词一行）。

---

## 处理流水线（每批 100 词）

```
┌─────────────────────────────────────────────────────────────┐
│  Step 1: 获取单词列表                                        │
│  修改 list_next.ps1 中的 startIdx，运行获取待处理单词及词性     │
└──────────────────────┬──────────────────────────────────────┘
                       ▼
┌─────────────────────────────────────────────────────────────┐
│  Step 2: AI 生成批次数据                                      │
│  根据单词列表 + 词性 + 中文释义，生成 batch_NNN_MMM.json        │
│  格式：紧凑 JSON，每词一行，5 条短语/词                         │
└──────────────────────┬──────────────────────────────────────┘
                       ▼
┌─────────────────────────────────────────────────────────────┐
│  Step 3: 自动修复格式错误                                      │
│  node temp/fix_batch.js temp/batch_NNN_MMM.json              │
│  修复 Write 工具产生的 「 字符编码问题（约 15 处/批）            │
└──────────────────────┬──────────────────────────────────────┘
                       ▼
┌─────────────────────────────────────────────────────────────┐
│  Step 4: 注入主文件                                           │
│  更新 run_inject.ps1 指向当前批次文件和 StartIndex              │
│  powershell -ExecutionPolicy Bypass -File run_inject.ps1     │
│  原子写入：写 .tmp → 校验词数 → Move-Item 替换原文件            │
└──────────────────────┬──────────────────────────────────────┘
                       ▼
┌─────────────────────────────────────────────────────────────┐
│  Step 5: 更新进度                                             │
│  更新 progress.json 的 completed_words 和 last_batch_end      │
└─────────────────────────────────────────────────────────────┘
```

---

## 脚本详解

### 1. `inject_batch.ps1` — 核心注入脚本

**功能**：读取主 JSON 文件和批次 JSON 文件，将批次中每个词的 phrases 替换到主文件中对应索引位置，原子写入。

**参数**：
| 参数 | 类型 | 说明 |
|------|------|------|
| MainFile | string | 主 JSON 文件路径 |
| BatchFile | string | 批次 JSON 文件路径 |
| StartIndex | int | 主文件中开始注入的索引 |
| PrettyPrint | bool | 是否美化输出（默认 false，使用压缩格式） |

**安全机制**：
- 逐词校验 word 名称匹配
- 写入临时文件 `.tmp`
- 读回临时文件验证词数一致
- 验证通过后才原子替换（Move-Item）

```powershell
# inject_batch.ps1
param(
    [string]$MainFile,
    [string]$BatchFile,
    [int]$StartIndex,
    [bool]$PrettyPrint = $false
)

# Read main JSON
Write-Output "Reading main file: $MainFile"
$mainContent = Get-Content $MainFile -Raw -Encoding UTF8
$mainData = $mainContent | ConvertFrom-Json
$originalCount = $mainData.Count
Write-Output "Original word count: $originalCount"

# Read batch data
Write-Output "Reading batch file: $BatchFile"
$batchContent = Get-Content $BatchFile -Raw -Encoding UTF8
$batchData = $batchContent | ConvertFrom-Json
Write-Output "Batch word count: $($batchData.Count)"

# Replace phrases for each word in the batch
for ($i = 0; $i -lt $batchData.Count; $i++) {
    $idx = $StartIndex + $i
    $batchWord = $batchData[$i].word
    $mainWord = $mainData[$idx].word
    
    if ($batchWord -ne $mainWord) {
        Write-Output "WARNING: Word mismatch at index $idx. Expected '$mainWord', got '$batchWord'"
        continue
    }
    
    # Replace phrases (handle case where word has no existing phrases property)
    if ($null -eq $mainData[$idx].PSObject.Properties['phrases']) {
        $mainData[$idx] | Add-Member -NotePropertyName 'phrases' -NotePropertyValue $batchData[$i].phrases
    } else {
        $mainData[$idx].phrases = $batchData[$i].phrases
    }
    $phraseCount = $batchData[$i].phrases.Count
    Write-Output "  [$idx] $mainWord -> $phraseCount phrases"
}

# Serialize
Write-Output "Serializing..."
if ($PrettyPrint) {
    $json = $mainData | ConvertTo-Json -Depth 20
} else {
    $json = $mainData | ConvertTo-Json -Depth 20 -Compress
}

# Write to temp file first
$tempFile = $MainFile + ".tmp"
[System.IO.File]::WriteAllText($tempFile, $json, [System.Text.Encoding]::UTF8)

# Validate temp file
Write-Output "Validating temp file..."
$testContent = Get-Content $tempFile -Raw -Encoding UTF8
$testData = $testContent | ConvertFrom-Json
if ($testData.Count -ne $originalCount) {
    Write-Output "ERROR: Word count mismatch after serialization! Expected $originalCount, got $($testData.Count)"
    exit 1
}

# Atomic replace
Move-Item -Path $tempFile -Destination $MainFile -Force
Write-Output "File updated successfully. Word count verified: $($testData.Count)"
```

### 2. `run_inject.ps1` — 注入包装器

**功能**：配置当前批次的文件路径和起始索引，调用 inject_batch.ps1。每批需更新 3 个值。

```powershell
# run_inject.ps1 — 每批更新 BatchFile 和 StartIndex
$dir = "c:\Users\webtu\Desktop\json"
$mainFile = (Get-ChildItem -Path $dir -Filter "1-*.json").FullName
$batchFile = Join-Path $dir "temp\batch_701_800.json"   # ← 每批更新

Write-Output "Main file: $mainFile"
Write-Output "Batch file: $batchFile"

& "$dir\inject_batch.ps1" -MainFile $mainFile -BatchFile $batchFile -StartIndex 701 -PrettyPrint $false
#                                                                                   ^^^ 每批更新
```

**切换文件时**：修改 `Filter "1-*.json"` 为对应文件编号（如 `"2-*.json"`）。

### 3. `list_next.ps1` — 列出待处理单词

**功能**：从主文件读取指定范围的单词，输出单词、词性信息，供 AI 生成短语时参考。

```powershell
# list_next.ps1 — 修改 startIdx 和 count 调整范围
$dir = "c:\Users\webtu\Desktop\json"
$mainFile = (Get-ChildItem -Path $dir -Filter "1-*.json").FullName
$content = Get-Content $mainFile -Raw -Encoding UTF8
$data = $content | ConvertFrom-Json
$startIdx = 801    # ← 每批更新
$count = 100       # ← 批次大小
for ($i = $startIdx; $i -lt ($startIdx + $count) -and $i -lt $data.Count; $i++) {
    $w = $data[$i]
    $types = ($w.translations | ForEach-Object { "$($_.type)" }) -join ","
    Write-Output "[$i] $($w.word) [$types]"
}
```

**输出示例**：
```
[801] not [adv,n]
[802] note [n,v]
[803] nothing [pron,n,adv]
...
```

### 4. `temp/fix_batch.js` — Node.js 格式修复脚本

**功能**：修复 Write 工具在生成大量 JSON 数据时产生的 `「` 字符编码错误。支持两种损坏模式。

**用法**：`node temp/fix_batch.js temp/batch_NNN_MMM.json`

```javascript
// fix_batch.js — 修复 Write 工具的两种格式错误
const fs = require('fs');
const file = process.argv[2];
if (!file) { console.log('Usage: node fix_batch.js <file>'); process.exit(1); }

let t = fs.readFileSync(file, 'utf8');
let fixCount = 0;

// Pattern 1: 「 was replaced by ":" (old corruption)
// "usage":"表示":"TEXT」"  →  "usage":"表示「TEXT」"
const p1 = /"usage":"表示":"([^"]*?)」/g;
t = t.replace(p1, (match, content) => { fixCount++; return `"usage":"表示「${content}」`; });

// Pattern 2: extra " between 「 and text (new corruption)  
// "usage":"表示「"TEXT」"  →  "usage":"表示「TEXT」"
const p2 = /「"([^"]*?)」/g;
t = t.replace(p2, (match, content) => { fixCount++; return `「${content}」`; });

try {
    const data = JSON.parse(t);
    console.log(`Fixed ${fixCount} corrupted entries. Word count: ${data.length}`);
    fs.writeFileSync(file, t, 'utf8');
    console.log('JSON is VALID. File saved.');
} catch(e) {
    console.log('ERROR: Still invalid after fix:', e.message);
    const pos = parseInt(e.message.match(/\d+/)[0]);
    console.log('Context:', JSON.stringify(t.substring(pos - 30, pos + 30)));
}
```

**两种损坏模式说明**：

| 模式 | 损坏表现 | 修复方法 | 频率 |
|------|----------|----------|------|
| Pattern 1 | `"usage":"表示":"TEXT」"` — `「` 被替换为 `":"` | 正则匹配 `":"TEXT」` → `「TEXT」` | ~15处/批 |
| Pattern 2 | `"usage":"表示「"TEXT」"` — `「` 后多一个 `"` | 正则匹配 `「"TEXT」` → `「TEXT」` | ~300处/批 |

### 5. `progress.json` — 断点续传进度追踪

```json
{
  "1-初中-顺序.json": {"status":"in_progress","completed_words":801,"total_words":3223,"last_batch_end":800},
  "2-高中-顺序.json": {"status":"pending","completed_words":0,"total_words":6008},
  "3-CET4-顺序.json": {"status":"pending","completed_words":0,"total_words":7508},
  "4-CET6-顺序.json": {"status":"pending","completed_words":0,"total_words":5651},
  "5-考研-顺序.json": {"status":"pending","completed_words":0,"total_words":9602},
  "6-托福-顺序.json": {"status":"pending","completed_words":0,"total_words":13477},
  "7-SAT-顺序.json": {"status":"pending","completed_words":0,"total_words":8887}
}
```

| 字段 | 说明 |
|------|------|
| status | `pending` / `in_progress` / `completed` |
| completed_words | 已完成词数（= last_batch_end + 1） |
| total_words | 文件总词数 |
| last_batch_end | 最后一批的结束索引 |

---

## 快速启动指南

### 从头开始处理某个文件

```
1. 修改 run_inject.ps1 中的 Filter 为目标文件编号
2. 设置 list_next.ps1 的 startIdx = 0
3. 运行 list_next.ps1 获取单词列表
4. 循环执行：
   a. AI 生成 batch 文件
   b. node temp/fix_batch.js temp/batch_NNN_MMM.json
   c. 更新 run_inject.ps1 的 BatchFile 和 StartIndex
   d. powershell -ExecutionPolicy Bypass -File run_inject.ps1
   e. 更新 progress.json
```

### 断点续传

```
1. 读取 progress.json 查看进度
2. 找到 status = "in_progress" 的文件
3. 设置 startIdx = last_batch_end + 1
4. 继续执行步骤 3 起的循环
```

### 切换到下一个文件

```
1. 将 progress.json 中当前文件 status 改为 "completed"
2. 将下一个文件 status 改为 "in_progress"
3. 修改 run_inject.ps1 和 list_next.ps1 的 Filter 为新文件编号
4. 设置 startIdx = 0
```

---

## 环境依赖

| 工具 | 用途 | 版本要求 |
|------|------|----------|
| PowerShell | 注入脚本执行 | Windows 内置 |
| Node.js | 格式修复脚本 | v14+ |
| AI 助手 | 生成短语数据 | 需能输出 JSON |

---

## 关键注意事项

1. **始终先修复再注入**：Write 工具会产生格式错误，必须先用 `fix_batch.js` 修复
2. **原子写入**：注入脚本先写 `.tmp` 文件，验证后才替换原文件，不会损坏主数据
3. **紧凑 JSON**：注入时使用 `-PrettyPrint $false`（`-Compress`），大幅减小文件体积
4. **批次大小**：推荐 100 词/批，Write 工具单次输出上限约 1000 行
5. **备份**：首次处理前务必备份全部原始 JSON 文件
6. **切换文件**：修改 `run_inject.ps1` 和 `list_next.ps1` 中的文件 Filter 模式
