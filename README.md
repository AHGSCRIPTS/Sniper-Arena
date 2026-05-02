local Rayfield = loadstring(game:HttpGet('https://sirius.menu/rayfield'))()

local Window = Rayfield:CreateWindow({
    Name = "Sniper Arena | Best Cheats",
    Icon = 0,
    LoadingTitle = "Sniper Arena | Best Cheats",
    LoadingSubtitle = "By AHG",
    Theme = "Default",
    DisableRayfieldPrompts = false,
    DisableBuildWarnings = false,
    ConfigurationSaving = {
        Enabled = true,
        FolderName = "AHG",
        FileName = "SniperArena",
    },
    Discord = {
        Enabled = false,
    },
    KeySystem = false,
})

-- ===================== GLOBALS =====================
getgenv().AHG = getgenv().AHG or {}
getgenv().AHG.AimbotEnabled = false
getgenv().AHG.AimbotPart = "Head"
getgenv().AHG.Smoothing = 5
getgenv().AHG.WallCheck = false
getgenv().AHG.FOVSize = 80
getgenv().AHG.ShowFOV = false
getgenv().AHG.FOVColor = Color3.fromRGB(255, 255, 255)
getgenv().AHG.ESPEnabled = false
getgenv().AHG.ESPBox = false
getgenv().AHG.ESPSkeleton = false
getgenv().AHG.ESPTracer = false
getgenv().AHG.ESPName = false
getgenv().AHG.ESPColor = Color3.fromRGB(255, 0, 0)

-- ===================== SERVICES =====================
local Players = game:GetService("Players")
local LocalPlayer = Players.LocalPlayer
local PlayerGui = LocalPlayer:WaitForChild("PlayerGui")
local Camera = workspace.CurrentCamera
local RunService = game:GetService("RunService")

-- ===================== FOV CIRCLE =====================
local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "AHG_FOV"
ScreenGui.ResetOnSpawn = false
ScreenGui.IgnoreGuiInset = true
ScreenGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
ScreenGui.Parent = PlayerGui

local FOVFrame = Instance.new("Frame")
FOVFrame.Name = "FOVCircle"
FOVFrame.AnchorPoint = Vector2.new(0.5, 0.5)
FOVFrame.BackgroundTransparency = 1
FOVFrame.BorderSizePixel = 0
FOVFrame.ZIndex = 999
FOVFrame.Position = UDim2.new(0.5, 0, 0.5, 0)
FOVFrame.Parent = ScreenGui

local UICorner = Instance.new("UICorner")
UICorner.CornerRadius = UDim.new(1, 0)
UICorner.Parent = FOVFrame

local UIStroke = Instance.new("UIStroke")
UIStroke.Thickness = 1.5
UIStroke.Color = getgenv().AHG.FOVColor
UIStroke.Parent = FOVFrame

local function UpdateFOV()
    local size = getgenv().AHG.FOVSize * 2
    FOVFrame.Size = UDim2.new(0, size, 0, size)
    FOVFrame.Position = UDim2.new(0.5, 0, 0.5, 0)
    FOVFrame.Visible = getgenv().AHG.ShowFOV and getgenv().AHG.AimbotEnabled
    UIStroke.Color = getgenv().AHG.FOVColor
end

-- ===================== ESP DRAWINGS =====================
local ESPObjects = {}

local function ClearESP()
    for _, t in pairs(ESPObjects) do
        for _, d in pairs(t) do
            if type(d) == "table" then
                for _, l in pairs(d) do
                    if l and l.Remove then l:Remove() end
                end
            elseif d and d.Remove then
                d:Remove()
            end
        end
    end
    ESPObjects = {}
end

local function HideESP(esp)
    esp.Box.Visible = false
    esp.NameDist.Visible = false
    esp.Tracer.Visible = false
    for _, l in pairs(esp.Skeleton) do l.Visible = false end
end

local function MakeLabel()
    local d = Drawing.new("Text")
    d.Text = ""
    d.Size = 13
    d.Color = getgenv().AHG.ESPColor
    d.Outline = true
    d.Center = true
    d.Visible = false
    return d
end

local function MakeBox()
    local d = Drawing.new("Square")
    d.Thickness = 1.5
    d.Color = getgenv().AHG.ESPColor
    d.Filled = false
    d.Visible = false
    return d
end

local function MakeLine()
    local d = Drawing.new("Line")
    d.Thickness = 1.5
    d.Color = getgenv().AHG.ESPColor
    d.Visible = false
    return d
end

local SKELETON_PARTS = {
    {"Head", "UpperTorso"},
    {"UpperTorso", "LowerTorso"},
    {"UpperTorso", "LeftUpperArm"},
    {"LeftUpperArm", "LeftLowerArm"},
    {"LeftLowerArm", "LeftHand"},
    {"UpperTorso", "RightUpperArm"},
    {"RightUpperArm", "RightLowerArm"},
    {"RightLowerArm", "RightHand"},
    {"LowerTorso", "LeftUpperLeg"},
    {"LeftUpperLeg", "LeftLowerLeg"},
    {"LeftLowerLeg", "LeftFoot"},
    {"LowerTorso", "RightUpperLeg"},
    {"RightUpperLeg", "RightLowerLeg"},
    {"RightLowerLeg", "RightFoot"},
}

local function GetESPForPlayer(player)
    if not ESPObjects[player] then
        local obj = {
            Box = MakeBox(),
            NameDist = MakeLabel(),
            Tracer = MakeLine(),
            Skeleton = {},
        }
        for i = 1, #SKELETON_PARTS do
            obj.Skeleton[i] = MakeLine()
        end
        ESPObjects[player] = obj
    end
    return ESPObjects[player]
end

local function RemoveESPForPlayer(player)
    if ESPObjects[player] then
        local t = ESPObjects[player]
        if t.Box then t.Box:Remove() end
        if t.NameDist then t.NameDist:Remove() end
        if t.Tracer then t.Tracer:Remove() end
        for _, l in pairs(t.Skeleton) do
            if l then l:Remove() end
        end
        ESPObjects[player] = nil
    end
end

-- ===================== AIMBOT =====================
local function GetClosestPlayer(localPos)
    local closestPlayer = nil
    local closestDist = getgenv().AHG.FOVSize
    local center = Camera.ViewportSize / 2

    for _, player in ipairs(Players:GetPlayers()) do
        if player == LocalPlayer then continue end
        local char = player.Character
        if not char then continue end

        local hum = char:FindFirstChildOfClass("Humanoid")
        if not hum or hum.Health == nil or hum.Health <= 0 then continue end

        local targetPart = char:FindFirstChild(getgenv().AHG.AimbotPart) or char:FindFirstChild("HumanoidRootPart")
        if not targetPart then continue end

        if getgenv().AHG.WallCheck then
            local origin = Camera.CFrame.Position
            local ray = Ray.new(origin, (targetPart.Position - origin).Unit * 1000)
            local hit = workspace:FindPartOnRayWithIgnoreList(ray, {LocalPlayer.Character, Camera})
            if hit and not char:IsAncestorOf(hit) then continue end
        end

        local screenPos, onScreen = Camera:WorldToViewportPoint(targetPart.Position)
        if not onScreen then continue end

        local dist = (Vector2.new(screenPos.X, screenPos.Y) - center).Magnitude
        if dist < closestDist then
            closestDist = dist
            closestPlayer = player
        end
    end
    return closestPlayer
end

-- ===================== MAIN LOOP =====================
RunService.RenderStepped:Connect(function()
    UpdateFOV()

    local localChar = LocalPlayer.Character
    local localRoot = localChar and localChar:FindFirstChild("HumanoidRootPart")
    local localPos = localRoot and localRoot.Position

    -- AIMBOT
    if getgenv().AHG.AimbotEnabled then
        local target = GetClosestPlayer(localPos)
        if target and target.Character then
            local part = target.Character:FindFirstChild(getgenv().AHG.AimbotPart)
                or target.Character:FindFirstChild("HumanoidRootPart")
            if part then
                local smooth = math.max(1, getgenv().AHG.Smoothing)
                local goalCF = CFrame.new(Camera.CFrame.Position, part.Position)
                Camera.CFrame = Camera.CFrame:Lerp(goalCF, 1 / smooth)
            end
        end
    end

    -- ESP
    if not getgenv().AHG.ESPEnabled then
        for _, player in ipairs(Players:GetPlayers()) do
            if player ~= LocalPlayer and ESPObjects[player] then
                HideESP(ESPObjects[player])
            end
        end
        return
    end

    local espColor = getgenv().AHG.ESPColor
    local viewX = Camera.ViewportSize.X
    local viewY = Camera.ViewportSize.Y

    for _, player in ipairs(Players:GetPlayers()) do
        if player == LocalPlayer then continue end

        local char = player.Character
        local esp = GetESPForPlayer(player)

        local hum = char and char:FindFirstChildOfClass("Humanoid")
        if not char or not hum or hum.Health == nil or hum.Health <= 0 then
            HideESP(esp)
            continue
        end

        local root = char:FindFirstChild("HumanoidRootPart")
        if not root then HideESP(esp) continue end

        if localPos and (root.Position - localPos).Magnitude > 1000 then
            HideESP(esp)
            continue
        end

        local rootScreen, rootOnScreen = Camera:WorldToViewportPoint(root.Position)
        if not rootOnScreen then HideESP(esp) continue end

        local headPart = char:FindFirstChild("Head")
        local feetScreen = Camera:WorldToViewportPoint(root.Position - Vector3.new(0, 3, 0))

        -- BOX
        if getgenv().AHG.ESPBox and headPart then
            local headScreen, headOnScreen = Camera:WorldToViewportPoint(headPart.Position + Vector3.new(0, 0.7, 0))
            if headOnScreen then
                local boxH = math.abs(headScreen.Y - feetScreen.Y)
                local boxW = boxH * 0.55
                esp.Box.Size = Vector2.new(boxW, boxH)
                esp.Box.Position = Vector2.new(rootScreen.X - boxW / 2, headScreen.Y)
                esp.Box.Color = espColor
                esp.Box.Visible = true
            else
                esp.Box.Visible = false
            end
        else
            esp.Box.Visible = false
        end

        -- NAME + DISTANCE
        if getgenv().AHG.ESPName and headPart then
            local headScreen, headOnScreen = Camera:WorldToViewportPoint(headPart.Position + Vector3.new(0, 0.6, 0))
            if headOnScreen then
                local dist = localPos and math.floor((root.Position - localPos).Magnitude) or 0
                esp.NameDist.Text = player.Name .. " | " .. dist .. "m"
                esp.NameDist.Position = Vector2.new(headScreen.X, headScreen.Y - 18)
                esp.NameDist.Color = espColor
                esp.NameDist.Visible = true
            else
                esp.NameDist.Visible = false
            end
        else
            esp.NameDist.Visible = false
        end

        -- TRACER
        if getgenv().AHG.ESPTracer then
            esp.Tracer.From = Vector2.new(viewX / 2, viewY)
            esp.Tracer.To = Vector2.new(rootScreen.X, rootScreen.Y)
            esp.Tracer.Color = espColor
            esp.Tracer.Visible = true
        else
            esp.Tracer.Visible = false
        end

        -- SKELETON
        for i, bones in ipairs(SKELETON_PARTS) do
            local p1 = char:FindFirstChild(bones[1])
            local p2 = char:FindFirstChild(bones[2])
            local line = esp.Skeleton[i]
            if getgenv().AHG.ESPSkeleton and p1 and p2 then
                local s1, on1 = Camera:WorldToViewportPoint(p1.Position)
                local s2, on2 = Camera:WorldToViewportPoint(p2.Position)
                if on1 and on2 then
                    line.From = Vector2.new(s1.X, s1.Y)
                    line.To = Vector2.new(s2.X, s2.Y)
                    line.Color = espColor
                    line.Visible = true
                else
                    line.Visible = false
                end
            else
                line.Visible = false
            end
        end
    end
end)

Players.PlayerRemoving:Connect(function(p)
    RemoveESPForPlayer(p)
end)

-- ===================== TABS =====================
local AimTab = Window:CreateTab("AIM", "crosshair")
local ESPTab = Window:CreateTab("ESP", "eye")

-- ===================== AIM TAB =====================
AimTab:CreateToggle({
    Name = "Enable Aimbot",
    CurrentValue = false,
    Flag = "AimbotEnabled",
    Callback = function(val)
        getgenv().AHG.AimbotEnabled = val
    end,
})

AimTab:CreateDropdown({
    Name = "Aim Target",
    Options = {"Head", "Torso"},
    CurrentOption = {"Head"},
    Flag = "AimbotPart",
    Callback = function(val)
        getgenv().AHG.AimbotPart = val[1] == "Torso" and "UpperTorso" or "Head"
    end,
})

AimTab:CreateSlider({
    Name = "Smoothing Aim",
    Range = {1, 10},
    Increment = 1,
    Suffix = "",
    CurrentValue = 5,
    Flag = "AimbotSmoothing",
    Callback = function(val)
        getgenv().AHG.Smoothing = val
    end,
})

AimTab:CreateToggle({
    Name = "Wall Check",
    CurrentValue = false,
    Flag = "WallCheck",
    Callback = function(val)
        getgenv().AHG.WallCheck = val
    end,
})

AimTab:CreateSlider({
    Name = "FOV Size",
    Range = {5, 150},
    Increment = 1,
    Suffix = "px",
    CurrentValue = 80,
    Flag = "FOVSize",
    Callback = function(val)
        getgenv().AHG.FOVSize = val
    end,
})

AimTab:CreateToggle({
    Name = "Show FOV",
    CurrentValue = false,
    Flag = "ShowFOV",
    Callback = function(val)
        getgenv().AHG.ShowFOV = val
    end,
})

AimTab:CreateColorPicker({
    Name = "FOV Color",
    Color = Color3.fromRGB(255, 255, 255),
    Flag = "FOVColor",
    Callback = function(val)
        getgenv().AHG.FOVColor = val
        UIStroke.Color = val
    end,
})

-- ===================== ESP TAB =====================
ESPTab:CreateToggle({
    Name = "Enable ESP",
    CurrentValue = false,
    Flag = "ESPEnabled",
    Callback = function(val)
        getgenv().AHG.ESPEnabled = val
        if not val then ClearESP() end
    end,
})

ESPTab:CreateToggle({
    Name = "ESP Box",
    CurrentValue = false,
    Flag = "ESPBox",
    Callback = function(val)
        getgenv().AHG.ESPBox = val
    end,
})

ESPTab:CreateToggle({
    Name = "ESP Skeleton",
    CurrentValue = false,
    Flag = "ESPSkeleton",
    Callback = function(val)
        getgenv().AHG.ESPSkeleton = val
    end,
})

ESPTab:CreateToggle({
    Name = "ESP Tracer",
    CurrentValue = false,
    Flag = "ESPTracer",
    Callback = function(val)
        getgenv().AHG.ESPTracer = val
    end,
})

ESPTab:CreateToggle({
    Name = "ESP Name + Distance",
    CurrentValue = false,
    Flag = "ESPName",
    Callback = function(val)
        getgenv().AHG.ESPName = val
    end,
})

ESPTab:CreateColorPicker({
    Name = "ESP Color",
    Color = Color3.fromRGB(255, 0, 0),
    Flag = "ESPColor",
    Callback = function(val)
        getgenv().AHG.ESPColor = val
    end,
})

-- ===================== LOAD CONFIG =====================
Rayfield:LoadConfiguration()

Rayfield:Notify({
    Title = "Sniper Arena | Best Cheats",
    Content = "Script loaded successfully!",
    Duration = 5,
    Image = 4483362458,
})
