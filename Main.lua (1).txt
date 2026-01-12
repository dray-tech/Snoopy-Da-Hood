--[[
    Da Hood Script - Linoria Library
    Optimized & Clean
]]

-- Load Linoria Library
local repo = 'https://raw.githubusercontent.com/violin-suzutsuki/LinoriaLib/main/'

local Library = loadstring(game:HttpGet(repo .. 'Library.lua'))()
local ThemeManager = loadstring(game:HttpGet(repo .. 'addons/ThemeManager.lua'))()
local SaveManager = loadstring(game:HttpGet(repo .. 'addons/SaveManager.lua'))()
Library.Toggles = Library.Toggles or {}
Library.Options = Library.Options or {}
local Toggles = Library.Toggles
local Options = Library.Options

-- Services
local Players = game:GetService('Players')
local RunService = game:GetService('RunService')
local Workspace = game:GetService('Workspace')
local UserInputService = game:GetService('UserInputService')
local GuiService = game:GetService('GuiService')
local TweenService = game:GetService('TweenService')

-- Local Player
local LocalPlayer = Players.LocalPlayer

-- Variables
local CashAuraEnabled = false
local CashAuraRange = 25
local CashDropEnabled = false
local CashDropDelay = 5
local ESPEnabled = false
local ShowNames = true
local MoneyESPEnabled = false
local ESPColor = Color3.fromRGB(255, 0, 0)
local ESPTextColor = Color3.fromRGB(255, 255, 255)
local ESPTextSize = 14
local UseHealthColor = true
local ESPObjects = {}
local MoneyESPObjects = {}

-- CFrame Speed Variables
local CFrameSpeedEnabled = false
local CFrameSpeedValue = 1.5

-- CFrame Fly Variables
local CFrameFlyEnabled = false
local CFrameFlySpeed = 50

-- Aimbot Variables
local AimbotEnabled = false
local AimbotFOV = 100
local AimbotTarget = "Head" -- "Head" or "HumanoidRootPart"
local FOVCircleEnabled = true
local FOVCircleTransparency = 0.5 -- 0 = invisible, 1 = fully opaque
local FOVCircleColor = Color3.fromRGB(255, 255, 255)
local TracersEnabled = false
local TracerColor = Color3.fromRGB(255, 0, 0)
local AimbotKey = Enum.UserInputType.MouseButton2 -- Right click to aim
local AimbotKeybind = "MouseButton2"
local CurrentTarget = nil
local AimbotSmoothing = 0.3 -- Lower default for smoother aim
local AimbotLockMode = "Camera" -- Locked to camera for reliability
local AimbotCameraTween = 0 -- Seconds; 0 = instant snap
local AimbotPrediction = 0
local AimbotHitChance = 100
local AimbotLOSCheck = false
local AimbotLockedColor = Color3.fromRGB(255, 80, 80)
local OriginalMouseDelta = UserInputService.MouseDeltaSensitivity
local AimbotKOCheck = false
local AimbotCrewCheck = false
local AimbotForceFieldCheck = false
local AimbotFOVFilled = false
local AimbotDownedGrabbedCheck = false
local AimbotGunOnly = false
local AimbotDynamicPrediction = false

-- Anti-slow WalkSpeed
local WalkSpeedEnabled = false
local WalkSpeedValue = 16
local WalkSpeedKeybind = "X"

-- Hitbox expander
local HitboxEnabled = false
local HitboxSize = 5

-- Noclip / fog / ambient
local NoclipEnabled = false
local NoFogEnabled = false
local AmbientColor = nil

-- Create FOV Circle
local FOVCircle = Drawing.new("Circle")
FOVCircle.Thickness = 2
FOVCircle.NumSides = 64
FOVCircle.Radius = AimbotFOV
FOVCircle.Filled = AimbotFOVFilled
FOVCircle.Visible = false
FOVCircle.Color = FOVCircleColor
FOVCircle.Transparency = FOVCircleTransparency

local FOVCircleOutline = Drawing.new("Circle")
FOVCircleOutline.Thickness = 3
FOVCircleOutline.NumSides = 64
FOVCircleOutline.Radius = AimbotFOV + 1
FOVCircleOutline.Filled = false
FOVCircleOutline.Visible = false
FOVCircleOutline.Color = Color3.fromRGB(0, 0, 0)
FOVCircleOutline.Transparency = FOVCircleTransparency

-- Tracer storage
local TracerLines = {}

-- Create Window
local Window = Library:CreateWindow({
    Title = 'Da Hood',
    Center = true,
    AutoShow = true,
    TabPadding = 8,
    MenuFadeTime = 0.2
})

-- Create Tabs
local Tabs = {
    Main = Window:AddTab('Main'),
    Aimbot = Window:AddTab('Aimbot'),
    Visuals = Window:AddTab('Visuals'),
    Settings = Window:AddTab('Settings')
}

-- =============================================
-- UTILITY FUNCTIONS
-- =============================================
local function GetCharacter()
    return LocalPlayer and LocalPlayer.Character
end

local function GetHRP()
    local char = GetCharacter()
    return char and char:FindFirstChild('HumanoidRootPart')
end

local function GetDistance(pos1, pos2)
    return (pos1 - pos2).Magnitude
end

local function IsTyping()
    -- Check if user is typing in chat or any textbox
    return UserInputService:GetFocusedTextBox() ~= nil
end

-- WalkSpeed helper
local function ApplyWalkSpeed()
    local character = GetCharacter()
    local humanoid = character and character:FindFirstChildOfClass('Humanoid')
    if not humanoid then return end
    humanoid.WalkSpeed = WalkSpeedEnabled and WalkSpeedValue or 16
end

local function HookWalkSpeed(character)
    local humanoid = character and character:FindFirstChildOfClass('Humanoid')
    if humanoid then
        humanoid:GetPropertyChangedSignal("WalkSpeed"):Connect(function()
            if WalkSpeedEnabled then
                humanoid.WalkSpeed = WalkSpeedValue
            end
        end)
        ApplyWalkSpeed()
    end
end

-- Noclip helper
local function SetNoclip(enabled)
    NoclipEnabled = enabled
end

-- Hitbox helper
local function UpdateHitboxes()
    if not HitboxEnabled then
        return
    end
    for _, player in ipairs(Players:GetPlayers()) do
        if player ~= LocalPlayer then
            local character = player.Character
            local hrp = character and character:FindFirstChild('HumanoidRootPart')
            if hrp then
                hrp.Size = Vector3.new(HitboxSize, HitboxSize, HitboxSize)
                hrp.CanCollide = false
                hrp.Transparency = math.clamp(hrp.Transparency, 0, 0.5)
            end
        end
    end
end

local function ResetHitboxes()
    for _, player in ipairs(Players:GetPlayers()) do
        if player ~= LocalPlayer then
            local character = player.Character
            local hrp = character and character:FindFirstChild('HumanoidRootPart')
            if hrp then
                hrp.Size = Vector3.new(2, 2, 1)
                hrp.CanCollide = true
            end
        end
    end
end

local OriginalFogStart = nil
local OriginalFogEnd = nil
local OriginalAmbient = nil

local NoclipConnection
local HitboxConnection

local function StartNoclip()
    if NoclipConnection then return end
    NoclipConnection = RunService.Stepped:Connect(function()
        if not NoclipEnabled then return end
        local character = GetCharacter()
        if character then
            for _, part in ipairs(character:GetDescendants()) do
                if part:IsA('BasePart') then
                    part.CanCollide = false
                end
            end
        end
    end)
end

local function StopNoclip()
    if NoclipConnection then
        NoclipConnection:Disconnect()
        NoclipConnection = nil
    end
end

local function StartHitboxLoop()
    if HitboxConnection then return end
    HitboxConnection = RunService.Heartbeat:Connect(function()
        if HitboxEnabled then
            UpdateHitboxes()
        end
    end)
end

local function StopHitboxLoop()
    if HitboxConnection then
        HitboxConnection:Disconnect()
        HitboxConnection = nil
    end
    ResetHitboxes()
end

local function ApplyFogSettings()
    local lighting = game:GetService('Lighting')
    if NoFogEnabled then
        OriginalFogStart = OriginalFogStart or lighting.FogStart
        OriginalFogEnd = OriginalFogEnd or lighting.FogEnd
        lighting.FogStart = 0
        lighting.FogEnd = 100000
    else
        if OriginalFogStart then lighting.FogStart = OriginalFogStart end
        if OriginalFogEnd then lighting.FogEnd = OriginalFogEnd end
    end
end

local function ApplyAmbient()
    local lighting = game:GetService('Lighting')
    if not OriginalAmbient then
        OriginalAmbient = { Ambient = lighting.Ambient, Outdoor = lighting.OutdoorAmbient }
    end
    if AmbientColor then
        lighting.Ambient = AmbientColor
        lighting.OutdoorAmbient = AmbientColor
    else
        if OriginalAmbient then
            lighting.Ambient = OriginalAmbient.Ambient
            lighting.OutdoorAmbient = OriginalAmbient.Outdoor
        end
    end
end
-- =============================================
-- CASH AURA SYSTEM
-- =============================================
local CashAuraConnection

local function CollectCash()
    local hrp = GetHRP()
    if not hrp then return end
    
    local myPosition = hrp.Position
    
    -- Da Hood stores dropped items in Workspace.Ignored.Drop
    local droppedItems = Workspace:FindFirstChild('Ignored') and Workspace.Ignored:FindFirstChild('Drop')
    if not droppedItems then return end
    
    for _, item in pairs(droppedItems:GetChildren()) do
        if not CashAuraEnabled then break end
        
        -- Cash in Da Hood is called "MoneyDrop"
        if item.Name == 'MoneyDrop' and item:IsA('BasePart') then
            local cashPosition = item.Position
            local distance = (myPosition - cashPosition).Magnitude
            
            -- Only pick up if within range
            if distance <= CashAuraRange then
                local clickDetector = item:FindFirstChildOfClass('ClickDetector')
                if clickDetector then
                    fireclickdetector(clickDetector)
                end
            end
        end
    end
end

local function StartCashAura()
    if CashAuraConnection then return end
    
    CashAuraConnection = task.spawn(function()
        while CashAuraEnabled do
            pcall(CollectCash)
            task.wait(0.1)
        end
    end)
end

local function StopCashAura()
    if CashAuraConnection then
        task.cancel(CashAuraConnection)
        CashAuraConnection = nil
    end
end

-- =============================================
-- CASH DROP SYSTEM
-- =============================================
local CashDropConnection

local function DropCash(amount)
    -- Da Hood: MainEvent with "DropMoney" and amount as STRING
    local ReplicatedStorage = game:GetService('ReplicatedStorage')
    local mainEvent = ReplicatedStorage:FindFirstChild('MainEvent')
    
    if mainEvent then
        pcall(function()
            mainEvent:FireServer('DropMoney', tostring(amount))
        end)
    end
end

local function StartCashDrop()
    if CashDropConnection then return end
    
    CashDropConnection = task.spawn(function()
        while CashDropEnabled do
            DropCash(15000)
            task.wait(1) -- Spam as fast as possible
        end
    end)
end

local function StopCashDrop()
    if CashDropConnection then
        task.cancel(CashDropConnection)
        CashDropConnection = nil
    end
end

-- =============================================
-- ESP SYSTEM (Optimized - No Lag)
-- =============================================
local function CreateESP(player)
    if player == LocalPlayer then return end
    if ESPObjects[player] then return end
    
    ESPObjects[player] = {}
    
    -- Create Billboard GUI for name
    local billboard = Instance.new('BillboardGui')
    billboard.Name = 'NameESP'
    billboard.AlwaysOnTop = true
    billboard.Size = UDim2.new(0, 200, 0, 50)
    billboard.StudsOffset = Vector3.new(0, 3, 0)
    billboard.ResetOnSpawn = false
    
    local nameLabel = Instance.new('TextLabel')
    nameLabel.Parent = billboard
    nameLabel.BackgroundTransparency = 1
    nameLabel.Size = UDim2.new(1, 0, 1, 0)
    nameLabel.Font = Enum.Font.GothamBold
    nameLabel.TextSize = ESPTextSize
    nameLabel.TextColor3 = ESPTextColor
    nameLabel.TextStrokeTransparency = 0
    nameLabel.TextStrokeColor3 = Color3.fromRGB(0, 0, 0)
    
    -- Create Highlight
    local highlight = Instance.new('Highlight')
    highlight.FillTransparency = 0.75
    highlight.FillColor = ESPColor
    highlight.OutlineTransparency = 0
    highlight.OutlineColor = ESPColor
    highlight.DepthMode = Enum.HighlightDepthMode.AlwaysOnTop
    
    ESPObjects[player] = {
        Billboard = billboard,
        NameLabel = nameLabel,
        Highlight = highlight
    }
    
    -- Update function
    local function UpdateESP()
        if not ESPEnabled then
            billboard.Enabled = false
            highlight.Enabled = false
            return
        end
        
        local character = player.Character
        local head = character and character:FindFirstChild('Head')
        local humanoid = character and character:FindFirstChildOfClass('Humanoid')
        
        if character and head and humanoid then
            billboard.Adornee = head
            billboard.Parent = head
            billboard.Enabled = ShowNames
            
            -- Update text settings
            nameLabel.TextSize = ESPTextSize
            nameLabel.TextColor3 = ESPTextColor
            
            highlight.Parent = character
            highlight.Enabled = true
            
            -- Update name with health
            local health = math.floor(humanoid.Health)
            local maxHealth = math.floor(humanoid.MaxHealth)
            nameLabel.Text = ShowNames and string.format('%s [%d/%d]', player.Name, health, maxHealth) or ''
            
            -- Color based on health or custom color
            if UseHealthColor then
                local healthPercent = humanoid.Health / humanoid.MaxHealth
                local color
                if healthPercent > 0.5 then
                    color = Color3.fromRGB(0, 255, 0)
                elseif healthPercent > 0.25 then
                    color = Color3.fromRGB(255, 255, 0)
                else
                    color = Color3.fromRGB(255, 0, 0)
                end
                highlight.OutlineColor = color
                highlight.FillColor = color
            else
                highlight.OutlineColor = ESPColor
                highlight.FillColor = ESPColor
            end
        else
            billboard.Enabled = false
            highlight.Enabled = false
        end
    end
    
    -- Connect to character changes
    player.CharacterAdded:Connect(function()
        task.wait(0.5)
        UpdateESP()
    end)
    
    -- Store update function
    ESPObjects[player].Update = UpdateESP
    UpdateESP()
end

local function RemoveESP(player)
    if ESPObjects[player] then
        if ESPObjects[player].Billboard then
            ESPObjects[player].Billboard:Destroy()
        end
        if ESPObjects[player].Highlight then
            ESPObjects[player].Highlight:Destroy()
        end
        ESPObjects[player] = nil
    end
end

local function RefreshAllESP()
    for player, objects in pairs(ESPObjects) do
        if objects.Update then
            pcall(objects.Update)
        end
    end
end

-- ESP Update Loop
local ESPUpdateConnection
local function StartESPLoop()
    if ESPUpdateConnection then return end
    ESPUpdateConnection = RunService.Heartbeat:Connect(function()
        if ESPEnabled then
            RefreshAllESP()
        end
    end)
end

-- Initialize ESP for existing players
local function InitializeESP()
    for _, player in pairs(Players:GetPlayers()) do
        CreateESP(player)
    end
end

-- Player connections
Players.PlayerAdded:Connect(function(player)
    CreateESP(player)
end)

Players.PlayerRemoving:Connect(function(player)
    RemoveESP(player)
end)

LocalPlayer.CharacterAdded:Connect(function(char)
    HookWalkSpeed(char)
end)

if LocalPlayer.Character then
    HookWalkSpeed(LocalPlayer.Character)
end

-- =============================================
-- MONEY ESP SYSTEM
-- =============================================
local MoneyESPConnection

local function ClearMoneyESP()
    for part, highlight in pairs(MoneyESPObjects) do
        if highlight then
            highlight:Destroy()
        end
    end
    MoneyESPObjects = {}
end

local function UpdateMoneyESP()
    if not MoneyESPEnabled then
        ClearMoneyESP()
        return
    end
    
    local droppedItems = Workspace:FindFirstChild('Ignored') and Workspace.Ignored:FindFirstChild('Drop')
    if not droppedItems then return end
    
    -- Remove highlights for money that no longer exists
    for part, highlight in pairs(MoneyESPObjects) do
        if not part or not part.Parent then
            if highlight then highlight:Destroy() end
            MoneyESPObjects[part] = nil
        end
    end
    
    -- Add highlights for new money
    for _, item in pairs(droppedItems:GetChildren()) do
        if item.Name == 'MoneyDrop' and item:IsA('BasePart') then
            if not MoneyESPObjects[item] then
                local highlight = Instance.new('Highlight')
                highlight.FillColor = Color3.fromRGB(0, 255, 0)
                highlight.FillTransparency = 0.5
                highlight.OutlineColor = Color3.fromRGB(0, 255, 0)
                highlight.OutlineTransparency = 0
                highlight.DepthMode = Enum.HighlightDepthMode.AlwaysOnTop
                highlight.Parent = item
                MoneyESPObjects[item] = highlight
            end
        end
    end
end

local function StartMoneyESP()
    if MoneyESPConnection then return end
    MoneyESPConnection = RunService.Heartbeat:Connect(function()
        if MoneyESPEnabled then
            pcall(UpdateMoneyESP)
        end
    end)
end

local function StopMoneyESP()
    if MoneyESPConnection then
        MoneyESPConnection:Disconnect()
        MoneyESPConnection = nil
    end
    ClearMoneyESP()
end

-- =============================================
-- CFRAME SPEED SYSTEM (Undetectable)
-- =============================================
local CFrameSpeedConnection
local MovingDirection = Vector3.new(0, 0, 0)

local function GetMovementDirection()
    local hrp = GetHRP()
    local humanoid = GetCharacter() and GetCharacter():FindFirstChildOfClass('Humanoid')
    if not hrp or not humanoid then return Vector3.new(0, 0, 0) end
    
    local moveDirection = humanoid.MoveDirection
    if moveDirection.Magnitude > 0 then
        return moveDirection.Unit
    end
    return Vector3.new(0, 0, 0)
end

local function StartCFrameSpeed()
    if CFrameSpeedConnection then return end
    
    CFrameSpeedConnection = RunService.Heartbeat:Connect(function(deltaTime)
        if not CFrameSpeedEnabled then return end
        
        local hrp = GetHRP()
        local humanoid = GetCharacter() and GetCharacter():FindFirstChildOfClass('Humanoid')
        if not hrp or not humanoid then return end
        
        local moveDir = GetMovementDirection()
        if moveDir.Magnitude > 0 then
            -- Apply extra movement based on CFrame (looks natural)
            local extraSpeed = (CFrameSpeedValue - 1) * humanoid.WalkSpeed * deltaTime
            local newPos = hrp.Position + (moveDir * extraSpeed)
            hrp.CFrame = CFrame.new(newPos) * CFrame.Angles(hrp.CFrame:ToEulerAnglesXYZ())
        end
    end)
end

local function StopCFrameSpeed()
    if CFrameSpeedConnection then
        CFrameSpeedConnection:Disconnect()
        CFrameSpeedConnection = nil
    end
end

-- =============================================
-- CFRAME FLY SYSTEM (With Animations & Collision)
-- =============================================
local CFrameFlyConnection
local IsFlying = false
local FlyUpHeld = false
local FlyDownHeld = false

local function StartCFrameFly()
    if CFrameFlyConnection then return end
    IsFlying = true
    
    CFrameFlyConnection = RunService.Heartbeat:Connect(function(deltaTime)
        if not CFrameFlyEnabled then return end
        
        local hrp = GetHRP()
        local humanoid = GetCharacter() and GetCharacter():FindFirstChildOfClass('Humanoid')
        if not hrp or not humanoid then return end
        
        -- Get movement from WASD (uses character direction, not camera)
        -- This lets you look around freely while moving
        local moveDirection = humanoid.MoveDirection
        local verticalDirection = 0
        
        -- Space = up, Ctrl = down
        if FlyUpHeld then
            verticalDirection = 1
        end
        if FlyDownHeld then
            verticalDirection = -1
        end
        
        -- Calculate horizontal movement (from WASD via MoveDirection)
        local horizontalMove = Vector3.new(moveDirection.X, 0, moveDirection.Z)
        
        -- Calculate vertical movement
        local verticalMove = Vector3.new(0, verticalDirection, 0)
        
        -- Combine movements
        local totalDirection = horizontalMove + verticalMove
        
        if totalDirection.Magnitude > 0 then
            -- Move character using CFrame
            local moveAmount = totalDirection.Unit * CFrameFlySpeed * deltaTime
            local newPosition = hrp.Position + moveAmount
            
            -- Keep current rotation (character faces where you walk)
            local currentRotation = hrp.CFrame - hrp.CFrame.Position
            hrp.CFrame = CFrame.new(newPosition) * currentRotation
        end
        
        -- Anti-gravity: Cancel out falling by countering gravity each frame
        -- This keeps you floating without noclip and keeps animations
        local antiGravity = Vector3.new(0, Workspace.Gravity * deltaTime, 0)
        hrp.Velocity = Vector3.new(hrp.Velocity.X * 0.9, 0, hrp.Velocity.Z * 0.9) + antiGravity * 0
        
        -- Keep vertical velocity at 0 to float (but horizontal stays for animations)
        hrp.Velocity = Vector3.new(hrp.Velocity.X, 0, hrp.Velocity.Z)
    end)
end

local function StopCFrameFly()
    if CFrameFlyConnection then
        CFrameFlyConnection:Disconnect()
        CFrameFlyConnection = nil
    end
    IsFlying = false
    
    -- Gentle landing - reduce fall speed
    local hrp = GetHRP()
    if hrp then
        hrp.Velocity = Vector3.new(hrp.Velocity.X, -10, hrp.Velocity.Z)
    end
end

-- Fly vertical controls (Space/Ctrl)
UserInputService.InputBegan:Connect(function(input, gameProcessed)
    if gameProcessed then return end
    if IsTyping() then return end
    if not CFrameFlyEnabled then return end
    
    if input.KeyCode == Enum.KeyCode.Space then 
        FlyUpHeld = true 
    end
    if input.KeyCode == Enum.KeyCode.LeftControl then 
        FlyDownHeld = true 
    end
end)

UserInputService.InputEnded:Connect(function(input)
    if input.KeyCode == Enum.KeyCode.Space then 
        FlyUpHeld = false 
    end
    if input.KeyCode == Enum.KeyCode.LeftControl then 
        FlyDownHeld = false 
    end
end)

-- =============================================
-- MAIN TAB UI
-- =============================================

-- =============================================
-- AIMBOT SYSTEM
-- =============================================
local Camera = Workspace.CurrentCamera

local function GetMousePosition()
    local rawPos = UserInputService:GetMouseLocation()
    return Vector2.new(rawPos.X, rawPos.Y)
end

local function WorldToScreen(position)
    local screenPos, onScreen = Camera:WorldToScreenPoint(position)
    return Vector2.new(screenPos.X, screenPos.Y), onScreen, screenPos.Z
end

local function GetDistanceFromMouse(position)
    local screenPos, onScreen = WorldToScreen(position)
    if not onScreen then return math.huge end
    local mousePos = GetMousePosition()
    return (screenPos - mousePos).Magnitude
end

local function IsPlayerValid(player)
    if player == LocalPlayer then return false end
    local character = player.Character
    if not character then return false end
    local humanoid = character:FindFirstChildOfClass("Humanoid")
    if not humanoid or humanoid.Health <= 0 then return false end
    local targetPart = character:FindFirstChild(AimbotTarget)
    if not targetPart then return false end
    if AimbotKOCheck or AimbotDownedGrabbedCheck then
        local bodyEffects = character:FindFirstChild("BodyEffects")
        local ko = bodyEffects and (bodyEffects:FindFirstChild("K.O") or bodyEffects:FindFirstChild("KO"))
        if AimbotKOCheck and ko and ko.Value then return false end
        if AimbotDownedGrabbedCheck then
            local grabbed = character:FindFirstChild("GRABBING_CONSTRAINT")
            if grabbed then return false end
        end
    end
    if AimbotForceFieldCheck and character:FindFirstChildOfClass("ForceField") then
        return false
    end
    if AimbotCrewCheck then
        local dataFolder = player:FindFirstChild("DataFolder")
        local info = dataFolder and dataFolder:FindFirstChild("Information")
        local crew = info and info:FindFirstChild("Crew")
        local localData = LocalPlayer:FindFirstChild("DataFolder")
        local localInfo = localData and localData:FindFirstChild("Information")
        local localCrew = localInfo and localInfo:FindFirstChild("Crew")
        if crew and localCrew and crew.Value ~= "" and localCrew.Value ~= "" and crew.Value == localCrew.Value then
            return false
        end
    end
    return true
end

local function HasLineOfSight(player)
    if not AimbotLOSCheck then return true end
    local character = player.Character
    if not character then return false end
    local targetPart = character:FindFirstChild(AimbotTarget)
    if not targetPart then return false end

    local ignoreList = {}
    for _, desc in ipairs(character:GetDescendants()) do
        table.insert(ignoreList, desc)
    end

    local parts = Camera:GetPartsObscuringTarget({ targetPart.Position }, ignoreList)
    return #parts == 0
end

local function IsTargetInFOV(player)
    local character = player.Character
    if not character then return false end
    local targetPart = character:FindFirstChild(AimbotTarget)
    if not targetPart then return false end
    return GetDistanceFromMouse(targetPart.Position) <= AimbotFOV
end

local function ShouldKeepTarget(player)
    return IsPlayerValid(player) and IsTargetInFOV(player) and HasLineOfSight(player)
end

local function MoveMouse(deltaX, deltaY)
    if mousemoverel then
        mousemoverel(deltaX, deltaY)
        return true
    end

    if mousemoveabs then
        local current = GetMousePosition()
        mousemoveabs(current.X + deltaX, current.Y + deltaY)
        return true
    end

    return false
end

local function GetClosestPlayer()
    local closestPlayer = nil
    local closestDistance = AimbotFOV
    
    for _, player in pairs(Players:GetPlayers()) do
        if IsPlayerValid(player) then
            local character = player.Character
            local targetPart = character:FindFirstChild(AimbotTarget)
            if targetPart then
                local distance = GetDistanceFromMouse(targetPart.Position)
                if distance < closestDistance then
                    closestDistance = distance
                    closestPlayer = player
                end
            end
        end
    end
    
    return closestPlayer
end

local ActiveTween
local function ChancePassed()
    if AimbotHitChance >= 100 then return true end
    return math.random(1, 100) <= AimbotHitChance
end

local function AdjustedTargetPosition(targetPart)
    local predicted = targetPart.Position
    if AimbotPrediction > 0 then
        predicted = predicted + (targetPart.Velocity * AimbotPrediction)
    end
    return predicted
end

local function SetCameraSensitivity(disable)
    if AimbotLockMode ~= "Camera" then return end
    if disable then
        OriginalMouseDelta = UserInputService.MouseDeltaSensitivity
        UserInputService.MouseDeltaSensitivity = 0
    else
        UserInputService.MouseDeltaSensitivity = OriginalMouseDelta
    end
end

local function AimAt(player)
    if not player then return end
    local character = player.Character
    if not character then return end
    local targetPart = character:FindFirstChild(AimbotTarget)
    if not targetPart then return end
    
    local targetPos = AdjustedTargetPosition(targetPart)
    if AimbotDynamicPrediction then
        local distance = (Camera.CFrame.Position - targetPos).Magnitude
        local velMag = targetPart.Velocity.Magnitude
        local scale = math.clamp(velMag / math.max(distance, 1), 0, 2)
        targetPos = targetPart.Position + (targetPart.Velocity * AimbotPrediction * scale)
    end
    local screenPos, onScreen = WorldToScreen(targetPos)
    
    if onScreen and ChancePassed() then
        if AimbotLockMode == "Camera" then
            if ActiveTween then
                ActiveTween:Cancel()
                ActiveTween = nil
            end

            if AimbotCameraTween > 0 then
                ActiveTween = TweenService:Create(Camera, TweenInfo.new(AimbotCameraTween, Enum.EasingStyle.Sine, Enum.EasingDirection.Out), {
                    CFrame = CFrame.new(Camera.CFrame.Position, targetPos)
                })
                ActiveTween:Play()
            else
                Camera.CFrame = CFrame.new(Camera.CFrame.Position, targetPos)
            end
            SetCameraSensitivity(true)
        else
            local mousePos = GetMousePosition()
            local delta = screenPos - mousePos
            
            -- Only move if delta is significant (prevents jitter)
            if delta.Magnitude > 1 then
                -- Smoothing: lower value = slower/smoother, higher = faster/snappier
                local moveX = delta.X * AimbotSmoothing
                local moveY = delta.Y * AimbotSmoothing
                
                -- Clamp movement to prevent overshooting
                moveX = math.clamp(moveX, -50, 50)
                moveY = math.clamp(moveY, -50, 50)

                if not MoveMouse(moveX, moveY) then
                    -- Fallback: snap camera toward target if mouse move APIs are missing
                    Camera.CFrame = CFrame.new(Camera.CFrame.Position, targetPos)
                end
            end
        end
    end
end

-- Clear all tracers
local function ClearTracers()
    for _, line in pairs(TracerLines) do
        if line then
            pcall(function() line:Remove() end)
        end
    end
    TracerLines = {}
end

-- Update tracers (reuse existing lines instead of recreating)
local function UpdateTracers()
    if not TracersEnabled then 
        ClearTracers()
        return 
    end
    
    local screenSize = Camera.ViewportSize
    local bottomCenter = Vector2.new(screenSize.X / 2, screenSize.Y)
    
    local validPlayers = {}
    for _, player in pairs(Players:GetPlayers()) do
        if player ~= LocalPlayer then
            local character = player.Character
            local humanoid = character and character:FindFirstChildOfClass("Humanoid")
            local hrp = character and character:FindFirstChild("HumanoidRootPart")
            
            if character and humanoid and humanoid.Health > 0 and hrp then
                local screenPos, onScreen, depth = WorldToScreen(hrp.Position)
                if onScreen and depth > 0 then
                    table.insert(validPlayers, {pos = screenPos, player = player})
                end
            end
        end
    end
    
    -- Remove extra tracer lines
    while #TracerLines > #validPlayers do
        local line = table.remove(TracerLines)
        if line then pcall(function() line:Remove() end) end
    end
    
    -- Update or create tracer lines
    for i, data in ipairs(validPlayers) do
        local line = TracerLines[i]
        if not line then
            line = Drawing.new("Line")
            TracerLines[i] = line
        end
        
        line.From = bottomCenter
        line.To = data.pos
        line.Color = TracerColor
        line.Thickness = 1
        line.Transparency = 1 -- Drawing API: 1 = fully visible
        line.Visible = true
    end
end

-- Aimbot update loop
local AimbotConnection
local AimbotHeld = false

local function StartAimbot()
    if AimbotConnection then return end
    
    AimbotConnection = RunService.RenderStepped:Connect(function()
        local ok, err = pcall(function()
            -- Update camera reference (can change)
            Camera = Workspace.CurrentCamera
            
            -- Update FOV Circle position and settings
            if FOVCircleEnabled and AimbotEnabled then
                local mousePos = GetMousePosition()
                FOVCircle.Position = mousePos
                FOVCircleOutline.Position = mousePos
                FOVCircle.Radius = AimbotFOV
                FOVCircleOutline.Radius = AimbotFOV + 1
                FOVCircleOutline.Transparency = FOVCircleTransparency
                FOVCircleOutline.Visible = true
                FOVCircleOutline.Color = Color3.fromRGB(0, 0, 0)
                FOVCircle.Color = FOVCircleColor
                FOVCircle.Transparency = FOVCircleTransparency
                FOVCircle.Filled = AimbotFOVFilled
                FOVCircle.Visible = true
            else
                FOVCircle.Visible = false
                FOVCircleOutline.Visible = false
            end
            
            -- Update tracers (runs even when aimbot is off, as long as tracers are enabled)
            UpdateTracers()
            
            -- Aimbot logic
        if AimbotEnabled and AimbotHeld then
            if AimbotGunOnly then
                local character = GetCharacter()
                local holdingTool = character and character:FindFirstChildOfClass("Tool")
                if not holdingTool then
                    return
                end
            end
                if not (CurrentTarget and ShouldKeepTarget(CurrentTarget)) then
                    CurrentTarget = GetClosestPlayer()
                end

                if CurrentTarget then
                    FOVCircle.Color = AimbotLockedColor
                    AimAt(CurrentTarget)
                else
                    FOVCircle.Color = FOVCircleColor
                end
            else
                CurrentTarget = nil
                FOVCircle.Color = FOVCircleColor
                SetCameraSensitivity(false)
            end
        end)

        if not ok then
            warn('[Aimbot Loop Error]', err)
        end
    end)
end

local function StopAimbot()
    if AimbotConnection then
        AimbotConnection:Disconnect()
        AimbotConnection = nil
    end
    if ActiveTween then
        ActiveTween:Cancel()
        ActiveTween = nil
    end
    SetCameraSensitivity(false)
    FOVCircle.Visible = false
    FOVCircleOutline.Visible = false
    ClearTracers()
end

local function UpdateAimbotLoop()
    if AimbotEnabled or TracersEnabled then
        StartAimbot()
    else
        StopAimbot()
    end
end

-- Aimbot input handling (right click to aim)
UserInputService.InputBegan:Connect(function(input, gameProcessed)
    if gameProcessed then return end
    if IsTyping() then return end
    
    local isMouse = input.UserInputType == Enum.UserInputType.MouseButton2 or input.UserInputType == Enum.UserInputType.MouseButton1
    local mouseMatch = (AimbotKeybind == "MouseButton2" and input.UserInputType == Enum.UserInputType.MouseButton2) or (AimbotKeybind == "MouseButton1" and input.UserInputType == Enum.UserInputType.MouseButton1)
    local keyMatch = (input.KeyCode ~= Enum.KeyCode.Unknown and AimbotKeybind == input.KeyCode.Name)

    if mouseMatch or keyMatch then
        AimbotHeld = true
        CurrentTarget = GetClosestPlayer()
    end
end)

UserInputService.InputEnded:Connect(function(input)
    local mouseMatch = (AimbotKeybind == "MouseButton2" and input.UserInputType == Enum.UserInputType.MouseButton2) or (AimbotKeybind == "MouseButton1" and input.UserInputType == Enum.UserInputType.MouseButton1)
    local keyMatch = (input.KeyCode ~= Enum.KeyCode.Unknown and AimbotKeybind == input.KeyCode.Name)

    if mouseMatch or keyMatch then
        AimbotHeld = false
        CurrentTarget = nil
        if ActiveTween then
            ActiveTween:Cancel()
            ActiveTween = nil
        end
        SetCameraSensitivity(false)
    end
end)

-- =============================================
-- AIMBOT TAB UI
-- =============================================
local AimbotSection = Tabs.Aimbot:AddLeftGroupbox('Aimbot')

AimbotSection:AddToggle('AimbotEnabled', {
    Text = 'Enable Aimbot',
    Default = false,
    Tooltip = 'Right-click to aim at players',
    Callback = function(Value)
        AimbotEnabled = Value
        UpdateAimbotLoop()
        if Value then
            Library:Notify('Aimbot enabled! Right-click to aim', 2)
        else
            Library:Notify('Aimbot disabled', 2)
        end
    end
})

AimbotSection:AddDropdown('AimbotTargetPart', {
    Values = { 'Head', 'HumanoidRootPart' },
    Default = 1,
    Multi = false,
    Text = 'Target Part',
    Tooltip = 'Which body part to aim at',
    Callback = function(Value)
        AimbotTarget = Value
    end
})

AimbotSection:AddSlider('AimbotSmoothing', {
    Text = 'Smoothing',
    Default = 0.3,
    Min = 0.05,
    Max = 1,
    Rounding = 2,
    Compact = false,
    Tooltip = 'Lower = smoother/slower, Higher = snappier',
    Callback = function(Value)
        AimbotSmoothing = Value
    end
})

AimbotSection:AddSlider('AimbotCameraTween', {
    Text = 'Camera Lock Tween (s)',
    Default = 0,
    Min = 0,
    Max = 1,
    Rounding = 2,
    Compact = false,
    Tooltip = '0 = instant snap, higher = slower camera tween (camera mode only)',
    Callback = function(Value)
        AimbotCameraTween = Value
    end
})

AimbotSection:AddSlider('AimbotPrediction', {
    Text = 'Aim Prediction',
    Default = 0,
    Min = 0,
    Max = 0.3,
    Rounding = 3,
    Compact = false,
    Tooltip = 'Multiply target velocity by this amount for lead',
    Callback = function(Value)
        AimbotPrediction = Value
    end
})

AimbotSection:AddSlider('AimbotHitChance', {
    Text = 'Hit Chance (%)',
    Default = 100,
    Min = 1,
    Max = 100,
    Rounding = 0,
    Compact = false,
    Tooltip = 'Randomly skip aim updates to look legit',
    Callback = function(Value)
        AimbotHitChance = Value
    end
})

AimbotSection:AddToggle('AimbotLOSCheck', {
    Text = 'Line of Sight Check',
    Default = false,
    Tooltip = 'Only aim if target is not behind walls',
    Callback = function(Value)
        AimbotLOSCheck = Value
    end
})

AimbotSection:AddToggle('AimbotKOCheck', {
    Text = 'Ignore KO\'d Targets',
    Default = false,
    Tooltip = 'Skip players that are downed',
    Callback = function(Value)
        AimbotKOCheck = Value
    end
})

AimbotSection:AddToggle('AimbotCrewCheck', {
    Text = 'Ignore Same Crew',
    Default = false,
    Tooltip = 'Skip players in your crew',
    Callback = function(Value)
        AimbotCrewCheck = Value
    end
})

AimbotSection:AddToggle('AimbotForceFieldCheck', {
    Text = 'Ignore ForceField',
    Default = false,
    Tooltip = 'Skip players with forcefield',
    Callback = function(Value)
        AimbotForceFieldCheck = Value
    end
})

AimbotSection:AddToggle('AimbotDownedGrabbedCheck', {
    Text = 'Ignore Downed/Grabbed',
    Default = false,
    Tooltip = 'Skip players that are downed or being grabbed',
    Callback = function(Value)
        AimbotDownedGrabbedCheck = Value
    end
})

AimbotSection:AddToggle('AimbotGunOnly', {
    Text = 'Gun Only',
    Default = false,
    Tooltip = 'Only aim when holding a tool/weapon',
    Callback = function(Value)
        AimbotGunOnly = Value
    end
})

AimbotSection:AddToggle('AimbotDynamicPrediction', {
    Text = 'Dynamic Prediction',
    Default = false,
    Tooltip = 'Scale prediction by target velocity/distance',
    Callback = function(Value)
        AimbotDynamicPrediction = Value
    end
})

AimbotSection:AddDropdown('AimbotKeybind', {
    Values = { 'MouseButton2', 'MouseButton1', 'Q', 'E', 'LeftShift', 'LeftAlt' },
    Default = 1,
    Multi = false,
    Text = 'Aim Key',
    Tooltip = 'Key/button to hold for aimbot',
    Callback = function(Value)
        AimbotKeybind = Value
    end
})

-- FOV Section
local FOVSection = Tabs.Aimbot:AddLeftGroupbox('FOV Circle')

FOVSection:AddToggle('FOVCircleEnabled', {
    Text = 'Show FOV Circle',
    Default = true,
    Tooltip = 'Shows the FOV circle on screen',
    Callback = function(Value)
        FOVCircleEnabled = Value
        FOVCircle.Visible = Value and AimbotEnabled
    end
})

FOVSection:AddToggle('FOVFilled', {
    Text = 'Filled FOV',
    Default = false,
    Tooltip = 'Fill the FOV circle',
    Callback = function(Value)
        AimbotFOVFilled = Value
        FOVCircle.Filled = Value
    end
})

FOVSection:AddSlider('FOVSize', {
    Text = 'FOV Size',
    Default = 100,
    Min = 20,
    Max = 500,
    Rounding = 0,
    Compact = false,
    Tooltip = 'Size of the aimbot FOV',
    Callback = function(Value)
        AimbotFOV = Value
        FOVCircle.Radius = Value
    end
})

FOVSection:AddSlider('FOVTransparency', {
    Text = 'FOV Opacity',
    Default = 0.5,
    Min = 0,
    Max = 1,
    Rounding = 2,
    Compact = false,
    Tooltip = '0 = invisible, 1 = fully visible',
    Callback = function(Value)
        FOVCircleTransparency = Value
        FOVCircle.Transparency = Value
    end
})

FOVSection:AddLabel('FOV Color'):AddColorPicker('FOVColor', {
    Default = Color3.fromRGB(255, 255, 255),
    Title = 'FOV Circle Color',
    Transparency = 0,
    Callback = function(Value)
        FOVCircleColor = Value
        FOVCircle.Color = Value
    end
})

FOVSection:AddLabel('Locked Color'):AddColorPicker('LockedColor', {
    Default = Color3.fromRGB(255, 80, 80),
    Title = 'Locked Target Color',
    Transparency = 0,
    Callback = function(Value)
        AimbotLockedColor = Value
    end
})

-- Tracers Section
local TracerSection = Tabs.Aimbot:AddLeftGroupbox('Tracers')

TracerSection:AddToggle('TracersEnabled', {
    Text = 'Enable Tracers',
    Default = false,
    Tooltip = 'Shows lines from screen bottom to players',
    Callback = function(Value)
        TracersEnabled = Value
        UpdateAimbotLoop()
        if not Value then
            ClearTracers()
        end
    end
})

TracerSection:AddLabel('Tracer Color'):AddColorPicker('TracerColor', {
    Default = Color3.fromRGB(255, 0, 0),
    Title = 'Tracer Color',
    Transparency = 0,
    Callback = function(Value)
        TracerColor = Value
    end
})

-- =============================================
-- MAIN TAB UI
-- =============================================
local CashSection = Tabs.Main:AddLeftGroupbox('Cash')

CashSection:AddToggle('CashAura', {
    Text = 'Cash Aura',
    Default = false,
    Tooltip = 'Teleports to nearby cash to collect it',
    Callback = function(Value)
        CashAuraEnabled = Value
        if Value then
            StartCashAura()
        else
            StopCashAura()
        end
    end
})

CashSection:AddSlider('CashRange', {
    Text = 'Aura Range',
    Default = 25,
    Min = 10,
    Max = 100,
    Rounding = 0,
    Compact = false,
    Callback = function(Value)
        CashAuraRange = Value
    end
})

CashSection:AddToggle('CashDrop', {
    Text = 'Auto Drop $15,000',
    Default = false,
    Tooltip = 'Spams dropping $15,000',
    Callback = function(Value)
        CashDropEnabled = Value
        if Value then
            StartCashDrop()
            Library:Notify('Spamming cash drop!', 2)
        else
            StopCashDrop()
            Library:Notify('Stopped cash drop', 2)
        end
    end
})

-- =============================================
-- MOVEMENT SECTION
-- =============================================
local MovementSection = Tabs.Main:AddLeftGroupbox('Movement')

MovementSection:AddToggle('CFrameSpeed', {
    Text = 'CFrame Speed',
    Default = false,
    Tooltip = 'Undetectable speed boost using CFrame',
    Callback = function(Value)
        CFrameSpeedEnabled = Value
        if Value then
            StartCFrameSpeed()
            Library:Notify('CFrame Speed enabled!', 2)
        else
            StopCFrameSpeed()
            Library:Notify('CFrame Speed disabled', 2)
        end
    end
}):AddKeyPicker('CFrameSpeedKey', {
    Default = 'V',
    SyncToggleState = true,
    Mode = 'Toggle',
    Text = 'CFrame Speed',
    NoUI = false
})

MovementSection:AddSlider('SpeedMultiplier', {
    Text = 'Speed Multiplier',
    Default = 1.5,
    Min = 1.1,
    Max = 5,
    Rounding = 1,
    Compact = false,
    Tooltip = '1.5-2x is safest, higher may get detected',
    Callback = function(Value)
        CFrameSpeedValue = Value
    end
})

MovementSection:AddDivider()

MovementSection:AddToggle('CFrameFly', {
    Text = 'CFrame Fly',
    Default = false,
    Tooltip = 'Fly using WASD + Space/Ctrl',
    Callback = function(Value)
        CFrameFlyEnabled = Value
        if Value then
            StartCFrameFly()
            Library:Notify('Flying! WASD to move, Space/Ctrl for up/down', 3)
        else
            StopCFrameFly()
            Library:Notify('Fly disabled', 2)
        end
    end
}):AddKeyPicker('FlyKey', {
    Default = 'F',
    SyncToggleState = true,
    Mode = 'Toggle',
    Text = 'Fly',
    NoUI = false
})

MovementSection:AddSlider('FlySpeed', {
    Text = 'Fly Speed',
    Default = 50,
    Min = 10,
    Max = 200,
    Rounding = 0,
    Compact = false,
    Tooltip = 'How fast you fly',
    Callback = function(Value)
        CFrameFlySpeed = Value
    end
})

MovementSection:AddDivider()

MovementSection:AddToggle('WalkSpeedToggle', {
    Text = 'Anti-Slow WalkSpeed',
    Default = false,
    Tooltip = 'Reapplies custom WalkSpeed to counter slow effects',
    Callback = function(Value)
        WalkSpeedEnabled = Value
        ApplyWalkSpeed()
        if not Value then
            WalkSpeedValue = math.max(WalkSpeedValue, 16)
            ApplyWalkSpeed()
        end
    end
}):AddKeyPicker('WalkSpeedKey', {
    Default = 'X',
    SyncToggleState = true,
    Mode = 'Toggle',
    Text = 'WalkSpeed Key',
    NoUI = false
})

MovementSection:AddSlider('WalkSpeedValue', {
    Text = 'WalkSpeed',
    Default = 16,
    Min = 16,
    Max = 500,
    Rounding = 0,
    Compact = false,
    Tooltip = 'Custom WalkSpeed when anti-slow is enabled',
    Callback = function(Value)
        WalkSpeedValue = Value
        if WalkSpeedEnabled then
            ApplyWalkSpeed()
        end
    end
})

-- =============================================
-- VISUALS TAB UI
-- =============================================
local ESPSection = Tabs.Visuals:AddLeftGroupbox('ESP')

ESPSection:AddToggle('ESPToggle', {
    Text = 'Enable ESP',
    Default = false,
    Tooltip = 'Shows players through walls',
    Callback = function(Value)
        ESPEnabled = Value
        if Value then
            InitializeESP()
            StartESPLoop()
        end
        RefreshAllESP()
    end
})

ESPSection:AddToggle('ShowNames', {
    Text = 'Show Names',
    Default = true,
    Tooltip = 'Shows player names above heads',
    Callback = function(Value)
        ShowNames = Value
    end
})

ESPSection:AddToggle('UseHealthColor', {
    Text = 'Health-Based Color',
    Default = true,
    Tooltip = 'Changes ESP color based on player health',
    Callback = function(Value)
        UseHealthColor = Value
    end
})

ESPSection:AddLabel('ESP Color'):AddColorPicker('ESPColor', {
    Default = Color3.fromRGB(255, 0, 0),
    Title = 'ESP Color',
    Transparency = 0,
    Callback = function(Value)
        ESPColor = Value
    end
})

ESPSection:AddLabel('Text Color'):AddColorPicker('TextColor', {
    Default = Color3.fromRGB(255, 255, 255),
    Title = 'Text Color',
    Transparency = 0,
    Callback = function(Value)
        ESPTextColor = Value
    end
})

ESPSection:AddSlider('TextSize', {
    Text = 'Text Size',
    Default = 14,
    Min = 8,
    Max = 24,
    Rounding = 0,
    Compact = false,
    Callback = function(Value)
        ESPTextSize = Value
    end
})

ESPSection:AddToggle('MoneyESP', {
    Text = 'Money ESP',
    Default = false,
    Tooltip = 'Highlights dropped cash',
    Callback = function(Value)
        MoneyESPEnabled = Value
        if Value then
            StartMoneyESP()
        else
            StopMoneyESP()
        end
    end
})

ESPSection:AddToggle('HitboxExpander', {
    Text = 'Hitbox Expander',
    Default = false,
    Tooltip = 'Enlarge enemy HumanoidRootPart hitboxes (visual/client-side)',
    Callback = function(Value)
        HitboxEnabled = Value
        if Value then
            StartHitboxLoop()
        else
            StopHitboxLoop()
        end
    end
})

ESPSection:AddSlider('HitboxSize', {
    Text = 'Hitbox Size',
    Default = 5,
    Min = 2,
    Max = 20,
    Rounding = 0,
    Compact = false,
    Tooltip = 'Size applied to enemy HRP when expander is on',
    Callback = function(Value)
        HitboxSize = Value
    end
})

ESPSection:AddToggle('NoclipToggle', {
    Text = 'Noclip (local)',
    Default = false,
    Tooltip = 'Disable collisions for your character',
    Callback = function(Value)
        SetNoclip(Value)
        if Value then
            StartNoclip()
        else
            StopNoclip()
        end
    end
})

ESPSection:AddToggle('NoFogToggle', {
    Text = 'No Fog',
    Default = false,
    Tooltip = 'Remove fog from the map',
    Callback = function(Value)
        NoFogEnabled = Value
        ApplyFogSettings()
    end
})

ESPSection:AddLabel('Ambient Color'):AddColorPicker('AmbientColor', {
    Default = Color3.fromRGB(127, 127, 127),
    Title = 'Ambient Color',
    Transparency = 0,
    Callback = function(Value)
        AmbientColor = Value
        ApplyAmbient()
    end
})

-- =============================================
-- SETTINGS TAB UI
-- =============================================
local MenuSection = Tabs.Settings:AddLeftGroupbox('Menu')

MenuSection:AddButton({
    Text = 'Unload Script',
    Func = function()
        -- Cleanup
        CashAuraEnabled = false
        CashDropEnabled = false
        MoneyESPEnabled = false
        CFrameSpeedEnabled = false
        CFrameFlyEnabled = false
        AimbotEnabled = false
        TracersEnabled = false
        WalkSpeedEnabled = false
        HitboxEnabled = false
        NoclipEnabled = false
        NoFogEnabled = false
        AmbientColor = nil
        StopCashAura()
        StopCashDrop()
        StopMoneyESP()
        StopCFrameSpeed()
        StopCFrameFly()
        StopAimbot()
        StopHitboxLoop()
        StopNoclip()
        ApplyFogSettings()
        ApplyAmbient()
        FOVCircle:Remove()
        ClearTracers()
        if ESPUpdateConnection then
            ESPUpdateConnection:Disconnect()
        end
        for player, _ in pairs(ESPObjects) do
            RemoveESP(player)
        end
        Library:Unload()
    end,
    DoubleClick = true,
    Tooltip = 'Double click to unload'
})

MenuSection:AddLabel('Menu Keybind'):AddKeyPicker('MenuKeybind', {
    Default = 'End',
    NoUI = true,
    Text = 'Menu Keybind'
})

Options = Library.Options or Options -- resync in case the library rebuilt this table
local menuKeyOption = Options and (Options.MenuKeybind or Options.MenuKeyBind)
if menuKeyOption then
    Library.ToggleKeybind = menuKeyOption
else
    Library.ToggleKeybind = Enum.KeyCode.End
end

-- Disable keybinds when typing in chat/textboxes
-- Block keybinds when focused on textbox (chat)
if typeof(Library.SetKeybindState) == "function" then
    UserInputService.TextBoxFocused:Connect(function()
        Library:SetKeybindState(false)
    end)
    
    UserInputService.TextBoxFocusReleased:Connect(function()
        Library:SetKeybindState(true)
    end)
end

-- Theme and Save Manager Setup
ThemeManager:SetLibrary(Library)
SaveManager:SetLibrary(Library)

SaveManager:IgnoreThemeSettings()
SaveManager:SetIgnoreIndexes({ 'MenuKeybind' })

ThemeManager:SetFolder('DaHoodScript')
SaveManager:SetFolder('DaHoodScript/configs')

-- Add Config Section (Right side)
local ConfigSection = Tabs.Settings:AddLeftGroupbox('Configuration')
SaveManager:BuildConfigSection(ConfigSection)

-- Add Theme Section (Right side, below config)
local ThemeSection = Tabs.Settings:AddLeftGroupbox('Themes')
ThemeManager:BuildThemeSection(ThemeSection)

-- Add Menu Settings (Left side)
local MenuSettingsSection = Tabs.Settings:AddLeftGroupbox('Menu Settings')

MenuSettingsSection:AddToggle('KeybindDisplay', {
    Text = 'Show Keybind List',
    Default = true,
    Tooltip = 'Shows active keybinds on screen',
    Callback = function(Value)
        Library.KeybindFrame.Visible = Value
    end
})

MenuSettingsSection:AddToggle('WatermarkToggle', {
    Text = 'Show Watermark',
    Default = true,
    Tooltip = 'Shows watermark on screen',
    Callback = function(Value)
        Library:SetWatermarkVisibility(Value)
    end
})

MenuSettingsSection:AddSlider('MenuTransparency', {
    Text = 'Menu Transparency',
    Default = 0,
    Min = 0,
    Max = 50,
    Rounding = 0,
    Compact = false,
    Tooltip = 'Adjusts menu background transparency',
    Callback = function(Value)
        Library:SetTransparency(Value / 100)
    end
})

-- Load autoload config
SaveManager:LoadAutoloadConfig()

-- Notify user
Library:Notify('Da Hood Script Loaded!', 3)
