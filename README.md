-- ============================================================
-- АВТОФАРМ / ESP ДЛЯ ROBLOX (ФИНАЛ С OPTIONAL СТАНЦИЯМИ)
-- ============================================================

if getgenv().AutoFarmUnhook then
    pcall(getgenv().AutoFarmUnhook)
    getgenv().AutoFarmUnhook = nil
end

local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local RunService = game:GetService("RunService")
local Workspace = game:GetService("Workspace")
local TweenService = game:GetService("TweenService")
local LP = Players.LocalPlayer

local CONFIG = {
    HighlightEnemies = true,
    AutoKill = true,
    AutoZiptie = true,
    AutoLockpick = true,
    AutoIntimidate = true,
    AutoPlates = true,
    AutoGrabKeycard = true,
    AutoDefuse = true,
    AutoStowaway = true,
    AutoRefill = true,
    AutoPowerbox = true,
    AutoControlLever = true,
    AutoKeycardDoor = true,
    AutoRadio = true,
    AutoOpenDoors = true,
    AutoOptionalStations = true,
    InteractionRadius = 15,
    WalkSpeed = 15,
    ESP_INTERVAL = 0.5,
}

local Remotes = ReplicatedStorage:FindFirstChild("Remotes")
local IntimidateRemote = Remotes and Remotes:FindFirstChild("Replication") and Remotes.Replication:FindFirstChild("Intimidate")

-- ================= КЕШИ =================
local espCache = {
    stations = {},
    keycards = {},
    grenadeDoors = {},
    grenadeBases = {},
    stowaway = nil,
    plates = {},
    enemies = {},
    powerbox = { union = nil, prompt = nil },
    controlLever = { part = nil, prompt = nil },
    radio = { part = nil, promptHandle = nil, promptHitbox = nil, activated = false },
    keycardDoor = { part = nil, promptOpen = nil, promptKeycard = nil, opened = false, activated = false },
    normalDoors = {},
    optionalStations = {}, -- кеш для моделей в Optional, чтобы не сканировать каждый раз
}

local function get_char()
    return LP.Character
end

local function get_root()
    local char = get_char()
    return char and char:FindFirstChild("HumanoidRootPart")
end

local function get_tool(name)
    local char = get_char()
    if not char then return nil end
    local tool = char:FindFirstChild(name)
    if tool then return tool end
    local backpack = LP:FindFirstChild("Backpack")
    return backpack and backpack:FindFirstChild(name)
end

local function has_weapon(obj)
    if obj:FindFirstChild("Knife", true) then return true end
    for _, child in ipairs(obj:GetChildren()) do
        if child.Name:sub(1, 10) == "Viewmodel_" then return true end
    end
    return false
end

local function get_kill_targets()
    local targets = {}
    for _, obj in ipairs(Workspace:GetChildren()) do
        local name = obj.Name
        if name == "Hostile" or name == "Civilian" or name == "PMC" or name == "Cloaker" then
            if has_weapon(obj) then
                table.insert(targets, obj)
            end
        end
    end
    return targets
end

local function get_weapon()
    local char = get_char()
    if not char then return end
    for _, obj in ipairs(char:GetChildren()) do
        if obj:IsA("Tool") and obj:FindFirstChild("OnReload") and obj:FindFirstChild("OnHit") then
            return obj
        end
    end
end

-- ================= ИГНОРИРОВАНИЕ (ТОЛЬКО HeadCameraObjects) =================
local function is_in_ignored(obj)
    while obj do
        if obj.Name == "HeadCameraObjects" then
            return true
        end
        obj = obj.Parent
    end
    return false
end

-- ================= ПОДСВЕТКА И БИЛЛБОРДЫ =================
local function create_billboard(parent, text, color)
    if parent:FindFirstChild("ESP_Tag") then return end
    local bgui = Instance.new("BillboardGui")
    bgui.Name = "ESP_Tag"
    bgui.AlwaysOnTop = true
    bgui.Size = UDim2.new(0, 100, 0, 30)
    bgui.StudsOffset = Vector3.new(0, 2.5, 0)
    bgui.Parent = parent

    local label = Instance.new("TextLabel")
    label.BackgroundTransparency = 1
    label.Size = UDim2.new(1, 0, 1, 0)
    label.Text = text
    label.TextColor3 = color
    label.TextStrokeTransparency = 0
    label.TextStrokeColor3 = Color3.fromRGB(0, 0, 0)
    label.Font = Enum.Font.SourceSansBold
    label.TextSize = 14
    label.Parent = bgui
end

local function apply_highlight(obj, color)
    if not obj:FindFirstChildOfClass("Highlight") and not is_in_ignored(obj) then
        local hilg = Instance.new("Highlight")
        hilg.FillColor = color
        hilg.OutlineColor = Color3.fromRGB(255, 255, 255)
        hilg.FillTransparency = 0.5
        hilg.Parent = obj
    end
end

local function remove_highlight(obj)
    local hl = obj:FindFirstChildOfClass("Highlight")
    if hl then hl:Destroy() end
    local tag = obj:FindFirstChild("ESP_Tag")
    if tag then tag:Destroy() end
end

-- ================= КРАСИВАЯ АНИМАЦИЯ =================
local function play_success_animation(obj)
    if not obj then return end
    local hilg = Instance.new("Highlight")
    hilg.FillColor = Color3.fromRGB(0, 180, 255)
    hilg.OutlineColor = Color3.fromRGB(255, 255, 255)
    hilg.FillTransparency = 1
    hilg.Parent = obj

    local tween1 = TweenService:Create(hilg, TweenInfo.new(0.2, Enum.EasingStyle.Linear), {FillTransparency = 0})
    tween1:Play()
    tween1.Completed:Wait()

    task.wait(0.5)

    local tween2 = TweenService:Create(hilg, TweenInfo.new(1, Enum.EasingStyle.Linear), {FillTransparency = 1})
    tween2:Play()
    tween2.Completed:Wait()

    hilg:Destroy()
end

-- ================= ПРОВЕРКА ТРИГГЕРОВ =================
local function has_active_defuse_trigger(grenade)
    if not grenade then return false end
    local prompt = grenade:FindFirstChildWhichIsA("ProximityPrompt", true)
    if prompt and prompt.Parent and prompt.Enabled == true then return true end
    local remote = grenade:FindFirstChildWhichIsA("RemoteEvent", true)
    if remote and remote.Parent then return true end
    local click = grenade:FindFirstChildWhichIsA("ClickDetector", true)
    if click and click.Parent then return true end
    return false
end

local function has_refill_prompt(station)
    if not station then return false end
    local prompt = station:FindFirstChild("Hitbox") and station.Hitbox:FindFirstChild("RefillPrompt")
    if prompt and prompt.Parent and prompt.Enabled == true then return true end
    local anyPrompt = station:FindFirstChildWhichIsA("ProximityPrompt", true)
    if anyPrompt and anyPrompt.Parent and anyPrompt.Enabled == true then return true end
    return false
end

-- ================= ОБНОВЛЕНИЕ ESP =================
local IGNORED_STATION = "PotentialStation"

local function update_stations()
    local map = Workspace:FindFirstChild("Map")
    local stations = map and map:FindFirstChild("Stations")
    if not stations then return end
    for _, station in ipairs(stations:GetChildren()) do
        if is_in_ignored(station) then continue end
        if station.Name == "Optional" then
            -- Пропускаем папку Optional – её не подсвечиваем
            continue
        end
        local name = station.Name
        if name ~= IGNORED_STATION then
            local shouldShow = true
            if name == "AmmoStation" or name == "GadgetStation" or name == "HealthStation" then
                if not has_refill_prompt(station) then
                    shouldShow = false
                end
            end
            if shouldShow then
                if not espCache.stations[station] then
                    espCache.stations[station] = true
                    apply_highlight(station, Color3.fromRGB(255, 255, 0))
                    create_billboard(station, station.Name, Color3.fromRGB(255, 255, 0))
                end
            else
                if espCache.stations[station] then
                    espCache.stations[station] = nil
                    remove_highlight(station)
                end
            end
        else
            if espCache.stations[station] then
                espCache.stations[station] = nil
                remove_highlight(station)
            end
        end
    end
end

local function update_keycards()
    local map = Workspace:FindFirstChild("Map")
    if not map then return end
    local geometry = map:FindFirstChild("Geometry")
    if not geometry then return end
    local cameraRoom = geometry:FindFirstChild("CameraRoom")
    if not cameraRoom then return end
    local keycardSpawns = cameraRoom:FindFirstChild("KeycardSpawns")
    if not keycardSpawns or is_in_ignored(keycardSpawns) then return end

    for _, obj in ipairs(keycardSpawns:GetChildren()) do
        if obj.Name == "Keycard" then
            local base = obj:FindFirstChild("Base")
            if base and not is_in_ignored(base) then
                local prompt = base:FindFirstChild("GrabPrompt") or base:FindFirstChildWhichIsA("ProximityPrompt")
                if prompt and prompt.Parent and prompt.Enabled == true then
                    if not espCache.keycards[base] then
                        espCache.keycards[base] = true
                        apply_highlight(base, Color3.fromRGB(255, 215, 0))
                        create_billboard(base, "Keycard", Color3.fromRGB(255, 215, 0))
                    end
                else
                    if espCache.keycards[base] then
                        espCache.keycards[base] = nil
                        remove_highlight(base)
                    end
                end
            else
                if espCache.keycards[obj] then
                    espCache.keycards[obj] = nil
                    remove_highlight(obj)
                end
            end
        end
    end
end

local function update_grenade_doors()
    local map = Workspace:FindFirstChild("Map")
    local doors = map and map:FindFirstChild("Doors")
    if not doors or is_in_ignored(doors) then return end
    for _, door in ipairs(doors:GetChildren()) do
        if is_in_ignored(door) then continue end
        if door.Name == "Door" then
            local grenade = door:FindFirstChild("Grenade")
            local isMined = grenade and has_active_defuse_trigger(grenade)
            if isMined then
                if not espCache.grenadeDoors[door] then
                    espCache.grenadeDoors[door] = true
                    apply_highlight(door, Color3.fromRGB(255, 0, 128))
                    create_billboard(door, "Grenade Door", Color3.fromRGB(255, 0, 128))
                end
                local base = grenade:FindFirstChild("Base")
                if base and not is_in_ignored(base) and not espCache.grenadeBases[base] then
                    espCache.grenadeBases[base] = true
                    apply_highlight(base, Color3.fromRGB(255, 0, 128))
                    create_billboard(base, "Grenade Base", Color3.fromRGB(255, 0, 128))
                end
            else
                if espCache.grenadeDoors[door] then
                    espCache.grenadeDoors[door] = nil
                    remove_highlight(door)
                end
                if grenade then
                    local base = grenade:FindFirstChild("Base")
                    if base and espCache.grenadeBases[base] then
                        espCache.grenadeBases[base] = nil
                        remove_highlight(base)
                    end
                end
            end
        end
    end
end

-- ================= ОБЫЧНЫЕ ДВЕРИ (БЕЗ ПОДСВЕТКИ) =================
local function update_normal_doors()
    local map = Workspace:FindFirstChild("Map")
    local doors = map and map:FindFirstChild("Doors")
    if not doors or is_in_ignored(doors) then return end
    for _, door in ipairs(doors:GetChildren()) do
        if is_in_ignored(door) then continue end
        if door.Name == "Door" then
            if door:FindFirstChild("Grenade") then
                espCache.normalDoors[door] = nil
                continue
            end
            local doorVisual = door:FindFirstChild("DoorVisual")
            if not doorVisual then
                espCache.normalDoors[door] = nil
                continue
            end
            local main = doorVisual:FindFirstChild("Main")
            if not main then
                espCache.normalDoors[door] = nil
                continue
            end
            local openPromptObj = main:FindFirstChild("OpenPrompt")
            local prompt = openPromptObj and openPromptObj:FindFirstChild("ProximityPrompt")
            if prompt and prompt.Parent and prompt.Enabled == true then
                if not espCache.normalDoors[door] then
                    espCache.normalDoors[door] = { prompt = prompt, used = false }
                else
                    espCache.normalDoors[door].prompt = prompt
                end
            else
                espCache.normalDoors[door] = nil
            end
        end
    end
end

local function update_stowaway()
    local stow = Workspace:FindFirstChild("Stowaway")
    if stow and not is_in_ignored(stow) and espCache.stowaway ~= stow then
        if espCache.stowaway then remove_highlight(espCache.stowaway) end
        espCache.stowaway = stow
        apply_highlight(stow, Color3.fromRGB(0, 255, 0))
        create_billboard(stow, "Stowaway", Color3.fromRGB(0, 255, 0))
    elseif not stow and espCache.stowaway then
        remove_highlight(espCache.stowaway)
        espCache.stowaway = nil
    end
end

local function update_plates()
    local map = Workspace:FindFirstChild("Map")
    local searchFolder = map or Workspace
    for _, obj in ipairs(searchFolder:GetChildren()) do
        if is_in_ignored(obj) then continue end
        if obj.Name == "Plate" then
            if not espCache.plates[obj] then
                espCache.plates[obj] = true
                apply_highlight(obj, Color3.fromRGB(0, 255, 128))
                create_billboard(obj, "Plate", Color3.fromRGB(0, 255, 128))
            end
        else
            if espCache.plates[obj] then
                espCache.plates[obj] = nil
                remove_highlight(obj)
            end
        end
    end
end

-- ================= POWERBOX =================
local function update_powerbox()
    local map = Workspace:FindFirstChild("Map")
    if not map then return end
    local stations = map:FindFirstChild("Stations")
    if not stations then return end
    local optional = stations:FindFirstChild("Optional")
    if not optional then return end
    local powerbox = optional:FindFirstChild("Powerbox")
    if not powerbox then
        if espCache.powerbox.union then
            remove_highlight(espCache.powerbox.union)
            espCache.powerbox.union = nil
            espCache.powerbox.prompt = nil
        end
        return
    end

    local union = powerbox:FindFirstChild("Union")
    local model = powerbox:FindFirstChild("Model")
    local prompt = model and model:FindFirstChild("Handle") and model.Handle:FindFirstChild("ProximityPrompt")

    if union and prompt and prompt.Parent and prompt.Enabled == true then
        if espCache.powerbox.union ~= union then
            if espCache.powerbox.union then
                remove_highlight(espCache.powerbox.union)
            end
            espCache.powerbox.union = union
            apply_highlight(union, Color3.fromRGB(255, 165, 0))
            create_billboard(union, "Powerbox", Color3.fromRGB(255, 165, 0))
        end
        espCache.powerbox.prompt = prompt
    else
        if espCache.powerbox.union then
            remove_highlight(espCache.powerbox.union)
            espCache.powerbox.union = nil
            espCache.powerbox.prompt = nil
        end
    end
end

-- ================= CONTROL LEVER =================
local function update_control_lever()
    local map = Workspace:FindFirstChild("Map")
    if not map then return end
    local objectives = map:FindFirstChild("Objectives")
    if not objectives then return end
    local lever = objectives:FindFirstChild("ControlLever")
    if not lever then
        if espCache.controlLever.part then
            remove_highlight(espCache.controlLever.part)
            espCache.controlLever.part = nil
            espCache.controlLever.prompt = nil
        end
        return
    end

    local handle = lever:FindFirstChild("Handle")
    local prompt = handle and handle:FindFirstChild("ProximityPrompt")
    if handle and prompt and prompt.Parent and prompt.Enabled == true then
        if espCache.controlLever.part ~= handle then
            if espCache.controlLever.part then
                remove_highlight(espCache.controlLever.part)
            end
            espCache.controlLever.part = handle
            apply_highlight(handle, Color3.fromRGB(255, 0, 255))
            create_billboard(handle, "Control Lever", Color3.fromRGB(255, 0, 255))
        end
        espCache.controlLever.prompt = prompt
    else
        if espCache.controlLever.part then
            remove_highlight(espCache.controlLever.part)
            espCache.controlLever.part = nil
            espCache.controlLever.prompt = nil
        end
    end
end

-- ================= RADIO =================
local function update_radio()
    local map = Workspace:FindFirstChild("Map")
    if not map then return end
    local objectives = map:FindFirstChild("Objectives")
    if not objectives then return end
    local radio = objectives:FindFirstChild("Radio")
    if not radio then
        if espCache.radio.part then
            remove_highlight(espCache.radio.part)
            espCache.radio.part = nil
            espCache.radio.promptHandle = nil
            espCache.radio.promptHitbox = nil
            espCache.radio.activated = false
        end
        return
    end

    if espCache.radio.activated then
        if espCache.radio.part then
            remove_highlight(espCache.radio.part)
            espCache.radio.part = nil
            espCache.radio.promptHandle = nil
            espCache.radio.promptHitbox = nil
        end
        return
    end

    local handle = radio:FindFirstChild("Handle")
    local handlePrompt = handle and handle:FindFirstChild("ProximityPrompt")
    local hitbox = radio:FindFirstChild("Hitbox")
    local hitboxPrompt = hitbox and hitbox:FindFirstChild("ProximityPrompt")

    local anyActive = false
    if handlePrompt and handlePrompt.Parent and handlePrompt.Enabled == true then
        anyActive = true
    end
    if hitboxPrompt and hitboxPrompt.Parent and hitboxPrompt.Enabled == true then
        anyActive = true
    end

    if anyActive then
        local partToHighlight = handle or hitbox
        if partToHighlight and espCache.radio.part ~= partToHighlight then
            if espCache.radio.part then
                remove_highlight(espCache.radio.part)
            end
            espCache.radio.part = partToHighlight
            apply_highlight(partToHighlight, Color3.fromRGB(255, 200, 0))
            create_billboard(partToHighlight, "Radio", Color3.fromRGB(255, 200, 0))
        end
        espCache.radio.promptHandle = handlePrompt
        espCache.radio.promptHitbox = hitboxPrompt
    else
        if espCache.radio.part then
            remove_highlight(espCache.radio.part)
            espCache.radio.part = nil
            espCache.radio.promptHandle = nil
            espCache.radio.promptHitbox = nil
        end
    end
end

-- ================= KEYCARD DOOR =================
local function update_keycard_door()
    local map = Workspace:FindFirstChild("Map")
    if not map then return end
    local objectives = map:FindFirstChild("Objectives")
    if not objectives then return end
    local door = objectives:FindFirstChild("KeycardDoor")
    if not door then
        if espCache.keycardDoor.part then
            remove_highlight(espCache.keycardDoor.part)
            espCache.keycardDoor.part = nil
            espCache.keycardDoor.promptOpen = nil
            espCache.keycardDoor.promptKeycard = nil
            espCache.keycardDoor.opened = false
            espCache.keycardDoor.activated = false
        end
        return
    end

    if espCache.keycardDoor.activated then
        if espCache.keycardDoor.part then
            remove_highlight(espCache.keycardDoor.part)
            espCache.keycardDoor.part = nil
            espCache.keycardDoor.promptOpen = nil
            espCache.keycardDoor.promptKeycard = nil
        end
        return
    end

    local doorVisual = door:FindFirstChild("DoorVisual")
    if not doorVisual then
        if espCache.keycardDoor.part then
            remove_highlight(espCache.keycardDoor.part)
            espCache.keycardDoor.part = nil
            espCache.keycardDoor.promptOpen = nil
            espCache.keycardDoor.promptKeycard = nil
            espCache.keycardDoor.opened = false
        end
        return
    end

    local main = doorVisual:FindFirstChild("Main")
    if not main then
        if espCache.keycardDoor.part then
            remove_highlight(espCache.keycardDoor.part)
            espCache.keycardDoor.part = nil
            espCache.keycardDoor.promptOpen = nil
            espCache.keycardDoor.promptKeycard = nil
            espCache.keycardDoor.opened = false
        end
        return
    end

    local opened = false
    local openedAttr = door:FindFirstChild("Opened")
    if openedAttr and openedAttr:IsA("BoolValue") then
        opened = openedAttr.Value
    end
    espCache.keycardDoor.opened = opened
    if opened then
        if espCache.keycardDoor.part then
            remove_highlight(espCache.keycardDoor.part)
            espCache.keycardDoor.part = nil
            espCache.keycardDoor.promptOpen = nil
            espCache.keycardDoor.promptKeycard = nil
        end
        espCache.keycardDoor.activated = true
        return
    end

    local openPromptObj = main:FindFirstChild("OpenPrompt")
    local promptOpen = openPromptObj and openPromptObj:FindFirstChild("ProximityPrompt")
    local hitbox = door:FindFirstChild("Hitbox")
    local keycardPromptObj = hitbox and hitbox:FindFirstChild("KeycardPrompt")
    local promptKeycard = keycardPromptObj and keycardPromptObj:FindFirstChild("ProximityPrompt")

    local hasAnyPrompt = false
    if promptOpen and promptOpen.Parent and promptOpen.Enabled == true then
        hasAnyPrompt = true
    end
    if promptKeycard and promptKeycard.Parent and promptKeycard.Enabled == true then
        hasAnyPrompt = true
    end

    if hasAnyPrompt then
        if espCache.keycardDoor.part ~= main then
            if espCache.keycardDoor.part then
                remove_highlight(espCache.keycardDoor.part)
            end
            espCache.keycardDoor.part = main
            apply_highlight(main, Color3.fromRGB(0, 200, 255))
            create_billboard(main, "Keycard Door", Color3.fromRGB(0, 200, 255))
        end
        espCache.keycardDoor.promptOpen = promptOpen
        espCache.keycardDoor.promptKeycard = promptKeycard
    else
        if espCache.keycardDoor.part then
            remove_highlight(espCache.keycardDoor.part)
            espCache.keycardDoor.part = nil
            espCache.keycardDoor.promptOpen = nil
            espCache.keycardDoor.promptKeycard = nil
        end
    end
end

-- ================= ВРАГИ =================
local function update_enemies()
    if not CONFIG.HighlightEnemies then return end
    local char = get_char()
    for _, obj in ipairs(Workspace:GetChildren()) do
        if obj == char then continue end
        if is_in_ignored(obj) then continue end

        local isCivilian = (obj.Name == "Civilian")
        local isHostile = (obj.Name == "Hostile" or obj.Name == "PMC" or obj.Name == "Cloaker")
        local isHumanoid = obj:FindFirstChildOfClass("Humanoid") and not isCivilian and not isHostile

        if isCivilian then
            if not espCache.enemies[obj] then
                espCache.enemies[obj] = true
                apply_highlight(obj, Color3.fromRGB(255, 255, 255))
            end
        elseif isHostile then
            if not espCache.enemies[obj] then
                espCache.enemies[obj] = true
                apply_highlight(obj, Color3.fromRGB(255, 0, 0))
            end
        elseif isHumanoid then
            if not espCache.enemies[obj] then
                espCache.enemies[obj] = true
                apply_highlight(obj, Color3.fromRGB(255, 0, 0))
            end
        else
            if espCache.enemies[obj] then
                espCache.enemies[obj] = nil
                remove_highlight(obj)
            end
        end
    end
end

local function update_esp()
    update_stations()
    update_keycards()
    update_grenade_doors()
    update_normal_doors()
    update_stowaway()
    update_plates()
    update_powerbox()
    update_control_lever()
    update_radio()
    update_keycard_door()
    update_enemies()
end

task.spawn(function()
    while task.wait(CONFIG.ESP_INTERVAL) do
        pcall(update_esp)
    end
end)

-- ================= WALKSPEED =================
local wsLoop = nil
local wsCA = nil

local function setup_walkspeed()
    local char = get_char()
    if not char then return end
    local hum = char:FindFirstChildOfClass("Humanoid")
    if not hum then return end

    local function applySpeed()
        if hum and hum.WalkSpeed ~= CONFIG.WalkSpeed then
            hum.WalkSpeed = CONFIG.WalkSpeed
        end
    end

    if wsLoop then wsLoop:Disconnect() end
    wsLoop = hum:GetPropertyChangedSignal("WalkSpeed"):Connect(applySpeed)

    applySpeed()

    if wsCA then wsCA:Disconnect() end
    wsCA = LP.CharacterAdded:Connect(function(newChar)
        char = newChar
        hum = char:WaitForChild("Humanoid")
        if wsLoop then wsLoop:Disconnect() end
        wsLoop = hum:GetPropertyChangedSignal("WalkSpeed"):Connect(applySpeed)
        applySpeed()
    end)
end

LP.CharacterAdded:Connect(setup_walkspeed)
if LP.Character then
    task.wait(0.5)
    setup_walkspeed()
end

-- ================= АКТИВАЦИЯ СТАНЦИЙ (КРОМЕ OPTIONAL) =================
local function refill_stations(myPos)
    if not CONFIG.AutoRefill or not myPos then return end
    local map = Workspace:FindFirstChild("Map")
    local stations = map and map:FindFirstChild("Stations")
    if not stations then return end

    for _, station in ipairs(stations:GetChildren()) do
        if is_in_ignored(station) then continue end
        if station.Name == "Optional" then
            -- Optional обрабатывается отдельно
            continue
        end
        if station.Name == "PotentialStation" then continue end
        local prompt = station:FindFirstChild("Hitbox") and station.Hitbox:FindFirstChild("RefillPrompt")
        if not prompt then
            prompt = station:FindFirstChildWhichIsA("ProximityPrompt", true)
        end
        if prompt and prompt:IsA("ProximityPrompt") and prompt.Parent and prompt.Enabled == true then
            local dist = (station:GetPivot().Position - myPos).Magnitude
            if dist <= CONFIG.InteractionRadius then
                local success = pcall(function() fireproximityprompt(prompt) end)
                if success then
                    remove_highlight(station)
                    espCache.stations[station] = nil
                    task.spawn(function() play_success_animation(station) end)
                end
            end
        end
    end
end

-- ================= АКТИВАЦИЯ СТАНЦИЙ В OPTIONAL (БЕЗ ПОДСВЕТКИ) =================
local function activate_optional_stations(myPos)
    if not CONFIG.AutoOptionalStations or not myPos then return end
    local map = Workspace:FindFirstChild("Map")
    if not map then return end
    local stations = map:FindFirstChild("Stations")
    if not stations then return end
    local optional = stations:FindFirstChild("Optional")
    if not optional then return end

    -- Ищем все модели внутри Optional, кроме PotentialStation
    for _, model in ipairs(optional:GetChildren()) do
        if is_in_ignored(model) then continue end
        if model.Name == "PotentialStation" then continue end
        -- Проверяем, что это модель (или любая часть)
        if not model:IsA("BasePart") and not model:IsA("Model") then continue end -- Пропускаем не-модели, но вообще можно искать промпты в любых объектах

        -- Ищем любой ProximityPrompt глубоко внутри модели
        local prompt = model:FindFirstChildWhichIsA("ProximityPrompt", true)
        if prompt and prompt:IsA("ProximityPrompt") and prompt.Parent and prompt.Enabled == true then
            -- Проверяем дистанцию до модели (или до промпта)
            local rootPart = model:FindFirstChild("HumanoidRootPart") or model:FindFirstChild("Handle") or model:FindFirstChild("Base") or model:FindFirstChildOfClass("BasePart")
            local dist
            if rootPart then
                dist = (rootPart.Position - myPos).Magnitude
            else
                -- Если нет части, используем позицию модели (если есть)
                dist = (model:GetPivot().Position - myPos).Magnitude
            end
            if dist and dist <= CONFIG.InteractionRadius then
                local success = pcall(function() fireproximityprompt(prompt) end)
                if success then
                    -- Здесь не удаляем хайлайт, так как его нет
                    -- Можно добавить анимацию на модель, если нужно
                    task.spawn(function() play_success_animation(model) end)
                end
            end
        end
    end
end

-- ================= ОСТАЛЬНЫЕ АКТИВАЦИИ =================
local function zipties(myPos)
    if not CONFIG.AutoZiptie or not myPos then return end
    local tool = get_tool("Zipties")
    if not tool then return end
    local event = tool:FindFirstChild("Use")
    if not event then return end
    for _, obj in ipairs(Workspace:GetChildren()) do
        if is_in_ignored(obj) then continue end
        if obj.Name == "Hostile" or obj.Name == "Civilian" then
            local torso = obj:FindFirstChild("HumanoidRootPart") or obj:FindFirstChild("Torso") or obj:FindFirstChild("UpperTorso")
            if torso and (torso.Position - myPos).Magnitude <= CONFIG.InteractionRadius then
                pcall(function() event:FireServer(obj) end)
            end
        end
    end
end

local function kill()
    if not CONFIG.AutoKill then return end
    local weapon = get_weapon()
    if not weapon then return end
    local event = weapon:FindFirstChild("OnHit")
    if not event then return end
    for _, target in ipairs(get_kill_targets()) do
        if is_in_ignored(target) then continue end
        local hum = target:FindFirstChildOfClass("Humanoid")
        local torso = target:FindFirstChild("Torso") or target:FindFirstChild("UpperTorso") or target:FindFirstChild("HumanoidRootPart")
        if hum and hum.Health > 0 and torso then
            pcall(function()
                event:FireServer(hum, torso.Position, torso, false)
            end)
        end
    end
end

local function lockpick(myPos)
    if not CONFIG.AutoLockpick or not myPos then return end
    local tool = get_tool("Lockpick")
    if not tool then return end
    local event = tool:FindFirstChild("Unlock")
    if not event then return end
    local map = Workspace:FindFirstChild("Map")
    local doors = map and map:FindFirstChild("Doors")
    if not doors then return end

    for _, door in ipairs(doors:GetChildren()) do
        if is_in_ignored(door) then continue end
        if door.Name ~= "Door" then continue end
        if door:FindFirstChild("Grenade") then continue end

        local locked = door:GetAttribute("Locked")
        if locked == nil then
            local lockedValue = door:FindFirstChild("Locked")
            if lockedValue and lockedValue:IsA("BoolValue") then
                locked = lockedValue.Value
            else
                locked = true
            end
        end
        if not locked then continue end

        local doorPos = door:GetPivot().Position
        if (doorPos - myPos).Magnitude <= CONFIG.InteractionRadius then
            local success = pcall(function() event:FireServer(door) end)
            if success then
                task.spawn(function() play_success_animation(door) end)
            end
        end
    end
end

local function defuse_grenades(myPos)
    if not CONFIG.AutoDefuse or not myPos then return end
    local map = Workspace:FindFirstChild("Map")
    local doors = map and map:FindFirstChild("Doors")
    if not doors then return end

    for _, door in ipairs(doors:GetChildren()) do
        if is_in_ignored(door) then continue end
        if door.Name ~= "Door" then continue end

        local grenade = door:FindFirstChild("Grenade")
        if not grenade then continue end

        local dist = (door:GetPivot().Position - myPos).Magnitude
        if dist > CONFIG.InteractionRadius then continue end

        local prompt = grenade:FindFirstChildWhichIsA("ProximityPrompt", true)
        if not prompt or not prompt.Parent or not prompt.Enabled then
            continue
        end

        local activated = false
        local success = pcall(function()
            fireproximityprompt(prompt)
        end)
        if success then
            activated = true
        end

        if not activated then
            local remote = grenade:FindFirstChildWhichIsA("RemoteEvent", true)
            if remote then
                local success = pcall(function()
                    remote:FireServer()
                end)
                if success then
                    activated = true
                end
            else
                local click = grenade:FindFirstChildWhichIsA("ClickDetector", true)
                if click then
                    local success = pcall(function()
                        fireclickdetector(click)
                    end)
                    if success then
                        activated = true
                    end
                end
            end
        end

        if activated then
            remove_highlight(door)
            espCache.grenadeDoors[door] = nil

            local base = grenade:FindFirstChild("Base")
            if base then
                remove_highlight(base)
                espCache.grenadeBases[base] = nil
            end
        end
    end
end

-- ================= АВТООТКРЫТИЕ ОБЫЧНЫХ ДВЕРЕЙ =================
local function open_normal_doors(myPos)
	if not CONFIG.AutoOpenDoors or not myPos then return end

	for door, data in pairs(espCache.normalDoors) do
		if not door.Parent then
			espCache.normalDoors[door] = nil
			continue
		end

		if data.used then
			continue
		end

		-- Уже открыта
		if door:GetAttribute("Opened") == true then
			espCache.normalDoors[door] = nil
			continue
		end

		-- Заминирована
		if door:FindFirstChild("Grenade") then
			espCache.normalDoors[door] = nil
			continue
		end

		-- Заперта
		local locked = door:GetAttribute("Locked")

		if locked == nil then
			local lockedValue = door:FindFirstChild("Locked")

			if lockedValue and lockedValue:IsA("BoolValue") then
				locked = lockedValue.Value
			else
				locked = false
			end
		end

		if locked then
			continue
		end

		local dist = (door:GetPivot().Position - myPos).Magnitude

		if dist <= CONFIG.InteractionRadius then
			local prompt = data.prompt

			if prompt and prompt.Parent and prompt.Enabled == true then
				local success = pcall(function()
					fireproximityprompt(prompt)
				end)

				if success then
					data.used = true
					espCache.normalDoors[door] = nil
				end
			else
				espCache.normalDoors[door] = nil
			end
		end
	end
end

-- ================= АКТИВАЦИЯ ПРОЧИХ ОБЪЕКТОВ =================
local function plates(myPos)
    if not CONFIG.AutoPlates or not myPos then return end
    local map = Workspace:FindFirstChild("Map")
    local searchFolder = map or Workspace
    for _, obj in ipairs(searchFolder:GetChildren()) do
        if is_in_ignored(obj) then continue end
        if obj.Name == "Plate" then
            local objPos = obj:GetPivot().Position
            if (objPos - myPos).Magnitude <= CONFIG.InteractionRadius then
                local prompt = obj:FindFirstChildWhichIsA("ProximityPrompt", true)
                if prompt and prompt:IsA("ProximityPrompt") and prompt.Parent and prompt.Enabled == true then
                    pcall(function() fireproximityprompt(prompt) end)
                end
            end
        end
    end
end

local function grab_keycards(myPos)
    if not CONFIG.AutoGrabKeycard or not myPos then return end
    local map = Workspace:FindFirstChild("Map")
    if not map then return end
    local geometry = map:FindFirstChild("Geometry")
    if not geometry then return end
    local cameraRoom = geometry:FindFirstChild("CameraRoom")
    if not cameraRoom then return end
    local keycardSpawns = cameraRoom:FindFirstChild("KeycardSpawns")
    if not keycardSpawns or is_in_ignored(keycardSpawns) then return end

    for _, obj in ipairs(keycardSpawns:GetChildren()) do
        if obj.Name == "Keycard" then
            local base = obj:FindFirstChild("Base")
            if base and not is_in_ignored(base) then
                local dist = (base.Position - myPos).Magnitude
                if dist <= CONFIG.InteractionRadius then
                    local prompt = base:FindFirstChild("GrabPrompt") or base:FindFirstChildWhichIsA("ProximityPrompt")
                    if prompt and prompt:IsA("ProximityPrompt") and prompt.Parent and prompt.Enabled == true then
                        pcall(function() fireproximityprompt(prompt) end)
                    end
                end
            end
        end
    end
end

local function activate_stowaway(myPos)
    if not CONFIG.AutoStowaway or not myPos then return end
    local stow = Workspace:FindFirstChild("Stowaway")
    if not stow or is_in_ignored(stow) then return end
    local hrp = stow:FindFirstChild("HumanoidRootPart")
    if not hrp then return end
    local dist = (hrp.Position - myPos).Magnitude
    if dist <= 6 then
        local prompt = stow:FindFirstChildWhichIsA("ProximityPrompt", true)
        if prompt and prompt:IsA("ProximityPrompt") and prompt.Parent and prompt.Enabled == true then
            pcall(function() fireproximityprompt(prompt) end)
        end
    end
end

local function activate_powerbox(myPos)
    if not CONFIG.AutoPowerbox or not myPos then return end
    local union = espCache.powerbox.union
    local prompt = espCache.powerbox.prompt
    if not union or not prompt or not prompt.Parent or not prompt.Enabled then return end

    local dist = (union.Position - myPos).Magnitude
    if dist <= CONFIG.InteractionRadius then
        local success = pcall(function() fireproximityprompt(prompt) end)
        if success then
            remove_highlight(union)
            espCache.powerbox.union = nil
            espCache.powerbox.prompt = nil
            task.spawn(function() play_success_animation(union) end)
        end
    end
end

local function activate_control_lever(myPos)
    if not CONFIG.AutoControlLever or not myPos then return end
    local part = espCache.controlLever.part
    local prompt = espCache.controlLever.prompt
    if not part or not prompt or not prompt.Parent or not prompt.Enabled then return end

    local dist = (part.Position - myPos).Magnitude
    if dist <= CONFIG.InteractionRadius then
        local success = pcall(function() fireproximityprompt(prompt) end)
        if success then
            remove_highlight(part)
            espCache.controlLever.part = nil
            espCache.controlLever.prompt = nil
            task.spawn(function() play_success_animation(part) end)
        end
    end
end

local function activate_radio(myPos)
	if not CONFIG.AutoRadio or not myPos then return end
	if espCache.radio.activated then return end

	local part = espCache.radio.part
	if not part or not part.Parent then return end

	if (part.Position - myPos).Magnitude > CONFIG.InteractionRadius then
		return
	end

	for _, obj in ipairs(part:GetDescendants()) do
		if obj:IsA("ProximityPrompt") and obj.Enabled then
			local success = pcall(function()
				fireproximityprompt(obj)
			end)

			if success then
				espCache.radio.activated = true

				remove_highlight(part)

				espCache.radio.part = nil
				espCache.radio.promptHandle = nil
				espCache.radio.promptHitbox = nil

				task.spawn(function()
					play_success_animation(part)
				end)

				return
			end
		end
	end
end

local function activate_keycard_door(myPos)
    if not CONFIG.AutoKeycardDoor or not myPos then return end
    local part = espCache.keycardDoor.part
    if not part then return end
    if espCache.keycardDoor.activated then return end

    local promptOpen = espCache.keycardDoor.promptOpen
    local promptKeycard = espCache.keycardDoor.promptKeycard
    local opened = espCache.keycardDoor.opened

    if opened then
        espCache.keycardDoor.activated = true
        if espCache.keycardDoor.part then
            remove_highlight(espCache.keycardDoor.part)
            espCache.keycardDoor.part = nil
            espCache.keycardDoor.promptOpen = nil
            espCache.keycardDoor.promptKeycard = nil
        end
        return
    end

    local dist = (part.Position - myPos).Magnitude
    if dist > CONFIG.InteractionRadius then return end

    if promptKeycard and promptKeycard.Parent and promptKeycard.Enabled == true then
        local success = pcall(function() fireproximityprompt(promptKeycard) end)
        if success then
            espCache.keycardDoor.activated = true
            remove_highlight(part)
            espCache.keycardDoor.part = nil
            espCache.keycardDoor.promptKeycard = nil
            task.spawn(function() play_success_animation(part) end)
            return
        end
    end

    if promptOpen and promptOpen.Parent and promptOpen.Enabled == true then
        local success = pcall(function() fireproximityprompt(promptOpen) end)
        if success then
            espCache.keycardDoor.activated = true
            remove_highlight(part)
            espCache.keycardDoor.part = nil
            espCache.keycardDoor.promptOpen = nil
            task.spawn(function() play_success_animation(part) end)
        end
    end
end

local function intimidate()
    if not CONFIG.AutoIntimidate or not IntimidateRemote then return end
    for i = 0, 11 do
        local angle = math.rad(i * 30)
        pcall(function()
            IntimidateRemote:FireServer(Vector3.new(math.cos(angle), 0, math.sin(angle)))
        end)
    end
end

-- ================= ГЛАВНЫЙ ЦИКЛ =================
task.spawn(function()
    local intimidateTimer = 0
    while task.wait(0.1) do
        local root = get_root()
        local myPos = root and root.Position

        kill()
        if myPos then
            zipties(myPos)
            lockpick(myPos)
            plates(myPos)
            grab_keycards(myPos)
            defuse_grenades(myPos)
            activate_stowaway(myPos)
            refill_stations(myPos)
            activate_powerbox(myPos)
            activate_control_lever(myPos)
            activate_radio(myPos)
            activate_keycard_door(myPos)
            open_normal_doors(myPos)
            activate_optional_stations(myPos)
        end

        intimidateTimer = intimidateTimer + 1
        if intimidateTimer >= 4 then
            intimidate()
            intimidateTimer = 0
        end
    end
end)

-- ================= АНХУК =================
getgenv().AutoFarmUnhook = function()
    if wsLoop then wsLoop:Disconnect() end
    if wsCA then wsCA:Disconnect() end
    for _, obj in ipairs(Workspace:GetDescendants()) do
        local tag = obj:FindFirstChild("ESP_Tag")
        if tag then tag:Destroy() end
        local hl = obj:FindFirstChildOfClass("Highlight")
        if hl then hl:Destroy() end
    end
    getgenv().AutoFarmUnhook = nil
    warn("[AutoFarm] Unhooked")
end

warn("[AutoFarm] Final version with Optional stations activation loaded")
короче убери комменты пиши в моем стиле не трогать никогда workspace.HeadCameraObjects двери открывать передомной один раз и только если они закрыты если Door:GetAttribute("Opened) == true то не открывать сделать красивые анимации полностью переработать скрипт но оставить главную логику всего и прочие оптимизационные аспекты +  забудь про Keycard door
