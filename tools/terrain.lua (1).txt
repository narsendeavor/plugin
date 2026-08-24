-- ╔══════════════════════════════════════════════════════════╗
-- ║    PRO TERRAIN EDITOR v3 - UNLIMITED + MOBILE FLY       ║
-- ║  • PREMIUM ZARS V3                                  ║
-- ║  • Drag GUI untuk pindah posisi                         ║
-- ║  • Pinch/slider untuk resize GUI                        ║
-- ║  • FLY MODE khusus HP (kontrol Roblox native)           ║
-- ╚══════════════════════════════════════════════════════════╝

local Players        = game:GetService("Players")
local RunService     = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local TweenService   = game:GetService("TweenService")

local player  = Players.LocalPlayer
local mouse   = player:GetMouse()
local camera  = workspace.CurrentCamera
local terrain = workspace.Terrain

-- ── Detect platform ──────────────────────────────────────
local isMobile = UserInputService.TouchEnabled and not UserInputService.KeyboardEnabled

-- ── State ────────────────────────────────────────────────
local paintingEnabled = false
local dragMode        = false
local isDragging      = false
local dragStartPos    = nil
local currentMaterial = Enum.Material.Grass
local currentShape    = "Ball"
local brushSize       = 10
local brushHeight     = 10
local savedData       = {}
local flatTerrainMode = false
local flatHeight      = 2
local flatSize        = 100

-- FLY state
local flyEnabled   = false
local flySpeed     = 40
local flyConn      = nil
local bodyVel      = nil
local bodyGyro     = nil

-- GUI Resize state
local guiScale     = 1.0
local BASE_W       = 300
local BASE_H       = 520

-- ════════════════════════════════════════════════════════
-- SCREEN GUI
-- ════════════════════════════════════════════════════════
local screenGui = Instance.new("ScreenGui", player.PlayerGui)
screenGui.Name          = "TerrainProV3"
screenGui.ResetOnSpawn  = false
screenGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling

-- ── HELPERS ──────────────────────────────────────────────
local function corner(obj, r)
    local c = Instance.new("UICorner", obj)
    c.CornerRadius = UDim.new(0, r or 6)
    return c
end

local function stroke(obj, col, th)
    local s = Instance.new("UIStroke", obj)
    s.Color     = col or Color3.fromRGB(70,100,180)
    s.Thickness = th  or 1.2
    s.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
    return s
end

local function gradient(obj, a, b, rot)
    local g = Instance.new("UIGradient", obj)
    g.Color    = ColorSequence.new(a, b)
    g.Rotation = rot or 90
    return g
end

local function mkBtn(parent, text, bgCol, x, y, w, h, cb)
    local f = Instance.new("Frame", parent)
    f.Position              = UDim2.new(0, x, 0, y)
    f.Size                  = UDim2.new(0, w, 0, h)
    f.BackgroundColor3      = bgCol
    f.BorderSizePixel       = 0
    corner(f, 6)
    stroke(f, Color3.fromRGB(60,60,80), 1)

    local btn = Instance.new("TextButton", f)
    btn.Size                = UDim2.new(1,0,1,0)
    btn.BackgroundTransparency = 1
    btn.Text                = text
    btn.TextSize            = 12
    btn.TextColor3          = Color3.new(1,1,1)
    btn.Font                = Enum.Font.GothamMedium
    btn.AutoButtonColor     = false

    -- hover effect
    btn.MouseEnter:Connect(function()
        TweenService:Create(f, TweenInfo.new(0.12), {BackgroundColor3 = bgCol:Lerp(Color3.new(1,1,1), 0.15)}):Play()
    end)
    btn.MouseLeave:Connect(function()
        TweenService:Create(f, TweenInfo.new(0.12), {BackgroundColor3 = bgCol}):Play()
    end)

    if cb then btn.MouseButton1Click:Connect(function() cb(btn) end) end
    return btn, f
end

local function mkLabel(parent, text, x, y, w, h, size, col, align)
    local lbl = Instance.new("TextLabel", parent)
    lbl.Position            = UDim2.new(0, x, 0, y)
    lbl.Size                = UDim2.new(0, w, 0, h)
    lbl.BackgroundTransparency = 1
    lbl.Text                = text
    lbl.TextSize            = size or 11
    lbl.TextColor3          = col or Color3.fromRGB(190,190,210)
    lbl.Font                = Enum.Font.Gotham
    lbl.TextXAlignment      = align or Enum.TextXAlignment.Left
    lbl.TextYAlignment      = Enum.TextYAlignment.Center
    return lbl
end

local function mkSep(parent, y)
    local line = Instance.new("Frame", parent)
    line.Position           = UDim2.new(0, 8, 0, y)
    line.Size               = UDim2.new(1, -16, 0, 1)
    line.BackgroundColor3   = Color3.fromRGB(60,60,90)
    line.BorderSizePixel    = 0
    return line
end

-- ════════════════════════════════════════════════════════
-- TOMBOL BUKA (selalu terlihat, bisa di-drag)
-- ════════════════════════════════════════════════════════
local openBtn = Instance.new("TextButton", screenGui)
openBtn.Size               = UDim2.new(0, 38, 0, 38)
openBtn.Position           = UDim2.new(0, 10, 0, 10)
openBtn.BackgroundColor3   = Color3.fromRGB(20,20,35)
openBtn.Text               = "⛏"
openBtn.TextSize            = 20
openBtn.TextColor3          = Color3.fromRGB(140,200,255)
openBtn.Font                = Enum.Font.GothamBold
openBtn.AutoButtonColor     = false

openBtn.Active             = true 
openBtn.Draggable          = true

corner(openBtn, 8)
stroke(openBtn, Color3.fromRGB(80,130,220), 1.5)

-- ════════════════════════════════════════════════════════
-- MAIN FRAME — dapat di-drag & di-resize
-- ════════════════════════════════════════════════════════
local mainFrame = Instance.new("Frame", screenGui)
mainFrame.Size             = UDim2.new(0, BASE_W, 0, BASE_H)
mainFrame.Position         = UDim2.new(0, 10, 0, 55)
mainFrame.BackgroundColor3 = Color3.fromRGB(10,10,18)
mainFrame.Visible          = false
mainFrame.ClipsDescendants = true
corner(mainFrame, 10)
stroke(mainFrame, Color3.fromRGB(60,90,180), 1.5)

-- ── TITLE BAR (drag handle) ───────────────────────────
local titleBar = Instance.new("Frame", mainFrame)
titleBar.Size              = UDim2.new(1,0,0,36)
titleBar.Position          = UDim2.new(0,0,0,0)
titleBar.BackgroundColor3  = Color3.fromRGB(18,18,32)
titleBar.BorderSizePixel   = 0
corner(titleBar, 10)
gradient(titleBar, Color3.fromRGB(30,50,110), Color3.fromRGB(10,15,40), 90)

local titleTxt = mkLabel(titleBar, "⛏  PRO TERRAIN EDITOR", 10, 0, 220, 36, 13, Color3.fromRGB(140,200,255))
titleTxt.Font = Enum.Font.GothamBold

-- Tombol X tutup
local closeBtn = Instance.new("TextButton", titleBar)
closeBtn.Size              = UDim2.new(0,28,0,28)
closeBtn.Position          = UDim2.new(1,-32,0,4)
closeBtn.BackgroundColor3  = Color3.fromRGB(180,30,30)
closeBtn.Text              = "✕"
closeBtn.TextSize          = 13
closeBtn.TextColor3        = Color3.new(1,1,1)
closeBtn.Font              = Enum.Font.GothamBold
closeBtn.AutoButtonColor   = false
corner(closeBtn, 6)
closeBtn.MouseButton1Click:Connect(function()
    mainFrame.Visible = false
    openBtn.Text = "⛏"
end)

-- ── DRAG MAINFRAME via title bar ─────────────────────
do
    local draggingFrame = false
    local dragOffset    = Vector2.new()

    titleBar.InputBegan:Connect(function(inp)
        if inp.UserInputType == Enum.UserInputType.MouseButton1
        or inp.UserInputType == Enum.UserInputType.Touch then
            draggingFrame = true
            local abs = mainFrame.AbsolutePosition
            local pos = inp.Position
            dragOffset = Vector2.new(pos.X - abs.X, pos.Y - abs.Y)
        end
    end)
    UserInputService.InputChanged:Connect(function(inp)
        if draggingFrame and (inp.UserInputType == Enum.UserInputType.MouseMovement
            or inp.UserInputType == Enum.UserInputType.Touch) then
            local pos = inp.Position
            mainFrame.Position = UDim2.new(0, pos.X - dragOffset.X, 0, pos.Y - dragOffset.Y)
        end
    end)
    UserInputService.InputEnded:Connect(function(inp)
        if inp.UserInputType == Enum.UserInputType.MouseButton1
        or inp.UserInputType == Enum.UserInputType.Touch then
            draggingFrame = false
        end
    end)
end

-- ── STATUS BAR (bawah frame) ─────────────────────────
local statusBar = Instance.new("Frame", mainFrame)
statusBar.Size             = UDim2.new(1,0,0,24)
statusBar.Position         = UDim2.new(0,0,1,-24)
statusBar.BackgroundColor3 = Color3.fromRGB(6,6,14)
statusBar.BorderSizePixel  = 0

local statusLbl = mkLabel(statusBar, "Siap | UNLIMITED", 8, 0, BASE_W-16, 24, 10, Color3.fromRGB(80,220,140))
statusLbl.TextXAlignment = Enum.TextXAlignment.Left

-- ── RESIZE SLIDER (kanan bawah) ───────────────────────
local resizeBtn = Instance.new("TextButton", mainFrame)
resizeBtn.Size             = UDim2.new(0,22,0,22)
resizeBtn.Position         = UDim2.new(1,-22,1,-22)
resizeBtn.BackgroundColor3 = Color3.fromRGB(60,80,160)
resizeBtn.Text             = "⤡"
resizeBtn.TextSize          = 14
resizeBtn.TextColor3        = Color3.new(1,1,1)
resizeBtn.Font              = Enum.Font.GothamBold
resizeBtn.AutoButtonColor   = false
corner(resizeBtn, 4)

do
    local resizing     = false
    local resizeStart  = Vector2.new()
    local sizeStart    = Vector2.new()

    resizeBtn.InputBegan:Connect(function(inp)
        if inp.UserInputType == Enum.UserInputType.MouseButton1
        or inp.UserInputType == Enum.UserInputType.Touch then
            resizing    = true
            resizeStart = Vector2.new(inp.Position.X, inp.Position.Y)
            local abs   = mainFrame.AbsoluteSize
            sizeStart   = Vector2.new(abs.X, abs.Y)
        end
    end)
    UserInputService.InputChanged:Connect(function(inp)
        if resizing and (inp.UserInputType == Enum.UserInputType.MouseMovement
            or inp.UserInputType == Enum.UserInputType.Touch) then
            local dx  = inp.Position.X - resizeStart.X
            local dy  = inp.Position.Y - resizeStart.Y
            local nw  = math.clamp(sizeStart.X + dx, 220, 500)
            local nh  = math.clamp(sizeStart.Y + dy, 300, 700)
            mainFrame.Size = UDim2.new(0, nw, 0, nh)
        end
    end)
    UserInputService.InputEnded:Connect(function(inp)
        if inp.UserInputType == Enum.UserInputType.MouseButton1
        or inp.UserInputType == Enum.UserInputType.Touch then
            resizing = false
        end
    end)
end

-- ── SCROLLING CONTENT ─────────────────────────────────
local scrollFrame = Instance.new("ScrollingFrame", mainFrame)
scrollFrame.Size              = UDim2.new(1,0,1,-60)  -- minus titleBar & statusBar
scrollFrame.Position          = UDim2.new(0,0,0,36)
scrollFrame.BackgroundTransparency = 1
scrollFrame.ScrollBarThickness = 4
scrollFrame.ScrollBarImageColor3 = Color3.fromRGB(80,120,220)
scrollFrame.CanvasSize        = UDim2.new(0,0,0,0)    -- akan di-update
scrollFrame.AutomaticCanvasSize = Enum.AutomaticSize.Y
scrollFrame.ElasticBehavior   = Enum.ElasticBehavior.WhenScrollable

local layout = Instance.new("UIListLayout", scrollFrame)
layout.Padding              = UDim.new(0,6)
layout.HorizontalAlignment  = Enum.HorizontalAlignment.Center
layout.SortOrder            = Enum.SortOrder.LayoutOrder

local padding = Instance.new("UIPadding", scrollFrame)
padding.PaddingTop    = UDim.new(0,6)
padding.PaddingBottom = UDim.new(0,30)
padding.PaddingLeft   = UDim.new(0,8)
padding.PaddingRight  = UDim.new(0,8)

-- ════════════════════════════════════════════════════════
-- SECTION BUILDER HELPERS
-- ════════════════════════════════════════════════════════
local function secHeader(text, order)
    local lbl = Instance.new("TextLabel", scrollFrame)
    lbl.Size               = UDim2.new(1,-4,0,20)
    lbl.BackgroundTransparency = 1
    lbl.Text               = text
    lbl.TextSize            = 11
    lbl.TextColor3          = Color3.fromRGB(100,170,255)
    lbl.Font                = Enum.Font.GothamBold
    lbl.TextXAlignment      = Enum.TextXAlignment.Left
    lbl.LayoutOrder         = order or 0
    return lbl
end

local function secLine(order)
    local f = Instance.new("Frame", scrollFrame)
    f.Size               = UDim2.new(1,-4,0,1)
    f.BackgroundColor3   = Color3.fromRGB(45,45,70)
    f.BorderSizePixel    = 0
    f.LayoutOrder        = order or 0
    return f
end

-- Grid container (untuk tombol material)
local function matGrid(order, cols, btnW, btnH, gap)
    local cont = Instance.new("Frame", scrollFrame)
    cont.Size              = UDim2.new(1,-4,0,0)
    cont.AutomaticSize     = Enum.AutomaticSize.Y
    cont.BackgroundTransparency = 1
    cont.LayoutOrder       = order
    local gl = Instance.new("UIGridLayout", cont)
    gl.CellSize            = UDim2.new(0, btnW, 0, btnH)
    gl.CellPadding         = UDim2.new(0, gap, 0, gap)
    gl.HorizontalAlignment = Enum.HorizontalAlignment.Left
    gl.SortOrder           = Enum.SortOrder.LayoutOrder
    return cont
end

-- Row container (tombol berjajar horizontal)
local function rowCont(order, h)
    local cont = Instance.new("Frame", scrollFrame)
    cont.Size              = UDim2.new(1,-4,0,h)
    cont.BackgroundTransparency = 1
    cont.LayoutOrder       = order
    local rl = Instance.new("UIListLayout", cont)
    rl.FillDirection        = Enum.FillDirection.Horizontal
    rl.Padding              = UDim.new(0,4)
    rl.VerticalAlignment    = Enum.VerticalAlignment.Center
    return cont
end

-- Simple toggling button
local function toggleBtn(parent, textOff, textOn, colOff, colOn, order, cb)
    local state = false
    local btn, f = mkBtn(parent, textOff, colOff, 0, 0, 0, 32, nil)
    btn.Size = UDim2.new(1,0,1,0)
    if f then f.Size = UDim2.new(1,-4,0,32) f.LayoutOrder = order end
    btn.MouseButton1Click:Connect(function()
        state = not state
        btn.Text            = state and textOn or textOff
        f.BackgroundColor3  = state and colOn or colOff
        cb(state, btn)
    end)
    return btn, f, function() return state end
end

-- ════════════════════════════════════════════════════════
-- KONTEN SCROLL
-- ════════════════════════════════════════════════════════

-- ── 1. TERRAIN ON/OFF ────────────────────────────────
local lo = 1
local onOffFrame = Instance.new("Frame", scrollFrame)
onOffFrame.Size              = UDim2.new(1,-4,0,36)
onOffFrame.BackgroundColor3  = Color3.fromRGB(100,0,0)
onOffFrame.LayoutOrder       = lo; lo = lo+1
corner(onOffFrame, 6)
stroke(onOffFrame, Color3.fromRGB(180,30,30), 1)

local onOffBtn = Instance.new("TextButton", onOffFrame)
onOffBtn.Size              = UDim2.new(1,0,1,0)
onOffBtn.BackgroundTransparency = 1
onOffBtn.Text              = "🔴  TERRAIN: OFF  —  Tap untuk aktifkan"
onOffBtn.TextSize           = 12
onOffBtn.TextColor3         = Color3.new(1,1,1)
onOffBtn.Font               = Enum.Font.GothamBold
onOffBtn.MouseButton1Click:Connect(function()
    paintingEnabled = not paintingEnabled
    if paintingEnabled then
        onOffFrame.BackgroundColor3 = Color3.fromRGB(0,110,0)
        onOffBtn.Text = "🟢  TERRAIN: ON  ✓"
        stroke(onOffFrame, Color3.fromRGB(30,200,60), 1.5)
    else
        onOffFrame.BackgroundColor3 = Color3.fromRGB(100,0,0)
        onOffBtn.Text = "🔴  TERRAIN: OFF  —  Tap untuk aktifkan"
        stroke(onOffFrame, Color3.fromRGB(180,30,30), 1)
    end
end)

-- ── 2. MATERIAL ALAM ─────────────────────────────────
secHeader("🌿 MATERIAL ALAM", lo); lo=lo+1

local matButtons = {}
local function selectMat(mat, btn)
    for _, b in pairs(matButtons) do
        local nc = b:GetAttribute("nc")
        if nc then b.Parent.BackgroundColor3 = nc end
    end
    currentMaterial = mat
    if btn then
        btn.Parent.BackgroundColor3 = Color3.fromRGB(0,160,90)
        statusLbl.Text = "Material: "..mat.Name
    end
end

local alamMats = {
    {"🌿 Rumput",       Enum.Material.Grass,       Color3.fromRGB(30,65,30)},
    {"🟫 Tanah",        Enum.Material.Ground,       Color3.fromRGB(55,40,25)},
    {"💩 Lumpur",       Enum.Material.Mud,          Color3.fromRGB(50,35,20)},
    {"🏜 Pasir",        Enum.Material.Sand,         Color3.fromRGB(70,60,25)},
    {"❄ Salju",         Enum.Material.Snow,         Color3.fromRGB(55,70,90)},
    {"🌱 Rmpt Daun",    Enum.Material.LeafyGrass,   Color3.fromRGB(25,70,30)},
    {"🛣 Aspal",        Enum.Material.Asphalt,      Color3.fromRGB(35,35,35)},
}

local grid1 = matGrid(lo, 2, 130, 28, 4); lo=lo+1
for _, item in ipairs(alamMats) do
    local btn, f = mkBtn(grid1, item[1], item[3], 0,0,130,28, nil)
    btn:SetAttribute("nc", item[3])
    f:SetAttribute("nc", item[3])
    table.insert(matButtons, btn)
    btn.MouseButton1Click:Connect(function() selectMat(item[2], btn) end)
end

-- ── 3. MATERIAL BATU & MINERAL ───────────────────────
secHeader("🪨 BATU & MINERAL", lo); lo=lo+1

local batuMats = {
    {"🪨 Batu",         Enum.Material.Rock,         Color3.fromRGB(55,50,45)},
    {"⬛ Slate",        Enum.Material.Slate,         Color3.fromRGB(40,40,45)},
    {"🌋 Basalt",       Enum.Material.Basalt,        Color3.fromRGB(35,30,30)},
    {"🔲 Cobblestone",  Enum.Material.Cobblestone,   Color3.fromRGB(55,50,45)},
    {"💎 Granite",      Enum.Material.Granite,       Color3.fromRGB(80,60,60)},
    {"🪸 Limestone",    Enum.Material.Limestone,     Color3.fromRGB(80,75,55)},
    {"🚶 Pavement",     Enum.Material.Pavement,      Color3.fromRGB(60,60,65)},
    {"🧂 Garam",        Enum.Material.Salt,          Color3.fromRGB(90,85,80)},
    {"🌋 Lava Retak",   Enum.Material.CrackedLava,   Color3.fromRGB(90,30,10)},
    {"🏛 Marble",       Enum.Material.Marble,        Color3.fromRGB(85,80,80)},
    {"🏖 Sandstone",    Enum.Material.Sandstone,     Color3.fromRGB(75,60,35)},
}

local grid2 = matGrid(lo, 2, 130, 28, 4); lo=lo+1
for _, item in ipairs(batuMats) do
    local btn, f = mkBtn(grid2, item[1], item[3], 0,0,130,28, nil)
    btn:SetAttribute("nc", item[3])
    f:SetAttribute("nc", item[3])
    table.insert(matButtons, btn)
    btn.MouseButton1Click:Connect(function() selectMat(item[2], btn) end)
end

-- ── 4. CAIRAN & ES ───────────────────────────────────
secHeader("🌊 CAIRAN & ES", lo); lo=lo+1

local cairMats = {
    {"💧 Air",          Enum.Material.Water,   Color3.fromRGB(20,50,100)},
    {"🧊 Es",           Enum.Material.Ice,     Color3.fromRGB(60,100,140)},
    {"🏔 Glacier",      Enum.Material.Glacier, Color3.fromRGB(50,90,130)},
}

local grid3 = matGrid(lo, 2, 130, 28, 4); lo=lo+1
for _, item in ipairs(cairMats) do
    local btn, f = mkBtn(grid3, item[1], item[3], 0,0,130,28, nil)
    btn:SetAttribute("nc", item[3])
    f:SetAttribute("nc", item[3])
    table.insert(matButtons, btn)
    btn.MouseButton1Click:Connect(function() selectMat(item[2], btn) end)
end

-- ── 5. MATERIAL BUATAN (terrain-valid only) ──────────
secHeader("🧱 BUATAN", lo); lo=lo+1

local buatanMats = {
    {"🧱 Bata",         Enum.Material.Brick,        Color3.fromRGB(80,35,20)},
    {"🏗 Beton",        Enum.Material.Concrete,     Color3.fromRGB(65,65,65)},
    {"🪵 Kayu",         Enum.Material.Wood,         Color3.fromRGB(70,45,20)},
    {"🪟 Papan Kayu",   Enum.Material.WoodPlanks,   Color3.fromRGB(65,42,18)},
    {"🔵 Pebble",       Enum.Material.Pebble,       Color3.fromRGB(55,55,60)},
    {"🟦 Ceramic",      Enum.Material.CeramicTiles, Color3.fromRGB(40,60,80)},
}

local grid4 = matGrid(lo, 2, 130, 28, 4); lo=lo+1
for _, item in ipairs(buatanMats) do
    local btn, f = mkBtn(grid4, item[1], item[3], 0,0,130,28, nil)
    btn:SetAttribute("nc", item[3])
    f:SetAttribute("nc", item[3])
    table.insert(matButtons, btn)
    btn.MouseButton1Click:Connect(function() selectMat(item[2], btn) end)
end

-- Tombol Hapus Terrain (Air)
local delFrame = Instance.new("Frame", scrollFrame)
delFrame.Size             = UDim2.new(1,-4,0,28)
delFrame.BackgroundColor3 = Color3.fromRGB(160,20,20)
delFrame.LayoutOrder      = lo; lo=lo+1
corner(delFrame, 6)
local delBtn = Instance.new("TextButton", delFrame)
delBtn.Size               = UDim2.new(1,0,1,0)
delBtn.BackgroundTransparency = 1
delBtn.Text               = "🗑  HAPUS TERRAIN (Air)"
delBtn.TextSize            = 12
delBtn.TextColor3          = Color3.new(1,1,1)
delBtn.Font                = Enum.Font.GothamMedium
delBtn.MouseButton1Click:Connect(function() currentMaterial = Enum.Material.Air; statusLbl.Text = "Material: Air (Hapus)" end)

secLine(lo); lo=lo+1

-- ── 6. BENTUK ────────────────────────────────────────
secHeader("⬤ BENTUK BRUSH", lo); lo=lo+1

local shapeRow = rowCont(lo, 32); lo=lo+1
local ballBtn, ballF = mkBtn(shapeRow, "● Bola",  Color3.fromRGB(0,100,160), 0,0,0,32, nil)
local blkBtn,  blkF  = mkBtn(shapeRow, "■ Kotak", Color3.fromRGB(50,50,80),  0,0,0,32, nil)
ballBtn.Size = UDim2.new(0.5,-2,1,0)
blkBtn.Size  = UDim2.new(0.5,-2,1,0)
ballF.Size   = UDim2.new(0.5,-2,1,0)
blkF.Size    = UDim2.new(0.5,-2,1,0)

ballBtn.MouseButton1Click:Connect(function()
    currentShape = "Ball"
    ballF.BackgroundColor3 = Color3.fromRGB(0,130,200)
    blkF.BackgroundColor3  = Color3.fromRGB(50,50,80)
    statusLbl.Text = "Bentuk: Bola"
end)
blkBtn.MouseButton1Click:Connect(function()
    currentShape = "Block"
    blkF.BackgroundColor3  = Color3.fromRGB(80,80,150)
    ballF.BackgroundColor3 = Color3.fromRGB(50,80,100)
    statusLbl.Text = "Bentuk: Kotak"
end)

-- ── 7. UKURAN & KETEBALAN ────────────────────────────
secHeader("📐 UKURAN & KETEBALAN", lo); lo=lo+1

-- Ukuran
local function makeSliderRow(parent, label, initVal, minVal, maxVal, step, orderVal, onChg)
    local cont = Instance.new("Frame", parent)
    cont.Size             = UDim2.new(1,-4,0,54)
    cont.BackgroundColor3 = Color3.fromRGB(16,16,28)
    cont.LayoutOrder      = orderVal
    corner(cont, 6)
    stroke(cont, Color3.fromRGB(40,40,70), 1)

    mkLabel(cont, label, 8, 2, 150, 18, 10, Color3.fromRGB(140,160,200))

    local valLbl = mkLabel(cont, tostring(initVal), 200, 2, 60, 18, 11, Color3.fromRGB(100,210,255))
    valLbl.TextXAlignment = Enum.TextXAlignment.Right

    local current = initVal

    -- Minus / Plus row
    local r = Instance.new("Frame", cont)
    r.Position            = UDim2.new(0,8,0,22)
    r.Size                = UDim2.new(1,-16,0,26)
    r.BackgroundTransparency = 1
    local rl = Instance.new("UIListLayout", r)
    rl.FillDirection       = Enum.FillDirection.Horizontal
    rl.Padding             = UDim.new(0,4)

    local function makeStepBtn(txt, col, delta)
        local b,f = mkBtn(r, txt, col, 0,0,0,26, function()
            current = math.clamp(current + delta, minVal, maxVal)
            valLbl.Text = tostring(current)
            onChg(current)
        end)
        b.Size = UDim2.new(0,38,1,0)
        f.Size = UDim2.new(0,38,1,0)
    end

    makeStepBtn("−"..step, Color3.fromRGB(50,30,60), -step)
    makeStepBtn("+"..step, Color3.fromRGB(30,50,60),  step)

    -- Big step
    if step < 10 then
        makeStepBtn("−10", Color3.fromRGB(60,25,25), -10)
        makeStepBtn("+10", Color3.fromRGB(25,60,25),  10)
    end

    local maxBtnTxt = tostring(maxVal)
    local mb,mf = mkBtn(r, "MAX", Color3.fromRGB(60,30,80), 0,0,0,26, function()
        current = maxVal; valLbl.Text = tostring(current); onChg(current)
    end)
    mb.Size = UDim2.new(0,38,1,0); mf.Size = UDim2.new(0,38,1,0)

    return cont
end

local sizeSlider   = makeSliderRow(scrollFrame, "Ukuran Brush", brushSize, 2, 128, 2, lo, function(v) brushSize   = v end); lo=lo+1
local heightSlider = makeSliderRow(scrollFrame, "Ketebalan",    brushHeight, 2, 128, 2, lo, function(v) brushHeight = v end); lo=lo+1

secLine(lo); lo=lo+1

-- ── 8. MODE BRUSH ────────────────────────────────────
secHeader("🎨 MODE BRUSH", lo); lo=lo+1

local dragF = Instance.new("Frame", scrollFrame)
dragF.Size             = UDim2.new(1,-4,0,32)
dragF.BackgroundColor3 = Color3.fromRGB(80,40,130)
dragF.LayoutOrder      = lo; lo=lo+1
corner(dragF, 6)
local dragModeBtn = Instance.new("TextButton", dragF)
dragModeBtn.Size               = UDim2.new(1,0,1,0)
dragModeBtn.BackgroundTransparency = 1
dragModeBtn.Text               = "🖱️  MODE: KLIK"
dragModeBtn.TextSize            = 12
dragModeBtn.TextColor3          = Color3.new(1,1,1)
dragModeBtn.Font                = Enum.Font.GothamBold
dragModeBtn.MouseButton1Click:Connect(function()
    dragMode = not dragMode
    dragModeBtn.Text           = dragMode and "✋  MODE: GESER ✓" or "🖱️  MODE: KLIK"
    dragF.BackgroundColor3     = dragMode and Color3.fromRGB(120,50,200) or Color3.fromRGB(80,40,130)
end)

local flatF = Instance.new("Frame", scrollFrame)
flatF.Size             = UDim2.new(1,-4,0,32)
flatF.BackgroundColor3 = Color3.fromRGB(30,60,100)
flatF.LayoutOrder      = lo; lo=lo+1
corner(flatF, 6)
local flatModeBtn = Instance.new("TextButton", flatF)
flatModeBtn.Size               = UDim2.new(1,0,1,0)
flatModeBtn.BackgroundTransparency = 1
flatModeBtn.Text               = "⬜  MODE: NORMAL"
flatModeBtn.TextSize            = 12
flatModeBtn.TextColor3          = Color3.new(1,1,1)
flatModeBtn.Font                = Enum.Font.GothamBold
flatModeBtn.MouseButton1Click:Connect(function()
    flatTerrainMode = not flatTerrainMode
    flatModeBtn.Text       = flatTerrainMode and "⬛  MODE: DATARAN ✓" or "⬜  MODE: NORMAL"
    flatF.BackgroundColor3 = flatTerrainMode and Color3.fromRGB(0,120,180) or Color3.fromRGB(30,60,100)
end)

secLine(lo); lo=lo+1

-- ── 9. DATARAN ────────────────────────────────────────
secHeader("🏔️ PENGATURAN DATARAN", lo); lo=lo+1

local flatHSlider = makeSliderRow(scrollFrame, "Tinggi Dataran", flatHeight, 1, 200, 1, lo, function(v) flatHeight = v end); lo=lo+1
local flatSSlider = makeSliderRow(scrollFrame, "Lebar Dataran",  flatSize,   20, 1000, 10, lo, function(v) flatSize  = v end); lo=lo+1

secLine(lo); lo=lo+1

-- ── 10. AKSI ─────────────────────────────────────────
secHeader("🔧 AKSI", lo); lo=lo+1

local function actionBtn(txt, col, orderVal, cb)
    local f = Instance.new("Frame", scrollFrame)
    f.Size             = UDim2.new(1,-4,0,32)
    f.BackgroundColor3 = col
    f.LayoutOrder      = orderVal
    corner(f, 6)
    local b = Instance.new("TextButton", f)
    b.Size               = UDim2.new(1,0,1,0)
    b.BackgroundTransparency = 1
    b.Text               = txt
    b.TextSize            = 12
    b.TextColor3          = Color3.new(1,1,1)
    b.Font                = Enum.Font.GothamMedium
    b.MouseButton1Click:Connect(function() cb(b) end)
    return b
end

actionBtn("🗑️  CLEAR SEMUA TERRAIN", Color3.fromRGB(180,20,20), lo, function(b)
    terrain:Clear()
    b.Text = "✓ Terrain Dibersihkan!"
    wait(1.5)
    b.Text = "🗑️  CLEAR SEMUA TERRAIN"
end); lo=lo+1

actionBtn("🔄  RESET DATA SAVED", Color3.fromRGB(140,90,0), lo, function(b)
    savedData = {}
    b.Text = "✓ Data Direset!"
    wait(1.5)
    b.Text = "🔄  RESET DATA SAVED"
end); lo=lo+1

secLine(lo); lo=lo+1

-- ── 11. GENERATE CODE ────────────────────────────────
secHeader("💾 EXPORT CODE", lo); lo=lo+1

-- Export Frame
local exportFrame = Instance.new("Frame", screenGui)
exportFrame.Size             = UDim2.new(0.92,0,0.8,0)
exportFrame.Position         = UDim2.new(0.04,0,0.1,0)
exportFrame.BackgroundColor3 = Color3.fromRGB(8,8,12)
exportFrame.Visible          = false
exportFrame.ZIndex           = 200
corner(exportFrame, 10)
stroke(exportFrame, Color3.fromRGB(60,90,180), 1.5)

local expTitle = mkLabel(exportFrame, "💾 GENERATED CODE", 12, 8, 300, 26, 13, Color3.fromRGB(140,200,255))
expTitle.Font = Enum.Font.GothamBold
expTitle.ZIndex = 201

local expClose = Instance.new("TextButton", exportFrame)
expClose.Size              = UDim2.new(0,30,0,30)
expClose.Position          = UDim2.new(1,-35,0,5)
expClose.BackgroundColor3  = Color3.fromRGB(180,20,20)
expClose.Text              = "✕"
expClose.TextSize           = 14
expClose.TextColor3         = Color3.new(1,1,1)
expClose.Font               = Enum.Font.GothamBold
expClose.ZIndex             = 205
corner(expClose, 6)
expClose.MouseButton1Click:Connect(function() exportFrame.Visible = false end)

local expScroll = Instance.new("ScrollingFrame", exportFrame)
expScroll.Size              = UDim2.new(1,-8,1,-70)
expScroll.Position          = UDim2.new(0,4,0,40)
expScroll.BackgroundTransparency = 1
expScroll.ScrollBarThickness = 5
expScroll.ZIndex            = 201

local tBox = Instance.new("TextBox", expScroll)
tBox.Size                   = UDim2.new(1,0,1,0)
tBox.MultiLine              = true
tBox.TextEditable           = false
tBox.TextColor3             = Color3.fromRGB(80,255,140)
tBox.BackgroundColor3       = Color3.new(0,0,0)
tBox.TextXAlignment         = Enum.TextXAlignment.Left
tBox.TextYAlignment         = Enum.TextYAlignment.Top
tBox.ClearTextOnFocus       = false
tBox.TextWrapped            = true
tBox.TextSize               = 12
tBox.Font                   = Enum.Font.Code
tBox.ZIndex                 = 202

local charLbl = mkLabel(exportFrame, "0 objek | 0 karakter", 8, -26, 300, 22, 10, Color3.fromRGB(140,140,160))
charLbl.Position = UDim2.new(0,8,1,-26)
charLbl.ZIndex   = 205

actionBtn("📋  GENERATE CODE", Color3.fromRGB(0,90,160), lo, function(b)
    local code = "-- PRO TERRAIN CODE | Total: "..#savedData.." objek\nlocal t=workspace.Terrain\n"
    for _, d in pairs(savedData) do
        local line
        if d.isFlat then
            line = ("t:FillBlock(CFrame.new(%.1f,%.1f,%.1f),Vector3.new(%.1f,%.1f,%.1f),Enum.Material.%s)\n"):format(
                d.pos.X,d.pos.Y,d.pos.Z, d.size,d.height,d.size, d.mat.Name)
        elseif d.shape == "Ball" then
            line = ("t:FillBall(Vector3.new(%.1f,%.1f,%.1f),%.1f,Enum.Material.%s)\n"):format(
                d.pos.X,d.pos.Y,d.pos.Z, d.size/2, d.mat.Name)
        else
            line = ("t:FillBlock(CFrame.new(%.1f,%.1f,%.1f),Vector3.new(%.1f,%.1f,%.1f),Enum.Material.%s)\n"):format(
                d.pos.X,d.pos.Y,d.pos.Z, d.size,d.height,d.size, d.mat.Name)
        end
        code = code..line
    end
    tBox.Text    = code
    charLbl.Text = #savedData.." objek | "..#code.." karakter"
    expScroll.CanvasSize = UDim2.new(0,0,0,tBox.TextBounds.Y+50)
    exportFrame.Visible  = true
end); lo=lo+1

secLine(lo); lo=lo+1

-- ══════════════════════════════════════════════════════════
secHeader("✈️ FLY MODE (KHUSUS HP)", lo); lo=lo+1

if isMobile then
    -- Info
    local infoLbl = Instance.new("TextLabel", scrollFrame)
    infoLbl.Size               = UDim2.new(1,-4,0,42)
    infoLbl.BackgroundColor3   = Color3.fromRGB(14,20,40)
    infoLbl.TextColor3         = Color3.fromRGB(180,200,255)
    infoLbl.Font               = Enum.Font.Gotham
    infoLbl.TextSize            = 10
    infoLbl.TextWrapped         = true
    infoLbl.Text               = "Kontrol: Thumbstick kiri = Gerak | Thumbstick kanan = Kamera\nTombol Jump = Naik | Tombol di bawah = Turun"
    infoLbl.LayoutOrder         = lo; lo=lo+1
    corner(infoLbl, 6)

    -- Fly Speed Slider
    local flySpeedSlider = makeSliderRow(scrollFrame, "Kecepatan Fly", flySpeed, 5, 200, 5, lo, function(v) flySpeed = v end); lo=lo+1

    -- Fly ON/OFF
    local flyFrame = Instance.new("Frame", scrollFrame)
    flyFrame.Size             = UDim2.new(1,-4,0,40)
    flyFrame.BackgroundColor3 = Color3.fromRGB(30,30,70)
    flyFrame.LayoutOrder      = lo; lo=lo+1
    corner(flyFrame, 8)
    stroke(flyFrame, Color3.fromRGB(80,80,200), 1.5)

    local flyBtn = Instance.new("TextButton", flyFrame)
    flyBtn.Size               = UDim2.new(1,0,1,0)
    flyBtn.BackgroundTransparency = 1
    flyBtn.Text               = "✈️  FLY: OFF"
    flyBtn.TextSize            = 14
    flyBtn.TextColor3          = Color3.new(1,1,1)
    flyBtn.Font                = Enum.Font.GothamBold

    -- Tombol Turun (saat fly aktif)
    local downBtnFrame = Instance.new("Frame", screenGui)
    downBtnFrame.Size          = UDim2.new(0,80,0,80)
    downBtnFrame.Position      = UDim2.new(1,-100, 1,-180)
    downBtnFrame.BackgroundColor3 = Color3.fromRGB(20,20,50)
    downBtnFrame.Visible       = false
    corner(downBtnFrame, 40)
    stroke(downBtnFrame, Color3.fromRGB(80,80,200), 2)

    local downBtn = Instance.new("TextButton", downBtnFrame)
    downBtn.Size               = UDim2.new(1,0,1,0)
    downBtn.BackgroundTransparency = 1
    downBtn.Text               = "▼\nTURUN"
    downBtn.TextSize            = 14
    downBtn.TextColor3          = Color3.fromRGB(140,200,255)
    downBtn.Font                = Enum.Font.GothamBold

    local isGoingDown = false
    downBtn.InputBegan:Connect(function(inp)
        if inp.UserInputType == Enum.UserInputType.Touch then isGoingDown = true end
    end)
    downBtn.InputEnded:Connect(function(inp)
        if inp.UserInputType == Enum.UserInputType.Touch then isGoingDown = false end
    end)

    -- Fungsi fly menggunakan BodyVelocity + BodyGyro
    local function startFly()
        local char = player.Character
        if not char then return end
        local hrp = char:FindFirstChild("HumanoidRootPart")
        local hum = char:FindFirstChildOfClass("Humanoid")
        if not hrp or not hum then return end

        hum.PlatformStand = true  -- disable default movement

        bodyVel = Instance.new("BodyVelocity", hrp)
        bodyVel.Velocity    = Vector3.new(0,0,0)
        bodyVel.MaxForce    = Vector3.new(1e5,1e5,1e5)

        bodyGyro = Instance.new("BodyGyro", hrp)
        bodyGyro.MaxTorque  = Vector3.new(1e5,1e5,1e5)
        bodyGyro.P          = 1e4
        bodyGyro.CFrame     = hrp.CFrame

        flyConn = RunService.RenderStepped:Connect(function(dt)
            if not flyEnabled then return end
            local char2 = player.Character
            if not char2 then return end
            local hrp2 = char2:FindFirstChild("HumanoidRootPart")
            if not hrp2 then return end

            -- Baca thumbstick kiri (gerak) dari kontroler Roblox mobile
            local moveVec = Vector3.new(0,0,0)

            -- Roblox menyediakan VirtualUser / MoveDirection dari Humanoid
            local hum2 = char2:FindFirstChildOfClass("Humanoid")
            if hum2 then
                local md = hum2.MoveDirection
                -- md adalah arah XZ di dunia
                moveVec = Vector3.new(md.X, 0, md.Z) * flySpeed
            end

            -- Naik = Jump (Roblox native jump button HP)
            if hum2 and hum2.Jump then
                moveVec = moveVec + Vector3.new(0, flySpeed * 0.7, 0)
            end

            -- Turun = tombol custom
            if isGoingDown then
                moveVec = moveVec + Vector3.new(0, -flySpeed * 0.7, 0)
            end

            -- Apply ke BodyVelocity
            if bodyVel and bodyVel.Parent then
                bodyVel.Velocity = moveVec
            end

            -- Rotasi mengikuti kamera (thumbstick kanan = kamera)
            if bodyGyro and bodyGyro.Parent then
                local camCF = camera.CFrame
                bodyGyro.CFrame = CFrame.new(hrp2.Position) * 
                    CFrame.Angles(0, math.atan2(-camCF.LookVector.X, -camCF.LookVector.Z), 0)
            end
        end)
    end

    local function stopFly()
        if flyConn then flyConn:Disconnect(); flyConn = nil end
        local char = player.Character
        if char then
            local hrp = char:FindFirstChild("HumanoidRootPart")
            local hum = char:FindFirstChildOfClass("Humanoid")
            if hrp then
                if bodyVel  then bodyVel:Destroy();  bodyVel  = nil end
                if bodyGyro then bodyGyro:Destroy(); bodyGyro = nil end
            end
            if hum then hum.PlatformStand = false end
        end
    end

    flyBtn.MouseButton1Click:Connect(function()
        flyEnabled = not flyEnabled
        if flyEnabled then
            startFly()
            flyFrame.BackgroundColor3 = Color3.fromRGB(0,100,200)
            flyBtn.Text               = "✈️  FLY: ON ✓  (Aktif)"
            stroke(flyFrame, Color3.fromRGB(0,180,255), 2)
            downBtnFrame.Visible      = true
            statusLbl.Text            = "FLY MODE AKTIF! Thumbstick = gerak, Jump = naik"
        else
            stopFly()
            flyFrame.BackgroundColor3 = Color3.fromRGB(30,30,70)
            flyBtn.Text               = "✈️  FLY: OFF"
            stroke(flyFrame, Color3.fromRGB(80,80,200), 1.5)
            downBtnFrame.Visible      = false
            statusLbl.Text            = "Fly dinonaktifkan."
        end
    end)

    -- Re-apply fly jika karakter respawn
    player.CharacterAdded:Connect(function()
        flyEnabled = false
        flyFrame.BackgroundColor3 = Color3.fromRGB(30,30,70)
        flyBtn.Text               = "✈️  FLY: OFF"
        downBtnFrame.Visible      = false
    end)

else
    -- Non-mobile
    local noMobileLbl = Instance.new("TextLabel", scrollFrame)
    noMobileLbl.Size               = UDim2.new(1,-4,0,36)
    noMobileLbl.BackgroundColor3   = Color3.fromRGB(14,14,14)
    noMobileLbl.TextColor3         = Color3.fromRGB(160,100,100)
    noMobileLbl.Font               = Enum.Font.GothamMedium
    noMobileLbl.TextSize            = 11
    noMobileLbl.TextWrapped         = true
    noMobileLbl.Text               = "⚠️ Fly Mode hanya tersedia di perangkat HP/Tablet."
    noMobileLbl.LayoutOrder         = lo; lo=lo+1
    corner(noMobileLbl, 6)
end

-- ══════════════════════════════════════════════════════════
-- OPEN BUTTON LOGIC
-- ══════════════════════════════════════════════════════════
openBtn.MouseButton1Click:Connect(function()
    mainFrame.Visible = not mainFrame.Visible
    openBtn.Text = mainFrame.Visible and "✕" or "⛏️"
end)

-- ══════════════════════════════════════════════════════════
-- DRAG PREVIEW (saat mode geser)
-- ══════════════════════════════════════════════════════════
local dragPreview = Instance.new("Part")
dragPreview.Anchored   = true
dragPreview.CanCollide = false
dragPreview.Material   = Enum.Material.ForceField
dragPreview.Color      = Color3.fromRGB(80,180,255)
dragPreview.Transparency = 0.6
dragPreview.Parent     = nil

local dpBB   = Instance.new("BillboardGui", dragPreview)
dpBB.Size    = UDim2.new(0,180,0,40)
dpBB.StudsOffset = Vector3.new(0,6,0)
dpBB.AlwaysOnTop = true
local dpTxt  = Instance.new("TextLabel", dpBB)
dpTxt.Size   = UDim2.new(1,0,1,0)
dpTxt.BackgroundTransparency = 1
dpTxt.TextColor3 = Color3.new(1,1,1)
dpTxt.TextStrokeTransparency = 0.3
dpTxt.Font   = Enum.Font.GothamBold
dpTxt.TextSize = 14

local function updateDragPreview(p1, p2)
    if not p1 or not p2 then dragPreview.Parent = nil return end
    local center = (p1+p2)/2
    local sz = Vector3.new(math.abs(p2.X-p1.X)+brushSize, brushHeight, math.abs(p2.Z-p1.Z)+brushSize)
    dragPreview.CFrame = CFrame.new(center)
    dragPreview.Size   = sz
    dragPreview.Parent = workspace
    dpTxt.Text         = ("%.0f×%.0f×%.0f"):format(sz.X, sz.Y, sz.Z)
end

local function createTerrainRegion(p1, p2, mat, shape, sz, ht)
    pcall(function()
        if shape == "Ball" then
            local dist  = (p2-p1).Magnitude
            local steps = math.max(5, math.floor(dist / math.max(1, sz/4)))
            for i=0, steps do
                terrain:FillBall(p1:Lerp(p2, i/steps), sz/2, mat)
            end
        else
            local minX = math.min(p1.X,p2.X); local maxX = math.max(p1.X,p2.X)
            local minZ = math.min(p1.Z,p2.Z); local maxZ = math.max(p1.Z,p2.Z)
            local cen  = Vector3.new((minX+maxX)/2,(p1.Y+p2.Y)/2,(minZ+maxZ)/2)
            terrain:FillBlock(CFrame.new(cen), Vector3.new(maxX-minX+sz,ht,maxZ-minZ+sz), mat)
        end
    end)
end

-- ══════════════════════════════════════════════════════════
-- PAINT / DRAG INPUT
-- ══════════════════════════════════════════════════════════
local function isOnUI(x, y)
    if not mainFrame.Visible then return false end
    local abs = mainFrame.AbsolutePosition
    local siz = mainFrame.AbsoluteSize
    return x >= abs.X and x <= abs.X+siz.X and y >= abs.Y and y <= abs.Y+siz.Y
end

mouse.Button1Down:Connect(function()
    if not paintingEnabled then return end
    if mouse.Target == nil then return end
    if isOnUI(mouse.X, mouse.Y) then return end

    pcall(function()
        if dragMode then
            isDragging    = true
            dragStartPos  = mouse.Hit.Position
            statusLbl.Text = "Geser mouse..."
        else
            if flatTerrainMode then
                local cp = mouse.Hit.Position
                local fp = Vector3.new(cp.X, flatHeight/2, cp.Z)
                table.insert(savedData, {pos=fp,mat=currentMaterial,shape="Block",size=flatSize,height=flatHeight,isFlat=true})
                terrain:FillBlock(CFrame.new(fp), Vector3.new(flatSize,flatHeight,flatSize), currentMaterial)
                statusLbl.Text = "Dataran dibuat | "..#savedData.." total"
            else
                table.insert(savedData, {pos=mouse.Hit.Position,mat=currentMaterial,shape=currentShape,size=brushSize,height=brushHeight,isFlat=false})
                if currentShape == "Ball" then
                    terrain:FillBall(mouse.Hit.Position, brushSize/2, currentMaterial)
                else
                    terrain:FillBlock(CFrame.new(mouse.Hit.Position), Vector3.new(brushSize,brushHeight,brushSize), currentMaterial)
                end
                statusLbl.Text = "Terrain ditambah | "..#savedData.." total"
            end
        end
    end)
end)

mouse.Button1Up:Connect(function()
    if isDragging and dragStartPos and mouse.Target then
        isDragging = false
        local ep   = mouse.Hit.Position
        table.insert(savedData, {pos=dragStartPos,pos2=ep,mat=currentMaterial,shape=currentShape,size=brushSize,height=brushHeight,isFlat=false,isDrag=true})
        createTerrainRegion(dragStartPos, ep, currentMaterial, currentShape, brushSize, brushHeight)
        dragStartPos = nil
        dragPreview.Parent = nil
        statusLbl.Text = "Region dibuat | "..#savedData.." total"
    end
end)

-- Mobile touch untuk paint
if isMobile then
    UserInputService.TouchStarted:Connect(function(inp, gpe)
        if gpe then return end
        if not paintingEnabled then return end
        if isOnUI(inp.Position.X, inp.Position.Y) then return end
        pcall(function()
            local unitRay = camera:ScreenPointToRay(inp.Position.X, inp.Position.Y)
            local result  = workspace:Raycast(unitRay.Origin, unitRay.Direction * 1000)
            if not result then return end
            if flatTerrainMode then
                local fp = Vector3.new(result.Position.X, flatHeight/2, result.Position.Z)
                table.insert(savedData, {pos=fp,mat=currentMaterial,shape="Block",size=flatSize,height=flatHeight,isFlat=true})
                terrain:FillBlock(CFrame.new(fp), Vector3.new(flatSize,flatHeight,flatSize), currentMaterial)
                statusLbl.Text = "Dataran dibuat | "..#savedData
            else
                local p = result.Position
                table.insert(savedData, {pos=p,mat=currentMaterial,shape=currentShape,size=brushSize,height=brushHeight,isFlat=false})
                if currentShape == "Ball" then
                    terrain:FillBall(p, brushSize/2, currentMaterial)
                else
                    terrain:FillBlock(CFrame.new(p), Vector3.new(brushSize,brushHeight,brushSize), currentMaterial)
                end
                statusLbl.Text = "Terrain HP | "..#savedData
            end
        end)
    end)
end

RunService.RenderStepped:Connect(function()
    if isDragging and dragStartPos and mouse.Target then
        updateDragPreview(dragStartPos, mouse.Hit.Position)
    end
end)

-- ════════════════════════════════════════════════════════
print("✓ PRO TERRAIN EDITOR v3 LOADED")
print("✓ Platform: "..(isMobile and "MOBILE (HP)" or "PC"))
print("✓ UNLIMITED MODE | Scroll Panel | Drag & Resize GUI")
if isMobile then print("✓ FLY MODE READY — Thumbstick + Jump") end






