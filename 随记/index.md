---
banner: "![[bg.png]]"
banner_y: 0.4
banner_x: 0.5
---

```dataviewjs
let pages = app.vault.getFiles()
  .filter(f => f.path.startsWith("随记/messages/") && f.extension === "md")
  .sort((a, b) => b.name.localeCompare(a.name)); // 按日期降序

// ===== avatar 配置 =====
let avatarMap = {
  me: app.vault.adapter.getResourcePath("随记/settings/avatar.png")
}

// ===== 清空容器 =====
dv.container.innerHTML = ""

// ===== 根容器 =====
let container = document.createElement("div")
container.className = "chat-container"

// ===== 封装：创建消息 DOM 元素 =====
function createMessageItem(sender, body, time, avatarMap) {
  let item = document.createElement("div")
  item.className = "chat-item" + (sender === "me" ? " chat-item-me" : "")

  // avatar
  let avatar = document.createElement("img")
  avatar.className = "chat-avatar"
  avatar.src = avatarMap[sender] || avatarMap["me"]

  // content
  let content = document.createElement("div")
  content.className = "chat-content"

  let message = document.createElement("div")
  message.className = "chat-message"
  message.innerText = body

  // time
  let timeElement = document.createElement("div")
  timeElement.className = "chat-time"
  timeElement.innerText = new Date(time).toLocaleString()

  content.appendChild(message)
  content.appendChild(timeElement)

  item.appendChild(avatar)
  item.appendChild(content)

  return item
}

// ===== 解析单个文件中的所有消息: 由于按天存储，一个文件可能存在多个数据 =====
function parseMessagesFromContent(text) {
  let messages = [];
  let parts = text.split("---");

  // 第一个元素通常为空（如果文件以 --- 开头）或 frontmatter 之前的内容
  // 从索引 1 开始，每两个元素为一组（yaml + body）
  for (let i = 1; i < parts.length; i += 2) {
    let yaml = parts[i].trim();
    let body = (parts[i + 1] || "").trim();
    if (!yaml) continue;

    let timeMatch = yaml.match(/^time:\s*(.+)$/m);
    let senderMatch = yaml.match(/^sender:\s*(.+)$/m);

    if (timeMatch && senderMatch) {
      messages.push({
        time: new Date(timeMatch[1]),
        sender: senderMatch[1].trim(),
        body: body
      });
    }
  }

  return messages;
}

// ===== 输入弹窗 =====
function showInput(callback) {
  let overlay = document.createElement("div")
  overlay.className = "chat-modal-overlay"

  // 点击遮罩层关闭
  overlay.onclick = (e) => {
    if (e.target === overlay) {
      document.body.removeChild(overlay)
    }
  }

  let box = document.createElement("div")
  box.className = "chat-modal-box"
  // 阻止点击box时冒泡到overlay
  box.onclick = (e) => e.stopPropagation()

  let input = document.createElement("input")
  input.placeholder = "输入 message..."
  input.className = "chat-modal-input"

  // ===== 创建按钮容器 =====
  let btnContainer = document.createElement("div")
  btnContainer.className = "chat-modal-buttons"

  // ===== 取消按钮 =====
  let cancelBtn = document.createElement("button")
  cancelBtn.innerText = "取消"
  cancelBtn.className = "chat-modal-btn chat-modal-btn-cancel"
  cancelBtn.onclick = () => {
    document.body.removeChild(overlay)
  }

  // ===== 确定按钮 =====
  let confirmBtn = document.createElement("button")
  confirmBtn.innerText = "确定"
  confirmBtn.className = "chat-modal-btn chat-modal-btn-confirm"
  confirmBtn.onclick = async () => {
    if (!input.value.trim()) return

    let now = new Date()
    // 按天存储
    let dateStr = now.getFullYear() +
      '-' + String(now.getMonth() + 1).padStart(2, '0') +
      '-' + String(now.getDate()).padStart(2, '0');
    let filePath = `随记/messages/${dateStr}.md`

    let content = `---
time: ${now}
sender: me
---

${input.value}
`
    let file = app.vault.getAbstractFileByPath(filePath);
    if (file) {
      await app.vault.append(file, "\n\n" + content.trim() + "\n"); // 文件存在，追加
    } else {
      await app.vault.create(filePath, content.trim() + "\n"); // 不存在，创建
    }

    document.body.removeChild(overlay)

    // message文件创建成功后，添加到dom元素后
    let newItem = createMessageItem("me", input.value, now, avatarMap)
    container.prepend(newItem)

    new Notice("Message 已发送")
  }

  // ===== 支持回车键发送 =====
  input.addEventListener("keydown", (e) => {
    if (e.key === "Enter") {
      confirmBtn.click()
    } else if (e.key === "Escape") {
      cancelBtn.click()
    }
  })

  // ===== 组装弹窗 =====
  box.appendChild(input)
  btnContainer.appendChild(cancelBtn)
  btnContainer.appendChild(confirmBtn)
  box.appendChild(btnContainer)
  overlay.appendChild(box)
  document.body.appendChild(overlay)

  // 自动聚焦输入框
  input.focus()
}

// ===== 按钮 =====
let btn = document.createElement("button")
btn.innerText = ""
btn.className = "btn-add-message"
btn.style.marginBottom = "10px"
btn.onclick = () => showInput()

dv.container.appendChild(btn)

// ===== 并行读取数据 =====
const allMessagesArrays = await Promise.all(
  pages.map(async file => {
    if (!file || file.extension !== "md") return [];

    let text = await app.vault.read(file);

    return parseMessagesFromContent(text); // 返回消息数组
  })
)

// ===== 拍平数组并排序 =====
let messages = allMessagesArrays.flat().sort((a, b) => b.time - a.time);

// ===== 渲染消息 =====
// 使用 documentFragment 避免频繁的dom插入
const fragment = new DocumentFragment();
for (let msg of messages) {
  if (!msg) continue;

  let item = createMessageItem(msg.p.sender, msg.body, msg.p.time, avatarMap)

  fragment.appendChild(item);
}
container.appendChild(fragment)

dv.container.appendChild(container)
```
