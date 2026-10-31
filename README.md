--// David Hub - Egg Finder (UI verde)
--// Top 5 ovos/pets recentes, Force Join, Min Value, Rarity, Auto Hop, Config

local Players = game:GetService("Players")
local TeleportService = game:GetService("TeleportService")
local HttpService = game:GetService("HttpService")
local UIS = game:GetService("UserInputService")
local LP = Players.LocalPlayer

--================ CONFIG ================--
local Config = {
    MinB = 1,           -- valor mínimo (B/s)
    MaxB = 10,          -- valor máximo (B/s)
    UIScale = 1,
    AutoHop = false,
    Delay = 2,          -- Force Join (2 a 5 s)
    Rarity = "All",
    ScanInterval = 3,
}

local RARITIES = { "All", "Common", "Rare", "Epic", "Legendary", "Mythic", "Divine" }
local RARITY_COLOR = {
    Common = Color3.fromRGB(200, 200, 200),
    Rare = Color3.fromRGB(80, 170, 255),
    Epic = Color3.fromRGB(190, 90, 255),
    Legendary = Color3.fromRGB(255, 190, 40),
    Mythic = Color3.fromRGB(255, 80, 120),
    Divine = Color3.fromRGB(170, 70, 255),
}

-- Ajuste se o jogo guarda pets/ovos em outra pasta
local function getContainers() return { workspace } end

--================ UTIL ================--
local suffix = { K = 1e3, M = 1e6, B = 1e9, T = 1e12 }

local function parseValue(str)
    str = tostring(str):upper():gsub(",", ""):gsub("%$", "")
    local num, suf = str:match("([%d%.]+)%s*([KMBT]?)")
    if not num then return nil end
    return tonumber(num) * (suffix[suf] or 1)
end

local function formatValue(n)
    if n >= 1e12 then return string.format("%.2fT", n / 1e12) end
    if n >= 1e9 then return string.format("%.2fB", n / 1e9) end
    if n >= 1e6 then return string.format("%.2fM", n / 1e6) end
    return tostring(n)
end

local function getValue(model)
    for _, k in ipairs({ "Income", "PerSecond", "Value", "Price", "Worth" }) do
        local a = model:GetAttribute(k)
        if a then return type(a) == "number" and a or parseValue(a) end
        local c = model:FindFirstChild(k)
        if c and c:IsA("ValueBase") then
            return type(c.Value) == "number" and c.Value or parseValue(c.Value)
        end
    end
    for _, d in ipairs(model:GetDescendants()) do
        if d:IsA("TextLabel") and d.Text:match("%d+%.?%d*%s*[KMBT]") then
            local v = parseValue(d.Text)
            if v then return v end
        end
    end
end

local function getRarity(model)
    local r = model:GetAttribute("Rarity")
    if r then return tostring(r) end
    local c = model:FindFirstChild("Rarity")
    if c and c:IsA("ValueBase") then return tostring(c.Value) end
end

local function getMult(model)
    local m = model:GetAttribute("Multiplier") or model:GetAttribute("Mult")
    return tonumber(m) or 1
end

local function isPet(model)
    if not model:IsA("Model") then return false end
    if Players:GetPlayerFromCharacter(model) then return false end
    if model:GetAttribute("IsPet") or model:GetAttribute("Pet") then return true end
    return model.Name:lower():find("pet") ~= nil or getValue(model) ~= nil
end

--================ UI ================--
local GREEN = Color3.fromRGB(46, 204, 113)
local GREEN_D = Color3.fromRGB(20, 120, 60)
local DARK = Color3.fromRGB(16, 40, 26)
local ROW = Color3.fromRGB(34, 58, 44)

local gui = Instance.new("ScreenGui")
gui.Name = "DavidHub"
gui.ResetOnSpawn = false
pcall(function() gui.Parent = (gethui and gethui()) or game:GetService("CoreGui") end)
if not gui.Parent then gui.Parent = LP:WaitForChild("PlayerGui") end

local uiScale = Instance.new("UIScale", gui)
uiScale.Scale = Config.UIScale

local function corner(o, r) Instance.new("UICorner", o).CornerRadius = UDim.new(0, r or 8) end
local function stroke(o, c, t)
    local s = Instance.new("UIStroke", o); s.Color = c; s.Thickness = t or 2
    s.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
end
local function label(parent, text, size, color, font)
    local l = Instance.new("TextLabel", parent)
    l.BackgroundTransparency = 1; l.Text = text; l.TextSize = size
    l.TextColor3 = color or Color3.new(1, 1, 1)
    l.Font = font or Enum.Font.GothamBold
    return l
end

local main = Instance.new("Frame", gui)
main.Size = UDim2.fromOffset(380, 440)
main.Position = UDim2.fromOffset(30, 70)
main.BackgroundColor3 = DARK
main.BorderSizePixel = 0
corner(main, 10); stroke(main, GREEN_D, 3)

-- Header
local header = Instance.new("Frame", main)
header.Size = UDim2.new(1, 0, 0, 46)
header.BackgroundColor3 = GREEN; header.BorderSizePixel = 0
corner(header, 10)
local grad = Instance.new("UIGradient", header)
grad.Color = ColorSequence.new(Color3.fromRGB(70, 230, 130), Color3.fromRGB(30, 150, 80))
grad.Rotation = 90

local title = label(header, "David Hub", 22, Color3.new(1, 1, 1), Enum.Font.GothamBlack)
title.Position = UDim2.fromOffset(12, 0); title.Size = UDim2.new(1, -170, 1, 0)
title.TextXAlignment = Enum.TextXAlignment.Left
local ts = Instance.new("UIStroke", title); ts.Thickness = 2; ts.Color = Color3.new(0, 0, 0)

local gear = Instance.new("TextButton", header)
gear.Size = UDim2.fromOffset(34, 34); gear.Position = UDim2.new(1, -150, 0.5, -17)
gear.BackgroundColor3 = DARK; gear.Text = "⚙"; gear.TextSize = 20
gear.TextColor3 = GREEN; gear.Font = Enum.Font.GothamBold; gear.BorderSizePixel = 0
corner(gear, 6); stroke(gear, Color3.fromRGB(120, 255, 170), 2)

local autoBtn = Instance.new("TextButton", header)
autoBtn.Size = UDim2.fromOffset(100, 34); autoBtn.Position = UDim2.new(1, -108, 0.5, -17)
autoBtn.BackgroundColor3 = Color3.fromRGB(40, 60, 50); autoBtn.Text = "AUTO OFF"
autoBtn.TextColor3 = Color3.new(1, 1, 1); autoBtn.Font = Enum.Font.GothamBold
autoBtn.TextSize = 13; autoBtn.BorderSizePixel = 0
corner(autoBtn, 6); stroke(autoBtn, Color3.fromRGB(150, 150, 150), 2)

autoBtn.MouseButton1Click:Connect(function()
    Config.AutoHop = not Config.AutoHop
    autoBtn.Text = Config.AutoHop and "AUTO ON" or "AUTO OFF"
    autoBtn.BackgroundColor3 = Config.AutoHop and GREEN_D or Color3.fromRGB(40, 60, 50)
end)

-- Arrastar
do
    local drag, start, orig
    header.InputBegan:Connect(function(i)
        if i.UserInputType == Enum.UserInputType.MouseButton1 or i.UserInputType == Enum.UserInputType.Touch then
            drag, start, orig = true, i.Position, main.Position
        end
    end)
    UIS.InputChanged:Connect(function(i)
        if drag and (i.UserInputType == Enum.UserInputType.MouseMovement or i.UserInputType == Enum.UserInputType.Touch) then
            local d = i.Position - start
            main.Position = UDim2.new(orig.X.Scale, orig.X.Offset + d.X, orig.Y.Scale, orig.Y.Offset + d.Y)
        end
    end)
    UIS.InputEnded:Connect(function(i)
        if i.UserInputType == Enum.UserInputType.MouseButton1 or i.UserInputType == Enum.UserInputType.Touch then drag = false end
    end)
end

-- Chips
local chipBar = Instance.new("Frame", main)
chipBar.Position = UDim2.fromOffset(8, 54); chipBar.Size = UDim2.new(1, -16, 0, 46)
chipBar.BackgroundTransparency = 1
local cl = Instance.new("UIListLayout", chipBar)
cl.FillDirection = Enum.FillDirection.Horizontal; cl.Padding = UDim.new(0, 6)

local function chip(top, value, color)
    local b = Instance.new("TextButton", chipBar)
    b.Size = UDim2.fromOffset(86, 46); b.BackgroundColor3 = color
    b.Text = ""; b.AutoButtonColor = true; b.BorderSizePixel = 0
    corner(b, 6); stroke(b, Color3.new(1, 1, 1), 2)
    local t = label(b, top, 10, Color3.new(1, 1, 1))
    t.Size = UDim2.new(1, 0, 0, 16); t.Position = UDim2.fromOffset(0, 3)
    local v = label(b, value, 15, Color3.new(1, 1, 1), Enum.Font.GothamBlack)
    v.Size = UDim2.new(1, 0, 0, 22); v.Position = UDim2.fromOffset(0, 20)
    return b, v
end

local fjBtn, fjVal = chip("FORCE JOIN", Config.Delay .. "s", Color3.fromRGB(46, 190, 100))
local mvBtn, mvVal = chip("MIN VALUE", Config.MinB .. "B/s", Color3.fromRGB(30, 140, 110))
local rrBtn, rrVal = chip("RARITY", Config.Rarity, Color3.fromRGB(40, 160, 90))
local acBtn, acVal = chip("ACCOUNT", LP.DisplayName:sub(1, 8), Color3.fromRGB(110, 170, 40))

fjBtn.MouseButton1Click:Connect(function()
    Config.Delay = Config.Delay >= 5 and 2 or Config.Delay + 1
    fjVal.Text = Config.Delay .. "s"
end)
mvBtn.MouseButton1Click:Connect(function()
    local opts = { 1, 2, 3, 5, 7, 10 }
    local idx = table.find(opts, Config.MinB) or 0
    Config.MinB = opts[idx % #opts + 1]
    mvVal.Text = Config.MinB .. "B/s"
end)
rrBtn.MouseButton1Click:Connect(function()
    local idx = table.find(RARITIES, Config.Rarity) or 1
    Config.Rarity = RARITIES[idx % #RARITIES + 1]
    rrVal.Text = Config.Rarity
end)

local sub = label(main, "Top 5 recent eggs  |  David Hub", 13, Color3.new(1, 1, 1), Enum.Font.GothamBold)
sub.Position = UDim2.fromOffset(0, 104); sub.Size = UDim2.new(1, 0, 0, 20)

-- Lista (Logs)
local list = Instance.new("Frame", main)
list.Position = UDim2.fromOffset(8, 128); list.Size = UDim2.new(1, -16, 0, 300)
list.BackgroundTransparency = 1
local ll = Instance.new("UIListLayout", list)
ll.Padding = UDim.new(0, 6); ll.SortOrder = Enum.SortOrder.LayoutOrder

-- Config panel (engrenagem)
local cfg = Instance.new("Frame", main)
cfg.Position = UDim2.fromOffset(8, 128); cfg.Size = UDim2.new(1, -16, 0, 300)
cfg.BackgroundColor3 = ROW; cfg.Visible = false; cfg.ZIndex = 5
corner(cfg, 8); stroke(cfg, GREEN_D, 2)

local function cfgRow(y, text, default, cb)
    local l = label(cfg, text, 13, GREEN)
    l.Position = UDim2.fromOffset(10, y); l.Size = UDim2.fromOffset(200, 28)
    l.TextXAlignment = Enum.TextXAlignment.Left; l.ZIndex = 6
    local t = Instance.new("TextBox", cfg)
    t.Position = UDim2.new(1, -90, 0, y); t.Size = UDim2.fromOffset(80, 28)
    t.BackgroundColor3 = DARK; t.TextColor3 = GREEN; t.Text = tostring(default)
    t.Font = Enum.Font.GothamBold; t.TextSize = 13; t.BorderSizePixel = 0; t.ZIndex = 6
    corner(t, 6)
    t.FocusLost:Connect(function() cb(t) end)
end

local cfgTitle = label(cfg, "Config", 16, Color3.new(1, 1, 1), Enum.Font.GothamBlack)
cfgTitle.Size = UDim2.new(1, 0, 0, 28); cfgTitle.ZIndex = 6

cfgRow(34, "Tamanho da UI (0.5 - 2)", Config.UIScale, function(t)
    local v = math.clamp(tonumber(t.Text) or 1, 0.5, 2)
    Config.UIScale = v; uiScale.Scale = v; t.Text = tostring(v)
end)
cfgRow(70, "Valor mín. (1 - 10 B/s)", Config.MinB, function(t)
    Config.MinB = math.clamp(tonumber(t.Text) or 1, 1, 10)
    t.Text = tostring(Config.MinB); mvVal.Text = Config.MinB .. "B/s"
end)
cfgRow(106, "Valor máx. (1 - 10 B/s)", Config.MaxB, function(t)
    Config.MaxB = math.clamp(tonumber(t.Text) or 10, 1, 10); t.Text = tostring(Config.MaxB)
end)

local status = label(cfg, "Status: aguardando", 12, GREEN, Enum.Font.Gotham)
status.Position = UDim2.fromOffset(10, 150); status.Size = UDim2.new(1, -20, 0, 40)
status.TextWrapped = true; status.TextXAlignment = Enum.TextXAlignment.Left; status.ZIndex = 6

gear.MouseButton1Click:Connect(function() cfg.Visible = not cfg.Visible end)

--================ JOIN (Force Join) ================--
local joining = false
local function joinServer(jobId)
    if joining then return end
    joining = true
    task.spawn(function()
        for s = Config.Delay, 1, -1 do
            status.Text = "Entrando em " .. s .. "s..."
            task.wait(1)
        end
        for attempt = 1, 10 do -- força a entrada
            status.Text = "Tentativa " .. attempt .. "/10"
            local ok = pcall(function()
                TeleportService:TeleportToPlaceInstance(game.PlaceId, jobId, LP)
            end)
            task.wait(ok and 5 or 1.5)
        end
        joining = false
    end)
end
TeleportService.TeleportInitFailed:Connect(function() joining = false end)

--================ LOGS (Top 5) ================--
local entries, seen, order = {}, {}, 0

local function ago(t)
    local d = os.time() - t
    if d < 60 then return d .. "s ago" end
    return math.floor(d / 60) .. "m ago"
end

local function addEntry(name, value, rarity, mult, jobId)
    order -= 1
    local f = Instance.new("Frame", list)
    f.Size = UDim2.new(1, 0, 0, 52); f.BackgroundColor3 = ROW
    f.BorderSizePixel = 0; f.LayoutOrder = order
    corner(f, 6); stroke(f, DARK, 2)

    local icon = label(f, "🥚", 24)
    icon.Size = UDim2.fromOffset(40, 52); icon.Position = UDim2.fromOffset(4, 0)

    local n = label(f, name, 15, RARITY_COLOR[rarity] or Color3.new(1, 1, 1), Enum.Font.GothamBlack)
    n.Position = UDim2.fromOffset(50, 4); n.Size = UDim2.new(1, -140, 0, 22)
    n.TextXAlignment = Enum.TextXAlignment.Left
    local ns = Instance.new("UIStroke", n); ns.Thickness = 1.5; ns.Color = Color3.new(0, 0, 0)

    local v = label(f, "$" .. formatValue(value) .. "/s", 12, Color3.fromRGB(110, 255, 140))
    v.Position = UDim2.fromOffset(50, 28); v.Size = UDim2.fromOffset(95, 18)
    v.TextXAlignment = Enum.TextXAlignment.Left

    local m = label(f, "x" .. string.format("%.2f", mult), 12, Color3.fromRGB(255, 200, 60))
    m.Position = UDim2.fromOffset(148, 28); m.Size = UDim2.fromOffset(45, 18)

    local t = label(f, ago(os.time()), 11, Color3.fromRGB(200, 200, 200), Enum.Font.Gotham)
    t.Position = UDim2.fromOffset(196, 28); t.Size = UDim2.fromOffset(60, 18)
    t.TextXAlignment = Enum.TextXAlignment.Left

    local j = Instance.new("TextButton", f)
    j.Size = UDim2.fromOffset(80, 42); j.Position = UDim2.new(1, -86, 0.5, -21)
    j.BackgroundColor3 = Color3.fromRGB(60, 220, 110); j.Text = "Join"
    j.TextColor3 = Color3.new(1, 1, 1); j.Font = Enum.Font.GothamBlack
    j.TextSize = 18; j.BorderSizePixel = 0
    corner(j, 6); stroke(j, Color3.fromRGB(10, 90, 40), 2)
    local js = Instance.new("UIStroke", j.Parent == f and j or j) -- contorno do texto
    js.ApplyStrokeMode = Enum.ApplyStrokeMode.Contextual; js.Thickness = 1.5; js.Color = Color3.fromRGB(10, 90, 40)
    j.MouseButton1Click:Connect(function() joinServer(jobId) end)

    table.insert(entries, { frame = f, label = t, time = os.time() })
    if #entries > 5 then
        entries[1].frame:Destroy()
        table.remove(entries, 1)
    end
end

task.spawn(function()
    while task.wait(5) do
        for _, e in ipairs(entries) do e.label.Text = ago(e.time) end
    end
end)

--================ SCAN ================--
local function scan()
    local minV, maxV = Config.MinB * 1e9, Config.MaxB * 1e9
    local found = false
    for _, root in ipairs(getContainers()) do
        for _, m in ipairs(root:GetDescendants()) do
            if m:IsA("Model") and isPet(m) then
                local v = getValue(m)
                local r = getRarity(m)
                local rarityOk = Config.Rarity == "All" or r == Config.Rarity
                if v and v >= minV and v <= maxV and rarityOk then
                    local key = game.JobId .. m:GetFullName()
                    if not seen[key] then
                        seen[key] = true
                        addEntry(m.Name, v, r, getMult(m), game.JobId)
                    end
                    found = true
                end
            end
        end
    end
    return found
end

--================ AUTO HOP ================--
local function nextServer()
    local url = ("https://games.roblox.com/v1/games/%d/servers/Public?sortOrder=Asc&limit=100"):format(game.PlaceId)
    local ok, res = pcall(function() return HttpService:JSONDecode(game:HttpGet(url)) end)
    if not ok or not res.data then return end
    for _, s in ipairs(res.data) do
        if s.id ~= game.JobId and s.playing < s.maxPlayers then return s.id end
    end
end

task.spawn(function()
    while task.wait(Config.ScanInterval) do
        local found = scan()
        if Config.AutoHop and not found and not joining then
            local id = nextServer()
            if id then joinServer(id) end
        end
    end
end)
