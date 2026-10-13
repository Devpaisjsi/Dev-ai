
<!DOCTYPE html>
<html lang="hi">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <meta name="theme-color" content="#10101a">
  <title>Dev AI</title>
  <style>
    * { box-sizing: border-box; }
    body {
      margin: 0; background: #10101a; color: #f7f7fb;
      font-family: Arial, sans-serif; min-height: 100vh;
    }
    header {
      padding: 18px; text-align: center; font-size: 24px;
      font-weight: bold; color: #ff4141; border-bottom: 1px solid #303040;
    }
    #status {
      text-align: center; color: #c4c4d4; padding: 12px;
      font-size: 13px;
    }
    #chat {
      max-width: 760px; margin: auto; padding: 14px;
      padding-bottom: 125px;
    }
    .msg {
      padding: 12px 14px; border-radius: 16px; margin: 10px 0;
      white-space: pre-wrap; overflow-wrap: anywhere; line-height: 1.5;
    }
    .user { background: #762dff; margin-left: 12%; }
    .ai { background: #242433; margin-right: 8%; }
    #welcome { color: #bcbccc; text-align: center; margin-top: 50px; }
    #controls {
      position: fixed; bottom: 0; left: 0; right: 0;
      padding: 12px; background: #151521; border-top: 1px solid #303040;
    }
    .inputrow {
      max-width: 760px; margin: auto; display: flex; gap: 8px;
    }
    input {
      min-width: 0; flex: 1; padding: 14px; border-radius: 14px;
      border: 1px solid #44445a; background: #222231; color: white;
      font-size: 16px; outline: none;
    }
    button {
      padding: 12px 16px; border: 0; border-radius: 12px;
      background: #ff4141; color: white; font-weight: bold;
    }
    button:disabled { opacity: .5; }
    #retry {
      display: block; margin: 8px auto; background: #343447;
    }
    small { display: block; text-align: center; color: #a7a7b8; margin-top: 8px; }
  </style>
</head>
<body>
  <header>🔥 Dev AI</header>
  <div id="status">AI model load ho raha hai…</div>
  <button id="retry" hidden>Model dobara load karo</button>
  <main id="chat">
    <div id="welcome">Namaste! Main Dev AI hoon.<br>Model load hone ke baad apna sawal poochho.</div>
  </main>
  <div id="controls">
    <div class="inputrow">
      <input id="prompt" placeholder="Yahan message likho…" autocomplete="off">
      <button id="send" disabled>Send</button>
    </div>
    <small>AI ke jawab galat bhi ho sakte hain.</small>
  </div>

  <script type="module">
    const statusEl = document.getElementById("status");
    const chat = document.getElementById("chat");
    const input = document.getElementById("prompt");
    const send = document.getElementById("send");
    const retry = document.getElementById("retry");
    const welcome = document.getElementById("welcome");

    let generator = null;
    let busy = false;

    function addMessage(text, who) {
      welcome.hidden = true;
      const div = document.createElement("div");
      div.className = "msg " + who;
      div.textContent = text;
      chat.appendChild(div);
      window.scrollTo(0, document.body.scrollHeight);
      return div;
    }

    async function loadModel() {
      if (busy || generator) return;
      busy = true;
      retry.hidden = true;
      send.disabled = true;
      statusEl.textContent = "AI library load ho rahi hai…";

      try {
        const { pipeline, env } = await import(
          "https://cdn.jsdelivr.net/npm/@huggingface/transformers@3.7.2"
        );
        env.allowLocalModels = false;

        statusEl.textContent =
          "AI model download ho raha hai. Pehli baar kuch minute lag sakte hain…";

        generator = await pipeline(
          "text-generation",
          "onnx-community/SmolLM2-135M-Instruct-ONNX",
          {
            dtype: "q4",
            progress_callback: (p) => {
              if (p && p.status === "progress" &&
                  typeof p.progress === "number") {
                statusEl.textContent =
                  "Model download: " + Math.round(p.progress) + "%";
              } else if (p && p.status === "downloading") {
                statusEl.textContent = "AI model download ho raha hai…";
              } else if (p && p.status === "ready") {
                statusEl.textContent = "Model ready ho raha hai…";
              }
            }
          }
        );

        statusEl.textContent = "✅ Dev AI ready hai! Message bhejo.";
        send.disabled = false;
      } catch (err) {
        console.error(err);
        generator = null;
        statusEl.textContent =
          "Model load nahi hua. Internet check karke dobara try karo.";
        retry.hidden = false;
      } finally {
        busy = false;
      }
    }

    async function sendMessage() {
      const question = input.value.trim();
      if (!question || !generator || busy) return;

      input.value = "";
      addMessage(question, "user");
      const answerBox = addMessage("Soch raha hoon…", "ai");
      busy = true;
      send.disabled = true;

      try {
        const result = await generator(
          [
            {
              role: "system",
              content: "You are Dev AI, a friendly assistant. Reply simply and helpfully. Reply in Hindi when the user writes in Hindi."
            },
            { role: "user", content: question }
          ],
          { max_new_tokens: 120, do_sample: false }
        );

        const output = result?.[0]?.generated_text;
        let answer = "";

        if (Array.isArray(output)) {
          answer = output[output.length - 1]?.content || "";
        } else if (typeof output === "string") {
          answer = output;
        }

        answerBox.textContent = answer.trim() || "Is baar jawab nahi mila. Dobara poochho.";
      } catch (err) {
        console.error(err);
        answerBox.textContent = "Jawab nahi ban paya. Chhota sawal poochh kar try karo.";
      } finally {
        busy = false;
        send.disabled = !generator;
      }
    }

    send.addEventListener("click", sendMessage);
    input.addEventListener("keydown", (e) => {
      if (e.key === "Enter") sendMessage();
    });
    retry.addEventListener("click", loadModel);

    loadModel();
  </script>
</body>
</html>
