-- Services
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local TweenService = game:GetService("TweenService")
local UserInputService = game:GetService("UserInputService")
local VirtualInputManager = game:GetService("VirtualInputManager")
local CoreGui = game:GetService("CoreGui")
local ReplicatedStorage = game:GetService("ReplicatedStorage")

local LocalPlayer = Players.LocalPlayer

-- Configuration
local Config = {
    Delay = 0.8,                        -- Transition delay to next money drop
    MaxAttempts = 2,                    -- Attempts before teleporting to the money drop
    TeleportWait = 4.0,                 -- Slower 4-second interval between teleports
    TeleportDistanceThreshold = 5,      -- Only teleport if the drop is 5 studs away or more
    CashierMoneyDropsToCollect = 6,     -- Number of money drops to collect per cashier before moving
    NotificationDuration = 3,
    RemovalRadius = 5                   -- Map objects within 5 studs around you are removed and return when you walk > 5 studs away
}

-- State Variables
local isAutoFarmRunning = false
local isCashierFarmRunning = false
local visitedDrops = {}
local lastTeleportTick = 0

-- Stats Tracking Variables (Using active farming time only for absolute rate precision)
local totalMoneyEarned = 0
local activeFarmingTime = 0
local lastTickTime = tick()

--[================================================================]--
--                            UTILITY FUNCTIONS                     --
--[================================================================]--

local function sendNotification(title, text)
    pcall(function()
        game:GetService("StarterGui"):SetCore("SendNotification", {
            Title = title;
            Text = text;
            Duration = Config.NotificationDuration;
        })
    end)
end

-- Anti-AFK
local function setupAntiAFK()
    local vu = game:GetService("VirtualUser")
    LocalPlayer.Idled:Connect(function()
        vu:Button2Down(Vector2.new(0,0), workspace.CurrentCamera.CFrame)
        task.wait(1)
        vu:Button2Up(Vector2.new(0,0), workspace.CurrentCamera.CFrame)
    end)
end
setupAntiAFK()

-- Auto Unequip Tool function
local function unequipActiveTool()
    local char = LocalPlayer.Character
    if not char then return end
    local humanoid = char:FindFirstChildOfClass("Humanoid")
    if humanoid then
        humanoid:UnequipTools()
    end
end

-- Equip Combat tool ("Combat" or fists/punch tool)
local function equipCombatTool()
    local char = LocalPlayer.Character
    if not char then return end
    local humanoid = char:FindFirstChildOfClass("Humanoid")
    if not humanoid then return end

    local backpack = LocalPlayer:FindFirstChildOfClass("Backpack")
    if backpack then
        local combatTool = backpack:FindFirstChild("Combat")
        if combatTool then
            humanoid:EquipTool(combatTool)
            return true
        end
    end

    local equippedCombat = char:FindFirstChild("Combat")
    if equippedCombat then
        return true
    end

    return false
end

-- Safely get CFrame for BaseParts or Models
local function getObjectCFrame(obj)
    if not obj then return nil end
    if obj:IsA("BasePart") then
        return obj.CFrame
    elseif obj:IsA("Model") then
        if obj.PrimaryPart then
            return obj.PrimaryPart.CFrame
        else
            return obj:GetPivot()
        end
    end
    return nil
end

-- Check if a cashier is valid to target based on health (must be between 1 and 100) and general broken state
local function isValidCashier(cashier)
    if not cashier or not cashier.Parent then return false end
    
    local isValid = false
    pcall(function()
        local humanoid = cashier:FindFirstChild("Humanoid")
        if humanoid then
            local hp = humanoid.Health
            if hp >= 1 and hp <= 100 then
                isValid = true
            end
        else
            if cashier:IsA("BasePart") then
                if cashier.Transparency < 0.9 and cashier.CanCollide then
                    isValid = true
                end
            elseif cashier:IsA("Model") then
                local primary = cashier.PrimaryPart or cashier:FindFirstChildWhichIsA("BasePart")
                if primary and primary.Transparency < 0.9 and primary.CanCollide then
                    isValid = true
                end
            end
        end
    end)
    
    return isValid
end

-- Try to parse the exact cash value from the money drop name or text labels
local function getMoneyValue(obj)
    local val = tonumber(obj.Name)
    if val then return val end

    for _, descendant in ipairs(obj:GetDescendants()) do
        if descendant:IsA("TextLabel") then
            local cleanText = descendant.Text:gsub("[^%d]", "")
            local parsedNum = tonumber(cleanText)
            if parsedNum and parsedNum > 0 then
                return parsedNum
            end
        end
    end
    
    return 25 -- Default fallback value estimation for standard drops if text is hidden
end

-- Real-time precise balance delta tracking hook
local lastKnownLeaderstatCash = nil
local function getPlayerLeaderstatCash()
    pcall(function()
        local leaderstats = LocalPlayer:FindFirstChild("leaderstats")
        if leaderstats then
            for _, stat in ipairs(leaderstats:GetChildren()) do
                local name = stat.Name:lower()
                if name:find("cash") or name:find("wallet") or name:find("money") then
                    if stat:IsA("IntValue") or stat:IsA("NumberValue") then
                        return stat.Value
                    end
                end
            end
        end
    end)
    return nil
end

local function addTrackedMoney(amount)
    totalMoneyEarned = totalMoneyEarned + (amount or 25)
end

-- Monitor true in-game leaderstats balance shifts to catch earnings that bypass standard drops
task.spawn(function()
    while true do
        task.wait(0.5)
        local currentCash = getPlayerLeaderstatCash()
        if currentCash then
            if lastKnownLeaderstatCash then
                local diff = currentCash - lastKnownLeaderstatCash
                if diff > 0 then
                    totalMoneyEarned = totalMoneyEarned + diff
                end
            end
            lastKnownLeaderstatCash = currentCash
        end
    end
end)

-- Simulate natural screen tap/hover directly on the MoneyDrop object with a right offset
local function interactWithMoneyNaturally(dropObj)
    if not dropObj or not dropObj.Parent then return end

    local camera = workspace.CurrentCamera
    local cf = getObjectCFrame(dropObj)
    if not cf or not camera then return end

    camera.CFrame = CFrame.new(camera.CFrame.Position, cf.Position)
    task.wait(0.06)

    local screenPos, onScreen = camera:WorldToViewportPoint(cf.Position)
    if not onScreen or screenPos.Z <= 0 then return end

    local x, y = screenPos.X + 40, screenPos.Y

    pcall(function()
        VirtualInputManager:SendMouseMoveEvent(x, y, workspace)
        task.wait(0.08)

        VirtualInputManager:SendMouseButtonEvent(x, y, 0, true, workspace, 0)
        task.wait(0.08)
        VirtualInputManager:SendMouseButtonEvent(x, y, 0, false, workspace, 0)
    end)
end

-- Teleport directly to money drops within 20 studs after breaking a cashier and collect them
local function collectMoneyDropsByTeleportingWithin20Studs()
    local startTime = tick()
    
    while (isAutoFarmRunning or isCashierFarmRunning) and (tick() - startTime < 4.5) do
        local char = LocalPlayer.Character
        if not char or not char:FindFirstChild("HumanoidRootPart") then break end
        local rootPart = char.HumanoidRootPart
        
        rootPart.Anchored = false

        local dropFolder = workspace:FindFirstChild("Ignored") and workspace.Ignored:FindFirstChild("Drop")
        if not dropFolder then 
            task.wait(0.3)
            continue 
        end

        local foundAny = false
        for _, obj in ipairs(dropFolder:GetChildren()) do
            if obj.Name == "MoneyDrop" and obj.Parent then
                local cf = getObjectCFrame(obj)
                if cf then
                    local dist = (cf.Position - rootPart.Position).Magnitude
                    if dist <= 20 then
                        foundAny = true
                        unequipActiveTool()
                        
                        rootPart.CFrame = cf + Vector3.new(0, 1.5, 0)
                        task.wait(0.15)
                        
                        addTrackedMoney(getMoneyValue(obj))
                        interactWithMoneyNaturally(obj)
                        task.wait(0.15)
                    end
                end
            end
        end

        if not foundAny then
            task.wait(0.2)
        else
            task.wait(0.1)
        end
    end
end

-- Reusable collection loop for general Auto Farm Cash
local function collectNearbyMoneyDrops(maxToCollect)
    local collectedCount = 0
    local collectTimeout = tick()

    while (isAutoFarmRunning or isCashierFarmRunning) and collectedCount < (maxToCollect or Config.CashierMoneyDropsToCollect) do
        if tick() - collectTimeout > 6 then break end

        local char = LocalPlayer.Character
        if not char or not char:FindFirstChild("HumanoidRootPart") then break end
        local rootPart = char.HumanoidRootPart
        rootPart.Anchored = false

        local moneyObj = (function()
            local dropFolder = workspace:FindFirstChild("Ignored") and workspace.Ignored:FindFirstChild("Drop")
            if not dropFolder then return nil end

            local bestObj = nil
            local bestScore = -math.huge
            local unvisitedDrops = {}

            for _, obj in ipairs(dropFolder:GetChildren()) do
                if obj.Name == "MoneyDrop" then
                    if not visitedDrops[obj] then
                        table.insert(unvisitedDrops, obj)
                        local cf = getObjectCFrame(obj)
                        if cf then
                            local dist = (cf.Position - rootPart.Position).Magnitude
                            local value = getMoneyValue(obj)
                            local score = value - (dist * 1.5)

                            if score > bestScore then
                                bestScore = score
                                bestObj = obj
                            end
                        end
                    end
                end
            end

            if #unvisitedDrops == 0 then
                table.clear(visitedDrops)
            end

            return bestObj
        end)()

        if moneyObj then
            local attempts = 0
            while (isAutoFarmRunning or isCashierFarmRunning) and moneyObj and moneyObj.Parent and moneyObj:IsDescendantOf(workspace) and attempts < Config.MaxAttempts do
                unequipActiveTool()
                
                local cf = getObjectCFrame(moneyObj)
                if cf then
                    local dist = (cf.Position - rootPart.Position).Magnitude
                    if dist >= Config.TeleportDistanceThreshold then
                        local currentTime = tick()
                        if currentTime - lastTeleportTick < Config.TeleportWait then
                            task.wait(Config.TeleportWait - (currentTime - lastTeleportTick))
                        end
                        rootPart.CFrame = cf + Vector3.new(0, 1.5, 0)
                        lastTeleportTick = tick()
                        task.wait(0.4)
                    end
                end
                
                addTrackedMoney(getMoneyValue(moneyObj))
                interactWithMoneyNaturally(moneyObj)
                attempts = attempts + 1
            end

            visitedDrops[moneyObj] = true
            collectedCount = collectedCount + 1
            task.wait(Config.Delay)
        else
            task.wait(0.5)
        end
    end
end

--[================================================================]--
--                    DEAD / DOWNED CHECK LOGIC                     --
--[================================================================]--

task.spawn(function()
    while true do
        task.wait(0.5)
        local char = LocalPlayer.Character
        if char then
            local humanoid = char:FindFirstChildOfClass("Humanoid")
            if humanoid and humanoid.Health <= 0 then
                if isAutoFarmRunning or isCashierFarmRunning then
                    pcall(function()
                        humanoid.Health = 0
                    end)
                    local oldChar = char
                    repeat task.wait(0.5) until LocalPlayer.Character ~= oldChar and LocalPlayer.Character:FindFirstChild("HumanoidRootPart")
                    task.wait(1)
                end
            end
        end
    end
end)

--[================================================================]--
--                            UI CREATION                           --
--[================================================================]--

if CoreGui:FindFirstChild("DaHoodAutoFarmUI") then
    CoreGui.DaHoodAutoFarmUI:Destroy()
end

local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "DaHoodAutoFarmUI"
ScreenGui.ResetOnSpawn = false
ScreenGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling

if syn and syn.protect_gui then
    syn.protect_gui(ScreenGui)
    ScreenGui.Parent = CoreGui
else
    ScreenGui.Parent = CoreGui
end

-- Main Frame
local MainFrame = Instance.new("Frame")
MainFrame.Name = "MainFrame"
MainFrame.Size = UDim2.new(0, 320, 0, 420)
MainFrame.Position = UDim2.new(0.5, -160, 0.5, -210)
MainFrame.BackgroundColor3 = Color3.fromRGB(20, 20, 25)
MainFrame.BorderSizePixel = 0
MainFrame.Active = true
MainFrame.Draggable = true
MainFrame.Parent = ScreenGui

local UICorner = Instance.new("UICorner")
UICorner.CornerRadius = UDim.new(0, 10)
UICorner.Parent = MainFrame

-- Top Bar
local TopBar = Instance.new("Frame")
TopBar.Name = "TopBar"
TopBar.Size = UDim2.new(1, 0, 0, 40)
TopBar.BackgroundColor3 = Color3.fromRGB(30, 30, 38)
TopBar.BorderSizePixel = 0
TopBar.Parent = MainFrame

local TopBarCorner = Instance.new("UICorner")
TopBarCorner.CornerRadius = UDim.new(0, 10)
TopBarCorner.Parent = TopBar

local TopBarFix = Instance.new("Frame")
TopBarFix.Size = UDim2.new(1, 0, 0, 10)
TopBarFix.Position = UDim2.new(0, 0, 1, -10)
TopBarFix.BackgroundColor3 = Color3.fromRGB(30, 30, 38)
TopBarFix.BorderSizePixel = 0
TopBarFix.Parent = TopBar

-- Title
local TitleLabel = Instance.new("TextLabel")
TitleLabel.Size = UDim2.new(1, -50, 1, 0)
TitleLabel.Position = UDim2.new(0, 15, 0, 0)
TitleLabel.BackgroundTransparency = 1
TitleLabel.Font = Enum.Font.GothamBold
TitleLabel.Text = "Da Hood Auto Farm"
TitleLabel.TextColor3 = Color3.fromRGB(240, 240, 240)
TitleLabel.TextSize = 14
TitleLabel.TextXAlignment = Enum.TextXAlignment.Left
TitleLabel.Parent = TopBar

-- Minimize Button
local MinimizeButton = Instance.new("TextButton")
MinimizeButton.Size = UDim2.new(0, 30, 0, 30)
MinimizeButton.Position = UDim2.new(1, -35, 0, 5)
MinimizeButton.BackgroundColor3 = Color3.fromRGB(45, 45, 55)
MinimizeButton.Font = Enum.Font.GothamBold
MinimizeButton.Text = "-"
MinimizeButton.TextColor3 = Color3.fromRGB(240, 240, 240)
MinimizeButton.TextSize = 16
MinimizeButton.Parent = TopBar

local MinCorner = Instance.new("UICorner")
MinCorner.CornerRadius = UDim.new(0, 6)
MinCorner.Parent = MinimizeButton

-- Container for content
local Container = Instance.new("ScrollingFrame")
Container.Name = "Container"
Container.Size = UDim2.new(1, -20, 1, -55)
Container.Position = UDim2.new(0, 10, 0, 45)
Container.BackgroundTransparency = 1
Container.BorderSizePixel = 0
Container.CanvasSize = UDim2.new(0, 0, 0, 360)
Container.ScrollBarThickness = 4
Container.Parent = MainFrame

-- Status Section
local StatusContainer = Instance.new("Frame")
StatusContainer.Size = UDim2.new(1, 0, 0, 45)
StatusContainer.BackgroundColor3 = Color3.fromRGB(28, 28, 35)
StatusContainer.Parent = Container

local StatusCorner = Instance.new("UICorner")
StatusCorner.CornerRadius = UDim.new(0, 6)
StatusCorner.Parent = StatusContainer

local StatusTitle = Instance.new("TextLabel")
StatusTitle.Size = UDim2.new(0.5, 0, 1, 0)
StatusTitle.Position = UDim2.new(0, 12, 0, 0)
StatusTitle.BackgroundTransparency = 1
StatusTitle.Font = Enum.Font.GothamMedium
StatusTitle.Text = "Status:"
StatusTitle.TextColor3 = Color3.fromRGB(180, 180, 190)
StatusTitle.TextSize = 13
StatusTitle.TextXAlignment = Enum.TextXAlignment.Left
StatusTitle.Parent = StatusContainer

local StatusLabel = Instance.new("TextLabel")
StatusLabel.Size = UDim2.new(0.5, 0, 1, 0)
StatusLabel.Position = UDim2.new(0.5, -12, 0, 0)
StatusLabel.BackgroundTransparency = 1
StatusLabel.Font = Enum.Font.GothamBold
StatusLabel.Text = "Stopped"
StatusLabel.TextColor3 = Color3.fromRGB(255, 80, 80)
StatusLabel.TextSize = 13
StatusLabel.TextXAlignment = Enum.TextXAlignment.Right
StatusLabel.Parent = StatusContainer

-- Stats Section (Money Made & Per Minute)
local StatsContainer = Instance.new("Frame")
StatsContainer.Size = UDim2.new(1, 0, 0, 80)
StatsContainer.Position = UDim2.new(0, 0, 0, 55)
StatsContainer.BackgroundColor3 = Color3.fromRGB(28, 28, 35)
StatsContainer.Parent = Container

local StatsCorner = Instance.new("UICorner")
StatsCorner.CornerRadius = UDim.new(0, 6)
StatsCorner.Parent = StatsContainer

local StatsHeader = Instance.new("TextLabel")
StatsHeader.Size = UDim2.new(1, -24, 0, 25)
StatsHeader.Position = UDim2.new(0, 12, 0, 5)
StatsHeader.BackgroundTransparency = 1
StatsHeader.Font = Enum.Font.GothamBold
StatsHeader.Text = "Session Statistics"
StatsHeader.TextColor3 = Color3.fromRGB(200, 200, 210)
StatsHeader.TextSize = 12
StatsHeader.TextXAlignment = Enum.TextXAlignment.Left
StatsHeader.Parent = StatsContainer

local MoneyMadeLabel = Instance.new("TextLabel")
MoneyMadeLabel.Size = UDim2.new(1, -24, 0, 20)
MoneyMadeLabel.Position = UDim2.new(0, 12, 0, 30)
MoneyMadeLabel.BackgroundTransparency = 1
MoneyMadeLabel.Font = Enum.Font.GothamMedium
MoneyMadeLabel.Text = "Earned: $0"
MoneyMadeLabel.TextColor3 = Color3.fromRGB(150, 220, 150)
MoneyMadeLabel.TextSize = 12
MoneyMadeLabel.TextXAlignment = Enum.TextXAlignment.Left
MoneyMadeLabel.Parent = StatsContainer

local MoneyPerMinLabel = Instance.new("TextLabel")
MoneyPerMinLabel.Size = UDim2.new(1, -24, 0, 20)
MoneyPerMinLabel.Position = UDim2.new(0, 12, 0, 50)
MoneyPerMinLabel.BackgroundTransparency = 1
MoneyPerMinLabel.Font = Enum.Font.GothamMedium
MoneyPerMinLabel.Text = "Rate: $0 / min"
MoneyPerMinLabel.TextColor3 = Color3.fromRGB(150, 200, 255)
MoneyPerMinLabel.TextSize = 12
MoneyPerMinLabel.TextXAlignment = Enum.TextXAlignment.Left
MoneyPerMinLabel.Parent = StatsContainer

-- Auto Farm Toggle Row
local ToggleContainer = Instance.new("Frame")
ToggleContainer.Size = UDim2.new(1, 0, 0, 50)
ToggleContainer.Position = UDim2.new(0, 0, 0, 145)
ToggleContainer.BackgroundColor3 = Color3.fromRGB(28, 28, 35)
ToggleContainer.Parent = Container

local ToggleCorner = Instance.new("UICorner")
ToggleCorner.CornerRadius = UDim.new(0, 6)
ToggleCorner.Parent = ToggleContainer

local ToggleTitle = Instance.new("TextLabel")
ToggleTitle.Size = UDim2.new(0.7, 0, 1, 0)
ToggleTitle.Position = UDim2.new(0, 12, 0, 0)
ToggleTitle.BackgroundTransparency = 1
ToggleTitle.Font = Enum.Font.GothamMedium
ToggleTitle.Text = "Auto Farm Cash"
ToggleTitle.TextColor3 = Color3.fromRGB(220, 220, 230)
ToggleTitle.TextSize = 13
ToggleTitle.TextXAlignment = Enum.TextXAlignment.Left
ToggleTitle.Parent = ToggleContainer

local ToggleBtn = Instance.new("TextButton")
ToggleBtn.Size = UDim2.new(0, 46, 0, 24)
ToggleBtn.Position = UDim2.new(1, -58, 0.5, -12)
ToggleBtn.BackgroundColor3 = Color3.fromRGB(50, 50, 60)
ToggleBtn.Text = ""
ToggleBtn.Parent = ToggleContainer

local ToggleBtnCorner = Instance.new("UICorner")
ToggleBtnCorner.CornerRadius = UDim.new(1, 0)
ToggleBtnCorner.Parent = ToggleBtn

local ToggleCircle = Instance.new("Frame")
ToggleCircle.Size = UDim2.new(0, 18, 0, 18)
ToggleCircle.Position = UDim2.new(0, 3, 0.5, -9)
ToggleCircle.BackgroundColor3 = Color3.fromRGB(240, 240, 240)
ToggleCircle.Parent = ToggleBtn

local CircleCorner = Instance.new("UICorner")
CircleCorner.CornerRadius = UDim.new(1, 0)
CircleCorner.Parent = ToggleCircle

-- Cashier Farm Toggle Row
local CashierContainer = Instance.new("Frame")
CashierContainer.Size = UDim2.new(1, 0, 0, 50)
CashierContainer.Position = UDim2.new(0, 0, 0, 205)
CashierContainer.BackgroundColor3 = Color3.fromRGB(28, 28, 35)
CashierContainer.Parent = Container

local CashierCorner = Instance.new("UICorner")
CashierCorner.CornerRadius = UDim.new(0, 6)
CashierCorner.Parent = CashierContainer

local CashierTitle = Instance.new("TextLabel")
CashierTitle.Size = UDim2.new(0.7, 0, 1, 0)
CashierTitle.Position = UDim2.new(0, 12, 0, 0)
CashierTitle.BackgroundTransparency = 1
CashierTitle.Font = Enum.Font.GothamMedium
CashierTitle.Text = "Auto Cashier Farm"
CashierTitle.TextColor3 = Color3.fromRGB(220, 220, 230)
CashierTitle.TextSize = 13
CashierTitle.TextXAlignment = Enum.TextXAlignment.Left
CashierTitle.Parent = CashierContainer

local CashierToggleBtn = Instance.new("TextButton")
CashierToggleBtn.Size = UDim2.new(0, 46, 0, 24)
CashierToggleBtn.Position = UDim2.new(1, -58, 0.5, -12)
CashierToggleBtn.BackgroundColor3 = Color3.fromRGB(50, 50, 60)
CashierToggleBtn.Text = ""
CashierToggleBtn.Parent = CashierContainer

local CashierBtnCorner = Instance.new("UICorner")
CashierBtnCorner.CornerRadius = UDim.new(1, 0)
CashierBtnCorner.Parent = CashierToggleBtn

local CashierToggleCircle = Instance.new("Frame")
CashierToggleCircle.Size = UDim2.new(0, 18, 0, 18)
CashierToggleCircle.Position = UDim2.new(0, 3, 0.5, -9)
CashierToggleCircle.BackgroundColor3 = Color3.fromRGB(240, 240, 240)
CashierToggleCircle.Parent = CashierToggleBtn

local CashierCircleCorner = Instance.new("UICorner")
CashierCircleCorner.CornerRadius = UDim.new(1, 0)
CashierCircleCorner.Parent = CashierToggleCircle

-- Update High-Precision Stats UI Loop
task.spawn(function()
    while true do
        task.wait(0.2)
        local currentTick = tick()
        local delta = currentTick - lastTickTime
        lastTickTime = currentTick

        if isAutoFarmRunning or isCashierFarmRunning then
            activeFarmingTime = activeFarmingTime + delta
        end

        local elapsedMinutes = activeFarmingTime / 60
        local rate = elapsedMinutes > 0 and math.floor(totalMoneyEarned / elapsedMinutes) or 0
        
        MoneyMadeLabel.Text = "Earned: $" .. tostring(totalMoneyEarned)
        MoneyPerMinLabel.Text = "Rate: $" .. tostring(rate) .. " / min"
    end
end)

--[================================================================]--
--                         AUTO FARM LOGIC                          --
--[================================================================]--

task.spawn(function()
    while true do
        if isAutoFarmRunning then
            collectNearbyMoneyDrops(1)
        else
            task.wait(0.5)
        end
    end
end)

--[================================================================]--
--                       CASHIER FARM LOGIC                         --
--[================================================================]--

task.spawn(function()
    while true do
        if isCashierFarmRunning then
            local cashiersFolder = workspace:FindFirstChild("Cashiers")
            local char = LocalPlayer.Character

            -- Comprehensive death/downed check function for current character state
            local function isPlayerDownedOrDead()
                local c = LocalPlayer.Character
                if not c then return true end
                local hum = c:FindFirstChildOfClass("Humanoid")
                if not hum or hum.Health <= 0 then return true end
                -- Da Hood knockout state check (BodyEffects handling)
                local bodyEffects = c:FindFirstChild("BodyEffects")
                if bodyEffects then
                    local ko = bodyEffects:FindFirstChild("Killed") or bodyEffects:FindFirstChild("Dead")
                    local grabbed = bodyEffects:FindFirstChild("Grabbed")
                    if (ko and ko.Value) or (grabbed and grabbed.Value) then
                        return true
                    end
                end
                return false
            end

            if cashiersFolder and char and char:FindFirstChild("HumanoidRootPart") and not isPlayerDownedOrDead() then
                local rootPart = char.HumanoidRootPart
                local cashiers = cashiersFolder:GetChildren()

                for _, cashier in ipairs(cashiers) do
                    if not isCashierFarmRunning or isPlayerDownedOrDead() then break end

                    if not isValidCashier(cashier) then continue end

                    local cashierCf = getObjectCFrame(cashier)

                    if cashierCf then
                        rootPart.Anchored = false
                        local currentTime = tick()
                        if currentTime - lastTeleportTick < Config.TeleportWait then
                            task.wait(Config.TeleportWait - (currentTime - lastTeleportTick))
                        end
                        
                        local targetBaseCFrame = cashierCf + Vector3.new(0, 3, 0)
                        rootPart.CFrame = targetBaseCFrame
                        lastTeleportTick = tick()
                        task.wait(0.4)

                        local lockedPositionCFrame = targetBaseCFrame - Vector3.new(0, 3, 0)
                        rootPart.CFrame = lockedPositionCFrame
                        rootPart.Anchored = true

                        equipCombatTool()
                        task.wait(0.2)

                        local startTime = tick()
                        while isCashierFarmRunning and cashier and cashier.Parent and not isPlayerDownedOrDead() do
                            if tick() - startTime > 15 then break end
                            if not isValidCashier(cashier) then break end

                            local currentDist = (rootPart.Position - lockedPositionCFrame.Position).Magnitude
                            if currentDist > 2 then
                                rootPart.Anchored = false
                                rootPart.CFrame = lockedPositionCFrame
                                task.wait(0.05)
                                rootPart.Anchored = true
                            end

                            equipCombatTool()

                            -- CHARGE ATTACK LOGIC (Set to 1.3s)
                            pcall(function()
                                local humanoid = char:FindFirstChildOfClass("Humanoid")
                                if humanoid then
                                    VirtualInputManager:SendMouseButtonEvent(0, 0, 0, true, workspace, 0)
                                    task.wait(1.3)
                                    VirtualInputManager:SendMouseButtonEvent(0, 0, 0, false, workspace, 0)
                                end
                            end)

                            task.wait(0.3)
                        end

                        unequipActiveTool()
                        task.wait(0.2)

                        rootPart.Anchored = false
                        
                        if not isPlayerDownedOrDead() then
                            collectMoneyDropsByTeleportingWithin20Studs()
                        end
                    end
                end
            else
                -- If dead, downed, or waiting for character load, pause and unanchor until respawned successfully
                if LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart") then
                    LocalPlayer.Character.HumanoidRootPart.Anchored = false
                end
                
                -- Wait until the character respawns fully and is ready/alive
                repeat
                    task.wait(0.5)
                until not isPlayerDownedOrDead() and LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart")
                
                task.wait(1) -- Extra stabilization buffer after full respawn before cycling back
            end
        else
            if LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart") then
                LocalPlayer.Character.HumanoidRootPart.Anchored = false
            end
            task.wait(0.5)
        end
    end
end)

-- Cleanup destroyed drops
task.spawn(function()
    while true do
        task.wait(10)
        for obj, _ in pairs(visitedDrops) do
            if not obj or not obj.Parent then
                visitedDrops[obj] = nil
            end
        end
    end
end)

--[================================================================]--
--                     UI INTERACTION LOGIC                         --
--[================================================================]--

local function updateStatusText()
    if isAutoFarmRunning and isCashierFarmRunning then
        StatusLabel.Text = "Farming Both"
        StatusLabel.TextColor3 = Color3.fromRGB(80, 255, 120)
    elseif isAutoFarmRunning then
        StatusLabel.Text = "Money Drops"
        StatusLabel.TextColor3 = Color3.fromRGB(80, 255, 120)
    elseif isCashierFarmRunning then
        StatusLabel.Text = "Cashiers"
        StatusLabel.TextColor3 = Color3.fromRGB(80, 255, 120)
    else
        StatusLabel.Text = "Stopped"
        StatusLabel.TextColor3 = Color3.fromRGB(255, 80, 80)
    end
end

ToggleBtn.MouseButton1Click:Connect(function()
    isAutoFarmRunning = not isAutoFarmRunning
    if isAutoFarmRunning then
        TweenService:Create(ToggleCircle, TweenInfo.new(0.2), {Position = UDim2.new(1, -21, 0.5, -9)}):Play()
        TweenService:Create(ToggleBtn, TweenInfo.new(0.2), {BackgroundColor3 = Color3.fromRGB(0, 170, 255)}):Play()
        sendNotification("Da Hood Auto Farm", "Money Drop Farm started!")
    else
        TweenService:Create(ToggleCircle, TweenInfo.new(0.2), {Position = UDim2.new(0, 3, 0.5, -9)}):Play()
        TweenService:Create(ToggleBtn, TweenInfo.new(0.2), {BackgroundColor3 = Color3.fromRGB(50, 50, 60)}):Play()
        sendNotification("Da Hood Auto Farm", "Money Drop Farm stopped.")
    end
    updateStatusText()
end)

CashierToggleBtn.MouseButton1Click:Connect(function()
    isCashierFarmRunning = not isCashierFarmRunning
    if isCashierFarmRunning then
        TweenService:Create(CashierToggleCircle, TweenInfo.new(0.2), {Position = UDim2.new(1, -21, 0.5, -9)}):Play()
        TweenService:Create(CashierToggleCircle, TweenInfo.new(0.2), {BackgroundColor3 = Color3.fromRGB(0, 170, 255)}):Play()
        sendNotification("Da Hood Auto Farm", "Cashier Farm started!")
    else
        TweenService:Create(CashierToggleCircle, TweenInfo.new(0.2), {Position = UDim2.new(0, 3, 0.5, -9)}):Play()
        TweenService:Create(CashierToggleCircle, TweenInfo.new(0.2), {BackgroundColor3 = Color3.fromRGB(50, 50, 60)}):Play()
        sendNotification("Da Hood Auto Farm", "Cashier Farm stopped.")
    end
    updateStatusText()
end)

local minimized = false
MinimizeButton.MouseButton1Click:Connect(function()
    minimized = not minimized
    if minimized then
        MinimizeButton.Text = "+"
        Container.Visible = false
        TweenService:Create(MainFrame, TweenInfo.new(0.2), {Size = UDim2.new(0, 320, 0, 40)}):Play()
    else
        MinimizeButton.Text = "-"
        Container.Visible = true
        TweenService:Create(MainFrame, TweenInfo.new(0.2), {Size = UDim2.new(0, 320, 0, 420)}):Play()
    end
end)
