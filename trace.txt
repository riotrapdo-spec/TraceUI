local Players = game:GetService("Players")
local TweenService = game:GetService("TweenService")
local RunService = game:GetService("RunService")
local HttpService = game:GetService("HttpService")
local UserInputService = game:GetService("UserInputService")
local StarterGui = game:GetService("StarterGui")
local Lighting = game:GetService("Lighting")
local Stats = game:GetService("Stats")
local TeleportService = game:GetService("TeleportService")

local Player = Players.LocalPlayer
local Mouse = Player:GetMouse()

local TraceUI = {}
TraceUI.__index = TraceUI

local Colors = {
    Background = Color3.fromRGB(12, 12, 18),
    DarkBackground = Color3.fromRGB(8, 8, 14),
    Panel = Color3.fromRGB(18, 18, 28),
    PanelLight = Color3.fromRGB(25, 25, 38),
    Accent = Color3.fromRGB(120, 80, 255),
    AccentDark = Color3.fromRGB(80, 50, 200),
    AccentGlow = Color3.fromRGB(150, 100, 255),
    Secondary = Color3.fromRGB(255, 60, 120),
    Tertiary = Color3.fromRGB(0, 220, 180),
    Text = Color3.fromRGB(240, 240, 255),
    TextDim = Color3.fromRGB(140, 140, 170),
    TextDark = Color3.fromRGB(80, 80, 110),
    Border = Color3.fromRGB(40, 40, 65),
    Success = Color3.fromRGB(50, 255, 130),
    Warning = Color3.fromRGB(255, 200, 50),
    Error = Color3.fromRGB(255, 60, 80),
    Transparent = Color3.fromRGB(0, 0, 0),
}

local function Create(className, properties, children)
    local instance = Instance.new(className)
    for prop, value in pairs(properties or {}) do
        if prop ~= "Parent" then
            instance[prop] = value
        end
    end
    for _, child in pairs(children or {}) do
        child.Parent = instance
    end
    if properties and properties.Parent then
        instance.Parent = properties.Parent
    end
    return instance
end

local function Tween(instance, duration, properties, style, direction)
    local tween = TweenService:Create(
        instance,
        TweenInfo.new(duration, style or Enum.EasingStyle.Quint, direction or Enum.EasingDirection.Out),
        properties
    )
    tween:Play()
    return tween
end

local function AddCorner(parent, radius)
    return Create("UICorner", {
        CornerRadius = UDim.new(0, radius or 8),
        Parent = parent
    })
end

local function AddStroke(parent, color, thickness, transparency)
    return Create("UIStroke", {
        Color = color or Colors.Border,
        Thickness = thickness or 1,
        Transparency = transparency or 0,
        Parent = parent
    })
end

local function AddPadding(parent, top, bottom, left, right)
    return Create("UIPadding", {
        PaddingTop = UDim.new(0, top or 8),
        PaddingBottom = UDim.new(0, bottom or 8),
        PaddingLeft = UDim.new(0, left or 8),
        PaddingRight = UDim.new(0, right or 8),
        Parent = parent
    })
end

local function AddGradient(parent, color1, color2, rotation)
    return Create("UIGradient", {
        Color = ColorSequence.new({
            ColorSequenceKeypoint.new(0, color1 or Colors.Accent),
            ColorSequenceKeypoint.new(1, color2 or Colors.Secondary)
        }),
        Rotation = rotation or 45,
        Parent = parent
    })
end

local function GetPlatform()
    if UserInputService.TouchEnabled and not UserInputService.KeyboardEnabled then
        return "Mobile"
    end
    return "PC"
end

local Platform = GetPlatform()
local ScaleFactor = Platform == "Mobile" and 0.85 or 1

local function CreateParticleSystem(parent, count, color1, color2)
    local particles = {}
    for i = 1, (count or 30) do
        local size = math.random(2, 6)
        local particle = Create("Frame", {
            Size = UDim2.new(0, size, 0, size),
            Position = UDim2.new(math.random() * 0.95, 0, math.random() * 0.95, 0),
            BackgroundColor3 = i % 2 == 0 and (color1 or Colors.Accent) or (color2 or Colors.Secondary),
            BackgroundTransparency = math.random(40, 80) / 100,
            BorderSizePixel = 0,
            ZIndex = 2,
            Parent = parent
        })
        AddCorner(particle, size)

        Create("ImageLabel", {
            Size = UDim2.new(3, 0, 3, 0),
            Position = UDim2.new(-1, 0, -1, 0),
            BackgroundTransparency = 1,
            Image = "rbxassetid://5028857084",
            ImageColor3 = particle.BackgroundColor3,
            ImageTransparency = 0.85,
            ZIndex = 1,
            Parent = particle
        })

        table.insert(particles, {frame = particle, speedX = (math.random() - 0.5) * 0.3, speedY = -math.random() * 0.2 - 0.05, baseSize = size})
    end

    local connection
    connection = RunService.RenderStepped:Connect(function(dt)
        if not parent or not parent.Parent then
            connection:Disconnect()
            return
        end
        for _, p in pairs(particles) do
            if p.frame and p.frame.Parent then
                local pos = p.frame.Position
                local newX = pos.X.Scale + p.speedX * dt
                local newY = pos.Y.Scale + p.speedY * dt

                if newX > 1 then newX = 0 elseif newX < 0 then newX = 1 end
                if newY < -0.05 then newY = 1.05 end

                p.frame.Position = UDim2.new(newX, 0, newY, 0)
                p.frame.BackgroundTransparency = 0.4 + math.sin(tick() * 2 + newX * 10) * 0.3
                p.frame.Size = UDim2.new(0, p.baseSize + math.sin(tick() * 3) * 1.5, 0, p.baseSize + math.sin(tick() * 3) * 1.5)
            end
        end
    end)

    return {particles = particles, connection = connection}
end

local function CreateRipple(button)
    button.ClipsDescendants = true

    button.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            local ripple = Create("Frame", {
                Size = UDim2.new(0, 0, 0, 0),
                Position = UDim2.new(0, input.Position.X - button.AbsolutePosition.X, 0, input.Position.Y - button.AbsolutePosition.Y),
                AnchorPoint = Vector2.new(0.5, 0.5),
                BackgroundColor3 = Color3.fromRGB(255, 255, 255),
                BackgroundTransparency = 0.7,
                BorderSizePixel = 0,
                ZIndex = button.ZIndex + 5,
                Parent = button
            })
            AddCorner(ripple, 999)

            local maxSize = math.max(button.AbsoluteSize.X, button.AbsoluteSize.Y) * 2.5
            Tween(ripple, 0.6, {Size = UDim2.new(0, maxSize, 0, maxSize), BackgroundTransparency = 1})

            task.delay(0.6, function()
                if ripple then ripple:Destroy() end
            end)
        end
    end)
end

local function GlitchText(label, finalText, duration)
    local chars = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789!@#$%^&*()_+-=[]{}|;':\",./<>?"
    local startTime = tick()

    local conn
    conn = RunService.RenderStepped:Connect(function()
        local elapsed = tick() - startTime
        local progress = math.min(elapsed / duration, 1)
        local revealCount = math.floor(progress * #finalText)

        local result = ""
        for i = 1, #finalText do
            if i <= revealCount then
                result = result .. finalText:sub(i, i)
            else
                local randIndex = math.random(1, #chars)
                result = result .. chars:sub(randIndex, randIndex)
            end
        end

        label.Text = result

        if progress >= 1 then
            label.Text = finalText
            conn:Disconnect()
        end
    end)

    return conn
end

local function CreateScannerLine(parent)
    local scanner = Create("Frame", {
        Size = UDim2.new(1, 0, 0, 2),
        Position = UDim2.new(0, 0, 0, 0),
        BackgroundColor3 = Colors.Accent,
        BackgroundTransparency = 0.3,
        BorderSizePixel = 0,
        ZIndex = 50,
        Parent = parent
    })

    Create("Frame", {
        Size = UDim2.new(1, 0, 0, 30),
        Position = UDim2.new(0, 0, 0.5, 0),
        AnchorPoint = Vector2.new(0, 0.5),
        BackgroundTransparency = 1,
        Parent = scanner
    })

    Create("Frame", {
        Size = UDim2.new(1, 0, 0, 20),
        AnchorPoint = Vector2.new(0, 0.5),
        Position = UDim2.new(0, 0, 0.5, 0),
        BackgroundColor3 = Colors.Accent,
        BackgroundTransparency = 0.85,
        BorderSizePixel = 0,
        Parent = scanner
    })

    task.spawn(function()
        while scanner and scanner.Parent do
            Tween(scanner, 2, {Position = UDim2.new(0, 0, 1, 0)}, Enum.EasingStyle.Linear)
            task.wait(2)
            scanner.Position = UDim2.new(0, 0, 0, 0)
        end
    end)

    return scanner
end

local function ShowLoadingScreen()
    local ScreenGui = Create("ScreenGui", {
        Name = "TraceUI_Loading",
        ResetOnSpawn = false,
        ZIndexBehavior = Enum.ZIndexBehavior.Sibling,
        IgnoreGuiInset = true,
        Parent = Player:WaitForChild("PlayerGui")
    })

    local Background = Create("Frame", {
        Size = UDim2.new(1, 0, 1, 0),
        BackgroundColor3 = Color3.fromRGB(5, 5, 10),
        BorderSizePixel = 0,
        Parent = ScreenGui
    })

    local GridBG = Create("Frame", {
        Size = UDim2.new(1.5, 0, 1.5, 0),
        Position = UDim2.new(-0.25, 0, -0.25, 0),
        BackgroundTransparency = 1,
        Parent = Background
    })

    for i = 0, 20 do
        Create("Frame", {
            Size = UDim2.new(1, 0, 0, 1),
            Position = UDim2.new(0, 0, i / 20, 0),
            BackgroundColor3 = Colors.Accent,
            BackgroundTransparency = 0.92,
            BorderSizePixel = 0,
            Parent = GridBG
        })
        Create("Frame", {
            Size = UDim2.new(0, 1, 1, 0),
            Position = UDim2.new(i / 20, 0, 0, 0),
            BackgroundColor3 = Colors.Accent,
            BackgroundTransparency = 0.92,
            BorderSizePixel = 0,
            Parent = GridBG
        })
    end

    local particleSystem = CreateParticleSystem(Background, 50, Colors.Accent, Colors.Secondary)
    CreateScannerLine(Background)

    for i = 1, 8 do
        local hex = Create("Frame", {
            Size = UDim2.new(0, math.random(60, 150), 0, math.random(60, 150)),
            Position = UDim2.new(math.random() * 0.8, 0, math.random() * 0.8, 0),
            BackgroundColor3 = Colors.Accent,
            BackgroundTransparency = 0.95,
            Rotation = math.random(0, 45),
            BorderSizePixel = 0,
            Parent = Background
        })
        AddCorner(hex, 4)
        AddStroke(hex, Colors.Accent, 1, 0.85)

        Tween(hex, math.random(30, 60) / 10, {
            Rotation = hex.Rotation + 360,
            Position = UDim2.new(math.random() * 0.8, 0, math.random() * 0.8, 0)
        }, Enum.EasingStyle.Linear)
    end

    local CenterFrame = Create("Frame", {
        Size = UDim2.new(0, 500 * ScaleFactor, 0, 400 * ScaleFactor),
        Position = UDim2.new(0.5, 0, 0.5, 0),
        AnchorPoint = Vector2.new(0.5, 0.5),
        BackgroundTransparency = 1,
        Parent = Background
    })

    local LogoGlow = Create("ImageLabel", {
        Size = UDim2.new(0, 250 * ScaleFactor, 0, 250 * ScaleFactor),
        Position = UDim2.new(0.5, 0, 0.25, 0),
        AnchorPoint = Vector2.new(0.5, 0.5),
        BackgroundTransparency = 1,
        Image = "rbxassetid://5028857084",
        ImageColor3 = Colors.Accent,
        ImageTransparency = 0.7,
        ZIndex = 1,
        Parent = CenterFrame
    })

    local LogoFrame = Create("ImageLabel", {
        Size = UDim2.new(0, 120 * ScaleFactor, 0, 120 * ScaleFactor),
        Position = UDim2.new(0.5, 0, 0.25, 0),
        AnchorPoint = Vector2.new(0.5, 0.5),
        BackgroundColor3 = Colors.Panel,
        BackgroundTransparency = 0.3,
        Image = "rbxassetid://71906491784395",
        ScaleType = Enum.ScaleType.Fit,
        ZIndex = 5,
        Parent = CenterFrame
    })
    AddCorner(LogoFrame, 60)
    AddStroke(LogoFrame, Colors.Accent, 2, 0)

    local Ring1 = Create("Frame", {
        Size = UDim2.new(0, 160 * ScaleFactor, 0, 160 * ScaleFactor),
        Position = UDim2.new(0.5, 0, 0.25, 0),
        AnchorPoint = Vector2.new(0.5, 0.5),
        BackgroundTransparency = 1,
        ZIndex = 3,
        Parent = CenterFrame
    })
    AddCorner(Ring1, 80)
    AddStroke(Ring1, Colors.Accent, 2, 0.3)

    local Ring2 = Create("Frame", {
        Size = UDim2.new(0, 190 * ScaleFactor, 0, 190 * ScaleFactor),
        Position = UDim2.new(0.5, 0, 0.25, 0),
        AnchorPoint = Vector2.new(0.5, 0.5),
        BackgroundTransparency = 1,
        ZIndex = 2,
        Parent = CenterFrame
    })
    AddCorner(Ring2, 95)
    AddStroke(Ring2, Colors.Secondary, 1.5, 0.5)

    local Ring3 = Create("Frame", {
        Size = UDim2.new(0, 220 * ScaleFactor, 0, 220 * ScaleFactor),
        Position = UDim2.new(0.5, 0, 0.25, 0),
        AnchorPoint = Vector2.new(0.5, 0.5),
        BackgroundTransparency = 1,
        ZIndex = 2,
        Parent = CenterFrame
    })
    AddCorner(Ring3, 110)
    AddStroke(Ring3, Colors.Tertiary, 1, 0.7)

    task.spawn(function()
        while Ring1 and Ring1.Parent do
            Ring1.Rotation = Ring1.Rotation + 1
            Ring2.Rotation = Ring2.Rotation - 0.7
            Ring3.Rotation = Ring3.Rotation + 0.4
            RunService.RenderStepped:Wait()
        end
    end)

    task.spawn(function()
        while LogoFrame and LogoFrame.Parent do
            Tween(LogoFrame, 1.5, {Size = UDim2.new(0, 130 * ScaleFactor, 0, 130 * ScaleFactor)}, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut)
            Tween(LogoGlow, 1.5, {ImageTransparency = 0.5, Size = UDim2.new(0, 280 * ScaleFactor, 0, 280 * ScaleFactor)}, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut)
            task.wait(1.5)
            Tween(LogoFrame, 1.5, {Size = UDim2.new(0, 120 * ScaleFactor, 0, 120 * ScaleFactor)}, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut)
            Tween(LogoGlow, 1.5, {ImageTransparency = 0.7, Size = UDim2.new(0, 250 * ScaleFactor, 0, 250 * ScaleFactor)}, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut)
            task.wait(1.5)
        end
    end)

    local Title = Create("TextLabel", {
        Size = UDim2.new(1, 0, 0, 45 * ScaleFactor),
        Position = UDim2.new(0, 0, 0.52, 0),
        BackgroundTransparency = 1,
        Text = "",
        TextColor3 = Colors.Text,
        TextSize = 38 * ScaleFactor,
        Font = Enum.Font.GothamBold,
        TextTransparency = 0,
        ZIndex = 10,
        Parent = CenterFrame
    })

    AddGradient(Title, Colors.Accent, Colors.Secondary, 0)

    local Subtitle = Create("TextLabel", {
        Size = UDim2.new(1, 0, 0, 25 * ScaleFactor),
        Position = UDim2.new(0, 0, 0.62, 0),
        BackgroundTransparency = 1,
        Text = "",
        TextColor3 = Colors.TextDim,
        TextSize = 16 * ScaleFactor,
        Font = Enum.Font.GothamMedium,
        TextTransparency = 0,
        ZIndex = 10,
        Parent = CenterFrame
    })

    local LoadBarBG = Create("Frame", {
        Size = UDim2.new(0.7, 0, 0, 6 * ScaleFactor),
        Position = UDim2.new(0.15, 0, 0.75, 0),
        BackgroundColor3 = Colors.PanelLight,
        BorderSizePixel = 0,
        ZIndex = 10,
        Parent = CenterFrame
    })
    AddCorner(LoadBarBG, 3)

    local LoadBarFill = Create("Frame", {
        Size = UDim2.new(0, 0, 1, 0),
        BackgroundColor3 = Colors.Accent,
        BorderSizePixel = 0,
        ZIndex = 11,
        Parent = LoadBarBG
    })
    AddCorner(LoadBarFill, 3)
    AddGradient(LoadBarFill, Colors.Accent, Colors.Secondary, 0)

    local LoadBarGlow = Create("Frame", {
        Size = UDim2.new(0, 0, 0, 20),
        Position = UDim2.new(0, 0, 0.5, 0),
        AnchorPoint = Vector2.new(0, 0.5),
        BackgroundColor3 = Colors.Accent,
        BackgroundTransparency = 0.7,
        BorderSizePixel = 0,
        ZIndex = 10,
        Parent = LoadBarBG
    })
    AddCorner(LoadBarGlow, 10)

    local StatusText = Create("TextLabel", {
        Size = UDim2.new(1, 0, 0, 20 * ScaleFactor),
        Position = UDim2.new(0, 0, 0.82, 0),
        BackgroundTransparency = 1,
        Text = "Initializing...",
        TextColor3 = Colors.TextDim,
        TextSize = 12 * ScaleFactor,
        Font = Enum.Font.GothamMedium,
        ZIndex = 10,
        Parent = CenterFrame
    })

    local PercentText = Create("TextLabel", {
        Size = UDim2.new(1, 0, 0, 20 * ScaleFactor),
        Position = UDim2.new(0, 0, 0.88, 0),
        BackgroundTransparency = 1,
        Text = "0%",
        TextColor3 = Colors.Accent,
        TextSize = 14 * ScaleFactor,
        Font = Enum.Font.GothamBold,
        ZIndex = 10,
        Parent = CenterFrame
    })

    Create("TextLabel", {
        Size = UDim2.new(1, 0, 0, 15),
        Position = UDim2.new(0, 0, 1, -25),
        BackgroundTransparency = 1,
        Text = "TraceUI v2.0 | " .. Platform .. " | " .. os.date("%H:%M"),
        TextColor3 = Colors.TextDark,
        TextSize = 11,
        Font = Enum.Font.Gotham,
        ZIndex = 10,
        Parent = Background
    })

    task.wait(0.5)
    GlitchText(Title, "TraceUI", 1.5)
    task.wait(1)
    GlitchText(Subtitle, "\" Innovation is Creation \"", 1.2)

    local loadSteps = {
        {text = "Connecting to TraceUI servers...", duration = 0.6},
        {text = "Loading core modules...", duration = 0.5},
        {text = "Initializing UI framework...", duration = 0.4},
        {text = "Loading wallpaper engine...", duration = 0.5},
        {text = "Setting up FPS booster...", duration = 0.6},
        {text = "Configuring server scanner...", duration = 0.5},
        {text = "Loading effects system...", duration = 0.4},
        {text = "Applying custom theme...", duration = 0.5},
        {text = "Loading decal: 71906491784395...", duration = 0.6},
        {text = "Finalizing setup...", duration = 0.4},
        {text = "Welcome to TraceUI!", duration = 0.3},
    }

    task.wait(0.8)

    local totalSteps = #loadSteps
    for i, step in ipairs(loadSteps) do
        StatusText.Text = step.text
        local targetPercent = math.floor((i / totalSteps) * 100)
        local targetScale = i / totalSteps

        Tween(LoadBarFill, step.duration, {Size = UDim2.new(targetScale, 0, 1, 0)})
        Tween(LoadBarGlow, step.duration, {Size = UDim2.new(targetScale, 0, 0, 20)})

        task.spawn(function()
            local startPercent = tonumber(PercentText.Text:gsub("%%", "")) or 0
            local frames = math.ceil(step.duration * 30)
            for f = 1, frames do
                local p = math.floor(startPercent + (targetPercent - startPercent) * (f / frames))
                PercentText.Text = p .. "%"
                task.wait(step.duration / frames)
            end
            PercentText.Text = targetPercent .. "%"
        end)

        task.wait(step.duration)
    end

    task.wait(0.5)

    local flash = Create("Frame", {
        Size = UDim2.new(1, 0, 1, 0),
        BackgroundColor3 = Colors.Accent,
        BackgroundTransparency = 1,
        ZIndex = 100,
        Parent = Background
    })

    Tween(flash, 0.15, {BackgroundTransparency = 0.3})
    task.wait(0.15)
    Tween(flash, 0.5, {BackgroundTransparency = 1})
    task.wait(0.3)

    Tween(Background, 0.8, {BackgroundTransparency = 1})
    for _, child in pairs(Background:GetDescendants()) do
        if child:IsA("Frame") then
            Tween(child, 0.6, {BackgroundTransparency = 1})
        elseif child:IsA("TextLabel") or child:IsA("TextButton") then
            Tween(child, 0.6, {TextTransparency = 1})
        elseif child:IsA("ImageLabel") then
            Tween(child, 0.6, {ImageTransparency = 1})
        elseif child:IsA("UIStroke") then
            Tween(child, 0.6, {Transparency = 1})
        end
    end

    if particleSystem and particleSystem.connection then
        particleSystem.connection:Disconnect()
    end

    task.wait(1)
    ScreenGui:Destroy()
end

local NotificationHolder

local function CreateNotificationSystem(parent)
    NotificationHolder = Create("Frame", {
        Size = UDim2.new(0, 300 * ScaleFactor, 1, 0),
        Position = UDim2.new(1, -310 * ScaleFactor, 0, 0),
        BackgroundTransparency = 1,
        ZIndex = 200,
        Parent = parent
    })

    Create("UIListLayout", {
        SortOrder = Enum.SortOrder.LayoutOrder,
        Padding = UDim.new(0, 8),
        HorizontalAlignment = Enum.HorizontalAlignment.Center,
        VerticalAlignment = Enum.VerticalAlignment.Bottom,
        Parent = NotificationHolder
    })

    AddPadding(NotificationHolder, 0, 10, 0, 0)
end

function TraceUI:Notify(title, message, duration, notifType)
    if not NotificationHolder then return end

    local typeColors = {
        info = Colors.Accent,
        success = Colors.Success,
        warning = Colors.Warning,
        error = Colors.Error,
    }

    local accentColor = typeColors[notifType or "info"] or Colors.Accent

    local Notif = Create("Frame", {
        Size = UDim2.new(1, -10, 0, 70 * ScaleFactor),
        BackgroundColor3 = Colors.Panel,
        BackgroundTransparency = 0.1,
        BorderSizePixel = 0,
        ZIndex = 201,
        ClipsDescendants = true,
        Parent = NotificationHolder
    })
    AddCorner(Notif, 10)
    AddStroke(Notif, accentColor, 1, 0.5)

    Create("Frame", {
        Size = UDim2.new(0, 4, 0.7, 0),
        Position = UDim2.new(0, 8, 0.15, 0),
        BackgroundColor3 = accentColor,
        BorderSizePixel = 0,
        ZIndex = 202,
        Parent = Notif
    })

    Create("TextLabel", {
        Size = UDim2.new(1, -30, 0, 20),
        Position = UDim2.new(0, 22, 0, 10),
        BackgroundTransparency = 1,
        Text = title or "Notification",
        TextColor3 = Colors.Text,
        TextSize = 14 * ScaleFactor,
        Font = Enum.Font.GothamBold,
        TextXAlignment = Enum.TextXAlignment.Left,
        ZIndex = 202,
        Parent = Notif
    })

    Create("TextLabel", {
        Size = UDim2.new(1, -30, 0, 30),
        Position = UDim2.new(0, 22, 0, 32),
        BackgroundTransparency = 1,
        Text = message or "",
        TextColor3 = Colors.TextDim,
        TextSize = 12 * ScaleFactor,
        Font = Enum.Font.Gotham,
        TextXAlignment = Enum.TextXAlignment.Left,
        TextWrapped = true,
        ZIndex = 202,
        Parent = Notif
    })

    local progressBar = Create("Frame", {
        Size = UDim2.new(1, 0, 0, 2),
        Position = UDim2.new(0, 0, 1, -2),
        BackgroundColor3 = accentColor,
        BorderSizePixel = 0,
        ZIndex = 203,
        Parent = Notif
    })

    Notif.Position = UDim2.new(1.5, 0, 0, 0)
    Tween(Notif, 0.5, {Position = UDim2.new(0, 5, 0, 0)}, Enum.EasingStyle.Back)

    Tween(progressBar, duration or 3, {Size = UDim2.new(0, 0, 0, 2)}, Enum.EasingStyle.Linear)

    task.delay(duration or 3, function()
        Tween(Notif, 0.4, {Position = UDim2.new(1.5, 0, 0, 0), BackgroundTransparency = 1})
        task.wait(0.5)
        if Notif then Notif:Destroy() end
    end)
end

function TraceUI.new()
    local self = setmetatable({}, TraceUI)

    ShowLoadingScreen()

    self.ScreenGui = Create("ScreenGui", {
        Name = "TraceUI_Main",
        ResetOnSpawn = false,
        ZIndexBehavior = Enum.ZIndexBehavior.Sibling,
        Parent = Player:WaitForChild("PlayerGui")
    })

    CreateNotificationSystem(self.ScreenGui)

    self.MainFrame = Create("Frame", {
        Size = UDim2.new(0, 620 * ScaleFactor, 0, 420 * ScaleFactor),
        Position = UDim2.new(0.5, 0, 0.5, 0),
        AnchorPoint = Vector2.new(0.5, 0.5),
        BackgroundColor3 = Colors.Background,
        BorderSizePixel = 0,
        ClipsDescendants = true,
        Parent = self.ScreenGui
    })
    AddCorner(self.MainFrame, 12)
    AddStroke(self.MainFrame, Colors.Border, 1, 0.3)

    self.WallpaperFrame = Create("ImageLabel", {
        Size = UDim2.new(1, 0, 1, 0),
        BackgroundTransparency = 1,
        ImageTransparency = 0.85,
        ScaleType = Enum.ScaleType.Crop,
        ZIndex = 0,
        Parent = self.MainFrame
    })

    self.BGParticles = CreateParticleSystem(self.MainFrame, 15, Colors.Accent, Colors.Secondary)

    self.TitleBar = Create("Frame", {
        Size = UDim2.new(1, 0, 0, 40 * ScaleFactor),
        BackgroundColor3 = Colors.DarkBackground,
        BackgroundTransparency = 0.3,
        BorderSizePixel = 0,
        ZIndex = 20,
        Parent = self.MainFrame
    })

    Create("Frame", {
        Size = UDim2.new(1, 0, 0, 1),
        Position = UDim2.new(0, 0, 1, 0),
        BackgroundColor3 = Colors.Accent,
        BackgroundTransparency = 0.6,
        BorderSizePixel = 0,
        ZIndex = 21,
        Parent = self.TitleBar
    })

    Create("ImageLabel", {
        Size = UDim2.new(0, 24 * ScaleFactor, 0, 24 * ScaleFactor),
        Position = UDim2.new(0, 10, 0.5, 0),
        AnchorPoint = Vector2.new(0, 0.5),
        BackgroundTransparency = 1,
        Image = "rbxassetid://71906491784395",
        ScaleType = Enum.ScaleType.Fit,
        ZIndex = 22,
        Parent = self.TitleBar
    })

    local TitleText = Create("TextLabel", {
        Size = UDim2.new(0, 150, 1, 0),
        Position = UDim2.new(0, 40 * ScaleFactor, 0, 0),
        BackgroundTransparency = 1,
        Text = "TraceUI",
        TextColor3 = Colors.Text,
        TextSize = 16 * ScaleFactor,
        Font = Enum.Font.GothamBold,
        TextXAlignment = Enum.TextXAlignment.Left,
        ZIndex = 22,
        Parent = self.TitleBar
    })
    AddGradient(TitleText, Colors.Accent, Colors.Secondary, 0)

    local VersionBadge = Create("TextLabel", {
        Size = UDim2.new(0, 35, 0, 16),
        Position = UDim2.new(0, 115 * ScaleFactor, 0.5, 0),
        AnchorPoint = Vector2.new(0, 0.5),
        BackgroundColor3 = Colors.Accent,
        BackgroundTransparency = 0.8,
        Text = "v2.0",
        TextColor3 = Colors.Accent,
        TextSize = 10,
        Font = Enum.Font.GothamBold,
        ZIndex = 22,
        Parent = self.TitleBar
    })
    AddCorner(VersionBadge, 4)

    self.FPSLabel = Create("TextLabel", {
        Size = UDim2.new(0, 80, 1, 0),
        Position = UDim2.new(1, -180 * ScaleFactor, 0, 0),
        BackgroundTransparency = 1,
        Text = "FPS: --",
        TextColor3 = Colors.Success,
        TextSize = 11 * ScaleFactor,
        Font = Enum.Font.GothamMedium,
        ZIndex = 22,
        Parent = self.TitleBar
    })

    task.spawn(function()
        while self.FPSLabel and self.FPSLabel.Parent do
            local fps = math.floor(1 / RunService.RenderStepped:Wait())
            if self.FPSLabel then
                self.FPSLabel.Text = "FPS: " .. fps
                if fps >= 50 then
                    self.FPSLabel.TextColor3 = Colors.Success
                elseif fps >= 30 then
                    self.FPSLabel.TextColor3 = Colors.Warning
                else
                    self.FPSLabel.TextColor3 = Colors.Error
                end
            end
        end
    end)

    local MinBtn = Create("TextButton", {
        Size = UDim2.new(0, 30 * ScaleFactor, 0, 30 * ScaleFactor),
        Position = UDim2.new(1, -70 * ScaleFactor, 0.5, 0),
        AnchorPoint = Vector2.new(0, 0.5),
        BackgroundColor3 = Colors.Warning,
        BackgroundTransparency = 0.8,
        Text = "—",
        TextColor3 = Colors.Warning,
        TextSize = 16,
        Font = Enum.Font.GothamBold,
        ZIndex = 22,
        Parent = self.TitleBar
    })
    AddCorner(MinBtn, 6)
    CreateRipple(MinBtn)

    MinBtn.MouseEnter:Connect(function()
        Tween(MinBtn, 0.2, {BackgroundTransparency = 0.5})
    end)
    MinBtn.MouseLeave:Connect(function()
        Tween(MinBtn, 0.2, {BackgroundTransparency = 0.8})
    end)

    local CloseBtn = Create("TextButton", {
        Size = UDim2.new(0, 30 * ScaleFactor, 0, 30 * ScaleFactor),
        Position = UDim2.new(1, -35 * ScaleFactor, 0.5, 0),
        AnchorPoint = Vector2.new(0, 0.5),
        BackgroundColor3 = Colors.Error,
        BackgroundTransparency = 0.8,
        Text = "✕",
        TextColor3 = Colors.Error,
        TextSize = 14,
        Font = Enum.Font.GothamBold,
        ZIndex = 22,
        Parent = self.TitleBar
    })
    AddCorner(CloseBtn, 6)
    CreateRipple(CloseBtn)

    CloseBtn.MouseEnter:Connect(function()
        Tween(CloseBtn, 0.2, {BackgroundTransparency = 0.5})
    end)
    CloseBtn.MouseLeave:Connect(function()
        Tween(CloseBtn, 0.2, {BackgroundTransparency = 0.8})
    end)

    local minimized = false
    local originalSize = self.MainFrame.Size

    MinBtn.MouseButton1Click:Connect(function()
        minimized = not minimized
        if minimized then
            Tween(self.MainFrame, 0.4, {Size = UDim2.new(0, 620 * ScaleFactor, 0, 40 * ScaleFactor)}, Enum.EasingStyle.Back, Enum.EasingDirection.In)
        else
            Tween(self.MainFrame, 0.4, {Size = originalSize}, Enum.EasingStyle.Back)
        end
    end)

    CloseBtn.MouseButton1Click:Connect(function()
        Tween(self.MainFrame, 0.5, {Size = UDim2.new(0, 0, 0, 0), BackgroundTransparency = 1})
        task.wait(0.5)
        self.ScreenGui:Destroy()
    end)

    self.SideBar = Create("Frame", {
        Size = UDim2.new(0, 55 * ScaleFactor, 1, -40 * ScaleFactor),
        Position = UDim2.new(0, 0, 0, 40 * ScaleFactor),
        BackgroundColor3 = Colors.DarkBackground,
        BackgroundTransparency = 0.2,
        BorderSizePixel = 0,
        ZIndex = 15,
        Parent = self.MainFrame
    })

    Create("Frame", {
        Size = UDim2.new(0, 1, 1, 0),
        Position = UDim2.new(1, 0, 0, 0),
        BackgroundColor3 = Colors.Border,
        BackgroundTransparency = 0.5,
        BorderSizePixel = 0,
        ZIndex = 16,
        Parent = self.SideBar
    })

    self.TabButtons = {}
    self.TabContents = {}
    self.ActiveTab = nil

    self.ContentArea = Create("Frame", {
        Size = UDim2.new(1, -55 * ScaleFactor, 1, -40 * ScaleFactor),
        Position = UDim2.new(0, 55 * ScaleFactor, 0, 40 * ScaleFactor),
        BackgroundTransparency = 1,
        ClipsDescendants = true,
        ZIndex = 10,
        Parent = self.MainFrame
    })

    local dragging, dragInput, dragStart, startPos

    self.TitleBar.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            dragging = true
            dragStart = input.Position
            startPos = self.MainFrame.Position

            input.Changed:Connect(function()
                if input.UserInputState == Enum.UserInputState.End then
                    dragging = false
                end
            end)
        end
    end)

    self.TitleBar.InputChanged:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch then
            dragInput = input
        end
    end)

    UserInputService.InputChanged:Connect(function(input)
        if input == dragInput and dragging then
            local delta = input.Position - dragStart
            self.MainFrame.Position = UDim2.new(
                startPos.X.Scale, startPos.X.Offset + delta.X,
                startPos.Y.Scale, startPos.Y.Offset + delta.Y
            )
        end
    end)

    self.MainFrame.Size = UDim2.new(0, 0, 0, 0)
    self.MainFrame.BackgroundTransparency = 1

    Tween(self.MainFrame, 0.6, {
        Size = UDim2.new(0, 620 * ScaleFactor, 0, 420 * ScaleFactor),
        BackgroundTransparency = 0
    }, Enum.EasingStyle.Back)

    task.wait(0.3)
    self:Notify("TraceUI", "Welcome! UI loaded successfully.", 3, "success")

    return self
end

function TraceUI:AddTab(name, icon)
    local tabIndex = #self.TabButtons + 1

    local TabBtn = Create("TextButton", {
        Size = UDim2.new(1, -10, 0, 45 * ScaleFactor),
        Position = UDim2.new(0, 5, 0, 5 + (tabIndex - 1) * (50 * ScaleFactor)),
        BackgroundColor3 = Colors.Accent,
        BackgroundTransparency = 0.95,
        Text = "",
        ZIndex = 17,
        Parent = self.SideBar
    })
    AddCorner(TabBtn, 8)
    CreateRipple(TabBtn)

    local TabIcon = Create("TextLabel", {
        Size = UDim2.new(1, 0, 0, 20),
        Position = UDim2.new(0, 0, 0, 5),
        BackgroundTransparency = 1,
        Text = icon or "⚙",
        TextColor3 = Colors.TextDim,
        TextSize = 18 * ScaleFactor,
        Font = Enum.Font.GothamBold,
        ZIndex = 18,
        Parent = TabBtn
    })

    local TabName = Create("TextLabel", {
        Size = UDim2.new(1, 0, 0, 12),
        Position = UDim2.new(0, 0, 1, -15),
        BackgroundTransparency = 1,
        Text = name,
        TextColor3 = Colors.TextDark,
        TextSize = 8 * ScaleFactor,
        Font = Enum.Font.GothamMedium,
        TextTruncate = Enum.TextTruncate.AtEnd,
        ZIndex = 18,
        Parent = TabBtn
    })

    local Indicator = Create("Frame", {
        Size = UDim2.new(0, 3, 0.5, 0),
        Position = UDim2.new(0, -1, 0.25, 0),
        BackgroundColor3 = Colors.Accent,
        BackgroundTransparency = 1,
        BorderSizePixel = 0,
        ZIndex = 19,
        Parent = TabBtn
    })
    AddCorner(Indicator, 2)

    local TabContent = Create("ScrollingFrame", {
        Size = UDim2.new(1, -20, 1, -20),
        Position = UDim2.new(0, 10, 0, 10),
        BackgroundTransparency = 1,
        ScrollBarThickness = 3,
        ScrollBarImageColor3 = Colors.Accent,
        BorderSizePixel = 0,
        CanvasSize = UDim2.new(0, 0, 0, 0),
        Visible = false,
        ZIndex = 11,
        Parent = self.ContentArea
    })

    Create("UIListLayout", {
        SortOrder = Enum.SortOrder.LayoutOrder,
        Padding = UDim.new(0, 8),
        Parent = TabContent
    })

    TabContent.ChildAdded:Connect(function()
        task.wait()
        local layout = TabContent:FindFirstChildOfClass("UIListLayout")
        if layout then
            TabContent.CanvasSize = UDim2.new(0, 0, 0, layout.AbsoluteContentSize.Y + 20)
        end
    end)

    table.insert(self.TabButtons, {button = TabBtn, icon = TabIcon, name = TabName, indicator = Indicator})
    table.insert(self.TabContents, TabContent)

    TabBtn.MouseButton1Click:Connect(function()
        self:SwitchTab(tabIndex)
    end)

    TabBtn.MouseEnter:Connect(function()
        if self.ActiveTab ~= tabIndex then
            Tween(TabBtn, 0.2, {BackgroundTransparency = 0.85})
            Tween(TabIcon, 0.2, {TextColor3 = Colors.Text})
        end
    end)
    TabBtn.MouseLeave:Connect(function()
        if self.ActiveTab ~= tabIndex then
            Tween(TabBtn, 0.2, {BackgroundTransparency = 0.95})
            Tween(TabIcon, 0.2, {TextColor3 = Colors.TextDim})
        end
    end)

    if tabIndex == 1 then
        self:SwitchTab(1)
    end

    local Tab = {}
    Tab.Content = TabContent
    Tab.UI = self

    function Tab:AddSection(title)
        return self.UI:CreateSection(TabContent, title)
    end

    function Tab:AddButton(text, callback)
        return self.UI:CreateButton(TabContent, text, callback)
    end

    function Tab:AddToggle(text, default, callback)
        return self.UI:CreateToggle(TabContent, text, default, callback)
    end

    function Tab:AddSlider(text, min, max, default, callback)
        return self.UI:CreateSlider(TabContent, text, min, max, default, callback)
    end

    function Tab:AddDropdown(text, options, default, callback)
        return self.UI:CreateDropdown(TabContent, text, options, default, callback)
    end

    function Tab:AddInput(text, placeholder, callback)
        return self.UI:CreateInput(TabContent, text, placeholder, callback)
    end

    function Tab:AddLabel(text)
        return self.UI:CreateLabel(TabContent, text)
    end

    function Tab:AddColorPicker(text, default, callback)
        return self.UI:CreateColorPicker(TabContent, text, default, callback)
    end

    function Tab:AddKeybind(text, default, callback)
        return self.UI:CreateKeybind(TabContent, text, default, callback)
    end

    return Tab
end

function TraceUI:SwitchTab(index)
    for i, tab in ipairs(self.TabButtons) do
        Tween(tab.button, 0.3, {BackgroundTransparency = 0.95})
        Tween(tab.icon, 0.3, {TextColor3 = Colors.TextDim})
        Tween(tab.name, 0.3, {TextColor3 = Colors.TextDark})
        Tween(tab.indicator, 0.3, {BackgroundTransparency = 1})

        if self.TabContents[i] then
            self.TabContents[i].Visible = false
        end
    end

    local activeTab = self.TabButtons[index]
    Tween(activeTab.button, 0.3, {BackgroundTransparency = 0.8})
    Tween(activeTab.icon, 0.3, {TextColor3 = Colors.Accent})
    Tween(activeTab.name, 0.3, {TextColor3 = Colors.Accent})
    Tween(activeTab.indicator, 0.3, {BackgroundTransparency = 0})

    if self.TabContents[index] then
        self.TabContents[index].Visible = true

        for i, child in ipairs(self.TabContents[index]:GetChildren()) do
            if child:IsA("Frame") or child:IsA("TextButton") then
                child.Position = UDim2.new(0.1, 0, child.Position.Y.Scale, child.Position.Y.Offset)
                child.BackgroundTransparency = 1
                task.delay(i * 0.03, function()
                    if child then
                        Tween(child, 0.4, {Position = UDim2.new(0, 0, child.Position.Y.Scale, child.Position.Y.Offset), BackgroundTransparency = child:GetAttribute("OriginalTransparency") or 0.1}, Enum.EasingStyle.Back)
                    end
                end)
            end
        end
    end

    self.ActiveTab = index
end

function TraceUI:CreateSection(parent, title)
    local Section = Create("Frame", {
        Size = UDim2.new(1, 0, 0, 30 * ScaleFactor),
        BackgroundTransparency = 1,
        ZIndex = 12,
        Parent = parent
    })
    Section:SetAttribute("OriginalTransparency", 1)

    Create("Frame", {
        Size = UDim2.new(0.15, 0, 0, 1),
        Position = UDim2.new(0, 0, 0.5, 0),
        AnchorPoint = Vector2.new(0, 0.5),
        BackgroundColor3 = Colors.Accent,
        BackgroundTransparency = 0.5,
        BorderSizePixel = 0,
        ZIndex = 13,
        Parent = Section
    })

    Create("TextLabel", {
        Size = UDim2.new(0.7, 0, 1, 0),
        Position = UDim2.new(0.17, 0, 0, 0),
        BackgroundTransparency = 1,
        Text = "  " .. (title or "Section") .. "  ",
        TextColor3 = Colors.Accent,
        TextSize = 12 * ScaleFactor,
        Font = Enum.Font.GothamBold,
        TextXAlignment = Enum.TextXAlignment.Left,
        TextTransparency = 0.3,
        ZIndex = 13,
        Parent = Section
    })

    Create("Frame", {
        Size = UDim2.new(0.4, 0, 0, 1),
        Position = UDim2.new(0.6, 0, 0.5, 0),
        AnchorPoint = Vector2.new(0, 0.5),
        BackgroundColor3 = Colors.Accent,
        BackgroundTransparency = 0.5,
        BorderSizePixel = 0,
        ZIndex = 13,
        Parent = Section
    })

    return Section
end

function TraceUI:CreateButton(parent, text, callback)
    local Button = Create("TextButton", {
        Size = UDim2.new(1, 0, 0, 38 * ScaleFactor),
        BackgroundColor3 = Colors.Panel,
        BackgroundTransparency = 0.1,
        Text = "",
        AutoButtonColor = false,
        ZIndex = 12,
        Parent = parent
    })
    AddCorner(Button, 8)
    AddStroke(Button, Colors.Border, 1, 0.6)
    CreateRipple(Button)
    Button:SetAttribute("OriginalTransparency", 0.1)

    local AccentBar = Create("Frame", {
        Size = UDim2.new(0, 4, 0.5, 0),
        Position = UDim2.new(0, 8, 0.25, 0),
        BackgroundColor3 = Colors.Accent,
        BorderSizePixel = 0,
        ZIndex = 13,
        Parent = Button
    })
    AddCorner(AccentBar, 2)
    AddGradient(AccentBar, Colors.Accent, Colors.Secondary, 90)

    Create("TextLabel", {
        Size = UDim2.new(1, -50, 1, 0),
        Position = UDim2.new(0, 20, 0, 0),
        BackgroundTransparency = 1,
        Text = text or "Button",
        TextColor3 = Colors.Text,
        TextSize = 13 * ScaleFactor,
        Font = Enum.Font.GothamMedium,
        TextXAlignment = Enum.TextXAlignment.Left,
        ZIndex = 13,
        Parent = Button
    })

    Create("TextLabel", {
        Size = UDim2.new(0, 20, 1, 0),
        Position = UDim2.new(1, -25, 0, 0),
        BackgroundTransparency = 1,
        Text = "→",
        TextColor3 = Colors.TextDim,
        TextSize = 14,
        Font = Enum.Font.GothamBold,
        ZIndex = 13,
        Parent = Button
    })

    Button.MouseEnter:Connect(function()
        Tween(Button, 0.2, {BackgroundTransparency = 0})
        Tween(AccentBar, 0.2, {Size = UDim2.new(0, 4, 0.7, 0)})
        local stroke = Button:FindFirstChildOfClass("UIStroke")
        if stroke then Tween(stroke, 0.2, {Color = Colors.Accent, Transparency = 0.3}) end
    end)
    Button.MouseLeave:Connect(function()
        Tween(Button, 0.2, {BackgroundTransparency = 0.1})
        Tween(AccentBar, 0.2, {Size = UDim2.new(0, 4, 0.5, 0)})
        local stroke = Button:FindFirstChildOfClass("UIStroke")
        if stroke then Tween(stroke, 0.2, {Color = Colors.Border, Transparency = 0.6}) end
    end)

    Button.MouseButton1Click:Connect(function()
        Tween(Button, 0.1, {Size = UDim2.new(1, -4, 0, 36 * ScaleFactor)})
        task.wait(0.1)
        Tween(Button, 0.1, {Size = UDim2.new(1, 0, 0, 38 * ScaleFactor)})

        if callback then
            callback()
        end
    end)

    return Button
end

function TraceUI:CreateToggle(parent, text, default, callback)
    local toggled = default or false

    local Toggle = Create("Frame", {
        Size = UDim2.new(1, 0, 0, 38 * ScaleFactor),
        BackgroundColor3 = Colors.Panel,
        BackgroundTransparency = 0.1,
        ZIndex = 12,
        Parent = parent
    })
    AddCorner(Toggle, 8)
    AddStroke(Toggle, Colors.Border, 1, 0.6)
    Toggle:SetAttribute("OriginalTransparency", 0.1)

    Create("TextLabel", {
        Size = UDim2.new(1, -70, 1, 0),
        Position = UDim2.new(0, 12, 0, 0),
        BackgroundTransparency = 1,
        Text = text or "Toggle",
        TextColor3 = Colors.Text,
        TextSize = 13 * ScaleFactor,
        Font = Enum.Font.GothamMedium,
        TextXAlignment = Enum.TextXAlignment.Left,
        ZIndex = 13,
        Parent = Toggle
    })

    local SwitchBG = Create("Frame", {
        Size = UDim2.new(0, 44 * ScaleFactor, 0, 22 * ScaleFactor),
        Position = UDim2.new(1, -55 * ScaleFactor, 0.5, 0),
        AnchorPoint = Vector2.new(0, 0.5),
        BackgroundColor3 = toggled and Colors.Accent or Colors.PanelLight,
        BorderSizePixel = 0,
        ZIndex = 13,
        Parent = Toggle
    })
    AddCorner(SwitchBG, 11)

    local Knob = Create("Frame", {
        Size = UDim2.new(0, 18 * ScaleFactor, 0, 18 * ScaleFactor),
        Position = toggled and UDim2.new(1, -20 * ScaleFactor, 0.5, 0) or UDim2.new(0, 2, 0.5, 0),
        AnchorPoint = Vector2.new(0, 0.5),
        BackgroundColor3 = Colors.Text,
        BorderSizePixel = 0,
        ZIndex = 14,
        Parent = SwitchBG
    })
    AddCorner(Knob, 9)

    local KnobGlow = Create("Frame", {
        Size = UDim2.new(2, 0, 2, 0),
        Position = UDim2.new(-0.5, 0, -0.5, 0),
        BackgroundColor3 = Colors.Accent,
        BackgroundTransparency = toggled and 0.7 or 1,
        BorderSizePixel = 0,
        ZIndex = 13,
        Parent = Knob
    })
    AddCorner(KnobGlow, 20)

    local ToggleBtn = Create("TextButton", {
        Size = UDim2.new(1, 0, 1, 0),
        BackgroundTransparency = 1,
        Text = "",
        ZIndex = 15,
        Parent = Toggle
    })
    CreateRipple(ToggleBtn)

    ToggleBtn.MouseButton1Click:Connect(function()
        toggled = not toggled

        Tween(SwitchBG, 0.3, {BackgroundColor3 = toggled and Colors.Accent or Colors.PanelLight})
        Tween(Knob, 0.3, {Position = toggled and UDim2.new(1, -20 * ScaleFactor, 0.5, 0) or UDim2.new(0, 2, 0.5, 0)}, Enum.EasingStyle.Back)
        Tween(KnobGlow, 0.3, {BackgroundTransparency = toggled and 0.7 or 1})

        if callback then
            callback(toggled)
        end
    end)

    ToggleBtn.MouseEnter:Connect(function()
        Tween(Toggle, 0.2, {BackgroundTransparency = 0})
    end)
    ToggleBtn.MouseLeave:Connect(function()
        Tween(Toggle, 0.2, {BackgroundTransparency = 0.1})
    end)

    local toggleObj = {
        Set = function(_, value)
            toggled = value
            Tween(SwitchBG, 0.3, {BackgroundColor3 = toggled and Colors.Accent or Colors.PanelLight})
            Tween(Knob, 0.3, {Position = toggled and UDim2.new(1, -20 * ScaleFactor, 0.5, 0) or UDim2.new(0, 2, 0.5, 0)})
            Tween(KnobGlow, 0.3, {BackgroundTransparency = toggled and 0.7 or 1})
            if callback then callback(toggled) end
        end
    }

    return toggleObj
end

function TraceUI:CreateSlider(parent, text, min, max, default, callback)
    min = min or 0
    max = max or 100
    default = default or min

    local SliderFrame = Create("Frame", {
        Size = UDim2.new(1, 0, 0, 55 * ScaleFactor),
        BackgroundColor3 = Colors.Panel,
        BackgroundTransparency = 0.1,
        ZIndex = 12,
        Parent = parent
    })
    AddCorner(SliderFrame, 8)
    AddStroke(SliderFrame, Colors.Border, 1, 0.6)
    SliderFrame:SetAttribute("OriginalTransparency", 0.1)

    Create("TextLabel", {
        Size = UDim2.new(0.6, 0, 0, 20),
        Position = UDim2.new(0, 12, 0, 5),
        BackgroundTransparency = 1,
        Text = text or "Slider",
        TextColor3 = Colors.Text,
        TextSize = 13 * ScaleFactor,
        Font = Enum.Font.GothamMedium,
        TextXAlignment = Enum.TextXAlignment.Left,
        ZIndex = 13,
        Parent = SliderFrame
    })

    local ValueLabel = Create("TextLabel", {
        Size = UDim2.new(0.3, 0, 0, 20),
        Position = UDim2.new(0.7, 0, 0, 5),
        BackgroundTransparency = 1,
        Text = tostring(default),
        TextColor3 = Colors.Accent,
        TextSize = 13 * ScaleFactor,
        Font = Enum.Font.GothamBold,
        TextXAlignment = Enum.TextXAlignment.Right,
        ZIndex = 13,
        Parent = SliderFrame
    })
    AddPadding(ValueLabel, 0, 0, 0, 12)

    local Track = Create("Frame", {
        Size = UDim2.new(1, -24, 0, 6 * ScaleFactor),
        Position = UDim2.new(0, 12, 0, 35 * ScaleFactor),
        BackgroundColor3 = Colors.PanelLight,
        BorderSizePixel = 0,
        ZIndex = 13,
        Parent = SliderFrame
    })
    AddCorner(Track, 3)

    local initScale = (default - min) / (max - min)
    local Fill = Create("Frame", {
        Size = UDim2.new(initScale, 0, 1, 0),
        BackgroundColor3 = Colors.Accent,
        BorderSizePixel = 0,
        ZIndex = 14,
        Parent = Track
    })
    AddCorner(Fill, 3)
    AddGradient(Fill, Colors.Accent, Colors.Secondary, 0)

    local SliderKnob = Create("Frame", {
        Size = UDim2.new(0, 16 * ScaleFactor, 0, 16 * ScaleFactor),
        Position = UDim2.new(initScale, 0, 0.5, 0),
        AnchorPoint = Vector2.new(0.5, 0.5),
        BackgroundColor3 = Colors.Text,
        BorderSizePixel = 0,
        ZIndex = 15,
        Parent = Track
    })
    AddCorner(SliderKnob, 8)
    AddStroke(SliderKnob, Colors.Accent, 2, 0)

    local SliderBtn = Create("TextButton", {
        Size = UDim2.new(1, 0, 1, 10),
        Position = UDim2.new(0, 0, 0, -5),
        BackgroundTransparency = 1,
        Text = "",
        ZIndex = 16,
        Parent = Track
    })

    local sliding = false

    local function UpdateSlider(input)
        local pos = math.clamp((input.Position.X - Track.AbsolutePosition.X) / Track.AbsoluteSize.X, 0, 1)
        local value = math.floor(min + (max - min) * pos)

        Tween(Fill, 0.1, {Size = UDim2.new(pos, 0, 1, 0)})
        Tween(SliderKnob, 0.1, {Position = UDim2.new(pos, 0, 0.5, 0)})
        ValueLabel.Text = tostring(value)

        if callback then callback(value) end
    end

    SliderBtn.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            sliding = true
            UpdateSlider(input)
        end
    end)

    SliderBtn.InputEnded:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            sliding = false
        end
    end)

    UserInputService.InputChanged:Connect(function(input)
        if sliding and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
            UpdateSlider(input)
        end
    end)

    return SliderFrame
end

function TraceUI:CreateDropdown(parent, text, options, default, callback)
    local opened = false
    options = options or {}

    local DropFrame = Create("Frame", {
        Size = UDim2.new(1, 0, 0, 38 * ScaleFactor),
        BackgroundColor3 = Colors.Panel,
        BackgroundTransparency = 0.1,
        ClipsDescendants = true,
        ZIndex = 12,
        Parent = parent
    })
    AddCorner(DropFrame, 8)
    AddStroke(DropFrame, Colors.Border, 1, 0.6)
    DropFrame:SetAttribute("OriginalTransparency", 0.1)

    local Header = Create("TextButton", {
        Size = UDim2.new(1, 0, 0, 38 * ScaleFactor),
        BackgroundTransparency = 1,
        Text = "",
        ZIndex = 13,
        Parent = DropFrame
    })
    CreateRipple(Header)

    Create("TextLabel", {
        Size = UDim2.new(0.5, 0, 1, 0),
        Position = UDim2.new(0, 12, 0, 0),
        BackgroundTransparency = 1,
        Text = text or "Dropdown",
        TextColor3 = Colors.Text,
        TextSize = 13 * ScaleFactor,
        Font = Enum.Font.GothamMedium,
        TextXAlignment = Enum.TextXAlignment.Left,
        ZIndex = 14,
        Parent = Header
    })

    local SelectedLabel = Create("TextLabel", {
        Size = UDim2.new(0.4, 0, 1, 0),
        Position = UDim2.new(0.5, 0, 0, 0),
        BackgroundTransparency = 1,
        Text = default or "Select...",
        TextColor3 = Colors.Accent,
        TextSize = 12 * ScaleFactor,
        Font = Enum.Font.GothamMedium,
        TextXAlignment = Enum.TextXAlignment.Right,
        ZIndex = 14,
        Parent = Header
    })
    AddPadding(SelectedLabel, 0, 0, 0, 30)

    local Arrow = Create("TextLabel", {
        Size = UDim2.new(0, 20, 1, 0),
        Position = UDim2.new(1, -25, 0, 0),
        BackgroundTransparency = 1,
        Text = "▼",
        TextColor3 = Colors.TextDim,
        TextSize = 10,
        Font = Enum.Font.GothamBold,
        ZIndex = 14,
        Parent = Header
    })

    local OptionContainer = Create("Frame", {
        Size = UDim2.new(1, -8, 0, 0),
        Position = UDim2.new(0, 4, 0, 40 * ScaleFactor),
        BackgroundTransparency = 1,
        ZIndex = 13,
        Parent = DropFrame
    })

    Create("UIListLayout", {
        SortOrder = Enum.SortOrder.LayoutOrder,
        Padding = UDim.new(0, 2),
        Parent = OptionContainer
    })

    for _, option in ipairs(options) do
        local OptBtn = Create("TextButton", {
            Size = UDim2.new(1, 0, 0, 30 * ScaleFactor),
            BackgroundColor3 = Colors.PanelLight,
            BackgroundTransparency = 0.3,
            Text = option,
            TextColor3 = Colors.TextDim,
            TextSize = 12 * ScaleFactor,
            Font = Enum.Font.GothamMedium,
            ZIndex = 15,
            Parent = OptionContainer
        })
        AddCorner(OptBtn, 6)

        OptBtn.MouseEnter:Connect(function()
            Tween(OptBtn, 0.15, {BackgroundTransparency = 0, TextColor3 = Colors.Accent})
        end)
        OptBtn.MouseLeave:Connect(function()
            Tween(OptBtn, 0.15, {BackgroundTransparency = 0.3, TextColor3 = Colors.TextDim})
        end)

        OptBtn.MouseButton1Click:Connect(function()
            SelectedLabel.Text = option
            opened = false
            Tween(DropFrame, 0.3, {Size = UDim2.new(1, 0, 0, 38 * ScaleFactor)}, Enum.EasingStyle.Back, Enum.EasingDirection.In)
            Tween(Arrow, 0.3, {Rotation = 0})
            if callback then callback(option) end
        end)
    end

    Header.MouseButton1Click:Connect(function()
        opened = not opened
        local totalHeight = 38 * ScaleFactor + (opened and (#options * (32 * ScaleFactor) + 8) or 0)
        Tween(DropFrame, 0.3, {Size = UDim2.new(1, 0, 0, totalHeight)}, Enum.EasingStyle.Back)
        Tween(Arrow, 0.3, {Rotation = opened and 180 or 0})
    end)

    return DropFrame
end

function TraceUI:CreateInput(parent, text, placeholder, callback)
    local InputFrame = Create("Frame", {
        Size = UDim2.new(1, 0, 0, 38 * ScaleFactor),
        BackgroundColor3 = Colors.Panel,
        BackgroundTransparency = 0.1,
        ZIndex = 12,
        Parent = parent
    })
    AddCorner(InputFrame, 8)
    AddStroke(InputFrame, Colors.Border, 1, 0.6)
    InputFrame:SetAttribute("OriginalTransparency", 0.1)

    Create("TextLabel", {
        Size = UDim2.new(0.4, 0, 1, 0),
        Position = UDim2.new(0, 12, 0, 0),
        BackgroundTransparency = 1,
        Text = text or "Input",
        TextColor3 = Colors.Text,
        TextSize = 13 * ScaleFactor,
        Font = Enum.Font.GothamMedium,
        TextXAlignment = Enum.TextXAlignment.Left,
        ZIndex = 13,
        Parent = InputFrame
    })

    local TextBox = Create("TextBox", {
        Size = UDim2.new(0.55, 0, 0, 26 * ScaleFactor),
        Position = UDim2.new(0.43, 0, 0.5, 0),
        AnchorPoint = Vector2.new(0, 0.5),
        BackgroundColor3 = Colors.DarkBackground,
        BackgroundTransparency = 0.3,
        Text = "",
        PlaceholderText = placeholder or "Enter value...",
        PlaceholderColor3 = Colors.TextDark,
        TextColor3 = Colors.Text,
        TextSize = 12 * ScaleFactor,
        Font = Enum.Font.GothamMedium,
        ClearTextOnFocus = false,
        ZIndex = 14,
        Parent = InputFrame
    })
    AddCorner(TextBox, 6)
    AddStroke(TextBox, Colors.Border, 1, 0.7)
    AddPadding(TextBox, 0, 0, 8, 8)

    TextBox.Focused:Connect(function()
        local stroke = TextBox:FindFirstChildOfClass("UIStroke")
        if stroke then Tween(stroke, 0.2, {Color = Colors.Accent, Transparency = 0}) end
    end)

    TextBox.FocusLost:Connect(function(enterPressed)
        local stroke = TextBox:FindFirstChildOfClass("UIStroke")
        if stroke then Tween(stroke, 0.2, {Color = Colors.Border, Transparency = 0.7}) end
        if callback then callback(TextBox.Text, enterPressed) end
    end)

    return TextBox
end

function TraceUI:CreateLabel(parent, text)
    local Label = Create("TextLabel", {
        Size = UDim2.new(1, 0, 0, 25 * ScaleFactor),
        BackgroundTransparency = 1,
        Text = text or "Label",
        TextColor3 = Colors.TextDim,
        TextSize = 12 * ScaleFactor,
        Font = Enum.Font.Gotham,
        TextXAlignment = Enum.TextXAlignment.Left,
        ZIndex = 13,
        Parent = parent
    })
    AddPadding(Label, 0, 0, 12, 0)
    Label:SetAttribute("OriginalTransparency", 1)

    local labelObj = {
        Set = function(_, newText)
            Label.Text = newText
        end
    }
    return labelObj
end

function TraceUI:CreateKeybind(parent, text, default, callback)
    local currentKey = default or Enum.KeyCode.E
    local listening = false

    local KeybindFrame = Create("Frame", {
        Size = UDim2.new(1, 0, 0, 38 * ScaleFactor),
        BackgroundColor3 = Colors.Panel,
        BackgroundTransparency = 0.1,
        ZIndex = 12,
        Parent = parent
    })
    AddCorner(KeybindFrame, 8)
    AddStroke(KeybindFrame, Colors.Border, 1, 0.6)
    KeybindFrame:SetAttribute("OriginalTransparency", 0.1)

    Create("TextLabel", {
        Size = UDim2.new(0.6, 0, 1, 0),
        Position = UDim2.new(0, 12, 0, 0),
        BackgroundTransparency = 1,
        Text = text or "Keybind",
        TextColor3 = Colors.Text,
        TextSize = 13 * ScaleFactor,
        Font = Enum.Font.GothamMedium,
        TextXAlignment = Enum.TextXAlignment.Left,
        ZIndex = 13,
        Parent = KeybindFrame
    })

    local KeyBtn = Create("TextButton", {
        Size = UDim2.new(0, 70 * ScaleFactor, 0, 26 * ScaleFactor),
        Position = UDim2.new(1, -80 * ScaleFactor, 0.5, 0),
        AnchorPoint = Vector2.new(0, 0.5),
        BackgroundColor3 = Colors.DarkBackground,
        Text = currentKey.Name,
        TextColor3 = Colors.Accent,
        TextSize = 12 * ScaleFactor,
        Font = Enum.Font.GothamBold,
        ZIndex = 14,
        Parent = KeybindFrame
    })
    AddCorner(KeyBtn, 6)
    AddStroke(KeyBtn, Colors.Accent, 1, 0.6)

    KeyBtn.MouseButton1Click:Connect(function()
        listening = true
        KeyBtn.Text = "..."
        Tween(KeyBtn, 0.2, {BackgroundColor3 = Colors.Accent})
        KeyBtn.TextColor3 = Colors.Background
    end)

    UserInputService.InputBegan:Connect(function(input, processed)
        if listening and input.UserInputType == Enum.UserInputType.Keyboard then
            currentKey = input.KeyCode
            KeyBtn.Text = currentKey.Name
            listening = false
            Tween(KeyBtn, 0.2, {BackgroundColor3 = Colors.DarkBackground})
            KeyBtn.TextColor3 = Colors.Accent
        end

        if not listening and input.KeyCode == currentKey and not processed then
            if callback then callback(currentKey) end
        end
    end)

    return KeybindFrame
end

function TraceUI:CreateColorPicker(parent, text, default, callback)
    default = default or Colors.Accent
    local selectedColor = default
    local pickerOpen = false

    local CPFrame = Create("Frame", {
        Size = UDim2.new(1, 0, 0, 38 * ScaleFactor),
        BackgroundColor3 = Colors.Panel,
        BackgroundTransparency = 0.1,
        ClipsDescendants = true,
        ZIndex = 12,
        Parent = parent
    })
    AddCorner(CPFrame, 8)
    AddStroke(CPFrame, Colors.Border, 1, 0.6)
    CPFrame:SetAttribute("OriginalTransparency", 0.1)

    Create("TextLabel", {
        Size = UDim2.new(0.6, 0, 0, 38 * ScaleFactor),
        Position = UDim2.new(0, 12, 0, 0),
        BackgroundTransparency = 1,
        Text = text or "Color",
        TextColor3 = Colors.Text,
        TextSize = 13 * ScaleFactor,
        Font = Enum.Font.GothamMedium,
        TextXAlignment = Enum.TextXAlignment.Left,
        ZIndex = 13,
        Parent = CPFrame
    })

    local ColorPreview = Create("TextButton", {
        Size = UDim2.new(0, 30 * ScaleFactor, 0, 22 * ScaleFactor),
        Position = UDim2.new(1, -45 * ScaleFactor, 0, 8 * ScaleFactor),
        BackgroundColor3 = selectedColor,
        Text = "",
        ZIndex = 14,
        Parent = CPFrame
    })
    AddCorner(ColorPreview, 6)
    AddStroke(ColorPreview, Colors.Border, 1, 0.3)

    local HueSlider = Create("Frame", {
        Size = UDim2.new(1, -24, 0, 20 * ScaleFactor),
        Position = UDim2.new(0, 12, 0, 45 * ScaleFactor),
        BackgroundColor3 = Color3.fromRGB(255, 255, 255),
        BorderSizePixel = 0,
        ZIndex = 14,
        Parent = CPFrame
    })
    AddCorner(HueSlider, 4)

    Create("UIGradient", {
        Color = ColorSequence.new({
            ColorSequenceKeypoint.new(0, Color3.fromHSV(0, 1, 1)),
            ColorSequenceKeypoint.new(0.17, Color3.fromHSV(0.17, 1, 1)),
            ColorSequenceKeypoint.new(0.33, Color3.fromHSV(0.33, 1, 1)),
            ColorSequenceKeypoint.new(0.5, Color3.fromHSV(0.5, 1, 1)),
            ColorSequenceKeypoint.new(0.67, Color3.fromHSV(0.67, 1, 1)),
            ColorSequenceKeypoint.new(0.83, Color3.fromHSV(0.83, 1, 1)),
            ColorSequenceKeypoint.new(1, Color3.fromHSV(1, 1, 1)),
        }),
        Parent = HueSlider
    })

    local HueKnob = Create("Frame", {
        Size = UDim2.new(0, 6, 1, 4),
        Position = UDim2.new(0, 0, 0, -2),
        BackgroundColor3 = Colors.Text,
        BorderSizePixel = 0,
        ZIndex = 15,
        Parent = HueSlider
    })
    AddCorner(HueKnob, 3)
    AddStroke(HueKnob, Colors.Background, 2)

    local HueInput = Create("TextButton", {
        Size = UDim2.new(1, 0, 1, 10),
        Position = UDim2.new(0, 0, 0, -5),
        BackgroundTransparency = 1,
        Text = "",
        ZIndex = 16,
        Parent = HueSlider
    })

    local hue, sat, val = 0.7, 1, 1
    local hueDragging = false

    HueInput.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            hueDragging = true
        end
    end)

    HueInput.InputEnded:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            hueDragging = false
        end
    end)

    UserInputService.InputChanged:Connect(function(input)
        if hueDragging and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
            hue = math.clamp((input.Position.X - HueSlider.AbsolutePosition.X) / HueSlider.AbsoluteSize.X, 0, 1)
            selectedColor = Color3.fromHSV(hue, sat, val)
            ColorPreview.BackgroundColor3 = selectedColor
            HueKnob.Position = UDim2.new(hue, -3, 0, -2)
            if callback then callback(selectedColor) end
        end
    end)

    ColorPreview.MouseButton1Click:Connect(function()
        pickerOpen = not pickerOpen
        Tween(CPFrame, 0.3, {Size = UDim2.new(1, 0, 0, pickerOpen and (75 * ScaleFactor) or (38 * ScaleFactor))}, Enum.EasingStyle.Back)
    end)

    return CPFrame
end

function TraceUI:SetWallpaper(imageId)
    if self.WallpaperFrame then
        self.WallpaperFrame.Image = "rbxassetid://" .. tostring(imageId)
        self.WallpaperFrame.ImageTransparency = 0.85
        self:Notify("Wallpaper", "Wallpaper updated!", 2, "success")
    end
end

function TraceUI:CopyGameId()
    local gameId = game.PlaceId
    if setclipboard then
        setclipboard(tostring(gameId))
        self:Notify("Game ID", "Copied: " .. tostring(gameId), 2, "success")
    else
        self:Notify("Game ID", "ID: " .. tostring(gameId) .. " (clipboard not available)", 3, "warning")
    end
end

function TraceUI:BoostFPS()
    local boosted = 0

    pcall(function()
        local terrain = workspace:FindFirstChildOfClass("Terrain")
        if terrain then
            terrain.WaterWaveSize = 0
            terrain.WaterWaveSpeed = 0
            terrain.WaterReflectance = 0
            terrain.WaterTransparency = 0
            boosted = boosted + 1
        end
    end)

    pcall(function()
        for _, effect in pairs(Lighting:GetChildren()) do
            if effect:IsA("BlurEffect") or effect:IsA("SunRaysEffect") or effect:IsA("BloomEffect") or effect:IsA("DepthOfFieldEffect") or effect:IsA("ColorCorrectionEffect") then
                effect.Enabled = false
                boosted = boosted + 1
            end
        end
    end)

    pcall(function()
        settings().Rendering.QualityLevel = Enum.QualityLevel.Level01
        boosted = boosted + 1
    end)

    pcall(function()
        Lighting.GlobalShadows = false
        Lighting.FogEnd = 9e9
        boosted = boosted + 1
    end)

    pcall(function()
        for _, v in pairs(workspace:GetDescendants()) do
            if v:IsA("Decal") or v:IsA("Texture") or v:IsA("ParticleEmitter") or v:IsA("Trail") or v:IsA("Smoke") or v:IsA("Fire") or v:IsA("Sparkles") then
                v:Destroy()
                boosted = boosted + 1
            end
        end
    end)

    pcall(function()
        UserSettings():GetService("UserGameSettings").SavedQualityLevel = Enum.SavedQualitySetting.QualityLevel1
        boosted = boosted + 1
    end)

    pcall(function()
        for _, v in pairs(workspace:GetDescendants()) do
            if v:IsA("MeshPart") then
                v.RenderFidelity = Enum.RenderFidelity.Performance
            end
        end
    end)

    self:Notify("FPS Boost", "Applied " .. boosted .. " optimizations!", 3, "success")
end

function TraceUI:FindLowPingServer()
    self:Notify("Server Scanner", "Scanning for low ping servers...", 3, "info")

    task.spawn(function()
        local success, result = pcall(function()
            local url = "https://games.roblox.com/v1/games/" .. game.PlaceId .. "/servers/Public?sortOrder=Asc&limit=100"
            local response = HttpService:JSONDecode(game:HttpGet(url))

            local bestServer = nil
            local bestPing = math.huge

            if response and response.data then
                for _, server in pairs(response.data) do
                    if server.playing and server.maxPlayers and server.playing < server.maxPlayers then
                        if server.ping and server.ping < bestPing then
                            bestPing = server.ping
                            bestServer = server
                        end
                    end
                end
            end

            return bestServer, bestPing
        end)

        if success and result then
            self:Notify("Server Found", "Best ping: " .. math.floor(result.ping or 0) .. "ms | Players: " .. (result.playing or "?"), 5, "success")

            pcall(function()
                TeleportService:TeleportToPlaceInstance(game.PlaceId, result.id, Player)
            end)
        else
            self:Notify("Server Scanner", "Could not find servers. Try again later.", 3, "error")
        end
    end)
end

function TraceUI:SetToggleKey(key)
    UserInputService.InputBegan:Connect(function(input, processed)
        if not processed and input.KeyCode == (key or Enum.KeyCode.RightShift) then
            self.MainFrame.Visible = not self.MainFrame.Visible
        end
    end)
end

function TraceUI:CreateMobileToggle()
    if Platform ~= "Mobile" then return end

    local ToggleBtn = Create("TextButton", {
        Size = UDim2.new(0, 45, 0, 45),
        Position = UDim2.new(0, 10, 0.5, 0),
        AnchorPoint = Vector2.new(0, 0.5),
        BackgroundColor3 = Colors.Accent,
        BackgroundTransparency = 0.3,
        Text = "T",
        TextColor3 = Colors.Text,
        TextSize = 18,
        Font = Enum.Font.GothamBold,
        ZIndex = 100,
        Parent = self.ScreenGui
    })
    AddCorner(ToggleBtn, 23)
    AddStroke(ToggleBtn, Colors.Accent, 2, 0.3)

    local draggingToggle = false
    local toggleDragStart, toggleStartPos

    ToggleBtn.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.Touch then
            draggingToggle = true
            toggleDragStart = input.Position
            toggleStartPos = ToggleBtn.Position
        end
    end)

    ToggleBtn.InputEnded:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.Touch then
            draggingToggle = false
        end
    end)

    UserInputService.InputChanged:Connect(function(input)
        if draggingToggle and input.UserInputType == Enum.UserInputType.Touch then
            local delta = input.Position - toggleDragStart
            ToggleBtn.Position = UDim2.new(
                toggleStartPos.X.Scale, toggleStartPos.X.Offset + delta.X,
                toggleStartPos.Y.Scale, toggleStartPos.Y.Offset + delta.Y
            )
        end
    end)

    ToggleBtn.MouseButton1Click:Connect(function()
        self.MainFrame.Visible = not self.MainFrame.Visible
        Tween(ToggleBtn, 0.2, {Rotation = self.MainFrame.Visible and 0 or 180})
    end)
end

function TraceUI:Destroy()
    if self.BGParticles and self.BGParticles.connection then
        self.BGParticles.connection:Disconnect()
    end
    if self.ScreenGui then
        Tween(self.MainFrame, 0.5, {Size = UDim2.new(0, 0, 0, 0), BackgroundTransparency = 1})
        task.wait(0.5)
        self.ScreenGui:Destroy()
    end
end

local UI = TraceUI.new()
UI:SetToggleKey(Enum.KeyCode.RightShift)
UI:CreateMobileToggle()

local HomeTab = UI:AddTab("Home", "🏠")

HomeTab:AddSection("Welcome")
HomeTab:AddLabel("TraceUI v2.0 - \"Innovation is Creation\"")
HomeTab:AddLabel("Platform: " .. Platform)
HomeTab:AddLabel("Game: " .. tostring(game.PlaceId))

HomeTab:AddSection("Quick Actions")

HomeTab:AddButton("📋 Copy Game ID", function()
    UI:CopyGameId()
end)

HomeTab:AddButton("🖼️ Set Default Wallpaper", function()
    UI:SetWallpaper("71906491784395")
end)

HomeTab:AddButton("⚡ Boost FPS", function()
    UI:BoostFPS()
end)

HomeTab:AddButton("🌐 Find Low Ping Server", function()
    UI:FindLowPingServer()
end)

local SettingsTab = UI:AddTab("Settings", "⚙")

SettingsTab:AddSection("Wallpaper")

SettingsTab:AddInput("Wallpaper ID", "Enter Roblox Image ID...", function(text, enter)
    if enter and text ~= "" then
        UI:SetWallpaper(text)
    end
end)

SettingsTab:AddSlider("Wallpaper Opacity", 0, 100, 15, function(value)
    if UI.WallpaperFrame then
        UI.WallpaperFrame.ImageTransparency = 1 - (value / 100)
    end
end)

SettingsTab:AddSection("UI Settings")

SettingsTab:AddToggle("Rainbow Border", false, function(state)
    if state then
        task.spawn(function()
            while state do
                local hue = tick() % 5 / 5
                local color = Color3.fromHSV(hue, 0.8, 1)
                local stroke = UI.MainFrame:FindFirstChildOfClass("UIStroke")
                if stroke then
                    stroke.Color = color
                end
                RunService.RenderStepped:Wait()
            end
        end)
    end
end)

SettingsTab:AddDropdown("UI Scale", {"Small", "Medium", "Large"}, "Medium", function(option)
    local scales = {Small = 0.75, Medium = 1, Large = 1.15}
    local scale = scales[option] or 1
    Tween(UI.MainFrame, 0.3, {
        Size = UDim2.new(0, 620 * scale * ScaleFactor, 0, 420 * scale * ScaleFactor)
    })
end)

SettingsTab:AddColorPicker("Accent Color", Colors.Accent, function(color)
    UI:Notify("Theme", "Accent color changed!", 2, "info")
end)

SettingsTab:AddKeybind("Toggle UI", Enum.KeyCode.RightShift, function()
    UI.MainFrame.Visible = not UI.MainFrame.Visible
end)

local PerfTab = UI:AddTab("Perf", "📊")

PerfTab:AddSection("FPS Optimization")

PerfTab:AddButton("🚀 Max FPS Boost", function()
    UI:BoostFPS()
end)

PerfTab:AddToggle("Remove Particles", false, function(state)
    if state then
        for _, v in pairs(workspace:GetDescendants()) do
            if v:IsA("ParticleEmitter") then
                v.Enabled = false
            end
        end
        UI:Notify("Performance", "Particles disabled", 2, "success")
    end
end)

PerfTab:AddToggle("Remove Shadows", false, function(state)
    Lighting.GlobalShadows = not state
    UI:Notify("Performance", state and "Shadows disabled" or "Shadows enabled", 2, "info")
end)

PerfTab:AddToggle("Low Quality Meshes", false, function(state)
    for _, v in pairs(workspace:GetDescendants()) do
        if v:IsA("MeshPart") then
            pcall(function()
                v.RenderFidelity = state and Enum.RenderFidelity.Performance or Enum.RenderFidelity.Automatic
            end)
        end
    end
end)

PerfTab:AddSlider("Render Distance", 100, 10000, 5000, function(value)
    pcall(function()
        Lighting.FogEnd = value
    end)
end)

PerfTab:AddSection("Server")

PerfTab:AddButton("🔍 Find Best Server", function()
    UI:FindLowPingServer()
end)

local pingLabel = PerfTab:AddLabel("Ping: Calculating...")

task.spawn(function()
    while true do
        local ping = math.floor(Stats.Network.ServerStatsItem["Data Ping"]:GetValue())
        pingLabel:Set("Ping: " .. ping .. "ms")
        task.wait(1)
    end
end)

local ToolsTab = UI:AddTab("Tools", "🔧")

ToolsTab:AddSection("Game Info")

ToolsTab:AddButton("📋 Copy Game ID", function()
    UI:CopyGameId()
end)

ToolsTab:AddButton("📋 Copy Server ID", function()
    if setclipboard then
        setclipboard(game.JobId)
        UI:Notify("Server ID", "Copied: " .. game.JobId:sub(1, 20) .. "...", 2, "success")
    end
end)

ToolsTab:AddButton("📋 Copy Player Count", function()
    local count = #Players:GetPlayers()
    local max = Players.MaxPlayers
    UI:Notify("Players", count .. "/" .. max .. " players", 2, "info")
end)

ToolsTab:AddSection("Teleport")

ToolsTab:AddInput("Place ID", "Enter Place ID to join...", function(text, enter)
    if enter and tonumber(text) then
        UI:Notify("Teleport", "Teleporting to " .. text .. "...", 3, "info")
        pcall(function()
            TeleportService:Teleport(tonumber(text), Player)
        end)
    end
end)

ToolsTab:AddButton("🔄 Rejoin Server", function()
    UI:Notify("Rejoin", "Rejoining server...", 2, "info")
    task.wait(1)
    pcall(function()
        TeleportService:TeleportToPlaceInstance(game.PlaceId, game.JobId, Player)
    end)
end)

ToolsTab:AddSection("Extras")

ToolsTab:AddButton("📸 Screenshot Mode", function()
    UI.MainFrame.Visible = false
    UI:Notify("Screenshot", "UI hidden for 5 seconds", 2, "info")
    task.wait(5)
    UI.MainFrame.Visible = true
end)

local InfoTab = UI:AddTab("Info", "ℹ")

InfoTab:AddSection("TraceUI Information")
InfoTab:AddLabel("Name: TraceUI v2.0")
InfoTab:AddLabel("\"Innovation is Creation\"")
InfoTab:AddLabel("Platform: " .. Platform)
InfoTab:AddLabel("Decal ID: 71906491784395")
InfoTab:AddLabel("")
InfoTab:AddSection("Controls")
InfoTab:AddLabel("PC: RightShift to toggle UI")
InfoTab:AddLabel("Mobile: Use floating button")
InfoTab:AddLabel("Drag title bar to move")
InfoTab:AddLabel("")
InfoTab:AddSection("Features")
InfoTab:AddLabel("✓ Custom Wallpaper System")
InfoTab:AddLabel("✓ Copy Game ID")
InfoTab:AddLabel("✓ FPS Booster")
InfoTab:AddLabel("✓ Low Ping Server Finder")
InfoTab:AddLabel("✓ Particle Effects")
InfoTab:AddLabel("✓ Animated Loading Screen")
InfoTab:AddLabel("✓ Mobile & PC Compatible")
InfoTab:AddLabel("✓ Color Picker / Keybinds")
InfoTab:AddLabel("✓ Rainbow Effects")

task.delay(1, function()
    UI:Notify("TraceUI", "\"Innovation is Creation\" - Press RightShift to toggle", 5, "info")
end)

return TraceUI