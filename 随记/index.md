---
banner: "![[bg.png]]"
banner_y: 0.4
banner_x: 0.5
---

```dataviewjs
let dailyFiles = app.vault.getFiles()
  .filter(f => f.path.startsWith("随记/messages/") && f.extension === "md")
  .sort((a, b) => b.name.localeCompare(a.name)); // 按日期降序

// ===== avatar 配置 =====
let avatarMap = {
  me: app.vault.adapter.getResourcePath("随记/settings/avatar.png")
}

// ===== 缓存已解析的消息（全局持久化，避免重复 I/O 和解析） =====
if (!window.messageCache) {
    window.messageCache = new Map()  // 只在首次运行时创建，key: file.path, value: 已解析的消息数组
    console.log("缓存初始化")
}
console.log("当前缓存大小：", window.messageCache.size)

// ===== 清空容器 =====
dv.container.innerHTML = ""

// ===== 根容器 =====
let chatContainer = document.createElement("div")
chatContainer.className = "chat-container"

// ===== 封装：创建消息 DOM 元素 =====
function createMessageItem(sender, body, time, avatarMap) {
  let chatItem = document.createElement("div")
  chatItem.className = "chat-item" + (sender === "me" ? " chat-item-me" : "")

  // avatar
  let avatarImg = document.createElement("img")
  avatarImg.className = "chat-avatar"
  avatarImg.src = avatarMap[sender] || avatarMap["me"]

  // content
  let contentWrapper = document.createElement("div")
  contentWrapper.className = "chat-content"

  let messageBody = document.createElement("div")
  messageBody.className = "chat-message"
  messageBody.innerText = body

  // time
  let timeLabel = document.createElement("div")
  timeLabel.className = "chat-time"
  timeLabel.innerText = new Date(time).toLocaleString()

  contentWrapper.appendChild(messageBody)
  contentWrapper.appendChild(timeLabel)

  chatItem.appendChild(avatarImg)
  chatItem.appendChild(contentWrapper)

  return chatItem
}

// ===== 解析单个文件中的所有消息: 由于按天存储，一个文件可能存在多个数据 =====
function parseMessagesFromContent(fileContent) {
  let messageList = [];
  let sections = fileContent.split("---");

  // 第一个元素通常为空（如果文件以 --- 开头）或 frontmatter 之前的内容
  // 从索引 1 开始，每两个元素为一组（yaml + body）
  const sectionsLen = sections.length;
  for (let i = 1; i < sectionsLen; i += 2) {
    let yamlBlock = sections[i].trim();
    let bodyText = (sections[i + 1] || "").trim();
    if (!yamlBlock) continue;

    let timeMatch = yamlBlock.match(/^time:\s*(.+)$/m);
    let senderMatch = yamlBlock.match(/^sender:\s*(.+)$/m);

    if (timeMatch && senderMatch) {
      messageList.push({
        time: new Date(timeMatch[1]),
        sender: senderMatch[1].trim(),
        body: bodyText
      });
    }
  }

  return messageList;
}

// ===== 输入弹窗 =====
function showInput(callback) {
  let modalOverlay = document.createElement("div")
  modalOverlay.className = "chat-modal-overlay"

  // 点击遮罩层关闭
  modalOverlay.onclick = (e) => {
    if (e.target === modalOverlay) {
      document.body.removeChild(modalOverlay)
    }
  }

  let modalBox = document.createElement("div")
  modalBox.className = "chat-modal-box"
  // 阻止点击box时冒泡到overlay
  modalBox.onclick = (e) => e.stopPropagation()

  let messageInput = document.createElement("input")
  messageInput.placeholder = "输入 message..."
  messageInput.className = "chat-modal-input"

  // ===== 创建按钮容器 =====
  let buttonGroup = document.createElement("div")
  buttonGroup.className = "chat-modal-buttons"

  // ===== 取消按钮 =====
  let cancelButton = document.createElement("button")
  cancelButton.innerText = "取消"
  cancelButton.className = "chat-modal-btn chat-modal-btn-cancel"
  cancelButton.onclick = () => {
    document.body.removeChild(modalOverlay)
  }

  // ===== 确定按钮 =====
  let confirmButton = document.createElement("button")
  confirmButton.innerText = "确定"
  confirmButton.className = "chat-modal-btn chat-modal-btn-confirm"
  confirmButton.onclick = async () => {
    if (!messageInput.value.trim()) return

    let now = new Date()
    // 按天存储
    let todayStr = now.getFullYear() +
      '-' + String(now.getMonth() + 1).padStart(2, '0') +
      '-' + String(now.getDate()).padStart(2, '0');
    let dailyFilePath = `随记/messages/${todayStr}.md`

    let messageBlock = `---
time: ${now}
sender: me
---

${messageInput.value}
`
    let targetFile;
    let existingFile = app.vault.getAbstractFileByPath(dailyFilePath);
    if (existingFile) {
      await app.vault.append(existingFile, "\n\n" + messageBlock.trim() + "\n"); // 文件存在，追加
      targetFile = existingFile
    } else {
      targetFile = await app.vault.create(dailyFilePath, messageBlock.trim() + "\n"); // 不存在，创建
    }

    // 缓存更新
    if (targetFile) {
      try {
        const updatedContent = await app.vault.read(targetFile);
        const updatedMessages = parseMessagesFromContent(updatedContent);
        window.messageCache.set(dailyFilePath, {
          mtime: targetFile.stat.mtime,
          messages: updatedMessages
        });
        console.log("🔄 已更新缓存", dailyFilePath);
      } catch (e) {
        // 读取失败则删除缓存，下次重新从磁盘读
        window.messageCache.delete(dailyFilePath);
        console.warn("缓存更新失败，已删除", dailyFilePath);
      }
    }

    document.body.removeChild(modalOverlay)

    // message文件创建成功后，添加到dom元素后
    let newChatItem = createMessageItem("me", messageInput.value, now, avatarMap)
    chatContainer.prepend(newChatItem)

    new Notice("Message 已发送")
  }

  // ===== 支持回车键发送 =====
  messageInput.addEventListener("keydown", (e) => {
    if (e.key === "Enter") {
      confirmButton.click()
    } else if (e.key === "Escape") {
      cancelButton.click()
    }
  })

  // ===== 组装弹窗 =====
  modalBox.appendChild(messageInput)
  buttonGroup.appendChild(cancelButton)
  buttonGroup.appendChild(confirmButton)
  modalBox.appendChild(buttonGroup)
  modalOverlay.appendChild(modalBox)
  document.body.appendChild(modalOverlay)

  // 自动聚焦输入框
  messageInput.focus()
}

// ===== 按钮 =====
let addMessageBtn = document.createElement("button")
addMessageBtn.innerText = ""
addMessageBtn.className = "btn-add-message"
addMessageBtn.style.marginBottom = "10px"
addMessageBtn.onclick = () => showInput()

dv.container.appendChild(addMessageBtn)

// ===== 并行读取数据（使用缓存） =====
const allMessageBatches = await Promise.all(
  dailyFiles.map(async file => {
    if (!file || file.extension !== "md") return [];

    // 如果缓存命中，直接返回
    let cached = window.messageCache.get(file.path)
    if (cached && cached.mtime === file.stat.mtime) {
      console.log("缓存命中", file.path);
      return cached.messages;
    }
    if (cached) {
      console.log("缓存失效（文件已更改）", file.path)
      window.messageCache.delete(file.path); // 删除失效的缓存数据
    }

    // 首次读取或缓存失效：读文件 -> 解析 -> 存入缓存
    console.log("从磁盘读取", file.path)
    let fileContent = await app.vault.read(file);
    let messages = parseMessagesFromContent(fileContent); // 返回解析后的消息数组
    window.messageCache.set(file.path, {mtime: file.stat.mtime, messages: messages});
    return messages;
  })
)

// ===== 拍平数组并排序 =====
let sortedMessages = allMessageBatches.flat().sort((a, b) => b.time - a.time);

// ===== 渲染消息 =====
// 使用 documentFragment 避免频繁的dom插入
const domFragment = new DocumentFragment();
for (let msg of sortedMessages) {
  if (!msg) continue;

  let chatItem = createMessageItem(msg.sender, msg.body, msg.time, avatarMap)

  domFragment.appendChild(chatItem);
}
chatContainer.appendChild(domFragment)

dv.container.appendChild(chatContainer)
```
