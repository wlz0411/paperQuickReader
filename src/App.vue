<template>
  <div id="app">
    <!-- 顶栏 -->
    <header class="top">
      <div class="top-inner">
        <!-- 顶部 Logo -->
        <div class="brand" style="display: flex; align-items: center; gap: 8px;">
          <img 
            src="/OWL.png" 
            alt="Logo" 
            style="width: 32px; height: 32px; object-fit: contain; mix-blend-mode: multiply;" 
          />
          <span>论文速读<span class="dot">.</span></span>
        </div>
        <div class="tagline">检索 · 总结 · 结构化笔记</div>
        <div class="top-actions">
          <div class="seg" role="group" aria-label="总结模板">
            <button type="button" data-tpl="review" aria-pressed="true" @click="handleTplChange('review')">文献综述向</button>
            <button type="button" data-tpl="exam" aria-pressed="false" @click="handleTplChange('exam')">考试复习向</button>
          </div>
          <button class="btn btn-ghost" id="copyBtn" type="button" @click="copyMarkdown">复制笔记</button>
        </div>
      </div>
    </header>

    <!-- 主体大布局 -->
    <div class="app-layout">
      <!-- 左侧历史记录栏 -->
      <aside class="history-sidebar">
        <div class="history-header">
          <span>历史记录</span>
          <button class="clear-btn" @click="clearHistory">清空</button>
        </div>
        <ul class="history-list">
          <li v-if="historyList.length === 0" class="empty-tip">暂无记录</li>
          
          <li 
            v-for="(item, index) in historyList" 
            :key="index" 
            @click="loadHistory(item)"
            class="history-item"
            :class="{ 'is-pinned': item.isPinned }"
          >
            <div class="h-content">
              <div class="h-title">{{ item.title }}</div>
              <div class="h-meta">{{ item.time }} · {{ TPL[item.template]?.label || '笔记' }}</div>
            </div>
            
            <div class="action-btns">
              <button class="icon-btn pin-btn" @click.stop="togglePin(index)" :title="item.isPinned ? '取消置顶' : '置顶该记录'">
                <svg viewBox="0 0 24 24" width="14" height="14" stroke="currentColor" stroke-width="2" fill="none" stroke-linecap="round" stroke-linejoin="round">
                  <path d="M21 10c0 7-9 13-9 13s-9-6-9-13a9 9 0 0 1 18 0z"></path>
                  <circle cx="12" cy="10" r="3"></circle>
                </svg>
              </button>
              
              <button class="icon-btn delete-btn" @click.stop="deleteHistoryItem(index)" title="删除该记录">
                <svg viewBox="0 0 24 24" width="14" height="14" stroke="currentColor" stroke-width="2" fill="none" stroke-linecap="round" stroke-linejoin="round">
                  <line x1="18" y1="6" x2="6" y2="18"></line>
                  <line x1="6" y1="6" x2="18" y2="18"></line>
                </svg>
              </button>
            </div>
          </li>
        </ul>
      </aside>

      <!-- 右侧主内容区 -->
      <main class="main-content">
        <div class="wrap">
          <!-- ================= 输入 ================= -->
          <section class="card" id="inputCard">
            <div class="input-main">
              <h1 class="card-title">输入研究主题或论文文本</h1>
              <p class="hint">支持关键词主题，也支持直接粘贴摘要或章节段落；单次上限 8000 字。</p>
              <div class="field">
                <textarea id="input" spellcheck="false" placeholder="例：大语言模型的幻觉问题&#10;&#10;或直接粘贴一段论文摘要 / 引言……"></textarea>
                <span class="counter" id="counter">0 / 8000</span>
              </div>
              <div class="row">
                <span class="hint" id="tplNote">按「研究现状 → 方法对比 → 开放问题」组织要点</span>
                <button class="btn btn-primary" id="submit" type="button">生成速读笔记</button>
              </div>
              
              <div class="debugbar">
                <span class="lb">原型调试</span>
                <button class="chip" data-sc="success" aria-pressed="true">成功</button>
                <button class="chip" data-sc="NO_RESULT" aria-pressed="false">NO_RESULT</button>
                <button class="chip" data-sc="LLM_TIMEOUT" aria-pressed="false">LLM_TIMEOUT</button>
                <button class="chip" data-sc="BAD_INPUT" aria-pressed="false">BAD_INPUT</button>
                <button class="chip" data-sc="UPSTREAM_ERROR" aria-pressed="false">UPSTREAM_ERROR</button>
                <button class="chip" data-sc="delay" aria-pressed="false">慢响应 8s</button>
                <button class="chip" id="groupChip" aria-pressed="false">分组视图</button>
              </div>
            </div>
            <div class="input-collapsed">
              <span class="ic-label">当前查看</span>
              <span class="ic-text" id="icText"></span>
              <button class="btn btn-ghost" id="editBtn" type="button">修改输入</button>
            </div>
          </section>

          <!-- ================= 状态区（光晕涟漪版） ================= -->
          <section class="loading" id="loading" aria-live="polite">
            <div class="load-card">
              <div class="load-head">
                
                <!-- 🌟 光晕涟漪加载动画 -->
                <div class="owl-reading-loader">
                  <img src="/OWL.png" alt="owl" class="owl-img" />
                  <div class="ripple"></div>
                  <div class="ripple ripple-delay"></div>
                </div>

                <div class="load-text-box">
                  <span class="load-text" id="loadMsg">正在检索相关文献...</span>
                  <span class="load-sub">已用 <span class="load-sec" id="sec">0</span> 秒，预计需 10-20 秒</span>
                </div>
              </div>
              <div class="bar"><span id="barFill"></span></div>
            </div>
          </section>

          <section class="error" id="error" aria-live="assertive">
            <div class="error-card">
              <span class="error-icon">!</span>
              <div>
                <div class="error-code" id="errCode">NO_RESULT</div>
                <p class="error-msg" id="errMsg"></p>
              </div>
            </div>
          </section>

          <!-- ================= 笔记结果区 ================= -->
          <section class="result" id="result" aria-live="polite">
            
            <!-- 多标签页导航栏 -->
            <div class="report-tabs" v-if="openReports.length > 0">
              <div 
                v-for="(tab, idx) in openReports" 
                :key="tab.title"
                class="report-tab"
                :class="{ active: activeReportId === tab.title }"
                @click="switchTab(tab)"
              >
                <span class="tab-title">{{ tab.title }}</span>
                <button class="tab-close" @click.stop="closeTab(idx)">×</button>
              </div>
            </div>

            <div class="note-grid">
              <nav class="toc" aria-label="笔记目录">
                <p class="toc-h">目 录</p>
                <ul class="toc-list" id="tocList"></ul>
              </nav>

              <article class="sheet" id="sheet">
                <header class="note-head">
                  <div class="note-eyebrow">速读笔记 · <span id="noteTpl">文献综述向</span></div>
                  <h1 class="note-title" id="noteTitle"></h1>
                  <div class="note-meta" id="noteMeta"></div>
                </header>

                <section class="ch" id="ch-1" data-toc="一">
                  <h2 class="ch-h"><span class="ch-no">一</span>总览</h2>
                  <p class="summary" id="summary"></p>
                </section>

                <section class="ch" id="ch-2" data-toc="二">
                  <h2 class="ch-h"><span class="ch-no">二</span>核心要点<span class="ch-cnt" id="ptsCnt"></span></h2>
                  <div class="skeleton" id="skeleton"></div>
                  <ol class="points" id="points"></ol>
                </section>

                <section class="ch" id="ch-3" data-toc="三">
                  <h2 class="ch-h"><span class="ch-no">三</span>关键术语<span class="ch-cnt" id="termsCnt"></span></h2>
                  <dl class="terms" id="terms"></dl>
                </section>

                <section class="ch" id="ch-4" data-toc="四">
                  <h2 class="ch-h"><span class="ch-no">四</span>延伸问题<span class="ch-cnt" id="qsCnt"></span></h2>
                  <ul class="questions" id="questions"></ul>
                </section>

                <section class="ch" id="ch-5" data-toc="五">
                  <h2 class="ch-h"><span class="ch-no">五</span>参考来源<span class="ch-cnt" id="srcCnt"></span></h2>
                  <ol class="sources" id="sources"></ol>
                </section>

                <footer class="sheet-foot">
                  <span>由「论文速读工具」生成 · 契约 v1.0</span>
                  <span>每条要点均可溯源至第五章来源</span>
                </footer>
              </article>
            </div>
          </section>
        </div>
      </main>
    </div>

    <!-- 悬浮层 -->
    <div class="pop" id="pop"></div>
    <div class="toast" id="toast"></div>
  </div>
</template>

<script setup>
import { onMounted, ref } from 'vue';

// ================= 1. 全局状态定义 =================
const historyList = ref([]);
const openReports = ref([]);    
const activeReportId = ref(''); 

const TPL = {
  review: { label: '文献综述向', note: '按「研究现状 → 方法对比 → 开放问题」组织要点', skeleton: ['研究现状', '方法对比', '开放问题'] },
  exam: { label: '考试复习向', note: '按「概念定义 → 关键机制 → 易错点」组织要点', skeleton: ['概念定义', '关键机制', '易错点'] }
};

let template = 'review';
let scenario = 'success';
let grouped = false;
let note = null;
let isBusy = false, timerId = null, msgTimer = null, startedAt = 0;
const MAX_LEN = 8000, TIMEOUT_S = 45;
const el = {}; 

// ================= 2. Mock 数据 =================
const MOCK = {
  success: {
    "ok": true,
    "summary": "大语言模型的幻觉指模型生成与事实不符却表述自信的内容，主流缓解路径是以 RAG 为代表的外部检索增强与事实一致性校准。",
    "points": [
      { "text": "幻觉指 LLM 生成与事实不符或无出处的内容，且表述高度自信，用户难以辨别", "source_ids": [1, 3] },
      { "text": "幻觉可分为事实性幻觉（与客观事实矛盾）与忠实性幻觉（与用户指令或上下文矛盾）", "source_ids": [2] },
      { "text": "RAG 通过在生成前引入外部检索证据，可显著降低事实性幻觉发生率", "source_ids": [1, 4] },
      { "text": "训练数据中的噪声、时效滞后与长尾知识缺失是幻觉的主要成因之一", "source_ids": [3] },
      { "text": "RLHF 对齐过程中追求人类偏好的流畅性，可能放大'自信地错'的倾向", "source_ids": [5] },
      { "text": "检索质量本身是 RAG 的瓶颈：检索不到或检索噪声大时，幻觉率不降反升", "source_ids": [4] }
    ],
    "terms": [
      { "term": "幻觉（Hallucination）", "explain": "模型输出与事实或输入不符的内容，常表现为自信的错误陈述" },
      { "term": "RAG（检索增强生成）", "explain": "生成前先从外部知识库检索相关文档，把证据一并交给模型作答的架构" },
      { "term": "事实一致性校准", "explain": "通过后处理或训练手段让模型输出与可验证事实对齐的方法族" }
    ],
    "questions": [
      "如何在不显著增加推理成本的前提下量化幻觉率？",
      "检索增强与模型内部知识冲突时，应优先信任哪一方？",
      "中文场景下幻觉评测基准有哪些，覆盖度是否足够？"
    ],
    "sources": [
      { "id": 1, "title": "Survey of Hallucination in Natural Language Generation", "url": "https://arxiv.org/abs/2202.03629" },
      { "id": 2, "title": "A Survey on Hallucination in Large Language Models", "url": "https://arxiv.org/abs/2311.05232" },
      { "id": 3, "title": "Siren's Song in the AI Ocean: A Survey on Hallucination in LLMs", "url": "https://arxiv.org/abs/2309.01219" },
      { "id": 4, "title": "Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks", "url": "https://arxiv.org/abs/2005.11401" },
      { "id": 5, "title": "Training Language Models to Follow Instructions (InstructGPT)", "url": "https://arxiv.org/abs/2203.02155" }
    ]
  },
  NO_RESULT:      { "ok": false, "error_code": "NO_RESULT",      "message": "未检索到与该主题相关的资料，请换个说法或扩充关键词后重试" },
  LLM_TIMEOUT:    { "ok": false, "error_code": "LLM_TIMEOUT",    "message": "生成超时（超过 45 秒），请稍后重试或缩短输入内容" },
  BAD_INPUT:      { "ok": false, "error_code": "BAD_INPUT",      "message": "输入为空或超过 8000 字上限，请检查后重新提交" },
  UPSTREAM_ERROR: { "ok": false, "error_code": "UPSTREAM_ERROR", "message": "检索服务暂时不可用，请稍后重试" }
};

const DEMO_SECTIONS = ['研究现状', '研究现状', '方法对比', '方法对比', '开放问题', '开放问题'];

// ================= 3. 核心逻辑方法 =================

function switchTab(tab) {
  activeReportId.value = tab.title;
  el.input.value = tab.title;
  renderCounter();
  template = tab.template || 'review';
  handleTplChange(template, true); // 静默切换
  renderNote(tab.data, tab.seconds || 0, tab.title);
}

function closeTab(index) {
  const tab = openReports.value[index];
  openReports.value.splice(index, 1);
  
  if (openReports.value.length === 0) {
    el.result.classList.remove('on');
    activeReportId.value = '';
    el.inputCard.classList.remove('collapsed');
  } else if (activeReportId.value === tab.title) {
    const nextTab = openReports.value[openReports.value.length - 1];
    switchTab(nextTab);
  }
}

function sortHistory(list) {
  return [...list].sort((a, b) => {
    if (a.isPinned && !b.isPinned) return -1;
    if (!a.isPinned && b.isPinned) return 1;
    return 0;
  });
}

function togglePin(index) {
  const item = historyList.value[index];
  item.isPinned = !item.isPinned;
  historyList.value = sortHistory(historyList.value);
  localStorage.setItem('quickReadHistory', JSON.stringify(historyList.value));
  toast(item.isPinned ? '已置顶记录' : '已取消置顶');
}

function handleTplChange(tplName, silent = false) {
  template = tplName;
  document.querySelectorAll('.seg button').forEach(x => x.setAttribute('aria-pressed', String(x.dataset.tpl === template)));
  if (el.tplNote) el.tplNote.textContent = TPL[template].note;
  if (note) {
    if (el.noteTpl) el.noteTpl.textContent = TPL[template].label;
    renderSkeleton();
    if (!silent) {
      toast('已切换为「' + TPL[template].label + '」');
    }
  }
}

function loadHistory(item) {
  if (!openReports.value.find(r => r.title === item.title)) {
    openReports.value.push(item); 
  }
  switchTab(item); 
}

function deleteHistoryItem(index) {
  const item = historyList.value[index];
  historyList.value.splice(index, 1);
  localStorage.setItem('quickReadHistory', JSON.stringify(historyList.value));
  
  const openIdx = openReports.value.findIndex(r => r.title === item.title);
  if (openIdx > -1) closeTab(openIdx);
  
  toast('已删除该条记录');
}

function clearHistory() {
  if (confirm('确定要清空所有历史记录吗？')) {
    historyList.value = [];
    localStorage.removeItem('quickReadHistory');
    openReports.value = [];
    el.result.classList.remove('on');
    el.inputCard.classList.remove('collapsed');
    toast('已清空历史记录');
  }
}

function copyMarkdown() {
  if (!note) { toast('请先生成笔记'); return; }
  const d = note.data;
  const md = [
    '# ' + note.title, '', '> 速读笔记 · ' + TPL[template].label + ' · 契约 v1.0', '',
    '## 一、总览', '', d.summary, '', '## 二、核心要点', '',
    ...(d.points || []).map((p, i) => '**2.' + (i + 1) + '** ' + p.text + ' （' + p.source_ids.map(x => '[S' + x + ']').join('') + '）'), '',
    '## 三、关键术语', '', ...(d.terms || []).map((t, i) => '**3.' + (i + 1) + ' ' + t.term + '**：' + t.explain), '',
    '## 四、延伸问题', '', ...(d.questions || []).map((q, i) => '**4.' + (i + 1) + '** ' + q), '',
    '## 五、参考来源', '', ...(d.sources || []).map(s => '- **[S' + s.id + ']** ' + s.title + ' — ' + s.url)
  ].join('\n');

  navigator.clipboard.writeText(md).then(() => toast('已复制为 Markdown 笔记')).catch(() => toast('复制失败，请检查浏览器权限'));
}

function toast(msg) {
  if (!el.toast) return;
  el.toast.textContent = msg; 
  el.toast.classList.add('on');
  clearTimeout(toast._t); 
  toast._t = setTimeout(() => el.toast.classList.remove('on'), 2000);
}

function renderCounter() {
  if (!el.input) return;
  const n = Array.from(el.input.value).length;
  el.counter.textContent = n + ' / ' + MAX_LEN;
  el.counter.className = 'counter' + (n > MAX_LEN ? ' over' : n > MAX_LEN * 0.9 ? ' warn' : '');
  el.submit.disabled = n > MAX_LEN || isBusy;
}

function setBusy(v) {
  isBusy = v;
  el.submit.disabled = v;
  el.submit.textContent = v ? '生成中…' : '生成速读笔记';
  el.loading.classList.toggle('on', v);
  if (v) { el.error.classList.remove('on'); el.result.classList.remove('on'); }
  renderCounter();
}

function startTimer() {
  startedAt = Date.now();
  el.sec.textContent = '0';
  el.barFill.style.width = '3%';

  const msgs = [
    '正在检索相关文献...',
    '正在阅读并理解核心段落...',
    '正在提取关键术语与要点...',
    '正在进行事实一致性校准...',
    '正在生成结构化速读笔记...'
  ];
  let msgIdx = 0;
  const msgEl = document.getElementById('loadMsg');
  if (msgEl) {
    msgEl.textContent = msgs[0];
    msgEl.style.opacity = 1;
  }

  msgTimer = setInterval(() => {
    msgIdx++;
    if (msgIdx < msgs.length && msgEl) {
      msgEl.style.opacity = 0;
      setTimeout(() => {
        msgEl.textContent = msgs[msgIdx];
        msgEl.style.opacity = 1;
      }, 200);
    }
  }, 3000);

  timerId = setInterval(() => {
    const s = Math.floor((Date.now() - startedAt) / 1000);
    el.sec.textContent = String(s);
    el.barFill.style.width = Math.min(100, (s / TIMEOUT_S) * 100) + '%';
    if (s >= TIMEOUT_S) { 
      stopTimer(); 
      finish(MOCK.LLM_TIMEOUT); 
    }
  }, 1000);
}

function stopTimer() { 
  if (timerId) { clearInterval(timerId); timerId = null; }
  if (msgTimer) { clearInterval(msgTimer); msgTimer = null; }
}

function showError(code, msg) {
  el.errCode.textContent = code;
  el.errMsg.textContent = msg;
  el.error.classList.add('on');
  el.result.classList.remove('on');
  el.inputCard.classList.remove('collapsed');
}

async function submit() {
  const raw = el.input.value.trim();
  if (!raw || Array.from(raw).length > MAX_LEN) {
    showError('BAD_INPUT', MOCK.BAD_INPUT.message);
    return;
  }
  setBusy(true);
  startTimer();
  try {
    const data = await quickRead(raw, template);
    stopTimer();
    finish(data, raw);
  } catch (e) {
    stopTimer();
    finish((e && e.name === 'AbortError') ? MOCK.LLM_TIMEOUT : MOCK.UPSTREAM_ERROR);
  }
}

function finish(data, raw) {
  stopTimer();
  setBusy(false);
  if (!data || data.ok !== true) {
    showError(data && data.error_code ? data.error_code : 'UPSTREAM_ERROR', (data && data.message) || MOCK.UPSTREAM_ERROR.message);
    return;
  }
  const seconds = Math.round((Date.now() - startedAt) / 1000);
  const title = raw || el.icText.textContent;
  
  const cleanTitle = title.trim();
  if (cleanTitle) {
    const existing = historyList.value.find(item => item.title === cleanTitle);
    const isPinned = existing ? existing.isPinned : false;

    let list = historyList.value.filter(item => item.title !== cleanTitle);
    const newReport = { 
      title: cleanTitle, data: data, template: template, seconds: seconds, 
      time: new Date().toLocaleString('zh-CN', { hour12: false }), isPinned: isPinned 
    };
    list.unshift(newReport);
    if (list.length > 15) list.pop();
    
    historyList.value = sortHistory(list);
    localStorage.setItem('quickReadHistory', JSON.stringify(historyList.value));

    if (!openReports.value.find(r => r.title === cleanTitle)) {
      openReports.value.push(newReport);
    }
    activeReportId.value = cleanTitle;
  }

  renderNote(data, seconds, title);
}

async function quickRead(input, tpl) {
  const USE_MOCK = import.meta.env.VITE_USE_MOCK === 'true';
  if (USE_MOCK) {
    await new Promise(r => setTimeout(r, scenario === 'delay' ? 8000 : 900));
    if (scenario === 'success' || scenario === 'delay') {
      const mockResult = JSON.parse(JSON.stringify(MOCK.success));
      mockResult.summary = `这是关于「${input}」的模拟数据总览：大语言模型的幻觉指模型生成与事实不符却表述自信的内容……`;
      return mockResult;
    }
    return MOCK[scenario];
  }

  const res = await fetch(import.meta.env.VITE_API_BASE + '/workflows/run', {
    method: 'POST', 
    headers: {
      'Content-Type': 'application/json',
      'Authorization': 'Bearer ' + import.meta.env.VITE_API_KEY
    },
    body: JSON.stringify({ 
      inputs: { input: input, template: tpl }, 
      response_mode: 'blocking',
      user: 'user-123' 
    }),
    signal: AbortSignal.timeout(45000)
  });
  
  const rawData = await res.json();
  
  if (rawData.code === 'unauthorized' || rawData.status === 401) {
    console.error("Dify 鉴权失败:", rawData);
    return { ok: false, error_code: "UPSTREAM_ERROR", message: "API 密钥无效，请检查 .env 配置" };
  }

  try {
    let llmOutput = rawData?.data?.outputs?.result 
                 || rawData?.data?.outputs?.text 
                 || rawData?.result 
                 || '';
    
    if (rawData.data && rawData.data.status === 'failed') {
       return { ok: false, error_code: "UPSTREAM_ERROR", message: "Dify 节点运行失败：" + (rawData.data.error || '未知错误') };
    }

    llmOutput = llmOutput.replace(/<think>[\s\S]*?<\/think>/g, '');
    llmOutput = llmOutput.replace(/```json/g, '').replace(/```/g, '').trim();

    const startIdx = llmOutput.indexOf('{');
    const endIdx = llmOutput.lastIndexOf('}');
    if (startIdx !== -1 && endIdx !== -1) {
      llmOutput = llmOutput.substring(startIdx, endIdx + 1);
    }

    return JSON.parse(llmOutput);
  } catch (e) {
    console.error("解析 Dify 返回的 JSON 失败：", e, rawData);
    return { ok: false, error_code: "UPSTREAM_ERROR", message: "解析后端数据失败" };
  }
}

function renderNote(d, seconds, title) {
  note = { data: d, seconds, title };
  el.noteTpl.textContent = TPL[template].label;
  el.noteTitle.textContent = title;
  if (el.icText) el.icText.textContent = title;

  el.noteMeta.innerHTML = '';
  
  const ptsLen = d.points ? d.points.length : 0;
  const trmLen = d.terms ? d.terms.length : 0;
  const qsLen = d.questions ? d.questions.length : 0;
  const srcLen = d.sources ? d.sources.length : 0;

  [
    ['生成时间', new Date().toLocaleString('zh-CN', { hour12: false })],
    ['耗时', seconds + 's'], ['要点', ptsLen + ' 条'], ['术语', trmLen + ' 个'], ['问题', qsLen + ' 个'], ['来源', srcLen + ' 条']
  ].forEach(([k, v]) => {
    const s = document.createElement('span');
    s.innerHTML = k + ' <b>' + esc(v) + '</b>';
    el.noteMeta.appendChild(s);
  });

  el.summary.textContent = d.summary || '';
  renderSkeleton();
  
  if (d.points && d.points.length > 0) { renderPoints(d); document.getElementById('ch-2').style.display = 'block'; } 
  else { document.getElementById('ch-2').style.display = 'none'; }

  if (d.terms && d.terms.length > 0) { renderTerms(d); document.getElementById('ch-3').style.display = 'block'; } 
  else { document.getElementById('ch-3').style.display = 'none'; }

  if (d.questions && d.questions.length > 0) { renderQuestions(d); document.getElementById('ch-4').style.display = 'block'; } 
  else { document.getElementById('ch-4').style.display = 'none'; }

  if (d.sources && d.sources.length > 0) { renderSources(d); document.getElementById('ch-5').style.display = 'block'; } 
  else { document.getElementById('ch-5').style.display = 'none'; }

  buildToc();
  el.result.classList.add('on');
  el.inputCard.classList.add('collapsed');
  el.result.scrollIntoView({ behavior: 'smooth', block: 'start' });
  requestAnimationFrame(spy);
}

// ================= 4. 生命周期初始化 =================
onMounted(() => {
  const $ = id => document.getElementById(id);
  Object.assign(el, {
    inputCard: $('inputCard'), input: $('input'), counter: $('counter'), submit: $('submit'),
    tplNote: $('tplNote'), icText: $('icText'), editBtn: $('editBtn'), copyBtn: $('copyBtn'),
    loading: $('loading'), sec: $('sec'), barFill: $('barFill'), error: $('error'), errCode: $('errCode'), errMsg: $('errMsg'),
    result: $('result'), noteTpl: $('noteTpl'), noteTitle: $('noteTitle'), noteMeta: $('noteMeta'), summary: $('summary'),
    skeleton: $('skeleton'), points: $('points'), terms: $('terms'), questions: $('questions'), sources: $('sources'),
    ptsCnt: $('ptsCnt'), termsCnt: $('termsCnt'), qsCnt: $('qsCnt'), srcCnt: $('srcCnt'), tocList: $('tocList'), pop: $('pop'), toast: $('toast')
  });

  const savedHistory = localStorage.getItem('quickReadHistory');
  if (savedHistory) {
    try {
      const parsed = JSON.parse(savedHistory);
      const validList = parsed.filter(item => typeof item === 'object' && item.data).map(item => ({
        ...item, isPinned: item.isPinned || false
      }));
      historyList.value = sortHistory(validList);
    } catch (e) { localStorage.removeItem('quickReadHistory'); }
  }

  el.submit.addEventListener('click', submit);
  el.input.addEventListener('input', renderCounter);
  el.input.addEventListener('keydown', e => { if ((e.ctrlKey || e.metaKey) && e.key === 'Enter') submit(); });
  el.editBtn.addEventListener('click', () => { el.inputCard.classList.remove('collapsed'); el.input.focus(); });
  
  document.querySelectorAll('.chip[data-sc]').forEach(c => {
    c.addEventListener('click', () => {
      scenario = c.dataset.sc;
      document.querySelectorAll('.chip[data-sc]').forEach(x => x.setAttribute('aria-pressed', String(x === c)));
    });
  });

  $('groupChip').addEventListener('click', function () {
    grouped = !grouped;
    this.setAttribute('aria-pressed', String(grouped));
    if (note) renderPoints(note.data);
  });

  el.input.value = '大语言模型的幻觉问题';
  renderCounter();

  setTimeout(() => { el.input.focus(); }, 800);
});

// ===== 渲染辅助函数 =====
function renderSkeleton() { el.skeleton.innerHTML = ''; const lb = document.createElement('span'); lb.className = 'lb'; lb.textContent = '模板骨架'; el.skeleton.appendChild(lb); TPL[template].skeleton.forEach((s, i) => { if (i) { const a = document.createElement('span'); a.className = 'sk-arrow'; a.textContent = '→'; el.skeleton.appendChild(a); } const b = document.createElement('span'); b.className = 'sk-step'; b.textContent = s; el.skeleton.appendChild(b); }); }
function renderPoints(d) { el.points.innerHTML = ''; el.ptsCnt.textContent = d.points.length + ' 条'; let lastSection = null; d.points.forEach((p, i) => { const section = grouped ? DEMO_SECTIONS[i] : null; if (grouped && section && section !== lastSection) { const g = document.createElement('li'); g.className = 'grp'; g.textContent = section; el.points.appendChild(g); lastSection = section; } const li = document.createElement('li'); li.className = 'pt'; const no = document.createElement('span'); no.className = 'pt-no'; no.textContent = '2.' + (i + 1); const body = document.createElement('div'); body.className = 'pt-body'; const pText = document.createElement('p'); pText.className = 'pt-text'; pText.textContent = p.text; const srcRow = document.createElement('div'); srcRow.className = 'pt-src'; const lb = document.createElement('span'); lb.className = 'lb'; lb.textContent = '出处'; srcRow.appendChild(lb); (p.source_ids || []).forEach(id => { const b = document.createElement('button'); b.type = 'button'; b.className = 'ref'; b.dataset.id = id; b.textContent = 'S' + id; b.addEventListener('click', ev => openPop(ev, id)); srcRow.appendChild(b); }); body.append(pText, srcRow); li.append(no, body); el.points.appendChild(li); }); }
function renderTerms(d) { el.terms.innerHTML = ''; el.termsCnt.textContent = d.terms.length + ' 个'; d.terms.forEach((t, i) => { const row = document.createElement('div'); row.className = 'tm'; const no = document.createElement('span'); no.className = 'tm-no'; no.textContent = '3.' + (i + 1); const body = document.createElement('div'); const term = document.createElement('p'); term.className = 'tm-term'; term.textContent = t.term; const ex = document.createElement('p'); ex.className = 'tm-explain'; ex.textContent = t.explain; body.append(term, ex); row.append(no, body); el.terms.appendChild(row); }); }
function renderQuestions(d) { el.questions.innerHTML = ''; el.qsCnt.textContent = d.questions.length + ' 个'; d.questions.forEach((q, i) => { const li = document.createElement('li'); li.className = 'qs'; const no = document.createElement('span'); no.className = 'qs-no'; no.textContent = '4.' + (i + 1); const p = document.createElement('p'); p.className = 'qs-text'; p.textContent = q; li.append(no, p); el.questions.appendChild(li); }); }
function renderSources(d) { el.sources.innerHTML = ''; el.srcCnt.textContent = d.sources.length + ' 条'; const backrefs = {}; d.points.forEach((p, i) => { (p.source_ids || []).forEach(id => { (backrefs[id] = backrefs[id] || []).push('2.' + (i + 1)); }); }); d.sources.forEach(s => { const li = document.createElement('li'); li.className = 'src'; li.dataset.src = s.id; const sn = document.createElement('span'); sn.className = 'sn'; sn.textContent = 'S' + s.id; const body = document.createElement('div'); body.className = 'src-body'; const a = document.createElement('a'); a.href = s.url; a.target = '_blank'; a.rel = 'noopener noreferrer'; a.textContent = s.title; const meta = document.createElement('div'); meta.className = 'src-meta'; const host = document.createElement('span'); host.className = 'host'; host.textContent = safeHost(s.url); meta.appendChild(host); const used = backrefs[s.id]; const br = document.createElement('span'); br.className = 'backref'; br.textContent = (used && used.length) ? ' · 被 ' + used.join('、') + ' 引用' : ' · 未被要点引用'; meta.appendChild(br); body.append(a, meta); li.append(sn, body); el.sources.appendChild(li); }); }
function buildToc() { const items = [['ch-1', '一', '总览', ''], ['ch-2', '二', '核心要点', note.data.points.length + ' 条'], ['ch-3', '三', '关键术语', note.data.terms.length + ' 个'], ['ch-4', '四', '延伸问题', note.data.questions.length + ' 个'], ['ch-5', '五', '参考来源', note.data.sources.length + ' 条']]; el.tocList.innerHTML = ''; items.forEach(([id, n, label, cnt]) => { const li = document.createElement('li'); const a = document.createElement('a'); a.href = '#' + id; a.dataset.target = id; a.innerHTML = '<span class="n">' + n + '</span><span>' + label + '</span>' + (cnt ? '<span class="c">' + cnt + '</span>' : ''); a.addEventListener('click', e => { e.preventDefault(); const top = document.getElementById(id).getBoundingClientRect().top + window.scrollY - 80; window.scrollTo({ top, behavior: 'smooth' }); }); li.appendChild(a); el.tocList.appendChild(li); }); }
function spy() { const links = [...el.tocList.querySelectorAll('a')]; const sections = links.map(a => document.getElementById(a.dataset.target)); let idx = 0; const y = window.scrollY + 120; sections.forEach((s, i) => { if (s && s.offsetTop <= y) idx = i; }); links.forEach((a, i) => a.classList.toggle('active', i === idx)); }
window.addEventListener('scroll', spy, { passive: true });
function openPop(ev, id) { ev.stopPropagation(); const s = note.data.sources.find(x => x.id === id); if (!s) return; clearCite(); document.querySelectorAll('.ref[data-id="' + id + '"]').forEach(r => r.classList.add('cite-on')); const row = document.querySelector('.src[data-src="' + id + '"]'); if (row) row.classList.add('cite-on'); el.pop.innerHTML = '<div class="pop-h">来源 S' + esc(id) + '</div><div class="pop-t">' + esc(s.title) + '</div><div class="pop-u">' + esc(s.url) + '</div><a class="pop-a" href="' + esc(s.url) + '" target="_blank" rel="noopener noreferrer">在新窗口打开 ↗</a>'; el.pop.classList.add('on'); const r = ev.currentTarget.getBoundingClientRect(); const pw = 290, margin = 8; let left = r.left + window.scrollX - 20; left = Math.min(left, window.scrollX + document.documentElement.clientWidth - pw - margin); left = Math.max(left, window.scrollX + margin); let top = r.bottom + window.scrollY + 8; if (r.bottom + 170 > window.innerHeight) top = r.top + window.scrollY - 168; el.pop.style.left = left + 'px'; el.pop.style.top = top + 'px'; }
function clearCite() { document.querySelectorAll('.cite-on').forEach(n => n.classList.remove('cite-on')); el.pop.classList.remove('on'); }
document.addEventListener('click', clearCite);
document.addEventListener('keydown', e => { if (e.key === 'Escape') clearCite(); });
function esc(v) { return String(v).replace(/[&<>"']/g, c => ({ '&': '&amp;', '<': '&lt;', '>': '&gt;', '"': '&quot;', "'": '&#39;' }[c])); }
function safeHost(url) { try { const u = new URL(url); return u.host + u.pathname; } catch { return url; } }
</script>

<style>
/* =========================================
   ✨ 视觉升级区：现代磨砂玻璃 + 高对比度渐变
   ========================================= */
:root{
  --ink:#1e293b; --ink-2:#334155; --ink-3:#64748b; --ink-4:#94a3b8;
  --line:#e2e8f0; --line-2:#f1f5f9;
  --bg:#f8fafc; --card:#fff;
  --brand:#2563eb; --brand-soft:#eff6ff; --brand-line:#bfdbfe;
  --accent:#d97706; --accent-soft:#fffbeb; --accent-line:#fde68a;
  --danger:#ef4444; --danger-soft:#fef2f2;
  --radius:16px;
  --shadow-sm: 0 1px 2px 0 rgba(0, 0, 0, 0.05);
  --shadow-md: 0 4px 6px -1px rgba(0, 0, 0, 0.05), 0 2px 4px -1px rgba(0, 0, 0, 0.03);
  --shadow-lg: 0 10px 25px -5px rgba(0, 0, 0, 0.05), 0 8px 10px -6px rgba(0, 0, 0, 0.01);
}
*{box-sizing:border-box}

html, body {
  margin: 0;
  padding: 0;
  background-color: #f0f4f8;
}

#app {
  min-height: 100vh;
  width: 100%;
  background-color: #f4f7f9;
  background-image: 
    radial-gradient(circle at 15% 20%, rgba(59, 130, 246, 0.15), transparent 30%),
    radial-gradient(circle at 85% 30%, rgba(139, 92, 246, 0.12), transparent 30%),
    radial-gradient(circle at 50% 80%, rgba(236, 72, 153, 0.08), transparent 30%),
    radial-gradient(#cbd5e1 1px, transparent 1px);
  background-size: 100% 100%, 100% 100%, 100% 100%, 24px 24px;
  display: flex;
  flex-direction: column;
  font-family: "Inter", "PingFang SC", "Microsoft YaHei", sans-serif;
  font-size: 15px; 
  line-height: 1.6; 
  color: var(--ink);
  -webkit-font-smoothing: antialiased;
  position: relative;
}
.card, .card-title, .hint, textarea#input, .sheet, .summary { text-align: left !important; }
.counter { background: transparent !important; box-shadow: none !important; border: none !important; padding: 0 !important; }

header.top{
  position: sticky; top: 0; z-index: 30;
  background: rgba(255, 255, 255, 0.75); backdrop-filter: saturate(180%) blur(12px);
  border-bottom: 1px solid rgba(255,255,255,0.3);
  box-shadow: 0 1px 2px rgba(0,0,0,0.02);
  width: 100%;
}
.top-inner{ max-width:1400px; margin:0 auto; padding:14px 32px; display:flex; align-items:center; gap:16px; flex-wrap:wrap; }
.brand{font-size:18px; font-weight:800; letter-spacing:-0.02em; color:#0f172a;}
.brand .dot{color:var(--brand)}
.tagline{font-size:13px; color:var(--ink-3); font-weight:500;}
.top-actions{margin-left:auto; display:flex; gap:12px; align-items:center;}
.seg{display:inline-flex; background:#f1f5f9; border:1px solid var(--line); border-radius:10px; padding:4px;}
.seg button{appearance:none; border:0; background:transparent; cursor:pointer; font:inherit; font-size:13px; font-weight:500; color:var(--ink-3); padding:6px 16px; border-radius:7px; transition:all .2s;}
.seg button[aria-pressed="true"]{background:#fff; color:var(--brand); font-weight:700; box-shadow:var(--shadow-sm);}

.app-layout { display: flex; max-width: 1400px; margin: 0 auto; width: 100%; flex: 1; }

.history-sidebar {
  width: 260px; flex: none; padding: 24px 20px;
  border-right: 1px solid var(--line);
  background: rgba(255, 255, 255, 0.5);
  position: sticky; top: 65px;
  height: calc(100vh - 65px);
  overflow-y: auto;
}
.history-sidebar::-webkit-scrollbar { width: 4px; }
.history-sidebar::-webkit-scrollbar-thumb { background: #cbd5e1; border-radius: 4px; }

.history-header { display: flex; justify-content: space-between; align-items: center; font-size: 13px; font-weight: 800; color: var(--ink-3); margin-bottom: 20px; padding-left: 4px; letter-spacing: 0.05em; }
.clear-btn { background: none; border: none; color: var(--ink-4); font-size: 12px; cursor: pointer; padding: 4px 8px; border-radius: 6px; transition: all 0.2s; }
.clear-btn:hover { background: var(--danger-soft); color: var(--danger); }
.history-list { list-style: none; margin: 0; padding: 0; }

.history-item { 
  display: flex; align-items: center; justify-content: space-between;
  padding: 12px 14px; border-radius: 12px; cursor: pointer; margin-bottom: 8px; 
  border: 1px solid transparent; transition: all 0.2s cubic-bezier(0.4, 0, 0.2, 1); position: relative;
  background: rgba(255, 255, 255, 0.4);
}
.history-item:hover { background: #fff; border-color: var(--line); box-shadow: var(--shadow-sm); transform: translateY(-1px); }

.history-item.is-pinned { background: var(--accent-soft); border-color: var(--accent-line); }
.history-item.is-pinned .h-title { color: var(--accent); }
.history-item.is-pinned:hover .h-title { color: var(--accent); }

.h-content { flex: 1; min-width: 0; }
.h-title { font-size: 13.5px; color: var(--ink-2); font-weight: 700; white-space: nowrap; overflow: hidden; text-overflow: ellipsis; margin-bottom: 4px; }
.history-item:hover .h-title { color: var(--brand); }
.h-meta { font-size: 11.5px; color: var(--ink-4); }
.history-list li.empty-tip { color: var(--ink-4); cursor: default; text-align: center; padding-top: 30px; font-size: 13px; }
.history-list li.empty-tip:hover { background: transparent; border-color: transparent; box-shadow: none; }

.action-btns { display: none; align-items: center; gap: 2px; margin-left: 8px; flex: none; }
.history-item:hover .action-btns { display: flex; }
.icon-btn { background: none; border: none; color: var(--ink-4); cursor: pointer; padding: 5px; border-radius: 6px; transition: all 0.2s; display: flex; align-items: center; justify-content: center; }
.pin-btn:hover { background: #fff; color: var(--accent); box-shadow: var(--shadow-sm); }
.history-item.is-pinned .pin-btn { display: flex; color: var(--accent); }
.history-item.is-pinned .action-btns { display: flex; }
.delete-btn:hover { background: #fff; color: var(--danger); box-shadow: var(--shadow-sm); }

.main-content { flex: 1; min-width: 0; }
.wrap { max-width: 100%; padding: 0 32px 96px; }

.card { 
  background: linear-gradient(180deg, #ffffff 0%, #fafbfc 100%); 
  border: 1px solid var(--line); border-radius: var(--radius); box-shadow: var(--shadow-md); 
  padding: 32px; margin-top: 24px; transition: transform 0.3s ease, box-shadow 0.3s ease; 
}
.card:focus-within { box-shadow: var(--shadow-lg); border-color: #cbd5e1; }
.card-title { font-size: 16px; font-weight: 800; margin: 0 0 6px; color: #0f172a; }
.hint { font-size: 13px; color: var(--ink-3); margin: 0; }
.field { position: relative; margin-top: 16px; }
textarea#input { 
  width: 100%; min-height: 120px; resize: vertical; padding: 16px 16px 36px; 
  border: 1px solid var(--line); border-radius: 12px; font: inherit; font-size: 15px; 
  line-height: 1.6; background: #fff; outline: none; transition: all 0.2s; 
  display: block; margin: 0; box-shadow: inset 0 2px 4px rgba(0,0,0,0.01);
}
textarea#input:focus { border-color: var(--brand); box-shadow: 0 0 0 4px var(--brand-soft); }
.counter { position: absolute; right: 16px; bottom: 12px; font-size: 12px; color: var(--ink-4); font-variant-numeric: tabular-nums; pointer-events: none; font-weight: 600; }
.counter.warn { color: var(--accent); }
.counter.over { color: var(--danger); }

.row { display: flex; align-items: center; gap: 16px; flex-wrap: wrap; margin-top: 20px; }
.btn { appearance: none; border: 1px solid transparent; border-radius: 10px; cursor: pointer; font: inherit; font-weight: 700; transition: all 0.2s cubic-bezier(0.4, 0, 0.2, 1); }
.btn-primary { 
  background: linear-gradient(135deg, #3b82f6 0%, #2563eb 100%); color: #fff; 
  box-shadow: 0 4px 12px rgba(37, 99, 235, 0.25); margin-left: auto; padding: 12px 28px; font-size: 15px; 
}
.btn-primary:hover { background: linear-gradient(135deg, #2563eb 0%, #1d4ed8 100%); transform: translateY(-2px); box-shadow: 0 6px 16px rgba(37, 99, 235, 0.35); }
.btn-primary:active { transform: translateY(0); box-shadow: 0 2px 4px rgba(37, 99, 235, 0.2); }
.btn-primary:disabled { background: #cbd5e1 !important; cursor: not-allowed; box-shadow: none !important; transform: none !important; }
.btn-ghost { background: #fff; border-color: var(--line); color: var(--ink-2); font-weight: 600; font-size: 13px; padding: 8px 16px; box-shadow: var(--shadow-sm); }
.btn-ghost:hover { border-color: var(--brand-line); color: var(--brand); background: var(--brand-soft); }

.debugbar { margin-top: 20px; padding: 12px 16px; background: #f8fafc; border: 1px dashed #cbd5e1; border-radius: 12px; display: flex; align-items: center; gap: 8px; flex-wrap: wrap; }
.debugbar .lb { font-size: 12px; font-weight: 700; color: var(--ink-4); margin-right: 4px; }
.chip { appearance: none; border: 1px solid #e2e8f0; background: #fff; border-radius: 999px; font: inherit; font-size: 12px; font-weight: 500; color: var(--ink-3); padding: 4px 12px; cursor: pointer; transition: all 0.2s; box-shadow: var(--shadow-sm); }
.chip:hover { border-color: #cbd5e1; color: var(--brand); transform: translateY(-1px); }
.chip[aria-pressed="true"] { background: var(--brand); border-color: var(--brand); color: #fff; font-weight: 700; box-shadow: 0 2px 8px rgba(37,99,235,0.3); }

#inputCard.collapsed { padding: 16px 24px; background: #fff; }
#inputCard.collapsed .input-main { display: none; }
.input-collapsed { display: none; align-items: center; gap: 12px; }
#inputCard.collapsed .input-collapsed { display: flex; }
.ic-label { font-size: 12px; font-weight: 700; color: var(--ink-4); flex: none; background: #f1f5f9; padding: 4px 8px; border-radius: 6px; }
.ic-text { font-size: 15px; color: var(--ink); font-weight: 800; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }
.input-collapsed .btn-ghost { margin-left: auto; flex: none; }

/* =========================================
   ✨ Loading 状态与光晕涟漪动画
   ========================================= */
.loading { display: none; margin-top: 24px; }
.loading.on { display: block; }
.load-card { 
  background: #fff; border: 1px solid var(--line); border-radius: var(--radius); 
  padding: 32px; 
  animation: cardPulse 2s infinite alternate;
}
@keyframes cardPulse {
  0% { box-shadow: 0 0 10px 0 rgba(37, 99, 235, 0.05); border-color: var(--line); }
  100% { box-shadow: 0 0 30px 4px rgba(37, 99, 235, 0.15); border-color: var(--brand-line); }
}

.load-head { display: flex; align-items: center; gap: 16px; }

.owl-reading-loader {
  position: relative;
  width: 60px;
  height: 60px;
  display: flex;
  justify-content: center;
  align-items: center;
  flex: none;
}

.owl-img {
  width: 38px;
  height: 38px;
  object-fit: contain;
  z-index: 2;
  animation: owlFloat 2s ease-in-out infinite;
  mix-blend-mode: multiply;
}

.ripple {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%) scale(0.6);
  width: 50px;
  height: 50px;
  border-radius: 50%;
  border: 2px solid var(--brand-line);
  opacity: 0;
  animation: pulseRipple 2s infinite ease-out;
  z-index: 1;
}

.ripple-delay {
  animation-delay: 1s;
}

@keyframes owlFloat {
  0%, 100% { transform: translateY(0); }
  50% { transform: translateY(-4px); }
}

@keyframes pulseRipple {
  0% { transform: translate(-50%, -50%) scale(0.6); opacity: 1; border-color: var(--brand); }
  100% { transform: translate(-50%, -50%) scale(1.6); opacity: 0; border-color: var(--brand-line); }
}

.load-text-box { display: flex; flex-direction: column; gap: 4px; }
.load-text { font-size: 15px; font-weight: 700; color: var(--ink); transition: opacity 0.2s ease; }
.load-sub { font-size: 12.5px; color: var(--ink-4); font-weight: 500; }
.load-sec { font-variant-numeric: tabular-nums; color: var(--brand); font-weight: 800; font-size: 13.5px; }

.bar { height: 6px; background: #e2e8f0; border-radius: 4px; margin-top: 20px; overflow: hidden; position: relative; }
.bar span { 
  display: block; height: 100%; width: 0; 
  background: linear-gradient(90deg, #3b82f6, #8b5cf6, #3b82f6); 
  background-size: 200% 100%; border-radius: 4px; 
  transition: width .5s ease-out; animation: flowGradient 2s linear infinite;
}
@keyframes flowGradient { 0% { background-position: 100% 0; } 100% { background-position: -100% 0; } }

.error { display: none; margin-top: 24px; }
.error.on { display: block; }
.error-card { border: 1px solid #fecaca; background: var(--danger-soft); border-radius: var(--radius); padding: 20px 24px; display: flex; gap: 16px; box-shadow: var(--shadow-sm); }
.error-icon { flex: none; width: 24px; height: 24px; border-radius: 50%; background: var(--danger); color: #fff; font-size: 14px; font-weight: 900; display: grid; place-items: center; margin-top: 2px; }
.error-code { font-size: 12px; letter-spacing: .05em; color: var(--danger); font-weight: 800; }
.error-msg { font-size: 15px; color: #991b1b; margin: 4px 0 0; font-weight: 500; }

.result { display: none; margin-top: 24px; }
.result.on { display: block; animation: slideUp 0.4s ease-out both; }
@keyframes slideUp { from { opacity: 0; transform: translateY(12px); } to { opacity: 1; transform: translateY(0); } }

.report-tabs { display: flex; gap: 8px; margin-bottom: 20px; overflow-x: auto; padding-bottom: 8px; }
.report-tabs::-webkit-scrollbar { height: 4px; }
.report-tabs::-webkit-scrollbar-thumb { background: #cbd5e1; border-radius: 4px; }
.report-tab { 
  display: flex; align-items: center; gap: 10px; padding: 10px 18px; 
  background: #fff; border: 1px solid var(--line); border-radius: 999px; 
  cursor: pointer; font-size: 13.5px; font-weight: 600; color: var(--ink-3); 
  white-space: nowrap; transition: all 0.25s cubic-bezier(0.4, 0, 0.2, 1); box-shadow: var(--shadow-sm); 
}
.report-tab:hover { border-color: #cbd5e1; transform: translateY(-1px); }
.report-tab.active { 
  background: var(--brand); color: #fff; border-color: var(--brand); 
  box-shadow: 0 4px 12px rgba(37,99,235,0.25); transform: translateY(-1px);
}
.tab-close { background: none; border: none; color: currentColor; opacity: 0.6; cursor: pointer; font-size: 18px; line-height: 1; padding: 0; transition: opacity 0.2s; }
.tab-close:hover { opacity: 1; }

.note-grid { display: grid; grid-template-columns: 1fr; gap: 28px; align-items: start; }
@media(min-width: 1100px) { .note-grid { grid-template-columns: 200px minmax(0, 1fr); } }

.toc { display: none; position: sticky; top: 90px; padding: 8px 0; }
@media(min-width: 1100px) { .toc { display: block; } }
.toc-h { font-size: 11px; letter-spacing: 0.15em; color: var(--ink-4); margin: 0 0 16px 16px; font-weight: 800; }
.toc-list { list-style: none; margin: 0; padding: 0; border-left: 2px solid #e2e8f0; }
.toc-list a { display: flex; align-items: baseline; gap: 8px; text-decoration: none; padding: 8px 12px 8px 16px; margin-left: -2px; border-left: 2px solid transparent; font-size: 13.5px; font-weight: 600; color: var(--ink-3); transition: all 0.2s; line-height: 1.5; border-radius: 0 8px 8px 0; }
.toc-list a:hover { color: var(--brand); background: #f8fafc; }
.toc-list a.active { color: var(--brand); font-weight: 800; border-left-color: var(--brand); background: var(--brand-soft); }
.toc-list .n { font-size: 12px; color: var(--ink-4); flex: none; }
.toc-list a.active .n { color: var(--brand); }
.toc-list .c { margin-left: auto; font-size: 11.5px; color: var(--ink-4); background: #f1f5f9; padding: 2px 6px; border-radius: 10px; }
.toc-list a.active .c { background: #fff; color: var(--brand); }

.sheet { 
  background: #ffffff; border: 1px solid var(--line); border-radius: 20px; 
  box-shadow: var(--shadow-lg); padding: 48px 56px 40px; min-width: 0; 
  position: relative; overflow: hidden;
}
.sheet::before {
  content: ''; position: absolute; top: 0; left: 0; right: 0; height: 6px;
  background: linear-gradient(90deg, #3b82f6, #8b5cf6, #ec4899);
}
@media(max-width: 640px) { .sheet { padding: 32px 24px; } }

.note-eyebrow { font-size: 12px; letter-spacing: 0.1em; color: var(--brand); font-weight: 800; text-transform: uppercase; }
.note-title { font-size: 32px; line-height: 1.3; font-weight: 900; margin: 12px 0 0; color: #0f172a; letter-spacing: -0.02em; }
.note-meta { display: flex; gap: 16px; flex-wrap: wrap; margin-top: 16px; font-size: 13px; color: var(--ink-4); font-weight: 500; }
.note-meta b { color: var(--ink-2); font-weight: 800; }
.note-head { padding-bottom: 28px; border-bottom: 2px solid #0f172a; margin-bottom: 32px; }

.ch + .ch { margin-top: 40px; padding-top: 36px; border-top: 1px solid var(--line-2); }
.ch-h { display: flex; align-items: baseline; gap: 12px; margin: 0 0 20px; font-size: 18px; font-weight: 800; color: #0f172a; }
.ch-no { font-size: 13px; color: #fff; background: var(--brand); border-radius: 6px; padding: 2px 8px; flex: none; font-weight: 800; box-shadow: 0 2px 4px rgba(37,99,235,0.3); }
.ch-cnt { margin-left: auto; font-size: 12px; color: var(--ink-4); font-weight: 600; background: #f1f5f9; padding: 2px 10px; border-radius: 99px; }
.summary { font-size: 18px; line-height: 1.8; margin: 0; color: var(--ink-2); font-weight: 500; position: relative; padding-left: 16px; }
.summary::before { content: ''; position: absolute; left: 0; top: 6px; bottom: 6px; width: 4px; background: var(--brand); border-radius: 4px; }

.skeleton { display: flex; align-items: center; gap: 10px; flex-wrap: wrap; padding: 12px 16px; background: #f8fafc; border: 1px solid var(--line); border-radius: 10px; margin-bottom: 24px; }
.skeleton .lb { font-size: 12px; font-weight: 700; color: var(--ink-4); }
.sk-step { font-size: 12.5px; color: var(--brand); background: #fff; border: 1px solid var(--brand-line); border-radius: 6px; padding: 3px 10px; font-weight: 700; box-shadow: var(--shadow-sm); }
.sk-arrow { color: #cbd5e1; font-size: 14px; font-weight: 900; }

ol.points { list-style: none; margin: 0; padding: 0; }
.pt { display: flex; gap: 16px; padding: 0 0 20px; transition: all 0.2s; }
.pt + .pt { padding-top: 20px; border-top: 1px dashed var(--line-2); }
.pt:last-child { padding-bottom: 0; }
.pt:hover .pt-text { color: #0f172a; }
.pt-no { 
  flex: none; width: 28px; height: 28px; border-radius: 50%; 
  background: var(--brand-soft); color: var(--brand); 
  font-size: 12px; font-weight: 800; display: grid; place-items: center; 
  margin-top: 2px; border: 1px solid var(--brand-line);
}
.pt-body { min-width: 0; flex: 1; }
.pt-text { margin: 0; font-size: 15.5px; line-height: 1.7; color: var(--ink-2); font-weight: 500; transition: color 0.2s; }
.pt-src { display: flex; align-items: center; gap: 8px; flex-wrap: wrap; margin-top: 10px; }
.pt-src .lb { font-size: 12px; color: var(--ink-4); font-weight: 700; }
.ref { 
  appearance: none; border: 1px solid var(--brand-line); background: var(--brand-soft); color: var(--brand); 
  font: inherit; font-size: 11.5px; font-weight: 800; line-height: 1; padding: 5px 8px; border-radius: 6px; 
  cursor: pointer; transition: all 0.2s; 
}
.ref:hover { background: var(--brand); color: #fff; border-color: var(--brand); transform: scale(1.05); }
.ref.cite-on { background: var(--accent); border-color: var(--accent); color: #fff; box-shadow: 0 2px 8px rgba(217,119,6,0.3); }

.grp { list-style: none; display: flex; align-items: center; gap: 12px; margin: 12px 0 20px; font-size: 13px; font-weight: 800; color: var(--accent); letter-spacing: 0.05em; }
.grp::after { content: ""; flex: 1; height: 1px; background: var(--accent-line); }
.grp:first-child { margin-top: 0; }

.terms { margin: 0; }
.tm { display: flex; gap: 16px; padding: 16px 0; transition: background 0.2s; border-radius: 8px; }
.tm:hover { background: #f8fafc; padding-left: 8px; padding-right: 8px; margin: 0 -8px; }
.tm + .tm { border-top: 1px solid var(--line-2); }
.tm-no { flex: none; font-size: 13px; color: var(--ink-4); font-weight: 800; padding-top: 2px; }
.tm-term { font-size: 15px; font-weight: 800; color: var(--brand); margin: 0 0 4px; }
.tm-explain { margin: 0; font-size: 14.5px; color: var(--ink-3); line-height: 1.6; }

ul.questions { list-style: none; margin: 0; padding: 0; }
.qs { display: flex; gap: 16px; padding: 12px 0; }
.qs-no { flex: none; font-size: 13px; color: var(--ink-4); font-weight: 800; padding-top: 3px; }
.qs-text { margin: 0; font-size: 15px; color: var(--ink-2); line-height: 1.6; font-weight: 500; }
.qs-text::before { content: "待查"; font-size: 11px; font-weight: 800; color: var(--accent); border: 1px solid var(--accent-line); background: var(--accent-soft); border-radius: 4px; padding: 2px 6px; margin-right: 8px; vertical-align: 2px; }

ol.sources { list-style: none; margin: 0; padding: 0; }
.src { display: flex; gap: 16px; padding: 16px 12px; border-radius: 10px; border: 1px solid transparent; transition: all 0.2s; }
.src + .src { border-top: 1px solid var(--line-2); border-radius: 0; }
.src:hover { background: #f8fafc; border-radius: 10px; border-color: var(--line); }
.src.cite-on { background: var(--accent-soft); border-color: var(--accent-line); border-radius: 10px; }
.sn { flex: none; font-size: 12px; font-weight: 800; color: var(--ink-2); background: #f1f5f9; border-radius: 6px; padding: 3px 8px; height: 22px; line-height: 16px; }
.src-body { min-width: 0; }
.src a { display: block; color: var(--ink); text-decoration: none; font-size: 14.5px; font-weight: 600; line-height: 1.5; transition: color 0.2s; }
.src a:hover { color: var(--brand); }
.src-meta { font-size: 12px; color: var(--ink-4); margin-top: 6px; font-weight: 500; }
.src-meta .host { word-break: break-all; }
.src-meta .backref { color: var(--accent); font-weight: 700; }

.sheet-foot { margin-top: 40px; padding-top: 24px; border-top: 1px solid var(--line); font-size: 12.5px; color: var(--ink-4); display: flex; justify-content: space-between; flex-wrap: wrap; font-weight: 500; }

.pop { 
  position: absolute; z-index: 60; width: 320px; background: rgba(255,255,255,0.95); 
  backdrop-filter: blur(10px); border: 1px solid var(--line); border-radius: 12px; 
  box-shadow: 0 20px 40px -10px rgba(0,0,0,0.15); padding: 16px 20px; display: none; font-size: 13px; 
}
.pop.on { display: block; animation: popIn 0.2s cubic-bezier(0.16, 1, 0.3, 1); }
@keyframes popIn { from{opacity:0; transform: scale(0.95) translateY(5px);} to{opacity:1; transform: scale(1) translateY(0);} }
.pop .pop-h { font-size: 11px; font-weight: 800; letter-spacing: 0.1em; color: var(--brand); margin-bottom: 8px; }
.pop .pop-t { font-size: 14px; font-weight: 700; color: var(--ink); line-height: 1.5; margin-bottom: 6px; }
.pop .pop-u { font-size: 11.5px; color: var(--ink-4); word-break: break-all; margin-bottom: 12px; }
.pop .pop-a { display: inline-block; font-size: 13px; color: var(--brand); text-decoration: none; font-weight: 800; background: var(--brand-soft); padding: 4px 10px; border-radius: 6px; transition: all 0.2s;}
.pop .pop-a:hover { background: var(--brand); color: #fff; }

.toast { position: fixed; left: 50%; bottom: 40px; transform: translate(-50%, 20px); background: rgba(15,23,42,0.9); backdrop-filter: blur(4px); color: #fff; font-size: 13.5px; font-weight: 600; padding: 10px 20px; border-radius: 999px; opacity: 0; pointer-events: none; transition: all 0.3s cubic-bezier(0.16, 1, 0.3, 1); z-index: 80; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.2); }
.toast.on { opacity: 1; transform: translate(-50%, 0); }

@media(max-width: 800px){
  .app-layout { flex-direction: column; }
  .history-sidebar { width: 100%; min-height: auto; position: static; border-right: none; border-bottom: 1px solid var(--line); padding: 16px; height: auto; background: transparent; }
  .history-list { display: flex; overflow-x: auto; gap: 10px; padding-bottom: 10px; }
  .history-item { margin-bottom: 0; white-space: nowrap; flex: none; background: #fff; border: 1px solid var(--line); }
  .action-btns { display: flex !important; }
  .wrap { padding: 0 16px 60px; }
  .card { padding: 20px; margin-top: 16px; }
  .top-inner { padding: 12px 16px; }
  .tagline { display: none; }
  .row .btn-primary { margin-left: 0; width: 100%; }
  .note-title { font-size: 24px; }
  .sheet { padding: 24px 20px; }
}

/* =========================================
   ✨ 入场动画 (Page Load Animations)
   ========================================= */
header.top {
  animation: slideDown 0.6s cubic-bezier(0.16, 1, 0.3, 1) both;
}
.history-sidebar {
  animation: slideRight 0.7s cubic-bezier(0.16, 1, 0.3, 1) both;
  animation-delay: 0.1s;
}
.card {
  animation: popInUp 0.8s cubic-bezier(0.16, 1, 0.3, 1) both;
  animation-delay: 0.2s;
}
.debugbar {
  animation: fadeIn 1s ease both;
  animation-delay: 0.5s;
}

@keyframes slideDown {
  from { opacity: 0; transform: translateY(-20px); }
  to { opacity: 1; transform: translateY(0); }
}
@keyframes slideRight {
  from { opacity: 0; transform: translateX(-30px); }
  to { opacity: 1; transform: translateX(0); }
}
@keyframes popInUp {
  from { opacity: 0; transform: translateY(30px) scale(0.98); }
  to { opacity: 1; transform: translateY(0) scale(1); }
}
@keyframes fadeIn {
  from { opacity: 0; }
  to { opacity: 1; }
}
</style>