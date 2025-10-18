-- Vixfer_Menu_System
-- Pegar este Script en ServerScriptService. Al correr, creará RemoteEvents y dos scripts:
--  - Server handler (servidor)
--  - Client UI (LocalScript en StarterPlayerScripts)
-- Diseñado para tu propio juego, legal y sin exploits.

local ReplicatedStorage = game:GetService("ReplicatedStorage")
local StarterPlayer = game:GetService("StarterPlayer")
local ServerStorage = game:GetService("ServerStorage")
local Workspace = game:GetService("Workspace")

-- Helper: create folder if not exists
local function ensureFolder(parent, name)
    local f = parent:FindFirstChild(name)
    if not f then
        f = Instance.new("Folder")
        f.Name = name
        f.Parent = parent
    end
    return f
end

-- Create RemoteEvents container
local remoteFolder = ensureFolder(ReplicatedStorage, "Vixfer_RemoteEvents")
local function ensureEvent(name)
    local e = remoteFolder:FindFirstChild(name)
    if not e then
        e = Instance.new("RemoteEvent")
        e.Name = name
        e.Parent = remoteFolder
    end
    return e
end

local RE_RequestRole      = ensureEvent("RequestRole")
local RE_RequestTeleport  = ensureEvent("RequestTeleport")
local RE_ChangeSpeed      = ensureEvent("ChangeSpeed")
local RE_RequestInventory = ensureEvent("RequestInventory")
local RE_ToggleAmbient    = ensureEvent("ToggleAmbient")
local RE_RequestRoundInfo = ensureEvent("RequestRoundInfo")
local RE_ToggleSpecMode   = ensureEvent("ToggleSpecMode")
local RE_ValidateLight    = ensureEvent("ValidateFlashlight") -- server can validate battery, etc.
local RE_CameraChange     = ensureEvent("CameraChange")
local RE_ToggleMenu       = ensureEvent("ToggleMenu") -- optional server hook

-- Ensure Tools folder in ReplicatedStorage (for WeaponDistributor demo)
local toolsFolder = ensureFolder(ReplicatedStorage, "Tools")

-- SERVER SCRIPT (will be created as child script)
local serverSource = [[
-- Vixfer_ServerHandler (auto)
local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Workspace = game:GetService("Workspace")
local toolsFolder = ReplicatedStorage:WaitForChild("Tools")
local remotes = ReplicatedStorage:WaitForChild("Vixfer_RemoteEvents")

local RE_RequestRole = remotes:WaitForChild("RequestRole")
local RE_RequestTeleport = remotes:WaitForChild("RequestTeleport")
local RE_ChangeSpeed = remotes:WaitForChild("ChangeSpeed")
local RE_RequestInventory = remotes:WaitForChild("RequestInventory")
local RE_ToggleAmbient = remotes:WaitForChild("ToggleAmbient")
local RE_RequestRoundInfo = remotes:WaitForChild("RequestRoundInfo")
local RE_ToggleSpecMode = remotes:WaitForChild("ToggleSpecMode")
local RE_ValidateLight = remotes:WaitForChild("ValidateFlashlight")
local RE_CameraChange = remotes:WaitForChild("CameraChange")

-- Basic role assignment & round manager (simple, for testing)
local ROUND_TIME = 90
local roundActive = false
local nightNumber = 0

local function clearRoles()
    for _,p in pairs(Players:GetPlayers()) do
        p:SetAttribute("Role", nil)
    end
end

local function assignRoles()
    local all = Players:GetPlayers()
    if #all == 0 then return end
    clearRoles()
    -- 1 murderer
    local murderer = all[math.random(1,#all)]
    murderer:SetAttribute("Role","Murderer")
    -- 1 sheriff if >= 2 players
    if #all >= 2 then
        local candidate
        repeat
            candidate = all[math.random(1,#all)]
        until candidate ~= murderer
        candidate:SetAttribute("Role","Sheriff")
    end
    for _,p in pairs(all) do
        if not p:GetAttribute("Role") then p:SetAttribute("Role","Innocent") end
    end
end

-- Round loop (non-blocking)
spawn(function()
    while true do
        wait(5)
        -- start round when >=1 player (you can change condition)
        if not roundActive and #Players:GetPlayers() > 0 then
            roundActive = true
            nightNumber = nightNumber + 1
            Workspace:SetAttribute("Night", nightNumber)
            -- assign roles
            assignRoles()
            -- notify clients about round start (clients can request roles)
            RE_RequestRoundInfo:FireAllClients({night = nightNumber, time = ROUND_TIME})
            -- round timer
            wait(ROUND_TIME)
            -- end round
            roundActive = false
            clearRoles()
            -- optional small pause
            wait(6)
        end
    end
end)

-- Remote handlers
RE_RequestRole.OnServerEvent:Connect(function(player)
    local role = player:GetAttribute("Role") or "None"
    RE_RequestRole:FireClient(player, role)
end)

RE_RequestTeleport.OnServerEvent:Connect(function(player, tpName)
    if typeof(tpName) ~= "string" then return end
    local teleports = Workspace:FindFirstChild("Teleports")
    if not teleports then return end
    local part = teleports:FindFirstChild(tpName)
    if part and part:IsA("BasePart") then
        local char = player.Character
        if char and char:FindFirstChild("HumanoidRootPart") then
            char:SetPrimaryPartCFrame(part.CFrame + Vector3.new(0,3,0))
        end
    end
end)

RE_ChangeSpeed.OnServerEvent:Connect(function(player, newSpeed)
    if typeof(newSpeed) ~= "number" then return end
    -- basic validation
    if newSpeed < 8 then newSpeed = 8 end
    if newSpeed > 40 then newSpeed = 40 end
    -- apply to humanoid if exists
    local char = player.Character
    if char and char:FindFirstChild("Humanoid") then
        char.Humanoid.WalkSpeed = newSpeed
    end
end)

RE_RequestInventory.OnServerEvent:Connect(function(player)
    -- send list of tool names in backpack
    local inv = {}
    for _,it in pairs(player.Backpack:GetChildren()) do
        if it:IsA("Tool") then table.insert(inv, it.Name) end
    end
    RE_RequestInventory:FireClient(player, inv)
end)

RE_ToggleAmbient.OnServerEvent:Connect(function(player, on)
    -- simple broadcast to all: toggle ambient audio
    RE_ToggleAmbient:FireAllClients(on)
end)

RE_RequestRoundInfo.OnServerEvent:Connect(function(player)
    RE_RequestRoundInfo:FireClient(player, {night = Workspace:GetAttribute("Night") or 0, time = ROUND_TIME, active = roundActive})
end)

RE_ToggleSpecMode.OnServerEvent:Connect(function(player, on)
    -- here server could mark attribute; clients implement camera change
    player:SetAttribute("Spectator", on and true or false)
end)

RE_ValidateLight.OnServerEvent:Connect(function(player, state)
    -- placeholder: server could validate battery, protect abuse
    -- we just replicate state back to client (ack)
    RE_ValidateLight:FireClient(player, state)
end)

RE_CameraChange.OnServerEvent:Connect(function(player, mode)
    -- store preferred camera mode
    player:SetAttribute("CameraMode", mode)
end)

-- Simple PlayerAdded: load default attributes
Players.PlayerAdded:Connect(function(p)
    p:SetAttribute("Role", nil)
    p:SetAttribute("Spectator", false)
    p:SetAttribute("CameraMode", "Default")
end)
]]

-- CLIENT LOCALSCRIPT (will be created in StarterPlayerScripts)
local clientSource = [[
-- Vixfer_ClientUI (auto)
-- LocalScript to place in StarterPlayerScripts. Builds UI and hotkeys.
local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local UserInputService = game:GetService("UserInputService")
local player = Players.LocalPlayer
local playerGui = player:WaitForChild("PlayerGui")
local remotes = ReplicatedStorage:WaitForChild("Vixfer_RemoteEvents")

local RE_RequestRole = remotes:WaitForChild("RequestRole")
local RE_RequestTeleport = remotes:WaitForChild("RequestTeleport")
local RE_ChangeSpeed = remotes:WaitForChild("ChangeSpeed")
local RE_RequestInventory = remotes:WaitForChild("RequestInventory")
local RE_ToggleAmbient = remotes:WaitForChild("ToggleAmbient")
local RE_RequestRoundInfo = remotes:WaitForChild("RequestRoundInfo")
local RE_ToggleSpecMode = remotes:WaitForChild("ToggleSpecMode")
local RE_ValidateLight = remotes:WaitForChild("ValidateFlashlight")
local RE_CameraChange = remotes:WaitForChild("CameraChange")
local RE_ToggleMenu = remotes:FindFirstChild("ToggleMenu")

-- create ScreenGui
local screenGui = Instance.new("ScreenGui")
screenGui.Name = "VixferMainGui"
screenGui.Parent = playerGui
screenGui.ResetOnSpawn = false

-- main frame (dark + neon blue accent)
local main = Instance.new("Frame", screenGui)
main.Name = "MainPanel"
main.Size = UDim2.new(0,860,0,520)
main.Position = UDim2.new(0.5, -430, 0.08, 0)
main.BackgroundColor3 = Color3.fromRGB(12,12,18)
main.BorderSizePixel = 0
main.AnchorPoint = Vector2.new(0.5, 0)

-- title
local title = Instance.new("TextLabel", main)
title.Size = UDim2.new(1, -20, 0, 64)
title.Position = UDim2.new(0,10,0,10)
title.BackgroundTransparency = 1
title.Font = Enum.Font.Bebas
title.TextScaled = true
title.Text = "99 NOCHES EN EL BOSQUE — vixfer0511"
title.TextColor3 = Color3.fromRGB(180,220,255)
title.TextStrokeTransparency = 0.7

-- left panel: profiles + inventory
local left = Instance.new("Frame", main)
left.Size = UDim2.new(0.45, -20, 0.78, -20)
left.Position = UDim2.new(0.025, 0, 0.14, 0)
left.BackgroundTransparency = 0.2
left.BackgroundColor3 = Color3.fromRGB(18,18,24)

local profLabel = Instance.new("TextLabel", left)
profLabel.Size = UDim2.new(1, -10, 0, 36)
profLabel.Position = UDim2.new(0,5,0,5)
profLabel.BackgroundTransparency = 1
profLabel.Font = Enum.Font.GothamBold
profLabel.Text = "Perfiles"
profLabel.TextColor3 = Color3.fromRGB(220,220,230)

local profilesList = Instance.new("ScrollingFrame", left)
profilesList.Size = UDim2.new(1, -10, 0.35, -10)
profilesList.Position = UDim2.new(0,5,0,46)
profilesList.CanvasSize = UDim2.new(0,0,0,0)
profilesList.BackgroundTransparency = 0.5

-- sample profiles
local sampleProfiles = {
    {name="vixfer0511", desc="Creador - Admin"},
    {name="Nocturno", desc="Runner"},
    {name="Guardia", desc="Sheriff main"},
    {name="Espía", desc="Inocente"}
}
for i,p in ipairs(sampleProfiles) do
    local b = Instance.new("TextButton", profilesList)
    b.Size = UDim2.new(1, -10, 0, 48)
    b.Position = UDim2.new(0,5,0,(i-1)*56 + 5)
    b.Text = p.name.." — "..p.desc
    b.BackgroundTransparency = 0.4
    b.Font = Enum.Font.Gotham
    b.TextColor3 = Color3.fromRGB(240,240,240)
end
profilesList.CanvasSize = UDim2.new(0,0,0,#sampleProfiles*56)

-- inventory display
local invLabel = Instance.new("TextLabel", left)
invLabel.Size = UDim2.new(1, -10, 0, 28)
invLabel.Position = UDim2.new(0,5,0, (0.35*left.Size.Y.Offset) + 60)
invLabel.BackgroundTransparency = 1
invLabel.Font = Enum.Font.Gotham
invLabel.Text = "Inventario (presiona 'Inventario')"
invLabel.TextColor3 = Color3.fromRGB(200,200,200)

local invList = Instance.new("TextLabel", left)
invList.Size = UDim2.new(1, -10, 0, 120)
invList.Position = UDim2.new(0,5,0, (0.35*left.Size.Y.Offset) + 90)
invList.BackgroundTransparency = 0.6
invList.Font = Enum.Font.Gotham
invList.TextWrapped = true
invList.RichText = true
invList.Text = "Sin datos"
invList.TextColor3 = Color3.fromRGB(230,230,230)

-- right panel: actions & controls
local right = Instance.new("Frame", main)
right.Size = UDim2.new(0.48, -20, 0.78, -20)
right.Position = UDim2.new(0.5, 10, 0.14, 0)
right.BackgroundTransparency = 0.2
right.BackgroundColor3 = Color3.fromRGB(15,15,20)

local function makeButton(parent, txt, ypos)
    local b = Instance.new("TextButton", parent)
    b.Size = UDim2.new(0.9,0,0,48)
    b.Position = UDim2.new(0.05, 0, ypos, 0)
    b.Text = txt
    b.Font = Enum.Font.GothamBold
    b.TextScaled = true
    b.BackgroundColor3 = Color3.fromRGB(25,25,35)
    b.TextColor3 = Color3.fromRGB(240,240,240)
    return b
end

local btn_viewRole = makeButton(right, "Ver mi rol (F2)", 0)
local btn_teleport = makeButton(right, "Teleport - TP_1 (F3)", 0.14)
local btn_speedUp = makeButton(right, "Aumentar Velocidad (F4)", 0.28)
local btn_toggleLight = makeButton(right, "Linterna ON/OFF (F5)", 0.42)
local btn_spectator = makeButton(right, "Activar Espectador", 0.56)
local btn_camToggle = makeButton(right, "Cambiar Cámara", 0.7)
local btn_roundInfo = makeButton(right, "Mostrar tiempo de ronda", 0.84)
local btn_ambient = makeButton(right, "Ambient ON/OFF", 0.98)

-- small HUD (top-left)
local hud = Instance.new("Frame", screenGui)
hud.Size = UDim2.new(0,240,0,90)
hud.Position = UDim2.new(0.01,0,0.01,0)
hud.BackgroundTransparency = 0.25
hud.BorderSizePixel = 0
local hudRole = Instance.new("TextLabel", hud)
hudRole.Size = UDim2.new(1, -10, 0.45, -10)
hudRole.Position = UDim2.new(0,5,0,5)
hudRole.BackgroundTransparency = 1
hudRole.TextScaled = true
hudRole.Font = Enum.Font.GothamBold
hudRole.Text = "Rol: ?"

local hudNight = Instance.new("TextLabel", hud)
hudNight.Size = UDim2.new(1, -10, 0.45, -10)
hudNight.Position = UDim2.new(0,5,0,45)
hudNight.BackgroundTransparency = 1
hudNight.TextScaled = true
hudNight.Font = Enum.Font.Gotham
hudNight.Text = "Noche: 0"

-- Toggle button (bottom-left)
local toggleBtn = Instance.new("TextButton", screenGui)
toggleBtn.Size = UDim2.new(0,64,0,64)
toggleBtn.Position = UDim2.new(0.02,0,0.9,-80)
toggleBtn.Text = "ON"
toggleBtn.Font = Enum.Font.GothamBold
toggleBtn.TextScaled = true
toggleBtn.BackgroundColor3 = Color3.fromRGB(18,140,255)
local menuVisible = true
toggleBtn.MouseButton1Click:Connect(function()
    menuVisible = not menuVisible
    main.Visible = menuVisible
    toggleBtn.Text = menuVisible and "ON" or "OFF"
    toggleBtn.BackgroundColor3 = menuVisible and Color3.fromRGB(18,140,255) or Color3.fromRGB(90,90,90)
    if RE_ToggleMenu then RE_ToggleMenu:FireServer(menuVisible) end
end)

-- FUNCTIONALITY

-- 1) Show role (request server)
local function showRole()
    RE_RequestRole:FireServer()
end
RE_RequestRole.OnClientEvent:Connect(function(role)
    hudRole.Text = "Rol: "..tostring(role)
    -- pop-up temporary
    local popup = Instance.new("TextLabel", screenGui)
    popup.Size = UDim2.new(0,300,0,80)
    popup.Position = UDim2.new(0.5,-150,0.75, -40)
    popup.BackgroundTransparency = 0.2
    popup.Font = Enum.Font.GothamBold
    popup.TextScaled = true
    popup.Text = "TU ROL: "..tostring(role)
    if role == "Murderer" then popup.TextColor3 = Color3.fromRGB(255,80,80) end
    if role == "Sheriff" then popup.TextColor3 = Color3.fromRGB(120,200,255) end
    if role == "Innocent" then popup.TextColor3 = Color3.fromRGB(200,200,200) end
    delay(3, function() popup:Destroy() end)
end)

btn_viewRole.MouseButton1Click:Connect(showRole)

-- 2) Teleport (request server)
local function teleportTo(name)
    RE_RequestTeleport:FireServer(name)
end
btn_teleport.MouseButton1Click:Connect(function()
    teleportTo("TP_1")
end)

-- 3) Change speed
local currentSpeed = 16
local function changeSpeed(delta)
    currentSpeed = math.clamp(currentSpeed + delta, 8, 40)
    RE_ChangeSpeed:FireServer(currentSpeed)
end
btn_speedUp.MouseButton1Click:Connect(function() changeSpeed(4) end)

-- 4) Flashlight toggle (local visual + server validation)
local flashlightOn = false
local flashPart -- optional visible part to simulate
local function toggleFlashlight()
    flashlightOn = not flashlightOn
    -- simple local visual: create a point light attached to character head
    local char = player.Character
    if char and char:FindFirstChild("Head") then
        if flashlightOn then
            local p = Instance.new("PointLight", char.Head)
            p.Name = "VixFlash"
            p.Range = 18
            p.Brightness = 3
            p.Shadows = true
            flashPart = p
        else
            local existing = char.Head:FindFirstChild("VixFlash")
            if existing then existing:Destroy() end
        end
    end
    RE_ValidateLight:FireServer(flashlightOn)
end
RE_ValidateLight.OnClientEvent:Connect(function(state)
    -- server ack (no-op for now)
end)
btn_toggleLight.MouseButton1Click:Connect(toggleFlashlight)

-- 5) Spectator mode (simple: make character transparent + free camera)
local specActive = false
local previousCameraType
local previousCameraSubject
local function toggleSpectator()
    specActive = not specActive
    RE_ToggleSpecMode:FireServer(specActive)
    local cam = workspace.CurrentCamera
    if specActive then
        -- detach camera
        previousCameraType = cam.CameraType
        previousCameraSubject = cam.CameraSubject
        cam.CameraType = Enum.CameraType.Scriptable
        -- simple free cam: look at player but don't move character (for demo)
        cam.CFrame = cam.CFrame * CFrame.new(0,5,10)
        local char = player.Character
        if char then
            for _,part in pairs(char:GetDescendants()) do
                if part:IsA("BasePart") then
                    part.Transparency = 0.5
                end
            end
        end
    else
        -- restore
        cam.CameraType = Enum.CameraType.Custom
        if player.Character and player.Character:FindFirstChild("Humanoid") then
            cam.CameraSubject = player.Character:FindFirstChild("Humanoid")
        end
        local char = player.Character
        if char then
            for _,part in pairs(char:GetDescendants()) do
                if part:IsA("BasePart") then
                    part.Transparency = 0
                end
            end
        end
    end
end
btn_spectator.MouseButton1Click:Connect(toggleSpectator)

-- 6) Camera change (first/third)
local camMode = "Default"
local function changeCamera()
    local cam = workspace.CurrentCamera
    if camMode == "Default" then
        cam.CameraType = Enum.CameraType.Custom
        camMode = "Third"
        RE_CameraChange:FireServer(camMode)
    else
        cam.CameraType = Enum.CameraType.Custom
        camMode = "Default"
        RE_CameraChange:FireServer(camMode)
    end
end
btn_camToggle.MouseButton1Click:Connect(changeCamera)

-- 7) Round info
local function requestRound()
    RE_RequestRoundInfo:FireServer()
end
RE_RequestRoundInfo.OnClientEvent:Connect(function(info)
    if info and type(info) == "table" then
        hudNight.Text = "Noche: "..tostring(info.night or 0)
        -- show timer popup
        local tpop = Instance.new("TextLabel", screenGui)
        tpop.Size = UDim2.new(0,280,0,60)
        tpop.Position = UDim2.new(0.5,-140,0.02, 0)
        tpop.BackgroundTransparency = 0.2
        tpop.Font = Enum.Font.GothamBold
        tpop.TextScaled = true
        tpop.Text = "Noche "..tostring(info.night or 0).." — Tiempo: "..tostring(info.time or 0)
        delay(4, function() if tpop and tpop.Parent then tpop:Destroy() end end)
    end
end)
btn_roundInfo.MouseButton1Click:Connect(requestRound)

-- 8) Inventory (request from server)
local function requestInventory()
    RE_RequestInventory:FireServer()
end
RE_RequestInventory.OnClientEvent:Connect(function(list)
    if type(list) == "table" then
        if #list == 0 then
            invList.Text = "Inventario vacío"
        else
            invList.Text = table.concat(list, "\n")
        end
    end
end)
-- map a small UI button to request inventory
-- create small Inventory button
local invBtn = Instance.new("TextButton", left)
invBtn.Size = UDim2.new(0.45, -10, 0, 40)
invBtn.Position = UDim2.new(0.05, 0, 1, -50)
invBtn.Text = "Inventario"
invBtn.Font = Enum.Font.GothamBold
invBtn.MouseButton1Click:Connect(requestInventory)

-- 9) Ambient sound toggle
local ambientOn = true
local ambientSound
local function toggleAmbient(state)
    ambientOn = not ambientOn
    if ambientOn then
        if not ambientSound then
            ambientSound = Instance.new("Sound", workspace)
            ambientSound.Name = "VixAmbient"
            ambientSound.Looped = true
            ambientSound.Volume = 0.5
            -- no external file: use a default sound id if you have one; left blank for user to replace
            -- ambientSound.SoundId = "rbxassetid://<TU_SONIDO>"
            ambientSound:Play()
        else
            ambientSound:Play()
        end
    else
        if ambientSound then ambientSound:Stop() end
    end
    -- notify server for record
    RE_ToggleAmbient:FireServer(ambientOn)
end
btn_ambient.MouseButton1Click:Connect(toggleAmbient)
RE_ToggleAmbient.OnClientEvent:Connect(function(state)
    -- server broadcast (if needed)
    ambientOn = state
    if not state and ambientSound then ambientSound:Stop() end
end)

-- 10) Close menu handled by toggle button above

-- HOTKEYS (F1-F5)
local KEY_MAP = {
    ToggleMenu = Enum.KeyCode.F1, -- toggle menu
    ShowRole = Enum.KeyCode.F2,
    QuickTP = Enum.KeyCode.F3,
    SpeedUp = Enum.KeyCode.F4,
    ToggleLight = Enum.KeyCode.F5,
}
UserInputService.InputBegan:Connect(function(input, gameProcessed)
    if gameProcessed then return end
    if input.KeyCode == KEY_MAP.ToggleMenu then
        menuVisible = not menuVisible
        main.Visible = menuVisible
    elseif input.KeyCode == KEY_MAP.ShowRole then
        showRole()
    elseif input.KeyCode == KEY_MAP.QuickTP then
        teleportTo("TP_1")
    elseif input.KeyCode == KEY_MAP.SpeedUp then
        changeSpeed(4)
    elseif input.KeyCode == KEY_MAP.ToggleLight then
        toggleFlashlight()
    end
end)

-- Initialize: request round info
delay(1, function() RE_RequestRoundInfo:FireServer() end)

]]

-- Create the server child script
local existingServer = script.Parent:FindFirstChild("Vixfer_ServerHandler")
if not existingServer then
    local s = Instance.new("Script")
    s.Name = "Vixfer_ServerHandler"
    s.Source = serverSource
    s.Parent = script.Parent
    print("Vixfer: ServerHandler created (Vixfer_ServerHandler).")
else
    print("Vixfer: ServerHandler already exists.")
end

-- Create local client script in StarterPlayerScripts
local starterScripts = StarterPlayer:FindFirstChild("StarterPlayerScripts")
if not starterScripts then
    starterScripts = Instance.new("Folder", StarterPlayer)
    starterScripts.Name = "StarterPlayerScripts"
end

local existingClient = starterScripts:FindFirstChild("Vixfer_ClientUI")
if not existingClient then
    local ls = Instance.new("LocalScript")
    ls.Name = "Vixfer_ClientUI"
    ls.Source = clientSource
    ls.Parent = starterScripts
    print("Vixfer: Client UI created (StarterPlayerScripts.Vixfer_ClientUI).")
else
    print("Vixfer: Client UI already exists.")
end

print("Vixfer_Menu_System: Setup complete. Rejoin or respawn a player to load the client UI if not present.")
