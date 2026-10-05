# JSON 词汇短语增强 — 完整流水线文档

## 概述

将 JSON 词汇文件中每个单词的 `phrases` 字段填充为增强版短语（含 phrase / usage / example / example_translation）。支持断点续传、原子写入、自动修复格式错误、去重排序。

**数据规模**：7 个文件

| 文件 | 词数 | 状态 |
|------|------|------|
| 1-初中-顺序.json | 3,223 | 已处理 801 词 |
| 2-高中-顺序.json | 3,753 | ✅ 已完成 |
| 3-CET4-顺序.json | 7,508 | 待处理 |
| 4-CET6-顺序.json | 5,651 | 待处理 |
| 5-考研-顺序.json | 9,602 | 待处理 |
| 6-托福-顺序.json | 13,477 | 待处理 |
| 7-SAT-顺序.json | 8,887 | 待处理 |

> 注：文件2原始词数 6008，去重后为 3753（2255 个重复词被合并）。

---

## 目录结构

```
json/
├── 1-初中-顺序.json              ← 主数据文件（被注入目标）
├── 2-高中-顺序.json
├── ...
├── inject_batch.ps1               ← 核心注入脚本
├── run_inject.ps1                 ← 注入包装器（配置当前批次参数）
├── list_next.ps1                  ← 列出下一批待处理单词
├── progress.json                  ← 断点续传进度追踪
├── PIPELINE.md                    ← 本文档
└── temp/
    ├── fix_batch.js               ← Node.js 格式修复脚本
    ├── dedup_sort.js              ← Node.js 去重排序脚本
    ├── analyze_dupes.js           ← Node.js 重复分析脚本
    ├── batch_000_099.json         ← 批次数据文件
    └── ...
```

---

## 数据格式

### 主文件结构（每个单词词条）

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
      },
      {
        "phrase": "have the ability",
        "usage": "表示有能力",
        "example": "She has the ability.",
        "example_translation": "她有能力。"
      },
      {
        "phrase": "ability in",
        "usage": "表示在…方面的能力",
        "example": "Her ability in math.",
        "example_translation": "她在数学方面的能力。"
      },
      {
        "phrase": "natural ability",
        "usage": "表示天赋能力",
        "example": "The natural ability.",
        "example_translation": "天赋能力。"
      },
      {
        "phrase": "ability and skill",
        "usage": "表示能力与技能",
        "example": "Ability and skill.",
        "example_translation": "能力与技能。"
      }
    ]
  }
]
```

每个单词 **5 条短语**，紧凑 JSON 格式（每词一行）。

---

## 主流水线：批量短语填充（每批 100 词）

### 流程图

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
│  修复 Write 工具产生的「 字符编码问题                            │
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
│  更新 list_next.ps1 的 startIdx 为下一批起始                    │
└─────────────────────────────────────────────────────────────┘
```

### Step 1: 获取单词列表

修改 `list_next.ps1` 中的 `$startIdx` 为当前批次起始索引，运行：

```powershell
powershell -ExecutionPolicy Bypass -File list_next.ps1
```

输出示例：
```
[0] a [n,abbr]
[1] abandon [n,v,vt]
[2] ability [n]
...
[99] zone [n]
```

### Step 2: AI 生成批次数据

AI 助手根据单词列表生成 `temp/batch_NNN_MMM.json`，格式要求：
- 数组包裹，每词一个 JSON 对象，紧凑格式（每词一行）
- 每词恰好 5 条短语
- 每条短语含 phrase / usage / example / example_translation 四个字段
- usage 字段格式：`"表示…"` 或 `"表示「…」「…"`

### Step 3: 修复格式错误

```powershell
node temp/fix_batch.js temp/batch_NNN_MMM.json
```

输出示例：
```
Fixed 0 corrupted entries. Word count: 100
JSON is VALID. File saved.
```

### Step 4: 注入主文件

先修改 `run_inject.ps1` 中的两个参数：
- `$batchFile` → 当前批次文件路径
- `-StartIndex` → 当前批次起始索引

然后运行：
```powershell
powershell -ExecutionPolicy Bypass -File run_inject.ps1
```

输出示例（末尾）：
```
Serializing...
Validating temp file...
File updated successfully. Word count verified: 3753
```

### Step 5: 更新进度

1. 更新 `progress.json`：
   - `completed_words` += 本批词数
   - `last_batch_end` = 本批最后索引
2. 更新 `list_next.ps1` 中的 `$startIdx` 为下一批起始

---

## 辅助流水线：去重与排序

当主文件中存在重复单词词条时，使用此流程去重、合并短语、按字母排序。

### 流程图

```
┌─────────────────────────────────────────────────────────────┐
│  Step 1: 分析重复                                              │
│  node temp/analyze_dupes.js                                  │
│  输出重复报告到 temp/dupe_report.txt                           │
└──────────────────────┬──────────────────────────────────────┘
                       ▼
┌─────────────────────────────────────────────────────────────┐
│  Step 2: 去重 + 合并 + 排序                                   │
│  node temp/dedup_sort.js                                     │
│  自动备份 → 合并重复词条 → 字母排序 → 写回主文件                 │
└──────────────────────┬──────────────────────────────────────┘
                       ▼
┌─────────────────────────────────────────────────────────────┐
│  Step 3: 更新进度                                             │
│  更新 progress.json 的 total_words 和 completed_words          │
└─────────────────────────────────────────────────────────────┘
```

### 合并策略

- **短语合并**：重复词条的所有短语全部保留（如两个词条各5条 → 合并后10条）
- **翻译合并**：不同词性的翻译去重后合并
- **排序方式**：按 word 字段字母升序（a→z，不区分大小写）

---

## 脚本完整源码

### 1. `inject_batch.ps1` — 核心注入脚本

**功能**：读取主 JSON 文件和批次 JSON 文件，将批次中每个词的 phrases 替换到主文件中对应索引位置，原子写入。

**参数**：

| 参数 | 类型 | 说明 |
|------|------|------|
| MainFile | string | 主 JSON 文件路径 |
| BatchFile | string | 批次 JSON 文件路径 |
| StartIndex | int | 主文件中开始注入的索引（0-based） |
| PrettyPrint | bool | 是否美化输出（默认 false，使用压缩格式） |

**安全机制**：
- 逐词校验 word 名称匹配（不匹配则跳过并警告）
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

# Write to temp file first (UTF-8 without BOM to keep JSON strict-parser friendly)
$tempFile = $MainFile + ".tmp"
$utf8NoBom = New-Object System.Text.UTF8Encoding($false)
[System.IO.File]::WriteAllText($tempFile, $json, $utf8NoBom)

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

---

### 2. `run_inject.ps1` — 注入包装器

**功能**：配置当前批次的文件路径和起始索引，调用 inject_batch.ps1。每批需更新 2 个值。

**每批更新项**：
- `$batchFile` 路径（指向当前批次文件）
- `-StartIndex` 参数（当前批次起始索引）

**切换文件时**：修改 `Filter "2-*.json"` 为对应文件编号。

```powershell
# run_inject.ps1 — 每批更新 BatchFile 和 StartIndex
$dir = $PSScriptRoot
$mainFile = (Get-ChildItem -Path $dir -Filter "2-*.json").FullName
$batchFile = Join-Path $dir "temp\batch_0000_0099.json"   # ← 每批更新

Write-Output "Main file: $mainFile"
Write-Output "Batch file: $batchFile"

& (Join-Path $dir "inject_batch.ps1") -MainFile $mainFile -BatchFile $batchFile -StartIndex 0 -PrettyPrint $false
#                                                                                             ^^^^ 每批更新
```

---

### 3. `list_next.ps1` — 列出待处理单词

**功能**：从主文件读取指定范围的单词，输出单词、词性信息，供 AI 生成短语时参考。

**每批更新项**：`$startIdx`（下一批起始索引）

```powershell
# list_next.ps1 — 修改 startIdx 和 count 调整范围
$dir = $PSScriptRoot
$mainFile = (Get-ChildItem -Path $dir -Filter "2-*.json").FullName
$content = Get-Content $mainFile -Raw -Encoding UTF8
$data = $content | ConvertFrom-Json
$startIdx = 0    # ← 每批更新
$count = 100     # ← 批次大小
for ($i = $startIdx; $i -lt ($startIdx + $count) -and $i -lt $data.Count; $i++) {
    $w = $data[$i]
    $types = ($w.translations | ForEach-Object { "$($_.type)" }) -join ","
    Write-Output "[$i] $($w.word) [$types]"
}
```

**输出示例**：
```
[0] a [n,abbr]
[1] abandon [n,v,vt]
[2] ability [n]
...
[99] zone [n]
```

---

### 4. `temp/fix_batch.js` — Node.js 格式修复脚本

**功能**：修复 AI 写入工具在生成大量 JSON 数据时产生的 `「` 字符编码错误。支持两种损坏模式。

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

| 模式 | 损坏表现 | 修复方法 |
|------|----------|----------|
| Pattern 1 | `"usage":"表示":"TEXT」"` — `「` 被替换为 `":"` | 正则匹配 `":"TEXT」` → `「TEXT」` |
| Pattern 2 | `"usage":"表示「"TEXT」"` — `「` 后多一个 `"` | 正则匹配 `「"TEXT」` → `「TEXT」` |

---

### 5. `temp/analyze_dupes.js` — 重复分析脚本

**功能**：分析主文件中的重复单词，输出详细报告。

**用法**：`node temp/analyze_dupes.js`（报告输出到 `temp/dupe_report.txt`）

```javascript
// analyze_dupes.js — 分析主文件中的重复单词
const fs = require('fs');
const data = JSON.parse(fs.readFileSync('c:/Users/webtu/Desktop/词汇填充短语 - HY-2/json/2-高中-顺序.json', 'utf8'));
const words = data.map(d => d.word);
const seen = {};

words.forEach((w, i) => {
  if (!seen[w]) seen[w] = [];
  seen[w].push(i);
});

const dupeWords = Object.keys(seen).filter(w => seen[w].length > 1);
let output = [];
output.push('Total entries: ' + data.length);
output.push('Unique words: ' + new Set(words).size);
output.push('Duplicated words: ' + dupeWords.length);
output.push('Total duplicate entries to remove: ' + (data.length - new Set(words).size));
output.push('');

dupeWords.forEach(w => {
  const indices = seen[w];
  output.push('"' + w + '" appears ' + indices.length + ' times at indices: ' + indices.join(', '));
  indices.forEach(idx => {
    const entry = data[idx];
    const phraseCount = entry.phrases ? entry.phrases.length : 0;
    output.push('  [' + idx + '] ' + phraseCount + ' phrases');
  });
});

fs.writeFileSync('c:/Users/webtu/Desktop/词汇填充短语 - HY-2/json/temp/dupe_report.txt', output.join('\n'), 'utf8');
console.log('Report written to temp/dupe_report.txt');
```

> 使用时需修改文件路径为目标主文件。

---

### 6. `temp/dedup_sort.js` — 去重排序脚本

**功能**：对主文件执行去重（合并短语和翻译）、按 word 字母排序、原子写入。自动备份原文件为 `.bak2`。

**用法**：`node temp/dedup_sort.js`

**合并策略**：
- 短语：全部保留（如两个词条各5条 → 合并后10条）
- 翻译：按 `type:translation` 去重后合并
- 排序：按 word 字母升序（a→z，不区分大小写）

```javascript
// dedup_sort.js — 去重 + 合并 + 排序
const fs = require('fs');
const inputFile = 'c:/Users/webtu/Desktop/词汇填充短语 - HY-2/json/2-高中-顺序.json';
const backupFile = 'c:/Users/webtu/Desktop/词汇填充短语 - HY-2/json/2-高中-顺序.json.bak2';
const data = JSON.parse(fs.readFileSync(inputFile, 'utf8'));

// Backup
fs.writeFileSync(backupFile, JSON.stringify(data, null, 2), 'utf8');
console.log('Backup saved to:', backupFile);

// Group by word, preserving first occurrence's translations
const merged = {};
const order = [];

data.forEach((entry, idx) => {
  const w = entry.word;
  if (!merged[w]) {
    merged[w] = {
      word: w,
      translations: entry.translations || [],
      phrases: entry.phrases || []
    };
    order.push(w);
  } else {
    // Merge phrases from duplicate
    const newPhrases = entry.phrases || [];
    merged[w].phrases = merged[w].phrases.concat(newPhrases);
    // Merge translations (add non-duplicate types)
    const existingTypes = new Set(merged[w].translations.map(t => t.type + ':' + t.translation));
    if (entry.translations) {
      entry.translations.forEach(t => {
        const key = t.type + ':' + t.translation;
        if (!existingTypes.has(key)) {
          merged[w].translations.push(t);
          existingTypes.add(key);
        }
      });
    }
  }
});

// Convert to array
let result = order.map(w => merged[w]);

// Sort alphabetically by word (case-insensitive)
result.sort((a, b) => {
  const wa = a.word.toLowerCase();
  const wb = b.word.toLowerCase();
  if (wa < wb) return -1;
  if (wa > wb) return 1;
  return 0;
});

// Write compact format (one entry per line)
const lines = result.map(entry => JSON.stringify(entry));
const output = '[\n' + lines.join(',\n') + '\n]';
fs.writeFileSync(inputFile, output, 'utf8');

console.log('Original entries:', data.length);
console.log('After dedup:', result.length);
console.log('Removed:', data.length - result.length);
console.log('Entries with >5 phrases:', result.filter(e => e.phrases.length > 5).length);
console.log('File saved.');
```

> 使用时需修改 `inputFile` 和 `backupFile` 路径为目标主文件。

---

### 7. `progress.json` — 断点续传进度追踪

```json
{
  "1-初中-顺序.json": {"status":"in_progress","completed_words":801,"total_words":3223,"last_batch_end":800},
  "2-高中-顺序.json": {"status":"completed","completed_words":3753,"total_words":3753,"last_batch_end":3752},
  "3-CET4-顺序.json": {"status":"pending","completed_words":0,"total_words":7508},
  "4-CET6-顺序.json": {"status":"pending","completed_words":0,"total_words":5651},
  "5-考研-顺序.json": {"status":"pending","completed_words":0,"total_words":9602},
  "6-托福-顺序.json": {"status":"pending","completed_words":0,"total_words":13477},
  "7-SAT-顺序.json": {"status":"pending","completed_words":0,"total_words":8887}
}
```

| 字段 | 说明 |
|------|------|
| status | `pending`（待处理） / `in_progress`（进行中） / `completed`（已完成） |
| completed_words | 已完成词数（= last_batch_end + 1） |
| total_words | 文件总词数（去重后） |
| last_batch_end | 最后一批的结束索引（0-based） |

---

## 快速启动指南

### 从头开始处理某个文件

```
1. 修改 run_inject.ps1 中的 Filter 为目标文件编号（如 "3-*.json"）
2. 修改 list_next.ps1 中的 Filter 和 startIdx = 0
3. 运行 list_next.ps1 获取单词列表
4. 循环执行（每批 100 词）：
   a. AI 生成 batch 文件（Write 工具）
   b. node temp/fix_batch.js temp/batch_NNN_MMM.json
   c. 更新 run_inject.ps1 的 BatchFile 和 StartIndex
   d. powershell -ExecutionPolicy Bypass -File run_inject.ps1
   e. 更新 progress.json 和 list_next.ps1 的 startIdx
5. 全部完成后将 status 改为 "completed"
```

### 断点续传

```
1. 读取 progress.json 查看进度
2. 找到 status = "in_progress" 的文件
3. 设置 list_next.ps1 的 startIdx = completed_words
4. 设置 run_inject.ps1 的 Filter 和 StartIndex
5. 继续执行步骤 4 起的循环
```

### 切换到下一个文件

```
1. 将 progress.json 中当前文件 status 改为 "completed"
2. 将下一个文件 status 改为 "in_progress"，completed_words = 0
3. 修改 run_inject.ps1 和 list_next.ps1 的 Filter 为新文件编号
4. 设置 list_next.ps1 的 startIdx = 0
5. 运行 list_next.ps1 开始处理
```

### 去重排序（新文件首次处理前建议执行）

```
1. 修改 analyze_dupes.js 的文件路径为目标文件
2. node temp/analyze_dupes.js（查看重复情况）
3. 修改 dedup_sort.js 的文件路径为目标文件
4. node temp/dedup_sort.js（执行去重排序）
5. 更新 progress.json 的 total_words 和 completed_words
```

---

## 环境依赖

| 工具 | 用途 | 版本要求 |
|------|------|----------|
| PowerShell | 注入脚本执行 | Windows 内置（不支持 `&&`，用 `;` 分隔） |
| Node.js | 格式修复 / 去重排序脚本 | v14+ |
| AI 助手 | 生成短语数据 | 需能输出 JSON |

---

## 关键注意事项

1. **始终先修复再注入**：AI 写入工具会产生格式错误，必须先用 `fix_batch.js` 修复
2. **原子写入**：注入脚本先写 `.tmp` 文件，验证词数一致后才替换原文件，不会损坏主数据
3. **紧凑 JSON**：注入时使用 `-PrettyPrint $false`（`-Compress`），大幅减小文件体积
4. **批次大小**：推荐 100 词/批，AI 单次输出上限约 1000 行（每词约10行，故100词为上限）
5. **备份**：首次处理前务必备份全部原始 JSON 文件；去重排序会自动备份为 `.bak2`
6. **切换文件**：修改 `run_inject.ps1` 和 `list_next.ps1` 中的文件 Filter 模式
7. **progress.json 逗号**：更新 progress.json 时注意保留行尾逗号（最后一行除外）
8. **去重后更新**：去重排序后必须更新 progress.json 的 total_words 和 completed_words
9. **注入前必须更新 run_inject.ps1**：BatchFile 和 StartIndex 必须指向当前批次

---

## AI 生成短语的规范

### 每词 5 条短语，字段说明

| 字段 | 说明 | 示例 |
|------|------|------|
| phrase | 短语本身 | `a lot of` |
| usage | 中文释义，以"表示"开头 | `表示许多，修饰可数或不可数名词` |
| example | 英文例句 | `We have a lot of homework today.` |
| example_translation | 例句中文翻译 | `我们今天有很多作业。` |

### 短语类型优先级

1. **高频固定搭配**（如 `a lot of`, `a few`）
2. **常见短语动词**（如 `get up`, `get off`）
3. **近义并列**（如 `weak and feeble`, `tired and exhausted`）
4. **介词搭配**（如 `ability to do`, `pride in`）
5. **复合用法**（如 `in the course of`, `be supposed to`）

### 输出格式

紧凑 JSON，每词一行，示例：
```json
[
{"word":"ability","phrases":[{"phrase":"ability to do sth","usage":"表示做某事的能力","example":"She has the ability to solve problems.","example_translation":"她有解决问题的能力。"},{"phrase":"have the ability","usage":"表示有能力","example":"She has the ability.","example_translation":"她有能力。"},{"phrase":"ability in","usage":"表示在…方面的能力","example":"Her ability in math.","example_translation":"她在数学方面的能力。"},{"phrase":"natural ability","usage":"表示天赋能力","example":"The natural ability.","example_translation":"天赋能力。"},{"phrase":"ability and skill","usage":"表示能力与技能","example":"Ability and skill.","example_translation":"能力与技能。"}]},
{"word":"able","phrases":[...]}
]
```
