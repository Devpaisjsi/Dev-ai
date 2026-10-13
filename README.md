
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <meta name="theme-color" content="#10121b">
  <title>Dev AI</title>
  <style>
    * { box-sizing: border-box; }
    body {
      margin: 0; background: #10121b; color: #f5f5fa;
      font-family: system-ui, sans-serif;
    }
    header {
      padding: 18px; text-align: center;
      border-bottom: 1px solid #303342;
    }
    header h2 { margin: 0; color: #a99cff; }
    #status {
      padding: 12px; text-align: center;
      color: #c5c6d4; font-size: 14px;
    }
    main {
      max-width: 700px; margin: auto; padding: 14px;
    }
    #chat {
      height: 58vh; min-height: 250px; overflow-y: auto;
      border: 1px solid #303342; border-radius: 14px;
      padding: 12px; background: #171a26;
    }
    .msg {
      padding: 11px 13px; margin: 10px 0;
      border-radius: 12px; white-space: pre-wrap;
      overflow-wrap: anywhere; line-height: 1.5;
    }
    .user { background: #37305d; margin-left: 12%; }
    .bot { background: #252938; margin-right: 8%; }
    #form { display: flex; gap: 8px; margin-top: 12px; }
    input {
      flex: 1; min-width: 0; padding: 14px;
      border: 1px solid #41445a; border-radius: 12px;
      background: #1c2030; color: white; font-size: 16px;
    }
    button {
      border: 0; border-radius: 12px; padding: 12px 16px;
      background: #8b79ff; color: #fff;
      font-size: 15px; font-weight: 600;
    }
    button:disabled { opacity: .5; }
    #retry { display: block; margin: 8px auto; }
    .hint { color: #aeb0c1; font-size: 12px; text-align: center; }
  </style>
</head>
<body>
  <header>
    <h2>✨ Dev AI</h2>
    <div>Apna personal AI assistant</div>
  </header>

  <div id="status">AI library load ho rahi hai...</div>
  <button id="retry" hidden>Try loading again</button>

  <main>
    <div id="chat">
      <div class="msg bot">Namaste! Main Dev AI hoon. Model load hone ke baad mujhse baat karo.</div>
    </div>
    <form id="form">
      <input id="question" placeholder="Message likho..." autocomplete="off">
      <button id="send" type="submit" disabled>Send</button>
    </form>
    <p class="hint">Pehli baar model download hone mein samay lag sakta hai.</p>
  </main>

  <script type="module">
    const MODEL_ID = 'onnx-community/SmolLM2-135M-Instruct-ONNX-MHA';

    const statusEl = document.querySelector('#status');
    const chat = document.querySelector('#chat');
    const form = document.querySelector('#form');
    const input = document.querySelector('#question');
    const send = document.querySelector('#send');
    const retry = document.querySelector('#retry');

    let pipeline;
    let generator;
    let busy = false;

    function status(text) {
      statusEl.textContent = text;
    }

    function addMessage(text, who) {
      const el = document.createElement('div');
      el.className = 'msg ' + who;
      el.textContent = text;
      chat.appendChild(el);
      chat.scrollTop = chat.scrollHeight;
      return el;
    }

    async function loadModel() {
      send.disabled = true;
      retry.hidden = true;
      status('AI library load ho rahi hai...');

      try {
        if (!pipeline) {
          const lib = await import(
            'https://cdn.jsdelivr.net/npm/@huggingface/transformers@3.7.2'
          );
          pipeline = lib.pipeline;
          lib.env.useBrowserCache = true;
          lib.env.allowLocalModels = false;
        }

        status('AI model download/load ho raha hai. Internet chalu rakho...');

        try {
          generator = await pipeline(
            'text-generation',
            MODEL_ID,
            {
              device: 'webgpu',
              dtype: 'q4f16',
              progress_callback: (p) => {
                if (p.status === 'progress' &&
                    Number.isFinite(p.progress)) {
                  status('Model download: ' +
                    Math.round(p.progress) + '%');
                } else if (p.status === 'download') {
                  status('Model ki file download ho rahi hai...');
                } else if (p.status === 'ready') {
                  status('Model ready ho raha hai...');
                }
              }
            }
          );
        } catch (gpuError) {
          console.warn('WebGPU unavailable; trying CPU/WASM', gpuError);
          generator = await pipeline(
            'text-generation',
            MODEL_ID,
            {
              device: 'wasm',
              dtype: 'q4',
              progress_callback: (p) => {
                if (p.status === 'progress' &&
                    Number.isFinite(p.progress)) {
                  status('Model download: ' +
                    Math.round(p.progress) + '%');
                } else {
                  status('CPU mode mein model load ho raha hai...');
                }
              }
            }
          );
        }

        status('✅ AI model ready! Apna message bhejo.');
        send.disabled = false;
      } catch (err) {
        console.error('Dev AI model error:', err);
        status('❌ Model load nahi hua: ' +
          (err?.message || String(err)).slice(0, 180));
        retry.hidden = false;
      }
    }

    retry.addEventListener('click', loadModel);

    form.addEventListener('submit', async (event) => {
      event.preventDefault();
      const prompt = input.value.trim();
      if (!prompt || !generator || busy) return;

      input.value = '';
      addMessage(prompt, 'user');
      const answerEl = addMessage('Soch raha hoon...', 'bot');
      busy = true;
      send.disabled = true;

      try {
        const result = await generator(
          [{ role: 'user', content: prompt }],
          {
            max_new_tokens: 120,
            do_sample: true,
            temperature: 0.7,
            repetition_penalty: 1.1
          }
        );

        const output = result?.[0]?.generated_text;
        let answer = '';

        if (Array.isArray(output)) {
          answer = output[output.length - 1]?.content || '';
        } else if (typeof output === 'string') {
          answer = output;
        }

        answerEl.textContent = answer.trim() ||
          'Jawab generate nahi hua. Dobara try karo.';
      } catch (err) {
        console.error(err);
        answerEl.textContent =
          'Jawab nahi bana. App refresh karke dobara try karo.';
      } finally {
        busy = false;
        send.disabled = false;
        chat.scrollTop = chat.scrollHeight;
      }
    });

    loadModel();
  </script>
</body>
</html>
