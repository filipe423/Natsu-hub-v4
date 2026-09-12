-- Natsu Hub UI - v2 (com Egg Panel + Steal)
-- Baseado na source "steal a egg ze"

local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local HttpService = game:GetService("HttpService")
local Workspace = game:GetService("Workspace")
local LocalPlayer = Players.LocalPlayer

-- ===== CONFIG GLOBAL =====
local Config = {
    AutoSteal = false,
    StealMode = "Tween",
    StealMinValue = 0,
    StealAreas = {"None"},
    StealRarities = {"None"},
    StealNamesFilter = {"None"},
    TweenSpeedMultiplier = 40,
    IsStealing = false,
    AntiGuardKnockback = true,
    EggPredictUI = false,
    WebhookEnabled = false,
    WebhookURL = "",
}

local SPOOF_CFG = {
    MULTIPLIER = 10, MIN_INPUT = 1, MAX_INPUT = 100, DEF_INPUT = 40,
    MIN_SPEED = 10, MAX_SPEED = 1000, DEF_SPEED = 400,
    HEADROOM = 1.35, CLAMP = 0.15, PADDING = 24,
}

-- ===== SAFE ZONE / CONSTANTES =====
local TWEEN_SAFE_ZONE_POSITION = Vector3.new(545, 71, -365)
local MOVEMENT_ARRIVAL_HOLD = 0.4
local LONG_FRAME_CLAMP = 0.15

-- ===== MÓDULOS DO JOGO =====
local EggState = require(ReplicatedStorage.Client.EggState)
local Assets = require(ReplicatedStorage.Data.Assets).Directory
local AreasData = require(ReplicatedStorage.Data.Areas).Directory
local AreaEggCycle = require(ReplicatedStorage.Shared.Util.AreaEggCycle)
local AreaEggSlotIdentity = require(ReplicatedStorage.Shared.Util.AreaEggSlotIdentity)
local AssetsMod = require(ReplicatedStorage.Data.Assets)
local AssetItems = require(ReplicatedStorage.Shared.Util.AssetItems)
local AssetEarnings = require(ReplicatedStorage.Shared.Util.AssetEarnings)
local PlotState = require(ReplicatedStorage.Client.PlotState)

-- ===== HELPERS =====
local function speedDialStuds()
    local input = math.clamp(math.floor((tonumber(Config.TweenSpeedMultiplier) or 40) + 0.5), 1, 100)
    return input * SPOOF_CFG.MULTIPLIER
end

local function isStealNightBlocked()
    local ok, isNight = pcall(AreaEggCycle.IsNightPhase, workspace:GetServerTimeNow())
    return ok and isNight == true
end

local function computeFirstAreaSlotKey(uid, areaId, nestId)
    local looksFirst = AreaEggSlotIdentity.LooksLikeFirstAreaUid or AreaEggSlotIdentity.IsFirstAreaUid
    local buildKey = AreaEggSlotIdentity.SlotKey or AreaEggSlotIdentity.BuildSlotKey
    local ok, isFirst = pcall(looksFirst, tostring(uid))
    if ok and isFirst and type(areaId) == "string" and type(nestId) == "string" then
        local ok2, key = pcall(buildKey, areaId, nestId)
        if ok2 then return key end
    end
    return nil
end

local function applyKillPartIgnore(char)
    local r = char and char:FindFirstChild("HumanoidRootPart")
    if r then r:SetAttribute("KillPartIgnore", true) end
end

-- ===== HOVER PAD (chão invisível pra voar grounded) =====
local HOVER_PAD_NAME = "__FyyHoverPad"
local hoverPad = nil
local hoverConn = nil, hoverPost = nil

local function purgeOrphans()
    for _, o in ipairs(Workspace:GetChildren()) do
        if o:IsA("BasePart") and o.Name == HOVER_PAD_NAME and o ~= hoverPad then
            pcall(o.Destroy, o)
        end
    end
end

local function syncHoverPad(target)
    if not hoverPad or not hoverPad.Parent then return false end
    local char = LocalPlayer.Character
    local root = char and char:FindFirstChild("HumanoidRootPart")
    local hum = char and char:FindFirstChildOfClass("Humanoid")
    if not root or not hum then return false end
    local pos = typeof(target) == "Vector3" and target or root.Position
    local feetOffset = hum.RigType == Enum.HumanoidRigType.R15
        and (root.Size.Y * 0.5 + math.max(hum.HipHeight, 0.5))
        or (root.Size.Y * 0.5 + 2)
    hoverPad.CFrame = CFrame.new(pos.X, pos.Y - feetOffset - 0.55, pos.Z)
    return true
end

local function startHoverPad()
    if not hoverPad or not hoverPad.Parent then
        purgeOrphans()
        if hoverPad then pcall(hoverPad.Destroy, hoverPad) end
        local p = Instance.new("Part")
        p.Name = HOVER_PAD_NAME
        p.Anchored = true
        p.CanCollide = true
        p.CanQuery = true
        p.CanTouch = false
        p.CastShadow = false
        p.Transparency = 1
        p.Size = Vector3.new(8, 1, 8)
        hoverPad = p
        p.Parent = Workspace
    end
    syncHoverPad()
    if not hoverConn then
        hoverConn = RunService.PreSimulation:Connect(function() syncHoverPad() end)
    end
    if not hoverPost then
        hoverPost = RunService.PostSimulation:Connect(function() syncHoverPad() end)
    end
end

local function stopHoverPad()
    if hoverConn then pcall(hoverConn.Disconnect, hoverConn); hoverConn = nil end
    if hoverPost then pcall(hoverPost.Disconnect, hoverPost); hoverPost = nil end
    if hoverPad then pcall(hoverPad.Destroy, hoverPad); hoverPad = nil end
    purgeOrphans()
end

-- ===== MOVEMENT SPOOF (básico — só pra não ser kickado) =====
local function installMovementSpoof(char, speed) end
local function clearMovementSpoof() end

-- ===== TWEEN MOVE TO =====
local activeMoves = {}
local function TweenMoveTo(hrp, hum, targetCF, isCancelled, holdAfter, earlyFn, flight)
    if not hrp or not hum or typeof(targetCF) ~= "CFrame" then return false end
    if isCancelled and isCancelled() then return false end

    local prev = activeMoves[hrp]
    if prev then prev.cancelled = true; activeMoves[hrp] = nil end

    local char = hrp.Parent
    if not char then return false end
    applyKillPartIgnore(char)
    startHoverPad()

    local camera = workspace.CurrentCamera
    if camera and camera.CameraSubject ~= hum then
        pcall(function() camera.CameraSubject = hum end)
    end

    local origAuto = hum.AutoRotate
    hum.AutoRotate = false

    local move = {cancelled = false}
    activeMoves[hrp] = move

    local liveSpeed = math.clamp(speedDialStuds(), SPOOF_CFG.MIN_SPEED, SPOOF_CFG.MAX_SPEED)
    installMovementSpoof(char, liveSpeed)

    local initDist = (targetCF.Position - hrp.Position).Magnitude
    local timeout = os.clock() + math.max(initDist / liveSpeed, 0.08) * 3 + 8
    local completed = false
    local arrivalStable = nil
    local earlyFired = false

    while not move.cancelled do
        if isCancelled and isCancelled() then break end
        if not hrp.Parent then break end
        if os.clock() >= timeout then break end

        local dt = RunService.Heartbeat:Wait()
        local sdt = math.max(math.min(dt, LONG_FRAME_CLAMP), 1/240)

        liveSpeed = math.clamp(speedDialStuds(), SPOOF_CFG.MIN_SPEED, SPOOF_CFG.MAX_SPEED)
        installMovementSpoof(char, liveSpeed)

        local pos = hrp.Position
        local remaining = targetCF.Position - pos
        local dist = remaining.Magnitude
        local step = liveSpeed * sdt

        if earlyFn and not earlyFired and dist <= 60 then
            earlyFired = true
            task.spawn(earlyFn)
        end

        local look = Vector3.new(hrp.CFrame.LookVector.X, 0, hrp.CFrame.LookVector.Z)
        local dir = remaining.Magnitude > 0.001 and remaining.Unit
            or (look.Magnitude > 0.001 and look.Unit or Vector3.new(0,0,-1))

        if dist <= math.max(step, 0.05) then
            syncHoverPad(targetCF.Position)
            hrp.CFrame = CFrame.lookAt(targetCF.Position, targetCF.Position + dir)
            hrp.AssemblyLinearVelocity = Vector3.zero
            hrp.AssemblyAngularVelocity = Vector3.zero
            arrivalStable = arrivalStable or os.clock()
            if os.clock() - arrivalStable >= MOVEMENT_ARRIVAL_HOLD then
                completed = true
                break
            end
        else
            arrivalStable = nil
            local actualStep = math.min(step, dist)
            local nextPos = pos + remaining.Unit * actualStep
            local vel = (nextPos - pos) / sdt
            syncHoverPad(nextPos)
            hrp.CFrame = CFrame.lookAt(nextPos, nextPos + dir)
            hrp.AssemblyLinearVelocity = vel
        end
    end

    if hum.Parent then hum.AutoRotate = origAuto end
    hrp.AssemblyLinearVelocity = Vector3.zero
    hrp.AssemblyAngularVelocity = Vector3.zero
    if not holdAfter then stopHoverPad() end
    return completed and not move.cancelled
end

-- ===== INSTANT CARRY (steal via teleporte) =====
local IC_SAFE_ZONE_POS = Vector3.new(545, 71, -365)
local IC_SAFE_ZONE_RADIUS = 30
local IC_SAFE_ZONE_VERTICAL_TOL = 18
local IC_MAX_TELEPORT_STUDS = 10000

local function icIsInSafeArea(root)
    if not root then return false end
    local d = root.Position - IC_SAFE_ZONE_POS
    return Vector2.new(d.X, d.Z).Magnitude <= IC_SAFE_ZONE_RADIUS
        and math.abs(d.Y) <= IC_SAFE_ZONE_VERTICAL_TOL
end

local function icTeleportRoot(root, dest)
    if not root or typeof(dest) ~= "CFrame" then return false end
    local ok = pcall(function()
        root.AssemblyLinearVelocity = Vector3.zero
        root.AssemblyAngularVelocity = Vector3.zero
        root.CFrame = dest
    end)
    if ok then syncHoverPad(dest.Position) end
    return ok
end

local function icTeleportHops(root, dest)
    if not root or typeof(dest) ~= "CFrame" then return false end
    local from = root.Position
    local to = dest.Position
    local dist = (to - from).Magnitude
    local hops = math.max(1, math.ceil(dist / IC_MAX_TELEPORT_STUDS))
    for i = 1, hops do
        local t = i / hops
        local point = from:Lerp(to, t)
        if not icTeleportRoot(root, CFrame.new(point) * dest.Rotation) then return false end
        if i < hops then task.wait() end
    end
    return true
end

local function icRequestCarry(uid, slotKey)
    local remotes = require(ReplicatedStorage.Shared.Remotes)
    local payload = { Uid = uid }
    if slotKey then payload.FirstAreaSlotKey = slotKey end
    return remotes.EggWorld.AskFieldEggCarry:InvokeServer(payload) == true
end

-- ===== AUTO STEAL LOOP (versão simplificada do teu) =====
local lastStealAreaByCategory = {}
local recentlyClaimed = {}

local function rarityOfRecord(record)
    local cat = record.AssetCategory
    local categoryData = cat and Assets[cat]
    local rarityData = categoryData and categoryData.Rarity
    if not rarityData then return "Unknown" end
    return rarityData.DisplayName or rarityData._id or "Unknown"
end

local RarityWeights = {
    Common=1, Uncommon=2, Rare=3, Epic=4, Legendary=5,
    Mythic=6, Cosmic=7, Secret=8, Eternal=9, Divine=10,
}

local function pickTarget(snapshot)
    local root = LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart")
    local best, bestW, bestD = nil, -1, math.huge

    local namesFilter = Config.StealNamesFilter or {}
    local areasFilter = Config.StealAreas or {}
    local rarFilter = Config.StealRarities or {}

    local hasName = #namesFilter > 0 and not table.find(namesFilter, "None")
    local hasArea = #areasFilter > 0 and not table.find(areasFilter, "None")
    local hasRar = #rarFilter > 0 and not table.find(rarFilter, "None")

    for _, rec in pairs((snapshot and snapshot.Records) or {}) do
        if recentlyClaimed[rec.Uid] then continue end
        if rec.State ~= "Slot" and rec.State ~= "Dropped" then continue end
        if not rec.BottomCFrame then continue end

        local rarity = rarityOfRecord(rec)
        local weight = RarityWeights[rarity] or 0
        local area = rec.AreaId
        local cat = rec.AssetCategory

        local areaOK = (not hasArea) or table.find(areasFilter, area) ~= nil
        local rarOK = (not hasRar) or table.find(rarFilter, rarity) ~= nil
        local nameOK = true
        if hasName then
            local catData = cat and Assets[cat]
            local disp = catData and catData.DisplayName
            nameOK = (table.find(namesFilter, cat) ~= nil) or (disp and table.find(namesFilter, disp) ~= nil)
        end
        local valOK = Config.StealMinValue <= 0
        if not valOK then
            local ok, v = pcall(function()
                return AssetEarnings.MutationOnlyRatePerSecond({
                    Category = cat, Scale = rec.AssetScale or 1,
                    Mutations = rec.Mutations or {}, BaseMutation = rec.BaseMutation,
                    EyeColor = rec.AssetEyeColor, ColorSeed = rec.AssetColorSeed,
                    ColorIndex = rec.AssetColorIndex, Gender = rec.Gender,
                    Personality = rec.Personality or "Normal",
                    HasBeenFirstPlaced = rec.HasBeenFirstPlaced == true,
                })
            end)
            valOK = ok and v and v > Config.StealMinValue
        end

        if areaOK and rarOK and nameOK and valOK then
            local d = root and (rec.BottomCFrame.Position - root.Position).Magnitude or 0
            if weight > bestW or (weight == bestW and d < bestD) then
                best, bestW, bestD = rec, weight, d
            end
        end
    end
    return best
end

local autoStealRunning = false
local function StartAutoSteal()
    if autoStealRunning then return end
    autoStealRunning = true

    task.spawn(function()
        while Config.AutoSteal do
            local ok, err = pcall(function()
                if isStealNightBlocked() then
                    Config.IsStealing = false
                    task.wait(0.5)
                    return
                end

                local now = os.clock()
                for uid, t in pairs(recentlyClaimed) do
                    if now - t > 5 then recentlyClaimed[uid] = nil end
                end

                local char = LocalPlayer.Character
                local hrp = char and char:FindFirstChild("HumanoidRootPart")
                local hum = char and char:FindFirstChildOfClass("Humanoid")
                if not hrp or not hum then task.wait(0.5); return end

                local snap = EggState.GetAreaEggSnapshot()

                -- já carregando? vai pra safe zone
                local carrying = nil
                for _, rec in pairs((snap and snap.Records) or {}) do
                    if rec.State == "Carried" and rec.CarrierUserId == LocalPlayer.UserId then
                        carrying = rec
                        break
                    end
                end

                if carrying then
                    Config.IsStealing = true
                    if not icIsInSafeArea(hrp) then
                        local safe = CFrame.new(IC_SAFE_ZONE_POS) * hrp.CFrame.Rotation
                        TweenMoveTo(hrp, hum, safe, function() return not Config.AutoSteal end, false, nil, true)
                    end
                    task.wait(0.2)
                    return
                end

                local target = pickTarget(snap)
                if not target then
                    if not icIsInSafeArea(hrp) then
                        local safe = CFrame.new(IC_SAFE_ZONE_POS) * hrp.CFrame.Rotation
                        TweenMoveTo(hrp, hum, safe, function() return not Config.AutoSteal end, false, nil, true)
                    end
                    task.wait(0.5)
                    return
                end

                Config.IsStealing = true
                local rarity = rarityOfRecord(target)
                lastStealAreaByCategory[target.AssetCategory] = target.AreaId

                local function cancelled() return not Config.AutoSteal or isStealNightBlocked() end

                local eggCF = target.BottomCFrame * CFrame.new(0, 3, 0)
                local slotKey = computeFirstAreaSlotKey(target.Uid, target.AreaId, target.NestId)

                if Config.StealMode == "Tween" then
                    TweenMoveTo(hrp, hum, eggCF, cancelled, true, nil, true)
                else
                    icTeleportHops(hrp, eggCF)
                end

                if cancelled() then Config.IsStealing = false; return end

                local confirmed = false
                local deadline = os.clock() + 2
                while not cancelled() and not confirmed and os.clock() < deadline do
                    local rec = EggState.GetAreaEggRecord(target.Uid)
                    if rec and rec.State == "Carried" and rec.CarrierUserId == LocalPlayer.UserId then
                        confirmed = true; break
                    end
                    pcall(icRequestCarry, target.Uid, slotKey)
                    task.wait(0.2)
                end

                if confirmed and not cancelled() then
                    local safe = CFrame.new(IC_SAFE_ZONE_POS) * hrp.CFrame.Rotation
                    if Config.StealMode == "Tween" then
                        TweenMoveTo(hrp, hum, safe, cancelled, false, nil, true)
                    else
                        icTeleportHops(hrp, safe)
                    end

                    local depDeadline = os.clock() + 4
                    while not cancelled() and os.clock() < depDeadline do
                        local rec = EggState.GetAreaEggRecord(target.Uid)
                        if not rec or rec.State == "Claimed" then break end
                        task.wait(0.1)
                    end
                    recentlyClaimed[target.Uid] = os.clock()
                end

                Config.IsStealing = false
                task.wait(0.05)
            end)

            if not ok then
                warn("[Natsu Steal] erro:", err)
                Config.IsStealing = false
                task.wait(0.5)
            end
        end
        autoStealRunning = false
        stopHoverPad()
    end)
end

-- ===== EGG PANEL (lista de eggs) =====
local EGG_PANEL_THEME = {
    BG = Color3.fromRGB(6, 14, 30),
    Surface = Color3.fromRGB(14, 26, 50),
    Input = Color3.fromRGB(10, 22, 44),
    Border = Color3.fromRGB(40, 70, 120),
    Accent = Color3.fromRGB(0, 122, 255),
    AccentSoft = Color3.fromRGB(150, 195, 255),
    Text = Color3.fromRGB(228, 236, 250),
    TextSec = Color3.fromRGB(155, 180, 215),
    TextMut = Color3.fromRGB(120, 148, 190),
}

local MUTATION_COLORS = {
    Rainbow = Color3.fromRGB(255, 60, 255),
    Golden = Color3.fromRGB(255, 234, 0),
    Silver = Color3.fromRGB(220, 220, 220),
}

local function buildRarityCatalog()
    local seen, list = {}, {}
    for _, a in pairs(Assets) do
        local r = a.Rarity
        if r and r.DisplayName and not seen[r.DisplayName] then
            seen[r.DisplayName] = true
            table.insert(list, {name = r.DisplayName, color = r.Color or Color3.new(1,1,1), order = r.RarityNumber or 0})
        end
    end
    table.sort(list, function(a,b) return a.order > b.order end)
    return list
end
local RarityCatalog = buildRarityCatalog()
local RarityColorByName = {Any = Color3.new(1,1,1)}
for _, r in ipairs(RarityCatalog) do RarityColorByName[r.name] = r.color end

local function getEggLogEntries()
    local entries = {}
    local snap = EggState.ReadFieldEggs()
    if snap and snap.Records then
        for _, rec in pairs(snap.Records) do
            if rec.State == "Slot" or rec.State == "Dropped" then
                local a = Assets[rec.AssetCategory]
                if a and a.Egg then
                    local areaData = AreasData[rec.AreaId]
                    local mutText = nil
                    if rec.Mutations and #rec.Mutations > 0 then
                        mutText = table.concat(rec.Mutations, " + ")
                    end
                    table.insert(entries, {
                        uid = rec.Uid,
                        petName = a.DisplayName,
                        eggName = a.Egg.DisplayName,
                        eggIcon = a.Egg.Icon,
                        weightKg = a.Egg.WeightKg,
                        rarityName = (a.Rarity and a.Rarity.DisplayName) or "Common",
                        rarityOrder = (a.Rarity and a.Rarity.RarityNumber) or 0,
                        mutationText = mutText,
                        area = (areaData and areaData.DisplayName) or rec.AreaId,
                    })
                end
            end
        end
    end
    table.sort(entries, function(a,b)
        if a.rarityOrder ~= b.rarityOrder then return a.rarityOrder > b.rarityOrder end
        return a.petName < b.petName
    end)
    return entries
end

local EggPanel = { UI = nil, State = { Search = "", Rarity = "Any" }, Dirty = true }

local function gotoAndClaim(uid)
    task.spawn(function()
        local prev = Config.TweenSpeedMultiplier
        Config.TweenSpeedMultiplier = 8 -- 80 studs/s
        pcall(function()
            local snap = EggState.ReadFieldEggs()
            local rec
            for _, r in pairs((snap and snap.Records) or {}) do
                if r.Uid == uid then rec = r; break end
            end
            if not rec then error("Egg sumiu") end

            local char = LocalPlayer.Character
            local hrp = char and char:FindFirstChild("HumanoidRootPart")
            local hum = char and char:FindFirstChildOfClass("Humanoid")
            if not hrp or not hum then error("Sem char") end

            local dest = CFrame.new(rec.BottomCFrame.Position + Vector3.new(0, 3.5, 0))
            TweenMoveTo(hrp, hum, dest, nil, false)
            local slotKey = computeFirstAreaSlotKey(rec.Uid, rec.AreaId, rec.NestId)
            pcall(icRequestCarry, uid, slotKey)
        end)
        Config.TweenSpeedMultiplier = prev
    end)
end

local function createEggCard(order, data)
    local theme = EGG_PANEL_THEME
    local card = Instance.new("Frame")
    card.Size = UDim2.new(1, 0, 0, 66)
    card.BackgroundColor3 = theme.Surface
    card.BackgroundTransparency = 0.08
    card.LayoutOrder = order
    card.Parent = EggPanel.UI.ListScroll
    Instance.new("UICorner", card).CornerRadius = UDim.new(0, 8)

    local stroke = Instance.new("UIStroke", card)
    stroke.Color = theme.Border
    stroke.Thickness = 1
    stroke.Transparency = 0.38

    local bar = Instance.new("Frame", card)
    bar.Size = UDim2.new(0, 4, 1, -12)
    bar.Position = UDim2.new(0, 0, 0, 6)
    bar.BackgroundColor3 = data.rarityColor or Color3.new(1,1,1)
    bar.BorderSizePixel = 0
    Instance.new("UICorner", bar).CornerRadius = UDim.new(1, 0)

    local icon = Instance.new("ImageLabel", card)
    icon.Size = UDim2.new(0, 40, 0, 40)
    icon.Position = UDim2.new(0, 12, 0.5, -20)
    icon.BackgroundColor3 = theme.Input
    icon.Image = data.icon or ""
    Instance.new("UICorner", icon).CornerRadius = UDim.new(0, 6)

    local title = Instance.new("TextLabel", card)
    title.Size = UDim2.new(1, -170, 0, 15)
    title.Position = UDim2.new(0, 60, 0, 4)
    title.BackgroundTransparency = 1
    title.TextXAlignment = Enum.TextXAlignment.Left
    title.Font = Enum.Font.GothamBold
    title.TextSize = 13
    title.TextColor3 = theme.Text
    title.TextTruncate = Enum.TextTruncate.AtEnd
    title.Text = data.title or ""

    local sub = Instance.new("TextLabel", card)
    sub.Size = UDim2.new(1, -170, 0, 13)
    sub.Position = UDim2.new(0, 60, 0, 19)
    sub.BackgroundTransparency = 1
    sub.TextXAlignment = Enum.TextXAlignment.Left
    sub.Font = Enum.Font.Gotham
    sub.TextSize = 10
    sub.TextColor3 = theme.TextSec
    sub.Text = data.subtitle or ""

    local meta = Instance.new("TextLabel", card)
    meta.Size = UDim2.new(1, -170, 0, 13)
    meta.Position = UDim2.new(0, 60, 0, 33)
    meta.BackgroundTransparency = 1
    meta.TextXAlignment = Enum.TextXAlignment.Left
    meta.Font = Enum.Font.Gotham
    meta.TextSize = 10
    meta.TextColor3 = data.metaColor or theme.TextSec
    meta.Text = data.meta or ""

    local rightTop = Instance.new("TextLabel", card)
    rightTop.Size = UDim2.new(0, 100, 0, 16)
    rightTop.Position = UDim2.new(1, -108, 0, 8)
    rightTop.BackgroundTransparency = 1
    rightTop.TextXAlignment = Enum.TextXAlignment.Right
    rightTop.Font = Enum.Font.GothamBold
    rightTop.TextSize = 12
    rightTop.TextColor3 = theme.AccentSoft
    rightTop.Text = data.area or ""

    local rightBot = Instance.new("TextLabel", card)
    rightBot.Size = UDim2.new(0, 100, 0, 14)
    rightBot.Position = UDim2.new(1, -108, 0, 26)
    rightBot.BackgroundTransparency = 1
    rightBot.TextXAlignment = Enum.TextXAlignment.Right
    rightBot.Font = Enum.Font.Gotham
    rightBot.TextSize = 10
    rightBot.TextColor3 = theme.TextMut
    rightBot.Text = string.format("%s kg", tostring(data.weight or 0))

    if data.uid then
        local goto = Instance.new("TextButton", card)
        goto.Size = UDim2.new(0, 62, 0, 18)
        goto.Position = UDim2.new(1, -70, 0, 42)
        goto.BackgroundColor3 = theme.Accent
        goto.Font = Enum.Font.GothamBold
        goto.TextSize = 11
        goto.Text = "Goto"
        goto.TextColor3 = Color3.fromRGB(6, 14, 30)
        Instance.new("UICorner", goto).CornerRadius = UDim.new(0, 6)
        goto.MouseButton1Click:Connect(function() gotoAndClaim(data.uid) end)
    end
end

local function renderEggLogs()
    if not EggPanel.UI then return end
    for _, c in ipairs(EggPanel.UI.ListScroll:GetChildren()) do
        if c:IsA("Frame") then c:Destroy() end
    end
    local entries = getEggLogEntries()
    local shown = 0
    for _, e in ipairs(entries) do
        local rarOK = EggPanel.State.Rarity == "Any" or e.rarityName == EggPanel.State.Rarity
        local searchOK = EggPanel.State.Search == ""
            or string.find(string.lower(e.petName), string.lower(EggPanel.State.Search), 1, true)
            or string.find(string.lower(e.eggName), string.lower(EggPanel.State.Search), 1, true)
        if rarOK and searchOK then
            shown = shown + 1
            local rColor = RarityColorByName[e.rarityName] or Color3.new(1,1,1)
            local mut = e.mutationText and ("Mut: " .. e.mutationText) or "Sem Mutação"
            createEggCard(shown, {
                icon = e.eggIcon, title = e.eggName, subtitle = e.petName,
                rarityColor = rColor, meta = e.rarityName .. " • " .. mut,
                metaColor = rColor, area = e.area, weight = e.weightKg, uid = e.uid,
            })
        end
    end
    if EggPanel.UI.CountLabel then
        EggPanel.UI.CountLabel.Text = shown .. " egg(s) no campo"
    end
end

local function makeEggPanel()
    local theme = EGG_PANEL_THEME
    local gui = Instance.new("ScreenGui")
    gui.Name = "NatsuEggPanel"
    gui.ResetOnSpawn = false
    gui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
    gui.Parent = game.CoreGui

    local frame = Instance.new("Frame", gui)
    frame.Size = UDim2.new(0, 560, 0, 420)
    frame.Position = UDim2.new(0.5, -280, 0.5, -210)
    frame.BackgroundColor3 = theme.BG
    frame.BorderSizePixel = 0
    frame.Active = true
    frame.Draggable = true
    Instance.new("UICorner", frame).CornerRadius = UDim.new(0, 8)
    local stroke = Instance.new("UIStroke", frame)
    stroke.Color = theme.Border
    stroke.Thickness = 1.2
    stroke.Transparency = 0.18

    local top = Instance.new("Frame", frame)
    top.Size = UDim2.new(1, 0, 0, 36)
    top.BackgroundColor3 = theme.BG
    top.BorderSizePixel = 0
    Instance.new("UICorner", top).CornerRadius = UDim.new(0, 8)

    local title = Instance.new("TextLabel", top)
    title.Size = UDim2.new(1, -60, 1, 0)
    title.Position = UDim2.new(0, 12, 0, 0)
    title.BackgroundTransparency = 1
    title.Text = "🔥 Natsu | Egg Panel"
    title.TextColor3 = theme.Text
    title.Font = Enum.Font.GothamBold
    title.TextSize = 13
    title.TextXAlignment = Enum.TextXAlignment.Left

    local close = Instance.new("TextButton", top)
    close.Size = UDim2.new(0, 24, 0, 24)
    close.Position = UDim2.new(1, -30, 0.5, -12)
    close.BackgroundColor3 = theme.Surface
    close.Text = "×"
    close.TextColor3 = theme.Text
    close.Font = Enum.Font.GothamBold
    close.TextSize = 14
    Instance.new("UICorner", close).CornerRadius = UDim.new(0, 6)
    close.MouseButton1Click:Connect(function() gui.Enabled = false end)

    local search = Instance.new("TextBox", frame)
    search.Size = UDim2.new(1, -80, 0, 26)
    search.Position = UDim2.new(0, 8, 0, 44)
    search.BackgroundColor3 = theme.Input
    search.TextColor3 = theme.Text
    search.PlaceholderText = "Buscar egg/pet..."
    search.PlaceholderColor3 = theme.TextMut
    search.Font = Enum.Font.Gotham
    search.TextSize = 12
    search.Text = ""
    search.ClearTextOnFocus = false
    search.TextXAlignment = Enum.TextXAlignment.Left
    Instance.new("UICorner", search).CornerRadius = UDim.new(0, 6)
    Instance.new("UIPadding", search).PaddingLeft = UDim.new(0, 8)

    local clear = Instance.new("TextButton", frame)
    clear.Size = UDim2.new(0, 62, 0, 26)
    clear.Position = UDim2.new(1, -70, 0, 44)
    clear.BackgroundColor3 = theme.Input
    clear.Text = "Limpar"
    clear.TextColor3 = theme.Text
    clear.Font = Enum.Font.GothamMedium
    clear.TextSize = 11
    Instance.new("UICorner", clear).CornerRadius = UDim.new(0, 6)
    clear.MouseButton1Click:Connect(function() search.Text = "" end)

    local chipScroll = Instance.new("ScrollingFrame", frame)
    chipScroll.Size = UDim2.new(1, -16, 0, 34)
    chipScroll.Position = UDim2.new(0, 8, 0, 76)
    chipScroll.BackgroundTransparency = 1
    chipScroll.BorderSizePixel = 0
    chipScroll.ScrollBarThickness = 0
    chipScroll.ScrollingDirection = Enum.ScrollingDirection.X
    chipScroll.CanvasSize = UDim2.new(0, 0, 0, 0)
    chipScroll.AutomaticCanvasSize = Enum.AutomaticSize.X
    local chipLayout = Instance.new("UIListLayout", chipScroll)
    chipLayout.FillDirection = Enum.FillDirection.Horizontal
    chipLayout.VerticalAlignment = Enum.VerticalAlignment.Center
    chipLayout.Padding = UDim.new(0, 6)

    local listScroll = Instance.new("ScrollingFrame", frame)
    listScroll.Size = UDim2.new(1, -16, 1, -126)
    listScroll.Position = UDim2.new(0, 8, 0, 118)
    listScroll.BackgroundTransparency = 1
    listScroll.BorderSizePixel = 0
    listScroll.ScrollBarThickness = 3
    listScroll.ScrollBarImageColor3 = theme.Border
    listScroll.CanvasSize = UDim2.new(0, 0, 0, 0)
    listScroll.AutomaticCanvasSize = Enum.AutomaticSize.Y
    local listLayout = Instance.new("UIListLayout", listScroll)
    listLayout.Padding = UDim.new(0, 6)
    listLayout.SortOrder = Enum.SortOrder.LayoutOrder

    local count = Instance.new("TextLabel", listScroll)
    count.Size = UDim2.new(1, 0, 0, 14)
    count.LayoutOrder = -1
    count.BackgroundTransparency = 1
    count.TextXAlignment = Enum.TextXAlignment.Left
    count.Font = Enum.Font.GothamMedium
    count.TextSize = 10
    count.TextColor3 = theme.TextSec
    count.Text = ""

    local function makeChip(name)
        local b = Instance.new("TextButton", chipScroll)
        b.Size = UDim2.new(0, math.max(50, #name * 7 + 24), 0, 24)
        b.BackgroundColor3 = theme.Surface
        b.Text = name
        b.TextColor3 = theme.TextSec
        b.Font = Enum.Font.GothamBold
        b.TextSize = 11
        Instance.new("UICorner", b).CornerRadius = UDim.new(1, 0)
        local s = Instance.new("UIStroke", b)
        s.Color = theme.Border
        s.Transparency = 0.4
        b.MouseButton1Click:Connect(function()
            EggPanel.State.Rarity = name
            for _, c in ipairs(chipScroll:GetChildren()) do
                if c:IsA("TextButton") then
                    if c.Text == name then
                        c.BackgroundColor3 = RarityColorByName[c.Text] or theme.Accent
                        c.TextColor3 = Color3.fromRGB(10,10,15)
                    else
                        c.BackgroundColor3 = theme.Surface
                        c.TextColor3 = theme.TextSec
                    end
                end
            end
            renderEggLogs()
        end)
        return b
    end
    makeChip("Any")
    for _, r in ipairs(RarityCatalog) do makeChip(r.name) end
    for _, c in ipairs(chipScroll:GetChildren()) do
        if c:IsA("TextButton") and c.Text == "Any" then
            c.BackgroundColor3 = theme.Accent
            c.TextColor3 = Color3.fromRGB(10,10,15)
        end
    end

    search:GetPropertyChangedSignal("Text"):Connect(function()
        EggPanel.State.Search = search.Text
        renderEggLogs()
    end)

    EggPanel.UI = { Gui = gui, Frame = frame, ListScroll = listScroll, CountLabel = count }
    return gui
end

local eggPanelLoopRunning = false
local function startEggPanelLoop()
    if eggPanelLoopRunning then return end
    eggPanelLoopRunning = true
    task.spawn(function()
        while Config.EggPredictUI do
            pcall(function()
                if EggPanel.UI then
                    renderEggLogs()
                end
            end)
            task.wait(2)
        end
        eggPanelLoopRunning = false
    end)
end

EggState.FieldShifted:Connect(function() EggPanel.Dirty = true end)
EggState.FieldGone:Connect(function() EggPanel.Dirty = true end)

-- ====== NATSU HUB UI ======
local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "NatsuHub"
ScreenGui.ResetOnSpawn = false
ScreenGui.Parent = game.CoreGui

local MainFrame = Instance.new("Frame", ScreenGui)
MainFrame.Size = UDim2.new(0, 500, 0, 400)
MainFrame.Position = UDim2.new(0.5, -250, 0.5, -200)
MainFrame.BackgroundColor3 = Color3.fromRGB(15, 25, 45)
MainFrame.BorderSizePixel = 0
MainFrame.Active = true
MainFrame.Draggable = true
Instance.new("UICorner", MainFrame).CornerRadius = UDim.new(0, 8)
local mStroke = Instance.new("UIStroke", MainFrame)
mStroke.Color = Color3.fromRGB(0, 120, 255)
mStroke.Thickness = 2

local TopBar = Instance.new("Frame", MainFrame)
TopBar.Size = UDim2.new(1, 0, 0, 35)
TopBar.BackgroundColor3 = Color3.fromRGB(0, 90, 200)
TopBar.BorderSizePixel = 0
Instance.new("UICorner", TopBar).CornerRadius = UDim.new(0, 8)
local topFix = Instance.new("Frame", TopBar)
topFix.BackgroundColor3 = Color3.fromRGB(0, 90, 200)
topFix.BorderSizePixel = 0
topFix.Position = UDim2.new(0, 0, 0.5, 0)
topFix.Size = UDim2.new(1, 0, 0.5, 0)

local Title = Instance.new("TextLabel", TopBar)
Title.BackgroundTransparency = 1
Title.Position = UDim2.new(0, 10, 0, 0)
Title.Size = UDim2.new(0.6, 0, 1, 0)
Title.Font = Enum.Font.GothamBold
Title.Text = "🔥 Natsu Hub"
Title.TextColor3 = Color3.new(1,1,1)
Title.TextSize = 18
Title.TextXAlignment = Enum.TextXAlignment.Left

local CloseBtn = Instance.new("TextButton", TopBar)
CloseBtn.BackgroundColor3 = Color3.fromRGB(200, 40, 40)
CloseBtn.Position = UDim2.new(1, -35, 0, 7)
CloseBtn.Size = UDim2.new(0, 22, 0, 22)
CloseBtn.Font = Enum.Font.GothamBold
CloseBtn.Text = "X"
CloseBtn.TextColor3 = Color3.new(1,1,1)
CloseBtn.TextSize = 12
Instance.new("UICorner", CloseBtn).CornerRadius = UDim.new(0, 4)
CloseBtn.MouseButton1Click:Connect(function() ScreenGui:Destroy() end)

local MinBtn = Instance.new("TextButton", TopBar)
MinBtn.BackgroundColor3 = Color3.fromRGB(200, 150, 0)
MinBtn.Position = UDim2.new(1, -62, 0, 7)
MinBtn.Size = UDim2.new(0, 22, 0, 22)
MinBtn.Font = Enum.Font.GothamBold
MinBtn.Text = "-"
MinBtn.TextColor3 = Color3.new(1,1,1)
MinBtn.TextSize = 14
Instance.new("UICorner", MinBtn).CornerRadius = UDim.new(0, 4)

local TabHolder = Instance.new("Frame", MainFrame)
TabHolder.BackgroundColor3 = Color3.fromRGB(10, 20, 38)
TabHolder.BorderSizePixel = 0
TabHolder.Position = UDim2.new(0, 0, 0, 35)
TabHolder.Size = UDim2.new(0, 100, 1, -35)

local MainTab = Instance.new("TextButton", TabHolder)
MainTab.BackgroundColor3 = Color3.fromRGB(0, 120, 255)
MainTab.Position = UDim2.new(0, 5, 0, 8)
MainTab.Size = UDim2.new(0, 90, 0, 35)
MainTab.Font = Enum.Font.GothamBold
MainTab.Text = "Main"
MainTab.TextColor3 = Color3.new(1,1,1)
MainTab.TextSize = 14
Instance.new("UICorner", MainTab).CornerRadius = UDim.new(0, 6)

local MiscTab = Instance.new("TextButton", TabHolder)
MiscTab.BackgroundColor3 = Color3.fromRGB(25, 40, 70)
MiscTab.Position = UDim2.new(0, 5, 0, 48)
MiscTab.Size = UDim2.new(0, 90, 0, 35)
MiscTab.Font = Enum.Font.GothamBold
MiscTab.Text = "Misc"
MiscTab.TextColor3 = Color3.fromRGB(180, 200, 230)
MiscTab.TextSize = 14
Instance.new("UICorner", MiscTab).CornerRadius = UDim.new(0, 6)

local Content = Instance.new("Frame", MainFrame)
Content.BackgroundTransparency = 1
Content.Position = UDim2.new(0, 100, 0, 35)
Content.Size = UDim2.new(1, -100, 1, -35)

local MainContent = Instance.new("Frame", Content)
MainContent.BackgroundTransparency = 1
MainContent.Size = UDim2.new(1, 0, 1, 0)
local MiscContent = Instance.new("Frame", Content)
MiscContent.BackgroundTransparency = 1
MiscContent.Size = UDim2.new(1, 0, 1, 0)
MiscContent.Visible = false

local function mkBtn(parent, name, text, y)
    local b = Instance.new("TextButton", parent)
    b.Name = name
    b.BackgroundColor3 = Color3.fromRGB(0, 90, 200)
    b.BorderSizePixel = 0
    b.Position = UDim2.new(0, 15, 0, y)
    b.Size = UDim2.new(1, -30, 0, 38)
    b.Font = Enum.Font.GothamSemibold
    b.Text = text
    b.TextColor3 = Color3.new(1,1,1)
    b.TextSize = 14
    Instance.new("UICorner", b).CornerRadius = UDim.new(0, 6)
    return b
end

local EggsBtn = mkBtn(MainContent, "EggsBtn", "🥚 Eggs (Open Panel)", 15)
local TpBtn = mkBtn(MainContent, "TpBtn", "📍 TP", 63)
local TeleguiadoBtn = mkBtn(MainContent, "TeleguiadoBtn", "🎯 Teleguiado", 111)
local SpeedBtn = mkBtn(MainContent, "SpeedBtn", "⚡ Speed", 159)

local StealToggle = mkBtn(MainContent, "StealToggle", "🕵 Auto Steal: OFF", 207)
StealToggle.BackgroundColor3 = Color3.fromRGB(120, 30, 30)

local SpeedInput = Instance.new("TextBox", MainContent)
SpeedInput.BackgroundColor3 = Color3.fromRGB(25, 40, 70)
SpeedInput.Position = UDim2.new(0, 15, 0, 253)
SpeedInput.Size = UDim2.new(1, -30, 0, 38)
SpeedInput.Font = Enum.Font.Gotham
SpeedInput.PlaceholderText = "Digite a velocidade..."
SpeedInput.Text = ""
SpeedInput.TextColor3 = Color3.new(1,1,1)
SpeedInput.PlaceholderColor3 = Color3.fromRGB(150, 170, 200)
SpeedInput.TextSize = 14
SpeedInput.Visible = false
Instance.new("UICorner", SpeedInput).CornerRadius = UDim.new(0, 6)

local SpeedConfirm = Instance.new("TextButton", MainContent)
SpeedConfirm.BackgroundColor3 = Color3.fromRGB(0, 180, 100)
SpeedConfirm.Position = UDim2.new(0, 15, 0, 297)
SpeedConfirm.Size = UDim2.new(1, -30, 0, 35)
SpeedConfirm.Font = Enum.Font.GothamBold
SpeedConfirm.Text = "✔ Confirmar Speed"
SpeedConfirm.TextColor3 = Color3.new(1,1,1)
SpeedConfirm.TextSize = 14
SpeedConfirm.Visible = false
Instance.new("UICorner", SpeedConfirm).CornerRadius = UDim.new(0, 6)

local speedOpen = false
SpeedBtn.MouseButton1Click:Connect(function()
    speedOpen = not speedOpen
    SpeedInput.Visible = speedOpen
    SpeedConfirm.Visible = speedOpen
end)

SpeedConfirm.MouseButton1Click:Connect(function()
    local v = tonumber(SpeedInput.Text)
    if v then
        Config.TweenSpeedMultiplier = math.clamp(v, 1, 100)
        SpeedBtn.Text = "⚡ Speed (lvl " .. Config.TweenSpeedMultiplier .. ")"
        SpeedInput.Visible = false
        SpeedConfirm.Visible = false
        speedOpen = false
    end
end)

EggsBtn.MouseButton1Click:Connect(function()
    Config.EggPredictUI = true
    if not EggPanel.UI then makeEggPanel() end
    EggPanel.UI.Gui.Enabled = true
    pcall(EggState.SyncFieldEggs)
    renderEggLogs()
    startEggPanelLoop()
end)

StealToggle.MouseButton1Click:Connect(function()
    Config.AutoSteal = not Config.AutoSteal
    if Config.AutoSteal then
        StealToggle.Text = "🕵 Auto Steal: ON"
        StealToggle.BackgroundColor3 = Color3.fromRGB(30, 140, 60)
        StartAutoSteal()
    else
        StealToggle.Text = "🕵 Auto Steal: OFF"
        StealToggle.BackgroundColor3 = Color3.fromRGB(120, 30, 30)
    end
end)

TpBtn.MouseButton1Click:Connect(function()
    local char = LocalPlayer.Character
    local hrp = char and char:FindFirstChild("HumanoidRootPart")
    if hrp then
        hrp.CFrame = CFrame.new(TWEEN_SAFE_ZONE_POSITION + Vector3.new(0, 5, 0))
    end
end)

TeleguiadoBtn.MouseButton1Click:Connect(function()
    Config.StealMode = (Config.StealMode == "Tween") and "Instant" or "Tween"
    TeleguiadoBtn.Text = "🎯 Teleguiado: " .. Config.StealMode
end)

-- ===== MISC TAB =====
local VoltaBtn = mkBtn(MiscContent, "VoltaBtn", "🔄 Modo Volta", 15)
local ModeBtn = mkBtn(MiscContent, "ModeBtn", "🚶 Modo: Walk", 63)
local WebhookToggle = mkBtn(MiscContent, "WebhookToggle", "🌐 Webhook: OFF", 111)

local voltaOn = false
VoltaBtn.MouseButton1Click:Connect(function()
    voltaOn = not voltaOn
    if voltaOn then
        VoltaBtn.Text = "🔄 Modo Volta [ON]"
        VoltaBtn.BackgroundColor3 = Color3.fromRGB(0, 180, 100)
    else
        VoltaBtn.Text = "🔄 Modo Volta"
        VoltaBtn.BackgroundColor3 = Color3.fromRGB(0, 90, 200)
    end
end)

local isFlying = false
ModeBtn.MouseButton1Click:Connect(function()
    if not isFlying then
        isFlying = true
        ModeBtn.Text = "🕊 Modo: Fly"
        ModeBtn.BackgroundColor3 = Color3.fromRGB(150, 0, 200)
        local char = LocalPlayer.Character
        if char and char:FindFirstChild("HumanoidRootPart") then
            local hrp = char.HumanoidRootPart
            local bg = Instance.new("BodyGyro", hrp)
            bg.P = 9e4; bg.maxTorque = Vector3.new(9e9,9e9,9e9); bg.cframe = hrp.CFrame
            local bv = Instance.new("BodyVelocity", hrp)
            bv.velocity = Vector3.zero; bv.maxForce = Vector3.new(9e9,9e9,9e9)
            local cam = workspace.CurrentCamera
            local conn
            conn = RunService.RenderStepped:Connect(function()
                if not isFlying then
                    conn:Disconnect(); bg:Destroy(); bv:Destroy(); return
                end
                local dir = Vector3.zero
                if UserInputService:IsKeyDown(Enum.KeyCode.W) then dir += cam.CFrame.LookVector end
                if UserInputService:IsKeyDown(Enum.KeyCode.S) then dir -= cam.CFrame.LookVector end
                if UserInputService:IsKeyDown(Enum.KeyCode.A) then dir -= cam.CFrame.RightVector end
                if UserInputService:IsKeyDown(Enum.KeyCode.D) then dir += cam.CFrame.RightVector end
                if UserInputService:IsKeyDown(Enum.KeyCode.Space) then dir += Vector3.new(0,1,0) end
                if UserInputService:IsKeyDown(Enum.KeyCode.LeftControl) then dir -= Vector3.new(0,1,0) end
                bv.velocity = dir * 60
                bg.cframe = cam.CFrame
            end)
        end
    else
        isFlying = false
        ModeBtn.Text = "🚶 Modo: Walk"
        ModeBtn.BackgroundColor3 = Color3.fromRGB(0, 90, 200)
    end
end)

WebhookToggle.MouseButton1Click:Connect(function()
    Config.WebhookEnabled = not Config.WebhookEnabled
    WebhookToggle.Text = Config.WebhookEnabled and "🌐 Webhook: ON" or "🌐 Webhook: OFF"
    WebhookToggle.BackgroundColor3 = Config.WebhookEnabled and Color3.fromRGB(0,180,100) or Color3.fromRGB(0,90,200)
end)

-- ===== Troca de Abas =====
MainTab.MouseButton1Click:Connect(function()
    MainContent.Visible = true; MiscContent.Visible = false
    MainTab.BackgroundColor3 = Color3.fromRGB(0,120,255)
    MiscTab.BackgroundColor3 = Color3.fromRGB(25,40,70)
    MainTab.TextColor3 = Color3.new(1,1,1)
    MiscTab.TextColor3 = Color3.fromRGB(180,200,230)
end)

MiscTab.MouseButton1Click:Connect(function()
    MainContent.Visible = false; MiscContent.Visible = true
    MiscTab.BackgroundColor3 = Color3.fromRGB(0,120,255)
    MainTab.BackgroundColor3 = Color3.fromRGB(25,40,70)
    MiscTab.TextColor3 = Color3.new(1,1,1)
    MainTab.TextColor3 = Color3.fromRGB(180,200,230)
end)

-- ===== Minimize =====
local minimized = false
MinBtn.MouseButton1Click:Connect(function()
    minimized = not minimized
    if minimized then
        MainFrame.Size = UDim2.new(0, 500, 0, 35)
        TabHolder.Visible = false
        Content.Visible = false
    else
        MainFrame.Size = UDim2.new(0, 500, 0, 400)
        TabHolder.Visible = true
        Content.Visible = true
    end
end)

-- ===== Character Added =====
LocalPlayer.CharacterAdded:Connect(function(c)
    applyKillPartIgnore(c)
    task.wait(0.3)
    local hum = c:FindFirstChildOfClass("Humanoid")
    if hum then hum.WalkSpeed = 16 end
end)

print("🔥 Natsu Hub v2 carregado! Eggs + Steal prontos.")
