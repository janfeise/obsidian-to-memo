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
function createMessageItem(msg, avatarMap) {

  let chatItem = document.createElement("div")

  chatItem.className =
    "chat-item" + (msg.sender === "me" ? " chat-item-me" : "")

  // ===== 写入定位信息 =====
  chatItem.dataset.id = msg.id
  chatItem.dataset.filePath = msg._filePath
  chatItem.dataset.msgIndex = msg._index
  chatItem.dataset.msgTime = msg.time.toISOString()

  // avatar
  let avatarImg = document.createElement("img")
  avatarImg.className = "chat-avatar"
  avatarImg.src = avatarMap[msg.sender] || avatarMap["me"]

  // content
  let contentWrapper = document.createElement("div")
  contentWrapper.className = "chat-content"

  let messageBody = document.createElement("div")
  messageBody.className = "chat-message"
  messageBody.innerText = msg.body

  // time
  let timeLabel = document.createElement("div")
  timeLabel.className = "chat-time"
  timeLabel.innerText = new Date(msg.time).toLocaleString()

  contentWrapper.appendChild(messageBody)
  contentWrapper.appendChild(timeLabel)

  chatItem.appendChild(avatarImg)
  chatItem.appendChild(contentWrapper)

  return chatItem
}

// ==================== 文件操作 ====================

// ===== 通过 id 在文件中定位消息的实际索引 =====
async function findMessageIndexById(filePath, messageId) {
  const file = app.vault.getAbstractFileByPath(filePath)
  if (!file) return -1
  const content = await app.vault.read(file)
  const messages = parseMessagesFromContent(content, filePath)
  return messages.findIndex(m => m.id === messageId)
}

// ===== 更新消息内容，用于编辑后更新 =====
async function updateMessageInFile(filePath, messageId, newBody) {
  const file = app.vault.getAbstractFileByPath(filePath)
  if (!file) return false

  const content = await app.vault.read(file)
  const messages = parseMessagesFromContent(content, filePath)
  const msgIndex = messages.findIndex(m => m.id === messageId)
  if (msgIndex === -1) return false

  const sections = content.split("---")
  const bodyIdx = 2 * msgIndex + 2
  if (bodyIdx >= sections.length) return false

  sections[bodyIdx] = "\n\n" + newBody + "\n"

  const yamlIdx = 2 * msgIndex + 1
  const now = new Date()
  sections[yamlIdx] = sections[yamlIdx].replace(
    /^time:\s*.+$/m,
    "time: " + now
  )

  const newContent = sections.join("---")
  await app.vault.modify(file, newContent)

  const updatedMessages = parseMessagesFromContent(newContent, filePath)
  window.messageCache.set(filePath, {
    mtime: file.stat.mtime,
    messages: updatedMessages
  })

  return { updatedMessages, newTime: now }
}

// ===== 删除消息 =====
async function deleteMessageFromFile(filePath, messageId) {
  const file = app.vault.getAbstractFileByPath(filePath)
  if (!file) return false

  const content = await app.vault.read(file)
  const messages = parseMessagesFromContent(content, filePath)
  const msgIndex = messages.findIndex(m => m.id === messageId)
  if (msgIndex === -1) return false

  const sections = content.split("---")
  const yamlIdx = 2 * msgIndex + 1
  const bodyIdx = 2 * msgIndex + 2

  sections.splice(bodyIdx, 1)
  sections.splice(yamlIdx, 1)

  const hasMessages = sections.some(
    (s, i) => i > 0 && i % 2 === 1 && s.trim()
  )

  if (!hasMessages) {
    await app.vault.delete(file)
    window.messageCache.delete(filePath)
    return { deleted: true, fileDeleted: true }
  }

  const newContent = sections.join("---")
  await app.vault.modify(file, newContent)

  const updatedMessages = parseMessagesFromContent(newContent, filePath)
  window.messageCache.set(filePath, {
    mtime: file.stat.mtime,
    messages: updatedMessages
  })

  return { deleted: true, fileDeleted: false }
}

// ==================== 编辑面板（Bottom Sheet） ====================

function openEditPanel(chatItem) {
  const filePath = chatItem.dataset.filePath
  const messageId = chatItem.dataset.id
  const currentBody = chatItem.querySelector(".chat-message").innerText

  let overlay = document.createElement("div")
  overlay.className = "chat-edit-overlay"

  overlay.addEventListener("click", (e) => {
    if (e.target === overlay) {
      document.body.removeChild(overlay)
    }
  })

  let panel = document.createElement("div")
  panel.className = "chat-edit-panel"
  panel.addEventListener("click", (e) => e.stopPropagation())

    // ===== dragHandle：仅视觉标识，无拖动功能 =====
  let dragHandle = document.createElement("div")
  dragHandle.className = "chat-edit-handle"
  panel.appendChild(dragHandle)

  // ===== 键盘适配：统一在 panel 上处理 =====
  let keyboardOffset = 0

  if (window.visualViewport) {
    const onViewportChange = () => {
      keyboardOffset = window.innerHeight - window.visualViewport.height
      panel.style.transform = keyboardOffset > 0
        ? `translateY(-${keyboardOffset}px)`
        : "translateY(0)"
    }
    window.visualViewport.addEventListener("resize", onViewportChange)
    // 关闭时清理
    const cleanup = () => {
      window.visualViewport.removeEventListener("resize", onViewportChange)
    }
    overlay.addEventListener("click", (e) => {
      if (e.target === overlay) cleanup()
    }, { once: true })
  }

  let timeDisplay = document.createElement("div")
  timeDisplay.className = "chat-edit-time"
  timeDisplay.innerText = new Date(chatItem.dataset.msgTime).toLocaleString()
  panel.appendChild(timeDisplay)

  let textarea = document.createElement("textarea")
  textarea.className = "chat-edit-textarea"
  textarea.value = currentBody
  panel.appendChild(textarea)

  let btnGroup = document.createElement("div")
  btnGroup.className = "chat-edit-buttons"

  let deleteBtn = document.createElement("button")
  deleteBtn.className = "chat-edit-btn-delete"
  deleteBtn.innerText = "删除"
  deleteBtn.addEventListener("click", () => {
    document.body.removeChild(overlay)
    showDeleteConfirm(chatItem, filePath, messageId)
  })
  btnGroup.appendChild(deleteBtn)

  let confirmBtn = document.createElement("button")
  confirmBtn.className = "chat-edit-btn-confirm"
  confirmBtn.innerText = "确定"
  confirmBtn.addEventListener("click", async () => {
    const newBody = textarea.value.trim()

    if (newBody === currentBody) {
      document.body.removeChild(overlay)
      return
    }

    if (!newBody) {
      document.body.removeChild(overlay)
      showDeleteConfirm(chatItem, filePath, messageId)
      return
    }

    confirmBtn.disabled = true
    confirmBtn.innerText = "保存中..."
    const result = await updateMessageInFile(filePath, messageId, newBody)
    confirmBtn.disabled = false

    if (result) {
      chatItem.querySelector(".chat-message").innerText = newBody
      chatItem.querySelector(".chat-time").innerText =
        new Date(result.newTime).toLocaleString()
      chatItem.dataset.msgTime = result.newTime.toISOString()

      document.body.removeChild(overlay)
      new Notice("已更新")
    } else {
      confirmBtn.innerText = "确定"
      new Notice("更新失败，请重试")
    }
  })
  btnGroup.appendChild(confirmBtn)

  panel.appendChild(btnGroup)
  overlay.appendChild(panel)
  document.body.appendChild(overlay)

  textarea.focus()
  textarea.setSelectionRange(textarea.value.length, textarea.value.length)
}

// ==================== 删除确认（Action Sheet） ====================

function showDeleteConfirm(chatItem, filePath, messageId) {
  let overlay = document.createElement("div")
  overlay.className = "chat-delete-overlay"

  overlay.addEventListener("click", (e) => {
    if (e.target === overlay) {
      document.body.removeChild(overlay)
    }
  })

  let sheet = document.createElement("div")
  sheet.className = "chat-delete-sheet"
  sheet.addEventListener("click", (e) => e.stopPropagation())

  let title = document.createElement("div")
  title.className = "chat-delete-title"
  title.innerText = "确定删除这条消息？"
  sheet.appendChild(title)

  let hint = document.createElement("div")
  hint.className = "chat-delete-hint"
  hint.innerText = "删除后不可恢复"
  sheet.appendChild(hint)

  let confirmBtn = document.createElement("button")
  confirmBtn.className = "chat-delete-btn-confirm"
  confirmBtn.innerText = "删除"
  confirmBtn.addEventListener("click", async () => {
    confirmBtn.disabled = true
    const result = await deleteMessageFromFile(filePath, messageId)
    document.body.removeChild(overlay)

    if (result?.deleted) {
      chatItem.style.transition = "all 0.25s ease-out"
      chatItem.style.opacity = "0"
      chatItem.style.transform = "translateX(30px)"
      chatItem.style.maxHeight = chatItem.offsetHeight + "px"
      requestAnimationFrame(() => {
        chatItem.style.maxHeight = "0"
        chatItem.style.margin = "0"
        chatItem.style.padding = "0"
      })
      setTimeout(() => chatItem.remove(), 300)
      new Notice("已删除")
    } else {
      new Notice("删除失败，请重试")
    }
  })
  sheet.appendChild(confirmBtn)

  let cancelBtn = document.createElement("button")
  cancelBtn.className = "chat-delete-btn-cancel"
  cancelBtn.innerText = "取消"
  cancelBtn.addEventListener("click", () => {
    document.body.removeChild(overlay)
  })
  sheet.appendChild(cancelBtn)

  overlay.appendChild(sheet)
  document.body.appendChild(overlay)
}

// ===== 生成唯一 id =====
function generateMessageId() {
  return "msg_" + Date.now() + "_" + Math.random().toString(36).slice(2, 8)
}

// ==================== 事件委托（核心：只有一个监听器） ====================

chatContainer.addEventListener("click", (e) => {
  const chatItem = e.target.closest(".chat-item-me")
  if (!chatItem) return
  if (e.target.closest(".chat-edit-panel") || e.target.closest(".chat-delete-sheet")) return
  openEditPanel(chatItem)
})

// ---- 长按 → 快捷删除 ----
let longPressTimer = null
let longPressTarget = null

chatContainer.addEventListener("touchstart", (e) => {
  const chatItem = e.target.closest(".chat-item-me")
  if (!chatItem) return
  longPressTarget = chatItem
  longPressTimer = setTimeout(() => {
    if (longPressTarget) {
      const filePath = longPressTarget.dataset.filePath
      const messageId = longPressTarget.dataset.id
      showDeleteConfirm(longPressTarget, filePath, messageId)
      longPressTarget = null
    }
  }, 600)
}, { passive: true })

chatContainer.addEventListener("touchend", () => {
  clearTimeout(longPressTimer)
  longPressTarget = null
})

chatContainer.addEventListener("touchmove", () => {
  clearTimeout(longPressTimer)
  longPressTarget = null
})

// ===== 解析单个文件中的所有消息: 由于按天存储，一个文件可能存在多个数据 =====
function parseMessagesFromContent(fileContent, filePath) {
  let messageList = [];
  let sections = fileContent.split("---");

  // 第一个元素通常为空（如果文件以 --- 开头）或 frontmatter 之前的内容
  // 从索引 1 开始，每两个元素为一组（yaml + body）
  let msgIndex = 0
  const sectionsLen = sections.length;
  for (let i = 1; i < sectionsLen; i += 2) {
    let yamlBlock = sections[i].trim();
    let bodyText = (sections[i + 1] || "").trim();
    if (!yamlBlock) continue;

    let idMatch = yamlBlock.match(/^id:\s*(.+)$/m)
    let timeMatch = yamlBlock.match(/^time:\s*(.+)$/m);
    let senderMatch = yamlBlock.match(/^sender:\s*(.+)$/m);

    if (timeMatch && senderMatch) {
      let messageId = idMatch ? idMatch[1].trim() : generateMessageId()

      messageList.push({
        id: messageId,
        time: new Date(timeMatch[1]),
        sender: senderMatch[1].trim(),
        body: bodyText,

        // ===== 定位信息 ====
        _filePath: filePath,
        _index: msgIndex
      });

      msgIndex++
    }
  }

  console.log(messageList);
  return messageList;
}

// ===== 输入弹窗 =====
function showInput() {
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
    let messageId = generateMessageId();

    let messageBlock = `---
id: ${messageId}
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
        const updatedMessages = parseMessagesFromContent(updatedContent, dailyFilePath);
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
    // 从缓存中取最后一条消息的 _index（比 length-1 更可靠）
    const lastMsg = updatedMessages[updatedMessages.length - 1]
    let newChatItem = createMessageItem({
      id: messageId,
      time: now,
      sender: "me",
      body: messageInput.value,
      _filePath: dailyFilePath,
      _index: lastMsg ? lastMsg._index : 0
    }, avatarMap)
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
    let messages = parseMessagesFromContent(fileContent, file.path); // 返回解析后的消息数组
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

  let chatItem = createMessageItem(msg, avatarMap)

  domFragment.appendChild(chatItem);
}
chatContainer.appendChild(domFragment)

dv.container.appendChild(chatContainer)
```
