-- Full merged ChanceBot + Rayfield UI + Scripts tab (YAHRM + Riddance)
-- Loads Rayfield, wires your original Chance Aimbot code to Rayfield controls.

-- // Load Rayfield
local Rayfield = loadstring(game:HttpGet('https://sirius.menu/rayfield'))()

-- // Services (same as original)
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local Stats = game:GetService("Stats")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local LocalPlayer = Players.LocalPlayer
local PlayerGui = LocalPlayer:WaitForChild("PlayerGui")
local StarterGui = game:GetService("StarterGui")

-- // Original variables & defaults (kept names from your old script)
local aimTargets = {"Slasher", "c00lkidd", "JohnDoe", "1x1x1x1", "Noli"}
local trackedAnimations = {
    ["103601716322988"] = true,
    ["133491532453922"] = true,
    ["86371356500204"] = true,
    ["76649505662612"] = true,
    ["81698196845041"] = true
}

-- state variables (initials from original)
local active = false
local predictionMode = "Velocity"
local spinDuration = 0.5
local aimMode = "Normal"
local messageWhenAim = false
local useCustomAnim = false
local customAnimId = nil
local autoCoinflip = false
local coinflipTargetCharge = "3"
local movementThreshold = 0.5
local aimDuration = 1.7

-- runtime state
local Humanoid, HRP = nil, nil
local lastTriggerTime = 0
local aiming = false
local shooting = false
local messageSentThisAim = false
local originalWS, originalJP, originalAutoRotate = nil, nil, nil
local prevFlintVisibleAim = false
local prevFlintVisibleShoot = false
local lastCoinflipTime = 0
local coinflipCooldown = 0.15

-- Remote ref (safe wait like original)
local RemoteEvent = ReplicatedStorage:WaitForChild("Modules"):WaitForChild("Network"):WaitForChild("RemoteEvent")

-- Helpers copied/adapted from original
local function setupCharacter(char)
    Humanoid = char:WaitForChild("Humanoid")
    HRP = char:WaitForChild("HumanoidRootPart")
end
if LocalPlayer.Character then setupCharacter(LocalPlayer.Character) end
LocalPlayer.CharacterAdded:Connect(setupCharacter)

local function getPingSeconds()
    local ok, pingStat = pcall(function()
        return Stats.Network.ServerStatsItem["Data Ping"]
    end)
    if ok and pingStat then
        return pingStat:GetValue() / 1000
    end
    return 0.1
end

local function getValidTarget()
    local killersFolder = workspace:FindFirstChild("Players") and workspace.Players:FindFirstChild("Killers")
    if killersFolder then
        for _, name in ipairs(aimTargets) do
            local target = killersFolder:FindFirstChild(name)
            if target and target:FindFirstChild("HumanoidRootPart") then
                return target.HumanoidRootPart, target:FindFirstChild("Humanoid")
            end
        end
    end
    return nil, nil
end

local function getPredictedAimPosPing(targetHRP, killerHumanoid)
    local ping = getPingSeconds()
    local velocity = targetHRP.Velocity
    if velocity.Magnitude <= movementThreshold then
        return targetHRP.Position
    end
    return targetHRP.Position + (velocity * ping)
end

local function getPredictedAimPosInfrontHRPPing(targetHRP)
    local ping = getPingSeconds()
    local studs = ping * 60
    if targetHRP.Velocity.Magnitude <= movementThreshold then
        return targetHRP.Position
    end
    return targetHRP.Position + (targetHRP.CFrame.LookVector * studs)
end

local function isFlintlockVisible()
    if not LocalPlayer.Character then return false end
    local flint = LocalPlayer.Character:FindFirstChild("Flintlock", true)
    if not flint then return false end

    if not (flint:IsA("BasePart") or flint:IsA("MeshPart") or flint:IsA("UnionOperation")) then
        flint = flint:FindFirstChildWhichIsA("BasePart", true)
        if not flint then return false end
    end

    if flint.Transparency >= 1 then
        return false
    end
    return true
end

local function sendChatMessage(text)
    if not text or text:match("^%s*$") then return end
    local ok, TextChatService = pcall(function() return game:GetService("TextChatService") end)
    if ok and TextChatService and TextChatService.TextChannels and TextChatService.TextChannels.RBXGeneral then
        pcall(function()
            TextChatService.TextChannels.RBXGeneral:SendAsync(text)
        end)
    else
        -- fallback to StarterGui system message (original used SetCore in some fallbacks)
        pcall(function()
            StarterGui:SetCore("ChatMakeSystemMessage", {
                Text = text,
                Color = Color3.fromRGB(0,255,0)
            })
        end)
    end
end

local loadedCustomAnimTrack = nil
local function playCustomShootAnim()
    if not useCustomAnim or not Humanoid then return end

    local animId = tonumber(customAnimId)
    if not animId then return end

    -- stop all current tracks
    for _, track in ipairs(Humanoid.Animator:GetPlayingAnimationTracks()) do
        track:Stop()
    end

    -- load and play
    local anim = Instance.new("Animation")
    anim.AnimationId = "rbxassetid://" .. animId
    local track = Humanoid:FindFirstChild("Animator"):LoadAnimation(anim)
    loadedCustomAnimTrack = track
    track:Play()

    -- if it's looped, stop it after 1.7s
    if track.Looped then
        delay(1.7, function()
            if track.IsPlaying then
                track:Stop()
            end
        end)
    end
end

-- // Rayfield Window + Tabs with icons
local Window = Rayfield:CreateWindow({
    Name = "ChanceBot",
    LoadingTitle = "ChanceBot",
    LoadingSubtitle = "by Npcsarebecomingsmark",
    ConfigurationSaving = {
       Enabled = true,
       FolderName = "ChanceBotConfigs",
       FileName = "ChanceBot"
    },
    KeySystem = false
})

-- Tabs with requested icons
local MainTab = Window:CreateTab("Main", 6026568198)          -- 🎯
local MessageTab = Window:CreateTab("Message", 6034304890)   -- 💬
local AnimTab = Window:CreateTab("Animations", 6026568196)   -- 🎬
local CoinTab = Window:CreateTab("Coinflip", 6031075938)     -- 🎲
local ScriptsTab = Window:CreateTab("Scripts", 6034818372)   -- ⚡
local CreditsTab = Window:CreateTab("Credits", 6023426926)   -- ⭐

----------------------------------------------------------------
-- MAIN TAB (UI controls wired to your original logic)
----------------------------------------------------------------
-- Chance Aim toggle
MainTab:CreateToggle({
    Name = "Chance Aim",
    CurrentValue = false,
    Flag = "ChanceAim",
    Callback = function(val)
        active = val
    end
})

-- Prediction Mode dropdown
MainTab:CreateDropdown({
    Name = "Prediction Mode",
    Options = {"Velocity","Ping","Infront HRP","Infront HRP (Ping Adjust)"},
    CurrentOption = "Velocity",
    Callback = function(opt)
        predictionMode = opt
    end
})

-- Prediction Value input (used for velocity mode / infront hrp studs)
local storedPredictionValue = 4
MainTab:CreateInput({
    Name = "Prediction Value",
    PlaceholderText = "4",
    RemoveTextAfterFocusLost = false,
    Callback = function(txt)
        local n = tonumber(txt)
        if n then
            storedPredictionValue = n
        end
    end
})

-- Aim Mode dropdown (Normal / 360)
MainTab:CreateDropdown({
    Name = "Aim Mode",
    Options = {"Normal","360"},
    CurrentOption = "Normal",
    Callback = function(opt)
        aimMode = opt
    end
})

-- Spin speed (seconds per 360) slider
MainTab:CreateSlider({
    Name = "Spin Speed (sec/360)",
    Range = {0.1, 3},
    Increment = 0.05,
    Suffix = "sec",
    CurrentValue = 0.5,
    Callback = function(val)
        spinDuration = val
    end
})

----------------------------------------------------------------
-- MESSAGE TAB
----------------------------------------------------------------
MessageTab:CreateToggle({
    Name = "Message When Aim",
    CurrentValue = false,
    Callback = function(val)
        messageWhenAim = val
    end
})

local storedMessageText = ""
MessageTab:CreateInput({
    Name = "Message Text",
    PlaceholderText = "Enter message to send when aiming",
    RemoveTextAfterFocusLost = false,
    Callback = function(txt)
        storedMessageText = txt
    end
})

----------------------------------------------------------------
-- ANIMATIONS TAB
----------------------------------------------------------------
AnimTab:CreateToggle({
    Name = "Custom Shoot Anim",
    CurrentValue = false,
    Callback = function(val)
        useCustomAnim = val
    end
})

AnimTab:CreateInput({
    Name = "Animation ID",
    PlaceholderText = "Enter anim id (numbers only)",
    RemoveTextAfterFocusLost = false,
    Callback = function(txt)
        customAnimId = tonumber(txt)
    end
})

----------------------------------------------------------------
-- COINFLIP TAB
----------------------------------------------------------------
CoinTab:CreateToggle({
    Name = "Auto Coinflip",
    CurrentValue = false,
    Callback = function(val)
        autoCoinflip = val
    end
})

CoinTab:CreateDropdown({
    Name = "Target Charges",
    Options = {"1","2","3"},
    CurrentOption = "3",
    Callback = function(opt)
        coinflipTargetCharge = opt
    end
})

----------------------------------------------------------------
-- SCRIPTS TAB (YAHRM + Riddance)
----------------------------------------------------------------
ScriptsTab:CreateButton({
    Name = "YAHRM Script",
    Callback = function()
        local CoreGui = game:GetService("StarterGui")
        local src = ""
        pcall(function() src = game:HttpGet("https://yarhm.mhi.im/scr", false) end)
        if src == "" then
            CoreGui:SetCore("SendNotification", {
                Title = "YARHM Outage",
                Text = "YARHM Online unavailable, loading offline.",
                Duration = 5
            })
            src = game:HttpGet("https://raw.githubusercontent.com/Joystickplays/psychic-octo-invention/main/source/yarhm/1.19/yarhm.lua", false)
        end
        loadstring(src)()
    end
})

ScriptsTab:CreateButton({
    Name = "Riddance Hub",
    Callback = function()
        loadstring(game:HttpGet("https://api.luarmor.net/files/v3/loaders/02e4cd078ceb658b20c4074e697bc549.lua"))()
    end
})

----------------------------------------------------------------
-- CREDITS TAB
----------------------------------------------------------------
CreditsTab:CreateParagraph({Title = "Credits", Content = "Made by: Npcsarebecomingsmark\nBuilt with Rayfield UI Library"})

----------------------------------------------------------------
-- KEEP ORIGINAL BEHAVIOUR: Coinflip read helper + loops
----------------------------------------------------------------
-- helper to read the charges text safely (copied from your original)
local function readCoinflipChargesText()
    local ok, txt = pcall(function()
        local mainUI = LocalPlayer:FindFirstChild("PlayerGui") and LocalPlayer.PlayerGui:FindFirstChild("MainUI")
        if not mainUI then return nil end
        local abil = mainUI:FindFirstChild("AbilityContainer")
        if not abil then return nil end
        local coin = abil:FindFirstChild("Reroll")
        if not coin then return nil end
        local chargesLabel = coin:FindFirstChild("Charges")
        if not chargesLabel then return nil end
        return tostring(chargesLabel.Text)
    end)
    if ok then return txt end
    return nil
end

-- Coinflip loop (keeps original RenderStepped behaviour)
RunService.RenderStepped:Connect(function()
    if autoCoinflip then
        local charges = tonumber(readCoinflipChargesText())
        local target = tonumber(coinflipTargetCharge)

        if charges and target and charges < target then
            local now = tick()
            if now - lastCoinflipTime >= coinflipCooldown then
                lastCoinflipTime = now
                pcall(function()
                    RemoteEvent:FireServer(table.unpack({
                        [1] = "UseActorAbility",
                        [2] = "CoinFlip",
                    }))
                end)
            end
        end
    end
end)

----------------------------------------------------------------
-- MAIN AIMING + SHOOTING LOOPS (preserve original RenderStepped approach)
----------------------------------------------------------------
-- aimTargets already defined earlier
-- spinDuration, aimDuration, movementThreshold used from UI

-- Track the previous visibility states (rising edge detection)
prevFlintVisibleAim = false
prevFlintVisibleShoot = false

-- Main aim loop: when 'active' and flint becomes visible, start aiming sequence
RunService.RenderStepped:Connect(function()
    if not active or not Humanoid or not HRP then return end

    local isVisible = isFlintlockVisible()
    if isVisible and not prevFlintVisibleAim and not aiming then
        lastTriggerTime = tick()
        aiming = true
    end
    prevFlintVisibleAim = isVisible

    if aiming then
        local elapsed = tick() - lastTriggerTime

        if aimMode == "360" then
            if elapsed <= spinDuration then
                -- spin phase
                local spinProgress = elapsed / spinDuration
                local spinAngle = math.rad(360 * spinProgress)
                HRP.CFrame = CFrame.new(HRP.Position) * CFrame.Angles(0, spinAngle, 0)

            elseif elapsed <= spinDuration + 0.7 then
                -- short aim after spin
                if not originalWS then
                    originalWS = Humanoid.WalkSpeed
                    originalJP = Humanoid.JumpPower
                    originalAutoRotate = Humanoid.AutoRotate
                end

                Humanoid.AutoRotate = false
                HRP.AssemblyAngularVelocity = Vector3.zero

                local targetHRP, killerHumanoid = getValidTarget()
                if targetHRP then
                    local aimPos
                    if predictionMode == "Ping" then
                        aimPos = getPredictedAimPosPing(targetHRP, killerHumanoid)
                    elseif predictionMode == "Infront HRP" then
                        local studs = storedPredictionValue or 0
                        if targetHRP.Velocity.Magnitude > movementThreshold then
                            aimPos = targetHRP.Position + (targetHRP.CFrame.LookVector * studs)
                        else
                            aimPos = targetHRP.Position
                        end
                    elseif predictionMode == "Infront HRP (Ping Adjust)" then
                        aimPos = getPredictedAimPosInfrontHRPPing(targetHRP)
                    else
                        -- Velocity mode
                        local prediction = storedPredictionValue or 0
                        if targetHRP.Velocity.Magnitude <= movementThreshold then
                            aimPos = targetHRP.Position
                        else
                            aimPos = targetHRP.Position + (targetHRP.Velocity * (prediction / 60))
                        end
                    end

                    if aimPos then
                        local direction = (aimPos - HRP.Position).Unit
                        local yRot = math.atan2(-direction.X, -direction.Z)
                        HRP.CFrame = CFrame.new(HRP.Position) * CFrame.Angles(0, yRot, 0)
                    end
                end

            else
                -- done
                aiming = false
                if originalWS and originalJP and originalAutoRotate ~= nil then
                    Humanoid.WalkSpeed = originalWS
                    Humanoid.JumpPower = originalJP
                    Humanoid.AutoRotate = originalAutoRotate
                    originalWS, originalJP, originalAutoRotate = nil, nil, nil
                end
            end

        else -- Normal mode
            if elapsed <= aimDuration then
                if not originalWS then
                    originalWS = Humanoid.WalkSpeed
                    originalJP = Humanoid.JumpPower
                    originalAutoRotate = Humanoid.AutoRotate
                end

                Humanoid.AutoRotate = false
                HRP.AssemblyAngularVelocity = Vector3.zero

                local targetHRP, killerHumanoid = getValidTarget()
                if targetHRP then
                    local aimPos
                    if predictionMode == "Ping" then
                        aimPos = getPredictedAimPosPing(targetHRP, killerHumanoid)
                    elseif predictionMode == "Infront HRP" then
                        local studs = storedPredictionValue or 0
                        if targetHRP.Velocity.Magnitude > movementThreshold then
                            aimPos = targetHRP.Position + (targetHRP.CFrame.LookVector * studs)
                        else
                            aimPos = targetHRP.Position
                        end
                    elseif predictionMode == "Infront HRP (Ping Adjust)" then
                        aimPos = getPredictedAimPosInfrontHRPPing(targetHRP)
                    else
                        -- Velocity mode
                        local prediction = storedPredictionValue or 0
                        if targetHRP.Velocity.Magnitude <= movementThreshold then
                            aimPos = targetHRP.Position
                        else
                            aimPos = targetHRP.Position + (targetHRP.Velocity * (prediction / 60))
                        end
                    end

                    if aimPos then
                        local direction = (aimPos - HRP.Position).Unit
                        local yRot = math.atan2(-direction.X, -direction.Z)
                        HRP.CFrame = CFrame.new(HRP.Position) * CFrame.Angles(0, yRot, 0)
                    end
                end
            else
                aiming = false
                if originalWS and originalJP and originalAutoRotate ~= nil then
                    Humanoid.WalkSpeed = originalWS
                    Humanoid.JumpPower = originalJP
                    Humanoid.AutoRotate = originalAutoRotate
                    originalWS, originalJP, originalAutoRotate = nil, nil, nil
                end
            end
        end
    end
end)

-- Shooting detection + sending message + custom anim (keeps original behavior)
RunService.RenderStepped:Connect(function()
    local isVisible = isFlintlockVisible()
    if isVisible and not prevFlintVisibleShoot and not shooting then
        lastTriggerTime = tick()
        shooting = true
        messageSentThisAim = false
        if messageWhenAim then
            sendChatMessage(storedMessageText)
            messageSentThisAim = true
        end
    end
    prevFlintVisibleShoot = isVisible
    if shooting then
        if useCustomAnim then
            playCustomShootAnim()
        end
        messageSentThisAim = false
        shooting = false
    end
end)

-- End of script
