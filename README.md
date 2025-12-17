-- Script Local (StarterPlayer > StarterPlayerScripts)
local player = game.Players.LocalPlayer
local playerGui = player:WaitForChild("PlayerGui")

-- Pastas
local spawnFolder = workspace:WaitForChild("Map"):WaitForChild("Spawn")
local herbsFolder = spawnFolder:WaitForChild("Herbs")
local manualsFolder = spawnFolder:WaitForChild("Manuals")

-- Serviços
local RunService = game:GetService("RunService")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local UserInputService = game:GetService("UserInputService")

-- Estados SEPARADOS
local espHerbsEnabled = false
local espManualsEnabled = false
local autoTeleportEnabled = false
local autoCollectEnabled = false
local espGuis = {}
local collecting = false

-- Estados para sistema de combate
local teleportNPCEnabled = false
local attackEnabled = false
local teleportConnection = nil
local attackConnection = nil
local facingConnection = nil
local lastTeleportTime = 0
local lastAttackTime = 0
local currentNPCIndex = 1
local teleportDelay = 0.3

-- Lista de pastas de NPCs para combate
local NPC_FOLDERS = {
    workspace.NPC.Spawns["Ancient Trainer"],
    workspace.NPC.Spawns.Bandit,
    workspace.NPC.Spawns["Bandit Boss"],
    workspace.NPC.Spawns["Corrupted Boss Monk"],
    workspace.NPC.Spawns["Demonic Sect Trainee"]
}

-- Debug
print("=== SISTEMA RAYFIELD FARM HELPER ===")

-- =================================================================
-- FUNÇÕES DO SISTEMA ORIGINAL (FARM)
-- =================================================================

-- Função para obter HumanoidRootPart
local function getHumanoidRootPart()
    local character = player.Character
    if character and character:FindFirstChild("HumanoidRootPart") then
        return character.HumanoidRootPart
    end
    return nil
end

-- Função para obter Character
local function getCharacter()
    return player.Character
end

-- Função para encontrar ProximityPrompt em um objeto
local function findProximityPrompt(obj)
    local prompt = obj:FindFirstChildWhichIsA("ProximityPrompt")
    if not prompt then
        for _, child in pairs(obj:GetDescendants()) do
            if child:IsA("ProximityPrompt") then
                prompt = child
                break
            end
        end
    end
    return prompt
end

-- Função para configurar TODOS os ProximityPrompts para COLETA INSTANTÂNEA
local function configureAllPrompts()
    local allObjects = {}
    
    -- Procurar em Herbs
    for _, part in pairs(herbsFolder:GetChildren()) do
        if part:IsA("Part") or part:IsA("BasePart") or part:IsA("Model") then
            for _, child in pairs(part:GetChildren()) do
                if child:IsA("Model") then
                    local prompt = findProximityPrompt(child)
                    if prompt then
                        -- CONFIGURAÇÃO ULTRA RÁPIDA
                        prompt.HoldDuration = 0  -- INSTANTÂNEO
                        prompt.MaxActivationDistance = 200  -- Distância muito maior
                        prompt.RequiresLineOfSight = false
                        prompt.ClickablePrompt = true
                        
                        table.insert(allObjects, {
                            Object = child,
                            Prompt = prompt,
                            Name = child.Name,
                            Type = "Herb"
                        })
                    end
                end
            end
        end
    end
    
    -- Procurar em Manuals
    for _, part in pairs(manualsFolder:GetChildren()) do
        if part:IsA("Part") or part:IsA("BasePart") or part:IsA("Model") then
            for _, child in pairs(part:GetChildren()) do
                if child:IsA("Model") then
                    local prompt = findProximityPrompt(child)
                    if prompt then
                        -- CONFIGURAÇÃO ULTRA RÁPIDA
                        prompt.HoldDuration = 0  -- INSTANTÂNEO
                        prompt.MaxActivationDistance = 200
                        prompt.RequiresLineOfSight = false
                        prompt.ClickablePrompt = true
                        
                        table.insert(allObjects, {
                            Object = child,
                            Prompt = prompt,
                            Name = child.Name,
                            Type = "Manual"
                        })
                    end
                end
            end
        end
    end
    
    return allObjects
end

-- Função para COLETA ULTRA RÁPIDA (todos de uma vez)
local function instantCollectAll()
    if not autoCollectEnabled or collecting then return end
    
    collecting = true
    
    local allObjects = configureAllPrompts()
    local collectedCount = 0
    
    -- Método 1: Ativar todos os prompts de uma vez (mais rápido)
    for _, obj in pairs(allObjects) do
        if obj.Prompt and obj.Prompt:IsA("ProximityPrompt") then
            -- Ativar o prompt INSTANTANEAMENTE
            fireproximityprompt(obj.Prompt)
            collectedCount = collectedCount + 1
            
            -- MÉTODO ALTERNATIVO para garantir
            task.spawn(function()
                obj.Prompt:InputHoldBegin()
                task.wait(0)
                obj.Prompt:InputHoldEnd()
            end)
        end
    end
    
    -- Pequena pausa e repete se ainda estiver ativado
    task.wait(0.1)
    collecting = false
    
    -- Se ainda estiver ativado, continua coletando
    if autoCollectEnabled then
        task.wait(0.05)
        instantCollectAll()
    end
end

-- Função para encontrar TODOS os objetos para ESP
local function findAllObjects()
    local allObjects = {}
    
    -- Procurar em Herbs
    for _, part in pairs(herbsFolder:GetChildren()) do
        if part:IsA("Part") or part:IsA("BasePart") or part:IsA("Model") then
            for _, child in pairs(part:GetChildren()) do
                if child:IsA("Model") then
                    local prompt = findProximityPrompt(child)
                    if prompt then
                        local targetPart = child.PrimaryPart or child:FindFirstChildWhichIsA("BasePart")
                        if targetPart then
                            table.insert(allObjects, {
                                Container = part,
                                Object = child,
                                Type = "Herb",
                                Name = child.Name,
                                Position = targetPart.Position,
                                Prompt = prompt
                            })
                        end
                    end
                end
            end
        end
    end
    
    -- Procurar em Manuals
    for _, part in pairs(manualsFolder:GetChildren()) do
        if part:IsA("Part") or part:IsA("BasePart") or part:IsA("Model") then
            for _, child in pairs(part:GetChildren()) do
                if child:IsA("Model") then
                    local prompt = findProximityPrompt(child)
                    if prompt then
                        local targetPart = child.PrimaryPart or child:FindFirstChildWhichIsA("BasePart")
                        if targetPart then
                            table.insert(allObjects, {
                                Container = part,
                                Object = child,
                                Type = "Manual",
                                Name = child.Name,
                                Position = targetPart.Position,
                                Prompt = prompt
                            })
                        end
                    end
                end
            end
        end
    end
    
    return allObjects
end

-- Criar ESP ScreenGui
local espScreenGui = Instance.new("ScreenGui")
espScreenGui.Name = "ESPScreenGui"
espScreenGui.ResetOnSpawn = false
espScreenGui.Enabled = true
espScreenGui.Parent = playerGui

-- Função para criar ESP
local function createESP(objData, isHerb)
    if not objData or not objData.Object then return end
    if espGuis[objData.Object] then return end
    
    local color = Color3.new(1, 0, 0)  -- Vermelho padrão (Manual)
    local text = "SKILL MANUAL"
    
    if isHerb then
        color = Color3.new(0, 1, 0)  -- Verde para Herb
        text = objData.Name or "HERB"
        
        if string.find(objData.Name:lower(), "rare") then
            color = Color3.new(1, 0.5, 0)
            text = "RARE HERB"
        elseif string.find(objData.Name:lower(), "spiritual") then
            color = Color3.new(0.5, 0, 1)
            text = "SPIRITUAL HERB"
        elseif string.find(objData.Name:lower(), "celestial") then
            color = Color3.new(0, 1, 1)
            text = "CELESTIAL HERB"
        elseif string.find(objData.Name:lower(), "common") then
            color = Color3.new(0, 1, 0)
            text = "COMMON HERB"
        end
    end
    
    local targetPart = objData.Object.PrimaryPart or objData.Object:FindFirstChildWhichIsA("BasePart")
    if not targetPart then return end
    
    local espGui = Instance.new("BillboardGui")
    espGui.Name = "ESP_" .. objData.Object.Name
    espGui.Size = UDim2.new(0, 200, 0, 40)
    espGui.AlwaysOnTop = true
    espGui.MaxDistance = 2000
    espGui.StudsOffset = Vector3.new(0, 4, 0)
    espGui.Adornee = targetPart
    espGui.Parent = espScreenGui
    
    local textLabel = Instance.new("TextLabel")
    textLabel.Size = UDim2.new(1, 0, 1, 0)
    textLabel.BackgroundTransparency = 0.7
    textLabel.BackgroundColor3 = Color3.new(0, 0, 0)
    textLabel.Text = text
    textLabel.TextColor3 = color
    textLabel.TextScaled = true
    textLabel.Font = Enum.Font.SourceSansBold
    textLabel.TextStrokeTransparency = 0
    textLabel.TextStrokeColor3 = Color3.new(0, 0, 0)
    textLabel.Parent = espGui
    
    espGuis[objData.Object] = {
        Gui = espGui,
        Type = isHerb and "Herb" or "Manual"
    }
end

-- Função para atualizar ESP
local function updateESP()
    for _, data in pairs(espGuis) do
        data.Gui:Destroy()
    end
    espGuis = {}
    
    if espHerbsEnabled then
        local allObjects = findAllObjects()
        for _, objData in pairs(allObjects) do
            if objData.Type == "Herb" then
                createESP(objData, true)
            end
        end
    end
    
    if espManualsEnabled then
        local allObjects = findAllObjects()
        for _, objData in pairs(allObjects) do
            if objData.Type == "Manual" then
                createESP(objData, false)
            end
        end
    end
end

-- Função para TELEPORTE (1 segundo entre teleportes)
local function teleportToObject(objData)
    if not objData or not objData.Object then return false end
    
    local character = getCharacter()
    if not character then return false end
    
    local humanoidRootPart = character:FindFirstChild("HumanoidRootPart")
    if not humanoidRootPart then return false end
    
    local targetPart = objData.Object.PrimaryPart or objData.Object:FindFirstChildWhichIsA("BasePart")
    if not targetPart then return false end
    
    local targetPosition = targetPart.Position + Vector3.new(0, 3, 0)
    humanoidRootPart.CFrame = CFrame.new(targetPosition)
    
    return true
end

-- Loop de TELEPORTE
local function autoTeleportLoop()
    if not autoTeleportEnabled then return end
    
    local allObjects = findAllObjects()
    
    for _, objData in pairs(allObjects) do
        if not autoTeleportEnabled then break end
        
        teleportToObject(objData)
        task.wait(1)  -- 1 segundo entre teleportes
    end
end

-- =================================================================
-- FUNÇÕES DO SISTEMA DE COMBATE (NPC TELEPORTE + ATAQUE)
-- =================================================================

-- Funções auxiliares para sistema de combate
local function isValidNPC(npc)
    if not npc or not npc:IsA("Model") then return false end
    if not npc.PrimaryPart then return false end
    local npcHumanoid = npc:FindFirstChild("Humanoid")
    if npcHumanoid then return npcHumanoid.Health > 0 end
    return true
end

local function getDistance(pos1, pos2)
    return (pos1 - pos2).Magnitude
end

local function faceNPC(npc)
    if not npc or not npc.PrimaryPart then return end
    local character = getCharacter()
    if not character then return end
    local rootPart = character:FindFirstChild("HumanoidRootPart")
    if not rootPart then return end
    
    local npcPosition = npc.PrimaryPart.Position
    local playerPosition = rootPart.Position
    local lookDirection = (npcPosition - playerPosition).Unit
    local lookCFrame = CFrame.new(playerPosition, playerPosition + lookDirection)
    rootPart.CFrame = lookCFrame
end

local function maintainFacingNPC(npc)
    if facingConnection then facingConnection:Disconnect() end
    
    facingConnection = RunService.Heartbeat:Connect(function()
        if npc and npc.PrimaryPart then
            faceNPC(npc)
        else
            if facingConnection then
                facingConnection:Disconnect()
                facingConnection = nil
            end
        end
    end)
end

local function teleportBehindAndFaceNPC(npc)
    if not npc or not npc.PrimaryPart then return false end
    local character = getCharacter()
    if not character then return false end
    local rootPart = character:FindFirstChild("HumanoidRootPart")
    if not rootPart then return false end
    
    local npcCFrame = npc.PrimaryPart.CFrame
    local npcLookVector = npcCFrame.LookVector
    local behindPosition = npcCFrame.Position - (npcLookVector * 5)  -- 5 unidades atrás
    local directionToNPC = (npcCFrame.Position - behindPosition).Unit
    local targetCFrame = CFrame.new(behindPosition, behindPosition + directionToNPC)
    
    character:PivotTo(targetCFrame)
    faceNPC(npc)
    return true
end

local function getNPCsInRadius()
    local character = getCharacter()
    if not character then return {} end
    local rootPart = character:FindFirstChild("HumanoidRootPart")
    if not rootPart then return {} end
    
    local playerPos = rootPart.Position
    local validNPCs = {}
    
    for _, folder in ipairs(NPC_FOLDERS) do
        if folder and folder:IsA("Folder") then
            for _, npc in ipairs(folder:GetChildren()) do
                if isValidNPC(npc) then
                    local npcPos = npc.PrimaryPart.Position
                    local distance = getDistance(playerPos, npcPos)
                    if distance <= 100 then  -- 100 unidades de raio
                        table.insert(validNPCs, {npc = npc, distance = distance})
                    end
                end
            end
        end
    end
    
    table.sort(validNPCs, function(a, b) return a.distance < b.distance end)
    return validNPCs
end

local function cycleTeleportBehindNPCs()
    local currentTime = tick()
    if currentTime - lastTeleportTime < teleportDelay then return end
    
    local validNPCs = getNPCsInRadius()
    if #validNPCs == 0 then
        if facingConnection then
            facingConnection:Disconnect()
            facingConnection = nil
        end
        return
    end
    
    local targetNPC = validNPCs[currentNPCIndex]
    if targetNPC and targetNPC.npc then
        if teleportBehindAndFaceNPC(targetNPC.npc) then
            maintainFacingNPC(targetNPC.npc)
            lastTeleportTime = currentTime
            currentNPCIndex = currentNPCIndex + 1
            if currentNPCIndex > #validNPCs then currentNPCIndex = 1 end
        end
    else
        currentNPCIndex = 1
    end
end

-- Função para disparar ataque M1
local function fireM1()
    ReplicatedStorage.Remotes.Events.Combat.Combat:FireServer(table.unpack({
        [1] = "M1",
        [2] = 4,
    }))
end

-- Função para ligar/desligar o teleporte NPC
local function toggleNPCTeleport()
    teleportNPCEnabled = not teleportNPCEnabled
    
    if teleportNPCEnabled then
        teleportConnection = RunService.Heartbeat:Connect(function()
            cycleTeleportBehindNPCs()
        end)
    else
        if teleportConnection then 
            teleportConnection:Disconnect() 
            teleportConnection = nil
        end
        if facingConnection then 
            facingConnection:Disconnect()
            facingConnection = nil
        end
    end
end

-- Função para ligar/desligar o ataque rápido
local function toggleFastAttack()
    attackEnabled = not attackEnabled
    
    if attackEnabled then
        attackConnection = RunService.Heartbeat:Connect(function(deltaTime)
            lastAttackTime = lastAttackTime + deltaTime
            if lastAttackTime >= 0.01 then  -- 0.01 segundos entre ataques
                fireM1()
                lastAttackTime = 0
            end
        end)
    else
        if attackConnection then 
            attackConnection:Disconnect() 
            attackConnection = nil
        end
    end
end

-- =================================================================
-- RAYFIELD UI IMPLEMENTAÇÃO
-- =================================================================

-- Carregar Rayfield
local Rayfield = loadstring(game:HttpGet("https://sirius.menu/rayfield"))()

-- Criar janela principal
local Window = Rayfield:CreateWindow({
    Name = "⚡ Farm Helper Pro",
    LoadingTitle = "Carregando Farm Helper...",
    LoadingSubtitle = "Sistema de Farm Automatizado",
    ConfigurationSaving = {
        Enabled = true,
        FolderName = "FarmHelperConfig",
        FileName = "Settings"
    },
    Discord = {
        Enabled = false,
        Invite = "noinvite",
        RememberJoins = true
    },
    KeySystem = false,
    KeySettings = {
        Title = "Farm Helper",
        Subtitle = "Insira a Key",
        Note = "Entre em contato para obter a key",
        FileName = "FarmKey",
        SaveKey = true,
        GrabKeyFromSite = false,
        Key = {""}
    }
})

-- Tab Principal
local MainTab = Window:CreateTab("Principal", 4483362458) -- Ícone de settings

-- Seção ESP
local EspSection = MainTab:CreateSection("Visual ESP")

local HerbToggle = MainTab:CreateToggle({
    Name = "🌿 ESP para Herbs",
    CurrentValue = false,
    Callback = function(Value)
        espHerbsEnabled = Value
        updateESP()
        Rayfield:Notify({
            Title = "ESP Herbs",
            Content = Value and "Ativado" or "Desativado",
            Duration = 2,
            Image = 4483362458
        })
    end
})

local ManualToggle = MainTab:CreateToggle({
    Name = "📚 ESP para Manuais",
    CurrentValue = false,
    Callback = function(Value)
        espManualsEnabled = Value
        updateESP()
        Rayfield:Notify({
            Title = "ESP Manuais",
            Content = Value and "Ativado" or "Desativado",
            Duration = 2,
            Image = 4483362458
        })
    end
})

-- Seção Coleta
local CollectSection = MainTab:CreateSection("Coleta Automática")

MainTab:CreateToggle({
    Name = "⚡ Coleta Instantânea",
    CurrentValue = false,
    Callback = function(Value)
        autoCollectEnabled = Value
        if Value then
            Rayfield:Notify({
                Title = "Coleta Automática",
                Content = "Iniciando coleta instantânea...",
                Duration = 2,
                Image = 4483362458
            })
            -- Inicia a coleta em uma thread separada
            task.spawn(function()
                while autoCollectEnabled do
                    instantCollectAll()
                    task.wait(0.1)
                end
            end)
        else
            collecting = false
            Rayfield:Notify({
                Title = "Coleta Automática",
                Content = "Coleta parada",
                Duration = 2,
                Image = 4483362458
            })
        end
    end
})


-- Seção Teleporte
local TeleportSection = MainTab:CreateSection("Teleporte Automático")

MainTab:CreateToggle({
    Name = "🚀 Teleporte Automático (Farm)",
    CurrentValue = false,
    Callback = function(Value)
        autoTeleportEnabled = Value
        if Value then
            Rayfield:Notify({
                Title = "Teleporte Automático",
                Content = "Iniciando teleporte entre objetos...",
                Duration = 3,
                Image = 4483362458
            })
            -- Inicia o teleporte em uma thread separada
            task.spawn(function()
                while autoTeleportEnabled do
                    autoTeleportLoop()
                    task.wait(0.5)
                end
            end)
        else
            Rayfield:Notify({
                Title = "Teleporte Automático",
                Content = "Teleporte parado",
                Duration = 2,
                Image = 4483362458
            })
        end
    end
})

-- Seção Configurações
local ConfigSection = MainTab:CreateSection("Configurações")

-- Slider para velocidade de teleporte
local teleportDelay = 1.0
MainTab:CreateSlider({
    Name = "⏱️ Velocidade Teleporte (segundos)",
    Range = {0.1, 5},
    Increment = 0.1,
    Suffix = "s",
    CurrentValue = 1.0,
    Callback = function(Value)
        teleportDelay = Value
        Rayfield:Notify({
            Title = "Configuração",
            Content = "Velocidade do teleporte: " .. Value .. "s",
            Duration = 2,
            Image = 4483362458
        })
    end
})

-- Botão para configurar prompts
MainTab:CreateButton({
    Name = "⚙️ Configurar Prompts",
    Callback = function()
        configureAllPrompts()
        Rayfield:Notify({
            Title = "Configuração",
            Content = "Todos os prompts configurados!",
            Duration = 3,
            Image = 4483362458
        })
    end
})

-- Botão para atualizar ESP
MainTab:CreateButton({
    Name = "🔄 Atualizar ESP",
    Callback = function()
        updateESP()
        Rayfield:Notify({
            Title = "ESP",
            Content = "ESP atualizado!",
            Duration = 2,
            Image = 4483362458
        })
    end
})

-- =================================================================
-- NOVA ABA: COMBATE
-- =================================================================

-- Criar aba de Combate
local CombatTab = Window:CreateTab("Combate", 4483362458)

-- Seção Teleporte NPC
CombatTab:CreateSection("Teleporte NPC")

-- Toggle para teleporte automático atrás de NPCs
CombatTab:CreateToggle({
    Name = "🚀 Teleporte Automático NPC",
    CurrentValue = false,
    Callback = function(Value)
        teleportNPCEnabled = Value
        toggleNPCTeleport()
        Rayfield:Notify({
            Title = "Teleporte NPC",
            Content = Value and "Teleporte NPC ativado!" or "Teleporte NPC desativado!",
            Duration = 2,
            Image = 4483362458
        })
    end
})

-- Slider para velocidade de teleporte NPC
CombatTab:CreateSlider({
    Name = "⚡ Velocidade Teleporte NPC",
    Range = {0.1, 2},
    Increment = 0.1,
    Suffix = "s",
    CurrentValue = 0.3,
    Callback = function(Value)
        teleportDelay = Value
        Rayfield:Notify({
            Title = "Velocidade Teleporte",
            Content = "Velocidade ajustada para: " .. Value .. "s",
            Duration = 2,
            Image = 4483362458
        })
    end
})

-- Seção Ataque
CombatTab:CreateSection("Ataque Rápido")

-- Toggle para ataque rápido M1
CombatTab:CreateToggle({
    Name = "⚔️ Ataque Rápido M1",
    CurrentValue = false,
    Callback = function(Value)
        attackEnabled = Value
        toggleFastAttack()
        Rayfield:Notify({
            Title = "Ataque Rápido",
            Content = Value and "Ataque rápido ativado!" or "Ataque rápido desativado!",
            Duration = 2,
            Image = 4483362458
        })
    end
})

-- Slider para velocidade de ataque
CombatTab:CreateSlider({
    Name = "💨 Velocidade de Ataque",
    Range = {0.001, 0.1},
    Increment = 0.001,
    Suffix = "s",
    CurrentValue = 0.01,
    Callback = function(Value)
        -- Atualiza a velocidade de ataque
        if attackConnection then
            attackConnection:Disconnect()
            lastAttackTime = 0
            attackConnection = RunService.Heartbeat:Connect(function(deltaTime)
                lastAttackTime = lastAttackTime + deltaTime
                if lastAttackTime >= Value then
                    fireM1()
                    lastAttackTime = 0
                end
            end)
        end
        Rayfield:Notify({
            Title = "Velocidade Ataque",
            Content = "Velocidade ajustada para: " .. Value .. "s",
            Duration = 2,
            Image = 4483362458
        })
    end
})

-- Botão para teleportar atrás de todos NPCs de uma vez
CombatTab:CreateButton({
    Name = "🎯 Teleportar Todos NPCs",
    Callback = function()
        local validNPCs = getNPCsInRadius()
        for _, npcData in ipairs(validNPCs) do
            teleportBehindAndFaceNPC(npcData.npc)
            task.wait(0.1)
        end
        Rayfield:Notify({
            Title = "Teleporte Completo",
            Content = "Teleportado atrás de " .. #validNPCs .. " NPCs!",
            Duration = 3,
            Image = 4483362458
        })
    end
})

-- Seção Configurações NPC
CombatTab:CreateSection("Configurações NPC")

-- Verificar pastas de NPCs
CombatTab:CreateButton({
    Name = "🔍 Verificar Pastas NPC",
    Callback = function()
        local foundCount = 0
        local notFound = {}
        
        for i, folder in ipairs(NPC_FOLDERS) do
            if folder then
                foundCount = foundCount + 1
            else
                table.insert(notFound, i)
            end
        end
        
        local message = "Pastas encontradas: " .. foundCount .. "/5\n"
        if #notFound > 0 then
            message = message .. "Pastas não encontradas: " .. table.concat(notFound, ", ")
        end
        
        Rayfield:Notify({
            Title = "Verificação NPC",
            Content = message,
            Duration = 5,
            Image = 4483362458
        })
    end
})

-- Botão para ativar ambos sistemas
CombatTab:CreateButton({
    Name = "🔥 Ativar Sistema Completo",
    Callback = function()
        -- Ativar teleporte NPC
        if not teleportNPCEnabled then
            teleportNPCEnabled = true
            toggleNPCTeleport()
        end
        
        -- Ativar ataque rápido
        if not attackEnabled then
            attackEnabled = true
            toggleFastAttack()
        end
        
        Rayfield:Notify({
            Title = "Sistema Completo",
            Content = "Teleporte NPC e Ataque Rápido ativados!",
            Duration = 3,
            Image = 4483362458
        })
    end
})

-- Seção Utilitários
local UtilSection = MainTab:CreateSection("Utilitários")

-- Botão para limpar ESP
MainTab:CreateButton({
    Name = "🧹 Limpar ESP",
    Callback = function()
        for _, data in pairs(espGuis) do
            data.Gui:Destroy()
        end
        espGuis = {}
        Rayfield:Notify({
            Title = "Limpeza",
            Content = "ESP limpo com sucesso!",
            Duration = 2,
            Image = 4483362458
        })
    end
})

-- Botão para informações
MainTab:CreateButton({
    Name = "ℹ️ Informações do Sistema",
    Callback = function()
        local objectCount = #findAllObjects()
        local npcCount = 0
        for _, folder in ipairs(NPC_FOLDERS) do
            if folder then
                npcCount = npcCount + #folder:GetChildren()
            end
        end
        
        Rayfield:Notify({
            Title = "Informações do Sistema",
            Content = "Objetos Farm: " .. objectCount .. 
                     "\nNPCs Encontrados: " .. npcCount .. 
                     "\nESP Herbs: " .. (espHerbsEnabled and "ON" or "OFF") .. 
                     "\nESP Manuais: " .. (espManualsEnabled and "ON" or "OFF") ..
                     "\nTeleporte NPC: " .. (teleportNPCEnabled and "ON" or "OFF") ..
                     "\nAtaque Rápido: " .. (attackEnabled and "ON" or "OFF"),
            Duration = 6,
            Image = 4483362458
        })
    end
})

-- Tab de Ajustes
local SettingsTab = Window:CreateTab("Configurações", 4483362458)

SettingsTab:CreateSection("Configurações da Interface")

-- Toggle para salvar configurações
SettingsTab:CreateToggle({
    Name = "💾 Salvar Configurações",
    CurrentValue = true,
    Callback = function(Value)
        Window.ConfigurationSaving.Enabled = Value
        Rayfield:Notify({
            Title = "Configurações",
            Content = Value and "Salvamento ativado" or "Salvamento desativado",
            Duration = 2,
            Image = 4483362458
        })
    end
})

-- Botão para destruir interface
SettingsTab:CreateButton({
    Name = "⚠️ Destruir Interface",
    Callback = function()
        Rayfield:Destroy()
        Rayfield:Notify({
            Title = "Interface",
            Content = "Interface destruída!",
            Duration = 3,
            Image = 4483362458
        })
    end
})

-- =================================================================
-- SISTEMA DE ATUALIZAÇÃO AUTOMÁTICA
-- =================================================================

-- Atualizar ESP periodicamente
task.spawn(function()
    while true do
        task.wait(2) -- Atualiza a cada 2 segundos
        if espHerbsEnabled or espManualsEnabled then
            updateESP()
        end
    end
end)

-- Monitorar quando novas Parts são adicionadas
task.spawn(function()
    while true do
        task.wait(1)
        herbsFolder.ChildAdded:Connect(function()
            task.wait(0.5)
            configureAllPrompts()
        end)
        
        manualsFolder.ChildAdded:Connect(function()
            task.wait(0.5)
            configureAllPrompts()
        end)
    end
end)

-- =================================================================
-- INICIALIZAÇÃO DO SISTEMA
-- =================================================================

-- Configuração inicial
task.wait(1)
configureAllPrompts()

Rayfield:Notify({
    Title = "Farm Helper Pro",
    Content = "Sistema carregado com sucesso!",
    Duration = 5,
    Image = 4483362458
})

print("=== FARM HELPER RAYFIELD PRONTO ===")
print("Use a tecla RightShift para abrir/fechar o menu")
print("Interface organizada em abas e seções")

-- Sistema de tecla para abrir/fechar
local menuVisible = true

UserInputService.InputBegan:Connect(function(input, gameProcessed)
    if not gameProcessed then
        if input.KeyCode == Enum.KeyCode.RightShift then
            menuVisible = not menuVisible
            if menuVisible then
                Rayfield:Notify({
                    Title = "Menu",
                    Content = "Menu mostrado (RightShift para esconder)",
                    Duration = 2,
                    Image = 4483362458
                })
            else
                Rayfield:Notify({
                    Title = "Menu",
                    Content = "Menu escondido (RightShift para mostrar)",
                    Duration = 2,
                    Image = 4483362458
                })
            end
        end
    end
end)

-- Limpeza quando necessário
game:GetService("Players").LocalPlayer.CharacterRemoving:Connect(function()
    autoTeleportEnabled = false
    autoCollectEnabled = false
    teleportNPCEnabled = false
    attackEnabled = false
    collecting = false
    
    if teleportConnection then teleportConnection:Disconnect() end
    if attackConnection then attackConnection:Disconnect() end
    if facingConnection then facingConnection:Disconnect() end
    
    for _, data in pairs(espGuis) do
        if data.Gui then
            data.Gui:Destroy()
        end
    end
    espGuis = {}
end)

-- Reconectar quando o player spawna
player.CharacterAdded:Connect(function()
    task.wait(2)
    if espHerbsEnabled or espManualsEnabled then
        updateESP()
    end
    configureAllPrompts()
    
    -- Reativar sistemas se estavam ativos
    if teleportNPCEnabled then
        toggleNPCTeleport()
    end
    if attackEnabled then
        toggleFastAttack()
    end
end)
