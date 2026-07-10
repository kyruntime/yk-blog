# Developer Pronunciation Blog Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a Chinese blog post for Java, Python, and Vue developers with an interactive pronunciation table whose English terms, IPA cells, and speaker buttons can pronounce the word.

**Architecture:** Add a focused MDX post that imports a reusable Astro component and structured term data. The component renders search, category filters, and a responsive table, then uses a small inline client script with the browser `SpeechSynthesis` API for click-to-speak behavior.

**Tech Stack:** Astro 6, MDX, TypeScript, Tailwind CSS classes already used by AstroPaper, browser `SpeechSynthesis`.

## Global Constraints

- The post is a blog article, not a standalone English learning platform.
- Do not ship custom audio files or external pronunciation service dependencies.
- Each pronunciation row must include a Chinese pronunciation approximation.
- Do not force every sound into Chinese characters when that would mislead readers.
- Mixed notation is allowed for tricky sounds such as `th`, `ki`, `kee`, and unstressed `uh`.
- `schema` must use `/ˈskiː.mə/` and a Chinese-friendly approximation like `斯 kee-muh`, not `斯基-ma` or `斯给-ma`.
- English terms, IPA cells, and speaker buttons must all trigger pronunciation of the same English term or `speakText` override.
- Search must cover English term, Chinese meaning, Chinese approximation, and common mistake.
- Category filtering must be available.
- The page must fit existing AstroPaper article typography and mobile behavior.

---

## File Structure

- Create `src/content/posts/developer-pronunciation-java-python-vue.mdx`
  - The blog article. Imports `PronunciationTable` and `pronunciationTerms`.
- Create `src/content/posts/_components/PronunciationTable.astro`
  - Interactive table UI, search/filter controls, and client-side speech behavior.
- Create `src/content/posts/_data/pronunciationTerms.ts`
  - Structured word list and TypeScript types.
- No global layout changes are expected.

---

### Task 1: Pronunciation Data

**Files:**
- Create: `src/content/posts/_data/pronunciationTerms.ts`

**Interfaces:**
- Produces: `type PronunciationCategory`
- Produces: `type PronunciationTerm`
- Produces: `const pronunciationCategories: readonly PronunciationCategory[]`
- Produces: `const pronunciationTerms: readonly PronunciationTerm[]`

- [ ] **Step 1: Create the data module**

Create `src/content/posts/_data/pronunciationTerms.ts` with this structure and initial rows. Keep every `zhPronunciation` field intentional; prefer mixed notation when pure Chinese characters would mislead.

```ts
export type PronunciationCategory =
  | "Java"
  | "Python"
  | "Vue"
  | "Frontend"
  | "Database"
  | "DevOps"
  | "General";

export type PronunciationTerm = {
  term: string;
  ipa: string;
  zhPronunciation: string;
  meaningZh: string;
  commonMistake: string;
  note: string;
  example?: string;
  category: PronunciationCategory;
  speakText?: string;
};

export const pronunciationCategories = [
  "Java",
  "Python",
  "Vue",
  "Frontend",
  "Database",
  "DevOps",
  "General",
] as const satisfies readonly PronunciationCategory[];

export const pronunciationTerms = [
  {
    term: "Java",
    ipa: "/ˈdʒɑː.və/",
    zhPronunciation: "扎-vuh",
    meaningZh: "Java 编程语言",
    commonMistake: "加瓦",
    note: "首音是 /dʒ/，不是中文拼音 j。",
    example: "This service is written in Java.",
    category: "Java",
  },
  {
    term: "JVM",
    ipa: "/ˌdʒeɪ viː ˈem/",
    zhPronunciation: "J-V-M",
    meaningZh: "Java 虚拟机",
    commonMistake: "j-v-m 按中文字母读",
    note: "按英文字母名逐个读。",
    example: "The JVM handles memory management.",
    category: "Java",
  },
  {
    term: "schema",
    ipa: "/ˈskiː.mə/",
    zhPronunciation: "斯 kee-muh",
    meaningZh: "模式、结构定义",
    commonMistake: "斯基马、斯给马、斯切马",
    note: "`sch` 在这里读 /sk/，`ee` 是长 /iː/，不要写成中文“基”。",
    example: "The schema migration failed.",
    category: "Python",
  },
  {
    term: "cache",
    ipa: "/kæʃ/",
    zhPronunciation: "凯什 / 开什",
    meaningZh: "缓存",
    commonMistake: "卡切、卡其",
    note: "读起来接近英文 cash。",
    example: "I will update the cache layer.",
    category: "General",
  },
  {
    term: "queue",
    ipa: "/kjuː/",
    zhPronunciation: "Q / kiu",
    meaningZh: "队列",
    commonMistake: "奎尤、库额",
    note: "虽然有五个字母，但只读一个音节。",
    example: "The async task is queued.",
    category: "General",
  },
  {
    term: "Vue",
    ipa: "/vjuː/",
    zhPronunciation: "view / viu",
    meaningZh: "Vue 前端框架",
    commonMistake: "V-U-E",
    note: "和 view 同音。",
    example: "We use Vue on the frontend.",
    category: "Vue",
    speakText: "view",
  },
] as const satisfies readonly PronunciationTerm[];
```

- [ ] **Step 2: Expand the list using the required inventory**

Replace the starter `pronunciationTerms` array with rows for every entry in "Appendix A: Required Initial Term Inventory" at the bottom of this plan. Keep the exact `schema` row from Step 1 unless later verification proves a better IPA source. Every row must have all required fields:

```ts
{
  term: "term from appendix",
  ipa: "/IPA from appendix/",
  zhPronunciation: "Chinese approximation from appendix",
  meaningZh: "Chinese meaning from appendix",
  commonMistake: "Common mistake from appendix",
  note: "One short note explaining the trap or confirming the convention.",
  example: "One realistic software sentence.",
  category: "Category from appendix",
  speakText: "Only when visible term needs a TTS override",
}
```

Use this completed example for `async`:

```ts
{
  term: "async",
  ipa: "/eɪˈsɪŋk/",
  zhPronunciation: "ay-SINK",
  meaningZh: "异步",
  commonMistake: "啊-sync",
  note: "首音是 /eɪ/，重音在 sync。",
  example: "The async task is queued.",
  category: "Python",
}
```

- [ ] **Step 3: Run type check**

Run:

```bash
npm run sync
npx astro check
```

Expected: commands complete without TypeScript errors.

- [ ] **Step 4: Commit**

```bash
git add src/content/posts/_data/pronunciationTerms.ts
git commit -m "feat: add developer pronunciation term data"
```

---

### Task 2: Interactive Pronunciation Table

**Files:**
- Create: `src/content/posts/_components/PronunciationTable.astro`

**Interfaces:**
- Consumes: `PronunciationTerm` from `src/content/posts/_data/pronunciationTerms.ts`
- Props: `{ terms: readonly PronunciationTerm[] }`
- Produces: client-side behavior for search, category filters, and speech controls.

- [ ] **Step 1: Create the Astro component**

Create `src/content/posts/_components/PronunciationTable.astro`.

```astro
---
import type { PronunciationTerm } from "../_data/pronunciationTerms";

type Props = {
  terms: readonly PronunciationTerm[];
};

const { terms } = Astro.props;
const categories = Array.from(new Set(terms.map(term => term.category)));
---

<section class="not-prose my-8" aria-label="开发者英文发音词表">
  <div class="mb-4 grid gap-3 sm:grid-cols-[1fr_auto]">
    <label class="block">
      <span class="mb-1 block text-sm font-medium text-foreground">搜索</span>
      <input
        data-pronunciation-search
        class="border-border bg-background text-foreground w-full rounded border px-3 py-2 text-sm"
        type="search"
        placeholder="搜索单词、中文含义、中文读法或常见误读"
      />
    </label>

    <label class="block">
      <span class="mb-1 block text-sm font-medium text-foreground">分类</span>
      <select
        data-pronunciation-category
        class="border-border bg-background text-foreground w-full rounded border px-3 py-2 text-sm sm:w-40"
      >
        <option value="">全部</option>
        {categories.map(category => <option value={category}>{category}</option>)}
      </select>
    </label>
  </div>

  <p
    data-pronunciation-fallback
    class="bg-muted text-muted-foreground mb-3 hidden rounded px-3 py-2 text-sm"
  >
    当前浏览器不支持语音朗读，你仍然可以参考音标和中文对照发音。
  </p>

  <div class="border-border overflow-x-auto rounded border">
    <table class="m-0 min-w-[920px] text-sm">
      <thead class="bg-muted/50">
        <tr>
          <th class="w-32 text-left">单词</th>
          <th class="w-32 text-left">音标</th>
          <th class="w-36 text-left">中文对照发音</th>
          <th class="w-28 text-left">中文含义</th>
          <th class="w-36 text-left">常见误读</th>
          <th class="text-left">说明 / 例句</th>
          <th class="w-20 text-left">发音</th>
        </tr>
      </thead>
      <tbody data-pronunciation-body>
        {
          terms.map(term => (
            <tr
              data-pronunciation-row
              data-category={term.category}
              data-search={`${term.term} ${term.ipa} ${term.zhPronunciation} ${term.meaningZh} ${term.commonMistake}`.toLowerCase()}
            >
              <td>
                <button
                  type="button"
                  class="text-accent underline decoration-dashed underline-offset-4"
                  data-speak={term.speakText ?? term.term}
                  aria-label={`朗读 ${term.term}`}
                >
                  {term.term}
                </button>
              </td>
              <td>
                <button
                  type="button"
                  class="font-mono text-foreground underline decoration-dotted underline-offset-4"
                  data-speak={term.speakText ?? term.term}
                  aria-label={`朗读 ${term.term}，音标 ${term.ipa}`}
                >
                  {term.ipa}
                </button>
              </td>
              <td class="font-semibold">{term.zhPronunciation}</td>
              <td>{term.meaningZh}</td>
              <td>{term.commonMistake}</td>
              <td>
                <div>{term.note}</div>
                {term.example && <div class="text-muted-foreground mt-1">{term.example}</div>}
              </td>
              <td>
                <button
                  type="button"
                  class="border-border hover:bg-muted rounded border px-2 py-1"
                  data-speak={term.speakText ?? term.term}
                  aria-label={`朗读 ${term.term}`}
                >
                  Listen
                </button>
              </td>
            </tr>
          ))
        }
      </tbody>
    </table>
  </div>
</section>
```

- [ ] **Step 2: Add client-side search and speech script**

Append this script at the end of the component:

```astro
<script>
  const searchInput = document.querySelector<HTMLInputElement>(
    "[data-pronunciation-search]"
  );
  const categorySelect = document.querySelector<HTMLSelectElement>(
    "[data-pronunciation-category]"
  );
  const fallback = document.querySelector<HTMLElement>(
    "[data-pronunciation-fallback]"
  );
  const rows = Array.from(
    document.querySelectorAll<HTMLTableRowElement>("[data-pronunciation-row]")
  );
  const speakButtons = Array.from(
    document.querySelectorAll<HTMLButtonElement>("[data-speak]")
  );

  const canSpeak = "speechSynthesis" in window && "SpeechSynthesisUtterance" in window;

  if (!canSpeak) {
    fallback?.classList.remove("hidden");
    speakButtons.forEach(button => {
      button.disabled = true;
      button.classList.add("cursor-not-allowed", "opacity-60");
    });
  }

  function applyFilters() {
    const query = searchInput?.value.trim().toLowerCase() ?? "";
    const category = categorySelect?.value ?? "";

    rows.forEach(row => {
      const matchesQuery = row.dataset.search?.includes(query) ?? true;
      const matchesCategory = !category || row.dataset.category === category;
      row.hidden = !(matchesQuery && matchesCategory);
    });
  }

  function speak(text: string) {
    if (!canSpeak) return;

    window.speechSynthesis.cancel();
    const utterance = new SpeechSynthesisUtterance(text);
    utterance.lang = "en-US";
    utterance.rate = 0.85;
    window.speechSynthesis.speak(utterance);
  }

  searchInput?.addEventListener("input", applyFilters);
  categorySelect?.addEventListener("change", applyFilters);

  speakButtons.forEach(button => {
    button.addEventListener("click", () => {
      const text = button.dataset.speak;
      if (text) speak(text);
    });
  });
</script>
```

- [ ] **Step 3: Run type check**

Run:

```bash
npx astro check
```

Expected: no Astro or TypeScript errors.

- [ ] **Step 4: Commit**

```bash
git add src/content/posts/_components/PronunciationTable.astro
git commit -m "feat: add interactive pronunciation table"
```

---

### Task 3: MDX Blog Article

**Files:**
- Create: `src/content/posts/developer-pronunciation-java-python-vue.mdx`

**Interfaces:**
- Consumes: `PronunciationTable`
- Consumes: `pronunciationTerms`

- [ ] **Step 1: Create the MDX post**

Create `src/content/posts/developer-pronunciation-java-python-vue.mdx`.

```mdx
---
title: "Java / Python / Vue 开发者常见英文发音词表"
description: "给中文母语开发者准备的技术英文发音速查表：包含音标、中文对照发音、常见误读，并支持点击单词或音标朗读。"
pubDatetime: 2026-07-10T00:00:00.000Z
featured: true
tags:
  - 英语
  - Java
  - Python
  - Vue
---

import PronunciationTable from "./_components/PronunciationTable.astro";
import { pronunciationTerms } from "./_data/pronunciationTerms";

写代码时，英文单词不一定要读得像播音员，但在会议、面试、技术分享里，至少要让别人听懂你在说哪个概念。

这篇不是系统英语课，而是一张面向开发者的发音速查表。你可以先看中文对照发音建立感觉，再用音标和点击朗读校准。

## 怎么用这张表

- 点英文单词、音标或 `Listen` 按钮，都可以朗读同一个英文词。
- 中文对照发音只是拐杖，不是标准答案。
- 遇到中文容易误导的音，我会混用 `kee`、`th`、`uh` 这类提示。
- 比如 `schema` 是 `/ˈskiː.mə/`，更适合记成 **斯 kee-muh**，不是“斯基马”。

## 常见坑

### 不要按拼音读缩写

`JVM`、`SQL`、`API` 这类词经常按英文字母名读。`Vue` 不是 `V-U-E`，它和 `view` 同音。

### 中文近似有时会害人

如果把 `schema` 写成“斯基-ma”，中文读者很容易读成 `si ji ma`。所以这张表会优先避免误导，必要时用中英混写。

### 长短元音要分清

`schema` 里的 `kee` 是长 `/iː/`。`ship` 和 `sheep`、`bit` 和 `beat` 这类差异，在技术词里也会出现。

## 发音词表

<PronunciationTable terms={pronunciationTerms} />

## 练习句

- The async task is queued.
- The schema migration failed.
- I will update the cache layer.
- We use Vue on the frontend and Python on the backend.
- The service runs on the JVM.
```

- [ ] **Step 2: Run Astro check**

Run:

```bash
npx astro check
```

Expected: the MDX import and content collection schema pass.

- [ ] **Step 3: Commit**

```bash
git add src/content/posts/developer-pronunciation-java-python-vue.mdx
git commit -m "feat: add developer pronunciation blog post"
```

---

### Task 4: Verification And Polish

**Files:**
- Modify if needed: `src/content/posts/_components/PronunciationTable.astro`
- Modify if needed: `src/content/posts/_data/pronunciationTerms.ts`
- Modify if needed: `src/content/posts/developer-pronunciation-java-python-vue.mdx`

**Interfaces:**
- Consumes all earlier deliverables.
- Produces a verified blog page.

- [ ] **Step 1: Run full build**

Run:

```bash
npm run build
```

Expected: `astro check`, `astro build`, `pagefind`, and asset copy complete successfully.

- [ ] **Step 2: Start local dev server**

Run:

```bash
npm run dev -- --host 127.0.0.1
```

Expected: Astro prints a local URL such as `http://127.0.0.1:4321/`.

- [ ] **Step 3: Manually verify the page**

Open the generated post URL. Verify:

```text
The page renders without MDX errors.
The table is readable on desktop width.
The table scrolls horizontally on mobile width.
Searching "schema" leaves the schema row visible.
Searching "斯 kee" leaves the schema row visible.
Filtering "Vue" shows Vue/frontend rows.
Clicking the term "schema" speaks schema.
Clicking the IPA "/ˈskiː.mə/" speaks schema.
Clicking Listen on the schema row speaks schema.
```

- [ ] **Step 4: Verify speech fallback in DevTools**

Temporarily run this in the browser console before reloading the page:

```js
delete window.speechSynthesis;
```

Expected: if the browser allows the override for the page session, controls become disabled and the fallback message appears. If the browser prevents deleting the property, record that limitation and verify the code path by temporarily changing `const canSpeak = false` locally, then revert the local change before committing.

- [ ] **Step 5: Final commit**

If verification required polish changes, commit them:

```bash
git add src/content/posts/_components/PronunciationTable.astro src/content/posts/_data/pronunciationTerms.ts src/content/posts/developer-pronunciation-java-python-vue.mdx
git commit -m "fix: polish pronunciation blog interactions"
```

If no polish changes were needed, do not create an empty commit.

---

## Appendix A: Required Initial Term Inventory

Use this inventory for the first implementation. The IPA and Chinese approximation are intentionally practical for developer learning; during implementation, verify any row that feels uncertain before committing.

| Term | Category | IPA | Chinese approximation | Meaning | Common mistake |
|---|---|---|---|---|---|
| Java | Java | /ˈdʒɑː.və/ | 扎-vuh | Java 语言 | 加瓦 |
| JVM | Java | /ˌdʒeɪ viː ˈem/ | J-V-M | Java 虚拟机 | 按中文字母读 |
| JDK | Java | /ˌdʒeɪ diː ˈkeɪ/ | J-D-K | Java 开发工具包 | 按中文字母读 |
| Spring | Java | /sprɪŋ/ | spring | Spring 框架 | 死不 ring |
| Maven | Java | /ˈmeɪ.vən/ | MAY-vuhn | 构建工具 | 马文 |
| Gradle | Java | /ˈɡreɪ.dəl/ | GRAY-duhl | 构建工具 | 格拉德 |
| daemon | Java | /ˈdiː.mən/ | DEE-muhn | 守护进程 | 迪蒙 |
| enum | Java | /ˈiː.nəm/ | EE-nuhm | 枚举 | e-num |
| generic | Java | /dʒəˈner.ɪk/ | juh-NEH-rik | 泛型 | gene-rik |
| Hibernate | Java | /ˈhaɪ.bɚ.neɪt/ | HIGH-bur-nate | Hibernate 框架 | hi-ber-nate |
| deprecated | Java | /ˈdep.rə.keɪ.tɪd/ | DEP-ruh-kay-tid | 已废弃 | de-pre-ci-ated |
| synchronized | Java | /ˈsɪŋ.krə.naɪzd/ | SING-kruh-nized | 同步的 | syn-chron-ized |
| annotation | Java | /ˌæn.əˈteɪ.ʃən/ | an-uh-TAY-shuhn | 注解 | anno-ta-tion |
| servlet | Java | /ˈsɝːv.lət/ | SURV-luht | Servlet | serve-let |
| repository | Java | /rɪˈpɑː.zə.tɔːr.i/ | ri-PAH-zuh-tory | 仓库层 | repo-si-tory |
| dependency | Java | /dɪˈpen.dən.si/ | di-PEN-duhn-see | 依赖 | de-pen-den-cy |
| injection | Java | /ɪnˈdʒek.ʃən/ | in-JEK-shuhn | 注入 | in-jek-tion |
| transaction | Java | /trænˈzæk.ʃən/ | tran-ZAK-shuhn | 事务 | trans-action |
| Python | Python | /ˈpaɪ.θɑːn/ | PIE-thon | Python 语言 | 派松 |
| Django | Python | /ˈdʒæŋ.ɡoʊ/ | JANG-go | Django 框架 | di-jan-go |
| FastAPI | Python | /fæst ˌeɪ piː ˈaɪ/ | fast A-P-I | FastAPI 框架 | fast-api 连读错 |
| async | Python | /eɪˈsɪŋk/ | ay-SINK | 异步 | 啊-sync |
| await | Python | /əˈweɪt/ | uh-WAIT | 等待异步结果 | a-wait 重读错 |
| tuple | Python | /ˈtuː.pəl/ | TOO-puhl | 元组 | 土普 |
| schema | Python | /ˈskiː.mə/ | 斯 kee-muh | 结构定义 | 斯基马、斯给马 |
| decorator | Python | /ˈdek.ə.reɪ.t̬ɚ/ | DEK-uh-ray-ter | 装饰器 | de-co-ra-tor |
| generator | Python | /ˈdʒen.ə.reɪ.t̬ɚ/ | JEN-uh-ray-ter | 生成器 | gene-ra-tor |
| virtualenv | Python | /ˈvɝː.tʃu.əl env/ | VUR-choo-uhl env | 虚拟环境 | virtual-env 每段都重读 |
| pip | Python | /pɪp/ | pip | Python 包工具 | P-I-P |
| pytest | Python | /ˈpaɪ.test/ | PIE-test | 测试框架 | pee-test |
| coroutine | Python | /ˌkoʊ.ruːˈtiːn/ | koh-roo-TEEN | 协程 | co-ro-tine |
| iterable | Python | /ˈɪt̬.ɚ.ə.bəl/ | IT-er-uh-buhl | 可迭代对象 | i-ter-able |
| dictionary | Python | /ˈdɪk.ʃə.ner.i/ | DIK-shuh-ner-ee | 字典 | dic-tion-ary |
| attribute | Python | /ˈæt.rɪ.bjuːt/ | AT-ri-byoot | 属性 | a-tri-bute |
| parameter | Python | /pəˈræm.ə.t̬ɚ/ | puh-RAM-uh-ter | 参数 | pa-ra-meter |
| argument | Python | /ˈɑːrɡ.jə.mənt/ | ARG-yuh-muhnt | 实参 | ar-gu-ment |
| Vue | Vue | /vjuː/ | view / viu | Vue 框架 | V-U-E |
| Vite | Frontend | /viːt/ | veet | 构建工具 | vite 像 bite |
| Nuxt | Vue | /nʌkst/ | nukst | Nuxt 框架 | next |
| Pinia | Vue | /ˈpiː.njə/ | PEE-nyuh | Pinia 状态库 | pi-ni-a |
| props | Vue | /prɑːps/ | props | 组件属性 | pro-ps 分开读 |
| emit | Vue | /iˈmɪt/ | ee-MIT | 发出事件 | e-mit 重读错 |
| component | Frontend | /kəmˈpoʊ.nənt/ | kuhm-POH-nuhnt | 组件 | com-po-nent 每段重读 |
| template | Frontend | /ˈtem.plət/ | TEM-pluht | 模板 | tem-plate |
| reactive | Vue | /riˈæk.tɪv/ | ree-AK-tiv | 响应式的 | re-active 分开读 |
| router | Vue | /ˈruː.t̬ɚ/ | ROO-ter | 路由器/路由库 | rau-ter |
| TypeScript | Frontend | /ˈtaɪp skrɪpt/ | TYPE-script | TypeScript | type-s-cript |
| JavaScript | Frontend | /ˈdʒɑː.və skrɪpt/ | JA-vuh-script | JavaScript | 加瓦-script |
| framework | Frontend | /ˈfreɪm.wɝːk/ | FRAME-work | 框架 | fra-me-work |
| lifecycle | Frontend | /ˈlaɪfˌsaɪ.kəl/ | LIFE-cycle | 生命周期 | li-fe-cycle |
| hydration | Frontend | /haɪˈdreɪ.ʃən/ | high-DRAY-shuhn | 水合 | hy-dra-tion |
| composable | Vue | /kəmˈpoʊ.zə.bəl/ | kuhm-POH-zuh-buhl | 可组合函数 | compose-able |
| ref | Vue | /ref/ | ref | Vue ref | R-E-F |
| slot | Vue | /slɑːt/ | slot | 插槽 | s-lot |
| cache | General | /kæʃ/ | 凯什 / 开什 | 缓存 | 卡切、卡其 |
| queue | General | /kjuː/ | Q / kiu | 队列 | 奎尤、库额 |
| suite | General | /swiːt/ | sweet | 套件 | su-it |
| issue | General | /ˈɪʃ.uː/ | ISH-oo | 问题/工单 | i-sue |
| repo | General | /ˈriː.poʊ/ | REE-poh | 代码仓库 | re-po |
| deploy | General | /dɪˈplɔɪ/ | di-PLOY | 部署 | de-ploy |
| merge | General | /mɝːdʒ/ | murj | 合并 | me-ge |
| branch | General | /bræntʃ/ | branch | 分支 | b-ranch |
| commit | General | /kəˈmɪt/ | kuh-MIT | 提交 | co-mmit |
| release | General | /rɪˈliːs/ | ri-LEASE | 发布 | re-lease |
| archive | General | /ˈɑːr.kaɪv/ | AR-kive | 归档 | ar-chive |
| module | General | /ˈmɑː.dʒuːl/ | MAH-jool | 模块 | mo-du-le |
| package | General | /ˈpæk.ɪdʒ/ | PAK-ij | 包 | pa-cka-ge |
| API | General | /ˌeɪ piː ˈaɪ/ | A-P-I | 接口 | api 当单词读 |
| CLI | General | /ˌsiː el ˈaɪ/ | C-L-I | 命令行界面 | cli 当单词读 |
| GUI | General | /ˌdʒiː juː ˈaɪ/ | G-U-I | 图形界面 | gui 当单词读 |
| bug | General | /bʌɡ/ | bug | 缺陷 | b-u-g |
| feature | General | /ˈfiː.tʃɚ/ | FEE-chur | 功能 | fea-ture |
| legacy | General | /ˈleɡ.ə.si/ | LEG-uh-see | 遗留系统 | le-ga-cy |
| refactor | General | /ˌriːˈfæk.tɚ/ | ree-FAK-ter | 重构 | re-factor |
| SQL | Database | /ˈsiː.kwəl/ | SEE-kwuhl | SQL | 只读 S-Q-L 或 sequel 混乱 |
| Redis | Database | /ˈred.ɪs/ | RED-is | Redis | ree-dis |
| Postgres | Database | /ˈpoʊst.ɡres/ | POST-gres | PostgreSQL 简称 | post-grees |
| MySQL | Database | /ˌmaɪ ˌes kjuː ˈel/ | My-S-Q-L | MySQL | my-sequel 固定化 |
| nginx | DevOps | /ˌen.dʒɪn ˈeks/ | engine-X | nginx 服务器 | en-ginks |
| Docker | DevOps | /ˈdɑː.kɚ/ | DAH-ker | 容器平台 | dock-er 重音错 |
| Kubernetes | DevOps | /ˌkuː.bɚˈnet̬.iːz/ | koo-ber-NET-eez | 容器编排 | koo-ber-ne-tees |
| cron | DevOps | /krɑːn/ | cron | 定时任务 | c-ron |
| proxy | DevOps | /ˈprɑːk.si/ | PROK-see | 代理 | pro-xy |
| cluster | DevOps | /ˈklʌs.tɚ/ | KLUS-ter | 集群 | clu-ster |
| ingress | DevOps | /ˈɪn.ɡres/ | IN-gress | 入口流量 | in-gress 重音错 |
| latency | DevOps | /ˈleɪ.tən.si/ | LAY-tuhn-see | 延迟 | la-ten-cy |
| throughput | DevOps | /ˈθruː.pʊt/ | THROO-put | 吞吐量 | through-put 分错 |
| observability | DevOps | /əbˌzɝː.vəˈbɪl.ə.t̬i/ | ub-zur-vuh-BIL-uh-tee | 可观测性 | observe-ability |
