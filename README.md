--[[
    ICE CORE | PRIVATE CHEAT EDITION (by AA)
    Modifications: Circle FOV Restored, Inventory Viewer (Backpack+Hand) Re-added
]]

local Players = game:GetService("Players")
local LocalPlayer = Players.LocalPlayer
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local TweenService = game:GetService("TweenService")
local HttpService = game:GetService("HttpService")
local Camera = workspace.CurrentCamera

-- // CLEANUP PREVIOUS EXECUTION
if _G.FenixUnload then pcall(function() _G.FenixUnload() end) end

local Connections = {}
local ESP_Texts = {}
local ESP_Tracers = {}
local ESP_Inventory = {}
local UIUpdaters = {} 

-- // PROMPT CACHE SYSTEM
local PromptCache = {}
for _, obj in ipairs(workspace:GetDescendants()) do
    if obj:IsA("ProximityPrompt") then PromptCache[obj] = true end
end
table.insert(Connections, workspace.DescendantAdded:Connect(function(obj)
    if obj:IsA("ProximityPrompt") then PromptCache[obj] = true end
end))
table.insert(Connections, workspace.DescendantRemoving:Connect(function(obj)
    if obj:IsA("ProximityPrompt") then PromptCache[obj] = nil end
end))

-- // LIMPEZA DE GHOST ESP
table.insert(Connections, Players.PlayerRemoving:Connect(function(player)
    if ESP_Tracers[player] then ESP_Tracers[player]:Remove(); ESP_Tracers[player] = nil end
    if ESP_Texts[player] then ESP_Texts[player]:Remove(); ESP_Texts[player] = nil end
    if ESP_Inventory[player] then ESP_Inventory[player]:Remove(); ESP_Inventory[player] = nil end
end))

-- // STATE & SETTINGS
_G.State = {
    -- PVP
    AimAssist = false,
    Lock = false, 
    TeamCheck = false,
    IgnoreNeutral = false,
    AimbotSmoothness = 25, 
    ShowFOV = true, 
    FOVRadius = 93, 
    UseFOV = true,
    AimPart = "Head",
    
    -- Visuals
    TextESP = false, 
    Tracers = false,
    InvESP = false,
    
    -- Misc
    CarBoost = false, CarSpeedMultiplier = 2, 
    InstantBrake = false,
    
    -- Farm
    AutoGari = false, FarmSpeed = 35,
    AutoLockpick = false,
    
    Visible = true
}
local State = _G.State
local LockedTarget = nil

-- // PREMIUM CONFIG SYSTEM
local CurrentConfigName = "Default"

local function SaveConfig()
    pcall(function()
        if writefile then 
            local filename = "IceCfg_" .. CurrentConfigName .. ".json"
            writefile(filename, HttpService:JSONEncode(State)) 
        end
    end)
end

local function LoadConfig(configName)
    pcall(function()
        local filename = "IceCfg_" .. configName .. ".json"
        if isfile and readfile and isfile(filename) then
            local decoded = HttpService:JSONDecode(readfile(filename))
            for k, v in pairs(decoded) do 
                if State[k] ~= nil then 
                    if k ~= "AutoGari" and k ~= "Lock" and k ~= "AimAssist" and k ~= "CarBoost" and k ~= "AutoLockpick" then
                        State[k] = v 
                    end
                end 
            end
            for _, updater in ipairs(UIUpdaters) do pcall(updater) end
        end
    end)
end

LoadConfig("Default")

local function Tween(obj, props, time)
    local tween = TweenService:Create(obj, TweenInfo.new(time or 0.3, Enum.EasingStyle.Quint, Enum.EasingDirection.Out), props)
    tween:Play()
    return tween
end

------------------------------------------------------------------------------------------------
-- // UI CONSTRUCTOR
------------------------------------------------------------------------------------------------
local Theme = {
    Background = Color3.fromRGB(15, 15, 20),
    Container = Color3.fromRGB(22, 22, 28),
    Accent = Color3.fromRGB(170, 0, 255), 
    Text = Color3.fromRGB(240, 240, 240),
    TextDark = Color3.fromRGB(150, 150, 150),
    Border = Color3.fromRGB(40, 40, 50)
}

local targetUI = nil
pcall(function() targetUI = gethui() end)
if not targetUI then targetUI = game:GetService("CoreGui") end
if targetUI:FindFirstChild("VeteranoUI") then targetUI.VeteranoUI:Destroy() end

local ScreenGui = Instance.new("ScreenGui", targetUI)
ScreenGui.Name = "VeteranoUI"
ScreenGui.ResetOnSpawn = false
_G.FenixUI = ScreenGui

local MainFrame = Instance.new("Frame", ScreenGui)
MainFrame.Size = UDim2.new(0, 500, 0, 350)
MainFrame.Position = UDim2.new(0.5, -250, 0.5, -175)
MainFrame.BackgroundColor3 = Theme.Background
Instance.new("UICorner", MainFrame).CornerRadius = UDim.new(0, 8)
Instance.new("UIStroke", MainFrame).Color = Theme.Border

-- DRAG LOGIC
local dragToggle, dragStart, startPos
MainFrame.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 then
        dragToggle = true; dragStart = input.Position; startPos = MainFrame.Position
    end
end)
UserInputService.InputChanged:Connect(function(input)
    if dragToggle and input.UserInputType == Enum.UserInputType.MouseMovement then
        local delta = input.Position - dragStart
        MainFrame.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + delta.X, startPos.Y.Scale, startPos.Y.Offset + delta.Y)
    end
end)
UserInputService.InputEnded:Connect(function(input) dragToggle = false end)

-- Sidebar & Buttons
local Sidebar = Instance.new("Frame", MainFrame)
Sidebar.Size = UDim2.new(0, 140, 1, 0); Sidebar.BackgroundColor3 = Theme.Container
Instance.new("UICorner", Sidebar).CornerRadius = UDim.new(0, 8)

local Title = Instance.new("TextLabel", Sidebar)
Title.Size = UDim2.new(1, 0, 0, 40); Title.Position = UDim2.new(0, 0, 0, 10); Title.BackgroundTransparency = 1
Title.Text = "Ice"; Title.TextColor3 = Theme.Accent; Title.Font = Enum.Font.GothamBold; Title.TextSize = 18

-- MARCA DE ÁGUA "by AA"
local Watermark = Instance.new("TextLabel", MainFrame)
Watermark.Size = UDim2.new(0, 100, 0, 20)
Watermark.Position = UDim2.new(1, -110, 1, -25)
Watermark.BackgroundTransparency = 1
Watermark.Text = "by AA"
Watermark.TextColor3 = Theme.TextDark
Watermark.TextTransparency = 0.6 
Watermark.Font = Enum.Font.Gotham
Watermark.TextSize = 11
Watermark.TextXAlignment = Enum.TextXAlignment.Right

local CloseBtn = Instance.new("TextButton", MainFrame)
CloseBtn.Size = UDim2.new(0, 30, 0, 30); CloseBtn.Position = UDim2.new(1, -35, 0, 5); CloseBtn.BackgroundTransparency = 1
CloseBtn.Text = "X"; CloseBtn.TextColor3 = Color3.fromRGB(200, 50, 50); CloseBtn.Font = Enum.Font.GothamBold; CloseBtn.TextSize = 16

local OpenBtn = Instance.new("TextButton", ScreenGui)
OpenBtn.Size = UDim2.new(0, 40, 0, 40); OpenBtn.Position = UDim2.new(0, 20, 0, 20); OpenBtn.BackgroundColor3 = Theme.Accent
OpenBtn.Text = "⚡"; OpenBtn.TextColor3 = Theme.Text; OpenBtn.Font = Enum.Font.GothamBold; OpenBtn.TextSize = 20; OpenBtn.Visible = false
Instance.new("UICorner", OpenBtn).CornerRadius = UDim.new(1, 0)

CloseBtn.Activated:Connect(function() MainFrame.Visible = false; OpenBtn.Visible = true end)
OpenBtn.Activated:Connect(function() MainFrame.Visible = true; OpenBtn.Visible = false end)

local ContentContainer = Instance.new("Frame", MainFrame)
ContentContainer.Size = UDim2.new(1, -150, 1, -40); ContentContainer.Position = UDim2.new(0, 140, 0, 30); ContentContainer.BackgroundTransparency = 1
local TabButtons = {}

local function CreateTab(name, icon, isFirst)
    local Btn = Instance.new("TextButton", Sidebar)
    Btn.Size = UDim2.new(1, -20, 0, 35); Btn.Position = UDim2.new(0, 10, 0, 80 + (#TabButtons * 40))
    Btn.BackgroundColor3 = isFirst and Theme.Accent or Theme.Background; Btn.BackgroundTransparency = isFirst and 0.8 or 1
    Btn.Text = "  " .. icon .. "  " .. name; Btn.TextColor3 = isFirst and Theme.Accent or Theme.TextDark; Btn.Font = Enum.Font.GothamBold; Btn.TextSize = 12; Btn.TextXAlignment = Enum.TextXAlignment.Left; Instance.new("UICorner", Btn).CornerRadius = UDim.new(0, 6)
    
    local Scroll = Instance.new("ScrollingFrame", ContentContainer)
    Scroll.Size = UDim2.new(1, -10, 1, 0); Scroll.Position = UDim2.new(0, 10, 0, 0); Scroll.BackgroundTransparency = 1; Scroll.ScrollBarThickness = 2; Scroll.Visible = isFirst
    local Layout = Instance.new("UIListLayout", Scroll); Layout.Padding = UDim.new(0, 8)
    Scroll.ChildAdded:Connect(function() Scroll.CanvasSize = UDim2.new(0, 0, 0, Layout.AbsoluteContentSize.Y + 10) end)
    table.insert(TabButtons, {Btn = Btn, Scroll = Scroll})
    Btn.Activated:Connect(function()
        for _, t in pairs(TabButtons) do
            t.Scroll.Visible = (t.Scroll == Scroll); t.Btn.BackgroundTransparency = (t.Scroll == Scroll) and 0.8 or 1; t.Btn.TextColor3 = (t.Scroll == Scroll) and Theme.Accent or Theme.TextDark
        end
    end)
    return Scroll
end

local function CreateTextBox(parent, placeholder, callback)
    local Frame = Instance.new("Frame", parent); Frame.Size = UDim2.new(1, -10, 0, 35); Frame.BackgroundColor3 = Theme.Container; Instance.new("UICorner", Frame).CornerRadius = UDim.new(0, 6); Instance.new("UIStroke", Frame).Color = Theme.Border
    local TextBox = Instance.new("TextBox", Frame); TextBox.Size = UDim2.new(1, -20, 1, 0); TextBox.Position = UDim2.new(0, 10, 0, 0); TextBox.BackgroundTransparency = 1; TextBox.PlaceholderText = placeholder; TextBox.Text = ""; TextBox.TextColor3 = Theme.Text; TextBox.Font = Enum.Font.Gotham; TextBox.TextSize = 12; TextBox.TextXAlignment = Enum.TextXAlignment.Left; TextBox.ClearTextOnFocus = false
    TextBox.FocusLost:Connect(function()
        if callback then pcall(callback, TextBox.Text) end
    end)
    return TextBox
end

local function CreateToggle(parent, text, key)
    local Frame = Instance.new("Frame", parent); Frame.Size = UDim2.new(1, -10, 0, 35); Frame.BackgroundColor3 = Theme.Container; Instance.new("UICorner", Frame).CornerRadius = UDim.new(0, 6)
    local Label = Instance.new("TextLabel", Frame); Label.Size = UDim2.new(1, -60, 1, 0); Label.Position = UDim2.new(0, 15, 0, 0); Label.BackgroundTransparency = 1; Label.Text = text; Label.TextColor3 = Theme.Text; Label.Font = Enum.Font.Gotham; Label.TextSize = 13; Label.TextXAlignment = Enum.TextXAlignment.Left
    local SwitchBg = Instance.new("TextButton", Frame); SwitchBg.Size = UDim2.new(0, 40, 0, 20); SwitchBg.Position = UDim2.new(1, -50, 0.5, -10); SwitchBg.BackgroundColor3 = State[key] and Theme.Accent or Theme.Border; SwitchBg.Text = ""; Instance.new("UICorner", SwitchBg).CornerRadius = UDim.new(1, 0)
    local Circle = Instance.new("Frame", SwitchBg); Circle.Size = UDim2.new(0, 16, 0, 16); Circle.Position = State[key] and UDim2.new(1, -18, 0.5, -8) or UDim2.new(0, 2, 0.5, -8); Circle.BackgroundColor3 = Color3.new(1,1,1); Instance.new("UICorner", Circle).CornerRadius = UDim.new(1, 0)
    
    table.insert(UIUpdaters, function()
        Tween(SwitchBg, {BackgroundColor3 = State[key] and Theme.Accent or Theme.Border}, 0.2)
        Tween(Circle, {Position = State[key] and UDim2.new(1, -18, 0.5, -8) or UDim2.new(0, 2, 0.5, -8)}, 0.2)
    end)

    SwitchBg.Activated:Connect(function()
        State[key] = not State[key]
        Tween(SwitchBg, {BackgroundColor3 = State[key] and Theme.Accent or Theme.Border}, 0.2)
        Tween(Circle, {Position = State[key] and UDim2.new(1, -18, 0.5, -8) or UDim2.new(0, 2, 0.5, -8)}, 0.2)
        SaveConfig()
    end)
end

local function CreateSlider(parent, text, min, max, key, isFloat)
    local Frame = Instance.new("Frame", parent); Frame.Size = UDim2.new(1, -10, 0, 50); Frame.BackgroundColor3 = Theme.Container; Instance.new("UICorner", Frame).CornerRadius = UDim.new(0, 6)
    local Label = Instance.new("TextLabel", Frame); Label.Size = UDim2.new(1, -20, 0, 20); Label.Position = UDim2.new(0, 15, 0, 5); Label.BackgroundTransparency = 1; 
    
    local displayValue = isFloat and (State[key] / 10) or State[key]
    Label.Text = text .. ": " .. tostring(displayValue)
    Label.TextColor3 = Theme.Text; Label.Font = Enum.Font.Gotham; Label.TextSize = 12; Label.TextXAlignment = Enum.TextXAlignment.Left
    
    local Track = Instance.new("TextButton", Frame); Track.Size = UDim2.new(1, -30, 0, 6); Track.Position = UDim2.new(0, 15, 0, 32); Track.BackgroundColor3 = Theme.Border; Track.Text = ""; Instance.new("UICorner", Track).CornerRadius = UDim.new(1, 0)
    local Fill = Instance.new("Frame", Track); Fill.Size = UDim2.new((State[key]-min)/(max-min), 0, 1, 0); Fill.BackgroundColor3 = Theme.Accent; Instance.new("UICorner", Fill).CornerRadius = UDim.new(1, 0)
    local dragging = false

    table.insert(UIUpdaters, function()
        local pos = math.clamp((State[key] - min) / (max - min), 0, 1)
        Fill.Size = UDim2.new(pos, 0, 1, 0)
        local newDisplay = isFloat and (State[key] / 10) or State[key]
        Label.Text = text .. ": " .. tostring(newDisplay)
    end)

    local function Update(input)
        local pos = math.clamp((input.Position.X - Track.AbsolutePosition.X) / Track.AbsoluteSize.X, 0, 1)
        State[key] = math.floor(min + (max - min) * pos); 
        local newDisplay = isFloat and (State[key] / 10) or State[key]
        Label.Text = text .. ": " .. tostring(newDisplay)
        Fill.Size = UDim2.new(pos, 0, 1, 0); SaveConfig()
    end
    Track.InputBegan:Connect(function(i) if i.UserInputType == Enum.UserInputType.MouseButton1 then dragging = true; Update(i) end end)
    UserInputService.InputEnded:Connect(function(i) if i.UserInputType == Enum.UserInputType.MouseButton1 then dragging = false end end)
    UserInputService.InputChanged:Connect(function(i) if dragging and i.UserInputType == Enum.UserInputType.MouseMovement then Update(i) end end)
end

local function CreateButton(parent, text, callback)
    local Frame = Instance.new("Frame", parent); Frame.Size = UDim2.new(1, -10, 0, 35); Frame.BackgroundColor3 = Theme.Container; Instance.new("UICorner", Frame).CornerRadius = UDim.new(0, 6); Instance.new("UIStroke", Frame).Color = Theme.Border
    local Btn = Instance.new("TextButton", Frame); Btn.Size = UDim2.new(1, 0, 1, 0); Btn.BackgroundTransparency = 1; Btn.Text = text; Btn.TextColor3 = Theme.Text; Btn.Font = Enum.Font.GothamBold; Btn.TextSize = 12
    Btn.Activated:Connect(function()
        Tween(Frame, {BackgroundColor3 = Theme.Accent}, 0.1)
        task.delay(0.1, function() Tween(Frame, {BackgroundColor3 = Theme.Container}, 0.1) end)
        if callback then pcall(callback) end
    end)
end

-- // TABS BUILDER
local TabPVP = CreateTab("PVP", "⚔", true)
local TabVisuals = CreateTab("Visuals", "👁", false)
local TabMisc = CreateTab("Misc", "⚙", false)
local TabFarm = CreateTab("Auto Farm", "💸", false)
local TabConfig = CreateTab("Configs", "💾", false)

-- PVP TAB
CreateToggle(TabPVP, "Aim Assist (Auto-Lock)", "AimAssist")
CreateToggle(TabPVP, "Aim Lock (Right-Click)", "Lock")
CreateToggle(TabPVP, "Team Check", "TeamCheck")
CreateToggle(TabPVP, "Ignore Neutral", "IgnoreNeutral")
CreateSlider(TabPVP, "Aimbot Smoothness", 1, 100, "AimbotSmoothness", false)
CreateToggle(TabPVP, "Show FOV Circle", "ShowFOV")
CreateSlider(TabPVP, "FOV Radius", 10, 800, "FOVRadius", false)

-- VISUALS TAB
CreateToggle(TabVisuals, "Name & HP ESP", "TextESP")
CreateToggle(TabVisuals, "Inventory Viewer", "InvESP")
CreateToggle(TabVisuals, "ESP Tracers", "Tracers")

-- MISC TAB
CreateToggle(TabMisc, "Car Speed Boost", "CarBoost")
CreateSlider(TabMisc, "Car Multiplier (x0.1)", 1, 50, "CarSpeedMultiplier", true) 
CreateToggle(TabMisc, "Instant Brake (J Key)", "InstantBrake")

local UnloadBtn = Instance.new("TextButton", TabMisc)
UnloadBtn.Size = UDim2.new(1, -10, 0, 35)
UnloadBtn.BackgroundColor3 = Color3.fromRGB(180, 40, 40)
UnloadBtn.Text = "UNLOAD SCRIPT"
UnloadBtn.TextColor3 = Theme.Text
UnloadBtn.Font = Enum.Font.GothamBold
UnloadBtn.TextSize = 12
Instance.new("UICorner", UnloadBtn).CornerRadius = UDim.new(0, 6)
UnloadBtn.Activated:Connect(function() _G.FenixUnload() end)

-- AUTO FARM TAB
CreateToggle(TabFarm, "Auto Gari (10x Trash)", "AutoGari")
CreateSlider(TabFarm, "Farm Fly Speed", 16, 150, "FarmSpeed", false)
CreateToggle(TabFarm, "Auto Lockpick (Car/Door)", "AutoLockpick")

-- CONFIGS TAB
CreateTextBox(TabConfig, "Nome da Config (Ex: Legit, Rage)", function(text)
    if text and text ~= "" then
        CurrentConfigName = text
    end
end)

CreateButton(TabConfig, "💾 Salvar Configuração", function()
    SaveConfig()
end)

CreateButton(TabConfig, "📂 Carregar Configuração", function()
    LoadConfig(CurrentConfigName)
end)

------------------------------------------------------------------------------------------------
-- // CORE ENGINE
------------------------------------------------------------------------------------------------

local FOVDraw = Drawing.new("Circle")
FOVDraw.Thickness = 1.5; 
FOVDraw.NumSides = 60; 
FOVDraw.Filled = false; 
FOVDraw.Visible = false; 
FOVDraw.ZIndex = 999; 
FOVDraw.Color = Color3.fromRGB(153, 50, 204)

_G.FenixUnload = function()
    SaveConfig()
    for _, conn in pairs(Connections) do pcall(function() conn:Disconnect() end) end
    if _G.FenixUI then pcall(function() _G.FenixUI:Destroy() end) end
    if FOVDraw then FOVDraw:Remove() end
    for _, v in pairs(ESP_Texts) do v:Remove() end
    for _, v in pairs(ESP_Tracers) do v:Remove() end
    for _, v in pairs(ESP_Inventory) do v:Remove() end
    table.clear(ESP_Texts); table.clear(ESP_Tracers); table.clear(ESP_Inventory); table.clear(PromptCache)
end

-- // INPUT HANDLER (Instant Brake = J)
local IsBraking = false
table.insert(Connections, UserInputService.InputBegan:Connect(function(input, gameProcessed)
    if gameProcessed then return end
    if input.KeyCode == Enum.KeyCode.J then IsBraking = true end 
end))
table.insert(Connections, UserInputService.InputEnded:Connect(function(input)
    if input.KeyCode == Enum.KeyCode.J then IsBraking = false end
end))

-- // PHYSICS ENGINE (HEARTBEAT - 0 Lag)
table.insert(Connections, RunService.Heartbeat:Connect(function()
    pcall(function()
        if not _G.FenixUI then return end
        if LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("Humanoid") then
            local hum = LocalPlayer.Character.Humanoid
            
            local currentSeat = hum.SeatPart
            if currentSeat and currentSeat:IsA("VehicleSeat") then
                local v = currentSeat:FindFirstAncestorOfClass("Model")
                if v and v.PrimaryPart then
                    if State.CarBoost and currentSeat.Throttle == 1 then
                        local boostValue = State.CarSpeedMultiplier / 10 
                        v.PrimaryPart.AssemblyLinearVelocity = v.PrimaryPart.AssemblyLinearVelocity + (v.PrimaryPart.CFrame.LookVector * boostValue)
                    end
                    if State.InstantBrake and IsBraking then
                        v.PrimaryPart.AssemblyLinearVelocity = Vector3.new(0, 0, 0)
                        v.PrimaryPart.AssemblyAngularVelocity = Vector3.new(0, 0, 0)
                    end
                end
            end
        end
    end)
end))

-- // AUTO LOCKPICK LOGIC
task.spawn(function()
    while task.wait(0.5) do
        if not _G.FenixUI then break end
        pcall(function()
            if State.AutoLockpick and LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart") then
                local myPos = LocalPlayer.Character.HumanoidRootPart.Position
                for obj, _ in pairs(PromptCache) do
                    if obj.Parent and obj.Enabled then
                        local txt = string.lower(obj.Name .. obj.ActionText)
                        if string.find(txt, "lockpick") or string.find(txt, "arrombar") or string.find(txt, "destrancar") or string.find(txt, "roubar") or string.find(txt, "furto") then
                            local parent = obj.Parent
                            local partPos = nil
                            
                            if parent:IsA("BasePart") then partPos = parent.Position
                            elseif parent:IsA("Model") and parent.PrimaryPart then partPos = parent.PrimaryPart.Position end
                            
                            if partPos and (myPos - partPos).Magnitude <= obj.MaxActivationDistance + 5 then
                                local oldHold = obj.HoldDuration
                                obj.HoldDuration = 0
                                pcall(function() fireproximityprompt(obj, 1) end)
                                obj.HoldDuration = oldHold
                            end
                        end
                    end
                end
            end
        end)
    end
end)

-- // AUTO-FARM COM TIMEOUT/BLACKLIST DE LIXOS BUGADOS (100% TWEEN NORMAL)
task.spawn(function()
    local isDelivering = false 
    local RetryCount = {}
    local BlacklistedPrompts = {}
    
    while task.wait(0.5) do
        if not _G.FenixUI then break end
        pcall(function() 
            local char = LocalPlayer.Character
            if not char then return end
            local hrp = char:FindFirstChild("HumanoidRootPart")
            if not hrp then return end
            
            -- AUTO GARI
            if State.AutoGari then
                local trashCount = 0; local trashInHand = false; local anyTrashTool = nil
                
                for _, obj in ipairs(LocalPlayer.Backpack:GetChildren()) do
                    if obj:IsA("Tool") and string.match(obj.Name:lower(), "lixo|saco|trash|garbage") then trashCount += 1; anyTrashTool = obj end
                end
                for _, obj in ipairs(char:GetChildren()) do
                    if obj:IsA("Tool") and string.match(obj.Name:lower(), "lixo|saco|trash|garbage") then trashCount += 1; trashInHand = true; anyTrashTool = obj end
                end
                
                if trashCount >= 10 then isDelivering = true elseif trashCount == 0 then isDelivering = false end
                if isDelivering and not trashInHand and anyTrashTool then char.Humanoid:EquipTool(anyTrashTool); task.wait(0.3) end
                
                local closestPrompt = nil; local shortestDist = math.huge
                
                for obj, _ in pairs(PromptCache) do
                    -- Se o prompt estiver na blacklist por estar bugado, ignora
                    if BlacklistedPrompts[obj] then continue end
                    
                    if obj.Parent and obj.Enabled then
                        local txt = string.lower(obj.Name .. obj.ActionText .. obj.ObjectText .. (obj.Parent.Name or ""))
                        local isDeliveryPoint = string.find(txt, "entregue") or string.find(txt, "entregar") or string.find(txt, "caçamba")
                        local isTrashItem = string.find(txt, "lixo") or string.find(txt, "saco") or string.find(txt, "coletar")
                        
                        local isValid = false
                        if isDelivering and (isDeliveryPoint or (isTrashItem and string.find(txt, "interagir") and not string.find(txt, "coletar"))) then isValid = true
                        elseif not isDelivering and isTrashItem and not isDeliveryPoint and not string.find(txt, "entregue") then isValid = true end
                        
                        local isOccupied = false
                        if isValid and not isDelivering and obj.Parent:IsA("BasePart") then
                            for _, otherPlayer in ipairs(Players:GetPlayers()) do
                                if otherPlayer ~= LocalPlayer and otherPlayer.Character and otherPlayer.Character:FindFirstChild("HumanoidRootPart") then
                                    local distOther = (otherPlayer.Character.HumanoidRootPart.Position - obj.Parent.Position).Magnitude
                                    if distOther < 10 then 
                                        isOccupied = true
                                        break
                                    end
                                end
                            end
                        end
                        
                        if isValid and not isOccupied and obj.Parent:IsA("BasePart") then
                            local d = (hrp.Position - obj.Parent.Position).Magnitude
                            if d < shortestDist then shortestDist = d; closestPrompt = obj end
                        end
                    end
                end
                
                if closestPrompt then
                    local targetCFrame = closestPrompt.Parent.CFrame * CFrame.new(0, 0, 2.5)
                    local dist = (hrp.Position - closestPrompt.Parent.Position).Magnitude
                    
                    if dist > 8 then
                        local timeToTravel = dist / State.FarmSpeed
                        TweenService:Create(hrp, TweenInfo.new(timeToTravel, Enum.EasingStyle.Linear), {CFrame = targetCFrame}):Play()
                        task.wait(timeToTravel) 
                    else
                        RetryCount[closestPrompt] = (RetryCount[closestPrompt] or 0) + 1
                        if RetryCount[closestPrompt] > 8 then
                            BlacklistedPrompts[closestPrompt] = true
                            return 
                        end
                    end
                    
                    if closestPrompt.Enabled then
                        local oldLos = closestPrompt.RequiresLineOfSight
                        closestPrompt.RequiresLineOfSight = false
                        closestPrompt.HoldDuration = 0
                        
                        pcall(function() fireproximityprompt(closestPrompt, 1) end)
                        
                        closestPrompt.RequiresLineOfSight = oldLos 
                        task.wait(isDelivering and 0.5 or 1.2)
                    end
                end
            else
                table.clear(RetryCount)
                table.clear(BlacklistedPrompts)
            end
        end)
    end
end)

-- // RENDER LOOP (AIMBOT & ESP LOGIC SLAAXIT)
table.insert(Connections, RunService.RenderStepped:Connect(function()
    pcall(function() 
        local centerScreen = Vector2.new(Camera.ViewportSize.X / 2, Camera.ViewportSize.Y / 2)

        if State.ShowFOV then
            FOVDraw.Visible = true; FOVDraw.Radius = State.FOVRadius; FOVDraw.Position = centerScreen
        else FOVDraw.Visible = false end

        -- // PVP LOGIC
        local isAiming = false
        pcall(function() isAiming = UserInputService:IsMouseButtonPressed(Enum.UserInputType.MouseButton2) end)
        
        local shouldLock = (State.Lock and isAiming) or State.AimAssist

        if shouldLock then
            LockedTarget = nil
            local shortest = State.ShowFOV and State.FOVRadius or math.huge
            for _, p in pairs(Players:GetPlayers()) do
                if p ~= LocalPlayer and p.Character and p.Character:FindFirstChild("Humanoid") and p.Character.Humanoid.Health > 0 then
                    if (not State.TeamCheck or p.Team ~= LocalPlayer.Team) and (not State.IgnoreNeutral or not p.Neutral) then
                        local part = p.Character:FindFirstChild(State.AimPart or "Head")
                        if part then
                            local pos, on = Camera:WorldToViewportPoint(part.Position)
                            if on then
                                local dist = (centerScreen - Vector2.new(pos.X, pos.Y)).Magnitude
                                if dist < shortest then shortest = dist; LockedTarget = part end
                            end
                        end
                    end
                end
            end
            
            if LockedTarget then
                local targetCFrame = CFrame.new(Camera.CFrame.Position, LockedTarget.Position)
                if State.AimbotSmoothness >= 100 then Camera.CFrame = targetCFrame
                else Camera.CFrame = Camera.CFrame:Lerp(targetCFrame, State.AimbotSmoothness / 100) end
            end
        else LockedTarget = nil end

        -- // ESP LOGIC DIRETO (RESTAURADO DA VERSÃO SLAAXIT)
        for _, player in ipairs(Players:GetPlayers()) do
            if player ~= LocalPlayer then
                local char = player.Character
                
                -- Desenha apenas se o jogador estiver vivo (HP > 0) e renderizado no mundo
                if char and char:IsDescendantOf(workspace) and char:FindFirstChild("Humanoid") and char:FindFirstChild("HumanoidRootPart") and char.Humanoid.Health > 0 then

                    local hrp = char.HumanoidRootPart
                    local hum = char.Humanoid
                    local pos, onScreen = Camera:WorldToViewportPoint(hrp.Position)
                    
                    -- TRACERS
                    if State.Tracers and onScreen and pos.Z > 0 then
                        if not ESP_Tracers[player] then 
                            ESP_Tracers[player] = Drawing.new("Line"); 
                            ESP_Tracers[player].Thickness = 1.5 
                        end
                        ESP_Tracers[player].Visible = true; 
                        ESP_Tracers[player].Color = Theme.Accent
                        ESP_Tracers[player].From = Vector2.new(Camera.ViewportSize.X / 2, Camera.ViewportSize.Y); 
                        ESP_Tracers[player].To = Vector2.new(pos.X, pos.Y)
                    else 
                        if ESP_Tracers[player] then ESP_Tracers[player].Visible = false end 
                    end

                    -- LOCALIZAR ITENS (INVENTORY VIEWER)
                    local invItems = {}
                    
                    if char then
                        for _, v in ipairs(char:GetChildren()) do
                            if v:IsA("Tool") then table.insert(invItems, v.Name .. " (Mão)") end
                        end
                    end
                    
                    if player:FindFirstChild("Backpack") then
                        for _, v in ipairs(player.Backpack:GetChildren()) do
                            if v:IsA("Tool") then table.insert(invItems, v.Name) end
                        end
                    end

                    if onScreen and pos.Z > 0 then
                        local topPos = Camera:WorldToViewportPoint(hrp.Position + Vector3.new(0, 2.5, 0))
                        local legPos = Camera:WorldToViewportPoint(hrp.Position - Vector3.new(0, 3.5, 0))
                        local height = math.max(math.abs(topPos.Y - legPos.Y), 8)
                        
                        -- TEXT & HP ESP
                        if State.TextESP then
                            if not ESP_Texts[player] then 
                                ESP_Texts[player] = Drawing.new("Text"); 
                                ESP_Texts[player].Size = 13; 
                                ESP_Texts[player].Center = true; 
                                ESP_Texts[player].Outline = true; 
                                ESP_Texts[player].Font = 2 
                            end
                            ESP_Texts[player].Visible = true; 
                            ESP_Texts[player].Position = Vector2.new(pos.X, pos.Y + (height/2) + 2)
                            ESP_Texts[player].Text = player.Name .. " [" .. math.floor(hum.Health) .. " HP]"; 
                            ESP_Texts[player].Color = Color3.new(1,1,1)
                        else 
                            if ESP_Texts[player] then ESP_Texts[player].Visible = false end 
                        end
                        
                        -- INVENTORY VIEWER (ESP de Inventário)
                        if State.InvESP and #invItems > 0 then
                            if not ESP_Inventory[player] then
                                ESP_Inventory[player] = Drawing.new("Text")
                                ESP_Inventory[player].Size = 12
                                ESP_Inventory[player].Center = true
                                ESP_Inventory[player].Outline = true
                                ESP_Inventory[player].Font = 2
                            end
                            ESP_Inventory[player].Visible = true
                            local yOffset = State.TextESP and 18 or 2 
                            ESP_Inventory[player].Position = Vector2.new(pos.X, pos.Y + (height/2) + yOffset)
                            ESP_Inventory[player].Text = "[" .. table.concat(invItems, ", ") .. "]"
                            ESP_Inventory[player].Color = Color3.fromRGB(200, 200, 200)
                        else
                            if ESP_Inventory[player] then ESP_Inventory[player].Visible = false end
                        end
                        
                    else 
                        if ESP_Texts[player] then ESP_Texts[player].Visible = false end 
                        if ESP_Inventory[player] then ESP_Inventory[player].Visible = false end
                    end
                else
                    if ESP_Texts[player] then ESP_Texts[player].Visible = false end
                    if ESP_Tracers[player] then ESP_Tracers[player].Visible = false end
                    if ESP_Inventory[player] then ESP_Inventory[player].Visible = false end
                end
            end
        end
    end)
end))
