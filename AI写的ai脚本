-- ============================================================
-- AI 助手 · 最终完整版（整合 Grok 修复 + 补充扫描范围）
-- 环境：Delta (安卓) + Rayfield + DeepSeek
-- 修复：深度思考稳定 / 手机端拦截 / 扫描性能 / 错误捕获 / 闲聊乱执行
-- 补充：playergui + replicatedfirst 扫描范围 + UI 文字扫描
-- ============================================================

-- ========== 0. 清理旧 UI ==========
pcall(function()
    local old = game:GetService("CoreGui"):FindFirstChild("Rayfield")
    if old then old:Destroy() end
end)

-- ========== 1. 配置 ==========
local API_KEY = "sk-你的新Key"   -- <--- 必须改成你自己的
local API_URL = "https://api.deepseek.com/chat/completions"
local MAX_TOOL_OUTPUT = 600
local MAX_SCAN_OUTPUT = 1500
local MAX_HISTORY_PAIRS = 6
local SCAN_DEPTH_LIMIT = 5
local SCAN_HARD_LIMIT = 150

-- ========== 手机端专用系统提示词 ==========
local SYSTEM_PROMPT = [[你是 Roblox Lua 专家，运行在 Delta 注入器（安卓手机端）环境中。

【重要环境提醒】
当前是安卓手机端，不是电脑端！

严格禁止使用以下电脑端写法：
- MouseButton1Click / MouseButton1Down / MouseButton2Click
- UserInputService 的键盘/鼠标相关（KeyCode、IsKeyDown、GetMouseLocation、MouseBehavior 等）
- ContextActionService 绑定键盘
- Mouse.Hit / Mouse.Target / Mouse.UnitRay
- 任何模拟鼠标点击或键盘输入的代码

必须使用手机端正确写法：
- 使用工具：Tool:Activate()
- 装备工具：tool.Parent = Character
- 移动：Humanoid:MoveTo(Vector3)
- 跳跃：Humanoid.Jump = true
- 改速度：Humanoid.WalkSpeed = 数值
- 传送：HumanoidRootPart.CFrame = CFrame.new(...)
- 模拟触碰：firetouchinterest(hrp, part, 0) 然后 firetouchinterest(hrp, part, 1)

【可用工具】
1. scan_workspace - 扫描游戏对象树（写代码前先扫描具体对象）
2. execute_lua - 执行 Lua 代码

【规则】
1. 涉及具体游戏对象（checkpoint、ammo、door、枪、工具等）时，优先调用 scan_workspace。
2. 通用操作（改速度、回血、飞行等）可直接 execute_lua。
3. 纯聊天直接回复文字，不要调用工具。
4. 代码必须简洁可执行，禁止 markdown 包裹，禁止写 return 语句。
5. 需求不明确时先问清楚。
6. 始终记住这是手机端环境。]]

-- ========== 2. 检查 Delta API ==========
local httpFunc = nil
if type(request) == "function" then
    httpFunc = request
elseif type(http_request) == "function" then
    httpFunc = http_request
elseif type(syn) == "table" and type(syn.request) == "function" then
    httpFunc = syn.request
end

if not httpFunc then
    warn("致命错误：当前 Delta 不支持 request / http_request")
    return
end

-- ========== 3. 加载 Rayfield ==========
local Rayfield = loadstring(game:HttpGet('https://sirius.menu/rayfield'))()

local Window = Rayfield:CreateWindow({
    Name = "AI 万能助手 · 最终完整版",
    LoadingTitle = "正在加载最终版...",
    LoadingSubtitle = "手机端专用 · 已修复全部已知问题",
    ConfigurationSaving = { Enabled = false },
    KeySystem = false
})

local Tab = Window:CreateTab("AI 控制台", 4483362458)

-- ========== 4. 服务 ==========
local HttpService = game:GetService("HttpService")
local Players = game:GetService("Players")
local LocalPlayer = Players.LocalPlayer

-- ============================================================
-- 5. 电脑端关键词拦截
-- ============================================================
local PC_KEYWORDS = {
    "MouseButton1Click",
    "MouseButton1Down",
    "MouseButton2Click",
    "MouseButton2Down",
    "MouseButton1Up",
    "MouseButton2Up",
    "UserInputService.InputBegan",
    "UserInputService.InputEnded",
    "UserInputService.InputChanged",
    "KeyCode.",
    "Enum.KeyCode",
    "IsKeyDown",
    "ContextActionService",
    "UserInputService.MouseBehavior",
    "GetMouseLocation",
    "Mouse.Hit",
    "Mouse.Target",
    "Mouse.UnitRay",
    "Mouse.Icon",
    "VirtualInputManager",
    "SendMouse",
    "SendKey"
}

local function containsPCKeyword(code)
    for _, kw in ipairs(PC_KEYWORDS) do
        if code:find(kw, 1, true) then
            return kw
        end
    end
    return nil
end

-- ============================================================
-- 6. execute_lua（正确错误捕获 + 手机端拦截）
-- ============================================================
local function executeLua(code)
    if type(loadstring) ~= "function" then
        return "失败：当前环境不支持 loadstring"
    end

    local pcKw = containsPCKeyword(code)
    if pcKw then
        return "拒绝执行：检测到电脑端写法（" .. pcKw .. "）。\n当前是安卓手机端，请使用 Tool:Activate() / Humanoid:MoveTo / 直接修改属性 / firetouchinterest，不要模拟鼠标键盘。"
    end

    local outputs = {}
    local realPrint = print
    local realWarn = warn

    local function capture(...)
        local parts = {}
        for i = 1, select("#", ...) do
            table.insert(parts, tostring(select(i, ...)))
        end
        table.insert(outputs, table.concat(parts, " "))
    end

    pcall(function() print = capture end)
    pcall(function() warn = capture end)

    local compileOk, fn = pcall(loadstring, code)
    if not compileOk or type(fn) ~= "function" then
        pcall(function() print = realPrint end)
        pcall(function() warn = realWarn end)
        return "编译失败：" .. tostring(fn)
    end

    local results = table.pack(pcall(fn))
    local execOk = results[1]

    pcall(function() print = realPrint end)
    pcall(function() warn = realWarn end)

    local outputText = ""
    if #outputs > 0 then
        local joined = table.concat(outputs, "\n")
        if #joined > MAX_TOOL_OUTPUT then
            joined = joined:sub(1, MAX_TOOL_OUTPUT) .. "...(已截断)"
        end
        outputText = "\n输出：\n" .. joined
    end

    if not execOk then
        return "执行失败：" .. tostring(results[2]) .. outputText
    end

    local returnText = ""
    if results.n > 1 then
        local retParts = {}
        for i = 2, results.n do
            table.insert(retParts, tostring(results[i]))
        end
        returnText = "\n返回值：" .. table.concat(retParts, ", ")
    end

    return "执行成功" .. returnText .. outputText
end

-- ============================================================
-- 7. 扫描工具（优化性能 + 更多范围 + UI 文字）
-- ============================================================
local function scanWorkspace(args)
    local keyword = (args.keyword or ""):lower()
    local maxResults = math.min(tonumber(args.maxResults) or 30, SCAN_HARD_LIMIT)
    local searchIn = args.searchIn or "workspace"
    local matchClass = args.matchClass == true

    local roots = {}

    if searchIn == "workspace" or searchIn == "all" then
        table.insert(roots, game:GetService("Workspace"))
    end
    if searchIn == "players" or searchIn == "all" then
        table.insert(roots, game:GetService("Players"))
    end
    if searchIn == "replicated" or searchIn == "all" then
        table.insert(roots, game:GetService("ReplicatedStorage"))
    end
    if searchIn == "character" or searchIn == "all" then
        local char = LocalPlayer.Character
        if char then table.insert(roots, char) end
    end
    if searchIn == "backpack" or searchIn == "all" then
        local bp = LocalPlayer:FindFirstChild("Backpack")
        if bp then table.insert(roots, bp) end
    end
    -- 新增：PlayerGui（游戏界面上的 UI）
    if searchIn == "playergui" or searchIn == "all" then
        local pg = LocalPlayer:FindFirstChild("PlayerGui")
        if pg then table.insert(roots, pg) end
    end
    -- 新增：ReplicatedFirst
    if searchIn == "replicatedfirst" or searchIn == "all" then
        table.insert(roots, game:GetService("ReplicatedFirst"))
    end

    if #roots == 0 then
        return "没有可搜索的根节点"
    end

    local results = {}
    local count = 0

    local function search(node, depth)
        if count >= maxResults or depth > SCAN_DEPTH_LIMIT then return end

        local children = node:GetChildren()
        for _, child in ipairs(children) do
            if count >= maxResults then return end

            local nameLower = child.Name:lower()
            local classLower = child.ClassName:lower()
            local matched = (keyword == "")
                or nameLower:find(keyword, 1, true)
                or (matchClass and classLower:find(keyword, 1, true))

            if matched then
                count = count + 1
                local valueStr = ""

                if child:IsA("ValueBase") then
                    pcall(function()
                        valueStr = " 值=" .. tostring(child.Value)
                    end)
                end

                -- 新增：UI 文字
                if child:IsA("TextLabel") or child:IsA("TextButton") then
                    pcall(function()
                        if child.Text ~= "" then
                            valueStr = valueStr .. " 文字=\"" .. child.Text:sub(1, 50) .. "\""
                        end
                    end)
                end

                table.insert(results, string.format(
                    "%s [%s] 父级=%s%s",
                    child:GetFullName(),
                    child.ClassName,
                    child.Parent and child.Parent.Name or "?",
                    valueStr
                ))
            end

            search(child, depth + 1)
        end
    end

    for _, root in ipairs(roots) do
        search(root, 0)
        if count >= maxResults then break end
    end

    if #results == 0 then
        return "没有找到匹配 '" .. keyword .. "' 的对象"
    end

    local header = "找到 " .. count .. " 个结果（最多显示 " .. maxResults .. " 个）:\n"
    local joined = table.concat(results, "\n")
    if #joined > MAX_SCAN_OUTPUT then
        joined = joined:sub(1, MAX_SCAN_OUTPUT) .. "\n...(结果过长已截断)"
    end

    return header .. joined
end

-- ============================================================
-- 8. 工具定义
-- ============================================================
local tools = {
    {
        type = "function",
        ["function"] = {
            name = "execute_lua",
            description = "执行任意 Roblox Lua 代码。注意：当前是安卓手机端，禁止使用任何鼠标、键盘相关 API。",
            parameters = {
                type = "object",
                properties = {
                    code = {
                        type = "string",
                        description = "完整可执行的 Lua 代码。不要 markdown，不要 return，不要鼠标键盘 API。"
                    }
                },
                required = { "code" }
            }
        }
    },
    {
        type = "function",
        ["function"] = {
            name = "scan_workspace",
            description = "扫描游戏对象树，查找名字或 ClassName 包含关键词的对象。写代码前强烈建议先扫描。",
            parameters = {
                type = "object",
                properties = {
                    keyword = {
                        type = "string",
                        description = "搜索关键词（如 checkpoint、ammo、door、Tool）。空字符串返回部分对象。"
                    },
                    maxResults = {
                        type = "integer",
                        description = "最多返回数量，默认 30，最大 150"
                    },
                    searchIn = {
                        type = "string",
                        enum = { "workspace", "players", "replicated", "character", "backpack", "playergui", "replicatedfirst", "all" },
                        description = "搜索范围，默认 workspace。all = 全部"
                    },
                    matchClass = {
                        type = "boolean",
                        description = "是否同时匹配 ClassName，默认 false"
                    }
                },
                required = { "keyword" }
            }
        }
    }
}

-- ============================================================
-- 9. 对话逻辑
-- ============================================================
local conversationHistory = {}

local function containsActionWord(text)
    local actionWords = {
        "改", "修改", "设置", "设为", "调", "变成", "变",
        "自动", "通关", "过关",
        "执行", "运行", "跑", "飞",
        "传送", "移动",
        "删", "删除", "移除", "去掉",
        "查", "查找", "找", "搜索", "读取", "获取", "看看", "扫描",
        "写", "创建", "生成", "添加",
        "给", "让", "帮",
        "用", "使用",
        "速度", "血量", "位置", "坐标",
        "子弹", "弹药", "无限",
        "玩家", "NPC", "对象", "物体",
        "工具", "背包", "枪",
        "检查点", "checkpoint"
    }
    for _, word in ipairs(actionWords) do
        if text:find(word, 1, true) then
            return true
        end
    end
    return false
end

local function trimHistory()
    local userIndices = {}
    for i, msg in ipairs(conversationHistory) do
        if msg.role == "user" then
            table.insert(userIndices, i)
        end
    end

    if #userIndices > MAX_HISTORY_PAIRS then
        local cutIndex = userIndices[#userIndices - MAX_HISTORY_PAIRS + 1]
        local newHistory = {}
        for _, msg in ipairs(conversationHistory) do
            if msg.role == "system" then
                table.insert(newHistory, msg)
                break
            end
        end
        for i = cutIndex, #conversationHistory do
            table.insert(newHistory, conversationHistory[i])
        end
        conversationHistory = newHistory
    end
end

local function sendToAI(messages, model, includeTools, toolChoice)
    local bodyTable = {
        model = model,
        messages = messages
    }

    if includeTools then
        bodyTable.tools = tools
        bodyTable.tool_choice = toolChoice or "auto"
    end

    local body = HttpService:JSONEncode(bodyTable)

    local ok, res = pcall(function()
        return httpFunc({
            Url = API_URL,
            Method = "POST",
            Headers = {
                ["Content-Type"] = "application/json",
                ["Authorization"] = "Bearer " .. API_KEY
            },
            Body = body
        })
    end)

    if not ok then return nil, "请求失败: " .. tostring(res) end
    if type(res) ~= "table" or not res.Body then return nil, "返回格式异常" end

    local ok2, data = pcall(function()
        return HttpService:JSONDecode(res.Body)
    end)
    if not ok2 then return nil, "JSON 解析失败" end
    if data.error then return nil, "API 错误: " .. (data.error.message or "未知") end

    return data, nil
end

local function chatWithTools(userMessage, deepThink)
    if #conversationHistory == 0 then
        table.insert(conversationHistory, { role = "system", content = SYSTEM_PROMPT })
    end

    local finalMsg = userMessage
    if deepThink then
        finalMsg = userMessage .. "\n\n请进行更深入、严谨的思考，先分析需求再决定是否调用工具，并写出更可靠的代码。"
    end

    table.insert(conversationHistory, { role = "user", content = finalMsg })
    trimHistory()

    local isAction = containsActionWord(userMessage)
    local model = "deepseek-chat"   -- 统一使用 chat，保证 tools 稳定
    local toolChoice = "auto"

    local maxLoops = 8
    local loopCount = 0
    local forcedOnce = false

    while loopCount < maxLoops do
        loopCount = loopCount + 1

        local data, err = sendToAI(conversationHistory, model, true, toolChoice)
        if not data then return err end

        if not data.choices or not data.choices[1] or not data.choices[1].message then
            return "API 返回结构异常（无 choices）"
        end

        local message = data.choices[1].message

        local assistantMsg = { role = "assistant", content = message.content or "" }
        if message.tool_calls then
            assistantMsg.tool_calls = message.tool_calls
        end

        if not message.tool_calls or #message.tool_calls == 0 then
            table.insert(conversationHistory, assistantMsg)

            if isAction and not forcedOnce and loopCount <= 2 then
                forcedOnce = true
                table.insert(conversationHistory, {
                    role = "user",
                    content = "这是操作类需求，请调用 scan_workspace 或 execute_lua 工具执行。记住当前是安卓手机端，禁止鼠标键盘写法。"
                })
                trimHistory()
            else
                return message.content or ""
            end
        else
            table.insert(conversationHistory, assistantMsg)
            trimHistory()

            for _, toolCall in ipairs(message.tool_calls) do
                local funcName = toolCall["function"].name
                local argsRaw = toolCall["function"].arguments

                local ok, args = pcall(function()
                    return HttpService:JSONDecode(argsRaw)
                end)

                local result
                if not ok then
                    result = "参数解析失败: " .. tostring(args)
                elseif funcName == "execute_lua" then
                    result = executeLua(args.code or "")
                    Rayfield:Notify({
                        Title = "AI 写的代码",
                        Content = tostring(args.code or ""):sub(1, 100),
                        Duration = 6
                    })
                elseif funcName == "scan_workspace" then
                    result = scanWorkspace(args or {})
                    Rayfield:Notify({
                        Title = "AI 扫描游戏",
                        Content = "关键词: " .. tostring((args and args.keyword) or ""),
                        Duration = 4
                    })
                else
                    result = "未知工具: " .. tostring(funcName)
                end

                table.insert(conversationHistory, {
                    role = "tool",
                    tool_call_id = toolCall.id,
                    content = result
                })
                trimHistory()
            end
        end
    end

    return "达到最大工具调用次数，可能陷入循环。"
end

-- ============================================================
-- 10. UI
-- ============================================================
local historyText = "等待输入..."

local historyParagraph = Tab:CreateParagraph({
    Title = "对话历史",
    Content = historyText
})

local function updateHistory(role, text)
    text = tostring(text or "")
    historyText = historyText .. "\n\n【" .. role .. "】\n" .. text
    pcall(function()
        historyParagraph:Set({ Title = "对话历史", Content = historyText })
    end)
end

local userInput = ""

Tab:CreateInput({
    Name = "输入内容",
    PlaceholderText = "例如：帮我自动过所有 checkpoint / 改速度100 / 无限弹药",
    RemoveTextAfterFocusLost = false,
    Callback = function(text)
        userInput = text or ""
    end
})

Tab:CreateButton({
    Name = "🤖 发送（普通模式）",
    Callback = function()
        if userInput == "" then
            Rayfield:Notify({ Title = "提示", Content = "请先输入内容", Duration = 3 })
            return
        end

        updateHistory("你", userInput)
        Rayfield:Notify({ Title = "AI", Content = "思考中...", Duration = 3 })

        local ok, reply = pcall(function()
            return chatWithTools(userInput, false)
        end)

        if not ok then
            Rayfield:Notify({ Title = "异常", Content = tostring(reply), Duration = 10 })
            updateHistory("系统", "异常: " .. tostring(reply))
            return
        end

        updateHistory("AI", reply)
    end
})

Tab:CreateButton({
    Name = "🧠 深度思考（更严谨）",
    Callback = function()
        if userInput == "" then
            Rayfield:Notify({ Title = "提示", Content = "请先输入内容", Duration = 3 })
            return
        end

        updateHistory("你", userInput .. "（深度思考）")
        Rayfield:Notify({ Title = "AI", Content = "深度思考中...", Duration = 4 })

        local ok, reply = pcall(function()
            return chatWithTools(userInput, true)
        end)

        if not ok then
            Rayfield:Notify({ Title = "异常", Content = tostring(reply), Duration = 10 })
            updateHistory("系统", "异常: " .. tostring(reply))
            return
        end

        updateHistory("AI", reply)
    end
})

Tab:CreateButton({
    Name = "清空对话",
    Callback = function()
        conversationHistory = {}
        historyText = "等待输入..."
        pcall(function()
            historyParagraph:Set({ Title = "对话历史", Content = historyText })
        end)
        Rayfield:Notify({ Title = "提示", Content = "对话已清空", Duration = 3 })
    end
})

Tab:CreateButton({
    Name = "关闭并销毁 UI",
    Callback = function()
        pcall(function() Rayfield:Destroy() end)
    end
})

print("✅ AI 万能助手 · 最终完整版 已加载（手机端专用）")
