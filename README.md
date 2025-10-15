--[[
    Vixfer Fly Menu
    Autor: Vixfer
    Descripción: Menú UI para activar/desactivar modo volar. El menú se puede mover con mouse/touch.
    El modo volar permite moverse con WASD y subir/bajar con Q/E.
--]]

local Players = game:GetService("Players")
local LocalPlayer = Players.LocalPlayer
local UserInputService = game:GetService("UserInputService")
local TweenService = game:GetService("TweenService")
local ReplicatedStorage = game:GetService("ReplicatedStorage")

-- Crear RemoteEvent para comunicar el estado de vuelo
local flyEvent = ReplicatedStorage:FindFirstChild("VixferFlyEvent")
if not flyEvent then
    flyEvent = Instance.new("RemoteEvent")
    flyEvent.Name = "VixferFlyEvent"
    flyEvent.Parent = ReplicatedStorage
end

-- Crear el menú UI
local screenGui = Instance.new("ScreenGui")
screenGui.Name = "VixferFlyMenu"
screenGui.ResetOnSpawn = false

local frame = Instance.new("Frame")
frame.Size = UDim2.new(0, 250, 0, 140)
frame.Position = UDim2.new(0.5, -125, 0.1, 0)
frame.BackgroundColor3 = Color3.fromRGB(35, 35, 50)
frame.BorderSizePixel = 0
frame.AnchorPoint = Vector2.new(0.5, 0)
frame.Active = true
frame.Draggable = true -- Para teclado/mouse

local uiCorner = Instance.new("UICorner")
uiCorner.CornerRadius = UDim.new(0, 16)
uiCorner.Parent = frame

local title = Instance.new("TextLabel")
title.Text = "Vixfer Fly Menu"
title.Font = Enum.Font.GothamBold
title.TextSize = 22
title.TextColor3 = Color3.fromRGB(255, 255, 255)
title.BackgroundTransparency = 1
title.Size = UDim2.new(1, 0, 0, 36)
title.Parent = frame

local flyButton = Instance.new("TextButton")
flyButton.Text = "Activar Volar"
flyButton.Font = Enum.Font.Gotham
flyButton.TextSize = 20
flyButton.TextColor3 = Color3.fromRGB(255,255,255)
flyButton.BackgroundColor3 = Color3.fromRGB(60, 120, 220)
flyButton.Size = UDim2.new(0.8, 0, 0, 40)
flyButton.Position = UDim2.new(0.1, 0, 0.45, 0)
flyButton.Parent = frame

local uiCornerBtn = Instance.new("UICorner")
uiCornerBtn.CornerRadius = UDim.new(0, 12)
uiCornerBtn.Parent = flyButton

local info = Instance.new("TextLabel")
info.Text = "Usa WASD para moverte\nQ/E para subir/bajar"
info.Font = Enum.Font.Gotham
info.TextSize = 16
info.TextColor3 = Color3.fromRGB(200, 200, 220)
info.BackgroundTransparency = 1
info.Size = UDim2.new(1, -20, 0, 40)
info.Position = UDim2.new(0, 10, 0.75, 0)
info.Parent = frame

frame.Parent = screenGui
screenGui.Parent = game:GetService("Players").LocalPlayer:WaitForChild("PlayerGui")

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

-- Volar loop
local RunService = game:GetService("RunService")
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

-- Seguridad: Si el server desactiva el vuelo, lo forzamos
flyEvent.OnClientEvent:Connect(function(state)
    setFly(state)
end)

--[[
    Vixfer Fly Menu
    Autor: Vixfer
    Descripción: Menú UI para activar/desactivar modo volar. El menú se puede mover con mouse/touch.
    El modo volar permite moverse con WASD y subir/bajar con Q/E.
--]]

local Players = game:GetService("Players")
local LocalPlayer = Players.LocalPlayer
local UserInputService = game:GetService("UserInputService")
local TweenService = game:GetService("TweenService")
local ReplicatedStorage = game:GetService("ReplicatedStorage")

-- Crear RemoteEvent para comunicar el estado de vuelo
local flyEvent = ReplicatedStorage:FindFirstChild("VixferFlyEvent")
if not flyEvent then
    flyEvent = Instance.new("RemoteEvent")
    flyEvent.Name = "VixferFlyEvent"
    flyEvent.Parent = ReplicatedStorage
end

-- Crear el menú UI
local screenGui = Instance.new("ScreenGui")
screenGui.Name = "VixferFlyMenu"
screenGui.ResetOnSpawn = false

local frame = Instance.new("Frame")
frame.Size = UDim2.new(0, 250, 0, 140)
frame.Position = UDim2.new(0.5, -125, 0.1, 0)
frame.BackgroundColor3 = Color3.fromRGB(35, 35, 50)
frame.BorderSizePixel = 0
frame.AnchorPoint = Vector2.new(0.5, 0)
frame.Active = true
frame.Draggable = true -- Para teclado/mouse

local uiCorner = Instance.new("UICorner")
uiCorner.CornerRadius = UDim.new(0, 16)
uiCorner.Parent = frame

local title = Instance.new("TextLabel")
title.Text = "Vixfer Fly Menu"
title.Font = Enum.Font.GothamBold
title.TextSize = 22
title.TextColor3 = Color3.fromRGB(255, 255, 255)
title.BackgroundTransparency = 1
title.Size = UDim2.new(1, 0, 0, 36)
title.Parent = frame

local flyButton = Instance.new("TextButton")
flyButton.Text = "Activar Volar"
flyButton.Font = Enum.Font.Gotham
flyButton.TextSize = 20
flyButton.TextColor3 = Color3.fromRGB(255,255,255)
flyButton.BackgroundColor3 = Color3.fromRGB(60, 120, 220)
flyButton.Size = UDim2.new(0.8, 0, 0, 40)
flyButton.Position = UDim2.new(0.1, 0, 0.45, 0)
flyButton.Parent = frame

local uiCornerBtn = Instance.new("UICorner")
uiCornerBtn.CornerRadius = UDim.new(0, 12)
uiCornerBtn.Parent = flyButton

local info = Instance.new("TextLabel")
info.Text = "Usa WASD para moverte\nQ/E para subir/bajar"
info.Font = Enum.Font.Gotham
info.TextSize = 16
info.TextColor3 = Color3.fromRGB(200, 200, 220)
info.BackgroundTransparency = 1
info.Size = UDim2.new(1, -20, 0, 40)
info.Position = UDim2.new(0, 10, 0.75, 0)
info.Parent = frame

frame.Parent = screenGui
screenGui.Parent = game:GetService("Players").LocalPlayer:WaitForChild("PlayerGui")

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

-- Volar loop
local RunService = game:GetService("RunService")
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

-- Seguridad: Si el server desactiva el vuelo, lo forzamos
flyEvent.OnClientEvent:Connect(function(state)
    setFly(state)
end)
