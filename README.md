-- 🌀 Noclip Pro de pruebas (solo en Studio)
-- Muestra pantalla "Vixfer presenta" + carga + botón activable
-- Hecho por ChatGPT 💚

local player = game.Players.LocalPlayer
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")

-- 🖤 Pantalla de presentación
local intro = Instance.new("Frame")
intro.Size = UDim2.new(1, 0, 1, 0)
intro.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
intro.ZIndex = 10
intro.Parent = script.Parent

local label = Instance.new("TextLabel", intro)
label.Size = UDim2.new(1, 0, 1, 0)
label.BackgroundTransparency = 1
label.Text = "✨ Vixfer presenta ✨\n\nNoclip :) Welcome"
label.TextColor3 = Color3.fromRGB(255, 255, 255)
label.Font = Enum.Font.GothamBold
label.TextScaled = true
label.ZIndex = 11

wait(3) -- Duración de pantalla de presentación
intro:Destroy()

-- 🧱 Crear botón automáticamente
local button = Instance.new("TextButton")
button.Parent = script.Parent
button.Text = "🚀 Activar Noclip"
button.BackgroundColor3 = Color3.fromRGB(45, 45, 55)
button.TextColor3 = Color3.fromRGB(255, 255, 255)
button.Font = Enum.Font.GothamBold
button.TextSize = 18
button.Size = UDim2.new(0, 160, 0, 50)
button.Position = UDim2.new(0.05, 0, 0.1, 0)
button.BackgroundTransparency = 0.1
button.BorderSizePixel = 0
button.Active = true
button.AutoButtonColor = true
button.ZIndex = 5

-- 🎨 Bordes y estilo
local corner = Instance.new("UICorner", button)
corner.CornerRadius = UDim.new(0, 20)
local stroke = Instance.new("UIStroke", button)
stroke.Thickness = 2
stroke.Color = Color3.fromRGB(100, 255, 150)

-- 🖐 Hacer el botón movible
local dragging = false
local mousePos, framePos
local function update(input)
	local delta = input.Position - mousePos
	button.Position = UDim2.new(framePos.X.Scale, framePos.X.Offset + delta.X, framePos.Y.Scale, framePos.Y.Offset + delta.Y)
end

button.InputBegan:Connect(function(input)
	if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
		dragging = true
		mousePos = input.Position
		framePos = button.Position
		input.Changed:Connect(function()
			if input.UserInputState == Enum.UserInputState.End then
				dragging = false
			end
		end)
	end
end)

UserInputService.InputChanged:Connect(function(input)
	if dragging and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
		update(input)
	end
end)

-- ⚙️ Control del Noclip
local noclipEnabled = false

local function setNoclip(state)
	local character = player.Character
	if not character then return end
	for _, part in pairs(character:GetDescendants()) do
		if part:IsA("BasePart") then
			part.CanCollide = not state
		end
	end
end

-- 💫 Pantalla de carga animada
local function createLoadingScreen()
	local screen = Instance.new("Frame")
	screen.Size = UDim2.new(1, 0, 1, 0)
	screen.BackgroundColor3 = Color3.fromRGB(10, 10, 10)
	screen.BackgroundTransparency = 0
	screen.ZIndex = 10
	screen.Parent = script.Parent

	local text = Instance.new("TextLabel", screen)
	text.Size = UDim2.new(1, 0, 0, 100)
	text.Position = UDim2.new(0, 0, 0.4, 0)
	text.Text = "Cargando..."
	text.TextColor3 = Color3.fromRGB(255, 255, 255)
	text.BackgroundTransparency = 1
	text.Font = Enum.Font.GothamBold
	text.TextScaled = true

	local barFrame = Instance.new("Frame", screen)
	barFrame.Size = UDim2.new(0.6, 0, 0.05, 0)
	barFrame.Position = UDim2.new(0.2, 0, 0.55, 0)
	barFrame.BackgroundColor3 = Color3.fromRGB(50, 50, 50)
	local barCorner = Instance.new("UICorner", barFrame)
	barCorner.CornerRadius = UDim.new(0, 10)

	local fill = Instance.new("Frame", barFrame)
	fill.BackgroundColor3 = Color3.fromRGB(0, 255, 120)
	fill.Size = UDim2.new(0, 0, 1, 0)
	local fillCorner = Instance.new("UICorner", fill)
	fillCorner.CornerRadius = UDim.new(0, 10)
	fill.ZIndex = 12

	-- Animación de carga
	for i = 0, 1, 0.02 do
		fill.Size = UDim2.new(i, 0, 1, 0)
		wait(0.03)
	end

	screen:Destroy()
end

-- 🚀 Click del botón para activar/desactivar Noclip
button.MouseButton1Click:Connect(function()
	if not noclipEnabled then
		button.Text = "🔄 Activando..."
		createLoadingScreen()
		noclipEnabled = true
		setNoclip(true)
		button.Text = "✅ Noclip Activado"
		button.BackgroundColor3 = Color3.fromRGB(60, 200, 100)
	else
		noclipEnabled = false
		setNoclip(false)
		button.Text = "🚀 Activar Noclip"
		button.BackgroundColor3 = Color3.fromRGB(45, 45, 55)
	end
end)

-- 🧲 Mantener Noclip activo
RunService.Stepped:Connect(function()
	if noclipEnabled and player.Character then
		for _, part in pairs(player.Character:GetDescendants()) do
			if part:IsA("BasePart") then
				part.CanCollide = false
			end
		end
	end
end)

