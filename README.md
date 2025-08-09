--[[ 🎯 FOV AIMBOT + ESP ULTRA SIMPLIFICADO
✅ Apenas 2 botões: FOV Aimbot + ESP
✅ Mira automaticamente na cabeça dos inimigos
✅ ESP mostra apenas inimigos do time oposto
✅ Se ninguém tiver time (como FIFA), considera todos como inimigos
✅ Sistema inteligente de detecção de times
✅ Ultra leve e otimizado
]]

-- CONFIG
local FOV_SIZE = 210
local FACE_OFFSET = 0.45
local FOV_CLOSE_DIST = 8 -- Distância para mira instantânea
local FOV_INSTANT_RANGE = 5 -- Range ainda mais próximo para mira super instantânea
local ESP_MAX_DISTANCE = 1000
local FOV_AIMBOT_CHECK_INTERVAL = 0.008

-- Cores para ESP
local TEAM_COLORS = {
    Axis = Color3.fromRGB(255, 165, 0), -- Laranja
    Allies = Color3.fromRGB(0, 255, 0), -- Verde
    NoTeam = Color3.fromRGB(255, 0, 0) -- Vermelho para sem time
}

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local VirtualInputManager = game:GetService("VirtualInputManager")
local LocalPlayer = Players.LocalPlayer
local Camera = workspace.CurrentCamera

-- Variáveis globais
local FOV_AIMBOT_ACTIVE = false
local ESP_ACTIVE = false
local CurrentTarget = nil
local PlayerESPs = {}
local ESPConnections = {}
local FOVAimbotButton, ESPButton

-- ============ UTILS ============

local function IsAlive(char)
    if not char then return false end
    local hum = char:FindFirstChildOfClass("Humanoid")
    return hum and hum.Health > 0
end

local function IsHeadVisible(myPos, targetChar)
    local head = targetChar and targetChar:FindFirstChild("Head")
    if not head then return false end
    local ray = Ray.new(myPos, (head.Position - myPos).Unit * (head.Position - myPos).Magnitude)
    local part = workspace:FindPartOnRayWithIgnoreList(ray, {LocalPlayer.Character, targetChar})
    return not part or part:IsDescendantOf(targetChar)
end

local function GetPlayerTeam(player)
    if not player then return "NoTeam" end
    
    -- Método 1: Verificar TeamColor
    if player.TeamColor then
        local teamColorName = tostring(player.TeamColor)
        if teamColorName:find("Bright orange") or teamColorName:find("Neon orange") or teamColorName:find("Orange") then
            return "Axis"
        elseif teamColorName:find("Bright green") or teamColorName:find("Green") or teamColorName:find("Lime green") then
            return "Allies"
        end
    end
    
    -- Método 2: Verificar Team object
    if player.Team then
        local teamName = player.Team.Name:lower()
        if teamName:find("axis") or teamName:find("orange") or teamName:find("german") or teamName:find("nazi") then
            return "Axis"
        elseif teamName:find("allies") or teamName:find("green") or teamName:find("american") or teamName:find("allied") then
            return "Allies"
        end
        
        -- Se tem team mas não é reconhecido, usa o nome do team como identificador
        if teamName ~= "" then
            return teamName
        end
    end
    
    -- Método 3: Verificar se tem character com uniform/team identifier
    if player.Character then
        for _, item in pairs(player.Character:GetChildren()) do
            if item:IsA("Shirt") or item:IsA("Pants") or item:IsA("Accessory") then
                local itemName = item.Name:lower()
                if itemName:find("axis") or itemName:find("german") or itemName:find("orange") then
                    return "Axis"
                elseif itemName:find("allies") or itemName:find("american") or itemName:find("green") then
                    return "Allies"
                end
            end
        end
    end
    
    return "NoTeam"
end

local function IsEnemy(player)
    if not player or player == LocalPlayer then return false end
    
    local myTeam = GetPlayerTeam(LocalPlayer)
    local theirTeam = GetPlayerTeam(player)
    
    -- Debug: Mostra os times detectados
    if myTeam ~= "NoTeam" or theirTeam ~= "NoTeam" then
        -- print("Meu time: " .. myTeam .. " | Time dele: " .. theirTeam .. " (" .. player.Name .. ")")
    end
    
    -- Se ambos não têm time definido, todos são inimigos (modo FIFA)
    if myTeam == "NoTeam" and theirTeam == "NoTeam" then
        return true
    end
    
    -- Se eu tenho time mas ele não tem, ele é inimigo
    if myTeam ~= "NoTeam" and theirTeam == "NoTeam" then
        return true
    end
    
    -- Se ele tem time mas eu não tenho, ele é inimigo
    if myTeam == "NoTeam" and theirTeam ~= "NoTeam" then
        return true
    end
    
    -- Se ambos temos times, são inimigos se forem diferentes
    if myTeam ~= "NoTeam" and theirTeam ~= "NoTeam" then
        return myTeam ~= theirTeam
    end
    
    return false
end

-- ============ ESP SYSTEM ============

local function CreateESP(player)
    if PlayerESPs[player] then PlayerESPs[player]:Destroy() end
    
    local char = player.Character
    if not char or not char:FindFirstChild("HumanoidRootPart") or not IsAlive(char) then return end
    
    local team = GetPlayerTeam(player)
    local espColor = TEAM_COLORS[team] or TEAM_COLORS.NoTeam
    
    local esp = Instance.new("BillboardGui")
    esp.Name = "PlayerESP_"..player.Name
    esp.Parent = char:FindFirstChild("HumanoidRootPart")
    esp.Size = UDim2.new(0, 45, 0, 22)
    esp.StudsOffset = Vector3.new(0, 2.2, 0)
    esp.AlwaysOnTop = true
    
    local nameLabel = Instance.new("TextLabel")
    nameLabel.Parent = esp
    nameLabel.Size = UDim2.new(1, 0, 0.7, 0)
    nameLabel.BackgroundTransparency = 1
    nameLabel.Text = player.Name:sub(1, 8)
    nameLabel.TextColor3 = espColor
    nameLabel.TextScaled = true
    nameLabel.Font = Enum.Font.GothamBold
    nameLabel.TextStrokeTransparency = 0
    nameLabel.TextStrokeColor3 = Color3.new(0, 0, 0)
    
    local distanceLabel = Instance.new("TextLabel")
    distanceLabel.Parent = esp
    distanceLabel.Size = UDim2.new(1, 0, 0.3, 0)
    distanceLabel.Position = UDim2.new(0, 0, 0.7, 0)
    distanceLabel.BackgroundTransparency = 1
    distanceLabel.Text = "0m"
    distanceLabel.TextColor3 = Color3.new(1, 1, 1)
    distanceLabel.TextScaled = true
    distanceLabel.Font = Enum.Font.Code
    distanceLabel.TextStrokeTransparency = 0
    distanceLabel.TextStrokeColor3 = Color3.new(0, 0, 0)
    
    PlayerESPs[player] = esp
    
    local conn
    conn = RunService.Heartbeat:Connect(function()
        if not player.Character or not LocalPlayer.Character then 
            conn:Disconnect() 
            return 
        end
        local theirHRP = player.Character:FindFirstChild("HumanoidRootPart")
        local myHRP = LocalPlayer.Character:FindFirstChild("HumanoidRootPart")
        if theirHRP and myHRP then
            local dist = math.floor((theirHRP.Position - myHRP.Position).Magnitude)
            distanceLabel.Text = dist.."m"
            esp.Enabled = dist <= ESP_MAX_DISTANCE and IsAlive(player.Character)
        end
    end)
    ESPConnections[player] = conn
end

local function RemoveESP(player)
    if PlayerESPs[player] then 
        PlayerESPs[player]:Destroy()
        PlayerESPs[player] = nil 
    end
    if ESPConnections[player] then 
        ESPConnections[player]:Disconnect()
        ESPConnections[player] = nil 
    end
end

local function UpdateESP()
    -- Remove todos os ESPs
    for p, esp in pairs(PlayerESPs) do RemoveESP(p) end
    PlayerESPs, ESPConnections = {}, {}
    
    if not ESP_ACTIVE then return end
    
    -- Cria ESP apenas para inimigos
    for _, player in pairs(Players:GetPlayers()) do
        if IsEnemy(player) and player.Character and IsAlive(player.Character) then
            CreateESP(player)
        end
    end
end

-- ============ FOV AIMBOT SYSTEM ============

local function FindFOVAimbotTarget()
    local lpchar = LocalPlayer.Character
    if not lpchar then return nil end
    local hrp = lpchar:FindFirstChild("HumanoidRootPart")
    if not hrp then return nil end
    
    local instantTarget, instantDist = nil, math.huge
    local closeTarget, closeDist = nil, math.huge
    local best, bestDist = nil, math.huge
    local screenCenter = Vector2.new(Camera.ViewportSize.X/2, Camera.ViewportSize.Y/2)
    
    for _, p in pairs(Players:GetPlayers()) do
        if IsEnemy(p) and p.Character and IsAlive(p.Character) then
            local head = p.Character:FindFirstChild("Head")
            local hrpTgt = p.Character:FindFirstChild("HumanoidRootPart")
            if head and hrpTgt then
                local distWorld = (hrp.Position - head.Position).Magnitude
                
                -- PRIORIDADE MÁXIMA: Alvos super próximos (5 studs) - Mira INSTANTÂNEA
                if distWorld <= FOV_INSTANT_RANGE then
                    if IsHeadVisible(hrp.Position, p.Character) then
                        if distWorld < instantDist then
                            instantDist = distWorld
                            instantTarget = p.Character
                        end
                    end
                -- PRIORIDADE ALTA: Alvos próximos (8 studs) - Mira instantânea
                elseif distWorld <= FOV_CLOSE_DIST then
                    if IsHeadVisible(hrp.Position, p.Character) then
                        if distWorld < closeDist then
                            closeDist = distWorld
                            closeTarget = p.Character
                        end
                    end
                else
                    -- PRIORIDADE NORMAL: Verifica se está dentro do FOV na tela
                    local headPos, onScreen = Camera:WorldToScreenPoint(head.Position)
                    if onScreen then
                        local screenPos = Vector2.new(headPos.X, headPos.Y)
                        local distScreen = (screenPos - screenCenter).Magnitude
                        if distScreen <= FOV_SIZE/2 and IsHeadVisible(hrp.Position, p.Character) then
                            if distWorld < bestDist then
                                bestDist = distWorld
                                best = p.Character
                            end
                        end
                    end
                end
            end
        end
    end
    
    -- Retorna por prioridade: super próximo > próximo > FOV normal
    return instantTarget or closeTarget or best
end

local function PredictTargetPosition(target, distance)
    local head = target:FindFirstChild("Head")
    local humanoidRootPart = target:FindFirstChild("HumanoidRootPart")
    local humanoid = target:FindFirstChildOfClass("Humanoid")
    
    if not head or not humanoidRootPart or not humanoid then return head.Position end
    
    -- Calcula predição baseada na velocidade do movimento
    local velocity = humanoidRootPart.Velocity
    local speed = velocity.Magnitude
    
    if speed > 2 then -- Se o alvo está se movendo
        -- Calcula tempo estimado para a bala chegar (baseado na distância)
        local bulletTime = distance / 500 -- Assumindo velocidade média de bala
        
        -- Predição básica de movimento
        local predictedPos = head.Position + (velocity * bulletTime * 0.3) -- Fator de predição ajustável
        return predictedPos
    end
    
    return head.Position
end

local function GetSmoothAimPosition(currentCFrame, targetPos, distance)
    local currentPos = currentCFrame.Position
    local currentLookVector = currentCFrame.LookVector
    local targetDirection = (targetPos - currentPos).Unit
    
    -- Calcula suavização baseada na distância
    local smoothFactor = 1.0
    if distance <= FOV_INSTANT_RANGE then
        smoothFactor = 1.0 -- Instantâneo para muito próximo
    elseif distance <= FOV_CLOSE_DIST then
        smoothFactor = 0.95 -- Quase instantâneo para próximo
    else
        smoothFactor = 0.85 -- Leve suavização para distante
    end
    
    -- Interpola entre direção atual e direção do alvo
    local newLookVector = currentLookVector:lerp(targetDirection, smoothFactor)
    
    return CFrame.new(currentPos, currentPos + newLookVector)
end

local function FOVAimbotThread()
    while FOV_AIMBOT_ACTIVE and RunService.RenderStepped:Wait(FOV_AIMBOT_CHECK_INTERVAL) do
        local lpchar = LocalPlayer.Character
        local aliveMe = IsAlive(lpchar)
        
        if not aliveMe then
            CurrentTarget = nil
            continue
        end
        
        local myHRP = lpchar:FindFirstChild("HumanoidRootPart")
        if not myHRP then continue end
        
        -- Verifica se o alvo atual ainda é válido e visível
        local currentTargetValid = false
        local currentTargetDistance = math.huge
        
        if CurrentTarget and IsAlive(CurrentTarget) and CurrentTarget:FindFirstChild("Head") then
            currentTargetDistance = (myHRP.Position - CurrentTarget.Head.Position).Magnitude
            if IsHeadVisible(myHRP.Position, CurrentTarget) then
                currentTargetValid = true
            end
        end
        
        -- Busca novo alvo (sempre verifica se há alguém mais próximo)
        local newTarget = FindFOVAimbotTarget()
        local shouldSwitchTarget = false
        
        if newTarget and IsAlive(newTarget) and newTarget:FindFirstChild("Head") then
            local newTargetDistance = (myHRP.Position - newTarget.Head.Position).Magnitude
            
            -- FORÇA TROCA se novo alvo está muito mais próximo (dentro dos ranges instantâneos)
            if newTargetDistance <= FOV_INSTANT_RANGE then
                -- Se novo alvo está em range super próximo, SEMPRE troca
                shouldSwitchTarget = true
            elseif newTargetDistance <= FOV_CLOSE_DIST and currentTargetDistance > FOV_CLOSE_DIST then
                -- Se novo alvo está em range próximo e atual não está, troca
                shouldSwitchTarget = true
            elseif not currentTargetValid then
                -- Se alvo atual não é mais válido, troca
                shouldSwitchTarget = true
            elseif newTarget ~= CurrentTarget and newTargetDistance < currentTargetDistance * 0.7 then
                -- Se novo alvo é significativamente mais próximo, troca
                shouldSwitchTarget = true
            end
        elseif not currentTargetValid then
            -- Se não há novo alvo e atual não é válido, limpa
            CurrentTarget = nil
        end
        
        if shouldSwitchTarget then
            CurrentTarget = newTarget
        end
        
        -- Mira no alvo se ele existir e for visível
        if CurrentTarget and CurrentTarget:FindFirstChild("Head") and IsAlive(CurrentTarget) then
            -- Dupla verificação de visibilidade antes de mirar
            if IsHeadVisible(myHRP.Position, CurrentTarget) then
                local headPos = CurrentTarget.Head.Position
                local targetDistance = (myHRP.Position - headPos).Magnitude
                
                -- Usa predição de movimento para alvos em movimento
                local predictedPos = PredictTargetPosition(CurrentTarget, targetDistance)
                
                -- Aplica offset facial baseado na direção que o alvo está olhando
                local headCFrame = CurrentTarget.Head.CFrame
                local lookVec = headCFrame.LookVector
                local facePos = predictedPos - lookVec * FACE_OFFSET
                
                -- Usa suavização inteligente baseada na distância
                local smoothedCFrame = GetSmoothAimPosition(Camera.CFrame, facePos, targetDistance)
                Camera.CFrame = smoothedCFrame
            else
                -- Se perdeu visibilidade, limpa o alvo atual para buscar novo na próxima iteração
                CurrentTarget = nil
            end
        end
    end
end

-- ============ UI SYSTEM ============

local function CreateButtons()
    local ScreenGui = Instance.new("ScreenGui")
    ScreenGui.Name = "FOVAimbotESPUI"
    ScreenGui.Parent = game:GetService("CoreGui")
    ScreenGui.ResetOnSpawn = false
    
    local function makeBtn(name, posY, text, color, fontColor)
        local btn = Instance.new("TextButton")
        btn.Name = name
        btn.Size = UDim2.new(0, 90, 0, 40)
        btn.Position = UDim2.new(0.85, 0, posY, 0)
        btn.TextSize = 11
        btn.BackgroundColor3 = color
        btn.TextColor3 = fontColor or Color3.new(1, 1, 1)
        btn.BorderSizePixel = 0
        btn.ZIndex = 1000
        btn.Text = text
        btn.Font = Enum.Font.GothamBold
        btn.Parent = ScreenGui
        
        -- Sistema de arrastar
        local isDragging = false
        local startPos = nil
        local startTime = 0
        
        btn.InputBegan:Connect(function(input)
            if input.UserInputType == Enum.UserInputType.Touch or input.UserInputType == Enum.UserInputType.MouseButton1 then
                isDragging = false
                startPos = input.Position
                startTime = tick()
            end
        end)
        
        btn.InputChanged:Connect(function(input)
            if input.UserInputType == Enum.UserInputType.Touch or input.UserInputType == Enum.UserInputType.MouseMovement then
                if startPos and (input.Position - startPos).Magnitude > 10 then
                    isDragging = true
                    local delta = input.Position - startPos
                    btn.Position = UDim2.new(0, btn.Position.X.Offset + delta.X, 0, btn.Position.Y.Offset + delta.Y)
                    startPos = input.Position
                end
            end
        end)
        
        btn.InputEnded:Connect(function(input)
            if input.UserInputType == Enum.UserInputType.Touch or input.UserInputType == Enum.UserInputType.MouseButton1 then
                if not isDragging and tick() - startTime < 0.3 then
                    if btn == FOVAimbotButton then
                        ToggleFOVAimbot()
                    elseif btn == ESPButton then
                        ToggleESP()
                    end
                end
                isDragging = false
                startPos = nil
            end
        end)
        
        return btn
    end
    
    FOVAimbotButton = makeBtn("FOVAimbotButton", 0.1, "♨️ OFF\nFOV Aimbot\n↔️ Arrastar", Color3.fromRGB(70, 0, 80), Color3.fromRGB(1, 0.7, 1))
    ESPButton = makeBtn("ESPButton", 0.18, "👁️ OFF\nESP Inimigos\n↔️ Arrastar", Color3.fromRGB(40, 40, 40), Color3.new(1, 1, 1))
end

function ToggleFOVAimbot()
    if not FOV_AIMBOT_ACTIVE then
        FOV_AIMBOT_ACTIVE = true
        FOVAimbotButton.Text = "♨️ ON\nFOV Aimbot\n↔️ Arrastar"
        FOVAimbotButton.BackgroundColor3 = Color3.fromRGB(120, 0, 140)
        coroutine.wrap(FOVAimbotThread)()
    else
        FOV_AIMBOT_ACTIVE = false
        FOVAimbotButton.Text = "♨️ OFF\nFOV Aimbot\n↔️ Arrastar"
        FOVAimbotButton.BackgroundColor3 = Color3.fromRGB(70, 0, 80)
        CurrentTarget = nil
    end
end

function ToggleESP()
    if not ESP_ACTIVE then
        ESP_ACTIVE = true
        ESPButton.Text = "👁️ ON\nESP Inimigos\n↔️ Arrastar"
        ESPButton.BackgroundColor3 = Color3.fromRGB(0, 80, 0)
        
        -- Debug: Mostra detecção de times
        local myTeam = GetPlayerTeam(LocalPlayer)
        print("🔍 ESP ATIVADO - Meu time detectado: " .. myTeam)
        print("📋 Lista de jogadores e seus times:")
        for _, player in pairs(Players:GetPlayers()) do
            if player ~= LocalPlayer then
                local theirTeam = GetPlayerTeam(player)
                local isEnemyStatus = IsEnemy(player) and "✅ INIMIGO" or "❌ ALIADO"
                print("   " .. player.Name .. " - Time: " .. theirTeam .. " - Status: " .. isEnemyStatus)
            end
        end
        
    else
        ESP_ACTIVE = false
        ESPButton.Text = "👁️ OFF\nESP Inimigos\n↔️ Arrastar"
        ESPButton.BackgroundColor3 = Color3.fromRGB(40, 40, 40)
    end
    UpdateESP()
end

-- ============ INICIALIZAÇÃO ============

local function Initialize()
    CreateButtons()
    
    -- Eventos de jogadores
    Players.PlayerAdded:Connect(function(player)
        player.CharacterAdded:Connect(function()
            task.wait(2)
            if ESP_ACTIVE then UpdateESP() end
        end)
    end)
    
    Players.PlayerRemoving:Connect(function(player)
        RemoveESP(player)
    end)
    
    -- Eventos do jogador local
    LocalPlayer.CharacterAdded:Connect(function()
        task.wait(3)
        if ESP_ACTIVE then UpdateESP() end
    end)
    
    -- Loop de atualização automática
    coroutine.wrap(function()
        while task.wait(5) do
            if ESP_ACTIVE then UpdateESP() end
        end
    end)()
end

-- Inicializar
Initialize()

print("🎯 FOV AIMBOT + ESP ULTRA SIMPLIFICADO LOADED ✅")
print("📋 FUNCIONALIDADES:")
print("   ♨️ FOV Aimbot: Mira automática na cabeça dos inimigos")
print("   👁️ ESP: Mostra apenas jogadores inimigos")
print("   🤖 Sistema inteligente: Detecta times automaticamente")
print("   🎮 Se não há times (FIFA): Todos são considerados inimigos")
print("   📱 Botões arrastáveis para personalizar posição")
