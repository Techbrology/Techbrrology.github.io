<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <title>Puter.js AI Chat</title>
</head>
<body>
  <h2>Puter AI Chat</h2>

  <input id="prompt" placeholder="Type a message..." style="width:300px;">
  <button onclick="send()">Send</button>

  <pre id="output"></pre>

  <script src="https://js.puter.com/v2/"></script>
  <script>
    async function send() {
      const prompt = document.getElementById("prompt").value;
      const output = document.getElementById("output");

      output.textContent = "Thinking...";

      const response = await puter.ai.chat(prompt);

      output.textContent = response.message;
    }
  </script>
</body>
</html>

