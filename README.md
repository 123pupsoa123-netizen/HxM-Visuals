--[[
    VISUAL HUB — одно-файловое меню (Roblox, Luau)
    Вкладки: Темы (градиент меню) / Мир (день-ночь, дождь) / Шейдеры / Персонаж / Настройки
    Всё работает только на клиенте — другие игроки этого не видят.

    Название меняется в HUB_NAME, ассеты — в блоках KORBLOX и RAIN_TEXTURE.
]]

local HUB_NAME     = "HxM Visuals"
local HUB_SUB      = "Visual Tools"
local CONFIG_FILE  = "MyHub_config.json"
local RAIN_TEXTURE = "rbxasset://textures/particles/sparkles_main.dds"

-- Ассеты фейк-корблокса (если нога выглядит странно — поменяй ID здесь)
local KORBLOX = {
    UpperLeg = "rbxassetid://902942096", -- R15
    LowerLeg = "rbxassetid://902942093", -- R15
    Foot     = "rbxassetid://902942089", -- R15
    R6Leg    = "rbxassetid://902942093", -- R6
    Texture  = "rbxassetid://902843398",
}

local DESIGN_W, DESIGN_H = 520, 380

----------------------------------------------------------------------
-- Сервисы
----------------------------------------------------------------------
local Players          = game:GetService("Players")
local Lighting         = game:GetService("Lighting")
local TweenService     = game:GetService("TweenService")
local UserInputService = game:GetService("UserInputService")
local RunService       = game:GetService("RunService")
local HttpService      = game:GetService("HttpService")
local Workspace        = game:GetService("Workspace")
local LocalPlayer      = Players.LocalPlayer

-- защита от двойного запуска
local genv = (getgenv and getgenv()) or _G
if genv.__HubCleanup then pcall(genv.__HubCleanup) end

----------------------------------------------------------------------
-- Хелперы
----------------------------------------------------------------------
local WHITE   = Color3.new(1, 1, 1)
local GRAY    = Color3.fromRGB(140, 140, 150)
local OFF_COL = Color3.fromRGB(72, 72, 80)
local ROW_BG  = Color3.fromRGB(40, 40, 45)

local function new(class, props, parent)
    local o = Instance.new(class)
    if props then
        for k, v in pairs(props) do o[k] = v end
    end
    if parent then o.Parent = parent end
    return o
end

local function corner(o, r)
    return new("UICorner", { CornerRadius = UDim.new(0, r) }, o)
end

local function stroke(o, color, thick, transp)
    return new("UIStroke", {
        Color = color, Thickness = thick or 1, Transparency = transp or 0,
        ApplyStrokeMode = Enum.ApplyStrokeMode.Border,
    }, o)
end

local conns = {}
local function connect(sig, fn)
    local c = sig:Connect(fn)
    conns[#conns + 1] = c
    return c
end

----------------------------------------------------------------------
-- Конфиг
----------------------------------------------------------------------
local cfg = { theme = 1, lang = "RU", anim = false }
pcall(function()
    if isfile and isfile(CONFIG_FILE) then
        local d = HttpService:JSONDecode(readfile(CONFIG_FILE))
        for k, v in pairs(d) do cfg[k] = v end
    end
end)

----------------------------------------------------------------------
-- Темы и тексты
----------------------------------------------------------------------
local THEMES = {
    { key = "t_blood",  color = Color3.fromRGB(226, 32, 44) },
    { key = "t_gold",   color = Color3.fromRGB(212, 160, 30) },
    { key = "t_honey",  color = Color3.fromRGB(240, 180, 70) },
    { key = "t_amber",  color = Color3.fromRGB(255, 170, 30) },
    { key = "t_rose",   color = Color3.fromRGB(240, 110, 175) },
    { key = "t_ice",    color = Color3.fromRGB(110, 200, 255) },
    { key = "t_violet", color = Color3.fromRGB(150, 90, 255) },
    { key = "t_mint",   color = Color3.fromRGB(60, 220, 160) },
    { key = "t_ocean",  color = Color3.fromRGB(40, 120, 255) },
    { key = "t_toxic",  color = Color3.fromRGB(140, 255, 60) },
}
if type(cfg.theme) ~= "number" or cfg.theme < 1 or cfg.theme > #THEMES then cfg.theme = 1 end

local L = {
    themes    = { EN = "Themes",     RU = "Темы" },
    world     = { EN = "World",      RU = "Мир" },
    shaders   = { EN = "Shaders",    RU = "Шейдеры" },
    character = { EN = "Character",  RU = "Персонаж" },
    settings  = { EN = "Settings",   RU = "Настройки" },

    hd_time   = { EN = "DAY / NIGHT", RU = "ДЕНЬ / НОЧЬ" },
    hd_rain   = { EN = "RAIN",        RU = "ДОЖДЬ" },
    time_lock = { EN = "Lock time",   RU = "Фиксировать время" },
    time      = { EN = "Time of day", RU = "Время суток" },
    day       = { EN = "☀ Day",       RU = "☀ День" },
    night     = { EN = "🌙 Night",    RU = "🌙 Ночь" },
    rain      = { EN = "Rain",        RU = "Дождь" },
    rain_int  = { EN = "Intensity",   RU = "Интенсивность" },

    sh_off       = { EN = "Off",       RU = "Выключено" },
    sh_cinematic = { EN = "Cinematic", RU = "Кино" },
    sh_vibrant   = { EN = "Vibrant",   RU = "Яркий" },
    sh_realistic = { EN = "Realistic", RU = "Реализм" },
    sh_moody     = { EN = "Moody",     RU = "Мрачный" },
    sh_warm      = { EN = "Warm",      RU = "Тёплый" },

    headless  = { EN = "Fake Headless", RU = "Безголовый (фейк)" },
    korblox   = { EN = "Fake Korblox",  RU = "Корблокс (фейк)" },
    rig       = { EN = "Rig type",      RU = "Тип рига" },
    char_note = { EN = "Visible only to you. Auto-detects R6 / R15 and re-applies after respawn.",
                  RU = "Видно только вам. Сам определяет R6 / R15 и применяется после респавна." },

    anim_grad = { EN = "Animated gradient",  RU = "Анимированный градиент" },
    reset     = { EN = "Reset all effects",  RU = "Сбросить все эффекты" },
    set_note  = { EN = "✕ closes the menu and removes all effects. — minimizes it.",
                  RU = "✕ закрывает меню и убирает все эффекты. — сворачивает." },
    saved     = { EN = "Config saved",             RU = "Конфиг сохранён" },
    nosave    = { EN = "Saving not supported",     RU = "Сохранение не поддерживается" },
    reset_done= { EN = "Effects reset",            RU = "Эффекты сброшены" },

    t_blood  = { EN = "Blood",  RU = "Blood" },
    t_gold   = { EN = "Gold",   RU = "Золото" },
    t_honey  = { EN = "Honey",  RU = "Мёд" },
    t_amber  = { EN = "Amber",  RU = "Янтарь" },
    t_rose   = { EN = "Rose",   RU = "Роза" },
    t_ice    = { EN = "Ice",    RU = "Лёд" },
    t_violet = { EN = "Violet", RU = "Фиолет" },
    t_mint   = { EN = "Mint",   RU = "Мята" },
    t_ocean  = { EN = "Ocean",  RU = "Океан" },
    t_toxic  = { EN = "Toxic",  RU = "Токсик" },
}

local lang = (cfg.lang == "EN") and "EN" or "RU"
local bindings = {}
local function tr(key)
    local e = L[key]
    return e and e[lang] or key
end
local function bind(obj, key, prop)
    prop = prop or "Text"
    obj[prop] = tr(key)
    bindings[#bindings + 1] = { obj, key, prop }
    return obj
end

----------------------------------------------------------------------
-- ScreenGui + акцентный цвет
----------------------------------------------------------------------
local gui = new("ScreenGui", {
    Name = HUB_NAME .. tostring(math.random(1000, 9999)),
    ResetOnSpawn = false,
    ZIndexBehavior = Enum.ZIndexBehavior.Sibling,
    IgnoreGuiInset = true,
    DisplayOrder = 999,
})
do
    local ok = pcall(function()
        if gethui then gui.Parent = gethui() else error("no gethui") end
    end)
    if not (ok and gui.Parent) then
        ok = pcall(function() gui.Parent = game:GetService("CoreGui") end)
    end
    if not (ok and gui.Parent) then
        gui.Parent = LocalPlayer:WaitForChild("PlayerGui")
    end
end

local accent = new("Color3Value", { Value = THEMES[cfg.theme].color }, gui)
local accentFns = {}
local function onAccent(fn)
    accentFns[#accentFns + 1] = fn
    fn(accent.Value)
end
connect(accent.Changed, function()
    for _, fn in ipairs(accentFns) do fn(accent.Value) end
end)

----------------------------------------------------------------------
-- Снимок освещения
----------------------------------------------------------------------
local ORIG = {}
for _, p in ipairs({ "Ambient", "OutdoorAmbient", "Brightness", "ClockTime", "FogEnd",
    "FogStart", "FogColor", "ExposureCompensation" }) do
    ORIG[p] = Lighting[p]
end
local function restoreLighting()
    for k, v in pairs(ORIG) do
        pcall(function() Lighting[k] = v end)
    end
end

----------------------------------------------------------------------
-- Время суток
----------------------------------------------------------------------
local timeState = { lock = false, value = ORIG.ClockTime }

----------------------------------------------------------------------
-- Дождь
----------------------------------------------------------------------
local rain = { on = false, rate = 700 }
local rainPart, rainEmitter

local function setRain(on)
    rain.on = on
    if on then
        if not (rainPart and rainPart.Parent) then
            rainPart = new("Part", {
                Name = "HubRain", Anchored = true, CanCollide = false, CanQuery = false,
                CanTouch = false, CastShadow = false, Transparency = 1,
                Size = Vector3.new(80, 1, 80),
            })
            rainEmitter = new("ParticleEmitter", {
                Texture = RAIN_TEXTURE,
                Color = ColorSequence.new(Color3.fromRGB(190, 215, 255)),
                Transparency = NumberSequence.new({
                    NumberSequenceKeypoint.new(0, 0.35),
                    NumberSequenceKeypoint.new(1, 0.6),
                }),
                Size = NumberSequence.new(0.25),
                Lifetime = NumberRange.new(0.9, 1.1),
                Speed = NumberRange.new(70, 90),
                EmissionDirection = Enum.NormalId.Bottom,
                SpreadAngle = Vector2.new(2, 2),
                Orientation = Enum.ParticleOrientation.VelocityParallel,
                LightEmission = 0.35,
                LightInfluence = 0,
                LockedToPart = false,
                Rate = rain.rate,
            }, rainPart)
            rainPart.Parent = Workspace.CurrentCamera or Workspace
        end
        rainEmitter.Rate = rain.rate
        rainEmitter.Enabled = true
        pcall(function()
            Lighting.FogColor = Color3.fromRGB(120, 130, 145)
            Lighting.FogEnd = math.min(ORIG.FogEnd, 1200)
            Lighting.Brightness = ORIG.Brightness * 0.75
        end)
    else
        if rainEmitter then rainEmitter.Enabled = false end
        pcall(function()
            Lighting.FogColor = ORIG.FogColor
            Lighting.FogEnd = ORIG.FogEnd
            Lighting.Brightness = ORIG.Brightness
        end)
    end
end

----------------------------------------------------------------------
-- Шейдеры (пресеты постобработки)
----------------------------------------------------------------------
local SHADER_ORDER = { "cinematic", "vibrant", "realistic", "moody", "warm" }
local SHADERS = {
    cinematic = {
        bloom = { Intensity = 0.5, Size = 30, Threshold = 1.8 },
        rays  = { Intensity = 0.10, Spread = 0.7 },
        color = { Brightness = 0, Contrast = 0.2, Saturation = 0.1, TintColor = Color3.fromRGB(255, 246, 236) },
        dof   = { FarIntensity = 0.12, FocusDistance = 70, InFocusRadius = 60, NearIntensity = 0 },
        atm   = { Density = 0.28, Offset = 0.15, Color = Color3.fromRGB(199, 210, 225),
                  Decay = Color3.fromRGB(106, 112, 125), Glare = 0.15, Haze = 1.4 },
        exposure = 0.15,
    },
    vibrant = {
        bloom = { Intensity = 0.35, Size = 24, Threshold = 2 },
        rays  = { Intensity = 0.15, Spread = 1 },
        color = { Brightness = 0.02, Contrast = 0.12, Saturation = 0.45, TintColor = WHITE },
        exposure = 0.05,
    },
    realistic = {
        bloom = { Intensity = 0.25, Size = 20, Threshold = 2.2 },
        rays  = { Intensity = 0.08, Spread = 0.6 },
        color = { Brightness = 0, Contrast = 0.08, Saturation = -0.05, TintColor = Color3.fromRGB(255, 252, 246) },
        atm   = { Density = 0.32, Offset = 0.25, Color = Color3.fromRGB(190, 205, 225),
                  Decay = Color3.fromRGB(92, 100, 115), Glare = 0.1, Haze = 1.8 },
        exposure = 0,
    },
    moody = {
        bloom = { Intensity = 0.7, Size = 36, Threshold = 1.4 },
        color = { Brightness = -0.06, Contrast = 0.28, Saturation = -0.25, TintColor = Color3.fromRGB(200, 215, 255) },
        dof   = { FarIntensity = 0.2, FocusDistance = 50, InFocusRadius = 40, NearIntensity = 0 },
        atm   = { Density = 0.4, Offset = 0, Color = Color3.fromRGB(120, 130, 155),
                  Decay = Color3.fromRGB(60, 65, 85), Glare = 0, Haze = 2.2 },
        exposure = -0.25,
    },
    warm = {
        bloom = { Intensity = 0.55, Size = 32, Threshold = 1.6 },
        rays  = { Intensity = 0.25, Spread = 1 },
        color = { Brightness = 0.03, Contrast = 0.1, Saturation = 0.2, TintColor = Color3.fromRGB(255, 224, 190) },
        atm   = { Density = 0.3, Offset = 0.2, Color = Color3.fromRGB(255, 214, 170),
                  Decay = Color3.fromRGB(150, 95, 60), Glare = 0.4, Haze = 1.6 },
        exposure = 0.1,
    },
}
local SHADER_CLASS = { bloom = "BloomEffect", rays = "SunRaysEffect", color = "ColorCorrectionEffect", dof = "DepthOfFieldEffect" }
local SHADER_NAME  = { bloom = "HubBloom", rays = "HubSunRays", color = "HubColor", dof = "HubDoF" }

local atmOwned, atmSnap
local function setAtmosphere(props)
    local atm = Lighting:FindFirstChildOfClass("Atmosphere")
    if not atm then
        atm = new("Atmosphere", { Name = "HubAtmosphere" }, Lighting)
        atmOwned = atm
    elseif atm ~= atmOwned and not atmSnap then
        atmSnap = { inst = atm, props = {} }
        for _, p in ipairs({ "Density", "Offset", "Color", "Decay", "Glare", "Haze" }) do
            atmSnap.props[p] = atm[p]
        end
    end
    for k, v in pairs(props) do
        pcall(function() atm[k] = v end)
    end
end
local function restoreAtmosphere()
    if atmOwned then
        pcall(function() atmOwned:Destroy() end)
        atmOwned = nil
    end
    if atmSnap then
        for k, v in pairs(atmSnap.props) do
            pcall(function() atmSnap.inst[k] = v end)
        end
        atmSnap = nil
    end
end

local function clearShader()
    for _, n in pairs(SHADER_NAME) do
        local e = Lighting:FindFirstChild(n)
        if e then e:Destroy() end
    end
    restoreAtmosphere()
    pcall(function() Lighting.ExposureCompensation = ORIG.ExposureCompensation end)
end

local function applyShader(name)
    clearShader()
    local p = SHADERS[name]
    if not p then return end
    for k, class in pairs(SHADER_CLASS) do
        if p[k] then
            local e = new(class, { Name = SHADER_NAME[k] })
            for prop, v in pairs(p[k]) do
                pcall(function() e[prop] = v end)
            end
            e.Parent = Lighting
        end
    end
    if p.atm then setAtmosphere(p.atm) end
    if p.exposure then
        pcall(function() Lighting.ExposureCompensation = p.exposure end)
    end
end

----------------------------------------------------------------------
-- Персонаж: фейк-хедлесс и фейк-корблокс (R6 + R15)
----------------------------------------------------------------------
local charState = { headless = false, korblox = false }
local snap = setmetatable({}, { __mode = "k" })
local createdMeshes = setmetatable({}, { __mode = "k" })

local function setProp(inst, prop, value)
    if not inst then return end
    local s = snap[inst]
    if not s then s = {}; snap[inst] = s end
    if s[prop] == nil then
        local ok, v = pcall(function() return inst[prop] end)
        if ok then s[prop] = v end
    end
    pcall(function() inst[prop] = value end)
end

local function restoreProps(inst)
    local s = snap[inst]
    if not s then return end
    for p, v in pairs(s) do
        pcall(function() inst[p] = v end)
    end
    snap[inst] = nil
end

local function applyHeadless(char, on)
    local head = char:FindFirstChild("Head")
    if not head then return end
    if on then
        setProp(head, "Transparency", 1)
        for _, d in ipairs(head:GetChildren()) do
            if d:IsA("Decal") then setProp(d, "Transparency", 1) end
        end
    else
        restoreProps(head)
        for _, d in ipairs(head:GetChildren()) do
            if d:IsA("Decal") then restoreProps(d) end
        end
    end
end

local function applyKorblox(char, on)
    local hum = char:FindFirstChildOfClass("Humanoid")
    local r6 = hum and hum.RigType == Enum.HumanoidRigType.R6
    if r6 then
        local leg = char:FindFirstChild("Right Leg")
        if not leg then return end
        local mesh = leg:FindFirstChildOfClass("SpecialMesh")
        if on then
            if not mesh then
                mesh = new("SpecialMesh", nil, leg)
                createdMeshes[char] = mesh
            end
            setProp(mesh, "MeshType", Enum.MeshType.FileMesh)
            setProp(mesh, "MeshId", KORBLOX.R6Leg)
            setProp(mesh, "TextureId", KORBLOX.Texture)
        else
            if createdMeshes[char] then
                createdMeshes[char]:Destroy()
                createdMeshes[char] = nil
            elseif mesh then
                restoreProps(mesh)
            end
        end
    else
        local map = {
            RightUpperLeg = KORBLOX.UpperLeg,
            RightLowerLeg = KORBLOX.LowerLeg,
            RightFoot     = KORBLOX.Foot,
        }
        for name, id in pairs(map) do
            local p = char:FindFirstChild(name)
            if p then
                if on then
                    setProp(p, "MeshId", id)
                    setProp(p, "TextureID", KORBLOX.Texture)
                else
                    restoreProps(p)
                end
            end
        end
    end
end

----------------------------------------------------------------------
-- ГЛАВНОЕ ОКНО
----------------------------------------------------------------------
local main = new("Frame", {
    Name = "Main",
    AnchorPoint = Vector2.new(0.5, 0.5),
    Position = UDim2.fromScale(0.5, 0.5),
    Size = UDim2.fromOffset(DESIGN_W, DESIGN_H),
    BackgroundColor3 = WHITE,
    BorderSizePixel = 0,
    ClipsDescendants = true,
}, gui)
corner(main, 14)
stroke(main, Color3.fromRGB(70, 70, 78), 1, 0.3)
local uiScale = new("UIScale", { Scale = 1 }, main)
local grad = new("UIGradient", { Rotation = 35 }, main)

onAccent(function(c)
    local base = Color3.fromRGB(26, 26, 30)
    grad.Color = ColorSequence.new({
        ColorSequenceKeypoint.new(0,    base),
        ColorSequenceKeypoint.new(0.36, base),
        ColorSequenceKeypoint.new(0.50, base:Lerp(c, 0.30)),
        ColorSequenceKeypoint.new(0.64, base),
        ColorSequenceKeypoint.new(1,    base:Lerp(c, 0.08)),
    })
end)

local baseScale = 1
local function fit()
    local cam = Workspace.CurrentCamera
    if not cam then return end
    local vp = cam.ViewportSize
    baseScale = math.clamp(math.min(vp.X * 0.9 / DESIGN_W, vp.Y * 0.9 / DESIGN_H), 0.4, 1.2)
    uiScale.Scale = baseScale
end
fit()
if Workspace.CurrentCamera then
    connect(Workspace.CurrentCamera:GetPropertyChangedSignal("ViewportSize"), fit)
end

-- Шапка ---------------------------------------------------------------
local header = new("Frame", { Name = "Header", Size = UDim2.new(1, 0, 0, 60), BackgroundTransparency = 1 }, main)

local logo = new("Frame", {
    Position = UDim2.fromOffset(12, 11), Size = UDim2.fromOffset(38, 38),
    BackgroundColor3 = accent.Value, BorderSizePixel = 0,
}, header)
corner(logo, 9)
new("TextLabel", {
    Size = UDim2.fromScale(1, 1), BackgroundTransparency = 1, Text = "🦊",
    TextSize = 22, Font = Enum.Font.GothamBold, TextColor3 = WHITE,
}, logo)
onAccent(function(c) logo.BackgroundColor3 = c end)

local titleLbl = new("TextLabel", {
    Position = UDim2.fromOffset(60, 9), Size = UDim2.fromOffset(150, 24),
    BackgroundTransparency = 1, Text = HUB_NAME, Font = Enum.Font.GothamBlack,
    TextSize = 19, TextXAlignment = Enum.TextXAlignment.Left, TextColor3 = accent.Value,
}, header)
onAccent(function(c) titleLbl.TextColor3 = c end)
new("TextLabel", {
    Position = UDim2.fromOffset(60, 33), Size = UDim2.fromOffset(150, 16),
    BackgroundTransparency = 1, Text = HUB_SUB, Font = Enum.Font.Gotham,
    TextSize = 12, TextXAlignment = Enum.TextXAlignment.Left, TextColor3 = GRAY,
}, header)

local function headerBtn(text, offsetX, cb)
    local b = new("TextButton", {
        AnchorPoint = Vector2.new(1, 0), Position = UDim2.new(1, offsetX, 0, 14),
        Size = UDim2.fromOffset(32, 32), BackgroundTransparency = 1, Text = text,
        TextSize = 20, Font = Enum.Font.GothamBold, TextColor3 = Color3.fromRGB(235, 235, 240),
        AutoButtonColor = false,
    }, header)
    connect(b.MouseButton1Click, cb)
    return b
end

-- переключатель языка EN | RU
local langPill = new("Frame", {
    AnchorPoint = Vector2.new(1, 0), Position = UDim2.new(1, -164, 0, 17),
    Size = UDim2.fromOffset(78, 26), BackgroundColor3 = Color3.fromRGB(30, 30, 34),
    BorderSizePixel = 0,
}, header)
corner(langPill, 8)
stroke(langPill, Color3.fromRGB(70, 70, 78), 1, 0.4)
local function pillBtn(text, x)
    local b = new("TextButton", {
        Position = UDim2.new(0, x, 0, 2), Size = UDim2.new(0.5, -3, 1, -4),
        BackgroundColor3 = accent.Value, BackgroundTransparency = 1, Text = text,
        Font = Enum.Font.GothamBold, TextSize = 12, TextColor3 = GRAY,
        AutoButtonColor = false, BorderSizePixel = 0,
    }, langPill)
    corner(b, 6)
    return b
end
local enBtn = pillBtn("EN", 2)
local ruBtn = pillBtn("RU", 37)
local function paintLang()
    local en = (lang == "EN")
    enBtn.BackgroundTransparency = en and 0 or 1
    enBtn.BackgroundColor3 = accent.Value
    enBtn.TextColor3 = en and WHITE or GRAY
    ruBtn.BackgroundTransparency = en and 1 or 0
    ruBtn.BackgroundColor3 = accent.Value
    ruBtn.TextColor3 = en and GRAY or WHITE
end
onAccent(paintLang)

new("Frame", {
    Position = UDim2.new(0, 0, 0, 60), Size = UDim2.new(1, 0, 0, 1),
    BackgroundColor3 = Color3.fromRGB(80, 80, 88), BackgroundTransparency = 0.5, BorderSizePixel = 0,
}, main)

-- Боковая панель ----------------------------------------------------
local sidebar = new("Frame", {
    Name = "Sidebar", Position = UDim2.fromOffset(0, 61), Size = UDim2.new(0, 68, 1, -61),
    BackgroundColor3 = Color3.fromRGB(20, 20, 23), BackgroundTransparency = 0.1, BorderSizePixel = 0,
}, main)
local indicator = new("Frame", {
    Position = UDim2.fromOffset(0, 20), Size = UDim2.fromOffset(3, 26),
    BackgroundColor3 = accent.Value, BorderSizePixel = 0,
}, sidebar)
corner(indicator, 2)
onAccent(function(c) indicator.BackgroundColor3 = c end)

-- Подвал --------------------------------------------------------------
local footer = new("Frame", {
    Name = "Footer", Position = UDim2.new(0, 0, 1, -26), Size = UDim2.new(1, 0, 0, 26),
    BackgroundColor3 = Color3.fromRGB(18, 18, 20), BackgroundTransparency = 0.05,
    BorderSizePixel = 0, ZIndex = 3,
}, main)
new("TextLabel", {
    Position = UDim2.fromOffset(14, 0), Size = UDim2.fromOffset(160, 26), BackgroundTransparency = 1,
    Text = HUB_NAME, Font = Enum.Font.Gotham, TextSize = 12, TextColor3 = GRAY,
    TextXAlignment = Enum.TextXAlignment.Left, ZIndex = 4,
}, footer)
local statusLbl = new("TextLabel", {
    AnchorPoint = Vector2.new(1, 0), Position = UDim2.new(1, -14, 0, 0), Size = UDim2.fromOffset(260, 26),
    BackgroundTransparency = 1, Text = "", Font = Enum.Font.GothamMedium, TextSize = 12,
    TextColor3 = accent.Value, TextXAlignment = Enum.TextXAlignment.Right, ZIndex = 4,
}, footer)
onAccent(function(c) statusLbl.TextColor3 = c end)

local toastToken = 0
local function toast(key)
    toastToken = toastToken + 1
    local my = toastToken
    statusLbl.Text = tr(key)
    task.delay(2.2, function()
        if toastToken == my and statusLbl.Parent then statusLbl.Text = "" end
    end)
end

local function saveConfig(silent)
    local ok = pcall(function()
        writefile(CONFIG_FILE, HttpService:JSONEncode(cfg))
    end)
    if not silent then toast(ok and "saved" or "nosave") end
end

-- Контент -------------------------------------------------------------
local content = new("Frame", {
    Name = "Content", Position = UDim2.fromOffset(68, 61), Size = UDim2.new(1, -68, 1, -87),
    BackgroundTransparency = 1,
}, main)
local sectionTitle = new("TextLabel", {
    Position = UDim2.fromOffset(16, 8), Size = UDim2.new(1, -30, 0, 20), BackgroundTransparency = 1,
    Font = Enum.Font.GothamMedium, TextSize = 13, TextColor3 = GRAY,
    TextXAlignment = Enum.TextXAlignment.Left, Text = "",
}, content)

local function newPage()
    local sf = new("ScrollingFrame", {
        Position = UDim2.fromOffset(0, 32), Size = UDim2.new(1, 0, 1, -32),
        BackgroundTransparency = 1, BorderSizePixel = 0, ScrollBarThickness = 3,
        ScrollBarImageColor3 = Color3.fromRGB(120, 120, 128),
        CanvasSize = UDim2.new(), AutomaticCanvasSize = Enum.AutomaticSize.Y, Visible = false,
    }, content)
    new("UIListLayout", { Padding = UDim.new(0, 6), SortOrder = Enum.SortOrder.LayoutOrder }, sf)
    new("UIPadding", {
        PaddingLeft = UDim.new(0, 14), PaddingRight = UDim.new(0, 16),
        PaddingTop = UDim.new(0, 2), PaddingBottom = UDim.new(0, 10),
    }, sf)
    return sf
end

-- Конструкторы элементов ---------------------------------------------
local function rowBase(page, h, class, extra)
    local props = {
        Size = UDim2.new(1, 0, 0, h), BackgroundColor3 = ROW_BG,
        BackgroundTransparency = 0.35, BorderSizePixel = 0,
    }
    if extra then
        for k, v in pairs(extra) do props[k] = v end
    end
    local f = new(class or "Frame", props, page)
    corner(f, 9)
    return f
end

local function rowLabel(parent, key, x)
    x = x or 14
    return bind(new("TextLabel", {
        BackgroundTransparency = 1, Position = UDim2.fromOffset(x, 0),
        Size = UDim2.new(1, -x - 70, 1, 0), Font = Enum.Font.GothamMedium, TextSize = 15,
        TextColor3 = Color3.fromRGB(235, 235, 240), TextXAlignment = Enum.TextXAlignment.Left, Text = "",
    }, parent), key)
end

local function addHeader(page, key)
    return bind(new("TextLabel", {
        Size = UDim2.new(1, 0, 0, 20), BackgroundTransparency = 1, Font = Enum.Font.GothamBold,
        TextSize = 12, TextColor3 = GRAY, TextXAlignment = Enum.TextXAlignment.Left,
    }, page), key)
end

local function addNote(page, key, h)
    return bind(new("TextLabel", {
        Size = UDim2.new(1, 0, 0, h or 40), BackgroundTransparency = 1, Font = Enum.Font.Gotham,
        TextSize = 13, TextColor3 = GRAY, TextWrapped = true,
        TextXAlignment = Enum.TextXAlignment.Left, TextYAlignment = Enum.TextYAlignment.Top,
    }, page), key)
end

local function flash(b)
    b.BackgroundTransparency = 0.05
    TweenService:Create(b, TweenInfo.new(0.25), { BackgroundTransparency = 0.35 }):Play()
end

local function addToggle(page, key, default, cb)
    local row = rowBase(page, 42)
    rowLabel(row, key)
    local sw = new("Frame", {
        AnchorPoint = Vector2.new(1, 0.5), Position = UDim2.new(1, -12, 0.5, 0),
        Size = UDim2.fromOffset(42, 22), BackgroundColor3 = OFF_COL, BorderSizePixel = 0,
    }, row)
    corner(sw, 11)
    local knob = new("Frame", {
        AnchorPoint = Vector2.new(0, 0.5), Position = UDim2.new(0, 3, 0.5, 0),
        Size = UDim2.fromOffset(16, 16), BackgroundColor3 = WHITE, BorderSizePixel = 0,
    }, sw)
    corner(knob, 8)
    local hit = new("TextButton", { Size = UDim2.fromScale(1, 1), BackgroundTransparency = 1, Text = "", ZIndex = 5 }, row)

    local state = default and true or false
    local obj = {}
    local function paint(animate)
        local pos = state and UDim2.new(1, -19, 0.5, 0) or UDim2.new(0, 3, 0.5, 0)
        local col = state and accent.Value or OFF_COL
        if animate then
            TweenService:Create(knob, TweenInfo.new(0.15), { Position = pos }):Play()
            TweenService:Create(sw, TweenInfo.new(0.15), { BackgroundColor3 = col }):Play()
        else
            knob.Position = pos
            sw.BackgroundColor3 = col
        end
    end
    function obj.Set(v, silent)
        state = v and true or false
        paint(true)
        if not silent and cb then cb(state) end
    end
    function obj.Get() return state end
    connect(hit.MouseButton1Click, function() obj.Set(not state) end)
    onAccent(function(c) if state then sw.BackgroundColor3 = c end end)
    paint(false)
    return obj
end

local function addButton(page, key, cb)
    local b = rowBase(page, 40, "TextButton", { Text = "", AutoButtonColor = false })
    bind(new("TextLabel", {
        Size = UDim2.fromScale(1, 1), BackgroundTransparency = 1, Font = Enum.Font.GothamBold,
        TextSize = 15, TextColor3 = WHITE, Text = "",
    }, b), key)
    connect(b.MouseButton1Click, function() flash(b); cb() end)
    return b
end

local function addDual(page, keyA, cbA, keyB, cbB)
    local holder = new("Frame", { Size = UDim2.new(1, 0, 0, 40), BackgroundTransparency = 1 }, page)
    local function half(key, cb, pos)
        local b = new("TextButton", {
            Position = pos, Size = UDim2.new(0.5, -3, 1, 0), BackgroundColor3 = ROW_BG,
            BackgroundTransparency = 0.35, BorderSizePixel = 0, Text = "", AutoButtonColor = false,
        }, holder)
        corner(b, 9)
        bind(new("TextLabel", {
            Size = UDim2.fromScale(1, 1), BackgroundTransparency = 1, Font = Enum.Font.GothamBold,
            TextSize = 15, TextColor3 = WHITE, Text = "",
        }, b), key)
        connect(b.MouseButton1Click, function() flash(b); cb() end)
    end
    half(keyA, cbA, UDim2.new(0, 0, 0, 0))
    half(keyB, cbB, UDim2.new(0.5, 3, 0, 0))
end

local function addOption(page, key, swatch, cb)
    local row = rowBase(page, 42, "TextButton", { Text = "", AutoButtonColor = false, BackgroundTransparency = 1 })
    local st = stroke(row, accent.Value, 1.5, 1)
    local x = 14
    if swatch then
        local sq = new("Frame", {
            Position = UDim2.new(0, 12, 0.5, -11), Size = UDim2.fromOffset(22, 22),
            BackgroundColor3 = swatch, BorderSizePixel = 0,
        }, row)
        corner(sq, 6)
        x = 48
    end
    bind(new("TextLabel", {
        Position = UDim2.fromOffset(x, 0), Size = UDim2.new(1, -x - 10, 1, 0), BackgroundTransparency = 1,
        Font = Enum.Font.GothamMedium, TextSize = 16, TextColor3 = WHITE,
        TextXAlignment = Enum.TextXAlignment.Left, Text = "",
    }, row), key)
    local obj = {}
    function obj.Select(v)
        TweenService:Create(row, TweenInfo.new(0.15), { BackgroundTransparency = v and 0.55 or 1 }):Play()
        st.Transparency = v and 0.2 or 1
    end
    onAccent(function(c) st.Color = c end)
    connect(row.MouseButton1Click, cb)
    return obj
end

local function addSlider(page, key, minV, maxV, default, fmt, cb)
    local row = rowBase(page, 56)
    bind(new("TextLabel", {
        Position = UDim2.fromOffset(14, 4), Size = UDim2.new(1, -100, 0, 24), BackgroundTransparency = 1,
        Font = Enum.Font.GothamMedium, TextSize = 15, TextColor3 = Color3.fromRGB(235, 235, 240),
        TextXAlignment = Enum.TextXAlignment.Left, Text = "",
    }, row), key)
    local val = new("TextLabel", {
        AnchorPoint = Vector2.new(1, 0), Position = UDim2.new(1, -14, 0, 4), Size = UDim2.fromOffset(80, 24),
        BackgroundTransparency = 1, Font = Enum.Font.GothamBold, TextSize = 14, TextColor3 = GRAY,
        TextXAlignment = Enum.TextXAlignment.Right, Text = "",
    }, row)
    local track = new("Frame", {
        Position = UDim2.new(0, 14, 0, 38), Size = UDim2.new(1, -28, 0, 6),
        BackgroundColor3 = OFF_COL, BorderSizePixel = 0,
    }, row)
    corner(track, 3)
    local fill = new("Frame", { Size = UDim2.fromScale(0, 1), BackgroundColor3 = accent.Value, BorderSizePixel = 0 }, track)
    corner(fill, 3)
    local knob = new("Frame", {
        AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.fromScale(0, 0.5),
        Size = UDim2.fromOffset(16, 16), BackgroundColor3 = WHITE, BorderSizePixel = 0,
    }, track)
    corner(knob, 8)
    local hit = new("TextButton", {
        Position = UDim2.new(0, 6, 0, 26), Size = UDim2.new(1, -12, 0, 28),
        BackgroundTransparency = 1, Text = "", ZIndex = 5,
    }, row)

    local value = default
    local obj = {}
    local function render()
        local a = (value - minV) / (maxV - minV)
        fill.Size = UDim2.fromScale(a, 1)
        knob.Position = UDim2.fromScale(a, 0.5)
        val.Text = fmt(value)
    end
    local function setFromX(x)
        local w = track.AbsoluteSize.X
        if w <= 0 then return end
        local a = math.clamp((x - track.AbsolutePosition.X) / w, 0, 1)
        value = minV + (maxV - minV) * a
        render()
        if cb then cb(value) end
    end
    function obj.Set(v, silent)
        value = math.clamp(v, minV, maxV)
        render()
        if not silent and cb then cb(value) end
    end
    function obj.Get() return value end

    local dragging = false
    connect(hit.InputBegan, function(input)
        local t = input.UserInputType
        if t == Enum.UserInputType.MouseButton1 or t == Enum.UserInputType.Touch then
            dragging = true
            page.ScrollingEnabled = false
            setFromX(input.Position.X)
        end
    end)
    connect(UserInputService.InputChanged, function(input)
        local t = input.UserInputType
        if dragging and (t == Enum.UserInputType.MouseMovement or t == Enum.UserInputType.Touch) then
            setFromX(input.Position.X)
        end
    end)
    connect(UserInputService.InputEnded, function(input)
        local t = input.UserInputType
        if dragging and (t == Enum.UserInputType.MouseButton1 or t == Enum.UserInputType.Touch) then
            dragging = false
            page.ScrollingEnabled = true
        end
    end)
    onAccent(function(c) fill.BackgroundColor3 = c end)
    render()
    return obj
end

local function addInfo(page, key, value)
    local row = rowBase(page, 40)
    rowLabel(row, key)
    local v = new("TextLabel", {
        AnchorPoint = Vector2.new(1, 0.5), Position = UDim2.new(1, -14, 0.5, 0), Size = UDim2.fromOffset(60, 24),
        BackgroundTransparency = 1, Font = Enum.Font.GothamBold, TextSize = 15,
        TextXAlignment = Enum.TextXAlignment.Right, Text = value or "—", TextColor3 = accent.Value,
    }, row)
    onAccent(function(c) v.TextColor3 = c end)
    return { Set = function(t) v.Text = t end }
end

----------------------------------------------------------------------
-- Страницы
----------------------------------------------------------------------
local themePage    = newPage()
local worldPage    = newPage()
local shaderPage   = newPage()
local charPage     = newPage()
local settingsPage = newPage()

-- Темы ---------------------------------------------------------------
local themeOpts = {}
local function selectTheme(i)
    for j, o in ipairs(themeOpts) do o.Select(j == i) end
    cfg.theme = i
    TweenService:Create(accent, TweenInfo.new(0.45, Enum.EasingStyle.Quad), { Value = THEMES[i].color }):Play()
    saveConfig(true)
end
for i, t in ipairs(THEMES) do
    themeOpts[i] = addOption(themePage, t.key, t.color, function() selectTheme(i) end)
end
themeOpts[cfg.theme].Select(true)

-- Мир: день/ночь + дождь --------------------------------------------
local lockToggle, timeSlider, rainToggle, rainSlider

addHeader(worldPage, "hd_time")
lockToggle = addToggle(worldPage, "time_lock", false, function(v)
    timeState.lock = v
    if v then timeState.value = timeSlider.Get() end
end)
addDual(worldPage,
    "day", function()
        timeSlider.Set(14, true); timeState.value = 14
        lockToggle.Set(true)
    end,
    "night", function()
        timeSlider.Set(0, true); timeState.value = 0
        lockToggle.Set(true)
    end)
timeSlider = addSlider(worldPage, "time", 0, 24, ORIG.ClockTime % 24, function(v)
    return string.format("%02d:%02d", math.floor(v) % 24, math.floor((v % 1) * 60))
end, function(v)
    timeState.value = v
    if not lockToggle.Get() then lockToggle.Set(true) end
end)

addHeader(worldPage, "hd_rain")
rainToggle = addToggle(worldPage, "rain", false, function(v) setRain(v) end)
rainSlider = addSlider(worldPage, "rain_int", 100, 2000, rain.rate, function(v)
    return tostring(math.floor(v))
end, function(v)
    rain.rate = math.floor(v)
    if rainEmitter then rainEmitter.Rate = rain.rate end
end)

-- Шейдеры -----------------------------------------------------------
local shaderOpts = {}
local function selectShader(idx, name)
    for j, o in pairs(shaderOpts) do o.Select(j == idx) end
    applyShader(name)
end
shaderOpts[0] = addOption(shaderPage, "sh_off", nil, function() selectShader(0, nil) end)
for i, name in ipairs(SHADER_ORDER) do
    shaderOpts[i] = addOption(shaderPage, "sh_" .. name, nil, function() selectShader(i, name) end)
end
shaderOpts[0].Select(true)

-- Персонаж ----------------------------------------------------------
local rigInfo
local headlessToggle, korbloxToggle
headlessToggle = addToggle(charPage, "headless", false, function(v)
    charState.headless = v
    local c = LocalPlayer.Character
    if c then applyHeadless(c, v) end
end)
korbloxToggle = addToggle(charPage, "korblox", false, function(v)
    charState.korblox = v
    local c = LocalPlayer.Character
    if c then applyKorblox(c, v) end
end)
rigInfo = addInfo(charPage, "rig", "—")
addNote(charPage, "char_note", 44)

local function updateRig(char)
    local hum = char and char:FindFirstChildOfClass("Humanoid")
    if hum then
        rigInfo.Set(hum.RigType == Enum.HumanoidRigType.R6 and "R6" or "R15")
    else
        rigInfo.Set("—")
    end
end

local function onCharacter(char)
    char:WaitForChild("Humanoid", 8)
    char:WaitForChild("Head", 8)
    task.wait(0.6)
    if not gui.Parent then return end
    updateRig(char)
    if charState.headless then applyHeadless(char, true) end
    if charState.korblox then applyKorblox(char, true) end
end
connect(LocalPlayer.CharacterAdded, function(c) task.spawn(onCharacter, c) end)
if LocalPlayer.Character then task.spawn(onCharacter, LocalPlayer.Character) end

-- Настройки ---------------------------------------------------------
local animToggle
animToggle = addToggle(settingsPage, "anim_grad", cfg.anim and true or false, function(v)
    cfg.anim = v
    if not v then
        TweenService:Create(grad, TweenInfo.new(0.3), { Rotation = 35 }):Play()
    end
    saveConfig(true)
end)

local function resetEffects()
    setRain(false)
    rainToggle.Set(false, true)
    timeState.lock = false
    lockToggle.Set(false, true)
    selectShader(0, nil)
    restoreLighting()
    toast("reset_done")
end
addButton(settingsPage, "reset", resetEffects)
addNote(settingsPage, "set_note", 40)

----------------------------------------------------------------------
-- Вкладки
----------------------------------------------------------------------
local TABS = {
    { key = "themes",    icon = "🎨", page = themePage },
    { key = "world",     icon = "🌤", page = worldPage },
    { key = "shaders",   icon = "✨", page = shaderPage },
    { key = "character", icon = "👤", page = charPage },
    { key = "settings",  icon = "⚙",  page = settingsPage },
}
local currentTab = 1

local function selectTab(i)
    currentTab = i
    for j, t in ipairs(TABS) do
        t.page.Visible = (j == i)
        TweenService:Create(t.btn, TweenInfo.new(0.15), { TextTransparency = (j == i) and 0 or 0.55 }):Play()
    end
    TweenService:Create(indicator, TweenInfo.new(0.2, Enum.EasingStyle.Quad),
        { Position = UDim2.fromOffset(0, 8 + (i - 1) * 56 + 12) }):Play()
    sectionTitle.Text = tr(TABS[i].key)
end

for i, t in ipairs(TABS) do
    t.btn = new("TextButton", {
        Position = UDim2.fromOffset(0, 8 + (i - 1) * 56), Size = UDim2.fromOffset(68, 50),
        BackgroundTransparency = 1, Text = t.icon, TextSize = 24, Font = Enum.Font.GothamBold,
        TextColor3 = WHITE, TextTransparency = 0.55, AutoButtonColor = false,
    }, sidebar)
    connect(t.btn.MouseButton1Click, function() selectTab(i) end)
end

----------------------------------------------------------------------
-- Язык
----------------------------------------------------------------------
local function setLang(l)
    lang = l
    cfg.lang = l
    for _, b in ipairs(bindings) do
        b[1][b[3]] = tr(b[2])
    end
    sectionTitle.Text = tr(TABS[currentTab].key)
    paintLang()
    saveConfig(true)
end
connect(enBtn.MouseButton1Click, function() setLang("EN") end)
connect(ruBtn.MouseButton1Click, function() setLang("RU") end)

----------------------------------------------------------------------
-- Кнопки шапки, сворачивание, закрытие, перетаскивание
----------------------------------------------------------------------
local bubble = new("TextButton", {
    Name = "Bubble", AnchorPoint = Vector2.new(0, 0.5), Position = UDim2.new(0, 14, 0.5, 0),
    Size = UDim2.fromOffset(46, 46), BackgroundColor3 = accent.Value, Text = "🦊", TextSize = 24,
    Font = Enum.Font.GothamBold, Visible = false, AutoButtonColor = false, BorderSizePixel = 0,
    ClipsDescendants = true,
}, gui)
corner(bubble, 12)
stroke(bubble, WHITE, 1, 0.7)
onAccent(function(c) bubble.BackgroundColor3 = c end)

local function setMinimized(v)
    main.Visible = not v
    bubble.Visible = v
    if not v then
        uiScale.Scale = baseScale * 0.9
        TweenService:Create(uiScale, TweenInfo.new(0.25, Enum.EasingStyle.Back, Enum.EasingDirection.Out),
            { Scale = baseScale }):Play()
    end
end
local bubbleDragging, bubbleMoved, bubbleDragStart, bubbleStartPos
connect(bubble.InputBegan, function(input)
    local t = input.UserInputType
    if t == Enum.UserInputType.MouseButton1 or t == Enum.UserInputType.Touch then
        bubbleDragging = true
        bubbleMoved = false
        bubbleDragStart = input.Position
        bubbleStartPos = bubble.Position
    end
end)
connect(UserInputService.InputChanged, function(input)
    local t = input.UserInputType
    if bubbleDragging and (t == Enum.UserInputType.MouseMovement or t == Enum.UserInputType.Touch) then
        local d = input.Position - bubbleDragStart
        if math.abs(d.X) > 4 or math.abs(d.Y) > 4 then bubbleMoved = true end
        bubble.Position = UDim2.new(bubbleStartPos.X.Scale, bubbleStartPos.X.Offset + d.X, bubbleStartPos.Y.Scale, bubbleStartPos.Y.Offset + d.Y)
    end
end)
connect(UserInputService.InputEnded, function(input)
    local t = input.UserInputType
    if bubbleDragging and (t == Enum.UserInputType.MouseButton1 or t == Enum.UserInputType.Touch) then
        bubbleDragging = false
        if not bubbleMoved then setMinimized(false) end
    end
end)

local function cleanup()
    pcall(setRain, false)
    pcall(applyShader, nil)
    timeState.lock = false
    pcall(restoreLighting)
    charState.headless = false
    charState.korblox = false
    local c = LocalPlayer.Character
    if c then
        pcall(applyHeadless, c, false)
        pcall(applyKorblox, c, false)
    end
    for _, cn in ipairs(conns) do
        pcall(function() cn:Disconnect() end)
    end
    if rainPart then pcall(function() rainPart:Destroy() end) end
    pcall(function() gui:Destroy() end)
    genv.__HubCleanup = nil
end
genv.__HubCleanup = cleanup

headerBtn("✕", -10, cleanup)
headerBtn("—", -46, function() setMinimized(true) end)
headerBtn("📁", -82, function() saveConfig(false) end)
headerBtn("⚙", -118, function() selectTab(5) end)

-- перетаскивание за шапку (мышь и тач)
local dragging, dragStart, startPos = false, nil, nil
connect(header.InputBegan, function(input)
    local t = input.UserInputType
    if t == Enum.UserInputType.MouseButton1 or t == Enum.UserInputType.Touch then
        dragging = true
        dragStart = input.Position
        startPos = main.Position
    end
end)
connect(UserInputService.InputChanged, function(input)
    local t = input.UserInputType
    if dragging and (t == Enum.UserInputType.MouseMovement or t == Enum.UserInputType.Touch) then
        local d = input.Position - dragStart
        main.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + d.X, startPos.Y.Scale, startPos.Y.Offset + d.Y)
    end
end)
connect(UserInputService.InputEnded, function(input)
    local t = input.UserInputType
    if t == Enum.UserInputType.MouseButton1 or t == Enum.UserInputType.Touch then
        dragging = false
    end
end)

----------------------------------------------------------------------
-- Главный цикл
----------------------------------------------------------------------
local acc = 0
connect(RunService.RenderStepped, function(dt)
    if timeState.lock then
        Lighting.ClockTime = timeState.value
    end
    if rain.on and rainPart then
        local cam = Workspace.CurrentCamera
        if cam then
            rainPart.CFrame = CFrame.new(cam.CFrame.Position + Vector3.new(0, 50, 0))
        end
    end
    if cfg.anim then
        grad.Rotation = (grad.Rotation + dt * 25) % 360
    end

    -- раз в ~1.5 сек. переприменяем эффекты персонажа (если игра их сбросила)
    acc = acc + dt
    if acc >= 1.5 then
        acc = 0
        local c = LocalPlayer.Character
        if c then
            if charState.headless then applyHeadless(c, true) end
            if charState.korblox then applyKorblox(c, true) end
        end
    end
end)

----------------------------------------------------------------------
-- Старт
----------------------------------------------------------------------
paintLang()
selectTab(1)
uiScale.Scale = baseScale * 0.9
TweenService:Create(uiScale, TweenInfo.new(0.3, Enum.EasingStyle.Back, Enum.EasingDirection.Out),
    { Scale = baseScale }):Play()

----------------------------------------------------------------------
-- ДОПОЛНИТЕЛЬНЫЕ ФУНКЦИИ (добавлено отдельным блоком, ничего выше не трогали)
----------------------------------------------------------------------
L.hd_extra   = { EN = "Extra",       RU = "Доп. функции" }
L.antiafk    = { EN = "Anti-AFK",    RU = "Анти-АФК" }
L.fullbright = { EN = "Fullbright",  RU = "Полная яркость" }
L.cam_fov    = { EN = "Camera FOV",  RU = "Обзор камеры (FOV)" }
L.fps_cap    = { EN = "FPS cap",     RU = "Лимит FPS" }
L.extra_note = {
    EN = "Anti-AFK stops idle kicks. FPS cap only works if your executor supports it.",
    RU = "Анти-АФК не даёт кикнуть за бездействие. Лимит FPS работает только если это поддерживает твой запускатор.",
}

local VirtualUser = game:GetService("VirtualUser")
local extraState = { antiafk = false }

addHeader(settingsPage, "hd_extra")

addToggle(settingsPage, "antiafk", false, function(v)
    extraState.antiafk = v
end)

addToggle(settingsPage, "fullbright", false, function(v)
    if v then
        setProp(Lighting, "Brightness", 3)
        setProp(Lighting, "OutdoorAmbient", Color3.fromRGB(180, 180, 180))
        setProp(Lighting, "Ambient", Color3.fromRGB(180, 180, 180))
        setProp(Lighting, "GlobalShadows", false)
    else
        restoreProps(Lighting)
    end
end)

local camFovSlider = addSlider(settingsPage, "cam_fov", 40, 120,
    (Workspace.CurrentCamera and Workspace.CurrentCamera.FieldOfView) or 70,
    function(v) return string.format("%d°", v) end,
    function(v)
        local cam = Workspace.CurrentCamera
        if cam then cam.FieldOfView = v end
    end)

addSlider(settingsPage, "fps_cap", 30, 240, 240,
    function(v) return tostring(math.floor(v)) end,
    function(v)
        if setfpscap then pcall(setfpscap, math.floor(v)) end
    end)

addNote(settingsPage, "extra_note", 46)

-- анти-афк: лёгкая имитация действия при простое
connect(LocalPlayer.Idled, function()
    if extraState.antiafk then
        pcall(function()
            VirtualUser:CaptureController()
            VirtualUser:ClickButton2(Vector2.new())
        end)
    end
end)

-- держим обзор камеры после респавна
connect(LocalPlayer.CharacterAdded, function()
    task.wait(1)
    local cam = Workspace.CurrentCamera
    if cam then cam.FieldOfView = camFovSlider.Get() end
end)

-- аккуратно возвращаем яркость при закрытии меню, не трогая исходный cleanup()
local prevCleanup = genv.__HubCleanup
genv.__HubCleanup = function()
    pcall(restoreProps, Lighting)
    if prevCleanup then prevCleanup() end
end

----------------------------------------------------------------------
-- ВИЗУАЛЬНЫЕ АКСЕССУАРЫ: нимб, крылья, аура, подсветка контура (только у тебя)
----------------------------------------------------------------------
L.hd_cosmetics   = { EN = "Cosmetics",    RU = "Аксессуары" }
L.wings          = { EN = "Wings",        RU = "Крылья" }
L.halo           = { EN = "Halo",         RU = "Нимб" }
L.aura           = { EN = "Aura",         RU = "Аура" }
L.outline        = { EN = "Outline glow", RU = "Подсветка контура" }
L.cosmetics_note = {
    EN = "Paste a catalog asset ID above each toggle, then switch it on. Visible only to you, re-applies after respawn.",
    RU = "Вставь ID предмета с каталога над нужным переключателем и включи его. Видно только тебе, само возвращается после респавна.",
}
L.asset_fail = {
    EN = "Couldn't load that asset ID (check the number, or your executor may block InsertService)",
    RU = "Не удалось загрузить этот ID (проверь число, либо твой запускатор блокирует InsertService)",
}

charState.wings   = false
charState.halo    = false
charState.aura    = false
charState.outline = false

local rigStore = setmetatable({}, { __mode = "k" }) -- [char] = { wings=Model, halo=Part, aura=Attachment, outline=Highlight }

local function rigOf(char)
    local r = rigStore[char]
    if not r then r = {}; rigStore[char] = r end
    return r
end

local function cosmeticAnchors(char)
    local hum = char:FindFirstChildOfClass("Humanoid")
    local r6 = hum and hum.RigType == Enum.HumanoidRigType.R6
    local torso = char:FindFirstChild(r6 and "Torso" or "UpperTorso")
    local head = char:FindFirstChild("Head")
    return torso, head
end

local InsertService = game:GetService("InsertService")

-- грузит настоящий предмет с маркетплейса по ID и цепляет его к персонажу
local function loadCatalogAsset(id, anchor, offset)
    local n = tonumber(id)
    if not n then return nil end
    local ok, holder = pcall(function() return InsertService:LoadAsset(n) end)
    if not ok or not holder then return nil end
    local accessory = holder:FindFirstChildWhichIsA("Accessory", true)
    if accessory then
        accessory.Parent = nil
        pcall(function() holder:Destroy() end)
        return accessory
    end
    local main = holder.PrimaryPart or holder:FindFirstChildWhichIsA("BasePart", true)
    if not main or not anchor then
        pcall(function() holder:Destroy() end)
        return nil
    end
    local relatives = {}
    for _, d in ipairs(holder:GetDescendants()) do
        if d:IsA("BasePart") then
            d.Anchored = false
            d.CanCollide = false
            d.CanQuery = false
            d.CanTouch = false
            if d ~= main then relatives[d] = main.CFrame:ToObjectSpace(d.CFrame) end
        end
    end
    main.CFrame = anchor.CFrame * (offset or CFrame.new())
    for d, rel in pairs(relatives) do
        d.CFrame = main.CFrame * rel
    end
    new("WeldConstraint", { Part0 = anchor, Part1 = main }, main)
    for d in pairs(relatives) do
        new("WeldConstraint", { Part0 = main, Part1 = d }, d)
    end
    holder.Name = "HubAsset"
    return holder
end

local function removeWings(char)
    local r = rigOf(char)
    if r.wings then r.wings:Destroy(); r.wings = nil end
end
local function addWings(char, id)
    removeWings(char)
    local hum = char:FindFirstChildOfClass("Humanoid")
    local torso = cosmeticAnchors(char)
    if not hum or not torso then return false end
    local inst = loadCatalogAsset(id, torso, CFrame.new(0, 0.3, 0.6))
    if not inst then return false end
    if inst:IsA("Accessory") then hum:AddAccessory(inst) else inst.Parent = char end
    rigOf(char).wings = inst
    return true
end

local function removeHalo(char)
    local r = rigOf(char)
    if r.halo then r.halo:Destroy(); r.halo = nil end
end
local function addHalo(char, id)
    removeHalo(char)
    local hum = char:FindFirstChildOfClass("Humanoid")
    local _, head = cosmeticAnchors(char)
    if not hum or not head then return false end
    local inst = loadCatalogAsset(id, head, CFrame.new(0, 0.6, 0))
    if not inst then return false end
    if inst:IsA("Accessory") then hum:AddAccessory(inst) else inst.Parent = char end
    rigOf(char).halo = inst
    return true
end

local function removeAura(char)
    local r = rigOf(char)
    if r.aura then r.aura:Destroy(); r.aura = nil end
end
local function addAura(char, id)
    removeAura(char)
    local hum = char:FindFirstChildOfClass("Humanoid")
    local torso = cosmeticAnchors(char)
    if not hum or not torso then return false end
    local inst = loadCatalogAsset(id, torso, CFrame.new(0, -1, 0))
    if not inst then return false end
    if inst:IsA("Accessory") then hum:AddAccessory(inst) else inst.Parent = char end
    rigOf(char).aura = inst
    return true
end

local function removeOutline(char)
    local r = rigOf(char)
    if r.outline then r.outline:Destroy(); r.outline = nil end
end
local function addOutline(char)
    removeOutline(char)
    local h = new("Highlight", {
        Name = "HubOutline", FillTransparency = 1,
        OutlineColor = accent.Value, OutlineTransparency = 0,
        DepthMode = Enum.HighlightDepthMode.AlwaysOnTop,
    }, char)
    rigOf(char).outline = h
end

local wingsBox, haloBox, auraBox

local function applyCosmetics(char)
    if charState.wings then addWings(char, wingsBox.Text) end
    if charState.halo then addHalo(char, haloBox.Text) end
    if charState.aura then addAura(char, auraBox.Text) end
    if charState.outline then addOutline(char) end
end

-- цвет темы применяется только к подсветке контура (у настоящих ассетов свой вид)
onAccent(function(c)
    for _, r in pairs(rigStore) do
        if r.outline then r.outline.OutlineColor = c end
    end
end)

addHeader(charPage, "hd_cosmetics")

local function idRow(page)
    local row = rowBase(page, 38)
    local box = new("TextBox", {
        Position = UDim2.fromOffset(12, 5), Size = UDim2.new(1, -24, 0, 28),
        BackgroundColor3 = Color3.fromRGB(26, 26, 30), BorderSizePixel = 0,
        Font = Enum.Font.Gotham, TextSize = 13, TextColor3 = WHITE,
        PlaceholderText = "1234567890", PlaceholderColor3 = GRAY,
        ClearTextOnFocus = false, Text = "",
    }, row)
    corner(box, 7)
    new("UIPadding", { PaddingLeft = UDim.new(0, 8), PaddingRight = UDim.new(0, 8) }, box)
    return box
end

wingsBox = idRow(charPage)
local wingsToggle
wingsToggle = addToggle(charPage, "wings", false, function(v)
    local c = LocalPlayer.Character
    if v then
        local ok = c and addWings(c, wingsBox.Text)
        if not ok then wingsToggle.Set(false, true); toast("asset_fail"); return end
        charState.wings = true
    else
        charState.wings = false
        if c then removeWings(c) end
    end
end)

haloBox = idRow(charPage)
local haloToggle
haloToggle = addToggle(charPage, "halo", false, function(v)
    local c = LocalPlayer.Character
    if v then
        local ok = c and addHalo(c, haloBox.Text)
        if not ok then haloToggle.Set(false, true); toast("asset_fail"); return end
        charState.halo = true
    else
        charState.halo = false
        if c then removeHalo(c) end
    end
end)

auraBox = idRow(charPage)
local auraToggle
auraToggle = addToggle(charPage, "aura", false, function(v)
    local c = LocalPlayer.Character
    if v then
        local ok = c and addAura(c, auraBox.Text)
        if not ok then auraToggle.Set(false, true); toast("asset_fail"); return end
        charState.aura = true
    else
        charState.aura = false
        if c then removeAura(c) end
    end
end)

addToggle(charPage, "outline", false, function(v)
    charState.outline = v
    local c = LocalPlayer.Character
    if c then if v then addOutline(c) else removeOutline(c) end end
end)
addNote(charPage, "cosmetics_note", 46)

connect(LocalPlayer.CharacterAdded, function(char)
    task.wait(1)
    if gui.Parent then applyCosmetics(char) end
end)
if LocalPlayer.Character then applyCosmetics(LocalPlayer.Character) end

----------------------------------------------------------------------
-- ЛОКАЛЬНАЯ МУЗЫКА (звучит только у тебя)
----------------------------------------------------------------------
L.hd_music   = { EN = "Music",    RU = "Музыка" }
L.music_play = { EN = "▶ Play",   RU = "▶ Играть" }
L.music_stop = { EN = "■ Stop",   RU = "■ Стоп" }
L.music_vol  = { EN = "Volume",   RU = "Громкость" }
L.music_note = {
    EN = "Paste a Roblox audio asset ID above. Only you can hear it.",
    RU = "Вставь ID аудио-ассета Roblox выше. Слышишь только ты.",
}

local musicSound = new("Sound", { Name = "HubMusic", Looped = true, Volume = 0.5 })
musicSound.Parent = LocalPlayer:WaitForChild("PlayerGui")

addHeader(settingsPage, "hd_music")
local musicRow = rowBase(settingsPage, 42)
local musicBox = new("TextBox", {
    Position = UDim2.fromOffset(12, 6), Size = UDim2.new(1, -24, 0, 30),
    BackgroundColor3 = Color3.fromRGB(26, 26, 30), BorderSizePixel = 0,
    Font = Enum.Font.Gotham, TextSize = 14, TextColor3 = WHITE,
    PlaceholderText = "1234567890", PlaceholderColor3 = GRAY,
    ClearTextOnFocus = false, Text = "",
}, musicRow)
corner(musicBox, 7)
new("UIPadding", { PaddingLeft = UDim.new(0, 8), PaddingRight = UDim.new(0, 8) }, musicBox)

addDual(settingsPage,
    "music_play", function()
        local id = musicBox.Text:match("%d+")
        if id then
            musicSound:Stop()
            musicSound.SoundId = "rbxassetid://" .. id
            musicSound.TimePosition = 0
            musicSound:Play()
        end
    end,
    "music_stop", function()
        musicSound:Stop()
    end)

addSlider(settingsPage, "music_vol", 0, 100, 50, function(v)
    return math.floor(v) .. "%"
end, function(v)
    musicSound.Volume = v / 100
end)
addNote(settingsPage, "music_note", 34)

-- аккуратно убираем новые аксессуары и музыку при закрытии меню
local prevCleanup2 = genv.__HubCleanup
genv.__HubCleanup = function()
    local c = LocalPlayer.Character
    if c then
        pcall(removeWings, c)
        pcall(removeHalo, c)
        pcall(removeAura, c)
        pcall(removeOutline, c)
    end
    pcall(function() musicSound:Stop(); musicSound:Destroy() end)
    if prevCleanup2 then prevCleanup2() end
end

----------------------------------------------------------------------
-- РАЗМЫТИЕ ФОНА (при открытии/сворачивании) + ГОРЯЧАЯ КЛАВИША
----------------------------------------------------------------------
local menuBlur = new("BlurEffect", { Name = "HubBlur", Size = 0 }, Lighting)
menuBlur.Size = main.Visible and 14 or 0

connect(main:GetPropertyChangedSignal("Visible"), function()
    TweenService:Create(menuBlur, TweenInfo.new(0.25), { Size = main.Visible and 14 or 0 }):Play()
end)

connect(UserInputService.InputBegan, function(input, gpe)
    if gpe then return end
    if input.KeyCode == Enum.KeyCode.RightShift then
        setMinimized(main.Visible)
    end
end)

local prevCleanup3 = genv.__HubCleanup
genv.__HubCleanup = function()
    pcall(function() menuBlur:Destroy() end)
    if prevCleanup3 then prevCleanup3() end
end

----------------------------------------------------------------------
-- НОВЫЙ БЛОК: свой HEX-цвет, FPS/пинг/часы, чистый экран, громкость,
-- ручной масштаб, шлейф частиц, табличка над головой, чат-пузыри
----------------------------------------------------------------------
L.hd_custom_color = { EN = "Custom color",  RU = "Свой цвет" }
L.apply           = { EN = "Apply",         RU = "Применить" }
L.outline_color   = { EN = "Outline color", RU = "Цвет подсветки" }
L.bad_hex         = { EN = "Invalid HEX color (use RRGGBB)", RU = "Неверный HEX-цвет (формат RRGGBB)" }
L.hd_trail        = { EN = "Particle trail", RU = "Шлейф частиц" }
L.trail           = { EN = "Trail",         RU = "Шлейф" }
L.hd_nametag      = { EN = "Nametag",       RU = "Табличка над головой" }
L.nametag         = { EN = "Nametag",       RU = "Табличка" }
L.hd_display      = { EN = "Display",       RU = "Экран" }
L.clean_screen    = { EN = "Clean screen (hide chat/playerlist)", RU = "Чистый экран (скрыть чат/список)" }
L.game_vol        = { EN = "Game volume",   RU = "Громкость игры" }
L.manual_scale    = { EN = "Manual menu scale", RU = "Ручной масштаб меню" }
L.scale_val       = { EN = "Scale",         RU = "Масштаб" }
L.overlay_stats   = { EN = "FPS / ping / clock overlay", RU = "Оверлей FPS/пинг/часы" }
L.chat_bubbles    = { EN = "Themed chat bubbles", RU = "Чат-пузыри в цвет темы" }

local StarterGui = game:GetService("StarterGui")
local TextChatService = game:GetService("TextChatService")
local Stats = game:GetService("Stats")

local function hexToColor3(hex)
    hex = tostring(hex or ""):gsub("^#", ""):gsub("%s", "")
    if #hex ~= 6 then return nil end
    local r = tonumber(hex:sub(1, 2), 16)
    local g = tonumber(hex:sub(3, 4), 16)
    local b = tonumber(hex:sub(5, 6), 16)
    if not (r and g and b) then return nil end
    return Color3.fromRGB(r, g, b)
end

-- свой цвет темы (HEX)
addHeader(themePage, "hd_custom_color")
local themeHexBox = idRow(themePage)
themeHexBox.PlaceholderText = "RRGGBB"
addButton(themePage, "apply", function()
    local c = hexToColor3(themeHexBox.Text)
    if not c then toast("bad_hex"); return end
    for _, o in ipairs(themeOpts) do o.Select(false) end
    TweenService:Create(accent, TweenInfo.new(0.45, Enum.EasingStyle.Quad), { Value = c }):Play()
end)

-- свой цвет подсветки контура, отдельно от темы
local outlineCustomColor = nil
local baseAddOutline = addOutline
addOutline = function(char)
    baseAddOutline(char)
    if outlineCustomColor then
        local r = rigOf(char)
        if r.outline then r.outline.OutlineColor = outlineCustomColor end
    end
end

addHeader(charPage, "outline_color")
local outlineHexBox = idRow(charPage)
outlineHexBox.PlaceholderText = "RRGGBB"
addButton(charPage, "apply", function()
    local c = hexToColor3(outlineHexBox.Text)
    if not c then toast("bad_hex"); return end
    outlineCustomColor = c
    local ch = LocalPlayer.Character
    if ch then
        local r = rigOf(ch)
        if r.outline then r.outline.OutlineColor = c end
    end
end)

-- шлейф частиц за персонажем
local function removeTrail(char)
    local r = rigOf(char)
    if r.trail then
        for _, inst in ipairs(r.trail) do pcall(function() inst:Destroy() end) end
        r.trail = nil
    end
end
local function addTrail(char)
    removeTrail(char)
    local torso = cosmeticAnchors(char)
    if not torso then return end
    local a0 = new("Attachment", { Position = Vector3.new(-0.4, 0.2, 0) }, torso)
    local a1 = new("Attachment", { Position = Vector3.new(0.4, 0.2, 0) }, torso)
    local trailInst = new("Trail", {
        Attachment0 = a0, Attachment1 = a1,
        Color = ColorSequence.new(accent.Value),
        Transparency = NumberSequence.new({
            NumberSequenceKeypoint.new(0, 0.3),
            NumberSequenceKeypoint.new(1, 1),
        }),
        Lifetime = 0.5,
    }, torso)
    rigOf(char).trail = { a0, a1, trailInst }
end
charState.trail = false
addHeader(charPage, "hd_trail")
addToggle(charPage, "trail", false, function(v)
    charState.trail = v
    local c = LocalPlayer.Character
    if c then if v then addTrail(c) else removeTrail(c) end end
end)

-- табличка с ником над головой
local function removeNametag(char)
    local r = rigOf(char)
    if r.nametag then r.nametag:Destroy(); r.nametag = nil end
end
local function addNametag(char)
    removeNametag(char)
    local _, head = cosmeticAnchors(char)
    if not head then return end
    local bgui = new("BillboardGui", {
        Name = "HubNametag", Adornee = head, Size = UDim2.fromOffset(160, 34),
        StudsOffset = Vector3.new(0, 2.1, 0), AlwaysOnTop = true,
    }, head)
    new("TextLabel", {
        Size = UDim2.fromScale(1, 1), BackgroundTransparency = 1, Text = LocalPlayer.Name,
        Font = Enum.Font.GothamBlack, TextSize = 18, TextColor3 = accent.Value,
        TextStrokeTransparency = 0.4, TextStrokeColor3 = Color3.new(0, 0, 0),
    }, bgui)
    rigOf(char).nametag = bgui
end
charState.nametag = false
addHeader(charPage, "hd_nametag")
addToggle(charPage, "nametag", false, function(v)
    charState.nametag = v
    local c = LocalPlayer.Character
    if c then if v then addNametag(c) else removeNametag(c) end end
end)

-- живой цвет темы для шлейфа и таблички
onAccent(function(c)
    for _, r in pairs(rigStore) do
        if r.nametag then
            local lbl = r.nametag:FindFirstChildOfClass("TextLabel")
            if lbl then lbl.TextColor3 = c end
        end
        if r.trail and r.trail[3] then r.trail[3].Color = ColorSequence.new(c) end
    end
end)

-- дозаряжаем шлейф/табличку после респавна, не трогая старую функцию
local baseApplyCosmetics = applyCosmetics
applyCosmetics = function(char)
    baseApplyCosmetics(char)
    if charState.trail then addTrail(char) end
    if charState.nametag then addNametag(char) end
end

-- чистый экран
addHeader(settingsPage, "hd_display")
addToggle(settingsPage, "clean_screen", false, function(v)
    pcall(function() StarterGui:SetCoreGuiEnabled(Enum.CoreGuiType.Chat, not v) end)
    pcall(function() StarterGui:SetCoreGuiEnabled(Enum.CoreGuiType.PlayerList, not v) end)
end)

-- общая громкость игры
local soundVolMult = 1
local soundBase = setmetatable({}, { __mode = "k" })
local function trackSoundVol(s)
    if not s:IsA("Sound") then return end
    if soundBase[s] == nil then soundBase[s] = s.Volume end
    s.Volume = soundBase[s] * soundVolMult
end
task.spawn(function()
    local list = game:GetDescendants()
    for i = 1, #list do
        local d = list[i]
        if d and d:IsA("Sound") then
            if soundBase[d] == nil then soundBase[d] = d.Volume end
            d.Volume = soundBase[d] * soundVolMult
        end
        if i % 250 == 0 then task.wait() end
    end
end)
connect(game.DescendantAdded, function(d) pcall(trackSoundVol, d) end)
addSlider(settingsPage, "game_vol", 0, 100, 100, function(v)
    return math.floor(v) .. "%"
end, function(v)
    soundVolMult = v / 100
    for s, base in pairs(soundBase) do
        if s.Parent then pcall(function() s.Volume = base * soundVolMult end) end
    end
end)

-- ручной масштаб меню (кроме авто-подгонки)
local manualScaleOn = false
local manualScaleVal = baseScale
addToggle(settingsPage, "manual_scale", false, function(v)
    manualScaleOn = v
    if v then uiScale.Scale = manualScaleVal else fit() end
end)
addSlider(settingsPage, "scale_val", 40, 150, math.floor(baseScale * 100), function(v)
    return math.floor(v) .. "%"
end, function(v)
    manualScaleVal = v / 100
    if manualScaleOn then uiScale.Scale = manualScaleVal end
end)
if Workspace.CurrentCamera then
    connect(Workspace.CurrentCamera:GetPropertyChangedSignal("ViewportSize"), function()
        if manualScaleOn then uiScale.Scale = manualScaleVal end
    end)
end

-- оверлей FPS / пинг / часы
local overlayGui = new("Frame", {
    Name = "HubOverlay", Position = UDim2.fromOffset(12, 12), Size = UDim2.fromOffset(120, 74),
    BackgroundColor3 = Color3.fromRGB(15, 15, 18), BackgroundTransparency = 0.25,
    BorderSizePixel = 0, Visible = false,
}, gui)
corner(overlayGui, 10)
local overlayFps = new("TextLabel", {
    Position = UDim2.fromOffset(10, 6), Size = UDim2.new(1, -20, 0, 20), BackgroundTransparency = 1,
    Font = Enum.Font.GothamBold, TextSize = 14, TextColor3 = WHITE,
    TextXAlignment = Enum.TextXAlignment.Left, Text = "FPS: --",
}, overlayGui)
local overlayPing = new("TextLabel", {
    Position = UDim2.fromOffset(10, 26), Size = UDim2.new(1, -20, 0, 20), BackgroundTransparency = 1,
    Font = Enum.Font.Gotham, TextSize = 13, TextColor3 = GRAY,
    TextXAlignment = Enum.TextXAlignment.Left, Text = "Ping: --",
}, overlayGui)
local overlayClock = new("TextLabel", {
    Position = UDim2.fromOffset(10, 46), Size = UDim2.new(1, -20, 0, 20), BackgroundTransparency = 1,
    Font = Enum.Font.Gotham, TextSize = 13, TextColor3 = GRAY,
    TextXAlignment = Enum.TextXAlignment.Left, Text = "--:--",
}, overlayGui)
addToggle(settingsPage, "overlay_stats", false, function(v)
    overlayGui.Visible = v
end)
local overlayAcc, overlayFrames = 0, 0
connect(RunService.RenderStepped, function(dt)
    if not overlayGui.Visible then return end
    overlayFrames = overlayFrames + 1
    overlayAcc = overlayAcc + dt
    if overlayAcc >= 0.5 then
        overlayFps.Text = string.format("FPS: %d", math.floor(overlayFrames / overlayAcc + 0.5))
        overlayFrames, overlayAcc = 0, 0
        local okp, pingVal = pcall(function()
            return Stats.Network.ServerStatsItem["Data Ping"]:GetValue()
        end)
        overlayPing.Text = "Ping: " .. (okp and (math.floor(pingVal) .. " ms") or "--")
        overlayClock.Text = os.date("%H:%M:%S")
    end
end)

-- чат-пузыри в цвет темы
local bubbleOn = false
local bubbleOrig = {}
local function setBubbleStyle(on)
    local ok, bc = pcall(function() return TextChatService.BubbleChatConfiguration end)
    if not ok or not bc then return false end
    if on then
        bubbleOrig.BackgroundColor3 = bc.BackgroundColor3
        bubbleOrig.TextColor3 = bc.TextColor3
        bubbleOrig.Font = bc.Font
        pcall(function()
            bc.BackgroundColor3 = accent.Value
            bc.TextColor3 = Color3.new(1, 1, 1)
            bc.Font = Enum.Font.GothamBold
        end)
    elseif bubbleOrig.BackgroundColor3 then
        pcall(function()
            bc.BackgroundColor3 = bubbleOrig.BackgroundColor3
            bc.TextColor3 = bubbleOrig.TextColor3
            bc.Font = bubbleOrig.Font
        end)
    end
    return true
end
addToggle(settingsPage, "chat_bubbles", false, function(v)
    bubbleOn = setBubbleStyle(v) and v or false
end)
onAccent(function(c)
    if bubbleOn then
        pcall(function() TextChatService.BubbleChatConfiguration.BackgroundColor3 = c end)
    end
end)

-- чистим новое при закрытии меню
local prevCleanup4 = genv.__HubCleanup
genv.__HubCleanup = function()
    local c = LocalPlayer.Character
    if c then
        pcall(removeTrail, c)
        pcall(removeNametag, c)
    end
    pcall(function() overlayGui:Destroy() end)
    pcall(function() setBubbleStyle(false) end)
    pcall(function() StarterGui:SetCoreGuiEnabled(Enum.CoreGuiType.Chat, true) end)
    pcall(function() StarterGui:SetCoreGuiEnabled(Enum.CoreGuiType.PlayerList, true) end)
    if prevCleanup4 then prevCleanup4() end
end

----------------------------------------------------------------------
-- ИКОНКА КНОПКИ ВЫЗОВА (картинка вместо смайла, 4 варианта) + бонус
----------------------------------------------------------------------
L.hd_bubble_icon = { EN = "Bubble icon",  RU = "Иконка кнопки" }
L.icon_1         = { EN = "Anime girl 1", RU = "Тян 1" }
L.icon_2         = { EN = "Anime girl 2", RU = "Тян 2" }
L.icon_3         = { EN = "Anime girl 3", RU = "Тян 3" }
L.icon_4         = { EN = "Anime girl 4", RU = "Тян 4" }
L.icon_default   = { EN = "Default (fox)", RU = "По умолчанию (лисёнок)" }
L.hd_greeting    = { EN = "Greeting",     RU = "Приветствие" }
L.greeting_note  = {
    EN = "A short welcome message flashes in the footer when the menu opens.",
    RU = "При открытии меню внизу на секунду появляется приветствие.",
}
L.greeting_msg = { EN = "Welcome back!", RU = "С возвращением!" }

local bubbleImg = new("ImageLabel", {
    Size = UDim2.fromScale(1, 1), BackgroundTransparency = 1,
    Image = "", ScaleType = Enum.ScaleType.Crop, Visible = false, ZIndex = 1,
}, bubble)
bubble.ZIndex = 2

addHeader(settingsPage, "hd_bubble_icon")

local ICONS = {
    { key = "icon_1", id = "6239938337" },
    { key = "icon_2", id = "8652665149" },
    { key = "icon_3", id = "11425468695" },
    { key = "icon_4", id = "10341849885" },
}

local iconOpts
local function selectIcon(i)
    for j, o in pairs(iconOpts) do o.Select(j == i) end
    if i == 0 then
        bubbleImg.Visible = false
        bubble.TextTransparency = 0
    else
        bubbleImg.Image = "rbxassetid://" .. ICONS[i].id
        bubbleImg.Visible = true
        bubble.TextTransparency = 1
    end
end

iconOpts = {}
iconOpts[0] = addOption(settingsPage, "icon_default", nil, function() selectIcon(0) end)
for i, ic in ipairs(ICONS) do
    local row = rowBase(settingsPage, 46, "TextButton", { Text = "", AutoButtonColor = false, BackgroundTransparency = 1 })
    local st = stroke(row, accent.Value, 1.5, 1)
    local img = new("ImageLabel", {
        Position = UDim2.new(0, 10, 0.5, -15), Size = UDim2.fromOffset(30, 30),
        BackgroundTransparency = 1, Image = "rbxassetid://" .. ic.id, ScaleType = Enum.ScaleType.Crop,
    }, row)
    corner(img, 8)
    bind(new("TextLabel", {
        Position = UDim2.fromOffset(50, 0), Size = UDim2.new(1, -60, 1, 0), BackgroundTransparency = 1,
        Font = Enum.Font.GothamMedium, TextSize = 15, TextColor3 = WHITE,
        TextXAlignment = Enum.TextXAlignment.Left, Text = "",
    }, row), ic.key)
    local obj = {}
    function obj.Select(v)
        TweenService:Create(row, TweenInfo.new(0.15), { BackgroundTransparency = v and 0.55 or 1 }):Play()
        st.Transparency = v and 0.2 or 1
    end
    onAccent(function(c) st.Color = c end)
    connect(row.MouseButton1Click, function() selectIcon(i) end)
    iconOpts[i] = obj
end
iconOpts[0].Select(true)

-- бонус: короткое приветствие при открытии меню
addHeader(settingsPage, "hd_greeting")
addNote(settingsPage, "greeting_note", 32)
connect(main:GetPropertyChangedSignal("Visible"), function()
    if main.Visible then toast("greeting_msg") end
end)

----------------------------------------------------------------------
-- ВОТЕРМАРКА, ФОН МЕНЮ, ШЛЯПА И ПЛАЩ
----------------------------------------------------------------------
L.bad_id      = { EN = "Invalid ID", RU = "Неверный ID" }
L.hd_menu_bg  = { EN = "Menu background image", RU = "Фон меню" }
L.hd_hat      = { EN = "China hat",  RU = "Китайская шляпа" }
L.hat         = { EN = "Hat",        RU = "Шляпа" }
L.hd_cloak    = { EN = "Cloak",      RU = "Плащ" }
L.cloak       = { EN = "Cloak",      RU = "Плащ" }

-- вотермарка сверху: FPS, пинг, ник, цвет темы, клик открывает меню
local watermark = new("TextButton", {
    Name = "HubWatermark", AnchorPoint = Vector2.new(0.5, 0),
    Position = UDim2.new(0.5, 0, 0, 8), Size = UDim2.fromOffset(240, 28),
    BackgroundColor3 = Color3.fromRGB(15, 15, 18), BackgroundTransparency = 0.15,
    Text = "", AutoButtonColor = false, BorderSizePixel = 0,
}, gui)
corner(watermark, 8)
local wmStroke = stroke(watermark, accent.Value, 1, 0.4)
local wmLabel = new("TextLabel", {
    Size = UDim2.fromScale(1, 1), BackgroundTransparency = 1,
    Font = Enum.Font.GothamBold, TextSize = 12, TextColor3 = accent.Value,
    Text = HUB_NAME,
}, watermark)
onAccent(function(c)
    wmStroke.Color = c
    wmLabel.TextColor3 = c
end)
connect(watermark.MouseButton1Click, function() setMinimized(main.Visible) end)

local wmAcc, wmFrames = 0, 0
connect(RunService.RenderStepped, function(dt)
    wmFrames = wmFrames + 1
    wmAcc = wmAcc + dt
    if wmAcc >= 0.5 then
        local fps = math.floor(wmFrames / wmAcc + 0.5)
        wmFrames, wmAcc = 0, 0
        local okp, pingVal = pcall(function()
            return Stats.Network.ServerStatsItem["Data Ping"]:GetValue()
        end)
        wmLabel.Text = string.format("%s | %d fps | %s ms | %s",
            HUB_NAME, fps, okp and tostring(math.floor(pingVal)) or "--", LocalPlayer.Name)
    end
end)

-- картинка на фоне самого меню
addHeader(settingsPage, "hd_menu_bg")
local menuBgBox = idRow(settingsPage)
menuBgBox.Text = "8652665149"
local menuBgImg = new("ImageLabel", {
    Size = UDim2.fromScale(1, 1), BackgroundTransparency = 1,
    Image = "rbxassetid://8652665149", ImageTransparency = 0.88,
    ScaleType = Enum.ScaleType.Crop, ZIndex = 0,
}, main)
addButton(settingsPage, "apply", function()
    local n = tonumber(menuBgBox.Text)
    if not n then toast("bad_id"); return end
    menuBgImg.Image = "rbxassetid://" .. tostring(n)
end)

-- китайская шляпа
local function removeHat(char)
    local r = rigOf(char)
    if r.hat then r.hat:Destroy(); r.hat = nil end
end
local function addHat(char, id)
    removeHat(char)
    local hum = char:FindFirstChildOfClass("Humanoid")
    local _, head = cosmeticAnchors(char)
    if not hum or not head then return false end
    local inst = loadCatalogAsset(id, head, CFrame.new(0, 0.4, 0))
    if not inst then return false end
    if inst:IsA("Accessory") then hum:AddAccessory(inst) else inst.Parent = char end
    rigOf(char).hat = inst
    return true
end
charState.hat = false
addHeader(charPage, "hd_hat")
local hatBox = idRow(charPage)
hatBox.Text = "5830797641"
local hatToggle
hatToggle = addToggle(charPage, "hat", false, function(v)
    local c = LocalPlayer.Character
    if v then
        local ok = c and addHat(c, hatBox.Text)
        if not ok then hatToggle.Set(false, true); toast("asset_fail"); return end
        charState.hat = true
    else
        charState.hat = false
        if c then removeHat(c) end
    end
end)

-- плащ
local function removeCloak(char)
    local r = rigOf(char)
    if r.cloak then r.cloak:Destroy(); r.cloak = nil end
end
local function addCloak(char, id)
    removeCloak(char)
    local hum = char:FindFirstChildOfClass("Humanoid")
    local torso = cosmeticAnchors(char)
    if not hum or not torso then return false end
    local inst = loadCatalogAsset(id, torso, CFrame.new(0, 0, 0.3))
    if not inst then return false end
    if inst:IsA("Accessory") then hum:AddAccessory(inst) else inst.Parent = char end
    rigOf(char).cloak = inst
    return true
end
charState.cloak = false
addHeader(charPage, "hd_cloak")
local cloakBox = idRow(charPage)
cloakBox.Text = "17730299338"
local cloakToggle
cloakToggle = addToggle(charPage, "cloak", false, function(v)
    local c = LocalPlayer.Character
    if v then
        local ok = c and addCloak(c, cloakBox.Text)
        if not ok then cloakToggle.Set(false, true); toast("asset_fail"); return end
        charState.cloak = true
    else
        charState.cloak = false
        if c then removeCloak(c) end
    end
end)

-- дозаряжаем шляпу/плащ после респавна
local baseApplyCosmetics2 = applyCosmetics
applyCosmetics = function(char)
    baseApplyCosmetics2(char)
    if charState.hat then addHat(char, hatBox.Text) end
    if charState.cloak then addCloak(char, cloakBox.Text) end
end

-- чистим новое при закрытии меню
local prevCleanup5 = genv.__HubCleanup
genv.__HubCleanup = function()
    local c = LocalPlayer.Character
    if c then
        pcall(removeHat, c)
        pcall(removeCloak, c)
    end
    pcall(function() watermark:Destroy() end)
    if prevCleanup5 then prevCleanup5() end
end

----------------------------------------------------------------------
-- СВОЙ СТИЛЬ КРЫЛЬЕВ/НИМБА/АУРЫ (вместо ассета, в цвет темы)
----------------------------------------------------------------------
L.hd_custom_style   = { EN = "Wings / halo / aura style", RU = "Стиль крыльев/нимба/ауры" }
L.custom_mode_wings = { EN = "Wings: built-in shape (not asset)", RU = "Крылья: свой стиль (не ассет)" }
L.custom_mode_halo  = { EN = "Halo: built-in shape (not asset)",  RU = "Нимб: свой стиль (не ассет)" }
L.custom_mode_aura  = { EN = "Aura: built-in shape (not asset)",  RU = "Аура: свой стиль (не ассет)" }
L.custom_style_note = {
    EN = "When on, that item ignores the pasted ID and uses a simple built-in shape colored to your theme instead.",
    RU = "Когда включено, этот пункт игнорирует вставленный ID и использует простую форму в цвет темы.",
}

local wingsMode, haloMode, auraMode = "asset", "asset", "asset"

local function addCustomWings(char)
    local torso = cosmeticAnchors(char)
    if not torso then return false end
    local model = new("Model", { Name = "HubWings" })
    for _, side in ipairs({ -1, 1 }) do
        local p = new("Part", {
            Name = "Wing", Anchored = false, CanCollide = false, CanQuery = false, CanTouch = false,
            CastShadow = false, Massless = true, Material = Enum.Material.Neon,
            Transparency = 0.2, Color = accent.Value, Size = Vector3.new(0.2, 2.6, 1.5),
        })
        p.CFrame = torso.CFrame * CFrame.new(side * 0.5, 0.4, 0.5) * CFrame.Angles(0, math.rad(side * 35), math.rad(side * -12))
        p.Parent = model
        new("SpecialMesh", { MeshType = Enum.MeshType.Wedge }, p)
        new("WeldConstraint", { Part0 = torso, Part1 = p }, p)
    end
    model.Parent = char
    rigOf(char).wings = model
    return true
end

local function addCustomHalo(char)
    local _, head = cosmeticAnchors(char)
    if not head then return false end
    local p = new("Part", {
        Name = "Halo", Shape = Enum.PartType.Cylinder, Anchored = false, CanCollide = false,
        CanQuery = false, CanTouch = false, CastShadow = false, Massless = true,
        Material = Enum.Material.Neon, Color = accent.Value, Transparency = 0.1,
        Size = Vector3.new(0.15, 1.6, 1.6),
    })
    p.CFrame = head.CFrame * CFrame.new(0, 1.1, 0) * CFrame.Angles(0, 0, math.rad(90))
    p.Parent = char
    new("WeldConstraint", { Part0 = head, Part1 = p }, p)
    rigOf(char).halo = p
    return true
end

local function addCustomAura(char)
    local torso = cosmeticAnchors(char)
    if not torso then return false end
    local att = new("Attachment", { Position = Vector3.new(0, -1, 0) }, torso)
    new("ParticleEmitter", {
        Texture = "rbxasset://textures/particles/sparkles_main.dds",
        Color = ColorSequence.new(accent.Value),
        Size = NumberSequence.new({
            NumberSequenceKeypoint.new(0, 0.4),
            NumberSequenceKeypoint.new(1, 0),
        }),
        Transparency = NumberSequence.new({
            NumberSequenceKeypoint.new(0, 0.2),
            NumberSequenceKeypoint.new(1, 1),
        }),
        Lifetime = NumberRange.new(0.8, 1.2),
        Speed = NumberRange.new(2, 4),
        Rate = 40,
        EmissionDirection = Enum.NormalId.Top,
        SpreadAngle = Vector2.new(20, 20),
        LightEmission = 0.6,
    }, att)
    rigOf(char).aura = att
    return true
end

local baseAddWings = addWings
addWings = function(char, id)
    if wingsMode == "custom" then
        removeWings(char)
        return addCustomWings(char)
    end
    return baseAddWings(char, id)
end

local baseAddHalo = addHalo
addHalo = function(char, id)
    if haloMode == "custom" then
        removeHalo(char)
        return addCustomHalo(char)
    end
    return baseAddHalo(char, id)
end

local baseAddAura = addAura
addAura = function(char, id)
    if auraMode == "custom" then
        removeAura(char)
        return addCustomAura(char)
    end
    return baseAddAura(char, id)
end

-- живой цвет темы только для СВОИХ (не ассетных) крыльев/нимба/ауры
onAccent(function(c)
    for _, r in pairs(rigStore) do
        if r.wings and r.wings:IsA("Model") and r.wings.Name == "HubWings" then
            for _, p in ipairs(r.wings:GetChildren()) do
                if p:IsA("BasePart") then p.Color = c end
            end
        end
        if r.halo and r.halo:IsA("BasePart") and r.halo.Name == "Halo" then
            r.halo.Color = c
        end
        if r.aura and r.aura:IsA("Attachment") then
            local em = r.aura:FindFirstChildOfClass("ParticleEmitter")
            if em then em.Color = ColorSequence.new(c) end
        end
    end
end)

addHeader(charPage, "hd_custom_style")
addToggle(charPage, "custom_mode_wings", false, function(v)
    wingsMode = v and "custom" or "asset"
    if charState.wings then
        local c = LocalPlayer.Character
        if c then addWings(c, wingsBox.Text) end
    end
end)
addToggle(charPage, "custom_mode_halo", false, function(v)
    haloMode = v and "custom" or "asset"
    if charState.halo then
        local c = LocalPlayer.Character
        if c then addHalo(c, haloBox.Text) end
    end
end)
addToggle(charPage, "custom_mode_aura", false, function(v)
    auraMode = v and "custom" or "asset"
    if charState.aura then
        local c = LocalPlayer.Character
        if c then addAura(c, auraBox.Text) end
    end
end)
addNote(charPage, "custom_style_note", 46)
