--[[
    Vixfer Fly/Noclip/Teleport Menu
    Autor: Vixfer
    Descripción: Menú UI para activar/desactivar modo volar, noclip y teletransporte a jugadores.
    El menú se puede mover con mouse/touch/teclado.
--]]

local Players = game:GetService("Players")
local LocalPlayer = Players.LocalPlayer
local UserInputService = game:GetService("UserInputService")
local TweenService = game:GetService("TweenService")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local RunService = game:GetService("RunService")

-- RemoteEvents
local flyEvent = ReplicatedStorage:FindFirstChild("VixferFlyEvent")
if not flyEvent then
    flyEvent = Instance.new("RemoteEvent")
    flyEvent.Name = "VixferFlyEvent"
    flyEvent.Parent = ReplicatedStorage
end

local noclipEvent = ReplicatedStorage:FindFirstChild("VixferNoclipEvent")
if not noclipEvent then
    noclipEvent = Instance.new("RemoteEvent")
    noclipEvent.Name = "VixferNoclipEvent"
    noclipEvent.Parent = ReplicatedStorage
end

local teleportEvent = ReplicatedStorage:FindFirstChild("VixferTeleportEvent")
if not teleportEvent then
    teleportEvent = Instance.new("RemoteEvent")
    teleportEvent.Name = "VixferTeleportEvent"
    teleportEvent.Parent = ReplicatedStorage
end

-- Crear el menú UI
local screenGui = Instance.new("ScreenGui")
screenGui.Name = "VixferFlyMenu"
screenGui.ResetOnSpawn = false

local frame = Instance.new("Frame")
frame.Size = UDim2.new(0, 270, 0, 260)
frame.Position = UDim2.new(0.5, -135, 0.1, 0)
frame.BackgroundColor3 = Color3.fromRGB(35, 35, 50)
frame.BorderSizePixel = 0
frame.AnchorPoint = Vector2.new(0.5, 0)
frame.Active = true
frame.Draggable = true -- Para teclado/mouse

local uiCorner = Instance.new("UICorner")
uiCorner.CornerRadius = UDim.new(0, 16)
uiCorner.Parent = frame

local title = Instance.new("TextLabel")
title.Text = "Vixfer"
title.Font = Enum.Font.GothamBold
title.TextSize = 24
title.TextColor3 = Color3.fromRGB(255, 255, 255)
title.BackgroundTransparency = 1
title.Size = UDim2.new(1, 0, 0, 36)
title.Parent = frame

-- Botón Volar
local flyButton = Instance.new("TextButton")
flyButton.Text = "Activar Volar"
flyButton.Font = Enum.Font.Gotham
flyButton.TextSize = 20
flyButton.TextColor3 = Color3.fromRGB(255,255,255)
flyButton.BackgroundColor3 = Color3.fromRGB(60, 120, 220)
flyButton.Size = UDim2.new(0.8, 0, 0, 36)
flyButton.Position = UDim2.new(0.1, 0, 0.18, 0)
flyButton.Parent = frame

local uiCornerBtn = Instance.new("UICorner")
uiCornerBtn.CornerRadius = UDim.new(0, 12)
uiCornerBtn.Parent = flyButton

-- Botón Noclip
local noclipButton = Instance.new("TextButton")
noclipButton.Text = "Activar Noclip"
noclipButton.Font = Enum.Font.Gotham
noclipButton.TextSize = 20
noclipButton.TextColor3 = Color3.fromRGB(255,255,255)
noclipButton.BackgroundColor3 = Color3.fromRGB(80, 180, 80)
noclipButton.Size = UDim2.new(0.8, 0, 0, 36)
noclipButton.Position = UDim2.new(0.1, 0, 0.36, 0)
noclipButton.Parent = frame

local uiCornerBtn2 = Instance.new("UICorner")
uiCornerBtn2.CornerRadius = UDim.new(0, 12)
uiCornerBtn2.Parent = noclipButton

-- Botón Teleport
local teleportButton = Instance.new("TextButton")
teleportButton.Text = "Teletransportarse"
teleportButton.Font = Enum.Font.Gotham
teleportButton.TextSize = 20
teleportButton.TextColor3 = Color3.fromRGB(255,255,255)
teleportButton.BackgroundColor3 = Color3.fromRGB(120, 80, 180)
teleportButton.Size = UDim2.new(0.8, 0, 0, 36)
teleportButton.Position = UDim2.new(0.1, 0, 0.54, 0)
teleportButton.Parent = frame

local uiCornerBtn3 = Instance.new("UICorner")
uiCornerBtn3.CornerRadius = UDim.new(0, 12)
uiCornerBtn3.Parent = teleportButton

-- Info
local info = Instance.new("TextLabel")
info.Text = "WASD: Moverse | Q/E: Subir/Bajar\nNoclip atraviesa paredes\nElige jugador para teleport"
info.Font = Enum.Font.Gotham
info.TextSize = 15
info.TextColor3 = Color3.fromRGB(200, 200, 220)
info.BackgroundTransparency = 1
info.Size = UDim2.new(1, -20, 0, 44)
info.Position = UDim2.new(0, 10, 0.72, 0)
info.TextWrapped = true
info.Parent = frame

-- Dropdown de jugadores para teleport
local playerDropdown = Instance.new("Frame")
playerDropdown.Size = UDim2.new(0.8, 0, 0, 36)
playerDropdown.Position = UDim2.new(0.1, 0, 0.62, 0)
playerDropdown.BackgroundColor3 = Color3.fromRGB(50, 50, 70)
playerDropdown.Visible = false
playerDropdown.Parent = frame
playerDropdown.ClipsDescendants = true

local uiCornerDrop = Instance.new("UICorner")
uiCornerDrop.CornerRadius = UDim.new(0, 10)
uiCornerDrop.Parent = playerDropdown

local dropdownLabel = Instance.new("TextLabel")
dropdownLabel.Text = "Selecciona jugador..."
dropdownLabel.Font = Enum.Font.Gotham
dropdownLabel.TextSize = 18
dropdownLabel.TextColor3 = Color3.fromRGB(255,255,255)
dropdownLabel.BackgroundTransparency = 1
dropdownLabel.Size = UDim2.new(1, -10, 1, 0)
dropdownLabel.Position = UDim2.new(0, 5, 0, 0)
dropdownLabel.TextXAlignment = Enum.TextXAlignment.Left
dropdownLabel.Parent = playerDropdown

local dropdownList = Instance.new("Frame")
dropdownList.Size = UDim2.new(1, 0, 0, 0)
dropdownList.Position = UDim2.new(0, 0, 1, 0)
dropdownList.BackgroundColor3 = Color3.fromRGB(40, 40, 60)
dropdownList.Visible = false
dropdownList.Parent = playerDropdown
dropdownList.ClipsDescendants = true

local uiCornerList = Instance.new("UICorner")
uiCornerList.CornerRadius = UDim.new(0, 8)
uiCornerList.Parent = dropdownList

local function updateDropdown()
    for _, child in dropdownList:GetChildren() do
        if child:IsA("TextButton") then
            child:Destroy()
        end
    end
    local y = 0
    for _, player in Players:GetPlayers() do
        if player ~= LocalPlayer then
            local btn = Instance.new("TextButton")
            btn.Text = player.DisplayName .. " (@" .. player.Name .. ")"
            btn.Font = Enum.Font.Gotham
            btn.TextSize = 16
            btn.TextColor3 = Color3.fromRGB(255,255,255)
            btn.BackgroundColor3 = Color3.fromRGB(60, 60, 90)
            btn.Size = UDim2.new(1, 0, 0, 28)
            btn.Position = UDim2.new(0, 0, 0, y)
            btn.Parent = dropdownList
            btn.AutoButtonColor = true
            btn.MouseButton1Click:Connect(function()
                dropdownLabel.Text = btn.Text
                playerDropdown.Visible = false
                dropdownList.Visible = false
                teleportEvent:FireServer(player.Name)
            end)
            y = y + 28
        end
    end
    dropdownList.Size = UDim2.new(1, 0, 0, y)
end

dropdownLabel.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
        updateDropdown()
        dropdownList.Visible = not dropdownList.Visible
    end
end)

Players.PlayerAdded:Connect(updateDropdown)
Players.PlayerRemoving:Connect(updateDropdown)

-- Mostrar/ocultar dropdown con botón teleport
teleportButton.MouseButton1Click:Connect(function()
    playerDropdown.Visible = not playerDropdown.Visible
    dropdownList.Visible = false
    dropdownLabel.Text = "Selecciona jugador..."
    updateDropdown()
end)

-- Drag para touch (móvil)
local dragging = false
local dragInput, mousePos, framePos

frame.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
        dragging = true
        mousePos = input.Position
        framePos = frame.Position
        input.Changed:Connect(function()
            if input.UserInputState == Enum.UserInputState.End then
                dragging = false
            end
        end)
    end
end)

frame.InputChanged:Connect(function(input)
    if dragging and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
        local delta = input.Position - mousePos
        frame.Position = UDim2.new(framePos.X.Scale, framePos.X.Offset + delta.X, framePos.Y.Scale, framePos.Y.Offset + delta.Y)
    end
end)

-- Script de volar
local flying = false
local flySpeed = 60

local function setFly(state)
    flying = state
    if flying then
        flyButton.Text = "Desactivar Volar"
        flyButton.BackgroundColor3 = Color3.fromRGB(220, 60, 60)
    else
        flyButton.Text = "Activar Volar"
        flyButton.BackgroundColor3 = Color3.fromRGB(60, 120, 220)
    end
    flyEvent:FireServer(flying)
end

flyButton.MouseButton1Click:Connect(function()
    setFly(not flying)
end)

-- Control de vuelo
local moveDirection = Vector3.new()
local upDown = 0

UserInputService.InputBegan:Connect(function(input, processed)
    if processed then return end
    if flying then
        if input.KeyCode == Enum.KeyCode.W then moveDirection = Vector3.new(0,0,-1)
        elseif input.KeyCode == Enum.KeyCode.S then moveDirection = Vector3.new(0,0,1)
        elseif input.KeyCode == Enum.KeyCode.A then moveDirection = Vector3.new(-1,0,0)
        elseif input.KeyCode == Enum.KeyCode.D then moveDirection = Vector3.new(1,0,0)
        elseif input.KeyCode == Enum.KeyCode.Q then upDown = -1
        elseif input.KeyCode == Enum.KeyCode.E then upDown = 1
        end
    end
end)

UserInputService.InputEnded:Connect(function(input)
    if flying then
        if input.KeyCode == Enum.KeyCode.W or input.KeyCode == Enum.KeyCode.S then moveDirection = Vector3.new(0,0,0)
        elseif input.KeyCode == Enum.KeyCode.A or input.KeyCode == Enum.KeyCode.D then moveDirection = Vector3.new(0,0,0)
        elseif input.KeyCode == Enum.KeyCode.Q or input.KeyCode == Enum.KeyCode.E then upDown = 0
        end
    end
end)

RunService.RenderStepped:Connect(function(dt)
    if flying and LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart") then
        local hrp = LocalPlayer.Character.HumanoidRootPart
        local cam = workspace.CurrentCamera
        local direction = (cam.CFrame:VectorToWorldSpace(moveDirection) + Vector3.new(0, upDown, 0)).Unit
        if direction.Magnitude > 0 then
            hrp.Velocity = direction * flySpeed
        else
            hrp.Velocity = Vector3.new(0,0,0)
        end
        LocalPlayer.Character.Humanoid.PlatformStand = true
    elseif LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("Humanoid") then
        LocalPlayer.Character.Humanoid.PlatformStand = false
    end
end)

-- Desactivar volar al morir
LocalPlayer.CharacterAdded:Connect(function(char)
    setFly(false)
end)

flyEvent.OnClientEvent:Connect(function(state)
    setFly(state)
end)

-- Noclip
local noclip = false
local function setNoclip(state)
    noclip = state
    if noclip then
        noclipButton.Text = "Desactivar Noclip"
        noclipButton.BackgroundColor3 = Color3.fromRGB(220, 120, 60)
    else
        noclipButton.Text = "Activar Noclip"
        noclipButton.BackgroundColor3 = Color3.fromRGB(80, 180, 80)
    end
    noclipEvent:FireServer(noclip)
end

noclipButton.MouseButton1Click:Connect(function()
    setNoclip(not noclip)
end)

-- Loop de noclip (solo cliente visual, server lo valida)
RunService.Stepped:Connect(function()
    if noclip and LocalPlayer.Character then
        for _, v in LocalPlayer.Character:GetChildren() do
            if v:IsA("BasePart") then
                v.CanCollide = false
            end
        end
    end
end)

noclipEvent.OnClientEvent:Connect(function(state)
    setNoclip(state)
end)

LocalPlayer.CharacterAdded:Connect(function(char)
    setNoclip(false)
end)

-- UI Parenting
frame.Parent = screenGui
screenGui.Parent = LocalPlayer:WaitForChild("PlayerGui")
