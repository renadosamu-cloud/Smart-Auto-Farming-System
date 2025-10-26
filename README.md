-- ======================================================================================
-- Smart Auto-Farming System (Compiled from 4 Scripts)
-- ======================================================================================

repeat task.wait() until game.Players.LocalPlayer and game:IsLoaded()

local player = game.Players.LocalPlayer
local rs = game:GetService("ReplicatedStorage")
local uis = game:GetService("UserInputService")
local http = game:GetService("HttpService")
local StarterGui = game:GetService("StarterGui")
local playerGui = player:WaitForChild("PlayerGui")

-- 1. CONFIGURATION & SETTINGS (จาก Script 1 และ 4)
-- ======================================================================================
local RESTART_WAVE = 50          -- รีสตาร์ทเมื่อถึง Wave นี้ (จาก Script 1)
local AUTO_RESTART_INTERVAL = 6 * 60  -- รีสตาร์ททุกๆ 6 นาที (จาก Script 1)
local MAX_RETRY_COUNT = 3        -- รีสตาร์ทเมื่อจบเกมครบ 3 ครั้ง (จาก Script 3)

local gameFolderPath = "VectorHub/Laststand/"
local filePath = gameFolderPath .. player.Name .. ".json"

_G.Settings = {
    AutoCard = true,
    AutoAbility = true,
    HalloweenP1 = false, -- Event Toggle 1
    HalloweenP2 = false, -- Event Toggle 2
}

-- 2. CARD PRIORITY (จาก Script 2)
-- ======================================================================================
-- ลำดับความสำคัญการเลือกการ์ด (ยิ่งต่ำยิ่งดี)
local cardPriority = {
    ["Weakened Resolve I"] = 13, ["Weakened Resolve II"] = 11, ["Weakened Resolve III"] = 4,
    ["Fog of War I"] = 12, ["Fog of War II"] = 10, ["Fog of War III"] = 5,
    ["Lingering Fear I"] = 6, ["Lingering Fear II"] = 2,
    ["Power Reversal I"] = 14, ["Power Reversal II"] = 9,
    ["Greedy Vampire's"] = 8, ["Hellish Gravity"] = 3, ["Deadly Striker"] = 7,
    ["Critical Denial"] = 1,
    ["Trick or Treat Coin Flip"] = 15,
    ["Devil's Sacrifice"] = 999, 
    ["Bullet Breaker I"] = 999, ["Bullet Breaker II"] = 999, ["Bullet Breaker III"] = 999,
    ["Hell Merchant I"] = 999, ["Hell Merchant II"] = 999, ["Hell Merchant III"] = 999,
    ["Hellish Warp I"] = 999, ["Hellish Warp II"] = 999,
    ["Fiery Surge I"] = 999, ["Fiery Surge II"] = 999,
    ["Grevious Wounds I"] = 999, ["Grevious Wounds II"] = 999,
    ["Scorching Hell I"] = 999, ["Scorching Hell II"] = 999,
    ["Fortune Flow"] = 999, ["Soul Link"] = 999,
}

-- 3. STATE & CONTROL VARIABLES (จาก Script 1 และ 3)
-- ======================================================================================
local restartRemote = rs.Remotes:WaitForChild("RestartMatch")
local setSettingsRemote = rs.Remotes:WaitForChild("SetSettings")
local abilityRemote = rs.Remotes:WaitForChild("Ability")
local seamlessRetryValue = playerGui:WaitForChild("Settings"):WaitForChild("SeamlessRetry")

local hasRestarted = false        -- ป้องกันการรีสตาร์ทซ้ำซ้อน
local lastAutoRestart = os.clock() -- เวลารีสตาร์ทตามเวลาล่าสุด
local matchCount = 0             -- ตัวนับรอบเกมที่จบลง
local isSelectingCard = false    -- สถานะการเลือกการ์ด (เพิ่มเข้ามาเพื่อซิงค์)

-- 4. UTILITY FUNCTIONS (ฟังก์ชันเสริม)
-- ======================================================================================

-- Notification (จาก Script 3)
local function notify(title, text)
    pcall(function()
        StarterGui:SetCore("SendNotification", {Title = title; Text = text; Duration = 3;})
    end)
end

-- Save/Load Settings (จาก Script 4)
local function LoadSettings()
    if not isfolder("VectorHub") then makefolder("VectorHub") end
    if not isfolder(gameFolderPath) then makefolder(gameFolderPath) end
    if not isfile(filePath) then
        writefile(filePath, http:JSONEncode(_G.Settings))
    else
        local success, data = pcall(function() return http:JSONDecode(readfile(filePath)) end)
        if success and type(data) == "table" then
            for i, v in pairs(data) do 
                if _G.Settings[i] ~= nil then _G.Settings[i] = v end 
            end
        end
    end
end

local function SaveSettings()
    writefile(filePath, http:JSONEncode(_G.Settings))
end

-- Set Seamless Retry (จาก Script 3)
local function setSeamlessRetry(state)
    setSettingsRemote:InvokeServer("SeamlessRetry")
    seamlessRetryValue.Value = state
end

-- Get Wave Label (จาก Script 1)
local function getWaveLabel()
    local success, label = pcall(function()
        return playerGui.MainUI.Top.Wave.Value
    end)
    return success and label or nil
end

-- 5. SMART RESTART LOGIC (รวม Script 1 และ 3)
-- ======================================================================================

local function smartRestart(reason)
    if hasRestarted or isSelectingCard then return end -- ห้ามรีสตาร์ทขณะเลือกการ์ด

    hasRestarted = true
    notify("Smart Restart", "Reason: " .. reason .. ". Waiting 10s...")
    setSeamlessRetry(false)     -- ปิด Seamless Retry ก่อนรีสตาร์ท

    task.wait(10)

    restartRemote:FireServer()
    
    -- รีเซ็ตสถานะ
    matchCount = 0 
    task.wait(5)
    hasRestarted = false
    lastAutoRestart = os.clock()
    setSeamlessRetry(true)
    notify("Restart Complete", "New round started. Seamless Retry Enabled.")
end

-- Wave Check (จาก Script 1)
local function checkWave()
    local waveLabel = getWaveLabel()
    if not waveLabel then return end

    local currentWave = tonumber(waveLabel.Text)
    if currentWave and currentWave >= RESTART_WAVE then
        smartRestart("Reached Wave " .. currentWave)
    end
end

-- EndGameUI Watch (จาก Script 3)
local function watchEndUI(c)
    if hasRestarted then return end 
    matchCount += 1
    notify("Match Finished", "Game Count: " .. matchCount .. "/" .. MAX_RETRY_COUNT)
    c.AncestryChanged:Wait() 
    if matchCount >= MAX_RETRY_COUNT then
        smartRestart("Reached Max Retries (" .. MAX_RETRY_COUNT .. ")")
    end
end

-- 6. AUTO CARD SELECTOR LOGIC (จาก Script 2)
-- ======================================================================================

local function getAvailableCards()
    local prompt = playerGui:FindFirstChild("Prompt")
    if not prompt then return nil end
    local frame = prompt:FindFirstChild("Frame")
    if not frame or not frame:FindFirstChild("Frame") then return nil end
    
    local cardButtons = {}
    for _, descendant in pairs(frame:GetDescendants()) do
        if descendant:IsA("TextLabel") and descendant.Parent and descendant.Parent:IsA("Frame") then
            local text = descendant.Text
            if cardPriority[text] then
                local button = descendant.Parent.Parent
                if button:IsA("GuiButton") or button:IsA("TextButton") or button:IsA("ImageButton") then
                    table.insert(cardButtons, {text = text, button = button})
                end
            end
        end
    end
    table.sort(cardButtons, function(a, b) return a.button.AbsolutePosition.X < b.button.AbsolutePosition.X end)
    local cards = {}
    for i, cardData in ipairs(cardButtons) do cards[i] = {name = cardData.text, button = cardData.button} end
    return #cards > 0 and cards or nil
end

local function findBestCard(availableCards)
    local bestIndex = 1
    local bestPriority = math.huge
    for cardIndex = 1, #availableCards do
        local cardData = availableCards[cardIndex]
        local priority = cardPriority[cardData.name] or 999
        if priority < bestPriority then
            bestPriority = priority
            bestIndex = cardIndex
        end
    end
    return availableCards[bestIndex]
end

local function pressConfirmButton()
    local confirmButton = playerGui:FindFirstChild("Prompt")?.Frame.Frame.Children[5].TextButton
    if confirmButton and confirmButton.TextLabel.Text == "Confirm" then
        pcall(function() confirmButton:Click() end) -- ใช้ .Click() แทนการ Fire Event
        notify("Card Selector", "Confirmed!")
        return true
    end
    return false
end

local function selectCard()
    local availableCards = getAvailableCards()
    if not availableCards then
        isSelectingCard = false
        return false
    end
    
    isSelectingCard = true
    
    local bestCardData = findBestCard(availableCards)
    notify("Card Selector", "Selecting: " .. (bestCardData.name or "Unknown"))
    
    -- คลิกการ์ด
    pcall(function() bestCardData.button:Click() end) -- ใช้ .Click()
    
    task.wait(0.2)
    
    pressConfirmButton()
    
    isSelectingCard = false
    return true
end


-- 7. AUTO ABILITY LOGIC (จาก Script 4)
-- ======================================================================================
local function autoAbility(currentWave)
    if not _G.Settings.AutoAbility then return end
    
    if currentWave and currentWave >= RESTART_WAVE then
        local bossFound = false
        for _, enemy in pairs(workspace.Enemies:GetChildren()) do
            if enemy.Name == "Boss" then
                bossFound = true
                break
            end
        end

        if bossFound then
            -- ใช้ความสามารถยูนิต LelouchEvo และ AsuraEvo
            local function fireAbility(unitName, abilityName)
                local unit = workspace.Towers:FindFirstChild(unitName)
                if unit then
                    local args = {unit, abilityName}
                    abilityRemote:InvokeServer(unpack(args))
                    task.wait(0.1)
                end
            end

            fireAbility("LelouchEvo", "Submission")
            fireAbility("LelouchEvo", "All of you, Die!")
            fireAbility("AsuraEvo", "Lines of Sanzu")
            notify("Auto Ability", "Fired Boss Killing Abilities!")
        end
    end
end

-- 8. UI & EVENT STARTER LOGIC (จาก Script 4)
-- ======================================================================================

local p1Button, p2Button -- ประกาศตัวแปรสำหรับปุ่ม

local function startEvent(eventName)
    if game.PlaceId == 12886143095 or game.PlaceId == 18583778121 then
        local eventFolder = rs:WaitForChild("Events"):WaitForChild("Hallowen2025")
        eventFolder:WaitForChild("Enter"):FireServer(eventName)
        task.wait(3)
        eventFolder:WaitForChild("Start"):FireServer(eventName)
        notify("Event Starter", "Started: " .. (eventName or "P1"))
    end
end

local function updateUI()
    if p1Button then
        local p1Enabled = _G.Settings.HalloweenP1
        p1Button.BackgroundColor3 = p1Enabled and Color3.fromRGB(0, 180, 90) or Color3.fromRGB(180, 50, 50)
        p1Button.Text = "P1: " .. (p1Enabled and "ON" or "OFF")
    end
    if p2Button then
        local p2Enabled = _G.Settings.HalloweenP2
        p2Button.BackgroundColor3 = p2Enabled and Color3.fromRGB(0, 180, 90) or Color3.fromRGB(180, 50, 50)
        p2Button.Text = "P2: " .. (p2Enabled and "ON" or "OFF")
    end
end

local function toggleP1()
    _G.Settings.HalloweenP1 = not _G.Settings.HalloweenP1
    SaveSettings()
    updateUI()
    if _G.Settings.HalloweenP1 then startEvent(nil) end
end

local function toggleP2()
    _G.Settings.HalloweenP2 = not _G.Settings.HalloweenP2
    SaveSettings()
    updateUI()
    if _G.Settings.HalloweenP2 then startEvent("Hallowen2025_P2") end
end

local function setupUI()
    local gui = Instance.new("ScreenGui")
    gui.Name = "HalloweenToggleGUI"
    gui.ResetOnSpawn = false
    gui.Parent = playerGui

    local function createButton(name, position, clickFunc)
        local button = Instance.new("TextButton")
        button.Size = UDim2.new(0, 180, 0, 40)
        button.Position = position
        button.BackgroundColor3 = Color3.fromRGB(40, 40, 40)
        button.TextColor3 = Color3.fromRGB(255, 255, 255)
        button.TextSize = 18
        button.Font = Enum.Font.GothamBold
        button.Parent = gui
        button.MouseButton1Click:Connect(clickFunc)
        return button
    end

    p1Button = createButton("P1Button", UDim2.new(0, 10, 0, 10), toggleP1)
    p2Button = createButton("P2Button", UDim2.new(0, 10, 0, 60), toggleP2)
    
    uis.InputBegan:Connect(function(input, gameProcessed)
        if gameProcessed then return end
        if input.KeyCode == Enum.KeyCode.PageUp then toggleP1()
        elseif input.KeyCode == Enum.KeyCode.PageDown then toggleP2() end
    end)

    updateUI()
end


-- 9. INITIALIZATION & MAIN LOOPS
-- ======================================================================================

LoadSettings()
setupUI()
notify("System Loaded", "Smart Auto-Farming System is Active!")

-- Auto Start Event
if _G.Settings.HalloweenP1 then task.defer(startEvent, nil) end
if _G.Settings.HalloweenP2 then task.defer(startEvent, "Hallowen2025_P2") end

-- Watch Wave for Auto Restart (จาก Script 1)
local waveLabel = getWaveLabel()
if waveLabel then
    waveLabel:GetPropertyChangedSignal("Text"):Connect(checkWave)
end

-- Watch EndGameUI for Match Count (จาก Script 3)
playerGui.ChildAdded:Connect(function(c)
    if c.Name == "EndGameUI" then
        watchEndUI(c)
    end
end)

-- Auto Restart by Time (จาก Script 1)
task.spawn(function()
    while task.wait(10) do
        local waveLabel = getWaveLabel()
        if waveLabel and os.clock() - lastAutoRestart >= AUTO_RESTART_INTERVAL then
            smartRestart("Timed Restart (" .. AUTO_RESTART_INTERVAL/60 .. " mins)")
        end
    end
end)

-- Main Control Loop: Auto Card Selector & Auto Ability (จาก Script 2 และ 4)
while task.wait(1) do
    -- Auto Card Selector (ความสำคัญสูงสุด)
    if _G.Settings.AutoCard then
        selectCard()
    end

    -- Auto Ability Check
    local waveLabel = getWaveLabel()
    if waveLabel then
        local currentWave = tonumber(waveLabel.Text)
        if currentWave then
            autoAbility(currentWave)
        end
    end
end
