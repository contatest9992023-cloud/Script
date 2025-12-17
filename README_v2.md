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
local autoCollectHerbsEnabled = false
local autoCollectManualsEnabled = false
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
-- FUNÇÕES AUXILIARES GERAIS
-- =================================================================

-- Função para obter HumanoidRootPart
local function getHumanoidRootPart()
    local character = player.Character
    if character and character:FindFirstChild("HumanoidRootPart") then
        return character.HumanoidRootPart
    end
    return nil
end

-- Função para obter distância
local function getDistance(pos1, pos2)
    return (pos1 - pos2).Magnitude
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

-- =================================================================
-- MELHORIAS NO SISTEMA DE COLETA
-- =================================================================

-- Função para configurar prompts de coleta
local function configurePrompts(folder, typeName, maxDistance)
    local allObjects = {}
    for _, part in pairs(folder:GetChildren()) do
        if part:IsA("Part") or part:IsA("BasePart") or part:IsA("Model") then
            for _, child in pairs(part:GetChildren()) do
                if child:IsA("Model") then
                    local prompt = findProximityPrompt(child)
                    if prompt then
                        -- Configuração para 10 metros
                        prompt.HoldDuration = 0
                        prompt.MaxActivationDistance = maxDistance or 10
                        prompt.RequiresLineOfSight = false
                        prompt.ClickablePrompt = true

                        table.insert(allObjects, {
                            Object = child,
                            Prompt = prompt,
                            Name = child.Name,
                            Type = typeName
                        })
                    end
                end
            end
        end
    end
    return allObjects
end

-- Função para coleta de Herbs
local function collectHerbs()
    if not autoCollectHerbsEnabled or collecting then return end
    collecting = true

    local allObjects = configurePrompts(herbsFolder, "Herb", 10)
    for _, obj in pairs(allObjects) do
        if obj.Prompt then
            fireproximityprompt(obj.Prompt)
        end
    end

    collecting = false
end

-- Função para coleta de Manuais
local function collectManuals()
    if not autoCollectManualsEnabled or collecting then return end
    collecting = true

    local allObjects = configurePrompts(manualsFolder, "Manual", 10)
    for _, obj in pairs(allObjects) do
        if obj.Prompt then
            fireproximityprompt(obj.Prompt)
        end
    end

    collecting = false
end

-- =================================================================
-- MELHORIAS NO SISTEMA ESP
-- =================================================================

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

    local color = isHerb and Color3.new(0, 1, 0) or Color3.new(1, 0, 0)
    local text = isHerb and "Herb" or "Manual"

    local targetPart = objData.Object.PrimaryPart or objData.Object:FindFirstChild("Part")
    if not targetPart then return end

    local espGui = Instance.new("BillboardGui")
    espGui.Name = "ESP_" .. objData.Object.Name
    espGui.Size = UDim2.new(0, 100, 0, 25) -- Redimensionado para menor tamanho
    espGui.AlwaysOnTop = true
    espGui.MaxDistance = 500
    espGui.StudsOffset = Vector3.new(0, 3, 0)
    espGui.Adornee = targetPart
    espGui.Parent = espScreenGui

    local textLabel = Instance.new("TextLabel")
    textLabel.Size = UDim2.new(1, 0, 1, 0)
    textLabel.BackgroundTransparency = 1
    textLabel.Text = text
    textLabel.TextColor3 = color
    textLabel.TextScaled = true
    textLabel.Font = Enum.Font.SourceSansBold
    textLabel.Parent = espGui

    espGuis[objData.Object] = {
        Gui = espGui,
        Type = isHerb and "Herb" or "Manual"
    }
end

-- Atualizar ESP
local function updateESP()
    for _, data in pairs(espGuis) do
        data.Gui:Destroy()
    end
    espGuis = {}

    if espHerbsEnabled then
        for _, objData in pairs(configurePrompts(herbsFolder, "Herb", 500)) do
            createESP(objData, true)
        end
    end

    if espManualsEnabled then
        for _, objData in pairs(configurePrompts(manualsFolder, "Manual", 500)) do
            createESP(objData, false)
        end
    end
end

-- =================================================================
-- MELHORIAS NO SISTEMA DE TELEPORTE NPC
-- =================================================================

-- Obter NPCs no mapa inteiro
local function getAllNPCs()
    local allNPCs = {}
    for _, folder in pairs(NPC_FOLDERS) do
        if folder and folder:IsA("Folder") then
            for _, npc in pairs(folder:GetChildren()) do
                table.insert(allNPCs, npc)
            end
        end
    end
    return allNPCs
end

-- Teleportar para NPC aleatório no mapa
local function teleportToNPC()
    if not teleportNPCEnabled then return end

    local allNPCs = getAllNPCs()
    for _, npc in pairs(allNPCs) do
        if npc and npc:IsA("Model") and npc.PrimaryPart then
            local character = getCharacter()
            local rootPart = character:FindFirstChild("HumanoidRootPart")
            if rootPart then
                rootPart.CFrame = npc.PrimaryPart.CFrame + Vector3.new(0, 3, 0)
            end
        end
    end
end

return true