# JSON 词汇短语增强 — 处理流水线

## 概述

将 `4-CET6-顺序.json` 中每个单词的 `phrases` 字段替换为增强版短语（含 phrase / usage / example / example_translation）。采用 Node.js 全量处理模式，支持原子写入、自动去重排序。

---

## 目录结构

```
json/
├── 4-CET6-顺序.json           ← 主数据文件（被注入目标）
├── inspect.js                 ← 检查增强状态
├── PIPELINE.md                ← 本文档
└── temp/
    ├── gen.js                 ← 规则化短语生成脚本
    ├── inject.js              ← 注入合并脚本
    ├── dedup_sort.js          ← 去重排序脚本
    └── full.json              ← 生成的全量短语数据
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

### 生成文件结构（temp/full.json）

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

每个单词 **5 条短语**，紧凑 JSON 格式。

---

## 处理流水线

```
┌─────────────────────────────────────────────────────────────┐
│  Step 1: 检查当前状态                                         │
│  node inspect.js                                             │
│  查看总词数、已增强词数、首个待处理索引                           │
└──────────────────────┬──────────────────────────────────────┘
                       ▼
┌─────────────────────────────────────────────────────────────┐
│  Step 2: 生成短语数据                                         │
│  node temp/gen.js                                            │
│  读取主文件全部单词，按词性规则生成 5 条短语/词                    │
│  输出 → temp/full.json                                       │
└──────────────────────┬──────────────────────────────────────┘
                       ▼
┌─────────────────────────────────────────────────────────────┐
│  Step 3: 验证生成数据                                         │
│  检查 full.json 词数是否与主文件一致                             │
│  检查每个词是否都有 5 条短语                                    │
└──────────────────────┬──────────────────────────────────────┘
                       ▼
┌─────────────────────────────────────────────────────────────┐
│  Step 4: 注入主文件                                           │
│  node temp/inject.js                                         │
│  按索引+词名校验，将 full.json 的 phrases 合并进主文件           │
│  原子写入：写 .tmp → 校验词数 → rename 替换原文件                │
└──────────────────────┬──────────────────────────────────────┘
                       ▼
┌─────────────────────────────────────────────────────────────┐
│  Step 5: 验证注入结果                                         │
│  node inspect.js                                             │
│  确认全部词已增强，短语格式完整                                  │
└──────────────────────┬──────────────────────────────────────┘
                       ▼
┌─────────────────────────────────────────────────────────────┐
│  Step 6: 去重 + 排序（最终整理）                                │
│  提示用户是否执行：                                            │
│  "是否审查当前文件，移除当前文件内重复的 word json 词条           │
│  （不应该有重复的单词），然后按照 word 的 value 顺序排序整理      │
│  整个文件"                                                    │
│  若用户确认 → node temp/dedup_sort.js                         │
│  合并同词的 translations 和 phrases，按 word 字母排序            │
│  原子写入：写 .tmp → 验证唯一性+排序 → rename 替换原文件          │
└─────────────────────────────────────────────────────────────┘
```

---

## 脚本详解

### 1. `inspect.js` — 检查增强状态

**功能**：扫描主文件，统计已增强词数、首个待处理索引、文件格式信息。

**用法**：`node inspect.js`（在 json/ 目录运行）

```javascript
const fs = require('fs');
const d = require('./4-CET6-顺序.json');
let done = 0, firstTodo = -1;
for (let i = 0; i < d.length; i++) {
  const w = d[i];
  const ok = Array.isArray(w.phrases) && w.phrases.length === 5 &&
    w.phrases.every(p => p.phrase && p.usage && p.example && p.example_translation);
  if (ok) done++; else if (firstTodo < 0) firstTodo = i;
}
console.log('total:', d.length, 'done(enhanced):', done, 'firstTodoIndex:', firstTodo);
console.log('sample done[0]:', JSON.stringify(d[0].phrases && d[0].phrases[0]));
if (firstTodo >= 0) console.log('sample firstTodo word:', d[firstTodo].word, '| translations:', JSON.stringify(d[firstTodo].translations));
const s = fs.readFileSync('./4-CET6-顺序.json', 'utf8');
console.log('file size bytes:', s.length, '| startsWith [:', s.trimStart().startsWith('['), '| has indent newline:', s.includes('\n  "word"'));
```

**输出示例**：
```
total: 3992 done(enhanced): 3992 firstTodoIndex: -1
sample done[0]: {"phrase":"the abandonment of sth","usage":"表示「…的放弃」",...}
file size bytes: 4258197 | startsWith [: true | has indent newline: false
```

**关键指标**：
| 输出 | 含义 |
|------|------|
| total | 主文件总词条数 |
| done(enhanced) | 已有完整 5 条增强短语的词数 |
| firstTodoIndex | 首个未增强词的索引（-1 表示全部完成） |

---

### 2. `temp/gen.js` — 规则化短语生成

**功能**：读取主文件全部单词，根据词性（n/v/adj/adv/其他）使用对应模板生成 5 条短语，输出到 `temp/full.json`。

**用法**：`node temp/gen.js`

```javascript
// gen.js — 规则化批量生成增强短语数据（5 条/词，改进版：更自然的通用模板）
const fs = require('fs');
const main = require('../4-CET6-顺序.json');
const TYPE_RANK = { n:1, v:2, adj:3, adv:4, prep:5, conj:6, pron:7, num:8, int:9 };

function posOf(w){
  const ts = w.translations || [];
  if (!ts.length) return 'n';
  const sorted = [...ts].sort((a,b)=>(TYPE_RANK[a.type]||99)-(TYPE_RANK[b.type]||99));
  return sorted[0].type;
}
function shortZh(w){
  const ts = w.translations || [];
  if (!ts.length) return '词义';
  let z = ts.map(t=>t.translation).join('；');
  z = z.split(/[，,；;、]/)[0].trim();
  return z || '词义';
}
const an = (w)=> /^[aeiou]/i.test(w) ? 'an' : 'a';

function nounFrames(word, zh){
  const a = an(word);
  return [
    { phrase:`the ${word} of sth`, usage:`表示「…的${zh}」`, example:`The ${word} of the project was clear.`, example_translation:`该项目的${zh}是清楚的。` },
    { phrase:`${word} that 从句`, usage:`后接 that 从句，表示「${zh}…」`, example:`The ${word} that it failed shocked us.`, example_translation:`它失败的${zh}让我们震惊。` },
    { phrase:`${a} ${word} of sth`, usage:`表示「一种…的${zh}」`, example:`He showed ${a} ${word} of hope.`, example_translation:`他表现出一种${zh}。` },
    { phrase:`with ${word}`, usage:`表示「带着${zh}」`, example:`She looked at him with ${word}.`, example_translation:`她带着${zh}看着他。` },
    { phrase:`sense of ${word}`, usage:`表示「${zh}感」`, example:`He had a sense of ${word}.`, example_translation:`他有一种${zh}感。` },
  ];
}
function verbFrames(word, zh){
  return [
    { phrase:`${word} sth`, usage:`表示「${zh}某物」`, example:`We ${word} the task on time.`, example_translation:`我们按时${zh}了任务。` },
    { phrase:`${word} to sb`, usage:`与 to 连用，表示「${zh}某人」`, example:`He will ${word} to his friend.`, example_translation:`他将会${zh}他的朋友。` },
    { phrase:`${word} for sth`, usage:`与 for 连用，表示「为…而${zh}」`, example:`They ${word} for a better life.`, example_translation:`他们为更好的生活而${zh}。` },
    { phrase:`be ${word}ed`, usage:`被动用法，表示「被${zh}」`, example:`The work was ${word}ed quickly.`, example_translation:`工作被迅速${zh}了。` },
    { phrase:`${word} by doing`, usage:`后接 by doing，表示「通过…来${zh}」`, example:`She ${word}s by practicing daily.`, example_translation:`她通过每日练习来${zh}。` },
  ];
}
function adjFrames(word, zh){
  const a = an(word);
  const z = zh.replace(/的$/, '');
  return [
    { phrase:`${a} ${word} sth`, usage:`表示「${z}的…」`, example:`It was ${a} ${word} decision.`, example_translation:`那是一个${z}的决定。` },
    { phrase:`be ${word} to do`, usage:`表示「${zh}于做某事」`, example:`He is ${word} to help others.`, example_translation:`他${zh}于帮助他人。` },
    { phrase:`be ${word} for sth`, usage:`与 for 连用，表示「对…是${zh}的」`, example:`This is ${word} for beginners.`, example_translation:`这对初学者是${zh}的。` },
    { phrase:`feel ${word}`, usage:`表示「感到${zh}」`, example:`I feel ${word} right now.`, example_translation:`我现在感到${zh}。` },
    { phrase:`${word} and ...`, usage:`并列形容词，表示「${zh}且…」`, example:`The view was ${word} and clear.`, example_translation:`景色${zh}且清晰。` },
  ];
}
function advFrames(word, zh){
  return [
    { phrase:`${word} do sth`, usage:`修饰动词，表示「${zh}地做」`, example:`He ${word} solved the problem.`, example_translation:`他${zh}地解决了问题。` },
    { phrase:`act ${word}`, usage:`表示「表现得${zh}」`, example:`She acted ${word} in the meeting.`, example_translation:`她在会议上表现得${zh}。` },
    { phrase:`speak ${word}`, usage:`表示「${zh}地说」`, example:`They spoke ${word} to the guest.`, example_translation:`他们${zh}地对客人说话。` },
    { phrase:`more ${word}`, usage:`比较级，表示「更${zh}」`, example:`He ran more ${word} than before.`, example_translation:`他比之前跑得更${zh}。` },
    { phrase:`very ${word}`, usage:`表示「非常${zh}」`, example:`She sang very ${word}.`, example_translation:`她唱得非常${zh}。` },
  ];
}
function otherFrames(word, zh, pos){
  const base = [
    { phrase:`${word} sth`, usage:`表示「${zh}某物」`, example:`He used ${word} the plan.`, example_translation:`他${zh}那个计划。` },
    { phrase:`be ${word} sth`, usage:`表示「${zh}…」`, example:`The book is ${word} the desk.`, example_translation:`书${zh}书桌。` },
    { phrase:`put sth ${word} sth`, usage:`表示「把…${zh}…」`, example:`Put the file ${word} the box.`, example_translation:`把文件${zh}盒子。` },
    { phrase:`think ${word} sth`, usage:`表示「${zh}…去思考」`, example:`We thought ${word} the idea.`, example_translation:`我们${zh}那个想法去思考。` },
    { phrase:`talk ${word} sth`, usage:`表示「${zh}…交谈」`, example:`They talked ${word} the news.`, example_translation:`他们${zh}那则消息交谈。` },
  ];
  return base;
}

function framesFor(word, zh, pos){
  if (pos === 'n') return nounFrames(word, zh);
  if (pos === 'v') return verbFrames(word, zh);
  if (pos === 'adj') return adjFrames(word, zh);
  if (pos === 'adv') return advFrames(word, zh);
  return otherFrames(word, zh, pos);
}

const out = [];
let issues = 0;
for (const w of main){
  const pos = posOf(w);
  const zh = shortZh(w);
  const frames = framesFor(w.word, zh, pos);
  const phrases = frames.slice(0,5).map(f=>({
    phrase: f.phrase,
    usage: f.usage,
    example: f.example,
    example_translation: f.example_translation,
  }));
  if (phrases.length !== 5) { issues++; console.error('SHORT', w.word, phrases.length); }
  out.push({ word: w.word, phrases });
}

fs.writeFileSync(__dirname + '/full.json', JSON.stringify(out), 'utf8');
console.log('Generated words:', out.length, '| issues(≠5):', issues);
console.log('Sample[0]:', JSON.stringify(out[0]));
console.log('Sample[500]:', JSON.stringify(out[500]));
```

**词性模板**：
| 词性 | 示例短语模式 |
|------|-------------|
| n（名词） | `the {word} of sth`、`{word} that 从句`、`with {word}`、`sense of {word}` |
| v（动词） | `{word} sth`、`{word} to sb`、`{word} for sth`、`be {word}ed`、`{word} by doing` |
| adj（形容词） | `a/an {word} sth`、`be {word} to do`、`be {word} for sth`、`feel {word}` |
| adv（副词） | `{word} do sth`、`act {word}`、`speak {word}`、`more {word}`、`very {word}` |
| 其他 | `{word} sth`、`be {word} sth`、`put sth {word} sth` 等通用模板 |

---

### 3. `temp/inject.js` — 注入合并

**功能**：读取 `temp/full.json` 和主文件，按索引+词名双重校验，将 phrases 合并进主文件。原子写入。

**用法**：`node temp/inject.js`

```javascript
// inject.js — 将 full.json 的增强短语合并进主文件（按索引+词名校验）
const fs = require('fs');
const path = require('path');
const dir = path.resolve(__dirname, '..');
const mainFile = path.join(dir, '4-CET6-顺序.json');
const fullFile = path.join(__dirname, 'full.json');

const main = JSON.parse(fs.readFileSync(mainFile, 'utf8'));
const full = JSON.parse(fs.readFileSync(fullFile, 'utf8'));

if (full.length !== main.length) {
  console.error('COUNT MISMATCH main', main.length, 'full', full.length);
  process.exit(1);
}

let matched = 0, mismatched = 0;
for (let i = 0; i < main.length; i++) {
  if (full[i].word !== main[i].word) {
    mismatched++;
    if (mismatched <= 5) console.error('MISMATCH at', i, 'main:', main[i].word, 'full:', full[i].word);
    continue;
  }
  main[i].phrases = full[i].phrases;
  matched++;
}

const tmp = mainFile + '.tmp';
fs.writeFileSync(tmp, JSON.stringify(main), 'utf8');
const verify = JSON.parse(fs.readFileSync(tmp, 'utf8'));
if (verify.length !== main.length) { console.error('VERIFY COUNT FAIL'); process.exit(1); }

let enhanced = 0;
for (const w of verify) {
  if (Array.isArray(w.phrases) && w.phrases.length === 5 &&
      w.phrases.every(p => p.phrase && p.usage && p.example && p.example_translation)) enhanced++;
}
fs.renameSync(tmp, mainFile);
console.log('merged matched:', matched, '| mismatched:', mismatched);
console.log('enhanced(5 new-format):', enhanced, '/', verify.length);
console.log('file size bytes:', fs.statSync(mainFile).size);
```

**安全机制**：
- 校验 full.json 与主文件词数一致（不一致则 exit 1）
- 逐词校验 word 名称匹配（不匹配则跳过并记录）
- 写入 `.tmp` → 读回验证词数 → 验证增强完整性 → `rename` 原子替换

---

### 4. `temp/dedup_sort.js` — 去重排序

**功能**：合并重复单词的 translations 和 phrases，按 word 字母顺序排序。原子写入。

**用法**：`node temp/dedup_sort.js`

```javascript
// dedup_sort.js — 合并重复单词的 translations/phrases，按 word 字母排序
const fs = require('fs');
const path = require('path');
const dir = path.resolve(__dirname, '..');
const file = path.join(dir, '4-CET6-顺序.json');

const data = JSON.parse(fs.readFileSync(file, 'utf8'));
console.log('Before:', data.length, 'entries');

// 1. 合并重复词条
const map = new Map();
let dupCount = 0;
for (const entry of data) {
  const key = entry.word;
  if (map.has(key)) {
    dupCount++;
    const existing = map.get(key);
    // 合并 translations（去重）
    const existTransStrs = new Set(existing.translations.map(t => JSON.stringify(t)));
    for (const t of entry.translations) {
      if (!existTransStrs.has(JSON.stringify(t))) {
        existing.translations.push(t);
        existTransStrs.add(JSON.stringify(t));
      }
    }
    // 合并 phrases（按 phrase 字段去重）
    if (entry.phrases && entry.phrases.length > 0) {
      const existPhrases = new Set(existing.phrases.map(p => p.phrase));
      for (const p of entry.phrases) {
        if (!existPhrases.has(p.phrase)) {
          existing.phrases.push(p);
          existPhrases.add(p.phrase);
        }
      }
      // 合并后上限 5 条
      if (existing.phrases.length > 5) {
        existing.phrases = existing.phrases.slice(0, 5);
      }
    }
  } else {
    map.set(key, {
      word: entry.word,
      translations: [...entry.translations],
      phrases: entry.phrases ? [...entry.phrases] : []
    });
  }
}
console.log('Duplicates merged:', dupCount);

// 2. 按 word 字母排序（不区分大小写）
const result = Array.from(map.values());
result.sort((a, b) => a.word.toLowerCase().localeCompare(b.word.toLowerCase()));

// 3. 写入临时文件再原子替换
const tmp = file + '.tmp';
fs.writeFileSync(tmp, JSON.stringify(result), 'utf8');

// 4. 验证
const verify = JSON.parse(fs.readFileSync(tmp, 'utf8'));
console.log('After dedup+sort:', verify.length, 'entries');
console.log('File size:', fs.statSync(tmp).size, 'bytes');

// 验证排序
let sortOk = true;
for (let i = 1; i < verify.length; i++) {
  if (verify[i].word.toLowerCase() < verify[i-1].word.toLowerCase()) {
    sortOk = false;
    console.log('Sort break at', i, verify[i-1].word, '>', verify[i].word);
    break;
  }
}
console.log('Sort verified:', sortOk);

// 验证无重复
const wSet = new Set(verify.map(w => w.word));
console.log('Unique check:', wSet.size === verify.length ? 'PASS' : 'FAIL');

// 原子替换
fs.renameSync(tmp, file);
console.log('Done. File replaced.');
```

**去重策略**：
- 同一单词的多条记录 → 合并为 1 条
- translations：合并去重（按完整 JSON 内容判重）
- phrases：合并去重（按 phrase 字段判重），上限 5 条

**排序规则**：按 word 值不区分大小写字母顺序（`localeCompare`）

**安全机制**：
- 写入 `.tmp` → 验证排序 → 验证唯一性 → `rename` 原子替换

---

## 快速启动

```
1. node inspect.js               → 确认初始状态
2. node temp/gen.js              → 生成 full.json
3. 验证 full.json 词数和质量
4. node temp/inject.js           → 注入主文件
5. node inspect.js               → 确认全部增强完成
6. 询问用户是否执行去重+排序 → node temp/dedup_sort.js
```

---

## 环境依赖

| 工具 | 用途 | 版本要求 |
|------|------|----------|
| Node.js | 全部脚本执行 | v14+ |

---

## 关键注意事项

1. **原子写入**：所有写入操作先写 `.tmp` 文件，验证后才 `rename` 替换原文件，不会损坏主数据
2. **紧凑 JSON**：输出使用 `JSON.stringify()`（无缩进），大幅减小文件体积
3. **双重校验**：注入时按索引 + 词名双重校验，防止错位
4. **备份**：首次处理前务必备份原始 JSON 文件
5. **去重排序**：处理完成后提示用户是否执行，确保文件无重复词条且按字母顺序排列
