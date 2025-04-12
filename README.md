-- LocalScript in StarterPlayerScripts

local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local RunService = game:GetService("RunService")
local LocalPlayer = Players.LocalPlayer
local Camera = workspace.CurrentCamera
local REFRESH_INTERVAL = 5
local HEAD_NAME = "Head"
local AIM_SMOOTHNESS = 0.2
local AIM_RADIUS = 150
local TAG_NAME = "PlayerNameTag"
local SHOW_LOCAL_PLAYER = true
local airJumpEnabled = false
local aiming = false

-- GUI Setup for Air Jump Toggle
local screenGui = Instance.new("ScreenGui")
screenGui.Name = "AirJumpToggleGui"
screenGui.ResetOnSpawn = false
screenGui.Parent = LocalPlayer:WaitForChild("PlayerGui")

local jumpButton = Instance.new("TextButton")
jumpButton.Size = UDim2.new(0, 200, 0, 50)
jumpButton.Position = UDim2.new(0, 20, 0, 20)
jumpButton.BackgroundColor3 = Color3.fromRGB(50, 50, 50)
jumpButton.TextColor3 = Color3.fromRGB(255, 255, 255)
jumpButton.TextScaled = true
jumpButton.Text = "Air Jump: OFF"
jumpButton.Font = Enum.Font.SourceSansBold
jumpButton.Parent = screenGui

local aimButton = Instance.new("TextButton")
aimButton.Size = UDim2.new(0, 200, 0, 50)
aimButton.Position = UDim2.new(0, 20, 0, 80)
aimButton.BackgroundColor3 = Color3.fromRGB(50, 50, 50)
aimButton.TextColor3 = Color3.fromRGB(255, 255, 255)
aimButton.TextScaled = true
aimButton.Text = "Aim Assist: OFF"
aimButton.Font = Enum.Font.SourceSansBold
aimButton.Parent = screenGui

-- Toggle Air Jump
jumpButton.MouseButton1Click:Connect(function()
	airJumpEnabled = not airJumpEnabled
	jumpButton.Text = airJumpEnabled and "Air Jump: ON" or "Air Jump: OFF"
	jumpButton.BackgroundColor3 = airJumpEnabled and Color3.fromRGB(0, 170, 0) or Color3.fromRGB(50, 50, 50)
end)

-- Toggle Aim Assist
aimButton.MouseButton1Click:Connect(function()
	aiming = not aiming
	aimButton.Text = aiming and "Aim Assist: ON" or "Aim Assist: OFF"
	aimButton.BackgroundColor3 = aiming and Color3.fromRGB(0, 170, 0) or Color3.fromRGB(50, 50, 50)
end)

-- Air Jump Logic
UserInputService.JumpRequest:Connect(function()
	if not airJumpEnabled then return end
	local character = LocalPlayer.Character
	if character and character:FindFirstChild("Humanoid") and character:FindFirstChild("HumanoidRootPart") then
		local humanoid = character:FindFirstChild("Humanoid")
		if humanoid:GetState() ~= Enum.HumanoidStateType.Freefall then return end

		-- Perform a manual upward velocity jump
		character.HumanoidRootPart.Velocity = Vector3.new(
			character.HumanoidRootPart.Velocity.X,
			50, -- Upward force
			character.HumanoidRootPart.Velocity.Z
		)
	end
end)

-- Utility: Convert 3D position to 2D screen point
local function worldToScreen(pos)
	local screenPoint, onScreen = Camera:WorldToViewportPoint(pos)
	return Vector2.new(screenPoint.X, screenPoint.Y), onScreen, screenPoint.Z
end

-- Utility: Distance from screen center
local function getScreenDistance(vec)
	local center = Vector2.new(Camera.ViewportSize.X / 2, Camera.ViewportSize.Y / 2)
	return (vec - center).Magnitude
end

-- Find closest visible head for Aim Assist
local function getClosestTarget()
	local closest = nil
	local closestDist = AIM_RADIUS

	for _, player in ipairs(Players:GetPlayers()) do
		if player ~= LocalPlayer and player.Character and player.Character:FindFirstChild(HEAD_NAME) then
			local head = player.Character[HEAD_NAME]
			local screenPos, onScreen, depth = worldToScreen(head.Position)

			if onScreen and depth > 0 then
				local dist = getScreenDistance(screenPos)
				if dist < closestDist then
					-- Raycast check (not aiming through walls)
					local origin = Camera.CFrame.Position
					local direction = (head.Position - origin)
					local rayParams = RaycastParams.new()
					rayParams.FilterDescendantsInstances = {LocalPlayer.Character}
					rayParams.FilterType = Enum.RaycastFilterType.Blacklist

					local result = workspace:Raycast(origin, direction, rayParams)
					if not result or result.Instance:IsDescendantOf(player.Character) then
						closest = head
						closestDist = dist
					end
				end
			end
		end
	end

	return closest
end

-- Aim Assist Logic (smooth aiming, only when right mouse button is held down)
local function startAimAssist()
	while aiming and UserInputService:IsMouseButtonPressed(Enum.UserInputType.MouseButton2) do
		local target = getClosestTarget()
		if target then
			local camPos = Camera.CFrame.Position
			local direction = (target.Position - camPos).Unit
			local targetCFrame = CFrame.new(camPos, camPos + direction)
			Camera.CFrame = Camera.CFrame:Lerp(targetCFrame, AIM_SMOOTHNESS)
		end
		wait(0.02)  -- Refresh rate for aiming
	end
end

-- Listen for when right mouse button is pressed
UserInputService.InputBegan:Connect(function(input)
	if input.UserInputType == Enum.UserInputType.MouseButton2 and aiming then
		startAimAssist()
	end
end)

-- Create and refresh nametags
local function clearNametags()
	for _, player in Players:GetPlayers() do
		if player.Character and player.Character:FindFirstChild(HEAD_NAME) then
			local head = player.Character[HEAD_NAME]
			local existingTag = head:FindFirstChild(TAG_NAME)
			if existingTag then
				existingTag:Destroy()
			end
		end
	end
end

local function createNametag(player)
	if not player.Character then return end
	local head = player.Character:FindFirstChild(HEAD_NAME)
	if not head or head:FindFirstChild(TAG_NAME) then return end
	if not SHOW_LOCAL_PLAYER and player == LocalPlayer then return end

	local billboard = Instance.new("BillboardGui")
	billboard.Name = TAG_NAME
	billboard.Adornee = head
	billboard.Size = UDim2.new(0, 100, 0, 30)
	billboard.StudsOffset = Vector3.new(0, 2.5, 0)
	billboard.AlwaysOnTop = true
	billboard.Parent = head

	local textLabel = Instance.new("TextLabel")
	textLabel.Size = UDim2.new(1, 0, 1, 0)
	textLabel.BackgroundTransparency = 1
	textLabel.TextScaled = true
	textLabel.Text = player.Name
	textLabel.TextColor3 = Color3.new(1, 1, 1)
	textLabel.Font = Enum.Font.SourceSansBold
	textLabel.Parent = billboard
end

local function applyNametags()
	for _, player in ipairs(Players:GetPlayers()) do
		if player.Character and player.Character:FindFirstChild(HEAD_NAME) then
			createNametag(player)
		end
	end
end

-- Refresh nametags every 5 seconds
while true do
	clearNametags()
	applyNametags()
	task.wait(REFRESH_INTERVAL)
end
