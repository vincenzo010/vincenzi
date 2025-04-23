
-- Create ScreenGui
local screenGui = Instance.new("ScreenGui")
screenGui.Parent = game.Players.LocalPlayer:WaitForChild("PlayerGui")
screenGui.ResetOnSpawn = false

-- Create Frame
local frame = Instance.new("Frame")
frame.Parent = screenGui
frame.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
frame.Size = UDim2.new(0, 200, 0, 100)
frame.Position = UDim2.new(0.5, -100, 0.5, -50)
frame.Active = true
frame.Draggable = true

-- Create On Button
local onButton = Instance.new("TextButton")
onButton.Parent = frame
onButton.BackgroundColor3 = Color3.fromRGB(0, 255, 0)
onButton.Size = UDim2.new(0, 60, 0, 30)
onButton.Position = UDim2.new(0, 20, 0, 20)
onButton.Text = "On"
onButton.TextScaled = true

-- Create Off Button
local offButton = Instance.new("TextButton")
offButton.Parent = frame
offButton.BackgroundColor3 = Color3.fromRGB(255, 0, 0)
offButton.Size = UDim2.new(0, 60, 0, 30)
offButton.Position = UDim2.new(0, 120, 0, 20)
offButton.Text = "Off"
offButton.TextScaled = true

-- Create Destroy Button
local destroyButton = Instance.new("TextButton")
destroyButton.Parent = frame
destroyButton.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
destroyButton.Size = UDim2.new(0, 160, 0, 30)
destroyButton.Position = UDim2.new(0, 20, 0, 60)
destroyButton.Text = "Destroy"
destroyButton.TextScaled = true

-- Create Status Indicator
local statusLabel = Instance.new("TextLabel")
statusLabel.Parent = frame
statusLabel.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
statusLabel.Size = UDim2.new(0, 200, 0, 30)
statusLabel.Position = UDim2.new(0, 0, 0, -30)
statusLabel.Text = "WallHop by RealPortube : Off"
statusLabel.TextColor3 = Color3.fromRGB(255, 0, 0)
statusLabel.TextScaled = true

-- Variables for Wallhop Functionality
local toggle = false
local InfiniteJumpEnabled = true -- Acts as debounce
local UserInputService = game:GetService("UserInputService")
local Players = game:GetService("Players")
local Workspace = game:GetService("Workspace")
local RunService = game:GetService("RunService")

local raycastParams = RaycastParams.new()
raycastParams.FilterType = Enum.RaycastFilterType.Blacklist
-- Filter list will be updated dynamically

-- Precise wall detection function - Returns RaycastResult
local function getWallRaycastResult()
local player = Players.LocalPlayer
local character = player.Character
if not character then return nil end
local humanoidRootPart = character:FindFirstChild("HumanoidRootPart")
if not humanoidRootPart then return nil end

raycastParams.FilterDescendantsInstances = {character}

local directions = {
    humanoidRootPart.CFrame.LookVector,
    -humanoidRootPart.CFrame.LookVector,
    humanoidRootPart.CFrame.RightVector,
    -humanoidRootPart.CFrame.RightVector
}
local detectionDistance = 2
local closestHit = nil
local minDistance = detectionDistance + 1

for _, direction in pairs(directions) do
    local ray = Workspace:Raycast(
        humanoidRootPart.Position,
        direction * detectionDistance,
        raycastParams
    )
    if ray and ray.Instance then
         if ray.Distance < minDistance then
             minDistance = ray.Distance
             closestHit = ray
         end
    end
end
return closestHit
end

-- Button Functions
onButton.MouseButton1Click:Connect(function()
statusLabel.Text = "WallHop by RealPortube : On"
statusLabel.TextColor3 = Color3.fromRGB(0, 255, 0)
toggle = true
end)

offButton.MouseButton1Click:Connect(function()
statusLabel.Text = "WallHop by RealPortube : Off"
statusLabel.TextColor3 = Color3.fromRGB(255, 0, 0)
toggle = false
end)

destroyButton.MouseButton1Click:Connect(function()
screenGui:Destroy()
end)

-- Wallhop Function - ADDED ROTATION BACK TOWARDS WALL
UserInputService.JumpRequest:Connect(function()
if not toggle or not InfiniteJumpEnabled then return end

local player = Players.LocalPlayer
local character = player.Character
local humanoid = character and character:FindFirstChildOfClass("Humanoid")
local rootPart = character and character:FindFirstChild("HumanoidRootPart")
local camera = Workspace.CurrentCamera

if not (humanoid and rootPart and camera) then return end

local wallRayResult = getWallRaycastResult()

if wallRayResult then
    InfiniteJumpEnabled = false -- Start debounce immediately

    -- 1. Calculate Base Direction Away from Wall
    local wallNormal = wallRayResult.Normal
    local horizontalWallNormal = Vector3.new(wallNormal.X, 0, wallNormal.Z).Unit
    if horizontalWallNormal.Magnitude < 0.1 then
         horizontalWallNormal = (rootPart.CFrame.LookVector * Vector3.new(1,0,1)).Unit
         if horizontalWallNormal.Magnitude < 0.1 then horizontalWallNormal = Vector3.new(0,0,-1) end
    end
    local baseDirectionAwayFromWall = horizontalWallNormal -- Store this original direction

    -- 2. Get Camera's Horizontal Look Direction
    local cameraLook = camera.CFrame.LookVector
    local horizontalCameraLook = Vector3.new(cameraLook.X, 0, cameraLook.Z).Unit
    if horizontalCameraLook.Magnitude < 0.1 then horizontalCameraLook = baseDirectionAwayFromWall end

    -- 3. Calculate Initial Rotation (Away from Wall, Camera Influenced)
    local maxInfluenceAngle = math.rad(40)
    local dot = math.clamp(baseDirectionAwayFromWall:Dot(horizontalCameraLook), -1, 1)
    local angleBetween = math.acos(dot)
    local cross = baseDirectionAwayFromWall:Cross(horizontalCameraLook)
    local rotationSign = math.sign(cross.Y)
    if rotationSign == 0 then angleBetween = 0 end
    local actualInfluenceAngle = math.min(angleBetween, maxInfluenceAngle)
    local adjustmentRotation = CFrame.Angles(0, actualInfluenceAngle * rotationSign, 0)
    local initialTargetLookDirection = adjustmentRotation * baseDirectionAwayFromWall

    -- 4. Apply Initial Rotation FIRST
    rootPart.CFrame = CFrame.lookAt(rootPart.Position, rootPart.Position + initialTargetLookDirection)

    -- 5. Wait a very short moment (one frame)
    RunService.Heartbeat:Wait()

    -- 6. Apply Jump
    local didJump = false
    if humanoid and humanoid:GetState() ~= Enum.HumanoidStateType.Dead then
         humanoid:ChangeState(Enum.HumanoidStateType.Jumping)
         didJump = true -- Track if the jump actually happened
    end

    -- 7. Apply Rotation BACK towards the wall IF the jump occurred
    if didJump then
         local directionTowardsWall = -baseDirectionAwayFromWall -- Reverse the original away direction
         -- Use the root part's current position as the origin for lookAt
         rootPart.CFrame = CFrame.lookAt(rootPart.Position, rootPart.Position + directionTowardsWall)
    end

    -- Optional: Velocity boost - Consider applying it *after* the jump state change
    -- if didJump then
    --    rootPart.Velocity = rootPart.Velocity + initialTargetLookDirection * 20 -- Boost away from wall
    -- end

    -- Post-jump cooldown
    task.wait(0.15)
    InfiniteJumpEnabled = true -- End debounce
end
end)

