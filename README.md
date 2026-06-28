--[[
    NOVA PANEL v3.0 | Universal Executor
    Aimbot + FOV Circle + ESP + Chams
    Funciona em todos os jogos Roblox
    Tecla INSERT para abrir/fechar
    Tecla DELETE para desativar tudo
    by davixp (Visual Redesign Edition)
--]]

-- Cleanup anterior
if _G.NovaPanel then
    pcall(function() _G.NovaPanel:Destroy() end)
    _G.NovaPanel = nil
end
if _G.NovaESP then
    for _, o in pairs(_G.NovaESP) do
        pcall(function()
            if o.BoxT then o.BoxT:Remove() end
            if o.BoxB then o.BoxB:Remove() end
            if o.BoxL then o.BoxL:Remove() end
            if o.BoxR then o.BoxR:Remove() end
            if o.Name then o.Name:Remove() end
            if o.Dist then o.Dist:Remove() end
            if o.HPbg then o.HPbg:Remove() end
            if o.HP then o.HP:Remove() end
            if o.Tracer then o.Tracer:Remove() end
            if o.Info then o.Info:Remove() end
            if o.Bones then
                for _, b in pairs(o.Bones) do b:Remove() end
            end
        end)
    end
    _G.NovaESP = nil
end
if _G.NovaFOV then
    pcall(function() _G.NovaFOV:Remove() end)
    _G.NovaFOV = nil
end
if _G.NovaConn then
    for _, c in pairs(_G.NovaConn) do
        pcall(function() c:Disconnect() end)
    end
    _G.NovaConn = nil
end

local Players          = game:GetService("Players")
local RunService       = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local TweenService     = game:GetService("TweenService")
local Camera           = workspace.CurrentCamera
local LocalPlayer      = Players.LocalPlayer
local Mouse            = LocalPlayer:GetMouse()

_G.NovaConn = {}

-- Verifica Drawing API
local hasDrawing = (typeof(Drawing) ~= "nil")
if not hasDrawing then
    pcall(function() hasDrawing = Drawing and Drawing.new end)
end

local Config = {
    -- Aimbot Principal
    AimbotEnabled    = false,
    AimbotKeyMouse   = true,
    AimbotPart       = "Head",
    AimbotSmooth     = 0.20,
    AimbotFOV        = 150,
    AimbotTeamCheck  = false,
    AimbotWallCheck  = false,
    AimbotPrediction = false,
    AimbotPredValue  = 0.10,
    -- Aimbot Modos
    AimbotHead       = true,
    AimbotLegit      = false,
    AimbotDelay      = false,
    AimbotDelayTime  = 0.05,
    AimbotBind       = "MouseButton2",
    AimbotRage       = false,
    AimbotRageSmooth = 0.02,
    SilentKill       = true,
    SilentKillDist   = 300,
    -- FOV
    FOVEnabled       = true,
    FOVThickness     = 1.5,
    -- ESP
    ESPEnabled       = false,
    ESPAll           = false,
    ESPBoxes         = false,
    ESPNames         = false,
    ESPHealth        = false,
    ESPDistance       = false,
    ESPSkeleton      = false,
    ESPTeamColor     = false,
    ESPMaxDist       = 1000,
    ESPBoxThick      = 1.5,
    ESPNameSize      = 14,
    ESPTracers       = false,
    ESPTracerOrigin  = "Bottom",
    ESPInfo          = false,
    -- Chams
    ChamsEnabled     = false,
    ChamsGreen       = false,
    ChamsTransparency= 0.3,
    ChamsOutline     = true,
    -- Misc
    Destruct         = false,
    StreamMode       = false,
}

local function getChar(p)
    if not p then return nil end
    return p.Character
end
local function getHum(p)
    local c = getChar(p)
    if not c then return nil end
    return c:FindFirstChildOfClass("Humanoid")
end
local function getRoot(p)
    local c = getChar(p)
    if not c then return nil end
    return c:FindFirstChild("HumanoidRootPart") or c:FindFirstChild("Torso")
end
local function getAimP(p)
    local c = getChar(p)
    if not c then return nil end
    return c:FindFirstChild(Config.AimbotPart) or c:FindFirstChild("HumanoidRootPart")
end
local function isAlive(p)
    local h = getHum(p)
    return h and h.Health > 0
end
local function sameTeam(p)
    if not LocalPlayer.Team then return false end
    if not p.Team then return false end
    return LocalPlayer.Team == p.Team
end

local function w2s(pos)
    local sp, on = Camera:WorldToViewportPoint(pos)
    return Vector2.new(sp.X, sp.Y), on, sp.Z
end

local function isVisible(part)
    if not part then return false end
    local ok, result = pcall(function()
        local orig = Camera.CFrame.Position
        local dir = (part.Position - orig)
        local params = RaycastParams.new()
        params.FilterType = Enum.RaycastFilterType.Exclude
        params.FilterDescendantsInstances = {LocalPlayer.Character, Camera}
        local rayResult = workspace:Raycast(orig, dir, params)
        if not rayResult then return true end
        return rayResult.Instance:IsDescendantOf(part.Parent)
    end)
    if ok then return result end
    return false
end

-- ESP
local ESPObjects = {}
_G.NovaESP = ESPObjects
local ChamsObjects = {}

local BONES = {
    {"Head","UpperTorso"},{"Head","Torso"},
    {"UpperTorso","LeftUpperArm"},{"UpperTorso","RightUpperArm"},
    {"LeftUpperArm","LeftLowerArm"},{"RightUpperArm","RightLowerArm"},
    {"LeftLowerArm","LeftHand"},{"RightLowerArm","RightHand"},
    {"UpperTorso","LowerTorso"},{"Torso","LowerTorso"},
    {"LowerTorso","LeftUpperLeg"},{"LowerTorso","RightUpperLeg"},
    {"LeftUpperLeg","LeftLowerLeg"},{"RightUpperLeg","RightLowerLeg"},
    {"LeftLowerLeg","LeftFoot"},{"RightLowerLeg","RightFoot"},
}

local function mkLine(color, thick, z)
    if not hasDrawing then return nil end
    local ok, l = pcall(function()
        local line = Drawing.new("Line")
        line.Visible = false
        line.Color = color
        line.Thickness = thick
        line.ZIndex = z or 5
        return line
    end)
    if ok then return l end
    return nil
end

local function mkText(size, color, center, outline, z)
    if not hasDrawing then return nil end
    local ok, t = pcall(function()
        local text = Drawing.new("Text")
        text.Visible = false
        text.Size = size
        text.Color = color
        text.Center = (center ~= false)
        text.Outline = (outline ~= false)
        text.OutlineColor = Color3.fromRGB(0, 0, 0)
        text.Font = 2
        text.ZIndex = z or 6
        return text
    end)
    if ok then return t end
    return nil
end

local function safeSetProp(obj, prop, val)
    if obj then
        pcall(function() obj[prop] = val end)
    end
end

local function safeRemove(obj)
    if obj then pcall(function() obj:Remove() end) end
end

local function createESP(player)
    if player == LocalPlayer then return end
    if ESPObjects[player] then return end
    local WHITE = Color3.fromRGB(255, 255, 255)
    local GREEN = Color3.fromRGB(0, 255, 100)
    local GREY  = Color3.fromRGB(200, 200, 200)
    local CYAN  = Color3.fromRGB(0, 200, 255)

    local obj = {
        Player = player,
        BoxT   = mkLine(WHITE, 1.5),
        BoxB   = mkLine(WHITE, 1.5),
        BoxL   = mkLine(WHITE, 1.5),
        BoxR   = mkLine(WHITE, 1.5),
        Name   = mkText(14, WHITE, true, true, 6),
        Dist   = mkText(13, GREY, true, true, 6),
        HPbg   = mkLine(Color3.fromRGB(0, 0, 0), 4, 4),
        HP     = mkLine(GREEN, 2, 5),
        Tracer = mkLine(CYAN, 1.5, 3),
        Info   = mkText(11, Color3.fromRGB(255, 200, 50), true, true, 6),
        Bones  = {},
    }
    for i = 1, #BONES do
        obj.Bones[i] = mkLine(WHITE, 1, 4)
    end
    ESPObjects[player] = obj
end

local function removeESP(player)
    local o = ESPObjects[player]
    if not o then return end
    safeRemove(o.BoxT); safeRemove(o.BoxB)
    safeRemove(o.BoxL); safeRemove(o.BoxR)
    safeRemove(o.Name); safeRemove(o.Dist)
    safeRemove(o.HPbg); safeRemove(o.HP)
    safeRemove(o.Tracer); safeRemove(o.Info)
    if o.Bones then
        for _, b in pairs(o.Bones) do safeRemove(b) end
    end
    ESPObjects[player] = nil

    local ch = ChamsObjects[player]
    if ch then
        pcall(function() ch:Destroy() end)
        ChamsObjects[player] = nil
    end
end

local function hideESPObj(o)
    safeSetProp(o.BoxT, "Visible", false)
    safeSetProp(o.BoxB, "Visible", false)
    safeSetProp(o.BoxL, "Visible", false)
    safeSetProp(o.BoxR, "Visible", false)
    safeSetProp(o.Name, "Visible", false)
    safeSetProp(o.Dist, "Visible", false)
    safeSetProp(o.HPbg, "Visible", false)
    safeSetProp(o.HP, "Visible", false)
    safeSetProp(o.Tracer, "Visible", false)
    safeSetProp(o.Info, "Visible", false)
    if o.Bones then
        for _, b in pairs(o.Bones) do
            safeSetProp(b, "Visible", false)
        end
    end
end

-- Chams
local function updateChams(player)
    local char = getChar(player)
    if not char or not isAlive(player) then
        local ch = ChamsObjects[player]
        if ch then
            pcall(function() ch:Destroy() end)
            ChamsObjects[player] = nil
        end
        return
    end

    if Config.ChamsEnabled then
        if not ChamsObjects[player] or not ChamsObjects[player].Parent then
            pcall(function()
                -- Remove antigo se existir
                local old = char:FindFirstChild("NovaChams")
                if old then old:Destroy() end
                local h = Instance.new("Highlight")
                h.Name = "NovaChams"
                h.Adornee = char
                h.DepthMode = Enum.HighlightDepthMode.AlwaysOnTop
                h.Parent = char
                ChamsObjects[player] = h
            end)
        end
        local h = ChamsObjects[player]
        if h and h.Parent then
            pcall(function()
                local chamsCol = Config.ChamsGreen and Color3.fromRGB(0, 255, 100) or Color3.fromRGB(160, 80, 255)
                h.FillColor = chamsCol
                h.FillTransparency = Config.ChamsTransparency
                h.OutlineColor = Config.ChamsOutline and Color3.fromRGB(255, 255, 255) or chamsCol
                h.OutlineTransparency = Config.ChamsOutline and 0 or 1
                h.Enabled = true
            end)
        end
    else
        local ch = ChamsObjects[player]
        if ch then
            pcall(function() ch.Enabled = false end)
        end
    end
end

-- ESP Update
local function updateESP()
    for player, o in pairs(ESPObjects) do
        local char = getChar(player)
        local hum  = getHum(player)
        local root = getRoot(player)

        pcall(function() updateChams(player) end)

        if not Config.ESPEnabled or not char or not hum or not root or not isAlive(player) then
            hideESPObj(o)
        elseif Config.AimbotTeamCheck and sameTeam(player) then
            hideESPObj(o)
        else
            local rootPos  = root.Position
            local headNode = char:FindFirstChild("Head")
            local headPos  = headNode and headNode.Position or (rootPos + Vector3.new(0, 3, 0))

            local rs, ron = w2s(rootPos)
            local hs, hon = w2s(headPos)
            local dist3D  = (rootPos - Camera.CFrame.Position).Magnitude

            if not ron or dist3D > Config.ESPMaxDist then
                hideESPObj(o)
            else
                local displayName = Config.StreamMode and "Player" or player.Name
                local color  = Config.ESPTeamColor and player.TeamColor and player.TeamColor.Color or Color3.fromRGB(255, 255, 255)
                local height = math.abs(hs.Y - rs.Y)
                if height < 10 then height = 50 end
                local width  = height * 0.55
                local top    = hs.Y - 2
                local bot    = rs.Y + 2
                local left   = rs.X - width
                local right  = rs.X + width

                local showBoxes    = Config.ESPBoxes or Config.ESPAll
                local showNames    = Config.ESPNames or Config.ESPAll
                local showHealth   = Config.ESPHealth or Config.ESPAll
                local showDist     = Config.ESPDistance or Config.ESPAll
                local showSkeleton = Config.ESPSkeleton or Config.ESPAll
                local showTracers  = Config.ESPTracers or Config.ESPAll
                local showInfo     = Config.ESPInfo or Config.ESPAll

                -- Boxes
                if showBoxes and o.BoxT then
                    pcall(function()
                        o.BoxT.From = Vector2.new(left, top);  o.BoxT.To = Vector2.new(right, top)
                        o.BoxB.From = Vector2.new(left, bot);  o.BoxB.To = Vector2.new(right, bot)
                        o.BoxL.From = Vector2.new(left, top);  o.BoxL.To = Vector2.new(left, bot)
                        o.BoxR.From = Vector2.new(right, top); o.BoxR.To = Vector2.new(right, bot)
                        for _, l in pairs({o.BoxT, o.BoxB, o.BoxL, o.BoxR}) do
                            l.Visible = true; l.Color = color; l.Thickness = Config.ESPBoxThick
                        end
                    end)
                else
                    safeSetProp(o.BoxT, "Visible", false)
                    safeSetProp(o.BoxB, "Visible", false)
                    safeSetProp(o.BoxL, "Visible", false)
                    safeSetProp(o.BoxR, "Visible", false)
                end

                -- Names
                if showNames and o.Name then
                    pcall(function()
                        o.Name.Visible = true
                        o.Name.Position = Vector2.new(rs.X, top - 18)
                        o.Name.Text = displayName
                        o.Name.Size = Config.ESPNameSize
                    end)
                else
                    safeSetProp(o.Name, "Visible", false)
                end

                -- Distance
                if showDist and o.Dist then
                    pcall(function()
                        o.Dist.Visible = true
                        o.Dist.Position = Vector2.new(rs.X, bot + 2)
                        o.Dist.Text = string.format("[%dm]", math.floor(dist3D / 3))
                    end)
                else
                    safeSetProp(o.Dist, "Visible", false)
                end

                -- Health Bar
                if showHealth and o.HPbg and o.HP then
                    pcall(function()
                        local maxH  = hum.MaxHealth > 0 and hum.MaxHealth or 100
                        local curH  = math.clamp(hum.Health, 0, maxH)
                        local ratio = curH / maxH
                        local barH  = bot - top
                        local barX  = left - 6
                        local fillY = bot - barH * ratio
                        o.HPbg.Visible = true; o.HPbg.From = Vector2.new(barX, top); o.HPbg.To = Vector2.new(barX, bot)
                        o.HP.Visible = true;   o.HP.From = Vector2.new(barX, fillY); o.HP.To = Vector2.new(barX, bot)
                        o.HP.Color = Color3.fromRGB(math.floor(255 * (1 - ratio)), math.floor(255 * ratio), 50)
                    end)
                else
                    safeSetProp(o.HPbg, "Visible", false)
                    safeSetProp(o.HP, "Visible", false)
                end

                -- Tracers
                if showTracers and o.Tracer then
                    pcall(function()
                        local originY = Camera.ViewportSize.Y
                        local originX = Camera.ViewportSize.X / 2
                        if Config.ESPTracerOrigin == "Center" then
                            originY = Camera.ViewportSize.Y / 2
                        elseif Config.ESPTracerOrigin == "Top" then
                            originY = 0
                        end
                        o.Tracer.Visible = true
                        o.Tracer.From = Vector2.new(originX, originY)
                        o.Tracer.To = Vector2.new(rs.X, rs.Y)
                        o.Tracer.Color = color
                    end)
                else
                    safeSetProp(o.Tracer, "Visible", false)
                end

                -- Info
                if showInfo and o.Info then
                    pcall(function()
                        local toolName = "None"
                        local tool = char:FindFirstChildOfClass("Tool")
                        if tool then toolName = tool.Name end
                        local maxH = hum.MaxHealth > 0 and hum.MaxHealth or 100
                        local curH = math.clamp(hum.Health, 0, maxH)
                        o.Info.Visible = true
                        o.Info.Position = Vector2.new(rs.X, bot + 16)
                        o.Info.Text = string.format("[%s | HP:%d]", toolName, math.floor(curH))
                    end)
                else
                    safeSetProp(o.Info, "Visible", false)
                end

                -- Skeleton
                if showSkeleton and o.Bones then
                    for i, bp in ipairs(BONES) do
                        local bn = o.Bones[i]
                        if bn then
                            local pA = char:FindFirstChild(bp[1])
                            local pB = char:FindFirstChild(bp[2])
                            if pA and pB then
                                local sA, oA = w2s(pA.Position)
                                local sB, oB = w2s(pB.Position)
                                if oA and oB then
                                    pcall(function()
                                        bn.Visible = true; bn.From = sA; bn.To = sB
                                        bn.Color = Color3.fromRGB(255, 255, 255)
                                    end)
                                else
                                    safeSetProp(bn, "Visible", false)
                                end
                            else
                                safeSetProp(bn, "Visible", false)
                            end
                        end
                    end
                else
                    if o.Bones then
                        for _, b in pairs(o.Bones) do
                            safeSetProp(b, "Visible", false)
                        end
                    end
                end
            end
        end
    end
end

-- FOV Circle
local FOVCircle = nil
if hasDrawing then
    pcall(function()
        FOVCircle = Drawing.new("Circle")
        FOVCircle.Visible = false
        FOVCircle.Color = Color3.fromRGB(255, 255, 255)
        FOVCircle.Radius = 150
        FOVCircle.Thickness = 1.5
        FOVCircle.Filled = false
        FOVCircle.ZIndex = 10
    end)
end
_G.NovaFOV = FOVCircle

-- Aimbot
local function getClosest()
    local center = Vector2.new(Camera.ViewportSize.X / 2, Camera.ViewportSize.Y / 2)
    local bestDist = Config.AimbotFOV
    local bestPart = nil
    for _, p in ipairs(Players:GetPlayers()) do
        if p ~= LocalPlayer and isAlive(p) then
            if not (Config.AimbotTeamCheck and sameTeam(p)) then
                local part = getAimP(p)
                if part then
                    if not Config.AimbotWallCheck or isVisible(part) then
                        local sp, on = w2s(part.Position)
                        if on then
                            local d = (sp - center).Magnitude
                            if d < bestDist then
                                bestDist = d
                                bestPart = part
                            end
                        end
                    end
                end
            end
        end
    end
    return bestPart
end

local lastAimbotTime = 0

local function runAimbot()
    if not Config.AimbotEnabled then return end

    local holding = false
    if Config.AimbotBind == "MouseButton2" then
        holding = UserInputService:IsMouseButtonPressed(Enum.UserInputType.MouseButton2)
    elseif Config.AimbotBind == "MouseButton1" then
        holding = UserInputService:IsMouseButtonPressed(Enum.UserInputType.MouseButton1)
    elseif Config.AimbotBind == "E" then
        holding = UserInputService:IsKeyDown(Enum.KeyCode.E)
    elseif Config.AimbotBind == "Q" then
        holding = UserInputService:IsKeyDown(Enum.KeyCode.Q)
    elseif Config.AimbotBind == "X" then
        holding = UserInputService:IsKeyDown(Enum.KeyCode.X)
    elseif Config.AimbotBind == "C" then
        holding = UserInputService:IsKeyDown(Enum.KeyCode.C)
    elseif Config.AimbotBind == "CapsLock" then
        holding = UserInputService:IsKeyDown(Enum.KeyCode.CapsLock)
    end

    if not holding then return end

    -- Delay
    if Config.AimbotDelay then
        local now = tick()
        if now - lastAimbotTime < Config.AimbotDelayTime then return end
        lastAimbotTime = now
    end

    local target = getClosest()
    if not target then return end
    local pos = target.Position

    -- Predicao
    if Config.AimbotPrediction then
        local root = target.Parent and target.Parent:FindFirstChild("HumanoidRootPart")
        if root then
            pcall(function()
                pos = pos + root.AssemblyLinearVelocity * Config.AimbotPredValue
            end)
        end
    end

    -- Smooth por modo
    local smooth
    if Config.AimbotRage then
        smooth = math.clamp(Config.AimbotRageSmooth, 0.001, 0.1)
    elseif Config.AimbotLegit then
        smooth = math.clamp(Config.AimbotSmooth * 1.5, 0.1, 0.95)
    else
        smooth = math.clamp(Config.AimbotSmooth, 0.01, 1.0)
    end

    Camera.CFrame = Camera.CFrame:Lerp(CFrame.new(Camera.CFrame.Position, pos), 1 - smooth)
end

-- Destruct
local destructedParts = {}
local function toggleDestruct(enabled)
    if enabled then
        pcall(function()
            for _, obj in ipairs(workspace:GetDescendants()) do
                if obj:IsA("BasePart") then
                    local isPlayer = false
                    for _, p in ipairs(Players:GetPlayers()) do
                        if p.Character and obj:IsDescendantOf(p.Character) then
                            isPlayer = true
                            break
                        end
                    end
                    if not isPlayer and obj.Name ~= "Baseplate" and obj.Name ~= "Terrain" then
                        if obj.Transparency < 1 then
                            table.insert(destructedParts, {
                                Part = obj,
                                OT = obj.Transparency,
                                OC = obj.CanCollide
                            })
                            obj.Transparency = 0.85
                            obj.CanCollide = false
                        end
                    end
                end
            end
        end)
    else
        for _, data in ipairs(destructedParts) do
            pcall(function()
                data.Part.Transparency = data.OT
                data.Part.CanCollide = data.OC
            end)
        end
        destructedParts = {}
    end
end

-- ══════════════════════════════════════
--  GUI
-- ══════════════════════════════════════
local SG = Instance.new("ScreenGui")
SG.Name = "NovaPanelGUI"
SG.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
SG.ResetOnSpawn = false
SG.IgnoreGuiInset = true
_G.NovaPanel = SG

pcall(function() SG.Parent = game:GetService("CoreGui") end)
if not SG.Parent then
    SG.Parent = LocalPlayer:WaitForChild("PlayerGui")
end

local C = {
    BG     = Color3.fromRGB(15, 15, 18),       -- Fundo principal escuro
    Surf   = Color3.fromRGB(22, 22, 26),       -- Cor dos cards/linhas
    Surf2  = Color3.fromRGB(28, 28, 34),       -- Elementos interativos normais
    Surf3  = Color3.fromRGB(36, 36, 44),       -- Hover / Ativo
    Border = Color3.fromRGB(35, 35, 40),       -- Bordas finas escuras
    White  = Color3.fromRGB(240, 240, 245),    -- Texto branco
    Dim    = Color3.fromRGB(140, 140, 150),    -- Texto cinza/escuro
    ON     = Color3.fromRGB(230, 40, 40),      -- Toggle Ativado (Vermelho Neon)
    OFF    = Color3.fromRGB(45, 45, 50),       -- Toggle Desativado
    Red    = Color3.fromRGB(230, 40, 40),      -- Botão Fechar
    Yellow = Color3.fromRGB(240, 180, 40),     -- Botão Minimizar
    Cyan   = Color3.fromRGB(230, 40, 40),      
    Purple = Color3.fromRGB(230, 40, 40),      
    Accent = Color3.fromRGB(230, 40, 40),      -- Cor de Destaque Vermelho
    TabAct = Color3.fromRGB(255, 255, 255),    
    TabIn  = Color3.fromRGB(130, 130, 140),    
}

local function corner(p, r)
    local c = Instance.new("UICorner")
    c.CornerRadius = UDim.new(0, r)
    c.Parent = p
    return c
end

local function stroke(p, col, thick)
    local s = Instance.new("UIStroke")
    s.Color = col or C.Border
    s.Thickness = thick or 1
    s.Parent = p
    return s
end

local function frame(parent, sz, pos, bg, r, transp)
    local f = Instance.new("Frame")
    f.Parent = parent
    f.Size = sz
    f.Position = pos or UDim2.new(0, 0, 0, 0)
    f.BackgroundColor3 = bg or C.Surf
    f.BorderSizePixel = 0
    f.BackgroundTransparency = transp or 0
    if r then corner(f, r) end
    return f
end

local function label(parent, txt, sz, col, pos, xAlign)
    local l = Instance.new("TextLabel")
    l.Parent = parent
    l.Text = txt
    l.TextSize = sz or 13
    l.TextColor3 = col or C.White
    l.BackgroundTransparency = 1
    l.Font = Enum.Font.GothamMedium
    l.TextXAlignment = xAlign or Enum.TextXAlignment.Left
    if pos then l.Position = pos end
    l.Size = UDim2.new(1, 0, 0, (sz or 13) + 6)
    return l
end

-- Main Window (Mais largo e retangular, visual moderno)
local Main = frame(SG, UDim2.new(0, 560, 0, 480), UDim2.new(0.5, -280, 0.5, -240), C.BG, 14)
Main.ClipsDescendants = true
stroke(Main, Color3.fromRGB(45, 45, 50), 1.5)

-- Tab Bar (Barra Lateral Esquerda)
local TabBar = frame(Main, UDim2.new(0, 75, 1, 0), UDim2.new(0, 0, 0, 0), Color3.fromRGB(10, 10, 12), 14)
local SidebarDivider = frame(Main, UDim2.new(0, 1, 1, 0), UDim2.new(0, 75, 0, 0), Color3.fromRGB(30, 30, 35))

-- Logo no Topo da Sidebar (Igual ao da foto)
local LogoContainer = frame(TabBar, UDim2.new(0, 50, 0, 50), UDim2.new(0.5, -25, 0, 12), Color3.fromRGB(20, 20, 24), 25)
stroke(LogoContainer, C.Accent, 1.5)
local LogoLabel = label(LogoContainer, "💀", 20, C.Accent, UDim2.new(0, 0, 0, 0), Enum.TextXAlignment.Center)
LogoLabel.Size = UDim2.new(1, 0, 1, 0)
LogoLabel.TextYAlignment = Enum.TextYAlignment.Center

local tabLayout = Instance.new("UIListLayout")
tabLayout.FillDirection = Enum.FillDirection.Vertical
tabLayout.SortOrder = Enum.SortOrder.LayoutOrder
tabLayout.Padding = UDim.new(0, 10)
tabLayout.HorizontalAlignment = Enum.HorizontalAlignment.Center
tabLayout.Parent = TabBar

local tabPad = Instance.new("UIPadding")
tabPad.PaddingTop = UDim.new(0, 78) -- Empurra os botões para baixo do logo
tabPad.PaddingBottom = UDim.new(0, 10)
tabPad.Parent = TabBar

-- Top Bar (Área Superior Direita)
local TopBar = frame(Main, UDim2.new(1, -75, 0, 44), UDim2.new(0, 75, 0, 0), C.BG, 0, 1)

local TitleLbl = label(TopBar, "  NOVA PANEL", 15, C.Accent)
TitleLbl.Font = Enum.Font.GothamBold
TitleLbl.Size = UDim2.new(1, -120, 1, 0)
TitleLbl.TextYAlignment = Enum.TextYAlignment.Center

local SubLbl = label(TopBar, "v3.0 | davixp  ", 10, C.Dim)
SubLbl.Size = UDim2.new(1, -75, 1, 0)
SubLbl.TextXAlignment = Enum.TextXAlignment.Right
SubLbl.TextYAlignment = Enum.TextYAlignment.Center

-- Botões do Topo Modernizados (Ficam escuros e brilham quando hover)
local function mkTopBtn(xOff, bg, txt)
    local b = Instance.new("TextButton")
    b.Parent = TopBar
    b.Size = UDim2.new(0, 24, 0, 24)
    b.Position = UDim2.new(1, xOff, 0.5, -12)
    b.BackgroundColor3 = Color3.fromRGB(20, 20, 24)
    b.Text = txt
    b.TextColor3 = Color3.fromRGB(150, 150, 160)
    b.TextSize = 10
    b.Font = Enum.Font.GothamBold
    b.BorderSizePixel = 0
    b.AutoButtonColor = false
    corner(b, 6)
    stroke(b, Color3.fromRGB(35, 35, 40), 1)

    b.MouseEnter:Connect(function()
        TweenService:Create(b, TweenInfo.new(0.15), {
            BackgroundColor3 = bg,
            TextColor3 = Color3.fromRGB(255, 255, 255)
        }):Play()
    end)
    b.MouseLeave:Connect(function()
        TweenService:Create(b, TweenInfo.new(0.15), {
            BackgroundColor3 = Color3.fromRGB(20, 20, 24),
            TextColor3 = Color3.fromRGB(150, 150, 160)
        }):Play()
    end)
    return b
end

local CloseBtn = mkTopBtn(-30, C.Red, "✕")
local MinBtn   = mkTopBtn(-60, C.Yellow, "─")

-- Content Frame (Área Principal Invisível para as páginas scrollarem)
local Content = frame(Main, UDim2.new(1, -90, 1, -85), UDim2.new(0, 82, 0, 50), C.BG, 0, 1)

-- Barra de Status Inferior (Igual ao "Injection Stats" da foto)
local StatusBar = frame(Main, UDim2.new(1, -90, 0, 30), UDim2.new(0, 82, 1, -40), Color3.fromRGB(10, 10, 12), 8)
stroke(StatusBar, Color3.fromRGB(30, 30, 35), 1)

local StatusIcon = frame(StatusBar, UDim2.new(0, 10, 0, 10), UDim2.new(0, 12, 0.5, -5), Color3.fromRGB(230, 40, 40), 5)
stroke(StatusIcon, Color3.fromRGB(255, 100, 100), 1)

local StatusText = label(StatusBar, "Status: Carregando...", 11, Color3.fromRGB(150, 150, 160), UDim2.new(0, 32, 0, 0))
StatusText.Size = UDim2.new(1, -40, 1, 0)
StatusText.TextYAlignment = Enum.TextYAlignment.Center

local Tabs = {}

local function newTab(name, icon)
    local displayIcon = icon
    if icon == "AIM" then displayIcon = "🎯"
    elseif icon == "ESP" then displayIcon = "📦"
    elseif icon == "CH" then displayIcon = "👥"
    elseif icon == "CFG" then displayIcon = "⚙"
    elseif icon == "?" then displayIcon = "❓"
    end

    local btn = Instance.new("TextButton")
    btn.Parent = TabBar
    btn.Size = UDim2.new(0, 50, 0, 50)
    btn.BackgroundColor3 = C.Surf
    btn.Text = displayIcon
    btn.TextColor3 = C.TabIn
    btn.TextSize = 18
    btn.Font = Enum.Font.GothamBold
    btn.BorderSizePixel = 0
    btn.AutoButtonColor = false
    corner(btn, 10)
    stroke(btn, Color3.fromRGB(30, 30, 35), 1)

    -- Pequeno indicador vermelho ativo na lateral do botão
    local indicator = frame(btn, UDim2.new(0, 3, 0, 20), UDim2.new(0, 2, 0.5, -10), C.Accent, 1.5)
    indicator.Visible = false

    local page = frame(Content, UDim2.new(1, 0, 1, 0), UDim2.new(0, 0, 0, 0), C.BG, 0, 1)
    page.Visible = false

    local scroll = Instance.new("ScrollingFrame")
    scroll.Parent = page
    scroll.Size = UDim2.new(1, 0, 1, 0)
    scroll.BackgroundTransparency = 1
    scroll.BorderSizePixel = 0
    scroll.ScrollBarThickness = 2
    scroll.ScrollBarImageColor3 = C.Accent
    scroll.AutomaticCanvasSize = Enum.AutomaticSize.Y
    scroll.CanvasSize = UDim2.new(0, 0, 0, 0)

    local sl = Instance.new("UIListLayout")
    sl.SortOrder = Enum.SortOrder.LayoutOrder
    sl.Padding = UDim.new(0, 6)
    sl.Parent = scroll

    local sp = Instance.new("UIPadding")
    sp.PaddingTop = UDim.new(0, 4)
    sp.PaddingLeft = UDim.new(0, 2)
    sp.PaddingRight = UDim.new(0, 6)
    sp.PaddingBottom = UDim.new(0, 8)
    sp.Parent = scroll

    Tabs[name] = { Btn = btn, Page = page, Scroll = scroll }

    btn.MouseButton1Click:Connect(function()
        for _, t in pairs(Tabs) do
            t.Btn.TextColor3 = C.TabIn
            t.Btn.BackgroundColor3 = C.Surf
            t.Btn.UIStroke.Color = Color3.fromRGB(30, 30, 35)
            t.Btn.Frame.Visible = false
            t.Page.Visible = false
        end
        btn.TextColor3 = C.TabAct
        btn.BackgroundColor3 = C.Surf3
        btn.UIStroke.Color = C.Accent
        indicator.Visible = true
        page.Visible = true
    end)

    return scroll
end

-- UI Components
local function section(parent, txt)
    local f = frame(parent, UDim2.new(1, 0, 0, 20), nil, C.BG, 0, 1)
    local l = label(f, txt:upper(), 10, C.Accent)
    l.Font = Enum.Font.GothamBold
    l.Size = UDim2.new(1, 0, 1, 0)
    l.TextYAlignment = Enum.TextYAlignment.Center
    return f
end

local function toggle(parent, txt, key, cb)
    local row = frame(parent, UDim2.new(1, 0, 0, 38), nil, C.Surf, 8)
    stroke(row, C.Border, 1)
    local l = label(row, "  " .. txt, 12, C.White)
    l.Size = UDim2.new(1, -58, 1, 0)
    l.TextYAlignment = Enum.TextYAlignment.Center

    local track = frame(row, UDim2.new(0, 36, 0, 18), UDim2.new(1, -46, 0.5, -9), C.OFF, 9)
    local knob = frame(track, UDim2.new(0, 14, 0, 14), UDim2.new(0, 2, 0.5, -7), Color3.fromRGB(255, 255, 255), 7)
    local trackStroke = stroke(track, Color3.fromRGB(45, 45, 50), 1)

    local function upd(v)
        TweenService:Create(track, TweenInfo.new(0.18, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {
            BackgroundColor3 = v and C.ON or C.OFF
        }):Play()
        TweenService:Create(knob, TweenInfo.new(0.18, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {
            Position = v and UDim2.new(1, -16, 0.5, -7) or UDim2.new(0, 2, 0.5, -7)
        }):Play()
        trackStroke.Color = v and Color3.fromRGB(255, 100, 100) or Color3.fromRGB(45, 45, 50)
    end
    upd(Config[key])

    local btn = Instance.new("TextButton")
    btn.Parent = row
    btn.Size = UDim2.new(1, 0, 1, 0)
    btn.BackgroundTransparency = 1
    btn.Text = ""
    btn.AutoButtonColor = false
    btn.MouseButton1Click:Connect(function()
        Config[key] = not Config[key]
        upd(Config[key])
        if cb then cb(Config[key]) end
    end)
    return row
end

local function slider(parent, txt, key, mn, mx, stp, sfx, cb)
    local wrap = frame(parent, UDim2.new(1, 0, 0, 54), nil, C.Surf, 8)
    stroke(wrap, C.Border, 1)

    local hdr = frame(wrap, UDim2.new(1, 0, 0, 22), nil, C.Surf, 0, 1)
    local lbl = label(hdr, "  " .. txt, 11, C.White)
    lbl.Size = UDim2.new(0.7, 0, 1, 0)
    lbl.TextYAlignment = Enum.TextYAlignment.Center

    local vlbl = label(hdr, tostring(Config[key]) .. (sfx or ""), 11, C.Accent)
    vlbl.Size = UDim2.new(0.3, -8, 1, 0)
    vlbl.TextXAlignment = Enum.TextXAlignment.Right
    vlbl.TextYAlignment = Enum.TextYAlignment.Center

    local track = frame(wrap, UDim2.new(1, -20, 0, 4), UDim2.new(0, 10, 1, -14), C.OFF, 2)
    local fill = frame(track, UDim2.new(0, 0, 1, 0), nil, C.Accent, 2)
    local kn = frame(fill, UDim2.new(0, 12, 0, 12), UDim2.new(1, -6, 0.5, -6), Color3.fromRGB(240, 240, 240), 6)
    stroke(kn, Color3.fromRGB(255, 255, 255), 1)

    local function setVal(v)
        v = math.clamp(v, mn, mx)
        if stp and stp > 0 then v = math.floor(v / stp + 0.5) * stp end
        Config[key] = v
        local r = (v - mn) / (mx - mn)
        fill.Size = UDim2.new(r, 0, 1, 0)
        vlbl.Text = tostring(math.floor(v * 100) / 100) .. (sfx or "")
        if cb then cb(v) end
    end
    setVal(Config[key])

    local drag = false
    track.InputBegan:Connect(function(i)
        if i.UserInputType == Enum.UserInputType.MouseButton1 then drag = true end
    end)
    local c1 = UserInputService.InputChanged:Connect(function(i)
        if drag and i.UserInputType == Enum.UserInputType.MouseMovement then
            local rel = math.clamp((i.Position.X - track.AbsolutePosition.X) / track.AbsoluteSize.X, 0, 1)
            setVal(mn + rel * (mx - mn))
        end
    end)
    local c2 = UserInputService.InputEnded:Connect(function(i)
        if i.UserInputType == Enum.UserInputType.MouseButton1 then drag = false end
    end)
    table.insert(_G.NovaConn, c1)
    table.insert(_G.NovaConn, c2)
    return wrap
end

local function dropdown(parent, txt, options, current, cb)
    local row = frame(parent, UDim2.new(1, 0, 0, 38), nil, C.Surf, 8)
    stroke(row, C.Border, 1)
    local l = label(row, "  " .. txt, 12, C.White)
    l.Size = UDim2.new(0.5, 0, 1, 0)
    l.TextYAlignment = Enum.TextYAlignment.Center

    local idx = 1
    for i, v in ipairs(options) do
        if v == current then idx = i end
    end

    local b = Instance.new("TextButton")
    b.Parent = row
    b.Size = UDim2.new(0, 130, 0, 24)
    b.Position = UDim2.new(1, -138, 0.5, -12)
    b.BackgroundColor3 = Color3.fromRGB(28, 28, 34)
    b.Text = current
    b.TextColor3 = C.Accent
    b.TextSize = 11
    b.Font = Enum.Font.GothamBold
    b.BorderSizePixel = 0
    b.AutoButtonColor = false
    corner(b, 6)
    stroke(b, C.Border, 1)

    b.MouseEnter:Connect(function()
        TweenService:Create(b, TweenInfo.new(0.1), { BackgroundColor3 = Color3.fromRGB(35, 35, 42) }):Play()
    end)
    b.MouseLeave:Connect(function()
        TweenService:Create(b, TweenInfo.new(0.1), { BackgroundColor3 = Color3.fromRGB(28, 28, 34) }):Play()
    end)
    b.MouseButton1Click:Connect(function()
        idx = idx % #options + 1
        b.Text = options[idx]
        if cb then cb(options[idx]) end
    end)
    return row
end

local function button(parent, txt, btnTxt, cb)
    local row = frame(parent, UDim2.new(1, 0, 0, 38), nil, C.Surf, 8)
    stroke(row, C.Border, 1)
    local l = label(row, "  " .. txt, 11, C.White)
    l.Size = UDim2.new(0.55, 0, 1, 0)
    l.TextYAlignment = Enum.TextYAlignment.Center

    local b = Instance.new("TextButton")
    b.Parent = row
    b.Size = UDim2.new(0, 110, 0, 24)
    b.Position = UDim2.new(1, -118, 0.5, -12)
    b.BackgroundColor3 = Color3.fromRGB(28, 28, 34)
    b.Text = btnTxt
    b.TextColor3 = C.Accent
    b.TextSize = 11
    b.Font = Enum.Font.GothamBold
    b.BorderSizePixel = 0
    b.AutoButtonColor = false
    corner(b, 6)
    stroke(b, C.Border, 1)

    b.MouseEnter:Connect(function()
        TweenService:Create(b, TweenInfo.new(0.1), { BackgroundColor3 = Color3.fromRGB(35, 35, 42) }):Play()
    end)
    b.MouseLeave:Connect(function()
        TweenService:Create(b, TweenInfo.new(0.1), { BackgroundColor3 = Color3.fromRGB(28, 28, 34) }):Play()
    end)
    b.MouseButton1Click:Connect(function()
        if cb then cb() end
    end)
    return row
end

local function infoRow(parent, txt)
    local row = frame(parent, UDim2.new(1, 0, 0, 34), nil, C.Surf, 8)
    stroke(row, C.Border, 1)
    local l = label(row, "  " .. txt, 11, C.Dim)
    l.Size = UDim2.new(1, -10, 1, 0)
    l.TextXAlignment = Enum.TextXAlignment.Left
    l.TextYAlignment = Enum.TextYAlignment.Center
    l.TextWrapped = true
    return row, l
end

-- ══════════════════════════════════════
--  TAB 1: AIMBOT
-- ══════════════════════════════════════
local AimScroll = newTab("Aimbot", "AIM")

section(AimScroll, "PRINCIPAL")
toggle(AimScroll, "Aimbot Ativado", "AimbotEnabled")

section(AimScroll, "MODOS DE AIMBOT")
toggle(AimScroll, "Aimbot Head (Head Lock)", "AimbotHead", function(v)
    if v then
        Config.AimbotPart = "Head"
        Config.AimbotLegit = false
        Config.AimbotRage = false
    end
end)
toggle(AimScroll, "Aimbot Legit (Smooth Alto)", "AimbotLegit", function(v)
    if v then
        Config.AimbotHead = false
        Config.AimbotRage = false
        Config.AimbotSmooth = 0.6
    end
end)
toggle(AimScroll, "Head Rage (Zero Smooth)", "AimbotRage", function(v)
    if v then
        Config.AimbotHead = false
        Config.AimbotLegit = false
        Config.AimbotPart = "Head"
        Config.AimbotRageSmooth = 0.02
    end
end)

section(AimScroll, "AIMBOT DELAY")
toggle(AimScroll, "Ativar Aimbot Delay", "AimbotDelay")
slider(AimScroll, "Delay (segundos)", "AimbotDelayTime", 0.01, 0.5, 0.01, "s")

section(AimScroll, "Silent Kill (FOV)")
toggle(AimScroll, "Silent Kill (Acerta alvos no FOV)", "SilentKill")
slider(AimScroll, "Distancia Max Silent", "SilentKillDist", 50, 800, 10, "m")
infoRow(AimScroll, "Esta funcao e projetada para direcionar disparos a alvos dentro do FOV")

section(AimScroll, "BIND DO AIMBOT")
dropdown(AimScroll, "Tecla do Aimbot", {
    "MouseButton2", "MouseButton1", "E", "Q", "X", "C", "CapsLock"
}, Config.AimbotBind, function(v)
    Config.AimbotBind = v
end)

section(AimScroll, "PARTE DO CORPO")
dropdown(AimScroll, "Aim Part", {
    "Head", "HumanoidRootPart", "UpperTorso", "Neck", "LowerTorso"
}, Config.AimbotPart, function(v)
    Config.AimbotPart = v
end)

section(AimScroll, "PRECISAO")
slider(AimScroll, "Suavidade (Smooth)", "AimbotSmooth", 0.01, 1.0, 0.01, "")
slider(AimScroll, "FOV Raio (pixels)", "AimbotFOV", 10, 800, 5, "px", function(v)
    if FOVCircle then FOVCircle.Radius = v end
end)

section(AimScroll, "PREDICAO")
toggle(AimScroll, "Predicao de Movimento", "AimbotPrediction")
slider(AimScroll, "Forca da Predicao", "AimbotPredValue", 0.01, 1.0, 0.01, "")

section(AimScroll, "FILTROS")
toggle(AimScroll, "Check de Time", "AimbotTeamCheck")
toggle(AimScroll, "Wall Check", "AimbotWallCheck")

section(AimScroll, "FOV CIRCLE")
toggle(AimScroll, "Mostrar Circulo FOV", "FOVEnabled", function(v)
    if FOVCircle then FOVCircle.Visible = Config.AimbotEnabled and v end
end)
slider(AimScroll, "Espessura do Circulo", "FOVThickness", 0.5, 5, 0.5, "px", function(v)
    if FOVCircle then FOVCircle.Thickness = v end
end)

-- ══════════════════════════════════════
--  TAB 2: ESP
-- ══════════════════════════════════════
local ESPScroll = newTab("ESP", "ESP")

section(ESPScroll, "PRINCIPAL")
toggle(ESPScroll, "ESP Ativado", "ESPEnabled")
toggle(ESPScroll, "ESP All (Liga Tudo)", "ESPAll")
slider(ESPScroll, "Distancia Maxima", "ESPMaxDist", 50, 3000, 50, "m")

section(ESPScroll, "ELEMENTOS")
toggle(ESPScroll, "Caixas (Boxes)", "ESPBoxes")
toggle(ESPScroll, "Nomes", "ESPNames")
toggle(ESPScroll, "Barra de Vida (HP)", "ESPHealth")
toggle(ESPScroll, "Distancia", "ESPDistance")
toggle(ESPScroll, "Esqueleto (Skeleton)", "ESPSkeleton")
toggle(ESPScroll, "Cor do Time", "ESPTeamColor")

section(ESPScroll, "LINHAS (TRACERS)")
toggle(ESPScroll, "Linhas (Tracers)", "ESPTracers")
dropdown(ESPScroll, "Origem da Linha", { "Bottom", "Center", "Top" }, Config.ESPTracerOrigin, function(v)
    Config.ESPTracerOrigin = v
end)

section(ESPScroll, "INFO")
toggle(ESPScroll, "Info (Arma + HP)", "ESPInfo")

section(ESPScroll, "TAMANHOS")
slider(ESPScroll, "Espessura das Caixas", "ESPBoxThick", 0.5, 5, 0.5, "px")
slider(ESPScroll, "Tamanho do Nome", "ESPNameSize", 10, 24, 1, "px")

-- ══════════════════════════════════════
--  TAB 3: CHAMS
-- ══════════════════════════════════════
local ChamsScroll = newTab("Chams", "CH")

section(ChamsScroll, "PAINEL CHAMS COMPLETO")
toggle(ChamsScroll, "Chams Ativado", "ChamsEnabled")
toggle(ChamsScroll, "Chams Verde", "ChamsGreen")
toggle(ChamsScroll, "Outline (Contorno)", "ChamsOutline")

section(ChamsScroll, "AJUSTES")
slider(ChamsScroll, "Transparencia", "ChamsTransparency", 0.0, 1.0, 0.05, "")

section(ChamsScroll, "STREAM MODE")
toggle(ChamsScroll, "Stream Mode (Oculta Nomes)", "StreamMode")
infoRow(ChamsScroll, "Stream Mode esconde nomes reais no ESP")

section(ChamsScroll, "DESTRUCT")
toggle(ChamsScroll, "Destruct (Mapa Transparente)", "Destruct", function(v)
    toggleDestruct(v)
end)
infoRow(ChamsScroll, "Destruct torna o mapa transparente")

-- ══════════════════════════════════════
--  TAB 4: MISC
-- ══════════════════════════════════════
local MiscScroll = newTab("Misc", "CFG")

section(MiscScroll, "INFORMACOES")

local _, gameLbl = infoRow(MiscScroll, "Jogo: carregando...")
pcall(function()
    local info = game:GetService("MarketplaceService"):GetProductInfo(game.PlaceId)
    gameLbl.Text = "  Jogo: " .. tostring(info.Name)
end)

local _, plrLbl = infoRow(MiscScroll, "Jogadores: " .. #Players:GetPlayers())
local c3 = RunService.Heartbeat:Connect(function()
    pcall(function()
        plrLbl.Text = "  Jogadores online: " .. #Players:GetPlayers()
    end)
end)
table.insert(_G.NovaConn, c3)

section(MiscScroll, "ACOES")
button(MiscScroll, "Recarregar ESP", "Recarregar", function()
    for p, _ in pairs(ESPObjects) do removeESP(p) end
    for _, p in ipairs(Players:GetPlayers()) do createESP(p) end
end)
button(MiscScroll, "Desativar Tudo", "Desativar", function()
    Config.AimbotEnabled = false
    Config.ESPEnabled = false
    Config.FOVEnabled = false
    Config.ChamsEnabled = false
    Config.SilentKill = false
    Config.ESPAll = false
    Config.Destruct = false
    toggleDestruct(false)
    if FOVCircle then FOVCircle.Visible = false end
end)
button(MiscScroll, "Fechar Painel", "Fechar", function()
    SG.Enabled = false
end)

section(MiscScroll, "STATUS EM TEMPO REAL")
local _, statusLbl = infoRow(MiscScroll, "")
statusLbl.TextYAlignment = Enum.TextYAlignment.Center
local c4 = RunService.Heartbeat:Connect(function()
    pcall(function()
        local ec = 0
        for _ in pairs(ESPObjects) do ec = ec + 1 end
        local mode = "Normal"
        if Config.AimbotRage then
            mode = "RAGE"
        elseif Config.AimbotLegit then
            mode = "LEGIT"
        elseif Config.AimbotHead then
            mode = "HEAD"
        end
        local newText = string.format(
            "Aim: %s [%s]  |  ESP: %s  |  Chams: %s  |  Players ESP: %d",
            Config.AimbotEnabled and "ON" or "OFF",
            mode,
            Config.ESPEnabled and "ON" or "OFF",
            Config.ChamsEnabled and "ON" or "OFF",
            ec
        )
        statusLbl.Text = "  " .. newText
        
        -- Atualiza também a Barra de Status inferior da Janela principal
        StatusText.Text = newText
        if Config.AimbotEnabled or Config.ESPEnabled or Config.ChamsEnabled then
            StatusIcon.BackgroundColor3 = Color3.fromRGB(60, 210, 110) -- Verde brilhante
            StatusIcon.UIStroke.Color = Color3.fromRGB(100, 255, 150)
        else
            StatusIcon.BackgroundColor3 = Color3.fromRGB(230, 40, 40) -- Vermelho
            StatusIcon.UIStroke.Color = Color3.fromRGB(255, 100, 100)
        end
    end)
end)
table.insert(_G.NovaConn, c4)

section(MiscScroll, "KEYBINDS")
local kbRow = frame(MiscScroll, UDim2.new(1, 0, 0, 70), nil, C.Surf2, 8)
stroke(kbRow, C.Border, 1)
local kbLbl = label(kbRow, "", 11, C.Dim)
kbLbl.Size = UDim2.new(1, 0, 1, 0)
kbLbl.TextWrapped = true
kbLbl.TextYAlignment = Enum.TextYAlignment.Center
kbLbl.Text = "  [INSERT] Abrir/Fechar Painel\n  [Bind] Ativar Aimbot\n  [DELETE] Desativar tudo"

-- ══════════════════════════════════════
--  TAB 5: INFO
-- ══════════════════════════════════════
local CreditsScroll = newTab("Info", "?")

section(CreditsScroll, "NOVA PANEL v3.0")
infoRow(CreditsScroll, "Desenvolvido por: davixp")
infoRow(CreditsScroll, "Aimbot: Head, Legit, Rage, Delay, Silent")
infoRow(CreditsScroll, "ESP: Boxes, Names, HP, Dist, Skeleton, Tracers")
infoRow(CreditsScroll, "Chams: Completo, Verde, Outline")
infoRow(CreditsScroll, "Extras: Destruct, Stream Mode, ESP All")

section(CreditsScroll, "VERSAO")
infoRow(CreditsScroll, "Versao: 3.0 FULL")
infoRow(CreditsScroll, "Executor: Universal")
infoRow(CreditsScroll, "Status: Funcionando")
infoRow(CreditsScroll, "Drawing API: " .. (hasDrawing and "Disponivel" or "Indisponivel"))

-- ══════════════════════════════════════
--  DRAGGING (Permite arrastar tanto pela TopBar quanto pela Sidebar)
-- ══════════════════════════════════════
local dragging = false
local dragStart = nil
local startPos = nil

local function setupDrag(gui)
    gui.InputBegan:Connect(function(i)
        if i.UserInputType == Enum.UserInputType.MouseButton1 then
            dragging = true
            dragStart = i.Position
            startPos = Main.Position
        end
    end)
end

setupDrag(TopBar)
setupDrag(TabBar)

local c5 = UserInputService.InputChanged:Connect(function(i)
    if dragging and i.UserInputType == Enum.UserInputType.MouseMovement then
        local d = i.Position - dragStart
        Main.Position = UDim2.new(
            startPos.X.Scale, startPos.X.Offset + d.X,
            startPos.Y.Scale, startPos.Y.Offset + d.Y
        )
    end
end)
table.insert(_G.NovaConn, c5)

local c6 = UserInputService.InputEnded:Connect(function(i)
    if i.UserInputType == Enum.UserInputType.MouseButton1 then
        dragging = false
    end
end)
table.insert(_G.NovaConn, c6)

-- ══════════════════════════════════════
--  MINIMIZE / CLOSE
-- ══════════════════════════════════════
local minimized = false
MinBtn.MouseButton1Click:Connect(function()
    minimized = not minimized
    TweenService:Create(Main, TweenInfo.new(0.2, Enum.EasingStyle.Quad), {
        Size = minimized and UDim2.new(0, 560, 0, 44) or UDim2.new(0, 560, 0, 480)
    }):Play()
    MinBtn.Text = minimized and "+" or "─"
end)

CloseBtn.MouseButton1Click:Connect(function()
    TweenService:Create(Main, TweenInfo.new(0.2), {
        Size = UDim2.new(0, 0, 0, 0),
        Position = UDim2.new(
            Main.Position.X.Scale, Main.Position.X.Offset + 280,
            Main.Position.Y.Scale, Main.Position.Y.Offset + 240
        )
    }):Play()
    task.delay(0.22, function()
        SG.Enabled = false
        Main.Size = UDim2.new(0, 560, 0, 480)
        Main.Position = UDim2.new(0.5, -280, 0.5, -240)
    end)
end)

-- ══════════════════════════════════════
--  KEYBINDS
-- ══════════════════════════════════════
local c7 = UserInputService.InputBegan:Connect(function(i, gp)
    if gp then return end
    if i.KeyCode == Enum.KeyCode.Insert then
        SG.Enabled = not SG.Enabled
        if SG.Enabled then
            Main.Size = UDim2.new(0, 0, 0, 0)
            Main.Position = UDim2.new(0.5, 0, 0.5, 0)
            TweenService:Create(Main, TweenInfo.new(0.25, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {
                Size = UDim2.new(0, 560, 0, 480),
                Position = UDim2.new(0.5, -280, 0.5, -240)
            }):Play()
        end
    end
    if i.KeyCode == Enum.KeyCode.Delete then
        Config.AimbotEnabled = false
        Config.ESPEnabled = false
        Config.FOVEnabled = false
        Config.ChamsEnabled = false
        Config.SilentKill = false
        Config.ESPAll = false
        Config.Destruct = false
        if FOVCircle then FOVCircle.Visible = false end
        toggleDestruct(false)
    end
end)
table.insert(_G.NovaConn, c7)

-- ══════════════════════════════════════
--  INICIALIZACAO
-- ══════════════════════════════════════
for _, p in ipairs(Players:GetPlayers()) do
    createESP(p)
end

local c8 = Players.PlayerAdded:Connect(createESP)
local c9 = Players.PlayerRemoving:Connect(removeESP)
table.insert(_G.NovaConn, c8)
table.insert(_G.NovaConn, c9)

local c10 = RunService.RenderStepped:Connect(function()
    -- FOV
    if FOVCircle then
        if Config.FOVEnabled and Config.AimbotEnabled then
            FOVCircle.Visible = true
            FOVCircle.Position = Vector2.new(Camera.ViewportSize.X / 2, Camera.ViewportSize.Y / 2)
            FOVCircle.Radius = Config.AimbotFOV
            FOVCircle.Thickness = Config.FOVThickness
        else
            FOVCircle.Visible = false
        end
    end

    pcall(runAimbot)
    pcall(updateESP)
end)
table.insert(_G.NovaConn, c10)

-- Aba padrão ao inicializar
Tabs["Aimbot"].Btn.TextColor3 = C.TabAct
Tabs["Aimbot"].Btn.BackgroundColor3 = C.Surf3
Tabs["Aimbot"].Btn.UIStroke.Color = C.Accent
Tabs["Aimbot"].Btn.Frame.Visible = true
Tabs["Aimbot"].Page.Visible = true

-- Animação de entrada
Main.Size = UDim2.new(0, 0, 0, 0)
Main.Position = UDim2.new(0.5, 0, 0.5, 0)
task.delay(0.1, function()
    TweenService:Create(Main, TweenInfo.new(0.3, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {
        Size = UDim2.new(0, 560, 0, 480),
        Position = UDim2.new(0.5, -280, 0.5, -240)
    }):Play()
end)
-- Aba de Login dentro do painel
local LoginFrame = frame(Main, UDim2.new(1, -90, 1, -85), UDim2.new(0, 82, 0, 50), C.Surf, 12)
LoginFrame.Visible = true
LoginFrame.ZIndex = 50 -- fica por cima do Content

-- Esconde o conteúdo até login
Content.Visible = false

-- Título
local LoginTitle = label(LoginFrame, "🔑 Login Necessário", 18, C.Accent, UDim2.new(0, 0, 0, 10), Enum.TextXAlignment.Center)
LoginTitle.Size = UDim2.new(1, 0, 0, 30)

-- Campo Usuário
local UserBox = Instance.new("TextBox")
UserBox.Parent = LoginFrame
UserBox.Size = UDim2.new(0.8, 0, 0, 30)
UserBox.Position = UDim2.new(0.1, 0, 0, 70)
UserBox.PlaceholderText = "Usuário"
UserBox.Text = ""
UserBox.TextSize = 14
UserBox.Font = Enum.Font.Gotham
UserBox.BackgroundColor3 = C.Surf2
UserBox.TextColor3 = C.White
corner(UserBox, 6)

-- Campo Senha
local PassBox = Instance.new("TextBox")
PassBox.Parent = LoginFrame
PassBox.Size = UDim2.new(0.8, 0, 0, 30)
PassBox.Position = UDim2.new(0.1, 0, 0, 120)
PassBox.PlaceholderText = "Senha"
PassBox.Text = ""
PassBox.TextSize = 14
PassBox.Font = Enum.Font.Gotham
PassBox.BackgroundColor3 = C.Surf2
PassBox.TextColor3 = C.White
PassBox.ClearTextOnFocus = false
corner(PassBox, 6)

-- Botão Login
local LoginBtn = Instance.new("TextButton")
LoginBtn.Parent = LoginFrame
LoginBtn.Size = UDim2.new(0.4, 0, 0, 32)
LoginBtn.Position = UDim2.new(0.3, 0, 0, 180)
LoginBtn.Text = "Entrar"
LoginBtn.TextColor3 = C.White
LoginBtn.Font = Enum.Font.GothamBold
LoginBtn.BackgroundColor3 = C.Accent
corner(LoginBtn, 8)

-- Ação do botão
LoginBtn.MouseButton1Click:Connect(function()
    if UserBox.Text == "davixp" and PassBox.Text == "130910" then
        StatusText.Text = "Status: Login bem-sucedido!"
        LoginFrame.Visible = false -- esconde login
        Content.Visible = true     -- mostra painel principal
    else
        StatusText.Text = "Status: Usuário ou senha incorretos!"
    end
end)

print("[Nova Panel v3.0] Redesenhado com sucesso! | by davixp")
print("[Nova Panel] INSERT = abrir/fechar | DELETE = desativar tudo")
print("[Nova Panel] Drawing API: " .. (hasDrawing and "OK" or "NAO DISPONIVEL"))
