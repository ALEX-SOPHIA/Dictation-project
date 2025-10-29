# 芬兰语听写网站（MVP）

一个最小可用版本，满足：
1) 用户输入文本；2) 将文本转成“自然”语音；3) 用户听写并校对。

下面给出两条路线：
- **Route A（零门槛原型）**：浏览器内置语音合成（Web Speech Synthesis API）。优点：无需服务器与密钥；缺点：不同浏览器/系统的芬兰语音质差异大，稳定性/自然度一般。
- **Route B（推荐生产）**：后端调用神经网络TTS（例如 Azure/Google/AWS）或离线引擎（Piper）。优点：声音更自然、可控；缺点：需要后端与（或）模型/云密钥。

---

## 目录
- [功能与交互流程](#功能与交互流程)
- [Route A：纯前端原型（可立即运行）](#route-a纯前端原型可立即运行)
  - [index.html](#indexhtml)
- [Route B：FastAPI 后端 + 神经TTS（推荐）](#route-bfastapi-后端--神经tts推荐)
  - [后端：main.py（FastAPI）](#后端mainpyfastapi)
  - [前端：index.html（调用后端TTS）](#前端indexhtml调用后端tts)
  - [运行步骤](#运行步骤)
- [判题与高亮比对逻辑](#判题与高亮比对逻辑)
- [可选增强](#可选增强)

---

## 功能与交互流程
1. 用户在左侧**输入待听写文本**（系统不会立刻显示给学习者，可由“出题人/家长/老师”输入）。
2. 点击**生成音频**并播放（可选择常速/慢速）。
3. 学习者在右侧**进行听写**。
4. 点击**检查**：展示**对/错统计、字符级/词级差异高亮**；可选择逐词揭示、再次慢速回放。

---

## Route A：纯前端原型（可立即运行）
> 适合先体验流程。如果你的浏览器支持芬兰语 `fi-FI` 的合成声音，会有较自然的效果；否则可能回退到其他声音。

### index.html
```html
<!doctype html>
<html lang="fi">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>Finnish Dictation (Prototype)</title>
  <style>
    body { font-family: system-ui, -apple-system, Segoe UI, Roboto, Arial, sans-serif; margin: 24px; }
    .wrap { display: grid; grid-template-columns: 1fr 1fr; gap: 16px; }
    textarea { width: 100%; min-height: 160px; padding: 8px; font-size: 16px; }
    .card { border: 1px solid #e5e7eb; border-radius: 12px; padding: 16px; box-shadow: 0 1px 4px rgba(0,0,0,0.06); }
    .row { display: flex; gap: 8px; flex-wrap: wrap; align-items: center; }
    button { padding: 8px 12px; border-radius: 10px; border: 1px solid #e5e7eb; background: #fff; cursor: pointer; }
    .btn-primary { background: #111827; color: #fff; }
    .muted { color: #6b7280; font-size: 14px; }
    .diff span.add { background: #d1fae5; }
    .diff span.del { background: #fee2e2; text-decoration: line-through; }
    .diff span.same { background: none; }
  </style>
</head>
<body>
  <h1>Finnish Dictation — Prototype (No Backend)</h1>
  <p class="muted">Tip: This uses the browser's speech synthesis. For best naturalness use the backend route below.</p>
  <div class="wrap">
    <div class="card">
      <h3>Source Text (teacher/parent)</h3>
      <textarea id="source" placeholder="Kirjoita teksti tähän..."></textarea>
      <div class="row" style="margin-top:8px">
        <label>Rate <input id="rate" type="range" min="0.5" max="1.2" step="0.05" value="1"></label>
        <label>Pitch <input id="pitch" type="range" min="0.7" max="1.3" step="0.05" value="1"></label>
        <button id="speak" class="btn-primary">Play</button>
        <button id="speakSlow">Play Slow</button>
      </div>
      <div class="muted" id="voiceInfo"></div>
    </div>

    <div class="card">
      <h3>Dictation (learner)</h3>
      <textarea id="answer" placeholder="Kuuntele ja kirjoita..."></textarea>
      <div class="row" style="margin-top:8px">
        <button id="check" class="btn-primary">Check</button>
        <button id="reveal">Reveal Next Word</button>
      </div>
      <div id="result" style="margin-top:12px"></div>
    </div>
  </div>

  <script>
    const $ = (s) => document.querySelector(s);
    const srcEl = $('#source');
    const ansEl = $('#answer');
    const resEl = $('#result');

    // ---- Voice selection (fi-FI preferred) ----
    let fiVoice = null;
    function pickVoice() {
      const voices = speechSynthesis.getVoices();
      const fi = voices.filter(v => /fi\-FI/i.test(v.lang) || /finnish/i.test(v.name));
      fiVoice = fi[0] || voices.find(v => v.default) || voices[0] || null;
      $('#voiceInfo').textContent = fiVoice ? `Using voice: ${fiVoice.name} (${fiVoice.lang})` : 'No voice available';
    }
    speechSynthesis.onvoiceschanged = pickVoice; pickVoice();

    function speak(text, rate=1, pitch=1){
      if(!text.trim()) return;
      const u = new SpeechSynthesisUtterance(text);
      if(fiVoice) u.voice = fiVoice;
      u.lang = fiVoice?.lang || 'fi-FI';
      u.rate = rate; u.pitch = pitch;
      speechSynthesis.cancel();
      speechSynthesis.speak(u);
    }

    $('#speak').onclick = () => speak(srcEl.value, parseFloat($('#rate').value), parseFloat($('#pitch').value));
    $('#speakSlow').onclick = () => speak(srcEl.value, 0.8, parseFloat($('#pitch').value));

    // ---- Diff (word-level simple LCS) ----
    function normalize(s){
      return s.replace(/[\n\r]/g, ' ').replace(/\s+/g, ' ').trim();
    }
    function diffWords(a, b){
      const A = normalize(a).split(' ');
      const B = normalize(b).split(' ');
      const m=A.length, n=B.length;
      const dp = Array.from({length:m+1}, ()=>Array(n+1).fill(0));
      for(let i=1;i<=m;i++) for(let j=1;j<=n;j++) dp[i][j] = A[i-1]===B[j-1]? dp[i-1][j-1]+1 : Math.max(dp[i-1][j], dp[i][j-1]);
      const out=[]; let i=m,j=n;
      while(i>0 && j>0){
        if(A[i-1]===B[j-1]){ out.push(['same', A[i-1]]); i--; j--; }
        else if(dp[i-1][j] >= dp[i][j-1]){ out.push(['del', A[i-1]]); i--; }
        else { out.push(['add', B[j-1]]); j--; }
      }
      while(i>0){ out.push(['del', A[i-1]]); i--; }
      while(j>0){ out.push(['add', B[j-1]]); j--; }
      return out.reverse();
    }

    function renderDiff(a,b){
      const d = diffWords(a,b);
      const html = d.map(([t,w])=>`<span class="${t}">${w}</span>`).join(' ');
      return `<div class="diff">${html}</div>`;
    }

    $('#check').onclick = () => {
      const s = normalize(srcEl.value).toLowerCase();
      const a = normalize(ansEl.value).toLowerCase();
      const d = renderDiff(s, a);
      const sCount = s ? s.split(' ').length : 0;
      const aCount = a ? a.split(' ').length : 0;
      const correct = s && a ? s.split(' ').filter((w,i)=> w===a.split(' ')[i]).length : 0; // rough
      resEl.innerHTML = `<div><b>Words (target/answer):</b> ${sCount} / ${aCount}</div>${d}`;
    };

    // Reveal next word
    let revealIndex = 0;
    $('#reveal').onclick = () => {
      const words = normalize(srcEl.value).split(' ');
      if(revealIndex < words.length){
        ansEl.value = (ansEl.value + ' ' + words[revealIndex]).trim();
        revealIndex++;
      }
    };
  </script>
</body>
</html>
```

---

## Route B：FastAPI 后端 + 神经TTS（推荐）
> 你可以接入 **Azure/Google/AWS** 的芬兰语神经语音，或使用 **Piper** 等离线TTS。下面示例用 **FastAPI** 提供 `/api/tts` 接口：前端将文本POST过去，返回 `audio/mpeg` 并播放。

### 后端：`main.py`（FastAPI）
> 以 Azure Speech 为例（声音更自然）。请在环境变量中配置 `AZURE_SPEECH_KEY` 和 `AZURE_SPEECH_REGION`。**注意：具体芬兰语音名称以你账号可用为准**（示例占位：`fi-FI-NooraNeural`），若报错，把 voice 名称换成你在 Azure 门户里能看到的芬兰语 Neural voice。

```python
# main.py
import os
from fastapi import FastAPI, HTTPException
from fastapi.responses import Response
from pydantic import BaseModel
import azure.cognitiveservices.speech as speechsdk

app = FastAPI()

class TTSReq(BaseModel):
    text: str
    voice: str | None = None  # e.g., "fi-FI-NooraNeural" (update to actual available voice)
    speaking_rate: float | None = 1.0  # 0.5~2.0

AZURE_KEY = os.getenv("AZURE_SPEECH_KEY")
AZURE_REGION = os.getenv("AZURE_SPEECH_REGION")
DEFAULT_VOICE = "fi-FI-NooraNeural"  # TODO: replace with an available Finnish neural voice in your subscription

if not AZURE_KEY or not AZURE_REGION:
    print("[WARN] Missing AZURE_SPEECH_KEY or AZURE_SPEECH_REGION env vars")

@app.post("/api/tts")
async def tts(req: TTSReq):
    text = (req.text or "").strip()
    if not text:
        raise HTTPException(400, detail="Empty text")
    voice = req.voice or DEFAULT_VOICE
    rate = req.speaking_rate or 1.0

    speech_config = speechsdk.SpeechConfig(subscription=AZURE_KEY, region=AZURE_REGION)
    speech_config.set_speech_synthesis_output_format(
        speechsdk.SpeechSynthesisOutputFormat.Audio16Khz128KBitRateMonoMp3
    )
    audio_config = speechsdk.audio.AudioOutputConfig(use_default_speaker=False)
    synthesizer = speechsdk.SpeechSynthesizer(speech_config=speech_config, audio_config=audio_config)

    # Use SSML for rate control
    ssml = f"""
    <speak version='1.0' xml:lang='fi-FI'>
      <voice name='{voice}'>
        <prosody rate='{int(rate*100)}%'>
          {speechsdk.ssml.escape_ssml(text)}
        </prosody>
      </voice>
    </speak>
    """.strip()

    result = synthesizer.speak_ssml_async(ssml).get()
    if result.reason == speechsdk.ResultReason.SynthesizingAudioCompleted:
        audio = result.audio_data
        return Response(content=audio, media_type="audio/mpeg")
    else:
        raise HTTPException(500, detail=f"Synthesis failed: {result.reason}")
```

> **离线方案（Piper）**：如果想完全离线，可将 `/api/tts` 改成调用 Piper 可执行文件与芬兰语模型（例如 *fi_FI* 语音）。思路：将文本写到临时文件，调用 `subprocess.run(["piper", "-m", model_path, "-f", wav_out, "-t", text])`，读回 `wav` 或转 `mp3` 后返回。

### 前端：`index.html`（调用后端TTS）
```html
<!doctype html>
<html lang="fi">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>Finnish Dictation</title>
  <style>
    body { font-family: system-ui, -apple-system, Segoe UI, Roboto, Arial, sans-serif; margin: 24px; }
    .wrap { display: grid; grid-template-columns: 1fr 1fr; gap: 16px; }
    textarea { width: 100%; min-height: 160px; padding: 8px; font-size: 16px; }
    .card { border: 1px solid #e5e7eb; border-radius: 12px; padding: 16px; box-shadow: 0 1px 4px rgba(0,0,0,0.06); }
    .row { display: flex; gap: 8px; flex-wrap: wrap; align-items: center; }
    button { padding: 8px 12px; border-radius: 10px; border: 1px solid #e5e7eb; background: #fff; cursor: pointer; }
    .btn-primary { background: #111827; color: #fff; }
    .muted { color: #6b7280; font-size: 14px; }
  </style>
</head>
<body>
  <h1>Finnish Dictation — Neural TTS</h1>
  <div class="wrap">
    <div class="card">
      <h3>Source Text</h3>
      <textarea id="source" placeholder="Kirjoita teksti tähän..."></textarea>
      <div class="row" style="margin-top:8px">
        <input id="voice" placeholder="fi-FI neural voice (optional)" style="flex:1" />
        <label>Rate <input id="rate" type="range" min="0.6" max="1.4" step="0.05" value="1"></label>
        <button id="gen" class="btn-primary">Generate & Play</button>
        <button id="genSlow">Play Slow</button>
      </div>
      <audio id="player" controls style="width:100%; margin-top:8px;"></audio>
    </div>

    <div class="card">
      <h3>Dictation</h3>
      <textarea id="answer" placeholder="Kuuntele ja kirjoita..."></textarea>
      <div class="row" style="margin-top:8px">
        <button id="check" class="btn-primary">Check</button>
        <button id="reset">Reset</button>
      </div>
      <div id="result" style="margin-top:12px"></div>
    </div>
  </div>

  <script>
    const $ = (s) => document.querySelector(s);
    const srcEl = $('#source');
    const ansEl = $('#answer');
    const resEl = $('#result');
    const player = $('#player');

    async function synth(rate){
      const text = srcEl.value.trim();
      if(!text) return;
      const voice = $('#voice').value.trim() || null; // e.g., fi-FI-NooraNeural (your available voice)
      const body = { text, voice, speaking_rate: rate };
      const r = await fetch('/api/tts', {
        method: 'POST', headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(body)
      });
      if(!r.ok){
        const msg = await r.text();
        alert('TTS failed: '+msg); return;
      }
      const blob = await r.blob();
      player.src = URL.createObjectURL(blob);
      player.play();
    }

    $('#gen').onclick = () => synth(parseFloat($('#rate').value) || 1.0);
    $('#genSlow').onclick = () => synth(0.85);

    function normalize(s){
      return s.toLowerCase().replace(/[\n\r]/g,' ').replace(/\s+/g,' ').trim();
    }

    function diffWords(a, b){
      const A = normalize(a).split(' '), B = normalize(b).split(' ');
      const m=A.length, n=B.length, dp=Array.from({length:m+1},()=>Array(n+1).fill(0));
      for(let i=1;i<=m;i++) for(let j=1;j<=n;j++) dp[i][j]=A[i-1]===B[j-1]?dp[i-1][j-1]+1:Math.max(dp[i-1][j],dp[i][j-1]);
      const out=[]; let i=m,j=n; while(i>0&&j>0){
        if(A[i-1]===B[j-1]){out.push(['same',A[i-1]]); i--; j--;}
        else if(dp[i-1][j]>=dp[i][j-1]){out.push(['del',A[i-1]]); i--;}
        else {out.push(['add',B[j-1]]); j--;}
      }
      while(i>0){out.push(['del',A[i-1]]); i--;} while(j>0){out.push(['add',B[j-1]]); j--;}
      return out.reverse();
    }

    $('#check').onclick = () => {
      const target = normalize(srcEl.value);
      const ans = normalize(ansEl.value);
      const d = diffWords(target, ans).map(([t,w])=>`<span class="${t}">${w}</span>`).join(' ');
      resEl.innerHTML = `<style>.add{background:#d1fae5}.del{background:#fee2e2;text-decoration:line-through}.same{}</style>` + d;
    };

    $('#reset').onclick = () => { ansEl.value=''; resEl.innerHTML=''; };
  </script>
</body>
</html>
```

### 运行步骤
1. **创建并激活环境**（可选）
   ```bash
   python -m venv .venv && source .venv/bin/activate  # Windows: .venv\\Scripts\\activate
   pip install fastapi uvicorn azure-cognitiveservices-speech
   ```
2. **设置环境变量**（以 Azure 为例）
   ```bash
   export AZURE_SPEECH_KEY=your_key
   export AZURE_SPEECH_REGION=your_region   # e.g., northeurope, westeurope
   ```
3. **启动后端**
   ```bash
   uvicorn main:app --reload --port 8000
   ```
4. **前端放同一目录下**，用任意静态服务器或直接用浏览器打开 `index.html`（若跨域，可将 `index.html` 也由 FastAPI 静态托管，或用 `python -m http.server` 起本地端口，确保前端请求指向 `http://localhost:8000/api/tts`）。

---

## 判题与高亮比对逻辑
- 目前使用**词级 LCS**（最长公共子序列）做增删改高亮；已做基本规范化（小写、合并空白）。
- 如需**字符级**或**带重音敏感/不敏感**切换（芬兰语 ä/ö/å），可在 `normalize` 中做更细规则，或加入 **Levenshtein** 距离显示相似度。
- 可按句子分片播放与判题（断句符号：`.?!;:` 等），支持“逐句播放/逐句听写/逐句检查”。

---

## 可选增强
- **慢速/停顿**：用 SSML `<break time="500ms"/>` 插入停顿、或 `<prosody rate="85%">` 控速。
- **自动分句**：后端切分句子并逐句合成音频，前端提供句子列表与逐句播放控件。
- **保存成绩**：本地 `localStorage` 或后端数据库（SQLite/Postgres）。
- **多音色选择**：前端下拉列出可用芬兰语音（Azure/Google 的 voices 列表从后端拉取）。
- **录音比对**：加入 **Web Speech Recognition** 进行“口述→拼写”练习（另一个模式）。
- **Piper 离线 TTS**：在树莓派或本地服务器部署，零成本、隐私友好；选择高质量 *fi-FI* 模型即可。

---

### 你现在可以怎么做
- 想直接体验流程：先用 **Route A** 的 `index.html` 打开就能用。
- 想要更自然的声音和稳定效果：按 **Route B** 配好 Azure（或改写为 Google/AWS/Piper），把 `voice` 设置为你账号可用的芬兰语 **Neural** 声音名。

