-- Checa se a GUI já existe
if game.CoreGui:FindFirstChild("CaioHub") then
    game.CoreGui.CaioHub:Destroy()
end

local player = game.Players.LocalPlayer
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")

-- Variáveis
local positions = {}
local maxPositions = 6
local character, humanoid, hrp
local isTeleguiding = false
local currentPositionIndex = 1

-- Atualizar referências do personagem
local function updateCharacter()
    character = player.Character or player.CharacterAdded:Wait()
    humanoid = character:WaitForChild("Humanoid")
    hrp = character:WaitForChild("HumanoidRootPart")

    -- === ANTI-DIED / Anti-travamento ===
    humanoid.HealthChanged:Connect(function()
        if humanoid.Health < humanoid.MaxHealth then
            humanoid.Health = humanoid.MaxHealth
        end
    end)
    humanoid.Died:Connect(function()
        humanoid.Health = humanoid.MaxHealth
    end)
end

updateCharacter()
player.CharacterAdded:Connect(updateCharacter)

-- GUI
local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "CaioHub"
ScreenGui.Parent = game.CoreGui

-- Ícone principal
local Icon = Instance.new("TextButton")
Icon.Size = UDim2.new(0,50,0,50)
Icon.Position = UDim2.new(0,50,0,50)
Icon.Text = "C"
Icon.Font = Enum.Font.GothamBold
Icon.TextScaled = true
Icon.TextColor3 = Color3.fromRGB(255,255,255)
Icon.BackgroundColor3 = Color3.fromRGB(0,0,0)
Icon.BorderSizePixel = 0
Icon.Parent = ScreenGui

local UICornerIcon = Instance.new("UICorner")
UICornerIcon.CornerRadius = UDim.new(0,10)
UICornerIcon.Parent = Icon

local UIStrokeIcon = Instance.new("UIStroke")
UIStrokeIcon.Color = Color3.fromRGB(255,255,255)
UIStrokeIcon.Thickness = 2
UIStrokeIcon.Parent = Icon

-- Frame principal
local Frame = Instance.new("Frame")
Frame.Size = UDim2.new(0,250,0,400)
Frame.Position = UDim2.new(0,120,0,50)
Frame.BackgroundColor3 = Color3.fromRGB(0,0,0)
Frame.BorderSizePixel = 0
Frame.Visible = false
Frame.Parent = ScreenGui

local UIStroke = Instance.new("UIStroke")
UIStroke.Color = Color3.fromRGB(255,255,255)
UIStroke.Thickness = 2
UIStroke.Parent = Frame

local UICorner = Instance.new("UICorner")
UICorner.CornerRadius = UDim.new(0,15)
UICorner.Parent = Frame

-- Título Frame
local Title = Instance.new("TextLabel")
Title.Size = UDim2.new(1,0,0,40)
Title.Position = UDim2.new(0,0,0,0)
Title.BackgroundColor3 = Color3.fromRGB(20,20,20)
Title.TextColor3 = Color3.fromRGB(255,255,255)
Title.Text = "Caio_hub"
Title.Font = Enum.Font.GothamBold
Title.TextScaled = true
Title.Parent = Frame

local TitleCorner = Instance.new("UICorner")
TitleCorner.CornerRadius = UDim.new(0,15)
TitleCorner.Parent = Title

local TitleStroke = Instance.new("UIStroke")
TitleStroke.Color = Color3.fromRGB(255,255,255)
TitleStroke.Thickness = 1
TitleStroke.Parent = Title

-- Função criar botões
local function createButton(text,posY)
    local btn = Instance.new("TextButton")
    btn.Size = UDim2.new(0,220,0,45)
    btn.Position = UDim2.new(0,15,0,posY)
    btn.BackgroundColor3 = Color3.fromRGB(40,40,40)
    btn.TextColor3 = Color3.fromRGB(255,255,255)
    btn.TextScaled = true
    btn.Font = Enum.Font.GothamBold
    btn.Text = text
    btn.Parent = Frame

    local stroke = Instance.new("UIStroke")
    stroke.Color = Color3.fromRGB(255,255,255)
    stroke.Thickness = 1
    stroke.Parent = btn

    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(0,10)
    corner.Parent = btn

    return btn
end

-- Botões
local ButtonMark = createButton("Marcar Posição",60)
local ButtonTP = createButton("Teleguiado",130)
local ButtonReset = createButton("Resetar Posições",200)

-- Abrir/fechar menu
Icon.MouseButton1Click:Connect(function()
    Frame.Visible = not Frame.Visible
end)

-- Arrastar frame
local dragging = false
local dragOffset
Frame.InputBegan:Connect(function(input)
    if input.UserInputType==Enum.UserInputType.MouseButton1 then
        dragging = true
        dragOffset = input.Position - Frame.AbsolutePosition
        input.Changed:Connect(function()
            if input.UserInputState==Enum.UserInputState.End then
                dragging = false
            end
        end)
    end
end)

UserInputService.InputChanged:Connect(function(input)
    if dragging and input.UserInputType==Enum.UserInputType.MouseMovement then
        Frame.Position = UDim2.new(0,input.Position.X-dragOffset.X,0,input.Position.Y-dragOffset.Y)
    end
end)

-- Função teleguiado atravessando obstáculos
function moveToPosition(targetPos,speed)
    speed = speed or 37.88 -- Velocidade fixa ultra
    while hrp and (hrp.Position - targetPos).Magnitude > 0.5 do
        local deltaTime = RunService.RenderStepped:Wait()
        local direction = (targetPos - hrp.Position)
        local distance = direction.Magnitude
        local moveVector = direction.Unit * math.min(speed*deltaTime,distance)

        -- Anti-puxão e bypass de obstáculos
        local ray = Ray.new(hrp.Position,moveVector)
        local hit = workspace:FindPartOnRayWithIgnoreList(ray,{character})
        if hit and hit.CanCollide then
            hrp.CFrame = hrp.CFrame + Vector3.new(moveVector.X,1,moveVector.Z)
        else
            hrp.CFrame = hrp.CFrame + moveVector
        end
    end
end

-- Botão marcar
ButtonMark.MouseButton1Click:Connect(function()
    if #positions < maxPositions then
        table.insert(positions, hrp.Position)
        ButtonMark.Text = "Posição "..#positions.." marcada"
    end
    if #positions==maxPositions then
        ButtonMark.Text = "Todas 6 posições marcadas"
    end
end)

-- Botão teleguiado
ButtonTP.MouseButton1Click:Connect(function()
    if #positions==0 then warn("Marque pelo menos uma posição!") return end
    if isTeleguiding then return end
    isTeleguiding = true
    currentPositionIndex = 1
    spawn(function()
        for i = currentPositionIndex, #positions do
            moveToPosition(positions[i],37.88)
            currentPositionIndex = i + 1
        end
        isTeleguiding = false
        currentPositionIndex = 1
    end)
end)

-- Botão resetar
ButtonReset.MouseButton1Click:Connect(function()
    positions = {}
    ButtonMark.Text = "Marcar Posição"
    isTeleguiding = false
    currentPositionIndex = 1
end)
