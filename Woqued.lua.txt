--[[
╔══════════════════════════════════════════╗
║        W  O  Q  U  E  D                 ║
║   Murder Mystery 2 Client • v1.0        ║
║   Toggle: [Insert] | Drag W button      ║
╚══════════════════════════════════════════╝
]]

-- ═══════════════════════════════════════════
--                 SERVICES
-- ═══════════════════════════════════════════
local Players          = game:GetService("Players")
local RunService       = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local TweenService     = game:GetService("TweenService")
local LP               = Players.LocalPlayer
local Cam              = workspace.CurrentCamera

-- ═══════════════════════════════════════════
--                  COLORS
-- ═══════════════════════════════════════════
local C = {
    bg       = Color3.fromRGB(14, 14, 20),
    sidebar  = Color3.fromRGB(19, 19, 26),
    content  = Color3.fromRGB(12, 12, 18),
    header   = Color3.fromRGB(21, 21, 29),
    modBg    = Color3.fromRGB(23, 23, 32),
    modHover = Color3.fromRGB(30, 30, 44),
    modOn    = Color3.fromRGB(27, 27, 38),
    catActive= Color3.fromRGB(27, 27, 38),
    catHover = Color3.fromRGB(24, 24, 34),
    text     = Color3.fromRGB(210, 210, 228),
    textDim  = Color3.fromRGB(130, 130, 150),
    textOff  = Color3.fromRGB(70,  70,  88),
    border   = Color3.fromRGB(35,  35,  50),
    accent   = Color3.fromRGB(255, 255, 255),
    blue     = Color3.fromRGB(80,  140, 255),
    red      = Color3.fromRGB(255, 70,  70),
    green    = Color3.fromRGB(80,  220, 120),
    orange   = Color3.fromRGB(255, 165, 60),
    purple   = Color3.fromRGB(180, 100, 255),
    teal     = Color3.fromRGB(80,  200, 200),
    floatBg  = Color3.fromRGB(26,  26,  36),
}

-- ═══════════════════════════════════════════
--             MODULE DEFINITIONS
-- ═══════════════════════════════════════════
local MODULES = {
    {id="silentaim",   name="SilentAim",   cat="Combat",   enabled=false,
     col=C.blue,   hasSet=true,  set={fov=150, showFov=true}},
    {id="antiaim",     name="AntiAim",     cat="Combat",   enabled=false,
     col=C.red,    hasSet=false},
    {id="autoshoot",   name="AutoShoot",   cat="Combat",   enabled=false,
     col=C.orange, hasSet=false},
    {id="bunnyhop",    name="BunnyHop",    cat="Movement", enabled=false,
     col=C.green,  hasSet=false},
    {id="esp",         name="ESP",         cat="Visuals",  enabled=false,
     col=C.accent, hasSet=false},
    {id="shoottrails", name="ShootTrails", cat="Visuals",  enabled=false,
     col=C.purple, hasSet=false},
    {id="cspov",       name="CsPov",       cat="Visuals",  enabled=false,
     col=C.teal,   hasSet=false},
    {id="watermark",   name="Watermark",   cat="HUD",      enabled=true,
     col=C.accent, hasSet=false},
    {id="bindshud",    name="BindsHUD",    cat="HUD",      enabled=true,
     col=C.accent, hasSet=false},
    {id="speedhud",    name="SpeedHUD",    cat="HUD",      enabled=true,
     col=C.green,  hasSet=false},
}
local MOD = {}
for _, m in ipairs(MODULES) do MOD[m.id] = m end

local CATS     = {"Combat", "Movement", "Visuals", "HUD"}
local CAT_ICON = {Combat="⚔", Movement="⇾", Visuals="◈", HUD="▦"}
local CAT_DESC = {
    Combat  = "Боевые функции",
    Movement= "Управление движением",
    Visuals = "Визуальные эффекты",
    HUD     = "Интерфейс и дисплей",
}

-- ═══════════════════════════════════════════
--                  STATE
-- ═══════════════════════════════════════════
local S = {
    open      = false,
    animating = false,
    cat       = "Combat",
    -- AntiAim
    aaAngle   = 0,
    aaPaused  = false,
    -- SilentAim
    saTarget  = nil,
    -- ShootTrails
    trails    = {},
    -- CsPov
    vmBobT    = 0,
    inspecting= false,
    -- FPS
    fpsCount  = 0,
    fpsTimer  = tick(),
    fps       = 0,
}

-- Drawing API availability
local HAS_DRAW = false
pcall(function()
    local t = Drawing.new("Text"); t:Remove()
    HAS_DRAW = true
end)

-- ═══════════════════════════════════════════
--                UTILITIES
-- ═══════════════════════════════════════════
local function newI(cls, props, par)
    local o = Instance.new(cls)
    for k, v in pairs(props or {}) do o[k] = v end
    if par then o.Parent = par end
    return o
end

local function corner(p, r)
    return newI("UICorner", {CornerRadius = UDim.new(0, r or 6)}, p)
end

local function stroke(p, col, th)
    return newI("UIStroke", {
        Color = col or C.border,
        Thickness = th or 1,
        ApplyStrokeMode = Enum.ApplyStrokeMode.Border,
    }, p)
end

local function tw(o, t, props, es, ed)
    TweenService:Create(o, TweenInfo.new(
        t or 0.15,
        es or Enum.EasingStyle.Quad,
        ed or Enum.EasingDirection.Out
    ), props):Play()
end

local function getChar() return LP.Character end
local function getHRP()
    local c = getChar(); return c and c:FindFirstChild("HumanoidRootPart")
end
local function getHum()
    local c = getChar(); return c and c:FindFirstChildOfClass("Humanoid")
end

local function isMurd(pl)
    local c = pl.Character
    if not c then return false end
    if c:FindFirstChild("Knife") then return true end
    local b = pl:FindFirstChild("Backpack")
    return b and b:FindFirstChild("Knife") ~= nil
end

local function isSher(pl)
    local c = pl.Character
    if not c then return false end
    if c:FindFirstChild("Gun") then return true end
    local b = pl:FindFirstChild("Backpack")
    return b and b:FindFirstChild("Gun") ~= nil
end

-- ═══════════════════════════════════════════
--           SCREEN GUI  +  FLOAT BUTTON
-- ═══════════════════════════════════════════
local SG = newI("ScreenGui", {
    Name = "Woqued", ResetOnSpawn = false,
    DisplayOrder = 999, IgnoreGuiInset = true,
})
pcall(function() SG.Parent = game:GetService("CoreGui") end)
if not SG.Parent then SG.Parent = LP.PlayerGui end

local floatBtn = newI("TextButton", {
    Size            = UDim2.new(0, 46, 0, 46),
    Position        = UDim2.new(0, 24, 0.5, -23),
    BackgroundColor3= C.floatBg,
    BorderSizePixel = 0,
    Text            = "W",
    TextColor3      = C.text,
    TextSize        = 22,
    Font            = Enum.Font.GothamBold,
    AutoButtonColor = false,
    Active          = true,
}, SG)
corner(floatBtn, 10)
stroke(floatBtn, C.border)
floatBtn.MouseEnter:Connect(function()
    tw(floatBtn, 0.12, {BackgroundColor3 = Color3.fromRGB(38, 38, 52)})
end)
floatBtn.MouseLeave:Connect(function()
    tw(floatBtn, 0.12, {BackgroundColor3 = C.floatBg})
end)

-- ═══════════════════════════════════════════
--               MAIN WINDOW
-- ═══════════════════════════════════════════
local WW, WH, SW = 710, 476, 212
local toggleMenu   -- forward declare

local win = newI("Frame", {
    Size             = UDim2.new(0, WW, 0, WH),
    Position         = UDim2.new(0.5, -WW/2, 0.5, -WH/2),
    BackgroundColor3 = C.bg,
    BorderSizePixel  = 0,
    Visible          = false,
    ClipsDescendants = true,
}, SG)
corner(win, 10)
stroke(win, C.border)

-- ── Header Bar ──────────────────────────────
local hdr = newI("Frame", {
    Size             = UDim2.new(1, 0, 0, 52),
    BackgroundColor3 = C.header,
    BorderSizePixel  = 0,
}, win)
newI("Frame", {
    Size             = UDim2.new(1, 0, 0, 1),
    Position         = UDim2.new(0, 0, 1, -1),
    BackgroundColor3 = C.border,
    BorderSizePixel  = 0,
}, hdr)

-- Logo box
local logoBox = newI("TextLabel", {
    Size             = UDim2.new(0, 34, 0, 34),
    Position         = UDim2.new(0, 12, 0.5, -17),
    BackgroundColor3 = C.accent,
    BorderSizePixel  = 0,
    Text             = "W",
    TextColor3       = C.bg,
    TextSize         = 18,
    Font             = Enum.Font.GothamBold,
    TextXAlignment   = Enum.TextXAlignment.Center,
    TextYAlignment   = Enum.TextYAlignment.Center,
}, hdr)
corner(logoBox, 7)

newI("TextLabel", {
    Size           = UDim2.new(0, 120, 0, 19),
    Position       = UDim2.new(0, 52, 0.5, -19),
    BackgroundTransparency = 1,
    Text           = "Woqued",
    TextColor3     = C.text,
    TextSize       = 14,
    Font           = Enum.Font.GothamBold,
    TextXAlignment = Enum.TextXAlignment.Left,
}, hdr)
newI("TextLabel", {
    Size           = UDim2.new(0, 120, 0, 14),
    Position       = UDim2.new(0, 52, 0.5, 2),
    BackgroundTransparency = 1,
    Text           = "MM2  ·  v1.0",
    TextColor3     = C.textDim,
    TextSize       = 10,
    Font           = Enum.Font.Gotham,
    TextXAlignment = Enum.TextXAlignment.Left,
}, hdr)

-- Active category name + description (center of header)
local hdrCatTitle = newI("TextLabel", {
    Size           = UDim2.new(0, 280, 0, 19),
    Position       = UDim2.new(0, SW + 14, 0.5, -19),
    BackgroundTransparency = 1,
    Text           = "Combat",
    TextColor3     = C.text,
    TextSize       = 14,
    Font           = Enum.Font.GothamBold,
    TextXAlignment = Enum.TextXAlignment.Left,
}, hdr)
local hdrCatDesc = newI("TextLabel", {
    Size           = UDim2.new(0, 280, 0, 13),
    Position       = UDim2.new(0, SW + 14, 0.5, 2),
    BackgroundTransparency = 1,
    Text           = CAT_DESC["Combat"],
    TextColor3     = C.textDim,
    TextSize       = 10,
    Font           = Enum.Font.Gotham,
    TextXAlignment = Enum.TextXAlignment.Left,
}, hdr)

-- Username top-right
newI("TextLabel", {
    Size               = UDim2.new(0, 180, 1, 0),
    Position           = UDim2.new(1, -188, 0, 0),
    BackgroundTransparency = 1,
    Text               = LP.Name .. "  ·  Woqued",
    TextColor3         = C.textDim,
    TextSize           = 10,
    Font               = Enum.Font.Gotham,
    TextXAlignment     = Enum.TextXAlignment.Right,
    TextYAlignment     = Enum.TextYAlignment.Center,
}, hdr)

-- ── Body (sidebar + content) ─────────────────
local body = newI("Frame", {
    Size               = UDim2.new(1, 0, 1, -52),
    Position           = UDim2.new(0, 0, 0, 52),
    BackgroundTransparency = 1,
}, win)

-- ═══════════════════════════════════════════
--                 SIDEBAR
-- ═══════════════════════════════════════════
local sidebar = newI("Frame", {
    Size             = UDim2.new(0, SW, 1, 0),
    BackgroundColor3 = C.sidebar,
    BorderSizePixel  = 0,
}, body)
-- Right border
newI("Frame", {
    Size             = UDim2.new(0, 1, 1, 0),
    Position         = UDim2.new(1, -1, 0, 0),
    BackgroundColor3 = C.border,
    BorderSizePixel  = 0,
}, sidebar)

-- Scroll list for categories
local catScroll = newI("ScrollingFrame", {
    Size                  = UDim2.new(1, 0, 1, -50),
    BackgroundTransparency= 1,
    BorderSizePixel       = 0,
    ScrollBarThickness    = 2,
    ScrollBarImageColor3  = C.border,
    AutomaticCanvasSize   = Enum.AutomaticSize.Y,
    CanvasSize            = UDim2.new(0, 0, 0, 0),
}, sidebar)
newI("UIListLayout", {SortOrder = Enum.SortOrder.LayoutOrder, Padding = UDim.new(0, 2)}, catScroll)
newI("UIPadding", {
    PaddingTop   = UDim.new(0, 8),
    PaddingLeft  = UDim.new(0, 6),
    PaddingRight = UDim.new(0, 6),
    PaddingBottom= UDim.new(0, 4),
}, catScroll)

-- Search bar (bottom of sidebar)
local searchBar = newI("Frame", {
    Size             = UDim2.new(1, 0, 0, 46),
    Position         = UDim2.new(0, 0, 1, -46),
    BackgroundColor3 = C.sidebar,
    BorderSizePixel  = 0,
}, sidebar)
newI("Frame", {
    Size = UDim2.new(1, 0, 0, 1), BackgroundColor3 = C.border, BorderSizePixel = 0,
}, searchBar)
local searchWrap = newI("Frame", {
    Size             = UDim2.new(1, -16, 0, 28),
    Position         = UDim2.new(0, 8, 0.5, -14),
    BackgroundColor3 = C.modBg,
    BorderSizePixel  = 0,
}, searchBar)
corner(searchWrap, 6)
stroke(searchWrap, C.border)
newI("TextLabel", {
    Size               = UDim2.new(0, 20, 1, 0),
    Position           = UDim2.new(0, 4, 0, 0),
    BackgroundTransparency = 1,
    Text               = "⌕",
    TextSize           = 14,
    TextColor3         = C.textOff,
    Font               = Enum.Font.GothamBold,
}, searchWrap)
local searchBox = newI("TextBox", {
    Size               = UDim2.new(1, -26, 1, 0),
    Position           = UDim2.new(0, 22, 0, 0),
    BackgroundTransparency = 1,
    PlaceholderText    = "Search...",
    PlaceholderColor3  = C.textOff,
    Text               = "",
    TextColor3         = C.text,
    TextSize           = 12,
    Font               = Enum.Font.Gotham,
    TextXAlignment     = Enum.TextXAlignment.Left,
    ClearTextOnFocus   = false,
}, searchWrap)

-- ═══════════════════════════════════════════
--               CONTENT AREA
-- ═══════════════════════════════════════════
local contentArea = newI("Frame", {
    Size             = UDim2.new(1, -SW, 1, 0),
    Position         = UDim2.new(0, SW, 0, 0),
    BackgroundColor3 = C.content,
    BorderSizePixel  = 0,
}, body)

local contentScroll = newI("ScrollingFrame", {
    Size                  = UDim2.new(1, 0, 1, 0),
    BackgroundTransparency= 1,
    BorderSizePixel       = 0,
    ScrollBarThickness    = 3,
    ScrollBarImageColor3  = C.border,
    AutomaticCanvasSize   = Enum.AutomaticSize.Y,
    CanvasSize            = UDim2.new(0, 0, 0, 0),
}, contentArea)
newI("UIPadding", {
    PaddingTop    = UDim.new(0, 10),
    PaddingLeft   = UDim.new(0, 10),
    PaddingRight  = UDim.new(0, 10),
    PaddingBottom = UDim.new(0, 10),
}, contentScroll)
newI("UIGridLayout", {
    CellSize             = UDim2.new(0.5, -6, 0, 40),
    CellPadding          = UDim2.new(0, 10, 0, 6),
    SortOrder            = Enum.SortOrder.LayoutOrder,
    HorizontalAlignment  = Enum.HorizontalAlignment.Left,
}, contentScroll)

-- ═══════════════════════════════════════════
--            CATEGORY BUTTONS
-- ═══════════════════════════════════════════
local catBtns   = {}  -- cat -> {btn, border, icon, lbl}
local modBtns   = {}  -- id  -> {btn, bar, lbl}
local settingsPop = nil
local buildModules  -- forward declare
local onToggle      -- forward declare

local function switchCat(newCat)
    if newCat == S.cat then return end
    local old = catBtns[S.cat]
    if old then
        tw(old.btn, 0.12, {BackgroundColor3 = C.sidebar})
        tw(old.border, 0.12, {BackgroundTransparency = 1})
        old.lbl.Font      = Enum.Font.Gotham
        old.lbl.TextColor3= C.textDim
        old.icon.TextColor3= C.textDim
    end
    S.cat = newCat
    local n = catBtns[newCat]
    if n then
        tw(n.btn, 0.12, {BackgroundColor3 = C.catActive})
        tw(n.border, 0.12, {BackgroundTransparency = 0})
        n.lbl.Font       = Enum.Font.GothamBold
        n.lbl.TextColor3 = C.text
        n.icon.TextColor3= C.text
    end
    hdrCatTitle.Text = newCat
    hdrCatDesc.Text  = CAT_DESC[newCat] or ""
    buildModules()
end

for i, cat in ipairs(CATS) do
    local isAct = (cat == S.cat)
    local btn = newI("TextButton", {
        Size             = UDim2.new(1, 0, 0, 42),
        BackgroundColor3 = isAct and C.catActive or C.sidebar,
        BorderSizePixel  = 0,
        Text             = "",
        AutoButtonColor  = false,
        LayoutOrder      = i,
    }, catScroll)
    corner(btn, 6)

    local bord = newI("Frame", {
        Size             = UDim2.new(0, 3, 0.6, 0),
        Position         = UDim2.new(0, 0, 0.2, 0),
        BackgroundColor3 = C.accent,
        BorderSizePixel  = 0,
        BackgroundTransparency = isAct and 0 or 1,
    }, btn)
    corner(bord, 2)

    local icon = newI("TextLabel", {
        Size               = UDim2.new(0, 22, 1, 0),
        Position           = UDim2.new(0, 12, 0, 0),
        BackgroundTransparency = 1,
        Text               = CAT_ICON[cat] or "•",
        TextColor3         = isAct and C.text or C.textDim,
        TextSize           = 14,
        Font               = Enum.Font.GothamBold,
    }, btn)

    local lbl = newI("TextLabel", {
        Size               = UDim2.new(1, -42, 1, 0),
        Position           = UDim2.new(0, 38, 0, 0),
        BackgroundTransparency = 1,
        Text               = cat,
        TextColor3         = isAct and C.text or C.textDim,
        TextSize           = 13,
        Font               = isAct and Enum.Font.GothamBold or Enum.Font.Gotham,
        TextXAlignment     = Enum.TextXAlignment.Left,
    }, btn)

    catBtns[cat] = {btn = btn, border = bord, icon = icon, lbl = lbl}
    btn.MouseButton1Click:Connect(function() switchCat(cat) end)
    btn.MouseEnter:Connect(function()
        if cat ~= S.cat then tw(btn, 0.1, {BackgroundColor3 = C.catHover}) end
    end)
    btn.MouseLeave:Connect(function()
        if cat ~= S.cat then tw(btn, 0.1, {BackgroundColor3 = C.sidebar}) end
    end)
end

-- ═══════════════════════════════════════════
--     SETTINGS POPUP  (SilentAim FOV etc.)
-- ═══════════════════════════════════════════
local function openSettingsFor(m)
    if settingsPop then settingsPop:Destroy() end
    if m.id ~= "silentaim" then return end

    settingsPop = newI("Frame", {
        Size             = UDim2.new(0, 234, 0, 116),
        Position         = UDim2.new(0.5, -117, 0.5, -58),
        BackgroundColor3 = C.header,
        BorderSizePixel  = 0,
        ZIndex           = 10,
    }, contentArea)
    corner(settingsPop, 8)
    stroke(settingsPop, C.border)

    newI("TextLabel", {
        Size               = UDim2.new(1, -32, 0, 30),
        Position           = UDim2.new(0, 12, 0, 0),
        BackgroundTransparency = 1,
        Text               = "SilentAim Settings",
        TextColor3         = C.text,
        TextSize           = 12,
        Font               = Enum.Font.GothamBold,
        TextXAlignment     = Enum.TextXAlignment.Left,
        TextYAlignment     = Enum.TextYAlignment.Center,
        ZIndex             = 10,
    }, settingsPop)

    -- Close button
    local closeBtn = newI("TextButton", {
        Size             = UDim2.new(0, 22, 0, 22),
        Position         = UDim2.new(1, -26, 0, 4),
        BackgroundColor3 = C.red,
        BorderSizePixel  = 0,
        Text             = "✕",
        TextColor3       = C.accent,
        TextSize         = 11,
        Font             = Enum.Font.GothamBold,
        AutoButtonColor  = false,
        ZIndex           = 10,
    }, settingsPop)
    corner(closeBtn, 5)
    closeBtn.MouseButton1Click:Connect(function()
        settingsPop:Destroy(); settingsPop = nil
    end)

    -- Separator
    newI("Frame", {
        Size = UDim2.new(1, -24, 0, 1), Position = UDim2.new(0, 12, 0, 30),
        BackgroundColor3 = C.border, BorderSizePixel = 0, ZIndex = 10,
    }, settingsPop)

    -- FOV label
    newI("TextLabel", {
        Size               = UDim2.new(0, 90, 0, 24),
        Position           = UDim2.new(0, 12, 0, 38),
        BackgroundTransparency = 1,
        Text               = "FOV Radius",
        TextColor3         = C.textDim,
        TextSize           = 11,
        Font               = Enum.Font.Gotham,
        TextXAlignment     = Enum.TextXAlignment.Left,
        ZIndex             = 10,
    }, settingsPop)

    local fovVal = newI("TextLabel", {
        Size               = UDim2.new(0, 44, 0, 24),
        Position           = UDim2.new(0, 112, 0, 38),
        BackgroundTransparency = 1,
        Text               = tostring(m.set.fov),
        TextColor3         = C.text,
        TextSize           = 12,
        Font               = Enum.Font.GothamBold,
        TextXAlignment     = Enum.TextXAlignment.Center,
        ZIndex             = 10,
    }, settingsPop)

    local function mkAdjBtn(label, xOff, action)
        local b = newI("TextButton", {
            Size             = UDim2.new(0, 26, 0, 24),
            Position         = UDim2.new(0, xOff, 0, 38),
            BackgroundColor3 = C.modBg,
            BorderSizePixel  = 0,
            Text             = label,
            TextColor3       = C.text,
            TextSize         = 15,
            Font             = Enum.Font.GothamBold,
            AutoButtonColor  = false,
            ZIndex           = 10,
        }, settingsPop)
        corner(b, 5)
        b.MouseButton1Click:Connect(function()
            action()
            fovVal.Text = tostring(m.set.fov)
            if HAS_DRAW and fovCircle then fovCircle.Radius = m.set.fov end
        end)
        return b
    end
    mkAdjBtn("−", 158, function() m.set.fov = math.max(30,  m.set.fov - 10) end)
    mkAdjBtn("+", 188, function() m.set.fov = math.min(500, m.set.fov + 10) end)

    -- Show FOV Circle toggle
    newI("TextLabel", {
        Size               = UDim2.new(0, 120, 0, 24),
        Position           = UDim2.new(0, 12, 0, 70),
        BackgroundTransparency = 1,
        Text               = "Show FOV Circle",
        TextColor3         = C.textDim,
        TextSize           = 11,
        Font               = Enum.Font.Gotham,
        TextXAlignment     = Enum.TextXAlignment.Left,
        ZIndex             = 10,
    }, settingsPop)

    local fovToggle = newI("TextButton", {
        Size             = UDim2.new(0, 50, 0, 22),
        Position         = UDim2.new(1, -62, 0, 71),
        BackgroundColor3 = m.set.showFov and C.blue or C.modBg,
        BorderSizePixel  = 0,
        Text             = m.set.showFov and "ON" or "OFF",
        TextColor3       = C.accent,
        TextSize         = 11,
        Font             = Enum.Font.GothamBold,
        AutoButtonColor  = false,
        ZIndex           = 10,
    }, settingsPop)
    corner(fovToggle, 5)
    fovToggle.MouseButton1Click:Connect(function()
        m.set.showFov = not m.set.showFov
        fovToggle.Text = m.set.showFov and "ON" or "OFF"
        tw(fovToggle, 0.12, {BackgroundColor3 = m.set.showFov and C.blue or C.modBg})
    end)
end

-- ═══════════════════════════════════════════
--             MODULE BUTTONS
-- ═══════════════════════════════════════════
local function refreshModBtn(m)
    local r = modBtns[m.id]; if not r then return end
    local on = m.enabled
    tw(r.btn, 0.12, {BackgroundColor3 = on and C.modOn or C.modBg})
    tw(r.bar, 0.12, {BackgroundTransparency = on and 0 or 1})
    r.lbl.TextColor3 = on and C.text or C.textDim
    r.lbl.Font       = on and Enum.Font.GothamBold or Enum.Font.Gotham
end

buildModules = function()
    -- Clear existing buttons
    for _, r in pairs(modBtns) do
        if r.btn and r.btn.Parent then r.btn:Destroy() end
    end
    modBtns = {}
    if settingsPop then settingsPop:Destroy(); settingsPop = nil end

    local n = 0
    for _, m in ipairs(MODULES) do
        if m.cat == S.cat then
            n = n + 1
            local on = m.enabled

            local btn = newI("TextButton", {
                Size             = UDim2.new(0, 1, 0, 1),
                BackgroundColor3 = on and C.modOn or C.modBg,
                BorderSizePixel  = 0,
                Text             = "",
                AutoButtonColor  = false,
                LayoutOrder      = n,
                ClipsDescendants = true,
            }, contentScroll)
            corner(btn, 7)

            -- Left colored bar (indicates active)
            local bar = newI("Frame", {
                Size             = UDim2.new(0, 3, 0.65, 0),
                Position         = UDim2.new(0, 0, 0.175, 0),
                BackgroundColor3 = m.col or C.accent,
                BorderSizePixel  = 0,
                BackgroundTransparency = on and 0 or 1,
            }, btn)
            corner(bar, 2)

            -- Module name
            local lbl = newI("TextLabel", {
                Size               = UDim2.new(1, m.hasSet and -58 or -30, 1, 0),
                Position           = UDim2.new(0, 14, 0, 0),
                BackgroundTransparency = 1,
                Text               = m.name,
                TextColor3         = on and C.text or C.textDim,
                TextSize           = 12,
                Font               = on and Enum.Font.GothamBold or Enum.Font.Gotham,
                TextXAlignment     = Enum.TextXAlignment.Left,
                TextYAlignment     = Enum.TextYAlignment.Center,
            }, btn)

            -- Gear button (settings)
            if m.hasSet then
                local gear = newI("TextButton", {
                    Size             = UDim2.new(0, 26, 0, 26),
                    Position         = UDim2.new(1, -30, 0.5, -13),
                    BackgroundColor3 = C.bg,
                    BorderSizePixel  = 0,
                    Text             = "⚙",
                    TextColor3       = C.textDim,
                    TextSize         = 12,
                    Font             = Enum.Font.GothamBold,
                    AutoButtonColor  = false,
                }, btn)
                corner(gear, 5)
                local mRef = m
                gear.MouseButton1Click:Connect(function() openSettingsFor(mRef) end)
                gear.MouseEnter:Connect(function() tw(gear, 0.1, {TextColor3 = C.text}) end)
                gear.MouseLeave:Connect(function() tw(gear, 0.1, {TextColor3 = C.textDim}) end)
            end

            modBtns[m.id] = {btn = btn, bar = bar, lbl = lbl}

            local mRef = m
            btn.MouseButton1Click:Connect(function()
                mRef.enabled = not mRef.enabled
                refreshModBtn(mRef)
                onToggle(mRef)
            end)
            btn.MouseEnter:Connect(function()
                tw(btn, 0.1, {BackgroundColor3 = C.modHover})
            end)
            btn.MouseLeave:Connect(function()
                tw(btn, 0.1, {BackgroundColor3 = mRef.enabled and C.modOn or C.modBg})
            end)
        end
    end
end
buildModules()

-- Search filter
searchBox:GetPropertyChangedSignal("Text"):Connect(function()
    local q = searchBox.Text:lower()
    for _, m in ipairs(MODULES) do
        if m.cat == S.cat then
            local r = modBtns[m.id]
            if r and r.btn then
                r.btn.Visible = (q == "" or m.name:lower():find(q, 1, true) ~= nil)
            end
        end
    end
end)

-- ═══════════════════════════════════════════
--                 HUD FRAMES
-- ═══════════════════════════════════════════

-- ── Watermark ───────────────────────────────
local wmFrame = newI("Frame", {
    Size             = UDim2.new(0, 220, 0, 56),
    Position         = UDim2.new(0, 10, 0, 10),
    BackgroundColor3 = C.bg,
    BorderSizePixel  = 0,
    Visible          = MOD.watermark.enabled,
}, SG)
corner(wmFrame, 8)
stroke(wmFrame, C.border)
-- Blue left accent
local wmAccent = newI("Frame", {
    Size             = UDim2.new(0, 3, 0.7, 0),
    Position         = UDim2.new(0, 0, 0.15, 0),
    BackgroundColor3 = C.blue,
    BorderSizePixel  = 0,
}, wmFrame)
corner(wmAccent, 2)
local wmTitleLbl = newI("TextLabel", {
    Size               = UDim2.new(1, -14, 0, 22),
    Position           = UDim2.new(0, 12, 0, 8),
    BackgroundTransparency = 1,
    Text               = "Woqued",
    TextColor3         = C.text,
    TextSize           = 13,
    Font               = Enum.Font.GothamBold,
    TextXAlignment     = Enum.TextXAlignment.Left,
}, wmFrame)
local wmStatsLbl = newI("TextLabel", {
    Size               = UDim2.new(1, -14, 0, 14),
    Position           = UDim2.new(0, 12, 0, 30),
    BackgroundTransparency = 1,
    Text               = "0 FPS  |  0ms",
    TextColor3         = C.textDim,
    TextSize           = 10,
    Font               = Enum.Font.Gotham,
    TextXAlignment     = Enum.TextXAlignment.Left,
}, wmFrame)

-- ── Speed HUD ───────────────────────────────
local speedFrame = newI("Frame", {
    Size             = UDim2.new(0, 110, 0, 52),
    Position         = UDim2.new(0.5, -55, 1, -70),
    BackgroundColor3 = C.bg,
    BorderSizePixel  = 0,
    Visible          = MOD.speedhud.enabled,
}, SG)
corner(speedFrame, 8)
stroke(speedFrame, C.border)
local speedVal = newI("TextLabel", {
    Size               = UDim2.new(1, 0, 0.6, 0),
    BackgroundTransparency = 1,
    Text               = "0",
    TextColor3         = C.green,
    TextSize           = 24,
    Font               = Enum.Font.GothamBold,
    TextXAlignment     = Enum.TextXAlignment.Center,
    TextYAlignment     = Enum.TextYAlignment.Bottom,
}, speedFrame)
newI("TextLabel", {
    Size               = UDim2.new(1, 0, 0.4, 0),
    Position           = UDim2.new(0, 0, 0.6, 0),
    BackgroundTransparency = 1,
    Text               = "SPEED  ups",
    TextColor3         = C.textDim,
    TextSize           = 9,
    Font               = Enum.Font.Gotham,
    TextXAlignment     = Enum.TextXAlignment.Center,
    TextYAlignment     = Enum.TextYAlignment.Center,
}, speedFrame)

-- ── Binds HUD ───────────────────────────────
local bindsFrame = newI("Frame", {
    Size             = UDim2.new(0, 160, 0, 36),
    Position         = UDim2.new(1, -170, 0, 10),
    BackgroundColor3 = C.bg,
    BorderSizePixel  = 0,
    Visible          = MOD.bindshud.enabled,
}, SG)
corner(bindsFrame, 8)
stroke(bindsFrame, C.border)
newI("TextLabel", {
    Size               = UDim2.new(1, -16, 0, 18),
    Position           = UDim2.new(0, 8, 0, 4),
    BackgroundTransparency = 1,
    Text               = "▸ ACTIVE",
    TextColor3         = C.textDim,
    TextSize           = 9,
    Font               = Enum.Font.GothamBold,
    TextXAlignment     = Enum.TextXAlignment.Left,
}, bindsFrame)
local bindsListLbl = newI("TextLabel", {
    Size               = UDim2.new(1, -16, 0, 0),
    Position           = UDim2.new(0, 8, 0, 20),
    BackgroundTransparency = 1,
    Text               = "No active modules",
    TextColor3         = C.text,
    TextSize           = 10,
    Font               = Enum.Font.Gotham,
    TextXAlignment     = Enum.TextXAlignment.Left,
    TextYAlignment     = Enum.TextYAlignment.Top,
    AutomaticSize      = Enum.AutomaticSize.Y,
    TextWrapped        = false,
}, bindsFrame)

local function updateBindsHUD()
    if not MOD.bindshud.enabled then
        bindsFrame.Visible = false; return
    end
    bindsFrame.Visible = true
    local lines = {}
    for _, m in ipairs(MODULES) do
        if m.enabled and m.id ~= "watermark" and m.id ~= "bindshud" and m.id ~= "speedhud" then
            table.insert(lines, "· " .. m.name)
        end
    end
    bindsListLbl.Text = #lines > 0 and table.concat(lines, "\n") or "No active modules"
    task.defer(function()
        local h = bindsListLbl.TextBounds.Y
        bindsFrame.Size = UDim2.new(0, 160, 0, 26 + h + 6)
    end)
end
updateBindsHUD()

-- ── Draggable helper for HUD frames ─────────
local function makeDraggable(frame)
    local d, s, p = false, nil, nil
    frame.InputBegan:Connect(function(inp)
        if inp.UserInputType == Enum.UserInputType.MouseButton1 then
            d = true; s = inp.Position; p = frame.Position
        end
    end)
    UserInputService.InputEnded:Connect(function(inp)
        if inp.UserInputType == Enum.UserInputType.MouseButton1 then d = false end
    end)
    UserInputService.InputChanged:Connect(function(inp)
        if d and inp.UserInputType == Enum.UserInputType.MouseMovement then
            local delta = inp.Position - s
            frame.Position = UDim2.new(
                p.X.Scale, p.X.Offset + delta.X,
                p.Y.Scale, p.Y.Offset + delta.Y
            )
        end
    end)
end
makeDraggable(wmFrame)
makeDraggable(speedFrame)
makeDraggable(bindsFrame)

-- ═══════════════════════════════════════════
--           DRAWING LAYER  (ESP / FOV)
-- ═══════════════════════════════════════════
local fovCircle = nil
local espData   = {}

if HAS_DRAW then
    fovCircle = Drawing.new("Circle")
    fovCircle.Radius   = MOD.silentaim.set.fov
    fovCircle.Thickness= 1.5
    fovCircle.Filled   = false
    fovCircle.Visible  = false
    fovCircle.Color    = Color3.fromRGB(255, 255, 255)
    fovCircle.NumSides = 60
end

local function getOrCreateESP(pl)
    if espData[pl] then return espData[pl] end
    if not HAS_DRAW then return nil end
    local d = {}
    d.box           = Drawing.new("Square")
    d.box.Filled    = false
    d.box.Thickness = 1.5
    d.box.Transparency = 1
    d.box.Visible   = false
    d.name          = Drawing.new("Text")
    d.name.Size     = 12
    d.name.Center   = true
    d.name.Outline  = true
    d.name.OutlineColor = Color3.fromRGB(0, 0, 0)
    d.name.Font     = 2
    d.name.Visible  = false
    espData[pl]     = d
    return d
end

local function cleanESP(pl)
    local d = espData[pl]; if not d then return end
    if d.box  then d.box:Remove()  end
    if d.name then d.name:Remove() end
    espData[pl] = nil
end
Players.PlayerRemoving:Connect(cleanESP)

-- ═══════════════════════════════════════════
--             SHOOT TRAILS
-- ═══════════════════════════════════════════
local function addTrail(from2D, to2D)
    if not MOD.shoottrails.enabled or not HAS_DRAW then return end
    local t = { startTime = tick(), duration = 0.65, lines = {} }
    -- Four overlapping lines: core + three glow layers
    local cfg = {
        {col=Color3.fromRGB(255,255,255), thick=1.2, alpha=1.00},
        {col=Color3.fromRGB(255,120,120), thick=3.0, alpha=0.45},
        {col=Color3.fromRGB(255, 60, 60), thick=6.0, alpha=0.18},
        {col=Color3.fromRGB(255, 20, 20), thick=11,  alpha=0.07},
    }
    for _, c in ipairs(cfg) do
        local l        = Drawing.new("Line")
        l.From         = from2D
        l.To           = to2D
        l.Color        = c.col
        l.Thickness    = c.thick
        l.Transparency = c.alpha
        l.Visible      = true
        table.insert(t.lines, l)
    end
    table.insert(S.trails, t)
end

local function updateTrails()
    if not HAS_DRAW then return end
    local now, i = tick(), 1
    local alphas = {1.00, 0.45, 0.18, 0.07}
    while i <= #S.trails do
        local t = S.trails[i]
        local age = now - t.startTime
        if age >= t.duration then
            for _, l in ipairs(t.lines) do l:Remove() end
            table.remove(S.trails, i)
        else
            local fade = 1 - (age / t.duration)
            for j, l in ipairs(t.lines) do
                l.Transparency = alphas[j] * fade
            end
            i = i + 1
        end
    end
end

-- ═══════════════════════════════════════════
--          CS POV  VIEWMODEL
-- ═══════════════════════════════════════════
local vmFrame = newI("ViewportFrame", {
    Size               = UDim2.new(1, 0, 0.65, 0),
    Position           = UDim2.new(0, 0, 0.35, 0),
    BackgroundTransparency = 1,
    LightDirection     = Vector3.new(-0.6, -1, -1),
    ImageColor3        = Color3.fromRGB(220, 220, 220),
    ImageTransparency  = 0,
    Visible            = false,
}, SG)
local vmCam = Instance.new("Camera")
vmCam.FieldOfView = 70
vmCam.CFrame      = CFrame.new(0, 0, 3)
vmCam.Parent      = vmFrame
vmFrame.CurrentCamera = vmCam

local vmModel = Instance.new("Model")
vmModel.Parent = vmFrame

local function mkVP(sz, col, mat)
    local p = Instance.new("Part")
    p.Size = sz; p.Color = col
    p.Material  = mat or Enum.Material.SmoothPlastic
    p.CanCollide= false; p.Anchored = true; p.CastShadow = false
    p.Parent = vmModel
    return p
end

local gloveC = Color3.fromRGB(28, 28, 33)
local gunC   = Color3.fromRGB(44, 44, 50)
local gunC2  = Color3.fromRGB(55, 55, 60)

-- Arms
local vmRA     = mkVP(Vector3.new(0.42, 0.42, 1.60), gloveC)
local vmLA     = mkVP(Vector3.new(0.38, 0.38, 1.40), gloveC)
-- Gun assembly
local vmGB     = mkVP(Vector3.new(0.21, 0.30, 1.10), gunC,  Enum.Material.Metal)
local vmGBrl   = mkVP(Vector3.new(0.11, 0.11, 0.75), gunC2, Enum.Material.Metal)
local vmGGrip  = mkVP(Vector3.new(0.19, 0.56, 0.22), gunC,  Enum.Material.Metal)
local vmGSlide = mkVP(Vector3.new(0.17, 0.17, 0.88), Color3.fromRGB(38, 38, 44), Enum.Material.Metal)
-- Sight dots
local vmSightF = mkVP(Vector3.new(0.06, 0.14, 0.06), Color3.fromRGB(255, 50, 50),   Enum.Material.Neon)
local vmSightR = mkVP(Vector3.new(0.06, 0.10, 0.06), Color3.fromRGB(220, 220, 220), Enum.Material.SmoothPlastic)

local function setVMIdle()
    vmRA.CFrame    = CFrame.new( 0.46,-0.52, 1.00)*CFrame.Angles(math.rad(-18), math.rad( 5),math.rad(-8))
    vmLA.CFrame    = CFrame.new(-0.40,-0.55, 1.30)*CFrame.Angles(math.rad(-12), math.rad(-4),math.rad(10))
    vmGB.CFrame    = CFrame.new( 0.22,-0.41, 0.84)
    vmGBrl.CFrame  = CFrame.new( 0.22,-0.41, 0.42)
    vmGGrip.CFrame = CFrame.new( 0.24,-0.62, 1.00)
    vmGSlide.CFrame= CFrame.new( 0.22,-0.37, 0.72)
    vmSightF.CFrame= CFrame.new( 0.22,-0.26, 0.47)
    vmSightR.CFrame= CFrame.new( 0.22,-0.26, 0.90)
end
setVMIdle()

-- Walk bob
local function updateVMBob(dt)
    local hrp = getHRP()
    if not hrp then return end
    local spd = hrp.Velocity.Magnitude
    if spd > 2 then
        S.vmBobT = S.vmBobT + dt * 8
        local bx = math.sin(S.vmBobT) * 0.014
        local by = math.abs(math.sin(S.vmBobT * 0.5)) * 0.012
        vmRA.CFrame    = CFrame.new(0.46+bx,-0.52-by,1.00)*CFrame.Angles(math.rad(-18),math.rad(5),math.rad(-8))
        vmGB.CFrame    = CFrame.new(0.22+bx,-0.41-by,0.84)
        vmGBrl.CFrame  = CFrame.new(0.22+bx,-0.41-by,0.42)
        vmGGrip.CFrame = CFrame.new(0.24+bx,-0.62-by,1.00)
        vmGSlide.CFrame= CFrame.new(0.22+bx,-0.37-by,0.72)
        vmSightF.CFrame= CFrame.new(0.22+bx,-0.26-by,0.47)
        vmSightR.CFrame= CFrame.new(0.22+bx,-0.26-by,0.90)
    else
        S.vmBobT = 0
        setVMIdle()
    end
    -- Slight camera-lag: pitch vmCam slightly with main camera
    local pitch = math.asin(math.clamp(Cam.CFrame.UpVector:Dot(Vector3.new(0,0,-1)),-1,1))
    vmCam.CFrame = CFrame.new(0,0,3) * CFrame.Angles(pitch * 0.12, 0, 0)
end

-- Inspect animation (F key)
local function doInspect()
    if S.inspecting then return end
    S.inspecting = true
    -- Raise gun
    TweenService:Create(vmRA, TweenInfo.new(0.28,Enum.EasingStyle.Quad,Enum.EasingDirection.Out),{
        CFrame=CFrame.new(0.46,-0.28,0.55)*CFrame.Angles(math.rad(-45),math.rad(25),math.rad(-10))
    }):Play()
    TweenService:Create(vmGB, TweenInfo.new(0.28,Enum.EasingStyle.Quad,Enum.EasingDirection.Out),{
        CFrame=CFrame.new(0.3,-0.18,0.45)
    }):Play()
    TweenService:Create(vmGBrl, TweenInfo.new(0.28,Enum.EasingStyle.Quad,Enum.EasingDirection.Out),{
        CFrame=CFrame.new(0.28,-0.18,0.15)*CFrame.Angles(0,math.rad(-15),0)
    }):Play()
    TweenService:Create(vmGSlide, TweenInfo.new(0.28,Enum.EasingStyle.Quad,Enum.EasingDirection.Out),{
        CFrame=CFrame.new(0.3,-0.14,0.32)
    }):Play()
    -- Return to idle after hold
    task.delay(1.1, function()
        TweenService:Create(vmRA,    TweenInfo.new(0.25,Enum.EasingStyle.Quad,Enum.EasingDirection.Out),
            {CFrame=CFrame.new(0.46,-0.52,1.00)*CFrame.Angles(math.rad(-18),math.rad(5),math.rad(-8))}):Play()
        TweenService:Create(vmGB,    TweenInfo.new(0.25,Enum.EasingStyle.Quad,Enum.EasingDirection.Out),
            {CFrame=CFrame.new(0.22,-0.41,0.84)}):Play()
        TweenService:Create(vmGBrl,  TweenInfo.new(0.25,Enum.EasingStyle.Quad,Enum.EasingDirection.Out),
            {CFrame=CFrame.new(0.22,-0.41,0.42)}):Play()
        TweenService:Create(vmGSlide,TweenInfo.new(0.25,Enum.EasingStyle.Quad,Enum.EasingDirection.Out),
            {CFrame=CFrame.new(0.22,-0.37,0.72)}):Play()
        task.wait(0.3)
        S.inspecting = false
    end)
end

-- ═══════════════════════════════════════════
--           SILENT AIM  HOOK
-- ═══════════════════════════════════════════
local function installSilentAim()
    local ok = pcall(function()
        local mt = getrawmetatable(game)
        setreadonly(mt, false)
        local oldNC = mt.__namecall
        mt.__namecall = newcclosure(function(self, ...)
            local method = getnamecallmethod()
            if (MOD.silentaim.enabled or MOD.autoshoot.enabled) and S.saTarget then
                -- Pause AntiAim during shot
                if MOD.antiaim.enabled then
                    S.aaPaused = true
                    task.delay(0.22, function() S.aaPaused = false end)
                end
                if method == "FindPartOnRayWithIgnoreList" or method == "FindPartOnRay" then
                    local args = {...}
                    if typeof(args[1]) == "Ray" then
                        local newDir = (S.saTarget.Position - args[1].Origin).Unit * 1000
                        args[1] = Ray.new(args[1].Origin, newDir)
                        return oldNC(self, unpack(args))
                    end
                elseif method == "Raycast" then
                    local args = {...}
                    if typeof(args[1]) == "Vector3" then
                        args[2] = S.saTarget.Position - args[1]
                        return oldNC(self, unpack(args))
                    end
                end
            end
            return oldNC(self, ...)
        end)
    end)
    if not ok then warn("[Woqued] SilentAim hook unavailable — executor may not support it") end
end
installSilentAim()

-- ═══════════════════════════════════════════
--     GUN EVENTS  (ShootTrails + AntiAim pause)
-- ═══════════════════════════════════════════
local function connectCharEvents(char)
    char.ChildAdded:Connect(function(child)
        if not child:IsA("Tool") or child.Name ~= "Gun" then return end
        child.Activated:Connect(function()
            -- Pause AntiAim during shot
            if MOD.antiaim.enabled then
                S.aaPaused = true
                task.delay(0.22, function() S.aaPaused = false end)
            end
            -- ShootTrails: draw line from gun barrel to impact
            if MOD.shoottrails.enabled and HAS_DRAW then
                local handle = child:FindFirstChildWhichIsA("BasePart")
                if handle then
                    local s2d, vis = Cam:WorldToViewportPoint(handle.Position)
                    if vis then
                        local mousePos = UserInputService:GetMouseLocation()
                        local ray = Cam:ScreenPointToRay(mousePos.X, mousePos.Y)
                        local rp  = RaycastParams.new()
                        rp.FilterDescendantsInstances = {char}
                        rp.FilterType = Enum.RaycastFilterType.Exclude
                        local hit    = workspace:Raycast(ray.Origin, ray.Direction * 500, rp)
                        local endPos = hit and hit.Position or (ray.Origin + ray.Direction * 500)
                        local e2d, _ = Cam:WorldToViewportPoint(endPos)
                        addTrail(
                            Vector2.new(s2d.X, s2d.Y),
                            Vector2.new(e2d.X, e2d.Y)
                        )
                    end
                end
            end
        end)
    end)
end
LP.CharacterAdded:Connect(connectCharEvents)
if LP.Character then connectCharEvents(LP.Character) end

-- ═══════════════════════════════════════════
--          MODULE TOGGLE  HANDLER
-- ═══════════════════════════════════════════
onToggle = function(m)
    updateBindsHUD()

    if m.id == "cspov" then
        if m.enabled then
            LP.CameraMaxZoomDistance = 0.5
            LP.CameraMinZoomDistance = 0.5
            vmFrame.Visible = true
        else
            LP.CameraMaxZoomDistance = 128
            LP.CameraMinZoomDistance = 0.5
            vmFrame.Visible = false
            setVMIdle()
        end

    elseif m.id == "watermark" then
        wmFrame.Visible = m.enabled

    elseif m.id == "bindshud" then
        bindsFrame.Visible = m.enabled
        if m.enabled then updateBindsHUD() end

    elseif m.id == "speedhud" then
        speedFrame.Visible = m.enabled

    elseif m.id == "antiaim" then
        if not m.enabled then
            S.aaAngle = 0; S.aaPaused = false
            local hrp = getHRP()
            -- Restore neutral orientation
            if hrp then hrp.CFrame = CFrame.new(hrp.Position) end
        end

    elseif m.id == "esp" then
        if not m.enabled then
            for _, d in pairs(espData) do
                if d.box  then d.box.Visible  = false end
                if d.name then d.name.Visible = false end
            end
        end

    elseif m.id == "silentaim" then
        if fovCircle then
            fovCircle.Visible = m.enabled and m.set.showFov
        end

    elseif m.id == "shoottrails" then
        if not m.enabled then
            for _, t in ipairs(S.trails) do
                for _, l in ipairs(t.lines) do l:Remove() end
            end
            S.trails = {}
        end
    end
end

-- ═══════════════════════════════════════════
--         MENU TOGGLE  +  ANIMATION
-- ═══════════════════════════════════════════
toggleMenu = function()
    if S.animating then return end
    S.animating = true

    if S.open then
        -- Close: shrink to centre then hide
        local sz  = win.Size
        local pos = win.Position
        local cx  = pos.X.Offset + sz.X.Offset * 0.5
        local cy  = pos.Y.Offset + sz.Y.Offset * 0.5
        local ns  = UDim2.new(0, sz.X.Offset  * 0.88, 0, sz.Y.Offset  * 0.88)
        local np  = UDim2.new(0, cx - ns.X.Offset * 0.5, 0, cy - ns.Y.Offset * 0.5)
        local t   = TweenService:Create(win,
            TweenInfo.new(0.17, Enum.EasingStyle.Quad, Enum.EasingDirection.In),
            {Size = ns, Position = np}
        )
        t:Play()
        t.Completed:Connect(function()
            win.Visible  = false
            win.Size     = sz;  win.Position = pos
            S.open       = false; S.animating = false
        end)
    else
        -- Open: start small, spring to full size
        local ts = UDim2.new(0, WW, 0, WH)
        local tp = UDim2.new(0.5, -WW/2, 0.5, -WH/2)
        win.Size     = UDim2.new(0, WW * 0.88, 0, WH * 0.88)
        win.Position = UDim2.new(0.5, -(WW * 0.88)/2, 0.5, -(WH * 0.88)/2)
        win.Visible  = true
        local t = TweenService:Create(win,
            TweenInfo.new(0.22, Enum.EasingStyle.Back, Enum.EasingDirection.Out),
            {Size = ts, Position = tp}
        )
        t:Play()
        t.Completed:Connect(function()
            S.open = true; S.animating = false
        end)
    end
end

-- ═══════════════════════════════════════════
--            DRAG  HANDLERS
-- ═══════════════════════════════════════════

-- Float button: drag + click detection
do
    local d, s, p, moved = false, nil, nil, false
    floatBtn.InputBegan:Connect(function(inp)
        if inp.UserInputType == Enum.UserInputType.MouseButton1 then
            d = true; moved = false; s = inp.Position; p = floatBtn.Position
        end
    end)
    UserInputService.InputEnded:Connect(function(inp)
        if d and inp.UserInputType == Enum.UserInputType.MouseButton1 then
            d = false
            if not moved then toggleMenu() end
        end
    end)
    UserInputService.InputChanged:Connect(function(inp)
        if d and inp.UserInputType == Enum.UserInputType.MouseMovement then
            local delta = inp.Position - s
            if delta.Magnitude > 5 then
                moved = true
                floatBtn.Position = UDim2.new(
                    p.X.Scale, p.X.Offset + delta.X,
                    p.Y.Scale, p.Y.Offset + delta.Y
                )
            end
        end
    end)
end

-- Main window: drag by header bar
do
    local d, s, p = false, nil, nil
    hdr.InputBegan:Connect(function(inp)
        if inp.UserInputType == Enum.UserInputType.MouseButton1 then
            d = true; s = inp.Position; p = win.Position
        end
    end)
    UserInputService.InputEnded:Connect(function(inp)
        if inp.UserInputType == Enum.UserInputType.MouseButton1 then d = false end
    end)
    UserInputService.InputChanged:Connect(function(inp)
        if d and inp.UserInputType == Enum.UserInputType.MouseMovement then
            local delta = inp.Position - s
            win.Position = UDim2.new(
                p.X.Scale, p.X.Offset + delta.X,
                p.Y.Scale, p.Y.Offset + delta.Y
            )
        end
    end)
end

-- ═══════════════════════════════════════════
--            MAIN  RENDER  LOOP
-- ═══════════════════════════════════════════
RunService.RenderStepped:Connect(function(dt)
    -- FPS counter
    S.fpsCount = S.fpsCount + 1
    if tick() - S.fpsTimer >= 1 then
        S.fps = S.fpsCount; S.fpsCount = 0; S.fpsTimer = tick()
    end

    local char = getChar()
    local hrp  = getHRP()
    local hum  = getHum()

    -- ── AntiAim ──────────────────────────────
    if MOD.antiaim.enabled and hrp then
        if not S.aaPaused then
            S.aaAngle = (S.aaAngle + 0.055) % (math.pi * 2)
        end
        -- Spin on Y axis + pitch 90° down (looking at floor)
        hrp.CFrame = CFrame.new(hrp.Position) * CFrame.Angles(math.pi / 2, S.aaAngle, 0)
    end

    -- ── BunnyHop ─────────────────────────────
    if MOD.bunnyhop.enabled and hum then
        if UserInputService:IsKeyDown(Enum.KeyCode.Space) then
            local st = hum:GetState()
            if st == Enum.HumanoidStateType.Running
            or st == Enum.HumanoidStateType.RunningNoPhysics
            or hum.FloorMaterial ~= Enum.Material.Air then
                hum.Jump = true
            end
        end
    end

    -- ── SilentAim target finding ─────────────
    S.saTarget = nil
    if (MOD.silentaim.enabled or MOD.autoshoot.enabled) and HAS_DRAW then
        local mousePos = UserInputService:GetMouseLocation()
        local closestDist = MOD.silentaim.set.fov

        for _, pl in ipairs(Players:GetPlayers()) do
            if pl ~= LP and pl.Character then
                local h = pl.Character:FindFirstChildOfClass("Humanoid")
                if h and h.Health > 0 then
                    local head = pl.Character:FindFirstChild("Head")
                    if head then
                        local sp, onScr = Cam:WorldToViewportPoint(head.Position)
                        if onScr then
                            local dist = (Vector2.new(sp.X, sp.Y) - mousePos).Magnitude
                            if dist < closestDist then
                                closestDist = dist; S.saTarget = head
                            end
                        end
                    end
                end
            end
        end

        -- FOV circle
        if fovCircle then
            if MOD.silentaim.enabled and MOD.silentaim.set.showFov then
                fovCircle.Position = mousePos
                fovCircle.Radius   = MOD.silentaim.set.fov
                fovCircle.Color    = S.saTarget
                    and Color3.fromRGB(255, 70, 70)
                    or  Color3.fromRGB(255, 255, 255)
                fovCircle.Visible  = true
            else
                fovCircle.Visible = false
            end
        end
    elseif fovCircle then
        fovCircle.Visible = false
    end

    -- ── ESP ───────────────────────────────────
    if MOD.esp.enabled and HAS_DRAW then
        for _, pl in ipairs(Players:GetPlayers()) do
            if pl ~= LP then
                local d = getOrCreateESP(pl)
                if d then
                    local c2 = pl.Character
                    local hrp2 = c2 and c2:FindFirstChild("HumanoidRootPart")
                    local h2   = c2 and c2:FindFirstChildOfClass("Humanoid")

                    if hrp2 and h2 and h2.Health > 0 then
                        local center, onScr = Cam:WorldToViewportPoint(hrp2.Position)
                        if onScr then
                            local topSP, _ = Cam:WorldToViewportPoint(hrp2.Position + Vector3.new(0, 3.2, 0))
                            local botSP, _ = Cam:WorldToViewportPoint(hrp2.Position - Vector3.new(0, 2.8, 0))
                            local bH = math.abs(topSP.Y - botSP.Y)
                            local bW = bH * 0.5

                            local col
                            if isMurd(pl) then
                                col = Color3.fromRGB(255, 60, 60)
                            elseif isSher(pl) then
                                col = Color3.fromRGB(60, 120, 255)
                            else
                                col = Color3.fromRGB(220, 220, 220)
                            end

                            local role = isMurd(pl) and " [M]" or (isSher(pl) and " [S]" or "")

                            d.box.Color    = col
                            d.box.Visible  = true
                            d.box.Size     = Vector2.new(bW, bH)
                            d.box.Position = Vector2.new(center.X - bW/2, topSP.Y)

                            d.name.Text     = pl.Name .. role
                            d.name.Color    = col
                            d.name.Visible  = true
                            d.name.Position = Vector2.new(center.X, topSP.Y - 15)
                        else
                            d.box.Visible = false; d.name.Visible = false
                        end
                    else
                        d.box.Visible = false; d.name.Visible = false
                    end
                end
            end
        end
    else
        -- Hide all ESP when disabled
        for _, d in pairs(espData) do
            if d.box  then d.box.Visible  = false end
            if d.name then d.name.Visible = false end
        end
    end

    -- ── ShootTrails fade ─────────────────────
    updateTrails()

    -- ── CsPov viewmodel update ───────────────
    if MOD.cspov.enabled and vmFrame.Visible then
        updateVMBob(dt)
    end

    -- ── Watermark update ─────────────────────
    if MOD.watermark.enabled and wmFrame.Visible then
        local ping = 0
        pcall(function() ping = math.floor(LP:GetNetworkPing() * 1000) end)
        wmStatsLbl.Text = string.format("%d FPS  |  %dms", S.fps, ping)
    end

    -- ── Speed HUD update ─────────────────────
    if MOD.speedhud.enabled and speedFrame.Visible and hrp then
        local spd = hrp.Velocity * Vector3.new(1, 0, 1)
        speedVal.Text = tostring(math.floor(spd.Magnitude))
    end
end)

-- ═══════════════════════════════════════════
--            INPUT  HANDLER
-- ═══════════════════════════════════════════
UserInputService.InputBegan:Connect(function(inp, gp)
    if gp then return end

    -- [Insert] toggles main menu
    if inp.KeyCode == Enum.KeyCode.Insert then
        toggleMenu()

    -- [F] triggers inspect animation when CsPov is on
    elseif inp.KeyCode == Enum.KeyCode.F then
        if MOD.cspov.enabled then
            doInspect()
        end
    end
end)

-- ═══════════════════════════════════════════
--  AUTORUN CHECK  –  print confirmation
-- ═══════════════════════════════════════════
print(string.rep("═", 46))
print("  W O Q U E D  loaded successfully!")
print("  Toggle: [Insert] or tap the W button")
print("  Drag the W button anywhere on screen")
print(string.rep("═", 46))
