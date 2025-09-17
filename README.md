local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local player = Players.LocalPlayer
local playerGui = player:WaitForChild("PlayerGui")

local GalaxyWorldFX = ReplicatedStorage:WaitForChild("Remotes"):WaitForChild("Effects"):WaitForChild("GalaxyWorldFX")

local selectedPlayer = nil
local playerButtons = {}
local espMarkers = {}
local turboLoop = {}
local turboPaused = {} 
local turboInterval = 3 -- intervalo inicial (em segundos)

local screenGui = Instance.new("ScreenGui")
screenGui.Name = "GojoDomainFXGui"
screenGui.Parent = playerGui
screenGui.ResetOnSpawn = false

-- Frame que vai segurar os botões principais
local dragFrame = Instance.new("Frame")
dragFrame.Size = UDim2.new(0,120,0,40)
dragFrame.Position = UDim2.new(0,20,0,20)
dragFrame.BackgroundTransparency = 1
dragFrame.Active = true
dragFrame.Parent = screenGui

-- Função para criar botão dentro do dragFrame
local function criarBotao(texto, cor, posX)
    local button = Instance.new("TextButton")
    button.Size = UDim2.new(0,30,0,30)
    button.Position = UDim2.new(0,posX,0,5)
    button.BackgroundTransparency = 0.2
    button.BackgroundColor3 = cor
    button.TextColor3 = Color3.fromRGB(255,255,255)
    button.Text = texto
    button.Font = Enum.Font.GothamBold
    button.TextScaled = true
    button.AutoButtonColor = false
    button.Parent = dragFrame
    return button
end

-- Botões principais
local destroyButton = criarBotao("❌", Color3.fromRGB(255,0,0), 0)
local turboButton = criarBotao("♾️", Color3.fromRGB(255,255,0), 35)
local buttonPrincipal = criarBotao("🔮", Color3.fromRGB(0,170,255), 70)

-- Sistema de arrastar (funciona no celular e PC)
do
    local dragging = false
    local dragStart, startPos

    local function update(input)
        local delta = input.Position - dragStart
        dragFrame.Position = UDim2.new(
            startPos.X.Scale,
            startPos.X.Offset + delta.X,
            startPos.Y.Scale,
            startPos.Y.Offset + delta.Y
        )
    end

    dragFrame.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 
        or input.UserInputType == Enum.UserInputType.Touch then
            dragging = true
            dragStart = input.Position
            startPos = dragFrame.Position

            input.Changed:Connect(function()
                if input.UserInputState == Enum.UserInputState.End then
                    dragging = false
                end
            end)
        end
    end)

    UserInputService.InputChanged:Connect(function(input)
        if dragging and (input.UserInputType == Enum.UserInputType.MouseMovement 
        or input.UserInputType == Enum.UserInputType.Touch) then
            update(input)
        end
    end)
end

--------------------------------------------------------------------------------
-- Daqui pra baixo é o SEU script original, só mudei o turboInterval pra 3
--------------------------------------------------------------------------------

-- Botões + e -
local plusButton = Instance.new("TextButton")
plusButton.Size = UDim2.new(0,30,0,30)
plusButton.Position = UDim2.new(0, 20, 1, -60)
plusButton.BackgroundColor3 = Color3.fromRGB(0,170,0)
plusButton.TextColor3 = Color3.fromRGB(255,255,255)
plusButton.Text = "+"
plusButton.Font = Enum.Font.GothamBold
plusButton.TextScaled = true
plusButton.Visible = false
plusButton.Parent = screenGui

local minusButton = Instance.new("TextButton")
minusButton.Size = UDim2.new(0,30,0,30)
minusButton.Position = UDim2.new(0, 55, 1, -60)
minusButton.BackgroundColor3 = Color3.fromRGB(170,0,0)
minusButton.TextColor3 = Color3.fromRGB(255,255,255)
minusButton.Text = "-"
minusButton.Font = Enum.Font.GothamBold
minusButton.TextScaled = true
minusButton.Visible = false
minusButton.Parent = screenGui

-- Botão Pausar/Retomar Turbo
local pauseResumeButton = Instance.new("TextButton")
pauseResumeButton.Size = UDim2.new(0,30,0,30)
pauseResumeButton.Position = UDim2.new(0, 20, 1, -95)
pauseResumeButton.BackgroundColor3 = Color3.fromRGB(50,50,200)
pauseResumeButton.TextColor3 = Color3.fromRGB(255,255,255)
pauseResumeButton.Text = "⏸️"
pauseResumeButton.Font = Enum.Font.GothamBold
pauseResumeButton.TextScaled = true
pauseResumeButton.Visible = false
pauseResumeButton.Parent = screenGui

-- Label do intervalo
local intervalLabel = Instance.new("TextLabel")
intervalLabel.Size = UDim2.new(0,40,0,30)
intervalLabel.Position = UDim2.new(0, 85, 1, -60)
intervalLabel.BackgroundTransparency = 0.2
intervalLabel.BackgroundColor3 = Color3.fromRGB(100,100,100)
intervalLabel.TextColor3 = Color3.fromRGB(255,255,255)
intervalLabel.Text = tostring(turboInterval).."s"
intervalLabel.Font = Enum.Font.GothamBold
intervalLabel.TextScaled = true
intervalLabel.Visible = false
intervalLabel.Parent = screenGui

local function atualizarLabel()
    intervalLabel.Text = tostring(turboInterval).."s"
end

plusButton.MouseButton1Click:Connect(function()
    turboInterval = turboInterval + 1
    atualizarLabel()
end)

minusButton.MouseButton1Click:Connect(function()
    turboInterval = math.max(1, turboInterval - 1)
    atualizarLabel()
end)

pauseResumeButton.MouseButton1Click:Connect(function()
    if not selectedPlayer then return end
    if not turboLoop[selectedPlayer] then return end

    turboPaused[selectedPlayer] = not turboPaused[selectedPlayer]
    if turboPaused[selectedPlayer] then
        pauseResumeButton.Text = "▶️"
    else
        pauseResumeButton.Text = "⏸️"
    end
end)

-- Função para remover jogador da lista
local function removerJogador(playerTarget)
    if playerButtons[playerTarget] then
        playerButtons[playerTarget]:Destroy()
        playerButtons[playerTarget] = nil
    end
    if espMarkers[playerTarget] then
        espMarkers[playerTarget]:Destroy()
        espMarkers[playerTarget] = nil
    end
    turboLoop[playerTarget] = nil
    turboPaused[playerTarget] = nil

    -- Reorganiza botões restantes
    local i = 0
    for _, b in pairs(playerButtons) do
        b.Position = UDim2.new(0,20,0,60 + i*16)
        i = i + 1
    end
end

-- Criar botão do jogador
local function criarBotaoDoJogador(playerTarget)
    if playerButtons[playerTarget] then return end
    local index = 0
    for _, _ in pairs(playerButtons) do index = index + 1 end

    local btn = Instance.new("TextButton")  
    btn.Size = UDim2.new(0,100,0,14)  
    btn.Position = UDim2.new(0,20,0,60 + index*16)  
    btn.BackgroundColor3 = Color3.fromRGB(0,255,0)  
    btn.TextColor3 = Color3.fromRGB(0,0,0)  
    btn.Text = playerTarget.Name  
    btn.Font = Enum.Font.GothamBold  
    btn.TextScaled = true  
    btn.Parent = screenGui  

    -- Clique manual
    btn.MouseButton1Click:Connect(function()  
        if turboLoop[playerTarget] then  
            turboLoop[playerTarget] = false  
        end  
        if playerTarget.Character and playerTarget.Character:FindFirstChild("HumanoidRootPart") then  
            GalaxyWorldFX:FireServer("B", playerTarget.Character, playerTarget.Character.HumanoidRootPart.CFrame)  
        end  
        removerJogador(playerTarget)
    end)  

    -- Remove automaticamente se o player morrer
    if playerTarget.Character and playerTarget.Character:FindFirstChild("Humanoid") then
        playerTarget.Character.Humanoid.Died:Connect(function()
            removerJogador(playerTarget)
        end)
    end
    -- Também remove quando respawnar, ligando de novo no Died
    playerTarget.CharacterAdded:Connect(function(char)
        local hum = char:WaitForChild("Humanoid")
        hum.Died:Connect(function()
            removerJogador(playerTarget)
        end)
    end)

    playerButtons[playerTarget] = btn
end

-- Atualiza ESP
RunService.RenderStepped:Connect(function()
    for p, highlight in pairs(espMarkers) do
        if p.Character then
            highlight.Adornee = p.Character
        end
    end
end)

-- Remove se player sair
Players.PlayerRemoving:Connect(function(p)
    removerJogador(p)
end)

-- Seleção pelo mouse
local mouse = player:GetMouse()
mouse.Button1Down:Connect(function()
    local target = mouse.Target
    if target and target.Parent then
        local targetPlayer = Players:GetPlayerFromCharacter(target.Parent)
        if targetPlayer then
            selectedPlayer = targetPlayer
            print("Jogador selecionado: " .. targetPlayer.Name)
        end
    end
end)

-- Disparar efeitos
local function dispararEventos(principal)
    if selectedPlayer and selectedPlayer.Character then
        local hrp = selectedPlayer.Character:FindFirstChild("HumanoidRootPart")
        if hrp then
            if principal then
                GalaxyWorldFX:FireServer("B", selectedPlayer.Character, hrp.CFrame)
                GalaxyWorldFX:FireServer("T", selectedPlayer.Character, hrp.CFrame)

                if not espMarkers[selectedPlayer] then  
                    local highlight = Instance.new("Highlight")  
                    highlight.Adornee = selectedPlayer.Character  
                    highlight.FillColor = Color3.fromRGB(0,255,0)  
                    highlight.FillTransparency = 0.5  
                    highlight.OutlineColor = Color3.fromRGB(0,255,0)  
                    highlight.OutlineTransparency = 0  
                    highlight.Parent = selectedPlayer.Character  
                    espMarkers[selectedPlayer] = highlight  
                end  

                criarBotaoDoJogador(selectedPlayer)  
            else  
                GalaxyWorldFX:FireServer("B", selectedPlayer.Character, hrp.CFrame)  
            end  
        end  
    else  
        warn("Nenhum jogador selecionado!")  
    end
end

buttonPrincipal.MouseButton1Click:Connect(function()
    dispararEventos(true)
end)

-- Botão Turbo ♾️
turboButton.MouseButton1Click:Connect(function()
    if not selectedPlayer then return end

    criarBotaoDoJogador(selectedPlayer)  
    local btn = playerButtons[selectedPlayer]  
    btn.BackgroundColor3 = Color3.fromRGB(255,0,0)  

    if not espMarkers[selectedPlayer] and selectedPlayer.Character then  
        local highlight = Instance.new("Highlight")  
        highlight.Adornee = selectedPlayer.Character  
        highlight.FillColor = Color3.fromRGB(0,255,0)  
        highlight.FillTransparency = 0.5  
        highlight.OutlineColor = Color3.fromRGB(0,255,0)  
        highlight.OutlineTransparency = 0  
        highlight.Parent = selectedPlayer.Character  
        espMarkers[selectedPlayer] = highlight  
    end  

    -- Evento B uma vez  
    if selectedPlayer.Character and selectedPlayer.Character:FindFirstChild("HumanoidRootPart") then  
        GalaxyWorldFX:FireServer("B", selectedPlayer.Character, selectedPlayer.Character.HumanoidRootPart.CFrame)  
    end  

    plusButton.Visible = true  
    minusButton.Visible = true  
    intervalLabel.Visible = true  
    pauseResumeButton.Visible = true  

    turboLoop[selectedPlayer] = true  
    turboPaused[selectedPlayer] = false
    spawn(function()  
        local loopStarted = false  
        while turboLoop[selectedPlayer] do  
            if selectedPlayer then  
                local hrp  
                if selectedPlayer.Character then  
                    hrp = selectedPlayer.Character:FindFirstChild("HumanoidRootPart") or selectedPlayer.Character:WaitForChild("HumanoidRootPart", 2)  
                end  

                if hrp and not turboPaused[selectedPlayer] then  
                    GalaxyWorldFX:FireServer("T", selectedPlayer.Character, hrp.CFrame)  
                    if not loopStarted then  
                        loopStarted = true  
                    else  
                        wait(turboInterval)  
                    end  
                else  
                    RunService.Heartbeat:Wait()
                end  

                if espMarkers[selectedPlayer] and selectedPlayer.Character then  
                    espMarkers[selectedPlayer].Adornee = selectedPlayer.Character  
                end  
            end  
        end  

        if btn then  
            btn.BackgroundColor3 = Color3.fromRGB(0,255,0)  
        end  
        plusButton.Visible = false  
        minusButton.Visible = false  
        intervalLabel.Visible = false  
        pauseResumeButton.Visible = false  
    end)
end)

destroyButton.MouseButton1Click:Connect(function()
    screenGui:Destroy()
    for _, esp in pairs(espMarkers) do
        esp:Destroy()
    end
    espMarkers = {}
    turboLoop = {}
    turboPaused = {}
    plusButton.Visible = false
    minusButton.Visible = false
    intervalLabel.Visible = false
    pauseResumeButton.Visible = false
end)
