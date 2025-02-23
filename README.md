local Players = game.Workspace.Players
local RunService = game:GetService("RunService")
local TweenService = game:GetService("TweenService")
local Debris = game:GetService("Debris")

-- GUI หลัก
local screenGui = Instance.new("ScreenGui")
screenGui.Parent = game.Players.LocalPlayer:WaitForChild("PlayerGui")

-- สร้าง TextLabel สำหรับ Killers
local killersLabel = Instance.new("TextLabel")
killersLabel.Size = UDim2.new(0, 200, 0, 50)
killersLabel.Position = UDim2.new(0, 10, 0, 10)
killersLabel.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
killersLabel.TextColor3 = Color3.new(1, 1, 1)
killersLabel.TextScaled = true
killersLabel.Parent = screenGui

-- สร้าง TextLabel สำหรับ Survivors
local survivorsLabel = Instance.new("TextLabel")
survivorsLabel.Size = UDim2.new(0, 200, 0, 50)
survivorsLabel.Position = UDim2.new(0, 10, 0, 70)
survivorsLabel.BackgroundColor3 = Color3.fromRGB(0, 0, 255)
survivorsLabel.TextColor3 = Color3.new(1, 1, 1)
survivorsLabel.TextScaled = true
survivorsLabel.Parent = screenGui

-- ฟังก์ชันสร้างแอนิเมชันดาวที่เปลี่ยนสีไปมา
local function createRisingStar(position, colors)
    local star = Instance.new("Part")
    star.Size = Vector3.new(0.1, 0.1, 0.1)
    star.Shape = Enum.PartType.Ball
    star.Material = Enum.Material.Neon
    star.Color = colors[1] -- เริ่มต้นที่สีแรก
    star.Position = position + Vector3.new(math.random(-2, 2), -2, math.random(-2, 2))
    star.Anchored = true
    star.CanCollide = false
    star.Parent = game.Workspace

    -- แอนิเมชันโผล่จากพื้น
    local appearTween = TweenService:Create(star, TweenInfo.new(0.6, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {
        Position = star.Position + Vector3.new(0, 2, 0),
        Size = Vector3.new(0.3, 0.3, 0.3),
        Transparency = 0
    })

    -- แอนิเมชันลอยขึ้นแล้วหายไป
    local fadeTween = TweenService:Create(star, TweenInfo.new(1.2, Enum.EasingStyle.Linear, Enum.EasingDirection.Out), {
        Position = star.Position + Vector3.new(0, 4, 0),
        Transparency = 1,
        Size = Vector3.new(0.1, 0.1, 0.1)
    })

    appearTween:Play()
    appearTween.Completed:Connect(function()
        fadeTween:Play()
    end)

    -- แอนิเมชันเปลี่ยนสีไปมา
    local index = 1
    local function animateColor()
        local nextIndex = (index % #colors) + 1
        local colorTween = TweenService:Create(star, TweenInfo.new(0.8, Enum.EasingStyle.Linear), {Color = colors[nextIndex]})
        colorTween:Play()
        colorTween.Completed:Connect(function()
            index = nextIndex
            if star.Parent then
                animateColor()
            end
        end)
    end
    animateColor()

    -- ลบดาวหลังจาก 1.8 วินาที
    Debris:AddItem(star, 1.8)
end

-- ฟังก์ชันสร้างเส้นขอบ ESP
local function createOutlineESP(model, color1, color2, starColors)
    local highlight = Instance.new("Highlight")
    highlight.Parent = model
    highlight.Adornee = model
    highlight.FillTransparency = 1
    highlight.OutlineTransparency = 0

    -- แอนิเมชันเปลี่ยนสีของเส้นขอบ
    local colors = {color1, color2}
    local index = 1
    task.spawn(function()
        while highlight.Parent do
            local nextIndex = (index % #colors) + 1
            local tween = TweenService:Create(highlight, TweenInfo.new(1, Enum.EasingStyle.Linear, Enum.EasingDirection.InOut), {OutlineColor = colors[nextIndex]})
            tween:Play()
            task.wait(1)
            index = nextIndex
        end
    end)

    -- สร้างดาวที่เปลี่ยนสีไปมา
    task.spawn(function()
        while model and model.Parent do
            local rootPart = model:FindFirstChild("HumanoidRootPart")
            if rootPart then
                createRisingStar(rootPart.Position, starColors)
            end
            task.wait(0.5)
        end
    end)
end

-- ฟังก์ชันสร้าง ESP สำหรับกลุ่ม
local function createOutlineESPForGroup(group, color1, color2, starColors)
    if group then
        for _, obj in pairs(group:GetChildren()) do
            local humanoid = obj:FindFirstChildOfClass("Humanoid")
            if humanoid and obj:FindFirstChild("HumanoidRootPart") then
                createOutlineESP(obj, color1, color2, starColors)
            end
        end
    end
end

-- ฟังก์ชันอัปเดต ESP
local function updateESP()
    while true do
        -- ลบ Highlight เก่าทั้งหมด
        for _, obj in pairs(Players:GetChildren()) do
            if obj:IsA("Model") and obj:FindFirstChild("Humanoid") then
                for _, highlight in pairs(obj:GetChildren()) do
                    if highlight:IsA("Highlight") then
                        highlight:Destroy()
                    end
                end
            end
        end

        -- อัปเดต ESP สำหรับ Killers (เส้นขอบแดงสลับเหลือง + ดาวแดงเหลือง)
        local killersGroup = Players:FindFirstChild("Killers")
        if killersGroup then
            createOutlineESPForGroup(killersGroup, Color3.fromRGB(255, 0, 0), Color3.fromRGB(255, 255, 0), {Color3.fromRGB(255, 0, 0), Color3.fromRGB(255, 255, 0)})
            killersLabel.Text = "Killers: " .. #killersGroup:GetChildren()
        else
            killersLabel.Text = "Killers: 0"
        end

        -- อัปเดต ESP สำหรับ Survivors (เส้นขอบน้ำเงินสลับฟ้า + ดาวฟ้าขาว)
        local survivorsGroup = Players:FindFirstChild("Survivors")
        if survivorsGroup then
            createOutlineESPForGroup(survivorsGroup, Color3.fromRGB(0, 0, 255), Color3.fromRGB(0, 191, 255), {Color3.fromRGB(0, 191, 255), Color3.fromRGB(255, 255, 255)})
            survivorsLabel.Text = "Survivors: " .. #survivorsGroup:GetChildren()
        else
            survivorsLabel.Text = "Survivors: 0"
        end

        task.wait(5)
    end
end

updateESP()

local Players = game:GetService("Players")
local TweenService = game:GetService("TweenService")
local RunService = game:GetService("RunService")
local Lighting = game:GetService("Lighting")
local LocalPlayer = Players.LocalPlayer
local Character = LocalPlayer.Character
local PlayerGui = LocalPlayer:FindFirstChildOfClass("PlayerGui")
local Camera = game.Workspace.CurrentCamera
local Debris = game:GetService("Debris")

-- **ตั้งค่าเริ่มต้น**
Camera.FieldOfView = 90 -- FOV ปกติ
local isDead = false -- ตรวจสอบสถานะผู้เล่น

if Character and Character:FindFirstChild("Humanoid") and PlayerGui then
    local humanoid = Character.Humanoid

    -- **สร้าง ScreenGui สำหรับ UI**
    local screenGui = Instance.new("ScreenGui")
    screenGui.Name = "CrosshairGui"
    screenGui.Parent = PlayerGui

    -- **สร้าง Crosshair**
    local crosshair = Instance.new("Frame")
    crosshair.Size = UDim2.new(0, 2, 0, 2)
    crosshair.Position = UDim2.new(0.5, -1, 0.45, -1)
    crosshair.BackgroundTransparency = 1
    crosshair.Parent = screenGui

    local stroke = Instance.new("UIStroke")
    stroke.Thickness = 1
    stroke.Color = Color3.fromRGB(255, 255, 255)
    stroke.Transparency = 0.4
    stroke.Parent = crosshair

    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(1, 0)
    corner.Parent = crosshair

    -- **สร้าง FPS Label**
    local fpsLabel = Instance.new("TextLabel")
    fpsLabel.Size = UDim2.new(0, 50, 0, 20)
    fpsLabel.Position = UDim2.new(0.5, -25, 0.48, 0)
    fpsLabel.BackgroundTransparency = 1
    fpsLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
    fpsLabel.TextScaled = true
    fpsLabel.Font = Enum.Font.SourceSansBold
    fpsLabel.Parent = screenGui

    -- **อัปเดตค่า FPS**
    local lastTime = tick()
    RunService.RenderStepped:Connect(function()
        local currentTime = tick()
        local fps = math.floor(1 / (currentTime - lastTime))
        fpsLabel.Text = "FPS: " .. tostring(fps)
        lastTime = currentTime
    end)

    -- **สร้างขอบแสงสีรุ้ง**
    local highlight = Instance.new("Highlight")
    highlight.Name = "RainbowHighlight"
    highlight.Adornee = Character
    highlight.FillTransparency = 1
    highlight.OutlineTransparency = 0
    highlight.Parent = Character

    -- **สร้างแสงรอบตัวเป็นสีรุ้ง (เพิ่มระยะเป็น 10)**
    local light = Instance.new("PointLight")
    light.Parent = Character:FindFirstChild("HumanoidRootPart")
    light.Range = 10 -- เปลี่ยนจาก 5 เป็น 10
    light.Brightness = 2
    light.Shadows = true

    -- **ฟังก์ชันเปลี่ยนสีขอบแสงและแสงรอบตัว**
    local function updateRainbowColor()
        local hue = 0
        while not isDead do
            local color = Color3.fromHSV(hue, 1, 1)
            highlight.OutlineColor = color
            light.Color = color
            hue = (hue + 0.02) % 1
            task.wait(0.1)
        end
    end

    task.spawn(updateRainbowColor)

    -- **ฟังก์ชันสร้างเอฟเฟคดาวสีรุ้ง**
    local function createStarEffect()
        if isDead then return end

        local star = Instance.new("Part")
        star.Size = Vector3.new(0.5, 0.5, 0.5)
        star.Shape = Enum.PartType.Ball
        star.Material = Enum.Material.Neon
        star.Color = Color3.fromHSV(math.random(), 1, 1)
        star.Position = Character.HumanoidRootPart.Position + Vector3.new(math.random(-2, 2), -3, math.random(-2, 2))
        star.Anchored = true
        star.CanCollide = false
        star.Parent = game.Workspace

        local tweenInfo = TweenInfo.new(1, Enum.EasingStyle.Quad, Enum.EasingDirection.Out)
        local goal = {Position = star.Position + Vector3.new(0, 5, 0), Transparency = 1}
        local tween = TweenService:Create(star, tweenInfo, goal)
        tween:Play()

        tween.Completed:Connect(function()
            star:Destroy()
        end)
    end

    -- **สร้างดาวทุก 0.5 วิ**
    while not isDead do
        createStarEffect()
        task.wait(0.5)
    end

    -- **ระบบเมื่อผู้เล่นตาย**
    humanoid.Died:Connect(function()
        isDead = true

        -- **ปิดเอฟเฟคแสง**
        highlight.Enabled = false
        light.Enabled = false

        -- **สร้างเอฟเฟคเบลอ**
        local blur = Instance.new("BlurEffect")
        blur.Size = 0
        blur.Parent = Lighting
        local blurTween = TweenService:Create(blur, TweenInfo.new(1, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {Size = 70})
        blurTween:Play()

        -- **เปลี่ยนสีหน้าจอเป็นแดง**
        local redScreen = Instance.new("Frame")
        redScreen.Size = UDim2.new(1, 0, 1, 0)
        redScreen.BackgroundColor3 = Color3.fromRGB(255, 0, 0)
        redScreen.BackgroundTransparency = 1
        redScreen.Parent = screenGui

        local redTween = TweenService:Create(redScreen, TweenInfo.new(1, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {BackgroundTransparency = 0.5})
        redTween:Play()

        -- **ลด FOV เป็น 50**
        local tweenInfo = TweenInfo.new(1.5, Enum.EasingStyle.Quad, Enum.EasingDirection.Out)
        local fovTween = TweenService:Create(Camera, tweenInfo, {FieldOfView = 50})
        fovTween:Play()
    end)
end

--[[
    WARNING: Heads up! This script has not been verified by ScriptBlox. Use at your own risk!
]]
local Sprinting = game:GetService("ReplicatedStorage").Systems.Character.Game.Sprinting
local m = require(Sprinting)
m.MaxStamina = 120
m.StaminaGain = 25
m.StaminaLoss = 4
m.SprintSpeed = 26
