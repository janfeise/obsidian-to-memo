---
banner: "![[bg.png]]"
banner_y: 0.4
banner_x: 0.5
---

```dataviewjs
let pages = dv.pages('"随记/messages"')
  .sort(p => p.time, 'desc')  // 'asc' 或 'desc'

// ===== avatar 配置 =====
let avatarMap = {
  me: app.vault.adapter.getResourcePath("随记/settings/avatar.png")
}

// ===== 清空容器 =====
dv.container.innerHTML = ""

// ===== 根容器 =====
let container = document.createElement("div")
container.className = "chat-container"

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
    let dateStr = now.getFullYear() +
      '-' + String(now.getMonth() + 1).padStart(2, '0') +
      '-' + String(now.getDate()).padStart(2, '0') +
      '-' + String(now.getHours()).padStart(2, '0') +
      String(now.getMinutes()).padStart(2, '0') +
      String(now.getSeconds()).padStart(2, '0')
    let filePath = `随记/messages/${dateStr}.md`

    let content = `---
time: ${now}
sender: me
---

${input.value}
`

    await app.vault.create(filePath, content)

    document.body.removeChild(overlay)
    new Notice("Message 已发送")

    // 修复：使用正确的方式刷新 DataviewJS
    app.workspace.activeLeaf.rebuildView()
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

// ===== 渲染消息 =====
for (let p of pages) {

  let item = document.createElement("div")
  item.className = "chat-item" + (p.sender === "me" ? " chat-item-me" : "")

  // avatar
  let avatar = document.createElement("img")
  avatar.className = "chat-avatar"
  avatar.src = avatarMap[p.sender] || avatarMap["me"]

  // content
  let content = document.createElement("div")
  content.className = "chat-content"

  let message = document.createElement("div")
  message.className = "chat-message"

  // 直接读取 body（去掉 YAML）
  let file = app.vault.getAbstractFileByPath(p.file.path)
  if (file && file.extension === "md") {
    let text = await app.vault.read(file)
    let parts = text.split("---")
    if (parts.length >= 3) {
      message.innerText = parts.slice(2).join("---").trim()
    } else {
      message.innerText = text.trim()
    }
  }

  // time
  let time = document.createElement("div")
  time.className = "chat-time"
  time.innerText = new Date(p.time).toLocaleString()

  content.appendChild(message)
  content.appendChild(time)

  item.appendChild(avatar)
  item.appendChild(content)

  container.appendChild(item)
}

dv.container.appendChild(container)
```
