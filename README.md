--[[
🎯 Rastreador de Frutas 🍈
by yScanor
Versão Final - Com filtro de NPCs, troca de servidor e verificação de posse
]]

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local TweenService = game:GetService("TweenService")
local UserInputService = game:GetService("UserInputService")
local Workspace = game:GetService("Workspace")
local TeleportService = game:GetService("TeleportService")
local HttpService = game:GetService("HttpService")

local LocalPlayer = Players.LocalPlayer
local Character = LocalPlayer.Character or LocalPlayer.CharacterAdded:Wait()
local HumanoidRootPart = Character:WaitForChild("HumanoidRootPart")

-- Lista de NPCs e objetos que não devem ser marcados
local IGNORED_NAMES = {
    "Anchor",
    "Model",
    "DefaultChalice",
    "GhostShip",
    "Dragon (East)",
    "Dragon (West)",
    "Dragon (East)-Dragon (East)",
    "Dragon (West)-Dragon (West)"
}

-- Configurações de troca de servidor
local serverHopDelay = 5 -- segundos
local isServerHopping = false
local lastServerHopTime = 0

-- Função para verificar se um modelo deve ser ignorado
local function shouldIgnoreModel(model)
    -- Verifica primeiro os nomes completos exatos
    if table.find(IGNORED_NAMES, model.Name) then
        return true
    end
    
    -- Verifica se contém partes dos nomes ignorados
    for _, ignoredName in ipairs(IGNORED_NAMES) do
        if string.find(model.Name, ignoredName, 1, true) then
            return true
        end
    end
    
    return false
end

-- Função para verificar se a fruta está na mochila do jogador
local function isFruitInBackpack(fruitName)
    local backpack = LocalPlayer:FindFirstChild("Backpack")
    if not backpack then return false end
    
    for _, item in ipairs(backpack:GetChildren()) do
        if item.Name == fruitName then
            return true
        end
    end
    return false
end

-- Função para verificar se a fruta está sendo segurada por alguém
local function isFruitBeingHeld(fruit)
    local handle = fruit:FindFirstChild("Handle")
    if not handle then return false end
    
    -- Verifica se há um WeldConstraint conectando o Handle a um Character
    for _, constraint in ipairs(handle:GetDescendants()) do
        if constraint:IsA("WeldConstraint") then
            local part1 = constraint.Part1
            local part0 = constraint.Part0
            
            -- Verifica se alguma das partes conectadas pertence a um Character
            if (part1 and part1:FindFirstAncestorOfClass("Model") and part1:FindFirstAncestorOfClass("Model"):FindFirstChild("Humanoid")) or
               (part0 and part0:FindFirstAncestorOfClass("Model") and part0:FindFirstAncestorOfClass("Model"):FindFirstChild("Humanoid")) then
                return true
            end
        end
    end
    return false
end

-- Função para encontrar frutas no workspace
local function findFruitsInWorkspace()
    local fruits = {}
    local function recursiveSearch(parent)
        for _, child in pairs(parent:GetChildren()) do
            if child:IsA("Model") and child:FindFirstChild("Handle") then
                if not shouldIgnoreModel(child) then
                    table.insert(fruits, child)
                end
            elseif child:IsA("Folder") or child:IsA("Model") then
                recursiveSearch(child)
            end
        end
    end
    recursiveSearch(Workspace)
    return fruits
end

-- Criação da interface
local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "FruitTrackerGui"
ScreenGui.ResetOnSpawn = false
ScreenGui.Parent = LocalPlayer:WaitForChild("PlayerGui")

local MainFrame = Instance.new("Frame")
MainFrame.Size = UDim2.new(0, 360, 0, 400)
MainFrame.Position = UDim2.new(0.05, 0, 0.3, 0)
MainFrame.BackgroundColor3 = Color3.fromRGB(30, 30, 35)
MainFrame.BorderSizePixel = 0
MainFrame.AnchorPoint = Vector2.new(0, 0)
MainFrame.Visible = true
MainFrame.Parent = ScreenGui
MainFrame.ClipsDescendants = true
MainFrame.BackgroundTransparency = 0
MainFrame.Name = "MainFrame"

local UICorner = Instance.new("UICorner", MainFrame)
UICorner.CornerRadius = UDim.new(0, 14)

local UIStroke = Instance.new("UIStroke", MainFrame)
UIStroke.Color = Color3.fromRGB(55, 55, 60)
UIStroke.Thickness = 2
UIStroke.Transparency = 0.6

local Title = Instance.new("TextLabel")
Title.Parent = MainFrame
Title.Size = UDim2.new(1, 0, 0, 38)
Title.BackgroundTransparency = 1
Title.Text = "🎯 Rastreador de Frutas 🍈"
Title.TextColor3 = Color3.fromRGB(230, 230, 230)
Title.Font = Enum.Font.GothamBold
Title.TextSize = 24
Title.TextXAlignment = Enum.TextXAlignment.Left
Title.Position = UDim2.new(0, 16, 0, 8)
Title.TextStrokeTransparency = 0.75

local Credit = Instance.new("TextLabel")
Credit.Parent = MainFrame
Credit.Size = UDim2.new(1, -16, 0, 18)
Credit.Position = UDim2.new(0, 0, 1, -28)
Credit.BackgroundTransparency = 1
Credit.Text = "by yScanor"
Credit.TextColor3 = Color3.fromRGB(150, 150, 150)
Credit.Font = Enum.Font.Gotham
Credit.TextSize = 14
Credit.TextXAlignment = Enum.TextXAlignment.Right

local CloseButton = Instance.new("TextButton")
CloseButton.Parent = MainFrame
CloseButton.Size = UDim2.new(0, 28, 0, 28)
CloseButton.Position = UDim2.new(1, -38, 0, 6)
CloseButton.BackgroundColor3 = Color3.fromRGB(200, 50, 50)
CloseButton.Text = "×"
CloseButton.TextColor3 = Color3.fromRGB(255, 255, 255)
CloseButton.Font = Enum.Font.GothamBold
CloseButton.TextSize = 22
CloseButton.AutoButtonColor = true
CloseButton.Name = "CloseButton"
CloseButton.AnchorPoint = Vector2.new(0, 0)
CloseButton.BorderSizePixel = 0
CloseButton.ClipsDescendants = false
CloseButton.BackgroundTransparency = 0

local CloseCorner = Instance.new("UICorner", CloseButton)
CloseCorner.CornerRadius = UDim.new(0, 6)

-- Controles principais
local ControlsFrame = Instance.new("Frame")
ControlsFrame.Parent = MainFrame
ControlsFrame.Size = UDim2.new(1, -32, 0, 56)
ControlsFrame.Position = UDim2.new(0, 16, 0, 60)
ControlsFrame.BackgroundTransparency = 1

local SwitchLabel = Instance.new("TextLabel")
SwitchLabel.Parent = ControlsFrame
SwitchLabel.Size = UDim2.new(0, 130, 1, 0)
SwitchLabel.BackgroundTransparency = 1
SwitchLabel.Text = "ESP Fruta"
SwitchLabel.TextColor3 = Color3.fromRGB(230, 230, 230)
SwitchLabel.Font = Enum.Font.GothamBold
SwitchLabel.TextSize = 18
SwitchLabel.TextXAlignment = Enum.TextXAlignment.Left
SwitchLabel.Position = UDim2.new(0, 0, 0, 0)

local SwitchButton = Instance.new("TextButton")
SwitchButton.Parent = ControlsFrame
SwitchButton.Size = UDim2.new(0, 55, 0.7, 0)
SwitchButton.Position = UDim2.new(0, 140, 0.15, 0)
SwitchButton.BackgroundColor3 = Color3.fromRGB(120, 120, 120)
SwitchButton.Text = "OFF"
SwitchButton.TextColor3 = Color3.fromRGB(255, 255, 255)
SwitchButton.Font = Enum.Font.GothamBold
SwitchButton.TextSize = 16
SwitchButton.AutoButtonColor = false
SwitchButton.Name = "SwitchButton"
SwitchButton.AnchorPoint = Vector2.new(0, 0)
SwitchButton.BackgroundTransparency = 0
SwitchButton.ClipsDescendants = true

local SwitchCorner = Instance.new("UICorner", SwitchButton)
SwitchCorner.CornerRadius = UDim.new(0, 10)

local TPButton = Instance.new("TextButton")
TPButton.Parent = ControlsFrame
TPButton.Size = UDim2.new(0, 100, 0.7, 0)
TPButton.Position = UDim2.new(0, 200, 0.15, 0)
TPButton.BackgroundColor3 = Color3.fromRGB(55, 110, 220)
TPButton.Text = "TP Fruta"
TPButton.TextColor3 = Color3.fromRGB(255, 255, 255)
TPButton.Font = Enum.Font.GothamBold
TPButton.TextSize = 18
TPButton.AutoButtonColor = true
TPButton.Name = "TPButton"
TPButton.AnchorPoint = Vector2.new(0, 0)

local TPCorner = Instance.new("UICorner", TPButton)
TPCorner.CornerRadius = UDim.new(0, 10)

-- Botão para trocar de servidor
local ServerHopFrame = Instance.new("Frame")
ServerHopFrame.Parent = MainFrame
ServerHopFrame.Size = UDim2.new(1, -32, 0, 40)
ServerHopFrame.Position = UDim2.new(0, 16, 0, 120)
ServerHopFrame.BackgroundTransparency = 1

local ServerHopButton = Instance.new("TextButton")
ServerHopButton.Parent = ServerHopFrame
ServerHopButton.Size = UDim2.new(1, 0, 1, 0)
ServerHopButton.BackgroundColor3 = Color3.fromRGB(220, 80, 80)
ServerHopButton.Text = "🔄 Trocar Servidor"
ServerHopButton.TextColor3 = Color3.fromRGB(255, 255, 255)
ServerHopButton.Font = Enum.Font.GothamBold
ServerHopButton.TextSize = 16
ServerHopButton.AutoButtonColor = true
ServerHopButton.Name = "ServerHopButton"

local ServerHopCorner = Instance.new("UICorner", ServerHopButton)
ServerHopCorner.CornerRadius = UDim.new(0, 10)

-- Lista de frutas
local FruitListFrame = Instance.new("Frame")
FruitListFrame.Parent = MainFrame
FruitListFrame.Size = UDim2.new(1, -32, 0, 180)
FruitListFrame.Position = UDim2.new(0, 16, 0, 170)
FruitListFrame.BackgroundColor3 = Color3.fromRGB(25, 25, 30)
FruitListFrame.BorderSizePixel = 0
FruitListFrame.Name = "FruitListFrame"

local FruitListCorner = Instance.new("UICorner", FruitListFrame)
FruitListCorner.CornerRadius = UDim.new(0, 10)

local FruitListTitle = Instance.new("TextLabel")
FruitListTitle.Parent = FruitListFrame
FruitListTitle.Size = UDim2.new(1, 0, 0, 28)
FruitListTitle.BackgroundTransparency = 1
FruitListTitle.Text = "📝 Lista de Frutas 🍈"
FruitListTitle.TextColor3 = Color3.fromRGB(230, 230, 230)
FruitListTitle.Font = Enum.Font.GothamBold
FruitListTitle.TextSize = 18
FruitListTitle.Position = UDim2.new(0, 10, 0, 0)
FruitListTitle.TextXAlignment = Enum.TextXAlignment.Left

local FruitListSubtitle = Instance.new("TextLabel")
FruitListSubtitle.Parent = FruitListFrame
FruitListSubtitle.Size = UDim2.new(1, 0, 0, 18)
FruitListSubtitle.BackgroundTransparency = 1
FruitListSubtitle.Text = "Selecione a Fruta para o TP"
FruitListSubtitle.TextColor3 = Color3.fromRGB(180, 180, 180)
FruitListSubtitle.Font = Enum.Font.Gotham
FruitListSubtitle.TextSize = 14
FruitListSubtitle.Position = UDim2.new(0, 10, 0, 28)
FruitListSubtitle.TextXAlignment = Enum.TextXAlignment.Left

local FruitListScrolling = Instance.new("ScrollingFrame")
FruitListScrolling.Parent = FruitListFrame
FruitListScrolling.Size = UDim2.new(1, -10, 1, -50)
FruitListScrolling.Position = UDim2.new(0, 5, 0, 50)
FruitListScrolling.BackgroundTransparency = 1
FruitListScrolling.BorderSizePixel = 0
FruitListScrolling.ScrollBarThickness = 6
FruitListScrolling.ScrollBarImageColor3 = Color3.fromRGB(100, 100, 100)
FruitListScrolling.CanvasSize = UDim2.new(0, 0, 0, 0)
FruitListScrolling.AutomaticCanvasSize = Enum.AutomaticSize.Y

local FruitListLayout = Instance.new("UIListLayout")
FruitListLayout.Parent = FruitListScrolling
FruitListLayout.SortOrder = Enum.SortOrder.LayoutOrder
FruitListLayout.Padding = UDim.new(0, 5)

-- Botão para reabrir a interface
local ReopenCircle = Instance.new("TextButton")
ReopenCircle.Parent = ScreenGui
ReopenCircle.Size = UDim2.new(0, 60, 0, 60)
ReopenCircle.Position = UDim2.new(0, 14, 0.9, 0)
ReopenCircle.BackgroundColor3 = Color3.fromRGB(120, 120, 120)
ReopenCircle.Text = "🍈"
ReopenCircle.TextColor3 = Color3.fromRGB(245, 245, 245)
ReopenCircle.Font = Enum.Font.GothamBold
ReopenCircle.TextSize = 38
ReopenCircle.AutoButtonColor = true
ReopenCircle.Visible = false
ReopenCircle.Name = "ReopenCircle"

local CircleCorner = Instance.new("UICorner", ReopenCircle)
CircleCorner.CornerRadius = UDim.new(1, 0)

-- Variáveis de controle
local tracking = false
local trackedFruits = {}
local distanceLabels = {}
local highlightObjects = {}
local selectedFruit = nil
local targetFruit = nil
local isTeleporting = false

-- Função para criar highlight
local function createHighlight(part)
    if highlightObjects[part] then return highlightObjects[part] end
    local highlight = Instance.new("Highlight")
    highlight.Adornee = part.Parent
    highlight.FillColor = Color3.fromRGB(255, 50, 50)
    highlight.OutlineColor = Color3.fromRGB(200, 0, 0)
    highlight.DepthMode = Enum.HighlightDepthMode.AlwaysOnTop
    highlight.Parent = part.Parent
    highlightObjects[part] = highlight
    return highlight
end

-- Função para remover highlight
local function removeHighlight(part)
    if highlightObjects[part] then
        highlightObjects[part]:Destroy()
        highlightObjects[part] = nil
    end
end

-- Função para criar/atualizar labels de distância
local function createOrUpdateDistanceLabel(fruit)
    local part = fruit:FindFirstChild("Handle") or fruit.PrimaryPart or fruit:FindFirstChildWhichIsA("BasePart")
    if not part then return end

    local label = distanceLabels[fruit]
    if not label then
        label = Instance.new("BillboardGui")
        label.Adornee = part
        label.Size = UDim2.new(0, 120, 0, 32)
        label.AlwaysOnTop = true
        label.StudsOffset = Vector3.new(0, 2.8, 0)

        local textLabel = Instance.new("TextLabel")    
        textLabel.Parent = label    
        textLabel.BackgroundTransparency = 0.5    
        textLabel.BackgroundColor3 = Color3.fromRGB(20, 20, 20)    
        textLabel.Size = UDim2.new(1, 0, 1, 0)    
        textLabel.TextColor3 = Color3.fromRGB(255, 0, 0)    
        textLabel.Font = Enum.Font.GothamBold    
        textLabel.TextSize = 15    
        textLabel.TextStrokeTransparency = 0    
        textLabel.TextWrapped = true    
        textLabel.TextYAlignment = Enum.TextYAlignment.Center    
        textLabel.Name = "DistanceText"    
        textLabel.ClipsDescendants = true    
        textLabel.BorderSizePixel = 0    
        textLabel.ZIndex = 10    

        label.Parent = ScreenGui    
        distanceLabels[fruit] = label
    end

    local distance = (HumanoidRootPart.Position - part.Position).Magnitude
    local textLabel = label:FindFirstChild("DistanceText")
    if textLabel then
        textLabel.Text = string.format("%s\n%.2f m", fruit.Name, distance)
    end
end

-- Função para remover labels de distância
local function removeDistanceLabel(fruit)
    if distanceLabels[fruit] then
        distanceLabels[fruit]:Destroy()
        distanceLabels[fruit] = nil
    end
end

-- Limpa todos os rastreamentos
local function clearAllTracking()
    for fruit, _ in pairs(distanceLabels) do
        removeDistanceLabel(fruit)
    end
    for part, _ in pairs(highlightObjects) do
        removeHighlight(part)
    end
end

-- Atualiza a lista de frutas rastreadas
local function updateTrackedFruits()
    clearAllTracking()
    trackedFruits = findFruitsInWorkspace()
end

-- Teleporta para uma fruta específica
local function teleportToFruit(fruit)
    if not fruit then return end
    if isTeleporting then return end
    
    -- Verifica se a fruta está sendo segurada por alguém
    if isFruitBeingHeld(fruit) then
        warn("A fruta está sendo segurada por alguém! Teleporte cancelado.")
        return
    end
    
    local part = fruit:FindFirstChild("Handle") or fruit.PrimaryPart or fruit:FindFirstChildWhichIsA("BasePart")
    if not part then return end

    isTeleporting = true
    targetFruit = fruit

    local speed = 250
    local hrp = HumanoidRootPart

    while targetFruit and targetFruit.Parent and isTeleporting do
        -- Verifica se a fruta já está na mochila
        if isFruitInBackpack(fruit.Name) then
            print("Fruta coletada! Teleporte concluído.")
            break
        end
        
        -- Verifica novamente se alguém pegou a fruta durante o teleporte
        if isFruitBeingHeld(fruit) then
            warn("Alguém pegou a fruta durante o teleporte! Cancelando...")
            break
        end
        
        local targetPos = part.Position + Vector3.new(0, 5, 0)
        local direction = (targetPos - hrp.Position).Unit
        local distance = (targetPos - hrp.Position).Magnitude

        if distance < 5 then    
            break    
        end    

        hrp.CFrame = hrp.CFrame + direction * math.min(speed * RunService.Heartbeat:Wait(), distance)
    end

    isTeleporting = false
    targetFruit = nil
end

-- Tenta coletar a fruta automaticamente
local function tryPickFruit(fruit)
    local part = fruit:FindFirstChild("Handle") or fruit.PrimaryPart or fruit:FindFirstChildWhichIsA("BasePart")
    if not part then return end
    local dist = (HumanoidRootPart.Position - part.Position).Magnitude
    if dist <= 5 then
        -- Verifica se a fruta não está com alguém e não está na mochila
        if not isFruitBeingHeld(fruit) and not isFruitInBackpack(fruit.Name) then
            -- Aqui você deve adicionar o código específico do jogo para coletar a fruta
            print("Tentando coletar a fruta:", fruit.Name)
        end
    end
end

-- Atualiza a lista de frutas na UI
local function updateFruitListUI()
    -- Limpa a lista atual
    for _, child in ipairs(FruitListScrolling:GetChildren()) do
        if child:IsA("TextButton") then
            child:Destroy()
        end
    end

    -- Adiciona as frutas à lista
    for _, fruit in ipairs(trackedFruits) do
        local part = fruit:FindFirstChild("Handle") or fruit.PrimaryPart or fruit:FindFirstChildWhichIsA("BasePart")
        if part and fruit.Parent then
            local fruitButton = Instance.new("TextButton")
            fruitButton.Parent = FruitListScrolling
            fruitButton.Size = UDim2.new(1, -10, 0, 30)
            fruitButton.BackgroundColor3 = selectedFruit == fruit and Color3.fromRGB(70, 70, 80) or Color3.fromRGB(40, 40, 45)
            fruitButton.Text = fruit.Name
            fruitButton.TextColor3 = Color3.fromRGB(230, 230, 230)
            fruitButton.Font = Enum.Font.Gotham
            fruitButton.TextSize = 16
            fruitButton.AutoButtonColor = true
            fruitButton.BorderSizePixel = 0
            
            local fruitCorner = Instance.new("UICorner", fruitButton)
            fruitCorner.CornerRadius = UDim.new(0, 6)
            
            fruitButton.MouseButton1Click:Connect(function()
                selectedFruit = fruit
                updateFruitListUI()
            end)
        end
    end
end

-- Função para trocar de servidor
local function serverHop()
    if isServerHopping then return end
    if os.time() - lastServerHopTime < serverHopDelay then return end
    
    isServerHopping = true
    lastServerHopTime = os.time()
    
    -- Tenta encontrar um novo servidor
    local success, result = pcall(function()
        local servers = HttpService:JSONDecode(game:HttpGet("https://games.roblox.com/v1/games/" .. game.PlaceId .. "/servers/Public?sortOrder=Asc&limit=100"))
        for _, server in ipairs(servers.data) do
            if server.playing < server.maxPlayers and server.id ~= game.JobId then
                TeleportService:TeleportToPlaceInstance(game.PlaceId, server.id, LocalPlayer)
                return true
            end
        end
        return false
    end)
    
    if not success then
        warn("Erro ao trocar de servidor:", result)
    end
    
    isServerHopping = false
end

-- Conexão dos botões
SwitchButton.MouseButton1Click:Connect(function()
    tracking = not tracking
    if tracking then
        SwitchButton.Text = "ON"
        SwitchButton.BackgroundColor3 = Color3.fromRGB(80, 220, 80)
        updateTrackedFruits()
        updateFruitListUI()
    else
        SwitchButton.Text = "OFF"
        SwitchButton.BackgroundColor3 = Color3.fromRGB(120, 120, 120)
        clearAllTracking()
    end
end)

TPButton.MouseButton1Click:Connect(function()
    if not tracking then return end
    
    local fruitToTeleport = selectedFruit
    
    if not fruitToTeleport then
        -- Teleporta para a fruta mais próxima se nenhuma estiver selecionada
        local closestFruit = nil
        local closestDistance = math.huge
        for _, fruit in pairs(trackedFruits) do
            local part = fruit:FindFirstChild("Handle") or fruit.PrimaryPart or fruit:FindFirstChildWhichIsA("BasePart")
            if part and fruit.Parent then
                local dist = (HumanoidRootPart.Position - part.Position).Magnitude
                if dist < closestDistance then
                    closestDistance = dist
                    closestFruit = fruit
                end
            end
        end
        fruitToTeleport = closestFruit
    end
    
    if fruitToTeleport then
        coroutine.wrap(function()
            teleportToFruit(fruitToTeleport)
        end)()
    end
end)

ServerHopButton.MouseButton1Click:Connect(function()
    coroutine.wrap(serverHop)()
end)

-- Atualização contínua
RunService.Heartbeat:Connect(function()
    if tracking then
        updateTrackedFruits()
        for _, fruit in pairs(trackedFruits) do
            local part = fruit:FindFirstChild("Handle") or fruit.PrimaryPart or fruit:FindFirstChildWhichIsA("BasePart")
            if part and fruit.Parent then
                createHighlight(part)
                createOrUpdateDistanceLabel(fruit)
                tryPickFruit(fruit)
            else
                removeHighlight(part)
                removeDistanceLabel(fruit)
            end
        end
        updateFruitListUI()
    end
end)

-- Controles de interface
CloseButton.MouseButton1Click:Connect(function()
    MainFrame.Visible = false
    ReopenCircle.Visible = true
end)

ReopenCircle.MouseButton1Click:Connect(function()
    MainFrame.Visible = true
    ReopenCircle.Visible = false
end)

-- Função para tornar a interface arrastável
local function makeDraggable(guiObject)
    local dragging = false
    local dragInput, mousePos, framePos

    guiObject.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 then
            dragging = true
            mousePos = input.Position
            framePos = guiObject.Position
            input.Changed:Connect(function()
                if input.UserInputState == Enum.UserInputState.End then
                    dragging = false
                end
            end)
        end
    end)

    guiObject.InputChanged:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseMovement then
            dragInput = input
        end
    end)

    UserInputService.InputChanged:Connect(function(input)
        if input == dragInput and dragging then
            local delta = input.Position - mousePos
            guiObject.Position = UDim2.new(framePos.X.Scale, framePos.X.Offset + delta.X, framePos.Y.Scale, framePos.Y.Offset + delta.Y)
        end
    end)
end

makeDraggable(MainFrame)
makeDraggable(ReopenCircle)

-- Inicialização
updateTrackedFruits()
