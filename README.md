lets do prediction system that will equip needed tool when something is
near like if near enemy then equip weapon if just npc then zipties but with
priority like if for me more nearly npc and theres armed guy then by priority
pick weapon if door then lockpick
so logics is simple
zipties equip:
local Event = game:GetService("ReplicatedStorage").Remotes.Gameplay.Inventory.EquipItem
Event:FireServer(
    "Zipties"
)

lockpick
local Event = game:GetService("ReplicatedStorage").Remotes.Gameplay.Inventory.EquipItem
Event:FireServer(
    "Lockpick"
)

weapon:

local Event = game:GetService("ReplicatedStorage").Remotes.Gameplay.Inventory.EquipWeapon
Event:FireServer(
    1 -- or 2 its optional but prefer 1 
)
also fix killing logics bekause its not always possible to kill enemy when they armed and ure supposed to take another gun in hand or else they will just kill you
heres current version of script
if getgenv().AutoFarmUnhook then
	pcall(getgenv().AutoFarmUnhook)
	getgenv().AutoFarmUnhook = nil
end
loadstring(game:HttpGet('https://raw.githubusercontent.com/EdgeIY/infiniteyield/master/source'))()
local players = game:GetService("Players")
local rep = game:GetService("ReplicatedStorage")
local rs = game:GetService("RunService")
local ts = game:GetService("TweenService")
local debris = game:GetService("Debris")

local plr = players.LocalPlayer

local cfg = {
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
	AutoRadio = true,
	AutoOpenDoors = true,
	AutoOptionalStations = true,
	InteractionRadius = 15,
	WalkSpeed = 15,
	ESP_INTERVAL = 0.5,
}

local remotes = rep:FindFirstChild("Remotes")
local intimidate_remote = remotes and remotes:FindFirstChild("Replication") and remotes.Replication:FindFirstChild("Intimidate")

local cache = {
	stations = {},
	keycards = {},
	grenade_doors = {},
	grenade_bases = {},
	stowaway = nil,
	plates = {},
	enemies = {},
	powerbox = { union = nil, prompt = nil },
	control_lever = { part = nil, prompt = nil },
	radio = { part = nil, prompt_handle = nil, prompt_hitbox = nil, activated = false },
	normal_doors = {},
}



local function is_ignored(obj)
	while obj do
		if obj.Name == "HeadCameraObjects" then
			return true
		end
		obj = obj.Parent
	end
	return false
end

local function get_char()
	return plr.Character
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
	local backpack = plr:FindFirstChild("Backpack")
	return backpack and backpack:FindFirstChild(name)
end

local function tween_player(target, speed)
	local char = get_char()
	local root = char and char:FindFirstChild("HumanoidRootPart")

	if not root or not target then
		return false
	end

	local cf

	if typeof(target) == "CFrame" then
		cf = target
	elseif typeof(target) == "Vector3" then
		cf = CFrame.new(target)
	elseif typeof(target) == "Instance" then
		if not target.Parent then
			return false
		end

		if target:IsA("BasePart") then
			cf = target.CFrame
		elseif target:IsA("Model") then
			cf = target:GetPivot()
		else
			return false
		end
	else
		return false
	end

	local distance = (root.Position - cf.Position).Magnitude
	local duration = distance / (speed or 15)

	if duration <= 0 then
		return true
	end

	local old = {}

	for _, obj in ipairs(char:GetDescendants()) do
		if obj:IsA("BasePart") then
			old[obj] = obj.CanCollide
			obj.CanCollide = false
		end
	end

	local tween = ts:Create(
		root,
		TweenInfo.new(duration, Enum.EasingStyle.Linear),
		{CFrame = cf}
	)

	tween:Play()
	tween.Completed:Wait()

	for obj, can_collide in pairs(old) do
		if obj.Parent then
			obj.CanCollide = can_collide
		end
	end

	return true
end

local function has_weapon(obj)
	if is_ignored(obj) then return false end
	if obj:FindFirstChild("Knife", true) then return true end
	for _, child in obj:GetChildren() do
		if child.Name:sub(1, 10) == "Viewmodel_" then return true end
	end
	return false
end

local function get_kill_targets()
	local targets = {}
	for _, obj in workspace:GetChildren() do
		if is_ignored(obj) then continue end
		local name = obj.Name
		if name == "Hostile" or name == "Civilian" or name == "PMC" or name == "Cloaker" or name == "Guard" then
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
	for _, obj in char:GetChildren() do
		if obj:IsA("Tool") and obj:FindFirstChild("OnReload") and obj:FindFirstChild("OnHit") then
			return obj
		end
	end
end

local function create_tag(parent, text, color)
	if is_ignored(parent) or parent:FindFirstChild("ESP_Tag") then return end
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

local function apply_hl(obj, color)
	if is_ignored(obj) then return end
	if not obj:FindFirstChildOfClass("Highlight") then
		local hl = Instance.new("Highlight")
		hl.FillColor = color
		hl.OutlineColor = Color3.fromRGB(255, 255, 255)
		hl.FillTransparency = 0.5
		hl.Parent = obj
	end
end

local function remove_hl(obj)
	local hl = obj:FindFirstChildOfClass("Highlight")
	if hl then hl:Destroy() end
	local tag = obj:FindFirstChild("ESP_Tag")
	if tag then tag:Destroy() end
end

local function play_anim(obj)
	if not obj or is_ignored(obj) then return end
	local hl = Instance.new("Highlight")
	hl.FillColor = Color3.fromRGB(0, 180, 255)
	hl.OutlineColor = Color3.fromRGB(255, 255, 255)
	hl.FillTransparency = 1
	hl.Parent = obj

	local tw1 = ts:Create(hl, TweenInfo.new(0.2, Enum.EasingStyle.Linear), {FillTransparency = 0})
	tw1:Play()
	tw1.Completed:Wait()

	task.wait(0.5)

	local tw2 = ts:Create(hl, TweenInfo.new(1, Enum.EasingStyle.Linear), {FillTransparency = 1})
	tw2:Play()
	tw2.Completed:Wait()

	hl:Destroy()
end

local function has_defuse_trigger(grenade)
	if not grenade or is_ignored(grenade) then return false end
	local prompt = grenade:FindFirstChildWhichIsA("ProximityPrompt", true)
	if prompt and prompt.Parent and prompt.Enabled then return true end
	local remote = grenade:FindFirstChildWhichIsA("RemoteEvent", true)
	if remote and remote.Parent then return true end
	local click = grenade:FindFirstChildWhichIsA("ClickDetector", true)
	if click and click.Parent then return true end
	return false
end

local function has_refill_prompt(station)
	if not station or is_ignored(station) then return false end
	local prompt = station:FindFirstChild("Hitbox") and station.Hitbox:FindFirstChild("RefillPrompt")
	if prompt and prompt.Parent and prompt.Enabled then return true end
	local any_prompt = station:FindFirstChildWhichIsA("ProximityPrompt", true)
	if any_prompt and any_prompt.Parent and any_prompt.Enabled then return true end
	return false
end

local function update_stations()
	local map = workspace:FindFirstChild("Map")
	local stations = map and map:FindFirstChild("Stations")
	if not stations or is_ignored(stations) then return end

	for _, station in stations:GetChildren() do
		if is_ignored(station) or station.Name == "Optional" or station.Name == "PotentialStation" then continue end

		local show = true
		local name = station.Name
		if name == "AmmoStation" or name == "GadgetStation" or name == "HealthStation" then
			if not has_refill_prompt(station) then
				show = false
			end
		end

		if show then
			if not cache.stations[station] then
				cache.stations[station] = true
				apply_hl(station, Color3.fromRGB(255, 255, 0))
				create_tag(station, station.Name, Color3.fromRGB(255, 255, 0))
			end
		else
			if cache.stations[station] then
				cache.stations[station] = nil
				remove_hl(station)
			end
		end
	end
end

local function update_keycards()
	local map = workspace:FindFirstChild("Map")
	local camera_room = map and map:FindFirstChild("Geometry") and map.Geometry:FindFirstChild("CameraRoom")
	local keycard_spawns = camera_room and camera_room:FindFirstChild("KeycardSpawns")

	if not keycard_spawns or is_ignored(keycard_spawns) then return end

	for _, obj in keycard_spawns:GetChildren() do
		if is_ignored(obj) or obj.Name ~= "Keycard" then continue end
		local base = obj:FindFirstChild("Base")
		if not base or is_ignored(base) then continue end

		local prompt = base:FindFirstChild("GrabPrompt") or base:FindFirstChildWhichIsA("ProximityPrompt")
		if prompt and prompt.Parent and prompt.Enabled then
			if not cache.keycards[base] then
				cache.keycards[base] = true
				apply_hl(base, Color3.fromRGB(255, 215, 0))
				create_tag(base, "Keycard", Color3.fromRGB(255, 215, 0))
			end
		else
			if cache.keycards[base] then
				cache.keycards[base] = nil
				remove_hl(base)
			end
		end
	end
end

local function update_grenade_doors()
	local map = workspace:FindFirstChild("Map")
	local doors = map and map:FindFirstChild("Doors")
	if not doors or is_ignored(doors) then return end

	for _, door in doors:GetChildren() do
		if is_ignored(door) or door.Name ~= "Door" then continue end

		local grenade = door:FindFirstChild("Grenade")
		local is_mined = grenade and has_defuse_trigger(grenade)

		if is_mined then
			if not cache.grenade_doors[door] then
				cache.grenade_doors[door] = true
				apply_hl(door, Color3.fromRGB(255, 0, 128))
				create_tag(door, "Grenade Door", Color3.fromRGB(255, 0, 128))
			end
			local base = grenade:FindFirstChild("Base")
			if base and not is_ignored(base) and not cache.grenade_bases[base] then
				cache.grenade_bases[base] = true
				apply_hl(base, Color3.fromRGB(255, 0, 128))
				create_tag(base, "Grenade Base", Color3.fromRGB(255, 0, 128))
			end
		else
			if cache.grenade_doors[door] then
				cache.grenade_doors[door] = nil
				remove_hl(door)
			end
			if grenade then
				local base = grenade:FindFirstChild("Base")
				if base and cache.grenade_bases[base] then
					cache.grenade_bases[base] = nil
					remove_hl(base)
				end
			end
		end
	end
end

local function update_normal_doors()
	local map = workspace:FindFirstChild("Map")
	local doors = map and map:FindFirstChild("Doors")
	if not doors or is_ignored(doors) then return end

	for _, door in doors:GetChildren() do
		if is_ignored(door) or door.Name ~= "Door" then continue end

		if door:FindFirstChild("Grenade") or door:GetAttribute("Opened") == true then
			cache.normal_doors[door] = nil
			continue
		end

		local door_vis = door:FindFirstChild("DoorVisual")
		local main = door_vis and door_vis:FindFirstChild("Main")
		local open_prompt_obj = main and main:FindFirstChild("OpenPrompt")
		local prompt = open_prompt_obj and open_prompt_obj:FindFirstChild("ProximityPrompt")

		if prompt and prompt.Parent and prompt.Enabled then
			if not cache.normal_doors[door] then
				cache.normal_doors[door] = { prompt = prompt, used = false }
			else
				cache.normal_doors[door].prompt = prompt
			end
		else
			cache.normal_doors[door] = nil
		end
	end
end

local function update_stowaway()
	local stow = workspace:FindFirstChild("Stowaway")
	if stow and not is_ignored(stow) and cache.stowaway ~= stow then
		if cache.stowaway then remove_hl(cache.stowaway) end
		cache.stowaway = stow
		apply_hl(stow, Color3.fromRGB(0, 255, 0))
		create_tag(stow, "Stowaway", Color3.fromRGB(0, 255, 0))
	elseif not stow and cache.stowaway then
		remove_hl(cache.stowaway)
		cache.stowaway = nil
	end
end

local function update_plates()
	local map = workspace:FindFirstChild("Map")
	local search_folder = map or workspace
	if is_ignored(search_folder) then return end

	for _, obj in search_folder:GetChildren() do
		if is_ignored(obj) then continue end
		if obj.Name == "Plate" then
			if not cache.plates[obj] then
				cache.plates[obj] = true
				apply_hl(obj, Color3.fromRGB(0, 255, 128))
				create_tag(obj, "Plate", Color3.fromRGB(0, 255, 128))
			end
		else
			if cache.plates[obj] then
				cache.plates[obj] = nil
				remove_hl(obj)
			end
		end
	end
end

local function update_powerbox()
	local map = workspace:FindFirstChild("Map")
	local optional = map and map:FindFirstChild("Stations") and map.Stations:FindFirstChild("Optional")
	local powerbox = optional and optional:FindFirstChild("Powerbox")

	if not powerbox or is_ignored(powerbox) then
		if cache.powerbox.union then
			remove_hl(cache.powerbox.union)
			cache.powerbox.union = nil
			cache.powerbox.prompt = nil
		end
		return
	end

	local union = powerbox:FindFirstChild("Union")
	local model = powerbox:FindFirstChild("Model")
	local prompt = model and model:FindFirstChild("Handle") and model.Handle:FindFirstChild("ProximityPrompt")

	if union and prompt and prompt.Parent and prompt.Enabled then
		if cache.powerbox.union ~= union then
			if cache.powerbox.union then remove_hl(cache.powerbox.union) end
			cache.powerbox.union = union
			apply_hl(union, Color3.fromRGB(255, 165, 0))
			create_tag(union, "Powerbox", Color3.fromRGB(255, 165, 0))
		end
		cache.powerbox.prompt = prompt
	else
		if cache.powerbox.union then
			remove_hl(cache.powerbox.union)
			cache.powerbox.union = nil
			cache.powerbox.prompt = nil
		end
	end
end

local function update_control_lever()
	local map = workspace:FindFirstChild("Map")
	local lever = map and map:FindFirstChild("Objectives") and map.Objectives:FindFirstChild("ControlLever")

	if not lever or is_ignored(lever) then
		if cache.control_lever.part then
			remove_hl(cache.control_lever.part)
			cache.control_lever.part = nil
			cache.control_lever.prompt = nil
		end
		return
	end

	local handle = lever:FindFirstChild("Handle")
	local prompt = handle and handle:FindFirstChild("ProximityPrompt")

	if handle and prompt and prompt.Parent and prompt.Enabled then
		if cache.control_lever.part ~= handle then
			if cache.control_lever.part then remove_hl(cache.control_lever.part) end
			cache.control_lever.part = handle
			apply_hl(handle, Color3.fromRGB(255, 0, 255))
			create_tag(handle, "Control Lever", Color3.fromRGB(255, 0, 255))
		end
		cache.control_lever.prompt = prompt
	else
		if cache.control_lever.part then
			remove_hl(cache.control_lever.part)
			cache.control_lever.part = nil
			cache.control_lever.prompt = nil
		end
	end
end

local function update_radio()
	local map = workspace:FindFirstChild("Map")
	local radio = map and map:FindFirstChild("Objectives") and map.Objectives:FindFirstChild("Radio")

	if not radio or is_ignored(radio) or cache.radio.activated then
		if cache.radio.part then
			remove_hl(cache.radio.part)
			cache.radio.part = nil
			cache.radio.prompt_handle = nil
			cache.radio.prompt_hitbox = nil
		end
		return
	end

	local handle = radio:FindFirstChild("Handle")
	local handle_prompt = handle and handle:FindFirstChild("ProximityPrompt")
	local hitbox = radio:FindFirstChild("Hitbox")
	local hitbox_prompt = hitbox and hitbox:FindFirstChild("ProximityPrompt")

	local active = (handle_prompt and handle_prompt.Parent and handle_prompt.Enabled) or (hitbox_prompt and hitbox_prompt.Parent and hitbox_prompt.Enabled)

	if active then
		local part_to_hl = handle or hitbox
		if part_to_hl and cache.radio.part ~= part_to_hl then
			if cache.radio.part then remove_hl(cache.radio.part) end
			cache.radio.part = part_to_hl
			apply_hl(part_to_hl, Color3.fromRGB(255, 200, 0))
			create_tag(part_to_hl, "Radio", Color3.fromRGB(255, 200, 0))
		end
		cache.radio.prompt_handle = handle_prompt
		cache.radio.prompt_hitbox = hitbox_prompt
	else
		if cache.radio.part then
			remove_hl(cache.radio.part)
			cache.radio.part = nil
			cache.radio.prompt_handle = nil
			cache.radio.prompt_hitbox = nil
		end
	end
end

local function update_enemies()
	if not cfg.HighlightEnemies then return end
	local char = get_char()

	for _, obj in workspace:GetChildren() do
		if obj == char or is_ignored(obj) then continue end

		local is_civilian = (obj.Name == "Civilian")
		local is_hostile = (obj.Name == "Hostile" or obj.Name == "PMC" or obj.Name == "Cloaker")
		local is_humanoid = obj:FindFirstChildOfClass("Humanoid") and not is_civilian and not is_hostile

		if is_civilian then
			if not cache.enemies[obj] then
				cache.enemies[obj] = true
				apply_hl(obj, Color3.fromRGB(255, 255, 255))
			end
		elseif is_hostile or is_humanoid then
			if not cache.enemies[obj] then
				cache.enemies[obj] = true
				apply_hl(obj, Color3.fromRGB(255, 0, 0))
			end
		else
			if cache.enemies[obj] then
				cache.enemies[obj] = nil
				remove_hl(obj)
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
	update_enemies()
end

task.spawn(function()
	while task.wait(cfg.ESP_INTERVAL) do
		pcall(update_esp)
	end
end)

local ws_loop = nil
local ws_ca = nil

local function setup_walkspeed()
	local char = get_char()
	if not char or is_ignored(char) then return end
	local hum = char:FindFirstChildOfClass("Humanoid")
	if not hum then return end

	local function apply_speed()
		if hum and hum.WalkSpeed ~= cfg.WalkSpeed then
			hum.WalkSpeed = cfg.WalkSpeed
		end
	end

	if ws_loop then ws_loop:Disconnect() end
	ws_loop = hum:GetPropertyChangedSignal("WalkSpeed"):Connect(apply_speed)
	apply_speed()

	if ws_ca then ws_ca:Disconnect() end
	ws_ca = plr.CharacterAdded:Connect(function(new_char)
		if is_ignored(new_char) then return end
		hum = new_char:WaitForChild("Humanoid")
		if ws_loop then ws_loop:Disconnect() end
		ws_loop = hum:GetPropertyChangedSignal("WalkSpeed"):Connect(apply_speed)
		apply_speed()
	end)
end

plr.CharacterAdded:Connect(setup_walkspeed)
if plr.Character then
	task.wait(0.5)
	setup_walkspeed()
end

local function refill_stations(pos)
	if not cfg.AutoRefill or not pos then return end
	local map = workspace:FindFirstChild("Map")
	local stations = map and map:FindFirstChild("Stations")
	if not stations or is_ignored(stations) then return end

	for _, station in stations:GetChildren() do
		if is_ignored(station) or station.Name == "Optional" or station.Name == "PotentialStation" then continue end

		local prompt = station:FindFirstChild("Hitbox") and station.Hitbox:FindFirstChild("RefillPrompt")
		if not prompt then
			prompt = station:FindFirstChildWhichIsA("ProximityPrompt", true)
		end

		if prompt and prompt:IsA("ProximityPrompt") and prompt.Parent and prompt.Enabled then
			local dist = (station:GetPivot().Position - pos).Magnitude
			if dist <= cfg.InteractionRadius then
				local ok = pcall(function() fireproximityprompt(prompt) end)
				if ok then
					remove_hl(station)
					cache.stations[station] = nil
					task.spawn(function() play_anim(station) end)
				end
			end
		end
	end
end

local function activate_optional_stations(pos)
	if not cfg.AutoOptionalStations or not pos then return end
	local map = workspace:FindFirstChild("Map")
	local optional = map and map:FindFirstChild("Stations") and map.Stations:FindFirstChild("Optional")
	if not optional or is_ignored(optional) then return end

	for _, model in optional:GetChildren() do
		if is_ignored(model) or model.Name == "PotentialStation" then continue end
		if not model:IsA("BasePart") and not model:IsA("Model") then continue end

		local prompt = model:FindFirstChildWhichIsA("ProximityPrompt", true)
		if prompt and prompt:IsA("ProximityPrompt") and prompt.Parent and prompt.Enabled then
			local part = model:FindFirstChild("HumanoidRootPart") or model:FindFirstChild("Handle") or model:FindFirstChild("Base") or model:FindFirstChildOfClass("BasePart")
			local dist = part and (part.Position - pos).Magnitude or (model:GetPivot().Position - pos).Magnitude

			if dist <= cfg.InteractionRadius then
				local ok = pcall(function() fireproximityprompt(prompt) end)
				if ok then
					task.spawn(function() play_anim(model) end)
				end
			end
		end
	end
end

local function zipties(pos)
	if not cfg.AutoZiptie or not pos then return end
	local tool = get_tool("Zipties")
	if not tool then return end
	local event = tool:FindFirstChild("Use")
	if not event then return end

	for _, obj in workspace:GetChildren() do
		if is_ignored(obj) then continue end
		if obj.Name == "Hostile" or obj.Name == "Civilian" then
			local torso = obj:FindFirstChild("HumanoidRootPart") or obj:FindFirstChild("Torso") or obj:FindFirstChild("UpperTorso")
			if torso and (torso.Position - pos).Magnitude <= cfg.InteractionRadius then
				pcall(function() event:FireServer(obj) end)
			end
		end
	end
end

local function kill()
	if not cfg.AutoKill then return end
	local weapon = get_weapon()
	if not weapon then return end
	local event = weapon:FindFirstChild("OnHit")
	if not event then return end

	for _, target in get_kill_targets() do
		if is_ignored(target) then continue end
		local hum = target:FindFirstChildOfClass("Humanoid")
		local torso = target:FindFirstChild("Torso") or target:FindFirstChild("UpperTorso") or target:FindFirstChild("HumanoidRootPart")
		if hum and hum.Health > 0 and torso then
			pcall(function()
				event:FireServer(hum, torso.Position, torso, false)
			end)
		end
	end
end

local function lockpick(pos)
	if not cfg.AutoLockpick or not pos then return end
	local tool = get_tool("Lockpick")
	if not tool then return end
	local event = tool:FindFirstChild("Unlock")
	if not event then return end
	local map = workspace:FindFirstChild("Map")
	local doors = map and map:FindFirstChild("Doors")
	if not doors or is_ignored(doors) then return end

	for _, door in doors:GetChildren() do
		if is_ignored(door) or door.Name ~= "Door" or door:FindFirstChild("Grenade") then continue end

		local locked = door:GetAttribute("Locked")
		if locked == nil then
			local locked_val = door:FindFirstChild("Locked")
			locked = locked_val and locked_val:IsA("BoolValue") and locked_val.Value or true
		end
		if not locked then continue end

		if (door:GetPivot().Position - pos).Magnitude <= cfg.InteractionRadius then
			local ok = pcall(function() event:FireServer(door) end)
			if ok then
				task.spawn(function() play_anim(door) end)
			end
		end
	end
end

local function defuse_grenades(pos)
	if not cfg.AutoDefuse or not pos then return end
	local map = workspace:FindFirstChild("Map")
	local doors = map and map:FindFirstChild("Doors")
	if not doors or is_ignored(doors) then return end

	for _, door in doors:GetChildren() do
		if is_ignored(door) or door.Name ~= "Door" then continue end
		local grenade = door:FindFirstChild("Grenade")
		if not grenade or is_ignored(grenade) then continue end

		if (door:GetPivot().Position - pos).Magnitude > cfg.InteractionRadius then continue end

		local prompt = grenade:FindFirstChildWhichIsA("ProximityPrompt", true)
		if not prompt or not prompt.Parent or not prompt.Enabled then continue end

		local activated = false
		if pcall(function() fireproximityprompt(prompt) end) then
			activated = true
		end

		if not activated then
			local remote = grenade:FindFirstChildWhichIsA("RemoteEvent", true)
			if remote then
				if pcall(function() remote:FireServer() end) then activated = true end
			else
				local click = grenade:FindFirstChildWhichIsA("ClickDetector", true)
				if click and pcall(function() fireclickdetector(click) end) then activated = true end
			end
		end

		if activated then
			remove_hl(door)
			cache.grenade_doors[door] = nil
			local base = grenade:FindFirstChild("Base")
			if base then
				remove_hl(base)
				cache.grenade_bases[base] = nil
			end
		end
	end
end

local function open_normal_doors(pos)
	if not cfg.AutoOpenDoors or not pos then return end

	for door, data in cache.normal_doors do
		if not door.Parent or is_ignored(door) then
			cache.normal_doors[door] = nil
			continue
		end

		if data.used then continue end

		if door:GetAttribute("Opened") == true then
			cache.normal_doors[door] = nil
			continue
		end

		if door:FindFirstChild("Grenade") then
			cache.normal_doors[door] = nil
			continue
		end

		local locked = door:GetAttribute("Locked")
		if locked == nil then
			local locked_val = door:FindFirstChild("Locked")
			locked = locked_val and locked_val:IsA("BoolValue") and locked_val.Value or false
		end
		if locked then continue end

		local dist = (door:GetPivot().Position - pos).Magnitude
		if dist <= cfg.InteractionRadius then
			local prompt = data.prompt
			if prompt and prompt.Parent and prompt.Enabled then
				local ok = pcall(function() fireproximityprompt(prompt) end)
				if ok then
					data.used = true
					cache.normal_doors[door] = nil
					task.spawn(function() play_anim(door) end)
				end
			else
				cache.normal_doors[door] = nil
			end
		end
	end
end

local function plates(pos)
	if not cfg.AutoPlates or not pos then return end
	local map = workspace:FindFirstChild("Map")
	local search_folder = map or workspace
	if is_ignored(search_folder) then return end

	for _, obj in search_folder:GetChildren() do
		if is_ignored(obj) or obj.Name ~= "Plate" then continue end
		if (obj:GetPivot().Position - pos).Magnitude <= cfg.InteractionRadius then
			local prompt = obj:FindFirstChildWhichIsA("ProximityPrompt", true)
			if prompt and prompt:IsA("ProximityPrompt") and prompt.Parent and prompt.Enabled then
				pcall(function() fireproximityprompt(prompt) end)
			end
		end
	end
end

local function grab_keycards(pos)
	if not cfg.AutoGrabKeycard or not pos then return end
	local map = workspace:FindFirstChild("Map")
	local camera_room = map and map:FindFirstChild("Geometry") and map.Geometry:FindFirstChild("CameraRoom")
	local keycard_spawns = camera_room and camera_room:FindFirstChild("KeycardSpawns")
	if not keycard_spawns or is_ignored(keycard_spawns) then return end

	for _, obj in keycard_spawns:GetChildren() do
		if is_ignored(obj) or obj.Name ~= "Keycard" then continue end
		local base = obj:FindFirstChild("Base")
		if not base or is_ignored(base) then continue end

		if (base.Position - pos).Magnitude <= cfg.InteractionRadius then
			local prompt = base:FindFirstChild("GrabPrompt") or base:FindFirstChildWhichIsA("ProximityPrompt")
			if prompt and prompt:IsA("ProximityPrompt") and prompt.Parent and prompt.Enabled then
				pcall(function() fireproximityprompt(prompt) end)
			end
		end
	end
end

local function activate_stowaway(pos)
	if not cfg.AutoStowaway or not pos then return end
	local stow = workspace:FindFirstChild("Stowaway")
	if not stow or is_ignored(stow) then return end
	local hrp = stow:FindFirstChild("HumanoidRootPart")
	if not hrp or is_ignored(hrp) then return end

	if (hrp.Position - pos).Magnitude <= 6 then
		local prompt = stow:FindFirstChildWhichIsA("ProximityPrompt", true)
		if prompt and prompt:IsA("ProximityPrompt") and prompt.Parent and prompt.Enabled then
			pcall(function() fireproximityprompt(prompt) end)
		end
	end
end

local function activate_powerbox(pos)
	if not cfg.AutoPowerbox or not pos then return end
	local union = cache.powerbox.union
	local prompt = cache.powerbox.prompt
	if not union or is_ignored(union) or not prompt or not prompt.Parent or not prompt.Enabled then return end

	if (union.Position - pos).Magnitude <= cfg.InteractionRadius then
		local ok = pcall(function() fireproximityprompt(prompt) end)
		if ok then
			remove_hl(union)
			cache.powerbox.union = nil
			cache.powerbox.prompt = nil
			task.spawn(function() play_anim(union) end)
		end
	end
end

local function activate_control_lever(pos)
	if not cfg.AutoControlLever or not pos then return end
	local part = cache.control_lever.part
	local prompt = cache.control_lever.prompt
	if not part or is_ignored(part) or not prompt or not prompt.Parent or not prompt.Enabled then return end

	if (part.Position - pos).Magnitude <= cfg.InteractionRadius then
		local ok = pcall(function() fireproximityprompt(prompt) end)
		if ok then
			remove_hl(part)
			cache.control_lever.part = nil
			cache.control_lever.prompt = nil
			task.spawn(function() play_anim(part) end)
		end
	end
end

local function activate_radio(pos)
	if not cfg.AutoRadio or not pos then return end

	local map = workspace:FindFirstChild("Map")
	local radio = map and map:FindFirstChild("Objectives") and map.Objectives:FindFirstChild("Radio")
	local handle = radio and radio:FindFirstChild("Handle")
	local prompt = handle and handle:FindFirstChild("ProximityPrompt")

	if prompt and prompt.Parent and prompt.Enabled then
		if (handle.Position - pos).Magnitude <= cfg.InteractionRadius then
			pcall(function()
				fireproximityprompt(prompt)
			end)
		end
	end
end

local function activate_explosives(pos)
	if not cfg.AutoDefuse or not pos then
		return
	end

	local explosives = {
		workspace.Map.Objectives:FindFirstChild("Explosive1"),
		workspace.Map.Objectives:FindFirstChild("Explosive2")
	}

	for _, explosive in ipairs(explosives) do
		if not explosive or is_ignored(explosive) then
			continue
		end

		local handle = explosive:FindFirstChild("Handle")
		local prompt = handle and handle:FindFirstChild("ProximityPrompt")

		if not handle or not prompt or not prompt.Parent or not prompt.Enabled then
			continue
		end

		if (handle.Position - pos).Magnitude > cfg.InteractionRadius then
			continue
		end

		local ok = pcall(function()
			fireproximityprompt(prompt)
		end)

		if ok then
			task.spawn(function()
				play_anim(explosive)
			end)
		end
	end
end

local function intimidate()
	if not cfg.AutoIntimidate or not intimidate_remote then return end
	for i = 0, 11 do
		local angle = math.rad(i * 30)
		pcall(function()
			intimidate_remote:FireServer(Vector3.new(math.cos(angle), 0, math.sin(angle)))
		end)
	end
end

local function find(path)
    local obj = workspace

    for _, name in path do
        obj = obj:FindFirstChild(name)

        if not obj then
            return nil
        end
    end

    return obj
end


-- autotween
-- autotween
local autofarm = false
if autofarm == true then
task.spawn(function()
    print("[AUTOTWEEN] START")

    local rs = game:GetService("ReplicatedStorage")
    local remotes = rs:WaitForChild("Remotes")

    local briefing = remotes:WaitForChild("Briefing")
    local set_ready = briefing:WaitForChild("SetReady")
    local skip_vote = briefing:WaitForChild("CutsceneSkipVote")

    local teleport = remotes:WaitForChild("Teleport")
    local replay_remote = teleport:WaitForChild("Replay")

    -- =========================================================
    -- 1. START
    -- =========================================================

    pcall(function()
        set_ready:FireServer()
    end)

    task.wait(0.3)

    pcall(function()
        skip_vote:FireServer()
    end)

    -- Если персонажа нет, повторяем SetReady раз в 10 секунд
    while true do
        local char = plr.Character

        if char and char.Parent == workspace then
            break
        end

        task.wait(10)

        if not plr.Character or plr.Character.Parent ~= workspace then
            pcall(function()
                set_ready:FireServer()
            end)
        end
    end

    print("[AUTOTWEEN] CHARACTER SPAWNED")

    -- =========================================================
    -- 2. DEATH -> REPLAY
    -- =========================================================

    task.spawn(function()
        local sent = false

        while task.wait(0.5) do
            local char = plr.Character
            local hum = char and char:FindFirstChildOfClass("Humanoid")

            if hum and hum.Health <= 0 then
                if not sent then
                    sent = true

                    print("[AUTOTWEEN] DEAD -> REPLAY")

                    pcall(function()
                        replay_remote:InvokeServer()
                    end)
                end
            else
                sent = false
            end
        end
    end)

    -- =========================================================
    -- 3. KEYCARD
    -- =========================================================

    while true do
        local keycard = find({
            "Map",
            "Geometry",
            "CameraRoom",
            "KeycardSpawns",
            "Keycard",
            "Base"
        })

        if not keycard then
            break
        end

        local root = get_root()

        if root then
            tween_player(keycard, 15)
        end

        task.wait(0.1)
    end

    print("[AUTOTWEEN] KEYCARD DONE")

    -- =========================================================
    -- 4. CONTROL LEVER
    -- =========================================================

    while true do
        local prompt = find({
            "Map",
            "Objectives",
            "ControlLever",
            "Handle",
            "ProximityPrompt"
        })

        if prompt and prompt.Parent and prompt.Enabled then
            local handle = prompt.Parent

            tween_player(handle, 15)

            local root = get_root()

            if root then
                activate_control_lever(root.Position)
            end

            -- Ждем именно изменения состояния
            repeat
                task.wait(0.1)
            until not prompt.Parent or not prompt.Enabled

            break
        end

        task.wait(0.1)
    end

    print("[AUTOTWEEN] CONTROL DONE")

    -- =========================================================
    -- 5. RADIO
    -- =========================================================

    while true do
        local prompt = find({
            "Map",
            "Objectives",
            "Radio",
            "Handle",
            "ProximityPrompt"
        })

        if prompt and prompt.Parent and prompt.Enabled then
            tween_player(prompt.Parent, 15)

            pcall(function()
                fireproximityprompt(prompt)
            end)

            repeat
                task.wait(0.1)
            until not prompt.Parent or not prompt.Enabled

            break
        end

        task.wait(0.1)
    end

    print("[AUTOTWEEN] RADIO DONE")

    -- =========================================================
    -- 6. EXPLOSIVE 1
    -- =========================================================

    while true do
        local prompt = find({
            "Map",
            "Objectives",
            "Radar",
            "Explosive1",
            "Handle",
            "ProximityPrompt"
        })

        if prompt and prompt.Parent and prompt.Enabled then
            tween_player(prompt.Parent, 15)

            pcall(function()
                fireproximityprompt(prompt)
            end)

            repeat
                task.wait(0.1)
            until not prompt.Parent or not prompt.Enabled

            break
        end

        task.wait(0.1)
    end

    print("[AUTOTWEEN] EXPLOSIVE 1 DONE")

    -- =========================================================
    -- 7. EXPLOSIVE 2
    -- =========================================================

    while true do
        local prompt = find({
            "Map",
            "Objectives",
            "Radar",
            "Explosive2",
            "Handle",
            "ProximityPrompt"
        })

        if prompt and prompt.Parent and prompt.Enabled then
            tween_player(prompt.Parent, 15)

            pcall(function()
                fireproximityprompt(prompt)
            end)

            repeat
                task.wait(0.1)
            until not prompt.Parent or not prompt.Enabled

            break
        end

        task.wait(0.1)
    end

    print("[AUTOTWEEN] EXPLOSIVE 2 DONE")

    -- =========================================================
    -- 8. HELIPAD
    -- =========================================================
    -- ВАЖНО:
    -- здесь НЕТ break после tween_player.
    -- Персонаж постоянно возвращается на точку,
    -- пока MissionComplete не появится.
    -- =========================================================

    while true do
        -- Проверяем завершение миссии
        local gui = plr:FindFirstChild("PlayerGui")
        local complete = gui and gui:FindFirstChild("MissionComplete", true)

        if complete then
            print("[AUTOTWEEN] MISSION COMPLETE")

            task.wait(10)

            pcall(function()
                replay_remote:InvokeServer()
            end)

            print("[AUTOTWEEN] REPLAY")

            break
        end

        -- Ищем точку вертолета
        local helipad = find({
            "Map",
            "Geometry",
            "HeliPadRoom"
        })

        if helipad then
            local children = helipad:GetChildren()
            local obj = children[38]

            if obj then
                local children2 = obj:GetChildren()
                local target = children2[4]

                if target then
                    local root = get_root()

                    if root then
                        -- Возвращаем персонажа на точку
                        -- каждые 0.1 сек, пока миссия не закончена
                        tween_player(target, 15)
                    end
                end
            end
        end

        task.wait(0.1)
    end

    print("[AUTOTWEEN] FINISHED")
end)
end

task.spawn(function()
	local intimidate_timer = 0
	while task.wait(0.1) do
		local root = get_root()
		local pos = root and root.Position

		kill()
		if pos then
			zipties(pos)
			lockpick(pos)
			plates(pos)
			grab_keycards(pos)
			defuse_grenades(pos)
			activate_stowaway(pos)
			refill_stations(pos)
			activate_powerbox(pos)
			activate_explosives(pos)
			activate_control_lever(pos)
			activate_radio(pos)
			open_normal_doors(pos)
			activate_optional_stations(pos)
		end

		intimidate_timer += 1
		if intimidate_timer >= 4 then
			intimidate()
			intimidate_timer = 0
		end
	end
end)

getgenv().AutoFarmUnhook = function()
	if ws_loop then ws_loop:Disconnect() end
	if ws_ca then ws_ca:Disconnect() end
	for _, obj in workspace:GetDescendants() do
		if is_ignored(obj) then continue end
		local tag = obj:FindFirstChild("ESP_Tag")
		if tag then tag:Destroy() end
		local hl = obj:FindFirstChildOfClass("Highlight")
		if hl then hl:Destroy() end
	end
	getgenv().AutoFarmUnhook = nil
end
